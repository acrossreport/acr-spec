
# ACR Template Specification v1.0

## 1. Overview

ACR (Across Report) defines a printer-independent template format using JSON.

ACR allows applications to generate printer-native commands without relying on printer drivers.

Supported output targets include:

- ESC/POS
- StarPRNT
- SATO
- TEC
- PDF
- PNG

ACR enables high-speed and cross-platform printing.

---

## 2. Coordinate System

Unit: dot

dot = 1 / DPI inch

Example:

203 DPI printer:

1 inch = 203 dot

Origin is top-left corner.

X increases to the right.  
Y increases downward.

All coordinates use absolute positioning.

---

## 3. Root Structure

Example:

```json
{
  "version": "1.0",
  "page": {},
  "controls": []
}
```

Fields:

version : string  
page : Page object  
controls : array of Control objects  

---

## 4. Page Object

Defines the logical page size.

```markdown id="a6pq1h"
Example:
```json
{
  "width": 576,
  "height": 800,
  "unit": "dot"
}
```
Fields:

width : number (dot)
height : number (dot)
unit : string ("dot")

Receipt printers may extend height dynamically.

---

## 5. Control Object

Defines a drawable element.

Common fields:

type : string
x : number (dot)
y : number (dot)

Each control represents a renderable object.

## 6. Text Control

Draws text.

Example:
```json
{
  "type": "text",
  "x": 0,
  "y": 0,
  "text": "Hello ACR",
  "font": "default",
  "size": 24
}
```

Fields:
text : string
font : string (optional)
size : number (optional)

## 7. Line Control

Draws a line between two points.

Example:
```json
{
  "type": "line",
  "x1": 0,
  "y1": 100,
  "x2": 576,
  "y2": 100,
  "width": 2
}
```

Fields:
x1 : number (dot)
y1 : number (dot)
x2 : number (dot)
y2 : number (dot)
width : number (dot)

## 8. Image Control

Draws an image.

Example:
```json
{
  "type": "image",
  "x": 0,
  "y": 0,
  "width": 200,
  "height": 100,
  "src": "logo.png"
}
```

Fields:

src : string
width : number (dot)
height : number (dot)

## 9. Barcode Control

Draws a barcode.

Example:
```json
{
  "type": "barcode",
  "x": 100,
  "y": 300,
  "data": "123456789012",
  "symbology": "CODE128"
}
```

Fields:

data : string
symbology : string

Supported types may include:
```json
CODE128
CODE39
EAN13
EAN8

## 10. ZIP Container Format

ACR templates may be packaged as ZIP files.

Structure:
```json
template.json
fonts/
images/
meta.json
```

template.json is required.

## 11. Coordinate Rules

Origin is top-left corner.

```json
X increases to the right.
Y increases downward.
```

Unit is dot.

## 12. Rendering Model

Rendering process:

Load template.json

Parse controls

Convert controls to native printer commands

Send commands to printer

No printer driver is required.

## 13. Compatibility

ACR can be implemented in:

```Code
Rust
C#
C++
Java
Swift
JavaScript
WebAssembly
```

ACR can generate output for:

```Command
ESC/POS
StarPRNT
SATO
TEC
PDF
PNG
```

## 14. Version

Specification version:

1.0
