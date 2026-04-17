Each layer has a single responsibility:

| Layer | Role |
|---|---|
| **Template** | JSON-based report definition. Platform-neutral. |
| **Layout Engine** | Resolves sections, binds data, calculates positions. |
| **Drawing Engine** | Renders to pixels using Google Skia. 1-dot precision. |
| **Output** | PDF, PNG, SVG, or any target format. |

This separation means:
- The same template runs on Windows, macOS, and Linux without modification.
- Preview in the designer is **pixel-identical** to the final printed output (WYSIWYG).
- Output format is a detail — not a constraint.

---

### Key Design Principles

**Printer independence**
ACR does not rely on printer drivers or OS print subsystems. Layout is calculated in device-independent units and rendered at the target resolution.

**JSON as the intermediate model**
Templates are defined in human-readable JSON. The layout engine produces a drawing model — also JSON — that can be inspected, cached, or transmitted independently of rendering.

**Skia-based rendering**
The drawing engine uses Google Skia, the same graphics library behind Chrome and Android. This guarantees consistent, high-fidelity output across all platforms.

**ActiveReports-compatible section model**
ACR adopts a familiar section-based structure (Report Header, Page Header, Detail, Group, Page Footer, Report Footer) compatible with ActiveReports conventions, minimizing migration effort for existing report definitions.

