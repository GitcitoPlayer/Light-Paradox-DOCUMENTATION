# 09 — Blueprint: BP_HUD_Game
### Blueprint: BP_HUD_Game
### Tipo: HUD Actor (hereda de HUD)
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspección directa — sesión Light Paradox (RuneAltar ScrollBox visibility)

---

## Contexto

`BP_HUD_Game` es el HUD Actor del juego. Es el punto de entrada para llegar al widget
`UI_HUD` y a todos los widgets hijos que viven dentro de él.

Es el Blueprint que se referencia al hacer la cadena:
`Get Player Controller → Get HUD → Cast To BP_HUD_Game`

---

## Variables confirmadas

| Variable | Tipo | Notas |
|---|---|---|
| `HUD` | User Widget Object Reference (UI_HUD) | Referencia al widget principal UI_HUD. Se asigna en la función Create HUD widget. |
| `HUDWidgetClass` | Widget Class | Clase del widget a instanciar. Apunta a UI_HUD. |

> **Nota:** `HUD` está declarada como `User Widget Object Reference` en la variable.
> Para acceder a miembros de `UI_HUD` desde fuera, se requiere un **Cast To UI_HUD**
> después de obtener esta variable.

---

## Funciones confirmadas

### Create HUD widget and add it to the viewport
**Propósito:** Instanciar UI_HUD y agregarlo al viewport al inicio del juego.

```
Create HUD →
  Is Valid (HUD) →
    Is Not Valid →
      Create User Widget Widget (Class: HUDWidgetClass, Owning Player: Get Owning Player) →
        Return Value →
      SET HUD = Return Value →
      Add to Viewport (Target: HUD)
    Is Valid → [termina, no crea duplicado]
```

> **Nota:** El nodo `Create HUD` (morado) al inicio del flujo es el evento o función
> que dispara esta lógica. No fue inspeccionado en detalle — puede ser un Custom Event
> o un Event BeginPlay override. Requiere verificación futura.

---

## Otras variables inferidas (no confirmadas)

| Variable | Tipo | Notas |
|---|---|---|
| `Death Menu` | Widget | Referida en el menú de variables — no inspeccionada |
| `Death Menu Widget Class` | Widget Class | Referida en el menú de variables — no inspeccionada |
| `Pause Menu` | Widget | Referida en el menú de variables — no inspeccionada |
| `Pause Menu Widget Class` | Widget Class | Referida en el menú de variables — no inspeccionada |

> Estas variables aparecieron en el buscador de nodos al hacer drag desde
> `As BP_HUD_Game`. Se documentan como existentes pero no fueron inspeccionadas.

---

## Rol en la cadena de visibilidad del RuneAltar

`BP_HUD_Game` es el punto intermedio obligatorio en la cadena de llamada desde
`BP_Building_Altar` hasta `UI_Character`:

```
BP_Building_Altar
  → Get Player Controller
  → Get HUD
  → Cast To BP_HUD_Game
  → Get HUD (variable)        ← vive aquí
  → Cast To UI_HUD
  → Get Character Information
  → Show Rune Slots / Hide Rune Slots
```

Sin el Cast To BP_HUD_Game no es posible acceder a la variable `HUD` tipada.

---

## Notas de arquitectura

- `BP_HUD_Game` no fue exportado como .txt en esta sesión.
  La documentación proviene de inspección directa y del buscador de nodos en Blueprint.
- Solo se documentaron los sistemas tocados durante la sesión de RuneAltar ScrollBox visibility.
- El resto de funciones y variables de este Blueprint no están documentadas aún.

---

*Archivo creado — sesión Light Paradox (RuneAltar ScrollBox visibility)*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
