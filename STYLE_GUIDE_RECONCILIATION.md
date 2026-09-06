# STYLE_GUIDE_RECONCILIATION — Layout

**Date:** 2025-03-27  
**Source document:** `/Users/bflbarlow/Websites/freeopentools/STYLE_GUIDE.md`  
**Target file:** `/Users/bflbarlow/Websites/layout/index.html`  
**Status:** ⚠️ Non-conforming (see details below)

This document enumerates every technical change required to bring the Layout page editor (`layout/index.html`) into full conformance with the Free Open Tools ecosystem style guide (`freeopentools/STYLE_GUIDE.md`). Each item is linked to the relevant style guide section and is prioritized as **Required** (must fix before shipping), **Recommended** (should fix), or **Optional** (nice to have).

---

## Table of Contents

1. [Color System (§2)](#1-color-system-2)
2. [Icons & Emojis (§3)](#2-icons--emojis-3)
3. [Project Identifier (§4)](#3-project-identifier-4)
4. [Typography (§5)](#4-typography-5)
5. [Layout & Spacing (§6)](#5-layout--spacing-6)
6. [Components (§7)](#6-components-7)
7. [Light & Dark Mode (§8)](#7-light--dark-mode-8)
8. [Accessibility (§9)](#8-accessibility-9)
9. [Performance & Architecture (§10)](#9-performance--architecture-10)
10. [Voice & Content (§11)](#10-voice--content-11)
11. [Shipping Checklist (§12)](#11-shipping-checklist-12)
12. [Summary of Changes](#12-summary-of-changes)

---

## 1. Color System (§2)

### 1.1 Replace custom color tokens with ecosystem tokens

**Style Guide requirement (§2.2):** Every tool must define exactly the shared CSS custom properties on `:root`, using the specified names and values. No tools may invent their own color tokens for UI chrome.

**Current layout (`index.html` `<style>` block, lines ~10–11):**

```css
:root{--bg-dark:#1e1e2e;--bg-darker:#181825;--bg-panel:#252538;--bg-canvas:#3a3a52;--bg-page:#fff;--border-color:#3a3a52;--border-light:#4a4a62;--text-primary:#e0e0f0;--text-secondary:#9a9ab0;--text-muted:#6a6a80;--accent:#6c8cff;--accent-hover:#5a7aff;--accent-dim:#6c8cff33;--danger:#ff6b6b;--shadow:0 4px 20px rgba(0,0,0,.3);--toolbar-h:52px;--sidebar-w:240px;--statush:28px;--pw:850px;--ph:1100px}
```

This replaces the ecosystem palette entirely with a custom set of 15 unique tokens. None of the required ecosystem tokens (`--color-accent`, `--color-bg`, `--color-text`, etc.) are present.

**Change required — Required:** Delete all custom color tokens and replace with the exact ecosystem token set from STYLE_GUIDE.md §2.2. The mapping is:

| Current layout token | Ecosystem equivalent | Notes |
|---|---|---|
| `--bg-dark` | `--color-bg` (dark theme) | `#0E0E10` vs `#1e1e2e` — new value |
| `--bg-darker` | `--color-bg-subtle` | `#17171A` vs `#181825` — close but must be exact |
| `--bg-panel` | `--color-surface` | `#1C1C20` vs `#252538` — new value |
| `--bg-canvas` | (use `--color-bg-subtle`) | Remove this token; canvas uses the existing surface |
| `--bg-page` | (keep as tool-specific) | But rename to `--lp-bg-page` or similar tool-prefixed var |
| `--border-color` | `--color-border` | `#2C2C32` vs `#3a3a52` — new value |
| `--border-light` | (remove) | Not in ecosystem; use `--color-border` for all borders |
| `--text-primary` | `--color-text` | `#F2F2F3` vs `#e0e0f0` — new value |
| `--text-secondary` | `--color-text-muted` | `#A3A3AB` vs `#9a9ab0` — close but must be exact |
| `--text-muted` | `--color-text-disabled` | `#5B5B63` vs `#6a6a80` — new value |
| `--accent` | `--color-accent` | **`#2563EB` vs `#6c8cff`** — significant color change |
| `--accent-hover` | `--color-accent-hover` | `#1D4ED8` vs `#5a7aff` — new value |
| `--accent-dim` | `--color-accent-soft` | `#2563EB1A` vs `#6c8cff33` — new value |
| `--danger` | `--color-danger` | `#DC2626` vs `#ff6b6b` — new value |
| `--shadow` | `--shadow-md` | `0 4px 12px rgba(0,0,0,0.10)` vs `0 4px 20px rgba(0,0,0,.3)` |

**Specific changes needed in `index.html`:**

1. Delete the entire `:root` block (line ~10) and replace with the ecosystem token block from STYLE_GUIDE.md §2.2.
2. Add a light-theme block (`:root, [data-theme="light"] { ... }`).
3. Add a dark-theme block (`[data-theme="dark"] { ... }`).
4. Keep tool-specific layout variables (`--toolbar-h`, `--sidebar-w`, `--statush`, `--pw`, `--ph`) but move them into a separate `--lp-*` namespaced block to avoid collision.
5. Update every CSS rule in the stylesheet to use the new token names.

### 1.2 Semantic color usage (§2.3)

**Style Guide rule:** Blue is reserved for primary actions, links, active/focused states. Semantic colors (success/warning/danger) used only for their meaning. Never rely on color alone to convey meaning.

**Current layout:** The blue accent `#6c8cff` is used generally for tooltips, active buttons, selected states, and border highlights. The `--danger` (`#ff6b6b`) is used on page delete buttons without a secondary non-color indicator.

**Change required — Required:**
- Replace all `--accent` references with `--color-accent`.
- Add a non-color indicator (icon + text label) to the page delete button — currently it's just a red circle with `×` and no accessible name.
- Ensure `--color-success` / `--color-warning` / `--color-danger` are used only for their semantic meaning.

### 1.3 Pure black/white usage (§2.3)

**Style Guide rule:** Pure black (`#000000`) and pure white (`#FFFFFF`) should be used sparingly. Prefer the near-black/near-white values.

**Current layout:** Uses `#000000` for text/stroke color defaults (e.g., `value="#000000"` on color inputs) and `#ffffff` for fill defaults. Also uses `#fff` / `#white` for the page canvas background and some UI elements (`toggle.innerHTML`, `resize-handle` background).

**Change required — Recommended:** Replace `#000000` default values with `--color-text` and `#ffffff` with `--color-surface` where they serve as UI element defaults. The page canvas background (`#fff`) is a tool-specific design concern and can remain white but should use `#FFFFFF` explicitly (not `#fff` shorthand for consistency).

---

## 2. Icons & Emojis (§3)

### 2.1 No emojis (§3.1)

**Style Guide rule:** Emojis are forbidden. Every emoji must be replaced with a flat SVG icon.

**Current layout violations — Required:**

| Location | Emoji/Unicode | Current usage | Replacement |
|---|---|---|---|
| `layer-list` icons | `\u25AD` (▭) | Rectangle layer icon | SVG rectangle icon (16px, Lucide/Heroicons style) |
| `layer-list` icons | `\u25CF` (●) | Circle layer icon | SVG circle icon |
| `layer-list` icons | `\u2571` (╱) | Line layer icon | SVG line icon |
| `layer-list` icons | `\uD83D\uDDBC` (🖼) | Image layer icon | SVG image icon |
| `layer-list` icons | `\u2757` (⚠) fallback | Unknown type | SVG "?" icon |
| `layer-visibility` | `\uD83D\uDC41` (👁) | Visible eye icon | SVG eye icon (open/closed) |
| `layer-visibility` hidden | `\uD83D\uDC41\u200D\uD83D\uDEB5` (👁‍🗨) | Hidden eye icon | SVG eye-off icon |
| Demo text 1 | `\u2022` (•) ×4 | Bullet points | Acceptable — this is a bullet/separator, not an emoji |
| Demo footer | `\u2022` (•) ×2 | Separators in footer text | Acceptable as a separator in rendered content |
| `delete-page-btn` | `\u00D7` (×) | Delete icon | Replace with SVG X icon |
| Demo content | `\u270E` (✎) pencil? | (None found) | — |

Additionally, the demo content text contains hardcoded Unicode bullets (`\u2022`). While not an emoji, these should ideally be replaced with proper SVGs or kept only as bullet-point text in user-facing content (the style guide prohibits emojis, not all Unicode punctuation).

### 2.2 Icon set consistency (§3.3)

**Style Guide requirement:** Pick one icon set (Lucide, Heroicons, or Phosphor) and use it for the entire ecosystem.

**Current layout:** Uses inline SVGs in the toolbar that visually resemble Lucide/Heroicons-style outline icons (24×24 viewBox, 2px stroke, `stroke-linecap="round"`). These are acceptable in style but must be verified for consistency.

**Change required — Required:** Decide on a single icon set and convert all icons to that set. The toolbar icons already match the Lucide/Heroicons aesthetic; convert the layer-list and visibility icons to match the same style, stroke width (2px), and size scale (16px or 20px for layer items).

### 2.3 Icon accessibility (§3.2)

**Style Guide rule:** Every icon that conveys meaning must have an accessible name via `aria-label` or a visually-hidden `<span>`. Decorative icons use `aria-hidden="true"`.

**Current layout:** Toolbar buttons have `title` attributes (e.g., `title="Select (V)"`) but no `aria-label`. This is insufficient for screen readers that don't expose `title` on buttons.

**Change required — Required:** Add `aria-label` to every icon button — the `title` value is a good starting point but must be duplicated into `aria-label`. Ensure all SVGs in the toolbar have `aria-hidden="true"` (or `role="presentation"`) since the button itself carries the accessible name.

---

## 3. Project Identifier (§4)

### 3.1 Formal name usage (§4.1, §4.3)

**Style Guide requirement:** Every tool must include a small, unobtrusive attribution with: the tool's name, the Free Open Tools mark, and a link to `benjaminbarlow.com`.

**Current layout:** Has no attribution whatsoever. No reference to Free Open Tools, no logo, no link to the author.

**Change required — Required:** Add a footer attribution bar (or small corner element) containing:
- The tool name ("Layout" or "Page Layout Editor")
- The stylized `free<span>open</span>tools` mark (with blue `open`)
- A link to `https://benjaminbarlow.com`

This must be visible but unobtrusive — small text, muted color, bottom of the page (below the canvas/status bar or as a subtle footer).

### 3.2 Logo treatment (§4.2)

**Style Guide requirement:** The identifier is rendered as `free<span>open</span>tools` with the `span` in `--color-accent`.

**Current layout:** No logo present.

**Change required — Required:** Add the ecosystem logo text to the attribution footer with the exact HTML/CSS from §4.2:

```html
<div class="logo-text">free<span>open</span>tools</div>
```

```css
.logo-text {
  color: var(--color-text);
  font-weight: 700;
}
.logo-text span {
  color: var(--color-accent);
}
```

### 3.3 No competing branding

**Style Guide rule:** No "back to all tools" links, breadcrumbs, or directory nav. Each tool stands on its own.

**Current layout:** No such nav exists — compliant as-is.

---

## 4. Typography (§5)

### 4.1 Font stack (§5.1)

**Style Guide requirement:** Use `--font-sans` and `--font-mono` as defined, using the exact system font stack. No custom/webfonts.

**Current layout (`index.html` line ~14):**

```css
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',system-ui,sans-serif;font-size:13px;...}
```

**Change required — Required:** 
1. Replace the inline `font-family` with `var(--font-sans)`.
2. Change `font-size: 13px` to `var(--text-sm)` (14px) — 13px is below the minimum body text of 14px (see §5.2).
3. Add the `--font-sans` and `--font-mono` declarations to the `:root` block (they are already in the ecosystem token set but must be included).

### 4.2 Type scale (§5.2)

**Style Guide requirement:** Use only the defined scale tokens.

**Current layout:** Uses ad-hoc sizes throughout:
- 13px on body (violates `--text-base` minimum of 16px for body text)
- 11px on sidebar headers, tab buttons, small buttons, status bar (between `--text-xs` 12px and `--text-sm` 14px — use `--text-xs`)
- 12px on zoom label, layer items (use `--text-xs`)
- 10px on panel section headers (violates — minimum is `--text-xs` 12px)
- 9px on ruler labels (violates — minimum usable is `--text-xs` 12px)
- Font-weight 600 on sidebar headers (OK as heading-like)

**Change required — Required:** Replace all font sizes with the ecosystem scale:

| Current size | Usage | New token |
|---|---|---|
| 13px | Body default | `--text-sm` (14px) — actual minimum for body-like text |
| 11px | Sidebar headers, tab buttons, small buttons, status bar | `--text-xs` (12px) |
| 12px | Zoom label, layer items | `--text-xs` (12px) — OK as-is |
| 10px | Panel section headers (`h4`) | `--text-xs` (12px) — no 10px allowed |
| 9px | Ruler number labels | `--text-xs` (12px) — significantly larger, may need layout adjustment |

### 4.3 Font weight rules (§5.2)

**Style Guide rule:** Font weight range: 400 (body), 500 (labels/emphasis), 600–700 (headings only).

**Current layout violations — Required:**
- `.sidebar-header h3` uses `font-weight: 600` — OK for heading-like text
- `.tab-btn` uses `font-weight: 600` — borderline, should be 500
- `.panel-section h4` uses `font-weight: 600` — borderline
- `.status-tool` uses `font-weight: 500` — OK

These are minor. The main issue is that the body text is at 13px weight 400 — the weight is correct but the size is wrong.

### 4.4 Line length (§5.2)

**Style Guide rule:** Paragraph text should stay within 45–75 characters per line (`max-width` on text blocks).

**Current layout:** The demo content includes multi-line text elements without any `max-width` constraint.

**Change required — Recommended:** Add `max-width` to text block rendering in `domEl()` to ensure readable line lengths, or consider this a tool-specific layout concern (the user controls text width via drag).

---

## 5. Layout & Spacing (§6)

### 5.1 Spacing scale (§6.1)

**Style Guide requirement:** All margins, padding, and gaps must use the ecosystem `--space-*` tokens. No arbitrary pixel values.

**Current layout violations — Required:** Every padding, margin, and gap value in the CSS uses ad-hoc pixel values:

| Current value | Context | Ecosystem token |
|---|---|---|
| `padding: 0 12px` | Toolbar | `--space-3` (12px) — OK as-is, matches exactly |
| `gap: 4px` | Toolbar group | `--space-1` (4px) |
| `gap: 2px` | Toolbar group items | `--space-1` (4px) — round up |
| `margin: 0 6px` | Divider | `--space-2` (8px) |
| `padding: 10px 12px` | Sidebar header | `--space-3` (12px) / `--space-3` (12px) |
| `padding: 8px` | Page list, page list items | `--space-2` (8px) — OK |
| `margin-bottom: 8px` | Page thumb | `--space-2` (8px) — OK |
| `padding: 4px 6px` | Thumb label | `--space-1` (4px) / `--space-2` (8px) |
| `padding: 8px` | Tab content/panel | `--space-2` (8px) |
| `margin-bottom: 16px` | Panel section | `--space-4` (16px) — OK |
| `gap: 6px` | Prop row | `--space-2` (8px) — round up |
| `margin-bottom: 6px` | Prop row | `--space-2` (8px) |
| `gap: 8px` | Layer item | `--space-2` (8px) — OK |
| `padding: 6px 8px` | Layer item | `--space-2` / `--space-2` |
| `padding: 12px` | Tab panel | `--space-3` (12px) |
| `padding: 8px` | Text element default | `--space-2` (8px) |
| `margin-bottom: 8px` | Thumbnail spacing | `--space-2` (8px) |
| `padding: 40px` | Canvas container | `--space-7` (48px) — round up to nearest token |
| `gap: 16px` | Status bar items | `--space-4` (16px) — OK |
| `padding: 0 12px` | Status bar | `--space-3` (12px) — OK |

**Note:** Some values like 2px, 6px, 10px don't exist in the 4px-based scale. These must round to the nearest token (2px→4px, 6px→8px, 10px→12px). This will cause minor visual changes to density.

### 5.2 Content container constraints (§6.2)

**Style Guide rule:** Content centered in a constrained container. Tool-focused layouts: `max-width: 720px`. Editors with side-by-side panels: `max-width: 960px–1100px`.

**Current layout:** The canvas page is fixed at `850px` wide (`--pw`), which falls within the 960–1100px range for workspace tools. The sidebar is `240px` wide.

**Change required — Recommended:** Wrap the entire layout in a container with `max-width: 1100px` and center it. Currently the layout fills the entire viewport width, which is fine for a workspace tool but should be constrained on very wide screens.

### 5.3 Corner radius (§6.2)

**Style Guide requirement:** Use the three radius tokens. No other values.

**Current layout violations — Required:**

| Current value | Context | Token |
|---|---|---|
| `6px` | Tool buttons | `--radius-sm` (6px) — OK |
| `4px` | Sidebar add button, font select, size input, style-btn, small-btn, layer items, page thumb | `--radius-sm` (6px) — round up |
| `2px` | Resize handles | `--radius-sm` (6px) — resize handles are 10×10px, 2px is small, but 6px is too large; consider if this is a "radius" in the ecosystem sense or a tool-specific detail |
| `50%` | Delete page button (circular) | Tool-specific — OK |
| `4px` | Demo content radius values (rounded rect) | Content data, not component — OK as stored value |
| `border-radius: inherit` | via `--radius: 0` | Tool data model — OK |

**Special case — resize handles:** The 2px radius on 10×10px resize handles is too small to reach `--radius-sm` (6px). These should be considered tool-specific UI and may use an ad-hoc value if documented as a deliberate deviation. However, the style guide says "Deviation requires a documented reason" — add a comment.

### 5.4 Shadows (§6.2)

**Style Guide requirement:** Use only the three shadow tokens. Shadows are subtle, used only for elevation.

**Current layout:** Uses a single custom shadow:

```css
--shadow:0 4px 20px rgba(0,0,0,.3)
```

This is heavier than `--shadow-md` (`0 4px 12px rgba(0,0,0,0.10)`).

**Change required — Required:** Replace with `var(--shadow-md)` for the canvas page shadow. Remove the custom `--shadow` variable.

### 5.5 Whitespace over borders (§6.2)

**Style Guide rule:** Use generous whitespace over borders/dividers to separate sections. Borders are a last resort.

**Current layout:** Uses many borders — toolbar dividers, sidebar borders, panel section borders, thumb borders, element borders. This is appropriate for a grid/workspace tool where visual boundaries define the editing surface.

**Change required — Recommended:** Audit whether all borders are necessary. The left/right sidebar borders and toolbar dividers are reasonable for a complex editor UI. No changes required if each border serves a functional purpose.

---

## 6. Components (§7)

### 6.1 Buttons (§7.1)

**Style Guide requirement:** Three button variants only (primary, secondary, ghost/text). Minimum touch target 44×44px. Minimum padding `--space-4` horizontal, `--space-3` vertical. Disabled state with `--color-text-disabled` and `cursor: not-allowed`. Hover, active, focus-visible, and disabled states required.

**Current layout:** Has the following button types:
- **Tool buttons** (`.tool-btn`): 36×36px — violates 44×44px minimum touch target
- **Primary button** (`.tool-btn.primary`): `padding: 6px 14px` — violates minimum `--space-3` (12px) vertical, `--space-4` (16px) horizontal
- **Small buttons** (`.small-btn`): `padding: 4px 10px` — violates minimums
- **Sidebar add button** (`.sidebar-add-btn`): 26×26px — violates 44×44px
- **Style buttons** (`.style-btn`): 32×32px — violates 44×44px
- **Tab buttons** (`.tab-btn`): variable padding, no minimum
- **Delete page button** (`.delete-page-btn`): 18×18px — violates 44×44px
- **Page thumb** (clickable): no explicit minimum

**Change required — Required:**
1. Increase all clickable buttons to 44×44px minimum hit area (use invisible padding or enlarged click area).
2. Replace ad-hoc padding with `--space-3` vertical, `--space-4` horizontal on primary and secondary buttons.
3. Add disabled states with `--color-text-disabled` and `cursor: not-allowed` (currently disabled opacity is `.3` with `cursor: not-allowed` — close but needs color change).
4. Add `:focus-visible` outlines to all buttons.

### 6.2 Forms & Inputs (§7.2)

**Style Guide requirement:** Every input has a visible, persistent label. Minimum height 44px on touch devices. Font size 16px minimum. Error states inline with icon+text. `aria-describedby` and `aria-invalid` for validation.

**Current layout:**
- Property panel inputs use label wrapping with inline labels (e.g., `<label>X <input>`)
- Inputs have `font-size: 12px` — violates the 16px minimum
- Many inputs are small (padding `4px 6px`, height likely ~28px) — violates 44px touch target
- Color inputs are 32px wide, 26px or 32px tall — violates 44px minimum
- No validation error states on any input
- No `aria-describedby` or `aria-invalid` attributes
- Placeholder text exists but is not used as label (labels are present) — OK

**Change required — Required:**
1. Increase input font size to `--text-base` (16px) minimum — this will also prevent iOS auto-zoom.
2. Increase input height to minimum 44px (padding and line-height).
3. Ensure all inputs have visible labels — the current wrapping `<label>` pattern is acceptable but verify screen reader association.
4. Add validation states for numeric inputs (min/max constraints) with inline error messages.
5. Add `aria-describedby` and `aria-invalid` on inputs where validation applies.

### 6.3 Cards (§7.3)

**Style Guide requirement:** Consistent padding (`--space-5`), `--radius-md`, `1px solid var(--color-border)`.

**Current layout:** The page thumbnails are card-like: `border: 2px solid transparent`, `border-radius: 4px`, padding via child elements. The demo content uses rounded rectangles with custom radius.

**Change required — Required:** 
- Page thumbnails: padding `--space-5` is too much for 212×275px thumbnails; use `--space-3` and document as tool-specific deviation.
- Border radius: change from 4px to `--radius-sm` (6px) or `--radius-md` (10px).
- Border: change from `2px solid transparent` / custom colors to `1px solid var(--color-border)`.
- Active/hover state: use subtle elevation (`--shadow-md`) as specified, not a full color change.

### 6.4 Project attribution (§7.4)

**Style Guide requirement:** Every tool includes a small, consistent footer or corner attribution with: tool name, Free Open Tools mark, and link to `benjaminbarlow.com`. No "back to all tools" links. Attribution visible but not intrusive.

**Current layout:** None present.

**Change required — Required:** Add a footer element below the status bar or integrated into it:

```html
<div class="lp-attribution">
  <span>Layout · </span>
  <span class="logo-text">free<span>open</span>tools</span>
  <span> · </span>
  <a href="https://benjaminbarlow.com" class="lp-author-link">Benjamin Barlow</a>
</div>
```

Style: `font-size: var(--text-xs)`, `color: var(--color-text-muted)`, unobtrusive positioning (bottom-right of the canvas area or below the status bar).

### 6.5 Feedback & States (§7.5)

**Style Guide requirement:** Actions >300ms show loading indicator. Destructive actions require confirmation. Empty/error/success states pair icon with plain-language text. Toasts consistent position, auto-dismiss 4–6s, dismissible and pausable.

**Current layout:**
- No loading indicators anywhere
- No confirmation on destructive actions (delete page, delete element)
- No toast/notification system
- No visible empty states (the canvas is never empty due to demo content)
- Image insertion can be slow (FileReader on large images) with no feedback

**Change required — Required:**
1. Add a confirmation step before deleting a page or element — either a modal (`[role="dialog"]`, `aria-modal="true"`) or an inline "Are you sure?" state.
2. Add an inline loading spinner (using `--color-accent`) for the image FileReader operation.
3. Add a toast notification system using the ecosystem pattern (bottom-right, auto-dismiss 4–6s, dismissible, pausable on hover/focus).
4. Add an empty state for the canvas when no pages or elements exist.

---

## 7. Light & Dark Mode (§8)

### 7.1 Theme support (§8.1)

**Style Guide requirement:** Every application must support both light and dark mode. Respect OS preference, allow manual override, persist in localStorage. No flash on load.

**Current layout:** Dark mode only. The entire color scheme is dark-theme, with `#fff` used only for the page canvas (simulating white paper). No light theme, no theme toggle, no `prefers-color-scheme` detection, no `data-theme` attribute.

**Change required — Required:**
1. Add the in-`<head>` no-flash theme detection script (STYLE_GUIDE §8.1):
   ```html
   <script>
     (function () {
       var saved = localStorage.getItem('theme');
       var theme = saved || (window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light');
       document.documentElement.setAttribute('data-theme', theme);
     })();
   </script>
   ```
2. Add a light theme block with appropriate colors (the page canvas `--lp-bg-page` should remain white in both themes since it simulates paper).
3. Add a theme toggle button in the toolbar or sidebar.
4. Make the toggle accessible (`aria-label`), use the moon/sun SVG icon pattern from the ecosystem `index.html`.

### 7.2 Contrast (§8.2)

**Style Guide requirement:** All text/background combinations must meet WCAG 2.1 AA (4.5:1 normal text, 3:1 large text). UI elements meet 3:1. Test every theme combination.

**Current layout (dark theme only):**
- `--text-primary (#e0e0f0)` on `--bg-dark (#1e1e2e)`: contrast ratio ~4.0:1 — **fails** AA (needs 4.5:1)
- `--text-secondary (#9a9ab0)` on `--bg-dark (#1e1e2e)`: contrast ratio ~2.5:1 — **fails** AA for normal text
- `--text-muted (#6a6a80)` on `--bg-dark (#1e1e2e)`: contrast ratio ~1.3:1 — **fails** entirely
- After switching to ecosystem colors, these values change to `#F2F2F3` on `#0E0E10` (~15:1 — passes), `#A3A3AB` on `#0E0E10` (~5.8:1 — passes), `#5B5B63` on `#0E0E10` (~2.3:1 — fails AA for normal text, OK for disabled/decoration).

**Change required — Required:** The ecosystem tokens are designed to meet AA. After migration:
- Verify `--color-text-muted` (#A3A3AB on #0E0E10) passes at ~5.8:1 — OK.
- Verify `--color-text-disabled` (#5B5B63 on #0E0E10) at ~2.3:1 — this is only used for disabled elements, which have relaxed requirements.
- Test the light theme similarly after designing it.

### 7.3 Non-color signals (§8.3)

**Style Guide rule:** Any meaning conveyed by color must have a second, non-color signal.

**Current layout violations — Required:**
- Delete page button: red circle only, no text label
- Selected state: accent border only, no thickness change or icon
- Active tool: accent background only
- Hidden layer items: reduced opacity only

**Change required:** Add text labels, icon changes, or weight/size changes alongside color for all state indicators.

### 7.4 Focus states (§8.4)

**Style Guide requirement:** All interactive elements must have visible `:focus-visible` style — 2px accent-colored outline with 2px offset. Never `outline: none` without a replacement.

**Current layout:** No focus styles defined anywhere. Focusable elements use browser defaults.

**Change required — Required:** Add the ecosystem focus style globally:

```css
:focus-visible {
  outline: 2px solid var(--color-accent);
  outline-offset: 2px;
}
```

Apply to all interactive elements: buttons, inputs, selects, textareas, page thumbnails, layer items, tab buttons, etc.

---

## 8. Accessibility (§9)

### 8.1 Semantic HTML (§9.1)

**Style Guide requirement:** Use real semantic HTML elements. One `<h1>` per page. Sequential heading levels. Descriptive `alt` text. Landmarks (`<header>`, `<main>`, `<footer>`, `<nav>`).

**Current layout analysis — Required:**
- Uses `<button>` for all buttons — good.
- Uses `<label>` wrapping inputs — good, but verify `for` attribute association.
- Uses `<select>`, `<input>`, `<textarea>` — good.
- **Missing:** No `<header>`, `<main>`, or `<footer>` landmarks. Only `<aside>` is used (for sidebars).
- **Missing:** No `<h1>`. The page title is in `<title>` but the visible heading is absent. The sidebar has `<h3>` but no `<h1>`.
- **Missing:** No `alt` attributes on images inserted into the canvas (image uploads create `<img>` with no `alt`).

**Change required:**
1. Wrap the toolbar in `<header>`.
2. Wrap the main canvas-area in `<main>`.
3. Wrap the attribution in `<footer>`.
4. Add `<h1>` for the tool title (visually hidden if preferred, but required for structure).
5. Sidebar `<h3>` is fine as a section heading subordinate to the page title.
6. In `domEl()`, when creating image elements, set `alt=""` by default (decorative) and provide a way for the user to set `alt` text.

### 8.2 Keyboard access (§9.2)

**Style Guide requirement:** Every interactive element reachable and operable via keyboard. Tab order follows visual/logical order. No keyboard traps. Modals trap focus. Skip to main content link.

**Current layout violations — Required:**
- **No skip-to-content link:** The first focusable element should be a skip link.
- **Page thumbnails:** Clickable but no explicit `tabindex` — verify they're reachable.
- **Layer items:** Clickable but no role or keyboard event handlers.
- **Color inputs:** Are `type="color"` which has mixed browser keyboard support — acceptable but test.
- **Resize handles:** Hidden DOM elements, not focusable — consider keyboard-based resize (optional enhancement).
- **Tab trap:** The sidebar tabs work correctly with click but don't follow WAI-ARIA tabs pattern (arrow keys, `role="tablist"`, etc.).

**Change required:**
1. Add a skip-to-content link as the first focusable element: `<a href="#canvas-area" class="skip-link">Skip to main content</a>`.
2. Add `tabindex="0"` and keyboard handlers (Enter/Space to activate) to layer items and page thumbnails without native button semantics.
3. Verify all inputs and buttons are in correct tab order.
4. Consider implementing arrow-key navigation for sidebar tabs.

### 8.3 Screen reader support (§9.3)

**Style Guide requirement:** Dynamic content changes announced via `aria-live`. Form controls associated with labels. Icon buttons have accessible names.

**Current layout violations — Required:**
- Dynamic content (page switch, element add/delete) not announced — add `aria-live="polite"` region for status messages.
- Icon-only buttons have `title` but no `aria-label`.
- Label-input association is via wrapping which is adequate but should be tested. Consider `for` attributes.
- No `aria-describedby` on any input.
- The `aria-live` status announcements for undo/redo/export actions are missing.

### 8.4 Reduced motion (§9.4)

**Style Guide requirement:** Respect `prefers-reduced-motion`. Disable non-essential animation. Never convey info through animation alone.

**Current layout:** Has CSS transitions (`.15s` on tool buttons, `.1s` on layer items, etc.) but no `prefers-reduced-motion` media query.

**Change required — Required:** Add the ecosystem reduced-motion rule:

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

### 8.5 Touch targets (§9.5)

**Style Guide requirement:** Minimum 44×44px interactive target size. 8px spacing between targets. No hover-only functionality.

**Current layout violations — Required:**
- Tool buttons: 36×36px — needs invisible padding to reach 44×44px
- Style buttons: 32×32px
- Sidebar add button: 26×26px
- Delete page button: 18×18px
- Color input: 32×26px / 32×32px
- Resize handles: 10×10px — too small for touch, but these are mouse-precision handles; consider adding invisible 44×44px touch targets for mobile

**Change required:** Pad all undersized buttons to 44×44px minimum using `padding` or pseudo-element expansion. Add `min-width: 44px` and `min-height: 44px` where needed.

### 8.6 Responsive design (§9.6)

**Style Guide requirement:** Works from 320px up through large displays. Usable at 200% zoom without loss of function. Reflow at 400% zoom without horizontal scrolling. Use `rem`/`em`, not fixed `px` for typography.

**Current layout:**
- Has two media queries (at 1100px and 800px) that reduce sidebar width
- Uses `px` for all font sizes and spacing
- Canvas area with overflow scroll handles narrow widths, but the page content has fixed 850×1100px dimensions that won't reflow
- At 200% zoom (effectively doubling everything), the fixed-width page elements will overflow

**Change required — Required:**
1. Add more granular responsive breakpoints (320px, 480px, 600px, 800px, 1100px).
2. At narrow widths (<600px): collapse sidebars into drawers, stack toolbar buttons.
3. Convert font sizes to `rem`/`em` (at least the UI chrome; the canvas page dimensions can remain px since they simulate paper).
4. Test at 200% zoom and ensure all controls are still reachable without horizontal scrolling.
5. Support both portrait and landscape orientations on mobile.

### 8.7 Viewport zoom (§9.6, §7.2)

**Style Guide requirement:** Minimum 16px font size on inputs prevents iOS auto-zoom. Already covered in §6.2 above.

---

## 9. Performance & Architecture (§10)

### 9.1 Framework weight (§10)

**Style Guide rule:** No unnecessary JS framework. Prefer vanilla JS/HTML/CSS.

**Current layout:** Zero dependencies, single-file, vanilla JS — **compliant as-is.** No changes needed.

### 9.2 Third-party resources (§3.2, §10)

**Style Guide rule:** No external CDNs, third-party scripts, ads, or trackers.

**Current layout:** No external resources — **compliant as-is.**

### 9.3 Offline-first (§10)

**Style Guide rule:** All tools function offline-first (static, client-side processing).

**Current layout:** Fully client-side, works from `file://` — **compliant as-is.**

---

## 10. Voice & Content (§11)

### 10.1 UI copy

**Style Guide rule:** Plain language, short and direct, no exclamation points, no dark patterns.

**Current layout analysis — Required:**
- Tool `title` attributes: "Select (V)", "Text (T)", "Rectangle (R)" — good, direct.
- Demo content: "Welcome to Layout", "A page layout editor built with HTML, CSS & JavaScript" — acceptable for demo content.
- Demo footer: "Try editing me!" — exclamation mark, borderline.
- Button text: "Export" — good.
- Placeholder text: "Enter text..." — fine.
- Error messages: None exist — need to add (e.g., "Could not load image. File may be too large or corrupted.").

**Change required:** Remove the exclamation from demo text. Add proper error messages for image load failures, export failures, and localStorage quota warnings.

### 10.2 Error messages (§11)

**Style Guide rule:** Errors explain what happened and how to fix it — never just "Something went wrong."

**Current layout:** No error handling visible in the JS. The `save()` function swallows exceptions silently (`catch(e) { /* quota exceeded, ignore */ }`). Image load failures are unhandled. Export failures are unhandled.

**Change required — Required:**
1. Add user-facing error messages for localStorage quota warnings.
2. Add error handling for FileReader image loads (file type validation, size limits).
3. Add error handling for export Blob creation.
4. Show errors via the toast notification system (see §6.5).

---

## 11. Shipping Checklist (§12)

This is the final readiness assessment against the STYLE_GUIDE §12 checklist:

| # | Checklist item | Status | Notes |
|---|---|---|---|
| 1 | Uses only shared color tokens (black/white/gray + one blue accent) | ❌ | All tokens are custom; accent is wrong blue |
| 2 | No emojis anywhere — flat SVG icons only | ❌ | Layer icons use Unicode glyphs; visibility toggle uses emoji |
| 3 | Works in both light and dark mode with no flash on load | ❌ | Dark-only; no theme detection script; no toggle |
| 4 | All text meets WCAG AA contrast in both themes | ❌ | Current dark theme fails; post-migration should pass |
| 5 | All interactive elements ≥44×44px with visible focus states | ❌ | Buttons are 26–36px; no focus styles |
| 6 | Fully operable by keyboard alone, no traps | ❌ | No skip link; layer items not keyboard accessible |
| 7 | Screen reader tested (VoiceOver/NVDA) for primary flow | ❌ | Not yet; aria-label, aria-live, landmarks all missing |
| 8 | Respects `prefers-reduced-motion` and `prefers-color-scheme` | ❌ | No media queries for either |
| 9 | No layout breakage at 320px width or 200% browser zoom | ⚠️ | Untested; fixed-width canvas will cause overflow |
| 10 | One clear primary action; no competing CTAs | ✅ | Export is the primary action; no competing CTAs |
| 11 | No unnecessary dependencies, trackers, or dark patterns | ✅ | Zero dependencies, no trackers, no dark patterns |
| 12 | Includes attribution (tool name, Free Open Tools mark, link to benjaminbarlow.com) | ❌ | No attribution present |

**Total: 2/12 passing, 1/12 marginal, 9/12 failing.**

---

## 12. Summary of Changes

### Required (must fix before shipping as a Free Open Tools tool)

| # | Area | Change | Effort |
|---|---|---|---|
| R1 | Color system | Replace all CSS custom properties with ecosystem tokens | Medium |
| R2 | Color system | Replace accent color `#6c8cff` → `#2563EB` across all UI | Medium |
| R3 | Icons | Replace Unicode/emoji layer icons with SVGs | Small |
| R4 | Icons | Add `aria-label` and `aria-hidden` to all icon buttons | Small |
| R5 | Branding | Add Free Open Tools attribution footer | Small |
| R6 | Typography | Update to ecosystem type scale and font variables | Small |
| R7 | Spacing | Convert all px values to `--space-*` tokens | Medium |
| R8 | Shadows | Replace custom shadow with ecosystem shadow | Small |
| R9 | Buttons | Increase all to 44×44px minimum; add proper states | Medium |
| R10 | Inputs | Increase to 16px font and 44px height minimum | Medium |
| R11 | Light/dark mode | Implement both themes with no-flash detection | Large |
| R12 | Focus states | Add ecosystem focus-visible style globally | Small |
| R13 | Semantic HTML | Add `<header>`, `<main>`, `<footer>`, `<h1>` landmarks | Small |
| R14 | Skip to content | Add skip link as first focusable element | Small |
| R15 | Keyboard | Add keyboard handlers to layer items, thumbnails | Medium |
| R16 | `aria-live` | Add status announcements for dynamic changes | Small |
| R17 | `prefers-reduced-motion` | Add ecosystem reduced-motion media query | Small |
| R18 | Error handling | Add user-facing error messages throughout | Medium |
| R19 | Confirmation | Add confirmation for destructive actions (delete) | Medium |
| R20 | Loading states | Add spinner for image loading | Small |
| R21 | Responsive | Add breakpoints for 320px, 480px, 600px | Medium |
| R22 | Rem/em | Convert UI font sizes to relative units | Medium |

### Recommended (should fix for quality)

| # | Area | Change | Effort |
|---|---|---|---|
| D1 | Color inputs | Replace `#000000` defaults with `--color-text` | Small |
| D2 | Toast system | Add toast/notification component | Medium |
| D3 | Empty states | Add empty state for zero pages/elements | Small |
| D4 | Line length | Add max-width to text elements | Small |
| D5 | Content container | Wrap layout in max-width container | Small |
| D6 | Focus trap | Implement proper modal focus management for confirmation dialogs | Medium |

### Optional (nice to have, future)

| # | Area | Change | Effort |
|---|---|---|---|
| O1 | Image alt text | Allow user to set alt text on images | Medium |
| O2 | Keyboard resize | Add keyboard controls for element resizing | Medium |
| O3 | WAI-ARIA tabs | Implement proper tabs pattern in sidebar | Medium |
| O4 | Contrast testing | Automated contrast validation across both themes | Small |
| O5 | Touch | Add invisible touch targets for resize handles | Small |

### Already compliant (no changes needed)

| Area | Status |
|---|---|
| Zero dependencies | ✅ |
| Vanilla JS (no framework) | ✅ |
| Offline-first, works from `file://` | ✅ |
| Inline SVGs (toolbar icons) | ✅ |
| No third-party resources | ✅ |
| Single-file architecture | ✅ |
| No trackers, ads, or dark patterns | ✅ |
| One clear primary action (Export) | ✅ |
| Keyboard shortcuts (V, T, R, C, L, Ctrl+Z, etc.) | ✅ |
| localStorage persistence | ✅ |
| Undo/redo with history | ✅ |

---

## Appendix: Migration Implementation Notes

### File structure after migration

The tool should remain a single-file application (`index.html`), but the structure inside the file changes:

```
layout/index.html
├── <script> (no-flash theme detection — in <head>)
├── <style>
│   ├── Ecosystem tokens (from STYLE_GUIDE §2.2, §4.2, §5.2, §6.1, §6.2)
│   ├── Light theme variables
│   ├── Dark theme variables
│   ├── Tool-specific layout variables (--lp-*)
│   ├── Global reset and focus styles
│   ├── Toolbar, sidebar, canvas (updated tokens)
│   ├── Responsive breakpoints
│   └── Reduced motion
├── <body>
│   ├── Skip link
│   ├── <header> (toolbar — add theme toggle button)
│   ├── <main> (canvas area and sidebars)
│   ├── <footer> (attribution bar)
│   └── <script> (application logic — mostly unchanged)
└── </html>
```

### URL and deployment

Per `APP_DIRECTORY.md`:
- **Current URL:** `layout.benjaminbarlow.com`
- **Future URL:** `layout.freeopentools.com`

No code changes needed for URLs — this is a deployment/DNS concern.

### Existing demo content

The demo content uses hardcoded colors (`#1a1a2e`, `#6a6a80`, `#6c8cff`, etc.) that are part of the element data model, not the UI. These are user data — they should remain as-is or the demo data should be regenerated using the new palette. The demo content is not subject to the style guide since it's sample user-created content, not tool UI.

### Breaking change: color input defaults

The color inputs default to `#000000` and `#ffffff`. After migration, the default fill for new elements should use `--color-surface` (which varies by theme) and default stroke should use `--color-text-muted`. However, since these are data-model values (stored in element state, not CSS), they must be specific hex values. Recommended new defaults: fill `#FFFFFF`, stroke `#111113`.

---

*End of reconciliation. Total items: 22 required, 6 recommended, 5 optional, 10 already compliant.*