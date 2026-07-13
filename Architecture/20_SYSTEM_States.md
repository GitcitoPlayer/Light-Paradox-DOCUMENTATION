# 20 — Sistema: States (Estados)
### Sistema: States — Sistema central de Estados de Light Paradox
### Base Asset: EasySurvivalRPGv5
### Fuente: Diseño del cliente + sesión Light Paradox
### Proyecto: Light Paradox · UE 5.4.4

---

## Contexto

El sistema de Estados es la capa de abstracción central sobre el sistema de
Status Effects de ESRPGv5. Un Estado es un contenedor de datos que agrupa
uno o más efectos y define el contexto y condiciones de su aplicación.

**Motivación:** Reemplazar la construcción manual de nodos en triggers, enemigos,
trampas, consumibles y runas por una sola llamada a `BP_StateApplier`.
El sistema es el punto de entrada único para aplicar cualquier conjunto de
efectos en cualquier contexto del juego.

---

## Arquitectura del sistema

```
[Cualquier fuente: trigger, enemigo, trampa, consumible, runa]
        │
        ▼
  BP_StateApplier (Blueprint Function Library)
    → Lee DT_States (fila del Estado)
    → Evalúa Rate (probabilidad)
    → Si pasa:
        → Itera Effects array
        → Por cada efecto:
            → Lee Duration de DT_StatusEffects
            → Aplica EffectsDurationOverride si corresponde
            → Llama BP_AbilitySystem.Load Status Effect (Target)
```

---

## STR_StateData — Row Struct

Asset creado: `STR_StateData`

| Campo | Tipo | Descripción |
|---|---|---|
| `Effects` | Array de DataTableRowHandle | Filas de `DT_StatusEffects` que componen este Estado. Un Estado puede contener N efectos. |
| `IsHitEffect` | Boolean | Define el contexto de activación. True = se inflige desde el portador hacia otro actor al golpear. False = se aplica sobre el target designado por el evento que llama a BP_StateApplier. No define quién recibe el efecto — define el contexto de la llamada. |
| `Rate` | Float (0–100) | Probabilidad porcentual de que el Estado se ejecute al activarse. 100 = siempre se aplica. |
| `Duration` | Float | Duración en segundos aplicada a los efectos cuando EffectsDurationOverride = True. 0 = infinito (el efecto persiste hasta que la fuente lo cancele explícitamente). |
| `EffectsDurationOverride` | Boolean | True = usar el valor de Duration de este Estado para todos los efectos contenidos, ignorando sus duraciones individuales en DT_StatusEffects. False = cada efecto usa su propia Duration del DT. |

---

## DT_States — Data Table

Asset creado: `DT_States` — Row Struct: `STR_StateData`

### Filas confirmadas

| Row Name | Effects | IsHitEffect | Rate | Duration | EffectsDurationOverride |
|---|---|---|---|---|---|
| `State_Poison` | `[Effect_Poison]` | False | 100 | 10.0 | True |

> **Nota:** `State_Slow` pendiente de crear siguiendo el mismo patrón.

### Notas de diseño

- Un Estado con `Duration = 0` y `EffectsDurationOverride = True` produce efectos
  infinitos. La cancelación depende de la fuente que los aplicó.
- `IsHitEffect` es una propiedad descriptiva del Estado. El sistema que hace la
  llamada a `BP_StateApplier` puede usarla como filtro para saber qué Estados
  aplicar en cada contexto.
- Múltiples efectos en un Estado se aplican simultáneamente al mismo target.

### Ejemplo — Estado con dos efectos (buff/debuff simultáneo)

```
Row Name: State_CursedHealing
  Effects:                [Effect_Healing, Effect_Slow]
  IsHitEffect:            False
  Rate:                   100
  Duration:               0        ← infinito
  EffectsDurationOverride: True
```

---

## BP_StateApplier — Blueprint Function Library

**Tipo:** Blueprint Function Library
**Estado:** Creado y compilando — pendientes menores (ver deuda técnica)

### Propósito

Punto de entrada único para aplicar un Estado a cualquier target desde
cualquier fuente del juego. Al ser Function Library, se llama directamente
sin instanciar ningún actor en el mundo.

> **Nota crítica:** BP_StateApplier NO debe ser un Actor. Intentar usarlo como
> Actor (via Spawn Actor) congela el editor de Unreal. Debe ser siempre
> Blueprint Function Library.

### Función: ApplyState

**Inputs:**

| Parámetro | Tipo | Descripción |
|---|---|---|
| `StateRowHandle` | DataTableRowHandle | Fila del Estado en DT_States |
| `Target` | Actor Object Reference | Actor que recibirá los efectos |
| `Instigator` | Actor Object Reference | Actor que origina la aplicación (puede ser null) |

### Flujo interno confirmado

```
Entry (StateRowHandle, Target, Instigator)
  → Break DataTableRowHandle (StateRowHandle)
      → Data Table, Row Name
  → Get Data Table Row (DT_States)
      Row Not Found → Return Node
      Row Found →
          Break STR_StateData
            → Effects, IsHitEffect, Rate, Duration, EffectsDurationOverride
          Random Integer in Range (0, 100)
          <= Rate (Float to Int)
          → Branch
              False → Return Node
              True  →
                For Each Loop (Effects)
                  Loop Body →
                    Break DataTableRowHandle (Array Element)
                      → Data Table, Row Name
                    Get Data Table Row (DT_StatusEffects)
                      Row Found →
                        Break STR_StatusEffectInstance → Duration
                        Select (Float)
                          False  → Duration de STR_StatusEffectInstance
                          True   → Duration de STR_StateData
                          Index  → EffectsDurationOverride
                        Cast To BP_Character_Player (Target)
                          Then →
                            Get Ability System Component
                            Make STR_SaveData_StatusEffect
                              Effect Handle:      Array Element
                              Instigator Is Owner: False
                              Stack:              1
                              Time Remaining:     Return Value del Select ← ⚠️ PENDIENTE CONECTAR
                            Load Status Effect
                              Target:    Ability System Component
                              Save Data: STR Save Data Status Effect
```

---

## BP_PoisonTrigger — Estado actual

**Estructura confirmada via captura:**

```
On Component Begin Overlap (Box)
  → Cast To BP_Character_Player (Other Actor)
      Cast Failed → [termina]
      Then →
        Apply State (BP_StateApplier)         ← ⚠️ pins sin conectar — pendiente
          State Row Handle: Make DataTableRowHandle
            Data Table: DT_States
            Row Name:   State_Poison
          Target:     [sin conectar — debe ser As BP Character Player]
          Instigator: [sin conectar]
        → Destroy Actor (self)
```

> **Nota:** El nodo `Apply State` ya está en el graph y conectado en exec.
> Faltan conectar los pins de datos: `State Row Handle`, `Target` e `Instigator`.

---

## Integración con Rune Words (Fase 4)

### Problema

Las runas son ítems con `EquipmentAttributes` (Gameplay Tags + valor numérico).
El cliente requiere que también puedan contener Estados para aplicar efectos
al jugador al equiparlas.

### Decisión arquitectural pendiente

El campo de Estados en el ítem de runa vivirá dentro de `EquipmentAttributes`
en `STR_ItemData`. La viabilidad exacta depende de la inspección de `STR_ItemData`
que ocurrirá en Fase 3.

> **⚠️ Inferencia (no confirmada):** `STR_ItemData` puede ser un struct cerrado
> del asset base. Si no admite campos adicionales sin romper ESRPGv5, se evaluará
> un struct paralelo. Requiere inspección directa antes de tomar decisiones.

### Casos de uso confirmados

**Caso 1 — Runa con IsHitEffect = True:**
Al equipar la runa, el jugador inflige el Estado a los actores que golpea.

**Caso 2 — Runa con IsHitEffect = False, Duration = 0:**
Al equipar la runa, el Estado se aplica al jugador de forma indefinida.
Al desequipar la runa, el Estado se cancela.

### Mecanismo de cancelación de efectos

ESRPGv5 probablemente tiene un mecanismo nativo de remoción de Status Effects
(inferencia basada en la existencia de `Effect_Bleeding` y la probabilidad de
que exista un ítem que lo cancele en el asset). Pendiente inspección en Fase 3.

La cancelación al desequipar runa es el único caso confirmado que se construirá.
Cancelación desde otros contextos queda fuera del alcance actual.

---

## Plan de fases

### Fase 1 — DT_States + BP_StateApplier base 🔄 En progreso
- ✅ `STR_StateData` creado
- ✅ `DT_States` creado con fila `State_Poison`
- ✅ `BP_StateApplier` (Function Library) creado y compilando
- ⚠️ Pin `Time Remaining` sin conectar en `Make STR_SaveData_StatusEffect`
- ⚠️ `BP_PoisonTrigger` — pins `State Row Handle`, `Target`, `Instigator` sin conectar
- ⏳ `BP_SlowTrigger` pendiente de actualizar
- ⏳ Enemigo pendiente de actualizar
- ⏳ `State_Slow` pendiente de crear en DT_States

**Estable cuando:** Triggers y enemigo aplican efectos vía BP_StateApplier sin nodos manuales.

### Fase 2 — IsHitEffect en enemigos ⏳
- Lógica en el enemigo para llamar a BP_StateApplier en evento de golpe
- BP_StateApplier filtra por IsHitEffect

**Estable cuando:** Enemigo aplica Estado de golpe al jugador sin nodos adicionales.

### Fase 3 — Inspección STR_ItemData + mecanismo de remoción ⏳
- Inspeccionar `STR_ItemData` — confirmar si admite campos nuevos
- Inspeccionar mecanismo de cancelación nativo de ESRPGv5 (vía Effect_Bleeding)
- Decisión arquitectural sobre campo de Estados en ítem de runa
- Documentar hallazgos

**Estable cuando:** Información suficiente para construir Fase 4 sin riesgo.

### Fase 4 — Estados en Rune Words ⏳
- Agregar campo de Estados al struct del ítem de runa
- Lógica de aplicación al equipar runa
- Lógica de cancelación al desequipar runa
- Soporte para Duration = 0 (infinito mientras runa esté equipada)

**Estable cuando:** Equipar/desequipar runa activa/cancela sus Estados correctamente.

---

## Pendientes fuera de alcance actual

| Caso de uso | Estado |
|---|---|
| Estados en consumibles | Pospuesto — uso ambiguo |
| Estados en trampas adicionales | Patrón cubierto por triggers — pospuesto |
| Cancelación de efectos desde contextos arbitrarios | Demasiado abierto para construir ahora |

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| `Time Remaining` sin conectar en BP_StateApplier | Pin `Time Remaining` del `Make STR_SaveData_StatusEffect` debe conectarse al `Return Value` del nodo `Select` | ⚠️ Próxima sesión |
| `BP_PoisonTrigger` pins sin conectar | `State Row Handle` → `Make DataTableRowHandle`. `Target` → `As BP Character Player`. `Instigator` → sin conectar por ahora | ⚠️ Próxima sesión |
| `BP_SlowTrigger` sin actualizar | Mismo proceso que BP_PoisonTrigger — crear `State_Slow` en DT_States primero | ⏳ Fase 1 |
| Enemigo sin actualizar | Reemplazar nodos manuales en `Trace Deal Damage` por llamada a `Apply State` | ⏳ Fase 1 |
| `State_Slow` no existe en DT_States | Crear fila siguiendo el mismo patrón que `State_Poison` | ⏳ Fase 1 |
| BP_StateApplier fue Actor antes de Function Library | Usar como Actor via Spawn Actor congela el editor — confirmado. Siempre debe ser Function Library | ✅ Resuelto |

---

*Archivo actualizado — sesión Light Paradox (Fase 1 en progreso)*
*Cambios: STR_StateData, DT_States, BP_StateApplier y BP_PoisonTrigger documentados con estado actual*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
