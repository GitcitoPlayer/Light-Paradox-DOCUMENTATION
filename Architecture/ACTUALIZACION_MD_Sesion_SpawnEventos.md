# ACTUALIZACIÓN DE .md — Sesión: eventos celestiales que cruzan medianoche + base del spawn por eventos
### Proyecto: Light Paradox · UE 5.4.4 · Base: EasySurvivalRPGv5
### Cómo usar este archivo
- `22_SYSTEM_LunarPhases_EclipseStates.md` (reemplazo completo) y `24_SYSTEM_ResourceSpawnPoint_Plan.md` (archivo nuevo) se entregan por separado: **reemplazar / agregar al proyecto**.
- Este archivo solo trae los **parches** para `21` y `23`. Cada sección indica el .md destino y si **reemplaza** o **agrega**.

---

## 1 — `21_SYSTEM_DayNightCycle_Migration.md`

### 1.1 REEMPLAZAR en el bloque `CheckLight (Custom Event)` del EventGraph, la línea de `Then 4`
```
      Then 4 → CheckCelestialEvents (Target: self). Ver `22_SYSTEM_LunarPhases_EclipseStates.md`
               Dentro, en cada ENTER/EXIT llama NotifyCelestialEvent(EventName, bStarted)
               → avisa a los actores con BPI_CelestialListener (ver `24_SYSTEM_ResourceSpawnPoint_Plan.md`)
```

### 1.2 AGREGAR a "Assets confirmados (Content/LightParadox/Blueprints/DayNightCycle)"
| Asset | Tipo | Notas |
|---|---|---|
| `BPI_CelestialListener` | Blueprint Interface | `OnCelestialEventStarted(EventName: Name)` y `OnCelestialEventEnded(EventName: Name)`. Mismo patrón que `BPI_IsLight`. Ver `22` |
| `E_CelestialEventType`, `STR_CelestialEvent`, `DT_CelestialEvents` | Enum / Struct / DataTable | Ver `22` |

### 1.3 AGREGAR a la tabla de funciones/variables de `BP_DayNight`
| Elemento | Notas |
|---|---|
| `NotifyCelestialEvent` (función, inputs `EventName` Name, `bStarted` Boolean) | `Get All Actors with Interface (BPI_CelestialListener)` + `For Each Loop` + `Branch (bStarted)` → `On Celestial Event Started/Ended (Message)`. Se llama solo en transiciones ENTER/EXIT |

### 1.4 AGREGAR a "Deuda técnica registrada"
| Problema | Notas | Estado |
|---|---|---|
| Log `Es de Noche` impreso durante toda la noche | No documentado. **Inferencia:** `Print String` de debug de la rama Poison (`Then 1` de `CheckLight`). Verificar y limpiar | ⏳ Pendiente |
| Nodo colapsado `ResolveEventTiming` en `CheckCelestialEvents` | Propuesto, sin aplicar. Ver `22` | ⏳ Pendiente |

### 1.4 ACTUALIZAR línea final de "Checklist de pendientes"
El punto 1 (`Day` por ciclo / DataTable de eventos) pasa a: "✅ eventos que cruzan medianoche resueltos (v1.1). Continuar con prioridad de eclipses y spawn por eventos (docs `22` y `24`)".

---

## 2 — `23_SYSTEM_ItemSpawner_Investigacion.md`

### 2.1 REEMPLAZAR la línea "Estado" del encabezado
```
### Estado: ✅ Prototipo v1 (Ruta B, placeholder) implementado y confirmado en juego · 🟡 Sustituido por el plan de `24_SYSTEM_ResourceSpawnPoint_Plan.md` (spawn real ligado a eventos celestiales). El placeholder sigue activo hasta confirmar el reemplazo.
```

### 2.2 AGREGAR al inicio de "Acciones pendientes — próxima sesión"
> **Actualización (sesión eventos celestiales):** la decisión Ruta A vs Ruta B **se tomó para ítems y flora: Ruta B** (actor nuevo `BP_ResourceSpawnPoint`, que no hereda de `BP_SpawnPoint`). Motivo: `BP_SpawnPoint` tiene `Spawn Faction`, `Spawn Relationships`, `Overridden Behavior` y `Character Class`, parámetros exclusivos de IA. Para **enemigos** queda la idea (inferencia) de una clase hija de `BP_SpawnPoint` que llame a `ActivateSpawnPoint`/`DeactivateSpawnPoint` según el evento (Etapa 3 del doc 24).
> Las acciones 1 y 2 de la tabla (tipo de `Character Class`, export de `FindSpawnClass`/`TrySpawn`/`CheckRespawn`/`StartRespawnTimer`) **solo siguen bloqueando la Etapa 3 de enemigos**. La acción 3 (`Probability` de `BP_Item`) sigue pendiente, baja prioridad.

### 2.3 AGREGAR nueva sección "Hallazgos de la Etapa 0 del spawn por eventos" (resumen; detalle en `24`)
- El asset **no tiene una función central "spawn item"**. Spawnea ítems con `SpawnActor BP Item` directo (`BP_Item.DecayTick`, proyectiles `Arrow`/`Bolt`, `BP_Projectile_Item_Base.TrySpawnItem`, `BP_DebugMenuComponent.SpawnRandomItem`).
- `Item Handle` de `BP_Item` es público y tiene **Expose on Spawn** (confirmado por el usuario).
- El nodo `SpawnActor BP Item` expone `Item Handle`, `Amount`, `Charges`, `Durability`, `Decay`, `Simulate Physics`, `Can Decay`, `Infinity Item`, `Instigator`. `Simulate Physics` se desmarca **en el nodo**; no hace falta apagarlo después del spawn (esto mejora el fix por instancia del bug "ítem atraviesa el suelo" de este doc, para ítems spawneados en runtime).
- `BP_GameInstance.SpawnItems` fue inspeccionada y **descartada** (física activa, sin `Return Node` visible en la captura, recibe `STR_ItemData`).
- `BP_ContainerComponent.SpawnDropItem` / `SpawnDropItemAtLocation` no se usarán (requieren contenedor, probable física de drop — inferencia).
- `BP_Building_Plant_Base.CollectPlant` tiene lógica de reaparición por tiempo para plantas ("try to respawn it after time"): referencia para flora en la Etapa 2.

### 2.4 AGREGAR a "Deuda técnica / hallazgos registrados"
| Problema | Notas | Estado |
|---|---|---|
| `BP_Item_NightOnly` y `Night Only Items` quedarán obsoletos | Se reemplazan por `BP_ResourceSpawnPoint` cuando el MVP funcione. No tocar hasta confirmar | ⏳ Pendiente de migración |
| Interacción de un `BP_Item` spawneado en runtime con `Save ID` y el sistema de guardado | No documentado. Probar recogerlo (inventario sin errores) en la prueba 1C | 🟡 Pendiente |

---
*Actualización generada — sesión Light Paradox (cruce de medianoche + Etapa 1A + Etapa 0 del spawn)*
