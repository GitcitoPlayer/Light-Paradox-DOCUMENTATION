# ETAPA 2 — Procedimiento v2: 19 slots de runa por familia (Body → Pants → Hands → Feet → Backpack → Tool)
### Proyecto: Light Paradox · UE 5.4.4 · Base: EasySurvivalRPGv5
### Basado en: docs 04, 06, 07, 08, 13, 20 + Etapa 1 (Head) ya probada
### Versión 2: incluye aclaración sobre el nodo `HeadRuneWord` de `CheckContainerSlotForItem`, destino confirmado del `Switch` y estado de cierre de sesión

Convención: **F** = nombre de la familia (Body, Pants, Hands, Feet, Backpack, Tool).
Inglés = nombres literales de Unreal. Las marcas ⚠️ son los puntos donde ya hubo bugs.

---

## ESTADO AL CERRAR LA SESIÓN

| Elemento | Estado |
|---|---|
| Etapa 1 (Head a 19 slots, índices 7 y 14–31) | ✅ Probada slot por slot, sin bugs destructivos |
| Fix `Is Valid (LockedSlot)` en `AddRuneToQueue` | ✅ Aplicado y confirmado: ya no salen los errores `Accessed None`. En el doc 13, el bug "Accessed None en pruebas de Etapa 1" pasa a **Resuelto**, y la fila de hardening también |
| `bDebugForceRuneSuccess` | Implementado para pruebas. ⏳ Dejar Default Value en False antes de entregar |
| Etapa 2 | ⏳ Pendiente: piloto Body |
| Resultado de la prueba de runa de otra familia (Parte 2) | ⏳ Se confirmará al terminar el piloto Body |
| Actualización de los .md 04, 06, 07, 08, 13, 16 | Ver `ACTUALIZACION_MD_Etapa1_Head19Slots.md` (aplicar primero) y actualizar tras cerrar la Etapa 2 |

---

## ACLARACIÓN: el nodo `HeadRuneWord` en `CheckContainerSlotForItem`

**Pregunta:** el nodo `Equal (Enum)` con el dropdown `HeadRuneWord` solo referencia Head. ¿Debe actualizarse en la Parte 0?

**Respuesta: no se modifica.** Es el bloque de la familia Head (`Equal HeadRuneWord` + `>= 14` + `<= 31` + `AND`). Si cambiaras ese dropdown a otra familia, Head dejaría de funcionar. La estructura es **un bloque por familia, todos alimentando el mismo `OR`**:

```
OR (8 pines al final)
  pin A ← Equal (EquipmentType == Slot)               ← existente, cubre slots 0–13
  pin B ← AND( == HeadRuneWord,     >= 14,  <= 31 )   ← existente (tu captura)
  pin C ← AND( == BodyRuneWord,     >= 32,  <= 49 )   ← Paso 7, Body
  pin D ← AND( == PantsRuneWord,    >= 50,  <= 67 )   ← Pants
  pin E ← AND( == HandsRuneWord,    >= 68,  <= 85 )   ← Hands
  pin F ← AND( == FeetRuneWord,     >= 86,  <= 103 )  ← Feet
  pin G ← AND( == BackpackRuneWord, >= 104, <= 121 )  ← Backpack
  pin H ← AND( == ToolRuneWord,     >= 122, <= 139 )  ← Tool
```

Cada familia agrega su propio bloque en el **Paso 7** copiando estos 4 nodos de Head. Nada de esto va en la Parte 0.

**Observación (inferencia, NO aplicar ahora):** la función también hace `AND` con `ParentResult`, y el Parent valida el `RestrictionQuery` de cada slot. Eso significa que el rechazo de una runa de otra familia probablemente ya lo hace el Parent, y que estos bloques por familia solo existen para destrabar el caso `EquipmentType != Slot`. Una versión genérica (un solo bloque para todas las familias) podría reemplazar los 7 bloques, pero no está probada y requiere comparar enums como números. Queda como pendiente de optimización para evaluar **después** de cerrar el piloto, junto con la unificación de los `Select`.

---

## PARTE 0 — Antes de empezar (una sola vez)

### 0.1 Reglas de oro de los enums
1. Los enumeradores **siempre se agregan al final**. Nunca insertar en medio, nunca reordenar.
2. **El orden de familias es fijo:** Body → Pants → Hands → Feet → Backpack → Tool. Los rangos de las tablas dependen de ese orden.
3. Agrega los enums de una familia **solo cuando vayas a terminar esa familia**. No agregues los de la siguiente por adelantado.
4. Cada familia agrega **18 enumeradores** (`FRuneWord_2` a `FRuneWord_19`). Si decides no hacer Tool, simplemente no agregas sus 18 y no se desperdicia ningún índice.
5. Los nombres en `E_EquipmentSlot` y `E_EquipmentType` deben ser **idénticos y en el mismo orden**.

### 0.2 Tabla maestra de rangos
| Familia | Slot base | Enums a agregar | Slots extra | `Equipment Slots` y `Container Slot Settings` quedan en | Widgets nuevos |
|---|---|---|---|---|---|
| Body | 8 | `BodyRuneWord_2`…`_19` | 32–49 | **50** elementos | `_1`…`_18` |
| Pants | 9 | `PantsRuneWord_2`…`_19` | 50–67 | **68** | `_1`…`_18` |
| Hands | 10 | `HandsRuneWord_2`…`_19` | 68–85 | **86** | `_1`…`_18` |
| Feet | 11 | `FeetRuneWord_2`…`_19` | 86–103 | **104** | `_1`…`_18` |
| Backpack | 12 | `BackpackRuneWord_2`…`_19` | 104–121 | **122** | `_1`…`_18` |
| Tool | 13 | `ToolRuneWord_2`…`_19` | 122–139 | **140** | `_1`…`_18` |

Fórmulas (s0 = primer slot extra de la familia):
- Widget `_K` (K = 1…18) tiene `SlotNumber = s0 + K − 1`.
- `SuccessChance` de `_K`: `_1`=90, `_2`=80, `_3`=70, `_4`=60, `_5`=50, `_6`=40, `_7`=30, `_8`=20, `_9`=10, `_10`=9, `_11`=8, `_12`=7, `_13`=6, `_14`=5, `_15`=4, `_16`=3, `_17`=2, `_18`=1. El widget base lleva `100`. (Se copian de Head, ya vienen bien.)
- El enum `FRuneWord_N` (N = 2…19) es el slot `s0 + N − 2`.

### 0.3 Inventario de lugares que usan el enum (evita repetir el Bug 4)
1. En Content Browser, clic derecho sobre `E_EquipmentSlot` → **Reference Viewer**.
2. Anota **todos** los Blueprints que aparecen. Mínimo esperados: `BP_EquipmentComponent`, `BP_Character_Player`, `UI_Character`, `UI_ItemSlot`.
3. Durante el piloto (Body), abre cada uno y busca nodos `Switch` o `Select` por `E_EquipmentSlot`. Cada valor nuevo del enum necesita su pin conectado donde corresponda. Un pin sin conectar no da error de compilación ni de consola.

### 0.4 Verificaciones de datos (una sola vez)
1. **`DT_GameplayTags_Items`:** confirma que existen `EasyRPG.Items.Equipment.BodyRuneWord`, `PantsRuneWord`, `HandsRuneWord`, `FeetRuneWord`, `BackpackRuneWord`, `ToolRuneWord`. Si falta alguno, agrégalo.
2. **`Local Equipment Types`** (variable local de `Item Is Equipment`, ver doc 06): el doc 06 solo documenta entradas hasta `PantsRuneWord`. Abre su Default Value y asegura que existan estas 6 entradas (tag → valor):
   - `…Equipment.BodyRuneWord` → `BodyRuneWord`
   - `…Equipment.PantsRuneWord` → `PantsRuneWord`
   - `…Equipment.HandsRuneWord` → `HandsRuneWord`
   - `…Equipment.FeetRuneWord` → `FeetRuneWord`
   - `…Equipment.BackpackRuneWord` → `BackpackRuneWord`
   - `…Equipment.ToolRuneWord` → `ToolRuneWord`
   Agrega las que falten (botón **+** del Map). Sin esto, `Item Is Equipment` no reconoce la runa y devuelve `Result = false`. Los slots extra **no** necesitan entradas nuevas (comparten el tag de su familia).
3. **`Container Slot Settings` índices 8 a 13:** confirma que su `RestrictionQuery` lleva `BodyRuneWord`, `PantsRuneWord`, `HandsRuneWord`, `FeetRuneWord`, `BackpackRuneWord`, `ToolRuneWord` respectivamente.
4. **Ítems de prueba:** al menos 2 Rune Words con el tag de cada familia (en `DT_Items`) y el cosmético de esa familia. Si no existen, duplica una fila de runa existente y cambia su tag (inferencia: no está documentado cómo están armados esos ítems).

### 0.5 Cambio único en `UI_ItemSlot.OnDrop` (UNA vez, antes del piloto)
Objetivo: que `IsValidRuneDestination` acepte todos los slots de runa de cualquier familia sin un OR por familia. Razón: dentro del `EquipmentContainer`, todo `SlotNumber >= 7` es un slot de runa (7–13 base, 14+ extras), y el `Cast To BP_EquipmentComponent` ya descarta otros contenedores.

1. Abre `UI_ItemSlot` → función `OnDrop`.
2. Localiza el bloque `IsValidRuneDestination`: `Equal (Integer) GET SlotNumber == 7`, `>= 14`, `<= 31`, `AND`, `OR`.
3. Clic derecho en el espacio vacío → busca `>= (Integer)` y créalo.
4. Pin A: `GET SlotNumber`. Pin B: escribe `7`.
5. Localiza el `AND` final que combina `Does Container Match Tag Query` con el resultado del `OR` viejo.
6. Desconecta de ese `AND` el cable que viene del `OR` viejo (Alt + clic sobre el cable).
7. Conecta la salida del nuevo `>=` a ese pin del `AND` final.
8. No borres los nodos viejos todavía: déjalos desconectados dentro de un Comment box con el texto `OBSOLETO - reemplazado por >= 7 (Etapa 2)`.
9. Compile + Save.

---

## PARTE 1 — Procedimiento por familia (repetir en orden)

Valores de cada familia: tabla 0.2. Los ejemplos usan **Body**.

### Paso 1 — `E_EquipmentSlot`
1. Abre el asset `E_EquipmentSlot`.
2. Clic en **Add Enumerator** 18 veces, al final.
3. Renombra cada uno (Display Name), en orden: `BodyRuneWord_2`, `BodyRuneWord_3`, … `BodyRuneWord_19`.
4. Verifica que el último índice sea el de la tabla (Body → 49). Save.

### Paso 2 — `E_EquipmentType`
Repite exactamente el Paso 1: mismos 18 nombres, mismo orden. Save.

### Paso 3 — Recompilar dependientes
Abre y compila `BP_EquipmentComponent`, `BP_Character_Player`, `UI_Character`, `UI_ItemSlot`. Corrige cualquier error (normalmente un `Switch`/`Select` que necesita Refresh Node).

### Paso 4 — `Equipment Slots`
1. `BP_Character_Player` → Components → `EquipmentContainer` → Details → `Equipment Slots`.
2. Clic en **+** 18 veces.
3. En cada elemento nuevo elige su enum: `[32]` = `BodyRuneWord_2`, `[33]` = `BodyRuneWord_3`, … `[49]` = `BodyRuneWord_19`.
4. ⚠️ El texto "N Array elements" debe coincidir con la tabla 0.2 (Body = 50).
5. Compile + Save.

### Paso 5 — `Container Slot Settings`
1. Mismo panel Details.
2. ⚠️ Los elementos **no se pueden insertar en medio**. Duplica **siempre el último elemento**: menú ⋮ → **Duplicate** (la copia queda al final).
3. Primera copia (Body: duplicar el índice 31): edita su `RestrictionQuery`: `All Expressions Match` → `All Tags Match` → tag `EasyRPG.Items.Equipment.BodyRuneWord`. Cambia `Name` a `Body Rune Slot`.
4. Duplica **ese nuevo último elemento** 17 veces más.
5. ⚠️ Verifica: tamaño igual a la tabla 0.2, sin elementos vacíos, e índices extremos del rango (32 y 49 para Body) con el tag correcto.
6. Compile + Save.

### Paso 6 — `Switch on E_EquipmentSlot` en `EquipmentChanged` ⚠️ (causa del Bug 4)
Confirmado por ti: existe un nodo `Set w/ Notify (Equiped F Mesh)` por cada cosmético, igual que `Equiped Head Mesh`. Se replica el mismo patrón de Head.
1. `BP_Character_Player` → `EquipmentChanged` → nodo `Switch on E_EquipmentSlot`.
2. Clic derecho sobre el nodo → **Refresh Node**. Aparecen los pines nuevos (`Body Rune Word 2` … `Body Rune Word 19`).
3. Conecta los 18 pines nuevos al `Set w/ Notify (Equiped Body Mesh)` al que ya va el pin base `Body Rune Word`. Ese es el mismo patrón que Head: todos sus `Head Rune Word N` van a `Equiped Head Mesh`.
4. Si el pin base `Body Rune Word` estuviera sin conectar, usa el mismo destino que el pin `Body` del cosmético y avísame, porque sería un hueco ya existente.
5. Usa Reroute si el cableado se vuelve ilegible.
6. ⚠️ Verificación final: ningún pin de ejecución nuevo debe quedar suelto.
7. Compile + Save.
Para Tool, el nodo equivalente puede tener otro nombre; usa el que usa su pin base.

### Paso 7 — `CheckContainerSlotForItem` (override) en `BP_EquipmentComponent` ⚠️ (causa del Bug 2)
**No modifiques el bloque de Head.** Se agrega un bloque nuevo por familia (ver la aclaración arriba).
1. Selecciona los 4 nodos del bloque Head: `Equal (Enum)` con `HeadRuneWord`, `>= 14`, `<= 31` y el `AND`. Ctrl+C, Ctrl+V.
2. En el `Equal (Enum)` pegado, cambia el dropdown de `HeadRuneWord` a `BodyRuneWord`.
3. Cambia `14` por `32` y `31` por `49` (valores s0 y s0 + 17 de la tabla 0.2).
4. Conecta el pin `EquipmentType` (cable teal que alimenta al `Equal` de Head) al primer pin del `Equal` nuevo. Conecta el pin `Slot` (cable verde que alimenta a los `>=` y `<=` de Head) al primer pin del `>=` nuevo y al del `<=` nuevo.
5. En el nodo `OR`, clic en **Add pin** y conecta la salida del `AND` nuevo a ese pin.
6. Compile + **Save**. ⚠️ Cierra y reabre la función para confirmar que el bloque sigue presente (el Bug 2 reapareció una vez por no guardar).

### Paso 8 — Widgets nuevos en `UI_Character`
1. Abre `UI_Character` → Designer → Hierarchy. Verifica que el base `Equipment_BodyRuneSlot` está dentro de `Body Rune Box` (VerticalBox).
2. Selecciona `Equipment_HeadRuneSlot_1` … `_18` (clic en el primero, Shift+clic en el último). Ctrl+C.
3. Selecciona `Body Rune Box` y Ctrl+V. Los widgets quedan dentro, después del base.
4. Renombra cada uno (F2) a `Equipment_BodyRuneSlot_1` … `_18`. ⚠️ Verifica **Is Variable** marcado en todos.
5. Cambia **solo** el `SlotNumber` según la fórmula (Body: `_1`=32, `_2`=33, … `_18`=49). `SuccessChance` ya viene copiado (90 … 1) y `Visibility` `Collapsed`. Verifícalos.
6. Verifica el base `Equipment_BodyRuneSlot`: `SlotNumber = 8`, `SuccessChance = 100`, `Collapsed`.
7. Compile.

### Paso 9 — Los 5 lugares de cableado en `UI_Character`
Al final haz Find References sobre `Equipment_BodyRuneSlot_18`: debe aparecer una vez en cada uno de los 5 lugares.

**9.1 `EventGraph` → `SetEquipmentReference`**
1. Localiza el último nodo de la cadena de `Set Container Reference` (su Exec de salida sin conexión).
2. Por cada widget nuevo (`_1` a `_18`): copia un `Set Container Reference` existente, encadena su Exec al final, `Target` ← `GET Equipment_BodyRuneSlot_N`, `Container` ← la variable `Equipment Reference`. Igual que hiciste con `_10`–`_18` de Head.

**9.2 `UpdateEquipmentSlotItem`**
1. En el `Select` que alimenta `Update Item Data`, verifica que `Body Rune Word` (8) → `Equipment_BodyRuneSlot` (probablemente ya está).
2. Conecta cada opción nueva:

| Enum / Slot | Widget |
|---|---|
| `Body Rune Word 2` (32) | `_1` |
| `Body Rune Word 3` (33) | `_2` |
| … | … |
| `Body Rune Word 19` (49) | `_18` |

Regla: el enum `FRuneWord_N` va al widget `_(N−1)`. Si el Select es por Integer, la opción es el `SlotNumber`.

**9.3 `UpdateRuneSlotVisibility`** (el enum es el del slot que **recibe** el ítem; el widget es el que se **revela**)
1. Abre el Collapsed Graph. Las opciones por enum ya aparecen en el Select. Crea una entrada nueva en el nodo `Inputs` por cada widget (arrastra desde el pin de la opción hacia el nodo `Inputs`).
2. Conecta:

| Pin del Select | Widget que revela |
|---|---|
| `Body` | `Equipment_BodyRuneSlot` |
| `Body Rune Word` (8) | `_1` |
| `Body Rune Word 2` (32) | `_2` |
| `Body Rune Word 3` | `_3` |
| … | … |
| `Body Rune Word 18` (48) | `_18` |
| `Body Rune Word 19` (49) | **ninguno** (sin conectar) |

3. Sal del Collapsed Graph. En el nodo colapsado conecta cada pin nuevo a `GET Equipment_BodyRuneSlot…`.
4. ⚠️ El pin `Body` (base) estaba sin conectar en tu captura; conéctalo.

**9.4 `GetNextRuneSlot`** (Select por Integer, `Current Slot Number`)
1. En el `Select`, clic en **Add pin** hasta que exista la opción del último slot de la familia (Body: Option 49).
2. Conecta:

| Option (SlotNumber) | Widget que devuelve |
|---|---|
| 8 | `Equipment_BodyRuneSlot_1` |
| 32 | `_2` |
| 33 | `_3` |
| … | … |
| 48 | `_18` |
| 49 | **sin conectar (None)** |

Regla: slot base → `_1`; slot `s0 + K − 1` → `_(K+1)`; último slot de la familia → `None`.
3. No toques el `Select` flotante de `E_EquipmentType`.

**9.5 `UpdateRuneRemovalLocks`** (cada familia necesita su propio bloque, encadenado)
1. Selecciona el bloque Head: `SET Local Rune Slots`, `For Loop`, los dos `GET`, el `+ 1`, los `Item Is Valid`, el `AND`, `Set Removal Locked` del loop, y el `GET [18]` con su `Set Removal Locked (Locked = False)` final. **No copies el `Make Array` ni sus `Get` de widgets.** Ctrl+C, Ctrl+V.
2. Encadena el Exec: salida del último `Set Removal Locked (False)` del bloque anterior → entrada del `SET Local Rune Slots` del bloque pegado.
3. Reconecta el pin `Container` (Function Entry) a los nodos `Get Item` del bloque pegado.
4. Crea un `Make Array` nuevo con **19 pines** `[0]`…`[18]`, en este orden: `Equipment_BodyRuneSlot`, `_1`, `_2` … `_18`. Conecta su salida al `SET Local Rune Slots` pegado.
5. Verifica `For Loop`: First Index `0`, Last Index `17`; `GET` final en `[18]`, `Locked` sin marcar.
6. Compile + Save.

### Paso 10 — Compile + Save All
`Ctrl + Shift + S`. Confirma que el ícono de guardado queda sin resaltar.

### Paso 11 — Prueba de la familia
Activa `bDebugForceRuneSuccess = True` en `UI_ItemSlot`. El cooldown de 5 s sigue activo (19 slots ≈ 95 s).
1. Equipa el cosmético de la familia en el Altar → se desbloquea el slot base.
2. Asigna runas en: base, `_1`, `_2`, `_9` → `_10` (cambio de 10 a 1 en 1), y `_16`, `_17`, `_18`.
3. Con dos runas del mismo tag: el atributo suma ambos valores (regresión Bug 4).
4. Cada slot revela al siguiente y el candado cae sobre el correcto. En `_18` no hay candado ni errores `Accessed None`.
5. Quita runas en orden descendente; las anteriores muestran el velo.
6. Cierra y reabre el editor: el bloque del Paso 7 sigue presente.

---

## PARTE 2 — Piloto Body: runa de otra familia (condicional)

Solo con Body terminado y una runa de Head:
1. Suelta una runa **Body** sobre un slot de runa **Head** ya desbloqueado.
2. Suelta una runa **Head** sobre un slot de runa **Body**.
3. **Si se rechaza limpiamente** (el ítem se queda, sin cooldown ni candado): no hace falta nada.
4. **Si aparece cooldown/candado o el ítem entra al slot equivocado:** aplica el blindaje en `UI_ItemSlot.OnDrop`, dentro de la rama `Cast Success` de `Cast To BP_EquipmentComponent`:
   1. Del pin `As BP Equipment Component`, llama a `Check Container Slot for Item`.
   2. `Slot` ← `GET SlotNumber`. `Item` ← `ItemData` de `BP_DraggedItem`.
   3. Crea un `AND` adicional: pin A ← el `AND` actual de "es runa Y destino válido", pin B ← `Return Value` de `Check Container Slot for Item`.
   4. Esa salida alimenta el `Branch` que decide entre "runa" y "movimiento normal".
   5. Compile + Save y repite las dos pruebas.

> No está documentado dónde se valida eso al mover ítems, por eso se prueba. **Anota aquí el resultado:** ______

---

## PARTE 3 — Orden y cierre

1. **Piloto: Body completo** (Pasos 1–11 + Parte 2).
2. Replicar en orden: Pants → Hands → Feet → Backpack → Tool.
3. Tool: la estructura de slots no depende de que armas/herramientas funcionen como cosméticos (Fase 3 del doc 16). Sus runas quedan ligadas al slot de equipamiento, no a la herramienta activa en mano. Deuda ya registrada.
4. Al terminar cada familia, anota longitudes finales de `Equipment Slots` y `Container Slot Settings` (50, 68, 86, 104, 122, 140) y cualquier desviación.
5. Al cerrar todo, pedir la actualización de los .md 04, 06, 07, 08, 13 y 16.
6. Antes de entregar al cliente: `bDebugForceRuneSuccess` Default Value en **False**; eliminar el `Print String "No es runa"`.

---

## Pendientes de optimización (no bloquean la Etapa 2)
| Pendiente | Notas |
|---|---|
| Unificar los 3 patrones de lookup de slot (`UpdateRuneSlotVisibility` enum, `GetNextRuneSlot` Integer, `UpdateRuneRemovalLocks` array) | `GetNextRuneSlot` como fuente única. Con 140 slots el Select Integer será difícil de mantener |
| Collapsed Graph de `UpdateRuneSlotVisibility` | Eliminarlo, o renombrar sus entradas a `Index N` si se conserva |
| `Select` flotante de `E_EquipmentType` en `GetNextRuneSlot` | Se dejó a propósito, para reemplazar el Integer en el futuro |
| Versión genérica del OR de `CheckContainerSlotForItem` | Ver aclaración al inicio. Evaluar tras el piloto |
| Persistencia de runas por ítem y slot de Tool dinámico | Doc 16, Fases 2 y 3. Prioridad de refactorización |

---

## Errores ya vividos, resumen
- Enums fuera de orden o no agregados al final.
- `Equipment Slots` más corto que el índice máximo (Bug 4).
- Pines sin conectar en `Switch on E_EquipmentSlot` (Bug 4).
- Familia sin su bloque en `CheckContainerSlotForItem` (Bug 2).
- No guardar tras un cambio (Bug 2 desapareció una vez).
- Olvidar `UpdateRuneRemovalLocks` al contar los lugares de cableado.
- Sugerencia errónea de cambiar `IsBlocked` a `bIsLocked` (la variable correcta es `IsBlocked`).
