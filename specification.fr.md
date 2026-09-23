# Spécification des modèles ACR v1.0

## Présentation

ACR (Across Report) définit un format de modèle de rapport indépendant des imprimantes, basé sur JSON.

ACR sépare la définition de la mise en page, le rendu et la génération de la sortie en couches distinctes.  
Un même modèle produit un résultat identique sur toute plateforme et pour toute cible de sortie.

> *Render once. Output everywhere.*

### Cibles de sortie prises en charge

| Cible | Description |
|-------|-------------|
| PDF | Sortie de documents haute fidélité |
| PNG | Image matricielle précise au pixel près |
| SVG | Sortie vectorielle redimensionnable |
| ESC/POS | Imprimantes thermiques de tickets |
| StarPRNT | Imprimantes Star Micronics |
| SATO | Imprimantes d'étiquettes SATO |
| TEC | Imprimantes Toshiba TEC |

---

## Architecture

```
Modèle (JSON)
    ↓
Moteur de mise en page  — résout les sections, lie les données, calcule les positions
    ↓
Modèle de dessin (JSON) — représentation intermédiaire, inspectable et mise en cache possible
    ↓
Moteur de dessin        — rendu avec Google Skia (précision au point près)
    ↓
Sortie                  — PDF / PNG / ESC/POS / StarPRNT / SATO / TEC
```

### Principes de conception

**Indépendance vis-à-vis des imprimantes**  
ACR ne dépend ni des pilotes d'imprimante ni des sous-systèmes d'impression du système d'exploitation.  
La mise en page est calculée en unités indépendantes du périphérique, puis rendue à la résolution cible.

**Garantie WYSIWYG**  
L'aperçu à l'écran est identique au pixel près à la sortie imprimée finale.  
Aucun écart, même d'un seul point, n'est admis.

**Aperçu sans matériel**  
Un aperçu complet et précis au pixel près est disponible sans imprimante physique ni pilote.

**JSON comme modèle intermédiaire**  
Le modèle et le modèle de dessin sont tous deux en JSON.  
Ils peuvent être inspectés, mis en cache, versionnés et transmis indépendamment du rendu.

**Rendu basé sur Skia**  
Le moteur de dessin utilise Google Skia, la bibliothèque graphique utilisée par Chrome et Android.  
Cela garantit une sortie cohérente et fidèle sur toutes les plateformes.

**Modèle de rapport par sections**  
ACR adopte une structure de mise en page par sections bien connue, largement utilisée dans les rapports d'entreprise.

---

## Système de coordonnées

- **Unité :** point (dot)
- **Définition :** 1 point = 1/DPI pouce  
  Exemple : à 203 DPI, 1 pouce = 203 points
- **Origine :** coin supérieur gauche de la page
- **Axe X :** croît vers la droite
- **Axe Y :** croît vers le bas
- **Positionnement :** toutes les coordonnées sont en positionnement absolu

---

## Structure du modèle

Un modèle est un fichier JSON unique dont la structure racine est la suivante.

```json
{
  "version": "1.0",
  "page": { ... },
  "datasource": { ... },
  "sections": [ ... ]
}
```

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `version` | string | ✓ | Version de la spécification. Actuellement `"1.0"` |
| `page` | object | ✓ | Définition de la taille de page et des marges |
| `datasource` | object | | Configuration de la liaison de données |
| `sections` | array | ✓ | Liste ordonnée des sections du rapport |

---

## Objet Page

Définit la taille logique de la page et les marges.

```json
{
  "width": 2480,
  "height": 3508,
  "unit": "dot",
  "dpi": 300,
  "margin": {
    "top": 118,
    "bottom": 118,
    "left": 118,
    "right": 118
  }
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `width` | number | Largeur de la page en points |
| `height` | number | Hauteur de la page en points |
| `unit` | string | Toujours `"dot"` |
| `dpi` | number | Résolution cible (par ex. 300, 203, 96) |
| `margin` | object | Marges de la page en points (top / bottom / left / right) |

**Formats de page courants à 300 DPI :**

| Papier | Largeur (points) | Hauteur (points) |
|--------|------------------|------------------|
| A4 | 2480 | 3508 |
| Letter | 2550 | 3300 |
| Ticket 80 mm | 945 | variable |

---

## Modèle de sections

ACR utilise un modèle de mise en page par sections, largement utilisé dans les rapports d'entreprise.  
Les sections sont traitées dans l'ordre et rendues séquentiellement sur la page.

### Types de sections

| Section | Rendu | Description |
|---------|-------|-------------|
| `ReportHeader` | Une fois au début du rapport | Titre, logo, métadonnées du rapport |
| `PageHeader` | En haut de chaque page | En-têtes de colonnes, titre de page |
| `GroupHeader` | Au début de chaque groupe de données (imbricable) | Libellé du groupe, en-tête de sous-total |
| `Detail` | Une fois par enregistrement | Lignes de contenu principal |
| `GroupFooter` | À la fin de chaque groupe de données (imbricable) | Sous-totaux du groupe |
| `PageFooter` | En bas de chaque page | Numéros de page, date |
| `ReportFooter` | Une fois à la fin du rapport | Totaux généraux, signatures |

### Définition d'une section

```json
{
  "type": "PageHeader",
  "height": 120,
  "canGrow": false,
  "elements": [ ... ]
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `type` | string | Type de section (voir le tableau ci-dessus) |
| `height` | number | Hauteur de la section en points |
| `canGrow` | boolean | Indique si la section s'agrandit pour s'adapter au contenu |
| `groupKey` | string | Champ de données utilisé pour le regroupement (GroupHeader / GroupFooter uniquement) |
| `elements` | array | Liste des contrôles de la section |

### Imbrication des groupes

Les sections GroupHeader et GroupFooter peuvent être imbriquées pour représenter des regroupements à plusieurs niveaux.

```json
[
  { "type": "GroupHeader", "groupKey": "department", "elements": [...] },
  { "type": "GroupHeader", "groupKey": "category",   "elements": [...] },
  { "type": "Detail",                                 "elements": [...] },
  { "type": "GroupFooter", "groupKey": "category",   "elements": [...] },
  { "type": "GroupFooter", "groupKey": "department",  "elements": [...] }
]
```

---

## Contrôles

Les contrôles sont les éléments dessinables d'une section.

### Champs communs

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `type` | string | ✓ | Type de contrôle |
| `x` | number | ✓ | Position X en points (depuis le bord gauche de la section) |
| `y` | number | ✓ | Position Y en points (depuis le bord supérieur de la section) |
| `width` | number | ✓ | Largeur en points |
| `height` | number | ✓ | Hauteur en points |
| `visible` | boolean | | Par défaut : `true` |

---

### TextBox

Affiche du texte avec contrôle de la police et de l'alignement.

```json
{
  "type": "TextBox",
  "x": 0,
  "y": 0,
  "width": 1200,
  "height": 80,
  "text": "{{ invoiceTitle }}",
  "font": {
    "family": "IPAexMincho",
    "size": 24,
    "bold": true,
    "italic": false
  },
  "alignment": "center",
  "verticalAlignment": "middle",
  "color": "#000000",
  "canGrow": true
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `text` | string | Contenu du texte. Prend en charge la liaison de données `{{ field }}` |
| `font.family` | string | Nom de la police |
| `font.size` | number | Taille de la police en points typographiques |
| `font.bold` | boolean | Gras |
| `font.italic` | boolean | Italique |
| `alignment` | string | `left` / `center` / `right` |
| `verticalAlignment` | string | `top` / `middle` / `bottom` |
| `color` | string | Couleur du texte en hexadécimal |
| `canGrow` | boolean | Agrandit la hauteur pour s'adapter au contenu |

---

### Line

Trace une ligne droite.

```json
{
  "type": "Line",
  "x": 0,
  "y": 118,
  "x2": 2244,
  "y2": 118,
  "lineWidth": 2,
  "color": "#000000"
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `x2` | number | Position X de fin en points |
| `y2` | number | Position Y de fin en points |
| `lineWidth` | number | Épaisseur du trait en points |
| `color` | string | Couleur du trait en hexadécimal |

---

### Rectangle

Dessine un rectangle plein ou avec contour.

```json
{
  "type": "Rectangle",
  "x": 0,
  "y": 0,
  "width": 2244,
  "height": 120,
  "lineWidth": 1,
  "borderColor": "#000000",
  "fillColor": "#F0F0F0"
}
```

---

### Image

Affiche une image.

```json
{
  "type": "Image",
  "x": 0,
  "y": 0,
  "width": 300,
  "height": 300,
  "src": "images/logo.png",
  "sizing": "fit"
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `src` | string | Chemin dans le conteneur ZIP |
| `sizing` | string | `fit` / `fill` / `clip` |

---

### Barcode

Affiche un code-barres ou un code QR.

```json
{
  "type": "Barcode",
  "x": 100,
  "y": 300,
  "width": 600,
  "height": 120,
  "data": "{{ orderCode }}",
  "symbology": "CODE128",
  "showText": true
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `data` | string | Données du code-barres. Prend en charge la liaison de données |
| `symbology` | string | `CODE128` / `CODE39` / `EAN13` / `EAN8` / `QR` |
| `showText` | boolean | Affiche le texte lisible sous le code-barres |

---

## Liaison de données

Les valeurs des champs sont liées à l'aide de la syntaxe `{{ fieldName }}` dans les propriétés `text` et `data`.

```json
{ "text": "Invoice No: {{ invoiceNumber }}" }
{ "text": "Total: {{ formatCurrency(totalAmount) }}" }
{ "text": "Page {{ pageNumber }} of {{ pageCount }}" }
```

### Variables intégrées

| Variable | Description |
|----------|-------------|
| `pageNumber` | Numéro de la page courante |
| `pageCount` | Nombre total de pages |
| `reportDate` | Date de génération du rapport |

---

## Format du conteneur ZIP

Les modèles ACR peuvent être regroupés dans une archive ZIP pour leur distribution.

```
template.acr  (ZIP)
├── template.json   ← Définition principale du modèle
├── meta.json       ← Métadonnées du modèle
├── fonts/          ← Fichiers de polices intégrés
└── images/         ← Images intégrées
```

---

## Modèle de rendu

```
1. Charger template.json
2. Lier la source de données aux sections
3. Évaluer les clés de groupe → déterminer la répétition des sections
4. Calculer la position des éléments (moteur de mise en page)
5. Produire le modèle de dessin (JSON)
6. Effectuer le rendu avec Google Skia (moteur de dessin)
7. Générer la sortie au format cible
```

Aucun pilote d'imprimante n'est nécessaire, à aucune étape.

---

## Langages d'implémentation

ACR peut être implémenté dans tout langage disposant de liaisons Skia ou d'une bibliothèque graphique 2D compatible.

| Langage | État |
|---------|------|
| Rust | Implémentation de référence ([acr-engine](https://github.com/acrossreport/acr-engine)) |
| C++ | Prévu |
| C# | Prévu |
| WebAssembly | Prévu |

---

## Comparaison

| Fonctionnalité | Outils de rapports traditionnels | ACR |
|----------------|----------------------------------|-----|
| Modèle de sections | ✓ | ✓ |
| Modèle JSON | — | ✓ |
| Indépendance vis-à-vis des imprimantes | — | ✓ |
| Garantie WYSIWYG | partielle | ✓ |
| Aperçu sans matériel | — | ✓ |

---

## Historique des versions

| Version | Date | Description |
|---------|------|-------------|
| 1.0 | 2026-02-26 | Version initiale |

---

*Spécification ACR — acrossreport/acr-spec*
