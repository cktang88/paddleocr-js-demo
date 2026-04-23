# paddleocr-js-demo

A single-file HTML demo that reads text from just about any file — images, HEIC photos, PDFs, and Office documents (DOCX / XLSX / PPTX) — entirely in your browser. No server, no upload, your file never leaves the device.

Images, HEIC photos, and PDF pages go through [PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR) (PP-OCRv5). Office documents read their native text directly; no OCR needed.

## Run it

You can't open `index.html` via `file://` — the SDK requires an HTTP origin so it can fetch model tarballs. Serve the folder locally:

```bash
python3 -m http.server 8000
```

Then open <http://localhost:8000/>.

Any static server works: `npx serve`, `php -S localhost:8000`, VS Code's Live Server, etc.

## How it works

The engine auto-initializes on page load (downloads ~10 MB of PP-OCRv5 mobile models, cached by the browser). Drop a file and it routes by type:

| File | Pipeline |
| --- | --- |
| JPG / PNG / WebP / GIF / BMP / TIFF | PaddleOCR directly |
| HEIC / HEIF | [heic2any](https://github.com/alexcorvi/heic2any) → PNG → PaddleOCR |
| PDF | [PDF.js](https://mozilla.github.io/pdf.js/) rasterizes each page → PaddleOCR per page |
| DOCX | [mammoth.js](https://github.com/mwilliamson/mammoth.js) extracts raw text |
| XLSX | [SheetJS](https://sheetjs.com/) exports each sheet as CSV |
| PPTX | [JSZip](https://stuk.github.io/jszip/) unpacks slides, DOMParser pulls `<a:t>` runs |

Heavy libraries (PDF.js, mammoth, SheetJS, JSZip, heic2any) load lazily the first time you drop a matching file — the base page stays small.

## Reading the results

- **Copy all** — one click copies the visible recognized text to the clipboard, newline-separated.
- **Low-confidence filter** — rows with PaddleOCR score `< 0.5` are hidden by default (they're usually noise). If any were hidden, a `Show N low-confidence rows` toggle appears beside the copy button.
- **Transcribe with VLM** — when the image is a photo/scene rather than a clean document, PaddleOCR can struggle. Pick a small vision-language model from the dropdown and click the button to re-transcribe the same image with [SmolVLM](https://huggingface.co/HuggingFaceTB/SmolVLM-256M-Instruct) via [Transformers.js](https://github.com/huggingface/transformers.js). Runs on WebGPU where available, falls back to WASM. Model weights download on first use (~180 MB for SmolVLM-256M, ~500 MB for SmolVLM-500M) and are cached by the browser.

## CDN wiring (why it looks unusual)

`@paddleocr/paddleocr-js` is published as pure ESM and depends on `@techstark/opencv-js`, which no generic CDN can serve as browser-ESM (jsDelivr's `+esm` errors on the `fs` import; esm.sh force-injects Node polyfills that crash on `process.binding`). So the page:

1. Loads OpenCV's UMD build via a plain `<script>` tag (sets `globalThis.cv`).
2. Uses an import map to shim `@techstark/opencv-js` → a data-URL module that re-exports `globalThis.cv` as default.
3. Loads the SDK from esm.sh with `?external=@techstark/opencv-js` so it emits a bare import the map can intercept.

The other libraries come from esm.sh and jsDelivr without special handling.

## Limitations

- **Office fidelity** is text-only — embedded images are not OCR'd, and layout/styling is dropped. If you need LibreOffice-grade rendering you'd have to fall back to a server or [ZetaOffice](https://zetaoffice.net/) (WASM LibreOffice, ~1 GB).
- **PDFs with scanned-only pages** are fine — they'll OCR page-by-page. Native-text PDFs also go through OCR rather than PDF.js's text layer, so selectable PDFs are slower than they'd need to be. Worth switching to PDF.js `getTextContent()` when the page has a text layer; skipped for now to keep the demo simple.
- **PaddleOCR.js** accepts `ImageBitmap | Blob | HTMLCanvasElement | ImageData | HTMLImageElement`; anything we can't decode via `createImageBitmap` gets converted first (that's what HEIC does).
