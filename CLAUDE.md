# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-file browser-based PDF editor. No build step, no dependencies to install. Open `index.html` directly in a browser or serve it via GitHub Pages at `https://jingzhi22.github.io/pdf-editor/`.

`index.html` is the only file. GitHub Pages serves it at the repo root; open it directly in a browser for local development.

## Architecture

Everything lives in one HTML file. Three CDN libraries do all the heavy lifting:

| Library | Version | Role |
|---|---|---|
| PDF.js 3.11.174 | cdnjs | Renders PDF pages to `<canvas>` elements for display |
| Fabric.js 5.3.0 | cdnjs | Interactive overlay canvas per page — handles object selection, drag, resize, rotate |
| pdf-lib 1.17.1 | jsdelivr | Loads original PDF bytes and writes text/images into it on export |

### Two-canvas layer per page

Each PDF page creates two stacked canvases inside `.page-wrap`:
1. `.pdf-c` — PDF.js renders the page background here (static, never touched again)
2. Fabric canvas inside `.fab-hold` — sits `position: absolute` on top, captures all user interaction

### Critical: ArrayBuffer ownership

PDF.js transfers its input `ArrayBuffer` to a Web Worker (detaching it). Always pass `pdfBytes.slice()` to PDF.js and keep `pdfBytes` untouched for pdf-lib to use at export time.

### Coordinate conversion (canvas → PDF)

Fabric origin is top-left; PDF origin is bottom-left. On export:
- `pdfX = obj.left / scale`
- `pdfY = pageHeight - (obj.top + fontSize * 0.95) / scale`  (text baseline approx)
- `pdfY = pageHeight - (obj.top / scale) - objHeightInPoints`  (images)
- `scale = 1.5` (render scale stored per-page in the `pages[]` array)

### Font mapping

Canvas fonts → pdf-lib standard fonts via `resolveFont()`. Five canvas font options collapse to three PDF families: Helvetica (Helvetica + Arial), Times (Times New Roman + Georgia), Courier (Courier New). Bold/italic pick the matching `StandardFonts` variant.

### Mode state machine

`mode` is `'select' | 'text' | 'image'`. `setMode()` updates `document.body.className`, button `.active` states, and Fabric's `canvas.selection` flag. Fabric `mouse:down` handlers check `mode` to decide whether to place a new object or defer to Fabric's built-in selection.

### Image export

Images must carry their original data URL. `fabric.Image` objects get `img.imageData = src` set at placement time; `exportPDF()` reads this property back. `obj._element.src` is a fallback.

## Deployment

```bash
# after editing index.html:
git add index.html && git commit -m "..." && git push
```

GitHub Pages auto-deploys from the `master` branch root (~60 s delay).
