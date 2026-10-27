# NOTAS PARA LOS .md — Piloto Body (Etapa 2)
### Proyecto: Light Paradox · UE 5.4.4 · Base: EasySurvivalRPGv5
### Cada sección indica el .md destino. Las marcas "inferencia" no están confirmadas por export o captura.

---

## 1 — `08_BLUEPRINT_UI_Character.md` · `UpdateRuneRemovalLocks` — AGREGAR a Deuda técnica

| Problema | Notas | Estado |
|---|---|---|
| Una variable local por familia | Todas las familias reutilizan `LocalRuneSlots`. Un bloque de familia mal enganchado (por ejemplo, el Exec de Body saliendo del `Loop Body` de Head) sobrescribe el array a mitad del loop y los velos dejan de aparecer (síntoma visto en el piloto Body: solo la primera runa de Head recibía velo). Corregir el cableado resolvió el síntoma. **Corrección recomendada, no aplicada:** una variable local por familia (`LocalRuneSlots_Body`, `LocalRuneSlots_Pants`, etc.), misma lección que el Bug 3 | ⏳ Pendiente — prioridad media |

Regla de cableado vigente: el Exec que entra al `SET` del bloque de cada familia debe venir **solo** del `Set Removal Locked (Locked = False)` final del bloque anterior (pin `Completed` del `For Loop`), nunca del `Loop Body`.

---

## 2 — `04_BLUEPRINT_UI_ItemSlot.md` — AGREGAR

### 2.1 Blindaje contra runas de otra familia (aplicado)
Durante el piloto Body, soltar una runa de una familia sobre un slot de otra familia disparaba cooldown y candado. Se aplicó el blindaje en `OnDrop`, dentro de `Cast Success` de `Cast To BP_EquipmentComponent`:
1. `Check Container Slot for Item` (Target: `As BP Equipment Component`), `Slot` ← `GET SlotNumber`, `Item` ← `ItemData` de `BP_DraggedItem`.
2. Su `Return Value` va a un `AND` junto con el `AND` de "es runa Y destino válido", que alimenta el `Branch` que decide entre runa y movimiento normal.

Estado: ✅ Aplicado y probado en el piloto Body. Debe replicarse conceptualmente sin cambios al resto de familias (la función usa el `RestrictionQuery` del slot, que ya se configura por familia).

---

## 3 — `13_WIDGET_UI_CraftingQueue.md` y `14_WIDGET_UI_QueueBlueprint_RuneAssignQueue.md` — AGREGAR bug nuevo

### 🐞 Bug — Dos cooldowns simultáneos (familias distintas) se pisan entre sí

**Detectado:** piloto Body, Etapa 2. **Estado: 🔴 Abierto — fix propuesto, no aplicado.**

**Reproducción:** soltar una runa en una familia (por ejemplo Body) y, durante los 5 s de su cooldown, otra en una familia distinta (por ejemplo Head). Resultado: un widget de cooldown se queda en el Crafting Queue y un candado permanece en el slot que debía desbloquearse.

**Por qué solo ocurre entre familias:** dentro de una misma familia, el slot siguiente permanece bloqueado hasta que termina el cooldown, así que no se pueden solapar dos cooldowns. Con 7 cadenas independientes, sí.

**Causas (la primera documentada, la segunda inferencia sobre el comportamiento del motor):**
1. `AddRuneToQueue` guarda el estado de la asignación en variables únicas (`PendingLockedSlot`, `PendingTarget*`, `PendingSource*`). La segunda llamada sobrescribe lo que la primera dejó. Ya estaba advertido: `Set Timer by Event` no pasa parámetros y `OnRuneAssignComplete` usa `Remove Index (0)`.
2. `Set Timer by Event` con el mismo evento (`OnRuneAssignComplete`) sobre el mismo objeto reemplaza el timer anterior en vez de crear uno nuevo. Solo se dispara un `OnRuneAssignComplete`: desbloquea el slot de la segunda llamada y remueve el widget `[0]` (el de la primera). El widget de la segunda se queda y el slot de la primera conserva su candado. Esto encaja con el síntoma observado.

**Fix propuesto (no aplicado): mover timer y datos al widget `UI_RuneAssignQueue`.** Cada instancia tiene su propio timer y su propio `LockedSlot`, así que no hay estado compartido.
1. `UI_RuneAssignQueue`: variable nueva `LockedSlot` (UI Item Slot, Object Reference). `InitRuneAssignQueue` recibe un input nuevo `LockedSlot` y lo guarda con `SET LockedSlot`.
2. Al final de `InitRuneAssignQueue` (después de `SET StartTime`): `Set Timer by Event` (Time ← `AssignDuration`, Looping = False, Event ← `Create Event` → `OnCooldownComplete`).
3. EventGraph del widget: Custom Event `OnCooldownComplete` (sin parámetros): `Is Valid (LockedSlot)` → `Is Valid`: `SET IsBlocked = False` sobre `LockedSlot` y `LockedSlot → LockIcon → Set Visibility (Collapsed)`; en ambas ramas, después: `Remove From Parent (self)`.
4. `UI_CraftingQueue.AddRuneToQueue`: conectar `LockedSlot` al pin nuevo de `Init Rune Assign Queue` y **eliminar** el `Set Timer by Event` + `Create Event` final (si no, el viejo y el nuevo se ejecutan a la vez).

**Deuda que resuelve al aplicarse:** `Remove Index (0)` hardcodeado en `OnRuneAssignComplete`, y las variables `Pending*` sin uso real.

**Limpieza posterior (no urgente):** `OnRuneAssignComplete` queda sin llamadas (marcar `OBSOLETO`), y `PendingLockedSlot`, `PendingTargetContainer`, `PendingTargetSlot`, `PendingSourceContainer`, `PendingSourceSlot` y `RuneQueueWidgets` quedan sin uso real. Eliminarlos tras validar.

**Nota para `14`:** la decisión "UI_RuneAssignQueue no tiene timer propio" se revierte con este fix. Motivo: con varias cadenas de runa en paralelo, el timer debe vivir en cada instancia.

---

## 4 — `ETAPA2_Procedimiento_Slots_Runa_v2.md` — CORRECCIONES para las familias siguientes

1. **Paso 9.2 (`UpdateEquipmentSlotItem`):** la tabla debe incluir el slot base. El `Select` por slot mapea cada slot al widget que lo **muestra**; no al siguiente:
   | Slot (enum) | Widget |
   |---|---|
   | `F Rune Word` (base) | `Equipment_FRuneSlot` |
   | `F Rune Word 2` (s0) | `_1` |
   | … | … |
   | `F Rune Word 19` (s0+17) | `_18` |
   En el piloto Body, `Body Rune Word` (8) quedó conectado a `_1`: la runa se dibujaba un slot más adelante, dejando un hueco visible. Los mapeos de `UpdateRuneSlotVisibility` y `GetNextRuneSlot` sí van al **siguiente** widget; los de `UpdateEquipmentSlotItem`, no.
2. **Paso 9.5 (`UpdateRuneRemovalLocks`):** usar una variable local por familia (ver sección 1) y verificar el cableado de Exec del bloque (solo desde `Completed`).
3. **Parte 2:** el blindaje es obligatorio (ver sección 2), no condicional. Mover el bloque a la Parte 0 de las familias siguientes: si no está aplicado, aplicarlo antes de probar.

---

## 5 — Bug abierto que sigue vigente (sin cambios)
Runa soltada sobre un slot de runa ya ocupado dispara cooldown y candado (docs 04, 13 y 20). Falta la decisión de diseño: rechazar o permitir reemplazo. Con el piloto Body probado, ahora se aplica a las 2 familias.

---
*Notas generadas durante la sesión Light Paradox — piloto Body, Etapa 2*
