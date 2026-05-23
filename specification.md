# ACR Template Specification v1.0

## Overview

ACR (Across Report) defines a printer-independent report template format based on JSON.

ACR separates layout definition, rendering, and output generation into distinct layers.  
The same template produces identical output on any platform and any output target.

> *Render once. Output everywhere.*

### Supported Output Targets

| Target | Description |
|--------|-------------|
| PDF | High-fidelity document output |
| PNG | Pixel-accurate raster image |
| SVG | Scalable vector output |
| ESC/POS | Thermal receipt printers |
| StarPRNT | Star Micronics printers |
| SATO | SATO label printers |
| TEC | Toshiba TEC printers |

---

## Architecture

```
Template (JSON)
    ↓
Layout Engine        — resolves sections, binds data, calculates positions
    ↓
Drawing Model (JSON) — intermediate representation, inspectable and cacheable
    ↓
Drawing Engine       — renders via Google Skia (1-dot precision)
    ↓
Output               — PDF / PNG / ESC/POS / StarPRNT / SATO / TEC
```

### Design Principles

**Printer independence**  
ACR does not rely on printer drivers or OS print subsystems.  
Layout is calculated in device-independent units and rendered at the target resolution.

**WYSIWYG guarantee**  
The on-screen preview is pixel-identical to the final printed output.  
Not a single dot of deviation is permitted.

**Hardware-free preview**  
A complete, pixel-accurate preview is available without physical printers or drivers.

**JSON as the intermediate model**  
Both the template and the drawing model are JSON.  
They can be inspected, cached, versioned, and transmitted independently of rendering.

**Skia-based rendering**  
The drawing engine uses Google Skia — the same graphics library behind Chrome and Android.  
This guarantees consistent, high-fidelity output across all platforms.

**ActiveReports-compatible section model**  
ACR adopts a section-based layout structure compatible with ActiveReports conventions.

---

## Coordinate System

- **Unit:** dot
- **Definition:** 1 dot = 1/DPI inch  
  Example: at 203 DPI, 1 inch = 203 dots
- **Origin:** top-left corner of the page
- **X-axis:** increases to the right
- **Y-axis:** increases downward
- **Positioning:** all coordinates use absolute positioning

---

## Template Structure

A template is a single JSON file with the following root structure.

```json
{
  "version": "1.0",
  "page": { ... },
  "datasource": { ... },
  "sections": [ ... ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | string | ✓ | Specification version. Currently `"1.0"` |
| `page` | object | ✓ | Page size and margin definitions |
| `datasource` | object | | Data binding configuration |
| `sections` | array | ✓ | Ordered list of report sections |

---

## Page Object

Defines the logical page size and margins.

```json
{
  "width": 2480,
  "height": 3508,
  "unit": "dot",
  "dpi": 300,
  "margin": {
    "top": 118,
    "bottom": 118,
    "left": 118,
    "right": 118
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `width` | number | Page width in dots |
| `height` | number | Page height in dots |
| `unit` | string | Always `"dot"` |
| `dpi` | number | Target resolution (e.g. 300, 203, 96) |
| `margin` | object | Page margins in dots (top / bottom / left / right) |

**Common page sizes at 300 DPI:**

| Paper | Width (dot) | Height (dot) |
|-------|-------------|--------------|
| A4 | 2480 | 3508 |
| Letter | 2550 | 3300 |
| Receipt 80mm | 945 | dynamic |

---

## Section Model

ACR uses a section-based layout model compatible with ActiveReports.  
Sections are processed in order and rendered to the page sequentially.

### Section Types

| Section | Rendered | Description |
|---------|----------|-------------|
| `ReportHeader` | Once at the start of the report | Title, logo, report metadata |
| `PageHeader` | Top of every page | Column headings, page title |
| `GroupHeader` | Start of each data group (nestable) | Group label, subtotal header |
| `Detail` | Once per data record | Main content rows |
| `GroupFooter` | End of each data group (nestable) | Group subtotals |
| `PageFooter` | Bottom of every page | Page numbers, date |
| `ReportFooter` | Once at the end of the report | Grand totals, signatures |

### Section Definition

```json
{
  "type": "PageHeader",
  "height": 120,
  "canGrow": false,
  "elements": [ ... ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Section type (see table above) |
| `height` | number | Section height in dots |
| `canGrow` | boolean | Whether the section expands to fit content |
| `groupKey` | string | Data field used for grouping (GroupHeader / GroupFooter only) |
| `elements` | array | List of controls within the section |

### Group Nesting

GroupHeader and GroupFooter sections can be nested to represent multi-level grouping.

```json
[
  { "type": "GroupHeader", "groupKey": "department", "elements": [...] },
  { "type": "GroupHeader", "groupKey": "category",   "elements": [...] },
  { "type": "Detail",                                 "elements": [...] },
  { "type": "GroupFooter", "groupKey": "category",   "elements": [...] },
  { "type": "GroupFooter", "groupKey": "department",  "elements": [...] }
]
```

---

## Controls

Controls are the drawable elements within a section.

### Common Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | ✓ | Control type |
| `x` | number | ✓ | X position in dots (from section left edge) |
| `y` | number | ✓ | Y position in dots (from section top edge) |
| `width` | number | ✓ | Width in dots |
| `height` | number | ✓ | Height in dots |
| `visible` | boolean | | Default: `true` |

---

### TextBox

Renders text with font and alignment control.

```json
{
  "type": "TextBox",
  "x": 0,
  "y": 0,
  "width": 1200,
  "height": 80,
  "text": "{{ invoiceTitle }}",
  "font": {
    "family": "IPAexMincho",
    "size": 24,
    "bold": true,
    "italic": false
  },
  "alignment": "center",
  "verticalAlignment": "middle",
  "color": "#000000",
  "canGrow": true
}
```

| Field | Type | Description |
|-------|------|-------------|
| `text` | string | Text content. Supports `{{ field }}` data binding |
| `font.family` | string | Font family name |
| `font.size` | number | Font size in points |
| `font.bold` | boolean | Bold |
| `font.italic` | boolean | Italic |
| `alignment` | string | `left` / `center` / `right` |
| `verticalAlignment` | string | `top` / `middle` / `bottom` |
| `color` | string | Text color in hex |
| `canGrow` | boolean | Expand height to fit content |

---

### Line

Draws a straight line.

```json
{
  "type": "Line",
  "x": 0,
  "y": 118,
  "x2": 2244,
  "y2": 118,
  "lineWidth": 2,
  "color": "#000000"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `x2` | number | End X position in dots |
| `y2` | number | End Y position in dots |
| `lineWidth` | number | Line thickness in dots |
| `color` | string | Line color in hex |

---

### Rectangle

Draws a filled or outlined rectangle.

```json
{
  "type": "Rectangle",
  "x": 0,
  "y": 0,
  "width": 2244,
  "height": 120,
  "lineWidth": 1,
  "borderColor": "#000000",
  "fillColor": "#F0F0F0"
}
```

---

### Image

Renders an image asset.

```json
{
  "type": "Image",
  "x": 0,
  "y": 0,
  "width": 300,
  "height": 300,
  "src": "images/logo.png",
  "sizing": "fit"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `src` | string | Path within the ZIP container |
| `sizing` | string | `fit` / `fill` / `clip` |

---

### Barcode

Renders a barcode or QR code.

```json
{
  "type": "Barcode",
  "x": 100,
  "y": 300,
  "width": 600,
  "height": 120,
  "data": "{{ orderCode }}",
  "symbology": "CODE128",
  "showText": true
}
```

| Field | Type | Description |
|-------|------|-------------|
| `data` | string | Barcode data. Supports data binding |
| `symbology` | string | `CODE128` / `CODE39` / `EAN13` / `EAN8` / `QR` |
| `showText` | boolean | Display human-readable text below barcode |

---

## Data Binding

Field values are bound using `{{ fieldName }}` syntax inside `text` and `data` properties.

```json
{ "text": "Invoice No: {{ invoiceNumber }}" }
{ "text": "Total: {{ formatCurrency(totalAmount) }}" }
{ "text": "Page {{ pageNumber }} of {{ pageCount }}" }
```

### Built-in Variables

| Variable | Description |
|----------|-------------|
| `pageNumber` | Current page number |
| `pageCount` | Total page count |
| `reportDate` | Report generation date |

---

## ZIP Container Format

ACR templates may be packaged as a ZIP archive for distribution.

```
template.acr  (ZIP)
├── template.json   ← Main template definition
├── meta.json       ← Template metadata
├── fonts/          ← Embedded font files
└── images/         ← Embedded image assets
```

---

## Rendering Model

```
1. Load template.json
2. Bind datasource to sections
3. Evaluate group keys → determine section repetition
4. Calculate element positions (Layout Engine)
5. Produce Drawing Model (JSON)
6. Render via Google Skia (Drawing Engine)
7. Output to target format
```

No printer driver is required at any stage.

---

## Implementation Languages

ACR can be implemented in any language with Skia bindings or a compatible 2D graphics library.

| Language | Status |
|----------|--------|
| Rust | Reference implementation ([acr-engine](https://github.com/acrossreport/acr-engine)) |
| C++ | Planned |
| C# | Planned |
| WebAssembly | Planned |

---

## Compatibility

| Feature | ActiveReports | ACR |
|---------|---------------|-----|
| Section model | ✓ | ✓ (compatible) |
| JSON template | — | ✓ |
| Printer-independent | — | ✓ |
| WYSIWYG guarantee | partial | ✓ |
| Hardware-free preview | — | ✓ |

---

## Changelog

| Version | Date | Description |
|---------|------|-------------|
| 1.0 | 2026-02-26 | Initial release |

---

*ACR Specification — acrossreport/acr-spec*
