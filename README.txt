S.P.L.O.T. - Scale Plot Layout Organizer Tool
=============================================

Plots a large DXF or DWG drawing at true scale across as many sheets as it
needs. Exports:
- PDF: all sheets in one file, with optional key sheet, matchlines,
  sheet labels and scale bars.
- DXF and/or DWG: one file per sheet (that sheet's area cut from the
  drawing at full size), plus an optional key sheet and plotting notes,
  in one .zip. DWG can be AutoCAD 2000 to 2018 format (2013 by default).
If you tick more than one, they all go in one .zip.

Files
-----
index.html         The app. Open it in a browser or put it on any web server.
dwg-converter.js   Lets the app OPEN DWG files (about 3.4 MB).
dwg-writer.js      Lets the app EXPORT DWG files (about 1.2 MB).
acad-ts-LICENSE.txt  Licence for dwg-writer.js.
Keep all of these in the same folder as index.html. Each one is only loaded
when needed; without them the app still works for DXF and PDF.

Running it
----------
- On your computer: double-click index.html.
- On a web server: copy all the files into the same folder of any static host
  (nginx, Apache, IIS, GitHub Pages, Netlify, Cloudflare Pages).
  Quick test: run  python -m http.server 8080  in this folder and open
  http://localhost:8080

Drawings are processed entirely in the browser. Nothing is uploaded.
Always print the PDF at 100% / Actual size.

Licences
--------
dwg-writer.js contains acad-ts 3.6.0 (https://github.com/node-projects/acad-ts),
a TypeScript port of ACadSharp, under the MIT licence (acad-ts-LICENSE.txt).
It was bundled for the browser with one small speed fix for large drawings.

dwg-converter.js contains LibreDWG (https://www.gnu.org/software/libredwg/),
compiled to WebAssembly by @mlightcad/libredwg-web 0.8.2
(https://github.com/mlightcad/libredwg-web). Both are licensed under the
GNU GPL v3. Using it yourself is fine. If you share or distribute the app
with dwg-converter.js included, the GPL v3 terms apply to that distribution.
LibreDWG reads most DWG versions from R13 to 2018+. If a file won't convert,
save it as DXF from your CAD program and open that instead.
