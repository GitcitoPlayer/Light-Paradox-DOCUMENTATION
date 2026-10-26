# ACTUALIZACIÓN DE .md — Etapa 1: Head de 10 a 19 slots de runa
### Proyecto: Light Paradox · UE 5.4.4 · Base: EasySurvivalRPGv5
### Cómo usar este archivo
Cada sección indica el .md destino y si **reemplaza** o **agrega** contenido. Las marcas `⏳ confirmar` son datos que no tengo verificados por captura: confírmalos en el editor antes de darlos por cerrados.

---

## 0 — CORRECCIÓN IMPORTANTE (error mío en una respuesta anterior)

En la verificación previa a la Etapa 1 te dije que, si `AddRuneToQueue` / `OnRuneAssignComplete` usaban `Set IsBlocked`, los cambiaras a `bIsLocked`. **Eso era incorrecto.** El doc 13 ya había resuelto esa contradicción: los nodos reales usan `IsBlocked` y `bIsLocked` es una variable huérfana. Tu log de runtime sigue mostrando `Set IsBlocked`, lo que indica que no se cambió nada. **No cambiar a `bIsLocked`.** Si en algún momento se hizo el cambio, revertirlo a `IsBlocked`.

---

## 1 — `04_BLUEPRINT_UI_ItemSlot.md`

### 1.1 REEMPLAZAR en la sección `OnDrop`, bloque `IsValidRuneDestination`
```
IsBaseRuneSlot       = Equal (Integer): GET SlotNumber == 7
InDuplicateRuneRange = AND( GET SlotNumber >= 14 , GET SlotNumber <= 31 )   ← antes <= 22
IsValidRuneDestination = OR(IsBaseRuneSlot, InDuplicateRuneRange)
```
> Rango vigente de Head: slot base 7 + slots 14–31 (19 slots de runa en total).
> ⚠️ Al replicar a las otras 6 familias este bloque deja de escalar (un OR por familia). Alternativa planificada (Etapa 2, paso G2): `GET SlotNumber >= 7` dentro del `EquipmentContainer`, porque el `Cast To BP_EquipmentComponent` ya descarta otros contenedores. No aplicada todavía.

### 1.2 AGREGAR — Variable de depuración
| Variable | Tipo | Notas |
|---|---|---|
| `bDebugForceRuneSuccess` | Boolean | Category `Debug`. Si es True, el `Branch` Éxito/Fallo de `OnDrop` entra siempre por ÉXITO. Se conecta con un nodo `OR`: A = resultado de `Random Integer in Range <= SuccessChance`, B = `Debug Force Rune Success`. **Default Value debe quedar en False antes de entregar al cliente.** ⏳ confirmar estado del default |

### 1.3 AGREGAR — Probabilidad real del roll (hallazgo, inferencia)
`Random Integer in Range (0..100)` es inclusivo en ambos extremos (101 valores). Con `SuccessChance = 1` la probabilidad real es ≈ 2/101, no 1%. Corrección propuesta: `Min = 1`, `Max = 100`. ⏳ confirmar si se aplicó. Inferencia sobre el comportamiento del nodo, no está en los exports.

### 1.4 Deuda técnica — AGREGAR filas
| Problema | Estado |
|---|---|
| `bDebugForceRuneSuccess` presente en el asset base (Rule 4.1). Dejar default False o eliminar al cerrar pruebas | ⏳ Pendiente |
| Verificar Min=1 en el roll de probabilidad | ⏳ Pendiente |

---

## 2 — `06_BLUEPRINT_BP_EquipmentComponent.md`

### 2.1 REEMPLAZAR tabla `E_EquipmentSlot` (añadir al final)
| Índice | Display Name |
|---|---|
| 22 | HeadRuneWord_10 |
| 23 | HeadRuneWord_11 |
| 24 | HeadRuneWord_12 |
| 25 | HeadRuneWord_13 |
| 26 | HeadRuneWord_14 |
| 27 | HeadRuneWord_15 |
| 28 | HeadRuneWord_16 |
| 29 | HeadRuneWord_17 |
| 30 | HeadRuneWord_18 |
| 31 | HeadRuneWord_19 |

`E_EquipmentType` se mantiene sincronizado con los mismos 9 valores nuevos (23–31) en el mismo orden.

### 2.2 REEMPLAZAR fórmula lógica final de `CheckContainerSlotForItem`
```
Result = ParentResult
     AND ItemIsEquipmentResult
     AND ( (EquipmentType == Slot)
           OR (EquipmentType == HeadRuneWord AND Slot >= 14 AND Slot <= 31) )   ← antes <= 22
```
El nodo `<=` (Integer) pasó de `22` a `31`.

### 2.3 AGREGAR nota
`Local Equipment Types` no requirió entradas nuevas en la Etapa 1: los slots 23–31 comparten el tag `EasyRPG.Items.Equipment.HeadRuneWord`.

---

## 3 — `07_BLUEPRINT_BP_Character_Player.md`

### 3.1 REEMPLAZAR en la tabla de variables de instancia
| Variable | Valor confirmado |
|---|---|
| `Equipment Slots` | **32 elementos (0–31)** ⏳ confirmar |
| `Container Slot Settings` | **32 elementos (0–31)** ⏳ confirmar. *Historial: el doc anterior decía 8; la captura de verificación mostró 23 (0–22) antes de la Etapa 1, sincronizado con `Equipment Slots`.* |
| `Slots` | 14 (capturado; no se tocó, no hay evidencia documentada de que deba coincidir con `Equipment Slots`) |

### 3.2 AGREGAR filas a la tabla `Container Slot Settings`
Índices 23–31: mismo `RestrictionQuery` que el índice 22 (`All Expressions Match` → `All Tags Match` → `EasyRPG.Items.Equipment.HeadRuneWord`). Referencia visual: el índice 22 tiene `Name: Head Rune Slot`, `Background: T_UI_Category_Magic`.

### 3.3 REEMPLAZAR la checklist rápida de slot nuevo (ahora 10 pasos)
1. Agregar el Gameplay Tag en `DT_GameplayTags_Items` (si la familia es nueva).
2. Agregar valores al final de `E_EquipmentSlot` (nunca reordenar).
3. Mismos valores, mismo orden, al final de `E_EquipmentType`.
4. Expandir `Equipment Slots` y asignar cada elemento a su enum. Verificar la longitud real.
5. Confirmar que `Local Equipment Types` cubre el tag.
6. Expandir `Container Slot Settings` con su `RestrictionQuery`, sin dejar elementos vacíos intermedios.
7. Conectar el exec de cada valor nuevo en `Switch on E_EquipmentSlot` (`EquipmentChanged`). Refresh Node si no aparecen.
8. Ajustar el `<=` del bloque OR en `CheckContainerSlotForItem` (o añadir bloque OR si es familia nueva).
9. **UI_Character** (ver doc 08, checklist de widgets): duplicar widgets, `SlotNumber`, `SuccessChance`, y 5 lugares de cableado.
10. **UI_ItemSlot.OnDrop**: ajustar el rango de `IsValidRuneDestination`.

---

## 4 — `08_BLUEPRINT_UI_Character.md`

### 4.1 REEMPLAZAR tabla "Slots de Head Rune Word confirmados"
| Widget | SlotNumber | SuccessChance |
|---|---|---|
| `Equipment_HeadRuneSlot` | 7 | 100 |
| `_1` | 14 | 90 |
| `_2` | 15 | 80 |
| `_3` | 16 | 70 |
| `_4` | 17 | 60 |
| `_5` | 18 | 50 |
| `_6` | 19 | 40 |
| `_7` | 20 | 30 |
| `_8` | 21 | 20 |
| `_9` | 22 | 10 |
| `_10` | 23 | 9 |
| `_11` | 24 | 8 |
| `_12` | 25 | 7 |
| `_13` | 26 | 6 |
| `_14` | 27 | 5 |
| `_15` | 28 | 4 |
| `_16` | 29 | 3 |
| `_17` | 30 | 2 |
| `_18` | 31 | 1 |

Todos con Visibility default `Collapsed` e `Is Variable` activo. Decisión del cliente: de 100 a 10 baja de 10 en 10, y de 10 a 1 baja de 1 en 1 (el 0% no tiene sentido; el último slot aloja la runa con 1%).

Relación general: el widget `_K` tiene `SlotNumber = 13 + K`, y su enum es `HeadRuneWord_(K+1)`.

### 4.2 AGREGAR — Los 5 lugares que referencian cada widget de slot de runa
Confirmado por Find References de `Equipment_HeadRuneSlot_9`:
| # | Lugar | Qué hay que hacer por cada widget nuevo |
|---|---|---|
| 1 | `EventGraph` → `SetEquipmentReference` | Conectar el widget al pin `Target` de `Set Container Reference` (el `Container` ya viene de `Equipment Reference`) |
| 2 | `UpdateEquipmentSlotItem` | Añadir su opción al `Select` que alimenta `Update Item Data` |
| 3 | `UpdateRuneSlotVisibility` | Añadir su widget al grafo colapsado (ver 4.3) |
| 4 | `GetNextRuneSlot` | Añadir su opción en el `Select` por Integer |
| 5 | `UpdateRuneRemovalLocks` | Añadirlo al `Make Array` y ajustar `Last Index` y el `GET` final |

### 4.3 REEMPLAZAR descripción de los tres patrones (hoy distintos)
| Función | Patrón | Índice del Select |
|---|---|---|
| `UpdateRuneSlotVisibility` | `Select` dentro de un Collapsed Graph | Enum `E_EquipmentSlot` (el slot que **recibe** el ítem → widget del **siguiente**) |
| `GetNextRuneSlot` | `Select` | Integer (`Current Slot Number`) |
| `UpdateRuneRemovalLocks` | `Make Array` + `For Loop` + `GET` | Índice del array |

Mapeo de `UpdateRuneSlotVisibility` (enum del slot que recibe → widget que se revela):
`Head` → base · `Head Rune Word` (7) → `_1` · `Head Rune Word 2` (14) → `_2` · … · `Head Rune Word 10` (22) → `_10` · … · `Head Rune Word 18` (30) → `_18` · `Head Rune Word 19` (31) → ninguno.

Mapeo de `GetNextRuneSlot`: 7 → `_1` · 14 → `_2` · 15 → `_3` · … · 21 → `_9` · 22 → `_10` · … · 30 → `_18` · 31 → `None`.

> Nota: las entradas del Collapsed Graph tienen nombres de enum (`Head Rune Word 1…18`) que no coinciden con los enums del Select. Funciona, pero confunde. Si se conserva el grafo, renombrar las entradas a `Index 0…18`.

### 4.4 REEMPLAZAR `UpdateRuneRemovalLocks`
```
Make Array: 19 pines [0..18] en orden: Equipment_HeadRuneSlot, _1, _2 … _18
For Loop: First Index 0 · Last Index 17        ← antes 8
  Loop Body: igual que antes (actual + siguiente ocupados → SetRemovalLocked)
Completed: GET LocalRuneSlots [18] → SetRemovalLocked (bLocked: False)   ← antes [9]
```
Fórmula para una familia de N slots: `Last Index = N − 2`, `GET final = [N − 1]`. ⏳ Confirmar que los 19 pines del `Make Array` quedaron completos (la captura se cortaba en `[14]`).

### 4.5 AGREGAR — Select flotante
En `GetNextRuneSlot` hay un `Select` por `E_EquipmentType` desconectado, dejado como alternativa futura. Por eso aparecen dos pines de entrada: `Current Slot Number` (el funcional) y `Current Slot Number 2` (sin uso), y dos `Get` de `_9` en Find References. No es un error.

### 4.6 Deuda técnica — AGREGAR filas
| Problema | Notas | Estado |
|---|---|---|
| Tres patrones distintos de lookup de slot | Unificar `UpdateRuneSlotVisibility` (enum), `GetNextRuneSlot` (Integer) y `UpdateRuneRemovalLocks` (array). Idealmente `GetNextRuneSlot` como fuente única | ⏳ Pendiente de optimización |
| Collapsed Graph de `UpdateRuneSlotVisibility` agrega un paso extra | Eliminarlo o renombrar sus entradas a `Index N` | ⏳ Pendiente |
| Select flotante en `GetNextRuneSlot` | Limpiar o conectar cuando se unifique | ⏳ Pendiente |
| Escalabilidad: 7 familias × 19 slots = 133 slots de runa | Un `Select` Integer con ~140 pines no es manejable (inferencia) | 🟡 Evaluar junto con el rediseño de persistencia |
| Documentar fórmula de `UpdateRuneRemovalLocks` | Ver 4.4 | ✅ Hecho aquí |

### 4.7 Decisiones de diseño — AGREGAR fila
| Decisión | Detalle |
|---|---|
| Probabilidad por slot | 100 a 10 de 10 en 10; 10 a 1 de 1 en 1. 19 slots por familia |

---

## 5 — `13_WIDGET_UI_CraftingQueue.md`

### 5.1 AGREGAR bug nuevo
**Errores `Accessed None` recurrentes en pruebas de la Etapa 1 — 🟡 No destructivos, sin fix aplicado.**

Log observado durante las pruebas de los 19 slots:
```
"Accessed None trying to read property LockedSlot".  Node: Set IsBlocked   Graph: AddRuneToQueue
"Accessed None trying to read property LockedSlot".  Node: Set Visibility  Graph: AddRuneToQueue
"Accessed None".                                      Node: Set Visibility  Graph: AddRuneToQueue
"Accessed None trying to read property PendingLockedSlot". Node: Set IsBlocked   Graph: EventGraph
"Accessed None trying to read property PendingLockedSlot". Node: Set Visibility  Graph: EventGraph
"Accessed None".                                      Node: Set Visibility  Graph: EventGraph
```
**Diagnóstico (inferencia, no verificada):** son los mismos errores del bug "candados fantasma", pero con otra causa. `GetNextRuneSlot` devuelve `None` por diseño para el último slot de la cadena (slot 31). Al asignar una runa ahí, `OnDrop` llama a `AddRuneToQueue` con `LockedSlot = None`. La función no se defiende: intenta escribir sobre `None`, crea igualmente un widget `UI_RuneAssignQueue` y arranca el timer, que luego falla en `OnRuneAssignComplete` con `PendingLockedSlot = None`. El comportamiento en juego "funciona" porque no hay nada que bloquear. **Para confirmar:** reproducir asignando una runa solo en el último slot y ver si el log aparece solo ahí.

**Fix propuesto (no aplicado):** la primera acción de `AddRuneToQueue` debe ser un `Is Valid` sobre `LockedSlot` (ver pasos en la respuesta de chat). Es el "Hardening opcional" que ya estaba en la deuda de este documento, ahora promovido a necesario. Efecto adicional: en el último slot ya no se crea un widget de cooldown huérfano.

### 5.2 Deuda técnica — ACTUALIZAR fila
| Hardening `IsValid(LockedSlot)` en `AddRuneToQueue` | Antes "no bloqueante". Ahora confirmado como origen probable de errores de log en el último slot de cada cadena | 🟡 Pendiente — prioridad media |

---

## 6 — `16_SYSTEM_RuneBinding_WeaponCosmetic.md` (progreso de Fase 1)

### AGREGAR bajo "Qué está prototipado"
- Head ampliado a 19 slots de runa (índices 7 y 14–31), con probabilidad decreciente 100→10 de 10 en 10 y 10→1 de 1 en 1. Probado slot por slot en juego sin bugs destructivos.

### AGREGAR consideración de diseño (inferencia)
La visión del cliente de la Mesa Rúnica como "ventana por pieza" implica potencialmente un solo juego de 19 slots reutilizado para la pieza seleccionada, en lugar de 7 × 19 = 133 slots fijos en `UI_Character`. Si esa visión se adopta, las familias restantes de la Etapa 2 serían trabajo a rehacer. Decidir esto antes de invertir en las 6 familias restantes.

---

## 7 — Plan de Etapa 2 (referencia, aún no ejecutada)

Rangos por familia (19 slots cada una: base + 18 extras):

| Familia | Slot base | Enums extras | Slots extras |
|---|---|---|---|
| Body | 8 | `BodyRuneWord_2`…`_19` | 32–49 |
| Pants | 9 | `PantsRuneWord_2`…`_19` | 50–67 |
| Hands | 10 | `HandsRuneWord_2`…`_19` | 68–85 |
| Feet | 11 | `FeetRuneWord_2`…`_19` | 86–103 |
| Backpack | 12 | `BackpackRuneWord_2`…`_19` | 104–121 |
| Tool | 13 | `ToolRuneWord_2`…`_19` | 122–139 |

Cableado extra respecto al plan original: **`UpdateRuneRemovalLocks` necesita su propio `Make Array` por familia** (omitido en el plan anterior). Piloto recomendado: Body completo antes de replicar.

Orden de agregado de enums obligatorio: Body → Pants → Hands → Feet → Backpack → Tool, para que los índices coincidan con esta tabla.

---
*Actualización generada tras cerrar Etapa 1 — sesión Light Paradox*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
