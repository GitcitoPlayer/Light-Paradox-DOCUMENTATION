# 15 -- Sistema: Status Effects (Estados)
### Sistema: Status Effects
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspeccion directa + CSV export + Reference Viewer + sesion Logica 6
### Proyecto: Light Paradox - UE 5.4.4

---

## Contexto

El sistema de Status Effects de ESRPGv5 — llamado **Estados** en Light Paradox —
es un sistema de efectos por datos. Cada efecto es una fila en `DT_StatusEffects`.
La logica de cada tipo de efecto vive en una clase Blueprint separada que hereda
de `BP_StatusEffect_Base`.

El sistema soporta:
- Efectos positivos (buffs): comida, curacion, escudo magico
- Efectos negativos (debuffs): dano por tick, inmovilizacion total
- Tick de atributos por intervalo configurable
- Modificacion de atributos del personaje mientras el efecto esta activo
- UI de iconos con tooltip y countdown
- Stacking controlado por `MaxStack`

---

## Mapa del sistema completo

```
[Trigger externo]
  (colision, enemigo, trampa, item, runa)
        |
        v
[BP_AbilitySystem]
  Load Status Effect(Save Data)
    - rompe STR_SaveData_StatusEffect → Effect Handle
    - Get Data Table Row(DT_StatusEffects) → STR_StatusEffectInstance
    - SpawnActor(EffectClass) → instancia BP_StatusEffect_*
    - Register Status Effect → Add Active Status Effect
        |
        v
[BP_StatusEffect_* (instancia activa)]
  BeginPlay → aplica logica del tipo de efecto
  (dano por tick / override speed / change state)
        |
        v
[BP_Character_Player]
  On Status Effect Added (AbilitySystemComponent)
    → Get Controller → Add Status Effect BPI (BPI_Player)
        |
        v
[UI_HUD]
  Add Status Effect → Update Widget (UI_StatusEffect)
    → Set Timer by Event → UpdateTimeRemaining (cada TickInterval)
        |
   (Duration termina)
        v
[BP_StatusEffect_*]
  Destroyed → sistema notifica
        |
        v
[BP_Character_Player]
  On Status Effect Removed (AbilitySystemComponent)
    → Get Controller → Remove Status Effect BPI
        |
        v
[UI_HUD]
  Remove Status Effect → quita icono del HUD
```

---

## STR_StatusEffectInstance -- Row Struct de DT_StatusEffects

Confirmado via inspeccion directa del struct y CSV export.

| Campo | Tipo | Notas |
|---|---|---|
| `Handle` | Data Table Row Handle | Referencia a la fila de este efecto en DT_StatusEffects |
| `EffectClass` | BP Status Effect (Class Reference) | Blueprint que implementa la logica del efecto |
| `EffectID` | Name | Identificador interno del efecto |
| `Name` | Text | Nombre visible del efecto |
| `Description` | Text | Descripcion visible del efecto |
| `Icon` | Texture2D | Icono visible en la UI |
| `IsPositiveEffect` | Boolean | True = buff, False = debuff |
| `Duration` | Float | Duracion total en segundos |
| `MaxStack` | Integer | Cuantas instancias pueden coexistir |
| `EffectAttributes` | Array de STR_Attribute | Atributos del efecto — tick interval, valores por segundo |
| `EffectAttributesMapped` | Map (Name -> STR_Attribute) | Atributos accesibles por clave string. Usado por BP_StatusEffect_TickDamage |
| `InstigatorDependencies` | Array de STR_AttributeD | Dependencias del instigador |
| `InstigatorDependenciesMapped` | Map (Name -> STR_AttributeD) | Igual, accesible por clave |
| `Options` | String | Opciones adicionales en texto libre |
| `CharacterAttributes` | Array de STR_Attribute | Atributos que se modifican en el personaje mientras el efecto esta activo |
| `CharacterAttributesMapped` | Map (Name -> STR_Attribute) | Igual, accesible por clave |
| `EffectDependencies` | Array de STR_AttributeD | Dependencias del efecto |
| `EffectDependenciesMapped` | Map (Name -> STR_AttributeD) | Igual, accesible por clave |
| `Handles` | Map (Name -> Data Table Row Handle) | Handles de otros efectos relacionados |

---

## STR_SaveData_StatusEffect -- struct de entrada a Load Status Effect

Confirmado via inspeccion directa de BP_AbilitySystem.

| Campo | Tipo | Notas |
|---|---|---|
| `Effect Handle` | Data Table Row Handle | Referencia a la fila en DT_StatusEffects |
| `Instigator Is Owner` | Boolean | Si el instigador es el owner del componente |
| `Stack` | Integer | Stack actual del efecto |
| `Time Remaining` | Float | **Tiempo restante en segundos.** Debe pasarse con el valor de Duration del efecto, no en 0. Si se pasa 0, el countdown en UI va a negativos. |

> **CRITICO:** `Time Remaining` debe inicializarse con el valor `Duration` del DT.
> Pasar `0.0` es el bug confirmado en sesion Logica 6 que produce countdown negativo en UI.
> Ver seccion "Bug confirmado y resuelto" mas abajo.

---

## Gameplay Tags confirmados en DT_StatusEffects

### EffectAttributes (array — logica del efecto en tick)

| Tag | Uso |
|---|---|
| `EasyRPG.Attributes.StatusEffect.TickInterval` | Intervalo en segundos entre cada tick del efecto |
| `EasyRPG.Attributes.StatusEffect.Health.Max%PerSecond` | Porcentaje del MaxHealth restaurado por segundo |
| `EasyRPG.Attributes.StatusEffect.Hunger.Max%PerSecond` | Porcentaje del MaxHunger restaurado por segundo |
| `EasyRPG.Attributes.StatusEffect.Oxygen.ValuePerSecond` | Oxigeno restaurado por segundo (valor absoluto) |

### EffectAttributesMapped (map — usado por BP_StatusEffect_TickDamage)

| Key (string) | Tag del Value | Uso |
|---|---|---|
| `DamagePerSecond` | `EasyRPG.Attributes.StatusEffect.Damage.ValuePerSecond` | Dano absoluto por segundo |
| `DamagePerTick` | `EasyRPG.Attributes.StatusEffect.Damage.Value` | Dano por tick (inferido) |
| `DamagePercentPerSecond` | `EasyRPG.Attributes.StatusEffect.Damage.Max%PerSecond` | Dano como % de MaxHealth por segundo |

> **Nota:** BP_StatusEffect_TickDamage lee los tres en cadena en su funcion
> `Update Variables`. Solo necesita uno para funcionar — los otros quedan en 0.
> Para Effect_Poison usar `DamagePerSecond`.

### CharacterAttributes (modificadores activos mientras el efecto esta vivo)

| Tag | Uso |
|---|---|
| `EasyRPG.Attributes.DamageSystem.Damage.Melee` | Modifica dano melee del personaje |
| `EasyRPG.Attributes.DamageSystem.Resistance.Melee%` | Modifica resistencia melee % |
| `EasyRPG.Attributes.DamageSystem.Resistance.Overall%` | Modifica resistencia global % |
| `EasyRPG.Attributes.Base.MaxHealth` | Modifica MaxHealth |

---

## Blueprints de Status Effect -- clases confirmadas

| Blueprint | Tipo de efecto | Efectos que lo usan |
|---|---|---|
| `BP_StatusEffect_ChangeState` | Modifica atributos via EffectAttributes + CharacterAttributes | Corn, Beet, Pumpkin, Cabbage, Healing |
| `BP_StatusEffect_TickDamage` | Aplica dano por tick usando EffectAttributesMapped | Bleeding, **Effect_Poison** |
| `BP_StatusEffect_OverrideSpeed` | Sobreescribe la velocidad del personaje via Override Walk Speed BPI | Trapped |
| `BP_StatusEffect_MagicShield` | Logica especial de escudo magico | MagicShield |
| `BP_StatusEffect_UnderwaterBreathing` | Logica especial de respiracion submarina | UnderwaterBreathing |

---

## Filas en DT_StatusEffects

| Row Name | EffectClass | IsPositive | Duration | MaxStack | Notas |
|---|---|---|---|---|---|
| `Effect_Corn` | BP_StatusEffect_ChangeState | True | 10s | 3 | Health 0.5%/s + Hunger 1.5%/s |
| `Effect_Beet` | BP_StatusEffect_ChangeState | True | 30s | 1 | Health+Hunger 0.33%/s + Melee+15 |
| `Effect_Pumpkin` | BP_StatusEffect_ChangeState | True | 60s | 1 | Health+Hunger 0.5%/s + Melee Resist 30% |
| `Effect_Cabbage` | BP_StatusEffect_ChangeState | True | 60s | 1 | Health+Hunger 0.15%/s + MaxHealth+25 |
| `Effect_UnderwaterBreathing` | BP_StatusEffect_UnderwaterBreathing | True | 60s | 1 | Oxygen +2.5/s |
| `Effect_Healing` | BP_StatusEffect_ChangeState | True | 5s | 1 | Health 5%/s |
| `Effect_MagicShield` | BP_StatusEffect_MagicShield | True | 30s | 1 | Overall Resist 50% |
| `Effect_Bleeding` | BP_StatusEffect_TickDamage | False | 10s | 1 | DamagePerSecond = 5.0 via EffectAttributesMapped |
| `Effect_Trapped` | BP_StatusEffect_OverrideSpeed | False | 1.5s | 1 | Velocidad 0 — sin EffectAttributes, hardcodeado en BP |
| `Effect_Poison` | BP_StatusEffect_TickDamage | False | **TBD** | **TBD** | DamagePerSecond = 2.0 — prototipo funcional Logica 6 |

> **Nota Effect_Poison:** Valores de Duration y MaxStack pendientes de decision del cliente.
> El prototipo usa Duration hardcodeada en BP_PoisonTrigger hasta implementar lectura del DT.

---

## BPI Events en UI_HUD -- confirmados

| BPI Event | Parametro | Funcion interna |
|---|---|---|
| `AddStatusEffect_BPI` | `Status Effect Data` (struct) | `Add Status Effect` |
| `RemoveStatusEffect_BPI` | `Status Effect Handle` | `Remove Status Effect` |
| `UpdateStatusEffect_BPI` | `Status Effect Data` (struct) | `Update Status Effect` |

---

## UI_StatusEffect -- comportamiento confirmado

`UI_StatusEffect` recibe `Effect Data` (STR_StatusEffectData) y arranca un timer
con `Set Timer by Event` usando `UpdateTimeRemaining` como evento repetido.

### Update Time Remaining (funcion interna)
```
Time Remaining = Effect Data.Time Remaining - Tick Interval
Set members in STR_StatusEffectData (Struct Ref: Effect Data, Time Remaining: resultado)
```

El widget **no calcula tiempo desde GameTime**. Hace countdown restando
`Tick Interval` de `Time Remaining` en cada tick del timer.

> **Implicacion critica:** Si `Time Remaining` llega al widget como `0.0`,
> el countdown comienza en 0 y va a negativos. Ver bug confirmado abajo.

---

## Bug confirmado y resuelto -- countdown negativo en UI

**Sesion:** Logica 6
**Sintoma:** El contador del icono de Estado en HUD comenzaba en 0 e iba a numeros negativos crecientes.
**Causa:** En `BP_PoisonTrigger`, el nodo `Make STR_SaveData_StatusEffect` tenia `Time Remaining = 0.0`.
**Fix aplicado (prototipo):** Hardcodear `Time Remaining` con el valor de `Duration` del efecto.
**Fix pendiente (produccion):** Leer `Duration` del DT via `Get Data Table Row` y pasarlo al pin `Time Remaining`. Ver seccion "Pendientes criticos".

---

## Patron de trigger -- confirmado y documentado

Cualquier actor que quiera aplicar un Estado al jugador sigue este patron minimo:

```
On Component Begin Overlap (o cualquier trigger)
  → Cast To BP_Character_Player (Other Actor)
      Cast Success →
        Get [AbilitySystemComponent] (de As BP_Character_Player)
        Make DataTableRowHandle
          Data Table: DT_StatusEffects
          Row Name: [nombre de la fila del efecto]
        Make STR_SaveData_StatusEffect
          Effect Handle: resultado del DataTableRowHandle
          Instigator Is Owner: False
          Stack: 1
          Time Remaining: [Duration del efecto]   ← CRITICO
        Load Status Effect (Target: AbilitySystemComponent, Save Data: resultado)
```

**Implementacion de referencia:** `BP_PoisonTrigger` — ver `19_BLUEPRINT_BP_PoisonTrigger.md`.

---

## Referenciadores de DT_StatusEffects

| Asset | Tipo | Rol |
|---|---|---|
| `BP_Character_Player` | Blueprint | Gestiona eventos On Status Effect Added/Updated/Removed |
| `BP_AbilitySystem` | Blueprint (componente) | Punto de entrada: Load Status Effect, Add Active Status Effect |
| `DT_Abilities` | DataTable | Referencia cruzada — no inspeccionada |
| `BP_Building_Trap_Beartrap` | Blueprint | Aplica Effect_Trapped via BP_Building_Trap_Base → Try Activate Trap |
| `UI_StatusEffectToolTip` | Widget Blueprint | Lee datos del efecto para mostrar tooltip |
| `UI_StatusEffect` | Widget Blueprint | Icono del efecto activo + countdown en HUD |
| `Overview_Main` | World | Pantalla principal / documentacion del asset |

---

## Pendientes criticos antes de produccion

| Pendiente | Por que es necesario |
|---|---|
| Fix Time Remaining — leer Duration del DT | Eliminar hardcode en BP_PoisonTrigger. Ver patron en seccion de trigger |
| Definir valores finales de Effect_Poison | Duration y MaxStack pendientes de decision del cliente |
| Implementar Effect_Slow | Requiere BP_StatusEffect_Slow (hijo de BP_StatusEffect_OverrideSpeed) + fila en DT. Ver 18_BLUEPRINT_BP_StatusEffect_OverrideSpeed.md |
| Inspeccionar UI_StatusEffectToolTip | Documentar variables y funciones del widget de tooltip |
| Inspeccionar BP_Building_Trap_Beartrap completo | Como patron de aplicacion desde trampa fisica |
| STR_StatusEffectData | Struct adicional encontrado — rol desconocido. Pendiente inspeccion |

---

## Nota para Logica 5 -- Estados y persistencia de runas

> **NOTA DE ARQUITECTURA PARA LOGICA 5**
>
> El sistema de Estados usa `STR_SaveData_StatusEffect` para serializar el estado
> activo de un efecto: `Effect Handle + Stack + Time Remaining`.
> Este patron de "guardar estado de un sistema activo en un struct serializable"
> es directamente relevante para la pregunta de Logica 5:
> **"¿Puede cada herramienta o cosmetico guardar su propia configuracion de runas?"**
>
> Si el sistema de runas sigue un patron similar — un struct con los handles de
> las runas asignadas + su estado — podria persistirse por item de la misma forma
> que los efectos activos se guardan en Save Data.
>
> Esto no es una decision tomada. Es arquitectura observable que debe considerarse
> al disenar la persistencia de runas en Logica 5.

---

*Archivo actualizado -- sesion Light Paradox (Logica 6 -- sistema completo mapeado)*
*Cambios: flujo completo documentado, STR_SaveData_StatusEffect documentado, bug countdown resuelto, patron de trigger confirmado, nota para Logica 5 agregada*
*Project: Light Paradox - Base: EasySurvivalRPGv5 - UE 5.4.4*
