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

## ⚠️ IMPORTANTE — Dos rutas independientes de modificación de stats (aclarado en sesión Bug 4)

Existen **dos sistemas separados** por los que un ítem puede modificar un stat del personaje. No confundirlos al diagnosticar bugs de "el atributo no sube":

### Ruta A — `EquipmentAttributes` (campo directo del ítem en `STR_ItemData` / `DT_Items`)
- Vive directamente en la fila del ítem en `DT_Items`, campo `EquipmentAttributes` (Array de `GameplayTag` + `Value`).
- **No depende de `ItemStates` ni de ningún Effect.** Es un array completamente independiente dentro del mismo struct.
- Se aplica (en teoría) mediante un nodo/función **`Update Equipment Attributes`**, mencionado en el flujo de `EquipmentChanged` de `BP_Character_Player`, pero **su interior nunca ha sido inspeccionado ni documentado en este proyecto**.
- **Estado: 🔴 Bug 4 activo** — ver sección Bug 4 más abajo. El bonus de esta ruta actualmente solo se aplica si el ítem *también* tiene `ItemStates` asignado, lo cual no debería ser necesario según el diseño de datos (son campos hermanos, no dependientes).

### Ruta B — `CharacterAttributes` (dentro de un `STR_StatusEffectInstance`, vía `ItemStates`)
- Vive en `DT_StatusEffects`, campo `CharacterAttributes`, dentro de la fila del Effect referenciado por un `ItemState` del ítem.
- Solo se aplica **mientras el actor del efecto (`BP_StatusEffect_*`) está vivo** — requiere que el ítem tenga `ItemStates` asignado, pase por `Apply State` → `Load Status Effect`, y se instancie el Effect correspondiente (normalmente `BP_StatusEffect_ChangeState`).
- Esta ruta **sí depende correctamente de `ItemStates`** — es su diseño esperado, no un bug.
- **Estado: ✅ Funcional**, confirmado en sesiones anteriores (Corn, Beet, Pumpkin, Cabbage, Healing, etc. — ver tabla de filas en `15_SYSTEM_StatusEffects.md`).

**Regla de diagnóstico:** antes de investigar cualquier reporte de "el stat no sube", confirmar primero en qué campo vive el bonus — `EquipmentAttributes` (Ruta A, ítem) o `CharacterAttributes` (Ruta B, Effect). Son independientes y sus bugs no se resuelven de la misma forma.

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
relacionado al Bug 3 (sesión anterior).

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
[después de Update Equipment Attributes]   ← ⚠️ ver Bug 4: este nodo es sospechoso de estar mal conectado

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

## Bug 2 — Segundo slot de runa no aplica atributos de equipamiento ✅ RESUELTO

**Estado final: ✅ CONFIRMADO RESUELTO** en la sesión de fix de `CheckContainerSlotForItem`.

### Síntoma
Al desbloquear el segundo slot de runa (`HeadRuneSlot_1`, índice 14) y
colocar una runa, los atributos/efectos no se aplicaban. Aparecía el debug
log "No es runa" (Print String temporal en `UI_ItemSlot.OnDrop`) de forma
intermitente/confusa durante el diagnóstico.

### Causa raíz confirmada

En `BP_EquipmentComponent` → override `CheckContainerSlotForItem`, la
comparación `EquipmentType == Slot` fallaba siempre para los slots
duplicados de Head Rune Word (índices 14 al 22), porque `Item Is Equipment`
siempre devuelve `EquipmentType = HeadRuneWord (7)` para cualquier Rune Word
de tipo Head — todos comparten el mismo Gameplay Tag y la misma entrada en
el Map `Local Equipment Types` (ver `06_BLUEPRINT_BP_EquipmentComponent.md`).

Esto hacía que `CheckContainerSlotForItem` devolviera `False` para los
slots 14-22, impidiendo que el ítem quedara correctamente registrado en el
`EquipmentContainer` en esos slots, lo cual cortaba en cascada la ejecución
correcta de `EquipmentChanged` y por lo tanto de `ApplyState`/atributos para
esos slots.

### Fix aplicado

Se agregó una condición `OR` en `CheckContainerSlotForItem` que acepta el
caso especial `EquipmentType == HeadRuneWord AND Slot dentro de [14, 22]`,
preservando la validación original para el resto de slots.

**Documentación completa del fix, con fórmula lógica y patrón para
replicar a otras familias de runa (Body/Pants/Hands/Feet/Backpack/Tool):**
ver `06_BLUEPRINT_BP_EquipmentComponent.md`, sección "CheckContainerSlotForItem — Override".

### Nota importante — se descubrió un bug derivado (ver Bug 4)

Al confirmar el fix del Bug 2 en juego, se detectó un **problema distinto y
nuevo**: el atributo de `EquipmentAttributes` (Ruta A, ver sección "Dos
rutas independientes" arriba) solo se aplica si el ítem *también* tiene
`ItemStates` asignado — lo cual no debería ser un requisito, porque
`EquipmentAttributes` e `ItemStates` son campos independientes del ítem.
Este es el **Bug 4**, documentado abajo, pendiente para la próxima sesión.

---

## Bug 3 — El efecto no se remueve al desequipar la runa ✅ RESUELTO

**Estado final: ✅ CONFIRMADO RESUELTO**, con fix aplicado y verificado por
el usuario en juego (el efecto de veneno se detiene correctamente al
desequipar la runa, tanto en configuración permanente como no permanente).

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

## 🔴 Bug 4 (NUEVO — abierto esta sesión) — EquipmentAttributes no se aplica sin ItemStates asignado

**Estado:** 🔴 Activo — diagnóstico iniciado, pendiente de confirmación en
próxima sesión antes de aplicar cualquier fix.

### Síntoma confirmado por el usuario

Con una Rune Word que tiene:
- `EquipmentAttributes`: 1 elemento (`EasyRPG.Attributes.Base.EnergyRegeneration`, Value `30.0`)
- `ItemStates`: 1 elemento (`DT_States` → `State_Poison`)

...al equipar la runa, `EnergyRegeneration` **sí** sube +30 correctamente,
por cada runa colocada.

Si se elimina el elemento de `ItemStates` (dejándolo en 0 elementos) mientras
`EquipmentAttributes` permanece igual (1 elemento, sin cambios), el bonus de
`EnergyRegeneration` **deja de aplicarse**.

### Por qué esto no debería pasar según el diseño de datos

`EquipmentAttributes` e `ItemStates` son dos campos **hermanos e
independientes** dentro de `STR_ItemData` — confirmado visualmente en el
Row Editor de `DT_Items` (ambos aparecen como campos separados al mismo
nivel, sin relación jerárquica entre sí). Además, se confirmó por captura
que `CharacterAttributes` dentro del `Effect_Poison` referenciado por
`State_Poison` está **vacío** (0 Array elements, 0 Map elements) — es decir,
el bonus de `EnergyRegeneration` no puede estar viniendo de la Ruta B
(`CharacterAttributes` vía Effect). Tiene que estar viniendo de la Ruta A
(`EquipmentAttributes` directo del ítem), que no debería tener ninguna
dependencia de `ItemStates`.

### Hipótesis principal (no confirmada — requiere inspección de grafo)

El nodo/función responsable de leer `EquipmentAttributes` y aplicarlo
(mencionado en el flujo de `EquipmentChanged` como `Update Equipment
Attributes`, **nunca inspeccionado en detalle en este proyecto**) podría
estar conectado, por error de wiring, al pin **`Loop Body`** del
`For Each Loop` que itera sobre `Item States` en el Bloque B (ver flujo de
`EquipmentChanged` arriba), en vez de tener su propia rama de ejecución
independiente y anterior a ese loop.

Esto explicaría el síntoma exactamente:
- Con `ItemStates` con ≥1 elemento → el `For Each Loop` dispara `Loop Body`
  al menos una vez → si `Update Equipment Attributes` cuelga de ahí, se
  ejecuta.
- Con `ItemStates` vacío (0 elementos) → el `For Each Loop` **nunca**
  dispara `Loop Body`, solo `Completed` → si `Update Equipment Attributes`
  está enganchado a `Loop Body`, nunca se ejecuta.

Es el mismo tipo de patrón de bug silencioso de wiring que produjo el Bug 3
(ver Lección de Arquitectura arriba), pero en el exec chain en vez de en un
pure function.

> **Nivel de confianza:** hipótesis fuerte basada en el patrón del síntoma,
> **no confirmada por inspección directa del grafo**. No aplicar ningún fix
> sin completar primero el diagnóstico de la checklist de abajo.

### Checklist de diagnóstico para la próxima sesión

1. Abrir `BP_Character_Player` → función `EquipmentChanged`.
2. Localizar el nodo `Update Equipment Attributes` (nombre puede variar
   ligeramente — buscar el nodo que efectivamente lee `EquipmentAttributes`
   del ítem).
3. Click en el pin de entrada de ejecución (flecha blanca izquierda) de ese
   nodo. Seguir el cable **hacia atrás** visualmente. Confirmar una de dos:
   - **(A)** Viene de un pin `Loop Body` de un `For Each Loop` sobre
     `Item States` → confirma la hipótesis.
   - **(B)** Viene de un `Sequence`, `Branch`, o está en la cadena principal
     fuera de cualquier loop → descarta la hipótesis, requiere inspección
     más profunda del interior de la función.
4. Si hay duda visual: agregar `Print String` (Development Only, target:
   **Output Log**, no pantalla) justo antes del nodo `Update Equipment
   Attributes`, texto: `"Update Equipment Attributes EJECUTADO"`.
5. Probar en PIE dos veces:
   - Equipar runa **con** `ItemStates` asignado → revisar si el log aparece.
   - Equipar runa **sin** `ItemStates` (0 elementos) → revisar si el log
     **no** aparece.
6. Según resultado:

| Resultado del log | Diagnóstico | Siguiente paso |
|---|---|---|
| No aparece cuando `ItemStates` está vacío | Hipótesis confirmada — wiring colgado del `Loop Body` | Mover el nodo fuera del loop, a su propia rama de ejecución previa (ej. vía `Sequence`), independiente de si `Item States` tiene elementos o no |
| Sí aparece siempre, incluso sin `ItemStates` | Hipótesis descartada — el problema está *dentro* de `Update Equipment Attributes` (posible `Branch` interno mal condicionado leyendo `ItemStates.Length`) | Requiere ver el interior completo de la función — no intentar fix sin esa inspección |

7. Una vez identificada la causa exacta, actualizar esta sección con el fix
   aplicado (siguiendo el mismo formato usado para Bug 2 y Bug 3), y
   documentar por primera vez el interior de `Update Equipment Attributes`
   en un archivo nuevo o en `07_BLUEPRINT_BP_Character_Player.md`, ya que
   actualmente no existe documentación de esa función en ningún .md del
   proyecto.

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
### Fase 4 — Estados en Rune Words ✅ (con Bug 2 resuelto, Bug 4 nuevo pendiente)
- Implementación completa ✅
- Bug 1 (ItemStates vacío) — ✅ RESUELTO
- Bug 2 (segundo slot no aplica atributos) — ✅ RESUELTO. Causa raíz:
  `CheckContainerSlotForItem` fallaba la comparación `EquipmentType == Slot`
  para slots duplicados de Head Rune Word. Fix: condición `OR` adicional.
  Ver `06_BLUEPRINT_BP_EquipmentComponent.md`.
- Bug 3 (remoción al desequipar no funciona) — ✅ RESUELTO. Causa raíz:
  pin `Handles` de `APPEND` y `Make STR_RuneStateHandles` leyendo
  directamente de una función pura (`Break STR_RuneStateHandles`) en vez de
  una variable persistente. Fix: Local Variable `LocalHandlesToSave`.
- Bug 4 (EquipmentAttributes depende erróneamente de ItemStates) —
  🔴 **NUEVO, activo.** Ver sección Bug 4 completa arriba. Prioridad para
  la próxima sesión.

---

## Pendientes fuera de alcance actual

| Caso de uso | Estado |
|---|---|
| BaseState en BP_Character_Base | Evaluar migración para todos los personajes | ⏳ |
| Estados en consumibles | Pospuesto — uso ambiguo | ⏳ |
| Causa raíz del re-disparo de EquipmentChanged en cada tick de daño | ⏳ Mitigado con guard, causa raíz sin inspeccionar. Sigue ocurriendo (2 veces por frame + re-disparo cada ~1s), pero ya no es bloqueante ahora que el Bug 3 está resuelto. |
| Revisar patrón "pure function → múltiples consumidores" en otros sistemas | Ver Lección de arquitectura arriba — candidato: `UI_CraftingQueue` y otros sistemas de queue/array | ⏳ |
| **Nuevo:** Documentar el interior de `Update Equipment Attributes` | Nunca inspeccionado en detalle — necesario para cerrar Bug 4 | 🔴 Prioridad próxima sesión |

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Error de stack duplicado en Load Status Effect | Mitigado con guard CheckStatusEffectByHandle en ApplyState | ✅ Mitigado |
| BaseState solo en BP_Character_Undead2 | Evaluar migración a BP_Character_Base | ⏳ |
| Pin ItemStates desconectado en Make Item (BP_ItemsLibrary) | Causaba Bug 1 completo | ✅ Resuelto |
| Bug 2 — CheckContainerSlotForItem fallaba para slots de runa duplicados | Causa raíz: comparación EquipmentType==Slot rota para índices 14-22. Fix: OR adicional. Ver `06_BLUEPRINT_BP_EquipmentComponent.md` | ✅ Resuelto |
| Bug 3 — remoción de Estado al desequipar no funciona | Causa raíz: pure function reevaluada en vez de variable persistente. Fix: Local Variable `LocalHandlesToSave`. | ✅ Resuelto |
| **Bug 4 — EquipmentAttributes no se aplica sin ItemStates** | Sospecha de wiring de `Update Equipment Attributes` colgado del `Loop Body` del For Each Loop de Item States. Ver checklist de diagnóstico completa arriba. | 🔴 **Activo — prioridad próxima sesión** |
| Print String "No es runa" debug temporal en UI_ItemSlot.OnDrop | Pendiente eliminar — su comportamiento confuso durante el diagnóstico del Bug 2 quedó explicado por la causa raíz real (CheckContainerSlotForItem), no por la lógica de detección de runa en sí | ⏳ |
| Causa raíz de re-disparo de EquipmentChanged por tick de daño | No inspeccionada — solo mitigada con guard. Confirmado que sigue ocurriendo, 2 veces por frame (Slot 0 + Slot 7) más el re-disparo cada ~1s | ⏳ Media prioridad |
| Print String de diagnóstico del Bug 3 (PUNTO 1, 1b, 1c, ADD REAL, FIND REMOVE, MAP REMOVE EJECUTADO, REMOVE - Success, EQCHANGED START, CONTAINS KEY) | Mantenidos intencionalmente por decisión del usuario, para mostrar el trabajo de debugging al cliente. Pendientes de limpieza en sesión futura. | ⏳ Limpieza pospuesta — decisión del usuario |
| `BP_RemoveEffectTrigger` (trigger de prueba) | Puede quedar en el nivel como herramienta de testing o eliminarse — decisión del usuario. | ⏳ Sin decisión — bajo impacto |
| Revisar patrón pure-function en otros sistemas de arrays/queues | Ver Lección de arquitectura arriba | ⏳ Sin prioridad asignada |
| Interior de `Update Equipment Attributes` sin documentar | Necesario para cerrar Bug 4 correctamente | 🔴 Prioridad próxima sesión |

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
> **Actualización sesión Bug 3:** el Bug 3 resuelto refuerza esta nota — el
> patrón de "Local Variable intermedia antes de Append/Make Struct" que
> resolvió el Bug 3 es directamente aplicable si Logica 5 termina usando un
> patrón similar de struct + array para persistir configuraciones de runas
> por ítem.
>
> **Actualización sesión Bug 4:** si `EquipmentAttributes` termina
> necesitando persistencia por-ítem también (no solo `ItemStates`), la
> solución de Bug 4 debe diseñarse teniendo en cuenta que ambos campos
> (`EquipmentAttributes` e `ItemStates`) eventualmente conviven en el mismo
> `STR_ItemData` y podrían compartir el mismo mecanismo de persistencia
> futuro. No tomar decisiones de Fase 2 de Logica 5 sin considerar esto.

---

*Archivo actualizado — sesión Light Paradox (Bug 2 RESUELTO, Bug 4 nuevo abierto)*
*Cambios: Bug 2 marcado como resuelto con causa raíz y referencia cruzada a 06_BLUEPRINT_BP_EquipmentComponent.md,
sección nueva "Dos rutas independientes de modificación de stats" (EquipmentAttributes vs CharacterAttributes)
agregada para prevenir futuras confusiones diagnósticas, Bug 4 documentado completo con hipótesis, nivel de
confianza explícito, y checklist de diagnóstico paso a paso lista para la próxima sesión*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
