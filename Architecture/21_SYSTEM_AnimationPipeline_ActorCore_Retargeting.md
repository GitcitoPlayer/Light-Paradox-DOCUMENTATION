# 21 — Sistema: Technical Animation Pipeline · Retargeting ActorCore
### Sistema: Importación y Retargeting de Rigs de Personaje (placeholder ActorCore)
### Base Asset: EasySurvivalRPGv5
### Fuente: Sesión de trabajo directa en Unreal Editor — sesión Light Paradox (Animation Pipeline)
### Proyecto: Light Paradox · UE 5.4.4

---

## Contexto y objetivo de la sesión

Light Paradox reutiliza el esqueleto y el Animation Blueprint (`ABP_Jess`) de
EasySurvivalRPGv5 como base técnica. El objetivo de esta sesión fue definir y
probar el flujo correcto para reemplazar el rig placeholder del asset base por
un rig de personaje distinto — en este caso, un rig descargado de **ActorCore**
(Reallusion / Character Creator, convención de huesos `CC3`/`CC4`) — sin
reescribir el sistema de animación existente desde cero.

**Decisión de diseño confirmada por el cliente:** a diferencia del sistema
modular de cosméticos de ESRPGv5 (Body/Head/Pants/Hands/Feet/Backpack como
Skeletal Mesh Components independientes), Light Paradox se inspira en Genshin
Impact — los cambios de vestimenta serán **skins completos** (un solo Skeletal
Mesh por outfit), no combinaciones modulares por pieza. El sistema modular de
ESRPGv5 no se elimina en esta sesión, pero se trata como **deuda técnica a
remover** una vez que el sistema de outfits único esté definido.

**Inferencia de proyecto (no confirmada como decisión final, pero orienta el
enfoque técnico):** se asume que el pipeline de ActorCore (auto-rigging desde
un mesh escaneado o modelado, más el sistema de retargeting documentado aquí)
podría usarse para generar el rig del personaje final **Hana** de forma
automática, y que solo un subconjunto de animaciones clave sería reemplazado
manualmente por un animador — no la totalidad del set de animaciones. Esta
sesión con el placeholder de ActorCore sirve como prueba de concepto de ese
flujo, no como el pipeline definitivo de producción.

---

## 1 — Comparación de Skeletons involucrados

| | `SK_Jess` (base ESRPGv5) | `party-m-0001_Skeleton` (ActorCore placeholder) |
|---|---|---|
| Convención de nombres | Estilo UE Mannequin simplificado | CC3/CC4 (Reallusion) |
| Root / Retarget Root | `pelvis` | `pelvis` |
| Spine | `spine_01 → spine_02 → spine_03` (3 huesos) | `spine_01 → spine_02 → spine_03 → spine_04 → spine_05` (5 huesos) |
| Brazos | `clavicle_l → upperarm_l → lowerarm_l → hand_l` | Igual, más huesos twist (`upperarmtwist01/02`, `forearmtwist01/02`, `elbowsharebone`) |
| Piernas | `thigh_l → calf_l → foot_l → ball_l` (nombres coinciden) | `thigh_l → calf_l → foot_l → ball_l` + twist bones (`thightwist01/02`, `calftwist01/02`, `kneesharebone`) |
| Dedos | `index/middle/ring/pinky/thumb_01_l → 02_l → 03_l` | Igual, más un hueso `metacarpal_l` extra por dedo (excepto thumb) |
| Sockets custom | `socket_backpack` (en spine_03), `hand_l_magic` (en hand_l) | No aplica — sockets viven en SK_Jess, no se replican |

**Regla aplicada de forma consistente en todo el pipeline:** cualquier hueso
adicional que exista en ActorCore pero no en SK_Jess (twist bones,
kneesharebone, elbowsharebone, metacarpal, dedos de pie individuales) **se
excluye de las Retarget Chains**. No se les asigna chain — quedan en `None` y
el sistema no requiere resolverlos.

---

## 2 — Arquitectura de mesh confirmada en BP_Character_Player

`BP_Character_Player` usa un componente principal `Mesh (CharacterMesh0)` más
cinco componentes secundarios: `BodyMesh`, `HeadMesh`, `PantsMesh`,
`HandsMesh`, `FeetMesh` (y `BackpackMesh`).

**Confirmado por inspección directa del grafo:** existe un nodo
`Set Leader Pose Component` que asigna `Mesh (CharacterMesh0)` como líder de
los cinco sub-meshes. Esto significa:

- Los sub-meshes **no calculan su propia pose** — siguen la pose del
  componente `Mesh` automáticamente
- Solo es necesario retargetear animaciones para el skeleton usado por el
  componente `Mesh` principal — los sub-meshes no requieren su propio Anim
  Class ni su propio set de animaciones

> **Dato no explicado, pendiente de investigación futura:** el componente
> `Mesh (CharacterMesh0)` originalmente carga `SK_Jess_Accessory_001`, no
> `SK_Jess_Body_001`. Es inusual que el componente líder cargue el mesh de
> "accesorio" en vez del cuerpo principal. No se investigó el motivo en esta
> sesión.

---

## 3 — ABP_Jess: estructura relevante para el retargeting

`ABP_Jess` (Target Skeleton original: `SK_Jess`) no contiene nodos de
manipulación de huesos por nombre hardcodeado (no hay `Transform (Modify)
Bone`, `Two Bone IK`, `Copy Bone` visibles en el AnimGraph). Su lógica está
construida sobre estas 4 State Machines dentro del AnimGraph principal:

| State Machine | Rol | Estados confirmados |
|---|---|---|
| `MainStateMachine` | Locomoción y estados de cuerpo completo | `Idle/Move`, `Swimming`, `Jump_Cycle`, `Jump_End`, `Crouching` |
| `StanceStateMachine` | Slots de disparo/uso de arma por encima de la locomoción | (no detallado en esta sesión — pendiente inspección) |
| `AimOffsetStateMachine` | Aim Offsets direccionales (`NoOffset`, `WateringCan`, `Torch`, `Aiming`, `BowAiming`, `ProceduralOffsets`) | Ver notas de Aim Offset abajo |
| `DeathStateMachine` | Pose de muerte | `DeathPose` |

Estos se combinan vía `Layered blend per bone` y `Blend Poses by bool` para
producir el `Output Animation Pose` final. Los `Layered blend per bone`
usan **Branch Filters** con nombres de hueso — estos Branch Filters están
escritos contra `SK_Jess` y **no requieren modificación** mientras `SK_Jess`
siga siendo el skeleton de gameplay (ver Sección 7, Decisión arquitectural
pendiente).

**Hallazgo clave de esta sesión:** dentro de `MainStateMachine → Idle/Move`,
la locomoción no usa AnimSequence sueltas — usa **6 Blend Space 2D**
seleccionados según la variable de Stance/arma equipada:

- `BS_Jess_IdleRun_Common`
- `BS_Jess_IdleRun_Rifle`
- `BS_Jess_IdleRun_Pistol`
- `BS_Jess_IdleRun_Blunderbuss`
- `BS_Jess_IdleRun_Bow`
- `BS_Jess_IdleRun_Combat`

Cada uno es un **asset independiente con Skeleton propio** (igual que una
AnimSequence) — no son intercambiables entre skeletons sin recrearlos.

---

## 4 — Pipeline de Retargeting (paso a paso, tal como se ejecutó y validó)

### 4.1 — Creación de IK Rigs

Se crearon dos IK Rig, uno por Skeleton:

| Asset | Skeleton | Retarget Root |
|---|---|---|
| `IK_Jess` | `SK_Jess` | `pelvis` |
| `IK_ActorCore_PartyM0001` | `party-m-0001_Skeleton` | `pelvis` |

**Retarget Chains configuradas (mismos nombres en ambos IK Rig):**

| Chain Name | SK_Jess (Start → End) | ActorCore (Start → End) |
|---|---|---|
| `Spine` | `spine_01 → spine_03` | `spine_01 → spine_05` |
| `LeftArm` | `clavicle_l → hand_l` | `clavicle_l → hand_l` |
| `RightArm` | `clavicle_r → hand_r` | `clavicle_r → hand_r` |
| `LeftLeg` | `thigh_l → foot_l` | `thigh_l → foot_l` |
| `RightLeg` | `thigh_r → foot_r` | `thigh_r → foot_r` |
| `ThumbLeft` / `ThumbRight` | `thumb_01_l/r → thumb_03_l/r` | Igual |
| `IndexLeft` / `IndexRight` | `index_01_l/r → index_03_l/r` | Igual (excluye `index_metacarpal`) |
| `MiddleLeft` / `MiddleRight` | `middle_01_l/r → middle_03_l/r` | Igual (excluye `middle_metacarpal`) |
| `RingLeft` / `RingRight` | `ring_01_l/r → ring_03_l/r` | Igual (excluye `ring_metacarpal`) |
| `PinkyLeft` / `PinkyRight` | `pinky_01_l/r → pinky_03_l/r` | Igual (excluye `pinky_metacarpal`) |

Ningún IK Goal fue configurado (retargeting puramente FK, `Rotation Mode:
Interpolated`). El Solver Stack de ambos IK Rig quedó vacío — confirmado como
no bloqueante para este tipo de retargeting.

### 4.2 — Creación del IK Retargeter

**Corrección importante de dirección aplicada durante la sesión:** el
Retargeter se creó inicialmente con `Source = IK_ActorCore_PartyM0001` /
`Target = IK_Jess`, asumiendo que las animaciones vivían en el rig de
ActorCore. Esto era incorrecto — **las animaciones reales viven únicamente en
SK_Jess** (heredadas de ESRPGv5); ActorCore no trae ninguna animación propia.

**Configuración final correcta:**
- **Source IK Rig Asset:** `IK_Jess`
- **Target IK Rig Asset:** `IK_ActorCore_PartyM0001`
- Asset: `RTG_ActorCore_to_Jess` (nombre conservado pese a la corrección de
  dirección — considerar renombrar en una limpieza futura para evitar
  confusión, ver Sección 8)

### 4.3 — Corrección de Retarget Pose (T-pose vs A-pose)

**Síntoma observado:** al reproducir cualquier animación, los brazos del
personaje ActorCore quedaban muy pegados al cuerpo y cruzados, aunque la
animación original no lo hiciera.

**Causa confirmada:** `SK_Jess` está en **T-pose** (brazos horizontales) en su
pose de referencia; el rig de ActorCore está en **A-pose** (brazos caídos
~45°). El retargeting FK con `Rotation Mode: Interpolated` compara rotación
relativa contra la pose de referencia de cada skeleton — la diferencia de
~45° entre ambas poses de referencia se interpretaba como parte del
movimiento de la animación, acumulándose en cada frame.

**Solución aplicada:** dentro de `RTG_ActorCore_to_Jess`, tab **Target**, se
creó un **Retarget Pose** nuevo (vía `Current Retarget Pose → Create`) y se
rotaron manualmente los huesos `clavicle_l/r`, `upperarm_l/r`, `lowerarm_l/r`
del rig de ActorCore **dentro de esta pose de retargeting únicamente** —
sin modificar el Skeleton asset ni el mesh real — hasta igualar visualmente
la orientación de T-pose de `SK_Jess`.

**Resultado:** confirmado funcional. Brazos, piernas y dedos se retargetean
correctamente sin cruce ni deformación al reproducir animaciones dentro del
visor del Retargeter.

### 4.4 — Export de animaciones

Ya se exportaron **442 AnimSequence** desde `IK_Jess` (Source) hacia
`party-m-0001_Skeleton`, guardadas en:

```
Content/Characters/ActorCore_PartyM0001/Animations_Retargeted/
```

> Nota de alcance: no todas las 442 están necesariamente integradas en el ABP
> todavía — el ABP solo usa un subconjunto real de referencias dentro de sus
> 4 State Machines. Las 442 quedan disponibles como pool completo para
> integrarse a demanda.

---

## 5 — Reconstrucción del ABP para ActorCore

### 5.1 — Ruta descartada: Compatible Skeletons

Se evaluó usar el sistema de **Compatible Skeletons** de UE5 (declarar
`party-m-0001_Skeleton` como compatible de `SK_Jess` para poder usar
`ABP_Jess_C` sin duplicar nada). **Se probó y se descartó.**

**Resultado de la prueba:** el mesh de ActorCore se veía deforme al animarse
(especialmente en brazos/antebrazos, zona de los huesos twist). La
documentación oficial de Epic confirma que Compatible Skeletons requiere
estructuras de jerarquía casi idénticas y proporciones de mesh similares —
condición que SK_Jess y ActorCore no cumplen (distinto número de huesos de
columna, huesos twist adicionales, proporciones de cuerpo distintas). Para
personajes con proporciones distintas la vía correcta es Animation
Retargeting; para estructuras de skeleton radicalmente distintas, es IK Rig
Retargeting — que es exactamente lo ya construido en la Sección 4.

**Se verificó y confirmó** que el array `Compatible Skeletons` en el asset
`SK_Jess_Skeleton` quedó en 0 elementos — nunca se llegó a activar de forma
persistente, así que no hay residuo de esta ruta que limpiar.

### 5.2 — Ruta aplicada: ABP duplicado con Target Skeleton propio

1. Se duplicó `ABP_Jess_C` → `ABP_ActorCore_PartyM0001`
2. Se cambió su **Target Skeleton** (Class Settings) a `party-m-0001_Skeleton`
3. **Hallazgo importante:** cambiar el Target Skeleton **no genera errores de
   compilación automáticos** en los nodos que siguen referenciando assets del
   skeleton viejo (AnimSequence o Blend Space de `SK_Jess`). El compilador no
   re-valida activamente cada referencia existente contra el nuevo Target
   Skeleton — el error solo aparece si se intenta asignar un asset
   incompatible nuevo desde el dropdown, o si la referencia se pierde por
   completo. **Esto implica que no hay atajo de detección automática de qué
   nodos faltan reemplazar — se revisan manualmente, State Machine por State
   Machine, estado por estado.**

### 5.3 — Reemplazo de AnimSequence sueltas

Para nodos simples tipo Sequence Player: seleccionar el nodo, campo
`Sequence`, reemplazar por el equivalente en
`Animations_Retargeted/`.

### 5.4 — Reemplazo de Blend Spaces (proceso más largo, ya iniciado)

Un Blend Space es un asset con Skeleton propio — no acepta simplemente
cambiar la animación referenciada, **hay que recrear el asset completo**
contra el nuevo Skeleton.

**Proceso confirmado y ejecutado para el primero de los 6:**

1. Inspeccionar el Blend Space original (`BS_Jess_IdleRun_Common`): tipo
   (2D), Axis Settings, y los 13 samples (animación + Direction + Speed +
   Rate Scale de cada uno)
2. Crear un Blend Space nuevo contra `party-m-0001_Skeleton`
   (`BS_ActorCore_IdleRun_Common`)
3. Configurar los mismos Axis Settings
4. Rellenar cada sample con la animación retargeteada equivalente, mismos
   valores de Direction/Speed/Rate Scale
5. Verificar visualmente (Ctrl+clic en distintos puntos del grid dentro del
   editor del Blend Space)
6. Reemplazar el nodo `BS_Jess_IdleRun_Common` por
   `BS_ActorCore_IdleRun_Common` dentro de `Idle/Move (state)`
7. Compilar, y probar en Play/Simulation sobre `BP_Character_Player`

**Estructura confirmada de `BS_Jess_IdleRun_Common` (documentada aquí para
referencia futura y para replicar en los 5 restantes):**

- Tipo: Blend Space 2D
- Horizontal Axis: `Direction`, rango -180.0 a 180.0, Grid Divisions 4
- Vertical Axis: `Speed`, rango 0.0 a 600.0, Grid Divisions 4
- 13 samples:

| # | Animation | Direction | Speed | Rate Scale |
|---|---|---|---|---|
| 0 | `A_Jess_Idle` | 0.0 | 0.0 | 1.0 |
| 1 | `A_Jess_Jog_F` | 0.0 | 450.0 | 1.0 |
| 2 | `A_Jess_Jog_L` | -90.0 | 450.0 | 1.0 |
| 3 | `A_Jess_Jog_R` | 90.0 | 450.0 | 1.0 |
| 4 | `A_Jess_Jog_B` | 180.0 | 450.0 | 1.0 |
| 5 | `A_Jess_Jog_B` | -180.0 | 450.0 | 1.0 |
| 6 | `A_Jess_Sprint` | 0.0 | 600.0 | 1.0 |
| 7 | `A_Jess_Idle` | 90.0 | 0.0 | 1.0 |
| 8 | `A_Jess_Idle` | 180.0 | 0.0 | 1.0 |
| 9 | `A_Jess_Idle` | -90.0 | 0.0 | 1.0 |
| 10 | `A_Jess_Idle` | -180.0 | 0.0 | 1.0 |
| 11 | `A_Jess_Jog_L` | -90.0 | 600.0 | **1.5** |
| 12 | `A_Jess_Jog_R` | 90.0 | 600.0 | **1.5** |

> Nota: los samples en Speed=600 lateral (11, 12) reutilizan las animaciones
> de Jog acelerado (Rate Scale 1.5) en vez de tener una animación de Sprint
> lateral dedicada. Solo el frente (sample 6, Direction=0) usa
> `A_Jess_Sprint` como animación propia. Este patrón probablemente se repita
> en los otros 5 Blend Spaces — a confirmar al inspeccionarlos.

**Resultado de esta prueba (punto donde se detiene la sesión):** confirmado
funcional. El personaje con mesh de ActorCore se mueve en Idle/Jog/Sprint
dentro de `BP_Character_Player` **sin deformación visible**, usando el Blend
Space reconstruido para el stance `Common`.

---

## 6 — Easy Fixes aplicados en esta sesión

### 6.1 — Doble mesh visible (Jess + ActorCore superpuestos)

**Síntoma:** al dar Play, se ven dos mallas superpuestas — la de ActorCore
(en el componente `Mesh`, ya reemplazado) y la de Jess (en los sub-meshes
`BodyMesh`, `HeadMesh`, `PantsMesh`, `HandsMesh`, `FeetMesh`, `BackpackMesh`,
que conservan su Skeletal Mesh original de ESRPGv5 aunque ya no se necesiten
al no usar el sistema modular).

**Fix aplicado (documentado como temporal, ver deuda técnica):**
desactivar la propiedad **Visible** (no `Hidden in Game`) en cada uno de los
6 componentes sub-mesh, desde el panel Details de `BP_Character_Player`. No
se modifica el Skeletal Mesh asignado, no se toca el Leader Pose Component,
no se eliminan los componentes.

**Por qué se eligió esta solución y no otra:** mínimo cambio posible,
totalmente reversible (basta con volver a marcar `Visible = true`), no
afecta ningún otro sistema (colisiones, sockets, Leader Pose Component). Es
la vía correcta según el principio de priorizar modificaciones mínimas antes
que reestructuras grandes.

---

## 7 — Deuda técnica registrada de esta sesión

| Problema | Notas | Prioridad | Estado |
|---|---|---|---|
| Sub-meshes modulares (`BodyMesh`, `HeadMesh`, `PantsMesh`, `HandsMesh`, `FeetMesh`, `BackpackMesh`) siguen existiendo en `BP_Character_Player` | Solo ocultos vía `Visible = false`. Deben eliminarse por completo cuando se defina el sistema de outfits únicos (visión Genshin) | 🟡 Media | Fix temporal aplicado — pendiente decisión de remoción definitiva |
| `RTG_ActorCore_to_Jess` conserva un nombre que sugiere la dirección incorrecta (Source/Target ya están corregidos, pero el nombre del asset no) | Renombrar a algo como `RTG_Jess_to_ActorCore` para evitar confusión futura | 🟢 Baja | Pendiente — cosmético, no bloquea funcionalidad |
| 5 de 6 Blend Spaces de `Idle/Move` (`Rifle`, `Pistol`, `Blunderbuss`, `Bow`, `Combat`) aún no reconstruidos para ActorCore | Mismo proceso que `Common`, documentado en Sección 5.4. Se asume misma estructura de ejes (Direction -180/180, Speed 0/600, Grid Divisions 4) — a confirmar por cada uno | 🔴 Alta | Pendiente — siguiente sesión |
| `StanceStateMachine`, `DeathStateMachine`, `AimOffsetStateMachine` sin revisar contra ActorCore | Solo se completó `MainStateMachine → Idle/Move`. Faltan: Shooting (Stance), Death Pose, y los 6 Aim Offsets (`NoOffset`, `WateringCan`, `Torch`, `Aiming`, `BowAiming`, `ProceduralOffsets`) | 🔴 Alta | Pendiente |
| Aim Offsets probablemente requieren el mismo proceso que Blend Spaces (asset con Skeleton propio) | Un Aim Offset es un tipo de asset distinto a AnimSequence normal — no confirmado si acepta el mismo flujo de "crear nuevo + rellenar" o si tiene particularidades propias | 🔴 Alta | Pendiente inspección |
| Preview Mesh del ABP no detecta el Skeletal Mesh de ActorCore en el picker | Posible desajuste de Skeleton asset entre el mesh y el Target Skeleton del ABP (dos assets Skeleton distintos con nombre similar), o filtro de UI no refrescado. Workaround sugerido: drag-and-drop directo desde Content Drawer en vez de usar el buscador | 🟡 Media | Pendiente verificación — no bloquea Play, solo cosmético en el editor del ABP |
| Componente `Mesh (CharacterMesh0)` originalmente carga `SK_Jess_Accessory_001` en vez de un mesh de cuerpo principal | Sin explicación confirmada — inusual que el componente líder cargue el "accesorio" | 🟢 Baja | Pendiente investigación, no bloqueante |
| `STR_ItemData` y sistema de Rune Binding (ver `16_SYSTEM_RuneBinding_WeaponCosmetic.md`) no evaluado en el contexto de este nuevo pipeline de animación | Fuera de alcance de esta sesión — mencionado aquí solo para no perder el cruce entre sistemas | 🟢 Baja | Sin relación directa confirmada aún |

---

## 8 — Plan de continuación sugerido (próxima sesión)

```
Fase A — Completar Idle/Move (locomoción base)
  → Reconstruir los 5 Blend Spaces restantes (Rifle, Pistol, Blunderbuss, Bow, Combat)
  → Mismo proceso documentado en Sección 5.4
  → Verificar cada stance en Play antes de continuar

Fase B — StanceStateMachine (Shooting / uso de arma)
  → Inspeccionar estructura (no se llegó a revisar en esta sesión)
  → Aplicar mismo criterio: Sequence Player simple → reemplazo directo;
    Blend Space → reconstrucción completa

Fase C — DeathStateMachine
  → Solo un estado (DeathPose) — probablemente una sola AnimSequence,
    reemplazo directo esperado

Fase D — AimOffsetStateMachine
  → Investigar si un Aim Offset asset acepta el mismo flujo de reconstrucción
    manual, o si tiene un proceso distinto en el editor de UE5.4.4
  → 6 estados a resolver: NoOffset, WateringCan, Torch, Aiming, BowAiming,
    ProceduralOffsets

Fase E — Limpieza de deuda técnica
  → Decidir si se eliminan por completo los sub-mesh components modulares
    o se conservan ocultos indefinidamente
  → Renombrar el Retargeter si se desea mayor claridad
  → Resolver el picker de Preview Mesh del ABP (cosmético)

Fase F — Evaluación para personaje final (Hana)
  → Confirmar si el pipeline de ActorCore (auto-rig) es viable para el
    modelo final de Hana
  → Definir con el animador cuáles animaciones del set completo requieren
    reemplazo manual dedicado vs cuáles pueden quedarse retargeteadas
    automáticamente sin pérdida de calidad perceptible
```

---

## 9 — Punto exacto donde se detiene esta sesión

**Confirmado y funcional:**
- IK Rig de ambos skeletons (`IK_Jess`, `IK_ActorCore_PartyM0001`) con chains
  completas: spine, brazos, piernas, y los 10 chains de dedos
- IK Retargeter (`RTG_ActorCore_to_Jess`) con dirección correcta
  (Source = Jess, Target = ActorCore) y Retarget Pose corregido (fix de
  T-pose vs A-pose)
- 442 animaciones exportadas a
  `Content/Characters/ActorCore_PartyM0001/Animations_Retargeted/`
- `ABP_ActorCore_PartyM0001` creado con Target Skeleton correcto
- `BS_ActorCore_IdleRun_Common` reconstruido y verificado
- `BP_Character_Player` con `Mesh (CharacterMesh0)` apuntando al Skeletal
  Mesh de ActorCore, `Anim Class = ABP_ActorCore_PartyM0001`
- Doble mesh resuelto vía easy fix de visibilidad
- **Prueba en Play: personaje se mueve en Idle/Jog/Sprint sin deformación,
  usando mesh completo de ActorCore** ← logro de la sesión, confirmado

**No completado — continúa en la siguiente sesión:**
- 5 Blend Spaces restantes de `Idle/Move`
- `StanceStateMachine`, `DeathStateMachine`, `AimOffsetStateMachine` completos
- Limpieza de deuda técnica listada en Sección 7

---

*Archivo creado — sesión Light Paradox (Technical Animation Pipeline · ActorCore Retargeting)*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
