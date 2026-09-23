# ADR-001 : ACR (Across Report Renderer) — Architecture Decision Record

**Statut :** Accepté  
**Créé le :** 2026-05-24  
**Auteur :** Hisanobu Araki (MaskedRiderSystem)  
**Version :** 0.x (en cours)

---

## Table des matières

1. [Contexte et problématique](#1-contexte-et-problématique)
2. [Index des décisions](#2-index-des-décisions)
3. [ADR-001 : Adopter Google Skia comme moteur de rendu](#adr-001--adopter-google-skia-comme-moteur-de-rendu)
4. [ADR-002 : Adopter JSON comme modèle de dessin intermédiaire des modèles](#adr-002--adopter-json-comme-modèle-de-dessin-intermédiaire-des-modèles)
5. [ADR-003 : Adopter une structure de rapport par sections](#adr-003--adopter-une-structure-de-rapport-par-sections)
6. [ADR-004 : Faire de la précision WYSIWYG au pixel près le principe de conception prioritaire](#adr-004--faire-de-la-précision-wysiwyg-au-pixel-près-le-principe-de-conception-prioritaire)
7. [ADR-005 : Adopter la philosophie de l'aperçu sans matériel](#adr-005--adopter-la-philosophie-de-laperçu-sans-matériel)
8. [Historique des versions](#historique-des-versions)

---

## 1. Contexte et problématique

Les environnements de production de rapports souffrent de problèmes chroniques et systémiques :

- **Variations de sortie dépendantes de l'environnement :** les écarts au pixel près dus aux différences de système d'exploitation, de pilote d'imprimante et de moteur de rendu des polices rendent presque impossible la garantie de la qualité des rapports.
- **Limites des moteurs commerciaux :** les moteurs de rapports du marché sont des boîtes noires qui ne peuvent pas répondre aux exigences opérationnelles propres à chaque site, comme les structures de sections complexes et les mises en page dynamiques.
- **Écart entre l'aperçu et l'impression :** même les produits présentés comme WYSIWYG produisent souvent des impressions différentes de l'aperçu à l'écran.
- **Licences et maintenabilité :** la dépendance à un éditeur particulier entraîne des coûts de licence permanents et une dette technique.

Pour résoudre ces problèmes, le moteur de rendu de rapports propriétaire **ACR (Across Report Renderer)** est conçu et développé.

---

## 2. Index des décisions

| ADR | Titre | Statut |
|-----|-------|--------|
| ADR-001 | Adopter Google Skia comme moteur de rendu | Accepté |
| ADR-002 | Adopter JSON comme modèle de dessin intermédiaire des modèles | Accepté |
| ADR-003 | Adopter une structure de rapport par sections | Accepté |
| ADR-004 | Faire de la précision WYSIWYG au pixel près le principe de conception prioritaire | Accepté |
| ADR-005 | Adopter la philosophie de l'aperçu sans matériel | Accepté |

---

## ADR-001 : Adopter Google Skia comme moteur de rendu

### Contexte

Un moteur graphique doit être choisi pour produire la sortie finale des rapports (PDF et aperçu à l'écran). Les candidats suivants ont été évalués :

- **GDI/GDI+ :** Windows uniquement. Problèmes de précision au pixel connus.
- **Cairo :** multiplateforme, mais la qualité du rendu du texte est instable en raison de la dépendance au moteur de polices sous-jacent.
- **Skia :** la base de rendu de Google Chrome, d'Android et de Flutter. Haute précision au pixel ; prend en charge la sortie PDF directe.
- **PDFium :** conçu uniquement pour la génération de PDF. Inadapté aux aperçus interactifs.

### Décision

**Adopter Google Skia comme unique moteur de rendu d'ACR.**

### Justification

1. **Précision au pixel constante :** Skia utilise le même chemin de code pour l'affichage à l'écran et la sortie PDF, ce qui garantit en théorie une correspondance parfaite entre l'aperçu et l'impression.
2. **Neutralité vis-à-vis de la plateforme :** des résultats de rendu identiques sont obtenus sous Windows, macOS et Linux.
3. **Fiabilité éprouvée :** Skia équipe le moteur de navigateur le plus utilisé au monde (Chrome), ce qui démontre une qualité et une stabilité suffisantes.
4. **Génération directe de PDF :** `SkDocument` permet une sortie PDF native, sans couche de conversion intermédiaire.
5. **Rendu des polices :** l'intégration avec FreeType, DirectWrite et CoreText permet d'utiliser avec précision les moteurs de polices natifs du système.

### Compromis

| Avantages | Inconvénients |
|-----------|---------------|
| Correspondance au pixel de haute précision | Nécessite de compiler et de distribuer une bibliothèque native |
| Chemin de code unifié pour le PDF et l'écran | Courbe d'apprentissage de l'API Skia |
| Garantie multiplateforme | Gestion des versions liée au cycle de publication de Chromium |

---

## ADR-002 : Adopter JSON comme modèle de dessin intermédiaire des modèles

### Contexte

Plusieurs formats ont été envisagés pour la définition des modèles de rapports : XML, binaire propriétaire et JSON. Dans un moteur de rapports, le modèle est l'élément central qui relie « l'intention exprimée » et « les instructions de dessin ».

### Décision

**Adopter JSON comme unique format de modèle de dessin intermédiaire pour les modèles ACR.**

JSON n'est pas traité comme un simple format de données. Dans ACR, il est positionné comme un **langage intermédiaire exprimant l'intention de dessin**.

### Justification

1. **Lisibilité humaine :** moins verbeux que XML, ce qui facilite considérablement le développement et le débogage des modèles.
2. **Définition de schéma typée :** une validation stricte via JSON Schema est simple à mettre en œuvre.
3. **Affinité avec les outils :** intégration facile avec les éditeurs visuels, les outils de comparaison et les pipelines CI/CD.
4. **Responsabilité claire en tant que modèle intermédiaire :** le modèle JSON définit *ce qu'il faut dessiner* ; la couche Skia définit *comment le dessiner*. La séparation des responsabilités est explicite.
5. **Génération dynamique :** la liaison de données et la logique conditionnelle s'expriment naturellement dans la structure JSON.

### Structure conceptuelle d'un modèle

L'exemple ci-dessous suit le format défini dans [specification.fr.md](specification.fr.md) (A4 à 300 DPI, en points).

```json
{
  "version": "1.0",
  "page": { "width": 2480, "height": 3508, "unit": "dot", "dpi": 300 },
  "sections": [
    {
      "type": "ReportHeader",
      "height": 236,
      "elements": [
        {
          "type": "TextBox",
          "x": 118, "y": 59, "width": 2244, "height": 118,
          "text": "{{ reportTitle }}",
          "font": { "family": "Arial", "size": 14, "bold": true },
          "alignment": "center"
        }
      ]
    }
  ]
}
```

### Compromis

| Avantages | Inconvénients |
|-----------|---------------|
| Lisible, facile à versionner | Plus verbeux que le binaire ; fichiers plus volumineux |
| Validation par schéma possible | Coût de maintenance continu du JSON Schema |
| Intégration facile aux outils | Limites pour exprimer des expressions ou scripts complexes |

---

## ADR-003 : Adopter une structure de rapport par sections

### Contexte

La question était de savoir s'il fallait utiliser la structure par sections largement adoptée dans les rapports d'entreprise (ReportHeader / PageHeader / GroupHeader / Detail / GroupFooter / PageFooter / ReportFooter) comme conception de référence du modèle de mise en page d'ACR.

### Décision

**Adopter la structure de rapport par sections comme modèle de mise en page standard d'ACR.**

Cela n'empêche pas les extensions propres à ACR ; la structure par sections est le *point de départ de la conception*, et non une contrainte stricte.

### Justification

1. **Valoriser les connaissances existantes :** de nombreux ingénieurs de terrain connaissent déjà le modèle par sections, ce qui réduit au minimum les coûts d'apprentissage.
2. **Réduction des coûts de migration :** les définitions de rapports existantes basées sur des sections peuvent être migrées progressivement vers ACR.
3. **Conception éprouvée :** la structure par sections a été validée par des décennies de traitement de rapports d'entreprise et reflète à un haut niveau les besoins opérationnels.
4. **Expressivité pour l'agrégation par groupe et le contrôle des sauts de page :** le modèle par sections représente naturellement les rapports d'agrégation complexes et les sauts de page conditionnels.

### Définition des sections

| Section | Rôle |
|---------|------|
| `ReportHeader` | Sortie une seule fois au début du rapport |
| `PageHeader` | Sortie en haut de chaque page |
| `GroupHeader` | Sortie au début de chaque groupe (imbricable) |
| `Detail` | Sortie répétée pour chaque enregistrement |
| `GroupFooter` | Sortie à la fin de chaque groupe (imbricable) |
| `PageFooter` | Sortie en bas de chaque page |
| `ReportFooter` | Sortie une seule fois à la fin du rapport |

### Compromis

| Avantages | Inconvénients |
|-----------|---------------|
| Aucun apprentissage supplémentaire pour les ingénieurs de terrain | Coût du maintien de la compatibilité avec les définitions de rapports existantes |
| Migration facile des rapports existants | Certains choix de conception propres à ACR peuvent être contraints |
| Hérite d'une philosophie de conception éprouvée | Risque d'hériter des limites des conceptions par sections traditionnelles |

---

## ADR-004 : Faire de la précision WYSIWYG au pixel près le principe de conception prioritaire

### Contexte

La question était de savoir s'il fallait placer « la précision de la correspondance entre l'aperçu à l'écran et l'impression » au sommet de la hiérarchie des principes de conception — au-dessus des performances, de la vitesse de développement et de la compatibilité.

### Décision

**ACR adopte « un WYSIWYG parfait au pixel près — sans un seul point d'écart » comme principe de conception prioritaire et inviolable.**

Ce principe prime sur les performances, la vitesse de développement et la compatibilité. Il ne peut faire l'objet d'aucun compromis.

### Justification

1. **Responsabilité sociale des rapports :** pour les documents ayant une valeur juridique (factures, bons de livraison, bulletins de paie), tout écart entre l'aperçu et l'impression constitue un risque commercial et juridique.
2. **Confiance des opérateurs :** la certitude que « l'impression correspondra exactement à ce que je vois dans l'aperçu » est le fondement de la productivité des opérateurs.
3. **Réduction des coûts de débogage :** la garantie de correspondance entre l'aperçu et l'impression réduit considérablement le coût d'investigation des bogues liés aux différences d'environnement.

### Contraintes d'implémentation

- Tous les calculs de coordonnées, le placement du texte et la logique de saut de page doivent partager le même chemin d'API Skia pour l'affichage à l'écran et la génération de PDF.
- Les métriques des polices (largeur, hauteur et crénage des glyphes) sont mises en cache à l'exécution, et les mêmes valeurs sont utilisées pour l'aperçu et la sortie PDF.
- Les valeurs fractionnaires inférieures au pixel doivent toujours suivre la même règle d'arrondi (floor/ceil) de manière cohérente.

---

## ADR-005 : Adopter la philosophie de l'aperçu sans matériel

### Contexte

La question était de savoir s'il fallait faire de « la capacité d'aperçu complet sans accès à des imprimantes physiques ou à des périphériques de sortie » une ligne directrice centrale du cycle de développement et de vérification.

### Décision

**ACR place la « philosophie sans matériel » au cœur de sa conception.**

ACR garantit que les développeurs et le personnel d'exploitation peuvent vérifier des aperçus précis sans imprimante, sans terminal de rapports ni papier spécial.

### Justification

1. **Efficacité du développement :** les cycles de création, de modification et de vérification des modèles peuvent être accélérés sans mettre en place de matériel physique.
2. **Intégration CI/CD :** la génération et la comparaison d'images d'aperçu dans des pipelines de tests automatisés permettent des tests de régression de la sortie des rapports.
3. **Suppression des contraintes géographiques et financières :** une sortie précise peut être vérifiée à distance, même lorsque les sites de développement et les équipements d'impression sont éloignés.
4. **Différenciateur technique clé (brevet en cours) :** « un aperçu parfait au pixel près sans matériel physique » est l'avantage technique central d'ACR. Le modèle de dessin intermédiaire qui le rend possible fait l'objet d'une demande de brevet.

### Exigences techniques de mise en œuvre

- Rendu hors écran via `SkSurface` de Skia pour générer des aperçus PNG/JPEG.
- Simulation logicielle du format de papier, des marges et de la résolution (DPI).
- Rendu du texte indépendant de l'environnement grâce à l'intégration des polices.
- Maintien d'une pile purement logicielle ne nécessitant aucun pilote d'imprimante virtuelle.

---

## Historique des versions

| Version | Date | Description | Auteur |
|---------|------|-------------|--------|
| 0.1 | 2026-05-24 | Version initiale | MaskedRiderSystem |
| 0.2 | 2026-09-23 | ADR-003 révisé en structure par sections ; exemple de l'ADR-002 aligné sur specification.md ; formulation de l'ADR-005 mise à jour (brevet en cours) | MaskedRiderSystem |

---

*Ce document est mis à jour en continu au fil de l'avancement du projet ACR.*  
*Une fois acceptée, le statut d'une ADR ne change que si elle est abandonnée ou remplacée.*
