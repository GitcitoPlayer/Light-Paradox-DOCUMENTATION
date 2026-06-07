# 12 — Blueprint: BP_CraftingComponent
### Componente: BP_CraftingComponent
### Tipo: Actor Component
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspección directa — sesión Light Paradox (Lógica 1 — Cooldown RuneAssign)

---

## Contexto

`BP_CraftingComponent` gestiona el sistema de crafteo del asset base.
Se instancia dentro de `BP_Building_Altar` como componente `CraftingComponent`.
Expone los dispatchers `On Opened` y `On Closed` que fueron usados en la sesión
de RuneAltar ScrollBox visibility para mostrar/ocultar los Rune Box.

---

## Variables relevantes confirmadas

| Variable | Tipo | Notas |
|---|---|---|
| `Blueprints Queue` | Array de STR_QueueBlueprint | Cola de crafteos activos. Cada elemento representa un ítem en proceso. |
| `Crafting Enabled` | Boolean | Gate del sistema de crafteo. Controla si `Crafting Tick` ejecuta. |
| `Crafting Tick Time` | Float | Intervalo del tick de crafteo. Tiempo que se resta a `Time Remaining` por tick. |
| `Multicraft Count` | Integer | Cantidad de crafteos simultáneos permitidos. |

---

## Structs relevantes confirmados

### STR_QueueBlueprint

| Campo | Tipo | Notas |
|---|---|---|
| `Blueprint` | STR_Blueprint | Datos del ítem a craftear |
| `Amount Remaining` | Integer | Cantidad restante por craftear |
| `Time Remaining` | Float | Tiempo restante en segundos para completar el crafteo actual |

### STR_Blueprint (campos confirmados desde UI_QueueBlueprint)

| Campo | Tipo | Notas |
|---|---|---|
| `Icon` | Texture2D | Icono del ítem a craftear |
| `Required Time` | Float | Tiempo total requerido para craftear |

---

## Funciones / Eventos relevantes confirmados

### Crafting Tick (Custom Event o Función)
**Tooltip:** "Crafting tick. Updates crafting blueprints time."

**Flujo confirmado:**

```
Crafting Tick →
  Branch (Condition: Crafting Enabled)
    False → [termina]
    True  →
      SET Local Length = Length(Blueprints Queue) - 1
      For Loop (First Index: 0, Last Index: Local Length)
        Loop Body →
          SET Local Index = Index - Multicraft Count
          Branch (Local Index < ?)
            → GET Blueprints Queue[Local Index]
              → Break STR_QueueBlueprint
                  → Blueprint
                  → Amount Remaining
                  → Time Remaining
              → Break STR_Blueprint
                  → Required Time
              → Time Remaining - Crafting Tick Time
              → Branch (Time Remaining > 0.0)
                  True →
                    Set members in STR_QueueBlueprint (Time Remaining actualizado)
                    → Update Queue Blueprint (Target: self, Index in Queue: Local Index,
                        Queue Blueprint: struct actualizado)
                  False →
                    Branch (Amount Remaining > 0)
                      True →
                        Set members in STR_QueueBlueprint (Amount Remaining - 1,
                          Time Remaining reseteado a Required Time)
                        → Update Queue Blueprint
                      False →
                        Remove Blueprint from Queue (Index in Queue: Local Index)
                        → Complete Crafting (Blueprint, Amount: 1)
```

> **Nota:** El flujo exacto del loop tiene lógica de Multicraft que no fue completamente
> documentada. Lo relevante para Lógica 1 es que el timer vive en `Time Remaining`
> del struct y se actualiza por tick desde aquí.

### Update Queue Blueprint (Función en BP_CraftingComponent)
Llama `UpdateCraftingQueueBlueprint` en `UI_CraftingQueue` vía la cadena HUD.
Pasa `Index in Queue` y el `STR_QueueBlueprint` actualizado al widget correspondiente.

### Remove Blueprint from Queue
Elimina un elemento del array `Blueprints Queue` por índice.

### Complete Crafting
Ejecuta la lógica de entrega del ítem crafteado al inventario del jugador.

---

## Dispatchers confirmados

| Dispatcher | Cuándo se dispara | Usado en |
|---|---|---|
| `On Opened` | Al abrir el panel de crafteo | BP_Building_Altar → ShowRuneSlots |
| `On Closed` | Al cerrar el panel de crafteo | BP_Building_Altar → HideRuneSlots |
| `On Crafting Started` | Al iniciar un crafteo activo | BP_Building_Altar → SET Crafting On = True |
| `On Crafting Ended` | Al terminar un crafteo activo | BP_Building_Altar → SET Crafting On = False |

---

## Rol en Lógica 1 — Cooldown RuneAssign

**Decisión tomada en sesión:** NO reutilizar `BP_CraftingComponent` para el cooldown
de asignación de runas. El sistema está acoplado a `STR_QueueBlueprint` y modificarlo
violaría Rule 4.1 (no editar assets base).

**Solución planificada:** Crear un timer propio en `UI_HUD` o `UI_Character` con
`Set Timer by Event`. Ver `14_WIDGET_UI_RuneAssignQueue.md` para la implementación.

---

## Notas de arquitectura

- `BP_CraftingComponent` es un asset base de ESRPGv5. No debe modificarse directamente
  según Rule 4.1 de `03_LIGHTPARADOX_PROJECT_RULES.md`.
- El timer de crafteo corre en el componente (Actor Component), no en el widget.
  El widget solo muestra el estado que el componente le envía.
- `Crafting Tick` es el único lugar donde `Time Remaining` se modifica.

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Flujo completo de Multicraft no documentado | La lógica de slots simultáneos dentro del For Loop no fue completamente inspeccionada | Pendiente |
| Funciones de inicio de crafteo no documentadas | Cómo se agrega un ítem a Blueprints Queue no fue inspeccionado | Pendiente |

---

*Archivo creado — sesión Light Paradox (Lógica 1 — análisis de factibilidad Cooldown)*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
