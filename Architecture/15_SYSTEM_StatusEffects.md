# 15 -- Sistema: Status Effects
### Sistema: Status Effects
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspeccion directa + CSV export + Reference Viewer -- sesion Light Paradox (Logica 6)
### Proyecto: Light Paradox - UE 5.4.4

---

## Contexto

El sistema de Status Effects de ESRPGv5 es un sistema de efectos por datos.
Cada efecto es una fila en `DT_StatusEffects`. La logica de cada tipo de efecto
vive en una clase Blueprint separada que hereda de una clase base de Status Effect.

El sistema ya soporta:
- Efectos positivos (buffs): comida, curacion, escudo magico
- Efectos negativos (debuffs): dano por tick, inmovilizacion total
- Tick de atributos por intervalo configurable
- Modificacion de atributos del personaje mientras el efecto esta activo
- UI de iconos con tooltip
- Stacking controlado por `MaxStack`

Este documento mapea el sistema existente antes de agregar los efectos
de Logica 6: **Veneno** y **Ralentizar**.

---

## Mapa del sistema

```
[Fuente del efecto]
  (enemigo, trampa, item, runa)
        |
        v
[BP_Character_Player]
  AddStatusEffect()
        |
        v
[BP_StatusEffect_* (instancia activa)]
  Hereda de clase base de Status Effect
  Lee datos de DT_StatusEffects
  Ejecuta logica del tipo de efecto
        |
        v
[BPI_HUD_Game]
  AddStatusEffect_BPI / UpdateStatusEffect_BPI / RemoveStatusEffect_BPI
        |
        v
[UI_HUD]
  AddStatusEffect / UpdateStatusEffect / RemoveStatusEffect
        |
        v
[UI_StatusEffect] (widget del icono en HUD)
  UI_StatusEffectToolTip (widget del tooltip al hacer hover)
```

---

## STR_StatusEffectInstance -- Row Struct de DT_StatusEffects

**Confirmado via inspeccion directa del struct.**

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
| `EffectAttributes` | Array de STR_Attribute | Atributos del efecto (tick interval, dano por tick, etc.) |
| `EffectAttributesMapped` | Map (Name -> STR_Attribute) | Atributos del efecto accesibles por clave string |
| `InstigatorDependencies` | Array de STR_AttributeD | Dependencias del instigador |
| `InstigatorDependenciesMapped` | Map (Name -> STR_AttributeD) | Igual, accesible por clave |
| `Options` | String | Opciones adicionales en texto libre |
| `CharacterAttributes` | Array de STR_Attribute | Atributos que se modifican en el personaje mientras el efecto esta activo |
| `CharacterAttributesMapped` | Map (Name -> STR_Attribute) | Igual, accesible por clave |
| `EffectDependencies` | Array de STR_AttributeD | Dependencias del efecto |
| `EffectDependenciesMapped` | Map (Name -> STR_AttributeD) | Igual, accesible por clave |
| `Handles` | Map (Name -> Data Table Row Handle) | Handles de otros efectos relacionados |

> **Nota:** `STR_StatusEffectData` existe como asset separado. No fue inspeccionado
> en esta sesion. Se documenta como pendiente.

---

## Gameplay Tags confirmados en DT_StatusEffects

### EffectAttributes (logica del efecto en tick)

| Tag | Uso |
|---|---|
| `EasyRPG.Attributes.StatusEffect.TickInterval` | Intervalo en segundos entre cada tick del efecto |
| `EasyRPG.Attributes.StatusEffect.Health.Max%PerSecond` | Porcentaje del MaxHealth restaurado por segundo |
| `EasyRPG.Attributes.StatusEffect.Hunger.Max%PerSecond` | Porcentaje del MaxHunger restaurado por segundo |
| `EasyRPG.Attributes.StatusEffect.Oxygen.ValuePerSecond` | Oxigeno restaurado por segundo (valor absoluto) |
| `EasyRPG.Attributes.StatusEffect.Damage.ValuePerSecond` | Dano aplicado por segundo (usado en Bleeding) |

### CharacterAttributes (modificadores activos mientras el efecto esta vivo)

| Tag | Uso |
|---|---|
| `EasyRPG.Attributes.DamageSystem.Damage.Melee` | Modifica dano melee del personaje |
| `EasyRPG.Attributes.DamageSystem.Resistance.Melee%` | Modifica resistencia melee % |
| `EasyRPG.Attributes.DamageSystem.Resistance.Overall%` | Modifica resistencia global % |
| `EasyRPG.Attributes.Base.MaxHealth` | Modifica MaxHealth |

---

## Blueprints de Status Effect -- clases confirmadas

Confirmados via Reference Viewer de DT_StatusEffects y CSV.

| Blueprint | Tipo de efecto | Efectos que lo usan |
|---|---|---|
| `BP_StatusEffect_ChangeState` | Modifica atributos via EffectAttributes + CharacterAttributes | Corn, Beet, Pumpkin, Cabbage, Healing |
| `BP_StatusEffect_TickDamage` | Aplica dano por tick usando EffectAttributesMapped["DamagePerSecond"] | Bleeding |
| `BP_StatusEffect_OverrideSpeed` | Sobreescribe la velocidad del personaje | Trapped |
| `BP_StatusEffect_MagicShield` | Logica especial de escudo magico | MagicShield |
| `BP_StatusEffect_UnderwaterBreathing` | Logica especial de respiracion submarina | UnderwaterBreathing |

> **Nota:** `BP_StatusEffect_ChangeState`, `BP_StatusEffect_TickDamage` y
> `BP_StatusEffect_OverrideSpeed` son las clases **base reutilizables**.
> `BP_StatusEffect_MagicShield` y `BP_StatusEffect_UnderwaterBreathing` son
> **clases preset** con logica especifica. No inspeccionadas en esta sesion.

---

## Filas existentes en DT_StatusEffects

| Row Name | EffectClass | IsPositive | Duration | MaxStack | Notas |
|---|---|---|---|---|---|
| `Effect_Corn` | BP_StatusEffect_ChangeState | True | 10s | 3 | Health 0.5%/s + Hunger 1.5%/s |
| `Effect_Beet` | BP_StatusEffect_ChangeState | True | 30s | 1 | Health+Hunger 0.33%/s + Melee+15 |
| `Effect_Pumpkin` | BP_StatusEffect_ChangeState | True | 60s | 1 | Health+Hunger 0.5%/s + Melee Resist 30% |
| `Effect_Cabbage` | BP_StatusEffect_ChangeState | True | 60s | 1 | Health+Hunger 0.15%/s + MaxHealth+25 |
| `Effect_UnderwaterBreathing` | BP_StatusEffect_UnderwaterBreathing | True | 60s | 1 | Oxygen +2.5/s |
| `Effect_Healing` | BP_StatusEffect_ChangeState | True | 5s | 1 | Health 5%/s |
| `Effect_MagicShield` | BP_StatusEffect_MagicShield | True | 30s | 1 | Overall Resist 50% |
| `Effect_Bleeding` | BP_StatusEffect_TickDamage | False | 10s | 1 | Dano 5/s via EffectAttributesMapped |
| `Effect_Trapped` | BP_StatusEffect_OverrideSpeed | False | 1.5s | 1 | Velocidad 0 (sin EffectAttributes -- hardcodeado en BP) |

---

## BPI Events en UI_HUD -- confirmados

**Confirmado via inspeccion directa del EventGraph de UI_HUD.**

Siguen el patron estandar de delegacion: BPI Event -> funcion interna homónima.

| BPI Event | Parametro de entrada | Funcion interna |
|---|---|---|
| `AddStatusEffect_BPI` | `Status Effect Data` (struct) | `Add Status Effect` |
| `RemoveStatusEffect_BPI` | `Status Effect Handle` | `Remove Status Effect` |
| `UpdateStatusEffect_BPI` | `Status Effect Data` (struct) | `Update Status Effect` |

> **Nota:** El tipo exacto del struct `Status Effect Data` no fue inspeccionado.
> Inferencia: probablemente es `STR_StatusEffectInstance` o un struct derivado.
> Requiere confirmacion al inspeccionar `Add Status Effect` en UI_HUD.

---

## Assets de UI confirmados

| Asset | Tipo | Rol |
|---|---|---|
| `UI_StatusEffect` | Widget Blueprint | Icono del efecto activo en el HUD |
| `UI_StatusEffectToolTip` | Widget Blueprint | Tooltip con nombre y descripcion al hacer hover |

> **Nota:** El contenido interno de ambos widgets no fue inspeccionado.
> Requieren inspeccion futura para documentar variables y funciones.

---

## Referenciadores de DT_StatusEffects

Confirmados via Reference Viewer.

| Asset | Tipo | Rol |
|---|---|---|
| `BP_Character_Player` | Blueprint | Aplica y gestiona efectos en el personaje jugador |
| `DT_Abilities` | DataTable | Referencia cruzada -- no inspeccionada |
| `BP_Building_Trap_Beartrap` | Blueprint | Aplica Effect_Trapped al personaje |
| `UI_StatusEffectToolTip` | Widget Blueprint | Lee datos del efecto para mostrar tooltip |
| `Overview_Main` | World | Pantalla principal / documentacion del asset |

---

## Analisis para Logica 6: Veneno y Ralentizar

### Efecto 1: Veneno

**Referencia Diablo:** Dano por tick que reduce HP progresivamente.
Puede tener stacking o intensificarse con multiples aplicaciones.

**Mapeo ESRPGv5:** Identico a `Effect_Bleeding`.
- **EffectClass:** `BP_StatusEffect_TickDamage`
- **Diferencia con Bleeding:** Solo valor de dano y duracion distintos.
- **No requiere Blueprint nuevo.** Solo una fila nueva en `DT_StatusEffects`.

**Inferencia sobre EffectAttributesMapped en Bleeding:**
La clave `"DamagePerSecond"` esta mapeada al tag `EasyRPG.Attributes.StatusEffect.Damage.ValuePerSecond`.
Veneno usaria el mismo patron con distinto valor.

**Decision de diseno pendiente:**
- Duracion de Veneno
- Dano por segundo de Veneno
- Stack maximo (1 o acumulable)
- Si el veneno puede tener dano escalado por stacks

---

### Efecto 2: Ralentizar

**Referencia Diablo:** Movimiento reducido a un porcentaje (no inmovilizacion total).

**Mapeo ESRPGv5:** Similar a `Effect_Trapped`, pero con velocidad reducida en lugar
de velocidad 0.

**Problema identificado:** `Effect_Trapped` usa `BP_StatusEffect_OverrideSpeed`
**sin EffectAttributes** -- la velocidad de 0 probablemente esta hardcodeada
dentro del Blueprint.

**Opciones segun lo documentado:**

| Opcion | Descripcion | Riesgo |
|---|---|---|
| A | Crear `BP_StatusEffect_Slow` hijo de `BP_StatusEffect_OverrideSpeed` con velocidad reducida configurable | Seguro -- no modifica base |
| B | Leer `Options` (String) del row en DT para configurar la velocidad dentro de `BP_StatusEffect_OverrideSpeed` | Requiere inspeccionar si el BP ya lee ese campo |
| C | Si `BP_StatusEffect_OverrideSpeed` ya recibe la velocidad como parametro desde el DT, solo agregar la fila con el valor correcto | Opcion ideal -- cero cambios de Blueprint |

**Prioridad de inspeccion:** Abrir `BP_StatusEffect_OverrideSpeed` y verificar si
lee algun attribute del DT para la velocidad, o si tiene la velocidad hardcodeada.
Esta inspeccion determina cual opcion usar.

---

## Pendientes criticos antes de implementar

| Pendiente | Por que es necesario |
|---|---|
| Inspeccionar `BP_StatusEffect_OverrideSpeed` | Determina si Ralentizar necesita Blueprint hijo o solo una fila nueva |
| Inspeccionar `BP_StatusEffect_TickDamage` | Confirmar que Veneno es solo una fila nueva en DT (inferencia actual) |
| Inspeccionar `Add Status Effect` en UI_HUD | Confirmar tipo exacto del struct que recibe |
| Inspeccionar `UI_StatusEffect` | Documentar variables y funciones del widget de icono |
| Inspeccionar `BP_Character_Player` -- funcion que aplica efectos | Identificar el punto de entrada para aplicar efectos desde enemigos |
| Determinar como aplica `BP_Building_Trap_Beartrap` el efecto | Ese patron sera el mismo para aplicar Veneno/Ralentizar desde enemigos |

---

## Deuda tecnica registrada

| Problema | Notas | Estado |
|---|---|---|
| `STR_StatusEffectData` sin documentar | Struct adicional encontrado -- rol desconocido | Pendiente inspeccion |
| `BP_StatusEffect_OverrideSpeed` sin inspeccionar | Critico para Ralentizar | Pendiente |
| `BP_StatusEffect_TickDamage` sin inspeccionar | Importante para confirmar Veneno | Pendiente |
| `UI_StatusEffect` sin inspeccionar | Widget de icono -- necesario para verificar UI correcta | Pendiente |
| Origen del efecto desde enemigos sin documentar | Como aplica Beartrap el efecto al personaje | Pendiente |

---

*Archivo creado -- sesion Light Paradox (Logica 6 -- mapeo inicial Status Effects)*
*Project: Light Paradox - Base: EasySurvivalRPGv5 - UE 5.4.4*
