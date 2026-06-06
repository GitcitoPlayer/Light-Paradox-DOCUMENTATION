# 11 — Blueprint: BP_Building_Altar
### Blueprint: BP_Building_Altar
### Tipo: Actor (Building)
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspección directa — sesión Light Paradox (RuneAltar ScrollBox visibility)

---

## Contexto

`BP_Building_Altar` es el actor del mundo que funciona como estación de crafteo
para el sistema de Rune Words del proyecto Light Paradox. Es un duplicado de uno
de los actores de crafting base de EasySurvivalRPGv5, extendido con lógica propia.

Está registrado en `DT_CraftingLists` como una entrada de crafting station,
lo que le permite definir qué ítems se craftean en él y habilitar el panel
de crafting de ESRPGv5 al interactuar.

---

## Componentes confirmados

| Componente | Tipo | Notas |
|---|---|---|
| `CraftingComponent` | BP_CraftingComponent | Gestiona el sistema de crafteo. Expone los dispatchers On Opened y On Closed. |
| `ContainerComponent` | BP_ContainerComponent | Contenedor de ítems del actor. Pasado a Open Crafting Place BPI al interactuar. |

---

## Variables confirmadas

| Variable | Tipo | Notas |
|---|---|---|
| `Opened` | Boolean | Indica si el panel de crafteo está abierto. Set via SET w/ Notify en On Opened y On Closed. |
| `Crafting On` | Boolean | Indica si hay un crafteo activo en curso. Set via SET w/ Notify en On Crafting Started y On Crafting Ended. |
| `Built` | Boolean | Indica si el actor fue construido. Usado en Begin Play para mostrar/ocultar meshes. |

---

## EventGraph — secciones documentadas

### Begin Play — Components initialization
```
Event BeginPlay →
  Parent: BeginPlay →
  Update Builded Meshes (Target: self) →
  Branch (Condition: Built) →
    True  → Switch Has Authority →
               Authority → Init Component (Target: BP_ContainerComponent) →
                           Init Component (Target: BP_CraftingComponent,
                             Container: ContainerComponent)
    False → [flujo de meshes ocultos]

UpdateBuildedMeshes (Custom Event) →
  Set Hidden in Game (Target: StaticMesh, New Hidden: Built, Propagate to Children: true) →
  Set Hidden in Game (Target: SkeletalMesh, New Hidden: NOT Built, Propagate to Children: true)
```

> **Nota:** El actor alterna entre StaticMesh y SkeletalMesh según el estado `Built`.
> Cuando está construido muestra el SkeletalMesh, cuando no está construido muestra
> el StaticMesh.

---

### Interactions
```
Event Try Interact BPI →
  Branch (Condition) →
    True  → Open Crafting Place BPI (Target: Player Controller,
               Crafting Component: CraftingComponent,
               Crafting Container: ContainerComponent,
               Only Crafting Queue: false,
               Disable Manual Crafting: false)
    False → [termina]

On Crafting Started (CraftingComponent) →
  SET Crafting On = True

On Crafting Ended (CraftingComponent) →
  SET Crafting On = False
```

> **Nota sobre Branch Condition en Try Interact BPI:** La condición está conectada
> al pin `Release` del evento de interacción. Inferencia: evalúa si el input fue
> liberado (no mantenido). No confirmado con inspección directa.

---

### RuneAltar ScrollBox visibility (lógica nueva — Light Paradox)
```
On Opened (CraftingComponent) →
  SET Opened = True →
  Cast To BP_HUD_Game (Object: Get Player Controller → Get HUD) →
    Then →
    Cast To UI_HUD (Object: Get HUD [variable de BP_HUD_Game] desde As BP_HUD_Game) →
      Then →
      Show Rune Slots (Target: Get Character Information desde As UI_HUD)

On Closed (CraftingComponent) →
  SET Opened = False →
  Cast To BP_HUD_Game (Object: Get Player Controller → Get HUD) →
    Then →
    Cast To UI_HUD (Object: Get HUD [variable de BP_HUD_Game] desde As BP_HUD_Game) →
      Then →
      Hide Rune Slots (Target: Get Character Information desde As UI_HUD)
```

> **Nota sobre On Opened / On Closed:** Estos dispatchers del `CraftingComponent`
> existían en el asset base pero no estaban siendo usados en `BP_Building_Altar`.
> Fueron copiados desde otro Blueprint de crafting de ESRPGv5 y conectados en
> esta sesión. Son los eventos correctos para detectar apertura y cierre del panel,
> a diferencia de `On Crafting Started / Ended` que se disparan al iniciar un crafteo.

---

## Cadena de llamada completa — visibilidad RuneAltar

```
BP_Building_Altar
  On Opened / On Closed (CraftingComponent)
        │
        ▼
  Get Player Controller (Index 0)
        │
        ▼
  Get HUD (función, Target: Player Controller)
        │
        ▼
  Cast To BP_HUD_Game
        │ Then
        ▼
  Get HUD (variable de BP_HUD_Game, desde As BP_HUD_Game)
        │
        ▼
  Cast To UI_HUD
        │ Then
        ▼
  Get Character Information (desde As UI_HUD)
        │
        ▼
  Show Rune Slots / Hide Rune Slots (en UI_Character)
```

---

## Registro en DT_CraftingLists

`BP_Building_Altar` tiene una entrada en `DT_CraftingLists` que define:
- Qué ítems pueden craftearse en este actor
- La configuración del panel de crafteo

> El contenido exacto de la fila en `DT_CraftingLists` no fue inspeccionado en
> esta sesión. Requiere documentación futura.

---

## Notas de arquitectura

- `BP_Building_Altar` es un duplicado de un actor de crafting base de ESRPGv5,
  no una clase hija. Esto es deuda técnica según Rule 4.1 de
  `03_LIGHTPARADOX_PROJECT_RULES.md`.
- La lógica de visibilidad del RuneAltar vive directamente en el actor del mundo,
  no en el HUD ni en el widget. Esta es la ruta más directa dado el sistema existente.
- `On Opened` y `On Closed` son los eventos correctos para detectar apertura/cierre
  del panel. `On Crafting Started` y `On Crafting Ended` son para crafteo activo —
  no usar para control de visibilidad.

---

## Deuda técnica registrada

| Problema | Regla violada | Estado |
|---|---|---|
| Duplicado del actor base, no clase hija | Rule 4.1 | Pendiente — evaluar migración a clase hija |

---

*Archivo creado — sesión Light Paradox (RuneAltar ScrollBox visibility)*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
