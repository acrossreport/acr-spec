# ADR-001: ACR (Across Report Renderer) — Architecture Decision Record

**Status:** Accepted  
**Created:** 2026-05-24  
**Author:** Hisanobu Araki (MaskedRiderSystem)  
**Version:** 0.x (In Progress)

---

## Table of Contents

1. [Background and Problem Statement](#1-background-and-problem-statement)
2. [Decision Index](#2-decision-index)
3. [ADR-001: Adopt Google Skia as the Rendering Engine](#adr-001-adopt-google-skia-as-the-rendering-engine)
4. [ADR-002: Adopt JSON as the Intermediate Drawing Model for Templates](#adr-002-adopt-json-as-the-intermediate-drawing-model-for-templates)
5. [ADR-003: Adopt ActiveReports-Compatible Section Structure](#adr-003-adopt-activereports-compatible-section-structure)
6. [ADR-004: Establish WYSIWYG Pixel Accuracy as the Primary Design Principle](#adr-004-establish-wysiwyg-pixel-accuracy-as-the-primary-design-principle)
7. [ADR-005: Adopt the Hardware-Free Preview Philosophy](#adr-005-adopt-the-hardware-free-preview-philosophy)
8. [Changelog](#changelog)

---

## 1. Background and Problem Statement

Production report-generation environments suffer from chronic, systemic problems:

- **Environment-dependent output variance:** Pixel-level discrepancies caused by OS, printer driver, and font renderer differences make it nearly impossible to guarantee report output quality.
- **Limitations of commercial engines:** Off-the-shelf report engines are black boxes that cannot accommodate site-specific operational requirements such as complex section structures and dynamic layouts.
- **Mismatch between preview and print:** Even products marketed as WYSIWYG frequently produce print results that differ from the on-screen preview.
- **Licensing and maintainability:** Dependency on a specific vendor generates ongoing license costs and technical debt.

To address these problems, the proprietary report rendering engine **ACR (Across Report Renderer)** is being designed and developed.

---

## 2. Decision Index

| ADR | Title | Status |
|-----|-------|--------|
| ADR-001 | Adopt Google Skia as the Rendering Engine | Accepted |
| ADR-002 | Adopt JSON as the Intermediate Drawing Model for Templates | Accepted |
| ADR-003 | Adopt ActiveReports-Compatible Section Structure | Accepted |
| ADR-004 | Establish WYSIWYG Pixel Accuracy as the Primary Design Principle | Accepted |
| ADR-005 | Adopt the Hardware-Free Preview Philosophy | Accepted |

---

## ADR-001: Adopt Google Skia as the Rendering Engine

### Context

A graphics engine must be selected to produce the final report output (PDF and on-screen preview). The following candidates were evaluated:

- **GDI/GDI+:** Windows-only. Known pixel accuracy issues.
- **Cairo:** Cross-platform, but text rendering quality is unstable due to dependency on the underlying font engine.
- **Skia:** The rendering foundation of Google Chrome, Android, and Flutter. High pixel accuracy; supports direct PDF output.
- **PDFium:** Designed solely for PDF generation. Unsuitable for interactive previews.

### Decision

**Adopt Google Skia as the sole rendering engine for ACR.**

### Rationale

1. **Consistent pixel accuracy:** Skia uses the same code path for both on-screen drawing and PDF output, guaranteeing theoretical pixel-perfect agreement between preview and print.
2. **Platform neutrality:** Identical rendering results are achievable on Windows, macOS, and Linux.
3. **Proven track record:** Skia powers the world's most widely used browser engine (Chrome), demonstrating sufficient quality and stability.
4. **Direct PDF generation:** `SkDocument` enables native PDF output, eliminating intermediate conversion layers.
5. **Font rendering:** Integration with FreeType, DirectWrite, and CoreText allows accurate use of OS-native font engines.

### Resulting Trade-offs

| Benefits | Drawbacks |
|----------|-----------|
| High-accuracy pixel matching | Requires building and distributing a native library |
| Unified code path for PDF and screen | Learning curve for the Skia API |
| Cross-platform guarantee | Version management tied to Chromium release cycle |

---

## ADR-002: Adopt JSON as the Intermediate Drawing Model for Templates

### Context

Several formats were considered for report template definitions: XML, proprietary binary, and JSON. In a report engine, the template is the central artifact that bridges "expressed intent" and "drawing instructions."

### Decision

**Adopt JSON as the sole intermediate drawing model format for ACR templates.**

JSON is not treated merely as a data format. Within ACR it is positioned as an **intermediate language for expressing drawing intent**.

### Rationale

1. **Human readability:** Less noisy than XML, making template development and debugging significantly easier.
2. **Type-safe schema definition:** Strict validation via JSON Schema is straightforward to implement.
3. **Toolchain affinity:** Easy integration with visual editors, diff tools, and CI/CD pipelines.
4. **Clear responsibility as an intermediate model:** The JSON template defines *what to draw*; the Skia layer defines *how to draw it*. Separation of concerns is explicit.
5. **Dynamic generation:** Data binding and conditional logic can be expressed naturally within the JSON structure.

### Conceptual Template Structure

```json
{
  "version": "1.0",
  "pageSize": { "width": 210, "height": 297, "unit": "mm" },
  "sections": [
    {
      "type": "ReportHeader",
      "height": 20,
      "elements": [
        {
          "type": "TextBox",
          "x": 10, "y": 5, "width": 190, "height": 10,
          "text": "{{ reportTitle }}",
          "font": { "family": "Arial", "size": 14, "bold": true },
          "alignment": "center"
        }
      ]
    }
  ]
}
```

### Resulting Trade-offs

| Benefits | Drawbacks |
|----------|-----------|
| Human-readable, easy to version-control | More verbose than binary; larger file size |
| Schema validation feasible | Ongoing cost of maintaining JSON Schema |
| Easy toolchain integration | Expressing complex expressions/scripts has limitations |

---

## ADR-003: Adopt ActiveReports-Compatible Section Structure

### Context

The question was whether to use the section structure of ActiveReports — widely adopted in production environments (PageHeader / GroupHeader / Detail / GroupFooter / PageFooter / ReportHeader / ReportFooter) — as the reference design for ACR's layout model.

### Decision

**Adopt the ActiveReports-compatible section structure as the standard layout model for ACR.**

This does not prevent ACR-specific extensions; compatibility is the *design starting point*, not a hard constraint.

### Rationale

1. **Leveraging existing knowledge:** Many field engineers already understand the ActiveReports conceptual model, minimizing learning costs.
2. **Reduced migration cost:** Existing ActiveReports report definitions can be incrementally migrated to ACR.
3. **Battle-tested design:** The ActiveReports section structure has been validated against over 30 years of report processing requirements and reflects operational needs at a high level.
4. **Expressiveness for group aggregation and page-break control:** The section model naturally represents complex aggregation reports and conditional page-break scenarios.

### Section Definitions

| Section | Role |
|---------|------|
| `ReportHeader` | Output once at the beginning of the entire report |
| `PageHeader` | Output at the top of each page |
| `GroupHeader` | Output at the start of each group (nestable) |
| `Detail` | Repeated output for each data record |
| `GroupFooter` | Output at the end of each group (nestable) |
| `PageFooter` | Output at the bottom of each page |
| `ReportFooter` | Output once at the end of the entire report |

### Resulting Trade-offs

| Benefits | Drawbacks |
|----------|-----------|
| No additional learning required for field engineers | Cost of maintaining full ActiveReports compatibility |
| Easy migration of existing report assets | Some ACR-specific design choices may be constrained |
| Inherits proven design philosophy | Risk of inheriting design limitations present in ActiveReports |

---

## ADR-004: Establish WYSIWYG Pixel Accuracy as the Primary Design Principle

### Context

The question was whether to place "accuracy of agreement between on-screen preview and print output" at the top of the design principle hierarchy — above performance, development velocity, and compatibility.

### Decision

**ACR adopts "pixel-perfect WYSIWYG — not a single dot of deviation" as its primary and inviolable design principle.**

This principle takes precedence over performance, development speed, and compatibility. It cannot be traded away.

### Rationale

1. **Social responsibility of report output:** For legally binding documents (invoices, delivery slips, payroll statements), any discrepancy between preview and print constitutes a business and legal risk.
2. **Field operator trust:** The confidence that "the printout will exactly match what I see in the preview" is the foundation of field operator productivity.
3. **Reduced debugging cost:** Guaranteed preview-print agreement dramatically reduces the investigation cost of bugs caused by environmental differences.

### Implementation Constraints

- All coordinate calculations, text placement, and page-break logic must share the same Skia API code path for both screen rendering and PDF generation.
- Font metrics (glyph width, height, kerning) are cached at runtime and identical values are referenced for both preview and PDF output.
- Sub-pixel fractional values must always apply the same rounding rule (floor/ceil) consistently.

---

## ADR-005: Adopt the Hardware-Free Preview Philosophy

### Context

The question was whether to make "complete preview capability without access to physical printers or report output devices" a central design guideline for the development and verification cycle.

### Decision

**ACR places the "hardware-free philosophy" at the center of its design.**

ACR guarantees that developers and operations staff can verify accurate previews without requiring a printer, report terminal, or specialized paper stock.

### Rationale

1. **Development efficiency:** Template creation, modification, and verification cycles can be accelerated without setting up physical hardware.
2. **CI/CD integration:** Generating and comparing preview images in automated test pipelines enables regression testing for report output.
3. **Eliminating geographic and cost barriers:** Accurate output can be verified remotely even when development facilities and print equipment are in different locations.
4. **Patent-level differentiation:** "Pixel-perfect preview without physical hardware" is the core technical differentiator of ACR.

### Technical Requirements for Realization

- Off-screen rendering via Skia's `SkSurface` to generate PNG/JPEG previews.
- Software simulation of paper size, margins, and resolution (DPI).
- Environment-independent text rendering via font embedding.
- Maintenance of a purely software-based stack that requires no virtual printer driver.

---

## Changelog

| Version | Date | Description | Author |
|---------|------|-------------|--------|
| 0.1 | 2026-05-24 | Initial draft | MaskedRiderSystem |

---

*This document is updated continuously as the ACR project progresses.*  
*Once accepted, an ADR's status changes only when it is deprecated or superseded.*
