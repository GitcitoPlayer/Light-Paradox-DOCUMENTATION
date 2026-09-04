# 22 — Sistema: Fases Lunares y Eclipses vinculadas a Estados
### Sistema: Correlación Ciclo Día/Noche → Estados (integración con sistema de States)
### Base Asset: EasySurvivalRPGv5 (Day/Night Module) + sistema propio de States (Light Paradox)
### Fuente: Diseño del cliente + documento original del programador (`Day_Night_Module.pdf`) — sesión Light Paradox
### Proyecto: Light Paradox · UE 5.4.4

---

## Contexto

`BP_DayNight` (ver `21_SYSTEM_DayNightCycle_Migration.md`) ya produce un
ciclo día/noche funcional a nivel visual. Objetivo de esta sesión: conectar
las fases del ciclo (fases lunares, fases de eclipse) al sistema de Estados
ya existente (ver `20_SYSTEM_States.md`) para que ciertos eventos de
gameplay ocurran **exclusivamente** mientras una fase específica está en
curso.

**Nada de este sistema está implementado todavía.** Este archivo documenta
el diseño, las preguntas abiertas, y el plan de prueba de concepto usando
`State_Poison` como Estado de testing.

---

## Diseño objetivo (confirmado por el cliente)

- Deben existir fases lunares vinculadas a eventos de gameplay.
- Deben existir fases de eclipse (solar y/o lunar) vinculadas a eventos de
  gameplay.
- Cada fase debe estar vinculada a un Estado (ver `20_SYSTEM_States.md`,
  `DT_States` / `BP_StateApplier`) que se aplica **solo** mientras esa fase
  está en curso, y se remueve cuando la fase termina.
- Confirmado por el documento original del programador (paráfrasis, no
  cita textual): los efectos lunares deben afectar tanto al jugador como a
  los enemigos, de forma independiente de la personalización del
  personaje. Esto sugiere que el trigger de fase debe aplicarse a
  cualquier actor con `BPI_Character`/`AbilitySystemComponent`, no
  únicamente al jugador — mismo patrón ya usado por
  `BP_StatusEffect_TickDamage`/`BP_AbilitySystem`, que no distingue entre
  jugador y NPC.

---

## Pregunta abierta — número real de fases (sin resolver, marcado como inferencia)

Del PDF original (ver también tablas completas en
`21_SYSTEM_DayNightCycle_Migration.md`):

**Fases lunares:**
La tabla "Moon Phases" del PDF muestra 3 ejemplos de valores del parámetro
`MoonPhase` (0.5 / 0.72 / 0.088) junto con capturas de distintas formas de
luna (llena, creciente, gibosa). **No queda claro** si el sistema original
define exactamente 3 fases discretas usando esos valores como umbrales, o
si `MoonPhase` es simplemente un valor continuo (Scalar Parameter, rango
0.0–1.0) y esos 3 ejemplos son solo puntos ilustrativos dentro de un rango
continuo que recorre las fases reales de la luna (nueva → creciente →
llena → menguante → nueva).

**Fases de eclipse:**
La tabla "Eclipse Parameters" del PDF muestra 3 ejemplos de
`EclipsePhase` (0.5 / 0.25 / 0.01) — pero a diferencia de la tabla de
Moon Phase, aquí también varía `SunScale` entre ejemplos (0.5 → 0.2 → 0.2).
Esto sugiere que estos 3 ejemplos pudieron ser ajustados manualmente caso
por caso para las capturas de pantalla, no que representen "3 fases"
predefinidas del sistema como tal.

**Conclusión provisional (marcada explícitamente como inferencia, NO
confirmada):** dado que tanto `MoonPhase` como `EclipsePhase` son
`Scalar Parameter Value` (Float 0.0–1.0) dentro de un Material Instance,
es más probable que ambos sean valores continuos controlados a lo largo
del ciclo — no un enum nativo de fases discretas en Unreal. Sin embargo,
el sistema de Estados (`DT_States`) necesita de todas formas trabajar con
rangos/umbrales discretos para poder decidir cuándo aplicar y remover un
Estado (ej. "Luna Llena" = `MoonPhase` entre 0.45–0.55). **Esta decisión de
diseño (cuántas fases discretas, y qué rango de valor corresponde a cada
una) no está tomada — debe definirse y aprobarse durante el testing de
esta sesión, empezando por el caso de prueba con `State_Poison`.**

---

## Plan de prueba de concepto — vincular `State_Poison` a una fase

Objetivo de esta sesión: validar el mecanismo completo de principio a fin
con un solo Estado ya existente y funcional (`State_Poison`, ver
`20_SYSTEM_States.md`), antes de decidir el diseño final de todas las
fases.

### Piezas ya existentes que se reutilizan (sin modificar)
- `DT_States` → fila `State_Poison` (`Effects: [Effect_Poison]`,
  `Rate: 100`, `Duration: 10.0`, `EffectsDurationOverride: True`).
- `BP_StateApplier.ApplyState` — ya tiene guard anti-duplicado
  (`Check Status Effect By Handle`) integrado, confirmado funcional en
  `20_SYSTEM_States.md`. Esto es importante: **no es necesario** construir
  un guard nuevo para evitar reaplicar el Estado en cada Tick mientras la
  fase está activa — `ApplyState` ya lo maneja.
- `Remove Status Effect By Handle` — patrón ya usado en Bloque A de
  `EquipmentChanged` (`20_SYSTEM_States.md`) para remover Estados por
  Handle.

### Pieza nueva a construir — detección de fase y trigger

**Pendiente de inspección antes de construir nada (acción inmediata de
esta sesión):**

1. Confirmar si `BP_DayNight` (o `BP_DayNightCycle`, ver discrepancia de
   nombre en `21_SYSTEM_DayNightCycle_Migration.md`) expone `MoonPhase`
   y/o `EclipsePhase` como **variables de Blueprint**, o si esos valores
   viven **únicamente** como parámetros de `MI_SkySpherePhases`
   (Material Instance) sin ninguna variable equivalente accesible desde
   Blueprint. Esto es crítico: si el valor solo existe en el Material
   Instance, se necesita un nodo `Get Scalar Parameter Value` sobre la
   referencia al Material Instance (o sobre el `MPC` si en algún momento
   se migra a un Material Parameter Collection) para poder leerlo desde
   lógica de Blueprint.
2. Si no existe ninguna variable ni forma de leer el valor desde
   Blueprint, la alternativa es que `BP_DayNight` calcule su propio
   "MoonPhase lógico" en paralelo (ej. una función que derive la fase a
   partir de `Time`/`Day`), duplicando el criterio que ya usa el shader.
   Esto es menos elegante (dos fuentes de verdad para lo mismo) pero puede
   ser necesario si el material no expone el dato de vuelta.

### Mecanismo de detección propuesto (a validar esta sesión)

Opción recomendada por prioridad de mínimo cambio (ver reglas del
proyecto — preferir ajustes sobre reestructuras):

```
[BP_DayNight — dentro del mismo lugar donde ya se actualiza Time cada frame,
 o en CheckLight si se prefiere evitar lógica extra por Tick]

  → GET MoonPhase actual (variable propia o leído del Material Instance)
  → In Range (Float): MoonPhase, Min, Max   ← rango de la fase objetivo (ej. Luna Llena)
  → Branch
      True  → [fase activa]
      False → [fase inactiva]
```

Para conectar esto al sistema de Estados sin duplicar aplicaciones cada
frame, se recomienda:

1. Guardar el resultado del `Branch` (fase activa / inactiva) en una
   variable booleana de instancia en `BP_DayNight`, ej.
   `bMoonPhase_Full_Active`.
2. Solo llamar a `ApplyState`/`Remove Status Effect By Handle` en el
   **cambio de estado** de esa variable (usar `SET w/ Notify` o comparar
   contra el valor anterior antes de escribir), no en cada Tick — esto
   evita spam de llamadas incluso aunque `ApplyState` ya tenga guard
   propio, y evita tener que remover/reaplicar el Estado en cada frame
   mientras la fase sigue activa.
3. `ApplyState` necesita un `Target` — para cumplir el requisito de
   "afecta a jugador y enemigos por igual", el trigger no puede limitarse
   a `Get Player Character`. Pendiente decidir: iterar sobre todos los
   actores con `BPI_Character` (patrón similar a
   `Get All Actors with Interface` ya usado en `BP_DayNight` para
   `BPI_IsLight`), o usar algún Subsystem/Manager central. **No
   implementado — decisión pendiente de esta sesión.**

### Prueba de concepto concreta — pasos sugeridos

1. Elegir un rango de prueba para `MoonPhase` que represente "Luna Llena"
   (ej. 0.45–0.55, ajustable).
2. En `BP_DayNight`, agregar la lectura de `MoonPhase` y el `In Range`
   descrito arriba.
3. En el cambio a `True`, llamar `ApplyState(State_Poison, Target: <actor
   de prueba>)`.
4. En el cambio a `False`, remover el Effect Handle correspondiente
   (guardado al aplicar) vía `Remove Status Effect By Handle`.
5. Verificar en juego: el jugador (o actor de prueba) recibe el efecto de
   veneno únicamente mientras `MoonPhase` está dentro del rango elegido, y
   deja de recibirlo (y el efecto activo se remueve, no solo deja de
   reaplicarse) al salir del rango.
6. Confirmar que reentrar al rango en un ciclo posterior vuelve a aplicar
   el Estado correctamente (sin quedar "atascado" en `bMoonPhase_Full_Active
   = True` de un ciclo anterior).

---

## Sesión de implementación — Prueba de concepto simplificada (Poison de noche)

**Decisión de scope tomada esta sesión:** en vez de esperar a definir fases
lunares/eclipse, se redujo el alcance a lo mínimo posible para validar el
mecanismo completo: aplicar `State_Poison` mientras es de noche (usando el
booleano "es de noche" que `BP_DayNight` ya calcula en `CheckLight`), sin
tocar `MoonPhase`/`EclipsePhase` todavía. Esto reemplaza, para el caso de
prueba, al enfoque inicial de un actor `BP_StateTrigger` independiente en
el nivel (overlap-based) — se descartó esa ruta a favor de reusar
`CheckLight`, que ya corre en timer y ya tiene el booleano de noche
calculado, evitando duplicar lógica de detección día/noche en dos lugares
distintos del proyecto.

### Wiring implementado en `BP_DayNight` — confirmado correcto por captura

```
CheckLight (exec)
  → Sequence
      Then 0 → Set Scalar Parameter Value → For Each Loop (All Lights in Level) → Turn on/Off Light   [cadena original, sin cambios]
      Then 1 → Branch
                  Condition ← NOT (mismo NOT que alimenta el For Each Loop — "es de noche")
                  True →
                    Get Player Character (Index 0)
                    → Cast To BP_Character_Base
                    Cast Success →
                      Make DataTableRowHandle (DT_States, State_Poison)
                      → Apply State (Target: As BP_Character_Base)
```

**Confirmado por captura de pantalla:** todos los pines están conectados
correctamente — `Array` del `For Each Loop` sí tiene `All Lights in Level`
conectado, `Condition` del `Branch` sí tiene un cable (no un checkbox
suelto) viniendo del `NOT`. El wiring en sí **no tiene el bug** — se
descarta la hipótesis inicial de pines desconectados.

### Bug confirmado — el Poison nunca se remueve, ni cruzando ciclos completos de día/noche

**Síntoma:** tras dos ciclos completos de día/noche en juego, el Poison
aplicado de noche nunca desaparece durante el día.

**Método de diagnóstico usado — lección de arquitectura para futuras
sesiones:** el primer intento de diagnóstico colocó un `Print String`
**antes** del `Branch`, conectado directo al `Then 1` del `Sequence`. Ese
print apareció constantemente, de día y de noche — lo cual **no es un bug**,
es el comportamiento esperado: `Then 1` se ejecuta en cada tick del timer
de `CheckLight` (cada `Total Day Night Time` segundos) sin importar el
booleano. Un print colocado antes de un `Branch` nunca puede confirmar en
qué rama del `Branch` está entrando la ejecución — **el print debe ir
después del `Branch`, en el pin específico (`True` o `False`) que se quiere
verificar.**

Al mover el `Print String` al pin `True` del `Branch`, se confirmó:

**Caso A confirmado:** el `Branch` entra en `True` de forma constante,
sin nunca pasar a `False` — es decir, el booleano "es de noche"
(`NOT (In Range(Time, Sunrise Time, Sunset Time))`) **nunca cambia a
falso**, sin importar cuántos ciclos de día/noche transcurran en juego.

**Esto descarta la hipótesis inicial** sobre `ApplyState`/`Time Remaining`
en `Load Status Effect` — el problema no es que el Effect no expire, es que
el trigger cree constantemente que es de noche y sigue reaplicando el
Estado en cada tick de `CheckLight` (el guard de `ApplyState` evita
duplicar la instancia, pero el efecto se ve "permanente" porque nunca deja
de estar en el rango que lo dispara).

### Hipótesis para la próxima sesión — posible causa raíz compartida con Problema Visual 1

**No confirmado, marcado explícitamente como hipótesis a validar:** el
mismo booleano `NOT (In Range(Time, Sunrise Time, Sunset Time))` alimenta
tanto la rama de luces (`For Each Loop → Turn on/Off Light`, cadena
original) como esta rama nueva de Poison. Si `Sunrise Time` y `Sunset Time`
están mal configurados en la instancia de `BP_DayNight` (ej. ambos en
`0.0`, o iguales entre sí, o fuera del rango esperado 0–24), **el mismo
bug podría explicar simultáneamente:**
1. Este bug nuevo (Poison "permanente", `In Range` siempre devuelve el
   mismo resultado).
2. **Problema Visual 1**, ya documentado en
   `21_SYSTEM_DayNightCycle_Migration.md` (el cielo se ilumina de noche
   como si fuera de día) — si el mismo `In Range` alimenta la lógica de
   luces del nivel, un `Sunrise/Sunset Time` mal configurado afectaría
   ambos sistemas a la vez, ya que comparten la misma condición booleana.

**Acción pendiente para la próxima sesión (primera prioridad):**
1. Seleccionar la instancia de `BP_DayNight` en el nivel (no la clase) →
   panel Details → confirmar los valores reales de `Sunrise Time` y
   `Sunset Time`.
2. Confirmar que no sean iguales entre sí, no sean ambos `0.0`, y estén
   dentro de un rango coherente con `Time` (0–24).
3. Si el fix es simplemente corregir esos dos valores, revalidar en juego
   tanto el Poison (debe desaparecer de día) como el Problema Visual 1
   (la iluminación debería empezar a comportarse distinto) — un solo fix
   podría resolver ambos pendientes.
4. Si los valores ya son correctos, revisar los checkboxes `Inclusive Min`
   / `Inclusive Max` de `In Range (Float)` — con ambos activados y
   `Sunrise Time`/`Sunset Time` mal ordenados (ej. Sunrise > Sunset) el
   rango lógico podría invertirse.

---

## Deuda técnica / pendientes registrados

| Problema | Notas | Estado |
|---|---|---|
| No confirmado si `MoonPhase`/`EclipsePhase` son legibles desde Blueprint | Puede que solo existan como parámetros de `MI_SkySpherePhases` sin variable equivalente en `BP_DayNight` | 🔴 Bloqueante — primera acción de esta sesión |
| Número real de fases lunares sin definir | PDF ambiguo entre 3 fases discretas o valor continuo — ver sección dedicada arriba | 🔴 Pendiente decisión de diseño |
| Número real de fases de eclipse sin definir | Mismo problema que fases lunares, agravado por `SunScale` variable entre ejemplos del PDF | 🔴 Pendiente decisión de diseño |
| Mecanismo de aplicar el Estado a jugador Y enemigos | Requiere iterar sobre actores con `BPI_Character`, no solo `Get Player Character` — sin implementar | 🔴 Pendiente diseño e implementación |
| Guard contra reaplicación en cada Tick mientras la fase está activa | Propuesta: variable booleana + detectar solo el cambio de estado, no el estado en sí | 🟡 Diseño propuesto, sin implementar |
| Prueba de concepto con `State_Poison` | Implementada — wiring confirmado correcto por captura, reusa `CheckLight` en vez de actor `BP_StateTrigger` independiente | ✅ Implementado — bug funcional pendiente, ver fila siguiente |
| **Bug — Poison nunca se remueve, `Branch` siempre entra en `True`** | Confirmado esta sesión: el booleano "es de noche" (`NOT(In Range(Time, Sunrise Time, Sunset Time))`) nunca cambia a `False`. Hipótesis principal: `Sunrise Time`/`Sunset Time` mal configurados en la instancia de `BP_DayNight` — posible causa raíz compartida con Problema Visual 1 de `21_SYSTEM_DayNightCycle_Migration.md`. Ver sección dedicada arriba. | 🔴 Pendiente — primera acción de la próxima sesión |

---

## Ver también

- `20_SYSTEM_States.md` — sistema de Estados (`DT_States`,
  `BP_StateApplier`), reutilizado sin modificar en este sistema.
- `21_SYSTEM_DayNightCycle_Migration.md` — sistema base de ciclo día/noche,
  incluye discrepancia de nombre `BP_DayNight` vs `BP_DayNightCycle` y la
  lista completa de parámetros de `MI_SkySpherePhases` confirmados por el
  PDF original.
- `15_SYSTEM_StatusEffects.md` — patrón de trigger mínimo
  (`Make DataTableRowHandle` → `Make STR_SaveData_StatusEffect` →
  `Load Status Effect`) reutilizable si se decide aplicar el Effect
  directamente en vez de pasar por la capa de State.

---

*Archivo creado — sesión Light Paradox (inicio del sistema de fases
lunares y eclipse vinculadas a Estados). Documenta diseño objetivo
confirmado por el cliente, ambigüedad sin resolver sobre el número real de
fases (lunares y eclipse) según el documento original del programador, y
plan de prueba de concepto usando `State_Poison` como Estado de testing
antes de tomar decisiones de diseño final.*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
