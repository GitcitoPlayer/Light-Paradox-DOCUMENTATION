# 20 — Sistema: States (Estados)
### Sistema: States — Sistema central de Estados de Light Paradox
### Base Asset: EasySurvivalRPGv5
### Fuente: Diseño del cliente + sesión Light Paradox
### Proyecto: Light Paradox · UE 5.4.4
### Actualizado: sesión Orden de remoción de runas (nota en Bug 5)

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

> **Nota de arquitectura confirmada:** `DT_States.Effects` ya es
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
- Se aplica mediante `Update Equipment Attributes` en el flujo de `EquipmentChanged` de `BP_Character_Player`; su interior fue inspeccionado en el diagnóstico de Bug 4 (confirmado correcto) pero no está documentado formalmente.
- **Estado: ✅ Bug 4 resuelto** — ver sección Bug 4.

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
confirmado funcional y confirmado como no relacionado al Bug 3.

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
nuevo**: el atributo de `EquipmentAttributes` (Ruta A) parecía solo aplicarse
si el ítem *también* tenía `ItemStates` asignado. Diagnóstico posterior
(ver Bug 4) **descartó esta correlación** — el bug es independiente de
`ItemStates` y vive en la cadena base de ESRPGv5.

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

> **Aplicación (sesión Orden de remoción de runas):** `UI_Character.UpdateRuneRemovalLocks`
> usa una Local Variable (`LocalRuneSlots`) para el array de slots por esta misma lección.

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
`ItemStates` — **descartada** mediante prueba de aislamiento.

### Historial de diagnóstico (hipótesis descartadas en el camino)

1. **Wiring de `EquipmentChanged` colgado de un `Loop Body`** — descartado
   por inspección directa: la cadena `Switch on E_EquipmentSlot → [SET w/
   Notify mesh] → Update Footstep Settings → Update Equipment Attributes`
   es lineal e incondicional.
2. **Relación con `ItemStates`** — descartado mediante 3 pruebas de
   aislamiento: re-equipar la misma runa sin `ItemStates` no corrige nada;
   esperar varios segundos no corrige nada; desconectar la lógica de Estados
   no cambia el patrón — el bug vive en la cadena base de ESRPGv5.
3. **Map por GameplayTag sobreescribiendo en `Get Items Equipment
   Attributes`** — descartado: la función no contiene ningún Map, solo un
   `For Each Loop` + `Add Attributes`.
4. **`Add Attribute` (`BP_GameLibrary`) reemplazando en vez de sumar** —
   descartado: la rama `True` contiene un nodo `+` sumando `Break.Value` con
   `Entry.Value` antes de `Set Array Elem`.
5. **`Get Equipment Items` filtrando/agrupando por `EquipmentType`** —
   descartado: es un getter trivial (`Return Node` directo sobre `Items`).

Un `Print String` con `Length (Array)` sobre `Equipment Attributes` antes de
`Update Equipment Attributes` confirmó el síntoma: `Length = 1` al equipar la
runa 1 (slot 7), y **el log ni siquiera se imprimía** al equipar la runa 2
(slot 14) — la ejecución no llegaba a ese punto para los slots duplicados.

### Causa raíz confirmada — dos problemas combinados, ninguno relacionado a suma de valores

**Causa 1 — `EquipmentSlots` con tamaño insuficiente.** El array
`EquipmentSlots` solo tenía **13 elementos (índices 0-12)**, mientras que el
enum `E_EquipmentSlot` llega hasta el índice 22 (`HeadRuneWord_10`). Los
slots 14-22 no tenían entrada real en el array de datos del componente.

**Causa 2 — pines de ejecución desconectados en `Switch on
E_EquipmentSlot` dentro de `EquipmentChanged`.** Los pines de salida de
`HeadRuneWord_2` a `HeadRuneWord_10` **nunca se conectaron al flujo
principal**, por lo que la cadena posterior (incluyendo `Update Equipment
Attributes`) nunca corría para esos slots, sin error de compilación ni de
consola.

> **Nota de arquitectura:** ninguna de las dos causas tiene relación con
> suma/acumulación de valores — toda la cadena matemática siempre estuvo
> correcta. La segunda runa **nunca llegaba a participar** en esa cadena.

### Fix aplicado

1. `BP_EquipmentComponent` → Details → `Equipment Slots`: se agregaron los
   elementos faltantes del índice 14 al 22, completando 23 elementos (0-22).
2. `BP_Character_Player` → `EquipmentChanged` → `Switch on E_EquipmentSlot`:
   se conectaron los pines de `Head Rune Word 2` a `Head Rune Word 10` al
   mismo `Set w/ Notify (Equiped Head Mesh)` que usa `Head Rune Word`.
3. Confirmado en juego: dos runas del mismo `GameplayTag` en slots distintos
   suman correctamente ambos valores de `EquipmentAttributes`.

### ⚠️ Este mismo par de pasos falta en la documentación de creación de slots — ver `07_BLUEPRINT_BP_Character_Player.md`

Ninguno de los dos pasos estaba en la checklist original de "seis puntos". Se
actualizó `07` agregándolos como pasos 7 y 8, para no repetir el bug al
replicar el sistema a Body/Pants/Hands/Feet/Backpack/Tool (ver
`16_SYSTEM_RuneBinding_WeaponCosmetic.md`, Fase 1).

### Hallazgo relacionado (bug distinto — ver Bug 5)

Durante las pruebas se detectó un bug en la cadena de visibilidad de slots de
runa: al desequipar la runa del slot 1 (con el slot 2 también ocupado), la
cadena de reacción visual deja slots vacíos de forma inconsistente.

---

## Bug 5 — Cascada de slots vacíos al desequipar runa de un slot inferior 🟡 MITIGADO POR DISEÑO (pendiente validar)

**Contexto:** detectado como efecto colateral durante pruebas de Bug 4.

**Actualización (sesión Orden de remoción de runas):** la solución de diseño que el usuario
tenía planeada se implementó: las runas ahora **solo pueden quitarse en orden descendente**
mediante `bRemovalLocked` + `RemovalLockOverlay` + `UI_Character.UpdateRuneRemovalLocks`
(ver `04_BLUEPRINT_UI_ItemSlot.md` y `08_BLUEPRINT_UI_Character.md`). Como el escenario que
disparaba el Bug 5 (quitar una runa de un slot inferior con uno superior ocupado) ya no
puede ejecutarse desde la UI, la cascada **no debería poder ocurrir**.

- **Es una inferencia:** el escenario original del Bug 5 no fue re-probado de forma
  específica y reportada. Validar explícitamente antes de cerrar.
- **Límite conocido:** el bloqueo es solo de UI. Si otro sistema llama directamente a
  `TryMoveItemToContainerSlot` (`BP_ContainerComponent`, asset base, Rule 4.1) se salta el
  bloqueo y la cascada podría reaparecer.
- Posible relación con el mismo problema de fondo del Bug 2 (EquipmentType compartido entre
  slots duplicados) sigue sin investigarse.

---

## Bug 6 (UI de runas) — Runa soltada sobre slot de runa ocupado dispara cooldown y candado 🔴 ABIERTO

Registrado en esta sesión. No pertenece al sistema de Estados, pero afecta la cadena de runas
y se anota aquí por contexto: soltar una runa del inventario sobre un slot de runa ya ocupado
(no bloqueado por orden, p. ej. la última runa asignada) dispara el flujo ÉXITO/FALLO,
el cooldown y el candado. Causa (inferida): `IsValidRuneDestination` en `UI_ItemSlot.OnDrop`
no comprueba que el slot destino esté vacío. Detalle, fix propuesto y decisión de diseño
pendiente (rechazar vs reemplazar): ver `04_BLUEPRINT_UI_ItemSlot.md` y
`13_WIDGET_UI_CraftingQueue.md`.

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
### Fase 4 — Estados en Rune Words ✅
- Implementación completa ✅
- Bug 1 (ItemStates vacío) — ✅ RESUELTO
- Bug 2 (segundo slot no aplica atributos) — ✅ RESUELTO. Fix: condición `OR` adicional en
  `CheckContainerSlotForItem`. Ver `06_BLUEPRINT_BP_EquipmentComponent.md`.
- Bug 3 (remoción al desequipar no funciona) — ✅ RESUELTO. Fix: Local Variable `LocalHandlesToSave`.
- Bug 4 (EquipmentAttributes no se acumulaba) — ✅ RESUELTO. Causa doble: `EquipmentSlots`
  13 de 23 elementos + pines sin conectar en `Switch on E_EquipmentSlot`. Checklist actualizada
  en `07_BLUEPRINT_BP_Character_Player.md`.
- Bug 5 (cascada de slots vacíos al desequipar) — 🟡 **Mitigado por diseño** con el orden de
  remoción forzado (`bRemovalLocked`); pendiente validación explícita. Ver sección Bug 5.

---

## Pendientes fuera de alcance actual

| Caso de uso | Estado |
|---|---|
| BaseState en BP_Character_Base | Evaluar migración para todos los personajes | ⏳ |
| Estados en consumibles | Pospuesto — uso ambiguo | ⏳ |
| Causa raíz del re-disparo de EquipmentChanged en cada tick de daño | ⏳ Mitigado con guard, causa raíz sin inspeccionar. Sigue ocurriendo (2 veces por frame + re-disparo cada ~1s), pero ya no es bloqueante. Confirmado que NO fue la causa de Bug 4. |
| Revisar patrón "pure function → múltiples consumidores" en otros sistemas | Ver Lección de arquitectura — candidato: `UI_CraftingQueue` y otros sistemas de queue/array | ⏳ |
| Documentar el interior de `Get Equipment Items`, `Exclude Broken Items`, `Get Items Equipment Attributes`, `Add Attribute` y `Update Equipment Attributes` | Inspeccionadas y confirmadas correctas durante Bug 4; falta redactar documentación formal en `06` o archivo nuevo | 🟡 Prioridad media |
| Validar el escenario original del Bug 5 con el orden de remoción activo | Inferencia: ya no debería ocurrir | 🟡 Pendiente |

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Error de stack duplicado en Load Status Effect | Mitigado con guard CheckStatusEffectByHandle en ApplyState | ✅ Mitigado |
| BaseState solo en BP_Character_Undead2 | Evaluar migración a BP_Character_Base | ⏳ |
| Pin ItemStates desconectado en Make Item (BP_ItemsLibrary) | Causaba Bug 1 completo | ✅ Resuelto |
| Bug 2 — CheckContainerSlotForItem fallaba para slots de runa duplicados | Fix: OR adicional. **⚠️ Verificar al abrir la próxima sesión que el fix sigue presente y guardado en el grafo.** | ✅ Resuelto (pendiente reverificación) |
| Bug 3 — remoción de Estado al desequipar no funciona | Fix: Local Variable `LocalHandlesToSave`. | ✅ Resuelto |
| Bug 4 — EquipmentAttributes no se acumulaba entre múltiples slots | Fix aplicado y confirmado en juego. | ✅ Resuelto |
| Bug 5 — cascada de slots vacíos al desequipar runa de slot inferior | Mitigado por orden de remoción forzado (`bRemovalLocked`). Pendiente validar y revisar bypass vía llamadas directas a `TryMoveItemToContainerSlot` | 🟡 Mitigado — pendiente validación |
| Bug 6 (UI) — runa sobre slot de runa ocupado dispara cooldown/candado | Ver `04` y `13` | 🔴 Abierto |
| Print String "No es runa" debug temporal en UI_ItemSlot.OnDrop | Pendiente eliminar | ⏳ |
| Causa raíz de re-disparo de EquipmentChanged por tick de daño | No inspeccionada — solo mitigada con guard. NO fue la causa de Bug 4. | ⏳ Media prioridad |
| Print String de diagnóstico del Bug 3 (PUNTO 1, 1b, 1c, ADD REAL, FIND REMOVE, MAP REMOVE EJECUTADO, REMOVE - Success, EQCHANGED START, CONTAINS KEY) | Mantenidos intencionalmente por decisión del usuario, para mostrar el trabajo de debugging al cliente. Pendientes de limpieza en sesión futura. | ⏳ Limpieza pospuesta — decisión del usuario |
| `BP_RemoveEffectTrigger` (trigger de prueba) | Puede quedar como herramienta de testing o eliminarse — decisión del usuario. | ⏳ Sin decisión — bajo impacto |
| Revisar patrón pure-function en otros sistemas de arrays/queues | Ver Lección de arquitectura | ⏳ Sin prioridad asignada |
| Interior de `Get Equipment Items` / `Exclude Broken Items` / `Get Items Equipment Attributes` / `Add Attribute` / `Update Equipment Attributes` sin documentar formalmente | Falta redactar su documentación formal en un .md | 🟡 Prioridad media |
| Print String de diagnóstico de Bug 4 (Length antes de Update Equipment Attributes) | Pendiente eliminar tras confirmar estabilidad del fix | ⏳ Limpieza pospuesta |

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
> **Actualización sesión Bug 3:** el patrón de "Local Variable intermedia antes de
> Append/Make Struct" que resolvió el Bug 3 es directamente aplicable si Logica 5
> usa un patrón similar de struct + array para persistir configuraciones de runas por ítem.
>
> **Actualización sesión Bug 4:** si `EquipmentAttributes` necesita persistencia
> por-ítem también (no solo `ItemStates`), la solución debe diseñarse teniendo en
> cuenta que ambos campos conviven en el mismo `STR_ItemData` y podrían compartir el
> mismo mecanismo de persistencia futuro. No tomar decisiones de Fase 2 de Logica 5
> sin considerar esto.
>
> **Actualización sesión Orden de remoción de runas:** el orden de remoción forzado
> (`bRemovalLocked`) y el recálculo en `UpdateRuneRemovalLocks` leen el estado de los slots
> del `EquipmentContainer`. Si la persistencia por ítem cambia cómo se cargan/descargan
> las runas al equipar/desequipar, ese recálculo deberá seguir llamándose tras cada carga.

---

*Archivo actualizado — sesión Light Paradox (Orden de remoción de runas)*
*Cambios: Bug 5 reclasificado como mitigado por diseño (orden de remoción forzado, pendiente validación y límite de UI-only); Bug 6 de UI (runa sobre slot ocupado) registrado con referencia a `04` y `13`; nota de aplicación de la lección del Bug 3 en `UpdateRuneRemovalLocks`; nota para Logica 5 ampliada. El resto del contenido (Bugs 1–4, ApplyState, DT_States) sin cambios.*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
