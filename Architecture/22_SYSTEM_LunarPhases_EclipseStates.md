# 22 — Sistema: Fases Lunares y Eclipses vinculadas a Estados
### Sistema: Correlación Ciclo Día/Noche → Eventos cósmicos → Estados (integración con sistema de States)
### Base Asset: EasySurvivalRPGv5 (Day/Night Module) + sistema propio de States (Light Paradox)
### Fuente: Diseño del cliente + documento original del programador (`Day_Night_Module.pdf`) + implementación en sesión Light Paradox
### Proyecto: Light Paradox · UE 5.4.4
### Actualizado: sesión Light Paradox — DataTable `DT_CelestialEvents` v1 (calendario por mes/día + hora, debug log)

---

## Contexto

`BP_DayNight` (ver `21_SYSTEM_DayNightCycle_Migration.md`) produce un ciclo
día/noche funcional y calibrado. Objetivo: conectar fases lunares y eclipses
al sistema de Estados (`20_SYSTEM_States.md`) para que ciertos eventos de
gameplay ocurran **exclusivamente** mientras una fase específica está en
curso.

**Estado actual (esta sesión):**
- ✅ Prueba de concepto Poison de noche (resuelta, ver sección más abajo).
- ✅ **Nuevo:** `DT_CelestialEvents` v1 implementado. Detecta eventos por
  **hora + día del mes + mes** e imprime ENTER/EXIT en el debug log. Sin
  efecto visual y sin Estados todavía (decisión del usuario para esta fase).
- 🔴 Pendientes de esta fase: eventos que cruzan medianoche, prioridad de
  eclipses sobre lunas, aplicar Estados y alimentar el visual (`MoonPhase`,
  `Eclipse Phase`). Ver "Pendientes para la siguiente sesión".

---

## Diseño objetivo (confirmado por el cliente)

- Fases lunares vinculadas a eventos de gameplay.
- Fases de eclipse (solar y/o lunar) vinculadas a eventos de gameplay.
- Cada fase se vincula a un Estado (`DT_States` / `BP_StateApplier`) que se
  aplica **solo** mientras esa fase está en curso y se remueve al terminar.
- Los efectos lunares afectan a jugador y enemigos por igual (paráfrasis del
  PDF original), por lo que el trigger no debe limitarse al jugador. Pendiente
  de implementar (ver pendientes).

### Reglas del calendario dadas por el cliente (esta sesión)

Con un calendario lunar de referencia (julio 2020: luna llena el día 5,
luna nueva el día 20, cuartos el 13 y el 27):

- Todos los meses tienen el mismo ciclo lunar, así que las lunas se definen
  **solo por día del mes**, ignorando el mes.
- Eclipses lunares ocurren en luna llena; eclipses solares en luna nueva.
- 4 eclipses al año (2 lunares y 2 solares), alternados, cada 3 meses.
  Ejemplo del cliente: mes 3 solo eclipse lunar; mes 6 solo eclipse solar.
- Un evento de eclipse **reemplaza** a la luna de ese día.

> **⚠️ Contradicción en el brief del cliente (pendiente confirmar):** dijo
> "cada sexta luna llena es un eclipse solar y cada sexta luna nueva es un
> eclipse lunar", pero también que lunares van en luna llena y solares en
> luna nueva. Se siguió lo segundo y los ejemplos (mes 3 día 5 = eclipse
> lunar; mes 6 día 20 = eclipse solar). Meses sugeridos: lunares 3 y 9;
> solares 6 y 12. **Inferencia mía, no confirmada.**

---

## ✅ Implementación v1 — `DT_CelestialEvents` (esta sesión)

**Estado:** implementado; el usuario confirmó que funciona ("parece que está
todo bien"). Los resultados detallados de las pruebas A/B de la tabla de más
abajo no fueron reportados fila por fila — validar con la prueba de `Day` fija
cuando se retome.

### Diseño — cambio respecto a la propuesta inicial

La propuesta inicial usaba `IntervalDays` y `DayOffset`. El usuario las
cambió por `EventMonths` y `EventDays` por ser más intuitivas para diseño
(se escribe el día exacto, no una aritmética). `StartHour`/`EndHour` se
conservan.

Para esto `Day` (contador corrido) se convierte a mes y día del mes con
`Days Per Month` y `Months Per Year` (variables de `BP_DayNight`, ver `21`).

### Assets (en `Content/LightParadox/Blueprints/DayNightCycle`)

**Enum `E_CelestialEventType`:** `MoonPhase`, `SolarEclipse`, `LunarEclipse`.

**Struct `STR_CelestialEvent` (Row Struct):**

| Campo | Tipo | Notas |
|---|---|---|
| `EventType` | `E_CelestialEventType` | Tipo de evento |
| `StartHour` | Float | Hora de inicio (0–24) |
| `EndHour` | Float | Hora de fin (0–24). **v1: debe ser mayor que `StartHour` (no cruza medianoche)** |
| `EventMonths` | Array de Integer | Meses en que ocurre. **Vacío o que contenga `0` = todos los meses** |
| `EventDays` | Array de Integer | Días del mes en que ocurre. **Vacío o que contenga `0` = todos los días** |

**DataTable `DT_CelestialEvents`** — filas de prueba (3 requeridas, 1 opcional):

| Row Name | EventType | StartHour | EndHour | EventMonths | EventDays |
|---|---|---|---|---|---|
| `Moon_Gibbous` | MoonPhase | 18 | 24 | (vacío) | 1, 2, 3, 4 |
| `Moon_Full` | MoonPhase | 18 | 24 | (vacío) | 5 |
| `Eclipse_Lunar` | LunarEclipse | 18 | 24 | 3, 9 | 5 |
| `Eclipse_Solar` (opcional) | SolarEclipse | 18 | 24 | 6, 12 | 20 |

> **Nota:** `EndHour = 24` es visualmente incorrecto (la luna se mete a las 6).
> Es una limitación de la v1; ver pendiente "eventos que cruzan medianoche".
> Un `EndHour = 6` hace que el evento **deje de funcionar** (comprobado por el
> usuario) porque `In Range` con `Min > Max` nunca es verdadero.

### Variables en `BP_DayNight` (ver `21` para la tabla completa)

`Days Per Month` (Integer, 30), `Months Per Year` (Integer, 12),
`Active Celestial Events` (Array de Name, variable real).

### Función `CheckCelestialEvents` (BP_DayNight, sin inputs)

Variables locales: `LocalDayOfMonth` (Integer), `LocalMonth` (Integer).

```
Entry
  → SET LocalDayOfMonth ← ((Day - 1) % Max(Days Per Month, 1)) + 1
  → SET LocalMonth      ← (((Day - 1) / Max(Days Per Month, 1)) % Max(Months Per Year, 1)) + 1
  → For Each Loop (Array: Get Data Table Row Names(DT_CelestialEvents))
      Loop Body → Get Data Table Row (DT_CelestialEvents, Row Name: Array Element)
        Row Found → Break STR_CelestialEvent
          HourMatch  = In Range (Time, StartHour, EndHour)   [Inclusive Min = true, Inclusive Max = false]
          MonthMatch = OR( Length(EventMonths) == 0,
                           Contains(EventMonths, 0),
                           Contains(EventMonths, LocalMonth) )
          DayMatch   = OR( Length(EventDays) == 0,
                           Contains(EventDays, 0),
                           Contains(EventDays, LocalDayOfMonth) )
          IsActive      = AND(HourMatch, MonthMatch, DayMatch)
          AlreadyActive = Contains(Active Celestial Events, Array Element)
          Branch (IsActive)
            True  → Branch (NOT AlreadyActive)
                      True → ADD Active Celestial Events ← Array Element
                           → Print String "[Celestial] ENTER: <row> | Month <m> Day <d> | Time <t>" (verde)
            False → Branch (AlreadyActive)
                      True → Remove Item Active Celestial Events ← Array Element
                           → Print String "[Celestial] EXIT: <row> | Month ... " (rojo)
```

**Llamada:** `CheckLight` → `Sequence` → `Then 4` → `CheckCelestialEvents`
(Target: self). Corre en el timer de `CheckLight`, igual que Poison, luces y
`Night Only Items`. No se creó un timer nuevo.

> **Lección (repetida del Bug 3 de `20_SYSTEM_States.md`):** `Active Celestial
> Events` debe ser una variable real; `Contains`/`Add`/`Remove` sobre una
> salida de función pura reevalúan desde cero y producen ENTER repetido.

> **Error de instrucción corregido durante la sesión:** en los pines de
> `Append` hay que **conectar** `GET Time`, `GET Day`, etc. Escribir esas
> palabras en el pin imprime el texto literal.

### Pruebas sugeridas (con `Day` fijo en la instancia — Day es Instance Editable)

| Prueba | Setup | Esperado |
|---|---|---|
| A — eclipse mes 3 | `Day = 64` (mes 3, día 4), `Start Time = 17` | Día 64 ~18: ENTER `Moon_Gibbous`; día 65 ~0: EXIT; día 65 ~18: ENTER `Moon_Full` **y** `Eclipse_Lunar` (dos eventos a la vez: pendiente de prioridad); día 66 ~0: EXIT ambos |
| B — filtro de meses | `Day = 4` (mes 1, día 4), `Start Time = 17` | Al pasar al día 5: ENTER `Moon_Full`, **sin** `Eclipse_Lunar` |

### Limitaciones conocidas de la v1

- No soporta eventos que cruzan medianoche (`StartHour > EndHour`).
- Dos eventos pueden estar activos a la vez (eclipse + luna). Sin prioridad.
- Granularidad: corre en el timer de `CheckLight`; entradas/salidas pueden
  retrasarse hasta un tick. Duración real del ciclo no medida.
- Solo imprime log; no aplica Estados ni cambia visual.
- No hay validación de datos (por ejemplo `EventDays` con valores fuera de
  1–`Days Per Month`).

---

## Pendientes para la siguiente sesión (correcciones pedidas por el usuario)

### 1. Eventos que cruzan medianoche (`EndHour` a las 6)

**Causa:** `In Range (Time, 18, 6)` tiene `Min > Max`, nunca es verdadero.

**Propuesta (inferencia, sin probar):**
1. Si `StartHour <= EndHour`: igual que hoy.
2. Si `StartHour > EndHour` (cruza medianoche): `HourMatch = Time >= StartHour`
   **OR** `Time < EndHour`.
3. Un evento pertenece al día en que **empieza**. En la parte de madrugada
   (`Time < EndHour`) `Day` ya subió, así que el cálculo de mes y día del mes
   debe usar `Day - 1` para esa fila. Si no, la luna del día 5 buscaría el
   día 6 y se apagaría a las 0:00.
4. Por lo anterior, el cálculo de `LocalDayOfMonth` y `LocalMonth` pasa
   **dentro** del loop (cada fila puede necesitar un `Day` distinto).
5. Borde: en `Day = 1` con `Time < EndHour`, `Day - 1` da 0; proteger con
   `Max (Integer)` contra 1.

Resultado esperado: `StartHour = 18`, `EndHour = 6` funciona para lunas y
eclipses lunares. El eclipse solar debe tener horario **de día** (ej. 10–14),
porque es cuando el sol está visible.

### 2. Prioridad: el eclipse reemplaza a la luna

| Opción | Cómo funciona | Cambio |
|---|---|---|
| **A. Por tipo (recomendada)** | Si hay algún evento activo de tipo `SolarEclipse` o `LunarEclipse`, las filas `MoonPhase` se consideran inactivas | Dos pasadas por la tabla y variable local `EclipseActive`. No cambia el struct |
| **B. Campo `Priority`** | Cada fila tiene `Priority` (Integer). Gana el mayor entre los activos | Campo nuevo en el struct. Escala mejor si hay más tipos |

Con A, el flujo ENTER/EXIT existente se encarga solo: al empezar el eclipse la
luna deja de estar activa (EXIT) y al terminar vuelve a imprimir ENTER si la
luna sigue en su horario. **Decisión entre A y B: pendiente del usuario.**

### 3. Otros pendientes de esta fase

| Pendiente | Notas |
|---|---|
| Confirmar con el cliente la contradicción de reglas de eclipse | Ver sección Diseño objetivo |
| Fila `Eclipse_Solar` con horario diurno | Depende del punto 1 |
| Aplicar Estados a jugador **y** enemigos | Iterar actores con `BPI_Character` (patrón `Get All Actors with Interface`, como `BPI_IsLight`) o un manager central. Sin implementar |
| Guardar/remover el Effect Handle al salir del evento | Reutilizar `Remove Status Effect By Handle` (ver `20_SYSTEM_States.md`, Bloque A) |
| Alimentar el visual (`MoonPhase`, `Eclipse Phase`) desde el evento | Ver nota de diseño abajo |
| Campo `StateRowHandle` en `STR_CelestialEvent` | Para vincular fila de `DT_States` |
| Validación de datos de la tabla | Días/meses fuera de rango, `StartHour == EndHour` |

> **Nota de diseño (decisión del usuario/cliente, anotada):** con la tabla por
> calendario, el que manda es el calendario y el visual del shader se
> alimentará **desde** el evento. Esto elimina el bloqueo anterior ("¿se puede
> leer `MoonPhase` desde Blueprint?") para decidir *cuándo* aplicar un Estado.
> Sigue pendiente saber cómo escribir `MoonPhase`/`Eclipse Phase` en
> `MI_SkySpherePhases` cuando llegue la fase visual.

---

## Contexto histórico — pregunta abierta sobre número de fases

> **Parcialmente superada** por el diseño por calendario, pero se conserva
> como referencia para la fase visual.

Del PDF original (tablas completas en `21_SYSTEM_DayNightCycle_Migration.md`):

- **Fases lunares:** la tabla "Moon Phases" muestra 3 ejemplos de `MoonPhase`
  (0.5 / 0.72 / 0.088). No queda claro si hay 3 fases discretas o un valor
  continuo 0.0–1.0.
- **Fases de eclipse:** 3 ejemplos de `EclipsePhase` (0.5 / 0.25 / 0.01) donde
  también varía `SunScale`; probablemente ajustados a mano para las capturas.
- **Inferencia (no confirmada):** ambos son `Scalar Parameter Value` continuos.
  Para la fase visual habrá que decidir qué valor escribir por evento.

---

## Prueba de concepto — Poison de noche (✅ resuelta en sesión anterior)

**Decisión de scope:** en vez de esperar a las fases, se aplicó `State_Poison`
mientras es de noche, usando el booleano "es de noche" que `BP_DayNight` ya
calcula en `CheckLight`. Se descartó un actor `BP_StateTrigger` independiente
(overlap) para no duplicar la lógica día/noche.

```
CheckLight (exec)
  → Sequence
      Then 0 → [luces, cadena original]
      Then 1 → Branch (Condition ← NOT(In Range(Time, Sunrise Time, Sunset Time)))
                  True → Get Player Character (0) → Cast To BP_Character_Base
                           Cast Success → Make DataTableRowHandle (DT_States, State_Poison)
                                        → Apply State (Target: As BP_Character_Base)
      Then 2 → [Night Only Items — ver 23]
      Then 3 → [SkyLight día/noche — ver 21]
      Then 4 → [CheckCelestialEvents — esta sesión]
```

**Bug resuelto:** el Poison nunca se removía. Causa raíz: `Sunrise Time` /
`Sunset Time` en `0.0` (sin Instance Editable). Fix: expuestas +
`Sunrise = 6.0` / `Sunset = 18.0`. Confirmado en juego. Detalle en `21`.

**Lección de diagnóstico:** un `Print String` antes de un `Branch` no puede
confirmar en qué rama entra la ejecución; debe ir **después**, en el pin
`True` o `False` específico.

**Pendiente heredado:** el Poison de noche solo se aplica al jugador y no se
remueve explícitamente al amanecer más allá del Duration del efecto; revisar
cuando se implemente el manejo de Estados por evento.

---

## Deuda técnica / pendientes registrados

| Problema | Notas | Estado |
|---|---|---|
| Eventos que cruzan medianoche | Ver pendiente 1 | 🔴 Siguiente sesión |
| Prioridad eclipse sobre luna | Ver pendiente 2 (opción A o B) | 🔴 Siguiente sesión |
| Contradicción de reglas de eclipse del cliente | Ver Diseño objetivo | 🟡 Confirmar con el cliente |
| Aplicar Estado a jugador Y enemigos | Requiere iterar actores con `BPI_Character` | 🔴 Pendiente diseño e implementación |
| Remover Estado al terminar el evento | Guardar Effect Handles aplicados por evento | 🔴 Pendiente |
| Alimentar visual (`MoonPhase`, `Eclipse Phase`) | Sin diseño | 🟡 Pendiente |
| Duración real del ciclo no medida | Afecta granularidad del log | ⏳ Pendiente medir |
| Desincronización Poison/luces vs visual (`Sun Yaw`) | **✅ Resuelta** — `Sunrise 6` / `Sunset 18` + `Sun Yaw` 90→450 Linear. Ver `21` | ✅ Resuelto |
| Poison "nunca se remueve" | ✅ Resuelto (Sunrise/Sunset en 0.0) | ✅ Resuelto |

---

## Ver también

- `20_SYSTEM_States.md` — `DT_States`, `BP_StateApplier` (reutilizado sin modificar).
- `21_SYSTEM_DayNightCycle_Migration.md` — ciclo base, `Day` por ciclo, `Days Per Month` / `Months Per Year`.
- `15_SYSTEM_StatusEffects.md` — patrón de trigger mínimo reutilizable.
- `23_SYSTEM_ItemSpawner_Investigacion.md` — `Night Only Items`, mismo hook `CheckLight`.

---

*Archivo actualizado — sesión Light Paradox (DT_CelestialEvents v1).*
*Cambios: añadida implementación v1 (`E_CelestialEventType`, `STR_CelestialEvent` con `EventMonths`/`EventDays`, `DT_CelestialEvents` con filas de prueba, función `CheckCelestialEvents`, `Then 4`); reglas de calendario del cliente y contradicción registrada; pendientes de la siguiente sesión (medianoche, prioridad de eclipses); plan de pruebas con `Day` fijo; sección de pregunta de fases marcada como parcialmente superada.*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
