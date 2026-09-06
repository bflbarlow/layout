# Layout — Page Layout Editor

A full-page layout editor (InDesign-lite / Canva-lite) built entirely with HTML, CSS, and JavaScript.

**Part of the free open tools ecosystem.** Conforms to `/Users/bflbarlow/Websites/freeopentools/STYLE_GUIDE.md`. See `STYLE_GUIDE_RECONCILIATION.md` for the full migration record.

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
- **Layer ordering** — Bring to front / send to back

### Page Management
- **Multi-page** — Add, remove, and switch between pages
- **Page thumbnails** — Visual page list in the sidebar

### Properties
- **Position & size** — x, y, width, height
- **Style** — Fill color, stroke color, stroke width, opacity, border radius
- **Text** — Font family, font size, bold, italic, underline, text color, text content

### Undo / Redo
- **Ctrl+Z** — Undo
- **Ctrl+Shift+Z** — Redo
- **50-step history** — Sufficient for most design workflows

### Persistence
- **Autosave** — Work is automatically saved to browser localStorage
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
  text: "Hello",
  fontFamily: "system-ui, sans-serif",
  fontSize: 16,
  fontWeight: "400",
  fontStyle: "normal",
  textDecoration: "none",
  textColor: "#000000",
  textAlign: "left",
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
├── index.html      # The entire application (60KB, 1541 lines)
└── REVIEW.md       # Technical review and development history
```

That's it. One file to distribute, one file to run.

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
# layout
