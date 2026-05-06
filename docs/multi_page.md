# Docsify + MathJax: Setup & Findings

Notes on turning a GitHub repo of `.md` files into a rendered static site with working LaTeX formulas.

---

## Stack

| Tool | Purpose |
|------|---------|
| [Docsify](https://docsify.js.org) | SPA docs generator, no build step, hash routing |
| [MathJax 3](https://www.mathjax.org) | LaTeX math rendering in browser |
| GitHub Pages | Hosting from `main` branch root |

---

## Why Docsify

- Zero build step — just `index.html` + `.md` files
- GitHub Pages serves it directly (no Jekyll interference, requires `.nojekyll`)
- Hash routing (`/#/slides/lecture_01`) lets Docsify fetch `.md` files as needed
- Sidebar defined in `_sidebar.md`

---

## File Structure

```
repo-root/
├── .nojekyll          # prevents GitHub Pages from running Jekyll
├── index.html         # Docsify shell (all config here)
├── _sidebar.md        # navigation tree
├── README.md          # homepage (loaded at /#/)
└── slides/
    ├── lecture_01.md
    └── ...
```

---

## GitHub Pages Setup

1. Go to repo **Settings → Pages**
2. Source: `Deploy from a branch`, branch `main`, folder `/` (root)
3. Site is served at `https://<username>.github.io/<repo>/`
4. Add `basePath` in Docsify config to match that subpath (see below)

---

## index.html — Minimal Working Config

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>ML Notes</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/docsify/themes/vue.css">
  <style>
    .MathJax { overflow-x: auto; overflow-y: hidden; }
  </style>
</head>
<body>
  <div id="app"></div>

  <script>
    window.$docsify = {
      name: 'ML Notes',
      nameLink: 'https://your-portfolio.github.io',  // logo click → home
      basePath: '/ai_product_engineer/',              // must match GitHub Pages subpath
      loadSidebar: true,
      subMaxLevel: 3,
      autoHeader: true,

      plugins: [
        function(hook) {
          // Step 1 — protect math from marked (beforeEach operates on raw markdown)
          hook.beforeEach(function(content) {
            function enc(s) {
              return s.replace(/&/g, '&amp;').replace(/"/g, '&quot;');
            }
            content = content.replace(/\$\$([\s\S]*?)\$\$/g, function(_, math) {
              return '<div class="math-block" data-math="' + enc(math) + '"></div>';
            });
            content = content.replace(/\$([^\n$]+?)\$/g, function(_, math) {
              return '<span class="math-inline" data-math="' + enc(math) + '"></span>';
            });
            return content;
          });

          // Step 2 — restore math in HTML string (afterEach is synchronous, before DOM update)
          hook.afterEach(function(html) {
            function dec(s) {
              return s.replace(/&amp;/g, '&').replace(/&quot;/g, '"');
            }
            html = html.replace(/<div class="math-block" data-math="([^"]*?)"><\/div>/g, function(_, math) {
              return '<div class="math-block">$$' + dec(math) + '$$</div>';
            });
            html = html.replace(/<span class="math-inline" data-math="([^"]*?)"><\/span>/g, function(_, math) {
              return '<span class="math-inline">$' + dec(math) + '$</span>';
            });
            return html;
          });
        }
      ],
    };
  </script>

  <!-- Step 3 — trigger MathJax after every navigation -->
  <!-- doneEach hook is unreliable — use standalone events instead -->
  <script>
    (function() {
      function typeset() {
        if (window.MathJax && MathJax.typesetPromise) {
          MathJax.typesetPromise();
        }
      }
      window.addEventListener('load',       function() { setTimeout(typeset, 300); });
      window.addEventListener('hashchange', function() { setTimeout(typeset, 300); });
    })();
  </script>

  <!-- MathJax config MUST come before the MathJax script tag -->
  <script>
    window.MathJax = {
      tex: {
        inlineMath:  [['$', '$'], ['\\(', '\\)']],
        displayMath: [['$$', '$$'], ['\\[', '\\]']],
      },
      svg: { fontCache: 'global' },
    };
  </script>
  <script src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-svg.js"></script>

  <script src="https://cdn.jsdelivr.net/npm/docsify/lib/docsify.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/docsify/lib/plugins/search.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-python.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-bash.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/docsify-copy-code@2/dist/docsify-copy-code.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/docsify-pagination/dist/docsify-pagination.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/docsify-count/dist/countable.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/docsify-tabs@1/dist/docsify-tabs.min.js"></script>
</body>
</html>
```

---

## Why This Math Pattern

### Problem: `marked` corrupts LaTeX

Docsify uses `marked` to convert markdown to HTML. `marked` treats `_` as italic and `\` as escape sequences, which breaks formulas like `\frac{a}{b}` or `x_i`.

### Failed approach: KaTeX (`docsify-katex`)

Tried `docsify-katex` plugin — formulas were not rendered at all. Version pinning (`katex@0.16.9`, `docsify-katex@1.4.4`, `marked@4`) didn't help. Abandoned.

### Failed approach: `doneEach` hook

`doneEach` is the natural place to call `MathJax.typesetPromise()` after DOM update. It was confirmed registered (`window.$docsify.plugins.length === 6`) but **never fired** in this multi-plugin configuration. Root cause unclear — likely a plugin interaction with the Docsify event lifecycle.

Confirmed with:
```js
// In browser console after page load:
document.querySelectorAll('[data-math]').length  // → 69 (elements were NEVER cleaned up)
```

### Working approach: `beforeEach` + `afterEach` + standalone events

| Hook | Input | Output | When |
|------|-------|--------|------|
| `beforeEach` | raw markdown string | modified markdown | before `marked` runs |
| `afterEach` | rendered HTML string | modified HTML | after `marked`, before DOM update |
| `doneEach` | — | — | after DOM update (**unreliable here**) |

The pattern:

1. **`beforeEach`**: Replace `$$...$$` and `$...$` with `<div/span data-math="...">` — safe placeholder that `marked` won't touch
2. **`afterEach`**: Restore placeholders back to `$$...$$` / `$...$` in the HTML string — synchronous, reliable
3. **`load` + `hashchange` events**: Call `MathJax.typesetPromise()` 300 ms after each navigation — decoupled from Docsify's hook system entirely

The 300 ms delay gives Docsify time to write the restored HTML to the DOM before MathJax scans it.

---

## _sidebar.md Format

```markdown
* [Home](/)

* **Section Title**
  * [Page Label](path/to/file.md)
  * [Another Page](path/to/other.md)
```

- Paths are relative to `basePath`
- Docsify highlights the active link automatically
- `loadSidebar: true` must be set in config

---

## Script Load Order (matters)

```
window.MathJax = { ... }      ← config object FIRST
mathjax@3/es5/tex-svg.js      ← MathJax library (reads window.MathJax on load)
docsify.min.js                ← Docsify core
search.min.js                 ← Docsify plugins...
prismjs components
docsify-copy-code
docsify-pagination
docsify-count
docsify-tabs
```

MathJax must be configured and loaded **before** Docsify, otherwise the `load` event fires before MathJax is ready and the first render shows raw `$...$`.

---

## Debugging Checklist

| Symptom | Check |
|---------|-------|
| Sidebar empty / 404 | `_sidebar.md` missing or `loadSidebar: true` not set |
| `.md` files 404 | `basePath` wrong or missing |
| Formulas not rendered | See math section above |
| Formulas render on first load but not after navigation | `hashchange` listener missing |
| `data-math` attributes visible in DOM | `doneEach` not firing; switch to standalone events |
| Backslashes doubled in formulas | `marked` escaping — protect math in `beforeEach` |
| Jekyll 404 on GitHub Pages | `.nojekyll` file missing in repo root |
