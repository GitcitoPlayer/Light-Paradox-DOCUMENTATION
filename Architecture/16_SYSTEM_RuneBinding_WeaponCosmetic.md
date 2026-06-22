# 16 — Sistema: Rune Binding por Ítem + Armas como Cosméticos
### Sistema: Persistencia de runas por ítem · Armas/Herramientas en slot cosmético
### Base Asset: EasySurvivalRPGv5
### Fuente: Diseño del cliente + análisis arquitectural — sesión Light Paradox (Lógica 4 / Lógica 5)
### Proyecto: Light Paradox · UE 5.4.4

---

## Contexto y visión de diseño

### Dinámica central confirmada por el cliente

El jugador obtiene runas individuales y las craftea en **Rune Words** (ítems nuevos con
propiedades). Las Rune Words se asignan a los slots de equipamiento desde la **Mesa Rúnica**.

Subdinámica confirmada:
- Solo se puede asignar una runa por día de juego
- Al asignar una runa, el siguiente slot entra en cooldown
- A mayor cantidad de runas asignadas, menor probabilidad de éxito en la siguiente asignación
- Si falla la asignación, el jugador es penalizado con esperar un día de juego completo

### Qué está prototipado (confirmado, Head únicamente)
- Asignación de Rune Words al slot de Head desde la Mesa Rúnica
- Restricción de acceso a la Mesa Rúnica para habilitar los slots
- Cooldown entre slots via `UI_CraftingQueue` → `UI_RuneAssignQueue`
- Sistema de probabilidad de éxito decreciente por slot (`SuccessChance` en `UI_ItemSlot`)
- `bIsLocked` para bloquear el siguiente slot durante el cooldown
- `GetNextRuneSlot` para identificar el slot siguiente a bloquear

### Qué NO está prototipado (deuda activa)
1. **Persistencia de runas por ítem** — las runas actuales viven en el personaje, no en el ítem
2. **Replicación de la configuración por pieza de equipamiento** — Body, Pants, Hands, Feet, Tool, Backpack
3. **Armas y herramientas como cosméticos** — acceso a slots de runa independiente del hotbar
4. **Slot cosmético dinámico** — el arma/herramienta activa en mano se refleja en el menú de equipamiento

---

## Problema arquitectural central

### Estado actual del sistema (confirmado en documentación)

```
[EquipmentContainer en BP_Character_Player]
  └── Slot 7  → HeadRuneWord (runa asignada al personaje)
  └── Slot 14 → HeadRuneWord_2
  └── Slot 15 → HeadRuneWord_3
  ...

[UI_Character]
  └── Equipment_HeadRuneSlot   (SlotNumber 7)
  └── Equipment_HeadRuneSlot_1 (SlotNumber 14)
  ...
```

**El problema:** Las runas están asignadas al slot del **personaje**, no al ítem de equipamiento.

Si el jugador se quita el casco y se pone otro, los slots de runa del personaje siguen
mostrando las runas del casco anterior. La configuración de runas no viaja con el ítem.

### Lo que se necesita (diseño objetivo)

```
[Item: Casco_A]  → guarda su propia configuración de runas [Runa1, Runa2, null, ...]
[Item: Casco_B]  → guarda su propia configuración de runas [null, null, null, ...]

Al equipar Casco_A → los slots de runa del personaje reflejan la config de Casco_A
Al equipar Casco_B → los slots de runa se vacían (Casco_B no tiene runas)
Al desequipar     → la config de runas de Casco_A se guarda en el ítem
```

---

## Análisis de dependencias arquitecturales

### Cadena de sistemas dependientes (confirmada en documentación)

```
STR_ItemData (struct base del ítem)
      ↓
BP_ContainerComponent → GetItem / SetItem / AddAmountToSlot
      ↓
BP_EquipmentComponent → CheckContainerSlotForItem / EquipmentSlots array
      ↓
BP_Character_Player   → EquipmentContainer (instancia) / ContainerSlotSettings
      ↓
UI_HUD                → UpdateContainerSlot_BPI → UpdateContainerSlot
      ↓
UI_Character          → UpdateEquipmentSlotItem → UpdateRuneSlotVisibility
      ↓
UI_ItemSlot           → SetItemData / OnDrop / bIsLocked / SuccessChance
      ↓
UI_CraftingQueue      → AddRuneToQueue → OnRuneAssignComplete
```

**Evaluación:** La cadena es larga. Cualquier cambio en `STR_ItemData` tiene efecto
en cascada hacia arriba en todos los sistemas. Esto confirma la inferencia del
Technical Designer: **no es un trabajo sencillo**.

### Punto crítico no documentado

`STR_ItemData` es el struct base de todos los ítems. No está documentado si:
- Tiene campos extensibles (arrays, maps) disponibles para agregar datos custom
- Si los ítems tienen un ID único persistente o solo un reference al Data Asset
- Si ESRPGv5 provee algún mecanismo nativo para datos custom por instancia de ítem

> **⚠️ Inferencia (no confirmada):** Es probable que `STR_ItemData` sea un struct
> fijo del asset base. Agregar campos custom requeriría modificar el struct o usar
> un sistema paralelo de persistencia. Requiere inspección directa antes de
> tomar decisiones de implementación.

---

## Sistema: Armas y Herramientas como Cosméticos

### Problema de diseño confirmado por el cliente

Con el sistema actual hay tres estados posibles para un arma/herramienta:

| Estado | Ubicación | Slots de runa |
|---|---|---|
| En inventario | InventoryContainer | No accesibles |
| En hotbar | HotbarContainer (inferido) | No accesibles |
| Equipada (slot de equipment) | EquipmentContainer | Accesibles |

El conflicto: si el jugador tiene la herramienta en el hotbar en lugar del slot de
equipamiento, **pierde acceso visual a sus slots de runa**, aunque la configuración
de runas no se pierda.

El jugador puede tener múltiples armas y herramientas, cada una con su propia
configuración de runas independiente. A ojos del jugador, cada arma que usa
ya tiene sus stats potenciados por sus propias runas — sean 2, 3 o las que sean.

### Decisiones de diseño confirmadas por el cliente

| Decisión | Detalle |
|---|---|
| Slot de Tool es único | Solo un slot de herramienta visible en el UI de equipamiento a la vez |
| El slot se actualiza dinámicamente | Muestra el arma/herramienta activa en ese momento |
| Cada arma guarda su propia config de runas | Independiente de las otras armas del jugador |
| Visión de UI/UX a futuro | La Mesa Rúnica será una ventana por pieza individual — el jugador coloca una pieza de equipamiento y la configura. Diseño menos invasivo visualmente. Esta visión no está implementada aún pero condiciona decisiones de arquitectura. |

### Solución propuesta por el Technical Designer

Usar el estado **"arma/herramienta activa en mano"** (cuando el personaje ejecuta
la acción de uso — click) como trigger para asignar dinámicamente esa herramienta
al único slot de Tool del `EquipmentContainer`.

```
[Jugador activa herramienta en mano (click / selección en hotbar)]
      ↓
[Detectar herramienta activa en mano]
      ↓
[Asignar dinámicamente al slot Tool del EquipmentContainer]
  → sobreescribe el arma anterior en ese slot
      ↓
[UI_Character refleja el nuevo ítem en el slot Tool]
  → carga la configuración de runas de esa herramienta específica
      ↓
[Al cambiar a otra herramienta → slot Tool se actualiza nuevamente]
  → la herramienta anterior conserva su config de runas en su propio ítem
```

**Estado de esta propuesta:** Diseño confirmado a nivel conceptual por el cliente.
No inspeccionado en ESRPGv5 si existe un evento o variable que exponga
"herramienta activa en mano". Requiere inspección de `BP_Character_Player`
y del sistema de hotbar antes de prototipar.

### Riesgo específico no resuelto

Si ESRPGv5 tiene lógica propia que reacciona al cambio del slot de Tool en
`EquipmentContainer` (stats, animaciones, efectos secundarios), el swap dinámico
podría tener efectos no deseados. Esto debe verificarse durante la inspección
del flujo de equip/unequip.

---

## Grado de dificultad estimado

| Componente | Dificultad | Razón |
|---|---|---|
| Persistencia de runas en `STR_ItemData` | 🔴 Alta | Depende de si el struct es extensible. Si no lo es, requiere sistema paralelo de persistencia. Cadena de dependencias larga. |
| Replicar slots Head → Body/Pants/Hands/Feet/Tool/Backpack | 🟡 Media | Trabajo repetitivo pero de bajo riesgo. Mismo patrón ya establecido con Head. |
| Armas/herramientas como cosméticos (slot cosmético estático) | 🟡 Media | Requiere entender el sistema de hotbar de ESRPGv5 y agregar lógica de swap en EquipmentContainer. |
| Slot cosmético dinámico (herramienta activa en mano) | 🔴 Alta | Requiere identificar el evento de "herramienta activa" en ESRPGv5, que no está documentado. Riesgo de colisión con lógica del asset base. |
| Cargar/descargar config de runas al equipar/desequipar | 🔴 Alta | Depende de la solución de persistencia. Requiere modificar flujo de equip/unequip de ESRPGv5. |

---

## Tomas de acción

### Acción inmediata — antes de prototipar cualquier cosa

| # | Acción | Objetivo |
|---|---|---|
| 1 | Inspeccionar `STR_ItemData` en el editor | Confirmar si tiene campos extensibles o si es struct cerrado |
| 2 | Inspeccionar `BP_Character_Player` — sistema de hotbar | Identificar si existe evento de "herramienta activa en mano" |
| 3 | Inspeccionar `BP_EquipmentComponent` — flujo de equip/unequip | Identificar el punto donde se puede interceptar el cambio de ítem equipado |
| 4 | Documentar hallazgos en archivos .md correspondientes | Antes de escribir un solo nodo Blueprint |

### Estrategia de desarrollo recomendada (confirmada con el cliente)

Desarrollar en paralelo con otros sistemas que **no tengan dependencia** de
Rune Binding ni de slot cosmético dinámico.

Orden sugerido para cuando se retome:

```
Fase 1 — Bajo riesgo, sin dependencias bloqueantes
  → Replicar el sistema de Head a Body/Pants/Hands/Feet/Tool/Backpack
  → Cada grupo sigue el mismo patrón de Head ya documentado
  → No requiere cambios en STR_ItemData

Fase 2 — Requiere decisión arquitectural previa
  → Definir mecanismo de persistencia de runas por ítem
  → Depende de inspección de STR_ItemData (Acción 1)

Fase 3 — Depende de Fase 2 + inspección de hotbar
  → Armas como cosméticos con slot cosmético dinámico
  → Cargar/descargar config de runas al equipar/desequipar

Fase 4 — Integración y testing
  → Verificar que el sistema completo no rompe el flujo de
    equip/unequip base de ESRPGv5
```

---

## Deuda técnica registrada

| Problema | Notas | Prioridad | Estado |
|---|---|---|---|
| `STR_ItemData` sin inspeccionar | Crítico — bloquea decisión de persistencia | 🔴 Alta | Pendiente inspección |
| Sistema de hotbar sin documentar | Necesario para slot cosmético dinámico | 🔴 Alta | Pendiente inspección |
| Flujo equip/unequip sin documentar | Punto de intercepción para cargar config de runas | 🔴 Alta | Pendiente inspección |
| Slots de runa para Body/Pants/Hands/Feet/Tool/Backpack | Trabajo conocido, patrón establecido con Head | 🟡 Media | Pendiente — post estabilización de Head |
| Mecanismo de persistencia de runas por ítem | Depende de STR_ItemData | 🔴 Alta | Pendiente decisión arquitectural |
| Slot cosmético dinámico (herramienta activa) | Depende de inspección de hotbar | 🟡 Media | Pendiente inspección |

---

## Validación de la inferencia del Technical Designer

La inferencia del Technical Designer es **correcta**:

- La cadena de Blueprints dependientes es larga y está confirmada en la documentación
- `STR_ItemData` es el cuello de botella arquitectural más crítico
- La solución del slot cosmético dinámico por herramienta activa en mano es
  la ruta más limpia disponible sin reestructurar el sistema de hotbar completo
- Desarrollar otros sistemas en paralelo mientras se investigan las dependencias
  es la estrategia correcta

Lo que no se puede confirmar aún sin inspección:
- Si `STR_ItemData` permite extensión sin romper el asset base
- Si existe el evento de "herramienta activa" en `BP_Character_Player`

---

*Archivo creado — sesión Light Paradox (Lógica 4 documentación inicial)*
*Actualizado — decisiones de diseño confirmadas: slot Tool único y dinámico, visión UI/UX Mesa Rúnica*
*Sistemas: Rune Binding por ítem · Armas como cosméticos · Slot cosmético dinámico*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
