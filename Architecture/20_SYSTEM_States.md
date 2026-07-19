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
| `IsHitEffect` | Boolean | Propiedad de diseño. True = Estado ofensivo, se aplica al golpear. False = Estado pasivo, se aplica al portador o por evento. No es un filtro de código — es una convención de diseño que condiciona qué flujo del enemigo lo ejecuta. |
| `Rate` | Float (0–100) | Probabilidad porcentual de que el Estado se ejecute al activarse. 100 = siempre se aplica. |
| `Duration` | Float | Duración en segundos. Usado cuando EffectsDurationOverride = True. 0 = infinito. |
| `EffectsDurationOverride` | Boolean | True = usar Duration de este Estado para todos los efectos, ignorando sus duraciones individuales en DT_StatusEffects. False = cada efecto usa su propia Duration. |
| `IsPermanent` | Boolean | True = el Estado no expira. Time Remaining se pasa como 0.0. Persiste hasta que la fuente lo cancele explícitamente. Ignora Duration y EffectsDurationOverride. |
| `AffectedFactions` | Array de E_Faction | Facciones que pueden recibir este Estado. Si el array está vacío = afecta a cualquier actor. |

### Semántica de IsHitEffect confirmada

| Contexto | IsHitEffect = False | IsHitEffect = True |
|---|---|---|
| **Runa equipada** | Efecto se aplica al jugador portador | Efecto se aplica a lo que el jugador golpea |
| **Enemigo** | Efecto se aplica al propio enemigo en Init Character | Efecto se aplica al actor golpeado en Trace Deal Damage |
| **Trigger de zona** | Efecto se aplica al actor que entra | N/A — triggers usan False por diseño |

> **Nota crítica:** `IsHitEffect` no es evaluado dentro de `BP_StateApplier`.
> Es una propiedad de diseño que el Blueprint del enemigo (u otra fuente) lee
> para decidir en qué momento llamar a `ApplyState` y con qué Target.

### Notas de diseño

- Un Estado con `Duration = 0` y `EffectsDurationOverride = True` produce efectos infinitos.
- `IsPermanent = True` tiene prioridad sobre `EffectsDurationOverride` y `Duration`.
- `AffectedFactions` vacío = sin restricción — afecta a cualquier actor.
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

---

## BP_StateApplier — Blueprint Function Library

**Tipo:** Blueprint Function Library
**Estado:** ✅ Funcional

> **Nota crítica:** BP_StateApplier NO debe ser un Actor. Intentar usarlo como
> Actor via Spawn Actor congela el editor de Unreal. Debe ser siempre
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
  → Get Data Table Row (DT_States)
      Row Not Found → Return Node
      Row Found →
          Break STR_StateData
          Random Integer in Range (0, 100) <= Float to Int (Rate)
          → Branch
              False → Return Node
              True  →
                For Each Loop (Effects)
                  Loop Body →
                    Break DataTableRowHandle (Array Element)
                    Get Data Table Row (DT_StatusEffects)
                      Row Found →
                        Break STR_StatusEffectInstance → Duration
                        Cast To BP_Character_Base (Target)
                          Then →
                            Get Faction → Enum to String
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
                    False → Duration de STR_StatusEffectInstance
                    True  → Duration de STR_StateData
                    Index → EffectsDurationOverride
                  Select 2 (Float)
                    False → Return Value de Select 1
                    True  → 0.0
                    Index → IsPermanent
                  Get Ability System Component (de As BP Character Base)
                  Make STR_SaveData_StatusEffect
                    Effect Handle:       Array Element
                    Instigator Is Owner: False
                    Stack:               1
                    Time Remaining:      Return Value de Select 2
                  Load Status Effect
                    Target:    Ability System Component
                    Save Data: STR Save Data Status Effect
                  [exec de Load Status Effect sin conexión — loop avanza solo]

                Completed → Return Node
```

### Notas de implementación

- `AffectedFactions` es tipo `Array de E_Faction` — `CONTAINS` compara valores
  de enum directamente. No requiere conversión a String ni a Name.
- `Get Faction` devuelve `E_Faction` — conecta directo al pin `Item` del `CONTAINS`.
- El exec de salida de `Load Status Effect` va sin conexión — el `For Each Loop`
  avanza automáticamente. Conectarlo de regreso al loop causa ciclo infinito
  y congela el editor.
- `Enum to Name` devuelve `NewEnumerator0`, `NewEnumerator1` — NO usar.
  `E_Faction` solo tiene Display Name, no Name interno. Usar `Enum to String`.

---

## E_Faction — confirmado

Variable en `BP_Character_Base` — categoría Settings/AI — tipo `E_Faction`.
Todos los personajes del juego heredan esta variable.

| Display Name |
|---|
| `Player` |
| `Undead` |
| `SmallAnimals` |
| `Human` |
| `Chickens` |
| `Pigs` |

---

## Integración con enemigos — BP_Character_Undead2

### Variable BaseState

| Propiedad | Valor |
|---|---|
| Nombre | `BaseState` |
| Tipo | `DataTableRowHandle` |
| Instance Editable | ✅ |
| Category | `Customization` |

Configurable desde el panel **Details** del actor en escena o desde el Blueprint.
No requiere editar nodos para cambiar el Estado del enemigo.

### Init Character (función override en BP_Character_Undead2)

Aplica el Estado base cuando `IsHitEffect = False` — el enemigo se aplica
el efecto a sí mismo al inicializarse.

```
[final de la cadena existente — después de Create Mark]
  → Get Data Table Row (BaseState)
      Row Not Found → Return Node
      Row Found →
          Break STR_StateData → IsHitEffect
          → Branch
              True  → Return Node  ← Estado ofensivo, no aplica en Init
              False →
                Item Handle Is Valid (BaseState)
                → Branch
                    False → Return Node
                    True  →
                      Apply State
                        State Row Handle: BaseState
                        Target: Self
                      → Return Node
```

> **Nota:** El Apply State debe ir después de `Parent: Init Character` y de
> toda la cadena de inicialización del Undead. Aplicarlo antes causa bug de
> vida en 0 que produce inmortalidad.

### Trace Deal Damage (función en BP_Character_Undead2)

Aplica el Estado al actor golpeado cuando `IsHitEffect = True`.

```
[después de Apply Advanced Point Damage]
  → Cast To BP_Character_Base (Damaged Actor)
      Cast Failed → Play Sound at Location
      Then →
        Get Data Table Row (BaseState)
          Row Not Found → [sin acción]
          Row Found →
              Break STR_StateData → IsHitEffect
              → Branch
                  False → [sin acción] ← Estado pasivo, ya se aplicó en Init
                  True  →
                    Apply State
                      State Row Handle: BaseState
                      Target: As BP Character Base
                      Instigator: [sin conectar]
```

> **Nota crítica:** `Cast Failed` debe ir a `Play Sound at Location` — no al
> flujo de Apply State. Conectar Cast Failed al Apply State causa error de
> garbage collection en BP_AbilitySystemComponent.

---

## Triggers de prueba

### StateTrigger (renombrado desde BP_PoisonTrigger)

```
On Component Begin Overlap (Box)
  → Cast To BP_Character_Base (Other Actor)
      Cast Failed → [termina]
      Then →
        Apply State
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

## Cobertura de Estados de gameplay

### ✅ Cubiertos completamente
Envenenar, Sangrado, Quemar, Congelar, Parálisis, Def aumentada, Atk aumentado, Def elemental

### 🟡 Cubiertos parcialmente
Petrificar, Fractura, Contusión, Herida grave, Desgarre, Invisibilidad

### 🔴 Requieren sistemas nuevos
Terror, Confundir, Cegar

---

## Plan de fases

### Fase 1 — DT_States + BP_StateApplier base ✅
- `STR_StateData`, `DT_States`, `BP_StateApplier` creados y funcionales
- Triggers actualizados — StateTrigger y BP_SlowTrigger
- Enemigo actualizado — BP_Character_Undead2
- Fix `Time Remaining` hardcodeado resuelto
- Sistema de facciones via `AffectedFactions` + `E_Faction`
- `IsPermanent` funcional

### Fase 2 — IsHitEffect como filtro de diseño ✅
- `IsHitEffect` condiciona flujo en Init Character y Trace Deal Damage
- `BaseState` configurable desde Details del actor
- Init Character aplica Estado pasivo (IsHitEffect = False) al enemigo
- Trace Deal Damage aplica Estado ofensivo (IsHitEffect = True) al target golpeado
- Pruebas confirmadas: veneno pasivo al inicio y veneno ofensivo al golpear

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

## Pendientes fuera de alcance actual

| Caso de uso | Estado |
|---|---|
| BaseState en BP_Character_Base | Evaluar si conviene mover la variable y lógica a la clase base para que todos los personajes la hereden — pendiente decisión del cliente |
| Estados en consumibles | Pospuesto — uso ambiguo |
| Cancelación de efectos desde contextos arbitrarios | Demasiado abierto para construir ahora |

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Error de stack duplicado en Load Status Effect | "BP_StatusEffect_TickDamage_C_1 is not valid (pending kill or garbage)" — ocurre cuando el efecto ya está activo y se aplica de nuevo. MaxStack = 1 destruye la instancia anterior antes de que la nueva termine de spawnear. No destructivo. Origen en BP_AbilitySystemComponent de ESRPGv5. | ⚠️ Pendiente investigación |
| BaseState solo en BP_Character_Undead2 | Evaluar migración a BP_Character_Base para todos los personajes | ⏳ Pendiente decisión |
| STR_ItemData sin inspeccionar | Crítico para Fase 4 | ⏳ Fase 3 |
| Mecanismo de remoción de Status Effects sin confirmar | Inspeccionar Effect_Bleeding + item de cura en ESRPGv5 | ⏳ Fase 3 |

---

*Archivo actualizado — sesión Light Paradox (Fase 1 ✅ + Fase 2 ✅)*
*Cambios: IsHitEffect documentado como propiedad de diseño, BaseState en Undead, flujo Init Character y Trace Deal Damage documentados, pruebas confirmadas*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
