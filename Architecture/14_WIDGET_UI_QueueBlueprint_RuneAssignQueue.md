# 14 — Widget: UI_QueueBlueprint + Plan UI_RuneAssignQueue
### Widget base: UI_QueueBlueprint
### Widget nuevo planificado: UI_RuneAssignQueue
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspección directa — sesión Light Paradox (Lógica 1 — Cooldown RuneAssign)

---

## UI_QueueBlueprint — Documentación base

### Contexto

`UI_QueueBlueprint` es el widget visual de cada ítem en la cola de crafteo.
Se instancia dinámicamente dentro de `UI_CraftingQueue` por cada ítem en queue.
Muestra: icono del ítem, tiempo restante, cantidad, y botón de cancelar.

### Hierarchy confirmado

```
[UI_QueueBlueprint]
  └── BlueprintSizeBox
        └── [Overlay]
              ├── Icon (Image)
              ├── [Size Box]
              │     └── BtnCancel (Button)
              │           └── [Overlay]
              │                 ├── [BtnCancelBackground] (Image)
              │                 └── BtnCancelIcon (Image)
              └── [Border]
                    └── [Horizontal Box]
                          ├── Time (Text) — ej: "10 sec"
                          └── Amount (Text) — ej: "x2"
```

### Variables confirmadas

| Variable | Tipo | Notas |
|---|---|---|
| `Queue Blueprint` | STR_QueueBlueprint | Struct con datos del crafteo. Se setea via Update Blueprint. |
| `Size` | Vector2D | Tamaño del widget. Pasado al crear desde UI_CraftingQueue (100x100). |

### EventGraph confirmado

#### Event Construct
```
Event Construct →
  Break STR_QueueBlueprint (Queue Blueprint)
    → Blueprint →
        Break STR_Blueprint
          → Icon (Texture2D)
  Break Vector2D (Size)
    → X → Set Width Override (Target: BlueprintSizeBox)
    → Y → Set Height Override (Target: BlueprintSizeBox)
  Make Brush from Texture (Icon, 256x256)
    → Set Brush (Target: Icon widget)
```

> **Nota:** Solo el icono se asigna en Construct. `Time` y `Amount` se actualizan
> externamente desde `BP_CraftingComponent` vía `UpdateCraftingQueueBlueprint`.

#### Cancel Button Events
```
On Clicked (BtnCancel) →
  Get Owning Player →
  Get Parent → Get Child Index (Content: Self) → Index in Queue
  Cancel Crafting BPI (Target: Player, Index in Queue) →
  Remove from Parent (Self) →
  Call On Removed (Target: Self, Crafting Blueprint: Queue Blueprint)

On Hovered (BtnCancel) →
  Set Brush from Texture (Target: BtnCancelIcon, Texture: T_UI_Button_0?) [hover state]

On Unhovered (BtnCancel) →
  Set Brush from Texture (Target: BtnCancelIcon, Texture: T_UI_Button_0?) [normal state]
```

### Update Blueprint (Función)
```
Entry (Queue Blueprint) →
  SET self.Queue Blueprint = Queue Blueprint
```

> Solo guarda el struct. La actualización visual del tiempo ocurre externamente.
> Inferencia: `Time` y `Amount` se actualizan vía binding o desde fuera — no confirmado.

---

## UI_RuneAssignQueue — Plan de implementación (Lógica 1)

### Decisión

No reutilizar `UI_QueueBlueprint` — está acoplado a `STR_QueueBlueprint` y a
`Cancel Crafting BPI`, ambos del sistema de crafteo de ESRPGv5.

Duplicar como `UI_RuneAssignQueue` con misma estructura visual pero lógica propia.

### Estructura visual planificada

Idéntica a `UI_QueueBlueprint`:
- `Icon` — icono de la runa siendo asignada
- `Time` — countdown en segundos
- `BtnCancel` — cancela la asignación y devuelve la runa al inventario

### Diferencias respecto a UI_QueueBlueprint

| Aspecto | UI_QueueBlueprint | UI_RuneAssignQueue |
|---|---|---|
| Struct de datos | STR_QueueBlueprint | Datos propios (Texture2D + Float) |
| Cancel action | Cancel Crafting BPI | Devolver runa al inventario |
| Timer owner | BP_CraftingComponent | Timer propio en UI_HUD o UI_Character |
| Índice en queue | Get Child Index desde panel | Variable propia |

### Inputs planificados para inicialización

- `RuneIcon` (Texture2D) — icono de la runa
- `AssignDuration` (Float) — duración del cooldown en segundos
- `TargetContainer` (BP_ContainerComponent) — contenedor destino
- `TargetSlot` (Integer) — slot destino de la runa
- `SourceContainer` (BP_ContainerComponent) — contenedor origen (inventario)
- `SourceSlot` (Integer) — slot origen de la runa

### Punto de entrada confirmado

El flujo de asignación de runas pasa por `OnDrop` en `UI_ItemSlot` →
`TryMoveItemToContainerSlot_BPI`. Ahí se intercepta para:
1. No ejecutar el move inmediatamente
2. Instanciar `UI_RuneAssignQueue` con los datos del drag
3. Iniciar el timer
4. Al completar → ejecutar el move
5. Al cancelar → no ejecutar el move, runa permanece en origen

### Timer planificado

```
Set Timer by Event (Duration: AssignDuration, Looping: false)
  → Event: OnAssignComplete
      → TryMoveItemToContainerSlot (ejecuta el move real)
      → Remove UI_RuneAssignQueue from Parent
```

> **Nota:** El timer vivirá en `UI_Character` o `UI_HUD` — no en el widget mismo.
> Pendiente de confirmar ubicación exacta al implementar.

---

## Estado de implementación

| Componente | Estado |
|---|---|
| Análisis de factibilidad | ✅ Completo |
| Decisión de arquitectura (duplicar vs reutilizar) | ✅ Tomada — duplicar |
| Creación de UI_RuneAssignQueue | ⏳ Pendiente — próxima sesión |
| Interceptar OnDrop en UI_ItemSlot | ⏳ Pendiente |
| Timer implementation | ⏳ Pendiente |
| Cancel logic | ⏳ Pendiente |

---

## Próxima sesión — Punto de entrada

Retomar desde la creación de `UI_RuneAssignQueue`:
1. Duplicar `UI_QueueBlueprint` → renombrar `UI_RuneAssignQueue`
2. Limpiar lógica interna del original
3. Implementar Event Construct con inputs propios
4. Implementar timer con `Set Timer by Event`
5. Interceptar `OnDrop` en `UI_ItemSlot` para runas

**Pregunta pendiente antes de implementar:**
¿Cómo distinguir en `OnDrop` si el ítem siendo dropeado es una runa
(para activar el cooldown) vs un cosmético u otro ítem (flujo normal)?
Probablemente via Gameplay Tag del ítem — confirmar en próxima sesión.

---

*Archivo creado — sesión Light Paradox (Lógica 1 — análisis completo, implementación pendiente)*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
