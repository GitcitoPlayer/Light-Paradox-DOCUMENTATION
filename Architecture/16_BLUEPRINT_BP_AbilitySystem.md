# 16 -- Blueprint: BP_AbilitySystem
### Componente: BP_AbilitySystem
### Tipo: Actor Component
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspeccion directa -- sesion Light Paradox (Logica 6)
### Proyecto: Light Paradox - UE 5.4.4

---

## Contexto

`BP_AbilitySystem` es el componente que gestiona todos los Status Effects activos
del personaje. Vive dentro de `BP_Character_Player` como componente instanciado.

Es el **punto de entrada obligatorio** para aplicar cualquier Estado al jugador.
Ningun sistema externo debe aplicar efectos directamente — siempre a traves de
este componente.

---

## Variables confirmadas

| Variable | Tipo | Notas |
|---|---|---|
| `Active Status Effects` | Array de BP_StatusEffect_Base | Lista de efectos activos actualmente en el personaje |
| `Owner Pawn` | Pawn Object Reference | Referencia al pawn que posee este componente |

---

## Funciones confirmadas

### Load Status Effect

**Proposito:** Punto de entrada publico para aplicar un Estado al personaje.
Recibe un struct de save data, lee el DT, instancia el Blueprint del efecto y lo registra.

**Input:**
- `Save Data` (STR_SaveData_StatusEffect)

**Output:**
- `Success` (Boolean)

**Flujo:**
```
Load Status Effect (Save Data)
  → Break STR_SaveData_StatusEffect
      → Effect Handle
      → Instigator Is Owner
      → Stack
      → Time Remaining
  → Break DataTableRowHandle
      → Data Table
      → Row Name
  → Get Data Table Row (Data Table, Row Name)
      → Row Found →
          Break STR_StatusEffectInstance
            → Handle
            → Effect Class
          Get Actor Transform (self)
          SpawnActor
            Class: Effect Class
            Transform: resultado
            Owner: Get Owner
            Effect Handle: Handle
            Instigator: [del Break SaveData]
          → Return Value (instancia BP_StatusEffect_*)
          → Register Status Effect (Target: self, Status Effect: instancia)
          → Set Stack (Target: instancia, Stack: Stack)
          → Set Duration (Target: instancia, Duration: Time Remaining)
          → Return Node (Success: true)
      → Row Not Found →
          → Return Node (Success: false)
```

> **CRITICO:** El pin `Duration` de `Set Duration` recibe el valor `Time Remaining`
> del struct de entrada. Este valor debe ser igual a la `Duration` del DT para que
> el countdown de UI funcione correctamente.
> Pasar `Time Remaining = 0.0` produce countdown negativo en UI_StatusEffect.

---

### Add Active Status Effect

**Proposito:** Registra una instancia ya existente de BP_StatusEffect_* en el
array de efectos activos y la conecta a los dispatchers del sistema.

**Input:**
- `Status Effect` (BP_StatusEffect_Base Object Reference)

**Output:**
- `Success` (Boolean)

**Flujo:**
```
Add Active Status Effect (Status Effect)
  → ADDUNIQUE → Active Status Effects
  → Call On Status Effect Added (Status Effect)
  → Bind Event to On Status Effect Removed → Bind_StatusEffectRemoved(StatusEffect)
  → Bind Event to On Status Effect Updated → Bind_StatusEffectUpdated(StatusEffect)
  → Get Character Attributes (Status Effect) → Character Attributes
  → Branch (IS VALID INDEX [0] en Character Attributes)
      True  → Update Status Effects Attributes → Return Node (Success: true)
      False → Return Node (Success: false)
```

> **Nota:** Esta funcion es llamada internamente por `Register Status Effect`,
> no por sistemas externos. El punto de entrada externo es `Load Status Effect`.

---

## Dispatchers / Eventos confirmados en BP_Character_Player

Cuando `BP_AbilitySystem` dispara sus eventos internos, `BP_Character_Player`
los escucha y notifica al HUD:

```
On Status Effect Added (AbilitySystemComponent)
  → Get Controller (self)
  → Get Status Effect Data (Target: Status Effect) → Status Effect Data
  → Add Status Effect BPI (Target: BPI_Player, Status Effect Data)

On Status Effect Updated (AbilitySystemComponent)
  → Get Controller (self)
  → Get Status Effect Data (Target: Status Effect) → Status Effect Data
  → Update Status Effect BPI (Target: BPI_Player, Status Effect Data)

On Status Effect Removed (AbilitySystemComponent)
  → Get Controller (self)
  → Get Status Effect Handle (Target: Status Effect) → Status Effect Handle
  → Remove Status Effect BPI (Target: BPI_Player, Status Effect Handle)
```

---

## Como referenciar BP_AbilitySystem desde un trigger externo

```
Cast To BP_Character_Player (Other Actor)
  Cast Success →
    As BP_Character_Player → Get [nombre del componente AbilitySystem]
    → Ability System Component (output)
    → Load Status Effect (Target: Ability System Component, Save Data: ...)
```

> **Nota:** El nombre exacto del componente en el panel Components de
> BP_Character_Player no fue inspeccionado con detalle. Buscar en el panel
> Components de BP_Character_Player el componente de tipo BP_AbilitySystem.

---

## Deuda tecnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Nombre exacto del componente en BP_Character_Player no confirmado | Inferencia: "AbilitySystemComponent" o "AbilitySystem" | Pendiente verificacion |
| Funciones internas Register Status Effect, Set Stack, Set Duration sin documentar | Vistas en flujo de Load Status Effect pero no inspeccionadas individualmente | Pendiente |
| Ediciones en asset base | No se sabe si BP_AbilitySystem tiene clase hija en Light Paradox | Pendiente verificacion |

---

*Archivo creado -- sesion Light Paradox (Logica 6)*
*Project: Light Paradox - Base: EasySurvivalRPGv5 - UE 5.4.4*
