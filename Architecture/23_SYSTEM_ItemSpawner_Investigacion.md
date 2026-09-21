# 23 — Sistema: Item Spawner / Spawn Nocturno — Investigación + Prototipo v1
### Sistema: Aparición condicional de ítems vinculada al ciclo día/noche
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspección directa del panel Details + Export .txt de EventGraph + implementación en sesión — Light Paradox
### Proyecto: Light Paradox · UE 5.4.4
### Estado: ✅ Prototipo v1 (Ruta B) implementado y confirmado en juego · 🔴 Investigación de Ruta A (spawn dinámico real) sigue pendiente

---

## Contexto y objetivo

Continuación del trabajo de `22_SYSTEM_LunarPhases_EclipseStates.md` (Poison
nocturno, ya resuelto y funcional). Siguiente prototipo: un ítem que
spawnee **exclusivamente de noche** o con **mayor probabilidad de noche**,
reutilizando — si es posible — el mismo booleano de noche ya calculado en
`CheckLight` (`BP_DayNight`), siguiendo la prioridad del proyecto de mínimo
cambio y reutilización de arquitectura existente.

Referencia visual usada por el cliente para iniciar la investigación: el
área de spawn de cangrejos en `Overview_Main` (nivel base de ESRPGv5).

**Actor de referencia inspeccionado en el nivel:** `BP_SpawnPoint20`
(instancia), clase `BP_SpawnPoint`.

---

## Hallazgo principal — dos sistemas distintos, NO intercambiables sin más análisis

Se inspeccionaron dos Blueprints distintos para determinar cuál es
responsable del spawn de criaturas y si alguno sirve para ítems:

| Blueprint | Rol confirmado | ¿Sirve para spawn de ítems? |
|---|---|---|
| `BP_SpawnPoint` | Spawner de personajes/criaturas (IA, enemigos, animales) | 🟡 Posible — depende del tipo exacto del pin `Character Class`. Pendiente confirmar. |
| `BP_Item` | Actor de ítem estático colocado en el nivel | ❌ Confirmado que NO tiene lógica de spawn condicional propia |

Esto confirma la inferencia inicial del usuario: **los ítems y las IAs
tienen lógica de spawn distinta**, o al menos `BP_Item` como tal no fue
diseñado para aparición condicional — ver detalle abajo.

---

## BP_SpawnPoint — Spawner de criaturas

### Variables confirmadas (panel Details de la instancia `BP_SpawnPoint20`)

| Variable | Valor confirmado | Notas |
|---|---|---|
| `Settings Handle` → `Data Table` / `Row Name` | `None` / `None` | No configurado en esta instancia — posible fuente de datos alternativa a `Character Class` directo, sin usar aquí |
| `Character Class` | `BP_Character_Crab` | **Class Reference**. Tipo exacto (a qué clase base está restringido el pin) — pendiente de confirmar, ver Acción Pendiente 1 |
| `Random Classes` | 0 Array elements | Array de clases alternativas para randomizar qué criatura spawnea — vacío en esta instancia |
| `Spawn Radius` | 300.0 | Radio del área de spawn |
| `Max Spawn Count` | 1 | Máximo de instancias vivas simultáneas |
| `Initial Spawn Count` | 1 | Cantidad al iniciar |
| `Initial Spawn Max` | True (checked) | |
| `Respawn Time` | 30.0 | Segundos entre respawns |
| `Respawn Count` | 1 | |
| `Respawn to Max` | False | |
| `Start Respawn on Count` | 0 | |
| `Start Respawn on Any Death` | False | |
| `Override Character Level` | False | |
| `Min Character Level` / `Max Character Level` | 1 / 1 | |
| `Overridden Loot List` | (array, no expandido) | Existe un campo de loot separado del `Character Class` — no inspeccionado en detalle |
| `Overridden Behavior` | None | |
| `Override Relationships` | False | |
| `Spawn Faction` | Player | Sistema de facciones — no aplica conceptualmente a un ítem estático |
| `Spawn Relationships` / `Relationships Handle` | (arrays, no expandidos) | Sistema de relaciones entre facciones — no aplica a ítems |

> **Nota de arquitectura:** la presencia de `Spawn Faction`, `Spawn
> Relationships` y `Relationships Handle` confirma que este spawner está
> diseñado pensando en actores con comportamiento de IA (relaciones
> hostil/neutral/aliado), no en objetos estáticos. Esto es evidencia — no
> aún confirmación definitiva — de que `BP_SpawnPoint` podría no ser
> directamente reutilizable para ítems sin adaptación.

### Funciones confirmadas (panel My Blueprint, categoría "Spawn Point")

```
TrySpawn
Spawn
GetSpawnLocationAndRotation
CharacterDied
ActivateSpawnPoint
DeactivateSpawnPoint
CheckRespawn
CheckOverlapActivation
CheckPlayerTagQuery
CheckGameTagQuery
InitSpawnedCharacter
StartRespawnTimer
CharacterDestroyed
RemoveSpawnedCharacter
TryRespawn
CheckOverlapDeactivation
FindSpawnClass
SetSpawnSettings
InitSpawnPoint
```

> Interior de estas funciones **no exportado todavía**. Ver Acción
> Pendiente 2 — se requiere el export `.txt` de `FindSpawnClass`,
> `TrySpawn`, `CheckRespawn` y `StartRespawnTimer` para confirmar si el
> sistema es reutilizable para ítems o si está atado estructuralmente a
> lógica de personaje/IA.

### EventGraph confirmado (export directo)

```
Begin play.
  Event BeginPlay
    → Init Spawn Point (Target: self)
      → Branch (Condition ← Activate on Begin Play)
          True  → Activate Spawn Point (Target: self)
          False → [sin conexión, no hace nada]

Add and remove active players.
  AddActivePlayer (Custom Event, Input: Player Controller)
    → ADDUNIQUE (Target Array: Active Players, Item: Player Controller)
      → Check Overlap Activation (Target: self)

  RemoveActivePlayer (Custom Event, Input: Player Controller)
    → REMOVE (Target Array: Active Players, Item: Player Controller)
      → Check Overlap Deactivation (Target: self)
```

> **Nota de arquitectura confirmada:** el spawner **no corre en un timer
> puro y constante**. Su activación depende de un array `Active Players`
> mantenido vía `AddActivePlayer`/`RemoveActivePlayer`, disparando
> `CheckOverlapActivation`/`CheckOverlapDeactivation`. Es decir: el spawn
> point solo está "activo" (y presumiblemente solo intenta spawnear/
> respawnear) mientras hay al menos un jugador dentro de su rango de
> relevancia — patrón de optimización estándar de ESRPGv5 para no correr
> lógica de spawn en zonas sin jugadores cerca.
>
> Esto es relevante para el diseño del hook nocturno: si se reutiliza este
> sistema, el gate de noche no puede simplemente vivir "siempre corriendo"
> — debe insertarse en un punto que sí se ejecute cuando el spawn point ya
> está activo (candidato más probable: dentro de `TrySpawn` o
> `CheckRespawn`, pendiente confirmar con el export).

---

## BP_Item — Actor de ítem estático (NO es un spawner)

### Contexto de la instancia inspeccionada

`BP_Item89` en el nivel, configurado con `Item Handle` → `DT_Items` /
`Equipment_WoodenHat`. Visualmente el ítem "reaparece" al interactuar
repetidamente, lo cual generó la duda inicial de si `BP_Item` tenía lógica
de spawn propia.

### Variables confirmadas (panel Details)

| Variable | Valor confirmado | Notas |
|---|---|---|
| `Item Handle` → `Data Table` / `Row Name` | `DT_Items` / `Equipment_WoodenHat` | Referencia fija y directa a la fila del ítem — sin randomización |
| `Amount` | 10 | |
| `Charges` | 0 | |
| `Durability` | 100.0 | |
| `Decay` | 0.0 | |
| `Simulate Physics` | False | |
| `Can Decay` | False | |
| **`Infinity Item`** | **True** | **Clave del comportamiento observado — ver explicación abajo** |
| `Save ID` | -697 | |
| `Actor Hidden In Game` | False | |
| `Editor Billboard Scale` | 0.2 | |
| `Can be Damaged` | True | |
| `Initial Life Span` | 0.0 | |
| `Spawn Collision Handling Method` | Always Spawn, Ignore Collisions | |

### Funciones confirmadas (panel My Blueprint, categoría "Item")

```
ConstructionInit
PlayDropSound
DecayTick
DestroyItem
Probability   ← nombre potencialmente relevante, interior no inspeccionado
```

> **⚠️ Pendiente crítico:** la función `Probability` no fue exportada
> todavía. Su nombre sugiere que podría existir lógica de probabilidad ya
> pensada para ítems en el asset base — no descartar `BP_Item` como
> candidato hasta confirmar el interior de esta función. Ver Acción
> Pendiente 3.

### EventGraph confirmado (secciones relevantes del export)

**Sección "Interactions" — explica el comportamiento "endless":**

```
Event Try Interact BPI (Input: From Interaction Object, Player Controller)
  → Branch (Release condition)
    → Get Inventory Container BPI (Target: Player Controller)
      → Try Add Item (Target: Inventory Container, Item: ItemData)
        → Play Sound (Multicast, Target: self)
          → Branch (Success de Try Add Item)
            → True →
                Branch (Condition ← NOT Infinity Item)
                  True  → Destroy Actor (Target: self)
                  False → [sin conexión — el actor permanece en el nivel]
            → False → [sin conexión]
```

**Confirmación de la causa raíz del comportamiento "endless":** con
`Infinity Item = True`, la rama `NOT Infinity Item` evalúa `False`, por lo
que `Destroy Actor` **nunca se ejecuta**. El actor `BP_Item` físico se
queda permanentemente en el nivel entregando una copia del ítem al
inventario en cada interacción exitosa. **No hay timer, no hay
`Random`, no hay condición de aparición/desaparición en ningún punto del
EventGraph inspeccionado.**

**Sección "Begin play" y "Effects":** exportadas en su totalidad — cubren
inicialización de decay/destroy timers, física de drop, sonido de drop y
efecto de splash en agua. Ninguna de estas secciones contiene lógica de
spawn condicional, probabilidad, ni referencias a hora del día.

### Conclusión confirmada sobre BP_Item

`BP_Item` **no está diseñado para aparición/spawn condicional**. Su
comportamiento "endless" es una propiedad de **persistencia** (`Infinity
Item` evita que se destruya al ser recogido), no de **generación**. No es,
en su forma actual, un candidato funcional para "aparece solo de noche" —
salvo que la función `Probability` (sin inspeccionar) contenga algo
relevante no visto todavía en el EventGraph.

---

## Comparación — por qué no son intercambiables sin adaptación

| Aspecto | BP_SpawnPoint | BP_Item |
|---|---|---|
| Qué gestiona | Aparición/desaparición/respawn de una instancia | Persistencia y entrega de un ítem ya colocado |
| Tiene timer de respawn | ✅ `Respawn Time`, `StartRespawnTimer` | ❌ No existe |
| Tiene lógica de probabilidad/randomización | 🟡 `Random Classes` (array, sin inspeccionar interior de selección) | 🟡 Función `Probability` sin inspeccionar |
| Depende de proximidad de jugador | ✅ `Active Players` / Overlap Activation | ❌ No aplica — el ítem ya está en el mundo |
| Pensado para IA/facciones | ✅ `Spawn Faction`, `Relationships` | ❌ No aplica |
| Se destruye/recrea | ✅ Es su función central | ❌ Solo se destruye una vez recogido (si `Infinity Item = False`) |

---

## Prototipo v1 (Ruta B) — ✅ CONFIRMADO FUNCIONAL EN JUEGO

### Decisión de scope

Se decidió prototipar **Ruta B** primero (sistema nuevo, pequeño, dedicado
a ítems) antes de seguir investigando si **Ruta A** (reutilizar
`BP_SpawnPoint` apuntando `Character Class` a una clase basada en
`BP_Item`) es viable a largo plazo. Ruta A queda pendiente — ver sección
"Acciones pendientes" más abajo, sin cambios respecto a la sesión anterior.

Para esta v1 **no se implementó spawn dinámico real** (aparición aleatoria
dentro de un radio). En su lugar: el ítem ya está colocado de forma fija
en el nivel (siguiendo el flujo de colocación manual ya usado por el
usuario — arrastrar `BP_Item`/hija, configurar `Row Name`), y se controla
su **visibilidad + colisión** en runtime según el mismo booleano de noche
ya calculado en `CheckLight`. Visualmente el resultado es indistinguible
de un spawn nocturno. Cuando se investigue Ruta A o un spawn dinámico real
con radio/probabilidad, el gate de noche construido acá se reutiliza sin
cambios — solo cambia qué sistema lo consume.

### Clase nueva — `BP_Item_NightOnly`

**Tipo:** Child Blueprint Class de `BP_Item` (respeta Rule 4.1 — no se
modificó el asset base).

**Custom Event: `SetNightVisible`**

**Input:** `bVisible` (Boolean)

```
SetNightVisible (bVisible)
  → Set Actor Hidden In Game (Target: self, New Hidden In Game: NOT bVisible)
    → Set Actor Enable Collision (Target: self, New Collision Enabled: bVisible)
```

**Event BeginPlay (propio de la clase hija, adicional al de `BP_Item`):**

```
Event BeginPlay → SetNightVisible (bVisible: False)
```

Arranca oculto/sin colisión hasta que `CheckLight` evalúe por primera vez
si es de noche o de día.

### Instancia en el nivel

Colocada siguiendo el flujo estándar ya usado por el proyecto: arrastrar
`BP_Item_NightOnly` a la escena → panel Details → `Item Handle` → `Row
Name` → `Component_Corn`. El resto de propiedades (Static Mesh, Material,
Amount, Durability, etc.) se auto-configuran igual que con `BP_Item`
normal — mecanismo de auto-configuración por `Row Name` aún no
inspeccionado a fondo (ver Deuda Técnica).

### Hook en `BP_DayNight`

**Variable nueva:** `Night Only Items` — Array de `BP_Item_NightOnly`,
Instance Editable. Permite escalar el sistema simplemente agregando más
instancias al array, sin tocar el grafo de nuevo.

**Modificación en `CheckLight`:** se agregó un tercer pin al `Sequence`
existente (`Then 2`), reutilizando el mismo `NOT(In Range(Time, Sunrise
Time, Sunset Time))` que ya alimenta la rama de luces y la de
`State_Poison` — sin duplicar lógica de día/noche en un tercer lugar del
proyecto, consistente con el patrón ya establecido en
`22_SYSTEM_LunarPhases_EclipseStates.md`.

```
CheckLight
  → Sequence
      Then 0 → [luces, sin cambios]
      Then 1 → [Poison, sin cambios]
      Then 2 → For Each Loop (Array: Night Only Items)
                 Loop Body → Array Element → Set Night Visible
                               (bVisible ← mismo NOT compartido con luces/Poison)
```

### Bug encontrado y resuelto — ítem atraviesa el suelo

**Síntoma:** al activarse de noche, el Corn caía y atravesaba el suelo en
vez de quedarse apoyado en su posición.

**Causa:** `BP_Item` (y por herencia `BP_Item_NightOnly`) tiene `Simulate
Physics` activo por defecto en su componente `StaticMesh`. Mientras el
ítem estaba oculto con colisión desactivada (`Set Actor Enable Collision:
False`), la física seguía simulando sin nada sólido con qué colisionar —
al reactivar la colisión, el objeto ya había caído fuera de su posición
original o seguía cayendo sin detenerse en el suelo.

**Fix aplicado (confirmado funcional):** desactivar `Simulate Physics` en
el componente `StaticMesh` de la instancia `BP_Item_NightOnly` colocada en
el nivel (panel Details → Static Mesh → Physics → `Simulate Physics` →
desmarcar).

**Confirmado en juego:** con `Simulate Physics = False`, el Corn aparece y
desaparece correctamente en su posición fija, sin atravesar el suelo.

**Nota de escalabilidad:** este fix es **por instancia**, no por clase —
cada nueva instancia de `BP_Item_NightOnly` que se coloque en el nivel
deberá tener `Simulate Physics` desactivado manualmente en su Static Mesh,
salvo que se decida más adelante desactivarlo por defecto en la clase hija
(no hecho en esta v1, para no alterar el comportamiento heredado de
`BP_Item` más de lo estrictamente necesario para el prototipo).

### Checklist de replicación para nuevos ítems nocturnos

1. Colocar `BP_Item_NightOnly` en el nivel (no `BP_Item` base).
2. Configurar `Item Handle` → `Row Name` con el ítem deseado (cualquier
   fila de `DT_Items`).
3. Desactivar `Simulate Physics` en su componente Static Mesh.
4. Agregar la instancia al array `Night Only Items` en `BP_DayNight`.
5. Play — validar aparición/desaparición con el ciclo.

---

## Acciones pendientes — próxima sesión

| # | Acción | Objetivo |
|---|---|---|
| 1 | Confirmar **Variable Type** exacto de `Character Class` y `Random Classes` en el panel Details de la variable (no del graph) dentro de `BP_SpawnPoint` | Determinar si el pin acepta cualquier subclase de Actor (→ `BP_Item` podría enchufarse) o está restringido a una clase base de personaje/IA |
| 2 | Exportar `.txt` de `FindSpawnClass`, `TrySpawn`, `CheckRespawn`, `StartRespawnTimer` (`BP_SpawnPoint`) | Confirmar si la lógica interna asume comportamiento de personaje (IA, animaciones, facciones) que rompería si el spawneado fuera un ítem |
| 3 | Exportar `.txt` de la función `Probability` (`BP_Item`) | Determinar si ya existe un mecanismo de probabilidad pensado para ítems en el asset base, antes de descartar esta ruta |
| 4 | Con los 3 puntos anteriores, decidir entre: **Ruta A** (reutilizar `BP_SpawnPoint` apuntando `Character Class`/`Random Classes` a una clase que herede de `BP_Item`) o **Ruta B** (Blueprint nuevo, liviano, combinando el patrón overlap-activation de `BP_SpawnPoint` + timer propio + gate de noche) | Definir arquitectura final antes de tocar cualquier grafo |
| 5 | Una vez elegida la ruta, diseñar el gate de noche reutilizando el mismo booleano `NOT(In Range(Time, Sunrise Time, Sunset Time))` ya usado en `CheckLight` (`BP_DayNight`), sin duplicar lógica de día/noche en un tercer lugar del proyecto — mismo patrón que `State_Poison` | Mantener consistencia arquitectural del proyecto |
| 6 | Definir con el cliente qué ítem se usará para el prototipo (random, aún no elegido) | Necesario antes de configurar `Item Handle` / `Character Class` en la instancia de prueba |

---

## Deuda técnica / hallazgos registrados

| Problema | Notas | Estado |
|---|---|---|
| Interior de `FindSpawnClass`, `TrySpawn`, `CheckRespawn`, `StartRespawnTimer` sin inspeccionar | Bloqueante para decidir Ruta A vs Ruta B | 🔴 Pendiente — acción inmediata próxima sesión |
| Función `Probability` en `BP_Item` sin inspeccionar | Nombre sugiere posible lógica de probabilidad ya existente para ítems — no descartar `BP_Item` sin confirmar | 🔴 Pendiente |
| Tipo exacto del pin `Character Class` / `Random Classes` sin confirmar | Determina si `BP_SpawnPoint` puede apuntar a una clase basada en `BP_Item` | 🔴 Pendiente |
| `Overridden Loot List` en `BP_SpawnPoint` sin expandir/inspeccionar | Podría ser un mecanismo de loot ya existente relevante para ítems, separado de `Character Class` | 🟡 Pendiente, prioridad media |
| `Settings Handle` (Data Table Row Handle) en `BP_SpawnPoint` sin configurar en la instancia inspeccionada | Posible ruta data-driven alternativa a configurar `Character Class` manualmente por instancia — no explorada | 🟡 Pendiente, prioridad media |
| `Simulate Physics` desactivado manualmente por instancia en `BP_Item_NightOnly` | Fix del bug "ítem atraviesa el suelo" al ocultar/mostrar vía colisión. Es un ajuste por instancia, no por clase — cada nueva instancia nocturna lo necesita manualmente. Evaluar en sesión futura si conviene desactivarlo por defecto en la clase hija una vez que el sistema escale a más ítems. | 🟡 Resuelto para v1 — replicación manual pendiente de automatizar |
| Mecanismo de auto-configuración de `BP_Item` por `Row Name` (Static Mesh, Amount, Durability, etc.) | Usado activamente por el flujo de colocación del usuario pero nunca inspeccionado — probablemente vive en `ConstructionScript` o `ConstructionInit` (ver `06_BLUEPRINT_BP_ContainerComponent.md`/panel My Blueprint de `BP_Item`, Imagen 6 de sesión anterior) | ⏳ Pendiente, prioridad baja — no bloqueante para v1 |

---

*Archivo actualizado — sesión Light Paradox (Prototipo v1, Ruta B, implementado y confirmado en juego).*
*Cambios: agregada clase `BP_Item_NightOnly` (child de `BP_Item`) con `SetNightVisible`; hook nuevo en `CheckLight` (`BP_DayNight`) reutilizando el booleano de noche compartido con luces y `State_Poison`; variable `Night Only Items` (array, Instance Editable) para escalar agregando instancias; bug de física (`Simulate Physics` atravesando el suelo) documentado y resuelto por instancia; checklist de replicación agregada. Investigación de Ruta A (`BP_SpawnPoint`) permanece pendiente sin cambios — ver sección "Acciones pendientes".*
*Continúa el trabajo de `22_SYSTEM_LunarPhases_EclipseStates.md`.*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
