# 24 — Sistema: BP_ResourceSpawnPoint — Spawn de ítems por eventos celestiales
### Sistema: Aparición de ítems (flora/recursos) y, más adelante, enemigos, ligada a `DT_CelestialEvents`
### Base Asset: EasySurvivalRPGv5
### Fuente: Diseño del usuario + `analisis_sistemas_spawn_mundos_abiertos.md` + inspección directa (Find in Blueprints, capturas de `SpawnItems` y `SpawnRandomItem`)
### Proyecto: Light Paradox · UE 5.4.4
### Estado: Etapa 0 ✅ completa · Etapa 1A ✅ implementada · Etapa 1B ⏳ siguiente (nada de `BP_ResourceSpawnPoint` construido aún)

---

## Contexto y objetivo

El placeholder de `23_SYSTEM_ItemSpawner_Investigacion.md` (`BP_Item_NightOnly` +
`Night Only Items` + `Then 2` de `CheckLight`) solo oculta/muestra una instancia
fija. Se sofistica: los ítems deben **aparecer cuando ocurre un evento celestial**
(ej. `Eclipse_Lunar`) y desaparecer al terminar.

Necesidad **explícita** del cliente: spawnear ítems o enemigos durante eventos
celestiales. Necesidad **implícita** (decisión del usuario): limitar cuántas
veces existe un elemento en el mundo.

Se mantiene **MVP por etapas**.

---

## Decisiones tomadas

| Decisión | Detalle |
|---|---|
| Flora e ítems **no** usan `BP_SpawnPoint` | Tiene parámetros exclusivos de IA (`Spawn Faction`, `Spawn Relationships`, `Overridden Behavior`, `Character Class`). Reutilizarlo obliga a ignorar media clase o a romper Rule 4.1 |
| Se crea `BP_ResourceSpawnPoint` (Actor nuevo, **no** hereda de `BP_SpawnPoint`) | Nombres de variables y funciones análogos a `BP_SpawnPoint` para que el diseño sea intuitivo |
| El spawn point **no calcula calendario** | Escucha a `BP_DayNight` vía `BPI_CelestialListener`; reutiliza los Row Names de `DT_CelestialEvents` |
| Enemigos: no crear sistema nuevo | **Inferencia:** una clase hija de `BP_SpawnPoint` que llame a sus `ActivateSpawnPoint`/`DeactivateSpawnPoint` cuando empiece/termine el evento. Interior de esas funciones sin exportar → Etapa 3 |
| `TrySpawn` usa su propio `SpawnActor BP Item` | Patrón de `BP_DebugMenuComponent.SpawnRandomItem`. No reutiliza `BP_GameInstance.SpawnItems` (ver hallazgos) |

---

## Qué del análisis de spawn entra en el MVP

| Concepto | MVP | Dónde vive |
|---|---|---|
| Spawn Conditions | ✅ Evento celestial | `BP_DayNight` ya lo calcula |
| Population Cap | ✅ Por punto (`Max Spawn Count`), sin Spawn Min/Low | Spawn point |
| Spawn Locations | ✅ Radio alrededor del actor | Spawn point |
| Persistencia al terminar el evento | ✅ Boolean `Despawn On Event End` | Spawn point |
| Respawn Rules | ❌ Etapa 2 | |
| Spawn Timing (timer) | ❌ Etapa 2 (el evento dispara el spawn) | |
| Cap global por ítem, Spawn Group, biomas, proximidad del jugador | ❌ Etapa 2 o posterior | |

> Del doc 23 solo consta `Max Spawn Count` **por punto** en `BP_SpawnPoint`.
> **No hay evidencia** de límite global por tipo de criatura (inferencia). El MVP replica el cap por punto.

---

## Arquitectura

```
BP_DayNight
  CheckCelestialEvents (ENTER / EXIT)
        │  NotifyCelestialEvent(EventName, bStarted)        ← ✅ implementado (1A)
        ▼
  Get All Actors with Interface (BPI_CelestialListener)
        │
        ▼
BP_ResourceSpawnPoint (⏳ 1B)
  OnCelestialEventStarted → ¿EventName ∈ Required Celestial Events? → ActivateSpawnPoint → TrySpawn
  OnCelestialEventEnded   → ¿EventName ∈ Required Celestial Events? → DeactivateSpawnPoint → (Despawn)
```

### Variables de `BP_ResourceSpawnPoint` (diseño)

| Variable | Tipo | Nota |
|---|---|---|
| `Item Handle` | DataTableRowHandle (`DT_Items`) | Igual que en `BP_Item` |
| `Item Amount` | Integer | Va al `Amount` del `BP_Item` |
| `Spawn Radius` | Float | Default 300 |
| `Max Spawn Count` | Integer | Cap por punto |
| `Initial Spawn Count` | Integer | Cuántos aparecen al iniciar el evento |
| `Required Celestial Events` | Array de Name | Filas de `DT_CelestialEvents`. **Vacío = no escucha nada** |
| `Despawn On Event End` | Boolean | Default True |
| `Spawned Items` | Array de `BP_Item` | **Variable real**, no salida de función pura (lección del Bug 3, doc 20) |

Funciones: `ActivateSpawnPoint`, `DeactivateSpawnPoint`, `TrySpawn`, `GetSpawnLocation`.
Eventos (de `BPI_CelestialListener`): `OnCelestialEventStarted(EventName)`, `OnCelestialEventEnded(EventName)`.

---

## Etapa 0 — Hallazgos de inspección (✅ completa)

### Búsquedas realizadas
`Drop Item` (solo `GetItemDropMesh`, devuelve un mesh), `Spawn Item` en `BP_Character_Player` (solo eventos BPI `Spawn Items BPI`, `DropUsableItem`, `AddRespawnItems_BPI`) y búsqueda general en todo el proyecto.

> **Corrección:** una sugerencia previa de buscar la función en `BP_ItemsLibrary` y `BP_Character_Player` no era el lugar correcto. El asset **no tiene una función central "spawn item"**.

### Cómo spawnea ítems el asset
`SpawnActor BP Item` directo en: `BP_Item.DecayTick`, `BP_Projectile_Arrow` / `BP_Projectile_Bolt.PlayHitEffects` (con variable `Spawn Item Handle`), `BP_Projectile_Item_Base.TrySpawnItem`, `BP_DebugMenuComponent.SpawnRandomItem`.

### `Item Handle` en `BP_Item`
Variable pública con **Expose on Spawn** marcado (confirmado por el usuario).

### `BP_DebugMenuComponent.SpawnRandomItem` (confirmado por captura)
Flujo: `Get Data Table Row Names (DT_Items)` → `Random Integer in Range` → `Make Item by ID` → `SpawnActor BP Item`. Ubicación: `Get Actor Location` + `Make Vector` con offset aleatorio.

Pines del `SpawnActor BP Item` en esa función:

| Pin | Valor |
|---|---|
| Class | `BP_Item` |
| Spawn Transform Location | actor location + offset aleatorio |
| Collision Handling Override | Default |
| Item Handle | del `Make Item by ID` |
| Amount | del `Make Item by ID` |
| Charges | `-1` |
| Durability | `-1.0` |
| Decay | `-1.0` |
| Simulate Physics | ✅ marcado (default) |
| Can Decay | ✅ marcado (default) |
| Infinity Item | desmarcado |
| Instigator | vacío |

> **Inferencia:** `-1` significa "usar el valor del Data Table". No confirmado por export.

### `BP_GameInstance.SpawnItems` (confirmado por captura)
Descripción: "Spawn items in range from target location". Inputs: `Items`, `Location`, `Min Range`, `Max Range`, `Min Height`, `Max Height`. Itera `Items`, valida con `Item Is Real`, obtiene clase con `Get Item Class`, y hace `SpawnActor` con `Collision Handling Override = Try To Adjust Location, But Always Spawn`, `Simulate Physics` marcado, `Random Rotator` y offset aleatorio.

**Descartada para nuestro caso:**
1. Dispersa ítems con física activa (como un drop) → ya causó el bug de atravesar el suelo (doc 23).
2. En la captura **no se ve un `Return Node`**: no devolvería los actores creados, necesarios para `Spawned Items`, el cap y el despawn (inferencia basada solo en la captura).
3. Recibe un array de `STR_ItemData`, no un `DataTableRowHandle`.

**Aporta como patrón:** `Collision Handling Override = Try To Adjust Location, But Always Spawn`, `Random Rotator` y obtener la clase con `Get Item Class` (validada con `Item Is Real`) en vez de fijar `BP_Item`. El MVP fija `BP_Item`; si algún ítem de `DT_Items` no es un `BP_Item` normal se verá en la prueba (no documentado).

### Funciones que NO se usarán
- `BP_ContainerComponent.SpawnDropItem` / `SpawnDropItemAtLocation`: requieren un `BP_ContainerComponent` como Target y probablemente aplican física de drop (inferencia).
- **Referencia para Etapa 2 (flora):** `BP_Building_Plant_Base.CollectPlant` — su comentario dice "try to respawn it after time"; ya existe lógica de reaparición por tiempo para plantas (interior sin inspeccionar).

---

## Etapa 1A — Notificación de eventos (✅ implementada)

Ver detalle completo en `22_SYSTEM_LunarPhases_EclipseStates.md` (sección "Notificación a suscriptores"): `BPI_CelestialListener`, `NotifyCelestialEvent` en `BP_DayNight`, llamadas en ENTER/EXIT de `CheckCelestialEvents`. Confirmada por log (`Listeners: 0` hasta crear el spawn point).

---

## Etapa 1B — `BP_ResourceSpawnPoint` (⏳ siguiente)

### Plan de nodos (aún sin construir)

1. Actor nuevo con las variables de arriba; agregar `BPI_CelestialListener` en Class Settings → Interfaces.
2. `OnCelestialEventStarted` / `OnCelestialEventEnded`: `Contains` de `Required Celestial Events` con `EventName` → `ActivateSpawnPoint` / `DeactivateSpawnPoint`.
3. `TrySpawn`:
   - `Switch Has Authority` primero (el doc 19 ya anotó que `BP_PoisonTrigger` no lo hace).
   - Purgar `Spawned Items` de elementos que no pasen `Is Valid`.
   - `Max Spawn Count − Length(Spawned Items)` → `For Loop` por esa cantidad.
   - Por iteración: `GetSpawnLocation` → `SpawnActor BP Item` → `Add` a `Spawned Items`.
4. `GetSpawnLocation`: offset 2D aleatorio dentro de `Spawn Radius` + `Line Trace By Channel` hacia abajo (Z+500 → Z−1000) y usar `Impact Point`. Sin impacto = no spawnea (inferencia: no depende de NavMesh).
5. `DeactivateSpawnPoint`: si `Despawn On Event End`, `Destroy Actor` sobre los válidos de `Spawned Items` y `Clear`.
6. `Print String` con prefijo `[Spawn]` para depurar, igual que `[Celestial]`.

### Pines del `SpawnActor BP Item` dentro de `TrySpawn`

| Pin | Valor | Razón |
|---|---|---|
| Class | `BP_Item` | Patrón de `SpawnRandomItem` |
| Spawn Transform Location | salida de `GetSpawnLocation` | |
| Spawn Transform Rotation | `Make Rotator` con Yaw = `Random Float in Range (0, 360)` | Opcional, variedad visual |
| Collision Handling Override | `Try To Adjust Location, But Always Spawn` | Patrón de `SpawnItems` |
| Item Handle | variable `Item Handle` del spawn point | |
| Amount | variable `Item Amount` | |
| Charges / Durability / Decay | `-1` / `-1.0` / `-1.0` | Mismos valores que usa el asset |
| **Simulate Physics** | **desmarcado** | Evita el bug de atravesar el suelo (doc 23). Se hace en el nodo, **no** después del spawn |
| **Can Decay** | desmarcado | Se parece a las instancias colocadas a mano (doc 23: `Can Decay = False`). Inferencia: con decay activo el ítem acabaría destruyéndose solo |
| **Infinity Item** | desmarcado | Al recogerlo se destruye (`Destroy Actor` en `Try Interact BPI`, doc 23); el actor inválido libera cupo en el cap |
| Instigator | vacío | |

El `Return Value` del `SpawnActor` va a `Add` sobre `Spawned Items` (variable real).

### Prueba 1C
1. Colocar el actor con `Item Handle` → `DT_Items` / `Component_Corn`, `Required Celestial Events` = `Eclipse_Lunar`, `Max Spawn Count` = 3.
2. Plan del doc 22 (prueba B): `Day = 64`, `Start Time = 17`. Esperado: ENTER `Eclipse_Lunar` el día 65 a las ~18:00 → 3 elotes; EXIT a las ~6:00 del día 66 → se despawnean.
3. Recoger uno: debe entrar al inventario **sin errores en el log**, y el cap no debe hacer spawn de más.

### Riesgos sin resolver
- Comportamiento de un `BP_Item` creado en runtime con `Save ID` y con el sistema de guardado (no documentado, inferencia). Guardar/cargar partida queda para una etapa posterior.
- `Despawn On Event End = False` dejaría ítems en el mundo hasta que el jugador los recoja (persistencia indefinida del análisis). No se tocará en el MVP.

---

## Etapas siguientes

| Etapa | Contenido |
|---|---|
| 2 | `Respawn Time`, `Random Items` (análogo a `Random Classes`), cap global por ítem, proximidad del jugador (`Active Players` + overlap como `BP_SpawnPoint`). Referencia de flora: `BP_Building_Plant_Base.CollectPlant` |
| 3 | `BP_SpawnPoint_Celestial` (hija de `BP_SpawnPoint`) para enemigos. Requiere exportar `ActivateSpawnPoint`, `DeactivateSpawnPoint` y `RemoveSpawnedCharacter` |
| Migración | Cuando el MVP funcione, `BP_Item_NightOnly` y `Night Only Items` (doc 23) quedan obsoletos. No tocarlos hasta confirmar el reemplazo |

---

## Deuda técnica / pendientes

| Problema | Notas | Estado |
|---|---|---|
| `BP_ResourceSpawnPoint` sin construir | Etapa 1B | ⏳ Siguiente |
| Función `Probability` de `BP_Item` sin inspeccionar | Heredado del doc 23 | 🟡 Baja prioridad |
| `BP_SpawnPoint` (`FindSpawnClass`, `TrySpawn`, `CheckRespawn`, `StartRespawnTimer`, `ActivateSpawnPoint`, `DeactivateSpawnPoint`, `RemoveSpawnedCharacter`) sin exportar | Necesario para Etapa 3 (enemigos) | ⏳ Pendiente |
| Interacción `BP_Item` runtime + `Save ID` + guardado | No documentado | 🟡 Pendiente |
| Debug `Print String` en `NotifyCelestialEvent` | Temporal | ⏳ Limpiar tras validar |

---

*Archivo creado — sesión Light Paradox (plan del spawn por eventos celestiales, Etapas 0 y 1A).*
*Ver `22_SYSTEM_LunarPhases_EclipseStates.md` (eventos y `BPI_CelestialListener`) y `23_SYSTEM_ItemSpawner_Investigacion.md` (placeholder y `BP_SpawnPoint`).*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
