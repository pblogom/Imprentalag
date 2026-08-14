# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Herramientas de imprenta" — a small set of standalone, single-file HTML tools for a print shop's (imprenta's)
day-to-day work: quoting jobs, imposing/repeating artwork on a sheet, adding bleed, checking client images for
legibility before printing, etc. Everything is in Spanish (Argentina), for print-shop staff and, in one case,
end clients.

There is no build system, no package manager, no bundler, and no test suite. Each `.html` file is a complete,
self-contained app: inline `<style>`, inline `<script>`, no imports between files. `index.html` is just a
directory of links to the other tools.

## Running / developing

There's nothing to install or build. Open any `.html` file directly in a browser, or serve the directory
statically, e.g.:

```
python3 -m http.server 8000
```

then visit `http://localhost:8000/index.html`. To work on one tool, just edit its file and reload the page —
there's no dev server, hot reload, linter, or test runner configured. Verify changes manually in the browser
(the `run` skill can drive this if asked to check a change works).

## Adding a new tool

1. Create `nombre-de-la-herramienta.html` as a fully self-contained file (own `<style>`, own `<script>`),
   following the structure and CSS variable conventions below.
2. Add a card for it in `index.html`'s list of `a.herramienta` links, matching the existing title +
   description + "Abrir →" pattern.

## Shared conventions across tools

- **Language**: all UI copy, variable/function names in comments, and toasts/messages are in Spanish. Keep new
  code consistent with that (e.g. `function calcular()`, `function pintarLista()`, `const S={...}` for state).
- **Fonts**: Google Fonts `Archivo` (variable weight/width) for body text and `IBM Plex Mono` for
  numeric/technical labels (measurements, prices, mono-spaced UI chrome), loaded via `<link>` in `<head>`.
- **Color system**: two generations of the same palette exist as CSS custom properties on `:root`. Match
  whichever a file already uses rather than mixing them:
  - Older/simpler palette (`index.html`, `armador-pliegos.html`, `preparador-volantes.html`,
    `revisa-tu-imagen.html`): `--papel`, `--tinta`, `--tinta-suave`, `--magenta`, `--linea`, `--blanco`, plus
    `--rojo`/`--ambar`/`--verde` for status states.
  - Newer/refined palette (`presupuestos.html`, `repetir-en-hoja.html`, `agregar-sangrado.html`): `--paper`,
    `--paper-2`, `--ink`, `--ink-2`, `--ink-3`, `--line`, `--line-2`, `--magenta` + `--magenta-soft`,
    `--green`/`--green-soft`, `--amber`/`--amber-soft`. Prefer this palette for new tools.
- **No frameworks**: plain DOM APIs (`document.querySelector`, template strings for rendering lists), no React/
  Vue/jQuery. Common short helpers recur per-file: `$=s=>document.querySelector(s)`, `pesos=n=>'$'+...` for
  currency formatting, `toast(msg)` for transient notifications.
- **External libs are loaded from CDN only when needed**, either via a static `<script src="https://cdnjs...">`
  tag or lazily through a `cargarScript(src)` helper that injects a `<script>` tag on demand (used for
  `tesseract.js`, so the ~heavy OCR engine isn't downloaded unless the user actually runs a legibility check):
  - `pdf.js` (cdnjs, pinned `3.11.174`) — rendering the first page of a client-supplied PDF to a canvas at a
    given DPI (typically `page.getViewport({scale: DPI/72})`).
  - `jsPDF` (cdnjs, pinned `2.5.1`) — assembling the final print-ready output as a PDF sized in millimeters
    (`new jsPDF({unit:'mm', format:[w,h], ...})`), so WhatsApp/email won't recompress it as an image.
  - `tesseract.js` (cdnjs, pinned `5.1.0`) — client-side OCR (Spanish worker, `Tesseract.createWorker('spa',1,...)`)
    used to detect text regions/legibility in artwork before printing.
  - All processing (image decode, canvas compositing, OCR, PDF generation) happens client-side in the browser;
    nothing is uploaded to a server.
- **Persistence**: only `presupuestos.html` persists data, via a small `store` wrapper around `localStorage`
  (`store.get(k)`/`store.set(k,v)`, JSON-encoded, wrapped in try/catch since `localStorage` can throw). That's
  where quote history, price-list overrides, and settings live.
- **Measurement conventions**: canvas/image work is done in pixels at an explicit DPI (almost always 300),
  converted to/from millimeters with helpers like `mmapx(mm) = Math.round(mm/25.4*DPI)`. Bleed (`sangrado`) and
  crop-mark length are typically small mm constants near the top of the script (e.g. `DPI=300, MARCA=3`).

## Tool-by-tool architecture notes

- **`presupuestos.html`** (largest/most complex file): a quoting/pricing calculator.
  - Hardcoded price data/curves near the top (`FUENTES`, `COLOR`, `BN`, `MEGA`, `TALON`, `TARJ`, tramo labels
    `T6`/`O6`) represent different suppliers' price lists and tiered pricing curves (`curvaPrecio(curva, cant,
    unidad)` looks up the right price tier for a quantity).
  - `CATS` defines the catalog of quotable item categories/products; `calcular()` is the core pricing engine
    that reads the current form state and produces a price breakdown.
  - Inflation adjustment: `factor(fecha)` compounds a configurable annual inflation rate (`CFG.inflAnual`)
    over the months elapsed since a price list's last-updated date, to keep old hardcoded prices roughly
    current without manual edits.
  - `overrides` let a user manually pin/override a computed line price (`over(id)`, `apagado(id)`).
  - A "cesta" (cart) of quote line items can be built up, saved as a whole quote to the `store`-backed registry
    (`renderRegistro`), and rendered as WhatsApp-ready text (`textoWA`, `compartir`) for sending to a client.
  - Ajustes (Settings) tab lets staff edit the price curves/multipliers at runtime (`renderCurvas`,
    `leerCurvas`, `cargarAjustes`) — this is the intended way to update prices, not editing the hardcoded
    constants directly, though the constants are the fallback/default (`DEF_CURVAS`).

- **`armador-pliegos.html`**: takes several individual flyers (uploaded as JPG or PDF, one per WhatsApp message
  from a client) and imposes them onto a fixed press sheet layout — either 9-up at 10×15cm on a 32.5×47.5cm
  sheet (full color) or 4-up at 11×17cm on a 22×34cm sheet (one color) — with crop marks, then exports as a
  single print-ready file. `PLIEGOS` defines the available sheet layouts (`cols`/`filas`); pieces are collected
  into a list and auto-distributed to fill sheet capacity (`autoDistribuir`, `capacidad`).

- **`repetir-en-hoja.html`**: given one flyer design plus its finished size and target press sheet, tiles as
  many copies as fit, sharing bleed between adjacent copies (so crop marks land exactly on the shared cut
  line) and adding an outer bleed border (mirrored, stretched, or solid-color via an eyedropper —
  `tomarColor`). Key pipeline: `detectarBlanco`/`aplicarTrim` auto-detects and trims a white border the client's
  file may already have; `armarPieza`/`completar`/`banda` build one tile plus its bleed band; `calcular` works
  out how many tiles fit the sheet and how much waste/loss results (`perdida`); a draggable magnifier
  (`dibujarLupa`/`moverLupa`) lets the user inspect the result at full resolution before exporting.

- **`agregar-sangrado.html`**: when client artwork already extends to the trim edge with no room to add bleed,
  this mirrors a 5mm band around each edge (or one of a couple of alternate edge-fill methods) and exports a
  print-ready PDF sized to the original + bleed.

- **`preparador-volantes.html`**: takes a client's flyer image, downsamples/crops it to a chosen finished size
  (default A5) at 300 dpi as a much lighter JPG, and — via `tesseract.js` OCR — flags which text regions will
  likely become illegible at that size/resolution before the file goes to print.

- **`revisa-tu-imagen.html`**: the client-facing counterpart to `preparador-volantes.html` — a link meant to be
  sent to a client whose artwork was AI-generated. Runs the same OCR-based legibility check in plain language,
  and if the image fails, generates a corrective prompt (`generarPrompt`) the client can feed back into their
  AI tool. On success, exports a print-ready PDF and can hand off directly to WhatsApp
  (`wa.me/<WHATSAPP_IMPRENTA>`, `btn-whatsapp` handler) addressed to the print shop. `WHATSAPP_IMPRENTA` near
  the top of the script is the shop's number (country code, no `+`/spaces) — currently left blank, which falls
  back to opening WhatsApp's contact picker instead of a fixed number.
