# ACR (Across Report) Template Specification

ACR is a printer-independent, coordinate-based reporting specification.

Unlike traditional ESC/POS or vendor SDK based printing,
ACR separates layout rendering from printer control.

ACR defines report layout in a device-independent coordinate system
and renders through a unified drawing engine.

---

## Why ACR?

Traditional printing relies on vendor-specific command sets such as ESC/POS.

ACR introduces a rendering-first architecture.

Instead of generating printer commands directly,
ACR defines layout once and renders consistently across:

- PDF
- Image (PNG)
- Printer
- WebAssembly environments

This ensures:

- Consistent output across devices
- No vendor lock-in
- Cross-platform compatibility
- Serverless direct printing capability

ACR is not just a template format.
It is a rendering architecture.

---

## Features

- Printer independent
- JSON-based template format
- Coordinate-based layout system
- High-performance rendering
- Cross-platform support (Windows, Linux, Android, iOS, WebAssembly)

---

## Specification

See `specification.md` for full details.

---

## Example

```json
{
  "version": "1.0",
  "page": {
    "width": 576,
    "height": 800,
    "unit": "dot"
  },
  "controls": [
    {
      "type": "text",
      "x": 0,
      "y": 0,
      "text": "ACR Sample"
    }
  ]
}

---

## Version

ACR Template Specification v1.0

ACR aims to redefine how reporting and printing are built in modern software systems.

