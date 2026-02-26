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
