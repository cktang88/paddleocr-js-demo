# paddleocr-js-test

A single-file HTML demo that runs [PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR) (PP-OCRv5) entirely in your browser — no server, no upload, your image never leaves the device.

Powered by [`@paddleocr/paddleocr-js`](https://github.com/PaddlePaddle/PaddleOCR/tree/main/paddleocr-js), ONNX Runtime Web, and OpenCV.js.

## Run it

You can't open `index.html` via `file://` — the SDK requires an HTTP origin so it can fetch model tarballs. Serve the folder locally:

```bash
python3 -m http.server 8000
```

Then open <http://localhost:8000/>.

Any static server works: `npx serve`, `php -S localhost:8000`, VS Code's Live Server, etc.

## How it works

- Engine auto-initializes on page load (downloads ~10 MB of PP-OCRv5 mobile detection + recognition models, cached by the browser).
- Drop a JPG/PNG/WebP/GIF/BMP on the drop zone (or click to pick).
- OCR runs automatically. Results show alongside the image (side-by-side for ≤10 lines, stacked below for more).

## CDN wiring (why it looks unusual)

`@paddleocr/paddleocr-js` is published as pure ESM and depends on `@techstark/opencv-js`, which no generic CDN can serve as browser-ESM (jsDelivr's `+esm` errors on the `fs` import; esm.sh force-injects Node polyfills that crash on `process.binding`). So the page:

1. Loads OpenCV's UMD build via a plain `<script>` tag (sets `globalThis.cv`).
2. Uses an import map to shim `@techstark/opencv-js` → a data-URL module that re-exports `globalThis.cv` as default.
3. Loads the SDK from esm.sh with `?external=@techstark/opencv-js` so it emits a bare import the map can intercept.

## Limitations

PaddleOCR.js only accepts raster images the browser can decode (`createImageBitmap`). No PDF, no DOCX, no multi-page TIFF. For PDFs you'd rasterize pages with PDF.js first and call `ocr.predict(canvas)` per page.
