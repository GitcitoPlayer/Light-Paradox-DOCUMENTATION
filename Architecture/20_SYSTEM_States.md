# 20 — Sistema: States (Estados)
### Sistema: States — Sistema central de Estados de Light Paradox
### Base Asset: EasySurvivalRPGv5
### Fuente: Diseño del cliente + sesión Light Paradox
### Proyecto: Light Paradox · UE 5.4.4

---

## Contexto

El sistema de Estados es la capa de abstracción central sobre el sistema de
Status Effects de ESRPGv5. Un Estado es un contenedor de datos que agrupa
uno o más efectos y define el contexto y condiciones de su aplicación.

**Motivación:** Reemplazar la construcción manual de nodos en triggers, enemigos,
trampas, consumibles y runas por una sola llamada a `BP_StateApplier`.
El sistema es el punto de entrada único para aplicar cualquier conjunto de
efectos en cualquier contexto del juego.

---

## Arquitectura del sistema

```
[Cualquier fuente: trigger, enemigo, trampa, consumible, runa]
        │
        ▼
  BP_StateApplier (Blueprint Function Library)
    → Lee DT_States (fila del Estado)
    → Evalúa Rate (probabilidad)
    → Si pasa:
        → Itera Effects array
        → Verifica AffectedFactions
        → Lee Duration de DT_StatusEffects
        → Aplica IsPermanent / EffectsDurationOverride
        → Llama BP_AbilitySystem.Load Status Effect (Target)
        → Guarda Effect Handle aplicado en output Applied Effect Handles
```

> **Nota de arquitectura confirmada esta sesión:** `DT_States.Effects` ya es
> un array de `DataTableRowHandle` apuntando a `DT_StatusEffects` — es decir,
> `ApplyState` resuelve la capa de State internamente. Lo que finalmente se
> registra en `Active Status Effects` (y lo que se compara al remover) es
> siempre el **Effect Handle**, nunca el State Handle, sin importar si el
> disparador original fue un State (`BP_StateApplier`) o un Effect aplicado
> de forma directa (patrón `BP_PoisonTrigger`). Ambos caminos son
> compatibles entre sí para efectos de remoción manual.

---

## STR_StateData — Row Struct

Asset: `STR_StateData`

| Campo | Tipo | Descripción |
|---|---|---|
| `Effects` | Array de DataTableRowHandle | Filas de `DT_StatusEffects` que componen este Estado. |
| `IsHitEffect` | Boolean | True = Estado ofensivo, se aplica al golpear. False = Estado pasivo. |
| `Rate` | Float (0–100) | Probabilidad porcentual de que el Estado se ejecute al activarse. |
| `Duration` | Float | Duración en segundos. Usado cuando EffectsDurationOverride = True. |
| `EffectsDurationOverride` | Boolean | True = usar Duration de este Estado para todos los efectos. |
| `IsPermanent` | Boolean | True = el Estado no expira. Time Remaining se pasa como 0.0. |
| `AffectedFactions` | Array de E_Faction | Facciones que pueden recibir este Estado. Vacío = cualquier actor. |

---

## DT_States — Data Table

Asset: `DT_States` — Row Struct: `STR_StateData`

| Row Name | Effects | IsHitEffect | Rate | Duration | EffectsDurationOverride | IsPermanent | AffectedFactions |
|---|---|---|---|---|---|---|---|
| `State_Poison` | `[Effect_Poison]` | False | 100 | 10.0 | True | False | [] |
| `State_Freeze` | `[Effect_Slow]` | False | 100 | 10.0 | True | False | [] |
| `State_PoisonHit` | `[Effect_Poison]` | True | 25 | 10.0 | True | False | [] |

---

## BP_StateApplier — Blueprint Function Library

**Tipo:** Blueprint Function Library
**Estado:** ✅ Funcional para aplicación. El output `AppliedEffectHandles`
confirmado funcional (sesión anterior) y confirmado nuevamente como no
relacionado al Bug 3 (esta sesión).

> **Nota crítica:** BP_StateApplier NO debe ser un Actor.

### Función: ApplyState

**Inputs:** `StateRowHandle`, `Target`, `Instigator`
**Outputs:** `AppliedEffectHandles` (Array de DataTableRowHandle)

### Flujo interno confirmado (con guard anti-duplicado)

```
Entry (StateRowHandle, Target, Instigator)
  → Get Data Table Row (DT_States)
      Row Found →
          Random Integer <= Rate → Branch
              True →
                Local Variable: AppliedEffectHandles
                For Each Loop (Effects)
                  → Faction check → [guard + aplicar efecto]

                [guard + aplicar efecto]
                  Check Status Effect By Handle (ASC, Handle)
                  → Branch (Result)
                      True  → ya activo, NO reaplica, NO hace Append
                      False → Load Status Effect → Append (AppliedEffectHandles)

                Completed → Return Node (Applied Effect Handles)
```

> **✅ CONFIRMADO:** en la primera aplicación de un Estado (guard = False,
> efecto no estaba activo), `AppliedEffectHandles` sale con longitud 1
> directamente del output de `Apply State`. **`ApplyState` no es ni fue la
> fuente del Bug 3.**

---

## Estados en Rune Words — Fase 4

### Structs y variables

**STR_RuneStateHandles** — `Handles`: Array de DataTableRowHandle
**ActiveRuneStates** en BP_Character_Player — Map (Integer → STR_RuneStateHandles), Category: `State`

### Flujo en EquipmentChanged (✅ actualizado con el fix del Bug 3)

```
[después de Update Equipment Attributes]

BLOQUE A — Remover Estados de runa anterior:
  FIND (ActiveRuneStates, Key: Slot)
    → Break STR_RuneStateHandles → Handles
    → For Each Loop
        Loop Body → Remove Status Effect By Handle (Array Element)
        Completed → Branch (Item Is Valid)
            False → REMOVE (Map: ActiveRuneStates, Key: Slot)

BLOQUE B — Aplicar Estados de runa nueva:
  Branch (Item Is Valid)
    True →
        Break STR_ItemData → Item States → For Each Loop
            Loop Body →
              Apply State (State Row Handle: Array Element, Target: Self)
                → Applied Effect Handles
              → Break STR_RuneStateHandles (del FIND) → Handles
              → SET LocalHandlesToSave ← Handles (del Break)          ✅ FIX
              → Append (Target Array: GET LocalHandlesToSave,
                        Source: Applied Effect Handles)                ✅ FIX
              → Make STR_RuneStateHandles
                  Handles: GET LocalHandlesToSave                      ✅ FIX
              → Add (Map: ActiveRuneStates, Key: Slot, Value: resultado del Make)
```

---

## Bug 1 — Estados de runa no se aplican al equipar

**Estado: ✅ RESUELTO.** Causa: pin `Item States` desconectado en `Make Item (pure)`
(`BP_ItemsLibrary`). Resuelto conectando el pin.

---

## Bug 2 — Segundo slot de runa no aplica atributos de equipamiento

**Estado:** ⏳ Pendiente — prioridad para la próxima sesión, ahora que el Bug 3 está cerrado.
**Síntoma:** Al desbloquear el segundo slot de runa (HeadRuneSlot_1) y colocar
la misma runa, los `EquipmentAttributes` no se aplican. Aparece el debug log
"No es runa" (Print String temporal en `UI_ItemSlot.OnDrop`).
**Requiere:** inspección de `UI_ItemSlot.OnDrop` y la lógica de detección de
runas por Gameplay Tag (`Does Container Match Tag Query`).

---

## Bug 3 — El efecto no se remueve al desequipar la runa ✅ RESUELTO

**Estado final: ✅ CONFIRMADO RESUELTO esta sesión**, con fix aplicado y
verificado por el usuario en juego (el efecto de veneno se detiene
correctamente al desequipar la runa, tanto en configuración permanente como
no permanente).

### Síntomas originales (ya no reproducibles tras el fix)

| Configuración | Comportamiento antes del fix |
|---|---|
| `IsPermanent = True` | Al quitar la runa, el efecto seguía activo indefinidamente. |
| `IsPermanent = False` | El efecto no se detenía al desequipar — seguía corriendo hasta expirar por su propio Duration natural. |

### Causa raíz confirmada — pure function reevaluada, no variable persistente

El pin `Handles` de dos nodos distintos en Bloque B —`APPEND` (Target Array)
y `Make STR_RuneStateHandles` (pin `Handles`)— estaban conectados **ambos,
por separado, directamente al mismo pin de salida** de
`Break STR_RuneStateHandles`, una **función pura** (sin pines de exec).

Una función pura no tiene memoria ni un único momento de evaluación: se
re-ejecuta de forma independiente cada vez que un pin la consulta. Cuando
`APPEND` tomaba ese `Handles` como su `Target Array` y agregaba el nuevo
elemento, la modificación ocurría sobre una **copia temporal**, propia de esa
ejecución de `APPEND` — no se guardaba en ningún lado persistente, porque el
origen (`Break`) no era una variable real. Cuando `Make STR_RuneStateHandles`
leía su propio `Handles` desde el mismo pin de `Break`, Unreal volvía a
evaluar la función pura desde cero, trayendo el array **original, sin el
elemento agregado por `Append`**. Por eso el struct guardado en
`ActiveRuneStates` siempre llegaba con `Length: 0`, aunque `Append` reportara
éxito.

### Fix aplicado

Se introdujo una **Local Variable** dentro de `EquipmentChanged`:

| Variable | Tipo |
|---|---|
| `LocalHandlesToSave` | Array de DataTableRowHandle |

Nuevo flujo en Bloque B:

```
Break STR_RuneStateHandles → Handles
  → SET LocalHandlesToSave ← Handles
  → APPEND (Target Array: GET LocalHandlesToSave, Source: Applied Effect Handles)
  → Make STR_RuneStateHandles (Handles: GET LocalHandlesToSave)
  → ADD (Map: ActiveRuneStates, Key: Slot, Value: resultado del Make)
```

Al usar una variable real (no el output directo de la función pura) tanto
para `APPEND` como para `Make STR_RuneStateHandles`, la modificación de
`Append` persiste y `Make` lee el array ya actualizado.

### Diagnóstico completo — historial de la sesión (para referencia futura, patrón reutilizable)

Se instrumentó la cadena completa con Print String, confirmados vía
**Output Log** (no HUD — el HUD reordena mensajes del mismo frame y no es
confiable para diagnósticos de orden de ejecución). Orden de descarte de
hipótesis, en la secuencia real en que se confirmaron:

1. **`ApplyState` funciona correctamente** — `AppliedEffectHandles` con
   longitud 1 confirmada directo en su output. Descartado como causa.
2. **Ciclo `ADD REAL: 1` seguido de `FIND REMOVE: 0`** confirmado
   consistente mientras la runa está equipada — apunta a un problema
   dentro del propio ciclo de guardado/lectura del Map.
3. **`Remove Status Effect By Handle` nunca se ejecutaba** — consecuencia
   directa del punto 2 (For Each Loop sobre array vacío).
4. **Hipótesis "el Map se autoborra en cada ciclo automático" descartada**
   — `REMOVE (Map)` (rama False de `Item Is Valid`) solo se ejecutaba al
   desequipar de verdad, no en cada re-disparo automático.
5. **Prueba de aislamiento con trigger directo (`BP_RemoveEffectTrigger`)**
   — se confirmó que `Remove Status Effect By Handle` y la función interna
   `Remove Status Effect from owner by handle` (en `BP_AbilitySystemComponent`)
   **funcionan correctamente** cuando se les da el Effect Handle directo,
   sin pasar por `ActiveRuneStates`. Esto acotó el bug al 100% al Map, no al
   sistema de remoción del Ability System.
   > Nota confirmada en esta prueba: el nivel de abstracción State vs Effect
   > no es relevante — `DT_States.Effects` ya contiene Effect Handles, así
   > que un trigger de remoción directa con `DT_StatusEffects → Effect_Poison`
   > es compatible sin importar si el efecto se aplicó originalmente vía
   > State (`ApplyState`) o vía Effect directo (`BP_PoisonTrigger`).
6. **`Contains` (Map) confirmó que la Key `7` sí persiste en
   `ActiveRuneStates`** (`Existe: true` de forma consistente) — esto
   descartó la Hipótesis A (variables Map desincronizadas entre Bloque A y
   B) y dejó a la Hipótesis B (struct guardado con array interno vacío)
   como única candidata.
7. **Print combinado `Existe + Handles Length`** confirmó
   `Existe: true - Handles Length: 0` — Hipótesis B confirmada con certeza.
8. **Inspección visual de las conexiones exactas** en Bloque B reveló que
   tanto `APPEND` como `Make STR_RuneStateHandles` leían del mismo pin de
   una función pura (`Break STR_RuneStateHandles`) — causa raíz identificada
   y corregida con la Local Variable `LocalHandlesToSave`.

### Print String de diagnóstico — estado tras el cierre del bug

El usuario decidió **mantener los Print String activos temporalmente** para
que el cliente pueda ver el trabajo de debugging realizado. Pendiente de
limpieza en sesión futura. Lista completa abajo, en la sección de Deuda técnica.

### Lección de arquitectura para el resto del proyecto

Este patrón — conectar múltiples nodos (especialmente uno que modifica por
referencia, como `Append`, y otro que lee después) directamente al mismo pin
de salida de una **función pura** que devuelve un array — es una fuente de
bugs silenciosos: no genera errores de compilación ni de consola, y el
síntoma (datos "perdidos" entre dos puntos que deberían ver lo mismo) puede
confundirse fácilmente con problemas de sincronización de variables. Vale la
pena revisar si este mismo patrón existe en otras partes del proyecto que
usen `Break` sobre structs con arrays internos seguidos de `Append`/`Add`/
modificación por referencia (por ejemplo, revisar `UI_CraftingQueue` y otros
sistemas de queue que manipulan arrays de widgets).

---

## Cobertura de Estados de gameplay

### ✅ Cubiertos completamente
Envenenar, Sangrado, Quemar, Congelar, Parálisis, Def aumentada, Atk aumentado, Def elemental

### 🟡 Cubiertos parcialmente
Petrificar, Fractura, Contusión, Herida grave, Desgarre, Invisibilidad

### 🔴 Requieren sistemas nuevos
Terror, Confundir, Cegar

---

## Plan de fases

### Fase 1 — DT_States + BP_StateApplier base ✅
### Fase 2 — IsHitEffect como filtro de diseño ✅
### Fase 3 — Inspección STR_ItemData + mecanismo de remoción ✅
### Fase 4 — Estados en Rune Words ✅ (con Bug 2 pendiente, prioridad menor)
- Implementación completa ✅
- Bug 1 (ItemStates vacío) — ✅ RESUELTO
- Bug 2 (segundo slot no aplica atributos) — ⏳ pendiente, próxima sesión
- Bug 3 (remoción al desequipar no funciona) — ✅ RESUELTO. Causa raíz:
  pin `Handles` de `APPEND` y `Make STR_RuneStateHandles` leyendo
  directamente de una función pura (`Break STR_RuneStateHandles`) en vez de
  una variable persistente. Fix: Local Variable `LocalHandlesToSave`.

---

## Pendientes fuera de alcance actual

| Caso de uso | Estado |
|---|---|
| BaseState en BP_Character_Base | Evaluar migración para todos los personajes | ⏳ |
| Estados en consumibles | Pospuesto — uso ambiguo | ⏳ |
| Causa raíz del re-disparo de EquipmentChanged en cada tick de daño | ⏳ Mitigado con guard, causa raíz sin inspeccionar. Sigue ocurriendo (2 veces por frame + re-disparo cada ~1s), pero ya no es bloqueante ahora que el Bug 3 está resuelto. |
| Revisar patrón "pure function → múltiples consumidores" en otros sistemas | Ver Lección de arquitectura arriba — candidato: `UI_CraftingQueue` y otros sistemas de queue/array | ⏳ Nuevo, sugerido esta sesión |

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Error de stack duplicado en Load Status Effect | Mitigado con guard CheckStatusEffectByHandle en ApplyState | ✅ Mitigado |
| BaseState solo en BP_Character_Undead2 | Evaluar migración a BP_Character_Base | ⏳ |
| Pin ItemStates desconectado en Make Item (BP_ItemsLibrary) | Causaba Bug 1 completo | ✅ Resuelto |
| Bug 3 — remoción de Estado al desequipar no funciona | Causa raíz: pure function reevaluada en vez de variable persistente. Fix: Local Variable `LocalHandlesToSave`. Ver sección Bug 3 completa arriba. | ✅ Resuelto |
| Bug 2 — Segundo slot de runa no aplica atributos | Sin retocar. Prioridad para próxima sesión. | ⏳ Prioridad menor |
| Print String "No es runa" debug temporal en UI_ItemSlot.OnDrop | Pendiente eliminar cuando se resuelva Bug 2 | ⏳ |
| Causa raíz de re-disparo de EquipmentChanged por tick de daño | No inspeccionada — solo mitigada con guard. Confirmado que sigue ocurriendo, 2 veces por frame (Slot 0 + Slot 7) más el re-disparo cada ~1s | ⏳ Media prioridad |
| Print String de diagnóstico del Bug 3 (PUNTO 1, 1b, 1c, ADD REAL, FIND REMOVE, MAP REMOVE EJECUTADO, REMOVE - Success, EQCHANGED START, CONTAINS KEY) | Mantenidos intencionalmente por decisión del usuario, para mostrar el trabajo de debugging al cliente. Pendientes de limpieza en sesión futura. | ⏳ Limpieza pospuesta — decisión del usuario |
| `BP_RemoveEffectTrigger` (trigger de prueba creado esta sesión) | Usado para aislar y confirmar que el sistema de remoción del Ability System funciona correctamente. Puede quedar en el nivel como herramienta de testing o eliminarse — decisión del usuario. | ⏳ Sin decisión — bajo impacto |
| Revisar patrón pure-function en otros sistemas de arrays/queues | Ver Lección de arquitectura arriba | ⏳ Nuevo — sugerido, sin prioridad asignada |

---

## Nota para Logica 5 -- Estados y persistencia de runas

> **NOTA DE ARQUITECTURA PARA LOGICA 5**
>
> El sistema de Estados usa `STR_SaveData_StatusEffect` para serializar el estado
> activo de un efecto: `Effect Handle + Stack + Time Remaining`.
> Este patrón de "guardar estado de un sistema activo en un struct serializable"
> es directamente relevante para la pregunta de Logica 5:
> **"¿Puede cada herramienta o cosmetico guardar su propia configuracion de runas?"**
>
> Si el sistema de runas sigue un patrón similar — un struct con los handles de
> las runas asignadas + su estado — podría persistirse por item de la misma forma
> que los efectos activos se guardan en Save Data.
>
> Esto no es una decisión tomada. Es arquitectura observable que debe considerarse
> al diseñar la persistencia de runas en Logica 5.
>
> **Actualización de esta sesión:** el Bug 3 resuelto refuerza esta nota — el
> patrón de "Local Variable intermedia antes de Append/Make Struct" que
> resolvió el Bug 3 es directamente aplicable si Logica 5 termina usando un
> patrón similar de struct + array para persistir configuraciones de runas
> por ítem. Tenerlo en cuenta al diseñar esa persistencia para evitar el
> mismo tipo de bug desde el inicio.

---

*Archivo actualizado — sesión Light Paradox (Bug 3 — RESUELTO, causa raíz confirmada y fix aplicado)*
*Cambios: Bug 3 marcado como resuelto con causa raíz completa documentada (pure function
Break STR_RuneStateHandles reevaluada en vez de variable persistente), fix con Local Variable
LocalHandlesToSave documentado paso a paso, historial completo de diagnóstico consolidado como
referencia reutilizable, lección de arquitectura agregada para revisar el mismo patrón en otros
sistemas, Bug 2 promovido a prioridad de la próxima sesión*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
