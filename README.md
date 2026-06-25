# FlexExport v1.2.0
**Configurable Export Button for Qlik Sense**

**Author:** Sheikh Asadullah Mahrab  
**Email:** sheikhasadullahmahrab@gmail.com  
**License:** MIT  

---

## What it does

FlexExport places a fully configurable export button on your Qlik sheet. No visible table — just a button. Users click it to download data as Excel (.xlsx) or CSV.

- Add any number of dimensions and measures (up to 50 each)
- Rename column headers using the standard Label field
- Exports 1M+ row datasets reliably
- Compressed XLSX output (~4× smaller than uncompressed)
- Date-time stamped filenames automatically
- Respects Qlik selections or exports all data
- Edit-mode click guard — no accidental exports while configuring

---

## Installation

**Qlik Cloud**
1. Management Console → Extensions → Add → upload `flex-export.zip`

**Qlik Sense Enterprise (QSEoW)**
1. QMC → Extensions → Import → upload `flex-export.zip`

**Qlik Sense Desktop**
1. Unzip into `C:\Users\%USERNAME%\Documents\Qlik\Sense\Extensions\flex-export\`

---

## Usage

1. In Edit mode, drag **FlexExport** from Custom Objects onto the sheet
2. Add dimensions/measures in the properties panel
3. Set the **Label** field on each dimension/measure to rename it in the export
4. Configure under **Export Settings** and **Column Options**
5. Switch to Analysis mode and click the button

---

## Properties

### Export Settings
| Property | Description |
|---|---|
| Button label | Text on the button |
| Export format | Excel, CSV, or Both |
| File name prefix | Prepended to the date-time stamp (blank = date-time only) |
| Data scope | Respect selections (Possible) or export all data |
| Excel sheet name | Tab name inside the .xlsx file |

### Column Options
| Property | Description |
|---|---|
| Max rows | 0 = all rows |
| Include row numbers | Prepend a # column |
| Page size | Rows per WebSocket fetch (default 500) |
| Parallel fetches | Concurrent page requests (default 6, max 12) |

---

## Technical notes

- **XLSX writer:** Raw XML construction with Shared String Table (SST) + fflate deflate compression. Handles 1M+ rows without V8 object property or string length limits.
- **Fetch strategy:** Session object (no pre-fetch), parallel pages, ordered slot assembly.
- **File naming:** `prefix_YYYYMMDD_HHmmss.xlsx` or just `YYYYMMDD_HHmmss.xlsx`

---

## Version history

| Version | Notes |
|---|---|
| 1.0.0 | Sequential fetch, SheetJS CSV/XLSX writer |
| 1.1.0 | Parallel fetch, adaptive page size |
| 1.2.0 | Raw XML XLSX builder, SST, fflate ZIP — no row limits, compressed output |
