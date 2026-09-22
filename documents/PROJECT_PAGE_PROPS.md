# Project & Page Properties — Technical Plan

> **Status:** DRAFT — reviewed once, still no implementation. See §0 for review notes and corrections.
> **Scope:** Give Layout a real document model: project-level and page-level properties that mirror what a print/layout application (InDesign, Scribus) calls *document setup* and *page setup* — physical page size in px/in/mm/pt, orientation, margins, columns/gutters, bleed/slug, grid/baseline grid, and output resolution (DPI).
> **Reference code:** all line numbers re-verified against `index.html` at **4696 lines**, current working tree (uncommitted — `git diff` shows large additions since the last commit: inline text editing, multi-tab read-only mode, sidebar collapse). Verify them again before editing; they will drift.

---

## 0. Technical Review — Corrections & Additions

This section records a follow-up technical review of the plan below. The working tree changed materially since §1–§19 were drafted (an inline-text-editing system and a multi-tab **read-only mode** were added), so every code anchor was re-checked and seven reviewable gaps (R1–R7) were found and closed. Nothing in §1–§19 was factually wrong about the *data model*; the gaps were in *integration with systems added after the draft* and in the load/persist plumbing.

### 0.1 Corrections applied

| # | Issue | Fix |
|---|---|---|
| R1 | §8/§13 cited `onPointerMove` drag/resize snap calls at "~2855/2874" — stale. The function is now at **`index.html:3924`**, with the actual `snap()` call sites at **3970-3974** (line drag), **4026-4032** (multi-select drag), and **4074-4077** (resize). | Line refs corrected throughout §8, §12, §13. |
| R2 | **The plan never accounted for `state.readOnly`.** A BroadcastChannel-based multi-tab leader/follower system (`setupBroadcastChannel()` 1733, `setReadOnly()` 1789) was added after the draft. Some handlers already guard this flag (`btn-save` 3033) but others do not (see R3). **The plan's new Document/Page panel bindings did not include this guard** — without it, a read-only follower tab could mutate `state.pages`/`state.project` locally, diverge from the leader, and get silently overwritten (or worse, race) on the next `reloadFromStorage()`. | Added §12.1 (new) with the required guard pattern; added to Phase 1 checklist and Pre-Flight Checklist. |
| R3 | **Pre-existing gaps, adjacent to this work:** `bindProps()`, `btn-new` and `btn-import` all had **no `state.readOnly` guard**, even though `btn-save` and the canvas input paths (`onPointerDown`, `onKey`) did. A follower tab could therefore edit element properties, import a file, or start a new project locally. `save()` no-ops in read-only mode so this is not permanent data loss, but it visibly forks the follower's canvas until the next broadcast/reload. Not *caused* by this plan, but sits in the exact files Phase 1/2 touch. | Flagged as a pre-existing gap to fix alongside Phase 1/2 (cheap, same guard, same files). **Fully closed:** Phase 1 guarded `btn-new`/`btn-import` and the map/text/inline-format listeners; Phase 2 guarded the remaining `bindProps()` listeners. See §12.1, §21.4, §22.4. **Correction note:** an earlier pass of this addendum claimed `btn-new`/`btn-import` were already guarded and cited the guard line for `btn-save` — that was wrong; corrected here and in §12.1/§15. |
| R4 | §7.2's `renderGuides()` used `z-index:999` for `#guides-layer` without checking it against the existing stacking context. Existing z-indices: `.grid-layer` 0, `.page-content` 1, elements `1..n` (`domEl()` 2022: `d.style.zIndex = i + 1`), `.resize-handle` 10, `.selection-ghost` 50, `#inline-toolbar` 10000 (fixed, off-page). A guides layer at 999 sits **above resize handles (10) and the selection ghost (50)**, which would visually bury the active selection's handles under a margin/column guide line at the page edge — a real, visible bug. | **As implemented, the guides layer uses `z-index:0`.** `.page-content` carries `z-index:1`, which makes it a *stacking context*: a sibling layer at `2` paints **above** `.page-content` and therefore above every element, handle, and ghost inside it, contradicting the stated intent. At `z-index:0` the guides paint after `.grid-layer` but before `.page-content`, so they sit behind all content while staying visible through empty space. See corrected §7.2 and §21. |
| R5 | §9.1's rewrite of `buildProjectData()` didn't account for the existing `save = function() {...}` **re-assignment wrapper** at `index.html:1676-1682`, which wraps the original `save()` to add auto-save-to-file + `broadcastStateChange()`. Unifying `save()`/`buildProjectData()` must preserve this wrapping order (the wrapper captures `_baseSave = save` *before* redefining `save`, so any edit to `save()`'s body must happen before that capture, or the wrapper must be re-pointed at the new body). | Added explicit instruction in §9.1. |
| R6 | §13 Phase 0 didn't mention that mutating `state.project` must call `emit()` (`index.html:1180`, subscriber list populated by `on(refresh)` at **2980**) or nothing re-renders — `saveSnapshot()` (1246) only manages the undo stack and `state.dirty`, it does not trigger a render. This is easy to miss because most existing mutators call both `saveSnapshot()` **and** `emit()` back-to-back (e.g. 4119, 4123, 4245). | Added explicit note to §12 and Phase 1 steps. |
| R7 | **Project-level edits would never autosave.** `state.dirty` is set in exactly one place — `saveSnapshot()` (`index.html:1250`) — and `autosave()` returns early when `!state.dirty` (`index.html:1557`). Project fields (units, DPI, grid, guides, meta) are *not* part of `state.pages`, so `saveSnapshot()` is the wrong tool for them (it would snapshot unchanged pages and pollute undo), and plain `emit()` would leave `state.dirty` false so autosave silently skips them — the change would only survive via the `beforeunload`/`pagehide` hooks (4573/4574). | Added a `markProjectDirty()` helper and a page-vs-project call table in §8; updated Phase 1 step 2, §14, Pre-Flight item 10, Tests 24-25, and Risks. |

### 0.2 Confirmed correct (spot-checked, no change needed)

- G1–G11 in §2.2 are all still accurate; `prop-doc-w`/`prop-doc-h`/`prop-doc-ori` (966-969, 2262-2263) still have **zero** `addEventListener` calls anywhere in the file — confirmed again by grep.
- `PW/PH/GRID` constants (1052), `state` (1153-1175, now also containing `readOnly`, `autoSaveFileHandle`, `channel`, `tabId` — none of which conflict with the new `project` field), `mkPage()` (1254), `save()`/`load()` (1525/1537), `buildProjectData()` (1571), `exportHTML()`/`exportPDF()` (1841/1919), `drawThumb()` (2117), `updateProps()` (2239), `bindProps()` (2402), `refresh()` (2938), `updatePageDimensions()` (2955), `renderRulers()` (3154), `createDemo()` (4356), `init()` (4559) — all line numbers unchanged from the original draft and still resolve to the correct code.
- The two-persistence-shape problem (G11) is unchanged and still needs Phase 0 consolidation.
- Unit/DPI model (§3, §4) needs no changes — it doesn't interact with anything added since the draft.

> **Post-implementation note:** all four phases are now implemented (§20–§24). The line numbers quoted in this review section predate implementation; use the re-verified §19 index.

---

## 1. Goal

Today a "document" in Layout is only an array of pages, each carrying an arbitrary `pw`/`ph` pair of pixels. There is no concept of a project, no physical units, no margins, no output resolution, and no preservation of intent (a 210 mm page and a 793.7 px page are the same thing to the parser).

This plan adds a **two-tier property model**:

| Tier | Examples | Lifetime |
|------|----------|----------|
| **Project** (document) | display units, DPI, default page size, default margins/columns/bleed, grid spacing, metadata | one per file |
| **Page** | size + orientation, margins, columns/gutter, bleed/slug, background, guides toggle | per page, overrides project defaults |

The guiding rules:

1. **Pixels are the internal unit; physical units are a display/IO concern.** No geometry changes representation in storage. (See §3.)
2. **DPI never changes layout geometry.** It only affects raster output and image-resolution checks. (See §4.)
3. **Nothing that currently works may shift.** Existing 850×1100 documents must open byte-for-byte identical in position and size. (See §9.)
4. **Guides are non-printing.** Margins, columns, and bleed are overlays, never part of export. (See §7.)

---

## 2. Current State (what exists / what's missing)

### 2.1 What exists

| Concern | Location | Notes |
|---|---|---|
| Page constants `PW=850, PH=1100` | `index.html:1052` | Module-level `var`; the literal default for every page |
| `--lp-pw` / `--lp-ph` CSS vars | `index.html:59-60` | Global; consumed by `.canvas-page`, `.ruler-h`, `.ruler-v` |
| Global `GRID = 10` | `index.html:1052` | Used by `snap()` (1192), `renderGrid()` (1225), `renderRulers()` (3154) — **hardcoded, not per-project** |
| `state` object | `index.html:1153` | Has `pages`, `zoom`, `pan`, tool, selection, undo. **No `project`/`doc` field.** |
| `mkPage(idx)` | `index.html:1254` | Only `{ id, name, elements, pw, ph }` |
| Page dimension application | `updatePageDimensions()` `index.html:2955` | Writes `--lp-pw`/`--lp-ph` on `document.documentElement`; sets `#page-content.minHeight` |
| Document panel markup | `index.html:966-969` | `#prop-doc-section` with `prop-doc-w`, `prop-doc-h`, `prop-doc-ori` (Letter/A4 presets) |
| Document panel population | `updateProps()` `index.html:2239-2264` | Shows the section only when nothing is selected; fills `w`/`h` from `pg.pw`/`pg.ph` |
| Rulers | `renderRulers()` `index.html:3154` | Draws tick marks every `GRID`, numbers every 100px |
| Grid overlay | `renderGrid()` `index.html:1225` | SVG with `GRID` spacing |
| Snapping | `snap(v)` `index.html:1192` | Always `Math.round(v / GRID) * GRID` |
| Persistence | `save()` 1525, `load()` 1537, `buildProjectData()` 1571 | Two competing shapes, both `v: 1`, neither has project data |
| Export | `exportHTML()` 1841, `exportPDF()` 1919 | Pixel sizes only; no `@page`, no physical units, no bleed/crop marks |
| Thumbnails | `drawThumb()` 2117 | Scales `pg.pw`/`pg.ph` into a fixed canvas — already page-aware |

### 2.2 What is missing or broken

| # | Gap | Evidence |
|---|---|---|
| G1 | **No project object** | `state` (1153) has no `project`; `save()` (1530) writes only `{v, ts, pages, idx}` |
| G2 | **Document size fields are not wired up** | Grep for `prop-doc-w`/`prop-doc-h`/`prop-doc-ori` returns only the markup (966) and the *write* in `updateProps()` (2262-2263). **There are no `addEventListener` calls for them anywhere.** The "Document" panel is currently display-only. (The archive's TECH_REVIEW claims this shipped; the listeners are absent in this tree.) |
| G3 | **No units** | Every input is implicitly px; `parseInt(v)` in `bindProps()` (2420+) |
| G4 | **No margins** | Nowhere in the model or UI |
| G5 | **No columns/gutter for the page** | `columnCount`/`columnGap` exist only per text element (`mkEl` 1305-1306), not as page guides |
| G6 | **No bleed/slug** | Absent |
| G7 | **No DPI** | Absent from model and export |
| G8 | **No baseline grid** | `renderGrid()` draws one square grid only |
| G9 | **No snapping to anything but the square grid** | `snap()` (1192) |
| G10 | **CSS vars are global, not scoped** | `updatePageDimensions()` writes `:root`; acceptable for one page at a time, but couples rulers/canvas to a single current page |
| G11 | **Two persistence shapes** | `save()` (1530) vs `buildProjectData()` (1571) — adding project data requires touching both plus all load paths |

### 2.3 Migration reality check

The current default page **850×1100 px is not a real paper size** (8.854×11.458 in). Do **not** silently snap existing documents to Letter/A4. Any migration must preserve `pw`/`ph` exactly and mark the page as `custom`.

---

## 3. Unit Model (the core design decision)

### 3.1 Canonical unit: the CSS reference pixel

CSS defines `1in = 96px` exactly. Every other unit is a fixed ratio of that:

```
1 in = 96 px
1 pt = 96/72   = 1.333333… px       (1/72 in)
1 pc = 12 pt   = 16 px
1 mm = 96/25.4 = 3.7795275591 px
1 cm = 96/2.54 = 37.7952755906 px
```

This means **a page expressed in px is already a physical size**. An 816×1056 px page *is* 8.5×11 in. There is no DPI term in the conversion.

### 3.2 Storage strategy: single source of truth

**Store all geometry in canonical px. Convert only at the UI/IO boundary.**

- `page.pw` / `page.ph`, `margins.*`, `bleed`, `slug`, element `x/y/w/h`, `padding`, `strokeWidth` — all remain px in `state.pages`.
- The project's `units` field controls how values are *displayed and parsed* in the properties panel, rulers, and export metadata.
- Display conversion rounds to a per-unit precision (see §3.4) but the stored value is never overwritten by a rounded value. A user typing `210 mm` produces `pw = 793.70` px; reading it back shows `210.0 mm`.

**Why not store physical units directly?** It is tempting to store `{ size: { w: 210, h: 297, unit: 'mm' } }` and derive `pw`/`ph`. That requires rewriting 15+ read sites (`clampPan` 1077, `renderGrid` 1229, `align` 1456, `exportHTML` 1863, `drawThumb` 2119, `updatePageDimensions` 2957, `renderRulers` 3161, …). Keeping px authoritative keeps the diff small and the undo/persistence format stable. The only cost is sub-micron rounding, which is well below print tolerance.

### 3.3 Conversion helpers

Add near the existing constant block (`index.html:1052`):

```js
/* --------------- Units --------------- */
var PX_PER_IN = 96;
var UNITS = {
  px: { label: 'px', perPx: 1 },
  in: { label: 'in', perPx: 1 / PX_PER_IN },
  cm: { label: 'cm', perPx: 2.54 / PX_PER_IN },
  mm: { label: 'mm', perPx: 25.4 / PX_PER_IN },
  pt: { label: 'pt', perPx: 72 / PX_PER_IN },
  pc: { label: 'pc', perPx: (72 / PX_PER_IN) / 12 }
};
var UNIT_PRECISION = { px: 0, in: 3, cm: 2, mm: 1, pt: 1, pc: 2 };

/* units-per-px -> px */
function toPx(value, unit) {
  var u = UNITS[unit] || UNITS.px;
  return value / u.perPx;
}
/* px -> units-per-px */
function fromPx(px, unit) {
  var u = UNITS[unit] || UNITS.px;
  return px * u.perPx;
}
function fmtUnit(px, unit) {
  var u = unit || (state.project && state.project.units) || 'px';
  var v = fromPx(px, u);
  var p = UNIT_PRECISION[u] || 0;
  return (p ? v.toFixed(p) : String(Math.round(v))) + ' ' + u;
}
function parseUnit(str, unit) {
  var n = parseFloat(String(str).replace(/[^0-9.\-]/g, ''));
  if (isNaN(n)) return 0;
  return toPx(n, unit || (state.project && state.project.units) || 'px');
}
```

> **Precision note:** `UNIT_PRECISION.px = 0` is intentional because existing integer element coordinates must not suddenly display as `100.000`. Page/margin fields may want 2 px decimals internally; display rounds them.

### 3.4 Rounding policy (avoid drift)

- Never convert px → unit → px on an internal code path. Conversion is one-way at the boundary.
- UI inputs display `fmtUnit` and write `parseUnit`.
- When a preset is chosen, compute px once from the canonical definition and store it. Do not round through the display unit.
- A `Preset` match test must use tolerance: `Math.abs(pg.pw - preset.w) < 0.5`.

---

## 4. DPI Model (disambiguating the common confusion)

DPI is **not** part of unit conversion and **must not** change `pw`/`ph`. It has exactly three legitimate jobs:

| Job | Meaning | Where used |
|---|---|---|
| **Raster export target** | Scale factor when rendering a page to a bitmap: `scale = dpi / 96`. An 816×1056 px page at 300 dpi → 2550×3300 px image. | New PNG/JPEG export |
| **Placed-image resolution check** | Effective PPI of an image = `imageNaturalPx / (displayedPhysicalInches)`. Warn when below the document `dpi`. | Image properties panel / preflight |
| **Document metadata** | Recorded intent, surfaced in export and filenames. | Project panel, export |

What DPI does **not** do:
- It does not scale the canvas or elements while editing.
- It does not affect the HTML export, which is resolution-independent (the browser lays it out natively and the print device rasterizes).
- It does not affect the PDF export: jsPDF embeds a raster rendered at a fixed `scale: 2`, independent of `project.dpi` (only the PNG/JPEG buttons use `dpi`).
- It is not needed to convert mm to px (the PDF page box uses physical size directly).

Add `state.project.dpi` (default `300`). Display it in the project panel with a short hint: *"Used for raster export and image resolution checks, not for page size."*

---

## 5. Proposed Data Model

### 5.1 Project object

```js
var PROJECT_DEFAULTS = {
  schema: 2,
  name: 'Untitled',
  units: 'px',                 // display unit: px | in | cm | mm | pt | pc
  dpi: 300,                    // raster target / image PPI threshold
  page: {                      // project-level page defaults
    preset: 'custom',          // label only; never the source of truth
    pw: 850,                   // canonical px
    ph: 1100,
    orientation: 'portrait',   // portrait | landscape
    margins: { top: 48, right: 48, bottom: 48, left: 48 }, // px; 0.5 in
    columns: 1,
    gutter: 24,                // px
    bleed: 0,                  // px (all sides)
    slug: 0                    // px (all sides)
  },
  grid: {
    spacing: 10,               // px, replaces the GRID constant
    subdivisions: 5,           // minor lines per major
    baseline: 0,               // px; 0 = off
    showBaseline: false
  },
  guides: {
    showMargins: true,
    showColumns: true,
    showBleed: false,
    snapToGuides: true
  },
  export: {                    // Phase 4
    includeBleed: false        // expand exported page by page.bleed + draw crop marks
  },
  meta: {
    title: '',
    author: '',
    description: '',
    created: null,             // ISO string
    modified: null
  }
};

function mkProject() { return JSON.parse(JSON.stringify(PROJECT_DEFAULTS)); }
```

### 5.2 Page object (extended)

```js
function mkPage(idx) {
  var pd = state.project ? state.project.page : PROJECT_DEFAULTS.page;
  return {
    id: 'p' + Date.now().toString(36) + '_' + idx,
    name: 'Page ' + (idx + 1),
    elements: [],
    /* Size (canonical px) — overrides project defaults */
    pw: pd.pw,
    ph: pd.ph,
    orientation: pd.orientation,
    preset: pd.preset,          // label only
    /* Guides (non-printing) */
    margins: JSON.parse(JSON.stringify(pd.margins)),
    columns: pd.columns,
    gutter: pd.gutter,
    bleed: pd.bleed,
    slug: pd.slug,
    background: '#FFFFFF',
    showGuides: true
  };
}
```

`pw`/`ph` keep their current names deliberately: the whole codebase reads them (see §5.3) and renaming causes needless churn. Add a comment that they are canonical CSS px.

### 5.3 Helpers

```js
function proj() { return state.project; }
function units() { return state.project ? state.project.units : 'px'; }

/* Content box after margins (px). */
function contentRect(p) {
  var m = (p && p.margins) || { top: 0, right: 0, bottom: 0, left: 0 };
  var W = (p && p.pw) || PW, H = (p && p.ph) || PH;
  return {
    x: m.left,
    y: m.top,
    w: Math.max(0, W - m.left - m.right),
    h: Math.max(0, H - m.top - m.bottom)
  };
}

/* Column guide rectangles within the content box. */
function columnRects(p) {
  var c = contentRect(p);
  var n = Math.max(1, p.columns || 1);
  var g = p.gutter || 0;
  var colW = (c.w - g * (n - 1)) / n;
  var out = [];
  for (var i = 0; i < n; i++) out.push({ x: c.x + i * (colW + g), w: colW });
  return out;
}
```

`align()` (`index.html:1453`) still uses `pw`/`ph` for centering. Decide per feature: page-align on **trim** (current behavior) vs **content box**. Recommendation: default to trim, add a "align to margins" option later (out of scope for v1).

---

## 6. Page Presets

All values canonical px at 96 px/in. ISO values are computed from mm and rounded to 2 dp.

| Preset | Physical | px (portrait) |
|---|---|---|
| Letter | 8.5 × 11 in | 816 × 1056 |
| Legal | 8.5 × 14 in | 816 × 1344 |
| Tabloid | 11 × 17 in | 1056 × 1632 |
| A3 | 297 × 420 mm | 1122.52 × 1587.40 |
| A4 | 210 × 297 mm | 793.70 × 1122.52 |
| A5 | 148 × 210 mm | 559.37 × 793.70 |
| A6 | 105 × 148 mm | 396.85 × 559.37 |
| B5 (ISO) | 176 × 250 mm | 665.20 × 944.88 |
| DL | 110 × 220 mm | 415.75 × 831.50 |
| Screen 1080p | 1920 × 1080 px | 1920 × 1080 |
| Screen 4K | 3840 × 2160 px | 3840 × 2160 |
| Custom | user | user |

```js
var PAGE_PRESETS = {
  'letter':  { label: 'Letter',  w: 816,  h: 1056 },
  'legal':   { label: 'Legal',   w: 816,  h: 1344 },
  'tabloid': { label: 'Tabloid', w: 1056, h: 1632 },
  'a3':      { label: 'A3',      w: 1122.52, h: 1587.40 },
  'a4':      { label: 'A4',      w: 793.70,  h: 1122.52 },
  'a5':      { label: 'A5',      w: 559.37,  h: 793.70 },
  'a6':      { label: 'A6',      w: 396.85,  h: 559.37 },
  'b5':      { label: 'B5',      w: 665.20,  h: 944.88 },
  'dl':      { label: 'DL',      w: 415.75,  h: 831.50 },
  'hd1080':  { label: '1080p',   w: 1920,    h: 1080 },
  'uhd4k':   { label: '4K',      w: 3840,    h: 2160 }
};
```

**Orientation** is applied when a preset is chosen: `landscape` swaps w/h. Store `orientation` as a label; the authoritative size is still `pw`/`ph`.

**Applying a preset to all pages:** provide "Apply page size to all pages" and "Apply margins to all pages" buttons; otherwise each page is independent.

---

## 7. Margins, Columns, Bleed, Slug, Guides

### 7.1 Semantics

- **Trim** = the page (`pw` × `ph`). This is `#canvas-page`.
- **Content box** = trim minus margins. Text/art should live here.
- **Columns** = equal vertical bands inside the content box separated by `gutter`.
- **Bleed** = art extendable area *outside* trim. Guide rectangle at `-bleed` … `+bleed`. Non-printing element, but affects print export (§10).
- **Slug** = outer margin beyond bleed for marks/info. Guide only in v1.

### 7.2 Rendering

Add one layer to the canvas DOM (`index.html:858`):

```html
<div class="canvas-page" id="canvas-page">
  <div class="grid-layer" id="grid-layer"></div>
  <div class="ruler-h" id="ruler-h"></div>
  <div class="ruler-v" id="ruler-v"></div>
  <div class="page-content" id="page-content"></div>
  <div class="guides-layer" id="guides-layer" aria-hidden="true"></div>  <!-- NEW -->
</div>
```

CSS:

```css
.guides-layer{position:absolute;inset:0;z-index:0;pointer-events:none;overflow:visible}
.guide-margin{border:1px dashed var(--guide-margin,#E0457B);position:absolute;box-sizing:border-box}
.guide-margin-band{position:absolute;background:#E0457B;opacity:.06} /* shaded margin area */
.guide-column{position:absolute;top:0;bottom:0;width:1px;border-left:1px dashed var(--guide-column,#38BDF8)}
.guide-bleed{border:1px solid var(--guide-bleed,#16A34A);position:absolute;box-sizing:border-box}
```

> **Corrected stacking order (see §0.1 R4).** The existing stacking context is: `.grid-layer` `z-index:0` → `.page-content` `z-index:1` → individual elements `z-index: i+1` (set per-element in `domEl()`, `index.html:2249`) → `.resize-handle` `z-index:10` → `.selection-ghost` `z-index:50` → `#inline-toolbar` `z-index:10000` (fixed, off-canvas). Putting guides at `999` would render them **above resize handles and the selection ghost**, burying the active selection's handles under a margin line whenever it sits near a guide — a real visual bug, not just a theoretical one, because default margins (0.5in/48px) are well within typical element positions.
>
> **Use `z-index:0`** (as implemented). The subtlety that an earlier pass missed: `.page-content` sets `z-index:1`, which makes it a *stacking context* — so a **sibling** layer at `2` paints *above* `.page-content` and therefore above every element, handle, and selection ghost inside it. At `z-index:0` the guides paint after `.grid-layer` but before `.page-content`: behind all content, yet visible through empty space. That is the intent — "visible through empty space, never obscuring the selection UI." If a future requirement needs guides drawn *over* opaque objects, revisit this value deliberately rather than defaulting to "on top of everything."

```js
function renderGuides(preview) {          /* `preview` = optional page copy for live preview */
  var layer = $('guides-layer');
  if (!layer) return;
  layer.innerHTML = '';
  var p = preview || cpage();
  if (!p || !state.project || p.showGuides === false) return;
  var g = state.project.guides;
  var W = p.pw || PW, H = p.ph || PH;
  var html = '';

  if (g.showBleed && p.bleed > 0) {
    var b = p.bleed;
    html += '<div class="guide-bleed" style="left:' + (-b) + 'px;top:' + (-b) + 'px;width:' +
      (W + 2 * b) + 'px;height:' + (H + 2 * b) + 'px"></div>';
  }
  if (g.showMargins) {
    var c = contentRect(p);
    /* Shaded bands over the four margin areas (as implemented — see §21.2). */
    if (c.y > 0) html += band(0, 0, W, c.y);
    if (c.y + c.h < H) html += band(0, c.y + c.h, W, H - c.y - c.h);
    if (c.x > 0) html += band(0, c.y, c.x, c.h);
    if (c.x + c.w < W) html += band(c.x + c.w, c.y, W - c.x - c.w, c.h);
    if (c.x > 0 || c.y > 0 || c.w < W || c.h < H)
      html += '<div class="guide-margin" style="left:' + c.x + 'px;top:' + c.y + 'px;width:' +
        c.w + 'px;height:' + c.h + 'px"></div>';
  }
  if (g.showColumns && p.columns > 1) {
    columnRects(p).forEach(function(col, i) {
      if (i === 0) return; /* content box left edge already drawn */
      html += '<div class="guide-column" style="left:' + col.x + 'px"></div>';
    });
  }
  layer.innerHTML = html;
}
```

**Margin indicator (as implemented).** The margin setting is shown two ways: a dashed `#E0457B` inset rectangle around the content box, **and** four low-opacity shaded bands filling the margin areas between the page edge and the content box (`opacity:.06`). The bands make the margin zone legible at a glance rather than a single thin line. The rectangle is skipped when all four margins are 0 (it would sit on the page edge).

**Live preview.** `renderGuides()` accepts an optional `preview` page object. `bindDocProps()` binds an `input` listener on the margin/columns/gutter/bleed fields that renders guides from a `JSON.parse(JSON.stringify(cpage()))` copy, so the indicator follows typing **without** mutating `state.pages` (and therefore without corrupting the undo stack). The existing `change` listener still performs the real snapshot-backed commit.

Call `renderGuides()` from `refresh()` (`index.html:2938`) next to `renderGrid()`.

### 7.3 Validation

Clamp margin inputs so `left + right < pw` and `top + bottom < ph`, with a minimum content width (e.g. 20 px). Show a toast and revert the field if invalid. Same for `gutter * (columns-1) < contentWidth`.

---

## 8. Grid & Baseline Grid

- Replace the `GRID` constant's role with `state.project.grid.spacing`, but **keep `GRID` as the fallback** so nothing breaks before the project migrates.
- `renderGrid()` (1225): use `state.project.grid.spacing`; optionally draw major lines every `spacing * subdivisions` and minor lines between.
- `snap()` (1192): use the project spacing. Add a guide-aware variant:

```js
function snapTo(v, targets, tolerance) {
  var best = v, bestD = tolerance == null ? 6 : tolerance;
  (targets || []).forEach(function(t) {
    var d = Math.abs(v - t);
    if (d < bestD) { bestD = d; best = t; }
  });
  return best;
}
/* Candidate guide positions for snapping. Does NOT include 0 or the trim
   edge — those are already handled by the existing grid snap, and adding
   them would make snapping to 0 sticky during normal drags. */
function guideTargets(p, axis) {
  var t = [];
  var c = contentRect(p);
  if (axis === 'x') t.push(c.x, c.x + c.w);
  else t.push(c.y, c.y + c.h);
  columnRects(p).forEach(function(col) { if (axis === 'x') t.push(col.x, col.x + col.w); });
  return t;
}
```

- Baseline grid: `project.grid.baseline` (px). When `> 0`, draw horizontal lines every `baseline` px in `#grid-layer` (distinct color), and optionally snap text `lineHeight` to it (defer snapping text — v1 draws the overlay only).

**Interaction change:** `snap()` is currently grid-only and always on. V1 must not change drag feel. Plan: keep grid snapping as-is; only add guide snapping when `project.guides.snapToGuides` is true *and* a field with a guide is within tolerance, preferring guides over grid. Touch points: `onPointerMove()` (`index.html:3924`), specifically the `snap()` call sites at **3970-3974** (single-element/line-endpoint drag), **4026-4032** (multi-select drag), and **4074-4077** (resize). Do this in a dedicated phase (§13 Phase 3) and test heavily.

> **As built (Phase 3).** `snap(v, axis)` keeps the exact grid formula and only consults guides when `axis` is given *and* `project.guides.snapToGuides` is on. Guides are `contentRect` edges + `columnRects` edges (no `0`/trim edge, per the note above). `GUIDE_SNAP_TOL = 6` px. With `snapToGuides` off, `snap(v,'x') === snap(v)` — byte-identical to the old behaviour, so the feature is a clean rollback flag. Resize passes `axis` **only for the edge that moves** (`w`→x, `n`→y); width/height stay pure-grid, so resizing never drags the opposite edge or the size. See §23.

> **Note on `emit()` / `saveSnapshot()` / `state.dirty`.** Two separate mechanisms are involved, and both are easy to get wrong:
> - `emit()` (`index.html:1180`, dispatches to subscribers registered via `on(refresh)` at **`index.html:2980`**) re-renders. `saveSnapshot()` (`index.html:1246`) does **not** render.
> - `saveSnapshot()` (`index.html:1250`) is the **only** place that sets `state.dirty = true`, and `autosave()` (`index.html:1555`) returns early when `!state.dirty`. So persistence via autosave depends entirely on `state.dirty`.
>
> Splitting the two kinds of field changes accordingly:
>
> | Change kind | Examples | Call |
> |---|---|---|
> | **Page-level** (fields live in `state.pages`) | page size, orientation, margins, columns/gutter, bleed, showGuides | `saveSnapshot(); /* mutate */ ; emit();` |
> | **Project-level** (fields live in `state.project`, *not* snapshotted) | units, DPI, grid spacing, guides toggles, meta | `markProjectDirty(); /* mutate */ ; emit();` |
>
> Add a one-line helper so project changes persist without polluting the undo stack:
> ```js
> function markProjectDirty() { state.dirty = true; }
> ```
> Calling `saveSnapshot()` for a **project-only** change is wrong twice over: it pushes a no-op undo entry (undo would appear to do nothing, since the snapshot contains only pages), and it implies undoability the plan explicitly does not provide (§14). Calling `emit()` for either kind is always required, and for project-level changes `emit()` alone is **not** enough — without `state.dirty` the autosave silently skips and the change is only written by the `beforeunload`/`pagehide` hooks (4573/4574), which is fragile.
>
> Existing drag/resize mutators call `saveSnapshot(); emit();` back-to-back (e.g. `index.html:4119`, `4123`, `4245`) — follow that for page-level fields. Note that `bindProps()` (`index.html:2432`) is inconsistent here: it calls neither `saveSnapshot()` nor `emit()` (it mutates via `updEl()` at 1372 and syncs the DOM directly), which is why element-property edits are neither undoable nor autosaved today. Do not copy that pattern.

---

## 9. Persistence & Migration

### 9.1 Unify the two shapes

`save()` (`index.html:1525`) and `buildProjectData()` (`index.html:1571`) write different objects. Consolidate:

```js
function buildProjectData() {
  return {
    v: 2,
    ts: new Date().toISOString(),
    project: state.project,
    pages: state.pages,
    idx: state.currentPageIndex,
    currentPage: cpage() ? cpage().id : null,
    zoom: state.zoom,
    panX: state.panX,
    panY: state.panY
  };
}
```

Make `save()` store `buildProjectData()` (minus the Live/Zoom keys, which stay separate for the existing `lp-zoom`/`lp-panX`/`lp-panY` restore in `init()`, ~`index.html:4629-4638`).

> **Save-wrapper ordering (see §0.1 R5).** `index.html:1676-1682` already re-assigns `save`:
> ```js
> var _baseSave = save;
> save = function() {
>   if (state.readOnly) return;
>   _baseSave();
>   if (state.autoSaveFileHandle) writeToAutoSaveFile();
>   broadcastStateChange();
> };
> ```
> `_baseSave` captures the **original** `save()` function object at the time this line runs (module init time). Any Phase 0 edit to `save()`'s body must be made in the original `function save() {...}` definition (`index.html:1525`) — i.e. edit the source before this wrapper line executes, not by trying to redefine `save` again afterward, which would break the read-only guard and the broadcast. `writeToAutoSaveFile()` and `broadcastStateChange()` both serialize via `buildProjectData()`, so once `buildProjectData()` includes `project`, auto-save-to-file and multi-tab sync pick it up automatically with no further changes.

### 9.2 Migration function (single entry point)

Every load path must funnel through one function:

- `load()` — `index.html:1537`
- `reloadFromStorage()` — `index.html:1806`
- Import handler — `index.html:3040`
- `init()` restore — `index.html:4559+`

```js
/* Tiny ES5 shallow-merge used by the migration below. */
function shallowMerge(base, over) {
  var out = {}, k;
  for (k in base) if (base.hasOwnProperty(k)) out[k] = base[k];
  for (k in over) if (over.hasOwnProperty(k) && over[k] !== undefined) out[k] = over[k];
  return out;
}

function normalizeProject(d) {
  if (!d || !Array.isArray(d.pages)) return null;

  /* --- project --- */
  var proj = d.project && typeof d.project === 'object' ? d.project : {};
  var base = mkProject();
  proj.schema = 2;
  proj.name = proj.name || base.name;
  proj.units = UNITS[proj.units] ? proj.units : 'px';
  proj.dpi = proj.dpi > 0 ? proj.dpi : 300;
  proj.page = shallowMerge(base.page, proj.page || {});
  proj.page.margins = shallowMerge(base.page.margins, (proj.page && proj.page.margins) || {});
  proj.grid = shallowMerge(base.grid, proj.grid || {});
  proj.guides = shallowMerge(base.guides, proj.guides || {});
  proj.meta = shallowMerge(base.meta, proj.meta || {});
  d.project = proj;

  /* --- pages (preserve existing pw/ph EXACTLY; mark custom) --- */
  d.pages.forEach(function(pg) {
    if (pg.pw == null) pg.pw = PW;
    if (pg.ph == null) pg.ph = PH;
    if (!pg.preset) pg.preset = 'custom';
    if (pg.orientation == null) pg.orientation = (pg.pw > pg.ph) ? 'landscape' : 'portrait';
    if (!pg.margins) pg.margins = { top: 0, right: 0, bottom: 0, left: 0 };
    if (pg.columns == null) pg.columns = 1;
    if (pg.gutter == null) pg.gutter = 24;
    if (pg.bleed == null) pg.bleed = 0;
    if (pg.slug == null) pg.slug = 0;
    if (pg.background == null) pg.background = '#FFFFFF';
    if (pg.showGuides == null) pg.showGuides = true;
  });

  d.v = 2;
  return d;
}
```

> **Call-site contract:** `normalizeProject()` returns a **new/updated plain object** — it does not touch `state`. Each of the four call sites must adopt *both* outputs:
> ```js
> var d = normalizeProject(rawData);
> if (!d) return;               /* or fall back to mkProject() for a fresh document */
> state.project = d.project;
> state.pages = d.pages;
> ```
> Forgetting `state.project = d.project` is the single easiest way to make the whole feature silently do nothing on load.

> **Deliberate migration choice:** existing pages get **zero margins** and `custom` preset. This guarantees no visual change to old work. New documents get the project default margins (0.5 in / 48 px). If product wants existing docs to gain visible margin guides, change only the migration default — but that is a visible change and should be called out in release notes.

### 9.3 `createDemo()` and New Project

- `createDemo()` (`index.html:4356`) creates a single default page. Give it `mkProject()` + `mkPage(0)` so the demo has real margins/guides.
- "New project" (`index.html:3013-3016`) currently hardcodes `{ pw: PW, ph: PH }`. Replace with `mkProject()` and `[mkPage(0)]`.
- Import (`3040`) must call `normalizeProject(d)` instead of ad-hoc `pw/ph` migration.

---

## 10. Export & Print

### 10.1 HTML/PDF export (`exportHTML()` 1841)

> **Status: implemented** — see §24.2. `exportHTML()` is now `exportBox()` + `exportElementsHTML()` + `cropMarksHTML()`; the plan below is kept for reference.

- Page div keeps trim `pw`/`ph`.
- Emit a `@page` rule in physical units so the browser print dialog defaults correctly:

```css
@page { size: 8.5in 11in; margin: 0; }
```

Compute using `fromPx(pg.pw,'in')`. For multi-size documents, `@page` is per-document; note the limitation and emit the first/current page size or per-page named pages (`@page page1 { size: … }` + `page: page1`) — document that most browsers only honor the first size.

- Page background: `background: <pg.background>`.
- Do **not** emit guides.
- PDF (`exportPDF()` 1919) should set the new window body to physical size and instruct "Actual size, no scaling".

### 10.2 Bleed / crop marks (optional, later)

> **Status: implemented** (option 2) — see §24.2/§24.5. Gated by `project.export.includeBleed`; the trim/bleed-box limitation stands.

True bleed requires trim/bleed boxes that browsers cannot set. Two honest options:

1. **Guides-only (v1):** bleed is a non-printing guide; no print change.
2. **Expanded canvas:** when "Export with bleed" is on, grow each exported page by `bleed` on all sides, offset elements by `+bleed`, and draw crop marks at the trim corners. The user prints "Actual size". This produces a correct-looking proof but not a print-shop PDF with real boxes. Document the limitation clearly.

### 10.3 Raster export (PNG/JPEG) at DPI

> **Status: implemented** — see §24.3/§24.4. `imagePPI(el)` was implemented without the unused `imgEl` first argument.

New, Phase 4. Zero-dependency constraint means no html2canvas. Recommended approach:

```
renderPage HTML -> <svg><foreignObject> (XHTML) -> data: URL
                -> Image -> drawImage onto canvas at (pw * dpi/96) × (ph * dpi/96)
```

Caveats to document: Safari's `foreignObject` support is weak, external Google Fonts may not load into the SVG image without inlining, and cross-origin images taint the canvas. Offer a fallback: render at `devicePixelRatio` and let the user print to PDF instead. DPI is otherwise still useful for the image-resolution warning:

```js
/* effective PPI of a placed image */
function imagePPI(imgEl, el) {
  var natural = imgEl.naturalWidth;              // image pixels
  var inches  = (el.w || 1) / PX_PER_IN;         // displayed physical width
  return natural / inches;
}
```

Warn when `imagePPI < project.dpi` in the Image properties section.

---

## 11. UI Plan

### 11.1 Properties panel

Rework `#prop-doc-section` (`index.html:966-969`) into two named sections. Keep the existing "show when nothing is selected" behavior from `updateProps()` (2243-2264); do not hide the Page panel when an element is selected — instead, consider adding a persistent **Document** tab. V1 recommendation: keep it in the Properties panel when nothing is selected, but also add a toolbar-level gear that focuses it.

```
Page
  Preset:        [ Custom ▾ ]   (Letter, Legal, Tabloid, A3…; selecting sets size)
  Size:          [ 850 ] × [ 1100 ]  [px ]     (unit selector here or project-level)
  Orientation:   ( ) Portrait  ( ) Landscape  [Swap]
  Margins:       T[48] R[48] B[48] L[48]  [Link]   [in ▾]
  Columns:       [1]  Gutter [24]
  Bleed:         [0]     Slug: [0]
  [ Apply size to all pages ]  [ Apply margins to all pages ]

Project
  Display units: [px ▾]
  DPI:           [300]        (hint: raster export / image checks)
  Grid:          Spacing [10]  Subdivisions [5]
  Baseline grid: [0]  [ ] Show
  Guides:        [x] Margins  [x] Columns  [ ] Bleed  [x] Snap to guides
  Title / Author
```

### 11.2 Unit handling in inputs

Every numeric input in the panel should render with its unit and parse through `parseUnit`. Add a tiny helper so `bindProps()` (2402) can stay table-driven:

```js
function bindUnitInput(id, getPx, setPx, unit, opts) {
  var inp = $(id); if (!inp) return;
  function refreshDisplay() { inp.value = opts.round0 ? Math.round(fromPx(getPx(), unit)) : +fromPx(getPx(), unit).toFixed(UNIT_PRECISION[unit] || 0); }
  inp.addEventListener('input', function() { setPx(parseUnit(this.value, unit)); });
  inp.addEventListener('blur', refreshDisplay);
  refreshDisplay();
  return refreshDisplay;
}
```

Phase 2 applies this to element `x/y/w/h` and spacing fields; Phase 1 uses it for page/margins only.

> **Unit-change refresh (gap to handle).** `unit` is captured when `bindUnitInput()` runs, so after the user changes `state.project.units` every bound field still shows the *old* unit until it is re-bound. Keep a tiny registry — e.g. `var unitBinders = [];` and `unitBinders.push(refreshDisplay);` inside `bindUnitInput()` — and call them all on every unit change so every field re-renders at once. `refresh()` alone is not sufficient: it does not re-run the input bindings.

> **As built (Phase 2).** The standalone `bindUnitInput()` sketch above was *not* used verbatim. Unit parsing was folded into the existing table-driven bindings in `bindProps()` (via `len: true` entries) plus the `setLenInput()` / `parseLenInput()` helpers, because that table already owns multi-select propagation and `syncElementDOM()`. The unit-change registry is also unnecessary in practice: the helpers read `units()` **live** (they never capture it), and every unit change runs `commitProject()` → `emit()` → `refresh()` → `updateProps()`, which re-populates all unit-aware fields. `setLenInput()` skips `document.activeElement` so a live re-render never clobbers the field being typed into. See §22.

### 11.3 Rulers in units (`renderRulers()` 3154)

- Tick spacing adapts: px → 10 px ticks, 100 px labels; `in` → 1/8 in ticks, 1 in labels; `mm` → 1 mm/5 mm ticks, 10 mm labels; `pt`/`pc` similar.
- Origin remains trim top-left (0,0). (A future option could set the ruler origin to the margin, like InDesign; out of scope.)

### 11.4 Accessibility (STYLE_GUIDE)

- All new inputs get `aria-label` and visible `<label>`.
- Unit selectors ≥ 44 px hit target.
- Preset dropdown keyboard operable (native `<select>`).
- Guides are decorative → `aria-hidden="true"` on `#guides-layer`.

---

## 12. Interaction & Integration Notes

| Existing system | Impact |
|---|---|
| `clampPan()` 1073 / `reClampPan()` | Uses `p.pw`/`p.ph` — unaffected. **After any page resize, call `clampPan()` + `applyTransform()`**, because the clamp bounds depend on page size. |
| `updatePageDimensions()` 2955 | Already sets `--lp-pw`/`--lp-ph` (which size `.canvas-page`, `.ruler-h`, `.ruler-v`) and `#page-content.minHeight`. Do **not** add explicit `#canvas-page` width/height — that fights the vars and desyncs the rulers. Only add `clampPan()`+`applyTransform()` after a page-size change. |
| `renderGrid()` 1225 | **Done (Phase 3):** uses `project.grid.spacing` × `subdivisions` for major/minor lines, plus a dashed baseline overlay when `project.grid.showBaseline` and `baseline > 0`. The defaults (10/5) reproduce the old 50 px major lines exactly. |
| `snap()` 1192 | **Done (Phase 3):** `snap(v, axis)` uses `project.grid.spacing`; when `project.guides.snapToGuides` is on and a content/column guide is within 6 px on that axis, the guide wins, else the grid. Omit `axis` for a pure grid snap (width/height). |
| `drawThumb()` 2117 | Already page-size aware; optionally draw a faint margin rectangle. |
| `align()` 1453 | Keep trim-based centering. |
| Auto-resize text (`applyAutoResize()` 2964) | Unaffected. |
| Multi-tab `reloadFromStorage()` 1806 | Must run `normalizeProject()` so a v1 tab's data still loads while another tab runs v2. |
| `beforeunload`/`pagehide` save (init 4559+) | Uses `save()` → now includes project. |

**Do not clamp elements to the content box.** Margins are guides, not constraints (InDesign does not move objects when margins change). Leave objects where they are; off-content objects still export.

### 12.1 Read-only / multi-tab guard (new — see §0.1 R2, R3)

A BroadcastChannel-based leader/follower system was added to the app after this plan was first drafted (`setupBroadcastChannel()` `index.html:1733`, `setReadOnly()` `index.html:1789`). When a second tab opens the same document, the loser becomes `state.readOnly = true` and gets a visible banner (`#readonly-banner`, `index.html:838`) plus a CSS rule that visually disables the toolbar (`body.read-only-mode .toolbar …`, `index.html:738-744`).

**Which mutating entry points are actually guarded today** (verified by `grep -n "state.readOnly" index.html`): `autosave()` (1556), the `save()` wrapper (1678), the BroadcastChannel handlers (1765-1790), `broadcastStateChange()`/`releaseChannel()` (1826/1832), the `storage` listener (1822), the `beforeunload`/`pagehide` save hooks (4573/4574), `btn-save` (**3033**), `onPointerDown()` (**3796**), `onKey()` (4158), and the canvas context-menu handler (4606).

**Gaps — all now closed.** The gaps were `bindProps()` input listeners, `btn-new`, and `btn-import`. **Phase 1** guarded `btn-new`, `btn-import`, the map/text/inline-format listeners, and every new Document/Page panel input; **Phase 2** closed the remaining `bindProps()` listeners (spacing table, `prop-va`/`prop-ov`/`prop-dir` selects, auto-resize, drop-cap/hyphens, image button/file input, and the text-content `blur` snapshot). Only `prop-text-content` `mouseup`/`keyup` remain unguarded, and those merely cache the text selection. See §21.4 and §22.4. The CSS `.read-only-mode` rule only targets `.toolbar`, so the right-hand properties sidebar — including the new Document/Page panel — stays fully interactive in a follower tab. Do not assume "it's a toolbar button, so it's handled."

**Every new Document/Page panel input added in Phase 1 must follow the same pattern:**

```js
el.addEventListener('input', function() {
  if (state.readOnly) return;
  /* page-level field   -> saveSnapshot(); mutate pg.*;        emit();
     project-level field -> markProjectDirty(); mutate state.project; emit();   (see §8) */
});
```

This is not optional: without it, a read-only follower tab can locally mutate `state.pages`/`state.project` through the new panel even though canvas interaction is correctly blocked, producing a silent fork that gets clobbered on the next `reloadFromStorage()` (`index.html:1806`) or, worse, races the leader's `broadcastStateChange()`.

**Adjacent pre-existing bug — fixed.** `bindProps()` — the *existing* X/Y/W/H/fill/stroke/opacity/etc. property inputs — had no `state.readOnly` guard. A follower tab could edit a selected element's properties through the sidebar even though direct canvas manipulation was blocked. Phase 1 added the guard to the map listener; Phase 2 added it to the rest of `bindProps()` (see §22.4).

| Function | Line (guard) | Guard present today? |
|---|---|---|
| `autosave()` | 1556 | ✅ |
| `save` (wrapper) | 1678 | ✅ |
| BroadcastChannel `storage`/msg handlers | 1765-1790, 1822 | ✅ |
| `broadcastStateChange()` / `releaseChannel()` | 1826 / 1832 | ✅ (checks `!state.readOnly`) |
| `beforeunload` / `pagehide` save hooks | 4573 / 4574 | ✅ |
| `btn-save` handler | 3033 (handler 3032) | ✅ |
| `onPointerDown()` | 3796 | ✅ |
| `onKey()` | 4158 | ✅ |
| context-menu handler | 4606 | ✅ |
| `bindProps()` input listeners | — | ❌ **pre-existing gap — fix alongside Phase 1** |
| `btn-new` handler | — | ❌ **pre-existing gap — add guard** |
| `btn-import` handler | — | ❌ **pre-existing gap — add guard** |
| New Document/Page panel bindings (Phase 1) | — | **must be added from the start** |

---

## 13. Implementation Phases

Ordered to keep each step independently shippable and reversible.

### Phase 0 — Foundations (no visible change)
> **Status: implemented** in `index.html` (uncommitted working tree). See §20 for as-built notes, exact locations, and verification. The numbered steps below are the original plan.
1. Add unit helpers + `UNITS`/`UNIT_PRECISION`/`PAGE_PRESETS` near `index.html:1052`.
2. Add `PROJECT_DEFAULTS`, `mkProject()`, `proj()`, `units()`, `contentRect()`, `columnRects()`.
3. Add `state.project` in `state` (1153) and initialize it in **exactly one** place in `init()` so it is never double-set:
   ```js
   var d = load();                                  /* existing call */
   state.project = d ? normalizeProject(d).project : mkProject();
   ```
   (then continue to set `state.pages` from `d` as today). Do not also call `mkProject()` unconditionally before this — the restore branch must win.
4. Write `normalizeProject(d)` and route `load()` (1537), `reloadFromStorage()` (1806), import (3040), and `init()` restore (~4626) through it. Each call site must assign `state.project = normalizeProject(d).project` (see the call-site contract in §9.2).
5. Unify `save()` (1525) and `buildProjectData()` (1571); bump `v` to 2. Respect the `save` re-assignment wrapper (`index.html:1676-1682`) — edit the original `function save()` body, not a second reassignment (see §9.1).
6. Extend `mkPage()` (1254) with `margins/columns/gutter/bleed/slug/background/showGuides`.

*Verify: open an existing saved project; nothing moves; v1 data migrates; `localStorage` blob now contains `project`.*

### Phase 1 — Page & Project UI (the requested feature)
> **Status: implemented** in `index.html` (uncommitted working tree). See §21 for as-built notes, exact locations, and verification. The numbered steps below are the original plan.
1. Rebuild `#prop-doc-section` markup (966) into Page + Project sections with preset, size, orientation, margins, columns/gutter, bleed, slug, units, DPI, grid, guides.
2. **Wire the existing-but-unbound `prop-doc-w`/`prop-doc-h`/`prop-doc-ori`** (fixes G2), then extend to the new fields. **Every new listener must start with `if (state.readOnly) return;`** (see §12.1). Then branch by field kind (see the table in §8):
   - **page-level** fields (size, orientation, margins, columns, bleed, showGuides) → `saveSnapshot(); /* mutate pg.* */ ; emit();`
   - **project-level** fields (units, DPI, grid, guides toggles) → `markProjectDirty(); /* mutate state.project */ ; emit();`
3. Add `renderGuides()` + `#guides-layer` (`z-index:0` — see corrected §7.2/§21.2) and call it from `refresh()`.
4. Add size/orientation presets and "apply to all pages".
5. Validate margins/columns; toast on invalid.
6. Keep `updatePageDimensions()` (2955) driving the page size through the existing `--lp-pw`/`--lp-ph` CSS vars — they already size `.canvas-page`, `.ruler-h`, and `.ruler-v`, so **do not** additionally set explicit `#canvas-page` width/height (that would fight the vars and desync the rulers). After a page-size change, call `clampPan(); applyTransform();`.
7. Rulers (3154) render in the chosen unit.
8. **Fix the adjacent pre-existing gaps:** add `if (state.readOnly) return;` to `bindProps()`'s input listeners (`index.html:2402`, guard near line 2432), to the `btn-new` handler (`index.html:3013`), and to the `btn-import` handler (`index.html:3040`) while you're already editing this region — see §12.1.

*Verify: set A4, set 1 in margins, see margin/column guides; reload persists; export unaffected; a read-only follower tab cannot change page size/margins.*

### Phase 2 — Units across element properties
> **Status: implemented** in `index.html` (uncommitted working tree). See §22 for as-built notes. The numbered steps below are the original plan.
1. Apply `bindUnitInput` to the X/Y/W/H map in `bindProps()` and spacing fields. Carry over the `state.readOnly` guard from Phase 1 step 8. *(As built: folded into the table-driven bindings as `len: true` entries plus `setLenInput()`/`parseLenInput()` — see §22.1.)*
2. Rulers already unit-aware from Phase 1.
3. Round-trip tests (px→in→px) for element positions.

### Phase 3 — Grid, baseline, snapping
> **Status: implemented** in `index.html` (uncommitted working tree). See §23 for as-built notes. The numbered steps below are the original plan.
1. Project-configurable grid spacing in `renderGrid()` and `snap()` — via `gridSpacing()` (with `GRID` as fallback) and `gridSubdivisions()`. New Document-panel **Grid** controls (spacing / subdivisions) and **Baseline** (value + Show). Ruler minor ticks now follow grid spacing.
2. Baseline grid overlay — dashed `#B9A6E0` horizontal lines in `#grid-layer`. Overlay only; text/line-height snapping is **not** implemented (deferred, as planned).
3. Guide snapping in `onPointerMove()` — `snap(v, 'x'|'y')` prefers a content/column guide within `GUIDE_SNAP_TOL` (6 px) on that axis, else falls back to the grid. Endpoint drag, multi-select drag, and the moving edge of a resize are guide-aware; static edges and width/height stay grid-only. Gated by `project.guides.snapToGuides` (default on, toggle in the Guides section).

### Phase 4 — Export/output
> **Status: implemented** in `index.html` (uncommitted working tree). See §24 for as-built notes, exact locations, and verification. The numbered steps below are the original plan.
1. Physical `@page` in `exportHTML()` (1841) and improved `exportPDF()` (1919); page background.
2. "Export with bleed" (expanded canvas + crop marks) — optional.
3. Raster PNG/JPEG at DPI via `foreignObject` (document caveats); image PPI warning in the Image section.

---

## 14. Pre-Flight Checklist

1. Confirm `#prop-doc-w/h/ori` really have no listeners (grep `prop-doc-`); G2 is a live bug.
2. Confirm every page read uses `pg.pw || PW` (there are ~15); none assume a project object exists.
3. Confirm `normalizeProject()` runs **before** any render on every load path (load, import, reloadFromStorage, init).
4. Confirm existing 850×1100 files open unchanged (pixel-diff a screenshot before/after).
5. Confirm guides layer is `pointer-events:none` and excluded from `exportHTML()`.
6. Confirm page resize calls `clampPan()` so the viewport doesn't sit outside bounds.
7. Confirm `GRID` fallback remains until all readers use `project.grid.spacing`.
8. Confirm no `execCommand`/inline-editing code is disturbed (undo snapshots serialize `state.pages` only; project-level changes must use `markProjectDirty()`, **not** `saveSnapshot()` — see §8 and the undo decision below).
9. **(New, §0.1 R2/R3)** Confirm every new Document/Page panel listener starts with `if (state.readOnly) return;`, and confirm the pre-existing gaps get the same guard in the same PR: `bindProps()` (2402), `btn-new` (3013), `btn-import` (3040).
10. **(New, §0.1 R6)** Confirm every new mutator calls `emit()` (1180, via `on(refresh)` at 2980) in addition to whichever dirty mechanism applies: page-level fields use `saveSnapshot()` (1246, which sets `state.dirty` at 1250); project-level fields use `markProjectDirty()` (new) because they are not in `state.pages`. A mutation that only sets `state.dirty` without `emit()` won't re-render, and one that calls `emit()` without setting `state.dirty` won't autosave.
11. **(New, §0.1 R5)** Confirm the `save` re-assignment wrapper (1676-1682) still wraps the *updated* `save()` body correctly — i.e. edits went into the original `function save()` (1525), not a competing redefinition.

> **Undo decision:** `saveSnapshot()` (1246) serializes only `state.pages`. Project changes are not undoable in v1. Page property changes *are* undoable because `page.*` fields live in `state.pages` — call `saveSnapshot()` before mutating a page. Project-level changes (units, DPI, grid, meta) bypass undo, so route them through `markProjectDirty()` (§8) instead: this persists them via autosave **without** adding a misleading no-op undo entry. Document both behaviors for users.

> **Status:** items 1–11 were exercised during Phases 0–4 (§20–§24). Item 1 (G2) was fixed in Phase 1; item 5 was re-confirmed in Phase 4 (the export no longer references guides/grid at all); item 9's pre-existing gaps (`bindProps()`, `btn-new`, `btn-import`) were closed in Phases 1–2. The `NNNN` line refs in items 9–11 predate implementation — use the §19 index.

---

## 15. Test Plan

| # | Test | Expected |
|---|---|---|
| 1 | Open a pre-existing 850×1100 project | No position/size change; preset shows "Custom"; margins 0 |
| 2 | New project | Defaults from `mkProject()`; 0.5 in margin guides visible |
| 3 | Choose A4 + portrait | size = 793.70 × 1122.52 px; ruler/units correct |
| 4 | Toggle landscape | w/h swap; guides recompute; elements stay put |
| 5 | Set margins T/R/B/L = 1 in (with units=in) | Guides at 96 px; values round-trip as 1.000 in |
| 6 | Set columns 3, gutter 12 mm | Two column guides inside content box |
| 7 | Enter invalid margins (left+right ≥ width) | Rejected with toast; previous value restored |
| 8 | Switch units px→mm→px | All displayed values correct within display precision |
| 9 | Reload page | project + page props persist |
| 10 | Export HTML | No guides; trim size correct; `@page` in inches |
| 11 | Print preview | Page defaults to physical size, no scaling (subject to browser) |
| 12 | Set DPI 300 | Canvas editing, page size, export HTML all unchanged; only metadata/raster path uses it |
| 13 | Apply page size to all pages | All pages match; thumbnails re-render; pan re-clamped |
| 14 | Multi-tab: open the same document in two tabs (BroadcastChannel leader/follower, `index.html:1733`) | Second tab becomes `state.readOnly`, shows the banner (`#readonly-banner`); its Document/Page panel inputs are inert (no `state.pages`/`state.project` mutation) even though they're still visible; closing the leader promotes the follower and re-enables editing |
| 15 | Resize page smaller while panned near boundary | `clampPan()` keeps canvas in view |
| 16 | Element at x=0 with margin 96 | Element not moved automatically (guides are non-printing) |
| 17 | Snapping with guides on | Drag near margin snaps; away from margin uses grid |
| 18 | Undo after page margin change | Reverts (page props live in `state.pages`) |
| 19 | Undo after unit change | No undo step (project-level); document behavior |
| 20 | Baseline grid 24 px | Overlay visible, distinct from square grid |
| 21 | Open an old `v:1` save in one tab while a second tab is the read-only follower | Leader migrates via `normalizeProject()`; follower's next `reloadFromStorage()` picks up `project` without needing a manual refresh |
| 22 | Import / New-project in a read-only follower tab | **Today neither `btn-import` (3040) nor `btn-new` (3013) checks `state.readOnly`** — importing/creating wipes the follower's canvas locally until the next broadcast/reload. After adding the guard (§12.1), both should be inert in the follower |
| 23 | Change display units px→in with the panel visible | Every bound field (page size, margins, X/Y/W/H) re-renders in the new unit simultaneously, with no stale values (see §11.2 unit registry) |
| 24 | Change only a project-level field (e.g. DPI 300→150), then wait ~2 s without reloading | The change is autosaved (`autosave()` fires because `markProjectDirty()` set `state.dirty`) and survives a reload; undo/redo does nothing for it (project-level, §14) |
| 25 | Change a page margin, then Ctrl+Z | Reverts the margin (page-level `saveSnapshot()`), and no stray “do-nothing” undo step is left behind |
| 26 | Export HTML (Phase 4) | `@page{size:<w>in <h>in;margin:0}` matches the page; page background applied; guides/grid absent |
| 27 | Enable "Export with bleed" with bleed 9 px, then export HTML | Page grows to `pw+18 × ph+18`; elements shift `+9`; 8 crop marks drawn at trim corners; `@page` grows accordingly |
| 28 | Export PNG at 300 DPI on an 850×1100 page | Downloaded raster is 2656×3438; page background and all visible elements present |
| 29 | Place a 96×96-px image and display it 2 in wide (DPI 300) | Image panel warns "Effective 48 PPI — below the 300 DPI target"; geometry unchanged |
| 30 | Toggle "Export with bleed" in a read-only follower tab | Inert (`state.readOnly` guard) |
| 31 | Set margins > 0 with "Show page guides" on (margin-indicator follow-up) | Dashed `#E0457B` rectangle **plus** four shaded margin bands (`#E0457B` @ 6 %) covering the margin areas appear on the page |
| 32 | Type into a margin field without blurring (margin-indicator follow-up) | Guides follow the typed value live; `state.pages[i].margins` is unchanged until `change`/blur, so Ctrl+Z still reverts to the pre-edit value |
| 33 | Set all four margins to 0 | No shaded bands and no inset rectangle (the rectangle would otherwise sit on the page edge) |
| 34 | Insert a 4000×1000 image (image-aspect follow-up, §25) | Element starts at 300×75 (longest side capped at 300, 4:1 preserved) — not a 300×300 square |
| 35 | Insert a 100×250 image | Element starts at 100×250 (natural size; no upscaling) |
| 36 | Insert a corrupt/non-decodable image | Toast "Failed to load image…"; no element added |
| 37 | Drag a corner resize handle on a locked image (aspect-lock follow-up, §26) | Ratio is preserved exactly (e.g. 300×150 → 500×250 for the SE handle); the opposite corner stays put |
| 38 | Untick "Preserve aspect ratio" and resize | Free resize (300×150 → 500×150); Shift-drag still constrains any element |
| 39 | Type into `#prop-w` on a locked image | `#prop-h` updates to match (and vice-versa); unlocked/non-image elements are unaffected |
| 40 | "Change Image" on a locked image with a different ratio | Box refits to the new ratio keeping the current longest side, and the canvas `<img>` updates immediately (not only after the next rebuild) |
| 41 | Stretch an image element to the wrong shape, then click "Reset to original ratio" (reset-ratio follow-up, §27) | Box snaps back to the source image's ratio, keeping the current longest side (300×300 → 300×150 for a 400×200 source) |
| 42 | Click "Reset to original ratio" on an element already at the correct ratio | Nothing moves; toast "Already at the original aspect ratio." |
| 43 | Click "Reset to original ratio" then Ctrl+Z | The pre-reset (distorted) box is restored — the reset takes an undo snapshot |
| 44 | Click the toolbar snapping button (snap follow-up, §28), then drag an element | The button turns off and the element follows the pointer exactly (no grid or guide snap) |
| 45 | Click it again (or press `S`) | The button lights up and dragging snaps to the grid again |
| 46 | Toggle snapping, then reload the page | The button comes back in the state you left it (per-browser preference), and the project is not marked dirty |
| 47 | Switch Document units to **inches** (grid-units follow-up, §29) | The Spacing field becomes `0.125`, grid lines are drawn every 12 px, and the ruler minor ticks sit exactly on them — every 8th grid line coincides with a 1 in ruler major |
| 48 | Switch to **millimetres** | Spacing becomes `2.5`; the 4th grid line lands on the 10 mm ruler major |
| 49 | Switch units with a custom grid (e.g. 20 px) | The step is re-expressed as the nearest clean value in the new unit (`0.25 in`); `subdivisions` is untouched; `0` baseline stays `0`; read-only ignores the change |
| 50 | Open a document saved before §29 with a non-px unit | The stored step is **not** rewritten on load (grid matches what was saved); it is only re-expressed when the unit selector is used again |
| 51 | With snapping **off**, hold **Shift** and drag an element (momentary-snap follow-up, §30) | The element snaps to the grid/guides while Shift is held, and the snap button shows the dashed "temporarily on" outline |
| 52 | Release Shift **mid-drag** (button still down) | The very next pointer move is free again — `snapActive()` is re-evaluated on every move, not latched at pointer-down |
| 53 | Hold Shift, then `Alt`/`Cmd`-Tab away (window blur) or switch tabs, then come back | Snapping is not stuck on — `blur` / `visibilitychange` clear `snapTemp` |
| 54 | Hold Shift while the master switch is already **on** | No visual change beyond the latched `.active` state (the dashed outline is suppressed by `:not(.active)`); snapping stays on after release |
| 55 | Select a shape and click each of the nine **Alignment** buttons with "Align to: Page" (alignment follow-up, §31) | The shape's bounding box is placed flush to that page edge/centre; the model *and* the rendered DOM agree; one undo step per click |
| 56 | Switch "Align to" to **Margins**, then to **Selection** with two or more shapes selected | Margins align to the content box (page minus margins); Selection aligns to the union bounding box of the selected elements |
| 57 | Align a **line** to a corner | Both endpoints (`x`,`x2`,`y`,`y2`) shift together — the line's bounding box is what is aligned |
| 58 | Click an align button when the element is **already** at that anchor, or when read-only | No mutation and **no undo step** is pushed (a no-op never creates dead history); read-only is a hard no-op |
| 59 | Export a document with a long text frame to PDF, then compare the wrap points to the canvas (PDF-fidelity follow-up, §32) | Every line breaks at the same word; the exported content width equals the canvas's (`width − 2×padding`, e.g. `184` not `200`) |
| 60 | Export a two-column frame and a rich-text frame (bold + `<p>` + paragraph spacing) | Column widths, line counts, glyph-run positions and total rendered text height are identical canvas vs export |
| 61 | Inspect the exported HTML `<head>` | It carries the canvas's box-sizing reset + `.el-text`/`.el-clip`/`.el-content`/`.el-image` rules and the elements carry `class="canvas-element el-text"`; there is **no** external `<link>`/web-font request (self-contained, offline-identical) |

---

## 16. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| **Unit rounding drift** | Single source (px); one-way conversion at boundaries; display-only rounding |
| **Page resize orphans/clips elements** | Never clamp objects on resize; allow off-canvas; warn only if content exceeds trim and auto-resize is on |
| **Migration shifts existing docs** | Zero margins + `custom` preset; preserve `pw`/`ph` exactly; pixel-diff test |
| **Guides leak into export** | Guides live in a DOM layer not read by `exportHTML()`; test |
| **Global CSS vars vs multi-size docs** | Vars are derived per current page; fine for one-at-a-time editing; if a spread/overview view ever lands, switch to inline styles |
| **Print accuracy** | Browsers assume 96 dpi and the print dialog may scale; emit `@page size`, instruct "Actual size"; cannot set trim/bleed boxes — document limitation. **Phase 4:** implemented via `@page` + page background (§24.2); trim/bleed-box limitation stands |
| **foreignObject raster export** | Browser/font/CORS caveats; ship behind an explicit button; fall back to print-to-PDF. **Phase 4:** implemented with `try/catch` + `img.onerror` toasts; PDF remains the documented fallback (§24.3) |
| **Snapping changes drag feel** | Phase 3 isolated; guides preferred only within tolerance; keep grid default; easy rollback via `snapToGuides` |
| **Two persistence shapes** | Unify in Phase 0 before adding fields |
| **Dropping `GRID` too early** | Keep fallback until every reader is migrated; grep for `GRID` before removal |
| **Read-only follower tab edits project/page state (§0.1 R2/R3)** | Guard every new panel listener with `state.readOnly`; fix the adjacent pre-existing gap in `bindProps()` in the same PR |
| **Guides layer buries selection handles (§0.1 R4)** | Use `z-index:2`, not `999`; verify visually against a selected element sitting on a margin/column guide before merging |
| **Project-level edits never autosave** | `state.dirty` is set only by `saveSnapshot()` (1250), and `autosave()` bails when not dirty (1557). Project fields (units/DPI/grid/meta) are not in `state.pages`, so route them through the new `markProjectDirty()` helper, not `saveSnapshot()` (which would also add no-op undo steps) |

---

## 17. Rollback

- **Phase 0:** project/migration code is additive. Revert the `save()`/`buildProjectData()` change to return to `v:1`; old blobs still load because `normalizeProject()` is only invoked on load and tolerates absence.
- **Phase 1 UI:** remove the new listener bindings; the markup can stay hidden. Guides: delete `renderGuides()` from `refresh()`.
- **Phase 2+:** each phase is feature-gated (`units` just changes display; `snapToGuides` flag; export options default off).
- **Phase 4 export:** `exportBox()` returns `off: 0` whenever `project.export.includeBleed` is false, so `exportElementsHTML(pg, 0)` reproduces the pre-Phase-4 markup byte-for-byte; raster buttons are additive and can be removed without touching the HTML/PDF path.
- Keep a kill switch for guides: `state.project.guides.showMargins = false` hides all overlays without code changes.

---

## 18. Open Questions / Out of Scope (v1)

**Resolved before Phase 1 (decisions taken):**
1. Default page size for *new* documents — **keep 850×1100** (`PROJECT_DEFAULTS.page` unchanged).
2. Margin defaults for migrated docs — **0.5 in (48 px)**, matching new docs. Safe because guides are non-printing overlays that were not rendered before Phase 1, so existing documents show no change other than the margin guides appearing at the 0.5 in default.
3. Are project-level fields (units, DPI) undoable? — **No.** Page-level changes are undoable via `saveSnapshot()`; project-level changes go through `markProjectDirty()` (autosave without polluting the undo stack). See §8 / §21.3.
4. Should the "Document" panel persist when an element is selected? — **No.** It is shown only in the `sel.length === 0` branch of `updateProps()` and hidden as soon as an element is selected. It stays in the Properties pane (no new tab).

**Explicitly out of scope for v1 (track for later):**
- Facing pages / spreads / master pages.
- Ruler origin offset to margins; drag-able custom guides.
- Color mode (RGB/CMYK), ICC profiles.
- True PDF trim/bleed boxes with printer marks. (Phase 4 ships a *proof-level* bleed: expanded canvas + crop marks, §24.2 — not real boxes.)
- Text baseline-grid snapping and named paragraph grids.
- Per-page units (units are project-wide).
- Raster export pixel-perfection across all browsers. (Phase 4 ships PNG/JPEG via `foreignObject` with documented caveats, §24.3.)

---

## 19. Appendix — Quick Reference

### Conversion table
| Unit | px per unit | units per px |
|---|---|---|
| px | 1 | 1 |
| in | 96 | 0.1041699 |
| cm | 37.79527591 | 0.2645865 |
| mm | 3.77952788 | 0.26458365 |
| pt | 1.33333365 | 0.75 |
| pc | 16 | 0.628 |

### Line-number index (current tree)
> **Phases 0–4 are now implemented** (§20–§24), plus nine follow-ups: the margin indicator (§21.2), image-insert aspect ratio (§25), the image aspect-ratio lock (§26), the reset-to-original-ratio button (§27), the snapping master switch (§28), the unit-aware grid step (§29), momentary snapping (hold Shift, §30), the shape-alignment panel (§31), and the PDF pixel-fidelity fix (§32). Phase 0 inserted the helper block; Phase 1 added the guides layer, unit-aware rulers, the Document-panel bindings, and read-only guards; Phase 2 made element geometry/spacing inputs unit-aware and closed the remaining `bindProps()` guards; Phase 3 made the grid project-configurable, added the baseline overlay and guide snapping, and removed the last hardcoded `GRID` reads; Phase 4 added physical `@page`/page-background export, optional bleed + crop marks, raster PNG/JPEG export, and the image-PPI warning; §31 replaced the dead `align()` helper with `alignSelection()` and added the Alignment panel; §32 made `exportElementsHTML()` reuse `domEl()` and mirrored the canvas CSS into the export `<style>` (and dropped the dead Google-Fonts `<link>`), so the print/PDF layout wraps identically to the canvas. Every line after `1052` shifted downward. The inline `index.html:NNNN` references elsewhere in this document were written before implementation and may be stale — **use this table (re-verified against the current working tree) when editing.**

The **as-built sections §20–§32** were re-verified against the current tree and are safe to trust. The **design sections §1–§18** were written before implementation and were *not* renumbered: their `index.html:NNNN` references point at the pre-implementation line numbers and should be read as "the original location of this symbol" rather than a live address. Symbol names are stable; line numbers are not. When re-verifying after a new edit, note that the §29 helper block sits just after `baselineSpacing()` (`index.html:1483`), the §29 `#prop-doc-units` handler just after `bindDocProps()`'s start, and the §30 `snapActive()`/`setSnapTemp()` helpers sit just after `setSnapEnabled()` (`index.html:1564`) — an insertion there shifts everything below it. The §30 keyboard wiring sits inside `init()` just after the existing `keyup` listener (`index.html:5771-5783`). The §31 helpers (`ALIGN_TARGETS`/`elBBox()`/`alignTargetBox()`/`alignSelection()`) replace the old `align()` at `index.html:1914-1999`, and `bindAlign()` sits between `bindProps()` and `bindPanelEditor()` (`index.html:3416`). The §32 change rewrote `exportElementsHTML()` (`index.html:2380`) and the `<style>`/font block inside `exportHTML()` (`index.html:2396`); it **shortened** `exportHTML()` by ~23 lines and grew `exportElementsHTML()` by ~4, so anchors from `exportElementsHTML` down were re-measured (see the table).

| Anchor | Line |
|---|---|
| `--lp-pw`/`--lp-ph` CSS | 59-60  |
| `.toolbar` CSS (+ **§28.6** `overflow-x:auto`) | 155-156  |
| **§31** `.align-grid` / `.align-btn` CSS | 370-375 |
| `.canvas-page` / `.page-content` CSS | 418-419  |
| `.guides-layer` CSS | 423  |
| `.guide-margin` / `.guide-column` / `.guide-bleed` / `.guide-slug` CSS | 424-428  |
| `.guide-margin-band` CSS (shaded margin area) | 425  |
| `.read-only-mode` CSS | 761  |
| `#readonly-banner` markup | 864  |
| Canvas DOM (`#canvas-page`, `#grid-layer`, rulers, `#page-content`, `#guides-layer`) | 884  |
| Element X/Y/W/H markup (`prop-x`/`y`/`w`/`h`) | 932  |
| **§31** Alignment panel markup (`#prop-align-section`, `#prop-align-target`, `#prop-align-grid`, 9 × `[data-align]`) | 936-953 |
| Document panel markup (`#prop-doc-section` … apply-all, incl. Grid/Baseline + `#prop-doc-exportbleed`) | 1010-1033 (export-bleed 1024, apply-all 1032)  |
| Image panel markup (`#prop-image-btn`, `#image-file-input`, **`#prop-image-lockaspect`** §26, **`#prop-image-reset-ratio`** §27, `#prop-image-ppi`) | 1035  |
| Toolbar export menu (`#btn-export` + `#export-drop` with `export-pdf/png/jpg/html` items) | 846 |
| **§28** `#grid-toggle` / `#snap-toggle` toolbar markup | 850 / **851** |
| **§30** `.tool-btn.snap-temp` CSS (dashed "temporarily on" outline) | 170-172 |
| `PW, PH, GRID, …` constants | 1116  |
| **Phase 0 helpers** (`PX_PER_IN`, `UNITS`, `UNIT_PRECISION`, `toPx`/`fromPx`, `fmtUnit`, `fmtNum`, `parseUnit`, `PAGE_PRESETS`) | 1125-1199  |
| **Phase 2 helpers** `setLenInput()` / `parseLenInput()` | 1171 / 1178 |
| `PROJECT_DEFAULTS` (incl. `export.includeBleed`) | 1202  |
| `mkProject()` | 1242  |
| `proj()` / `units()` | 1244 / 1245 |
| `markProjectDirty()` | 1249  |
| `contentRect()` | 1252  |
| `columnRects()` | 1264  |
| `shallowMerge()` | 1275  |
| `normalizeProject()` (clamps `grid.*`, merges `export.*`, backfills `el.lockAspect` §26) | 1284 (lock backfill 1323)  |
| `state` (starts with `project: null`; **§28** `snapEnabled: true` at 1448, **§30** `snapTemp: false` at 1449, **§31** `alignTarget: 'page'` at 1450) | 1429  |
| `emit()` / `on()` | 1460 / 1464 |
| **Phase 3 grid helpers** `gridSpacing()` / `gridSubdivisions()` / `baselineSpacing()` / `nearestTarget()` / `guideTargets()` / `GUIDE_SNAP_TOL` | 1473 / 1478 / 1483 / 1522 / 1533 / 1549 |
| `snap(v, axis)` (guide-aware, grid fallback; **§28** master gate) | 1550  |
| **§28** `setSnapEnabled(on, quiet)` | 1564  |
| **§30** `snapActive()` / `setSnapTemp(on)` (momentary snapping) | 1580 / 1583 |
| `renderGrid()` (project spacing/subdivisions + baseline overlay) | 1620  |
| `renderGuides(preview)` (margin bands + live preview) | 1646  |
| `saveSnapshot()` | 1685  |
| `mkPage()` (extended) | 1693  |
| `mkEl()` (image branch sets `lockAspect`, §26) | 1750 (1797)  |
| `getEl()` | 1816  |
| **§31** `ALIGN_TARGETS` / `elBBox()` / `alignTargetBox()` / `alignSelection()` | 1914 / 1917 / 1930 / 1956 |
| `save()` (original body) | 2039  |
| `load()` (routes through `normalizeProject`) | 2053  |
| `buildProjectData()` (`v:2`, includes `project`) | 2079  |
| `setReadOnly()` | 2298  |
| `reloadFromStorage()` (assigns `state.project`) | 2315  |
| **Phase 4** `exportBox()` / `cropMarksHTML()` / `exportElementsHTML()` | 2352 / 2363 / 2380 |
| `exportHTML()` (physical `@page`, bg, bleed; excludes guides; **§32** reuses `domEl()` markup + mirror CSS, no web-font link) | 2396  |
| **Phase 4** `imagePPI()` | 2452  |
| **§27** `naturalFitBox()` / `naturalImageSize()` | 2465 / 2474 |
| `exportPDF()` (waits for load, prints) | 2486  |
| **Phase 4** `exportRaster(format)` | 2503  |
| `domEl()` (per-element `z-index: i+1`) | 2641  |
| `drawThumb()` | 2747  |
| `updateProps()` (unit-aware element fields + doc panel + PPI + lock checkbox + **§31** align section) | 2869 (align section 2877)  |
| `syncElementDOM()` (syncs image `src`, §26.5) | 3014  |
| `bindProps()` (guarded; `len: true` entries; W/H coupling) | 3056 (lock handler 3334; **§27 reset handler 3348**)  |
| **§31** `bindAlign()` (target select + delegated 3×3 grid click) | 3416 |
| **§27** Change Image refit via `naturalFitBox()` | 3395  |
| `refresh()` (calls `renderGrid()` then `renderGuides()`) | 3726  |
| `updatePageDimensions()` | 3744  |
| `btn-new` handler (guarded; resets `state.project`) | 3802  |
| `btn-save` handler (guarded) | 3823  |
| `btn-import` handler (guarded) | 3831  |
| `btn-export` export-menu handler (`bindExportMenu`) | 4059 |
| **§28** `#grid-toggle` / `#snap-toggle` click handlers | 3892 / 3899 |
| `renderRulers()` (unit-aware; minor ticks = grid spacing) | 3957  |
| `rulerMajorPx()` | 4023  |
| `commitPage()` / `commitProject()` | 4035 / 4044 |
| `bindDocProps()` (incl. Grid/Baseline, export-bleed, live preview) | 4123  |
| **§29** `#prop-doc-units` change handler (snaps grid step) | 4205-4216 |
| `showHandles()` (`.resize-handle` + `dataset.handle`) | 4284 |
| `onPointerDown()` (guarded; `resizeOrig` incl. `lockAspect`) | 4883 |
| `onPointerMove()` (**aspect-lock resize block**) | 5010 |
| `snap()` call sites | 5056-5060 (endpoint), 5112-5118 (drag), 5162-5164 (resize) |
| `onKey()` (guarded; **§28** `S` shortcut) | 5268 |
| `IMAGE_INSERT_MAX` / `fitImageSize()` / `onImageInsert()` (aspect-ratio insert, §25) | 5456 / 5461 / 5468 |
| `createDemo()` | 5504 |
| `init()` (**§30** Shift keydown/keyup/blur wiring 5750-5761; **§28** snap restore 5816) | 5707 |
| `init()` sets `state.project` | 5780 |
| **§29** `GRID_UNIT_LADDER` | 1494-1501 |
| **§29** `snapLenToUnit(px, unit)` | 1505-1516 |
| **§29** `r2(n)` | 1519 |
| `.resize-handle` / `.selection-ghost` / `#inline-toolbar` z-index | 435 (`z-index:10`) / 447 (`z-index:50`) / 91 (`z-index:10000`) |
---

## 20. As-Built Notes — Phase 0 (implemented)

> Implemented in `index.html` on the uncommitted working tree. No visible or behavioral change. Verified by `node --check`, isolated unit assertions, and a headless fake-DOM harness that runs the real `init()`.

### 20.1 Added (single block, `index.html:1056-1222`)
- `PX_PER_IN = 96`; `UNITS` (`px`/`in`/`cm`/`mm`/`pt`/`pc`); `UNIT_PRECISION`.
- `toPx()` / `fromPx()` / `fmtUnit()` / `parseUnit()`.
- `PAGE_PRESETS` (Letter, Legal, Tabloid, A3–A6, B5, DL, 1080p, 4K) in canonical px.
- `PROJECT_DEFAULTS`, `mkProject()` (deep clone), `proj()`, `units()`.
- `markProjectDirty()` — sets `state.dirty` for project-level (non-page) edits.
- `contentRect(p)` / `columnRects(p)` — margin/column geometry.
- `shallowMerge()` — ES5 merge used by migration.
- `normalizeProject(d)` — the single migration entry point.

### 20.2 Data-model changes
- `state.project` added as the first key of `state` (`index.html:1403`).
- `mkPage()` (`index.html:1615`) seeds `pw/ph/orientation/preset/margins/columns/gutter/bleed/slug/background/showGuides` from `state.project.page` (falls back to `PROJECT_DEFAULTS.page`).
- Persisted shape is `v:2` and includes `project` (`buildProjectData()` `index.html:1940`). `save()` (`index.html:1900`) serializes via `buildProjectData()` and keeps zoom/pan in their separate `lp-zoom`/`lp-panX`/`lp-panY` keys. The `save` re-assignment wrapper (`index.html:2047`) is untouched, so auto-save-to-file and multi-tab broadcast pick up `project` automatically.

### 20.3 Load paths routed through `normalizeProject()`
| Path | Location | Assignment |
|---|---|---|
| localStorage | `load()` `index.html:1914` | returns normalized doc |
| storage event / follower | `reloadFromStorage()` `index.html:2176` | `state.project = d.project` |
| JSON import | `btn-import` handler `index.html:3682` | `state.project = d.project` |
| first paint | `init()` `index.html:5550` | `state.project = saved ? saved.project : mkProject()` |
| New project | `btn-new` handler `index.html:3653` | `state.project = mkProject(); state.pages = [mkPage(0)]` |

### 20.4 Migration guarantees (verified)
- Existing pages keep `pw`/`ph` **exactly**; `preset` becomes `custom` and `margins` inherit the project default (`48px` = 0.5 in). **No visual change at Phase 0** because guides were not rendered yet; from Phase 1 the margin guides appear at the 0.5 in default (decision §18.2).
- New projects get `PROJECT_DEFAULTS.page.margins = 48px` (0.5 in).
- Missing project/units/DPI/grid/guides/meta are backfilled from defaults; an unknown `units` falls back to `px`; `dpi <= 0` falls back to `300`.
- Corrupt, page-less, and empty-page blobs return `null` from `load()` and fall through to `createDemo()`.

### 20.5 Verification performed
- `node --check` on both extracted inline scripts: clean.
- 40+ isolated unit assertions on the helpers + migration (unit round-trips, preset values, deep clone, v1/v2/custom merge, content/column rects, clamping): all pass.
- Headless fake-DOM harness running the real `init()` for six inputs — fresh, `v:1` (850×1100), `v:2` (mm / 150 dpi), empty-pages, page-less, corrupt — **zero runtime errors**; `state.project`, page fields, `buildProjectData().v === 2`, and the persisted `project` all behave as specified.

### 20.6 Explicitly NOT in Phase 0 (deferred)
- No UI, no `renderGuides()`, no `#guides-layer`, no rulers-in-units (Phase 1).
- No `state.readOnly` guards added to `bindProps()`/`btn-new`/`btn-import` (added later in Phase 1 — see §21.4).
- `GRID` constant is still used by `renderGrid()`/`snap()` (Phase 3).

---

## 21. As-Built Notes — Phase 1 (implemented)

> Implemented in `index.html` on the same uncommitted working tree as Phase 0. This is the user-visible feature: page/project properties, non-printing guides, unit-aware rulers, and a live Document panel. Verified by `node --check` and an extended fake-DOM harness that drives the real handlers (29 assertions, all pass).

### 21.1 What Phase 1 added

**CSS** (`index.html:409-414`)
- `.guides-layer` (`position:absolute; inset:0; z-index:0; pointer-events:none; overflow:visible`).
- `.guide-margin` (dashed `#E0457B`), `.guide-column` (dashed `#38BDF8`), `.guide-bleed` (solid `#16A34A`), `.guide-slug` (dotted `#A855F7`).
- `.guide-margin-band` (solid `#E0457B` at `opacity:.06`) — shaded margin areas (added in the margin-indicator follow-up, see §21.2).

**DOM** (`index.html:866`)
- `<div class="guides-layer" id="guides-layer" aria-hidden="true"></div>` appended as the last child of `#canvas-page` (a sibling of `#page-content`).

**Document panel markup** (`index.html:974-992`) — full rebuild of `#prop-doc-section`:
- Project-level: `prop-doc-units` (px/in/cm/mm/pt/pc), `prop-doc-dpi`.
- Page-level: `prop-doc-preset` (Custom, Letter, Legal, Tabloid, A3–A6, B5, DL, 1080p, 4K), `prop-doc-w`/`prop-doc-h`, `prop-doc-ori` (portrait/landscape), `prop-doc-mt/mr/mb/ml`, `prop-doc-cols`, `prop-doc-gutter`, `prop-doc-bleed`, `prop-doc-slug`, `prop-doc-showguides`.
- Project guide toggles: `prop-doc-g-margins`, `prop-doc-g-cols`, `prop-doc-g-bleed`, `prop-doc-g-snap`.
- Action: `prop-doc-apply-all` ("Apply size to all pages").

**JS**
- `fmtNum()` + `INPUT_PRECISION` (`index.html:1126-1127`) — number-only formatting for `<input type=number>` (no unit suffix, trailing zeros trimmed).
- `renderGuides()` (`index.html:1568`) — builds one `innerHTML` string of absolutely-positioned guide divs; called from `refresh()` right after `renderGrid()`.
- `rulerMajorPx(u)` (`index.html:3874`) and a rewritten unit-aware `renderRulers()` (`index.html:3808`) — minor ticks every `GRID` px, major ticks/labels at round values in the current unit.
- `commitPage()` / `commitProject()` (`index.html:3886`/`3895`), `setPageSize()`, `setOrientation()`, `setMargin()`, `setColumns()`, `setGutter()` (`index.html:3902-3970`).
- `bindDocProps()` (`index.html:3974`) — wires every field above with `change` listeners, each beginning with `if (state.readOnly) return;`.
- `updateProps()` doc branch (`index.html:2757`) now populates **all** panel fields (was only W/H) via `fmtNum(..., units())`. This fixes **G2**: `prop-doc-ori` was previously neither read nor written; it is now a real portrait/landscape control.
- `init()` calls `bindDocProps()` immediately after `bindProps()` (`index.html:5552`). The bind runs before `state.project` is assigned (`index.html:5610`), which is safe: the handlers read `units()`/`state.project` lazily at event time, never at bind time.

### 21.2 Guides layer & stacking correction

The original plan put `#guides-layer` at `z-index:2`. That is wrong: `.page-content` has `z-index:1` and therefore establishes a stacking context, so a **sibling** layer at `2` paints *above* `.page-content` and above every element inside it. Phase 1 uses **`z-index:0`**, which paints after `.grid-layer` but before `.page-content` — behind all elements, handles (10), and the selection ghost (50), yet visible through empty space. See §0.1 R4 and the corrected §7.2.

`renderGuides()` bails when the page is missing, `state.project` is missing, or `p.showGuides === false`. The column guide skips index 0 (the content-box left edge is already drawn by the margin guide). Guides are a pure DOM overlay and are **never** written into `state.pages`, so `exportHTML()`/`exportPDF()` are untouched and guides remain non-printing.

**Margin-indicator follow-up (post-Phase-4).** A user asked for margin settings to "put a visual indicator on the page." The dashed inset rectangle already did this, but it is a single thin line, so it was made unmistakable:

- `renderGuides()` now also emits four `.guide-margin-band` divs covering the top/bottom/left/right margin areas (a 6 % magenta tint), so the non-content zone reads at a glance. The inset rectangle is skipped when all four margins are 0 (it would otherwise sit on the page edge).
- `renderGuides(preview)` accepts an optional page object. `bindDocProps()` adds `input` listeners to `prop-doc-mt/mr/mb/ml`, `prop-doc-cols`, `prop-doc-gutter`, and `prop-doc-bleed` that call `previewGuides()`, which renders guides from a `JSON.parse(JSON.stringify(cpage()))` copy. This makes the indicator follow typing **without** mutating `state.pages`, so the undo stack stays clean; the pre-existing `change` listener still does the snapshot-backed commit. Both `input` and `change` remain `state.readOnly`-guarded.

Verified in headless Chromium: 4 bands render (≈72.6k tinted pixels at `#FDF4F7`), a live `input` preview moves the guide to `left:150px` while `state.pages[0].margins` stays unchanged, the subsequent `change` commits `left:150`, and read-only blocks both. Also covered by the harness `marginband` suite (13 assertions).

### 21.3 Page-vs-project commit semantics (matches §18.3)

| Change | Helper | Undo | Autosave |
|---|---|---|---|
| Page size / orientation / preset | `setPageSize()` / `setOrientation()` | yes (`saveSnapshot()`) | yes |
| Margins / columns / gutter / bleed / slug / `showGuides` | `setMargin()` / `setColumns()` / `setGutter()` / `commitPage()` | yes | yes |
| Units / DPI / guide toggles | `commitProject()` | **no** | yes (`markProjectDirty()`) |

Every mutation ends in `emit()` so `refresh()` re-renders the canvas, guides, and rulers. `setMargin()` rejects margins that would exceed the page (minus 20 px) with a toast and a panel re-sync; `setGutter()` rejects a gutter too large for the column count. `setOrientation()` swaps custom dimensions, or reads `PAGE_PRESETS` when a named preset is active.

### 21.4 Read-only guards (closes R2/R3)

Added `if (state.readOnly) return;` to:
- `bindProps()` element listeners — position/size map, `prop-tc`, inline-format selects, `prop-fs`, `prop-tcl` (`index.html:2935+`).
- `btn-new` (`index.html:3653`) and `btn-import` (`index.html:3682`).
- All `bindDocProps()` listeners.

`state.readOnly` references in `index.html` went from 15 (pre-existing) to **36**. No unguarded mutating entry point for page/project state remains.

### 21.5 Verification performed
- `node --check` on both extracted inline scripts: clean.
- Fake-DOM harness extended to drive the real handlers — 29 assertions, all pass:
  - `fmtNum` (px/in/mm round-trips, trailing-zero trim); `rulerMajorPx`.
  - Guides: margin rect at 0.5 in, no column guide at 1 column, two column guides at 3 columns, bleed guide, `showGuides:false` → empty layer.
  - `setPageSize('a4')` → preset/`orientation`; `setOrientation('landscape')` swaps; `setMargin` clamps/rejects; `setColumns`/`setGutter`.
  - Undo accounting: a page change pushes exactly one snapshot; a `commitProject()` change pushes **none** but sets `state.dirty`.
  - `updateProps()` populates the panel in the active unit (W/H/margins/gutter/bleed/slug), including `prop-doc-units`/`prop-doc-preset`/`prop-doc-ori`.
  - `renderRulers()` produces labeled major ticks in the current unit.
- Migration harness still green across all six inputs (`none`/`v1`/`v2`/`empty`/`corrupt`/`nopages`), `errors=0`.

### 21.6 Cleanup, deferrals & known limitations
- Removed the leftover `console.log('[DEBUG] …')` calls (and a `getComputedStyle` ancestor-walk) from `updateProps()`; the function is called on every selection change, so the walk was also a per-render cost.
- **Deferred to Phase 2:** units on *element* properties (x/y/w/h/font-size inputs still show raw px) — only the page/project panel and rulers are unit-aware today. *(Done in Phase 2 — see §22.)*
- **Deferred to Phase 3:** grid/baseline snapping and `GRID`-constant removal; `snap()` still snaps to the fixed 10 px grid, not to guides. *(Done in Phase 3 — see §23.)*
- **Known limitation:** `renderRulers()` rebuilds all tick DOM on every `refresh()` (i.e. every drag frame). Acceptable at current page sizes; revisit with a canvas-based ruler if it becomes a hot path.
- **Known limitation:** the bleed/slug guides render but do not yet enlarge the export/print box (true trim/bleed boxes are out of scope for v1 — §18).
- **Not yet done:** no manual browser pass (the harness is headless); a visual check of guide colors/weights across light/dark themes is still recommended before shipping.

---

## 22. As-Built Notes — Phase 2 (implemented)

> Implemented in `index.html` on the same uncommitted working tree. Element geometry and spacing inputs now display and accept the project's display unit, and the remaining unguarded element-property listeners in `bindProps()` were closed. Verified by `node --check` and 21 new harness assertions (all pass), with the Phase 0/1 suites still green.

### 22.1 Unit-aware element inputs
- New helpers (`index.html:1131` / `1138`):
  - `setLenInput(id, px)` — writes `fmtNum(px, units())` into a numeric input, **skipping `document.activeElement`** so a re-render never clobbers the field being typed into.
  - `parseLenInput(value, dflt)` — parses a unit-aware input back to px (float); empty / lone `-` / `.` fall back to `dflt` so clearing a field does not snap it to 0.
- `bindProps()`'s table gained `len: true, dflt` entries for `prop-x/y/w/h` and `prop-x2/y2`; the listener now rounds length fields to integer px (`Math.round(parseLenInput(...))`).
- The "spacing bindings" table gained `len: true, dflt` for `prop-pad` (8), `prop-ps` (0), `prop-ti` (0), `prop-cg` (20), `prop-minh` (0), `prop-maxh` (0); these keep **fractional** px (no rounding).
- `updateProps()` (`index.html:2751`) now populates all of those fields through `setLenInput()`.

### 22.2 What stays in px (deliberate)
Font size (`prop-fs`), letter spacing (`prop-ls`), line height (`prop-lh`, a unitless multiplier), column count (`prop-cc`), stroke width (`prop-sw`), and corner radius (`prop-radius`) remain in px. Font size / letter spacing / line height are typographic values (InDesign keeps them in pt/em regardless of the ruler unit), and stroke width / radius are decoration-scale dimensions where unit conversion produces unusable values (e.g. `0.001 in`). The split is intentional.

### 22.3 Unit-change refresh
No registry was needed (contrast §11.2). `setLenInput()` / `parseLenInput()` read `units()` **live** — they never capture it — and every unit change runs `commitProject()` → `emit()` → `refresh()` → `updateProps()`, which re-populates all fields. A harness assertion confirms `cm` re-renders `prop-x` after the unit switch.

### 22.4 Read-only guards — `bindProps()` now fully covered
Phase 1 guarded the map / text / inline-format listeners. Phase 2 closed the rest of `bindProps()` (`index.html:2935`): the spacing table, the `prop-va`/`prop-ov`/`prop-dir` select table, the auto-resize checkbox, the drop-cap/hyphens checkboxes, the image button, the image file input, and the `prop-text-content` `blur` snapshot. The only remaining unguarded listeners are `prop-text-content` `mouseup`/`keyup`, which merely cache the text selection and mutate nothing. `state.readOnly` references: 15 (pre-existing) → 36 (Phase 1) → **43** (Phase 2).

### 22.5 Verification performed
- `node --check` on both inline scripts: clean.
- 21 new harness assertions, all pass:
  - px mode shows raw values; `in` mode converts (`96px → "1"`, `48px → "0.5"`, `8px → "0.083"`).
  - Typing `2` in `in` mode stores `x = 192px`; `192px → "2"` round-trips.
  - Geometry rounds (`10.6px → 11`); spacing keeps fractions (`letterSpacing 0.1`, `padding 12.5`).
  - Empty / lone-sign input falls back to the field default.
  - A focused field is not overwritten by `updateProps()`.
  - `readOnly` blocks `prop-x`, `prop-pad`, `prop-cg`, `prop-lh`.
  - Unit switch to `cm` re-renders `prop-x`.
- Phase 0/1 suites unchanged: 29/29; all six load modes `errors=0`.

### 22.6 Known limitations / follow-ups
- **Pre-existing, out of scope:** element geometry/style edits made through the panel (`x/y/w/h/fill/stroke/spacing`) are not pushed to the undo stack and do not set `state.dirty`, so they are not undoable and may not autosave until another action dirties the document. Phase 2 only changed *units*, not the commit semantics; flagged for a dedicated fix (the same `saveSnapshot()`/`emit()` treatment the canvas drag path already uses).
- No manual browser pass yet (the harness is headless).
- ~~Phase 3 still owns `GRID` → `project.grid.spacing` and guide snapping.~~ Done in Phase 3 — see §23.

---

## 23. As-Built Notes — Phase 3 (implemented)

> Implemented in `index.html` on the same uncommitted working tree. The grid is now driven by `state.project.grid`, a baseline overlay can be drawn, and dragging/resizing prefers nearby page guides over the grid. `GRID` is now referenced only as a fallback inside `gridSpacing()`. Verified by `node --check` and 37 new harness assertions (all pass), with the Phase 0/1/2 suites still green.

### 23.1 Grid & baseline config
- New helpers (`index.html:1444`/`1449`/`1454`): `gridSpacing()` (project spacing, `GRID` fallback, clamped `>= 1`), `gridSubdivisions()` (clamped `1..50`), `baselineSpacing()` (`> 0` or `0`).
- `renderGrid()` (`index.html:1542`) now draws minor lines every `spacing` and major lines every `spacing * subdivisions` (index-counter based, so no float-modulo pitfalls), and a dashed `#B9A6E0` baseline overlay when `grid.showBaseline && baseline > 0`. Removed the dead `cc` variable. With the defaults (10/5) the output is identical to the pre-Phase-3 grid.
- `renderRulers()` (`index.html:3808`) minor ticks now use `Math.max(2, gridSpacing())` instead of `GRID`, so rulers track the grid.
- New Document-panel rows: **Grid → Spacing / Subdiv**, **Baseline → value / Show** (`index.html:998-1000`). Spacing and baseline are unit-aware (`fmtNum`/`parseUnit`); subdivisions is a plain integer. Wired in `bindDocProps()` (`index.html:3974`) via `commitProject()` (non-undoable, autosaved) and populated in `updateProps()`.
- `normalizeProject()` (`index.html:1257`) now clamps `grid.spacing`/`grid.subdivisions`/`grid.baseline` on load, so a hand-edited or corrupt file can't inject `NaN`/huge values.

### 23.2 Guide snapping
- New helpers (`index.html:1460`/`1471`): `nearestTarget(v, targets, tol)` and `guideTargets(p, axis)` (content-box edges + column edges for one axis).
- `snap(v, axis)` (`index.html:1488`): computes the grid value first, then — only if `axis` is given and `project.guides.snapToGuides` is on — returns the nearest guide within `GUIDE_SNAP_TOL = 6` px, else the grid value. `snap(v)` (no axis) is unchanged grid snapping.
- Call sites updated (`index.html:4899-4903` endpoint drag, `4955-4961` multi-select drag, `5005-5007` resize). For resize, `axis` is passed **only for the edge that moves** (`dir` contains `w`/`n`); the static edge keeps its old grid snap and width/height stay grid-only — so an east/south resize can never shift the opposite edge.

### 23.3 Behaviour change & rollback
- With `snapToGuides` on (the default), dragging near a margin or column edge now snaps to it. Turning the **Snap** checkbox off makes `snap(v, axis)` identical to the old `snap(v)`, so the whole feature rolls back with one toggle (and the code path with one argument).
- **Pre-existing quirk preserved:** a resize still grid-snaps the *static* edge (as it did before), so an off-grid element can shift slightly on the opposite side. Fixing it was out of scope for a snapping phase; noted as a follow-up.

### 23.4 Read-only guards
All four new grid/baseline controls are guarded (`if (state.readOnly) return;`). `state.readOnly` references: 43 → **47**.

### 23.5 Verification performed
- `node --check` on both inline scripts: clean.
- 37 new harness assertions, all pass: helper defaults/fallbacks/clamps; `snap()` grid-only when axis omitted or `snapToGuides` off; guide-preferred at the left/right/top margin and at a column edge; grid when no guide is near; `renderGrid()` major/minor/baseline on/off; panel population and round-trip (incl. `0.5 in → 48 px`); read-only blocks spacing and baseline; `normalizeProject()` clamps.
- Phase 1/2 suites unchanged (29/29, 21/21); all six load modes `errors=0`; the Phase 3 suite also passes under every load mode.

### 23.6 Known limitations / follow-ups
- Baseline grid is **overlay-only** — text/line-height is not snapped to it (as planned in §8).
- No manual browser pass yet (the harness is headless); a visual check of the baseline colour and guide-snap feel is recommended.
- The pre-existing resize static-edge quirk in §23.3.

---

## 24. As-Built Notes — Phase 4 (implemented)

All three Phase 4 sub-items are implemented in `index.html` (uncommitted working tree), with no new dependencies.

### 24.1 Data model — `project.export.includeBleed`
- `PROJECT_DEFAULTS` (`index.html:1175-1206`, export line 1204) gained `export: { includeBleed: false }`.
- `normalizeProject()` (`index.html:1257`, merge at 1274) merges it (`pr.export = shallowMerge(base.export, pr.export || {})`) and coerces with `!!pr.export.includeBleed`, so v1 files and partial objects default to `false`.
- It is **project-level** (non-undoable, autosaved via `markProjectDirty()`), consistent with the §8 page-vs-project table.

### 24.2 HTML/PDF export (closes §10.1, §10.2)
`exportHTML()` was split into three helpers so the same element markup serves HTML and raster:
- `exportBox(pg)` (`index.html:2213`) — returns `{bleed, off, pw, ph, ow, oh}`; `bleed` is 0 unless `project.export.includeBleed`, and `off === bleed`.
- `exportElementsHTML(pg, off)` (`index.html:2237`) — the per-element loop, now with `+off` applied to every `x`/`y` (and to the line's `minX`/`minY`, with the `<line>` endpoints recomputed as `el.x - minX + off`).
- `cropMarksHTML(off, pw, ph)` (`index.html:2224`) — eight 1px ticks at the trim corners, only when `off >= 4`, `len = min(off, 12)`.
- `exportHTML()` (`index.html:2280`) now emits `@page{size:<ow>in <oh>in;margin:0;}` (via `fromPx`, 4-decimal) for the **first** page, `html,body{margin:0;padding:0}`, a grey proof backdrop that becomes `none` under `@media print`, per-page `background:<pg.background>`, `print-color-adjust:exact`, and the bleed-expanded page box + crop marks when enabled. Guides/grid were never referenced and remain absent (verified).

**Honest limitation (unchanged from §10.2):** this is a correct-looking proof, not a print-shop PDF with real trim/bleed boxes — browsers cannot set those. Multi-size documents still get a single `@page` size (browsers honour only the first).

`exportPDF()` now builds a real `.pdf` file instead of opening the print dialog: for each page it builds the same DOM stage (`buildExportStage()`) the HTML export uses, rasterises it with html2canvas (`scale: 2`), and adds it to a jsPDF document (`new jsPDF(orient,'px',[ow,oh])`, `addPage` for subsequent pages), then `pdf.save(<project>.pdf)`. The vendor libraries live in `vendor/html2canvas.min.js` and `vendor/jspdf.umd.min.js` and are loaded via `<script>` tags — the same stack the Diagram project uses. The `@page`/print path now exists only for the standalone HTML export file.

### 24.3 Raster PNG/JPEG (closes §10.3)
`exportRaster(format)` now uses html2canvas (the SVG `<foreignObject>` path was removed):
1. Reads `dpi` from `project.dpi`, computes `scale = dpi / PX_PER_IN`, `W = round(pw*scale)`, `H = round(ph*scale)`; bails with a toast above 80 MPx.
2. Builds a live stage with `buildExportStage()` — the exact same markup + `exportElementCSS()` the HTML/PDF export uses.
3. Calls `html2canvas(host, {scale, backgroundColor, width: pw, height: ph})` and downloads the resulting canvas via `canvas.toBlob()` (fallback `toDataURL`), named `<project>-p<page>.<png|jpg>`.

Shared helpers keep the three exporters from drifting: `exportElementCSS()` is the single source of the export stylesheet, `buildExportStage(pg, box)` is the single DOM builder, and `solidBg(pg)` forces an opaque background for JPEG/PDF (JPEG has no alpha channel).

Caveats surfaced as toasts: cross-origin images can taint the canvas (`toBlob` throws → caught), and html2canvas cannot reproduce every CSS feature (e.g. multi-column flow/hyphenation) exactly as the browser's own layout does.

The toolbar has a single **Export** button (`#btn-export`, `.tool-btn.export-btn` — the same 18px download icon and accent square the Diagram project uses, placed right after the New/Save/Import group) that opens a dropdown (`#export-drop`, `.export-item` rows) with four format options — PDF (all pages, jsPDF download), PNG and JPG (current page at project DPI), and HTML (standalone print file). This mirrors the Diagram project's export menu. The menu is `position:fixed` so the toolbar's `overflow-x:auto` cannot clip it; `bindExportMenu()` positions it at `br.left + br.width - 200` / `br.bottom` (the Diagram formula), toggles `.visible`/`aria-expanded`, closes on outside-click and Escape, and dispatches each item to `exportPDF()`/`exportRaster()`/`exportHTMLFile()`. Export is not read-only-guarded (consistent with the previous export buttons).

### 24.4 Image PPI warning (closes §10.3 last paragraph)
- `imagePPI(el)` (`index.html:2334`) = `naturalWidth / (el.w / PX_PER_IN)`, or `null` when the element is not an image or the rendered `<img>` is not found. **Never affects geometry** (§4).
- `updateProps()`'s image branch (`index.html:2848`) writes `Effective <n> PPI` into `#prop-image-ppi`, amber (`#D97706`) when below `project.dpi`, muted otherwise; hidden for non-images.

### 24.5 UI additions
- `#prop-doc-exportbleed` checkbox in the Document panel's *Bleed & Slug* section (markup at `index.html:997`); `bindDocProps()` handler calls `commitProject(function(pr){ pr.export.includeBleed = on; })` behind a `state.readOnly` guard, and `updateProps()` populates it.
- `#prop-image-ppi` line under the Image panel's "Change Image" button (markup at `index.html:1009`).

### 24.6 Verification performed
- `node --check` on all inline scripts: clean.
- Harness suite `phase4`: **26 assertions, all pass** — `exportBox` off/on, crop-mark count/guard, per-element `+off` (text/line/`<line>` endpoints), `@page` inches (850px → 8.8542in; with 9px bleed → 9.0417in), page background, no guides/grid, `print-color-adjust`, panel round-trip + read-only block, `imagePPI` null cases, `normalizeProject` export defaults/coercion, and `exportPDF`/`exportRaster` not throwing.
- **Real-DOM integration test** (jsdom): `exportHTML` parses as HTML (rich text, drop-cap class, `dir`, `rgba()` stroke, image `src` preserved); bleed output parses with 8 crop marks.
- **Headless-Chromium smoke test** (after the html2canvas switch): opening `#btn-export` shows all four `.export-item`s, positions the menu under the button, sets `aria-expanded`, and closes on outside-click. Choosing the PNG/JPG items renders an 850×1100 stage at scale 3.125 (300 DPI → 2656×3437 canvas) and downloads an `image/png` / `image/jpeg` blob; the PDF item renders at scale 2 (1700×2200), creates `new jsPDF('p','px',[850,1100])`, calls `addImage(...)` and `save(...)`; the HTML item downloads a `text/html` blob. No console errors.
- Phase 1/2/3 suites unchanged (29/29, 21/21, 37/37); all six load modes `errors=0`, `buildV=2`.
- `state.readOnly` reference count: 47 → **48**.

### 24.7 Known limitations / follow-ups
- Bleed export is a **proof**, not a press-ready PDF (no real TrimBox/BleedBox) — bleed is drawn as an expanded page plus crop marks in the raster.
- Raster/PDF fidelity depends on html2canvas: cross-origin images and advanced CSS (multi-column, hyphenation) are the usual failure modes; the HTML export (browser-native layout) remains the pixel-perfect fallback.
- No manual browser pass yet (harness + jsdom are headless); a visual check of crop-mark weight and raster output in a real browser is recommended.
- `@page` is single-size; multi-size documents print at the first page's size.

---

## 25. As-Built Notes — Image insert aspect ratio (follow-up)

> A separate, small follow-up (not part of Phases 0–4). Reported: "The image is set to be square, but it needs to start at the same aspect ratio of the image that is added."

### 25.1 Problem

`onImageInsert()` created every inserted image with a hardcoded box:

```js
var el = addElement('image', { src: ev.target.result, x: 100, y: 100, w: 300, h: 300 });
```

So a 4000×1000 photo and a 500×2000 portrait both landed as a 300×300 square and were stretched by the renderer. The image element **is** the frame here (there is no separate frame + fit-options model as in InDesign), so the element must start at the source aspect ratio.

### 25.2 Fix (`index.html:5311-5345`, with `naturalImageSize()` at `index.html:2356`)

```js
/* Longest side (px) an inserted image is scaled down to fit. */
var IMAGE_INSERT_MAX = 300;

/* Fit natural pixel dimensions into an IMAGE_INSERT_MAX-px box while preserving
   the source aspect ratio. Images smaller than the box keep their natural size
   (no blurry upscaling). */
function fitImageSize(nw, nh) {
  nw = Math.max(1, Math.round(nw) || 1);
  nh = Math.max(1, Math.round(nh) || 1);
  var s = Math.min(1, IMAGE_INSERT_MAX / Math.max(nw, nh));
  return { w: Math.max(1, Math.round(nw * s)), h: Math.max(1, Math.round(nh * s)) };
}
```

`onImageInsert()` now probes the decoded data URL with a throwaway `Image()` and only then calls `addElement()`:

```js
reader.onload = function(ev) {
  var src = ev.target.result;
  var probe = new Image();
  probe.onload = function() {
    var d = fitImageSize(probe.naturalWidth || probe.width, probe.naturalHeight || probe.height);
    var el = addElement('image', { src: src, x: 100, y: 100, w: d.w, h: d.h });
    select(el.id, false);
    setTool('select');
  };
  probe.onerror = function() { showToast('Failed to load image. The file may be corrupted.', 'error'); };
  probe.src = src;
};
```

### 25.3 Behavior

| Source | Inserted box |
|---|---|
| 4000×1000 (4:1) | 300×75 |
| 400×200 (2:1) | 300×150 |
| 850×1100 (portrait) | 232×300 |
| 500×500 (square) | 300×300 |
| 100×250 (small portrait) | 100×250 (natural size — no upscaling) |
| 60×60 (icon) | 60×60 (natural size) |

Rules: the longest side is capped at `IMAGE_INSERT_MAX` (300 px); smaller images keep their natural pixel size; the aspect ratio is always preserved; the result is never smaller than 1×1. Position stays at `(100, 100)` and the element is still selected on insert.

### 25.4 Verification

- Harness `aspect` suite — 12 assertions on `fitImageSize()` (landscape/portrait/square/panorama/tall/small, rounding, degenerate `0×0`, the 300-px cap): **12/12 pass**.
- Real headless Chromium end-to-end: PNGs of 400×200, 100×250, and 60×60 were turned into `File` objects, pushed through `onImageInsert()` with a real `FileReader`, and the resulting elements measured as **300×150, 100×250, 60×60** respectively (`added: true` for each, zero JS errors).

### 25.5 Scope / not changed

- Resizing an image element after insert is unaffected by §25 (free resize still allowed) — the aspect lock arrived later, in §26.
- No new data fields in §25; `imagePPI()` (§24) continues to work because it reads the rendered `<img>`'s `naturalWidth`.
- "Change Image" behaviour was left alone by §25 and was later made aspect-aware by §26.4 (only when the element is locked).

---

## 26. As-Built Notes — Image aspect-ratio lock (follow-up)

> Follow-up to §25. Requested: "Add a new 'preserve aspect ratio' checkbox for the image props to lock in the aspect ratio on images."

### 26.1 Data model

One new boolean on **image elements only**: `lockAspect`.

| Where | Value |
|---|---|
| `mkEl('image', …)` (`index.html:1719`) | defaults to `true` (matches §25's "start at the source aspect ratio" intent); honours an explicit `o.lockAspect` |
| `mkEl('text'/'rect'/'line', …)` | absent (`undefined`) — the lock is image-only |
| `normalizeProject()` (`index.html:1296`) | backfills `lockAspect = true` on image elements that lack it, leaves other types untouched, and preserves an explicit `false` |

`lockAspect` is a **resize constraint only** — it is never read by `domEl()`, `exportHTML()`, `exportElementsHTML()`, or `exportRaster()`, so it cannot change rendered output or existing geometry. Old documents therefore still open with zero visual/positional change; only future resize interactions differ.

### 26.2 UI (`index.html:1009`)

One row in the existing Image panel, using the same markup pattern as the Document-panel checkboxes:

```html
<div class="prop-row"><label style="flex:1"><input type="checkbox" id="prop-image-lockaspect" checked> Preserve aspect ratio</label></div>
```

- Populated by `updateProps()` (`index.html:2846`): `lockEl.checked = el.type === 'image' && el.lockAspect !== false`.
- Wired in `bindProps()` (`index.html:3220`) with a `change` listener that starts with `if (state.readOnly) return;` and writes `lockAspect` to every selected **image** element (non-images in a mixed selection are skipped).

### 26.3 Resize (`onPointerMove()`, `index.html:4853`, block at 5008-5027)

The drag-resize block already computed `nw`/`nh` from the handle direction and snapped them. It now applies the constraint after snapping:

```js
if (r.lockAspect || e.shiftKey) {
  var ratio = r.h > 0 ? r.w / r.h : 1;
  var hasX = dir.indexOf('e') !== -1 || dir.indexOf('w') !== -1;
  var hasY = dir.indexOf('s') !== -1 || dir.indexOf('n') !== -1;
  if (hasX && hasY) {
    if (Math.abs(sw - r.w) >= Math.abs(sh - r.h)) sh = Math.round(sw / ratio);
    else sw = Math.round(sh * ratio);
  } else if (hasX) { sh = Math.round(sw / ratio); }
  else { sw = Math.round(sh * ratio); }
  sw = Math.max(1, sw); sh = Math.max(1, sh);
  if (dir.indexOf('w') !== -1) sx = r.x + r.w - sw;   /* keep the opposite edge fixed */
  if (dir.indexOf('n') !== -1) sy = r.y + r.h - sh;
}
```

Details:
- `resizeOrig` (`index.html:4785`) now carries `lockAspect` so the flag is read from the pre-drag snapshot.
- **Corner handles**: the axis the pointer moved furthest drives the other, so the element tracks the cursor on the dominant axis.
- **Edge handles** (e.g. `e`): the perpendicular dimension follows; the element is anchored at the top-left of its other axis (the N/W anchors above still apply when the handle has those components).
- The **ratio is applied after snapping** and the derived dimension is *not* snapped, so grid snap can't break the ratio. The derived side is floored at 1 px.
- **Holding Shift constrains any element**, locked or not — the conventional editor behaviour, and consistent with Shift already meaning "constrain" for the line tool.

### 26.4 Width/Height inputs (`bindProps()`, coupling at `index.html:2982`)

Typing into `#prop-w` or `#prop-h` also honours the lock. Ratios are captured *before* the edit (so the first value isn't already overwritten), then the other dimension is derived per element. Multi-select with mixed ratios is handled per element; unlocked images and non-images are untouched.

### 26.5 "Change Image" is now aspect-aware (`index.html:3240`)

Two changes to the `#image-file-input` handler:
1. It probes the replacement with an `Image()` and, **when the element is locked**, refits the box to the new image's ratio keeping the element's current longest side (`box = max(w,h)`; `s = box / max(nw,nh)`). When unlocked, the old behaviour is kept (box unchanged). *(The refit maths was later factored into the shared `naturalFitBox()` helper — §27.3.)*
2. It now calls `syncElementDOM(id)` + `updateProps()`. This fixed a pre-existing bug: the handler only called `updEl()`, and `syncElementDOM()` never wrote `src`, so **"Change Image" previously updated the data model but not the canvas `<img>`** until the next full rebuild. `syncElementDOM()` now syncs `src` for image elements.

### 26.6 Verification

- **Real headless Chromium, 26 assertions across two probes, all pass.**
  - *Resize / inputs / migration (20)*: `mkEl` defaults (`true` for images, explicit `false` honoured, absent on text); `normalizeProject` migration (backfills `true`, preserves `false`); Image panel shown + checkbox reflects state both ways; `prop-w`→`h` and `prop-h`→`w` coupling while locked (500×250, 200×100); simulated pointer drags through the real `#canvas-container` handlers — `se` locked (200×100 → 500×250, ratio exactly 2), `se` unlocked (200×100 → 500×150, both axes free), `se` + Shift unlocked (ratio 2), `nw` locked (SE corner held at 400,250, ratio 2); read-only blocks the checkbox.
  - *Change Image (6)*: a real `File` (built from a PNG data URL, pushed through `DataTransfer` into `#image-file-input`) — locked adopts the new ratio (300×150 → 120×300 for a 100×250 PNG) and keeps the longest side at 300; `src` updated in both the model and the canvas `<img>` **immediately**; unlocked keeps the 300×150 box; read-only blocks the whole path.
  - Zero JS errors in either probe (`window.onerror` + `unhandledrejection` collectors).
- **Harness `lockaspect` suite** (node, no DOM) — 14 assertions on defaults and `normalizeProject` migration: **14/14 pass**.
- **Regressions**: `phase1` 29/29, `phase2` 21/21, `phase3` 37/37, `phase4` 26/26, `marginband` 13/13, `aspect` 12/12, jsdom 25/25; all six load modes `errors: []`; `node --check` clean. `state.readOnly` reference count 49 → 50.
- **Incidental cleanup**: four stray `console.log('[DEBUG] …')` lines (two in `select()`, two in `onPointerDown()`) that were present only in the uncommitted working tree — `HEAD` has none — were removed. No behavioural change; `select()` and the pointer paths are otherwise identical.

### 26.7 Known limitations / notes

- The lock governs **resize and W/H entry**, not rotation or radius; those are independent properties.
- Corner-resize picks the dominant axis; a diagonal drag therefore snaps to the pointer's larger movement rather than following it exactly on both axes.
- Rotating an element then resizing uses the unrotated box (pre-existing behaviour, unchanged).
- Checkboxes inside `.prop-row` inherit a broad rule (`.prop-row label input{flex:1;min-height:44px}`, `index.html:369`) that stretches their layout box. Chrome draws `appearance:auto` checkboxes natively, so the painted control stays normal-sized; the new checkbox uses the exact same markup pattern as the existing `#prop-doc-*` checkboxes, so it is consistent with them. Tidying that shared rule is a separate, deliberately-avoided change here.

---

## 27. As-Built Notes — Reset to original aspect ratio (follow-up)

> Follow-up to §26. Requested: "Add in a button to reset to the original aspect ratio."

### 27.1 UI (`index.html:1009`)

One button in the Image panel, directly under the "Preserve aspect ratio" checkbox, using the same `small-btn` + `prop-row` pattern as "Change Image" and "Apply size to all pages":

```html
<div class="prop-row"><button class="small-btn" id="prop-image-reset-ratio" style="flex:1" title="Resize the element box back to the source image's aspect ratio">Reset to original ratio</button></div>
```

Renders full-width (215×44 in the test viewport), no text overflow, and inherits the identical font/padding of "Change Image". It lives inside `#prop-image-section`, so it is only reachable when an image element is selected.

### 27.2 Behaviour

`bindProps()` handler (`index.html:3225-3254`):

1. `if (state.readOnly) return;` — guarded like every other mutating control.
2. Filters `state.selectedElements` down to image elements (multi-select works; non-images are skipped).
3. For each, resolves the **natural pixel size** via `naturalImageSize()` (§27.3) and computes the target box via `naturalFitBox()`.
4. If the box already matches, nothing is written. Otherwise `updEl()` + `syncElementDOM()`.
5. When all callbacks finish: an error toast if any image's dimensions could not be read, a neutral "Already at the original aspect ratio." toast if nothing needed changing, and — only when something *did* change — `saveSnapshot()` then `emit()`.

Deliberate choices:

- **Geometry only.** The button does *not* flip `lockAspect`. If the lock is off, the shape is corrected once and stays unlocked (the user's checkbox state is never silently rewritten); if the lock is on, the corrected shape is then preserved by the existing §26 resize rules.
- **Keeps the current longest side**, the same rule as insert (§25) and Change Image (§26.5). This bounds the change and makes the three paths produce identical results for identical inputs.
- **Undoable.** Unlike the W/H number inputs (a pre-existing gap), this is a discrete click action, so it follows the `removeElement()`/drag convention of `mutate → saveSnapshot() → emit()`. `undo()` restores the pre-reset box.
- **Async-tolerant.** `naturalImageSize()` reads `naturalWidth`/`naturalHeight` off the already-rendered `<img>` when it has decoded (the normal case, fully synchronous), and otherwise probes `el.src` with a throwaway `Image()`. Both paths converge on the same callback.

### 27.3 Shared helpers (`index.html:2347` / `2356`)

```js
/* Box that matches the source image's natural aspect ratio, keeping the element's
   current longest side (same rule as insert §25 and Change Image §26.5). */
function naturalFitBox(el, nw, nh) {
  if (!el || !(nw > 0) || !(nh > 0)) return null;
  var box = Math.max(1, Math.max(el.w || 1, el.h || 1));
  var s = box / Math.max(nw, nh);
  return { w: Math.max(1, Math.round(nw * s)), h: Math.max(1, Math.round(nh * s)) };
}

/* Natural pixel size: from the decoded <img> when possible, else probe `src`.
   Always calls cb(w, h) — cb(0, 0) on failure. */
function naturalImageSize(id, src, cb) { … }
```

`naturalFitBox()` is pure (no DOM, no state) and is now the single source of truth for the ratio refit — the Change Image handler (§26.5, `index.html:3240`) was refactored to call it instead of duplicating the maths. That refactor is covered by the existing Change Image assertions (§26.6).

### 27.4 Verification

- **Real headless Chromium, 22 assertions, all pass**, zero JS errors (`window.onerror` + `unhandledrejection` collectors):
  - button exists, label, visible only in the Image panel;
  - distorted 300×300 → **300×150** for a 400×200 source, longest side kept, DOM box updated, no toast;
  - clicking again is a no-op **and** shows "Already at the original aspect ratio.";
  - `undo()` after a reset restores the pre-reset 300×300 box;
  - multi-select: tall source 300×150 → **120×300**, already-square element untouched;
  - broken `src` → box unchanged + "Could not read the image dimensions." error toast;
  - read-only blocks the button entirely;
  - lock **off** → geometry corrected, `lockAspect` still `false`;
  - lock **on** → a subsequent `se` resize still holds the ratio (200×100 → 500×250);
  - Change Image regression after the `naturalFitBox()` refactor (300×150 → 120×300, `src` synced).
- **Harness `resetratio` suite** (node, no DOM) — 16 pure-function assertions on `naturalFitBox()`: distortion fixes, already-correct no-ops, tall/wide/extreme ratios, `null`/zero/negative natural sizes, degenerate and 1-px boxes, and idempotency: **16/16 pass**.
- **Regressions**: `phase1` 29/29, `phase2` 21/21, `phase3` 37/37, `phase4` 26/26, `marginband` 13/13, `aspect` 12/12, `lockaspect` 14/14, jsdom 25/25; all six load modes `errors: []`; `node --check` clean. `state.readOnly` reference count 50 → 51.

### 27.5 Known limitations / notes

- "Original" means the **currently loaded image's** natural size. After a "Change Image", resetting restores the *replacement's* ratio, not the ratio of the image that was originally inserted. There is no separate record of the first-inserted image.
- The longest-side rule can grow one axis noticeably (a 100×600 box on a square source becomes 600×600). This is the same trade-off §25/§26.5 already made; a "keep width" variant would need its own button to stay unambiguous.
- `object-fit: cover` means a distorted box crops rather than stretches, so the button is really "stop cropping, show the whole image".
- Rotation is ignored (the unrotated `w`/`h` are used), consistent with §26.7.
- Broken/unloadable `src` values can only be detected asynchronously, so the error toast appears a moment after the click.

---

## 28. As-Built Notes — Snapping master switch (follow-up)

> Requested: "We need a snap button like the `/Users/bflbarlow/Websites/diagram` project where it can be turned on to snap and turned off to have more free control."

Phase 3 (§23) made the grid project-configurable and added guide snapping, but snapping itself was **always on** — the only way to stop it was to set grid spacing to a value that happened to suit, or turn `snapToGuides` off (which only disabled the guide half). This adds the missing master switch.

### 28.1 Toolbar button (`index.html:842`)

A toggle button immediately after `#grid-toggle` in the same toolbar group, using the diagram project's icon (four corner dots + dashed alignment edges) so it reads the same in both apps:

```html
<button class="tool-btn active" id="snap-toggle" aria-label="Toggle snapping" aria-pressed="true" title="Toggle Snapping (S)"><svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">…</svg></button>
```

It reuses the existing `.tool-btn` / `.tool-btn.active` styling (`index.html:161-169`), so the on-state is the accent-filled button already used by `#grid-toggle`, the select tool and the theme toggle. `aria-pressed` mirrors the visual state for screen readers. Starts `active` in the markup so there is no flash before `init()` runs.

### 28.2 State and the gate (`index.html:1421`, `1488`)

```js
var state = { …, snapEnabled: true, … };

function snap(v, axis) {
  if (!state.snapEnabled) return Math.round(v);   /* §28 */
  var gs = gridSpacing();
  …
}
```

One early return covers **every** snap site, because all of them already funnel through `snap()` — endpoint drag (`4899-4903`), element drag (`4955-4961`) and resize (`5005-5007`). Nothing else in the codebase computes a snapped coordinate.

With the switch off the value is still rounded to a whole CSS px. That is deliberate: geometry is stored in canonical integer px everywhere (§3), sub-pixel positions would introduce float drift into the model and into `pw`/`ph`-relative maths, and 1px is far finer than any grid the user would have had on. So "off" means *free positioning*, not *float positioning*.

### 28.3 Toggle function (`index.html:1502`)

```js
function setSnapEnabled(on, quiet) {
  state.snapEnabled = !!on;
  var b = $('snap-toggle');
  if (b) {
    b.classList.toggle('active', state.snapEnabled);
    b.setAttribute('aria-pressed', state.snapEnabled ? 'true' : 'false');
  }
  localStorage.setItem('lp-snap', state.snapEnabled ? '1' : '0');
  if (!quiet) showToast('Snapping ' + (state.snapEnabled ? 'on' : 'off'));
}
```

`quiet` is used by the `init()` restore (`index.html:5646`) so loading a page does not fire a toast.

The toast matters more here than for `#grid-toggle`: grid visibility is self-evident, whereas snapping is invisible until you drag something. The toast is the only immediate confirmation that the click registered.

### 28.4 Keyboard shortcut (`index.html:5115-5124`)

`S` toggles it, matching the diagram project. The handler is placed **before** `if (state.readOnly) return;` so the shortcut works in read-only mode exactly like the toolbar button (snapping is a view preference, not a document edit). It bails when inline-editing or when focus is in an `INPUT`/`TEXTAREA`/`SELECT`/contenteditable so typing `s` into the property panel never toggles anything. `v`/`t`/`l` (tools) already occupy the other letter keys.

### 28.5 Scope: view preference, not document data

This is the key design decision. Snapping is stored in `state.snapEnabled` and `localStorage['lp-snap']`, **not** in `state.project`, following the existing `state.showGrid` / `lp-show-grid` precedent (`index.html:3743`).

| Consequence | Why it is correct |
|---|---|
| Not written by `buildProjectData()`, so the saved format is unchanged | No `v:3` bump, no migration, no `normalizeProject()` change |
| Not on the undo stack | Undo should not replay "you turned snapping off" |
| Does not call `markProjectDirty()` | Toggling does not make the document unsaved |
| Works in read-only mode | Another tab editing the file does not stop this tab from choosing how to drag |
| Survives reloads per browser | Matches the grid toggle |

`state.project.guides.snapToGuides` keeps its own meaning and is untouched: the master switch says *whether anything snaps*, the project checkbox says *whether guides participate*. With the master on and `snapToGuides` off you get grid-only snapping; with the master off neither applies.

### 28.6 Incidental fix — toolbar overflow (`index.html:155-156`)

Adding a 7th button to the zoom group pushed the toolbar's natural width from **1435px to 1495px**. The toolbar had no overflow handling, so between ~1435px and ~1494px viewport width the last button (`#jpeg-btn`) rendered past the right edge at `right=1483` — visible to `getBoundingClientRect()` but off-screen and unclickable. 1440px is an extremely common laptop width, so this was a real regression.

```css
.toolbar{…;overflow-x:auto;overflow-y:hidden;scrollbar-width:none;-ms-overflow-style:none}
.toolbar::-webkit-scrollbar{display:none}
```

The toolbar now scrolls instead of clipping. The scrollbar is hidden so the fixed `--lp-toolbar-h: 52px` height is preserved (a visible scrollbar would steal ~10px and squash the 44px buttons); `overflow-y:hidden` is required because `overflow-x:auto` with `overflow-y:visible` is invalid and browsers coerce it to `auto`. Trackpad / shift-wheel scrolling and Tab-focus both reach the buttons — browsers scroll a focused child into view automatically.

Measured: at 1680px nothing changes (no scrollbar, `scrollHeight === clientHeight`, no vertical overflow); at 1440px and 1280px the toolbar scrolls and `#jpeg-btn` becomes reachable. This also fixes the pre-existing clipping below 1435px.

### 28.7 Verification

- **Harness `snaptoggle` suite** (node, real `state`/`snap()`/`setSnapEnabled()` with the fake DOM + stubbed `localStorage`) — **33/33 pass**:
  - default on; `snap(23)`→20, `snap(23.6)`→20, `snap(50,'x')`→48 (guide), `snap(803,'x')`→802, `snap(103,'y')`→100;
  - off: `snap(23)`→23, `snap(23.4)`→23, `snap(23.6)`→24, `snap(50,'x')`→50, `snap(803,'x')`→803, `snap(103,'y')`→103, `snap(4)`→4, `snap(5)`→5;
  - axis invariance with the switch off: `snap(v,'x') === snap(v)` and `snap(v,'y') === snap(v)`;
  - custom grid 25 ignored while off (`snap(30)`→30) and applied when back on (`snap(30)`→25, `snap(38)`→50);
  - `snapToGuides` still honoured with the master on;
  - coercion `setSnapEnabled(0/1/null)`;
  - **no undo snapshot** and **no dirty flag** from toggling;
  - `snapEnabled` absent from `buildProjectData()` output while `project.guides.snapToGuides` is still present.
- **Real headless Chromium `p4` probe — 41/41 pass**, zero JS errors: markup position (`gt.nextElementSibling === btn`), icon/viewBox, label + title, initial `active` + `aria-pressed`, 44×44 render; drag and `se`-resize land on the grid when on (100→120) and land exactly when off (→123/117); a guide at margin 48 is ignored when off (→46); `S` and `s` both toggle; `S` is ignored while focus is in an input; the toolbar button and the `S` shortcut both work under `setReadOnly(true)`; no undo/dirty side effects.
- **`p5` probe (localStorage pre-seeded to `lp-snap=0`) — 9/9 pass**: the button restores off with `aria-pressed="false"`, no toast on load, the grid toggle stays independently off, `#grid-layer` stays hidden, free drag still works, and the header has no horizontal overflow.
- **Screenshot pixel check** (PIL, dark theme): snap button centre `(37,99,235)` = `--color-accent` when on, `(23,23,26)` = toolbar background when off, with the icon strokes present in both states (58 / 40 light pixels in the 44×44 crop).
- **Regressions**: `phase1` 29/29, `phase2` 21/21, `phase3` 37/37, `phase4` 26/26, `marginband` 13/13, `aspect` 12/12, `lockaspect` 14/14, `resetratio` 16/16, jsdom 25/25; browser probes p1 20/20, p2 6/6, p3 22/22; all six load modes `errors: []`; `node --check` clean. (`phase3`'s snap assertions still pass unchanged because the default is on — the rollback path is `state.snapEnabled === true`.)

### 28.8 Known limitations / notes

- Snapping is a per-browser preference, so it is not carried with an exported/imported project and not shared between machines. This matches the grid toggle; making it project data would mean a format bump for something that is really an interaction preference.
- Arrow-key nudging already moves by 1px / 10px (`CONFIG.arrowStep` / `arrowStepBig`) and does **not** consult `snap()` — unlike the diagram project, where the snap flag also changes the arrow step. Left as-is: 1px is finer than any usable grid, and changing it would alter existing keyboard behaviour.
- The toolbar is now a horizontal scroll container. Any future toolbar dropdown/popover would need `position:fixed` or a portal, since `overflow` clips absolutely-positioned descendants.
- In read-only mode the button is dimmed by `body.read-only-mode .toolbar button:not(…)` (`index.html:752-758`) yet remains clickable. That is pre-existing behaviour shared with `#grid-toggle`, not new here.
- No "snap" indicator on the canvas itself (e.g. a highlighted guide line while snapped) — the §23 guide overlay is static. Out of scope for this request.

---

## 29. As-Built Notes — Unit-aware grid step (follow-up)

### 29.1 The symptom

The document unit was threaded through the *value fields* and the *ruler*, but not the
grid **step**. Switching the unit re-labelled the Spacing field (10 px → `0.104` in →
`2.6` mm) while `grid.spacing` stayed at `10` px, so the drawn lines never landed on a
ruler graduation:

| Units | Ruler majors | Grid lines (before) | Spacing field (before) |
|-------|--------------|---------------------|------------------------|
| px    | every 100 px | 0, 10, 20, …        | `10`   |
| in    | every 1 in (96 px)  | 0, 10, 20, … (0.104 in each) | `0.104` |
| mm    | every 10 mm (37.8 px)| 0, 10, 20, … (2.6 mm each)   | `2.6`   |

The field *converted*, so the grid looked unit-aware in the panel, but on the canvas no
grid line ever coincided with an inch / millimetre mark — and the ruler minor ticks
(which follow `gridSpacing()`, `index.html:3851`) inherited the same off-unit step.

### 29.2 Design decision

**When the document unit changes, re-express the grid step as the nearest *clean value
in the new unit*.** That is the only moment the user has expressed intent to change the
measurement system, so it is the only moment we touch the stored grid. In particular:

- **Not on load.** `normalizeProject()` is untouched, so every saved document opens with
  exactly the grid it was saved with (§9's migration rule still holds byte-for-byte).
- **`subdivisions` is not modified.** Majors remain `spacing × subdivisions`, as in §23.
  Auto-selecting a subdivision count would silently rewrite a user setting, and would do
  so inconsistently on repeated unit switches — see §29.6.
- **The canonical-px data model is unchanged.** `grid.spacing` / `grid.baseline` are still
  plain canonical px; the ladder is a display/IO-only conversion layer exactly like §3.

### 29.3 As-built

Added to the grid-helper block, immediately after `baselineSpacing()`:

| Symbol | Line | Purpose |
|--------|------|---------|
| `GRID_UNIT_LADDER` | `index.html:1465-1473` | Per-unit ladder of "clean" steps, in unit values. |
| `snapLenToUnit(px, unit)` | `index.html:1476-1487` | Canonical px → nearest ladder value in `unit` → canonical px. `0` (and any non-positive / `NaN`) returns `0`, so a disabled baseline stays disabled. Falls back to the px ladder for an unknown unit. |
| `r2(n)` | `index.html:1490` | 2-decimal rounding. |

The `change` handler for `#prop-doc-units` (`index.html:4089-4102`) now reads:

```js
var unitsSel = $('prop-doc-units');
if (unitsSel) unitsSel.addEventListener('change', function() {
  if (state.readOnly) return;
  var u = this.value;
  if (!UNITS[u]) u = 'px';
  commitProject(function(pr) {
    /* Re-express the grid step in the new unit so the grid overlay (and the
       ruler minor ticks that follow it) line up with the ruler graduations. */
    pr.grid.spacing = snapLenToUnit(parseFloat(pr.grid.spacing), u);
    pr.grid.baseline = snapLenToUnit(parseFloat(pr.grid.baseline), u);
    pr.units = u;
  });
});
```

Rounding was also threaded into the two renderers that accumulate a fractional step, so
metric grids do not leak float tails (`2.5 mm = 9.4488188…` px):

- `snap()` (`index.html:1521`) now returns `r2(Math.round(v / gs) * gs)`.
- `renderGrid()` (`index.html:1575`) writes `r2(x)` / `r2(y)`; the baseline pass
  (`index.html:1591-1595`) does the same.

`r2` is a **no-op for every integer-px grid** (the px default, and anything the pre-Phase-3
code produced), so nothing about the existing behaviour changes.

### 29.4 The ladders

```js
var GRID_UNIT_LADDER = {
  px: [1, 2, 5, 10, 20, 25, 50, 100, 200, 500],
  in: [1 / 64, 1 / 32, 1 / 16, 1 / 8, 1 / 4, 1 / 2, 1, 2, 4],
  cm: [0.1, 0.2, 0.25, 0.5, 1, 2, 5, 10],
  mm: [0.5, 1, 2, 2.5, 5, 10, 20, 50],
  pt: [0.5, 1, 2, 3, 4, 6, 12, 24, 36, 72],
  pc: [0.125, 0.25, 0.5, 1, 2, 6, 12]
};
```

Each ladder is built so the chosen step **divides the unit's ruler-major interval exactly**
where the physical ratio allows it, so the grid automatically lines up with the numbered
ruler marks:

| Unit → | 10 px becomes | px | Grid lines divide a ruler major |
|--------|---------------|----|--------------------------------|
| in | `1/8 in` | 12 px | `96 / 12 = 8` ✓ |
| mm | `2.5 mm` | 9.4488 px | `37.795 / 9.4488 = 4` ✓ |
| cm | `0.25 cm` | 9.4488 px | `37.795 / 9.4488 = 4` ✓ |
| pt | `6 pt` | 8 px | `16 / 8 = 2` ✓ |
| pc | `0.5 pc` | 8 px | `16 / 8 = 2` ✓ |
| px | `10 px` | 10 px | unchanged |

Selection is by smallest absolute difference in unit space. Notable convert-back cases:
`0.5 in` (48 px) stays 48 px, `0.25 in` (24 px) stays 24 px, and `10 px → in → px`
returns to `10 px` (12 px is not on the px ladder).

### 29.5 Worked example (verified in Chromium)

With `units = in` and `grid = 0.125`:

```
gridSpacing = 12
ruler minors = [0, 12, 24, 36, 48, 60, 72, 84, …]   ← identical to the grid lines
grid lines   = [0, 12, 24, 36, 48, 60, 72, 84, …]
ruler majors = [0, 96, 192, 288, …]                  ← 1 in marks
grid line at 96 = true                               ← 8th grid line = 1 in
```

A pixel scan of a headless screenshot at 1× confirms the rendered minor lines are exactly
**12 px** apart (previously 10 px), i.e. the change reaches the actual pixels, not just the
DOM.

### 29.6 Why `subdivisions` is deliberately left alone

The obvious extra step would be to also pick a `subdivisions` value that makes the *major*
lines coincide with the *number of* ruler majors (e.g. `8` for inches so majors land on
1 in). It was rejected for two reasons:

1. **It would override a user setting.** `subdivisions` is a first-class field in the
   Document panel; re-deriving it on a unit change is a silent side effect the user did
   not ask for.
2. **It cannot be made consistent.** If the rule were "align majors only while
   `subdivisions` is still the default 5", then `px → in` would set 8, and the next unit
   change (`in → mm`) would see a non-default 8 and *not* align — leaving majors at
   `2.5 mm × 8 = 20 mm` against a 10 mm ruler major. Either every unit change rewrites the
   setting or none does; leaving it alone is the smaller, predictable rule.

Consequence: in inches the darker (major) grid lines sit every `0.625 in` while the ruler
numbers are every `1 in`. The *minor* lines — the ones that matter for "is this grid in
inches?" — line up exactly. Aligning majors is a small, self-contained follow-up if it is
wanted; it would be `grid.subdivisions = rulerMajorPx(u) / snappedStep` on the same
handler.

### 29.7 Verification

- New harness suite **`gridunits` — 41/41 pass** (`node harness.js index.html none gridunits`):
  pure `snapLenToUnit` cases (all six units, `0`, negative, `NaN`, unknown unit,
  idempotence, exact division of the ruler major), `r2`, and integration through the real
  `change` handler (field text, `gridSpacing()`, `baselineSpacing()`, subdivisions
  untouched, rendered SVG containing `M96 0`, read-only ignored, px default invariant
  `10 × 5 = 50` preserved).
- Chromium probe: **16/16 pass**, `ERRORS=[]` — ruler minors byte-equal the grid lines in
  inches, grid and ruler major coincide at 37.8 px in millimetres, px restored to 10.
- Full regression: `phase1`…`phase4`, `marginband`, `aspect`, `lockaspect`, `resetratio`,
  `snaptoggle` all green; all six load modes (`none`/`v1`/`v2`/`empty`/`corrupt`/`nopages`)
  report `errors: []`; `jsdom` 25/25.

### 29.8 Known limitations / notes

- A document **saved before this change** with a non-px unit keeps its old px step until the
  unit selector is used again (there is no "clean up on load" by design, §29.2). Only
  pre-existing test documents are affected — the feature is unreleased.
- Because the selector's `change` event does not fire when the value is unchanged, an
  already-loaded stale document is fixed by choosing *any other* unit and back, not by
  re-selecting the same one.
- Metric/ladder steps are irrational in px, so snapped element coordinates can be
  fractional (e.g. `18.9`). `r2` bounds the precision to 2 decimals; sub-pixel CSS/SVG
  coordinates are valid and the exported HTML carries them through unchanged.
- The ladder is a heuristic: a deliberately odd step (e.g. `7 px`) is re-expressed on a
  unit change (`7 px → 0.0625 in`). That is intentional — a unit change re-derives the
  grid — but it is lossy, so it is not applied on load.
- Ruler **major** lines and grid **major** lines can differ (see §29.6).

---

## 30. As-Built Notes — Momentary snapping (hold Shift, follow-up)

> Requested: "When the snap button (master toggle `state.snapEnabled`) is **off**, holding **Shift** should temporarily enable snapping until Shift is released."

§28 added the master snapping switch, but it is a latched, per-browser preference: with it off there is no way to snap a single placement without first clicking the button (and then remembering to click it back). This follow-up adds a **momentary** override — hold Shift to snap while the key is down.

### 30.1 Design: a second, transient flag — not a toggle of the persisted one

The naive implementation ("on Shift keydown, call `setSnapEnabled(true)`; on keyup, call it again") is wrong: it would write `localStorage['lp-snap']`, flash the latched `.active` button, fire a toast on every keypress, and — worst of all — **lose the user's persisted choice** if the keyup were ever missed.

Instead the feature adds a second, purely transient flag:

```js
snapTemp: false,                 /* index.html:1425 — never persisted, never undone */
```

and the master gate now reads a small helper rather than the persisted flag directly:

```js
function snap(v, axis) {
  if (!snapActive()) return Math.round(v);   /* §30: was `!state.snapEnabled` */
  …
}

function snapActive() { return !!(state.snapEnabled || state.snapTemp); }
function setSnapTemp(on) {
  on = !!on;
  if (state.snapTemp === on) return;          /* idempotent — key repeat is a no-op */
  state.snapTemp = on;
  var b = $('snap-toggle');
  if (b) b.classList.toggle('snap-temp', on);
}
```

`snapTemp` is view state in exactly the sense of §28.5: it is **not** in `state.project`, **not** written by `buildProjectData()`, **not** pushed onto the undo stack, does **not** call `markProjectDirty()`, and works in **read-only** mode. The saved format is unchanged (`v:2`), so there is no migration and no `normalizeProject()` change.

Because `snap()` re-reads `snapActive()` on **every** call, releasing Shift mid-drag takes effect on the very next pointer move — the snap decision is never latched at pointer-down.

### 30.2 As-built anchors

| What | Where |
|---|---|
| `.tool-btn.snap-temp` CSS (dashed outline affordance) | `index.html:170-172` |
| `state.snapTemp: false` | `index.html:1425` |
| `snap()` gate → `snapActive()` | `index.html:1526` |
| `snapActive()` / `setSnapTemp(on)` | `index.html:1555` / `1558` |
| Shift keyup → `setSnapTemp(false)` (inside the existing keyup listener) | `index.html:5653` |
| Shift keydown + `blur` + `visibilitychange` safety net | `index.html:5655-5665` |
| `#snap-toggle` `title` mentions the shortcut | `index.html:845` |

### 30.3 Why a separate keydown listener (not `onKey()`)

Shift is bound through a **dedicated** `document` `keydown` listener registered next to the existing `keyup` listener, rather than inside `onKey()` (`index.html:5172`):

- `onKey()` begins with `if (state.readOnly) return;`, and momentary snapping must keep working in read-only mode (it is a view preference, like the button itself — §28.5).
- `onKey()` also bails when inline-editing or when focus is in an `INPUT`/`TEXTAREA`/`SELECT`/contenteditable, so typing a capital letter in the property panel would otherwise arm/disarm snapping.
- Shift is a modifier, not an action: it must not be consumed or `preventDefault()`-ed.

`setSnapTemp(true)` is idempotent, so auto-repeat keydown events are harmless. The three release paths — `keyup`, `window` `blur`, and `document` `visibilitychange` when hidden — exist so the flag can never get stuck on when the keyup is delivered to another window/tab (a real risk with `Alt`/`Cmd`-Tab while the key is held).

### 30.4 Affordance

```css
.tool-btn.snap-temp:not(.active){outline:2px dashed var(--color-accent);outline-offset:-3px;color:var(--color-accent)}
```

The `:not(.active)` guard means the dashed outline only appears when the master switch is **off** — i.e. only when Shift is actually doing something. When the switch is already on, the button keeps its normal latched accent fill and nothing changes on release. The outline is drawn inside the 44 px button (`outline-offset:-3px`) so it does not shift layout, and it uses `--color-accent` so it reads identically in both themes.

### 30.5 Interaction with the existing Shift bindings

Shift was already used for other things, and none collides because none routes through `snap()`:

| Existing Shift use | Location | Interaction |
|---|---|---|
| Lock the move axis (horizontal/vertical) | `onPointerMove()`, drag branch | Unaffected: applies to moving an element body, after `snap()` has run. |
| Constrain aspect ratio while resizing (also forced by `lockAspect`, §26) | `onPointerMove()`, `index.html:5069-5088` | Composes: holding Shift now also *snaps* the resize, which is what you want when constraining to a grid-friendly ratio. The ratio-preserving dimension is still snapped-after-constraint (§26.4). |
| Constrain line-draw to 15° increments | `onPointerMove()`, line-draw branch | Unaffected: the line-draw branch runs before the drag branch and never calls `snap()`. |
| Constrain a **line-endpoint** drag to 15° increments from the opposite end | `onPointerMove()`, endpoint-dragging branch | Replaces the grid `snap()` for the duration of the drag: the dragged end keeps its distance from the fixed end and its angle snaps to the nearest 15°. Mirrors the line-draw constraint. See `PROJECT_PAGE_PROPS.md` §30.5. |
| `Shift+Arrow` = large arrow-key nudge | `onKey()`, `index.html:5266-5275` | Unaffected: arrow nudging deliberately does **not** consult `snap()` (§28.8). |

Note the pre-existing local variable `snap` (the constrained angle) inside the line-draw block shadows the global `snap()` function; it is scoped to that block and is unrelated to this feature.

### 30.6 Verification

- **Harness `shiftsnap` suite — 38/38 pass** (`node harness.js index.html none shiftsnap`): `snapActive()` truth table for all four `snapEnabled`×`snapTemp` combinations; `snap()` gate honours `snapTemp` when the master is off (grid and guide snapping both re-enabled); `setSnapTemp` idempotence and coercion; the `.snap-temp` class toggling and the `:not(.active)` suppression; **no undo snapshot** and **no dirty flag**; `snapTemp` absent from `buildProjectData()`; the keydown/keyup/blur/visibilitychange wiring driven through the fake DOM's real listener registry.
- **Real headless Chromium probe — 19/19 pass, `ERRORS=[]`**: an actual pointer drag with the master off moves freely to `(153,127)`; the same drag with a real Shift `keydown` held snaps to `(150,130)` and lands on a 10 px multiple; releasing Shift **mid-drag** re-frees the next move (`150,130 → 153,127`); with the master on, Shift is a no-op (still `150,130`) and a plain drag still snaps; `keydown Shift → snapActive()===true`, `keyup → false`, with the drag result changing accordingly; the `.snap-temp` class is present while held, absent after release, never sets `.active`, and coexists with `.active` when the master is on.
  - *(The probe initially reported 4 "failures" because it set the `shiftKey` flag on the synthetic `PointerEvent`s instead of dispatching a real Shift `keydown`/`keyup`; the feature is keydown-driven, so the probe was corrected to match real user input. No product code changed as a result.)*
- **Regressions**: `phase1`–`phase4`, `marginband`, `aspect`, `lockaspect`, `resetratio`, `snaptoggle` (33/33), `gridunits` (41/41) all green; all six load modes (`none`/`v1`/`v2`/`empty`/`corrupt`/`nopages`) report `errors: []`; `node --check` clean.

### 30.7 Known limitations / notes

- Like §28, this is a per-browser view preference and is not saved with the project.
- The dashed outline is the only on-canvas feedback; there is still no "snap line" highlight (§28.8).
- Shift must be held **before** the pointer move that should snap. Pressing Shift and dragging in the same instant works in practice because the keydown is delivered first, but a synthetic event stream that only sets `shiftKey` on the pointer events will not arm it — by design.
- Momentary snapping does not change the arrow-key step; 1 px nudges remain un-snapped (§28.8).

---


## 31. As-Built Notes — Shape alignment panel (follow-up)

> Requested: add a 3×3 alignment control so a shape (or any element) can be placed flush to the page, the page margins, or the selection's bounding box.

### 31.1 What was there before

The tree already contained a function named `align(how)` (originally ~`index.html:1883`), but `grep` confirmed it was **dead code** — defined and never called. (The only other `align`-ish references were the Text panel's `Ctrl+Shift+L/C/R` justify shortcuts, which act on `#prop-text-content`, not on shapes.) Rather than leave a second, confusingly-named function in place, §31 **replaced it outright** with `alignSelection(anchor, target)`.

### 31.2 Design: exact geometry, a target box, and a no-op-free undo step

Three decisions shape the implementation:

1. **Alignment is exact, not snapped.** The deltas are applied directly and rounded to whole CSS px (`Math.round`), **not** routed through `snap()`. An explicit "align left" should put the edge exactly at the guide, not at the nearest grid line — and it must behave identically whether snapping is on or off.
2. **A target box, not just the page.** `alignTargetBox(kind)` returns the rectangle the anchor is measured against:
   - `page` → `{x:0, y:0, w:pw, h:ph}`;
   - `margins` → the page inset by `p.margins` (clamped so a margin larger than half the page cannot invert the box);
   - `selection` → the union bounding box of all selected elements, or `null` when fewer than two are selected.
3. **A no-op never pushes an undo step.** The function builds a `plan[]` of `{element, dx, dy}` **first**, drops entries where `dx === 0 && dy === 0`, and returns early when the plan is empty. Only then does it mutate, call `saveSnapshot()` (post-mutation, matching the rest of the app — `addElement`/`addPage`/`removeElement`/resize all snapshot *after* the change so the undo-stack top always equals the live state), and `emit()`.

`elBBox(el)` gives every element type an axis-aligned box: rectangles/images/text use `{x, y, w, h}`; lines are reduced from their two endpoints (`min`/`abs`) so **both** endpoints move together.

```js
var ALIGN_TARGETS = ['page', 'margins', 'selection'];

function alignSelection(anchor, target) {
  if (state.readOnly) return;                     /* mutating → guarded */
  var ids = state.selectedElements;
  if (!ids || !ids.length) return;
  var kind = ALIGN_TARGETS.indexOf(target) === -1 ? state.alignTarget : target;
  var box  = alignTargetBox(kind);
  if (!box) { /* selection target needs ≥2 elements */ … return; }
  var v = anchor.charAt(0), h = anchor.charAt(1); /* t|m|b + l|c|r */
  …
  var plan = [];
  ids.forEach(function(id) {
    var g = getEl(id), b = elBBox(g.el);
    var tx = h === 'l' ? box.x : h === 'c' ? box.x + (box.w - b.w) / 2 : box.x + box.w - b.w;
    var ty = v === 't' ? box.y : v === 'm' ? box.y + (box.h - b.h) / 2 : box.y + box.h - b.h;
    var dx = Math.round(tx) - b.x, dy = Math.round(ty) - b.y;
    if (dx || dy) plan.push({ g: g, dx: dx, dy: dy });
  });
  if (!plan.length) return;                        /* no-op → no undo step */
  plan.forEach(function(it) {
    it.g.el.x += it.dx; it.g.el.y += it.dy;
    if (it.g.el.type === 'line') { it.g.el.x2 += it.dx; it.g.el.y2 += it.dy; }
    syncElementDOM(it.g.el.id);
  });
  saveSnapshot(); updateProps(); emit();
}
```

The `anchor` string is validated (`'tmb'` for the vertical letter, `'lcr'` for the horizontal) so a malformed `data-align` is ignored rather than mis-parsed. The `target` argument is optional: when it is not one of `ALIGN_TARGETS` the current `state.alignTarget` is used, which keeps the function testable without touching the DOM.

### 31.3 Target is view state, not document data

Like the snapping master switch (§28.5), the chosen target is a **per-browser preference**:

```js
alignTarget: 'page',             /* index.html:1450 — view state */
```

It lives in `state`, is restored in `bindAlign()` from `localStorage['lp-align-target']` (validated against `ALIGN_TARGETS`), and is written back on `change`. It is **not** in `state.project`, **not** serialized by `buildProjectData()`, **not** pushed onto the undo stack, does **not** call `markProjectDirty()`, and is **not** read-only-guarded (choosing a target mutates nothing). The saved format is unchanged (`v:2`), so there is no migration and no `normalizeProject()` change.

### 31.4 UI

Markup (Properties ▸ **Alignment**, between Position and Text):

- a `<select id="prop-align-target">` with **Page / Margins / Selection**;
- a `role="group"` `.align-grid` (`#prop-align-grid`) of nine `.align-btn`s, each carrying `data-align="tl|tc|tr|ml|mc|mr|bl|bc|br"` plus a `title` and `aria-label`, and an inline SVG whose faint 18×18 square is the *target* and whose solid 8×8 block sits at the corresponding 3×3 position.

The section is shown for **any** selected element and hidden when nothing is selected, alongside Position/Box (`updateProps()`, `index.html:2898`). Click handling is a single delegated listener on `#prop-align-grid`:

```js
grid.addEventListener('click', function(e) {
  if (state.readOnly) return;                       /* mutating → guarded */
  var btn = e.target && e.target.closest ? e.target.closest('[data-align]') : null;
  if (!btn || !grid.contains(btn)) return;
  alignSelection(btn.dataset.align, state.alignTarget);
});
```

CSS:

```css
.align-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:4px;margin-top:var(--space-1)}
.align-btn{…min-height:38px;…}                    /* 3 columns share the panel width */
.align-btn:active{background:var(--color-accent);border-color:var(--color-accent);color:#FFFFFF}
.align-btn svg{width:16px;height:16px;display:block}
```

The buttons use the existing `--color-border`/`--color-surface`/`--color-accent` tokens so they read identically in both themes. The target `<select>` reuses the shared `.prop-row label input` styling rather than a bespoke rule.

### 31.5 As-built anchors

| What | Where |
|---|---|
| `.align-grid` / `.align-btn` CSS | `index.html:370-375` |
| Alignment panel markup (`#prop-align-section`, `#prop-align-target`, `#prop-align-grid`) | `index.html:936-953` |
| `state.alignTarget: 'page'` | `index.html:1450` |
| `ALIGN_TARGETS` / `elBBox()` / `alignTargetBox()` / `alignSelection()` | `index.html:1914` / `1917` / `1930` / `1956` |
| `updateProps()` shows/hides the section | `index.html:2898` |
| `bindAlign()` (restore + select + delegated click) | `index.html:3437` |
| `bindAlign()` called from `init()` | `index.html:5731` |

### 31.6 Verification

- **Harness `align` suite — 52/52 pass** (`node harness.js index.html none align`): all nine page anchors on an 850×1100 page; the margins target (box `40,100,760,920`); the selection-union target; line bbox movement (both endpoints); single-element + selection target → no change + toast; **no-op never pushes an undo step**; read-only hard no-op; undo restores position; a malformed anchor is ignored; the default `state.alignTarget`; and source-level checks that the markup and `bindAlign()` wiring exist.
- **Real headless Chromium probe — 37/37 pass, `ERRORS=[]`**: real `.click()` on each of the nine buttons through the delegated listener, DOM `style.left/top` sync, section show/hide, read-only, and undo.
- **jsdom suite — 42/42 pass** (`/tmp/jsdom_align.js`), the same behaviours against a real DOM implementation.
- **Regressions**: the full harness sweep (`phase1`–`phase4`, `marginband`, `aspect`, `lockaspect`, `resetratio`, `snaptoggle` 33/33, `gridunits` 41/41, `shiftsnap` 38/38) all green; all six load modes report `errors: []`; `node --check` clean on both `<script>` blocks; the pre-existing browser probes p1–p5 and the Shift probe re-run green.

### 31.7 Known limitations / notes

- The target choice is **per-browser**, not per-document (mirrors §28/§30).
- There is no "align to spread" or "align to bleed" target; only page / margins / selection.
- Buttons are ~38 px tall, below the 44 px touch minimum (matches the existing property rows; not a regression).
- Alignment is a discrete click, so no key-repeat guard is needed.
- Rotation is ignored — alignment uses the unrotated `{x,y,w,h}` box, consistent with resize (§26.7) and export.
- Alignment is not routed through `snap()`, so it is unaffected by the snapping master switch (§28) or momentary Shift snapping (§30).

---

## 32. As-Built Notes — PDF / print pixel fidelity (follow-up)

> Reported: exported PDFs were "funky" — a text box wrapped at different words than the on-screen canvas. PDF is the primary output and must match the canvas exactly.

### 32.1 Root cause: two parallel renderers that drift

The canvas and the export built the same element markup in **two different places**:

- the on-screen page used `domEl()` (`index.html:2641`) + `styleTextBox()` (`index.html:2609`) — the real renderer; while
- `exportElementsHTML()` (`index.html:2380`) hand-wrote its own `<div style="…">` strings.

A hand-written copy of a renderer drifts the moment either side changes. At the time of the report the export copy was missing four things that control **text layout**:

1. the global reset `*,*::before,*::after{box-sizing:border-box;margin:0;padding:0}` — so the exported outer box rendered as `content-box` and a `width:200px;padding:8px` frame laid out its text in **200 px** instead of the canvas's **184 px** (`200 − 2×8`);
2. the `.el-text` class (`white-space:pre-wrap;word-break:break-word`) on the outer box;
3. `.el-content{display:block;width:100%;box-sizing:border-box}` on the inner content box;
4. the rich-text inheritance rule `p,div,span{font:inherit;color:inherit}`.

Because every one of those changes the content width or the white-space/word-break behaviour, **the wrap points moved** — which is exactly what the user saw. Two smaller divergences rode along: a stroked text frame lost its border, and `paraSpacing` (the `margin-bottom` on `<p>`/`<div>`) was not applied.

A quick reproduction confirmed it: same text, canvas-CSS markup measured **184 px / 9 line boxes**, export-CSS markup **200 px / 5 line boxes**.

### 32.2 Fix: one renderer, mirrored CSS, no web fonts

`exportElementsHTML()` no longer writes markup. It calls the canvas builder and serializes the result:

```js
function exportElementsHTML(pg, off) {
  var wrap = document.createElement('div');
  pg.elements.forEach(function(el, i) {
    if (!el.visible) return;
    var node = domEl(el, i);                 /* the exact canvas builder */
    if (off) {                               /* bleed shift, post-build */
      node.style.left = (parseFloat(node.style.left) + off) + 'px';
      node.style.top  = (parseFloat(node.style.top)  + off) + 'px';
    }
    wrap.appendChild(node);
  });
  return wrap.innerHTML;
}
```

`domEl()` writes `left`/`top` for **every** element type (lines use their min corner), so the `parseFloat` bleed shift is safe for text, images, shapes and lines alike.

`exportHTML()` then ships the **same rules the canvas uses** in its embedded `<style>` (`index.html:2415-2424`): the global reset, `.canvas-element{position:absolute;border:1px solid transparent}`, `.el-text`, `.el-clip`, `.el-content`, `.el-image` / `.el-image img`, and `.drop-cap::first-letter`. Because the markup and the stylesheet both come from the canvas side, the print layout and the screen layout are produced by the same code path.

The dead **Google-Fonts `<link>` was removed**. The app only offers six families (Georgia, Times New Roman, Arial, Helvetica Neue, Courier New, system-ui), all locally installed, and the canvas loads no web fonts at all. A `<link>` could only (a) block the print on a slow/offline request, or (b) — if a saved document carried a custom `font-family` — download a face the canvas never had, re-breaking the match. The export is now self-contained.

### 32.3 Why this is the right shape of fix

Element markup now has a **single source of truth**. Any future change to `domEl()`/`styleTextBox()` flows to PDF automatically; there is no second renderer left to drift. The raster path (`exportRaster()`) already reused `domEl()`, which is why PNG/JPEG never showed the bug.

### 32.4 As-built anchors

| What | Where |
|---|---|
| `exportElementsHTML()` (now `domEl()`-based) | `index.html:2380` |
| `exportHTML()` (mirror CSS block) | `index.html:2396` (CSS 2415-2424) |
| `styleTextBox()` / `domEl()` (the canvas renderers) | `index.html:2609` / `2641` |
| Canvas rules mirrored: `.el-content` / `.el-text` / `.el-clip` / `.el-image` | `index.html:102` / `444` / `474` / `445-446` |
| `exportPDF()` (writes `exportHTML()`, prints on load) | `index.html:2486` |
| `exportRaster()` (already `domEl()`-based) | `index.html:2503` |

### 32.5 Verification

- **Real headless Chromium probe — 24/24 pass, `ERRORS=[]`** (`/tmp/pdf_probe.html`): renders three text frames (plain single-column; two-column with `columnGap`; rich text with `<b>`, `<p>`, `paraSpacing`, `textIndent`, `letterSpacing`), measures the glyph-run positions / line count / rendered text height on the canvas and again inside an iframe loaded with `exportHTML()`, and asserts they are **byte-identical** (after normalising to the content-box origin). All three cases: *content width matches, line count matches, glyph-run positions match ("identical"), rendered height matches*. The probe also asserts the exported markup carries `class="canvas-element el-text"` + `class="el-content"`, that the legacy hand-built div is gone, that a stroked frame exports its border, that the bleed offset is applied, and that the export has **no `<link>`/web-font reference**.
- **Harness `phase4` suite — all pass**: the harness's fake DOM gained minimal `innerHTML`/`outerHTML` serialization (style keys + children) so the `domEl()`-produced markup can be asserted in Node.
- **jsdom suite — all pass** (`/tmp/jsdom_test.js`), including the bleed image-offset assertion (re-pointed at the real `.el-image` node).
- **Regressions**: the full harness sweep (`phase1`–`phase4`, `marginband`, `aspect`, `lockaspect`, `resetratio`, `snaptoggle` 33/33, `gridunits` 41/41, `shiftsnap` 38/38, `align` 52/52) all green; `jsdom_align` 42/42; browser probes p1–p5, the Shift probe and the grid probe re-run green; `node --check` clean on both `<script>` blocks; no stray `console.log`/`debugger`.

### 32.6 Known limitations / notes

- The exported page still clips at the trim box (`.page{overflow:hidden}`), while the **live canvas does not** clip at the page edge. This is intentional for print (paper clips at the trim anyway) and predates §32; it is not the wrapping bug.
- Fonts are **not embedded**. The PDF uses whatever the printing browser resolves for the same `font-family` stack the canvas used; because the stack is all system fonts, this matches on the same machine. A machine without, say, Georgia falls back to `serif` in both places, so the match holds.
- `@page` still carries a single size for multi-size documents (§24.7), and bleed is a proof rather than a press-ready trim/bleed box (§24.7).
- `exportPDF()` prints after `onload` + a 250 ms settle (1200 ms fallback); it does not explicitly wait on `document.fonts.ready` — unnecessary while no web fonts are involved.

---

## 33. As-Built Notes — Unique z-height & element naming (follow-up)

### 33.1 Motivation

Layout's stacking order used to be the **array order** of `page.elements` alone: `domEl()` wrote `z-index = i + 1`, and the Layers panel simply reversed the array. There was no way to name an element, and the four z actions only moved an element to either end of the array (front/back), never a step. This follow-up ports the **diagram project's** z-height model and adds editable names. It, along with the other work in this session (right-drag pan, horizontal-only align, the reworked Shift bindings, unclipped text-box handles, larger resize hit targets, and the single export menu), ships as **v0.4.0**.

### 33.2 The invariant: no two elements share a zHeight

Every element carries an integer `zHeight`. The invariant, mirroring the diagram tool, is:

- `zHeight` values on a page are **dense, unique ranks `0..N-1`** (0 = bottom of the stack).
- Rendering derives `z-index = zHeight + 1` (1-based, because 0 is falsy in some engines), so **array order no longer decides stacking** — only `zHeight` does.
- The invariant is re-imposed (`normalizeZHeights()`) on every structural change and on `refresh()`, so it is idempotent and cannot drift.

Legacy projects (no `zHeight`) are backfilled in `normalizeProject()` by array index (`el.zHeight = idx`), so an old file loads with its original visual stacking intact.

### 33.3 Z-order helpers

A dedicated block replaces the old `bringForward`/`sendBackward` pair:

| Function | Purpose | Anchor |
|---|---|---|
| `zElementList()` | Page elements as `{obj,id,z,order}`, sorted by `(z, arrayIndex)` | `index.html:1919` |
| `normalizeZHeights()` | Dense re-rank `0..N-1` in current stack order; idempotent | `index.html:1935` |
| `computeZOrder()` | id → 1-based z-index map for renderers | `index.html:1940` |
| `zElementRank(id)` | Current 0-based rank of an element, or `-1` | `index.html:1946` |
| `moveZToRank(id, rank)` | Splice to a rank (clamped), re-densify; returns `false` when absent | `index.html:1954` |
| `zLabel(id)` | `name` fallback `id` | `index.html:1966` |
| `bringToFront` / `sendToBack` | Move to last / first rank | `index.html:1971` / `1978` |
| `bringForward` / `sendBackward` | One rank up / down (no-op at the ends) | `index.html:1985` / `1993` |

The mutation helpers call `saveSnapshot()` + `emit()`; the toolbar and keyboard iterate the current selection and invoke them once per selected element.

### 33.4 Render & hit-test integration

- `mkEl()` seeds `zHeight` (default `0`) and honours `opts.name` (`index.html:1799–1800`).
- `domEl()` writes `d.style.zIndex = (el.zHeight != null ? el.zHeight : i) + 1` (`index.html:2861`).
- `normalizeZHeights()` is called from `addElement`, `removeElement`, `dupElement`, and `refresh()`.
- `drawThumb()` (canvas thumbnails) sorts a copy of the page's elements by `zHeight` before drawing (`index.html:2970`).
- `findEl()` no longer returns the last DOM node under the cursor: it walks every hit and keeps the **highest `zHeight`** (`index.html:4743`), which is required once stacking is decoupled from array/DOM order.
- Export inherits z for free because `exportElementsHTML()` / `buildExportStage()` reuse `domEl()`.

### 33.5 Naming

- `mkEl()` defaults to a random `Type NN` name, but `opts.name` wins, so `dupElement()`'s `Name copy` is preserved.
- **Properties ▸ Name** (`#prop-name`, `index.html:969`): `updateProps()` reflects it (guarded by `document.activeElement` so typing is not clobbered); the `input` handler writes `updEl(id,{name})` for every selected element, re-renders the Layers panel, marks the project dirty and schedules an autosave; `blur` normalises the field from state and takes one undo snapshot.
- **Layers panel** (`renderLayers()`, `index.html:3013`): each row shows the name; **double-click renames in place** via `startLayerRename()` (`index.html:3124`) — commit on Enter/blur, revert on Esc.
- Both entry points stay in sync because they write the same `el.name` and each triggers `renderLayers()`.

### 33.6 Layers-panel z field

Each row carries a `.layer-z` number input showing the element's rank. Changing it calls `moveZToRank(eid, value)` (clamped), then `saveSnapshot()` + `emit()`. The list keeps its **long-standing top-first order** (highest `zHeight` at the top), so the About-page tip "elements at the top of the list appear in front" remains true — unlike the diagram tool, whose list is bottom-first. The Properties panel gained a matching **`Z`** number field next to W/H (`#prop-z`, `index.html:971`); its `change` handler runs the same `moveZToRank` for every selected element.

Rows are also **drag-to-reorder** (`item.draggable = true`). During `dragover` the dragged row is moved live in the DOM (top half of a target inserts above it, bottom half below); `drop` (bound once on the list, so it also works on empty space) reads the final DOM order and hands it to `applyLayerOrder(idsTopFirst)` (`index.html:2011`), which sets `rank = N-1-domIndex`, re-densifies and records one undo snapshot. `dragend` then `emit()`s to rebuild from state, or calls `renderLayers()` to restore the DOM when the drag was cancelled (no drop). Dragging is suppressed in read-only mode and when the gesture starts on the z input, the visibility toggle, or an in-progress rename (`mousedown` sets a `dragBlocked` flag).

### 33.7 Toolbar & shortcuts

The old two-button group (front/back) became four buttons — **Bring to front / Bring forward / Send backward / Send to back** (`#bring-front`, `#bring-forward`, `#send-backward`, `#send-back`, `index.html:876–879`), bound at `index.html:4251–4261`. Keyboard (`onKey`, `index.html:5757–5766`): `Ctrl+]` / `Ctrl+[` step, `Ctrl+Shift+]` / `Ctrl+Shift+[` jump to front/back. On US layouts `Shift+]` produces `}`, so both `}`/`{` are accepted.

### 33.8 Verification (headless Chromium)

- **Invariant:** the default project renders 8 elements with DOM `z-index` `1..8`, all unique; the Layers panel shows unique ranks; a legacy project (3 elements, no `zHeight`) loads with `z-index` `1/2/3` and names preserved.
- **Reordering:** select the first element → `#bring-front` → rank 7/7; `#send-backward` → 6; `#bring-forward` → 7; `#send-back` → 0; `Ctrl+]` → 1; `Ctrl+Shift+{` → 0. Uniqueness holds after every step.
- **Layers z input:** setting a row's z to `N-1` moves that element to the top (`z-index = N`); out-of-range `999`/`-5` clamp to top/bottom.
- **Naming:** double-click rename writes `Hero Title` to both the layer list and `#prop-name`; typing in `#prop-name` updates the layer list live.
- **Selection:** pointer-down on the last-created element selects it (z-aware `findEl`), and `node --check` is clean with **no console errors** on load or export.

### 33.9 Known limitations / notes

- The **Layers list order is top-first** (layout's existing convention), not bottom-first as in the diagram tool; the z numbers therefore read high-to-low down the list. Only the *mechanism* (unique, editable ranks) was ported.
- Multi-element toolbar/keyboard reordering applies front/back one element at a time in selection order, so a multi-selection is re-stacked in selection order rather than as a group. Single-selection behaviour — the common case — is exact.
- §30 ("momentary snapping") is now **historical**: the momentary `state.snapTemp` flag it describes was removed; §30.5 already records the surviving `Shift` bindings (move axis-lock, resize ratio-lock, line-draw / line-end 15° snap).

---


*End of draft — this is a starting point, not a final spec. Phases 0–4 are implemented (§20–§24) plus the margin-indicator (§21.2), image-aspect-ratio (§25), image aspect-lock (§26), reset-to-original-ratio (§27), snapping master switch (§28), unit-aware grid step (§29), momentary snapping (§30, now historical), shape-alignment panel (§31), PDF pixel fidelity (§32), and unique z-height & element naming (§33) follow-ups; §18 records the resolved open questions. See §0 for the technical review pass and the corrections it produced.*
