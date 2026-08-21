# 21 — Sistema: Technical Animation Pipeline · Retargeting de Rigs de Personaje
### Sistema: Importación y Retargeting de Rigs (placeholders ActorCore y Advance Skeleton)
### Base Asset: EasySurvivalRPGv5
### Fuente: Sesión de trabajo directa en Unreal Editor — sesión Light Paradox (Animation Pipeline)
### Proyecto: Light Paradox · UE 5.4.4

---

## Contexto y objetivo de la sesión

Light Paradox reutiliza el esqueleto y el Animation Blueprint (`ABP_Jess`) de
EasySurvivalRPGv5 como base técnica. El objetivo de esta documentación es
definir el flujo correcto para reemplazar el rig placeholder del asset base
por un rig de personaje distinto, sin reescribir el sistema de animación
existente desde cero.

Se probaron **dos rutas** con dos placeholders distintos:

1. **ActorCore** (Reallusion / Character Creator, convención CC3/CC4) —
   pipeline **manual completo**, documentado en la Sección A
2. **Advance Skeleton** (addon de rigging de Maya) — pipeline **mayormente
   automatizado** usando herramientas nativas de UE5.4 (Auto Retarget Chains,
   Duplicate and Retarget Animation Blueprint), documentado en la Sección B

**Decisión de diseño confirmada por el cliente:** a diferencia del sistema
modular de cosméticos de ESRPGv5 (Body/Head/Pants/Hands/Feet/Backpack como
Skeletal Mesh Components independientes), Light Paradox se inspira en Genshin
Impact — los cambios de vestimenta serán **skins completos** (un solo
Skeletal Mesh por outfit), no combinaciones modulares por pieza. El sistema
modular de ESRPGv5 no se elimina en esta sesión, pero se trata como **deuda
técnica a remover**.

**Inferencia de proyecto (no confirmada como decisión final, pero orienta el
enfoque técnico):** se asume que un addon de auto-rigging (Advance Skeleton
u otro similar) podría usarse para generar el rig del personaje final
**Hana** de forma automática, reemplazando solo un subconjunto de
animaciones clave con trabajo de animador dedicado — no la totalidad del set.
Ambos experimentos de esta sesión sirven como prueba de concepto de ese
flujo, no como el pipeline definitivo de producción.

---

## Sección A — Pipeline manual (ActorCore)

### A.1 — Comparación de Skeletons

| | `SK_Jess` (base ESRPGv5) | `party-m-0001_Skeleton` (ActorCore) |
|---|---|---|
| Convención de nombres | Estilo UE Mannequin simplificado | CC3/CC4 (Reallusion), prefijo `cc_base_` |
| Spine | `spine_01 → spine_02 → spine_03` (3 huesos) | `spine_01 → ... → spine_05` (5 huesos) |
| Brazos/Piernas/Dedos | Nombres base coinciden (`clavicle_l`, `thigh_l`, `hand_l`, etc.) | Igual + huesos twist/metacarpal/kneesharebone con prefijo `cc_base_` |

**Regla aplicada:** cualquier hueso adicional en ActorCore sin equivalente en
SK_Jess (twist bones, kneesharebone, elbowsharebone, metacarpal, dedos de pie
individuales) se excluye de las Retarget Chains — queda en `None`.

### A.2 — Arquitectura de mesh en BP_Character_Player

`BP_Character_Player` usa un componente principal `Mesh (CharacterMesh0)` más
cinco sub-componentes (`BodyMesh`, `HeadMesh`, `PantsMesh`, `HandsMesh`,
`FeetMesh`, `BackpackMesh`). Confirmado por inspección del grafo: existe un
nodo `Set Leader Pose Component` que asigna `Mesh` como líder de los cinco
sub-meshes — **solo es necesario retargetear animaciones para el skeleton
usado por `Mesh`**, los sub-meshes heredan la pose automáticamente.

> Dato sin explicar: `Mesh (CharacterMesh0)` originalmente carga
> `SK_Jess_Accessory_001`, no un mesh de cuerpo principal. No investigado.

### A.3 — Estructura de ABP_Jess relevante

4 State Machines: `MainStateMachine` (Idle/Move, Swimming, Jump_Cycle,
Jump_End, Crouching), `StanceStateMachine`, `AimOffsetStateMachine`,
`DeathStateMachine`. Sin nodos de manipulación de huesos hardcodeados en el
AnimGraph principal — la lógica usa `Layered blend per bone` con Branch
Filters (escritos contra `SK_Jess`, no requieren cambio si `SK_Jess` sigue
siendo el skeleton de gameplay).

**Hallazgo clave:** `MainStateMachine → Idle/Move` no usa AnimSequence
sueltas — usa **6 Blend Space 2D** (`BS_Jess_IdleRun_Common/Rifle/Pistol/
Blunderbuss/Bow/Combat`), cada uno un asset con Skeleton propio.

### A.4 — Pipeline ejecutado paso a paso

1. **IK Rigs** creados manualmente: `IK_Jess` (Root `pelvis`) e
   `IK_ActorCore_PartyM0001` (Root `pelvis`), con Retarget Chains creadas a
   mano una por una: `Spine`, `LeftArm`/`RightArm`, `LeftLeg`/`RightLeg`, y
   10 chains de dedos (`ThumbLeft/Right`, `IndexLeft/Right`, etc.)
2. **IK Retargeter** (`RTG_ActorCore_to_Jess`): Source = `IK_Jess`,
   Target = `IK_ActorCore_PartyM0001` (corrección de dirección aplicada
   durante la sesión — inicialmente se armó al revés)
3. **Corrección de Retarget Pose (T-pose vs A-pose):** SK_Jess está en
   T-pose, ActorCore en A-pose. Sin corrección, el retargeting FK
   interpretaba la diferencia de pose como parte del movimiento —
   producía brazos cruzados/pegados al cuerpo en cualquier animación.
   Fix: crear un Retarget Pose nuevo en el tab Target y rotar manualmente
   `clavicle_l/r`, `upperarm_l/r`, `lowerarm_l/r` hasta igualar la
   orientación de origen. Confirmado funcional tras el fix.
4. **Export de 442 animaciones** a `Animations_Retargeted/`
5. **Reconstrucción manual de ABP:** se descartó **Compatible Skeletons**
   (probado, causó deformación por diferencia de proporciones y huesos
   twist sin resolver — documentación oficial de Epic confirma que requiere
   jerarquías casi idénticas). Se duplicó `ABP_Jess_C` a mano, se cambió su
   Target Skeleton, y se reemplazaron uno por uno los nodos de animación,
   incluyendo la **reconstrucción manual completa de los 6 Blend Space**
   (inspeccionar samples del original, crear asset nuevo contra el nuevo
   skeleton, rellenar cada sample con la animación retargeteada
   correspondiente).

**Resultado confirmado:** personaje con mesh de ActorCore se mueve en
Idle/Jog/Sprint sin deformación, dentro de `BP_Character_Player`.

### A.5 — Easy fix aplicado (aplica también a la Sección B)

**Síntoma:** doble mesh visible (Jess + placeholder superpuestos), porque los
6 sub-mesh components conservan su Skeletal Mesh original de ESRPGv5.

**Fix:** desactivar `Visible` (no `Hidden in Game`) en `BodyMesh`, `HeadMesh`,
`PantsMesh`, `HandsMesh`, `FeetMesh`, `BackpackMesh` desde Details de
`BP_Character_Player`. No se toca el Skeletal Mesh asignado ni el Leader
Pose Component — mínimo cambio, totalmente reversible.

---

## Sección B — Pipeline automatizado (Advance Skeleton)

### B.1 — Comparación de nomenclatura: hallazgo favorable

El rig exportado por Advance Skeleton usa una convención **casi idéntica**
a `SK_Jess` en la cadena FK principal — coincidencia exacta confirmada en:
`root`, `pelvis`, `spine_01`, `clavicle_l`, `upperarm_l`, `lowerarm_l`,
`hand_l`, `thigh_l`, `calf_l`, `foot_l`, `ball_l`, `neck_01`, `head`, y
también en los huesos helper de IK: `ik_foot_root`, `ik_hand_root`,
**`ik_hand_gun`** (nombre específico, coincidencia poco casual — sugiere que
Advance Skeleton está pensado para compatibilidad con la convención estándar
de Epic).

Diferencias (mismo criterio de exclusión de chains que en Sección A):
`spine_01` a `spine_05` (5 huesos vs 3), huesos twist
(`upperarm_twist_01_l/02_l`, etc.), huesos `*_metacarpal_l/r`.

**Hallazgo menor:** el dedo anular (`ring`) no tiene segmentos
(`ring_01_l/02_l/03_l` no existen como hijos de `ring_metacarpal_l`) —
confirmado vía warning del Auto Retarget Chains ("not found in this
skeleton, but is used by Fortnite Humanoid"). No es un fallo del proceso,
es una característica del rig placeholder — pendiente confirmar con la
artista si es intencional.

### B.2 — Pelvis / IK helper bones: confirmado no problemático

Rotar `pelvis` mueve todo el personaje — comportamiento esperado de
jerarquía de huesos (pelvis es padre de columna y piernas), no relacionado
al retargeting. Los huesos `ik_foot_root`/`ik_hand_root` son helpers de
referencia, no forman parte de ninguna Retarget Chain, y no fueron
necesarios para que la locomoción funcionara correctamente (confirmado en
ambos placeholders).

### B.3 — A-pose confirmada también en Advance Skeleton

Igual que ActorCore, el rig de Advance Skeleton está en A-pose. A diferencia
de ActorCore, **no fue necesario repetir el ajuste manual de Retarget Pose**
— el resultado se vio correcto en Idle y en Jog sin ese paso. Hipótesis no
confirmada: el proceso automático (que usa internamente los templates
`UE4 Mannequin` / `Fortnite Humanoid` de Epic) puede resolver la alineación
de pose de forma nativa para rigs que sigan esa convención — no verificado
a fondo, queda como nota para investigar si se repite el patrón con el rig
final de Hana.

### B.4 — Auto Retarget Chains: funcionó, redujo trabajo significativamente

1. Importado el FBX de Advance Skeleton a una carpeta limpia, generando su
   propio Skeleton nuevo (`Prueba_Rig_Hadita_Pelona`)
2. `Create IK Rig` sobre ese Skeleton, Retarget Root fijado a mano en
   `pelvis`
3. Botón **Auto Retarget Chains** (toolbar del editor de IK Rig) — generó
   automáticamente prácticamente todas las chains necesarias (Spine, Neck,
   Head, LeftArm/RightArm, LeftLeg/RightLeg, y cadenas de dedos con
   metacarpals separados), replicando el mismo criterio que se había armado
   a mano para ActorCore — **sin crear las 15 chains manualmente**

### B.5 — Ruta descartada: IK Rig/Retargeter manual, reemplazado por Export Retarget Assets

Se empezó a armar `IK_Prueba_Rig_Hadita_Pelona` +
`RTG_Jess_to_AdvancedSkeleton` de forma manual (mismo patrón que Sección A),
pero se descubrió que la ventana **Retarget Animations** (accesible por clic
derecho sobre una AnimSequence → **Retarget Animation Assets**), con el
checkbox `Auto Generate Retargeter` activado, genera su propia configuración
completa usando templates internos de Epic. El botón **Export Retarget
Assets** dentro de esa ventana permite guardar esa configuración generada
como assets reutilizables — se usó este camino porque dio mejor resultado
visual, generando:

- `IK_AutoGeneratedSource` (reemplaza a `IK_Jess` armado a mano)
- `IK_AutoGeneratedTarget` (reemplaza a `IK_Prueba_Rig_Hadita_Pelona` armado
  a mano)
- `RTG_AutoGenerated` (reemplaza al Retargeter manual)

**Decisión tomada:** se descartaron los assets armados manualmente en favor
de este set autogenerado, dentro de esta misma branch de prueba.

**Nota de mapeo:** al crear el Retargeter manual antes de descubrir este
camino, se observó que el dropdown **Auto-Map Chains** (dentro del tab
Chain Mapping del editor del IK Retargeter) es el botón correcto para
completar la columna `Source Chain` cuando se arma un Retargeter desde cero
— información útil para cualquier Retargeter futuro que se decida construir
manualmente.

### B.6 — Verificación de animación individual

Se probó `Retarget Animations` sobre `A_Jess_Idle` y `A_Jess_Combat_OH_Death`
de forma aislada — ambas se vieron correctas. La deformación leve observada
en `A_Jess_Combat_OH_Death` se confirmó como problema de **skinning weights
del mesh placeholder** (reproducible incluso dentro de Maya, sin pasar por
Unreal) — descartado como problema del pipeline, es una nota para la artista
sobre el placeholder.

### B.7 — Duplicate and Retarget Animation Blueprint: automatizó los Blend Spaces por completo

Se ejecutó **Retarget Animation Assets → Duplicate and Retarget Animation
Blueprint** sobre `ABP_Jess_C`, usando `RTG_AutoGenerated` (con `Auto
Generate Retargeter` desmarcado y el Retarget Asset seleccionado
manualmente) — resultado renombrado `ABP_AdvancedSkeleton`.

**Confirmado por inspección directa:** dentro de `MainStateMachine →
Idle/Move`, los 6 nodos Blend Space Player quedaron intactos y
funcionando, apuntando a Blend Space **nuevos y propios**, generados
automáticamente contra el Skeleton de Advance Skeleton — mismos 13 samples,
mismos valores de Direction/Speed/Rate Scale que el original, cada sample
apuntando a su animación ya retargeteada. Verificado visualmente moviendo el
Preview Point dentro del Blend Space — correcto.

**Esto es el hallazgo principal de la sesión:** a diferencia de la Sección A
(donde los 6 Blend Space se reconstruyeron a mano, el paso más costoso de
todo el pipeline), aquí el proceso automático los generó y remapeó sin
intervención manual.

### B.8 — Bug encontrado: deformación de proporciones en el ABP autogenerado

**Síntoma:** al asignar `ABP_AdvancedSkeleton` (el resultado 100%
automático de B.7) en `BP_Character_Player`, el personaje aparece
desproporcionado (cabeza y hombros/pecho notablemente agrandados, resto del
cuerpo pequeño) — consistente en el preview del propio ABP, en el viewport
de `BP_Character_Player`, y en Play mode.

**Causas investigadas y descartadas:**
- **Post Process Anim Blueprint del Skeletal Mesh:** el proceso automático
  también duplicó `ABP_Jess_IK` (un Post Process ABP con un sistema de Leg
  IK que usa Virtual Bones: `VB root_foot_l/r`, `VB thigh_l_calf_l/r`,
  definidos manualmente dentro del Skeleton `SK_Jess`, no replicables por
  ninguna Retarget Chain). La copia duplicada de `ABP_Jess_IK` marca
  errores de compilación (`Bone not found in Skeleton`) porque esos Virtual
  Bones no existen en el nuevo Skeleton. **Sin embargo, se confirmó que el
  campo Post Process Anim Blueprint del Skeletal Mesh de
  `Prueba_Rig_Hadita_Pelona` ya estaba en `None`** — por lo tanto esta
  copia rota no se está ejecutando y **queda descartada como causa directa**
  de la deformación visible.
- **Mesh/Skeleton en sí:** descartado — la Reference Pose del Skeletal Mesh
  se ve proporcionada correctamente en su propio editor, sin animación de
  por medio.
- **Retarget Chain de Head/Neck mal configurada:** revisado, `Translation
  Mode` en `None` (configuración esperada) — no es la causa evidente.

**Prueba de aislamiento (confirma dónde SÍ está el bug):** se configuró
`BP_Character_Player` con el Skeletal Mesh de `Prueba_Rig_Hadita_Pelona`
pero volviendo el `Anim Class` a `ABP_Jess_C` (el original, sin retargeting),
y dentro de ese ABP se reemplazó manualmente la referencia de animación de
Idle por su versión retargeteada. **Resultado: las proporciones del
personaje se corrigen por completo.** Esto confirma que el mesh, el
skeleton, y las animaciones/Blend Spaces retargeteados están todos
correctos — **el bug vive específicamente dentro del asset
`ABP_AdvancedSkeleton` generado por Duplicate and Retarget Animation
Blueprint, por una causa aún no identificada** (no genera ningún error de
compilación visible).

**Estado:** 🔴 No resuelto — causa raíz desconocida. No bloqueante gracias al
workaround de la Sección B.9.

### B.9 — Workaround aplicado (temporal, documentado como deuda técnica)

Dado el bug de B.8, se optó por **no usar `ABP_AdvancedSkeleton` como base**.
En su lugar, dentro de una copia de `ABP_Jess_C` (el ABP original, que ya
sabemos que funciona correctamente con el mesh de Advance Skeleton según la
prueba de aislamiento), se copió y pegó directamente el nodo Blend Space
Player `BS_Jess_IdleRun_Common` desde `ABP_AdvancedSkeleton` (el asset
autogenerado, roto a nivel de ABP pero con Blend Spaces internos correctos),
reconectando sus pines manualmente.

**Alcance actual de este workaround:** solo el Blend Space de `Common`
(Idle/Run del stance sin arma) fue reemplazado por este método. Los otros 5
Blend Space de `Idle/Move` (`Rifle`, `Pistol`, `Blunderbuss`, `Bow`,
`Combat`), y los otros 3 State Machines (`StanceStateMachine`,
`DeathStateMachine`, `AimOffsetStateMachine`) **no han sido tocados
todavía** — siguen en su estado original de `ABP_Jess_C` sin retargeting.

Se documenta explícitamente como **solución no elegante y temporal** — el
copy-paste manual de nodos entre dos ABP es funcional pero no escala bien a
40+ nodos de animación repartidos en 4 State Machines. Sirve para continuar
validando el pipeline mientras se investiga la causa raíz de B.8 en paralelo
o se decide construir el ABP final completamente a mano (mismo proceso
documentado en la Sección A.4, punto 5) reutilizando los Blend Space y
AnimSequence ya generados automáticamente.

---

## Comparación de esfuerzo entre ambas rutas

| Paso del pipeline | ActorCore (manual) | Advance Skeleton (automatizado) |
|---|---|---|
| Creación de IK Rig + Chains | Manual, hueso por hueso (15 chains) | Auto Retarget Chains — generado automáticamente |
| Corrección T-pose/A-pose | Manual, requerida y confirmada necesaria | No fue necesaria en las pruebas realizadas (Idle, Jog) |
| Export de animaciones | Export Selected Animations, manual por lote | Igual, sin diferencia relevante |
| Reconstrucción de Blend Spaces | 100% manual (inspección + creación + relleno de 13 samples ×6) | Generados automáticamente por Duplicate and Retarget Animation Blueprint |
| ABP resultante utilizable directamente | Sí (construido a mano, sin bugs) | No — bug de proporciones sin causa identificada (Sección B.8) |

---

## Deuda técnica registrada (consolidada, ambas rutas)

| Problema | Notas | Prioridad | Estado |
|---|---|---|---|
| **Bug de proporciones en ABP autogenerado (`ABP_AdvancedSkeleton`)** | Cabeza/pecho deformados sin error de compilación visible. Aislado a nivel de asset ABP, no de mesh/skeleton/animación. Causa raíz desconocida | 🔴 Alta | Investigación pendiente — no bloqueante gracias a workaround B.9 |
| Workaround de copy-paste de nodo Blend Space entre ABPs | Solo `BS_Jess_IdleRun_Common` reemplazado así. No escalable a los ~40 nodos restantes del ABP completo | 🔴 Alta | Temporal — reemplazar por reconstrucción manual completa (patrón Sección A.4.5) o resolver B.8 |
| 5 Blend Space restantes de `Idle/Move` (Advance Skeleton) | `Rifle`, `Pistol`, `Blunderbuss`, `Bow`, `Combat` — ya existen generados automáticamente y verificados en B.7, solo falta conectarlos igual que Common | 🟡 Media | Pendiente — mecánico, sin riesgo conocido |
| `StanceStateMachine`, `DeathStateMachine`, `AimOffsetStateMachine` (Advance Skeleton) | No revisados en esta sesión para esta ruta | 🔴 Alta | Pendiente |
| Sub-meshes modulares (`BodyMesh`, etc.) siguen existiendo en `BP_Character_Player` | Solo ocultos vía `Visible = false`. Eliminar cuando se defina el sistema de outfits único | 🟡 Media | Fix temporal aplicado |
| Dedo anular sin segmentar en rig de Advance Skeleton | Confirmado por warning de Auto Retarget Chains, no un fallo del proceso | 🟢 Baja | Pendiente confirmar con artista si es intencional |
| Sistema de Leg IK (`ABP_Jess_IK`) no tiene equivalente en ningún placeholder retargeteado | Depende de Virtual Bones definidos manualmente por Skeleton — no se replica vía Retarget Chains. Sin impacto confirmado en el resultado visual actual (campo Post Process en None) | 🟢 Baja | Sin decisión — evaluar si se necesita recrear para el personaje final |
| `ABP_Jess_C` original (Idle reemplazado manualmente durante la prueba de aislamiento de B.8) | Verificar que no haya quedado un cambio residual no deseado en el asset original tras la prueba de diagnóstico | 🟡 Media | Verificar en próxima sesión |
| Preview Mesh del ABP no detecta Skeletal Mesh en el picker (visto con ActorCore) | Posible desajuste de Skeleton asset entre mesh y Target Skeleton, o filtro de UI sin refrescar | 🟢 Baja | Cosmético, no bloquea Play |
| Componente `Mesh (CharacterMesh0)` carga `SK_Jess_Accessory_001` en vez de un mesh de cuerpo | Sin explicación confirmada | 🟢 Baja | Pendiente investigación |

---

## Plan de continuación sugerido

```
Fase A — Resolver o rodear el bug de B.8
  → Opción 1: investigar causa raíz (bisección dentro del AnimGraph de
    ABP_AdvancedSkeleton, ej. aislar el nodo Mesh Space Blending de la
    sección Full Body Blends)
  → Opción 2 (más rápida, recomendada si el tiempo apremia): abandonar
    ABP_AdvancedSkeleton como base, construir manualmente igual que en
    Sección A.4.5, reutilizando los Blend Space y AnimSequence ya
    generados automáticamente — evita depender del asset con bug

Fase B — Completar Idle/Move para Advance Skeleton
  → Conectar los 5 Blend Space restantes (ya existen, generados en B.7)

Fase C — StanceStateMachine, DeathStateMachine, AimOffsetStateMachine
  → Mismo criterio: reutilizar assets ya retargeteados por el proceso
    automático donde existan, completar manualmente donde falten

Fase D — Verificación final en Play
  → Idle/Walk/Run confirmado ya en la prueba de aislamiento (con ABP_Jess_C
    + Idle reemplazado) — repetir con el ABP definitivo una vez resuelta
    la Fase A

Fase E — Limpieza de deuda técnica
  → Eliminar sub-mesh components modulares o mantener ocultos indefinidamente
  → Confirmar estado de ABP_Jess_C original tras la prueba de diagnóstico

Fase F — Evaluación para personaje final (Hana)
  → Con el hallazgo de Advance Skeleton (nomenclatura compatible + Duplicate
    and Retarget Animation Blueprint funcional para Blend Spaces), evaluar
    si el pipeline automatizado es viable como base para Hana, resolviendo
    primero el bug de B.8 para no heredarlo en producción
  → Definir con el animador cuáles animaciones requieren reemplazo manual
    dedicado vs cuáles pueden quedarse retargeteadas automáticamente
```

---

## Punto exacto donde se detiene esta sesión

**Confirmado y funcional (Sección A, ActorCore):** pipeline manual completo,
sin bugs conocidos, personaje se mueve sin deformación en Idle/Jog/Sprint.

**Confirmado y funcional (Sección B, Advance Skeleton):**
- Auto Retarget Chains generó correctamente las chains del IK Rig
- `RTG_AutoGenerated` (vía Export Retarget Assets) funciona correctamente
  para animaciones individuales, sin requerir fix manual de T-pose/A-pose
- `Duplicate and Retarget Animation Blueprint` generó y remapeó
  correctamente los 6 Blend Space de `Idle/Move` — confirmado por
  inspección, sin necesidad de reconstrucción manual
- Prueba de aislamiento confirma: mesh, skeleton y animaciones
  retargeteadas están correctos: usando `ABP_Jess_C` original con el Idle
  reemplazado a mano, el personaje se ve proporcionado y se mueve bien

**No resuelto:** el ABP `ABP_AdvancedSkeleton` generado 100% automáticamente
tiene un bug de proporciones sin causa identificada. Workaround temporal
aplicado (copy-paste de un solo nodo Blend Space hacia una copia de
`ABP_Jess_C`) — cubre únicamente el stance `Common` de `Idle/Move`. El resto
del ABP (5 Blend Space restantes + 3 State Machines completos) queda
pendiente para la próxima sesión, ya sea completando el mismo workaround
nodo por nodo o resolviendo la causa raíz del bug primero.

---

*Archivo actualizado — sesión Light Paradox (Technical Animation Pipeline ·
Advance Skeleton, pipeline automatizado)*
*Cambios: Sección B agregada completa (experimento Advance Skeleton),
comparación de esfuerzo entre ambas rutas, deuda técnica consolidada, plan
de continuación actualizado*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
