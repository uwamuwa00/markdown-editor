# Code Structure

## File Overview

- **Single file application**: `index.html` (~50KB)
- **Self-contained**: No build step, no npm, no local dependencies
- **CDN dependencies**: Mermaid.js v11, KaTeX 0.16.9
- **Architecture**: Two-class component model + global function layer

---

## HTML Structure

```
<head>
  <style>                        CSS variables, layout, theming, components
  </style>
</head>
<body>
  <div class="container">
    <div class="toolbar">        Formatting buttons, export, theme, spellcheck
    <div class="content">
      <div class="editor-panel"> Textarea (left)
      <div class="panel-divider"> 1px vertical separator
      <div class="preview-panel"> Rendered output (right)
    <div class="status-bar">     Char/word/line count, cursor pos, save status
  </div>
  <div class="toast">            Notification overlay
  <div class="modal-overlay" id="tableModal">    Table builder
  <div class="modal-overlay" id="formulaModal">  Formula picker (V2)
  <script>                       All JavaScript
</body>
```

---

## CSS Architecture

### Theme System
```css
:root { }                    /* Light mode variables */
[data-theme="dark"] { }      /* Dark mode overrides */
```

Key variables: `--bg-primary`, `--bg-secondary`, `--bg-tertiary`, `--text-primary`, `--text-secondary`, `--border-color`, `--accent-color`

Theme applied to `<html>` element via `document.documentElement.setAttribute('data-theme', ...)` — cascades to all elements including modals and KaTeX.

### KaTeX Dark Mode Fix (V2)
```css
[data-theme="dark"] .katex,
[data-theme="dark"] .katex * { color: var(--text-primary) !important; }
[data-theme="dark"] .katex .frac-line { background: var(--text-primary) !important; }
[data-theme="dark"] .katex svg path { fill: var(--text-primary) !important; }
```
Covers all KaTeX internals: text spans, SVG paths, fraction lines, root symbols.

### Panel Layout (V2)
```css
.editor-panel  { background-color: var(--bg-primary); }
.preview-panel { background-color: var(--bg-secondary); }
.panel-divider { width: 1px; background: var(--border-color); }

[data-theme="dark"] .editor-panel  { background-color: #161616; }
[data-theme="dark"] .preview-panel { background-color: #1e1e1e; }
```

### Modal System
```css
.modal-overlay        { opacity: 0; visibility: hidden; pointer-events: none; }
.modal-overlay.show   { opacity: 1; visibility: visible; pointer-events: all; }
```
Both Table Builder and Formula Picker use this same pattern.

---

## JavaScript Architecture

### Class 1 — MarkdownParser

Responsible for converting raw Markdown text to HTML.

```javascript
class MarkdownParser {
    constructor() {
        this.rules = [ /* priority-ordered regex rules */ ]
    }

    parse(text) {
        // 1. Extract LaTeX → placeholder Map  (before markdown rules — V2)
        // 2. parseCodeBlocks()                (Mermaid + code highlight)
        // 3. parseTables()
        // 4. parseLists()
        // 5. Apply this.rules                 (headings, bold, italic, links...)
        // 6. Restore LaTeX from placeholder Map
        // 7. HTML cleanup
    }

    parseCodeBlocks(html)    // Mermaid → mermaidStore Map + code highlight
    parseTables(html)        // Markdown tables → <table>
    parseLists(html)         // ul/ol with class-based conflict prevention
    highlightSyntax(code, lang)  // JS / Python / CSS / HTML coloring
}
```

**Parsing rule priority** (inside `this.rules`):
1. Headings H6 → H1 (most specific first)
2. Bold+italic, bold, italic
3. Strikethrough
4. Inline code
5. Links, images
6. HR
7. Paragraph breaks (`\n\n` → `</p><p>`)
8. Line breaks (`\n` → `<br>`)

---

### Class 2 — MarkdownEditor

Responsible for UI state, event handling, toolbar actions, persistence.

```javascript
class MarkdownEditor {
    constructor() {
        this.editor  = document.getElementById('editor')
        this.preview = document.getElementById('preview')
        this.parser  = new MarkdownParser()
        this.saveTimer    = null
        this.debounceTimer = null
        this.currentRows  = 3
        this.currentCols  = 3
    }

    // Lifecycle
    init()
    loadFromStorage()
    setupEventListeners()
    setupKeyboardShortcuts()   // Ctrl+B/I/K/H/S

    // Rendering
    updatePreview()
    renderMermaidDiagrams()    // reads from mermaidStore Map
    updateStats()
    updateCursorPosition()

    // Toolbar actions
    insertMarkdown(before, after)
    insertHeading()
    insertCode()
    insertCodeBlock()
    insertLink()
    insertImage()
    insertTable()              // opens tableModal
    insertList(type)
    insertQuote()
    insertHR()
    insertMermaid()            // inserts default flowchart snippet

    // Table builder
    showTableModal()
    updateTablePreview()
    generateTableMarkdown()

    // Persistence
    autoSave()
    showToast(message, type)

    // Utilities
    replaceSelection(replacement)
    replaceCurrentLine(newLine)
    clearContent()
    initializeMermaid()
}
```

---

### Global Functions

#### Toolbar delegates (call app.method)
```javascript
insertMarkdown(before, after)
insertHeading()
insertCode()
insertCodeBlock()
insertLink()
insertImage()
insertTable()
insertList(type)
insertQuote()
insertHR()
insertMermaid()
clearContent()
```

#### UI
```javascript
toggleTheme()          // [data-theme] toggle + localStorage + re-render
toggleFullscreen(panel)
toggleSpellcheck()     // V2 — toggles spellcheck attr + ABC button style
```

#### Table modal
```javascript
updateTableSize(type, delta)
addRow()
addCol()
closeTableModal()
insertTableToEditor()
```

#### Export
```javascript
exportMarkdown()   // downloads .md
exportHTML()       // downloads standalone .html
```

#### Formula Picker (V2)
```javascript
openFormulaPopup()     // shows formulaModal, resets state
closeFormulaPopup()    // hides formulaModal
fpShowCategory(id, tabEl)   // renders card grid for selected category
fpSelectCard(cardEl, latex) // loads latex into input + live preview
fpLiveUpdate()         // re-renders KaTeX preview from input value
insertFormula()        // wraps in $...$ or $$...$$ + calls app.replaceSelection()
```

---

### Global State

```javascript
const mermaidStore = new Map()   // id → raw Mermaid code (newline-safe)
let app                          // MarkdownEditor instance
let fpSelectedCard = null        // currently selected formula card element
const FP_CATEGORIES = [...]      // formula database — 8 categories, 90+ formulas
```

---

## Key Design Patterns

### 1. Debounced Input
```javascript
handleInput() {
    clearTimeout(this.debounceTimer);
    this.debounceTimer = setTimeout(() => {
        this.updatePreview();
        this.updateStats();
        this.autoSave();
    }, 100);
}
```

### 2. Map-based Mermaid Storage
`outerHTML` serialization loses newlines. Solution: store raw Mermaid code in a Map, write only an empty `<div id="...">` to HTML, read from Map at render time.
```javascript
mermaidStore.set(id, code.trim());
return `<div class="mermaid-wrapper" id="${id}"></div>`;
// later:
mermaidStore.forEach(async (code, id) => {
    const { svg } = await mermaid.render(svgId, code);
    wrapper.innerHTML = svg;
});
```

### 3. Placeholder-based LaTeX Parsing (V2)
Markdown `_italic_` rule was mutating LaTeX subscripts (`x_i` → broken). Fix: extract LaTeX into a placeholder Map before any markdown rules run, restore after.
```javascript
// Before markdown rules:
text = text.replace(/\$\$([\s\S]*?)\$\$/g, (match, formula) => {
    const key = 'LATEXBLOCK' + latexIdx++ + 'XEND';
    latexStore.set(key, katex.renderToString(formula.trim(), { displayMode: true }));
    return key;
});
// After all markdown rules:
latexStore.forEach((rendered, key) => {
    html = html.split(key).join(rendered);
});
```

### 4. Unified Modal Pattern
Both Table Builder and Formula Picker share the same CSS show/hide mechanism:
```javascript
document.getElementById('formulaModal').classList.add('show');    // open
document.getElementById('formulaModal').classList.remove('show'); // close
```
Overlay click and ESC key close any open modal.

### 5. Class-based List Conflict Prevention
Unordered and ordered list items are tagged with temporary classes before wrapping, preventing regex cross-contamination:
```javascript
// ul-item → wrap in <ul>
// ol-item → wrap in <ol>
// class attribute stripped after wrapping
```

### 6. Theme Cascade
Theme applied to `<html>` root element — `[data-theme="dark"]` selector reaches all nested elements including dynamically created modal content and KaTeX-rendered HTML.

---

## Initialization Flow

```
DOMContentLoaded
  └── app = new MarkdownEditor()
        ├── loadFromStorage()       restore last session
        ├── setupEventListeners()   input, scroll, click
        ├── setupKeyboardShortcuts()
        ├── updatePreview()         initial render
        └── updateStats()           initial counts

  └── fpShowCategory('basic', firstTab)   formula picker grid pre-built
  └── formulaModal overlay listener       ESC / overlay close
  └── tableModal overlay listener
```

---

## File Size Budget

| Section | Approx. lines |
|---------|--------------|
| CSS (styles) | ~700 |
| HTML structure | ~120 |
| MarkdownParser class | ~200 |
| MarkdownEditor class | ~400 |
| Formula picker data + functions | ~200 |
| Global functions + init | ~150 |
| **Total** | **~1800** |

---

## V2 Changes Summary

| What changed | Where |
|---|---|
| `parse()` — LaTeX extracted before rules | MarkdownParser |
| `parseLatexFormulas()` — removed (dead code) | MarkdownParser |
| KaTeX dark mode CSS fix | `<style>` block |
| Panel divider element | HTML structure |
| Panel tone differentiation CSS | `<style>` block |
| `formulaModal` HTML block | HTML structure |
| `FP_CATEGORIES` data + all `fp*` functions | Global scope |
| `toggleSpellcheck()` | Global functions |
| `Σ Formula` toolbar button | HTML toolbar |
| `ABC` spellcheck toolbar button | HTML toolbar |
| `spellcheck="false"` on textarea | HTML |

---

*Last updated: March 2026*
