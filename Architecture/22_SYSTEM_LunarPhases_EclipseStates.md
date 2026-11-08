# 22 — Sistema: Fases Lunares y Eclipses vinculadas a Estados
### Sistema: Correlación Ciclo Día/Noche → Eventos cósmicos → Estados / Spawn (integración con States y con `BP_ResourceSpawnPoint`)
### Base Asset: EasySurvivalRPGv5 (Day/Night Module) + sistema propio de States (Light Paradox)
### Fuente: Diseño del cliente + documento original del programador (`Day_Night_Module.pdf`) + implementación en sesión Light Paradox
### Proyecto: Light Paradox · UE 5.4.4
### Actualizado: sesión Light Paradox — `DT_CelestialEvents` v1.1 (eventos que cruzan medianoche) + `BPI_CelestialListener` / `NotifyCelestialEvent` (Etapa 1A del spawn por eventos)

---

## Contexto

`BP_DayNight` (ver `21_SYSTEM_DayNightCycle_Migration.md`) produce un ciclo
día/noche funcional y calibrado. Objetivo: conectar fases lunares y eclipses
al sistema de Estados (`20_SYSTEM_States.md`) y al spawn de ítems/enemigos
(`24_SYSTEM_ResourceSpawnPoint_Plan.md`) para que ciertos eventos de
gameplay ocurran **exclusivamente** mientras una fase específica está en
curso.

**Estado actual (esta sesión):**
- ✅ Prueba de concepto Poison de noche (resuelta, ver sección más abajo).
- ✅ `DT_CelestialEvents` v1: detecta eventos por **hora + día del mes + mes** e imprime ENTER/EXIT en el debug log.
- ✅ **Nuevo — v1.1:** los eventos que **cruzan medianoche** (`StartHour > EndHour`, ej. 18 → 6) funcionan. Confirmado por el usuario en juego: la luna entra el día 4 a las 18:00 y sale el día 5 a las 6:00, y el cambio de día se verificó en los Details de la instancia de `BP_DayNight`.
- ✅ **Nuevo — Etapa 1A:** `BPI_CelestialListener` + `NotifyCelestialEvent` avisan a cualquier actor suscrito cuando un evento empieza o termina. Confirmado por log: `[Notify] Moon_Gibbous | Listeners: 0` justo después de cada ENTER y EXIT (0 listeners porque todavía no existe ningún spawn point).
- 🔴 Pendientes de esta fase: prioridad de eclipses sobre lunas, aplicar Estados a jugador y enemigos, alimentar el visual (`MoonPhase`, `Eclipse Phase`). Ver "Pendientes".

---

## Diseño objetivo (confirmado por el cliente)

- Fases lunares vinculadas a eventos de gameplay.
- Fases de eclipse (solar y/o lunar) vinculadas a eventos de gameplay.
- Cada fase se vincula a un Estado (`DT_States` / `BP_StateApplier`) que se
  aplica **solo** mientras esa fase está en curso y se remueve al terminar.
- Cada fase puede además hacer **aparecer ítems o enemigos** mientras dura (ver doc 24).
- Los efectos lunares afectan a jugador y enemigos por igual (paráfrasis del
  PDF original), por lo que el trigger no debe limitarse al jugador. Pendiente
  de implementar (ver pendientes).

### Reglas del calendario dadas por el cliente

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
> solares 6 y 12. **Inferencia, no confirmada.**

---

## ✅ Implementación v1.1 — `DT_CelestialEvents`

### Diseño

Se usan `EventMonths` y `EventDays` (en vez de `IntervalDays`/`DayOffset`)
por ser más intuitivas para diseño. `Day` (contador corrido) se convierte a
mes y día del mes con `Days Per Month` y `Months Per Year` (variables de
`BP_DayNight`, ver `21`).

### Assets (en `Content/LightParadox/Blueprints/DayNightCycle`)

**Enum `E_CelestialEventType`:** `MoonPhase`, `SolarEclipse`, `LunarEclipse`.

**Struct `STR_CelestialEvent` (Row Struct):**

| Campo | Tipo | Notas |
|---|---|---|
| `EventType` | `E_CelestialEventType` | Tipo de evento |
| `StartHour` | Float | Hora de inicio (0–24) |
| `EndHour` | Float | Hora de fin (0–24). **v1.1: puede ser menor que `StartHour` (cruza medianoche)** |
| `EventMonths` | Array de Integer | Meses en que ocurre. **Vacío o que contenga `0` = todos los meses** |
| `EventDays` | Array de Integer | Días del mes en que ocurre. **Vacío o que contenga `0` = todos los días** |

**DataTable `DT_CelestialEvents`** — filas de prueba (✅ `EndHour` corregido esta sesión):

| Row Name | EventType | StartHour | EndHour | EventMonths | EventDays |
|---|---|---|---|---|---|
| `Moon_Gibbous` | MoonPhase | 18 | **6** | (vacío) | 1, 2, 3, 4 |
| `Moon_Full` | MoonPhase | 18 | **6** | (vacío) | 5 |
| `Eclipse_Lunar` | LunarEclipse | 18 | **6** | 3, 9 | 5 |
| `Eclipse_Solar` (opcional) | SolarEclipse | **10** | **14** | 6, 12 | 20 |

> `Eclipse_Solar` con horario diurno es **inferencia de diseño** (el sol debe
> estar visible). ⏳ Confirmar si la fila ya existe en el DataTable y con qué horas.
>
> Con el calendario actual (`Sunrise 6` / `Sunset 18`, doc 21) `EndHour = 6`
> coincide con el amanecer: la luna se "apaga" cuando sale el sol.

### Variables en `BP_DayNight` (ver `21` para la tabla completa)

`Days Per Month` (Integer, 30), `Months Per Year` (Integer, 12),
`Active Celestial Events` (Array de Name, variable real).

### Regla de pertenencia al día (v1.1)

Un evento pertenece al **día en que empieza**. Si el evento cruza medianoche
y la hora actual está en la parte de madrugada (`Time < EndHour`), `Day` ya
subió, así que el mes y el día del mes se calculan con **`Day − 1`**
(`LocalEffectiveDay`). Si no, la luna del día 5 buscaría el día 6 y se
apagaría a las 0:00 (síntoma original: EXIT a las `0.02`).

### Función `CheckCelestialEvents` (BP_DayNight, sin inputs) — flujo v1.1

> **No se revisó el export de esta función en la sesión**: el flujo se
> reconstruyó de la documentación de v1 + las instrucciones aplicadas.
> ⏳ Exportar `BP_DayNight CheckCelestialEvents Function.txt` para validar.

Variables locales: `LocalDayOfMonth` (Integer), `LocalMonth` (Integer),
✅ **nueva** `LocalEffectiveDay` (Integer).

```
Entry
  → For Each Loop (Array: Get Data Table Row Names(DT_CelestialEvents))
      Loop Body → Get Data Table Row (DT_CelestialEvents, Row Name: Array Element)
        Row Found → Break STR_CelestialEvent

          ┌─ [Nodo colapsado propuesto: ResolveEventTiming — ver sección abajo] ─┐
          │ CrossesMidnight = StartHour > EndHour
          │ EarlyPart       = CrossesMidnight AND (Time < EndHour)
          │ SET LocalEffectiveDay ← Max( Day − Select(EarlyPart: 0/1) , 1 )
          │ SET LocalDayOfMonth   ← ((LocalEffectiveDay - 1) % Max(Days Per Month, 1)) + 1
          │ SET LocalMonth        ← (((LocalEffectiveDay - 1) / Max(Days Per Month, 1)) % Max(Months Per Year, 1)) + 1
          │ HourMatch = Select(CrossesMidnight):
          │               False → In Range (Time, StartHour, EndHour)   [Inclusive Min true, Inclusive Max false]
          │               True  → OR( Time >= StartHour , Time < EndHour )
          └──────────────────────────────────────────────────────────────────────┘

          MonthMatch = OR( Length(EventMonths) == 0, Contains(EventMonths, 0), Contains(EventMonths, LocalMonth) )
          DayMatch   = OR( Length(EventDays) == 0,   Contains(EventDays, 0),   Contains(EventDays, LocalDayOfMonth) )
          IsActive      = AND(HourMatch, MonthMatch, DayMatch)
          AlreadyActive = Contains(Active Celestial Events, Array Element)
          Branch (IsActive)
            True  → Branch (NOT AlreadyActive)
                      True → ADD Active Celestial Events ← Array Element
                           → Print String "[Celestial] ENTER: <row> | Month <m> Day <d> | Time <t>" (verde)
                           → ✅ Notify Celestial Event (Target: self, EventName ← Array Element, bStarted = True)
            False → Branch (AlreadyActive)
                      True → Remove Item Active Celestial Events ← Array Element
                           → Print String "[Celestial] EXIT: <row> | Month ... " (rojo)
                           → ✅ Notify Celestial Event (Target: self, EventName ← Array Element, bStarted = False)
```

> **Cambio estructural v1.1:** `SET LocalDayOfMonth` y `SET LocalMonth` pasaron
> de estar **antes** del loop a estar **dentro** de él (cada fila puede
> necesitar un `Day` distinto).
>
> **Nota del log:** el `Print String` de ENTER/EXIT ya no mostró `Day`/`Month`
> en la sesión de validación (el usuario reportó que no ve un log con los días;
> pudo desconectarse al reorganizar). Si se quiere ver el día efectivo en el
> log, conectar `GET LocalDayOfMonth` y `GET LocalMonth` (variables, no texto
> literal en el pin de `Append`).

### Nodo colapsado propuesto — `ResolveEventTiming` (⏳ pendiente de aplicar)

Motivo: la matemática del cruce de medianoche saturó la función. Se propone
un único **Collapsed Graph** (Collapse Nodes, no Function ni Macro) con este
alcance:

- **Desde:** el primer nodo de la lógica de tiempo (`>` Float `StartHour > EndHour`).
- **Hasta:** el `Select` Boolean que produce `HourMatch`.
- **Incluye:** `>`, `<`, `AND` (EarlyPart), `Select` Integer 0/1, `-`, `Max`,
  `SET LocalEffectiveDay`, `SET LocalDayOfMonth`, `SET LocalMonth`, y el
  bloque de `HourMatch` (`In Range`, `>=`, `<`, `OR`, `Select` Boolean).
- **No incluye:** `Get Data Table Row`, `Break STR_CelestialEvent`,
  `MonthMatch`, `DayMatch`, `IsActive`, `AlreadyActive`, los `Branch` y los
  `Print String`/`Notify`. Esos son la lógica "legible" de la función.
- **Pines esperados** (se crean solos al colapsar): Exec in/out; entradas
  `StartHour`, `EndHour`, `Time`, `Day`, `Days Per Month`, `Months Per Year`;
  salida `HourMatch`.

Ver instrucciones paso a paso en la respuesta de la sesión. Al aplicarlo,
registrar aquí que el nodo existe y confirmar que `HourMatch` sigue conectado
al `AND` de `IsActive`.

### Pruebas (con `Day` fijo en la instancia — `Day` es Instance Editable)

| Prueba | Setup | Esperado v1.1 |
|---|---|---|
| A — luna a través de la noche | `Day = 4`, `Start Time = 17` | 18:00 día 4: ENTER `Moon_Gibbous` + `[Notify]`. **Sin** EXIT a las 0:00. 6:00 día 5: EXIT + `[Notify]`. 18:00 día 5: ENTER `Moon_Full`; 6:00 día 6: EXIT |
| B — eclipse mes 3 | `Day = 64` (mes 3, día 4), `Start Time = 17` | día 65 ~18: ENTER `Moon_Full` **y** `Eclipse_Lunar` (dos eventos a la vez: pendiente de prioridad) |

Resultado reportado por el usuario (Prueba A): ENTER a las 18:00 y EXIT a las
~6:00 con la luna entrando y saliendo del cielo, con el cambio de día entre 4 y 5
verificado en Details de la instancia. Prueba B: ⏳ no reportada.

### Notificación a suscriptores — Etapa 1A (✅ implementada y confirmada por log)

**Blueprint Interface `BPI_CelestialListener`** (`Content/LightParadox/Blueprints/DayNightCycle`):

| Función | Input |
|---|---|
| `OnCelestialEventStarted` | `EventName` (Name) |
| `OnCelestialEventEnded` | `EventName` (Name) |

**Función `NotifyCelestialEvent` (BP_DayNight)** — Inputs: `EventName` (Name), `bStarted` (Boolean):

```
Function Entry
  → Get All Actors with Interface (Interface: BPI_CelestialListener)
  → [Print String debug temporal: "[Notify] <EventName> | Listeners: <Length(Out Actors)>"]
  → For Each Loop (Array: Out Actors)
      Loop Body → Branch (Condition ← bStarted)
          True  → On Celestial Event Started (Message) — Target: Array Element, EventName ← EventName
          False → On Celestial Event Ended   (Message) — Target: Array Element, EventName ← EventName
```

Se llama **solo** en transiciones ENTER/EXIT desde `CheckCelestialEvents`
(no por tick): costo despreciable. Es el mismo patrón que `BPI_IsLight`.

Log confirmado:
```
[Celestial] ENTER:Moon_Gibbous| Month1Day4| Time18.020229
[Notify] Moon_Gibbous| Listeners: 0
[Celestial] EXIT:Moon_Gibbous| Month1Day5| Time0.020366     ← antes del fix v1.1 (EndHour 24)
[Notify] Moon_Gibbous| Listeners: 0
```

> **Lección de diagnóstico:** `EndHour = 24` hacía EXIT a las `0.02` porque
> `Time` vuelve a 0; `EndHour = 6` sin lógica de cruce nunca activa el evento
> (`In Range` con `Min > Max`). Hacen falta ambos: el dato **y** la comprobación.

### Limitaciones conocidas

- Dos eventos pueden estar activos a la vez (eclipse + luna). Sin prioridad.
- Granularidad: corre en el timer de `CheckLight`; entradas/salidas pueden
  retrasarse hasta un tick. Duración real del ciclo no medida.
- Solo imprime log y notifica; no aplica Estados ni cambia visual.
- No hay validación de datos (`EventDays` fuera de 1–`Days Per Month`, `StartHour == EndHour`).
- En `Day = 1` con `Time < EndHour` el `Max(…, 1)` busca los eventos del día 1 como si hubieran empezado ayer. Inofensivo con `Start Time = 8`.
- Se observó un log `Es de Noche` impreso durante toda la noche. No está en la documentación de `CheckLight`; **inferencia:** es un `Print String` de debug de la rama Poison (`Then 1`). ⏳ Verificar y limpiar.

---

## Pendientes para la siguiente sesión

### 1. Prioridad: el eclipse reemplaza a la luna

| Opción | Cómo funciona | Cambio |
|---|---|---|
| **A. Por tipo (recomendada)** | Si hay algún evento activo de tipo `SolarEclipse` o `LunarEclipse`, las filas `MoonPhase` se consideran inactivas | Dos pasadas por la tabla y variable local `EclipseActive`. No cambia el struct |
| **B. Campo `Priority`** | Cada fila tiene `Priority` (Integer). Gana el mayor entre los activos | Campo nuevo en el struct. Escala mejor si hay más tipos |

Con A, el flujo ENTER/EXIT existente se encarga solo. **Decisión entre A y B: pendiente del usuario.**
No bloquea el spawn por eventos: `BP_ResourceSpawnPoint` escucha por nombre de fila (`Eclipse_Lunar`), así que `Moon_Full` activo al mismo tiempo no interfiere. Sí importa para Estados/visual.

### 2. Otros pendientes de esta fase

| Pendiente | Notas |
|---|---|
| Aplicar el nodo colapsado `ResolveEventTiming` | Ver sección arriba |
| Exportar `CheckCelestialEvents` y validar el flujo v1.1 | No se revisó el export |
| Confirmar con el cliente la contradicción de reglas de eclipse | Ver Diseño objetivo |
| Confirmar fila `Eclipse_Solar` con horario diurno | Ver tabla |
| Aplicar Estados a jugador **y** enemigos | Iterar actores con `BPI_Character` (patrón `Get All Actors with Interface`) o un manager central. Se podría suscribir vía `BPI_CelestialListener`. Sin implementar |
| Guardar/remover el Effect Handle al salir del evento | Reutilizar `Remove Status Effect By Handle` (ver `20_SYSTEM_States.md`, Bloque A) |
| Alimentar el visual (`MoonPhase`, `Eclipse Phase`) desde el evento | Sigue sin saberse cómo escribir en `MI_SkySpherePhases` |
| Campo `StateRowHandle` en `STR_CelestialEvent` | Para vincular fila de `DT_States` |
| Validación de datos de la tabla | Días/meses fuera de rango, `StartHour == EndHour` |
| Quitar el `Print String` debug de `NotifyCelestialEvent` | Cuando el spawn point esté validado |

> **Nota de diseño (decisión del usuario/cliente):** con la tabla por calendario,
> el que manda es el calendario y el visual del shader se alimentará **desde** el evento.

---

## Contexto histórico — pregunta abierta sobre número de fases

> **Parcialmente superada** por el diseño por calendario; se conserva como
> referencia para la fase visual.

Del PDF original (tablas completas en `21`): la tabla "Moon Phases" muestra 3
ejemplos de `MoonPhase` (0.5 / 0.72 / 0.088) y la de eclipse 3 ejemplos de
`EclipsePhase` (0.5 / 0.25 / 0.01) donde también varía `SunScale`.
**Inferencia (no confirmada):** ambos son `Scalar Parameter Value` continuos.

---

## Prueba de concepto — Poison de noche (✅ resuelta en sesión anterior)

**Decisión de scope:** se aplicó `State_Poison` mientras es de noche, usando el
booleano "es de noche" que `BP_DayNight` ya calcula en `CheckLight`.

```
CheckLight (exec)
  → Sequence
      Then 0 → [luces]
      Then 1 → Branch (Condition ← NOT(In Range(Time, Sunrise Time, Sunset Time)))
                  True → Get Player Character (0) → Cast To BP_Character_Base
                           Cast Success → Make DataTableRowHandle (DT_States, State_Poison)
                                        → Apply State (Target: As BP_Character_Base)
      Then 2 → [Night Only Items — ver 23]
      Then 3 → [SkyLight día/noche — ver 21]
      Then 4 → [CheckCelestialEvents]
```

**Bug resuelto:** el Poison nunca se removía. Causa: `Sunrise Time` / `Sunset Time` en `0.0`. Fix: expuestas + `6.0` / `18.0`.
**Lección:** un `Print String` antes de un `Branch` no confirma en qué rama entra la ejecución; debe ir **después**, en el pin específico.
**Pendiente heredado:** el Poison de noche solo se aplica al jugador y no se remueve explícitamente al amanecer más allá del Duration del efecto.

---

## Deuda técnica / pendientes registrados

| Problema | Notas | Estado |
|---|---|---|
| Eventos que cruzan medianoche | ✅ Resuelto v1.1 (dato `EndHour = 6` + `LocalEffectiveDay` + `HourMatch` con `OR`) | ✅ Resuelto |
| Nodo colapsado `ResolveEventTiming` | Propuesto, sin aplicar | ⏳ Pendiente |
| Prioridad eclipse sobre luna | Opción A o B | 🔴 Pendiente decisión |
| Contradicción de reglas de eclipse del cliente | Ver Diseño objetivo | 🟡 Confirmar con el cliente |
| Aplicar Estado a jugador Y enemigos | Posible vía `BPI_CelestialListener` | 🔴 Pendiente diseño |
| Remover Estado al terminar el evento | Guardar Effect Handles aplicados por evento | 🔴 Pendiente |
| Alimentar visual (`MoonPhase`, `Eclipse Phase`) | Sin diseño | 🟡 Pendiente |
| Duración real del ciclo no medida | Afecta granularidad del log | ⏳ Pendiente medir |
| Log `Es de Noche` sin documentar | Posible `Print String` de debug en Poison | ⏳ Verificar y limpiar |
| Print String debug en `NotifyCelestialEvent` | Temporal | ⏳ Limpieza pospuesta |

---

## Ver también

- `20_SYSTEM_States.md` — `DT_States`, `BP_StateApplier`.
- `21_SYSTEM_DayNightCycle_Migration.md` — ciclo base, `Day` por ciclo, `Days Per Month` / `Months Per Year`.
- `23_SYSTEM_ItemSpawner_Investigacion.md` — `Night Only Items` (placeholder), hallazgos de spawn.
- `24_SYSTEM_ResourceSpawnPoint_Plan.md` — `BP_ResourceSpawnPoint`, consumidor de `BPI_CelestialListener`.

---

*Archivo actualizado — sesión Light Paradox (v1.1 cruce de medianoche + Etapa 1A notificación).*
*Cambios: `EndHour` de las filas de prueba corregido a 6; lógica de cruce de medianoche con `LocalEffectiveDay`; `SET LocalDayOfMonth/LocalMonth` movidos dentro del loop; `BPI_CelestialListener` y `NotifyCelestialEvent` documentados; llamadas a `Notify Celestial Event` en ENTER/EXIT; nodo colapsado `ResolveEventTiming` propuesto; pendiente 1 (medianoche) cerrado; log `Es de Noche` registrado.*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
