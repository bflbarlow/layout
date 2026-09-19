# Inline Text Editing — Technical Plan

> **Review changelog:**
> - Pass 1: Cross-checked all code references against the actual `index.html` (3371 lines). Corrected §2.3's coordinate-mapping model — zoom/pan is a single CSS `transform` on `#canvas-page` (`applyTransform()`, line ~1122), not manual per-element `x*zoom+panX` math; the earlier draft's approach would have desynced editing elements from the app's actual zoom system and broken WYSIWYG at any zoom other than 100%. Propagated that fix into §4.4 (toolbar positioning — now reads `getBoundingClientRect()` instead of recomputing screen position) and §8.3 (removed incorrect "disable transform during edit" advice, which directly contradicted how the app renders). Fixed a real logic bug in §2.1's double-click detection (state was stored on a freshly-allocated object from `findEl()`, so it could never persist between clicks — replaced with module-level `lastClickId`/`lastClickTime` vars, consistent with the file's existing `dragging`/`resizing` pattern). Clarified §7.3 that `updatePanelEditorButtons()` already exists in `index.html` (line ~2316) and should be extended, not reimplemented. Corrected §10/§12 — `state.editingElementId` must NOT be added to `saveSnapshot()`/`undo()`/`redo()` (which serialize only `state.pages`) or to `save()`/`load()` (localStorage), since resuming inline-edit-mode across an undo step or a page reload is undefined/unsafe behavior, not a feature.
> - Pass 2 (implementation): **Architecture change — `.el-content` separation.** During implementation it became clear that making the `.canvas-element` box itself `contentEditable` created a class of bugs where one DOM node had to be two things at once (box chrome + text content): browser editing mutations rewrote the box's inline `style`/structure, the box's `display:flex` was overwritten by the browser, and the panel rich-text editor's `dom.innerHTML = val` wiped the resize handles that `showHandles()` had appended as box children. The shipped design therefore split each **text** element into two nodes: `.canvas-element` stays the layout/chrome box (position, size, padding, frame, background, border, radius, rotation, flex/vertical-align, overflow), and a new inner `.el-content` div holds all typography (font, alignment, line-height, letter-spacing, text-indent, columns, drop-cap, hyphens, `dir`) and is the **only** node that ever receives `contenteditable`. Both `domEl()` (normal render) and `applyElementStyles()` (enter-edit) now delegate to a single routine, `styleTextBox(box, content, el)`, so WYSIWYG is guaranteed by construction rather than by duplicating a style list in two places — see the new §1.4 for the exact split and the reasons for it. Consequently the `contentEditable` selectors in §5.1/§8.2/§9/§12 that were written as `.canvas-element[contenteditable]` are now `.el-content[contenteditable]`, and §2/§3's `enterInlineEdit`/`exitInlineEdit` snippets operate on the content node while styling the box. Also fixed two pre-existing bugs surfaced by the refactor: (a) `if (!el.visible) d.style.display = 'none'` ran before the type branch, so `styleTextBox`'s `display:flex` re-showed hidden text elements — the visibility check now runs after all branches; (b) the panel editor handlers wrote `dom.innerHTML` to the box, wiping handles — they now target `.el-content`. Live input sync moved from a per-node listener (which `renderPage()` detached mid-edit) to a delegated `input` listener on the stable `#page-content`, and `enterInlineEdit` no longer calls `emit()` (a re-render would detach the node and lose the caret), using `updateProps()` instead.
> - Pass 3 (bug fix — inline font controls): Fixed all four advanced inline-toolbar controls (font family, size, weight, colour), which were non-functional in real use. Two independent causes: **(1)** the font-family handler used `document.execCommand('fontName', …)`, which Chromium silently no-ops whenever the value contains a space or comma — and every `<option>` in the toolbar is a stack like `Georgia, serif` or `'Times New Roman', serif` (verified directly: `fontName` with `Arial` works, with `Georgia, serif` / `Georgia,serif` returns `true` but changes nothing, with or without `styleWithCSS`). Replaced it with the same `Range`-wrapping approach already used for size/weight, which preserves the full stack and yields `<span style="font-family: Georgia, serif">`. **(2)** selects and number/colour inputs steal focus when clicked, collapsing the live DOM selection before the `change`/`input` handler runs — so even size/weight/colour, which worked in synthetic tests with a programmatic selection, did nothing for a real user. Added `inlineSavedRange` with `captureInlineSelection()` / `restoreInlineSelection()`: the last non-empty range inside the active `.el-content` is snapshotted on `selectionchange` (and re-snapshotted after each command), and every toolbar handler restores it before issuing the command. Also added `user-select:text` to `.el-content[contenteditable]` so the editable is selectable despite `body{user-select:none}`. Verified: all four apply to a full or partial selection, survive a simulated focus steal, and persist into `el.richText`/localStorage and the re-rendered DOM.
> - Pass 4 (bug fix — dropdown "flashing"): The inline toolbar's font-family and font-weight `<select>`s opened and instantly closed (reported as "flashing"), so they could not be used. Root cause: `bindInlineToolbar()` attached its generic action handler with `toolbar.querySelectorAll('[data-action]')`, which also matched the `<select>`/`<input>` controls. Clicking a dropdown therefore ran the button handler, whose `restoreInlineSelection()` called `content.focus()` and yanked focus out of the just-opened native popup, closing it. Fixed by (a) scoping the generic handler to `button[data-action]`; (b) giving `restoreInlineSelection(skipFocus)` a no-focus mode used by the four style controls (they restore the `Range` without focusing, so an open dropdown/picker is undisturbed); and (c) replacing the last `execCommand` uses in those controls (colour) with `Range`-wrapping so no focus is required at all. Two follow-on fixes were needed to make wrapping robust: `wrapInlineRange()` now always re-points the selection at the wrapper's contents (Chromium's `surroundContents` leaves the range on the old container, which made `captureInlineSelection()` save a stale range and re-styling nest a new `<span>` per event); and because wrapping fires no `input` event, the controls now persist explicitly (`saveSnapshot()`/`save()`), with the colour picker reusing its previous wrapper on live `input` (snapshot only on `change`). Verified: clicking each control leaves focus on it (dropdown/picker stays open), applying a value updates the selection and keeps focus, three consecutive colour picks produce a single updated span, and all prior tests still pass.
> - Pass 5 (bug fix — font-size looked broken): The four advanced controls were never synchronised with the selected text, so the font-size box sat at its HTML default (`16`) even when the selected heading was `48px` — changing it appeared to do nothing or produced wildly wrong sizes. `updateInlineToolbarState()` now reads `getComputedStyle()` of the caret/selection node (via a new `selectionStyleNode()`) and writes the real values back into the font-family, font-size, font-weight and colour controls (with a new `cssColorToHex()` for `rgb()`→`#rrggbb`), skipping any control the user is currently interacting with. Verified: on entering edit and after selecting-all, the controls report `48 / Georgia, serif / 700 / #111113`; typing a new size and using the spinner both apply correctly.


> **Goal:** Eliminate friction between what a user intends to present as text and the final product. The user must feel confident that an edit they make in the text box will appear exactly as intended — pixel-perfect WYSIWYG.
>
> **Current state:** Text editing happens in the Properties panel (`#prop-text-content`, a contentEditable div). Canvas text elements are rendered as plain `<div>`s with `innerHTML = el.richText || el.text`. There is **no inline canvas editing** — you cannot double-click a text box on the canvas to edit it in place. Formatting is applied via `document.execCommand()` on the panel editor, not on the canvas element itself.
>
> **Scope:** This document covers the full technical plan for adding **inline canvas-level text editing** — double-click to enter edit mode, WYSIWYG editing directly on the canvas, with pixel-perfect rendering.
>
> **Implementation status (2025-09-16): SHIPPED.** The feature is implemented in `index.html`, with one architecture change from the original draft: text lives in an inner `.el-content` node rather than on the box itself — see **§1.4** for the shipped design, and Pass 2 of the changelog above for why. Verified in Chromium: entering/exiting edit mode produces byte-identical computed styles on both nodes; only `.el-content` is editable; formatting survives blur, panel edits, re-renders, and export; auto-resize is stable.

---

## §1. Architecture Overview

### 1.1 Current Flow vs. Target Flow

**Current flow:**
```
User edits #prop-text-content (panel RTE)
  → input event fires
  → state.selectedElements[].richText updated
  → canvas DOM element.innerHTML patched directly (no full re-render)
  → format buttons → execCommand on #prop-text-content
  → selection saved/restored via savedRange
```

**Target flow (inline canvas editing):**
```
User double-clicks canvas text element
  → enter inline edit mode on that element
  → the element's inner .el-content node becomes contentEditable on-canvas
  → caret/selection managed in-place
  → formatting toolbar reflects current selection state
  → blur/Escape → exit edit mode, save to state
  → pixel-perfect rendering matches panel RTE exactly
```

### 1.2 Key Design Decisions

1. **Single source of truth:** The data model (`el.richText` + `el.text`) remains the single source. The canvas DOM element is always a reflection of state.
2. **Inline contentEditable on an inner content node:** The element's inner `.el-content` node becomes `contentEditable` during edit mode — no overlay, no iframe, no shadow DOM. The outer `.canvas-element` box keeps its layout/chrome and is never made editable (see §1.4). This guarantees pixel-perfect WYSIWYG while structurally preventing the box from being mutated by the browser's editing engine.
3. **No dual editing:** When in inline edit mode, the panel `#prop-text-content` is disabled/hidden for that element to avoid sync conflicts.
4. **Zoom-aware:** Editing coordinates and caret positions must be calculated in page-space, not screen-space, to handle zoom correctly (see §2.3 for the corrected model — zoom is a single ancestor CSS transform, not per-element math).
5. **Known limitation — `execCommand`:** This plan continues the existing codebase's reliance on `document.execCommand()` (already used throughout `bindPanelEditor()` and `bindProps()`). `execCommand` is a [deprecated Web API](https://developer.mozilla.org/en-US/docs/Web/API/Document/execCommand) with no formal replacement standard yet (the emerging `contenteditable` + `beforeinput`/`EditContext` APIs are still maturing and inconsistently supported as of this writing). It is used here for consistency with the existing panel editor and because it remains functional in all three target browsers (Chrome 90+, Firefox 88+, Safari 14+ per the README). **This is a deliberate, scoped risk, not an oversight** — if browser vendors remove `execCommand` support in the future, both the panel editor and this inline-editing plan will need a coordinated rewrite onto a non-`execCommand` rich-text approach (e.g. a manual `Range`-based formatting engine). Flagging this now so it is not mistaken for a solved problem.

### 1.3 State Machine for Text Edit Mode

```
IDLE
  → double-click on text element → ENTER_EDIT
  → Escape / blur → IDLE

ENTER_EDIT
  → formatting applied via toolbar → ENTER_EDIT
  → typing → ENTER_EDIT (live sync to state)
  → Escape → IDLE (save snapshot)
  → click outside canvas → IDLE (save snapshot)
```

### 1.4 Implemented architecture — `.el-content` separation

> **This subsection describes what was actually shipped and supersedes the single-node
> snippets elsewhere in this document.** Where §2/§3 show `dom.setAttribute('contenteditable', ...)`
> on the box, read `content.setAttribute(...)` on `.el-content`; where §5.1/§8.2/§9/§12 write
> `.canvas-element[contenteditable]`, read `.el-content[contenteditable]`.

Every **text** element renders as two nested nodes:

```html
<div class="canvas-element el-text" data-id="...">   <!-- BOX: layout + chrome -->
  <div class="el-content">…richText…</div>            <!-- CONTENT: typography -->
</div>
```

**Responsibility split** (chosen so nothing that must not inherit is left on the flex box):

| Concern | Node | Why |
|---|---|---|
| `left/top/width/height`, `padding`, `background`, `border`, `border-radius`, `box-shadow`, `opacity`, `transform: rotate`, `z-index`, `overflow`, `display:flex`, `justify-content` (vertical-align), `align-items` | `.canvas-element` (box) | Layout/chrome belongs to the box; handles and selection border attach here. |
| `font-family/size/weight/style`, `color`, `text-align`, `line-height`, `letter-spacing`, `word-spacing`, `text-transform`, `text-decoration`, `text-indent`, `text-shadow`, `hyphens`, `column-count`, `column-gap`, `dir` | `.el-content` (content) | `column-count`/`column-gap` are **not inherited**, so they must sit on the node that actually holds the inline text. Drop-cap (`::first-letter`) must sit on a non-flex block container. Everything else is put here for clarity and because it is the node the user edits. |
| `white-space: pre-wrap`, `word-break: break-word`, `min-width/min-height` (`.el-text` class) | `.canvas-element` (box) | Inherited by `.el-content`; kept on the box so the box's intrinsic sizing stays correct. |

**Single source of truth — `styleTextBox(box, content, el)`:** one function sets *both* nodes' inline styles. `domEl()`'s text branch calls it during normal renders; `applyElementStyles(box, el)` calls it when entering edit mode. Because both paths funnel through the same routine, entering/leaving edit mode cannot change any computed style — the WYSIWYG guarantee is structural, not a maintenance promise. `applyElementStyles()` is therefore literally `styleTextBox(box, getContentEl(box), el)`.

**Helpers:**
- `getContentEl(box)` → the box's `:scope > .el-content` child (or `null`).
- `getEditingBox()` → `document.querySelector('.canvas-element[data-id="' + state.editingElementId + '"]')`.
- `getActiveEditable()` → `getContentEl(getEditingBox())` — the only node that is ever `contentEditable`.
- `showInlineToolbar(dom)` defensively resolves a `.el-content` argument up to its parent box before positioning, so callers passing either node work.

**Consequences for the rest of this document:**
1. The canvas DOM is still a pure reflection of `el.richText`/`el.text`; the *content node's* `innerHTML` is the live value during editing.
2. `renderPage()`'s mid-edit guard saves and restores `getContentEl(box).innerHTML` (and re-sets `contenteditable` on the content node), never the box.
3. `showHandles()` intentionally skips `state.editingElementId` — resize handles are hidden while text-editing (matching Google Slides), so they can never be appended inside the editable.
4. The panel editor's live-patch handlers write `getContentEl(dom).innerHTML`, not `dom.innerHTML`, so they no longer destroy the box's handles.
5. `exportHTML()` emits the same nested structure (`<div style=box><div class="el-content…" style=content>…</div></div>`), keeping export pixel-identical to the canvas.


---

## §2. Entering Inline Edit Mode

### 2.1 Double-Click Detection

**Location:** `onPointerDown` in `index.html` (~line 2720)

**Implementation:**

```js
// Module-level state, alongside the existing `dragging`/`resizing`/`drawing`
// vars (index.html ~line 2705). `f` from findEl() is a fresh object on
// every call, so double-click state CANNOT be stored on it — it must live
// in a persistent variable scoped outside onPointerDown.
var lastClickId = null, lastClickTime = 0;
var DOUBLE_CLICK_MS = 300;

// In onPointerDown, after hit testing:
var f = findEl(pt.x, pt.y);
if (f && f.data.el.type === 'text') {
  var now = Date.now();
  if (f.data.el.id === lastClickId && now - lastClickTime < DOUBLE_CLICK_MS) {
    enterInlineEdit(f.data.el.id);
    lastClickId = null;
    lastClickTime = 0;
    return;
  }
  lastClickId = f.data.el.id;
  lastClickTime = now;
}
```

**Key considerations:**
- Must distinguish single-click (select) from double-click (edit). Use a 300ms debounce window.
- Single-click sets up the potential for double-click but does not enter edit mode.
- If the user clicks on a resize handle, the double-click logic must not trigger.
- The double-click must occur **inside** the text content area, not on the border/handles.

### 2.2 Making the Canvas Element Editable

**Function: `enterInlineEdit(elementId)`**

```js
function enterInlineEdit(elementId) {
  var d = getEl(elementId);
  if (!d) return;
  
  // Exit any existing edit mode first
  exitInlineEdit(true);
  
  state.editingElementId = elementId;
  var dom = document.querySelector('.canvas-element[data-id="' + elementId + '"]');
  if (!dom) return;
  
  // Make element contentEditable
  dom.setAttribute('contenteditable', 'true');
  dom.spellcheck = true;
  
  // Apply all current styles explicitly (contentEditable may reset some)
  applyElementStyles(dom, d.el);
  
  // Ensure the element is in the DOM with correct innerHTML
  dom.innerHTML = d.el.richText || d.el.text;
  
  // Prevent pointer events that would interfere with editing
  dom.style.pointerEvents = 'auto';
  
  // Disable resize handles and selection border for this element
  hideHandles(); // Remove resize handles
  
  // Disable panel text editor for this element
  var panelEditor = $('prop-text-content');
  if (panelEditor) panelEditor.setAttribute('contenteditable', 'false');
  
  // Focus the element after a frame (ensures DOM is ready)
  requestAnimationFrame(function() {
    dom.focus();
    // Place caret at end of text by default
    var range = document.createRange();
    var sel = window.getSelection();
    range.selectNodeContents(dom);
    range.collapse(false);
    sel.removeAllRanges();
    sel.addRange(range);
  });
  
  // Show formatting toolbar (if not already visible)
  showInlineToolbar(dom);
  
  emit(); // Update UI
}
```

**Critical: `applyElementStyles(dom, el)`**

This must replicate exactly what `domEl()` does for text elements (line ~1475-1560), but applied as inline styles to a contentEditable element. Must handle:
- `fontFamily`, `fontSize`, `fontWeight`, `fontStyle`
- `textColor` (as `color`)
- `textAlign`
- `lineHeight`
- `letterSpacing`
- `padding` (as `padding` on the editable element)
- `textIndent`
- `columnCount`, `columnGap`
- `hyphens`
- `dir`
- `verticalAlign` (via flexbox `justifyContent`)
- `overflow`
- `background` (fill)
- `border` (stroke)
- `borderRadius`
- `opacity`
- `textDecoration` (per-element, not per-selection)

### 2.3 Zoom-Aware Coordinate Mapping

**Correction of an earlier draft of this plan:** the app does **not** compute screen positions by hand from `state.zoom`/`state.panX`/`state.panY`. Zoom and pan are applied as a single CSS `transform` on `#canvas-page` (see `applyTransform()`, line ~1122: `transform: translate(panX, panY) scale(zoom)`). All element `left`/`top`/`width`/`height` styles are always set in **unzoomed page-space** (`domEl()`, line ~1531 onward) and the browser does the zoom math for free via the CSS transform on the ancestor.

This has an important consequence for inline editing: **do not reimplement zoom math for element positioning.** The existing `pageCoords(e)` (alias for `toCanvas()`, line ~1108) already converts a screen-space pointer event into page-space coordinates correctly, by measuring `#page-content` (which sits *inside* the transformed `#canvas-page`) with `getBoundingClientRect()`:

```js
// Existing, unmodified — do not duplicate this logic
function toCanvas(cx, cy) {
  var ct = $('page-content');
  var r = ct.getBoundingClientRect();
  return { x: (cx - r.left) / state.zoom, y: (cy - r.top) / state.zoom };
}
function pageCoords(e) { return toCanvas(e.clientX, e.clientY); }
```

For the **reverse** direction (page-space → screen-space, needed to position the floating inline toolbar, §4.4), the reliable approach is to read the *already-positioned* canvas element's own `getBoundingClientRect()` rather than recomputing the transform manually — this automatically stays correct regardless of zoom, pan, sidebar collapse state, or rotation:

```js
function elementScreenRect(dom) {
  return dom.getBoundingClientRect(); // already accounts for zoom/pan/rotation
}
```

All toolbar-positioning code in §4.4 and edit-mode style code in §8.3 below has been corrected to use this approach instead of manual zoom/pan arithmetic.

---

## §3. Exiting Inline Edit Mode

### 3.1 Exit Triggers

Exit inline edit mode on:
1. **Escape key** — cancel edit, save snapshot
2. **Blur** (focus leaves the element) — save changes
3. **Click on another element** — save changes, switch selection
4. **Tool switch** — save changes
5. **Explicit "done" button** in inline toolbar

### 3.2 `exitInlineEdit(saveChanges)`

```js
function exitInlineEdit(saveChanges) {
  var id = state.editingElementId;
  if (!id) return;
  
  var dom = document.querySelector('.canvas-element[data-id="' + id + '"]');
  if (!dom) { state.editingElementId = null; return; }
  
  if (saveChanges) {
    var el = getEl(id);
    if (el) {
      // Sync DOM back to state
      el.richText = dom.innerHTML;
      el.text = dom.innerHTML.replace(/<[^>]*>/g, '');
      
      // Normalize: remove empty spans, clean up execCommand artifacts
      normalizeRichText(el);
      
      // Re-render to ensure consistency
      renderPage();
      
      // Save to undo stack
      saveSnapshot();
      save(); // localStorage
    }
  } else {
    // Restore original content
    var el = getEl(id);
    if (el) {
      dom.innerHTML = el.richText || el.text;
    }
  }
  
  // Clean up
  dom.removeAttribute('contenteditable');
  dom.spellcheck = false;
  
  state.editingElementId = null;
  
  // Re-enable panel text editor
  var panelEditor = $('prop-text-content');
  if (panelEditor) panelEditor.setAttribute('contenteditable', 'true');
  
  // Hide inline toolbar
  hideInlineToolbar();
  
  emit();
}
```

### 3.3 Rich Text Normalization

`execCommand` produces messy HTML (random `<span>` tags, `font` tags in older browsers, inconsistent styling). The `normalizeRichText()` function must:

1. **Remove empty elements:** `<span></span>`, `<b></b>` with no text content
2. **Merge adjacent identical spans:** `<span style="color:red">a</span><span style="color:red">b</span>` → `<span style="color:red">ab</span>`
3. **Remove redundant inline styles:** Styles that match the parent element's computed styles
4. **Normalize `execCommand` artifacts:**
   - Convert `<font>` tags to `<span style="...">` (Chrome/Firefox compatibility)
   - Convert `style="color: ..."` to consistent format
   - Remove `mso-` prefixed styles (Word compatibility artifacts)
5. **Ensure paragraph structure:** Wrap in `<p>` tags for proper rendering
6. **Strip zero-opacity and transparent colors** that serve no purpose

```js
function normalizeRichText(el) {
  var html = el.richText;
  
  // Step 1: Remove <font> tags (execCommand produces these in some browsers)
  html = html.replace(/<\/?font[^>]*>/gi, function(match, p1, offset, string) {
    // Extract style attribute if present
    var styleMatch = match.match(/style="([^"]*)"/i);
    return styleMatch ? '<span style="' + styleMatch[1] + '">' : '';
  });
  
  // Step 2: Remove empty spans
  html = html.replace(/<span(\s[^>]*)?><\/span>/gi, '');
  
  // Step 3: Remove spans with no effective style (inherits from parent)
  // This requires DOM parsing — use a temporary element
  var temp = document.createElement('div');
  temp.innerHTML = html;
  
  function removeInheritedStyles(node) {
    if (node.nodeType !== 1) return; // not an element
    if (node.tagName.toLowerCase() !== 'span') {
      for (var i = 0; i < node.childNodes.length; i++) {
        removeInheritedStyles(node.childNodes[i]);
      }
      return;
    }
    
    var computed = window.getComputedStyle(node);
    var parent = node.parentElement;
    if (!parent || parent.tagName.toLowerCase() !== 'span') return;
    
    var parentComputed = window.getComputedStyle(parent);
    
    // Check each style property
    var styles = node.style;
    var allInherited = true;
    for (var j = 0; j < styles.length; j++) {
      var prop = styles[j];
      if (computed.getPropertyValue(prop) !== parentComputed.getPropertyValue(prop)) {
        allInherited = false;
        break;
      }
    }
    
    if (allInherited && node.childNodes.length === 0) {
      // Empty span with no effective style — unwrap
      while (node.firstChild) {
        parent.insertBefore(node.firstChild, node);
      }
      parent.removeChild(node);
    } else if (allInherited && node.textContent.trim() === '') {
      // Empty span with text but no meaningful content
      while (node.firstChild) {
        parent.insertBefore(node.firstChild, node);
      }
      parent.removeChild(node);
    } else {
      for (var i = 0; i < node.childNodes.length; i++) {
        removeInheritedStyles(node.childNodes[i]);
      }
    }
  }
  
  removeInheritedStyles(temp);
  el.richText = temp.innerHTML;
  
  // Step 4: Ensure content is wrapped in <p> tags for proper paragraph rendering
  if (!el.richText.match(/^<p>/i) && !el.richText.match(/^<div>/i)) {
    el.richText = '<p>' + el.richText + '</p>';
  }
}
```

---

---

## §4. Inline Formatting Toolbar

### 4.1 Toolbar Design

The inline toolbar must appear **above the editing element** on the canvas, positioned relative to the element's bounding box. It provides the same formatting controls as the panel RTE toolbar but is contextually attached to the active edit.

**Visual layout:**
```
┌─────────────────────────────────────────────┐
│  [B][I][U][S]  [x²][x₂]  [Tx]  │  [
│  [Aa] [12] [400] [🎨]              │
└─────────────────────────────────────────────┘
  ↑ positioned above the editing element
  ↑ follows element if resized during edit
```

### 4.2 Toolbar DOM Structure

Create the toolbar once at app init, hide by default:

```html
<!-- Add to index.html <body>, after #page-content -->
<div id="inline-toolbar" class="inline-toolbar" style="display:none;position:absolute;z-index:1000;">
  <!-- Formatting row -->
  <div class="inline-toolbar-row">
    <button class="inline-btn" data-action="bold" title="Bold (Ctrl+B)"><b>B</b></button>
    <button class="inline-btn" data-action="italic" title="Italic (Ctrl+I)"><i>I</i></button>
    <button class="inline-btn" data-action="underline" title="Underline (Ctrl+U)"><u>U</u></button>
    <button class="inline-btn" data-action="strikethrough" title="Strikethrough"><del>S</del></button>
    <span class="inline-sep"></span>
    <button class="inline-btn" data-action="superscript" title="Superscript">x²</button>
    <button class="inline-btn" data-action="subscript" title="Subscript">x₂</button>
    <span class="inline-sep"></span>
    <button class="inline-btn" data-action="clearFormat" title="Clear Formatting">Tx</button>
    <span class="inline-sep"></span>
    <button class="inline-btn" data-action="justifyLeft" title="Align Left">≡</button>
    <button class="inline-btn" data-action="justifyCenter" title="Align Center">≡</button>
    <button class="inline-btn" data-action="justifyRight" title="Align Right">≡</button>
    <span class="inline-sep"></span>
    <button class="inline-btn" data-action="insertUnorderedList" title="Bullet List">•≡</button>
    <button class="inline-btn" data-action="insertOrderedList" title="Numbered List">1≡</button>
    <span class="inline-sep"></span>
    <button class="inline-btn" data-action="insertLink" title="Insert Link">🔗</button>
    <button class="inline-btn" data-action="removeLink" title="Remove Link">🔗</button>
  </div>
  <!-- Style row -->
  <div class="inline-toolbar-row">
    <select class="inline-select" data-action="fontName" title="Font Family">
      <option value="Georgia, serif">Georgia</option>
      <option value="Arial, sans-serif">Arial</option>
      <option value="'Helvetica Neue', sans-serif">Helvetica</option>
      <option value="'Courier New', monospace">Courier</option>
      <option value="system-ui, sans-serif">System</option>
    </select>
    <input class="inline-input" type="number" data-action="fontSize" min="6" max="200" value="16" title="Font Size" style="width:52px">
    <select class="inline-select" data-action="fontWeight" title="Font Weight" style="width:72px">
      <option value="100">Thin</option>
      <option value="300">Light</option>
      <option value="400">Regular</option>
      <option value="500">Medium</option>
      <option value="600">SemiBold</option>
      <option value="700">Bold</option>
      <option value="900">Black</option>
    </select>
    <input class="inline-input" type="color" data-action="foreColor" value="#111113" title="Text Color" style="width:32px;height:28px;padding:1px;border-radius:4px;cursor:pointer">
    <input class="inline-input" type="color" data-action="hiliteColor" value="#FFFF00" title="Highlight Color" style="width:32px;height:28px;padding:1px;border-radius:4px;cursor:pointer">
  </div>
  <!-- Action row -->
  <div class="inline-toolbar-row">
    <button class="inline-btn inline-btn-done" data-action="done" title="Done (Escape)">✓</button>
  </div>
</div>
```

### 4.3 CSS for Inline Toolbar

```css
.inline-toolbar {
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-sm);
  box-shadow: var(--shadow-lg);
  padding: 4px;
  display: flex;
  flex-direction: column;
  gap: 4px;
  user-select: none;
  -webkit-user-select: none;
}

.inline-toolbar-row {
  display: flex;
  gap: 2px;
  align-items: center;
  flex-wrap: nowrap;
}

.inline-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 26px;
  min-height: 26px;
  padding: 0 6px;
  border: 1px solid transparent;
  border-radius: 3px;
  background: transparent;
  color: var(--color-text);
  font-size: 12px;
  cursor: pointer;
  line-height: 1;
  transition: background 0.1s;
}

.inline-btn:hover {
  background: var(--color-bg-subtle);
  border-color: var(--color-border);
}

.inline-btn:active,
.inline-btn.active {
  background: var(--color-accent);
  color: #fff;
  border-color: var(--color-accent);
}

.inline-btn.done {
  background: var(--color-success);
  color: #fff;
  font-size: 14px;
  min-width: 32px;
}

.inline-sep {
  display: inline-block;
  width: 1px;
  height: 18px;
  margin: 0 3px;
  background: var(--color-border);
}

.inline-select,
.inline-input {
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  color: var(--color-text);
  padding: 2px 4px;
  border-radius: 3px;
  font-size: 11px;
  min-height: 28px;
  font-family: inherit;
}

.inline-select:focus,
.inline-input:focus {
  border-color: var(--color-accent);
  outline: none;
  box-shadow: 0 0 0 2px rgba(45,126,255,.2);
}
```

### 4.4 Toolbar Positioning

The toolbar must position itself **above** the editing element, accounting for:
- Canvas zoom level
- Canvas pan offset
- Available viewport space (don't go off-screen)
- Element rotation (if any)

**Important:** Do not recompute the element's screen position from `state.zoom`/`state.panX`/`state.panY` by hand (an earlier draft of this section did, and it was wrong — see the correction in §2.3). Zoom/pan is a single CSS `transform` on `#canvas-page`; the *editing element itself* is the source of truth for where it currently sits on screen. Read its rect directly with `getBoundingClientRect()`, which is correct at any zoom, pan, or rotation:

```js
function showInlineToolbar(dom) {
  var toolbar = $('inline-toolbar');
  if (!toolbar) return;
  
  // dom.getBoundingClientRect() already reflects the ancestor CSS transform
  // (zoom + pan) applied to #canvas-page — no manual math needed.
  var elRect = dom.getBoundingClientRect();
  
  // Make toolbar visible first (off-screen) so offsetWidth/Height are accurate
  toolbar.style.visibility = 'hidden';
  toolbar.style.display = 'flex';
  var toolbarWidth = toolbar.offsetWidth;
  var toolbarHeight = toolbar.offsetHeight;
  
  var left = elRect.left + elRect.width / 2 - toolbarWidth / 2;
  var top = elRect.top - toolbarHeight - 8;
  
  // Not enough room above — place below the element instead
  if (top < 8) {
    top = elRect.bottom + 8;
    toolbar.classList.add('below');
  } else {
    toolbar.classList.remove('below');
  }
  
  // Clamp horizontally to viewport
  left = Math.max(8, Math.min(left, window.innerWidth - toolbarWidth - 8));
  
  toolbar.style.left = left + 'px';
  toolbar.style.top = top + 'px';
  toolbar.style.visibility = '';
  
  // Update toolbar state to reflect current selection
  updateInlineToolbarState();
}

function hideInlineToolbar() {
  var toolbar = $('inline-toolbar');
  if (toolbar) toolbar.style.display = 'none';
}
```

Because this reads the live DOM rect, no separate repositioning math is needed for zoom, pan, or panel-collapse changes — `showInlineToolbar(dom)` can simply be re-invoked after any of those events (see §6.6, §9.10, §9.11).

### 4.5 Toolbar Event Binding

```js
function bindInlineToolbar() {
  var toolbar = $('inline-toolbar');
  if (!toolbar) return;
  
  // Binding for all toolbar buttons
  var actions = {
    'bold': 'bold',
    'italic': 'italic',
    'underline': 'underline',
    'strikethrough': 'strikeThrough',
    'superscript': 'superscript',
    'subscript': 'subscript',
    'clearFormat': 'removeFormat',
    'justifyLeft': 'justifyLeft',
    'justifyCenter': 'justifyCenter',
    'justifyRight': 'justifyRight',
    'insertUnorderedList': 'insertUnorderedList',
    'insertOrderedList': 'insertOrderedList',
    'insertLink': 'createLink',
    'removeLink': 'removeFormat', // simplified — actual link removal is complex
    'done': 'done'
  };
  
  toolbar.querySelectorAll('[data-action]').forEach(function(btn) {
    btn.addEventListener('click', function(e) {
      e.preventDefault();
      var action = this.dataset.action;
      
      if (action === 'done') {
        exitInlineEdit(true);
        return;
      }
      
      // Get the editing element
      var id = state.editingElementId;
      var dom = document.querySelector('.canvas-element[data-id="' + id + '"]');
      if (!dom || !dom.isContentEditable) return;
      
      dom.focus();
      
      // Handle special cases
      if (action === 'createLink') {
        var url = prompt('Enter link URL:', 'https://');
        if (url) {
          document.execCommand('createLink', false, url);
        }
      } else if (action === 'removeFormat') {
        document.execCommand('removeFormat', false, null);
        // Also remove link if in selection
        document.execCommand('unlink', false, null);
      } else {
        document.execCommand(action, false, null);
      }
      
      // Update toolbar state after change
      updateInlineToolbarState();
    });
  });
  
  // Font size input
  var fsInput = toolbar.querySelector('[data-action="fontSize"]');
  if (fsInput) {
    fsInput.addEventListener('input', function() {
      var id = state.editingElementId;
      var dom = document.querySelector('.canvas-element[data-id="' + id + '"]');
      if (!dom) return;
      dom.focus();
      var val = parseInt(this.value) || 16;
      
      // execCommand fontSize only supports 1-7, so use span wrapping
      var sel = window.getSelection();
      if (!sel.rangeCount || !sel.toString()) return;
      
      var range = sel.getRangeAt(0);
      var wrap = document.createElement('span');
      wrap.style.fontSize = val + 'px';
      try {
        range.surroundContents(wrap);
      } catch (e) {
        var frag = range.extractContents();
        wrap.appendChild(frag);
        range.insertNode(wrap);
      }
      sel.removeAllRanges();
      sel.addRange(range);
      
      updateInlineToolbarState();
    });
  }
  
  // Font family select
  var ffSelect = toolbar.querySelector('[data-action="fontName"]');
  if (ffSelect) {
    ffSelect.addEventListener('change', function() {
      var dom = getActiveEditable();
      if (!dom) return;
      dom.focus();
      document.execCommand('fontName', false, this.value);
      updateInlineToolbarState();
    });
  }
  
  // Font weight select
  var fwSelect = toolbar.querySelector('[data-action="fontWeight"]');
  if (fwSelect) {
    fwSelect.addEventListener('change', function() {
      var dom = getActiveEditable();
      if (!dom) return;
      dom.focus();
      var wrap = document.createElement('span');
      wrap.style.fontWeight = this.value;
      var sel = window.getSelection();
      if (!sel.rangeCount) return;
      var range = sel.getRangeAt(0);
      if (!sel.toString()) return;
      try {
        range.surroundContents(wrap);
      } catch (e) {
        var frag = range.extractContents();
        wrap.appendChild(frag);
        range.insertNode(wrap);
      }
      sel.removeAllRanges();
      sel.addRange(range);
      updateInlineToolbarState();
    });
  }
  
  // Text color input
  var tcInput = toolbar.querySelector('[data-action="foreColor"]');
  if (tcInput) {
    tcInput.addEventListener('input', function() {
      var dom = getActiveEditable();
      if (!dom) return;
      dom.focus();
      document.execCommand('styleWithCSS', false, true);
      document.execCommand('foreColor', false, this.value);
      updateInlineToolbarState();
    });
  }
  
  // Highlight color input
  var hcInput = toolbar.querySelector('[data-action="hiliteColor"]');
  if (hcInput) {
    hcInput.addEventListener('input', function() {
      var dom = getActiveEditable();
      if (!dom) return;
      dom.focus();
      document.execCommand('styleWithCSS', false, true);
      document.execCommand('hiliteColor', false, this.value);
      updateInlineToolbarState();
    });
  }
}

function getActiveEditable() {
  var id = state.editingElementId;
  if (!id) return null;
  return document.querySelector('.canvas-element[data-id="' + id + '"]');
}
```

### 4.6 Toolbar State Reflection

The toolbar buttons must reflect the **current selection state** within the editable element. This is critical for user confidence — the user must see which formats are active.

```js
function updateInlineToolbarState() {
  var dom = getActiveEditable();
  if (!dom) return;
  dom.focus();
  
  var toolbar = $('inline-toolbar');
  if (!toolbar) return;
  
  // Update button active states
  var fmtButtons = {
    'bold': 'bold',
    'italic': 'italic',
    'underline': 'underline',
    'strikethrough': 'strikeThrough',
    'superscript': 'superscript',
    'subscript': 'subscript'
  };
  
  Object.keys(fmtButtons).forEach(function(btnId) {
    var btn = toolbar.querySelector('[data-action="' + btnId + '"]');
    if (!btn) return;
    var isActive = document.queryCommandState(fmtButtons[btnId]);
    btn.classList.toggle('active', isActive);
  });
  
  // Update font family select
  var ffSelect = toolbar.querySelector('[data-action="fontName"]');
  if (ffSelect) {
    var currentFont = document.queryCommandValue('fontName');
    if (currentFont) {
      // Try to match to an option
      for (var i = 0; i < ffSelect.options.length; i++) {
        if (ffSelect.options[i].value.toLowerCase() === currentFont.toLowerCase().replace(/['"]/g, '')) {
          ffSelect.value = ffSelect.options[i].value;
          break;
        }
      }
    }
  }
  
  // Update font size input
  var fsInput = toolbar.querySelector('[data-action="fontSize"]');
  if (fsInput) {
    // queryCommandValue('fontSize') returns 1-7, not px value
    // We need to compute from the actual DOM
    var sel = window.getSelection();
    if (sel.rangeCount) {
      var node = sel.anchorNode;
      if (node && node.nodeType === 3) node = node.parentElement;
      while (node && node !== dom) {
        if (node.nodeType === 1 && node.style.fontSize) {
          fsInput.value = parseInt(node.style.fontSize) || 16;
          break;
        }
        node = node.parentElement;
      }
    }
  }
  
  // Update font weight select
  var fwSelect = toolbar.querySelector('[data-action="fontWeight"]');
  if (fwSelect) {
    var sel = window.getSelection();
    if (sel.rangeCount) {
      var node = sel.anchorNode;
      if (node && node.nodeType === 3) node = node.parentElement;
      while (node && node !== dom) {
        if (node.nodeType === 1 && node.style.fontWeight) {
          fwSelect.value = node.style.fontWeight;
          break;
        }
        node = node.parentElement;
      }
    }
  }
  
  // Update text color
  var tcInput = toolbar.querySelector('[data-action="foreColor"]');
  if (tcInput) {
    var color = document.queryCommandValue('foreColor');
    if (color) {
      // Convert rgb() to hex
      var match = color.match(/rgb\((\d+),\s*(\d+),\s*(\d+)\)/);
      if (match) {
        var hex = '#' + match[1].toString(16).padStart(2,'0') + 
                  match[2].toString(16).padStart(2,'0') + 
                  match[3].toString(16).padStart(2,'0');
        tcInput.value = hex;
      }
    }
  }
}
```

### 4.7 Toolbar Lifecycle

```js
// In init():
bindInlineToolbar();

// When entering edit mode:
showInlineToolbar(dom);

// When exiting edit mode:
hideInlineToolbar();

// On zoom change: reposition toolbar
// On pan change: reposition toolbar
// On element resize during edit: reposition toolbar
```

---

## §5. Selection & Caret Management

### 5.1 The Core Problem

Browser `Selection` and `Range` APIs work on the DOM tree. When a user selects text in the canvas element, the selection exists **inside that element's DOM**. The challenge is:

1. **Preserving selection across state updates** — When the panel updates or the toolbar changes a format, the selection can be lost.
2. **Cross-element selections** — If the user selects text that spans multiple child nodes (e.g., across `<p>` tags), `surroundContents()` will throw.
3. **Zoom changes** — The visual selection highlight must remain accurate at any zoom level.
4. **Rotation** — The element may be rotated; selection coordinates must account for this.

### 5.2 Selection Preservation Strategy

```js
// Store a serializable representation of the selection
var savedSelection = null;

function saveSelection() {
  var dom = getActiveEditable();
  if (!dom) return;
  
  var sel = window.getSelection();
  if (!sel.rangeCount) return;
  
  var range = sel.getRangeAt(0);
  
  // Serialize to a path-based representation
  savedSelection = {
    startPath: getPathToNode(range.startContainer, range.startOffset),
    endPath: getPathToNode(range.endContainer, range.endOffset),
    isCollapsed: sel.isCollapsed
  };
}

function restoreSelection() {
  if (!savedSelection) return;
  
  var dom = getActiveEditable();
  if (!dom) { savedSelection = null; return; }
  
  var sel = window.getSelection();
  sel.removeAllRanges();
  
  var startNode = getNodeAtPath(dom, savedSelection.startPath);
  var endNode = getNodeAtPath(dom, savedSelection.endPath);
  
  if (startNode && endNode) {
    var range = document.createRange();
    range.setStart(startNode.node, startNode.offset);
    range.setEnd(endNode.node, endNode.offset);
    sel.addRange(range);
  }
  
  savedSelection = null;
}

// Path representation: array of {tag, index} for each ancestor
function getPathToNode(node, offset) {
  var path = [];
  var current = node;
  while (current && current !== document.body && current.parentElement) {
    var parent = current.parentElement;
    var children = Array.prototype.slice.call(parent.children || parent.childNodes);
    var index = children.indexOf(current);
    path.push({ tag: current.tagName, index: index, text: current.nodeType === 3 ? current.textContent : null });
    current = parent;
  }
  return { path: path, offset: offset };
}

function getNodeAtPath(container, pathData) {
  var current = container;
  for (var i = pathData.path.length - 1; i >= 0; i--) {
    var target = pathData.path[i];
    var children = Array.prototype.slice.call(current.children || current.childNodes);
    var found = children[target.index];
    if (!found) return null;
    current = found;
  }
  return { node: current, offset: pathData.offset };
}
```

### 5.3 Cross-Element Selection Handling

When the user's selection spans multiple child elements, `execCommand` and `surroundContents()` behave differently. The key insight: **`execCommand` operates on the selection in the DOM, not on a specific element**. As long as focus is on the contentEditable element, `execCommand` works correctly regardless of which child nodes are selected.

```js
function applyFormatToSelection(command, value) {
  var dom = getActiveEditable();
  if (!dom) return false;
  
  var sel = window.getSelection();
  if (!sel.rangeCount || sel.isCollapsed) return false;
  
  // Save selection before execCommand (which may modify DOM)
  saveSelection();
  
  document.execCommand(command, false, value || null);
  
  // Restore selection
  restoreSelection();
  
  // Sync to state
  syncEditToState();
  
  return true;
}

// For operations that need wrapping (font size, font weight):
function wrapSelection(wrapperTag, styleObj) {
  var dom = getActiveEditable();
  if (!dom) return false;
  
  var sel = window.getSelection();
  if (!sel.rangeCount || sel.isCollapsed) return false;
  
  saveSelection();
  
  var range = sel.getRangeAt(0);
  var wrapper = document.createElement(wrapperTag);
  Object.keys(styleObj).forEach(function(prop) {
    wrapper.style.setProperty(prop, styleObj[prop]);
  });
  
  try {
    range.surroundContents(wrapper);
  } catch (e) {
    // Selection spans multiple nodes — extract and wrap
    var frag = range.extractContents();
    wrapper.appendChild(frag);
    range.insertNode(wrapper);
    // Expand range to include the wrapper
    range.setStartAfter(wrapper);
    range.collapse(false);
  }
  
  sel.removeAllRanges();
  sel.addRange(range);
  
  syncEditToState();
  return true;
}
```

### 5.4 Live Sync During Editing

The canvas DOM element must stay in sync with the data model during editing:

```js
function syncEditToState() {
  var id = state.editingElementId;
  if (!id) return;
  
  var dom = document.querySelector('.canvas-element[data-id="' + id + '"]');
  if (!dom) return;
  
  var el = getEl(id);
  if (!el) return;
  
  // Update state from DOM
  el.richText = dom.innerHTML;
  el.text = dom.innerHTML.replace(/<[^>]*>/g, '');
  
  // Debounced save
  clearTimeout(state.editSaveTimer);
  state.editSaveTimer = setTimeout(function() {
    saveSnapshot();
    save();
  }, 300);
}
```

### 5.5 Caret Position After Format Application

After applying a format, the caret should remain where the user expects:

- **Bold/Italic/Underline:** Caret stays at its current position
- **Font size change:** Caret stays at its current position within the wrapped span
- **Alignment change:** Caret stays at its current position
- **List toggle:** Caret stays at its current position
- **Link insertion:** Caret is placed at the end of the inserted link
- **Clear formatting:** Caret stays at its current position

The `saveSelection()` / `restoreSelection()` pair handles all of these automatically.

---

---

## §6. Canvas Interaction During Edit Mode

### 6.1 Pointer Event Handling

When in inline edit mode, pointer events on the canvas must be carefully managed:

| Event | Behavior |
|-------|----------|
| Click inside editing element | Pass through to browser (caret placement, text selection) |
| Click outside editing element | Exit edit mode, save changes, select clicked element |
| Click on another text element | Exit edit mode, save changes, select new element |
| Click on non-text element | Exit edit mode, save changes, select non-text element |
| Double-click inside editing element | Ignore (already in edit mode) |
| Double-click outside editing element | Standard double-click behavior |
| Drag inside editing element | Pass through (text selection) |
| Drag outside editing element | Exit edit mode first |
| Resize handle click | Exit edit mode first |
| Escape key | Exit edit mode, cancel (don't save) |
| Enter key | Insert line break (native browser behavior) |
| Ctrl+A | Select all text in the editing element |
| Ctrl+C/V/X | Standard clipboard (native browser behavior) |
| Ctrl+Z | Undo in the editor (native browser behavior) |
| Ctrl+B/I/U | Apply formatting (handled by toolbar + native) |

### 6.2 Pointer Event Modifications

**In `onPointerDown` (~line 2720):**

```js
function onPointerDown(e) {
  // If in edit mode, check if click is inside the editing element
  if (state.editingElementId) {
    var editDom = document.querySelector('.canvas-element[data-id="' + state.editingElementId + '"]');
    if (editDom && editDom.contains(e.target)) {
      // Click is inside the editing element — let browser handle it
      return;
    }
    
    // Click is outside — check if it's on the toolbar
    if (e.target.closest('#inline-toolbar')) {
      return; // Let toolbar handle it
    }
    
    // Click is outside — exit edit mode
    exitInlineEdit(true);
  }
  
  // ... rest of existing pointer down logic ...
}
```

**In `onPointerMove` (~line 2860):**

```js
function onPointerMove(e) {
  // If in edit mode and pointer is outside editing element
  if (state.editingElementId) {
    var editDom = document.querySelector('.canvas-element[data-id="' + state.editingElementId + '"]');
    if (editDom && !editDom.contains(e.target)) {
      // Pointer has moved outside — exit edit mode
      exitInlineEdit(true);
    }
  }
  
  // ... rest of existing pointer move logic ...
}
```

**In `onPointerUp` (~line 2900):**

```js
function onPointerUp(e) {
  // If in edit mode, let browser handle the click
  if (state.editingElementId) {
    return; // Don't process as canvas interaction
  }
  
  // ... rest of existing pointer up logic ...
}
```

### 6.3 Preventing Text Selection on Non-Editable Elements

When the user drags from inside the editing element to outside, the browser may try to extend the selection to elements outside the editable region. Prevent this:

```css
/* Add to CSS */
.canvas-element:not([contenteditable]) {
  -webkit-user-select: none;
  user-select: none;
}

.inline-toolbar {
  -webkit-user-select: none;
  user-select: none;
}
```

### 6.4 Keyboard Event Handling

Modify `onKey` (~line 2910) to handle edit-mode-specific shortcuts:

```js
function onKey(e) {
  var key = e.key.toLowerCase();
  var ctrl = e.ctrlKey || e.metaKey;
  
  /* If in inline edit mode, handle edit-specific keys */
  if (state.editingElementId) {
    var editDom = getActiveEditable();
    if (editDom) {
      // Escape — exit edit mode
      if (key === 'escape') {
        e.preventDefault();
        exitInlineEdit(true);
        return;
      }
      
      // Tab — insert tab character (not switch focus)
      if (key === 'tab') {
        e.preventDefault();
        document.execCommand('insertText', false, '\t');
        return;
      }
      
      // Ctrl+K — insert link
      if (ctrl && key === 'k') {
        e.preventDefault();
        var url = prompt('Enter link URL:', 'https://');
        if (url) {
          document.execCommand('createLink', false, url);
        }
        return;
      }
      
      // Ctrl+Shift+K — remove link
      if (ctrl && e.shiftKey && key === 'k') {
        e.preventDefault();
        document.execCommand('unlink', false, null);
        return;
      }
      
      // Ctrl+Shift+S — superscript
      if (ctrl && e.shiftKey && key === 's') {
        e.preventDefault();
        document.execCommand('superscript', false, null);
        return;
      }
      
      // Ctrl+Shift+Comma — subscript
      if (ctrl && e.shiftKey && key === ',') {
        e.preventDefault();
        document.execCommand('subscript', false, null);
        return;
      }
      
      // Ctrl+L — justify left
      if (ctrl && e.shiftKey && key === 'l') {
        e.preventDefault();
        document.execCommand('justifyLeft', false, null);
        return;
      }
      
      // Ctrl+E — justify center
      if (ctrl && e.shiftKey && key === 'e') {
        e.preventDefault();
        document.execCommand('justifyCenter', false, null);
        return;
      }
      
      // Ctrl+R — justify right
      if (ctrl && e.shiftKey && key === 'r') {
        e.preventDefault();
        document.execCommand('justifyRight', false, null);
        return;
      }
      
      // Ctrl+Shift+J — justify full
      if (ctrl && e.shiftKey && key === 'j') {
        e.preventDefault();
        document.execCommand('justifyFull', false, null);
        return;
      }
    }
  }
  
  /* ... rest of existing key handling ... */
}
```

### 6.5 Auto-Resize During Editing

When `autoResize` is enabled, the text box must grow/shrink as content changes:

```js
function handleAutoResize(dom, el) {
  if (!el.autoResize) return;
  
  // Calculate new height based on scrollHeight
  var newH = dom.scrollHeight;
  
  // Apply padding back
  var pad = el.padding != null ? el.padding : 8;
  newH += pad * 2;
  
  // Apply min/max constraints
  if (el.minH > 0) newH = Math.max(newH, el.minH);
  if (el.maxH > 0) newH = Math.min(newH, el.maxH);
  
  // Only update if changed by more than 1px (avoid infinite loops)
  if (Math.abs(newH - el.h) > 1) {
    el.h = newH;
    
    // Update DOM directly (no full re-render)
    dom.style.height = newH + 'px';
    
    // Reposition toolbar if visible
    if (state.editingElementId) {
      showInlineToolbar(dom);
    }
    
    // Notify state change
    saveSnapshot();
    emit();
  }
}
```

Attach to the `input` event on the editing element:

```js
// In enterInlineEdit():
dom.addEventListener('input', function() {
  var el = getEl(state.editingElementId);
  if (el) handleAutoResize(dom, el);
  syncEditToState();
});
```

### 6.6 Overflow Handling During Editing

When `overflow` is `scroll` or `visible`, the editing element must show scrollbars or extend beyond its bounds:

```css
/* For scroll overflow */
.canvas-element[contenteditable][data-overflow="scroll"] {
  overflow-y: auto;
  overflow-x: hidden;
}

/* For visible overflow */
.canvas-element[contenteditable][data-overflow="visible"] {
  overflow: visible;
}

/* For clip overflow */
.canvas-element[contenteditable][data-overflow="clip"],
.canvas-element[contenteditable] {
  overflow: hidden;
}
```

---

## §7. Panel Integration

### 7.1 Panel Text Editor Sync

When the user is in inline edit mode, the panel `#prop-text-content` must reflect the current content but must not accept input:

```js
// In enterInlineEdit():
var panelEditor = $('prop-text-content');
if (panelEditor) {
  panelEditor.setAttribute('contenteditable', 'false');
  panelEditor.style.opacity = '0.5';
  panelEditor.style.pointerEvents = 'none';
}

// In exitInlineEdit():
var panelEditor = $('prop-text-content');
if (panelEditor) {
  panelEditor.setAttribute('contenteditable', 'true');
  panelEditor.style.opacity = '';
  panelEditor.style.pointerEvents = '';
  // Update panel editor to reflect the saved content
  panelEditor.innerHTML = el.richText || el.text;
}
```

### 7.2 Properties Panel Updates

When a text element is selected (in or out of edit mode), the properties panel must show the correct values:

```js
// In updateProps() (~line 1770):
if (el.type === 'text') {
  var ed = $('prop-text-content');
  if (ed && state.editingElementId !== el.id) {
    // Only update if not currently editing this element
    if (document.activeElement !== ed) {
      ed.innerHTML = el.richText || el.text;
    }
  }
  // ... rest of existing property updates ...
}
```

### 7.3 Format Button State in Panel

The panel's format buttons (bold, italic, etc.) must reflect the state of the **currently selected text** within the editing element.

**Note:** `updatePanelEditorButtons()` already exists in `index.html` (inside `bindPanelEditor()`, line ~2316) and is already wired to `#prop-text-content`'s `keyup`/`mouseup` events. No new function is needed for the *panel's own* editor. What's new for inline canvas editing is making this same logic fire when the selection changes **inside the canvas element**, not just inside `#prop-text-content` — because once inline editing exists, `document.queryCommandState()` reflects whichever contentEditable region currently has focus, regardless of which one the user is nominally looking at.

Extend the existing binding to also listen on the canvas editing element:

```js
// Extend enterInlineEdit() to reuse the existing updatePanelEditorButtons()
// (already defined in bindPanelEditor's closure — expose it or duplicate
// the small lookup table if closure access isn't convenient):
dom.addEventListener('keyup', updatePanelEditorButtons);
dom.addEventListener('mouseup', updatePanelEditorButtons);
```

This keeps a single implementation of the button-state logic rather than introducing a duplicate, so the panel buttons and any future inline-toolbar buttons stay in sync by construction.

### 7.4 Toolbar-to-Panel Sync

Changes made via the inline toolbar must update the panel's font/size/color inputs:

```js
// After any format change in inline toolbar:
function syncPanelFromInlineEdit() {
  var id = state.editingElementId;
  if (!id) return;
  var el = getEl(id);
  if (!el) return;
  
  // Update font family
  var ff = $('prop-ff');
  if (ff) {
    var currentFont = document.queryCommandValue('fontName');
    if (currentFont) ff.value = currentFont.replace(/['"]/g, '');
  }
  
  // Update font size — computed from selection
  var fs = $('prop-fs');
  if (fs) {
    var sel = window.getSelection();
    if (sel.rangeCount) {
      var node = sel.anchorNode;
      if (node && node.nodeType === 3) node = node.parentElement;
      while (node && node !== document.body) {
        if (node.style && node.style.fontSize) {
          fs.value = parseInt(node.style.fontSize);
          break;
        }
        node = node.parentElement;
      }
    }
  }
  
  // Update text color
  var tc = $('prop-tc');
  if (tc) {
    var color = document.queryCommandValue('foreColor');
    if (color) {
      var match = color.match(/rgb\((\d+),\s*(\d+),\s*(\d+)\)/);
      if (match) {
        tc.value = '#' + match[1].toString(16).padStart(2,'0') + 
                   match[2].toString(16).padStart(2,'0') + 
                   match[3].toString(16).padStart(2,'0');
      }
    }
  }
}
```

---

## §8. Pixel-Perfect Rendering Guarantees

### 8.1 The Core Problem

The user's confidence that "what they see is what they get" depends on **identical rendering** between:
1. The inline editing element on the canvas
2. The exported HTML
3. The panel text editor preview
4. The page thumbnail

Any discrepancy between these breaks trust.

### 8.2 Rendering Consistency Rules

**Rule 1: Single style computation path.**
All text rendering must use the same CSS property values. Never compute styles differently for edit mode vs. display mode.

**Rule 2: Inline styles for edit mode.**
The contentEditable element must use inline styles (not class-based styles) to match the display element exactly.

**Rule 3: No browser default style differences.**
Browser default styles for contentEditable differ from normal divs. Override all defaults:

```css
.canvas-element[contenteditable] {
  /* Reset browser defaults for contentEditable */
  outline: none;
  caret-color: var(--color-accent);
  cursor: text;
  
  /* Match the display element exactly */
  font-family: inherit;
  font-size: inherit;
  line-height: inherit;
  color: inherit;
  
  /* Prevent unwanted behavior */
  -webkit-user-modify: read-write;
  -moz-user-modify: read-write;
  overflow-wrap: break-word;
  word-wrap: break-word;
  hyphens: auto;
  
  /* Ensure text renders the same */
  -webkit-text-size-adjust: none;
  text-size-adjust: none;
}

/* Ensure child elements inherit correctly */
.canvas-element[contenteditable] p,
.canvas-element[contenteditable] div,
.canvas-element[contenteditable] span {
  font: inherit;
  color: inherit;
}
```

**Rule 4: Font loading synchronization.**
If Google Fonts are used, ensure fonts are loaded before editing begins:

```js
// Track font load status
var fontLoadPromises = [];

function ensureFontsLoaded(fontFamilies) {
  fontFamilies.forEach(function(fontFamily) {
    if (fontFamily.indexOf('google') !== -1 || fontFamily.indexOf('fonts.googleapis') !== -1) {
      // For Google Fonts, wait for document.fonts.ready
      fontLoadPromises.push(document.fonts.ready);
    }
  });
}

// In exportHTML(): ensure fonts are embedded before generating HTML
// In enterInlineEdit(): ensure fonts are loaded before focusing
```

### 8.3 Zoom-Level Rendering Accuracy

At non-100% zoom, text rendering can appear slightly different due to sub-pixel rendering and font hinting at different scales. This is a **rendering** concern, not a positioning concern — do not confuse it with coordinate mapping (already covered, correctly, in §2.3).

**Do not disable or bypass the existing zoom transform.** `applyTransform()` (line ~1122) scales `#canvas-page` as a whole via `transform: translate(...) scale(...)`. This is precisely what makes editing WYSIWYG-consistent with the non-editing render: the *same* DOM node, with the *same* unzoomed inline styles, is simply displayed at a different scale. Introducing a second, parallel "screen-space" coordinate system for edit mode (as an earlier draft of this section proposed) would mean the editing element no longer shares styles with the display element, defeating the entire pixel-perfect goal of this plan. `applyElementStyles(dom, el)` (§2.2) must set `left`/`top`/`width`/`height`/`fontSize`/etc. in **unzoomed page-space px**, identically to `domEl()` — exactly like every other canvas element, editable or not.

What *does* need attention at extreme zoom levels:

1. **Sub-pixel font rendering.** Force consistent antialiasing so text doesn't look different when the same scaled font size lands on a fractional device pixel at one zoom level vs. a whole pixel at another:

```css
.canvas-element[contenteditable] {
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-rendering: optimizeLegibility;
}
```

2. **Caret and selection handle scaling.** The native caret and selection highlight are drawn by the browser *inside* the scaled element, so they scale automatically with `transform: scale()`. No extra work is needed, but test at extreme zoom (25%, 400%) to confirm the caret remains visible and clickable — at very low zoom the caret can become sub-pixel thin.

3. **Toolbar stays unscaled.** The inline toolbar (§4) lives *outside* `#canvas-page` and must not inherit the zoom transform — it should always render at 100% UI scale regardless of canvas zoom, matching how the properties panel and other chrome behave. Confirm its DOM parent is not a descendant of `#canvas-page`.

### 8.4 Export Fidelity

The exported HTML must match the on-canvas rendering exactly:

```js
// In exportHTML():
function exportHTML() {
  // ... existing export code ...
  
  // For text elements, ensure all inline styles are preserved
  state.pages.forEach(function(pg) {
    pg.elements.forEach(function(el) {
      if (el.type === 'text') {
        // Ensure richText is clean
        if (el.richText) {
          // Clean up any editing artifacts
          el.richText = cleanExportHTML(el.richText);
        }
      }
    });
  });
  
  // ... rest of export ...
}

function cleanExportHTML(html) {
  // Remove contentEditable artifacts
  html = html.replace(/contenteditable\s*=\s*["'][^"']*["']/gi, '');
  
  // Remove empty spans
  html = html.replace(/<span\s*style\s*=\s*["'][^"']*["']\s*>\s*<\/span>/gi, '');
  
  // Normalize font tags to spans
  html = html.replace(/<font\s*([^>]*)>(.*?)<\/font>/gi, function(match, attrs, content) {
    var style = '';
    var colorMatch = attrs.match(/color\s*:\s*([^;>]+)/i);
    if (colorMatch) style += 'color:' + colorMatch[1] + ';';
    var faceMatch = attrs.match(/face\s*:\s*([^;>]+)/i);
    if (faceMatch) style += 'font-family:' + faceMatch[1] + ';';
    var sizeMatch = attrs.match(/size\s*:\s*([^;>]+)/i);
    if (sizeMatch) {
      // Convert font size 1-7 to px
      var sizes = [10, 13, 16, 18, 24, 32, 48];
      var size = sizes[parseInt(sizeMatch[1]) - 1] || 16;
      style += 'font-size:' + size + 'px;';
    }
    return '<span style="' + style + '">' + content + '</span>';
  });
  
  return html;
}
```

---

---

## §9. Edge Cases and Failure Modes

### 9.1 Rotation Handling

Text elements can be rotated (0-360°). During inline editing:

**Problem:** A rotated element's bounding box is no longer axis-aligned. The contentEditable div must match the rotated bounding box exactly.

**Solution:** Apply the same `transform: rotate()` to the contentEditable element as to the display element:

```js
function applyElementStyles(dom, el) {
  // ... other styles ...
  
  if (el.rotation) {
    dom.style.transform = 'rotate(' + el.rotation + 'deg)';
    dom.style.transformOrigin = 'center center';
  } else {
    dom.style.transform = '';
  }
}
```

**Caveat:** `transform` changes the hit-testing behavior. During edit mode, pointer events must account for the rotation when determining if a click is inside the element.

### 9.2 Column Layout

When `columnCount > 1`, text flows between columns:

**Problem:** `execCommand` operates on the selection within the contentEditable element, but column layout can split the DOM structure in unexpected ways.

**Solution:** No special handling needed — `execCommand` works on the DOM selection regardless of CSS columns. However, the toolbar positioning must account for multi-column layouts:

```js
// In showInlineToolbar():
// For multi-column, position toolbar at the element's bounding box,
// not at the first column's position
var boundingRect = dom.getBoundingClientRect();
var toolbarWidth = toolbar.offsetWidth || 600;
var left = boundingRect.left + boundingRect.width / 2 - toolbarWidth / 2;
var top = boundingRect.top - toolbarHeight - 10;
```

### 9.3 Vertical Alignment

When `verticalAlign` is `middle` or `bottom`, the text content is positioned via flexbox `justifyContent`. During editing:

**The contentEditable element must maintain the same flexbox structure:**

```js
// In applyElementStyles():
dom.style.display = 'flex';
dom.style.flexDirection = 'column';
dom.style.justifyContent = el.verticalAlign === 'middle' ? 'center' : 
                            el.verticalAlign === 'bottom' ? 'flex-end' : 'flex-start';
```

**Caveat:** When `autoResize` is enabled, the flexbox alignment changes as the element grows. This is correct behavior — the text should remain aligned as the box resizes.

### 9.4 Drop Caps

Drop caps use `::first-letter` pseudo-element:

**Problem:** `contentEditable` elements do not render `::first-letter` in all browsers consistently.

**Solution:** Temporarily disable drop cap styles during editing:

```js
// In enterInlineEdit():
if (el.dropCap) {
  dom.classList.add('drop-cap');
  // Store original state
  dom.dataset.originalDropCap = 'true';
}

// In exitInlineEdit():
if (dom.dataset.originalDropCap) {
  dom.classList.remove('drop-cap');
  delete dom.dataset.originalDropCap;
}
```

### 9.5 RTL (Right-to-Left) Text

**Problem:** `contentEditable` handles RTL differently across browsers. Selection and caret positioning can be reversed.

**Solution:** Ensure the `dir` attribute is set correctly:

```js
// In applyElementStyles():
dom.dir = el.dir || 'ltr';
dom.style.direction = el.dir || 'ltr';
```

**Test:** Verify caret positioning, text selection, and alignment in RTL mode.

### 9.6 Undo/Redo in Inline Edit Mode

**Problem:** The browser's native undo/redo (Ctrl+Z) operates on the contentEditable element's internal history, which is separate from the app's undo/redo stack.

**Solution:**
1. **Let browser handle undo/redo during editing.** This provides the expected text-editing experience.
2. **On exit, push the final state to the app's undo stack.** The app's undo/redo operates at the element level (whole element changes), not the character level.
3. **Document this distinction** for users.

```js
// In exitInlineEdit():
// Push to app undo stack
saveSnapshot();

// Note: Browser undo history is discarded on exit.
// If we wanted to preserve it, we'd need to sync the contentEditable
// undo stack to the app stack, which is complex and error-prone.
```

### 9.7 Spell Check

**Problem:** Browser spell check underlines words with red squiggly lines, which can be distracting during design work.

**Solution:** Spell check is enabled by default (`spellcheck="true"`). Users who want to disable it can:
1. Right-click misspelled words and select "Ignore" (browser native)
2. We could add a toggle in the properties panel for spell check

### 9.8 Image Insert During Editing

**Problem:** Inserting images via `execCommand('insertImage')` or `document.execCommand('insertHTML')` can break the contentEditable element's state.

**Solution:** Use a controlled insertion method:

```js
function insertImageIntoEditor(src, width, height) {
  var dom = getActiveEditable();
  if (!dom) return;
  
  var img = document.createElement('img');
  img.src = src;
  img.style.maxWidth = '100%';
  img.style.height = height ? height + 'px' : 'auto';
  img.draggable = false;
  
  var sel = window.getSelection();
  if (sel.rangeCount) {
    var range = sel.getRangeAt(0);
    range.deleteContents();
    range.insertNode(img);
    // Place cursor after the image
    range.setStartAfter(img);
    range.collapse(true);
    sel.removeAllRanges();
    sel.addRange(range);
  }
  
  syncEditToState();
}
```

### 9.9 Link Editing During Editing

**Problem:** `execCommand('createLink')` creates `<a href="...">` elements. Editing link URLs during inline editing requires a custom UI.

**Solution:** Add a link editing popup when the user clicks an existing link in edit mode:

```js
// In the editing element, listen for link clicks:
dom.addEventListener('click', function(e) {
  var link = e.target.closest('a');
  if (link) {
    e.preventDefault();
    var newUrl = prompt('Edit link URL:', link.href);
    if (newUrl !== null) {
      link.href = newUrl;
      syncEditToState();
    }
  }
});
```

### 9.10 Zoom Changes During Editing

**Problem:** When the user zooms in/out while editing, the toolbar position and element dimensions change.

**Solution:** Listen for zoom changes and reposition the toolbar:

```js
// In the zoom functions:
function zoomIn() {
  state.zoom = clamp(state.zoom + 0.25, state.minZoom, state.maxZoom);
  clampPan();
  applyTransform();
  
  // If in edit mode, reposition toolbar
  if (state.editingElementId) {
    var dom = getActiveEditable();
    if (dom) showInlineToolbar(dom);
  }
}
```

### 9.11 Pan Changes During Editing

**Problem:** When the user pans the canvas while editing, the toolbar may move off-screen.

**Solution:** Listen for pan changes and reposition the toolbar:

```js
// After applyTransform():
if (state.editingElementId) {
  var dom = getActiveEditable();
  if (dom) showInlineToolbar(dom);
}
```

---

## §10. Implementation Order

### Phase 1: Foundation (Core Infrastructure)

1. **Add `state.editingElementId` to state management**
   - Add to the `state` object literal (line ~1069) as `editingElementId: null`
   - **Do not** add it to `saveSnapshot()`/`undo()`/`redo()` — those serialize only `state.pages` (line 1153: `JSON.stringify(state.pages)`), which is the per-element content history. "Currently editing" is transient UI state, not document content, and must never be undoable or redoable itself (undoing should undo *text changes*, not "was I in edit mode").
   - **Do not** persist it in `save()`/`load()` (localStorage) either — reloading the page should never resume inline edit mode with a stale contentEditable DOM. Always start `editingElementId: null` on load.

2. **Implement `enterInlineEdit(elementId)`**
   - Make element contentEditable
   - Apply all styles
   - Focus and place caret
   - Save to state

3. **Implement `exitInlineEdit(saveChanges)`**
   - Remove contentEditable
   - Sync to state
   - Clean up

4. **Add double-click detection in `onPointerDown`**
   - 300ms debounce
   - Inside text element only
   - Not on handles/borders

5. **Modify pointer events to respect edit mode**
   - onPointerDown: pass through if inside editor
   - onPointerMove: exit if dragged out
   - onPointerUp: pass through if in edit mode

6. **Add `applyElementStyles(dom, el)` helper**
   - Replicate all CSS from `domEl()` for text elements
   - Ensure contentEditable compatibility

### Phase 2: Inline Toolbar

7. **Add inline toolbar HTML to DOM**
   - Positioning structure
   - All formatting buttons
   - Font/size/weight/color inputs

8. **Add inline toolbar CSS**
   - Styling matching existing `.rte-*` classes
   - Positioning logic
   - Responsive behavior

9. **Implement toolbar positioning**
   - Above or below element
   - Viewport clamping
   - Zoom/pan aware

10. **Implement toolbar event binding**
    - execCommand for all formats
    - Span wrapping for font size/weight
    - Color input handlers
    - Selection state reflection

### Phase 3: Selection & Sync

11. **Implement selection preservation**
    - saveSelection/restoreSelection
    - Path-based serialization
    - Cross-node selection handling

12. **Implement live sync during editing**
    - input event handler
    - Debounced save
    - Auto-resize integration

13. **Implement panel integration**
    - Disable panel editor during inline edit
    - Sync panel values from inline state
    - Restore panel on exit

### Phase 4: Polish & Edge Cases

14. **Handle rotation**
    - Transform during editing
    - Hit testing with rotation
    - Toolbar positioning for rotated elements

15. **Handle columns**
    - Toolbar positioning for multi-column
    - execCommand behavior in columns

16. **Handle vertical alignment**
    - Flexbox during editing
    - Auto-resize with alignment

17. **Handle RTL**
    - Direction attribute
    - Caret positioning
    - Alignment

18. **Handle zoom/pan during editing**
    - Toolbar repositioning
    - Coordinate mapping

19. **Implement keyboard shortcuts**
    - Escape to exit
    - Ctrl+B/I/U during editing
    - Tab to insert
    - Ctrl+K for links

20. **Implement rich text normalization**
    - Clean up execCommand artifacts
    - Remove empty spans
    - Merge identical spans
    - Normalize paragraph structure

### Phase 5: Export & Testing

21. **Update exportHTML()**
    - Clean exported HTML
    - Preserve all inline styles
    - Handle all edge cases

22. **Add comprehensive testing**
    - Pixel-perfect comparison: edit → exit → verify DOM matches
    - Zoom level testing (50%, 100%, 200%)
    - Rotation testing
    - Column testing
    - RTL testing
    - Undo/redo testing
    - Export fidelity testing

23. **Performance optimization**
    - Debounce saveSnapshot()
    - Minimize DOM queries
    - Optimize normalizeRichText()

---

## §11. Testing Checklist

### Pixel-Perfect Verification

For each test case, compare:
1. The inline editing DOM (while editing)
2. The canvas DOM (after exiting edit mode)
3. The exported HTML
4. The panel editor DOM

All four must produce identical rendering.

| Test Case | Verify |
|-----------|--------|
| Single word bold | Font weight, color, size identical |
| Mixed formatting (bold + italic + color) | Each span renders correctly |
| Font size change (via toolbar) | Size matches input value |
| Font family change | Family renders correctly |
| Text color change | Color matches picker value |
| Highlight color | Background color correct |
| Line height change | Leading matches value |
| Letter spacing | Tracking matches value |
| Paragraph spacing | Margin-bottom on <p> tags |
| Text indent | text-indent on first <p> |
| Left/center/right/justify alignment | Text alignment matches |
| Multi-column layout | Columns render correctly |
| Vertical alignment (top/middle/bottom) | Flexbox alignment correct |
| Auto-resize | Height grows/shrinks correctly |
| Overflow scroll | Scrollbar appears when needed |
| Drop cap | First letter styled correctly |
| Hyphenation | Hyphens appear in justified text |
| RTL text | Direction and alignment correct |
| Rotation | Element renders at correct angle |
| Zoom 50% | Text renders clearly |
| Zoom 200% | Text renders clearly |
| Export HTML | All formatting preserved |
| Undo after edit | State reverts correctly |
| Redo after edit | State restores correctly |
| Panel changes during inline edit | Panel reflects current state |
| Inline toolbar changes | Panel inputs update |
| Escape during edit | Exit without saving |
| Click outside during edit | Exit and save |
| Tool switch during edit | Exit and save |
| Page switch during edit | Exit and save |
| Browser undo during edit | Works correctly |
| Browser redo during edit | Works correctly |
| Copy/paste during edit | Formatting preserved |
| Special character insertion | Character appears correctly |
| Link insertion | Link rendered correctly |
| Link editing | URL updates correctly |
| Bullet list | List renders correctly |
| Numbered list | List renders correctly |
| Clear formatting | All inline styles removed |

---

## §12. Migration Notes for index.html

### Files to Modify

Only `index.html` needs modification — this is a single-file app.

### New State Fields

```js
state.editingElementId = null;  // ID of element currently being edited inline
state.editSaveTimer = null;      // Timer for debounced save during editing
```

### New Functions

| Function | Purpose |
|----------|---------|
| `enterInlineEdit(elementId)` | Enter edit mode on a canvas text element |
| `exitInlineEdit(saveChanges)` | Exit edit mode, optionally saving changes |
| `applyElementStyles(dom, el)` | Apply all text styles to a DOM element |
| `showInlineToolbar(dom)` | Show and position the inline formatting toolbar |
| `hideInlineToolbar()` | Hide the inline formatting toolbar |
| `updateInlineToolbarState()` | Update toolbar button states to reflect selection |
| `bindInlineToolbar()` | Bind event handlers to toolbar elements |
| `getActiveEditable()` | Get the currently editing DOM element |
| `syncEditToState()` | Sync editing DOM to data model |
| `saveSelection()` | Serialize current selection |
| `restoreSelection()` | Restore serialized selection |
| `normalizeRichText(el)` | Clean up rich text HTML |
| `handleAutoResize(dom, el)` | Handle auto-resize during editing |
| `cleanExportHTML(html)` | Clean HTML for export |

### CSS Additions

- `.inline-toolbar` — Toolbar container
- `.inline-toolbar-row` — Toolbar row
- `.inline-btn` — Toolbar button
- `.inline-sep` — Toolbar separator
- `.inline-select` — Toolbar select input
- `.inline-input` — Toolbar input
- `.canvas-element[contenteditable]` — Edit mode styles
- Various attribute selectors for overflow, direction, etc.

### HTML Additions

- `#inline-toolbar` — The inline formatting toolbar (new)

### Modified Functions

| Function | Modification |
|----------|-------------|
| `onPointerDown` | Add edit-mode passthrough, double-click detection |
| `onPointerMove` | Add edit-mode drag-out detection |
| `onPointerUp` | Add edit-mode passthrough |
| `onKey` | Add edit-mode keyboard shortcuts |
| `updateProps` | Add panel editor disable during inline edit |
| `bindProps` | Add panel sync from inline edit |
| `exitInlineEdit` | Add cleanup of all edit-mode artifacts |
| `init` | Ensure `state.editingElementId` starts `null`; call `bindInlineToolbar()` |
| `exportHTML` | Add HTML cleaning |
| `zoomIn/zoomOut` | Add toolbar repositioning |
| `applyTransform` | Add toolbar repositioning |

---

*End of document.*


