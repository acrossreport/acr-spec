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

**Section-based report model**  
ACR adopts a familiar section-based structure (Report Header / Page Header / Group Header / Detail / Group Footer / Page Footer / Report Footer), widely used in business reporting, minimizing migration effort for existing report definitions.

**Hardware-free preview**  
A complete, pixel-accurate preview is available without physical printers, drivers, or specialized hardware. What you see is exactly what will be printed.

---

## Repository Structure

```
acr-spec/
├── README.md                  ← This file (English)
├── README.ja.md               ← 日本語版
├── README.fr.md               ← Version française
├── specification.md           ← Full ACR specification (English)
├── specification.ja.md        ← 仕様（日本語）
├── specification.fr.md        ← Spécification (français)
├── design-decisions.md        ← Architecture Decision Records (English)
├── design-decisions.ja.md     ← アーキテクチャ決定記録（日本語）
└── design-decisions.fr.md     ← Historique des décisions d'architecture (français)
```

---

## Architecture Decision Records (ADR)

Design decisions are documented as ADRs in [design-decisions.md](design-decisions.md).

| ADR | Title | Status |
|-----|-------|--------|
| ADR-001 | Adopt Google Skia | Accepted |
| ADR-002 | Adopt JSON as the Intermediate Drawing Model for Templates | Accepted |
| ADR-003 | Adopt a Section-Based Report Structure | Accepted |
| ADR-004 | WYSIWYG Precision as the Top Design Principle | Accepted |
| ADR-005 | Adopt a Hardware-Free Preview Philosophy | Accepted |

日本語版: [design-decisions.ja.md](design-decisions.ja.md) / Français : [design-decisions.fr.md](design-decisions.fr.md)

---

## Documentation

- [Specification](specification.md) — Full ACR template and rendering specification
- [Architecture Decision Records](design-decisions.md) — Design rationale behind ACR

---

## Status

ACR is under active development. The specification and schema are stabilizing.  
See [acrossreport.com](https://acrossreport.com) for products built on this specification.

---

## License

See [LICENSE](LICENSE) for details.
