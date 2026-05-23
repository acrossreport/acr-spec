# ACR Specification

**ACR (Across Report)** is a printer-independent reporting specification designed for modern software systems.

> *Render once. Output everywhere.*

ACR separates layout description, rendering, and output generation —  
allowing the same template to be rendered across multiple platforms and output formats.

---

## Why ACR

Traditional reporting systems are tightly coupled to specific printers, rendering engines, or frameworks.

ACR introduces a clean, layered architecture:

```
Template → Layout Engine → Drawing Engine → Output
```

Each layer has a single responsibility:

| Layer              | Role                                                                 |
|--------------------|----------------------------------------------------------------------|
| **Template**       | JSON-based report definition. Platform-neutral.                      |
| **Layout Engine**  | Resolves sections, binds data, calculates positions.                 |
| **Drawing Engine** | Renders to pixels using Google Skia. 1-dot precision.                |
| **Output**         | PDF, PNG, SVG, or any target format.                                 |

This separation means:

- The same template runs on Windows, macOS, and Linux without modification.
- Preview in the designer is **pixel-identical** to the final printed output (WYSIWYG).
- Output format is a detail — not a constraint.

---

## Key Design Principles

**Printer independence**  
ACR does not rely on printer drivers or OS print subsystems. Layout is calculated in device-independent units and rendered at the target resolution.

**JSON as the intermediate model**  
Templates are defined in human-readable JSON. The layout engine produces a drawing model — also JSON — that can be inspected, cached, or transmitted independently of rendering.

**Skia-based rendering**  
The drawing engine uses Google Skia, the same graphics library behind Chrome and Android. This guarantees consistent, high-fidelity output across all platforms.

**ActiveReports-compatible section model**  
ACR adopts a familiar section-based structure (Report Header / Page Header / Group Header / Detail / Group Footer / Page Footer / Report Footer) compatible with ActiveReports conventions, minimizing migration effort for existing report definitions.

**Hardware-free preview**  
A complete, pixel-accurate preview is available without physical printers, drivers, or specialized hardware. What you see is exactly what will be printed.

---

## Repository Structure

```
acr-spec/
├── README.md                  ← This file
├── specification.md           ← Full ACR specification
├── template.schema.json       ← JSON Schema for template validation
├── docs/
│   └── adr/                   ← Architecture Decision Records
│       ├── README.md
│       ├── ADR-001-ACR-ja.md  ← アーキテクチャ決定記録（日本語）
│       └── ADR-001-ACR-en.md  ← Architecture Decision Records (English)
└── examples/
    └── invoice/               ← Sample invoice template
```

---

## Architecture Decision Records (ADR)

Design decisions are documented as ADRs in [`docs/adr/`](docs/adr/).

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-001](docs/adr/ADR-001-ACR-en.md) | Adopt Google Skia / JSON Model / Section Structure / WYSIWYG / Hardware-free Preview | Accepted |

日本語版: [`docs/adr/ADR-001-ACR-ja.md`](docs/adr/ADR-001-ACR-ja.md)

---

## Documentation

- [Specification](specification.md) — Full ACR template and rendering specification
- [JSON Schema](template.schema.json) — Machine-readable schema for template validation
- [ADR (English)](docs/adr/ADR-001-ACR-en.md) — Architecture decisions in English
- [ADR（日本語）](docs/adr/ADR-001-ACR-ja.md) — アーキテクチャ決定記録

---

## Status

ACR is under active development. The specification and schema are stabilizing.  
See [acr-engine](https://github.com/acrossreport/acr-engine) for the reference implementation in Rust.

---

## License

See [LICENSE](LICENSE) for details.
