# Layout — Page Layout Editor

A full-page layout editor (InDesign-lite / Canva-lite) for building beautiful documents — programs, magazines, playbills, posters, and more. Built entirely with HTML, CSS, and JavaScript.

**Part of the free open tools ecosystem.** Conforms to `/Users/bflbarlow/Websites/freeopentools/STYLE_GUIDE.md`. See `STYLE_GUIDE_RECONCILIATION.md` for the full migration record.

---

## 🎯 Objectives — Text Design Without Friction

The core purpose of Layout is to let users combine text, images, and lines to build beautiful documents. **Right now, text editing is the weakest link.** The following objectives define the path to making every text box a "mini Google Docs" — a fully capable rich-text environment where everything possible with text is possible.

### ═══════════════════════════════════════════
### 1. Rich Text Editing Inside the Box
### ═══════════════════════════════════════════

Every text box must be a fully functional rich-text editor on double-click.

- **Inline formatting** — Bold, italic, underline, strikethrough, superscript, subscript. All work with keyboard shortcuts (Ctrl+B/I/U) and toolbar buttons.
- **Mixed formatting** — One paragraph bold, the next not. A single word highlighted in a different color. Multiple fonts and sizes within the same text box.
- **Font family per selection** — Change the font of selected text, not the whole box.
- **Font size per selection** — Change size of selected text, with a dropdown or up/down stepper.
- **Text color per selection** — Color picker applies to highlighted text, not the whole element.
- **Background/highlight color** — Add a highlight (marker) effect to selected text.
- **Clear formatting** — One-click button to strip all inline formatting from selected text.

### ═══════════════════════════════════════════
### 2. Paragraph & Line Controls
### ═══════════════════════════════════════════

- **Line height (leading)** — Adjustable per text box or per paragraph. Crucial for magazine and playbill layouts.
- **Letter spacing (tracking)** — Adjustable per selection or per text box. Essential for headlines, titles, and poster text.
- **Paragraph spacing** — Margin before/after paragraphs. Independent of line height.
- **Text indent** — First-line indent for paragraphs.
- **Bulleted & numbered lists** — Toggle bullet and ordered lists inside the text box.
- **Text alignment** — Left, center, right, justify. Works per paragraph.
- **Vertical alignment** — Top, middle, bottom alignment of text within the text box bounds.
- **Text direction** — Left-to-right and right-to-left support.
- **Columns** — Divide a text box into multiple columns (2, 3, 4) for magazine-style layouts.

### ═══════════════════════════════════════════
### 3. Text Box & Overflow Management
### ═══════════════════════════════════════════

- **Auto-resize** — Option for the text box to grow vertically (or horizontally) to fit all content, so users never have to guess the right height.
- **Overflow handling** — Options: clip, overflow (scrollable), or continue to a linked text box (see "Text Threading" below).
- **Text threading (linked text boxes)** — Connect two or more text boxes so text flows from one to the next. This is a hallmark of professional layout tools (InDesign, Quark) and essential for magazines, playbills, and multi-page documents.
- **Padding inside text box** — Inset spacing between the text and the box border. Independent of the box position.
- **Min/max height** — Constrain how much a text box can grow when auto-resizing.

### ═══════════════════════════════════════════
### 4. Typography & Advanced Text Features
### ═══════════════════════════════════════════

- **Font weights** — Support for numeric weights beyond just 400/700 (300, 500, 600, 800, 900) so users can use variable fonts or font families with many weights.
- **Font variants** — Small caps, all caps, ligatures, and other OpenType features.
- **Drop caps** — The first letter of a paragraph rendered large and decorative (common in magazines).
- **Text on path** — Text that follows a curved or angled line.
- **Text rotation** — Rotate the entire text box freely (not just 90° increments).
- **Hyphenation** — Automatic hyphenation for justified text.
- **Tab stops** — Custom tab stops for precise columnar alignment within a text box.

### ═══════════════════════════════════════════
### 5. Inline Elements & Rich Content
### ═══════════════════════════════════════════

- **Inline images** — Insert small images (icons, logos, decorative elements) that flow with the text.
- **Hyperlinks** — Add clickable links to selected text. Export preserves them.
- **Special characters** — Easy insert of em-dash, en-dash, bullet, copyright, trademark, and other common typographic symbols.
- **Find & replace** — Search within a text box or across all text boxes in the document.

### ═══════════════════════════════════════════
### 6. Text Styles & Presets
### ═══════════════════════════════════════════

- **Paragraph styles** — Save and apply named styles (Heading 1, Body, Caption, etc.) that bundle font, size, weight, leading, tracking, color, and alignment.
- **Character styles** — Save and apply named inline styles (Bold Red, Small Caps, etc.).
- **Quick style picker** — A floating palette or dropdown in the toolbar to apply styles with one click.
- **Style inheritance** — Changes to a paragraph style propagate to all text using that style (like InDesign or Word).

### ═══════════════════════════════════════════
### 7. Properties Panel Integration
### ═══════════════════════════════════════════

The right-side Properties panel must expose all text controls in a clean, organized way:

- **Text Content tab** — The textarea for editing raw text (updated live).
- **Typography section** — Font family, size, weight, line height, letter spacing, text color, highlight color.
- **Paragraph section** — Alignment, indent, paragraph spacing, bullets, numbering, columns.
- **Text Box section** — Padding, vertical alignment, auto-resize toggle, overflow mode, min/max height.
- **Styles section** — Paragraph style dropdown, character style dropdown, style save/delete buttons.
- **Advanced section** — Text direction, hyphenation toggle, tab stops.

### ═══════════════════════════════════════════
### 8. Export Fidelity
### ═══════════════════════════════════════════

- **Rich text preserved in export** — The exported standalone HTML file must retain all inline formatting, fonts, colors, spacing, and styles.
- **Linked text boxes reflow** — If text threading is used, the exported file must preserve the flow.
- **Font embedding** — Option to inline web-safe fonts or embed Google Fonts links in the export.
- **Print-ready output** — PDF export is implemented with html2canvas + jsPDF (the same stack as the Diagram project); the standalone HTML export remains available for browser-native print.

### ═══════════════════════════════════════════
### 9. Quality of Life
### ═══════════════════════════════════════════

- **Live preview** — All text changes render instantly on the canvas.
- **Non-destructive editing** — Undo/redo works for every text operation (including inline formatting changes).
- **Spell check** — Enable the browser's native spellcheck on editable text boxes.
- **Keyboard shortcuts** — All standard text editing shortcuts work inside the text box (Ctrl+B/I/U, Ctrl+Shift+L/C/E/R for alignment, Ctrl+Z for undo, etc.).
- **Right-click context menu** — Basic text editing options (cut, copy, paste, select all) work naturally.
- **Drag-and-drop text** — Move selected text within or between text boxes.

---

**How to use this document:** Each objective above is a candidate for a GitHub issue or a development milestone. The items are ordered roughly by priority — start with Section 1 (rich text editing inside the box) as the foundation, then build up through paragraphs, overflow, typography, and styles.

---

## Migration Status (2025-03-27)

| # | Checklist Item | Status | Notes |
|---|---|---|---|
| 1 | Uses shared color tokens | ✅ | Ecosystem tokens, light + dark theme |
| 2 | No emojis — flat SVG icons | ✅ | Layer panel uses inline SVGs, not Unicode/emoji |
| 3 | Light + dark mode, no flash | ✅ | No-flash detection in `<head>`, theme toggle in toolbar |
| 4 | All text meets WCAG AA contrast | ✅ | Ecosystem tokens provide compliant contrast |
| 5 | Elements ≥44×44px, focus states | ✅ | All buttons/inputs padded; `:focus-visible` globally |
| 6 | Keyboard operable, skip link | ✅ | Skip link, tabindex/keyboard on layers/thumbnails |
| 7 | Screen reader support | ⚠️ | `aria-label`, `aria-live`, landmarks added; needs full NVDA/VoiceOver testing |
| 8 | Respects reduced-motion + prefers-color-scheme | ✅ | Both media queries present |
| 9 | 320px width, 200% zoom | ⚠️ | Breakpoints added; fixed canvas page may overflow at high zoom |
| 10 | One clear primary action | ✅ | Export button is the sole primary CTA |
| 11 | No unnecessary dependencies | ✅ | Zero dependencies, no trackers |
| 12 | Free Open Tools attribution | ✅ | Footer with logo-text and benjaminbarlow.com link |

**Overall:** 10/12 complete, 2/12 partially complete (screen reader testing, extreme zoom testing).

---

## 🚀 No Server Required

**This is a zero-dependency, no-server project.** Simply double-click `index.html` to open it in your browser. Everything is self-contained in a single file — no build step, no web server, no external dependencies.

### Quick Start

1. **Open:** Double-click `index.html` in your browser
2. **Edit:** Use the toolbar to draw shapes, add text, and insert images
3. **Export:** Click "Export" to save your design as a standalone HTML file

That's it. No `npm install`, no `node server`, no setup.

## Features

### Tools
- **Select (V)** — Click to select, drag to move elements
- **Text (T)** — Click to place text elements
- **Rectangle (R)** — Click and drag to draw rectangles
- **Circle (C)** — Click and drag to draw circles
- **Line (L)** — Click and drag to draw lines
- **Image Insert** — Upload images to place on the canvas

### Canvas Operations
- **Multi-select** — Ctrl+Click (or Cmd+Click on Mac) to select multiple elements
- **Resize** — Drag handles on selected elements (8-point resize)
- **Drag to move** — Click and drag any element to reposition it
- **Align** — Left, center, and right alignment for selected elements
- **Layer ordering** — Unique z-height per element (no two share a rank): drag rows in the Layers panel, use Bring Forward / Send Backward, or Bring to Front / Send to Back (toolbar or `Ctrl+]`/`Ctrl+[`, with `Shift` for front/back). Reorder from the Layers panel or the Properties **Z** field.
- **Naming** — Rename an element from the Properties **Name** field or by double-clicking its name in the Layers panel; names appear beside every layer.

### Rich Text Editing
- **Inline editing** — Double-click any text box to edit directly on the canvas
- **Formatting** — Bold, italic, underline, strikethrough, superscript, subscript
- **Inline images** — Insert images inside text via the toolbar
- **Hyperlinks** — Insert/remove links (Ctrl+K) or toolbar buttons
- **Special characters** — 60+ character picker popup (©, ®, ™, arrows, math, etc.)
- **Find & replace** — Search all text elements (Ctrl+F) with replace & replace-all
- **Lists & indents** — Bulleted/numbered lists, indent/outdent buttons

### Typography & Layout
- **Full font weights** — 100 through 900 (Thin to Black)
- **Alignment** — Left, center, right, justify
- **Line height & letter spacing** — Leading and tracking controls
- **Paragraph spacing & indent** — Per-element paragraph spacing and text indent
- **Columns** — Multi-column text with gap control
- **Text direction** — LTR / RTL
- **Drop caps** — First-letter styling toggle
- **Hyphenation** — Auto-hyphenation toggle
- **Rotation** — Per-element rotation (0–360°) with accurate hit-testing

### Text Box Controls
- **Padding** — Internal text box padding
- **Vertical alignment** — Top, middle, bottom
- **Auto-resize** — Height grows to fit content
- **Overflow** — Clip / visible / scrollable
- **Min/max height** — Constraints for auto-resize

### Styles System
- **Built-in styles** — Heading 1–3, Body
- **Save / update / delete** — Reuse formatting as named styles
- **Propagation** — Update a style and all elements using it update automatically

### Page Management
- **Multi-page** — Add, remove, and switch between pages
- **Page thumbnails** — Visual page list in the sidebar

### Properties
- **Position & size** — x, y, width, height, rotation (0–360°)
- **Style** — Fill color, stroke color, stroke width, opacity, border radius
- **Style** — Reusable paragraph/character styles (save, update, delete, propagate)
- **Text** — Font family, font size, bold, italic, underline, strikethrough, superscript, subscript, text color, text content
- **Typography** — Line height (leading), letter spacing (tracking), alignment (left/center/right/justify), font weights (100–900)
- **Text Box** — Padding, vertical alignment, auto-resize height, min/max height, overflow mode (clip / visible / scroll), columns, column gap, text direction (LTR/RTL), paragraph spacing, text indent, drop caps, hyphenation
- **Formatting** — Inline images, hyperlinks, special characters, find & replace

### Undo / Redo
- **Ctrl+Z** — Undo
- **Ctrl+Shift+Z** — Redo
- **50-step history** — Sufficient for most design workflows

### Persistence
- **Autosave** — Work is automatically saved to browser localStorage
- **Auto-save to file** — In Chrome/Edge, Save opens a native file picker and the project is continuously written to that file (the green dot on the Save button indicates active auto-save; clicking it again downloads a backup)
- **Multi-tab safety** — Opening a second browser tab makes it read-only so two tabs can't overwrite each other; the second tab takes over editing when the first closes
- **Restore** — Open the file again and your work is restored
- **Export** — Save your design as a standalone HTML file

### Keyboard Shortcuts
| Key | Action |
|-----|--------|
| `V` | Select tool |
| `T` | Text tool |
| `R` | Rectangle tool |
| `C` | Circle tool |
| `L` | Line tool |
| `Ctrl+D` | Duplicate selected |
| `Ctrl+A` | Select all |
| `Delete` | Delete selected |
| `Escape` | Deselect |
| `Ctrl+F` | Find & replace |
| `Ctrl+]` / `Ctrl+[` | Bring forward / Send backward |
| `Ctrl+Shift+]` / `Ctrl+Shift+[` | Bring to front / Send to back |
| `Ctrl+K` | Insert hyperlink |
| `Ctrl+Shift+L/E/R` | Align left/center/right |
| `Ctrl+Shift+B` | Unordered list |
| `Ctrl+Shift+7` | Numbered list |

## Architecture

### Single-File Design

The entire application is contained in one `index.html` file:
- **HTML** — Application chrome (toolbar, sidebars, panels)
- **CSS** — All styling inlined in a `<style>` block
- **JavaScript** — All logic inlined in a `<script>` block

This design was chosen because:
1. **Zero setup** — Double-click to open, works immediately
2. **No server** — Works from `file://` protocol
3. **Portable** — Copy the file anywhere and it works
4. **No dependencies** — No npm packages, no CDN links, no build tools

### Code Structure

The JavaScript is organized into logical modules within the single file:

```
index.html
├── <style> (CSS)
└── <script> (JavaScript)
    ├── State management (store, history, selectors)
    ├── Rendering (canvas, pages, layers, properties, rulers)
    ├── Interaction (tools, select-tool, draw-tool, keyboard)
    ├── UI (toolbar, tabs)
    ├── I/O (persistence, export)
    └── Utilities (DOM helpers, geometry)
```

#### State Management
- Single source of truth for all page and element data
- Pub/sub pattern for rendering updates
- Undo/redo with 50-step history
- localStorage autosave with 1.5s debounce
- Optional native file auto-save via the File System Access API (Chrome/Edge), with the file handle persisted in IndexedDB
- Multi-tab coordination via BroadcastChannel — one editor tab, all others read-only

#### Rendering
- Full DOM rebuild on state changes (simpler and less error-prone)
- Canvas rendering for page thumbnails
- Resize handles for selected elements
- Rulers with grid snapping

#### Interaction
- Click-to-select and drag-to-move on canvas elements
- 8-point resize handles
- Tool switching with keyboard shortcuts
- Arrow-key nudging (1px / 10px with Shift)

### Data Model

Elements are plain serializable objects:
```js
{
  id: "el-123",
  type: "rect" | "circle" | "text" | "line" | "image",
  x: 100,
  y: 200,
  w: 300,
  h: 200,
  fill: "#ffffff",
  stroke: "#000000",
  strokeWidth: 1,
  opacity: 1,
  radius: 0,
  visible: true,
  name: "Rectangle 1",
  // Text properties (if type === "text")
  // Text properties (if type === "text")
  text: "Hello",
  richText: "<p>Hello</p>",
  fontFamily: "system-ui, sans-serif",
  fontSize: 16,
  fontWeight: "400",
  fontStyle: "normal",
  textDecoration: "none",
  textColor: "#000000",
  textAlign: "left",
  lineHeight: 1.5,
  letterSpacing: 0,
  padding: 8,
  verticalAlign: "top",
  autoResize: false,
  overflow: "hidden",
  minH: 20,
  maxH: 0,
  // Image properties (if type === "image")
  src: "data:image/png;base64,..."
}
```

Pages are arrays of elements:
```js
{ id: "page-1", name: "Page 1", elements: [...] }
```

## Troubleshooting

### "Nothing happens when I double-click index.html"
- Make sure your browser allows local file access (some browsers block it for security)
- Try opening from a different browser (Chrome, Firefox, Safari all work)

### "My work disappeared after refreshing"
- Work is saved to localStorage, which is per-browser and per-domain
- If you cleared browser data or switched browsers, your work won't be there
- Use **Export** to save your design as a standalone HTML file

### "Elements aren't selecting/moving"
- Make sure you're using the **Select tool** (V) or have clicked on an element
- Check that the element isn't hidden (eye icon in Layers panel)
- Verify you're not in a drawing tool mode (R, C, L, T)

### "Export doesn't work"
- The export creates a Blob URL and triggers a download
- Some browsers may block popups or downloads from local files
- Check your browser's download folder or popup blocker settings

## File Structure

```
layout/
├── index.html      # The entire application (2625 lines, single-file)
└── README.md       # This file
```

## Browser Support

Works in all modern browsers that support:
- ES6+ (let, const, arrow functions, template literals)
- CSS Grid and Flexbox
- localStorage
- File API (for image insertion)

Tested in:
- Chrome 90+
- Firefox 88+
- Safari 14+

## License

MIT