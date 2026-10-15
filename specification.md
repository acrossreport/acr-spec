# ACR Template Specification

## Overview

ACR (Across Report) is a printer-independent report template format based on JSON.

ACR separates layout definition, rendering, and output generation into distinct layers.
The same template produces identical output on any platform and any output target.

> *Render once. Output everywhere.*

This document describes the template format as actually consumed by the ACR engine
(`acr_core`) and its clients (`acr-viewer-vscode`, `acr-viewer`, `acr-cli`). It supersedes
earlier drafts of this specification that used a different, unimplemented schema
(lowercase `x`/`y`/`width`/`height`, `TextBox` controls, `{{ field }}` mustache binding).
Templates produced by the `RPX2ACR` conversion tool, and all current sample templates,
follow the schema below.

### Confirmed Output Targets

| Target | Description |
|--------|-------------|
| PDF | Document output |
| PNG | Pixel-accurate raster image (one file per page, optionally packaged as a ZIP) |

Additional output targets (SVG, ESC/POS, StarPRNT, SATO, TEC) have been discussed as
future direction but are not confirmed as implemented in the current engine. Do not rely
on this document for their field-level format until they ship.

---

## Architecture

```
Template (JSON)
    ↓
Layout Engine        — resolves sections, binds data, calculates positions
    ↓
Drawing Engine        — renders via Google Skia (1-dot precision)
    ↓
Output                — PDF / PNG
```

### Design Principles

**Printer independence**
ACR does not rely on printer drivers or OS print subsystems.
Layout is calculated in device-independent units and rendered at the target resolution.

**WYSIWYG guarantee**
The on-screen preview is pixel-identical to the final printed output.

**Hardware-free preview**
A complete, pixel-accurate preview is available without physical printers or drivers.

**JSON as the template format**
Templates are plain JSON, inspectable and version-controllable independently of rendering.

**Skia-based rendering**
The drawing engine uses Google Skia — the same graphics library behind Chrome and Android.

**Section-based report model**
ACR adopts a section-based layout structure widely used in legacy business reporting
tools, to minimize the cost of migrating existing report definitions (see
[design-decisions.md](design-decisions.md), ADR-003).

---

## Coordinate System

- **Unit:** twip (1/1440 inch; 1 inch = 1440 twips)
- **Origin:** top-left corner of the page / section
- **X-axis:** increases to the right
- **Y-axis:** increases downward
- **Positioning:** control coordinates (`Left`/`Top`) are relative to the top-left corner
  of the section that contains them, not the page
- Line controls additionally use `X1`/`Y1`/`X2`/`Y2`, also in twips relative to the
  section

---

## Template Structure

A template is a single JSON file with the following root structure.

```json
{
  "ACR": {
    "Meta": { ... },
    "Report": {
      "PageSettings": { ... },
      "Sections": { "Section": [ ... ] }
    }
  }
}
```

The legacy root key `AcReport` is also accepted in place of `ACR.Report` (both resolve to
the same `Report` object).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ACR.Meta` | object | | Template metadata (see below). Not used by the layout/drawing engines. |
| `ACR.Report.PageSettings` | object | ✓ | Paper size, orientation, margins |
| `ACR.Report.Sections.Section` | array | ✓ | Ordered list of report sections |

### Meta Object

Informational only; carried over from the `RPX2ACR` conversion step and not interpreted
by the rendering engine.

```json
{
  "Author": "RPX2ACR",
  "CreatedAt": "2026-03-26 15:47:38",
  "LastEditor": "",
  "Source": "RPX",
  "UpdatedAt": "2026-03-26 15:47:38",
  "Version": "1.0"
}
```

---

## PageSettings Object

Defines the physical page size, orientation, and margins, all in twips.

```json
{
  "PaperName": "ユーザー定義のサイズ",
  "PaperSize": 9,
  "PaperWidth": 11906,
  "PaperHeight": 16838,
  "Orientation": 2,
  "TopMargin": 850,
  "BottomMargin": 397,
  "LeftMargin": 680,
  "RightMargin": 567
}
```

| Field | Type | Description |
|-------|------|-------------|
| `PaperWidth` | number | Paper width in twips (portrait orientation) |
| `PaperHeight` | number | Paper height in twips (portrait orientation) |
| `PaperSize` | number | Windows `DMPAPER_*` paper size code |
| `PaperName` | string | Display name of the paper size |
| `Orientation` | number | `1` = portrait, `2` = landscape. When `2`, the effective content width/height is `PaperHeight`×`PaperWidth` (swapped) |
| `TopMargin` / `BottomMargin` / `LeftMargin` / `RightMargin` | number | Page margins in twips |

**Note:** unlike the paper dimensions, margins are each a separate top-level field —
there is no nested `margin` object.

---

## Section Model

Sections are listed in `ACR.Report.Sections.Section` (an array) and rendered to the page
in order. The engine also accepts a bare array at `Sections`, or `Section`/`Controls`
wrapped differently — see "Schema tolerance" below.

### Confirmed Section Types

| `Type` value | Rendered | Description |
|--------------|----------|-------------|
| `ACR.PageHeader` | Top of every page | Column headings, page title |
| `ACR.GroupHeader` | Start of each data group (nestable) | Group label, subtotal header |
| `ACR.Detail` | Once per data record | Main content rows |
| `ACR.GroupFooter` | End of each data group (nestable) | Group subtotals |
| `ACR.PageFooter` | Bottom of every page | Page numbers, date |

`ACR.ReportHeader` and `ACR.ReportFooter` are described conceptually in
[design-decisions.md](design-decisions.md) (ADR-003) as once-per-report sections, but no
sample template using them has been confirmed yet — treat their field-level format as
unconfirmed until verified against the engine.

### Section Fields

```json
{
  "Name": "groupheader1",
  "Type": "ACR.GroupHeader",
  "Height": 850,
  "BackColor": 16777215,
  "DataField": "TENPOCD",
  "GroupKeepTogether": 1,
  "KeepTogether": 1,
  "NewPage": 1,
  "RepeatStyle": 4,
  "Control": [ ... ],
  "_tag": "Section"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `Name` | string | Section name, used as its display label in the VS Code preview |
| `Type` | string | Section type, see table above |
| `Height` | number | Section height in twips |
| `BackColor` | number | Background color, see "Color Format" below |
| `DataField` | string | Group key field (`ACR.GroupHeader` / `ACR.GroupFooter` only) |
| `GroupKeepTogether`, `KeepTogether`, `NewPage`, `RepeatStyle` | number | Group/page-break control flags carried over from the source report tool. Exact enum semantics are not fully documented here yet — treat existing sample values as the reference. |
| `rowsPerPage` | number | `ACR.Detail` only; `0` means unlimited |
| `Control` | array | Controls within the section (see below) |
| `_tag` | string | Always `"Section"`; legacy discriminator field, safe to keep as-is |

### Group Nesting

`ACR.GroupHeader` / `ACR.GroupFooter` sections can be paired and nested for multi-level
grouping, following the usual section order:

```
ACR.PageHeader
ACR.GroupHeader   (outer group)
ACR.GroupHeader   (inner group)
ACR.Detail
ACR.GroupFooter   (inner group)
ACR.GroupFooter   (outer group)
ACR.PageFooter
```

---

## Controls

Controls are the drawable elements within a section, listed under the section's
`Control` array.

### Schema tolerance

The engine accepts controls under any of these three shapes (all equivalent):

```json
"Control": [ ... ]
"Controls": [ ... ]
"Controls": { "Control": [ ... ] }
```

### Common Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `Type` | string | ✓ | Control type, see below |
| `Name` | string | ✓ | Control name. Shown as a placeholder label (`[Name]`) when an `ACR.Field` has no sample `Text` |
| `Left` | number | ✓ | X position in twips, relative to the section's top-left corner (`X` is also accepted) |
| `Top` | number | ✓ | Y position in twips, relative to the section's top-left corner (`Y` is also accepted) |
| `Width` | number | ✓ | Width in twips |
| `Height` | number | ✓ | Height in twips |
| `Style` | string | | CSS-like declaration list, see "Style" below |
| `_tag` | string | | Always `"Control"`; legacy discriminator field |

### Confirmed Control Types

#### `ACR.Label`

Static text that does not change with the data source (e.g. column headings).

```json
{
  "Type": "ACR.Label",
  "Name": "lblcampany",
  "Text": "会社名：",
  "Style": "font-family: ＭＳ ゴシック; font-size: 9.75pt; font-weight: bold",
  "Left": 0, "Top": 28.34646, "Width": 1133.858, "Height": 226.7716
}
```

| Field | Type | Description |
|-------|------|-------------|
| `Text` | string / number | The literal text to display |

#### `ACR.Field`

Data-bound text. The value shown at render time comes from `DataField`, resolved
against the active data record; `Text` (if present) is only a design-time sample value
shown in the VS Code preview and is not used for the final render.

```json
{
  "Type": "ACR.Field",
  "Name": "txtunitcost",
  "DataField": "UNITCOST",
  "OutputFormat": "¥#,##0",
  "CurrencyCulture": "ja-JP",
  "Multiline": 0,
  "Style": "font-family: MS UI Gothic; font-size: 8.25pt; text-align: right; vertical-align: middle",
  "Left": 10771.65, "Top": 28.34646, "Width": 1020.473, "Height": 255.118
}
```

| Field | Type | Description |
|-------|------|-------------|
| `DataField` | string | Name of the data source field this control is bound to |
| `Text` | string / number | Design-time sample value only (not rendered) |
| `OutputFormat` | string | .NET-style format string applied to the bound value, e.g. `"yyyy/MM/dd"`, `"¥#,##0"`, `"#,##0.0"` |
| `CurrencyCulture` | string | Locale used together with `OutputFormat` for currency formatting, e.g. `"ja-JP"` |
| `Multiline` | number | `0` = single line, `1` = wraps within the control bounds |
| `SummaryType`, `SummaryRunning` | number | Aggregate/running-value controls (e.g. page number, page count). Exact enum semantics are not fully documented here yet — treat existing sample values as the reference. |

Fields without a bound data record yet (e.g. page number before layout runs) can carry a
literal `Text` that is used as the initial/default value.

#### `ACR.Shape`

A filled and/or outlined rectangle, typically used as a row/column background.

```json
{
  "Type": "ACR.Shape",
  "Name": "shape2",
  "BackColor": 14474460,
  "BackStyle": 1,
  "LineColor": 16777215,
  "RoundingRadius": 9.999999,
  "Left": 0, "Top": 283.4646, "Width": 15590.55, "Height": 283.4646
}
```

| Field | Type | Description |
|-------|------|-------------|
| `BackColor` | number | Fill color, see "Color Format" below |
| `BackStyle` | number | `0` = transparent (no fill), `1` = filled with `BackColor` |
| `LineColor` | number | Border color |
| `LineStyle` | number | `1` = solid (default), `2` = dashed, `3` = dotted |
| `RoundingRadius` | number | Corner radius in twips; omit or `0` for square corners |

#### `ACR.Line`

A straight line, positioned independently of `Left`/`Top`/`Width`/`Height`.

```json
{
  "Type": "ACR.Line",
  "Name": "line1",
  "X1": 0, "Y1": 1417.323,
  "X2": 15590.55, "Y2": 1417.323,
  "LineColor": 0,
  "LineWeight": 1,
  "LineStyle": 1
}
```

| Field | Type | Description |
|-------|------|-------------|
| `X1`, `Y1` | number | Start point in twips, relative to the section |
| `X2`, `Y2` | number | End point in twips, relative to the section |
| `LineColor` | number | Line color, see "Color Format" below |
| `LineWeight` | number | Line thickness in twips (minimum effective value is 1) |
| `LineStyle` / `BorderStyle` | number | `1` = solid (default), `2` = dashed, `3` = dotted |

### Control Types Referenced but Not Yet Confirmed

The VS Code extension's data-field tooling (`list_data_fields`) classifies bound fields
as `'Text' \| 'Barcode' \| 'Image' \| 'Group'`, implying `ACR.Barcode` and `ACR.Image` (or
similarly named) control types exist in the engine. No sample template containing one was
available at the time of writing, so their field-level JSON shape is not documented here.
Do not assume a shape for them (e.g. do not assume `data`/`symbology`/`src`/`sizing`
fields from the old draft spec) without confirming against an actual template or the
engine source.

---

## Style

`Style` is a `;`-separated list of `property: value` declarations, parsed
case-insensitively on the property name, similar to an inline CSS `style` attribute.
Only the following properties are interpreted by the drawing/preview engines:

| Property | Values | Applies to |
|----------|--------|------------|
| `font-family` | font name | Label / Field |
| `font-size` | number + `pt` | Label / Field |
| `font-weight` | `normal` / `bold` | Label / Field |
| `font-style` | `normal` / `italic` | Label / Field |
| `text-align` | `left` / `center` / `right` | Label / Field |
| `vertical-align` | `top` / `middle` / `bottom` | Label / Field |

`ddo-char-set` and other declarations may appear (carried over from the source report
tool) but are not interpreted by the engine.

Example: `"font-family: ＭＳ ゴシック; font-size: 9.75pt; font-weight: bold; text-align: center; vertical-align: middle"`

---

## Color Format

Colors (`BackColor`, `LineColor`) are a single integer encoding `0xRRGGBB` — e.g.
`16777215` = `0xFFFFFF` = white, `0` = `0x000000` = black. This is a plain RGB integer,
**not** the Windows OLE `0x00BBGGRR` byte order.

---

## Data Binding

Unlike field-call-style template languages, ACR does not embed binding expressions
inside `Text`. Instead:

- An `ACR.Field` control's `DataField` names the data source field to pull the rendered
  value from.
- `ACR.GroupHeader` / `ACR.GroupFooter` sections' `DataField` names the field used to
  detect group boundaries.
- Any literal `Text` on an `ACR.Field` is a design-time placeholder only.

The data source itself (its own JSON shape, how it's supplied to the engine, and how
`OutputFormat`/`CurrencyCulture` map to format strings) is outside the scope of this
document and should be documented separately once confirmed against the engine.

---

## Rendering Model

```
1. Load template JSON (ACR.Report, or legacy AcReport)
2. Resolve PageSettings (swap width/height when Orientation == 2)
3. Bind data source records to Detail / GroupHeader / GroupFooter sections
4. Resolve group boundaries via each group section's DataField
5. Render each section's Control list via Google Skia (1-dot/twip precision)
6. Output to PDF or PNG
```

No printer driver is required at any stage.

---

## Implementation Languages

| Language | Status |
|----------|--------|
| Rust | Reference implementation (`acr_core`, private repository) |

---

## Changelog

| Version | Date | Description |
|---------|------|-------------|
| 1.0 | 2026-02-26 | Initial release (draft schema, not implemented) |
| 2.0 | 2026-10-15 | Rewritten to match the schema actually produced by `RPX2ACR` and consumed by `acr_core` / `acr-viewer-vscode` |

---

*ACR Specification — acrossreport/acr-spec*
