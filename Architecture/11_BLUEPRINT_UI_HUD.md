# 10 — Blueprint: UI_HUD
### Widget: UI_HUD
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspección directa — sesión Light Paradox (RuneAltar ScrollBox visibility)

---

## Contexto

`UI_HUD` es el widget principal del juego. Contiene como hijos todos los widgets
de la interfaz, incluyendo `UI_Character`. Es instanciado y agregado al viewport
por `BP_HUD_Game` al inicio del juego.

Es el Blueprint donde, según `03_LIGHTPARADOX_PROJECT_RULES.md`, debe vivir
el gate `bSlotUpdateEnabled` en la clase hija `UI_HUD_LP`.

---

## Variables confirmadas

| Variable | Tipo | Notas |
|---|---|---|
| `CharacterInformation` | UI_Character | Referencia al widget hijo UI_Character. Expuesta automáticamente al activar **Is Variable** en el widget dentro del Hierarchy de UI_HUD. |

> **Cómo se genera:** En el panel Hierarchy de UI_HUD, el widget `UI_Character`
> tiene activado el checkbox **Is Variable**. Esto hace que Unreal exponga
> automáticamente la referencia como variable accesible desde Blueprint.

---

## Widgets hijos confirmados

| Widget | Tipo | Notas |
|---|---|---|
| `UI_Character` | UI_Character | Widget de atributos y equipment slots del personaje. Accesible vía variable `CharacterInformation`. |

> **Nota:** Otros widgets hijos de `UI_HUD` no fueron inspeccionados en esta sesión.

---

## Relación con la arquitectura del proyecto

Según `03_LIGHTPARADOX_PROJECT_RULES.md`:

| Clase base | Clase hija Light Paradox | Estado |
|---|---|---|
| `UI_HUD` (base ESRPGv5) | `UI_HUD_LP` | **Pendiente de crear** |

El gate `bSlotUpdateEnabled` y la función `SetSlotUpdateEnabled` deben implementarse
en `UI_HUD_LP`, no en `UI_HUD` directamente. Editar `UI_HUD` base es deuda técnica.

---

## Rol en la cadena de visibilidad del RuneAltar

`UI_HUD` es el contenedor que permite llegar a `CharacterInformation` (UI_Character)
desde fuera del widget:

```
Cast To BP_HUD_Game
  → Get HUD (variable de BP_HUD_Game, tipo User Widget)
  → Cast To UI_HUD
  → Get Character Information   ← vive aquí
  → Show Rune Slots / Hide Rune Slots
```

El Cast To UI_HUD es necesario porque la variable `HUD` en `BP_HUD_Game` está
declarada como `User Widget Object Reference` genérico, no como `UI_HUD` tipado.

---

## Notas de arquitectura

- `UI_HUD` no fue exportado como .txt en esta sesión.
  La documentación proviene de inspección directa durante la sesión de RuneAltar.
- Solo se documentaron los sistemas tocados en esta sesión.
- El resto de funciones, variables y widgets hijos de `UI_HUD` no están documentados aún.

---

## Deuda técnica registrada

| Problema | Regla violada | Estado |
|---|---|---|
| Gate bSlotUpdateEnabled no implementado | Rule 2.1 | Pendiente — requiere crear UI_HUD_LP |
| Modificaciones directas en base asset | Rule 4.1 | Pendiente — migrar a UI_HUD_LP |

---

*Archivo creado — sesión Light Paradox (RuneAltar ScrollBox visibility)*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
