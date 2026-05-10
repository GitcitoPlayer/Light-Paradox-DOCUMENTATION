# 01 — Unreal Engine Blueprint Core Rules

> Scope: Boolean-gated execution control across Branch logic, Widget communication,
> Inventory UI flow, Struct assignment safety, and Boolean gating patterns.
> These are enforcement rules, not tutorials.

---

## 1. Blueprint Branch Logic

### Rule 1.1 — Always Gate Before Acting
Every execution path that modifies state, spawns an actor, or triggers UI **must** pass through a `Branch` node before it executes. No action node should ever be connected directly to an Event pin without a validity check.

```
[Event] → [IsValid / Boolean Check] → [Branch]
                                          ├── True  → [Action]
                                          └── False → [No-op / Early Return]
```

### Rule 1.2 — One Branch Per Condition
Do not chain multiple conditions into a single Branch using long AND/OR expressions unless the logic is trivial. For compound conditions, break each concern into its own named Boolean variable, then AND them explicitly before the Branch.

```
bIsInventoryOpen (bool)
bPlayerHasItem   (bool)
→ AND → Branch → True → Show Slot Widget
```

### Rule 1.3 — Never Branch on Unvalidated References
Before any `Branch` that tests object state, run `IsValid` on the object reference first. A Branch on a null object **will not fire False** — it will throw a Blueprint runtime warning and silently skip.

```
[GetPlayerController] → [IsValid] → Branch
                                       ├── True  → [Cast / Access]
                                       └── False → [Log / Abort]
```

### Rule 1.4 — Use Sequence Nodes to Separate Gate from Effect
When a Branch's True path has multiple effects (e.g., play sound AND open widget AND set variable), use a `Sequence` node. Never chain side-effect nodes in a single long wire from Branch True.

---

## 2. Widget Communication

### Rule 2.1 — Widgets Never Pull Data Directly
Widgets must not call `GetGameInstance`, `GetPlayerController`, or access global state themselves. All data must be **pushed** to the widget via a function or dispatcher called from owning logic.

```
[PlayerController] → [SetInventoryData(Items[])] → [InventoryWidget]
```

### Rule 2.2 — Gate Widget Calls Behind Existence Checks
Before calling any function on a widget reference, check:
1. The widget variable is not null (`IsValid`)
2. The widget is currently added to the viewport (`IsInViewport`)

Calling functions on a widget that is not in the viewport can produce silent failures.

### Rule 2.3 — Use Event Dispatchers for Widget → Game Communication
Widgets must never call gameplay logic directly. All outbound widget events (button clicks, slot selection, close button) must fire a **dispatcher** that the owning controller or component is bound to.

```
[Widget: OnSlotClicked] → [Dispatcher: OnItemSelected(SlotIndex)]
[PlayerController: Bind → OnItemSelected] → [Branch: bInventoryOpen] → [UseItem]
```

### Rule 2.4 — Bind Dispatchers After Widget Creation, Not Before
Always create the widget first, then immediately bind its dispatchers. Binding before creation produces null-reference binding that silently does nothing.

```
[CreateWidget] → [AddToViewport] → [Bind Dispatcher]
```

---

## 3. Inventory UI Flow

### Rule 3.1 — Single Open/Close Authority
Only one object (typically the PlayerController or HUD) may open or close the Inventory widget. No other class calls `AddToViewport` or `RemoveFromParent` on the inventory widget directly.

### Rule 3.2 — Boolean Lock on Open/Close
Use a `bInventoryOpen` boolean to prevent double-open or double-close calls. Every open/close function must branch on this flag before acting, then flip it after.

```
[OpenInventory]
  → Branch (bInventoryOpen == False)
      ├── True  → CreateWidget → AddToViewport → SET bInventoryOpen = True
      └── False → [Return / No-op]
```

### Rule 3.3 — Clear Widget Reference on Close
When closing the Inventory, call `RemoveFromParent`, then **set the widget variable to null**. Holding a stale reference to a removed widget causes false `IsValid` checks to pass.

```
[CloseInventory]
  → RemoveFromParent
  → SET InventoryWidgetRef = null
  → SET bInventoryOpen = False
```

### Rule 3.4 — Populate Before Showing
Always populate widget data (fill slots, set counts, apply icons) **before** calling `AddToViewport`. Never populate after showing — it causes a visible one-frame flash of empty slots.

```
[SetInventoryData] → [PopulateSlots] → [AddToViewport]
```

### Rule 3.5 — Slot Index Bounds Check
Before accessing an inventory array by slot index (from widget click events), always validate the index with `IsValidIndex`. An out-of-bounds access on a TArray crashes the editor and packaged builds.

---

## 4. Struct Assignment Safety

### Rule 4.1 — Never Modify a Struct In-Place via Reference
Blueprint structs are value types. Modifying a struct field directly from a "Get" reference does **not** update the original. Always:
1. Get the struct (copies it)
2. Break the struct
3. Modify the field
4. Make a new struct
5. Set it back

```
GET Items[i] → Break Struct → Modify Field → Make Struct → SET Items[i]
```

### Rule 4.2 — Initialize Structs Before Use
Never leave struct variables at Blueprint defaults and assume fields are zero/false/empty. Explicitly initialize all struct variables in `BeginPlay` or in a dedicated `InitializeInventory` function.

### Rule 4.3 — Validate Struct Fields Before Branching on Them
When a struct contains object references (e.g., `ItemDataAsset`), run `IsValid` on those fields **after** breaking the struct and **before** branching on them. A struct can be non-null while containing a null asset reference.

```
Break ItemStruct → [ItemDataAsset ref] → IsValid → Branch
                                                       ├── True  → Use Asset
                                                       └── False → Log Warning
```

### Rule 4.4 — Use Make Struct for Atomic Updates
When updating multiple fields of a struct that other systems read, use a single `Make Struct` → `SET` operation. Never set fields one at a time across multiple frames — other systems may read a partially-updated struct mid-frame.

---

## 5. Boolean Gating Patterns

### Rule 5.1 — Canonical Gate Pattern
For any system that must block execution until a condition is met, use the following canonical pattern. Do not deviate from it for consistency across the project.

```
[Trigger Event]
  → Branch (bGateCondition == True)
      ├── True  → [Execute Logic] → [Optionally: SET bGateCondition = False to consume gate]
      └── False → [Silent return OR Queue for later]
```

### Rule 5.2 — Name Booleans by the Permission They Grant
Boolean gate variables must be named to express the permission, not the state. Prefer:
- `bCanInteract` over `bInteractionBlocked`
- `bInventoryReady` over `bInventoryLoading`
- `bInputEnabled` over `bInputFrozen`

This ensures Branch True paths are always the "allowed to proceed" path, which is the expected reading direction.

### Rule 5.3 — Never Invert the Gate at the Branch
Do not use `NOT` nodes before a Branch to flip a Boolean at call time. The inversion must live in the variable's semantic meaning. Passing `NOT bIsLocked` into a Branch is a reading hazard — rename to `bIsUnlocked` instead.

```
❌  [NOT bIsLocked] → Branch (True = proceed)
✅  [bIsUnlocked]   → Branch (True = proceed)
```

### Rule 5.4 — Consume One-Shot Gates Immediately
When a Boolean gate is meant to fire only once (e.g., first-pickup trigger, tutorial step), set it to `False` as the **first node** inside the True branch, before any logic executes. This prevents re-entry if the logic itself triggers the same event.

```
Branch (bFirstPickup)
  └── True → SET bFirstPickup = False → [Play Tutorial] → [Grant Item]
```

### Rule 5.5 — Do Not Use Delays as Gates
Never use `Delay` nodes to hold off execution that should be controlled by a Boolean. Delays are non-deterministic under lag, frame-rate variation, and editor PIE resets. Use an explicit Boolean set by the system that finishes (e.g., animation notify, async load complete callback).

```
❌  Delay(0.5s) → OpenChest
✅  [OnChestAnimComplete Notify] → Branch (bChestAnimDone) → OpenChest
```

### Rule 5.6 — Gate Re-entry on Expensive Operations
Any Blueprint path that calls async operations (load asset, query database, HTTP request) must set a `bOperationInProgress` Boolean immediately and branch on it at entry. Clear it in the completion callback only.

```
[RequestItemLoad]
  → Branch (NOT bLoadInProgress)
      ├── True  → SET bLoadInProgress = True → [Async Load] → [OnLoaded: SET bLoadInProgress = False]
      └── False → [Return / Ignore duplicate call]
```

---

## Quick Reference — Boolean Gate Checklist

| Situation | Gate Variable | Set True | Set False |
|---|---|---|---|
| Inventory open/close | `bInventoryOpen` | On widget added | On widget removed |
| Item use cooldown | `bCanUseItem` | On cooldown end | On item used |
| Widget populated | `bInventoryReady` | After data pushed | On inventory closed |
| Async load pending | `bLoadInProgress` | Before async call | In load callback |
| One-shot event | `bEventFired` | First node in True path | Never reset |
| Input enabled | `bInputEnabled` | On gameplay resume | On cutscene/menu open |

---

*End of 01_UNREAL_CORE_RULES.md*
