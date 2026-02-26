# ACR (Across Report) Template Specification

ACR is a printer-independent template format for high-speed and direct printing.

ACR separates layout rendering from printer control.

## Why ACR?

Traditional printing relies on vendor-specific command sets such as ESC/POS.

ACR introduces a rendering-first architecture.

Instead of generating printer commands directly,
ACR defines layout in a device-independent coordinate system
and renders through a unified drawing engine.

This ensures:

- Consistent output across PDF, Image, and Printer
- No vendor lock-in
- WebAssembly compatibility
- Serverless direct printing capability

ACR is not just a template format.
It is a rendering architecture.


## Features

- Printer independent
- JSON based template format
- High performance rendering
- Direct conversion to ESC/POS, Star, SATO, TEC, and other printer command languages
- Cross-platform support (Windows, Linux, Android, iOS, WebAssembly)

## Specification

See specification.md for full details.

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
```

---

## Files

- Specification: [specification.md](./specification.md)
- JSON Schema: [template.schema.json](./template.schema.json)
- Example Template: [examples/receipt.json](./examples/receipt.json)

## Version

ACR Template Specification v1.0


