# ACR (Across Report) Template Specification

ACR is a printer-independent template format for high-speed and direct printing.

ACR allows applications to generate printer-native commands from a unified JSON template without relying on printer drivers.

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
