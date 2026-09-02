# ACR Free Canvas Template Specification (Draft v0.1)

## 1. Overview

Free Canvas is an absolute-coordinate-based layout mode that coexists alongside the existing section-based template mode (ActiveReports RPX/RDLX-compatible).

Free Canvas primarily targets:

- Reproducing existing paper forms as-is after scanning/photographing them in the field, via the pngback pipeline (OCR + ruled-line detection import)
- Free-form documents not bound to the section concept (page header/detail/footer, etc.) — labels, receipts, arbitrary-format documents

Free Canvas shares the same "JSON intermediate drawing model → Skia backend conversion" pipeline as the section-based mode; there is no difference at the output stage (PDF / PNG / ESC-POS / ZPL / SBPL / TPCL). The only difference is confined to the **input JSON structure** of the template.

## 2. Coordinate System & Units

Free Canvas inherits the coordinate system definition of the section-based templates as-is (no independent definition is introduced).

- Unit: mm (internally held as integer values at 1/100 mm precision to eliminate rounding error)
- Origin: top-left of the page (0, 0)
- Axis direction: X = positive to the right, Y = positive downward
- Page size: `page.width_mm` / `page.height_mm` (reuses the existing page definition keys)

pngback (`png_to_json.py` / `layout_analyze.py`) is responsible for the px → mm conversion; Free Canvas does not assume the scale calculation involved when `--dpi` and `--paper-size` are specified. Free Canvas only ever receives **already-converted mm values**.

> Open issue: the origin-shift problem on the pngback side when "the PNG does not cover the entire page (i.e., it has been cropped/trimmed)" (a known issue noted in pngback.md). Until this is resolved, the Free Canvas import JSON carries an optional `origin_offset_mm: {x, y}` field so the shift can be recorded explicitly (a provisional measure, still under discussion).

## 3. Top-Level Structure

```json
{
  "schema_version": "freecanvas-0.1",
  "canvas": {
    "page": { "width_mm": 210.0, "height_mm": 297.0 },
    "origin_offset_mm": { "x": 0.0, "y": 0.0 }
  },
  "elements": [ ... ],
  "source": {
    "type": "pngback_ocr",
    "generated_by": "acrpng2json",
    "original_image": "001.png",
    "dpi": 96
  }
}
```

- `schema_version`: a version string explicitly marking this as the Free Canvas-specific schema (kept distinct from the section-based template's `schema_version`)
- `source`: traceability information indicating whether the template originated from pngback (optional; can be omitted for manually authored templates)

## 4. Element Placement Model

Unlike the section-based mode, where elements belong to a row/band, each element in Free Canvas **carries its page coordinates directly**.

```json
{
  "id": "el_0001",
  "type": "text",
  "x_mm": 12.5,
  "y_mm": 20.0,
  "width_mm": 60.0,
  "height_mm": 6.0,
  "z_index": 0,
  "rotation_deg": 0.0,
  "confidence": 0.94,
  "props": { ... }
}
```

Common fields:

| Field | Description |
|---|---|
| `id` | Unique ID within the template |
| `type` | `text` / `line` / `rect` / `image` / `barcode`, etc. (reuses the existing element type definitions from the section-based mode) |
| `x_mm`, `y_mm` | Top-left coordinates of the element's bounding box |
| `width_mm`, `height_mm` | Bounding box size |
| `z_index` | Stacking order (defaults to array order if omitted) |
| `rotation_deg` | Rotation angle (defaults to 0 if omitted) |
| `confidence` | Confidence score when derived from OCR/ruled-line detection (0.0–1.0; omitted for manually authored elements) |
| `props` | Type-specific properties (conforms to the existing per-element-type definitions from the section-based mode) |

**Policy**: The internal schema of `type` and `props` is reused as-is from the section-based mode's existing definitions; Free Canvas does not introduce its own separate set of rendering properties. This keeps the intermediate drawing-command generation logic shared (avoiding duplicated implementation and branching).

## 5. Ruled-Line Elements

pngback's ruled-line detection results are represented as `type: "line"` and `type: "rect"` elements.

```json
{
  "id": "el_0002",
  "type": "line",
  "x_mm": 10.0, "y_mm": 30.0,
  "width_mm": 190.0, "height_mm": 0.0,
  "props": { "stroke_width_mm": 0.2, "orientation": "horizontal" }
}
```

- `orientation` is a rendering-time optimization hint and is not strictly required, since it can effectively be derived from `width_mm`/`height_mm`
- Table (grid) ruled lines are represented as a collection of individual `line` elements; the logical structure of a table (e.g., cell merging) is out of scope for this v0.1 (future extension: `type: "table"`)

## 6. Coexistence Rules with the Section-Based Mode

1. A single template file must be in either Free Canvas mode or section-based mode — never a mix (determined by `schema_version`)
2. Converting a Free Canvas template back into section-based form after editing in AcrDesigner is not supported (the flow is one-directional: pngback → Free Canvas → manual fine-tuning; reverse conversion to section-based mode is out of scope)
3. The precision guarantee for printing/preview (WYSIWYG accurate to a single dot) applies equally to Free Canvas. Coordinate conversion and rounding processing use the same code path shared with the section-based mode

## 7. Open Items (Pending Discussion)

- Handling of `origin_offset_mm` (provisional; to be reconsidered once the pngback-side issue is resolved)
- Table structure (logical grouping of ruled lines including cell merging)
- Whether AcrDesigner should surface a UI warning based on a `confidence` threshold
- Scope of publication in the acr-spec repository (the JSON intermediate model schema itself aligns with the public-release policy, but how much of the pngback integration to expose is undecided)
