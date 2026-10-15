# Spécification des modèles ACR

## Vue d'ensemble

**ACR (Across Report)** est un format de modèle de rapport basé sur JSON, indépendant
de l'imprimante.

ACR sépare la définition de la mise en page, le rendu et la génération de la sortie en
couches distinctes. Le même modèle produit une sortie identique sur n'importe quelle
plateforme et vers n'importe quel format de sortie.

> *Render once. Output everywhere.*

Ce document décrit le format de modèle tel qu'il est réellement lu par le moteur ACR
(`acr_core`) et ses clients (`acr-viewer-vscode`, `acr-viewer`, `acr-cli`). Il remplace
les versions précédentes de cette spécification, qui décrivaient un schéma différent et
jamais implémenté (`x`/`y`/`width`/`height` en minuscules, contrôles `TextBox`, liaison
de type moustache `{{ field }}`). Les modèles produits par l'outil de conversion
`RPX2ACR`, ainsi que tous les modèles d'exemple actuels, suivent le schéma ci-dessous.

### Formats de sortie confirmés

| Cible | Description |
|--------|-------------|
| PDF | Sortie document |
| PNG | Image matricielle de précision pixel (un fichier par page, pouvant être regroupé en ZIP) |

D'autres formats de sortie (SVG, ESC/POS, StarPRNT, SATO, TEC) ont été évoqués comme
orientation future, mais leur implémentation dans le moteur actuel n'est pas confirmée.
Ne vous basez pas sur ce document pour leur format au niveau des champs tant qu'ils ne
sont pas livrés.

---

## Architecture

```
Modèle (JSON)
    ↓
Moteur de mise en page   — résout les sections, lie les données, calcule les positions
    ↓
Moteur de rendu          — rendu via Google Skia (précision 1 point)
    ↓
Sortie                   — PDF / PNG
```

### Principes de conception

**Indépendance vis-à-vis de l'imprimante**
ACR ne dépend ni des pilotes d'imprimante ni des sous-systèmes d'impression du système
d'exploitation. La mise en page est calculée dans des unités indépendantes du
périphérique, puis rendue à la résolution cible.

**Garantie WYSIWYG**
L'aperçu à l'écran est identique au pixel près au résultat imprimé final.

**Aperçu sans matériel**
Un aperçu complet et précis au pixel près est disponible sans imprimante ni pilote
physique.

**JSON comme format de modèle**
Les modèles sont du JSON simple, inspectable et versionnable indépendamment du rendu.

**Rendu basé sur Skia**
Le moteur de rendu utilise Google Skia — la même bibliothèque graphique que celle
utilisée par Chrome et Android.

**Modèle de rapport basé sur des sections**
ACR adopte une structure de mise en page par sections, largement utilisée dans les
outils de reporting métier traditionnels, afin de minimiser le coût de migration des
définitions de rapports existantes (voir [design-decisions.fr.md](design-decisions.fr.md),
ADR-003).

---

## Système de coordonnées

- **Unité :** twip (1/1440 de pouce ; 1 pouce = 1440 twips)
- **Origine :** coin supérieur gauche de la page / section
- **Axe X :** croissant vers la droite
- **Axe Y :** croissant vers le bas
- **Positionnement :** les coordonnées des contrôles (`Left`/`Top`) sont relatives au
  coin supérieur gauche de la section qui les contient, et non de la page
- Les contrôles de type ligne utilisent en plus `X1`/`Y1`/`X2`/`Y2`, également en twips,
  relatifs à la section

---

## Structure du modèle

Un modèle est un unique fichier JSON avec la structure racine suivante.

```json
{
  "ACR": {
    "Meta": { ... },
    "Report": {
      "PageSettings": { ... },
      "Sections": { "Section": [ ... ] }
    }
  }
}
```

La clé racine historique `AcReport` est également acceptée à la place de `ACR.Report`
(les deux pointent vers le même objet `Report`).

| Champ | Type | Obligatoire | Description |
|-------|------|----------|-------------|
| `ACR.Meta` | object | | Métadonnées du modèle (voir ci-dessous). Non utilisé par les moteurs de mise en page / rendu. |
| `ACR.Report.PageSettings` | object | ✓ | Taille, orientation et marges de la page |
| `ACR.Report.Sections.Section` | array | ✓ | Liste ordonnée des sections du rapport |

### Objet Meta

Informatif uniquement ; repris de l'étape de conversion `RPX2ACR` et non interprété par
le moteur de rendu.

```json
{
  "Author": "RPX2ACR",
  "CreatedAt": "2026-03-26 15:47:38",
  "LastEditor": "",
  "Source": "RPX",
  "UpdatedAt": "2026-03-26 15:47:38",
  "Version": "1.0"
}
```

---

## Objet PageSettings

Définit la taille physique de la page, son orientation et ses marges, toutes en twips.

```json
{
  "PaperName": "ユーザー定義のサイズ",
  "PaperSize": 9,
  "PaperWidth": 11906,
  "PaperHeight": 16838,
  "Orientation": 2,
  "TopMargin": 850,
  "BottomMargin": 397,
  "LeftMargin": 680,
  "RightMargin": 567
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `PaperWidth` | number | Largeur du papier en twips (orientation portrait) |
| `PaperHeight` | number | Hauteur du papier en twips (orientation portrait) |
| `PaperSize` | number | Code de taille de papier Windows `DMPAPER_*` |
| `PaperName` | string | Nom affiché de la taille de papier |
| `Orientation` | number | `1` = portrait, `2` = paysage. Lorsque `2`, la largeur/hauteur effective du contenu est `PaperHeight`×`PaperWidth` (inversée) |
| `TopMargin` / `BottomMargin` / `LeftMargin` / `RightMargin` | number | Marges de page en twips |

**Remarque :** contrairement aux dimensions du papier, les marges sont chacune un champ
de premier niveau distinct — il n'y a pas d'objet `margin` imbriqué.

---

## Modèle de section

Les sections sont listées dans `ACR.Report.Sections.Section` (un tableau) et rendues sur
la page dans cet ordre. Le moteur accepte également un tableau nu sous `Sections`, ou une
autre forme pour `Section`/`Controls` — voir « Tolérance de schéma » ci-dessous.

### Types de section confirmés

| Valeur de `Type` | Rendu | Description |
|--------------|----------|-------------|
| `ACR.PageHeader` | Haut de chaque page | Titres de colonnes, titre de page |
| `ACR.GroupHeader` | Début de chaque groupe de données (imbricable) | Libellé de groupe, en-tête de sous-total |
| `ACR.Detail` | Une fois par enregistrement | Contenu principal des lignes |
| `ACR.GroupFooter` | Fin de chaque groupe de données (imbricable) | Sous-totaux de groupe |
| `ACR.PageFooter` | Bas de chaque page | Numéros de page, date |

`ACR.ReportHeader` et `ACR.ReportFooter` sont décrits de manière conceptuelle dans
[design-decisions.fr.md](design-decisions.fr.md) (ADR-003) comme des sections rendues
une seule fois par rapport, mais aucun modèle d'exemple les utilisant n'a été confirmé à
ce jour — considérez leur format au niveau des champs comme non confirmé tant qu'il n'a
pas été vérifié dans le moteur.

### Champs de section

```json
{
  "Name": "groupheader1",
  "Type": "ACR.GroupHeader",
  "Height": 850,
  "BackColor": 16777215,
  "DataField": "TENPOCD",
  "GroupKeepTogether": 1,
  "KeepTogether": 1,
  "NewPage": 1,
  "RepeatStyle": 4,
  "Control": [ ... ],
  "_tag": "Section"
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `Name` | string | Nom de la section, également utilisé comme libellé d'affichage dans l'aperçu VS Code |
| `Type` | string | Type de section (voir tableau ci-dessus) |
| `Height` | number | Hauteur de la section en twips |
| `BackColor` | number | Couleur de fond, voir « Format des couleurs » ci-dessous |
| `DataField` | string | Champ clé de regroupement (`ACR.GroupHeader` / `ACR.GroupFooter` uniquement) |
| `GroupKeepTogether`, `KeepTogether`, `NewPage`, `RepeatStyle` | number | Indicateurs de contrôle de groupe/saut de page hérités de l'outil de rapport d'origine. La sémantique exacte de ces valeurs n'est pas encore entièrement documentée ici — se référer aux valeurs des exemples existants. |
| `rowsPerPage` | number | `ACR.Detail` uniquement ; `0` signifie illimité |
| `Control` | array | Contrôles de la section (voir ci-dessous) |
| `_tag` | string | Toujours `"Section"` ; champ discriminant hérité, peut être conservé tel quel |

### Imbrication de groupes

Les sections `ACR.GroupHeader` / `ACR.GroupFooter` peuvent être appariées et imbriquées
pour représenter un regroupement multi-niveaux, en suivant l'ordre habituel des
sections :

```
ACR.PageHeader
ACR.GroupHeader   (groupe externe)
ACR.GroupHeader   (groupe interne)
ACR.Detail
ACR.GroupFooter   (groupe interne)
ACR.GroupFooter   (groupe externe)
ACR.PageFooter
```

---

## Contrôles

Les contrôles sont les éléments dessinables au sein d'une section, listés dans le
tableau `Control` de la section.

### Tolérance de schéma

Le moteur accepte les contrôles sous l'une de ces trois formes (toutes équivalentes) :

```json
"Control": [ ... ]
"Controls": [ ... ]
"Controls": { "Control": [ ... ] }
```

### Champs communs

| Champ | Type | Obligatoire | Description |
|-------|------|----------|-------------|
| `Type` | string | ✓ | Type de contrôle, voir ci-dessous |
| `Name` | string | ✓ | Nom du contrôle. Affiché comme libellé de remplacement (`[Name]`) lorsqu'un `ACR.Field` n'a pas de `Text` d'exemple |
| `Left` | number | ✓ | Position X en twips, relative au coin supérieur gauche de la section (`X` également accepté) |
| `Top` | number | ✓ | Position Y en twips, relative au coin supérieur gauche de la section (`Y` également accepté) |
| `Width` | number | ✓ | Largeur en twips |
| `Height` | number | ✓ | Hauteur en twips |
| `Style` | string | | Liste de déclarations de type CSS, voir « Style » ci-dessous |
| `_tag` | string | | Toujours `"Control"` ; champ discriminant hérité |

### Types de contrôle confirmés

#### `ACR.Label`

Texte statique qui ne change pas avec la source de données (par ex. les en-têtes de
colonnes).

```json
{
  "Type": "ACR.Label",
  "Name": "lblcampany",
  "Text": "会社名：",
  "Style": "font-family: ＭＳ ゴシック; font-size: 9.75pt; font-weight: bold",
  "Left": 0, "Top": 28.34646, "Width": 1133.858, "Height": 226.7716
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `Text` | string / number | Le texte littéral à afficher |

#### `ACR.Field`

Texte lié aux données. La valeur affichée au moment du rendu provient de `DataField`,
résolu par rapport à l'enregistrement de données actif ; `Text` (s'il est présent) n'est
qu'une valeur d'exemple affichée au moment de la conception dans l'aperçu VS Code, et
n'est pas utilisée pour le rendu final.

```json
{
  "Type": "ACR.Field",
  "Name": "txtunitcost",
  "DataField": "UNITCOST",
  "OutputFormat": "¥#,##0",
  "CurrencyCulture": "ja-JP",
  "Multiline": 0,
  "Style": "font-family: MS UI Gothic; font-size: 8.25pt; text-align: right; vertical-align: middle",
  "Left": 10771.65, "Top": 28.34646, "Width": 1020.473, "Height": 255.118
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `DataField` | string | Nom du champ de la source de données auquel ce contrôle est lié |
| `Text` | string / number | Valeur d'exemple de conception uniquement (non rendue) |
| `OutputFormat` | string | Chaîne de format de style .NET appliquée à la valeur liée, par ex. `"yyyy/MM/dd"`, `"¥#,##0"`, `"#,##0.0"` |
| `CurrencyCulture` | string | Locale utilisée avec `OutputFormat` pour le format monétaire, par ex. `"ja-JP"` |
| `Multiline` | number | `0` = une seule ligne, `1` = retour à la ligne dans les limites du contrôle |
| `SummaryType`, `SummaryRunning` | number | Contrôles de valeur agrégée/en cours (par ex. numéro de page, nombre de pages). La sémantique exacte de ces valeurs n'est pas encore entièrement documentée ici — se référer aux valeurs des exemples existants. |

Les champs qui ne sont pas encore liés à un enregistrement de données (par ex. le numéro
de page avant le calcul de la mise en page) peuvent porter un `Text` littéral utilisé
comme valeur initiale/par défaut.

#### `ACR.Shape`

Un rectangle rempli et/ou avec contour, typiquement utilisé comme arrière-plan de
ligne/colonne.

```json
{
  "Type": "ACR.Shape",
  "Name": "shape2",
  "BackColor": 14474460,
  "BackStyle": 1,
  "LineColor": 16777215,
  "RoundingRadius": 9.999999,
  "Left": 0, "Top": 283.4646, "Width": 15590.55, "Height": 283.4646
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `BackColor` | number | Couleur de remplissage, voir « Format des couleurs » ci-dessous |
| `BackStyle` | number | `0` = transparent (pas de remplissage), `1` = rempli avec `BackColor` |
| `LineColor` | number | Couleur de la bordure |
| `LineStyle` | number | `1` = plein (défaut), `2` = tirets, `3` = pointillés |
| `RoundingRadius` | number | Rayon des coins en twips ; omettre ou `0` pour des coins carrés |

#### `ACR.Line`

Une ligne droite, positionnée indépendamment de `Left`/`Top`/`Width`/`Height`.

```json
{
  "Type": "ACR.Line",
  "Name": "line1",
  "X1": 0, "Y1": 1417.323,
  "X2": 15590.55, "Y2": 1417.323,
  "LineColor": 0,
  "LineWeight": 1,
  "LineStyle": 1
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `X1`, `Y1` | number | Point de départ en twips, relatif à la section |
| `X2`, `Y2` | number | Point d'arrivée en twips, relatif à la section |
| `LineColor` | number | Couleur de la ligne, voir « Format des couleurs » ci-dessous |
| `LineWeight` | number | Épaisseur de la ligne en twips (valeur effective minimale : 1) |
| `LineStyle` / `BorderStyle` | number | `1` = plein (défaut), `2` = tirets, `3` = pointillés |

### Types de contrôle référencés mais non encore confirmés

L'outillage de champs de données de l'extension VS Code (`list_data_fields`) classe les
champs liés comme `'Text' \| 'Barcode' \| 'Image' \| 'Group'`, ce qui suggère l'existence
de types de contrôle `ACR.Barcode` et `ACR.Image` (ou de noms similaires) dans le moteur.
Aucun modèle d'exemple en contenant n'était disponible au moment de la rédaction ; leur
forme JSON au niveau des champs n'est donc pas documentée ici. Ne présumez pas de leur
forme (par ex. les champs `data`/`symbology`/`src`/`sizing` de l'ancien brouillon de
spécification) sans l'avoir confirmée sur un modèle réel ou dans le code source du
moteur.

---

## Style

`Style` est une liste de déclarations `propriété: valeur` séparées par `;`, de forme
similaire à l'attribut `style` d'un CSS en ligne. Le nom de propriété est interprété sans
tenir compte de la casse. Seules les propriétés suivantes sont interprétées par les
moteurs de rendu/aperçu :

| Propriété | Valeurs | S'applique à |
|----------|--------|------------|
| `font-family` | nom de police | Label / Field |
| `font-size` | nombre + `pt` | Label / Field |
| `font-weight` | `normal` / `bold` | Label / Field |
| `font-style` | `normal` / `italic` | Label / Field |
| `text-align` | `left` / `center` / `right` | Label / Field |
| `vertical-align` | `top` / `middle` / `bottom` | Label / Field |

D'autres déclarations comme `ddo-char-set` peuvent apparaître (héritées de l'outil de
rapport d'origine) mais ne sont pas interprétées par le moteur.

Exemple : `"font-family: ＭＳ ゴシック; font-size: 9.75pt; font-weight: bold; text-align: center; vertical-align: middle"`

---

## Format des couleurs

Les couleurs (`BackColor`, `LineColor`) sont un entier unique encodant `0xRRGGBB` — par
ex. `16777215` = `0xFFFFFF` = blanc, `0` = `0x000000` = noir. Il s'agit d'un simple
entier RGB, **et non** de l'ordre d'octets OLE de Windows (`0x00BBGGRR`, ordre BGR).

---

## Liaison de données

Contrairement aux langages de modèle utilisant des expressions de liaison intégrées, ACR
n'intègre pas d'expression de liaison dans `Text`. À la place :

- Le `DataField` d'un contrôle `ACR.Field` nomme le champ de la source de données duquel
  provient la valeur rendue.
- Le `DataField` des sections `ACR.GroupHeader` / `ACR.GroupFooter` nomme le champ
  utilisé pour détecter les limites de groupe.
- Tout `Text` littéral sur un `ACR.Field` n'est qu'un espace réservé de conception.

La source de données elle-même (sa propre structure JSON, la manière dont elle est
fournie au moteur, et la correspondance entre `OutputFormat`/`CurrencyCulture` et les
chaînes de format) est hors du périmètre de ce document et devrait être documentée
séparément une fois confirmée dans le moteur.

---

## Modèle de rendu

```
1. Charger le JSON du modèle (ACR.Report, ou l'ancien AcReport)
2. Résoudre PageSettings (inverser largeur/hauteur lorsque Orientation == 2)
3. Lier les enregistrements de la source de données aux sections Detail / GroupHeader / GroupFooter
4. Résoudre les limites de groupe via le DataField de chaque section de groupe
5. Rendre la liste Control de chaque section via Google Skia (précision 1 point/twip)
6. Produire la sortie en PDF ou PNG
```

Aucun pilote d'imprimante n'est requis à aucune étape.

---

## Langages d'implémentation

| Langage | Statut |
|----------|--------|
| Rust | Implémentation de référence (`acr_core`, dépôt privé) |

---

## Historique des modifications

| Version | Date | Description |
|---------|------|-------------|
| 1.0 | 2026-02-26 | Version initiale (schéma de brouillon, non implémenté) |
| 2.0 | 2026-10-15 | Réécriture pour correspondre au schéma réellement produit par `RPX2ACR` et lu par `acr_core` / `acr-viewer-vscode` |

---

*ACR Specification — acrossreport/acr-spec*
