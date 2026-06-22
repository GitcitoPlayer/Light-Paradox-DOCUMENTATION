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
Clase hija creada: `BP_StatusEffect_Slow` (velocidad reducida — prototipo funcional Logica 6).

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

## BP_StatusEffect_Slow -- clase hija creada (Logica 6)

`BP_StatusEffect_Slow` es la clase hija de `BP_StatusEffect_OverrideSpeed`
creada en sesion Logica 6 para implementar el Estado Ralentizar.

### Como se creo
1. Content Browser → clic derecho sobre `BP_StatusEffect_OverrideSpeed`
2. **Create Child Blueprint Class**
3. Nombre: `BP_StatusEffect_Slow`

### Como cambiar la velocidad en una clase hija

Las variables heredadas no aparecen en el panel **My Blueprint** del child.
Para cambiar `Overrided Walk Speed` en `BP_StatusEffect_Slow`:

1. Abre `BP_StatusEffect_Slow`
2. Toolbar superior → boton **Class Defaults**
3. Panel **Details** muestra todas las variables heredadas del parent
4. Busca `Overrided Walk Speed` → cambiar valor (prueba actual: **TBD — pendiente cliente**)
5. `Overrided Walk Interp Speed`: dejar en `0.0`
6. Compile → Save

> **Nota:** Si `Overrided Walk Speed` no aparece en Class Defaults, la variable
> en el parent no tiene **Instance Editable** activado. Solucion: abrir
> `BP_StatusEffect_OverrideSpeed` → seleccionar la variable → activar
> **Instance Editable** → Compile → volver al child.

### Fila en DT_StatusEffects

| Campo | Valor |
|---|---|
| `Handle.Data Table` | `DT_StatusEffects` ← **configurar manualmente** |
| `Handle.Row Name` | `Effect_Slow` ← **configurar manualmente** |
| `EffectClass` | `BP_StatusEffect_Slow` |
| `IsPositiveEffect` | False |
| `Duration` | `5.0` (prueba) |
| `MaxStack` | `1` |
| `Name` | `Slow` |
| `EffectAttributes` | vacio |
| `EffectAttributesMapped` | vacio |

> El campo `Handle` debe configurarse manualmente en filas nuevas.
> Ver `15_SYSTEM_StatusEffects.md` seccion "Configuracion del campo Handle".

### Trigger de prueba
`BP_SlowTrigger` — patron identico a `BP_PoisonTrigger` con Row Name `Effect_Slow`.
Ver `19_BLUEPRINT_BP_PoisonTrigger.md`.

---

## Deuda tecnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Velocidad reducida de Slow sin definir | Pendiente decision del cliente | Pendiente |
| Override Walk Speed BPI sin documentar | BPI que vive en BPI_Character — no inspeccionado | Pendiente |

---

*Archivo actualizado -- sesion Light Paradox (Logica 6 -- BP_StatusEffect_Slow creado)*
*Cambios: clase hija documentada, proceso de Class Defaults documentado, fila DT documentada con paso de Handle*
*Project: Light Paradox - Base: EasySurvivalRPGv5 - UE 5.4.4*
