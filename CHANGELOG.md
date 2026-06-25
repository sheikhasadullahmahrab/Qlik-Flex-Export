# FlexExport — Changelog & Development Journey

**Author:** Sheikh Asadullah Mahrab  
**Email:** sheikhasadullahmahrab@gmail.com  
**Repository:** https://github.com/sheikhasadullahmahrab/Qlik-Flex-Export

---

## v1.2.0 — Current Release
**Theme: Compressed XLSX output via raw XML pipeline. No row limits.**

### What changed
- Replaced SheetJS worksheet object with a raw XML builder (`buildSheetBytes`)
- Introduced Shared String Table (SST) — unique strings stored once, cells store integer indexes
- All XML built as `Uint8Array` chunks, merged with `.set()` — bypasses V8 512MB string limit
- Integrated `fflate` library for proper deflate ZIP compression of XLSX parts
- SST compressed at level 9, worksheet at level 6 — matches Qlik's native compression ratio
- Date-time stamped filenames (`YYYYMMDD_HHmmss.xlsx` / `.csv`)
- Optional file name prefix in properties panel
- Named and branded as **FlexExport**

### Challenges solved
- **V8 string length limit (`Invalid string length`)** — `Array.join('')` on 1M row XML exceeded V8's ~512MB string cap. Fixed by encoding each row individually to `Uint8Array` and concatenating byte arrays instead of strings.
- **V8 object property limit (`Too many properties to enumerate`)** — SheetJS `aoa_to_sheet` creates one JS object property per cell. At 21 cols × 1M rows = 21M properties this crashed. Fixed by never creating a worksheet object at all — raw XML instead.

### Expected output
- 1,043,133 rows × 21 columns
- Fetch time: ~150s (parallel pages)
- File size: 40–80MB XLSX (vs 320MB uncompressed, vs 80MB Qlik native)

---

## v1.1.0
**Theme: Parallel fetching — significant speed improvement.**

### What changed
- Parallel page fetching with configurable concurrency (default 6, max 12)
- Increased default page size from 100 to 500 rows
- Pre-allocated result slots for in-order row assembly despite parallel arrival
- Added `concurrency` property to the properties panel
- Both `pageSize` and `concurrency` exposed and tunable by end user

### Performance
- Fetch time reduced from 325s (v1.0 sequential) to ~150s (v1.1 parallel, 6× concurrency)
- 1,044 pages × 500 rows → ~209 parallel batches of 6

### Challenges solved
- **Out-of-order page arrival** — parallel fetches complete in unpredictable order. Fixed with `resultSlots[pageIndex]` pre-allocation; rows flattened in index order after all pages complete.

---

## v1.0.0
**Theme: Working baseline — session objects, paginated fetch, CSV output.**

### What changed
- Replaced `app.getObject()` (persistent hypercube) with `app.model.enigmaModel.createSessionObject()` — eliminates "Result too large" error caused by Qlik Cloud pre-computing the full hypercube on object open
- `qInitialDataFetch: []` — zero rows pre-fetched at paint time
- Sequential paginated fetch, 100 rows per page
- Progress bar with live row counter
- Edit-mode click guard — button disabled during sheet editing
- Full console logging at every step (`[CEB]` prefix, F12 visible)
- Auto-switch to CSV for datasets > 200K rows (later raised to Excel's true limit)
- Chunked Blob CSV writer (no single giant string)
- SheetJS bundled inline (avoids Qlik Cloud RequireJS path resolution failures)

### Challenges solved in v1.0 development (cumulative from prototype)

| Error | Cause | Fix |
|---|---|---|
| `XLSX is not a function` | RequireJS AMD conflict with SheetJS | Bundled SheetJS inline as IIFE, sets `window.__ceb_XLSX` |
| `Unexpected token '<'` | `require.toUrl()` resolved to HTML 404 page | Eliminated external file — inline bundle only |
| `xlsx loaded but XLSX.utils missing` | Named AMD module `"xlsx"` not resolving | Same fix — inline IIFE bypasses RequireJS entirely |
| `app.createSessionObject is not a function` | Wrong API — `qlik.currApp()` wrapper, not enigma handle | `app.model.enigmaModel.createSessionObject()` |
| `The calculation page is too large` | `qInitialDataFetch: [{ qWidth:50, qHeight:500 }]` pre-fetching on object load | Set to `[]` — no pre-fetch |
| `Result too large` | Qlik Cloud WebSocket message size limit per response | Reduced to 100 rows/page; later moved to session objects |
| `Too many properties to enumerate` | SheetJS `aoa_to_sheet` creates 21M JS object properties | Raw XML builder in v1.2 |
| `Invalid string length` | `Array.join('')` on 280MB of XML fragments | `Uint8Array` chunked encoding in v1.2 |

---

## Architecture Overview

```
Button click
    │
    ├─ STEP 1: Validate SheetJS / fflate loaded
    ├─ STEP 2: Read column definitions from layout (no data fetch)
    ├─ STEP 3: Build session hypercube definition (qInitialDataFetch:[])
    ├─ STEP 4: Build display headers array
    ├─ STEP 5: createSessionObject → getLayout (row count only)
    ├─ STEP 6: Parallel paginated fetch
    │           ├─ Dispatch N concurrent getHyperCubeData calls
    │           ├─ Store results in resultSlots[pageIndex]
    │           └─ Flatten in order after all pages complete
    ├─ STEP 7: Data ready
    │
    ├─ [CSV path]
    │   └─ Chunked Blob builder → triggerDownload(.csv)
    │
    └─ [XLSX path]
        ├─ STEP 8a: buildSST — scan all cells, collect unique strings
        ├─ STEP 8b: buildSheetBytes — row-by-row Uint8Array (no string join)
        ├─ STEP 8c: buildSstBytes — sharedStrings.xml as Uint8Array
        ├─ STEP 8d: buildXlsxZip — fflate.zipSync all XLSX parts
        └─ triggerDownload(.xlsx)
```

---

## Why not use Qlik's native export API?

Qlik's internal `ExportData` QIX method produces server-side binary XLSX (80MB for this dataset). However, **`ExportData` is explicitly not supported in Qlik Cloud SaaS editions** for extensions. The WebSocket/enigma API (JSON over WebSocket) is the only data path available to extensions, which carries ~4× overhead vs server binary. FlexExport's v1.2 compressed XLSX output (40–80MB) bridges most of this gap.

---

## File Structure

```
flex-export/
├── flex-export.js      Main extension — SheetJS + fflate bundled inline
├── flex-export.css     Button and progress bar styles  
├── flex-export.qext    Extension metadata (Qlik registration)
└── README.md           Installation and usage guide
```

---

## Dependencies (bundled inline)

| Library | Version | Purpose |
|---|---|---|
| SheetJS (xlsx) | 0.18.5 | CSV writer + XLSX utilities |
| fflate | 0.8.2 | Deflate compression + ZIP assembly |

Both libraries are bundled directly inside `flex-export.js` as self-executing IIFEs to avoid Qlik Cloud's RequireJS path resolution issues with external files.

---

## Patch — Responsive Button/Progress Swap
**Commit:** UX improvement for small/single-line containers

### Problem
When the FlexExport control was placed in a small or single-line box on the Qlik sheet, the progress bar and status text appeared below the button — often clipped or pushed outside the container, making it invisible to the user.

### Fix
- Wrapped buttons in `.ceb-btn-area` div
- On export start: button area hides, progress bar expands to fill the exact same space
- On export complete/error: progress hides, button reappears
- Progress label uses `text-overflow: ellipsis` — truncates cleanly in narrow containers
- Works correctly in any container size including single-line boxes

### Behaviour
```
Before click : [ Export Data ▼ ]
During export: [████████░░░░░░░░░░░░]
               Fetching 423,000 / 1,04...
After done   : [ Export Data ▼ ]
```
