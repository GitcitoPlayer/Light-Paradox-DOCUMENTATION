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

## 🔴 Bug 4 (NUEVO — abierto esta sesión) — EquipmentAttributes no se aplica sin ItemStates asignado

**Estado:** 🔴 Activo — diagnóstico iniciado, pendiente de confirmación en
próxima sesión antes de aplicar cualquier fix.

### Síntoma original reportado (correlación con ItemStates — DESCARTADA)

Con una Rune Word que tiene:
- `EquipmentAttributes`: 1 elemento (`EasyRPG.Attributes.Base.EnergyRegeneration`, Value `30.0`)
- `ItemStates`: 1 elemento (`DT_States` → `State_Poison`)

...al equipar la runa en el **primer slot**, `EnergyRegeneration` sí sube
+30 correctamente. Al equipar una **segunda** runa igual en el segundo slot,
el atributo no vuelve a subir. La hipótesis inicial fue que esto dependía de
`ItemStates` — **descartada, ver sección "Síntoma real confirmado" abajo.**

### ✅ Wiring de `EquipmentChanged` confirmado — NO es la causa

Se inspeccionó directamente el fragmento del grafo entre el `Switch on
E_EquipmentSlot` (que actualiza los meshes visibles por slot) y
`Update Equipment Attributes`. Confirmado por captura:

```
Switch on E_EquipmentSlot → [SET w/ Notify de mesh por slot, uno por cada
  Equipment Slot] → (todas las ramas convergen en un nodo único) →
  Update Footstep Settings → Update Equipment Attributes
```

`Update Equipment Attributes` recibe su input `Equipment Attributes` desde
esta cadena de funciones puras encadenadas (ninguna documentada aún en el
proyecto — candidatas para inspección):

```
Get Equipment Items (Target: Equipment Container)
  → Exclude Broken Items
  → Get Items Equipment Attributes
  → Equipment Attributes (output)
```

**Confirmado: no hay ningún `For Each Loop` de `Item States` antes de
`Update Equipment Attributes`.** La cadena es lineal e incondicional — se
ejecuta siempre que cambia el equipo, sin importar si el ítem tiene
`ItemStates` o no. Esto **descarta por completo** la hipótesis original de
"wiring colgado del `Loop Body`".

### Prueba de aislamiento — confirmado que `ItemStates` NO es la variable relevante

El usuario ejecutó tres pruebas control:

1. **Re-equipar la misma runa (sin ItemStates) en el mismo slot / en otro
   slot vacío** → el atributo sigue sin sumar. Descarta timing/lectura
   obsoleta de `Get Equipment Items` como causa (una relectura forzada no
   corrige nada).
2. **Esperar varios segundos sin tocar nada tras equipar sin ItemStates**
   → el atributo nunca "se corrige solo". Descarta que dependa de un
   re-disparo posterior de `EquipmentChanged` vía tick de Status Effect.
3. **Desconectar por completo la lógica de Estados de Light Paradox
   (Bloque A/B) de `EquipmentChanged`, dejando solo la cadena base de
   ESRPGv5** → el patrón se mantiene idéntico: el primer slot suma el
   atributo, los siguientes no, **sin importar `ItemStates` en absoluto**.

**Conclusión confirmada: Bug 4 no tiene relación con `ItemStates` ni con
la lógica de Light Paradox. El bug vive completamente dentro de la cadena
base de ESRPGv5** (`Get Equipment Items` → `Exclude Broken Items` →
`Get Items Equipment Attributes` → `Update Equipment Attributes`).

### Síntoma real confirmado

Al equipar múltiples Rune Words del mismo tipo de atributo
(`EnergyRegeneration`) en slots distintos: **solo el valor del primer slot
llega a aplicarse.** Los valores de slots subsecuentes no se suman —
tampoco se sobreescriben visiblemente porque el valor es idéntico (30.0 en
ambos casos), lo que hace parecer que "no pasa nada" en el segundo slot.

### Hallazgo relacionado (bug distinto, registrado pero fuera de alcance por ahora)

Durante las pruebas se detectó un bug adicional en la cadena de visibilidad
de slots de runa: al desequipar la runa del slot 1 (con el slot 2 también
ocupado), la cadena de reacción visual dejaba un par de slots vacíos de
forma inconsistente; al volver a colocar una runa en el slot vacío
resultante, esa runa sí sumaba correctamente su valor sobre el atributo ya
existente. **El usuario ya tiene una solución de diseño planeada para esto
— no se investiga en esta fase.** Registrado como Bug 5 en la tabla de
Deuda Técnica al final de este archivo, solo como pista relacionada: es
coherente con la hipótesis de que la agregación de atributos (o la cadena
de visibilidad de runas) agrupa por `EquipmentType`/tag compartido en vez
de por slot individual — el mismo problema de fondo que causó el Bug 2
(ver `06_BLUEPRINT_BP_EquipmentComponent.md`, donde todos los slots
duplicados de Head Rune Word comparten `EquipmentType = HeadRuneWord (7)`).

### Nueva hipótesis principal — Map keyed por GameplayTag sobreescribiendo en vez de acumular

Candidato: `Get Items Equipment Attributes` (o la función real donde ocurra
la agregación — ubicación exacta aún sin confirmar, puede vivir en
`BP_EquipmentComponent` o `BP_ItemsLibrary`, mismo patrón de incertidumbre
que `Item Is Equipment`) probablemente itera los ítems equipados y agrega
cada `GameplayTag` + `Value` a un **Map** usando un nodo `Add`/`Set` **sin
verificar antes si esa Key ya existe**. Si dos ítems equipados comparten el
mismo `GameplayTag` (`EnergyRegeneration`), el segundo **sobreescribe** la
entrada del primero en el Map en vez de sumarse a ella. Como ambas runas de
prueba tienen el mismo valor (30.0), el resultado final se ve idéntico a
"no se aplicó nada nuevo" — pero en realidad hubo una sobreescritura
silenciosa con el mismo número.

> **Nivel de confianza:** hipótesis, no confirmada. Requiere inspección
> directa del interior de la función real de agregación antes de aplicar
> cualquier fix.

### Checklist de diagnóstico para la próxima sesión

1. Abrir `BP_EquipmentComponent`. Si `Get Items Equipment Attributes` no
   está ahí, buscarla en `BP_ItemsLibrary` (mismo patrón de incertidumbre
   que `Item Is Equipment` — confirmar ubicación real y documentarla).
2. Doble click para abrir el interior de la función.
3. Buscar visualmente un nodo tipo **Map** (variable Map, o nodo
   `Add (Map)` / `Set (Map)`) dentro de un loop que itere sobre los ítems
   equipados.
4. Si se encuentra: click en el pin de entrada de ejecución del nodo
   `Add`/`Set` y seguir el cable **hacia atrás**. Confirmar si antes hay un
   `Find (Map)` + `Branch` (que verificaría si la Key ya existe y sumaría
   al valor existente) o si el `Add`/`Set` corre directo sin ese chequeo.
5. Si hay duda visual: agregar un `Print String` (Development Only,
   **Output Log**) justo antes del nodo que entrega el `Equipment
   Attributes` de salida de la función, imprimiendo el `Length` del
   array/map resultante y, si es posible, cada elemento (`GameplayTag` +
   `Value`) vía un `For Each Loop` temporal de depuración.
6. Probar en PIE: equipar runa en slot 1 → revisar log → equipar runa en
   slot 2 → revisar log de nuevo.
7. Comparar resultados:

| Resultado del segundo log | Diagnóstico | Siguiente paso |
|---|---|---|
| Muestra 2 entradas separadas, cada una con 30.0 | La agregación en este punto es correcta — el problema está más adelante, dentro de `Update Equipment Attributes` mismo (posiblemente toma solo el primer/último elemento del array en vez de sumar todos) | Inspeccionar el interior de `Update Equipment Attributes` |
| Muestra 1 sola entrada con 30.0 | Confirma la hipótesis — el Map sobreescribe por Key en vez de acumular, dentro de `Get Items Equipment Attributes` | Insertar un `Find (Map)` + `Branch` antes del `Add`/`Set`: si la Key existe, sumar el nuevo `Value` al existente antes de escribir; si no existe, `Add` normal |

8. Una vez identificada la causa exacta, actualizar esta sección con el fix
   aplicado (mismo formato que Bug 2 y Bug 3), y documentar por primera vez
   el interior de las funciones involucradas (`Get Equipment Items`,
   `Exclude Broken Items`, `Get Items Equipment Attributes`,
   `Update Equipment Attributes`) en un archivo nuevo o en
   `07_BLUEPRINT_BP_Character_Player.md` / `06_BLUEPRINT_BP_EquipmentComponent.md`
   según dónde vivan realmente — actualmente ninguna tiene documentación en
   el proyecto.

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
- Bug 4 (EquipmentAttributes no se acumula entre múltiples slots) —
  🔴 **Activo. Hipótesis corregida esta sesión** — descartada relación con
  `ItemStates` (confirmado por prueba de aislamiento con la lógica de
  Estados desconectada). Nueva hipótesis: Map keyed por GameplayTag
  sobreescribiendo en vez de acumular, dentro de la cadena base de ESRPGv5
  (`Get Equipment Items` → `Exclude Broken Items` →
  `Get Items Equipment Attributes` → `Update Equipment Attributes`).
  Checklist de diagnóstico lista para la próxima sesión — ver sección
  Bug 4 completa arriba.
- Bug 5 (cascada de slots vacíos al desequipar runa de un slot inferior) —
  🟡 **Nuevo, detectado como efecto colateral durante pruebas de Bug 4.**
  Con Bug 4 en curso no bloqueante para el gameplay actual. El usuario ya
  tiene solución de diseño planeada — no se investiga en esta fase. Ver
  nota en sección Bug 4 ("Hallazgo relacionado").

---

## Pendientes fuera de alcance actual

| Caso de uso | Estado |
|---|---|
| BaseState en BP_Character_Base | Evaluar migración para todos los personajes | ⏳ |
| Estados en consumibles | Pospuesto — uso ambiguo | ⏳ |
| Causa raíz del re-disparo de EquipmentChanged en cada tick de daño | ⏳ Mitigado con guard, causa raíz sin inspeccionar. Sigue ocurriendo (2 veces por frame + re-disparo cada ~1s), pero ya no es bloqueante ahora que el Bug 3 está resuelto. Confirmado esta sesión que este re-disparo NO es la causa de Bug 4 (probado con espera prolongada sin corrección espontánea del atributo). |
| Revisar patrón "pure function → múltiples consumidores" en otros sistemas | Ver Lección de arquitectura arriba — candidato: `UI_CraftingQueue` y otros sistemas de queue/array | ⏳ |
| **Actualizado:** Documentar el interior de `Get Equipment Items`, `Exclude Broken Items`, `Get Items Equipment Attributes` y `Update Equipment Attributes` | Ninguna de las cuatro tiene documentación en el proyecto. Necesario para cerrar Bug 4 — ver checklist en sección Bug 4 | 🔴 Prioridad próxima sesión |
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
| **Bug 4 — EquipmentAttributes no se acumula entre múltiples slots del mismo tipo** | Descartada relación con ItemStates. Nueva hipótesis: Map por GameplayTag sobreescribiendo en Get Items Equipment Attributes (o función equivalente de ESRPGv5, no documentada). Ver checklist de diagnóstico completa en sección Bug 4. | 🔴 **Activo — prioridad próxima sesión** |
| **Bug 5 (nuevo) — cascada de slots vacíos al desequipar runa de slot inferior** | Detectado durante pruebas de Bug 4. Al desequipar runa del slot 1 con slot 2 ocupado, la cadena de visibilidad deja slots vacíos inconsistentes; el slot vacío resultante sí acumula el atributo correctamente al recibir una runa nueva. Posible relación con el mismo problema de fondo del Bug 2 (EquipmentType compartido entre slots duplicados). Usuario ya tiene solución de diseño planeada. | 🟡 Registrado — no investigar todavía, por decisión del usuario |
| Print String "No es runa" debug temporal en UI_ItemSlot.OnDrop | Pendiente eliminar — su comportamiento confuso durante el diagnóstico del Bug 2 quedó explicado por la causa raíz real (CheckContainerSlotForItem), no por la lógica de detección de runa en sí | ⏳ |
| Causa raíz de re-disparo de EquipmentChanged por tick de daño | No inspeccionada — solo mitigada con guard. Confirmado que sigue ocurriendo, 2 veces por frame (Slot 0 + Slot 7) más el re-disparo cada ~1s. Confirmado esta sesión que NO es la causa de Bug 4. | ⏳ Media prioridad |
| Print String de diagnóstico del Bug 3 (PUNTO 1, 1b, 1c, ADD REAL, FIND REMOVE, MAP REMOVE EJECUTADO, REMOVE - Success, EQCHANGED START, CONTAINS KEY) | Mantenidos intencionalmente por decisión del usuario, para mostrar el trabajo de debugging al cliente. Pendientes de limpieza en sesión futura. | ⏳ Limpieza pospuesta — decisión del usuario |
| `BP_RemoveEffectTrigger` (trigger de prueba) | Puede quedar en el nivel como herramienta de testing o eliminarse — decisión del usuario. | ⏳ Sin decisión — bajo impacto |
| Revisar patrón pure-function en otros sistemas de arrays/queues | Ver Lección de arquitectura arriba | ⏳ Sin prioridad asignada |
| Interior de `Get Equipment Items` / `Exclude Broken Items` / `Get Items Equipment Attributes` / `Update Equipment Attributes` sin documentar | Necesario para cerrar Bug 4 correctamente. Ninguna de las cuatro funciones ha sido inspeccionada en detalle en el proyecto. | 🔴 Prioridad próxima sesión |

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

*Archivo actualizado — sesión Light Paradox (Bug 4 — hipótesis corregida tras pruebas de aislamiento, Bug 5 nuevo registrado)*
*Cambios: Bug 4 — descartada la correlación con ItemStates tras tres pruebas de aislamiento (re-equipar sin ItemStates,
espera prolongada, desconexión total de la lógica de Estados de Light Paradox); confirmado por captura que el wiring
de EquipmentChanged entre Switch on E_EquipmentSlot y Update Equipment Attributes es lineal, sin loops — hipótesis
de "Loop Body colgado" descartada; nueva hipótesis: Map por GameplayTag sobreescribiendo en vez de acumular dentro
de la cadena base de ESRPGv5 (Get Equipment Items → Exclude Broken Items → Get Items Equipment Attributes →
Update Equipment Attributes); checklist de diagnóstico reescrita para la nueva hipótesis; Bug 5 nuevo registrado
(cascada de slots vacíos al desequipar runa de slot inferior) — pendiente por decisión del usuario, con solución de
diseño ya planeada; advertencia agregada en Bug 2 para reverificar que el fix del OR sigue guardado en el grafo real*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
