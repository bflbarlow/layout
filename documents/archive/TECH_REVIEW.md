# Technical Review — Layout Text Editing

> **Date:** 2025-07-16  
> **Scope:** Full audit of `index.html` (2625 lines, single-file) against the objectives defined in `README.md`  
> **Status:** 9 of 9 objective areas have critical gaps. The UI chrome, page management, canvas interaction, and persistence are solid. The text editing layer is the bottleneck.

---

## Table of Contents

1. [Rich Text Editing Inside the Box](#1-rich-text-editing-inside-the-box)
2. [Paragraph & Line Controls](#2-paragraph--line-controls)
3. [Text Box & Overflow Management](#3-text-box--overflow-management)
4. [Typography & Advanced Text Features](#4-typography--advanced-text-features)
5. [Inline Elements & Rich Content](#5-inline-elements--rich-content)
6. [Text Styles & Presets](#6-text-styles--presets)
7. [Properties Panel Integration](#7-properties-panel-integration)
8. [Export Fidelity](#8-export-fidelity)
9. [Quality of Life](#9-quality-of-life)
10. [Architecture Cross-Cutting Issues](#10-architecture-cross-cutting-issues)
11. [Implementation Roadmap](#11-implementation-roadmap)

---

## 1. Rich Text Editing Inside the Box

### Current State

Text editing relies entirely on `contentEditable` + the deprecated `document.execCommand()` API (lines 1637–1706). The data model uses a flat `richText` string (innerHTML) side-by-side with a plain `text` field, but the tooling around it is minimal.

**What exists:**
- Double-click to enter edit mode (`dblclick` → `contentEditable = true`, line 1282)
- Blur saves `innerHTML` to `richText` and `textContent` to `text` (lines 1299–1301)
- Toolbar buttons for bold, italic, underline, font family, font size, text color, alignment — all use `execCommand`
- `richText` is stored and rendered via `innerHTML` on each page render (line 1265)

### Gaps

| # | Feature | Missing | Impact |
|---|---------|---------|--------|
| 1.1 | **Strikethrough** | No toolbar button, no `execCommand('strikeThrough')` binding | Users cannot add strikethrough to text |
| 1.2 | **Superscript / Subscript** | No UI, no `execCommand('superscript'/'subscript')` | No scientific or footnote notation |
| 1.3 | **Per-selection font family** | `execCommand('fontName')` exists but font-family dropdown only applies when editing (line 1637). When not editing, it applies to the whole element. | Works partially, but the non-editing fallback overwrites all text |
| 1.4 | **Per-selection font size** | `execCommand('fontSize')` is hacky — it uses fixed size '7' then wraps in a `<span>` (lines 1645–1652). The font-size input applies to the whole element when not editing. | Bumpy UX; sizes don't round-trip cleanly through the data model |
| 1.5 | **Per-selection text color** | `execCommand('foreColor')` works inline. But the `textColor` property only stores a single color. | Color changes on non-editing mode overwrite the whole box |
| 1.6 | **Background / highlight color** | No `execCommand('backColor')` or highlight UI | No marker/highlighter effect |
| 1.7 | **Clear formatting** | No button, no `execCommand('removeFormat')` | No way to strip inline formatting |
| 1.8 | **execCommand deprecation** | `document.execCommand` is deprecated in all browsers and may be removed. | Future breakage risk. No fallback to `document.queryCommandState` for correctness. |
| 1.9 | **Rich text round-trip** | `prop-text-content` textarea binds to `el.text` (plain text, lines 1565–1573). When the textarea is edited, it sets `richText: val` (plain text), destroying formatting. | Editing through the Properties panel nukes all inline formatting |
| 1.10 | **Undo during editing** | No snapshot on keystroke inside contentEditable; `saveSnapshot()` only fires on blur. | Ctrl+Z inside the text box triggers the app-level undo (undoes element moves/creations) instead of text changes |

### Required Changes

- Replace `execCommand` with a modern `document.execCommand` wrapper or a lightweight `Range`/`Selection` manipulation library
- Add strikethrough, superscript, subscript, highlight, and clear-format buttons to the toolbar
- Fix the `prop-text-content` textarea to preserve rich text (use `innerHTML` of the selected element, not `textContent`)
- Add per-keystroke or debounced undo snapshots for text content
- Add `queryCommandState` polling to keep toolbar buttons in sync with the cursor position

---

## 2. Paragraph & Line Controls

### Current State

The data model has exactly one paragraph-level property: `textAlign` (supports `left`, `center`, `right`). Everything else is missing.

**Data model fields** (lines 981–989):
```js
b.fontFamily, b.fontSize, b.fontWeight, b.fontStyle, b.textDecoration, b.textAlign, b.textColor
```

That's it. **No fields exist** for line height, letter spacing, paragraph spacing, indent, lists, vertical alignment, columns, or text direction.

### Gaps

| # | Feature | Missing | Impact |
|---|---------|---------|--------|
| 2.1 | **Line height (leading)** | No `lineHeight` property on text elements, no CSS `line-height` set in `domEl()` (line 1265), no UI control | Text lines are always browser-default leading (typically 1.2×). Magazine/playbill layouts cannot be created. |
| 2.2 | **Letter spacing (tracking)** | No `letterSpacing` property, no `letter-spacing` CSS, no UI | Headlines and poster text cannot be tracked out |
| 2.3 | **Paragraph spacing** | No `marginTop`/`marginBottom` on paragraphs. `contentEditable` uses `<div>` or `<p>` implicitly. | Multiple paragraphs run together with no gap control |
| 2.4 | **Text indent** | No `textIndent` property, no CSS `text-indent` | No first-line indent for body text |
| 2.5 | **Bulleted & numbered lists** | No `execCommand('insertOrderedList'/'insertUnorderedList')` bindings, no toolbar buttons | No list creation |
| 2.6 | **Justify alignment** | Not in the `textAlign` select (`prop-ta` only has left/center/right, line 767) | No justified text for magazine columns |
| 2.7 | **Vertical alignment** | No `verticalAlign` property, no CSS `align-items` / `justify-content` on the text box | Text always starts at the top of the box |
| 2.8 | **Text direction** | No `dir` attribute on text elements | No RTL support for Hebrew/Arabic |
| 2.9 | **Columns** | No `column-count` / `column-gap` on text boxes | No magazine-style multi-column text |

### Required Changes

- Add `lineHeight`, `letterSpacing`, `textIndent`, `verticalAlign`, `columnCount`, `columnGap`, `dir` to the text element data model
- Apply `line-height`, `letter-spacing`, `text-indent`, `column-count`, `column-gap`, `dir` in `domEl()` CSS
- Add justify option to alignment controls
- Add list buttons with `execCommand('insertOrderedList'/'insertUnorderedList')` or equivalent
- Add vertical alignment picker (top/middle/bottom) to properties panel
- Add columns control (number + gap) to properties panel

---

## 3. Text Box & Overflow Management

### Current State

Text boxes are fixed-size containers with `overflow: hidden` (CSS `.el-text`, line 397). When editing, overflow becomes `visible` (line 398). There is no auto-resize, no overflow options, no text threading, no padding control.

### Gaps

| # | Feature | Missing | Impact |
|---|---------|---------|--------|
| 3.1 | **Auto-resize** | No `autoResize` property, no logic to expand the box to fit content | Users must manually size text boxes. Content that exceeds the box is hidden (clip) |
| 3.2 | **Overflow modes** | No `overflow` property on text elements. Only `overflow:hidden` (CSS) / `overflow:visible` (when editing) | No scrollable text boxes, no overflow-to-linked |
| 3.3 | **Text threading (linked boxes)** | No `nextTextboxId` or `linkedTo` property. No mechanism to chain text boxes. | Cannot flow text from one box to another. This is a flagship feature of professional layout tools |
| 3.4 | **Padding** | `.el-text` has hardcoded `padding: var(--space-2)` (8px, line 397). No per-element padding property. | All text boxes have identical padding. Cannot inset text independently of box position. |
| 3.5 | **Min/max height** | No `minH`/`maxH` properties on text elements | Auto-resize would grow unbounded. No constraints. |

### Required Changes

- Add `autoResize`, `overflow`, `padding`, `minH`, `maxH`, `nextTextboxId` fields to the text element data model
- Add auto-resize toggle + overflow mode dropdown to the properties panel
- Implement auto-resize logic: on each render (and on content change), measure `scrollHeight` vs `clientHeight` and grow the box if `autoResize` is true, respecting `maxH`
- Implement text threading: when overflow is "link" and `nextTextboxId` is set, overflow text should flow into the linked box. This requires:
  - A content measurement / text truncation algorithm
  - A linked-list traversal during rendering and export
  - A visual indicator (e.g., a "+" badge on boxes with overflow)
- Make `padding` a per-element property rendered via inline style, not hardcoded CSS
- Add padding controls (top, right, bottom, left or uniform) to the properties panel

---

## 4. Typography & Advanced Text Features

### Current State

Font weight is limited to `'400'` and `'700'` (the `prop-fw` select, line 770). Font variants, drop caps, text on path, rotation, hyphenation, and tab stops are entirely absent.

### Gaps

| # | Feature | Missing | Impact |
|---|---------|---------|--------|
| 4.1 | **Full font weight range** | `prop-fw` only has Regular (400) and Bold (700). No 100, 200, 300, 500, 600, 800, 900. | Variable fonts and font families with many weights cannot be used |
| 4.2 | **Font variants** | No `font-variant-caps` (small-caps, all-small-caps, all-petite-caps), no `font-variant-ligatures`, no `font-feature-settings` | No OpenType feature control |
| 4.3 | **Drop caps** | No `::first-letter` styling or dedicated drop-cap UI | No decorative first letters for magazine articles |
| 4.4 | **Text on path** | No SVG `<textPath>` or curved text support | Cannot create circular/curved text for posters |
| 4.5 | **Text rotation** | No `transform: rotate()` on text elements. Elements only have x, y, w, h. | Text boxes are always axis-aligned |
| 4.6 | **Hyphenation** | No `hyphens: auto` CSS, no hyphenation dictionary | Justified text has poor word spacing |
| 4.7 | **Tab stops** | No `tab-size` or tab-stop configuration | Cannot align columns within a text box |

### Required Changes

- Expand `prop-fw` to include all standard numeric weights (100–900)
- Add `font-variant-caps` and `font-feature-settings` to the data model, with UI controls in properties
- Add drop cap toggle (wraps first letter in `<span class="drop-cap">` with larger font size)
- Add `rotation` property to all elements (not just text). Apply via `transform: rotate(Xdeg)` in `domEl()` and export. Adjust hit-testing to account for rotation.
- Add `hyphens: auto` toggle to text element properties
- Add `tab-size` property to text elements

---

## 5. Inline Elements & Rich Content

### Current State

Text boxes support only plain text and inline HTML from `contentEditable`. No inline images, hyperlinks, special character insertion, or find/replace exists.

### Gaps

| # | Feature | Missing | Impact |
|---|---------|---------|--------|
| 5.1 | **Inline images** | No way to insert an image inside a text box. The `image` type creates a separate element. | Cannot add icons or inline graphics inside text |
| 5.2 | **Hyperlinks** | No `execCommand('createLink')` binding, no link UI, no `<a>` tag preservation in export | Text cannot be clickable. Export loses any links users might type manually in contentEditable |
| 5.3 | **Special characters** | No character picker or palette | Users must know HTML entities or copy-paste special characters |
| 5.4 | **Find & replace** | No search functionality | No way to find or replace text across the document |

### Required Changes

- Add "Insert Image" option inside text editing mode (file picker → inserts `<img>` at cursor via `execCommand('insertImage')` or Range API)
- Add link button: opens a small dialog for URL input, uses `execCommand('createLink')`, with unlink support
- Add special character picker (palette popup with common typographic characters: em-dash, en-dash, bullet, ©, ™, °, §, etc.)
- Add find/replace: simple modal with search input, match count, replace, replace-all buttons. Search across all text elements on the current page.

---

## 6. Text Styles & Presets

### Current State

No style system exists. Every text element is styled independently by setting its individual properties.

### Gaps

| # | Feature | Missing | Impact |
|---|---------|---------|--------|
| 6.1 | **Paragraph styles** | No `styleId` or `paragraphStyle` property. No style definition storage. | Each text element must be formatted manually. No consistency across the document. |
| 6.2 | **Character styles** | No inline style classes or character style definitions | Cannot apply a named style to selected text within a paragraph |
| 6.3 | **Quick style picker** | No UI dropdown or palette for applying styles | High friction for applying styles |
| 6.4 | **Style inheritance** | No parent-style → child-style propagation | Changing a style definition does not update elements that use it |

### Required Changes

- Add a `styles` object to the state (separate from pages):
  ```js
  state.styles = {
    paragraphStyles: [
      { id: 'ps1', name: 'Heading 1', fontFamily: '...', fontSize: 36, fontWeight: '700', lineHeight: 1.2, ... },
      { id: 'ps2', name: 'Body', fontFamily: '...', fontSize: 12, lineHeight: 1.5, ... }
    ],
    characterStyles: [
      { id: 'cs1', name: 'Bold Red', fontWeight: '700', textColor: '#FF0000' }
    ]
  }
  ```
- Add `paragraphStyleId` and `characterStyleId` fields to text elements
- Add a "Styles" section to the properties panel with style picker dropdowns
- Add style save/delete/update functionality
- Implement style inheritance: when a paragraph style is updated, iterate all elements referencing it and apply the new properties

---

## 7. Properties Panel Integration

### Current State

The properties panel (lines 743–778) has three relevant sections for text:
- **Text** — A `<textarea>` for plain text content
- **Text Style** — Font, Size, Weight, Align, Color

### Gaps

| # | Issue | Details |
|---|-------|---------|
| 7.1 | **Text content textarea destroys formatting** | `prop-text-content` binds to `el.text` (plain text, line 1565). On input, it sets `{ text: val, richText: val }` (line 1572), overwriting any inline formatting. **Fix:** Show the `innerHTML` of the selected element and update `richText` on edit. |
| 7.2 | **No Typography section** | Missing: line height, letter spacing, font variant controls |
| 7.3 | **No Paragraph section** | Missing: alignment (justify), indent, paragraph spacing, bullets, numbering, columns |
| 7.4 | **No Text Box section** | Missing: padding, vertical alignment, auto-resize toggle, overflow mode, min/max height |
| 7.5 | **No Styles section** | Missing: paragraph style dropdown, character style dropdown |
| 7.6 | **No Advanced section** | Missing: text direction, hyphenation toggle, tab stops |
| 7.7 | **Multi-select text editing** | When multiple text elements are selected, the properties panel only shows the first element's properties. Changing any property applies to all selected elements — but the text content textarea is read from the first element. | Inconsistent: changing font size applies to all, but the textarea only shows one element's content. |

### Required Changes

- Restructure the properties panel into organized sections matching the README objectives:
  - **Text Content** — Rich textarea (preserves HTML)
  - **Typography** — Font family, size, weight, line height, letter spacing, text color, highlight color
  - **Paragraph** — Alignment (with justify), indent, paragraph spacing, lists, columns
  - **Text Box** — Padding, vertical align, auto-resize, overflow, min/max height
  - **Styles** — Paragraph style, character style
  - **Advanced** — Text direction, hyphenation, tab stops
- Fix the textarea to use `innerHTML` instead of `textContent`
- Handle multi-select gracefully: show "Multiple" or a count indicator for fields that differ

---

## 8. Export Fidelity

### Current State

The `exportHTML()` function (lines 1205–1237) generates a standalone HTML document. It uses `el.text` (plain text) not `el.richText` for the content.

```js
// Line 1226 — uses plain text, not rich text:
var inner = el.type === 'text' ? el.text : '';
```

### Gaps

| # | Issue | Details |
|---|-------|---------|
| 8.1 | **Plain text export** | Uses `el.text` (line 1226) instead of `el.richText`. **All inline formatting is lost in export.** This is the single most critical bug in the export. |
| 8.2 | **Missing CSS properties** | The export CSS (lines 1211–1213) does not include `line-height`, `letter-spacing`, `text-indent`, `padding`, `column-count`, `column-gap`, `vertical-align`, `hyphens`, `white-space` — even if those fields are added to the data model. |
| 8.3 | **No font embedding** | No `<link>` to Google Fonts or other font services. Exported HTML uses only font-family names, which may not render on other machines. |
| 8.4 | **No PDF export** | No PDF generation. The only export is HTML. |
| 8.5 | **No linked-box reflow** | If text threading is implemented, the export must render the full text flow across linked boxes. Currently it renders each element independently. |
| 8.6 | **No page size control** | Page size is hardcoded to 850×1100px (`PW`, `PH`). No user-configurable page size or orientation. |

### Required Changes

- Fix line 1226: use `el.richText` instead of `el.text` for text content
- Expand the export CSS template to include all text formatting properties
- Add font embedding: optionally include `<link href="https://fonts.googleapis.com/..." rel="stylesheet">` based on fonts used in the document
- Add PDF export option (use `window.print()` with `@page` CSS or a library-free approach)
- Add page size / orientation controls to the properties panel, and pass them through to export

---

## 9. Quality of Life

### Current State

Basic keyboard shortcuts exist (V, T, R, C, L, Ctrl+Z, Ctrl+D, etc.). Spellcheck is not enabled. Right-click context menu is prevented. No live preview of text changes beyond the full DOM rebuild.

### Gaps

| # | Issue | Details |
|---|-------|---------|
| 9.1 | **Live preview during editing** | Text changes are rendered via `innerHTML` on each emit. But the edit-blur cycle (lines 1295–1304) is the only save point — there's no incremental update during typing. | Fine for now since `contentEditable` renders natively, but the `text` and `richText` fields are only updated on blur. |
| 9.2 | **Undo for text operations** | Undo snapshots the entire pages array. When editing a text box, Ctrl+Z inside the box will undo the last text edit (native contentEditable undo), but then the next Ctrl+Z will trigger the app-level undo (which may undo element moves/creations, not text). | Confusing undo behavior. The app undo stack and the text editing undo stack are separate and can conflict. |
| 9.3 | **Spellcheck** | No `spellcheck="true"` attribute on editable text elements | No browser spellcheck underlines |
| 9.4 | **Text editing keyboard shortcuts** | Only Ctrl+B/I/U work (via `execCommand`). Ctrl+Shift+L/C/E/R for alignment are not bound. Ctrl+Z/X/C/V for undo/cut/copy/paste work natively in contentEditable. | Partial shortcut coverage |
| 9.5 | **Right-click context menu** | Prevented on the canvas (line 2590: `e.preventDefault()`) | No native cut/copy/paste/select-all via context menu |
| 9.6 | **Drag-and-drop text** | Not implemented. Drag on canvas is for element movement only. | Cannot move selected text within or between text boxes |
| 9.7 | **Thumbnail rendering ignores rich text** | `drawThumb()` (lines 1380–1397) uses `ctx.fillText(el.text, ...)` — plain text, with only `fontWeight === '700'` for bold. Rich text formatting is lost in thumbnails. | Thumbnails are misleading for text-heavy pages |

### Required Changes

- Add `spellcheck="true"` to the `contentEditable` div in `domEl()` (line 1266)
- Add keyboard shortcut bindings for alignment inside text editing (Ctrl+Shift+L/C/E/R)
- Re-enable context menu on canvas (at least for text elements when editing)
- Improve thumbnail rendering to use `richText` or at minimum render all text formatting properties
- Consider adding a debounced text update during editing (not just on blur) so the properties panel stays in sync

---

## 10. Architecture Cross-Cutting Issues

### 10.1 Single-File Scaling

The project is at 2625 lines and growing. The objectives in this document will likely double or triple the codebase. The single-file architecture was chosen for zero-setup portability, but at this scale, maintenance becomes difficult.

**Recommendation:** Keep the single-file product for distribution, but develop the code in separate modules with a build step (even a simple concatenation script). This is a significant architectural decision — defer unless the codebase becomes unmanageable.

### 10.2 Data Model Versioning

There is already a migration path for legacy data (lines 1180–1186). As new fields are added for line height, letter spacing, padding, etc., the migration function must be extended to add defaults for each new field.

**Recommendation:** Add a central `migrateElement(el)` function that checks a `version` field and applies migrations sequentially. Increment the version with each batch of new fields.

### 10.3 Undo/Redo Granularity

The current undo system snapshots the entire `state.pages` array (line 1006). This is O(n) per snapshot and loses cursor position, selection state, and scroll position. For text editing, this means every text change (on blur) snapshots the entire page state.

**Recommendation:** Continue with whole-page snapshots for now (it's simple and works at this scale). Add a `snapshotTextOnly` flag or use a debounced text-snapshot approach to avoid excessive undo stack entries during typing.

### 10.4 execCommand Replacement Strategy

`document.execCommand()` is deprecated across all major browsers. While it still works, it may be removed. A replacement strategy is needed.

**Recommendation:** Build a thin wrapper around the `document.execCommand` API that:
- Uses `execCommand` when available
- Falls back to manual `Range`/`Selection` manipulation for browsers that remove it
- Unifies the editing and non-editing code paths

### 10.5 CSS Class vs Inline Style for Text

The `.el-text` CSS class (line 397) hardcodes `padding: var(--space-2)` and `overflow: hidden`. When per-element padding and overflow are added, these must be moved to inline styles.

**Recommendation:** Remove hardcoded padding and overflow from `.el-text` CSS. Apply them as inline styles in `domEl()` based on the element's properties.

### 10.6 Hit Testing with Rotation

If rotation is added to elements, the current AABB hit test (`hitTest()`, line 1880) will fail for rotated elements. A point-in-transformed-rect test is needed.

**Recommendation:** Use `document.elementFromPoint()` instead of manual hit testing, or implement point-in-transformed-AABB calculation.

---

## 11. Implementation Roadmap

### ✅ Phase 1 — Foundation (COMPLETE — 2025-07-16)

| # | Item | Done? |
|---|------|-------|
| 1 | Fix the export bug: use `el.richText` instead of `el.text` | ✅ |
| 2 | Fix the properties textarea: use `innerHTML`, preserve rich text | ✅ |
| 3 | Add `lineHeight`, `letterSpacing` to data model, CSS, properties panel | ✅ |
| 4 | Add `padding` to data model, remove hardcoded CSS padding | ✅ |
| 5 | Expand font weight options (100–900) | ✅ |
| 6 | Add justify alignment option | ✅ |
| 7 | Add strikethrough, superscript, subscript, clear-format buttons | ✅ |
| 8 | Add `spellcheck=true` to editable text elements | ✅ |

### ✅ Phase 2 — Paragraph & Text Box Controls (COMPLETE — 2025-07-16)

| # | Item | Done? |
|---|------|-------|
| 1 | Vertical alignment (top/middle/bottom) | ✅ |
| 2 | Auto-resize toggle with min/max height | ✅ |
| 3 | Overflow mode selector (clip / visible / scroll) | ✅ |
| 4 | Paragraph spacing and text indent to properties | ✅ |
| 5 | Bulleted and numbered list toolbar buttons | ✅ |
| 6 | Column count + gap controls | ✅ |
| 7 | Text direction (LTR/RTL) control | ✅ |

### ✅ Phase 3 — Styles System (COMPLETE — 2025-07-16)

1. ~~Implement `state.styles` with paragraph and character style definitions~~ ✅ Paragraph styles implemented. Character styles deferred — inline formatting handled by the contentEditable toolbar.
2. ~~Add style picker UI to the properties panel~~ ✅ Dropdown with (None) + all styles, plus New/Delete/Update buttons
3. ~~Implement style save/delete/update~~ ✅ `createParagraphStyle()` from element, `deleteParagraphStyle()` removes style + unlinks elements, `updateParagraphStyle()` propagates changes to all referencing elements
4. ~~Implement style inheritance (propagate changes to all referencing elements)~~ ✅ `propagateStyle()` overwrites all style props on all elements with the matching `paragraphStyleId`

### ✅ Phase 4 — Advanced Features (COMPLETE — 2025-07-16)

| # | Item | Done? |
|---|------|-------|
| 1 | Text rotation (per-element, 0–360°, CSS transform, export) | ✅ |
| 2 | Inline images (insert into contentEditable via file picker) | ✅ |
| 3 | Hyperlinks (insert/remove links via toolbar) | ✅ |
| 4 | Special character picker (60+ chars in popup grid) | ✅ |
| 5 | Find & replace (dialog, prev/next, replace, replace all, Ctrl+F) | ✅ |
| 6 | Drop caps (first-letter styling, toggle checkbox, export) | ✅ |
| 7 | Hyphenation toggle (CSS hyphens: auto, export support) | ✅ |

*Text threading (linked text boxes) deferred — requires full overflow reflow engine.*

### ✅ Phase 5 — Polish & Export (COMPLETE — 2025-07-16)

| # | Item | Done |
|---|------|------|
| 1 | Font embedding in export | ✅ Google Fonts link auto-generated from fonts used |
| 2 | PDF export | ✅ Opens HTML in new window → `window.print()` |
| 3 | Page size / orientation controls | ✅ Per-page pw/ph, Document panel section with presets (Letter, A4) |
| 4 | Thumbnail rich text rendering | ✅ Strips HTML tags from richText, scales to page dimensions |
| 5 | Per-keystroke undo snapshots | ✅ Debounced (800ms) snapshot on text editor input + on blur |
| 6 | Context menu for text | ✅ Re-enabled on `.el-text` canvas elements for native cut/copy/paste |
| 7 | Alignment keyboard shortcuts | ✅ Ctrl+Shift+L/C/R for left/center/right when text editor focused |

---

## Summary

| Area | Status | Critical Gaps | Effort |
|------|--------|---------------|--------|
| Rich text editing | ⚠️ Improved | `execCommand` deprecation, per-selection formatting polish, textarea fixed | Low |
| Paragraph & line controls | ✅ Complete | Phase 1&2 complete | Low |
| Text box management | ✅ Complete | Auto-resize, overflow, padding, columns, direction done; threading deferred | Medium |
| Typography | ⚠️ Improved | Full weights, rotation, drop caps, hyphenation done; variants pending | Low |
| Inline content | ✅ Added | Inline images, hyperlinks, special chars, find/replace all implemented | Low |
| Styles | ✅ Complete | Paragraph styles: create, update, delete, apply, propagate | Low |
| Properties panel | ✅ Complete | All sections present; multi-select polish pending | Low |
| Export | ✅ Complete | Rich text, rotation, hyphens, font embedding, PDF, page size | Low |
| Quality of life | ✅ Complete | Spellcheck, find/replace, undo snapshots, context menu, alignment shortcuts | Low |

**Progress as of 2025-07-16:** Phase 1 ✅, Phase 2 ✅, Phase 3 ✅, Phase 4 ✅, Phase 5 ✅. All phases complete.

**Status:** Production-ready for single-page layout creation with rich text.

**Total remaining effort:** ~5–8 focused development sessions

**The single highest-impact fix** is the export bug (8.1) — it takes 1 line change and makes the product actually useful for producing documents. **The highest-impact feature** is auto-resize + overflow (3.1–3.3), which removes the biggest friction point for users placing text.