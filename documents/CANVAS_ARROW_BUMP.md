# Canvas Arrow-Key Bumping

Port the arrow-key shape-movement feature from the diagram project to the layout project.

## 1. Goal

Allow users to nudge selected elements one pixel (or one grid-snap increment) per arrow-key press, with Shift+arrow increasing the step size. The feature must provide live visual feedback on every keydown and batch the undo snapshot per contiguous key-hold gesture (same as mouse-drag semantics).

## 2. Reference Implementation: diagram project

The diagram project (`/Users/bflbarlow/Websites/diagram/app.js`) has a complete, tested arrow-key bumping system:

| Concept | Location | Description |
|---------|----------|-------------|
| `ARROW_DELTA` mapping | `app.js:34-38` | Maps `ArrowUp`/`Down`/`Left`/`Right` to `{dx, dy}` vectors |
| `CONFIG.arrowStep` | `app.js:28` | Default 1 px per press |
| `CONFIG.arrowStepBig` | `app.js:29` | Default 10 px per Shift+press |
| Arrow move in `keydown` | `app.js:3393-3418` | Core logic: read delta, compute step, apply to each unlocked selected shape, `clampShape`, call `render()` + `scheduleArrowUndo()` |
| Undo batching (`scheduleArrowUndo`) | `app.js:919-925` | Sets `_arrowMoveActive` on first press of a gesture, sets 500ms fallback timer |
| Undo commit (`commitArrowMove`) | `app.js:928-931` | Clears active flag + timer, calls `pushUndo()` |
| Undo commit via `keyup` | `app.js:3422` | `document.addEventListener('keyup', ...)` calls `commitArrowMove()` when arrow key released |

### 2.1 Step size logic (diagram `app.js:3395-3398`)

```js
var step = e.shiftKey
    ? (S.snapToGrid ? S.gridSize : CONFIG.arrowStepBig)
    : CONFIG.arrowStep;
```

- **No modifier:** `CONFIG.arrowStep` (1 px)
- **Shift pressed:** If `snapToGrid` is on, use `S.gridSize` (20 px); otherwise use `CONFIG.arrowStepBig` (10 px)

### 2.2 Position update (diagram `app.js:3400-3411`)

```js
shapeSel().forEach(function(id) {
    var s = findShape(id);
    if (!s || s.locked) return;
    if (S.snapToGrid && e.shiftKey) {
        s.x = snap(s.x + d.dx * step);
        s.y = snap(s.y + d.dy * step);
    } else {
        s.x += d.dx * step;
        s.y += d.dy * step;
    }
    clampShape(s);
    moved = true;
});
```

- Each element's `x`/`y` is updated directly
- If snap-to-grid is on **and** Shift is held, use `snap()` on the result (snap to grid cell)
- Otherwise, plain addition
- `clampShape()` keeps the shape within valid bounds
- `moved` flag gates `render()` + `scheduleArrowUndo()`

## 3. Layout-Specific Adaptations

### 3.1 Direction mapping

Identical logic — reuse the same `ARROW_DELTA` object:

```js
var ARROW_DELTA = {
    ArrowUp:    { dx: 0,  dy: -1 },
    ArrowDown:  { dx: 0,  dy: 1  },
    ArrowLeft:  { dx: -1, dy: 0  },
    ArrowRight: { dx: 1,  dy: 0  }
};
```

Place this near the top of the JS file, after the `CONFIG` object (currently around line 973).

### 3.2 Config constants

Add to `CONFIG` object (currently defined at lines 972-975):

```js
var CONFIG = {
    panMargin: 100,
    zoomStep: 0.1,
    arrowStep: 1,       // px per arrow-key press (no modifier)
    arrowStepBig: 10    // px per Shift+arrow-key press
};
```

**Layout-specific note:** Layout has no `snapToGrid` toggle or `gridSize` state. Its `snap()` function (line 1085) always snaps to `GRID` (10px). The arrow-key step logic should be tuned accordingly:

| Condition | Step size | Behavior |
|-----------|-----------|----------|
| Arrow alone | `CONFIG.arrowStep` (1 px) | Free-form nudge, no grid snap |
| Shift+arrow | `CONFIG.arrowStepBig` (10 px) | Free-form 10px nudge, no grid snap |

Unlike diagram, there is no `snapToGrid`-dependent branch because layout's `snap()` is always active for mouse-drag operations (line 2855, 2874) but arrow keys should **not** snap — they nudge in raw pixel increments so the user can position freely. The Shift modifier simply makes the nudge bigger. (If grid snapping on Shift+arrow is desired later, `GRID` (10) can substitute for `S.gridSize`.)

### 3.3 Element lookup

Diagram uses `findShape(id)` → returns shape object directly. Layout uses `getEl(id)` → returns `{ el, page }` where `el` is the element data. The arrow key handler will use `getEl()` and operate on `result.el`:

```js
state.selectedElements.forEach(function(id) {
    var res = getEl(id);
    if (!res) return;
    var el = res.el;
    // ... modify el.x, el.y ...
});
```

### 3.4 DOM update

Diagram calls `render()` after each arrow move — a full re-render. Layout should follow its drag pattern: directly update the DOM element's style after `updEl()`, avoiding a full `renderPage()` for performance during rapid keydowns.

The existing drag code (lines 2855-2877) shows the pattern:

```js
updEl(id, { x: snap(...), y: snap(...) });
// Direct DOM update:
var de = ct.querySelector('[data-id="' + id + '"]');
if (de) { de.style.left = ...; de.style.top = ...; }
```

For arrow bumping, use the same approach:

```js
updEl(id, { x: el.x, y: el.y });
var de = ct.querySelector('[data-id="' + id + '"]');
if (de) { de.style.left = el.x + 'px'; de.style.top = el.y + 'px'; }
```

Where `ct` is `$('page-content')`.

### 3.5 Undo batching

Layout's undo system works differently from diagram's:

| | diagram | layout |
|--|---------|--------|
| Undo stack | `S.undoStack` in state | `state.undoStack` |
| Push undo | `pushUndo()` — pushes current state snapshot | `saveSnapshot()` — pushes snapshot, clears redo, sets `state.dirty = true` |
| Re-render after undo | `render()` | `emit()` → `refresh()` (full rebuild) |

The arrow-key undo batching pattern from diagram:

```js
// On first arrow keydown of a gesture:
saveSnapshot();   // captures the pre-move state exactly once

// On subsequent keydowns (same gesture):
// ... just update positions, no snapshot ...

// On keyup:
// ... nothing extra needed; the snapshot was already pushed at gesture start
```

This is actually simpler than diagram's approach because layout's `saveSnapshot()` pushes the **current** state at the start of the gesture, and all subsequent keydowns modify state from that baseline. On undo, the user sees the pre-gesture positions. On redo, they see the post-gesture final positions.

The pattern:

```js
// Module-level tracking variable (add near other state tracking)
var _arrowMoveActive = false;

// In keydown handler:
if (ARROW_DELTA[e.key] && state.selectedElements.length > 0) {
    preventIfEditable(e);
    if (!_arrowMoveActive) {
        _arrowMoveActive = true;
        saveSnapshot();  // capture pre-move state once
    }
    // ... apply moves ...
    // ... update DOM ...
}

// In keyup handler:
if (ARROW_DELTA[e.key]) {
    _arrowMoveActive = false;
}
```

**No timer fallback needed** for layout because:
- Layout's undo is not a `logAction`/`pushUndo` two-step; `saveSnapshot()` is self-contained
- The 500ms timer in diagram was a fallback for when `keyup` is missed (focus loss, alt-tab). This edge case is minor and can be omitted in the initial implementation. Add it only if focus-loss scenarios prove problematic.

### 3.6 Active element guard

Arrow keys must be blocked when focus is inside an editable text element (same as the existing `delete`/`backspace` guard at line 2948). The current code pattern:

```js
var ae = document.activeElement;
if (ae && (ae.isContentEditable || ae.tagName === 'INPUT' || ae.tagName === 'TEXTAREA' || ae.tagName === 'SELECT')) return;
```

Use this same guard for arrow-key moves. Create a small helper to avoid repeating:

```js
function isEditing() {
    var ae = document.activeElement;
    return ae && (ae.isContentEditable || ae.tagName === 'INPUT' || ae.tagName === 'TEXTAREA' || ae.tagName === 'SELECT');
}
```

(Optional — can inline the guard if preferred.)

## 4. Implementation Plan

### Step 1: Add `ARROW_DELTA` mapping

Insert after the `CONFIG` object (around line 975):

```js
var ARROW_DELTA = {
    ArrowUp:    { dx: 0,  dy: -1 },
    ArrowDown:  { dx: 0,  dy: 1  },
    ArrowLeft:  { dx: -1, dy: 0  },
    ArrowRight: { dx: 1,  dy: 0  }
};
var _arrowMoveActive = false;
```

### Step 2: Extend `CONFIG` with arrow step constants

```js
var CONFIG = {
    panMargin: 100,
    zoomStep: 0.1,
    arrowStep: 1,
    arrowStepBig: 10
};
```

### Step 3: Add arrow-key handler in `onKey`

Insert the arrow-key block at the end of `onKey()`, after the alignment shortcuts (after the `if (ctrl && e.shiftKey)` block, around line 2993):

```js
// ===== Arrow-key element movement =====
if (ARROW_DELTA[e.key] && state.selectedElements.length > 0) {
    var ae = document.activeElement;
    if (ae && (ae.isContentEditable || ae.tagName === 'INPUT' || ae.tagName === 'TEXTAREA' || ae.tagName === 'SELECT')) return;
    e.preventDefault();
    if (!_arrowMoveActive) {
        _arrowMoveActive = true;
        saveSnapshot();
    }
    var d = ARROW_DELTA[e.key];
    var step = e.shiftKey ? CONFIG.arrowStepBig : CONFIG.arrowStep;
    var moved = false;
    state.selectedElements.forEach(function(id) {
        var res = getEl(id);
        if (!res) return;
        var el = res.el;
        el.x += d.dx * step;
        el.y += d.dy * step;
        el.x = Math.max(0, el.x);
        el.y = Math.max(0, el.y);
        moved = true;
    });
    if (moved) {
        state.selectedElements.forEach(function(id) {
            updEl(id, {});
            var de = $('page-content').querySelector('[data-id="' + id + '"]');
            if (de) {
                var el = getEl(id);
                if (el) {
                    de.style.left = el.el.x + 'px';
                    de.style.top = el.el.y + 'px';
                }
            }
        });
    }
    return;
}
```

**Notes:**
- `updEl(id, {})` with an empty patch triggers the existing update path (validates index, sets `state.dirty`, etc.) without overwriting any fields
- The element's `x`/`y` was already modified directly on the object (via `res.el.x += ...`), so `updEl` sees the new values
- `Math.max(0, ...)` prevents elements from moving off the page top/left edge. Layout has no `clampShape()` equivalent; this minimal clamp replaces it. For full bounds clamping (prevent movement beyond page width/height), add bottom/right clamping as well (see §4.2)
- The `return` prevents the arrow key from falling through to tool shortcuts (`v`, `t`, `l`)

### Step 4: Add `keyup` handler to commit the undo batch

Add after the `document.addEventListener('keydown', onKey)` registration around line 3279:

```js
document.addEventListener('keyup', function(e) {
    if (ARROW_DELTA[e.key]) _arrowMoveActive = false;
});
```

### 4.1 Step size reference

| Input | Step px | `e.shiftKey` | Effect |
|-------|---------|-------------|--------|
| ArrowRight | 1 | No | Move right 1px |
| Shift+ArrowRight | 10 | Yes | Move right 10px |
| ArrowUp | 1 | No | Move up 1px |
| Shift+ArrowUp | 10 | Yes | Move up 10px |

### 4.2 Optional: Full bounds clamping

Diagram's `clampShape(s)` prevents shapes from being moved off-canvas in all four directions. Layout can add a simple bounds check inside the arrow-key handler:

```js
var p = cpage();
if (p) {
    el.x = Math.max(0, Math.min(el.x, (p.pw || PW) - el.w));
    el.y = Math.max(0, Math.min(el.y, (p.ph || PH) - el.h));
}
```

Replace the single `Math.max(0, ...)` lines with this four-direction clamp if desired. Default recommendation: add it — it's a one-line change per axis and prevents elements from being lost off the right/bottom edge.

## 5. Pre-Flight Checklist

1. Confirm `ARROW_DELTA` keys match `e.key` values produced by browser for all four arrow keys
2. Confirm the `isEditing` guard fires correctly when a text element is being edited (contentEditable div, input, textarea, select)
3. Confirm that after `onKey` returns, no other shortcut handler processes the arrow key
4. Verify `_arrowMoveActive` is reset on keyup even if focus is lost mid-gesture (test with tab/alt-tab while holding arrow key)

## 6. Test Plan

| # | Test | Expected |
|---|------|----------|
| 1 | Select an element, press ArrowRight once | Element moves 1px right |
| 2 | Press ArrowRight five more times (same gesture) | Element moves 5px more right, single undo step |
| 3 | Release arrow key, press ArrowLeft once | Element moves 1px left |
| 4 | Undo (Ctrl+Z) after arrow moves | All arrow moves in that gesture undone (one step) |
| 5 | Redo (Ctrl+Shift+Z) after undo | Arrow moves reapplied |
| 6 | Shift+ArrowRight | Element moves 10px right |
| 7 | Shift+ArrowUp | Element moves 10px up |
| 8 | Select multiple elements, press ArrowRight | All selected elements move 1px right |
| 9 | Arrow key while editing text in a text box | No movement (guard prevents it) |
| 10 | Arrow key while input/textarea is focused | No movement |
| 11 | Rapidly tap ArrowRight 20 times | Moves 20px, single undo step (no memory of intermediate positions) |
| 12 | Press and hold ArrowRight | Repeats keydown events (OS repeats), moves continuously, single undo step |
| 13 | Tab away while holding ArrowRight | Undo snapshot committed on keyup when focus returns, or on next arrow press |
| 14 | Arrow key at page left/top edge (x=0 or y=0) | Element stops at 0, does not go negative |
| 15 | (If full bounds clamp added) Arrow key at page right/bottom edge | Element stops at page boundary, does not go off-page |

## 7. Rollback

If arrow-key bumping causes issues:

1. Comment out the arrow-key block in `onKey()` (Step 3)
2. Comment out the `keyup` handler (Step 4)
3. Remove `_arrowMoveActive` and `ARROW_DELTA` declarations (Step 1)

Each piece is self-contained and can be removed independently.
