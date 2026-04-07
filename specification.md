# ACR Template Specification v2.0

**ACR (Across Report)** — Printer-Independent Section-Based Report Template Specification

> **Status:** Beta
> This specification may change until the first stable release.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Scope](#2-scope)
3. [Core Design Principles](#3-core-design-principles)
4. [Terminology](#4-terminology)
5. [Coordinate System and Units](#5-coordinate-system-and-units)
6. [Page Model](#6-page-model)
7. [Section Model](#7-section-model)
8. [Section Rendering Order](#8-section-rendering-order)
9. [Pagination Rules](#9-pagination-rules)
10. [Group Processing Rules](#10-group-processing-rules)
11. [Detail Processing Rules](#11-detail-processing-rules)
12. [Controls](#12-controls)
13. [Control Common Properties](#13-control-common-properties)
14. [Label Control](#14-label-control)
15. [Field Control](#15-field-control)
16. [Line Control](#16-line-control)
17. [Rectangle Control](#17-rectangle-control)
18. [Planned Controls](#18-planned-controls)
19. [Data Model](#19-data-model)
20. [Field Resolution Rules](#20-field-resolution-rules)
21. [Formatting Rules](#21-formatting-rules)
22. [Template Root Structure](#22-template-root-structure)
23. [Normative Rendering Constraints](#23-normative-rendering-constraints)
24. [Output Model](#24-output-model)
25. [Minimal Example](#25-minimal-example)
26. [Extended Example](#26-extended-example)
27. [Compatibility Policy](#27-compatibility-policy)
28. [Non-Goals](#28-non-goals)
29. [Versioning Policy](#29-versioning-policy)
30. [Changelog](#30-changelog)

---

## 1. Purpose

ACR is a printer-independent report template specification for defining fixed-layout reports in JSON.

Its purpose is to separate:

* template structure,
* data binding,
* layout processing, and
* output rendering.

The same template and dataset should be renderable to multiple output formats without changing the report definition itself.

---

## 2. Scope

This specification defines:

* the page model,
* the section model,
* the control model,
* data binding behavior,
* pagination behavior,
* group break behavior,
* rendering order, and
* output expectations.

This specification does **not** define:

* visual designer implementation,
* editor UI behavior,
* printer driver behavior,
* operating system specific font substitution rules,
* sub-report execution.

---

## 3. Core Design Principles

### 3.1 Printer Independence

ACR templates must not depend on any specific printer driver.

### 3.2 Fixed Layout

ACR is a fixed-layout model.
All controls use absolute coordinates.
There is no HTML-like flow layout and no automatic reflow.

### 3.3 Section-Based Processing

ACR uses a banded section structure compatible with traditional report engines.
The report is built by processing sections in a deterministic vertical sequence.

### 3.4 Data and Layout Separation

Template JSON defines layout.
Data JSON defines row values.
The renderer combines both at runtime.

### 3.5 Deterministic Rendering

Given the same template, data, and rendering environment, the engine should produce the same layout result.

---

## 4. Terminology

| Term           | Meaning                                               |
| -------------- | ----------------------------------------------------- |
| Template       | JSON document describing page, sections, and controls |
| Data           | JSON document providing row data                      |
| Page           | One logical output page                               |
| Section        | One band in the report layout                         |
| Control        | One drawable item inside a section                    |
| Detail         | Repeating section rendered for each data row          |
| Group          | Logical row grouping based on a field value           |
| Printable Area | Page area remaining after margins are applied         |
| Current Row    | Data row currently being rendered                     |
| Next Row       | Next data row used for break detection                |

---

## 5. Coordinate System and Units

### 5.1 Unit

ACR uses **twips** as the canonical coordinate unit.

* 1 inch = 1440 twips
* 1 point = 20 twips
* 1 mm ≈ 56.69 twips

All page sizes, margins, section heights, and control bounds are expressed in twips.

### 5.2 Origin

The coordinate origin is the top-left corner of the containing area.

* X increases left to right
* Y increases top to bottom

### 5.3 Coordinate Space

* Page-level values are relative to the logical page
* Section control coordinates are relative to the top-left of the section

### 5.4 Numeric Values

Implementations may accept integer or floating-point numeric values.
Internally, engines may round as needed for rendering, but layout interpretation must remain logically stable.

---

## 6. Page Model

The page object defines logical page size and margins.

```json
{
  "page": {
    "width": 11906,
    "height": 16838,
    "margin_top": 1440,
    "margin_bottom": 1440,
    "margin_left": 1440,
    "margin_right": 1440
  }
}
```

### 6.1 Page Fields

| Field           | Type   | Required | Description            |
| --------------- | ------ | -------- | ---------------------- |
| `width`         | number | Yes      | Page width in twips    |
| `height`        | number | Yes      | Page height in twips   |
| `margin_top`    | number | No       | Top margin in twips    |
| `margin_bottom` | number | No       | Bottom margin in twips |
| `margin_left`   | number | No       | Left margin in twips   |
| `margin_right`  | number | No       | Right margin in twips  |

### 6.2 Printable Area

Printable area is calculated as:

* printable width = page width - left margin - right margin
* printable height = page height - top margin - bottom margin

All section placement must occur within the printable area.

---

## 7. Section Model

ACR uses a section-based structure.
A section is a vertical band with a fixed height and a list of controls.

### 7.1 Supported Section Types

* `report_header`
* `page_header`
* `group_header`
* `detail`
* `group_footer`
* `page_footer`
* `report_footer`

### 7.2 Section Basics

Each section has:

* a type,
* a fixed height,
* zero or more controls.

Example:

```json
{
  "height": 600,
  "controls": []
}
```

### 7.3 Section Constraints

* Sections are processed vertically, not layered as free-floating bands
* Section height is fixed during layout
* Controls belong to exactly one section
* Section structure must not be rewritten during rendering

---

## 8. Section Rendering Order

The logical rendering order of a report is:

1. `report_header` once at report start
2. `page_header` at the top of each page
3. zero or more `group_header` sections when a group opens
4. `detail` for each data row
5. zero or more `group_footer` sections when a group closes
6. `page_footer` at the bottom of each page
7. `report_footer` once at report end

### 8.1 First Page

If defined, `report_header` is processed before the first detail row.
`page_header` is still processed for the first page.

### 8.2 Last Page

If defined, `report_footer` is processed after the last data-dependent section.

---

## 9. Pagination Rules

Pagination is section-based.
A section is placed only if enough remaining vertical space exists in the current page.

### 9.1 Remaining Height

Remaining height is calculated inside the printable area after accounting for already placed sections and reserved footer space if applicable.

### 9.2 Page Break Condition

A new page must be started when the next required section cannot fit in the available remaining height.

### 9.3 Page Header and Footer Behavior

* `page_header` is rendered on every page if defined
* `page_footer` is rendered on every page if defined

### 9.4 Explicit New Page

If a section has a `new_page` behavior flag in an implementation, the engine must start a new page before rendering that section.

### 9.5 No Section Splitting

A section is treated as a single layout unit.
It must not be split across multiple pages unless the implementation explicitly defines a special future behavior.

---

## 10. Group Processing Rules

Groups are based on field value transitions.

### 10.1 Group Definition

A group is defined by a field name.

Example:

```json
{
  "group_data_field": "customer_code"
}
```

### 10.2 Group Open

A group opens when:

* processing begins for the first row, or
* the current row group value differs from the previous row group value.

### 10.3 Group Break

A group break occurs when:

* the current row value differs from the next row value, or
* there is no next row.

### 10.4 Group Header

When a group opens, the corresponding `group_header` must be rendered before the row detail associated with that group.

### 10.5 Group Footer

When a group closes, the corresponding `group_footer` must be rendered after the final detail row belonging to that group.

### 10.6 Nested Groups

If multiple groups are supported, they must be processed from outermost to innermost on open, and from innermost to outermost on close.

---

## 11. Detail Processing Rules

### 11.1 Detail Iteration

The `detail` section is rendered once for each row in `rows`.

### 11.2 Row Order

Rows are processed in source order.
ACR does not define sorting behavior in the specification itself.
If sorting is needed, it must be applied before rendering or by an engine-specific extension.

### 11.3 Empty Dataset

If `rows` is empty:

* `detail` is not rendered,
* group sections are not rendered,
* report-level and page-level sections may still render depending on engine policy.

For stable behavior, implementations should document how empty reports are handled.

---

## 12. Controls

A control is a drawable object placed inside a section.

Supported current controls:

* `label`
* `field`
* `line`
* `rectangle`

Planned controls:

* `image`
* `barcode`

---

## 13. Control Common Properties

All controls share the following base properties.

| Field    | Type   | Required | Description                  |
| -------- | ------ | -------- | ---------------------------- |
| `type`   | string | Yes      | Control type                 |
| `x`      | number | Yes      | Left position within section |
| `y`      | number | Yes      | Top position within section  |
| `width`  | number | Yes      | Control width                |
| `height` | number | Yes      | Control height               |

### 13.1 Bounds

Control coordinates are relative to the containing section.

### 13.2 Overflow

This specification does not define auto-growth or auto-shrink behavior.
If text exceeds available bounds, behavior is implementation-defined unless extended later.

---

## 14. Label Control

A label renders static text defined directly in the template.

```json
{
  "type": "label",
  "x": 0,
  "y": 0,
  "width": 3000,
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

| Field       | Type    | Required | Description               |
| ----------- | ------- | -------- | ------------------------- |
| `text`      | string  | Yes      | Static text               |
| `font_name` | string  | No       | Font family               |
| `font_size` | number  | No       | Font size in points       |
| `bold`      | boolean | No       | Bold style                |
| `italic`    | boolean | No       | Italic style              |
| `color`     | string  | No       | Text color in `#RRGGBB`   |
| `align`     | string  | No       | `left`, `center`, `right` |
| `valign`    | string  | No       | `top`, `middle`, `bottom` |

---

## 15. Field Control

A field renders data-bound text resolved from the current row.

```json
{
  "type": "field",
  "x": 0,
  "y": 0,
  "width": 3000,
  "height": 400,
  "field": "customer_name",
  "font_size": 12,
  "align": "left",
  "format": ""
}
```

| Field       | Type    | Required | Description               |
| ----------- | ------- | -------- | ------------------------- |
| `field`     | string  | Yes      | Data key name             |
| `font_name` | string  | No       | Font family               |
| `font_size` | number  | No       | Font size in points       |
| `bold`      | boolean | No       | Bold style                |
| `italic`    | boolean | No       | Italic style              |
| `color`     | string  | No       | Text color                |
| `align`     | string  | No       | `left`, `center`, `right` |
| `valign`    | string  | No       | `top`, `middle`, `bottom` |
| `format`    | string  | No       | Formatting instruction    |

---

## 16. Line Control

A line renders a straight line using the control bounds.

```json
{
  "type": "line",
  "x": 0,
  "y": 0,
  "width": 5000,
  "height": 0,
  "color": "#000000",
  "thickness": 20
}
```

| Field       | Type   | Required | Description    |
| ----------- | ------ | -------- | -------------- |
| `color`     | string | No       | Line color     |
| `thickness` | number | No       | Line thickness |

Horizontal line: use `height = 0`
Vertical line: use `width = 0`

---

## 17. Rectangle Control

A rectangle renders a bordered and/or filled rectangle.

```json
{
  "type": "rectangle",
  "x": 0,
  "y": 0,
  "width": 5000,
  "height": 800,
  "fill_color": "#F5F5F5",
  "border_color": "#CCCCCC",
  "border_thickness": 15,
  "radius": 0
}
```

| Field              | Type   | Required | Description            |
| ------------------ | ------ | -------- | ---------------------- |
| `fill_color`       | string | No       | Fill color or `none`   |
| `border_color`     | string | No       | Border color or `none` |
| `border_thickness` | number | No       | Border thickness       |
| `radius`           | number | No       | Corner radius          |

---

## 18. Planned Controls

### 18.1 Image

Image is planned but not normative in the current stable control set.

### 18.2 Barcode

Barcode is planned but not normative in the current stable control set.

Planned controls must not be treated as required behavior by current engines unless explicitly implemented.

---

## 19. Data Model

The data document is row-based.

```json
{
  "rows": [
    {
      "customer_name": "Acme Corp",
      "invoice_no": "INV-001",
      "amount": 150000
    }
  ]
}
```

| Field  | Type  | Required | Description          |
| ------ | ----- | -------- | -------------------- |
| `rows` | array | Yes      | Array of row objects |

Each row is a key-value map.

---

## 20. Field Resolution Rules

### 20.1 Detail Section Resolution

In `detail`, a `field` control resolves against the current row.

### 20.2 Non-Detail Resolution

In report-level and page-level sections, if field-based controls are allowed by an implementation, they resolve using the engine's current row context.
A common policy is to use the first row when available.
That policy must be documented by the engine.

### 20.3 Missing Field

If the specified field key does not exist, the rendered value must be treated as an empty string.

### 20.4 Null Value

If a field value is null, the engine should render it as an empty string unless engine-specific formatting rules specify otherwise.

---

## 21. Formatting Rules

Field controls may support formatting.

Examples:

* `#,##0`
* `#,##0.00`
* `YYYY-MM-DD`

This specification defines the existence of a `format` property, but exact format token behavior may remain engine-specific until the formatting grammar is fully standardized.

---

## 22. Template Root Structure

A template root contains page definition and section definitions.

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
  "report_header": { ... },
  "page_header": { ... },
  "detail": { ... },
  "page_footer": { ... },
  "report_footer": { ... }
}
```

### 22.1 Root Fields

| Field           | Type            | Required | Description             |
| --------------- | --------------- | -------- | ----------------------- |
| `version`       | string          | Yes      | Specification version   |
| `page`          | object          | Yes      | Page definition         |
| `report_header` | object          | No       | Report start section    |
| `page_header`   | object          | No       | Per-page header         |
| `group_header`  | object or array | No       | Group header definition |
| `detail`        | object          | Yes      | Detail section          |
| `group_footer`  | object or array | No       | Group footer definition |
| `page_footer`   | object          | No       | Per-page footer         |
| `report_footer` | object          | No       | Report end section      |

Implementations may support either a single group band or multiple grouped bands.
The actual accepted structure must be reflected in the schema used by that engine version.

---

## 23. Normative Rendering Constraints

The following constraints are normative:

1. Section order must remain stable.
2. Existing layout logic must not be silently changed by the template processor.
3. The section hierarchy is part of report integrity and must be preserved.
4. Pagination must be based on available remaining height.
5. Detail rows must be processed sequentially.
6. Group break judgment must be based on value transition logic.
7. A control must not render outside its section coordinate system conceptually, even if a renderer clips or rounds visually.

---

## 24. Output Model

ACR is intended to support multiple output targets.
Current and planned examples include:

| Format      | Description                          |
| ----------- | ------------------------------------ |
| PDF         | Multi-page document output           |
| PNG package | Per-page image output                |
| ZIP package | Bundle output such as PDF and images |
| ESC/POS     | Planned thermal printer output       |
| ZPL         | Planned label printer output         |

The specification defines layout behavior independently of output format.

---

## 25. Minimal Example

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

---

## 26. Extended Example

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
    "height": 1200,
    "controls": [
      {
        "type": "label",
        "x": 0,
        "y": 200,
        "width": 9000,
        "height": 600,
        "text": "INVOICE",
        "font_size": 28,
        "bold": true,
        "align": "center"
      }
    ]
  },
  "page_header": {
    "height": 700,
    "controls": [
      {
        "type": "label",
        "x": 0,
        "y": 100,
        "width": 3000,
        "height": 400,
        "text": "Customer",
        "bold": true
      },
      {
        "type": "label",
        "x": 3000,
        "y": 100,
        "width": 3000,
        "height": 400,
        "text": "Invoice No.",
        "bold": true
      },
      {
        "type": "label",
        "x": 6000,
        "y": 100,
        "width": 3000,
        "height": 400,
        "text": "Amount",
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
        "x": 0,
        "y": 100,
        "width": 3000,
        "height": 400,
        "field": "customer_name"
      },
      {
        "type": "field",
        "x": 3000,
        "y": 100,
        "width": 3000,
        "height": 400,
        "field": "invoice_no"
      },
      {
        "type": "field",
        "x": 6000,
        "y": 100,
        "width": 3000,
        "height": 400,
        "field": "amount",
        "align": "right",
        "format": "#,##0"
      },
      {
        "type": "line",
        "x": 0,
        "y": 580,
        "width": 9000,
        "height": 0,
        "thickness": 15
      }
    ]
  },
  "page_footer": {
    "height": 500,
    "controls": [
      {
        "type": "label",
        "x": 0,
        "y": 50,
        "width": 9000,
        "height": 300,
        "text": "ACR Engine",
        "align": "center",
        "font_size": 9
      }
    ]
  }
}
```

---

## 27. Compatibility Policy

ACR is conceptually compatible with section-based report systems such as ActiveReports in the following sense:

* band-oriented layout,
* repeated detail rows,
* page header/footer model,
* group header/footer model.

However, ACR is defined as a JSON-based specification and is not intended to duplicate every legacy feature.

---

## 28. Non-Goals

The following are explicitly outside the current scope:

* free-flow text layout,
* browser DOM rendering rules,
* nested sub-report execution,
* runtime section restructuring,
* automatic designer-specific behaviors not represented in JSON.

---

## 29. Versioning Policy

* Major version changes may introduce structural changes.
* Minor version changes may add optional fields or clarify rules.
* Engines should validate template versions before rendering.

---

## 30. Changelog

| Version | Summary                                                                                                                          |
| ------- | -------------------------------------------------------------------------------------------------------------------------------- |
| 2.0     | Reorganized full specification with section model, pagination rules, group rules, control definitions, and normative constraints |
| 1.x     | Early draft-level overview and structure definitions                                                                             |
