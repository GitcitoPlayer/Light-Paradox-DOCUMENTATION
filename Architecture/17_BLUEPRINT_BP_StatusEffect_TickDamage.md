# 17 -- Blueprint: BP_StatusEffect_TickDamage
### Blueprint: BP_StatusEffect_TickDamage
### Tipo: Actor (hereda de BP_StatusEffect_Base)
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspeccion directa -- sesion Light Paradox (Logica 6)
### Proyecto: Light Paradox - UE 5.4.4

---

## Contexto

`BP_StatusEffect_TickDamage` es la clase de efecto que aplica dano al personaje
por tick durante su duracion. Es reutilizable por datos — la cantidad de dano
se configura en `DT_StatusEffects`, no en el Blueprint.

Efectos que lo usan: `Effect_Bleeding`, `Effect_Poison`.

---

## Funciones confirmadas

### Tick Actions

**Proposito:** Ejecutada cada tick del efecto. Verifica si el owner sigue vivo
y aplica dano o destruye el efecto.

```
Tick Actions (Delta Second)
  → Get Owner (self) → Return Value
  → Is Alive BPI (Target: BPI_Character, Target: Return Value)
      → Result (Boolean)
  → Branch (Result)
      True  → Deal Damage (Target: self, Delta Second)
                → Return Node (Success: true)
      False → Destroy Actor (Target: self)
                → Return Node
```

### Update Variables

**Proposito:** Lee `EffectAttributesMapped` del DT y extrae los tres posibles
valores de dano. Solo uno tendra valor — los otros quedan en 0.

```
Update Variables
  → Parent: Update Variables → Success
  → Break STR_StatusEffectInstance
      → Effect Attributes (array)
      → Effect Attributes Mapped (map)
  → Branch (IS VALID INDEX [0] en Effect Attributes)
      True  → SET Local Attributes
  → Get Attribute Value (Attributes: Local Attributes,
      Attribute: EasyRPG.Attributes.StatusEffect.Damage.Value)
      → Value → SET Damage Per Tick
  → Branch (Success)
      True  → continua
  → Get Attribute Value (Attributes: Local Attributes,
      Attribute: EasyRPG.Attributes.StatusEffect.Damage.ValuePerSecond)
      → Value → SET Damage Per Second
  → Branch (Success)
      True  → continua
  → Get Attribute Value (Attributes: Local Attributes,
      Attribute: EasyRPG.Attributes.StatusEffect.Damage.Max%PerSecond)
      → Value → SET Damage Percent Per Second
  → Return Node (Success: true)
```

---

## Variables de instancia inferidas

| Variable | Tipo | Notas |
|---|---|---|
| `Damage Per Tick` | Float | Dano por tick absoluto |
| `Damage Per Second` | Float | Dano por segundo absoluto |
| `Damage Percent Per Second` | Float | Dano como % de MaxHealth por segundo |

---

## Como configurar un efecto que use esta clase

En `DT_StatusEffects`, en la columna `EffectAttributesMapped`:

| Key (string) | Gameplay Tag | Value |
|---|---|---|
| `DamagePerSecond` | `EasyRPG.Attributes.StatusEffect.Damage.ValuePerSecond` | valor numerico de dano/s |

Solo se necesita una entrada. Las otras claves (`DamagePerTick`, `DamagePercentPerSecond`)
pueden dejarse vacias si no se usan.

**Ejemplo Effect_Poison:**
- Key: `DamagePerSecond`
- Tag: `EasyRPG.Attributes.StatusEffect.Damage.ValuePerSecond`
- Value: `2.0`

---

## Deuda tecnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Funcion Deal Damage no inspeccionada | Visto en flujo de Tick Actions pero no su implementacion interna | Pendiente |
| Edicion en asset base | No se sabe si tiene clase hija en Light Paradox | Pendiente verificacion |

---

*Archivo creado -- sesion Light Paradox (Logica 6)*
*Project: Light Paradox - Base: EasySurvivalRPGv5 - UE 5.4.4*
