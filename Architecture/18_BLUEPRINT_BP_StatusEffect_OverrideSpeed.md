# 18 -- Blueprint: BP_StatusEffect_OverrideSpeed
### Blueprint: BP_StatusEffect_OverrideSpeed
### Tipo: Actor (hereda de BP_StatusEffect_Base)
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspeccion directa -- sesion Light Paradox (Logica 6)
### Proyecto: Light Paradox - UE 5.4.4

---

## Contexto

`BP_StatusEffect_OverrideSpeed` es la clase de efecto que sobreescribe la
velocidad de movimiento del personaje. Al activarse establece una velocidad
fija. Al destruirse restaura la velocidad normal.

Efectos que lo usan: `Effect_Trapped` (velocidad 0 — inmovilizacion total).
Efectos pendientes: `Effect_Slow` (velocidad reducida — pendiente Logica 6).

---

## Variables de instancia confirmadas

| Variable | Tipo | Default Value | Notas |
|---|---|---|---|
| `Overrided Walk Speed` | Float | `0.0` | Velocidad a la que se fija el personaje al activarse |
| `Overrided Walk Interp Speed` | Float | `0.0` | Velocidad de interpolacion al cambiar |

> **Nota critica:** Las variables existen y son modificables. El valor `0.0`
> por defecto produce inmovilizacion total (comportamiento de Effect_Trapped).
> Para Effect_Slow se necesita una clase hija que cambie estos valores.
> Las variables NO se leen del DT — viven en el Blueprint.

---

## EventGraph confirmado

### Begin Play -- aplica override de velocidad

```
Event BeginPlay
  → Parent: BeginPlay
  → Get Owner (self) → Return Value
  → Override Walk Speed BPI (Target: BPI_Character)
      Target: Return Value
      Should Override Walk Speed: True (checkbox marcado)
      Overrided Walk Speed: GET Overrided Walk Speed
      Overrided Walk Interp Speed: GET Overrided Walk Interp Speed
```

### Event Destroyed -- restaura velocidad

```
Event Destroyed
  → Parent: Destroyed
  → Get Owner (self) → Return Value
  → Override Walk Speed BPI (Target: BPI_Character)
      Target: Return Value
      Should Override Walk Speed: False (checkbox vacio)
      Overrided Walk Speed: 0.0 (literal)
      Overrided Walk Interp Speed: 0.0 (literal)
```

---

## Plan para Effect_Slow -- pendiente Logica 6

Para implementar Ralentizar sin modificar el asset base:

### Paso 1 -- Crear BP_StatusEffect_Slow
1. Content Browser → clic derecho sobre `BP_StatusEffect_OverrideSpeed`
2. **Create Child Blueprint Class**
3. Nombre: `BP_StatusEffect_Slow`

### Paso 2 -- Cambiar Default Values
1. Abre `BP_StatusEffect_Slow`
2. Panel **My Blueprint → Variables**
3. Selecciona `Overrided Walk Speed`
4. En **Details → Default Value**: cambiar a la velocidad reducida deseada
   (ejemplo: `200.0` si la velocidad normal es `400.0` — 50%)
5. `Overrided Walk Interp Speed`: ajustar segun necesidad

### Paso 3 -- Agregar fila Effect_Slow en DT_StatusEffects
| Campo | Valor |
|---|---|
| `EffectClass` | `BP_StatusEffect_Slow` |
| `IsPositiveEffect` | False |
| `Duration` | TBD (decision del cliente) |
| `MaxStack` | 1 |
| `Name` | Slow |
| `EffectAttributes` | vacio |
| `EffectAttributesMapped` | vacio |

> **Nota:** No requiere EffectAttributes porque la velocidad vive en el BP,
> igual que Effect_Trapped. Esta es la misma arquitectura del asset base.

### Paso 4 -- Trigger de prueba
Mismo patron que `BP_PoisonTrigger` con Row Name `Effect_Slow`.
Ver `19_BLUEPRINT_BP_PoisonTrigger.md` para el patron completo.

---

## Deuda tecnica registrada

| Problema | Notas | Estado |
|---|---|---|
| BP_StatusEffect_Slow sin crear | Pendiente sesion Logica 6 siguiente | Pendiente |
| Velocidad reducida de Slow sin definir | Pendiente decision del cliente | Pendiente |
| Override Walk Speed BPI sin documentar | BPI que vive en BPI_Character — no inspeccionado | Pendiente |

---

*Archivo creado -- sesion Light Paradox (Logica 6)*
*Project: Light Paradox - Base: EasySurvivalRPGv5 - UE 5.4.4*
