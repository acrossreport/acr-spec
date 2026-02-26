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

---

## 3. Root Structure

Example:

```json
{
  "version": "1.0",
  "page": {},
  "controls": []
}

---

## 4. Page Object

Example:

```json
{
  "width": 576,
  "height": 800,
  "unit": "dot"
}

---

```markdown
Fields:

width : number (dot)  
height : number (dot)  
unit : "dot"  

Notes:

width and height define logical page size.

Receipt printers may extend height dynamically.

---

## 5. Control Object

Control defines a drawable element.

Common fields:

type : string  
x : number (dot)  
y : number (dot)  

---

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

---

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

<img width="504" height="422" alt="image" src="https://github.com/user-attachments/assets/7029f5d0-6b46-45b1-ba08-7f8db86a6105" />

```markdown
Fields:

x1 : number (dot) — start position X  
y1 : number (dot) — start position Y  
x2 : number (dot) — end position X  
y2 : number (dot) — end position Y  
width : number (dot) — line thickness  

Notes:

Coordinates are absolute positions in the page coordinate system.

---

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

9. Barcode Control
Draws a barcode.

Example:
{
  "type": "barcode",
  ...
}

Fields:

data : string — barcode content  
symbology : string — barcode type  

Notes:

Common symbologies include:

CODE128  
CODE39  
EAN13  
EAN8  

---

## 10. ZIP Container Format

ACR templates may be packaged as ZIP files.

Structure:

template.json  
fonts/  
images/  
meta.json  

Description:

template.json — main template definition  
fonts/ — optional font files  
images/ — optional image resources  
meta.json — optional metadata  

---

## 11. Coordinate Rules

Origin is top-left corner.

Coordinate directions:

X increases to the right  
Y increases downward  

Unit is dot.

---

## 12. Rendering Model

Rendering process:

1. Load template.json  
2. Parse controls  
3. Convert controls to native printer commands  
4. Send commands to printer  

No printer driver is required.

---

## 13. Design Goals

Printer independent  
High performance  
Simple JSON structure  
Cross-platform compatibility  

---

## 14. Compatibility

ACR can be implemented in:

Rust  
C#  
C++  
Java  
Swift  
JavaScript  
WebAssembly  

Target outputs include:

ESC/POS  
StarPRNT  
SATO  
TEC  
PDF  
PNG  

---

## 15. Version

Specification version:

1.0



