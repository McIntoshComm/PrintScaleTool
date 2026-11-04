# S.P.L.O.T.: project notes

**S.P.L.O.T.** (Scale Plot Layout Organizer Tool), first built as "DXF Plot Tiler", takes a large CAD drawing, scales it to a true architectural scale, and splits it across as many printable sheets as it needs.

- **Location:** `C:\Users\Jase\Documents\projects\Print_scale_tool\`
- **Also published as:** a private page in Claude
- **Built:** October 27–28, 2026

---

## 1. How it started

### The original job

The drawing was **AI-Xflr.dxf**, a floor plan of a large industrial site. It had to be printed at **1/16" = 1'-0"** on **30" × 42" (ARCH E1)** paper, keeping true scale.

What we found about the drawing:

| Item | Value |
|---|---|
| Units | Inches (architectural) |
| Visible plan size | about 2,745' × 1,523' |
| Size on paper at 1/16" = 1'-0" | about 171.5" × 95.2" |
| Odd detail | `$EXTMIN` reaches far to the left of the plan because of stray objects, so it can't be used as the plan's edge |
| Check | A 24'-wide door measured 25' in the drawing, which confirmed the drawing is at real size |

### Settling the scale

- A literal **1:16** (3/4" = 1'-0") would have needed about **2,000 sheets**.
- "One sixteenth scale" in drafting means **1/16" = 1'-0"**, which is **1:192**. That gave **15 sheets**.

### Why CADCraft couldn't do it

CADCraft's web version (cadcraft.org) failed on every PDF route. Plot and Export to PDF both stopped with:

> "paths are not available on the web"

Its Scale box also rounds 1/192 to 0.0052, which is slightly off. So the first PDF was made by pulling the DXF out of the app and drawing each sheet with custom code in the browser. That code became the app.

### The result

- 15 portrait ARCH E1 sheets plus a key plan
- 6 columns × 3 rows, with 3 empty tiles skipped
- Each sheet has a label and a scale bar

---

## 2. How the app grew

| Step | What was added |
|---|---|
| 1 | First tiled PDF made by hand inside the CADCraft tab |
| 2 | Written instructions for doing it manually: window coordinates and Fit to paper |
| 3 | **The web app:** open a DXF, choose scale, paper and orientation, preview the sheets, make the PDF |
| 4 | Self-hosting: one `index.html` saved to `Print_scale_tool` |
| 5 | **Opening DWG files** (`dwg-converter.js`, LibreDWG) |
| 6 | **DXF export**, one file per sheet in a zip; key-sheet and matchline checkboxes |
| 7 | **DWG export** (`dwg-writer.js`, acad-ts) with a version menu; light/dark toggle with sun and moon icons |
| 8 | Renamed **S.P.L.O.T.**: Scale Plot Layout Organizer Tool |

---

## 3. What the app does

1. **Open** a `.dxf` or `.dwg` file by dropping it on the page or choosing it. Everything happens in the browser, and nothing is uploaded.
2. **Scale:**
   - Architectural presets from 1/32" to 3" = 1'-0"
   - Engineering presets from 1" = 10' to 1" = 100'
   - Ratio presets, or a custom ratio
   - Drawing units are read from `$INSUNITS`, and you can change them.
3. **Sheet:**
   - ARCH, ANSI and ISO sizes, or a custom size
   - Orientation: Fewest (picks whichever needs fewer sheets), Portrait or Landscape
   - Overlap between sheets, in paper inches
4. **Preview:** the drawing with the sheet grid over it. Tap a sheet to leave it out or put it back. Empty sheets are skipped automatically.
5. **Export (any mix):**
   - **PDF:** all sheets in one file
   - **DXF:** one file per sheet
   - **DWG:** one file per sheet, AutoCAD 2000 to 2018 format, 2013 by default
   - Picking more than one gives one zip.
6. **Sheets include:**
   - Key sheet
   - Matchlines labelled "MATCH LINE - SEE SHEET n"
   - Sheet label and scale bar
   - Overlap marks
   - Black lines in the PDF
7. **Light/dark mode** button. It remembers your choice in that browser.

---

## 4. Files

```
Print_scale_tool/
├── index.html            The app: HTML, CSS and all core JavaScript in one file (~72 KB)
├── dwg-converter.js      Opens DWG files: LibreDWG compiled to WebAssembly (~3.4 MB)
├── dwg-writer.js         Exports DWG files: acad-ts bundled for the browser (~1.2 MB)
├── acad-ts-LICENSE.txt   MIT licence for dwg-writer.js
└── README.txt            How to run it, plus licence notes
```

- The two `.js` helpers load **only when needed**. Without them, PDF and DXF still work.
- Everything is a **classic `<script>`**, with no ES modules and no separate `.wasm` file. That's why it works when you double-click `index.html` (`file://`) as well as on any web server.
- The only outside request is Google Fonts. If it's blocked, the page uses system fonts.

---

## 5. How it works

### 5.1 The scale math

All sizes are worked out in **paper inches per drawing unit**:

```
k = (inches per drawing unit) / ratio
    e.g. inches drawing, 1/16" = 1'-0"  →  k = 1 / 192

one sheet covers  (paper width / k) × (paper height / k)  drawing units
    30 × 42 in at 1:192  →  5,760" × 8,064"  (480' × 672')

columns = ceil((drawing width on paper  − overlap) / (sheet width  − overlap))
rows    = ceil((drawing height on paper − overlap) / (sheet height − overlap))
```

- The grid is **centred** on the drawing.
- Sheets that touch no line work are marked empty and skipped.
- In the PDF, 1 drawing unit = `k × 72` points. The page is exactly the sheet size, so the scale comes out exact with no rounding.

### 5.2 Reading the DXF (custom parser in `index.html`)

1. **Split the file** into group-code/value pairs, held in a typed array plus an array of strings.
2. **Read the sections:**
   - **HEADER:** `$INSUNITS`, `$LTSCALE`
   - **TABLES:** layers (colour, on/off, frozen, plot flag, line type, line weight) and line types (dash patterns)
   - **BLOCKS:** block definitions and their base points
   - **ENTITIES:** model space only (paper-space items, code 67, are skipped)
3. **Turn every entity into simple shapes in world coordinates:**

| Entity | How it's handled |
|---|---|
| LINE, LWPOLYLINE, POLYLINE | Polylines; arcs written as bulges become line segments |
| ARC, CIRCLE, ELLIPSE | Line segments, about one every 5° |
| SPLINE | Worked out exactly (de Boor, including weights); falls back to fit points or control points |
| HATCH (solid only) | Filled outline from polyline, line or arc edges |
| SOLID, TRACE, 3DFACE, LEADER | Filled shapes or lines |
| TEXT, ATTRIB, MTEXT | Text with alignment; MTEXT formatting codes removed, `%%c` / `%%d` / `%%p` converted |
| INSERT, MINSERT, DIMENSION | Block contents placed with their own position, rotation and scale, nested up to 16 deep; mirrored entities (extrusion −Z) handled |

4. **Work out each shape's style**, following AutoCAD's ByLayer and ByBlock rules: layer, colour, line type, line type scale and line weight. Layers that are off, frozen or set not to plot are left out.

### 5.3 Making the PDF (hand-written, no library)

- **Lines:** PDF 1.4 path commands (`m`, `l`, `S`, `f`)
- **Line types:** the DXF dash patterns × `$LTSCALE` × each entity's line type scale
- **Line weights:** hundredths of a millimetre converted to points (default 0.25 mm)
- **Text:** built-in Helvetica fonts. Alignment uses an estimated text width of 0.52 × text height.
- **Compression:** each page is compressed with the browser's `CompressionStream("deflate")`
- **Extras:**
  - Key sheet: the whole drawing, scaled to fit, with red sheet outlines and numbers
  - Labels and scale bar: a white box in the bottom-left corner of each sheet
  - Matchlines: a heavy dash-dot line on each edge shared with another printed sheet, labelled. It sits mid-overlap, or on the edge when there's no overlap.

### 5.4 Making the DXF and DWG files

Both formats are built from the same **sheet model**:

1. Keep only the shapes that touch the sheet.
2. Cut lines at the sheet edge (Liang–Barsky line clipping).
3. Cut filled shapes at the sheet edge (Sutherland–Hodgman clipping).
4. Keep text only if its insertion point is on the sheet.
5. Add:
   - `SHEET-BORDER`: the plot window rectangle
   - `SHEET-LABEL`
   - `MATCHLINE`
   - `SHEET-GRID`, on the key sheet only

**DXF writer** (custom, in `index.html`):

- Writes **R12 (AC1009)** DXF, the oldest and most widely readable kind
- Layer and line-type names are cleaned to letters, numbers, `$`, `_` and `-`, at most 31 characters
- Filled areas become `SOLID` triangles (ear-clipping); if that fails, they fall back to outlines
- Special characters are written as `\U+XXXX`

**DWG writer** (`dwg-writer.js`):

- Builds a drawing with **acad-ts** using these entities:
  - `Line`
  - `LwPolyline`
  - `TextEntity`
  - solid `Hatch` (smaller than DXF's triangles)
- Also sets:
  - real layers and line types
  - `$INSUNITS` and `$LTSCALE`
- Then saves it with `DwgWriter` in the chosen AutoCAD version

**Zip** (custom, in `index.html`):

- Standard zip format with CRC-32 checksums
- Files compressed with `CompressionStream("deflate-raw")` when it makes them smaller
- Includes `How to plot these sheets.txt`, which lists each sheet's plot-window corners

### 5.5 Opening DWG files

1. `dwg-converter.js` unpacks LibreDWG (WebAssembly, stored inside the file as compressed text).
2. LibreDWG's `dwg_write_dxf` converts the DWG to DXF in memory.
3. The DXF parser above takes over.

### 5.6 Saving files

- **Inside Claude:** the page asks permission to save through Claude's **downloads** feature.
- **Self-hosted:** a normal browser download.

---

## 6. How the helper files were built

These steps ran in Claude's cloud workspace. Re-run them only if you're updating a library.

### dwg-converter.js (opens DWG)

1. Download `@mlightcad/libredwg-web@0.8.2` from npm.
2. Take `wasm/libredwg-web.js` (Emscripten glue code) and `wasm/libredwg-web.wasm` (10.8 MB).
3. Edit the glue so it runs as a classic script:
   - replace `import.meta.url` with `self.location.href`
   - remove `export default`
4. Compress the `.wasm` (2.5 MB) and store it inside the file as base64 text.
5. Wrap it all in a self-contained function that creates `window.DWG2DXF.convert(arrayBuffer)` and returns `{ dxf: Uint8Array }`. It passes the WebAssembly in through `wasmBinary`, so it never has to fetch a file.

### dwg-writer.js (exports DWG)

1. Download `@node-projects/acad-ts@3.6.0` from npm (MIT).
2. **Speed fix:** in `dist/IO/DWG/DwgStreamWriters/DwgObjectWriter.js`, change `_writeBlockHeader` to use a `Set` and a `Map` instead of `Array.includes` and `Array.find`.
   - The original slowed down sharply as drawings got bigger.
   - With the fix, the busiest sheet went from **150 s to about 8 s**.
3. Bundle it with esbuild 0.24.2:

```bash
# entry.js
export { DwgWriter, DwgReader, DxfReader, DxfWriter, ACadVersion, CadDocument,
         Line, LwPolyline, LwPolylineVertex, Solid, TextEntity, Layer, LineType,
         LineTypeSegment, Color, XY, XYZ, Hatch, HatchBoundaryPath,
         HatchBoundaryPathPolyline, HatchPattern, TextHorizontalAlignment
       } from '@node-projects/acad-ts';

esbuild entry.js --bundle --format=iife --global-name=AcadTs \
  --minify --target=es2020 --platform=browser --outfile=acad.min.js
```

4. Add a licence comment at the top and `window.AcadTs=AcadTs;` at the end.

### index.html

- During development, the page was written as a Claude artifact. The self-hosted version wraps it in a standard `<!doctype html>` page.
- The page has no build step: the HTML, CSS and JS are all written in the one file.

---

## 7. How it was tested

- **AI-Xflr.dxf:**
  - 15 sheets at 1:192, matching the first hand-made PDF
  - All three exports with matchlines and a 1" overlap: about 57 s, 44 MB zip
- **Checking the PDFs:** pages rendered with `pdftoppm` and inspected. Page sizes confirmed with `pdfinfo` (2160 × 3024 pt = 30 × 42 in).
- **Checking the DXFs:** every sheet opened with `ezdxf` (`recover.readfile`) with no errors, and sheets were rendered to images to check matchlines and labels.
- **Checking the DWGs:**
  - Opened with a separate program (LibreDWG) and converted back to DXF with no errors
  - Entity counts matched the DXF version of each sheet
- **Opening DWGs:** LibreDWG's sample files saved as AutoCAD 2000, 2004, 2007, 2010, 2013 and 2018 all opened, each in under a second.
- **Ways of running it:** checked with Playwright and Chromium, both at `http://localhost` and by opening the file directly (`file://`). Phone-width layout checked: the page doesn't scroll sideways.
- **Not tested yet:** opening the exported DWG files in AutoCAD or CADCraft.

---

## 8. Known limits

| Area | Limit |
|---|---|
| Fonts | Text uses Helvetica, not your CAD fonts. Alignment uses estimated text width. |
| Hatches | Only solid fills come through; patterned hatches are skipped. |
| Not drawn | Images, wipeouts, points, XLINEs and RAYs |
| DXF export | R12 has no line weights. Blocks are broken into plain lines. Fills become many triangles, so busy sheets get large: sheet 12 is 31 MB. |
| DWG export | Blocks are broken into plain lines. The DWG is built from scratch, so blocks, styles and dimension styles from the source drawing aren't kept. |
| Matchlines with no overlap | They sit right on the paper edge, which most plotters can't print. Use an overlap of 1" or more. |
| Speed | Each sheet is built in one go, so the page can pause briefly on very busy sheets. |
| Opening DWGs | Very large or unusual DWGs might not convert. If so, save as DXF from your CAD program. |
| Browsers | Needs a recent Chrome, Edge, Firefox or Safari (2023+) for compression and DWG unpacking. |

---

## 9. Licences

| Component | Licence | Notes |
|---|---|---|
| `index.html` (app code) | Yours | Written for this project |
| `dwg-writer.js`: acad-ts 3.6.0 (port of ACadSharp) | MIT | Free to share; keep `acad-ts-LICENSE.txt` with it |
| `dwg-converter.js`: LibreDWG + @mlightcad/libredwg-web 0.8.2 | GPL v3 | Fine for your own use. If you share the app with this file included, the GPL v3 terms apply to that copy. |

**Idea for later:** acad-ts can also **read** DWG files. Switching DWG opening to acad-ts would remove the GPL part completely, but it would need testing on real drawings first.

---

## 10. Ideas for next steps

- Open the exported DWGs in AutoCAD or CADCraft and adjust the default version if needed.
- Use DWG layouts (one paper-space tab per sheet) instead of cut-out files.
- Do the slow work in the background so the page doesn't pause on huge drawings.
- Add a colour PDF preset, a choice of title block, or a sheet-name prefix.
- Switch DWG opening to acad-ts (MIT) to drop the GPL part.
