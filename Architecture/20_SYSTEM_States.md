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
rutas independientes" arriba) parecía solo aplicarse si el ítem *también*
tenía `ItemStates` asignado. Diagnóstico posterior (ver Bug 4 completo abajo)
**descartó esta correlación** — el bug es independiente de `ItemStates` y
vive en la cadena base de ESRPGv5.

### ⚠️ Advertencia — verificar que este fix sigue guardado antes de la próxima sesión

En una sesión posterior se detectó que el bloque `OR` (B1/B2/B3) de esta
sección **no estaba presente** en el grafo real de `CheckContainerSlotForItem`
— la función había vuelto a mostrar solo la comparación `==` original sin el
`OR`. Causa más probable: el fix se aplicó pero no se guardó (`Compile` +
`Save`) antes de cerrar el editor/proyecto. **Antes de continuar con
cualquier otro bug, confirmar en el editor que el bloque `OR` completo
(Equal contra `Head Rune Word` + `Slot >= 14` + `Slot <= 22` + `AND` + `OR`
final) sigue presente en `CheckContainerSlotForItem`. Si no está, rehacerlo
siguiendo la fórmula lógica de esta sección y confirmar Compile + Save
explícitamente (ícono de disco debe dejar de estar resaltado) antes de
cerrar Unreal.**

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

## Bug 4 — EquipmentAttributes no se acumulaba entre múltiples slots del mismo tipo ✅ RESUELTO

**Estado final: ✅ CONFIRMADO RESUELTO por el usuario en juego.**

### Síntoma original reportado (correlación con ItemStates — DESCARTADA)

Con una Rune Word que tiene:
- `EquipmentAttributes`: 1 elemento (`EasyRPG.Attributes.Base.EnergyRegeneration`, Value `30.0`)
- `ItemStates`: 1 elemento (`DT_States` → `State_Poison`)

...al equipar la runa en el **primer slot**, `EnergyRegeneration` sí sube
+30 correctamente. Al equipar una **segunda** runa igual en el segundo slot,
el atributo no vuelve a subir. La hipótesis inicial fue que esto dependía de
`ItemStates` — **descartada** mediante prueba de aislamiento (ver historial
de diagnóstico abajo).

### Historial de diagnóstico (hipótesis descartadas en el camino)

Antes de llegar a la causa raíz real, se descartaron en orden:

1. **Wiring de `EquipmentChanged` colgado de un `Loop Body`** — descartado
   por inspección directa: la cadena `Switch on E_EquipmentSlot → [SET w/
   Notify mesh] → Update Footstep Settings → Update Equipment Attributes`
   es lineal e incondicional, sin ningún loop de `Item States` antes.
2. **Relación con `ItemStates`** — descartado mediante 3 pruebas de
   aislamiento del usuario: re-equipar la misma runa sin `ItemStates` no
   corrige nada; esperar varios segundos no corrige nada (descarta
   dependencia del re-disparo de `EquipmentChanged` por tick de Status
   Effect); desconectar por completo la lógica de Estados de Light Paradox
   no cambia el patrón — confirma que el bug vive en la cadena base de
   ESRPGv5, sin relación con `ItemStates`.
3. **Map por GameplayTag sobreescribiendo en `Get Items Equipment
   Attributes`** — descartado por inspección directa: la función no
   contiene ningún Map, solo un `For Each Loop` + `Add Attributes`.
4. **`Add Attribute` (función singular en `BP_GameLibrary`) reemplazando en
   vez de sumar** — descartado por inspección directa: la rama `True` del
   `Branch` (cuando el `GameplayTag` ya existe en el array) sí contiene un
   nodo `+` (Float + Float) sumando `Break.Value` (valor viejo) con
   `Entry.Value` (valor nuevo) antes de escribir con `Set Array Elem`. La
   suma matemática siempre estuvo correcta en esta función.
5. **`Get Equipment Items` filtrando/agrupando por `EquipmentType`** —
   descartado por inspección directa: la función es un getter trivial
   (`Return Node` directo sobre la variable `Items`), sin ningún filtro.

Un `Print String` con `Length (Array)` sobre el array `Equipment Attributes`
justo antes de `Update Equipment Attributes` confirmó el síntoma exacto:
`Length = 1` al equipar la runa 1 (slot 7), y **el log ni siquiera se
imprimía** al equipar la runa 2 (slot 14) — señal de que la ejecución no
estaba llegando hasta ese punto para los slots de runa duplicados.

### Causa raíz confirmada — dos problemas combinados, ninguno relacionado a suma de valores

**Causa 1 — `EquipmentSlots` con tamaño insuficiente.** El array
`EquipmentSlots` (Instance Editable, en el panel Details del componente
`EquipmentComponent`) solo tenía **13 elementos (índices 0-12)**, mientras
que el enum `E_EquipmentSlot` llega hasta el índice 22
(`HeadRuneWord_10`). Los slots de runa duplicados (14-22) no tenían
entrada real en el array de datos del componente — aunque
`CheckContainerSlotForItem` (tras el fix del Bug 2) ya dejara pasar la
validación lógica de esos slots, no había dónde "aterrizar" el dato
realmente.

**Causa 2 — pines de ejecución desconectados en `Switch on
E_EquipmentSlot` dentro de `EquipmentChanged`.** Al crear los valores de
enum `HeadRuneWord_2` a `HeadRuneWord_10` en sesiones anteriores (para el
sistema de runas múltiples), sus pines de salida de ejecución en el nodo
`Switch on E_EquipmentSlot` de la función `EquipmentChanged`
(`BP_Character_Player`) **nunca se conectaron al flujo principal**. Esto
significa que, incluso si el ítem hubiera estado correctamente registrado
en el contenedor, la cadena de ejecución que sigue después del `Switch`
—incluyendo, más adelante, `Update Equipment Attributes`— **nunca corría**
para esos slots. Esta es la causa directa de que el `Length` no subiera y
el segundo `Print String` ni siquiera imprimiera: la ejecución se detenía
en un pin sin conexión dentro del `Switch`, sin ningún error de
compilación ni de consola.

> **Nota de arquitectura:** ninguna de las dos causas tiene relación con
> suma/acumulación de valores de atributos — toda la cadena matemática
> (`Add Attribute`, `Get Items Equipment Attributes`, etc.) siempre estuvo
> correcta. El bug era que la segunda runa **nunca llegaba a participar**
> en esa cadena en absoluto, por dos razones de configuración/wiring
> independientes entre sí.

### Fix aplicado

1. En `BP_EquipmentComponent` → panel Details → `Equipment Slots`: se
   agregaron los elementos faltantes desde el índice 14 hasta el 22
   (`HeadRuneWord_2` a `HeadRuneWord_10`), completando el array a 23
   elementos totales (0-22), sincronizado con `E_EquipmentSlot`.
2. En `BP_Character_Player` → función `EquipmentChanged` → nodo `Switch on
   E_EquipmentSlot`: se conectaron los pines de ejecución de salida de
   `Head Rune Word 2` hasta `Head Rune Word 10` (que estaban sueltos, sin
   conexión) hacia el mismo `Set w/ Notify (Equiped Head Mesh)` que ya
   usaba el pin `Head Rune Word` (slot base, índice 7) — mismo patrón que
   los demás valores del Switch, que convergen todos en la actualización
   de mesh correspondiente antes de continuar hacia `Update Footstep
   Settings` → `Update Equipment Attributes`.
3. Confirmado en juego: equipar dos runas del mismo `GameplayTag` en slots
   distintos ahora suma correctamente ambos valores de `EquipmentAttributes`.

### ⚠️ Este mismo par de pasos falta en la documentación de creación de slots — ver `07_BLUEPRINT_BP_Character_Player.md`

Ninguno de los dos pasos del fix estaba en la checklist de "seis puntos
sincronizados" que documenta `07_BLUEPRINT_BP_Character_Player.md` para
crear un slot de equipamiento nuevo. Se actualizó ese archivo agregando
estos dos puntos como pasos 7 y 8 de la checklist, para que no se repita
este bug al replicar el sistema de runas múltiples a Body/Pants/Hands/
Feet/Backpack/Tool (ver `16_SYSTEM_RuneBinding_WeaponCosmetic.md`, Fase 1).

### Hallazgo relacionado (bug distinto, registrado pero fuera de alcance por ahora)

Durante las pruebas de diagnóstico se detectó un bug adicional en la
cadena de visibilidad de slots de runa: al desequipar la runa del slot 1
(con el slot 2 también ocupado), la cadena de reacción visual deja un par
de slots vacíos de forma inconsistente; al volver a colocar una runa en el
slot vacío resultante, esa runa sí suma correctamente su valor sobre el
atributo ya existente. **El usuario ya tiene una solución de diseño
planeada para esto — no se investiga en esta fase.** Registrado como Bug 5
en la tabla de Deuda Técnica al final de este archivo.

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
- Bug 4 (EquipmentAttributes no se acumulaba entre múltiples slots) —
  ✅ **RESUELTO esta sesión.** Causa raíz doble: (1) array `EquipmentSlots`
  en `BP_EquipmentComponent` con solo 13 de 23 elementos necesarios; (2)
  pines de ejecución de `HeadRuneWord_2` a `HeadRuneWord_10` sin conectar
  en `Switch on E_EquipmentSlot` dentro de `EquipmentChanged`. Ver sección
  Bug 4 completa arriba. Checklist de creación de slots actualizada en
  `07_BLUEPRINT_BP_Character_Player.md`.
- Bug 5 (cascada de slots vacíos al desequipar runa de un slot inferior) —
  🟡 **Detectado como efecto colateral durante pruebas de Bug 4.** No
  bloqueante para el gameplay actual. El usuario ya tiene solución de
  diseño planeada — no se investiga en esta fase. Ver nota en sección
  Bug 4 ("Hallazgo relacionado").

---

## Pendientes fuera de alcance actual

| Caso de uso | Estado |
|---|---|
| BaseState en BP_Character_Base | Evaluar migración para todos los personajes | ⏳ |
| Estados en consumibles | Pospuesto — uso ambiguo | ⏳ |
| Causa raíz del re-disparo de EquipmentChanged en cada tick de daño | ⏳ Mitigado con guard, causa raíz sin inspeccionar. Sigue ocurriendo (2 veces por frame + re-disparo cada ~1s), pero ya no es bloqueante ahora que el Bug 3 está resuelto. Confirmado que este re-disparo NO fue la causa de Bug 4. |
| Revisar patrón "pure function → múltiples consumidores" en otros sistemas | Ver Lección de arquitectura arriba — candidato: `UI_CraftingQueue` y otros sistemas de queue/array | ⏳ |
| Documentar el interior de `Get Equipment Items`, `Exclude Broken Items`, `Get Items Equipment Attributes`, `Add Attribute` y `Update Equipment Attributes` | Ninguna tiene documentación formal en un archivo dedicado todavía, aunque ya fueron inspeccionadas durante el diagnóstico de Bug 4 (confirmadas funcionalmente correctas). Documentar su interior en `06_BLUEPRINT_BP_EquipmentComponent.md` o archivo nuevo cuando haya oportunidad | 🟡 Prioridad media — ya no bloquea nada, es documentación pendiente |
| Bug 5 — cascada de slots vacíos al desequipar runa | Detectado como efecto colateral durante pruebas de Bug 4. Solución de diseño ya planeada por el usuario — no investigar todavía | 🟡 Pendiente, sin prioridad asignada por decisión del usuario |

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Error de stack duplicado en Load Status Effect | Mitigado con guard CheckStatusEffectByHandle en ApplyState | ✅ Mitigado |
| BaseState solo en BP_Character_Undead2 | Evaluar migración a BP_Character_Base | ⏳ |
| Pin ItemStates desconectado en Make Item (BP_ItemsLibrary) | Causaba Bug 1 completo | ✅ Resuelto |
| Bug 2 — CheckContainerSlotForItem fallaba para slots de runa duplicados | Causa raíz: comparación EquipmentType==Slot rota para índices 14-22. Fix: OR adicional. Ver `06_BLUEPRINT_BP_EquipmentComponent.md`. **⚠️ Verificar al abrir la próxima sesión que el fix sigue presente y guardado en el grafo — se detectó que pudo no haberse guardado tras la sesión en que se aplicó.** | ✅ Resuelto (pendiente reverificación) |
| Bug 3 — remoción de Estado al desequipar no funciona | Causa raíz: pure function reevaluada en vez de variable persistente. Fix: Local Variable `LocalHandlesToSave`. | ✅ Resuelto |
| **Bug 4 — EquipmentAttributes no se acumulaba entre múltiples slots del mismo tipo** | Causa raíz doble: `EquipmentSlots` con tamaño insuficiente (13 de 23) + pines de ejecución sin conectar en `Switch on E_EquipmentSlot` para `HeadRuneWord_2`-`HeadRuneWord_10`. Fix aplicado y confirmado en juego. Checklist de creación de slots actualizada. | ✅ **Resuelto** |
| **Bug 5 — cascada de slots vacíos al desequipar runa de slot inferior** | Detectado durante pruebas de Bug 4. Al desequipar runa del slot 1 con slot 2 ocupado, la cadena de visibilidad deja slots vacíos inconsistentes; el slot vacío resultante sí acumula el atributo correctamente al recibir una runa nueva. Posible relación con el mismo problema de fondo del Bug 2 (EquipmentType compartido entre slots duplicados). Usuario ya tiene solución de diseño planeada. | 🟡 Registrado — no investigar todavía, por decisión del usuario |
| Print String "No es runa" debug temporal en UI_ItemSlot.OnDrop | Pendiente eliminar — su comportamiento confuso durante el diagnóstico del Bug 2 quedó explicado por la causa raíz real (CheckContainerSlotForItem), no por la lógica de detección de runa en sí | ⏳ |
| Causa raíz de re-disparo de EquipmentChanged por tick de daño | No inspeccionada — solo mitigada con guard. Confirmado que sigue ocurriendo, 2 veces por frame (Slot 0 + Slot 7) más el re-disparo cada ~1s. Confirmado que NO fue la causa de Bug 4. | ⏳ Media prioridad |
| Print String de diagnóstico del Bug 3 (PUNTO 1, 1b, 1c, ADD REAL, FIND REMOVE, MAP REMOVE EJECUTADO, REMOVE - Success, EQCHANGED START, CONTAINS KEY) | Mantenidos intencionalmente por decisión del usuario, para mostrar el trabajo de debugging al cliente. Pendientes de limpieza en sesión futura. | ⏳ Limpieza pospuesta — decisión del usuario |
| `BP_RemoveEffectTrigger` (trigger de prueba) | Puede quedar en el nivel como herramienta de testing o eliminarse — decisión del usuario. | ⏳ Sin decisión — bajo impacto |
| Revisar patrón pure-function en otros sistemas de arrays/queues | Ver Lección de arquitectura arriba | ⏳ Sin prioridad asignada |
| Interior de `Get Equipment Items` / `Exclude Broken Items` / `Get Items Equipment Attributes` / `Add Attribute` / `Update Equipment Attributes` sin documentar formalmente | Ya inspeccionadas y confirmadas correctas durante el diagnóstico de Bug 4 — falta únicamente redactar su documentación formal en un .md | 🟡 Prioridad media |
| Print String de diagnóstico de Bug 4 (Length antes de Update Equipment Attributes) | Pendiente eliminar tras confirmar estabilidad del fix en sesiones futuras | ⏳ Limpieza pospuesta |

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

*Archivo actualizado — sesión Light Paradox (Bug 4 RESUELTO)*
*Cambios: Bug 4 confirmado y resuelto — causa raíz doble: (1) array EquipmentSlots en BP_EquipmentComponent con solo
13 de 23 elementos necesarios (faltaban índices 14-22, HeadRuneWord_2 a HeadRuneWord_10); (2) pines de ejecución de
esos mismos valores de enum sin conectar en el nodo Switch on E_EquipmentSlot dentro de EquipmentChanged
(BP_Character_Player), lo que impedía que la cadena de ejecución llegara hasta Update Equipment Attributes para esos
slots. Ambas causas confirmadas y corregidas en juego. Historial completo de hipótesis descartadas documentado
(Map por GameplayTag, Add Attribute reemplazando en vez de sumar, Get Equipment Items filtrando) para referencia
futura. Bug 5 (cascada de slots vacíos al desequipar) permanece registrado como pendiente, sin investigar por
decisión del usuario. Ver 07_BLUEPRINT_BP_Character_Player.md para la checklist de creación de slots actualizada
con los dos pasos que causaron este bug.*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
