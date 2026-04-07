# ACR Template Specification v2.0

**ACR (Across Report Renderer)** — Printer-Independent Report Template Specification

> **Status:** Beta — Specification subject to change until v1.0 stable release (planned July 2026).

---

## Table of Contents

1. [Overview](#1-overview)
2. [Design Philosophy](#2-design-philosophy)
3. [Coordinate System](#3-coordinate-system)
4. [Template Root Structure](#4-template-root-structure)
5. [Page Object](#5-page-object)
6. [Section Types](#6-section-types)
7. [Control Types](#7-control-types)
   - [Label](#71-label)
   - [Field](#72-field)
   - [Line](#73-line)
   - [Rectangle](#74-rectangle)
   - [Image](#75-image-planned)
   - [Barcode](#76-barcode-planned)
8. [Data Binding](#8-data-binding)
9. [Output Formats](#9-output-formats)
10. [Minimal Example](#10-minimal-example)
11. [Full Example](#11-full-example)
12. [Changelog](#12-changelog)

---

## 1. Overview

ACR defines a **printer-independent report template format** using JSON.

The rendering pipeline is:

```
template.json + data.json
        │
        ▼
   ACR Engine (Rust / Skia)
        │
        ├──▶ PDF
        ├──▶ PNG (per page)
        ├──▶ ZIP (PDF + PNG package)
        ├──▶ ESC/POS  (planned)
        └──▶ ZPL      (planned)
```

ACR separates **layout definition**, **data binding**, and **rendering**, enabling the same template to produce output across multiple platforms and formats without any printer driver.

---

## 2. Design Philosophy

### JSON as Intermediate Drawing Model

ACR treats the JSON template as an **intermediate drawing model** — not a markup language, not a DSL, but a structured description of what to draw and where.

This means:

- Every control has an **absolute position** — no flow layout, no reflow
- Coordinates are resolved at template design time, not at render time
- The engine performs **pixel-perfect rendering** (WYSIWYG) regardless of host OS

### Section-Based Layout (ActiveReports-Compatible)

ACR adopts a **band-based section model** compatible with ActiveReports:

```
┌─────────────────────┐
│     ReportHeader    │  printed once at start of report
├─────────────────────┤
│     PageHeader      │  printed at top of every page
├─────────────────────┤
│       Detail        │  repeated for each data row
├─────────────────────┤
│     PageFooter      │  printed at bottom of every page
├─────────────────────┤
│     ReportFooter    │  printed once at end of report
└─────────────────────┘
```

### No Sub-Reports

Because ACR uses JSON as its intermediate model, sub-reports are intentionally **not supported**. The Detail section's data repetition model covers the equivalent use cases.

---

## 3. Coordinate System

### Unit: Twips

ACR uses **twips** as its coordinate unit throughout.

```
1 inch  = 1440 twips
1 mm    = 56.69 twips  (approx)
1 point = 20 twips
```

| Paper Size | Width (twips) | Height (twips) |
|---|---|---|
| A4 Portrait | 11906 | 16838 |
| A4 Landscape | 16838 | 11906 |
| Letter Portrait | 12240 | 15840 |
| Receipt 80mm | 4536 | (dynamic) |

### Origin

```
(0, 0) ── X ──▶
   │
   Y
   │
   ▼
```

- Origin is the **top-left corner** of the section
- X increases to the right
- Y increases downward
- All coordinates are **absolute** within the section

---

## 4. Template Root Structure

```json
{
  "version": "2.0",
  "page": { ... },
  "report_header": { ... },
  "page_header":   { ... },
  "detail":        { ... },
  "page_footer":   { ... },
  "report_footer": { ... }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `version` | string | ✅ | Specification version (`"2.0"`) |
| `page` | Page object | ✅ | Page size and margin definition |
| `report_header` | Section object | ☐ | Printed once at report start |
| `page_header` | Section object | ☐ | Printed at top of every page |
| `detail` | Section object | ✅ | Repeated for each data row |
| `page_footer` | Section object | ☐ | Printed at bottom of every page |
| `report_footer` | Section object | ☐ | Printed once at report end |

---

## 5. Page Object

Defines the logical page size and margins.

```json
{
  "page": {
    "width":  11906,
    "height": 16838,
    "margin_top":    1440,
    "margin_bottom": 1440,
    "margin_left":   1440,
    "margin_right":  1440
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `width` | number | ✅ | Page width in twips |
| `height` | number | ✅ | Page height in twips |
| `margin_top` | number | ☐ | Top margin in twips (default: 0) |
| `margin_bottom` | number | ☐ | Bottom margin in twips (default: 0) |
| `margin_left` | number | ☐ | Left margin in twips (default: 0) |
| `margin_right` | number | ☐ | Right margin in twips (default: 0) |

---

## 6. Section Types

Each section is an object with a `height` and a `controls` array.

```json
{
  "height": 2880,
  "controls": [
    { ... },
    { ... }
  ]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `height` | number | ✅ | Section height in twips |
| `controls` | array | ✅ | Array of control objects |

### Section Rendering Rules

| Section | Rendered |
|---|---|
| `report_header` | Once — first page, before `page_header` |
| `page_header` | Every page — top of printable area |
| `detail` | Once per data row — repeated until data exhausted |
| `page_footer` | Every page — bottom of printable area |
| `report_footer` | Once — last page, after last `detail` |

---

## 7. Control Types

All controls share these common fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | ✅ | Control type identifier |
| `x` | number | ✅ | Left position within section (twips) |
| `y` | number | ✅ | Top position within section (twips) |
| `width` | number | ✅ | Control width (twips) |
| `height` | number | ✅ | Control height (twips) |

---

### 7.1 Label

Renders **static text** — text fixed in the template, not bound to data.

```json
{
  "type": "label",
  "x": 720,
  "y": 360,
  "width": 5040,
  "height": 400,
  "text": "Invoice",
  "font_name": "Noto Sans JP",
  "font_size": 24,
  "bold": true,
  "italic": false,
  "color": "#000000",
  "align": "left",
  "valign": "top"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `text` | string | ✅ | Static text content |
| `font_name` | string | ☐ | Font family name |
| `font_size` | number | ☐ | Font size in points |
| `bold` | boolean | ☐ | Bold (default: false) |
| `italic` | boolean | ☐ | Italic (default: false) |
| `color` | string | ☐ | Text color in `#RRGGBB` (default: `"#000000"`) |
| `align` | string | ☐ | Horizontal alignment: `"left"` / `"center"` / `"right"` |
| `valign` | string | ☐ | Vertical alignment: `"top"` / `"middle"` / `"bottom"` |

---

### 7.2 Field

Renders **data-bound text** — value replaced at render time from the data JSON.

```json
{
  "type": "field",
  "x": 720,
  "y": 800,
  "width": 5040,
  "height": 400,
  "field": "customer_name",
  "font_name": "Noto Sans JP",
  "font_size": 14,
  "color": "#333333",
  "align": "left",
  "format": ""
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `field` | string | ✅ | Key name in the data JSON row |
| `font_name` | string | ☐ | Font family name |
| `font_size` | number | ☐ | Font size in points |
| `bold` | boolean | ☐ | Bold (default: false) |
| `italic` | boolean | ☐ | Italic (default: false) |
| `color` | string | ☐ | Text color in `#RRGGBB` |
| `align` | string | ☐ | Horizontal alignment: `"left"` / `"center"` / `"right"` |
| `format` | string | ☐ | Format string (e.g. `"#,##0"` for number, `"YYYY-MM-DD"` for date) |

---

### 7.3 Line

Renders a **straight line**.

```json
{
  "type": "line",
  "x": 720,
  "y": 1440,
  "width": 10440,
  "height": 0,
  "color": "#000000",
  "thickness": 20
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `color` | string | ☐ | Line color in `#RRGGBB` (default: `"#000000"`) |
| `thickness` | number | ☐ | Line thickness in twips (default: 20) |

> For a horizontal line, set `height` to `0` and use `width` for length.  
> For a vertical line, set `width` to `0` and use `height` for length.

---

### 7.4 Rectangle

Renders a **filled or outlined rectangle**.

```json
{
  "type": "rectangle",
  "x": 720,
  "y": 360,
  "width": 10440,
  "height": 800,
  "fill_color": "#F5F5F5",
  "border_color": "#CCCCCC",
  "border_thickness": 15,
  "radius": 0
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `fill_color` | string | ☐ | Fill color in `#RRGGBB` or `"none"` for transparent |
| `border_color` | string | ☐ | Border color in `#RRGGBB` or `"none"` for no border |
| `border_thickness` | number | ☐ | Border thickness in twips (default: 20) |
| `radius` | number | ☐ | Corner radius in twips (default: 0) |

---

### 7.5 Image *(Planned)*

Renders an **embedded or referenced image**.

> 🗓 Planned for a future release.

```json
{
  "type": "image",
  "x": 720,
  "y": 360,
  "width": 2880,
  "height": 2880,
  "src": "logo.png",
  "fit": "contain"
}
```

| Field | Type | Description |
|---|---|---|
| `src` | string | Image file path or base64 data URI |
| `fit` | string | Sizing mode: `"contain"` / `"cover"` / `"fill"` / `"none"` |

---

### 7.6 Barcode *(Planned)*

Renders a **barcode or QR code**.

> 🗓 Planned for a future release.

```json
{
  "type": "barcode",
  "x": 720,
  "y": 2880,
  "width": 5760,
  "height": 1440,
  "data": "1234567890",
  "symbology": "CODE128",
  "show_text": true
}
```

| Field | Type | Description |
|---|---|---|
| `data` | string | Barcode data string |
| `symbology` | string | Barcode type (see below) |
| `show_text` | boolean | Display human-readable text below barcode |

Planned symbologies:

| Symbology | Description |
|---|---|
| `CODE128` | Code 128 (alphanumeric) |
| `CODE39` | Code 39 |
| `EAN13` | EAN-13 |
| `EAN8` | EAN-8 |
| `QR` | QR Code |
| `JAN` | JAN (Japanese Article Number) |

---

## 8. Data Binding

### Data JSON Structure

The data JSON passed to the engine must follow this structure:

```json
{
  "rows": [
    {
      "customer_name": "Acme Corp",
      "invoice_no": "INV-2026-001",
      "amount": 150000
    },
    {
      "customer_name": "Globex Inc",
      "invoice_no": "INV-2026-002",
      "amount": 280000
    }
  ]
}
```

### Binding Rules

- The `detail` section is rendered **once per row** in the `rows` array.
- `report_header`, `page_header`, `page_footer`, `report_footer` are rendered with the **first row's values** when field references are used.
- If a `field` key does not exist in the data row, the field renders as **empty string**.

### Field Reference in Controls

In a `Field` control, set `"field": "key_name"` to bind to the corresponding key in each data row:

```json
{ "type": "field", "field": "customer_name", ... }
```

At render time, `customer_name` is replaced with the actual value from the data row.

---

## 9. Output Formats

| Format | Extension | Description |
|---|---|---|
| PDF | `.pdf` | Multi-page PDF — all pages in one file |
| PNG Package | `.zip` | ZIP containing `manifest.json` + per-page PNG images |
| ESC/POS | — | Thermal printer commands *(planned)* |
| ZPL | — | Zebra label printer commands *(planned)* |

### ZIP Package Structure

```
output.zip
├── manifest.json
└── pages/
    ├── 001.png
    ├── 002.png
    └── ...
```

`manifest.json` contains:

```json
{
  "format": "ACR-PNG-PACKAGE",
  "version": 1,
  "engine_version": "0.1.0",
  "page_count": 3,
  "dpi": 96,
  "width_twips": 11906,
  "height_twips": 16838,
  "created_at": "2026-04-08T09:00:00+09:00"
}
```

---

## 10. Minimal Example

A minimal template with a single detail section:

**template.json**

```json
{
  "version": "2.0",
  "page": {
    "width": 11906,
    "height": 16838,
    "margin_top": 1440,
    "margin_bottom": 1440,
    "margin_left": 1440,
    "margin_right": 1440
  },
  "detail": {
    "height": 600,
    "controls": [
      {
        "type": "field",
        "x": 0,
        "y": 100,
        "width": 9000,
        "height": 400,
        "field": "name",
        "font_size": 12
      }
    ]
  }
}
```

**data.json**

```json
{
  "rows": [
    { "name": "Alice" },
    { "name": "Bob" },
    { "name": "Charlie" }
  ]
}
```

**CLI usage**

```bash
acr_cli template.json data.json
```

---

## 11. Full Example

An invoice template using all currently available section types and controls:

**template.json**

```json
{
  "version": "2.0",
  "page": {
    "width": 11906,
    "height": 16838,
    "margin_top": 1440,
    "margin_bottom": 1440,
    "margin_left": 1440,
    "margin_right": 1440
  },
  "report_header": {
    "height": 2880,
    "controls": [
      {
        "type": "label",
        "x": 0, "y": 200,
        "width": 9026, "height": 800,
        "text": "INVOICE",
        "font_name": "Noto Sans JP",
        "font_size": 32,
        "bold": true,
        "align": "center"
      },
      {
        "type": "line",
        "x": 0, "y": 1200,
        "width": 9026, "height": 0,
        "thickness": 30
      }
    ]
  },
  "page_header": {
    "height": 800,
    "controls": [
      {
        "type": "rectangle",
        "x": 0, "y": 0,
        "width": 9026, "height": 700,
        "fill_color": "#F0F0F0",
        "border_color": "none"
      },
      {
        "type": "label",
        "x": 0, "y": 150,
        "width": 3000, "height": 400,
        "text": "Customer",
        "font_size": 10,
        "bold": true
      },
      {
        "type": "label",
        "x": 3000, "y": 150,
        "width": 3000, "height": 400,
        "text": "Invoice No.",
        "font_size": 10,
        "bold": true
      },
      {
        "type": "label",
        "x": 6000, "y": 150,
        "width": 3026, "height": 400,
        "text": "Amount",
        "font_size": 10,
        "bold": true,
        "align": "right"
      }
    ]
  },
  "detail": {
    "height": 600,
    "controls": [
      {
        "type": "field",
        "x": 0, "y": 100,
        "width": 3000, "height": 400,
        "field": "customer_name",
        "font_size": 11
      },
      {
        "type": "field",
        "x": 3000, "y": 100,
        "width": 3000, "height": 400,
        "field": "invoice_no",
        "font_size": 11
      },
      {
        "type": "field",
        "x": 6000, "y": 100,
        "width": 3026, "height": 400,
        "field": "amount",
        "font_size": 11,
        "align": "right",
        "format": "#,##0"
      },
      {
        "type": "line",
        "x": 0, "y": 580,
        "width": 9026, "height": 0,
        "color": "#DDDDDD",
        "thickness": 15
      }
    ]
  },
  "page_footer": {
    "height": 600,
    "controls": [
      {
        "type": "line",
        "x": 0, "y": 0,
        "width": 9026, "height": 0,
        "thickness": 20
      },
      {
        "type": "label",
        "x": 0, "y": 100,
        "width": 9026, "height": 400,
        "text": "ACR Engine — Across Report Renderer",
        "font_size": 9,
        "color": "#888888",
        "align": "center"
      }
    ]
  }
}
```

**data.json**

```json
{
  "rows": [
    { "customer_name": "Acme Corp",   "invoice_no": "INV-2026-001", "amount": 150000 },
    { "customer_name": "Globex Inc",  "invoice_no": "INV-2026-002", "amount": 280000 },
    { "customer_name": "Initech Ltd", "invoice_no": "INV-2026-003", "amount": 95000  }
  ]
}
```

---

## 12. Changelog

| Version | Date | Changes |
|---|---|---|
| 2.0 | 2026-04-08 | Section model (ReportHeader/PageHeader/Detail/PageFooter/ReportFooter), Twips coordinate system, Label/Field/Line/Rectangle controls, Data binding specification, ZIP output format |
| 1.0 | 2026-02-26 | Initial release — dot coordinate system, basic control types |

---

## License

This specification is published by the [Across Report](https://github.com/acrossreport) project.

For inquiries: [hisa.araki@gmail.com](mailto:hisa.araki@gmail.com)

---

*ACR Engine (Across Report Renderer) — Powered by Google Skia*
