# 03 — Light Paradox: Project Rules
### Base Asset: EasySurvivalRPGv5
### Scope: UI Data Flow Safety · Boolean Gating · Preventing Disabled UI Updates

---

## Project Context

**Project:** Light Paradox
**Base Asset:** EasySurvivalRPGv5
**Active Technical Goal:** A Boolean gate must be evaluated before any item data is assigned
to `UI_ItemSlot` inside `UI_HUD`. If the Boolean is `False`, execution stops and
no data reaches the slot widget under any circumstance.

The rules below are binding for all work touching the Inventory UI data path.

---

## System Map

```
[Inventory System / Source Data]
        │
        ▼
  [UI_HUD Widget]
    bSlotUpdateEnabled ◄── gate lives here
        │
        ▼ (only if True)
  [UI_ItemSlot Widget]
    ← receives item data
    ← updates icon / count / state
```

No data crosses from `UI_HUD` into `UI_ItemSlot` unless `bSlotUpdateEnabled` is `True`.
This is the single enforced rule. All rules below exist to uphold it.

---

## Section 1 — UI Data Flow Safety

### Rule 1.1 — UI_HUD Is the Only Entry Point for Item Data
No external class, component, or system may call functions on `UI_ItemSlot` directly.
All item data enters the Inventory UI through `UI_HUD` exclusively.

```
❌  PlayerController → UI_ItemSlot.SetItemData(...)
✅  PlayerController → UI_HUD.UpdateInventorySlot(SlotIndex, ItemData)
```

### Rule 1.2 — UI_ItemSlot Exposes One Data-Intake Function
`UI_ItemSlot` must have exactly one public function that accepts item data:
`SetItemData(ItemData, SlotIndex)`. No other function on `UI_ItemSlot` may write
to its visual state. All internal updates (icon, stack count, border highlight) are
triggered from within `SetItemData` only.

### Rule 1.3 — UI_HUD Owns the Slot Widget References
`UI_HUD` holds all references to `UI_ItemSlot` instances (via a named slot array
or panel children). `UI_ItemSlot` widgets hold no references back to `UI_HUD` and
no references to any gameplay object.

### Rule 1.4 — Item Data Is Passed by Value, Not by Reference
When `UI_HUD` calls `SetItemData` on a `UI_ItemSlot`, it must pass a copy of the
item struct, not a live reference into the inventory array. Widget data and inventory
array data must not share the same struct instance.

### Rule 1.5 — No Data Assignment on Invisible Widgets
`UI_HUD` must not push data to a `UI_ItemSlot` that is currently hidden
(`Visibility == Hidden` or `Collapsed`). Check slot visibility before calling
`SetItemData`. Assigning data to a hidden slot wastes cycles and produces
stale state when the slot becomes visible again.

```
[GetSlotVisibility] → Branch (== Visible)
    ├── True  → [Branch: bSlotUpdateEnabled] → SetItemData
    └── False → [Return]
```

---

## Section 2 — Conditional Gating Using Boolean

### Rule 2.1 — The Gate Variable: bSlotUpdateEnabled
A single Boolean variable named `bSlotUpdateEnabled` lives on `UI_HUD`.
It is the sole condition that permits or denies item data reaching `UI_ItemSlot`.

| Property | Value |
|---|---|
| Variable Name | `bSlotUpdateEnabled` |
| Type | Boolean |
| Owner | `UI_HUD` |
| Default Value | `False` |
| Visibility | Private |

Default is `False`. The system that initializes `UI_HUD` must explicitly set
`bSlotUpdateEnabled = True` when the inventory is ready to receive data.
Nothing is permitted to assume it starts `True`.

### Rule 2.2 — Gate Position Is Fixed
The `bSlotUpdateEnabled` Branch must be the **last** check before `SetItemData`
is called — it is not an early filter upstream. Visibility checks, index validation,
and struct validation happen first. The Boolean gate is the final lock.

```
[UpdateInventorySlot called on UI_HUD]
     │
     ▼
[IsValidIndex: SlotIndex]
     ├── False → Return
     ▼
[IsValid: ItemData.DataAsset]
     ├── False → Return
     ▼
[SlotVisibility == Visible]
     ├── False → Return
     ▼
[Branch: bSlotUpdateEnabled]
     ├── False → Return  ← hard stop, nothing executes past here
     └── True  → [UI_ItemSlot.SetItemData(ItemData, SlotIndex)]
```

### Rule 2.3 — Only UI_HUD Reads bSlotUpdateEnabled
No external class evaluates `bSlotUpdateEnabled`. It is a private internal gate.
Other systems do not check it before calling `UI_HUD.UpdateInventorySlot` — they
call unconditionally. `UI_HUD` handles the gate internally.

### Rule 2.4 — Only One Authoritative Setter for bSlotUpdateEnabled
A single function on `UI_HUD` sets this Boolean: `SetSlotUpdateEnabled(bEnabled)`.
No Blueprint outside `UI_HUD` may write directly to the variable. Calls to the
setter must be logged in comments at each call site explaining why the gate is
being opened or closed.

```
✅  UI_HUD.SetSlotUpdateEnabled(True)   ← called by HUD init after data ready
✅  UI_HUD.SetSlotUpdateEnabled(False)  ← called when inventory is closing
❌  UI_HUD.bSlotUpdateEnabled = True    ← direct variable write, never permitted
```

### Rule 2.5 — Gate Must Be Closed Before Widget Is Removed
`SetSlotUpdateEnabled(False)` must be called before `UI_HUD` calls
`RemoveFromParent` on any `UI_ItemSlot`, and before `UI_HUD` itself is removed
from the viewport. Closing the gate first prevents any in-flight event from
writing to a widget mid-removal.

```
[CloseInventory]
  → UI_HUD.SetSlotUpdateEnabled(False)
  → [Clear Slot References]
  → [RemoveFromParent: UI_HUD]
```

---

## Section 3 — Preventing UI Updates When Disabled

### Rule 3.1 — False Gate Means Hard Stop, Not Deferred Execution
When `bSlotUpdateEnabled` is `False`, item data is **discarded**. It is not queued,
not cached on `UI_HUD`, and not retried. The caller is responsible for re-issuing
the update once the gate is open if that behavior is needed.

```
❌  Queue item data and flush when bSlotUpdateEnabled becomes True
✅  Discard. External system re-calls UpdateInventorySlot when appropriate.
```

### Rule 3.2 — No Bypasses, No Overrides
There is no "force update" path, no "emergency write" parameter, and no function
signature variant that skips the gate. Any future request to add a bypass must
be treated as a design defect, not a feature.

### Rule 3.3 — UI_ItemSlot Has No Self-Update Logic
`UI_ItemSlot` must not subscribe to any game event, dispatcher, or tick that
would allow it to update its own display independently. All updates arrive from
`UI_HUD` through `SetItemData`. Self-updating slots bypass the gate entirely
and are strictly forbidden.

```
❌  UI_ItemSlot binds to OnInventoryChanged dispatcher
✅  UI_HUD handles OnInventoryChanged → gates → calls UI_ItemSlot.SetItemData
```

### Rule 3.4 — Tick Is Disabled on UI_ItemSlot
`UI_ItemSlot` must have Tick disabled. No polling, no per-frame checks, no
continuous refresh. Updates are event-driven and gated. Any slot behavior that
appears to require Tick must be re-implemented using timers initiated by `UI_HUD`.

### Rule 3.5 — State After Gate Close Is Frozen, Not Cleared
When `SetSlotUpdateEnabled(False)` is called, existing slot visuals are not reset
or cleared. They remain in their last valid state. Clearing on disable is a
separate explicit operation and must never happen automatically inside the gate setter.

```
[SetSlotUpdateEnabled(False)]
  → SET bSlotUpdateEnabled = False
  → [No other action — slot visuals unchanged]
```

If a visual reset is needed on close, it is called explicitly and separately by
the system that owns the close flow.

### Rule 3.6 — Log Gate State Changes During Development
While in development (not Shipping builds), every call to `SetSlotUpdateEnabled`
must print a screen log with:
- The new Boolean value
- The caller context (use a string parameter or calling function name)
- The frame number

This makes gate transitions visible during iteration and catches unexpected
open/close sequences immediately.

```
Print: "[UI_HUD] bSlotUpdateEnabled → TRUE | Caller: InitHUD | Frame: 1042"
Print: "[UI_HUD] bSlotUpdateEnabled → FALSE | Caller: CloseInventory | Frame: 2187"
```

---

## Section 4 — EasySurvivalRPGv5 Compatibility Rules

### Rule 4.1 — Do Not Modify Base Asset Widget Classes Directly
`UI_HUD` and `UI_ItemSlot` modifications must be made in child Blueprint classes
that inherit from the EasySurvivalRPGv5 originals. The gate variable and all
associated logic live in the child classes only.

| Base Asset Class | Light Paradox Child Class |
|---|---|
| `UI_HUD` (base) | `UI_HUD_LP` (child, add gate here) |
| `UI_ItemSlot` (base) | `UI_ItemSlot_LP` (child, lock intake here) |

### Rule 4.2 — Override, Do Not Duplicate
When EasySurvivalRPGv5 provides a function that routes data to slots, override it
in the child class and insert the gate at the top of the override. Do not copy
the base function body into a new function — call `Parent:FunctionName` after
the gate passes.

```
[Override: UpdateSlot]
  → Branch (bSlotUpdateEnabled)
      ├── False → Return
      └── True  → [Parent: UpdateSlot]
```

### Rule 4.3 — Document Every Override at the Function Level
Every overridden function in `UI_HUD_LP` and `UI_ItemSlot_LP` must have a
Blueprint function description comment stating:
- Why it is overridden
- What the override adds (e.g., "Adds bSlotUpdateEnabled gate before parent call")
- Which base asset version it targets (e.g., "EasySurvivalRPGv5 — verified v5.0")

---

## Enforcement Summary

| Rule | Requirement | Violation Risk |
|---|---|---|
| 1.1 | Data enters only through UI_HUD | Slot updated without gate |
| 1.2 | One intake function on UI_ItemSlot | Gate bypassed via side function |
| 2.1 | bSlotUpdateEnabled default = False | Premature slot updates on init |
| 2.2 | Gate is final check in chain | Data reaches slot before validation |
| 2.4 | One setter function only | Untracked gate writes |
| 2.5 | Gate closed before widget removal | Write to removed widget |
| 3.1 | No queue on False | Deferred updates bypass intent |
| 3.2 | No bypass path exists | Gate rendered meaningless |
| 3.3 | UI_ItemSlot has no self-update | Gate bypassed by slot itself |
| 3.4 | Tick disabled on UI_ItemSlot | Silent per-frame gate bypass |
| 4.1 | Child classes only, never edit base | Base asset corrupted |

---

*End of 03_LIGHTPARADOX_PROJECT_RULES.md*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
