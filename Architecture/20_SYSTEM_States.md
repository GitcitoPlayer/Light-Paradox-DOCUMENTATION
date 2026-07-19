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
            → Verifica AffectedFactions
            → Lee Duration de DT_StatusEffects
            → Aplica IsPermanent / EffectsDurationOverride
            → Llama BP_AbilitySystem.Load Status Effect (Target)
```

---

## STR_StateData — Row Struct

Asset: `STR_StateData`

| Campo | Tipo | Descripción |
|---|---|---|
| `Effects` | Array de DataTableRowHandle | Filas de `DT_StatusEffects` que componen este Estado. Un Estado puede contener N efectos. |
| `IsHitEffect` | Boolean | Define el contexto de activación. True = se inflige desde el portador hacia otro actor al golpear. False = se aplica sobre el target designado por el evento que llama a BP_StateApplier. |
| `Rate` | Float (0–100) | Probabilidad porcentual de que el Estado se ejecute al activarse. 100 = siempre se aplica. |
| `Duration` | Float | Duración en segundos. Usado cuando EffectsDurationOverride = True. 0 = infinito. |
| `EffectsDurationOverride` | Boolean | True = usar Duration de este Estado para todos los efectos, ignorando sus duraciones individuales en DT_StatusEffects. False = cada efecto usa su propia Duration. |
| `IsPermanent` | Boolean | True = el Estado no expira. Time Remaining se pasa como 0.0. Persiste hasta que la fuente lo cancele explícitamente. Ignora Duration y EffectsDurationOverride. |
| `AffectedFactions` | Array de E_Faction | Facciones que pueden recibir este Estado. Si el array está vacío = afecta a cualquier actor. |

### Semántica de IsHitEffect confirmada

| Contexto | IsHitEffect = False | IsHitEffect = True |
|---|---|---|
| **Runa equipada** | Efecto se aplica al jugador portador | Efecto se aplica a lo que el jugador golpea |
| **Enemigo** | Efecto se aplica al propio enemigo | Efecto se aplica a lo que el enemigo golpea |
| **Trigger de zona** | Efecto se aplica al actor que entra | N/A — triggers usan False por diseño |

### Notas de diseño

- Un Estado con `Duration = 0` y `EffectsDurationOverride = True` produce efectos
  infinitos. La cancelación depende de la fuente que los aplicó.
- `IsPermanent = True` tiene prioridad sobre `EffectsDurationOverride` y `Duration`.
- `AffectedFactions` vacío = sin restricción de facción — afecta a cualquier actor.
- Múltiples efectos en un Estado se aplican simultáneamente al mismo target.

---

## DT_States — Data Table

Asset: `DT_States` — Row Struct: `STR_StateData`

### Filas confirmadas

| Row Name | Effects | IsHitEffect | Rate | Duration | EffectsDurationOverride | IsPermanent | AffectedFactions |
|---|---|---|---|---|---|---|---|
| `State_Poison` | `[Effect_Poison]` | False | 100 | 10.0 | True | False | [] |
| `State_Freeze` | `[Effect_Slow]` | False | 100 | 10.0 | True | False | [] |
| `State_PoisonHit` | `[Effect_Poison]` | True | 25 | 10.0 | True | False | [] |

> **Nota:** `State_PoisonHit` es la fila usada por el enemigo — Rate 25 para que
> no envenene en cada golpe. `State_Poison` es para triggers de zona — Rate 100.

---

## BP_StateApplier — Blueprint Function Library

**Tipo:** Blueprint Function Library
**Estado:** ✅ Funcional

### Propósito

Punto de entrada único para aplicar un Estado a cualquier target desde
cualquier fuente del juego. Al ser Function Library se llama directamente
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
            → Effects, IsHitEffect, Rate, Duration,
              EffectsDurationOverride, IsPermanent, AffectedFactions
          Random Integer in Range (0, 100) <= Float to Int (Rate)
          → Branch
              False → Return Node
              True  →
                For Each Loop (Effects)
                  Loop Body →
                    Break DataTableRowHandle (Array Element)
                    Get Data Table Row (DT_StatusEffects)
                      Row Found →
                        Break STR_StatusEffectInstance → Duration (efecto)
                        Cast To BP_Character_Base (Target)
                          Then →
                            Get Faction → Enum to String → LocalFactionString
                            IS EMPTY (AffectedFactions)
                            → Branch
                                True  → [aplicar efecto]
                                False →
                                    CONTAINS (AffectedFactions, Get Faction)
                                    → Branch
                                        True  → [aplicar efecto]
                                        False → [sin acción]

                [aplicar efecto]
                  Select 1 (Float)
                    False (Option 0) → Duration de STR_StatusEffectInstance
                    True  (Option 1) → Duration de STR_StateData
                    Index → EffectsDurationOverride
                  Select 2 (Float)
                    False (Option 0) → Return Value de Select 1
                    True  (Option 1) → 0.0
                    Index → IsPermanent
                  Get Ability System Component (de As BP Character Base)
                  Make STR_SaveData_StatusEffect
                    Effect Handle:      Array Element
                    Instigator Is Owner: False
                    Stack:              1
                    Time Remaining:     Return Value de Select 2
                  Load Status Effect
                    Target:    Ability System Component
                    Save Data: STR Save Data Status Effect
                  [exec de Load Status Effect sin conexión — loop avanza solo]

                Completed → Return Node
```

### Notas de implementación

- `AffectedFactions` es tipo `Array de E_Faction` — el `CONTAINS` compara
  directamente valores de enum. No requiere conversión a String ni a Name.
- `Get Faction` devuelve `E_Faction` — se conecta directo al pin `Item` del `CONTAINS`.
- El exec de salida de `Load Status Effect` va sin conexión — el `For Each Loop`
  avanza automáticamente. Conectarlo de regreso al loop causa ciclo infinito y
  congela el editor.

---

## E_Faction — confirmado

Asset: `E_Faction` — vive en `BP_Character_Base` (Settings/AI)
Todos los personajes del juego heredan esta variable.

| Display Name | Notas |
|---|---|
| `Player` | Jugador |
| `Undead` | Enemigos no-muertos |
| `SmallAnimals` | Animales pequeños |
| `Human` | Humanos NPC |
| `Chickens` | Gallinas |
| `Pigs` | Cerdos |

> **Nota:** El enum solo tiene Display Name — no tiene Name interno separado.
> Por esto se usa `Enum to String` en lugar de `Enum to Name` para la comparación.
> `Enum to Name` devuelve `NewEnumerator0`, `NewEnumerator1`, etc. — no usar.

---

## DT_Relationships — confirmado

Asset: `DT_Relationships`

| Campo | Tipo | Descripción |
|---|---|---|
| `FriendlyFactions` | Array de E_Faction | Facciones aliadas |
| `EnemyFactions` | Array de E_Faction | Facciones enemigas |

Usado por ESRPGv5 para definir relaciones entre facciones. No se modifica —
`BP_StateApplier` usa `E_Faction` directamente sin consultar este DT.

---

## Triggers de prueba

### StateTrigger (renombrado desde BP_PoisonTrigger)

Patrón base para cualquier trigger de zona que aplique un Estado.

```
On Component Begin Overlap (Box)
  → Cast To BP_Character_Base (Other Actor)
      Cast Failed → [termina]
      Then →
        Apply State (BP_StateApplier)
          State Row Handle: Make DataTableRowHandle
            Data Table: DT_States
            Row Name:   [nombre del Estado]
          Target:     As BP Character Base
          Instigator: [sin conectar]
        → Destroy Actor (self)
```

### BP_SlowTrigger

Mismo patrón que StateTrigger. Row Name: `State_Freeze`.

---

## Integración con enemigos

### BP_Character_Undead2 — Trace Deal Damage

Después de `Apply Advanced Point Damage`:

```
Cast To BP_Character_Base (Damaged Actor)
  Cast Failed → Play Sound at Location
  Then →
    Apply State (BP_StateApplier)
      State Row Handle: Make DataTableRowHandle
        Data Table: DT_States
        Row Name:   State_PoisonHit
      Target:     As BP Character Base
      Instigator: [sin conectar]
```

> **Nota:** `Cast Failed` va a `Play Sound at Location` — no debe conectarse
> a `Apply State`. Error anterior: Cast Failed conectado al flujo de Apply State
> causaba el error de garbage collection.

---

## Cobertura de Estados de gameplay

### ✅ Cubiertos completamente

| Estado | Implementación |
|---|---|
| Envenenar | `BP_StatusEffect_TickDamage` — probado |
| Sangrado | `Effect_Bleeding` ya existe en ESRPGv5 |
| Quemar | Mismo patrón que veneno |
| Congelar | `BP_StatusEffect_OverrideSpeed` velocidad 0 |
| Parálisis | Mismo patrón que Congelar |
| Def aumentada | `BP_StatusEffect_ChangeState` + CharacterAttributes |
| Atk aumentado | Mismo patrón que Def aumentada |
| Def elemental | Mismo patrón — modificador de resistencia |

### 🟡 Cubiertos parcialmente

| Estado | Qué falta |
|---|---|
| Petrificar | Override velocidad cubierto — falta bloquear acciones del jugador |
| Fractura | Necesita definición exacta del cliente |
| Contusión | Necesita definición exacta del cliente |
| Herida grave (Crítico +15%) | Depende de si ESRPGv5 expone CriticalChance como atributo modificable |
| Desgarre (Daño crítico +30%) | Misma situación que Herida grave |
| Invisibilidad | Depende de sistema de aggro/detección de ESRPGv5 |

### 🔴 Requieren sistemas nuevos

| Estado | Razón |
|---|---|
| Terror | Requiere modificar comportamiento de IA |
| Confundir | Requiere invertir controles o redirigir IA |
| Cegar | Requiere efecto de post-process en cámara |

---

## Plan de fases

### Fase 1 — DT_States + BP_StateApplier base ✅
- `STR_StateData` creado
- `DT_States` creado con filas State_Poison, State_Freeze, State_PoisonHit
- `BP_StateApplier` (Function Library) funcional
- Triggers actualizados — StateTrigger y BP_SlowTrigger
- Enemigo actualizado — BP_Character_Undead2
- Fix `Time Remaining` hardcodeado — resuelto via BP_StateApplier
- Sistema de facciones via `AffectedFactions` + `E_Faction`
- `IsPermanent` funcional

### Fase 2 — IsHitEffect como filtro ⏳
- Lógica en BP_StateApplier para usar IsHitEffect como filtro de contexto
- Enemigo: IsHitEffect = True aplica efecto al jugador al golpear ✅ (funcional)
- Pendiente: implementar filtro formal de IsHitEffect dentro de BP_StateApplier

### Fase 3 — Inspección STR_ItemData + mecanismo de remoción ⏳
- Inspeccionar `STR_ItemData` — confirmar si admite campos nuevos
- Inspeccionar mecanismo de cancelación nativo de ESRPGv5 (vía Effect_Bleeding)
- Decisión arquitectural sobre campo de Estados en ítem de runa

### Fase 4 — Estados en Rune Words ⏳
- Campo de Estados en ítem de runa
- Lógica de aplicación al equipar runa
- Lógica de cancelación al desequipar runa
- Soporte para IsPermanent = True mientras runa esté equipada

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Error de stack duplicado en Load Status Effect | "BP_StatusEffect_TickDamage_C_1 is not valid (pending kill or garbage)" — ocurre cuando el efecto ya está activo y se aplica de nuevo. MaxStack = 1 destruye la instancia anterior antes de que la nueva termine de spawnear. No es destructivo. Origen en BP_AbilitySystemComponent de ESRPGv5. | ⚠️ Pendiente investigación |
| IsHitEffect sin filtro formal en BP_StateApplier | El campo existe en DT_States pero BP_StateApplier no lo evalúa aún como condición de flujo | ⏳ Fase 2 |
| STR_ItemData sin inspeccionar | Crítico para Fase 4 | ⏳ Fase 3 |
| Mecanismo de remoción de Status Effects sin confirmar | Inspeccionar Effect_Bleeding + item de cura en ESRPGv5 | ⏳ Fase 3 |

---

*Archivo actualizado — sesión Light Paradox (Fase 1 completa + Fase 2 en progreso)*
*Cambios: BP_StateApplier funcional documentado, sistema de facciones con E_Faction, IsPermanent, triggers y enemigo actualizados, cobertura de Estados evaluada*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
