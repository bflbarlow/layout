# Canvas Conform — Aligning layout with diagram's pan/zoom/clamp system

> **Review changelog:**
> - 2025-09-16 (pass 1): Corrected all line numbers to match actual `index.html` source. Fixed wrong `togglePanel()` reference (layout has `toggleLeftSidebar()` + `toggleSidebar()`). Fixed wrong `loadFromLocalStorage` reference (layout has `load()` at line 1395 + separate zoom/pan localStorage keys at 3267-3274). Merged duplicate "Pinch zoom pan adjustment" / "Pinch pan delta" entries. Fixed `clamp()` argument order bug in zoom buttons snippet. Noted that `bindSidebarResize()` exists (was incorrectly marked "Not present"). Added layout persistence differences section. Noted `canvas-container` vs `canvas-scroll` rect inconsistency in existing pinch code.
> - 2025-09-16 (pass 2): **Critical fix** — discovered layout has no `clamp()` helper at all (previously falsely stated "both projects already have this"); added as explicit step 1 in implementation order. Corrected wheel-handler line numbers (`onWheel` is 2890–2904, not ~2890-2905). Corrected diagram's middle/right-click pan line refs (1736/1745/1990, not 1735/1745/1989). Added §4.1 documenting two genuine *feature* gaps beyond clamping — layout has no deferred right-click pan and no keyboard +/− zoom, both of which diagram supports — since true alignment covers functionality, not just the clamp safety net. Added naming-collision warning: `#canvas-container` refers to different elements in each project (diagram's viewport vs. layout's inner padded wrapper). Documented that layout's `refresh()`/`emit()` pattern (17+ call sites) calls `applyTransform()` on nearly every state mutation, making the "never clamp inside `applyTransform()`" rule even more critical in layout than diagram — added corresponding pre-flight check and test case.

## 1. Goal

Bring the layout project's canvas handling into alignment with the diagram project's proven approach to pan clamping, zoom anchors, and container resize handling. The diagram project (`/Users/bflbarlow/Websites/diagram/app.js`) has a complete, tested pan clamp system that avoids the trackpad wheel bug the layout project previously hit (documented in `documents/archive/SMALL_REQUESTS.md`).

## 2. Why This Matters

The layout project's `SMALL_REQUESTS.md` notes:

> **Pan clamping**: Add back after solving the clamping + trackpad interaction bug. The naive `Math.max/min` in `applyTransform` interfered with two-finger macOS trackpad wheel events during leftward panning.

The diagram project solved this exact problem. The root cause was identical in both projects: clamping inside `applyTransform()` — a pure render function called from many places — caused every render (including unrelated re-renders) to snap panX/Y back to the boundary, fighting real-time trackpad deltas.

**The fix is the same in both projects: clamp at mutation sites only, never inside `applyTransform()`.**

**⚠ This rule is even more critical in layout than in diagram.** Layout's `refresh()` (line 2352) calls `applyTransform()` unconditionally on every call, and `refresh()` runs on every `emit()` — the pub/sub notification fired after essentially every state mutation (17+ call sites: `addElement`, `updEl`, `removeElement`, `undo`, `redo`, page switches, etc., not just pan/zoom actions). This means `applyTransform()` in layout fires far more often, and for reasons completely unrelated to panning, than in diagram. Any clamp logic placed inside `applyTransform()` would therefore snap pan back to the boundary on *every unrelated edit*, not just on renders following a pan gesture — a stronger version of the same bug that hit layout originally. This is the single strongest argument for keeping clamp logic strictly at the mutation sites listed in §4, and never touching `applyTransform()` itself.

## 3. Reference Implementation: diagram project

All clamping logic lives in `/Users/bflbarlow/Websites/diagram/app.js`. Key reference:

| Concept | Location | Description |
|---------|----------|-------------|
| `clampPan()` | app.js:207 | Pure clamp function — reads container rect, computes bounds, clamps panX/Y |
| `reClampPan()` | app.js:215 | Same as `clampPan()` + calls `applyTransform()` — for resize triggers |
| `CONFIG.panMargin` | app.js:29 | Default 100px margin before clamping triggers |
| Two-pass clamp | app.js:2481, 2489 | Before + after zoom-at-cursor to preserve cursor anchor |
| Clamp at load | app.js:611, 3185, 3388 | After restoring panX/Y from all 3 load paths |
| Clamp on zoom buttons | app.js:2645, 2651 | After changing zoom (no anchor to preserve) |
| Clamp on keyboard +/- | app.js:3388-3389 | After changing zoom |
| Clamp on panel collapse | app.js:2692 | `reClampPan()` after `togglePanel()` |
| Clamp on panel resize | app.js:2726 | `reClampPan()` after pointerup on panel resize handle |
| Clamp on window resize | app.js:3603 | `reClampPan()` on `window.resize` |

### 3.1 Clamp formula (identical in both projects)

```
cw = canvasW * zoom, ch = canvasH * zoom
M = CONFIG.panMargin (default 100)

panX_min = M - cw
panX_max = containerWidth - M
panY_min = M - ch
panY_max = containerHeight - M
```

**Layout adaptation:** `canvasW`/`canvasH` → `cpage().pw`/`cpage().ph`. `zoom` → `state.zoom`. `container` → `$('canvas-scroll')`.

### 3.2 clampPan() helper (copy verbatim)

```js
function clampPan() {
    var r = container.getBoundingClientRect();
    var cw = S.canvasW * S.zoom, ch = S.canvasH * S.zoom;
    var m = CONFIG.panMargin;
    S.panX = clamp(S.panX, m - cw, r.width - m);
    S.panY = clamp(S.panY, m - ch, r.height - m);
}

function reClampPan() {
    var r = container.getBoundingClientRect();
    var cw = S.canvasW * S.zoom, ch = S.canvasH * S.zoom;
    var m = CONFIG.panMargin;
    S.panX = clamp(S.panX, m - cw, r.width - m);
    S.panY = clamp(S.panY, m - ch, r.height - m);
    applyTransform();
}
```

**CRITICAL RULE: Never modify `applyTransform()` to include clamping.** It is a pure render function. Clamping inside it is the exact mechanism that broke panning in the layout project.

## 4. Layout Project — Mutation Sites Requiring Clamp

The layout project's `index.html` has the following mutation sites that need `clampPan()` after pan modification:

| Mutation site | Line | Status | Action |
|---------------|------|--------|--------|
| Pointer drag pan (`onPointerMove` → `isPanning` branch) | 2802 | ❌ No clamp | Add `clampPan()` after panX/Y update |
| Wheel pan (plain scroll, `!state.pinchStart` branch) | 2900–2902 | ❌ No clamp | Add `clampPan()` after delta |
| Wheel zoom (Ctrl/Cmd+wheel) | 2890–2899 (`onWheel`) | ❌ No clamp | Add two-pass: `clampPan()` before zoom + `clampPan()` after |
| Pinch zoom+pan (single block in `onPointerMove`) | 2781–2800 | ❌ No clamp | Add `clampPan()` after panX/Y adjustment |
| Zoom button `zoomIn()` | 2965 | ❌ No clamp | Add `clampPan()` after zoom change |
| Zoom button `zoomOut()` | 2970 | ❌ No clamp | Add `clampPan()` after zoom change |
| `btn-new` (new project) | 2427 | ❌ No clamp | Add `clampPan()` after panX/Y reset |
| `btn-import` import | 2457 | ❌ No clamp | Add `clampPan()` after panX/Y restore |
| `reset-view` button | 2520 | ❌ No clamp | Add `clampPan()` after panX/Y reset |
| `load()` (localStorage data) | 1395 | ❌ No clamp | Add `clampPan()` after pages loaded |
| Zoom/pan localStorage restore | 3267–3274 | ❌ No clamp | Add `clampPan()` after zoom/pan restored |
| Window resize | Not bound | ❌ No handler | Bind `reClampPan()` to `window.resize` |
| Left sidebar collapse (`toggleLeftSidebar`) | 3121 | ❌ No clamp | Add `reClampPan()` in `toggleLeftSidebar()` |
| Right sidebar collapse (`toggleSidebar`) | 3155 | ❌ No clamp | Add `reClampPan()` in `toggleSidebar()` |
| Right sidebar resize handle pointerup | ~3203 | ❌ No clamp | Add `reClampPan()` after width set |

**Note:** Layout has no `togglePanel()` function (unlike diagram). Sidebar collapse is handled by two separate functions: `toggleLeftSidebar()` (line 3121) and `toggleSidebar()` (line 3155). Layout also has no multi-tab coordination (no BroadcastChannel, no `storage` event listener), so there is no `loadFromLocalStorage` / storage event path to clamp.

**Note:** `bindSidebarResize()` exists at line 3150 with a `pointerup` handler — it was incorrectly marked as "Not present" in an earlier draft.

### 4.1 Feature gaps beyond clamping (true alignment requires these too)

The user's request is to align both projects on canvas *functionality*, not just clamping. A line-by-line comparison of pan/zoom entry points shows layout is missing two input methods diagram supports. These are not clamp bugs — they are missing features:

| Feature | diagram | layout | Gap |
|---------|---------|--------|-----|
| Middle-click drag pan (button 1) | ✅ `app.js:1736` | ✅ `index.html:2715` | None |
| Deferred right-click pan (button 2, click-vs-drag) | ✅ `app.js:1745` (start) + `app.js:1990` (drag threshold check) (see archived `documents/archive/RIGHT_CLICK_PAN.md`) | ❌ Not implemented — `onPointerDown` has no `e.button === 2` branch | Layout has a `contextmenu` listener (`index.html:3243`) that only suppresses the native menu (except over `.el-text` elements) — it does not trigger pan on right-click-drag |
| Two-finger trackpad wheel pan | ✅ | ✅ | None |
| Pinch-to-zoom + pan (touch) | ✅ | ✅ | None |
| Ctrl/Cmd+wheel zoom-at-cursor | ✅ | ✅ | None |
| Zoom buttons | ✅ (`CONFIG.zoomStep` = 0.1, `app.js:2645/2651`) | ✅ (hardcoded 0.25, `index.html:2965/2970`) | Step size differs (cosmetic only) |
| Keyboard `+`/`-` zoom | ✅ `app.js:3388-3389` | ❌ Not implemented — no `+`/`-`/`=` key handling found anywhere in `index.html` | Missing entirely |
| Pan clamping (this doc's main topic) | ✅ | ❌ | Subject of §4 above |

**Recommendation:** If "canvas pan/movement work" alignment is meant to cover input parity (not just the clamp safety net), add deferred right-click pan and keyboard `+`/`-` zoom to layout as follow-up work. These are out of scope for the clamp implementation itself but should be tracked so the two projects don't silently diverge. Suggest filing as a follow-up item rather than bundling into the clamp change, since right-click pan in particular is nontrivial (see diagram's own `documents/archive/RIGHT_CLICK_PAN.md` for the full breakage analysis diagram went through when adding it).

## 5. Layout-Specific Considerations

### 5.1 Container vs. page-content

The layout project uses `canvas-page` (`#canvas-page`) as the transformed element (a `<div>`, not a `<canvas>`). The container for the clamp rect is `canvas-scroll` (`#canvas-scroll`) — the viewport with `overflow:hidden`. The clamp formula uses the container's `getBoundingClientRect()`:

```js
var r = $('canvas-scroll').getBoundingClientRect();
```

**⚠ Inconsistency in existing code:** The pinch zoom code (line 2786: `var ccEl = $('canvas-container');`) uses `canvas-container` for its rect calculation, not `canvas-scroll`. `canvas-container` is a `display:inline-block` child of `canvas-scroll` with `padding: var(--space-7)` (48px, CSS line 384/44). Diagram has no equivalent inconsistency — it uses a single `container` variable (`#canvas-container`, diagram's viewport element) everywhere, including its own pinch code (`app.js:2009`: `var r = container.getBoundingClientRect();`). For layout's clamp formula, use `canvas-scroll` (the viewport). Consider also fixing the pinch code's `ccEl` reference to `canvas-scroll` for consistency — otherwise the pinch anchor math and the clamp math will be computing screen-space coordinates relative to two different rects (offset by the 48px container padding), which could cause the clamp to be off by that padding amount when combined with pinch.

### 5.2 Page dimensions vs. canvas dimensions

Diagram uses `S.canvasW` / `S.canvasH`. Layout uses per-page dimensions (`state.pages[i].pw` / `state.pages[i].ph`). Use the current page's dimensions:

```js
var p = cpage();
var cw = (p ? p.pw : PW) * state.zoom;   // PW = 850 (line 960)
var ch = (p ? p.ph : PH) * state.zoom;   // PH = 1100 (line 960)
```

`PW` and `PH` are module-level constants defined at line 960: `var PW = 850, PH = 1100, GRID = 10, MAX_UNDO = 50, SAVE_DELAY = 1500, SK = 'layout-editor-data';`

### 5.3 No `CONFIG` object

Layout currently has no `CONFIG` object. Constants are scattered: `PW`/`PH`/`GRID`/`MAX_UNDO` at line 960, `state.minZoom`/`state.maxZoom` in state, and hardcoded zoom deltas (`0.25` for buttons, `0.1` for wheel). Add a `CONFIG` object:

```js
var CONFIG = {
    panMargin: 100,
    zoomStep: 0.1
};
```

**⚠ Note:** Layout's zoom buttons use `±0.25` while wheel uses `±0.1`. Consider unifying to `CONFIG.zoomStep` for consistency. Diagram uses `CONFIG.zoomStep: 0.1` for both.

### 5.4 Pinch zoom pan order

Layout's pinch zoom (inside `onPointerMove`, lines ~2781-2800) does pan adjustment in two steps:
1. Zoom about midpoint: `panX = icx - (icx - panX) * zFrac`
2. Track midpoint movement: `panX += cx - icx`

The clamp should happen **after step 2** (single post-adjustment clamp). This differs from diagram's Ctrl+wheel zoom which uses two-pass (before + after) because:
- Pinch zoom starts from a valid position (user can't be at an invalid pan without previously being clamped)
- The midpoint tracking is a direct translation that doesn't compound with zoom math
- A single post-adjustment clamp is simpler and sufficient

**However**, for consistency with diagram's two-pass approach, you could add a pre-clamp before step 1. This is the safer approach and matches diagram's pattern exactly.

### 5.5 Zoom-at-cursor two-pass

For Ctrl/Cmd+wheel zoom (`onWheel`, lines 2890–2899), use the two-pass pattern:

```js
// Before zoom
clampPan();
// ... zoom computation ...
// After zoom
clampPan();
```

This ensures the zoom math starts from a valid position and the final result stays in bounds.

## 6. Shared Constants and Helpers

### 6.1 `clamp()` helper

**⚠ Correction:** Diagram has this helper (`app.js:204`). **Layout does NOT** — a repo-wide search (`grep -n "clamp" layout/index.html`) returns zero matches. This must be added to layout as a new function, not assumed to already exist:
```js
function clamp(v, lo, hi) { return Math.max(lo, Math.min(hi, v)); }
```
All `clampPan()`/`reClampPan()`/zoom-button code in this document that calls `clamp(...)` depends on this helper existing in layout first. Add it in step 1 of the implementation order (§10), before `clampPan()`.

### 6.2 `CONFIG.panMargin = 100`

Default 100px margin. Can be set to a very large number (e.g. 100000) as a kill switch to effectively disable clamping without removing code.

### 6.3 `clampPan()` / `reClampPan()`

Copy the diagram project's implementation verbatim. The only difference is the container element selector (`$('canvas-scroll')` in layout vs `container` in diagram).

## 7. Layout-Specific Persistence Notes

Layout's persistence mechanism differs from diagram's:

| Aspect | diagram | layout |
|--------|---------|--------|
| State key | `diagram-state` (localStorage) | `layout-editor-data` (`SK` at line 960) + separate `lp-zoom`/`lp-panX`/`lp-panY` |
| Save function | `saveState()` (debounced) | `save()` (synchronous) + `autosave()` (debounced) |
| Load function | `loadFromLocalStorage()` | `load()` at line 1395 |
| Multi-tab | BroadcastChannel + `storage` event | None |
| Pan/zoom persistence | In main state blob | Separate `lp-zoom`, `lp-panX`, `lp-panY` keys |

**Important:** Layout's `load()` function (line 1395) loads pages from `SK` key. The zoom/pan are restored separately at lines 3267-3274 in `init()`. Both paths need `clampPan()` after restoring pan values.

**Important:** Layout's `save()` function (line 1383) persists panX/Y to separate localStorage keys (`lp-panX`, `lp-panY`). This is different from diagram's single `diagram-state` blob. The clamp should be applied **before** `save()` is called to ensure persisted values are in bounds.

## 8. Pre-Flight Checklist (same as diagram's PAN_CLAMP.md §7)

Before implementing clamp in the layout project, verify:

1. **Confirm `applyTransform()` has zero side effects beyond setting `canvas-page.style.transform`.** Grep all call sites and confirm none expect it to clamp/validate state. In layout specifically, remember `applyTransform()` is called by `refresh()` (line 2352), which runs on every `emit()` — 17+ call sites covering nearly every state mutation, not just pan/zoom. Do not add clamping to `applyTransform()` or `refresh()` under any circumstance.

2. **Manually test leftward panning specifically** — the original bug report called out "leftward panning" by name. Confirm both pan directions are symmetric after implementation.

3. **Test with the sidebar both expanded and collapsed**, and at multiple zoom levels.

4. **Keep `CONFIG.panMargin` easy to set to a very large number** as a kill switch.

5. **Test editing an element while panned near the clamp boundary** (e.g. drag/resize/type text with the canvas panned to its leftmost extent). Since `refresh()` fires on every edit and calls `applyTransform()`, confirm the pan value itself is untouched by unrelated edits — only the mutation sites in §4 should ever change `state.panX`/`state.panY`.

## 9. Rollback Plan (same as diagram's PAN_CLAMP.md §8)

If panning regresses after shipping clamp in layout:

1. Set `CONFIG.panMargin` to a very large value to confirm clamp is the cause.
2. Revert wheel handler clamp first (most likely source of trackpad jitter).
3. Individual clamp sites are independent and can be removed selectively.

## 10. Implementation Order

Recommended order to minimize risk and enable incremental testing:

1. Add the `clamp(v, lo, hi)` helper (**layout has none — this does not already exist**, unlike diagram)
2. Add `clampPan()` and `reClampPan()` helpers (depend on step 1)
3. Add `CONFIG` object with `panMargin` and `zoomStep` (consider unifying zoom step from 0.25/0.1 to single value)
4. Clamp pointer drag pan (simplest, easiest to test)
5. Clamp plain scroll wheel pan
6. Clamp Ctrl/Cmd+wheel zoom (two-pass)
7. Clamp pinch zoom/pan
8. Clamp zoom buttons
9. Clamp all load paths (`load()` at line 1395, zoom/pan restore at 3267-3274, import at 2457, new project at 2427, reset-view at 2520)
10. Add window resize handler → `reClampPan()`
11. Add sidebar collapse handlers (`toggleLeftSidebar` at 3121, `toggleSidebar` at 3155) → `reClampPan()`
12. Add sidebar resize `pointerup` → `reClampPan()`
13. Fix pinch zoom to use `canvas-scroll` rect instead of `canvas-container` (consistency)
14. Manual test per §8 checklist
15. (Optional, separate follow-up) Consider adding deferred right-click pan and keyboard `+`/`-` zoom for full feature parity — see §4.1. Not required for clamp correctness.

**⚠ Note:** Layout's `applyTransform()` (line 1081) has a side effect — it updates `zoom-label` DOM element. More frequent calls from clamp mean more DOM writes. Diagram's `applyTransform()` has the same pattern (updates `zoomDisplay`).

## 11. Test Plan (adapted from diagram's PAN_CLAMP.md §5)

| # | Test | Expected |
|---|------|----------|
| 1 | Pan canvas fully left | Right edge of canvas visible (within margin) |
| 2 | Pan canvas fully right | Left edge of canvas visible (within margin) |
| 3 | Pan canvas fully up/down | Same for vertical |
| 4 | Zoom in at clamp boundary | Cursor anchor preserved, no flicker |
| 5 | Zoom out at clamp boundary | Cursor anchor preserved, no flicker |
| 6 | Two-finger trackpad wheel pan | Smooth inertia, no jitter at boundary |
| 7 | Window resize while panned | Pan stays within bounds |
| 8 | Collapse/expand sidebar | Pan re-clamped to new container width |
| 9 | Import saved state with extreme pan | Clamp to valid range on load |
| 10 | New project / reset-view | Pan centered and valid |
| 11 | Pinch pan on touch device | Clamp applies correctly |
| 12 | Drag/resize/edit an element while panned to clamp boundary (layout-specific) | Pan value does not jump or jitter — `refresh()`'s `applyTransform()` call must not alter panX/Y |

## 12. Key Differences from Diagram Project

| Aspect | diagram | layout |
|--------|---------|--------|
| Transformed element | `<canvas>` | `#canvas-page` div |
| Container for clamp rect | `#canvas-container` | `#canvas-scroll` |
| Canvas dimensions | `S.canvasW` / `S.canvasH` | `cpage().pw` / `cpage().ph` (per-page) |
| State object | `S` | `state` |
| Config object | `CONFIG` | None (add `CONFIG`) |
| Panel resize handle | `#panel-resize-handle` | `#sidebar-resize-handle` |
| Panel collapse | `togglePanel()` | `toggleLeftSidebar()` + `toggleSidebar()` |
| Min/max zoom | `CONFIG.minZoom` / `CONFIG.maxZoom` | `state.minZoom` / `state.maxZoom` |
| Zoom step | `CONFIG.zoomStep` | Hardcoded `0.1` (wheel) / `0.25` (buttons) |
| Persistence | Single `diagram-state` blob | `SK` blob + separate `lp-zoom`/`lp-panX`/`lp-panY` |
| Multi-tab | BroadcastChannel + `storage` event | None |
| Save function | `saveState()` (debounced) | `save()` (sync) + `autosave()` (debounced) |
| Load function | `loadFromLocalStorage()` | `load()` (1395) + zoom/pan restore (3267-3274) |
| `applyTransform()` side effect | Updates `zoomDisplay` | Updates `zoom-label` |
| Page constants | None | `PW=850`, `PH=1100`, `GRID=10` (line 960) |
| `clamp()` helper | Exists (`app.js:204`) | **Does not exist — must be added** |
| Keyboard `+`/`-` zoom | ✅ `app.js:3388-3389` | ❌ Not implemented (feature gap, see §4.1) |
| Deferred right-click pan | ✅ `app.js:1745`/`1990` | ❌ Not implemented (feature gap, see §4.1) |

**⚠ Naming collision:** Both projects have an element with `id="canvas-container"`, but they refer to different things. In diagram, `#canvas-container` is the fixed-position viewport (`position: fixed`, `overflow: hidden`) — the element the `container` variable points to and the correct rect source for clamping. In layout, `#canvas-container` is an inner `display:inline-block` wrapper with 48px padding, nested *inside* `#canvas-scroll` (the actual viewport). When adapting diagram's code, do not assume `#canvas-container` means the same thing in both files — in layout, the clamp rect must come from `#canvas-scroll`, not `#canvas-container`.

## 13. Appendix: Reference Code Snippets

### clampPan() for layout (adapted)

```js
function clampPan() {
    var r = $('canvas-scroll').getBoundingClientRect();
    var p = cpage();
    var cw = (p ? p.pw : PW) * state.zoom;
    var ch = (p ? p.ph : PH) * state.zoom;
    var m = CONFIG.panMargin;
    state.panX = clamp(state.panX, m - cw, r.width - m);
    state.panY = clamp(state.panY, m - ch, r.height - m);
}

function reClampPan() {
    var r = $('canvas-scroll').getBoundingClientRect();
    var p = cpage();
    var cw = (p ? p.pw : PW) * state.zoom;
    var ch = (p ? p.ph : PH) * state.zoom;
    var m = CONFIG.panMargin;
    state.panX = clamp(state.panX, m - cw, r.width - m);
    state.panY = clamp(state.panY, m - ch, r.height - m);
    applyTransform();
}
```

### Wheel handler (Ctrl+wheel zoom, two-pass)

```js
function onWheel(e) {
    e.preventDefault();
    if (e.ctrlKey || e.metaKey) {
        clampPan();  // #1 — before zoom
        var d = e.deltaY > 0 ? -CONFIG.zoomStep : CONFIG.zoomStep;
        var nz = clamp(state.zoom + d, state.minZoom, state.maxZoom);
        var cb = e.currentTarget.getBoundingClientRect();
        state.panX = (e.clientX - cb.left) - ((e.clientX - cb.left) - state.panX) * (nz / state.zoom);
        state.panY = (e.clientY - cb.top) - ((e.clientY - cb.top) - state.panY) * (nz / state.zoom);
        state.zoom = nz;
        applyTransform();
        clampPan();  // #2 — after zoom
    } else if (!state.pinchStart) {
        state.panX -= e.deltaX;
        state.panY -= e.deltaY;
        clampPan();
        applyTransform();
    }
}
```

### Pointer drag pan

```js
if (state.isPanning) {
    state.panX += e.clientX - state.panStart.x;
    state.panY += e.clientY - state.panStart.y;
    state.panStart = { x: e.clientX, y: e.clientY };
    clampPan();
    applyTransform();
    return;
}
```

### Pinch zoom/pan

```js
// After computing new panX/Y from pinch math:
clampPan();
applyTransform();
```

### Zoom buttons

```js
function zoomIn() {
    state.zoom = clamp(state.zoom + 0.25, state.minZoom, state.maxZoom);
    clampPan();
    applyTransform();
}

function zoomOut() {
    state.zoom = clamp(state.zoom - 0.25, state.minZoom, state.maxZoom);
    clampPan();
    applyTransform();
}
```

**⚠ Bug fix:** The original draft had `clamp(state.maxZoom, state.zoom + 0.25)` which is `clamp(lo, hi, val)` — wrong argument order. `clamp()` is defined as `clamp(v, lo, hi)`. Corrected to `clamp(state.zoom + 0.25, state.minZoom, state.maxZoom)`.

### Window resize

```js
window.addEventListener('resize', function() {
    reClampPan();
});
```

### Sidebar collapse

```js
// In toggleLeftSidebar() (line 3121) and toggleSidebar() (line 3155):
reClampPan();
```

### Panel resize pointerup

```js
// After setting sidebar width on pointerup:
reClampPan();
```
