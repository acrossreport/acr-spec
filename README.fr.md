# Spécification ACR

**ACR (Across Report)** est une spécification de rapports indépendante des imprimantes, conçue pour les systèmes logiciels modernes.

> *Render once. Output everywhere.*

ACR sépare la description de la mise en page, le rendu et la génération de la sortie —
ce qui permet au même modèle d'être rendu sur plusieurs plateformes et formats de sortie.

---

## Pourquoi ACR

Les systèmes de reporting traditionnels sont étroitement liés à des imprimantes, moteurs
de rendu ou frameworks spécifiques.

ACR introduit une architecture en couches claire :

```
Modèle → Moteur de mise en page → Moteur de rendu → Sortie
```

Chaque couche a une responsabilité unique :

| Couche              | Rôle                                                                 |
|--------------------|----------------------------------------------------------------------|
| **Modèle**         | Définition de rapport basée sur JSON. Neutre vis-à-vis de la plateforme. |
| **Moteur de mise en page** | Résout les sections, lie les données, calcule les positions.  |
| **Moteur de rendu** | Rendu en pixels via Google Skia. Précision 1 point.                 |
| **Sortie**          | PDF, PNG, ou tout format cible.                                     |

Cette séparation permet :

- Le même modèle s'exécute sur Windows, macOS et Linux sans modification.
- L'aperçu dans le designer est **identique au pixel près** au résultat imprimé final (WYSIWYG).
- Le format de sortie est un détail — pas une contrainte.

---

## Principes de conception clés

**Indépendance vis-à-vis de l'imprimante**
ACR ne dépend ni des pilotes d'imprimante ni des sous-systèmes d'impression du système
d'exploitation. La mise en page est calculée dans des unités indépendantes du
périphérique, puis rendue à la résolution cible.

**JSON comme modèle intermédiaire**
Les modèles sont définis en JSON lisible par l'humain. Le moteur de mise en page produit
également du JSON, inspectable, cacheable ou transmissible indépendamment du rendu.

**Rendu basé sur Skia**
Le moteur de rendu utilise Google Skia, la même bibliothèque graphique que celle
utilisée par Chrome et Android. Cela garantit une sortie cohérente et de haute fidélité
sur toutes les plateformes.

**Modèle de rapport basé sur des sections**
ACR adopte une structure familière par sections (en-tête de rapport / en-tête de page /
en-tête de groupe / détail / pied de groupe / pied de page / pied de rapport), largement
utilisée dans le reporting métier, minimisant l'effort de migration des définitions de
rapports existantes.

**Aperçu sans matériel**
Un aperçu complet et précis au pixel près est disponible sans imprimante, pilote ou
matériel spécialisé physique. Ce que vous voyez est exactement ce qui sera imprimé.

---

## Structure du dépôt

```
acr-spec/
├── README.md                  ← Version anglaise
├── README.ja.md               ← 日本語版
├── README.fr.md               ← Ce fichier
├── specification.md           ← Spécification complète (anglais)
├── specification.ja.md        ← 仕様（日本語）
├── specification.fr.md        ← Spécification complète (français)
├── design-decisions.md        ← Architecture Decision Records (anglais)
├── design-decisions.ja.md     ← アーキテクチャ決定記録（日本語）
└── design-decisions.fr.md     ← Historique des décisions d'architecture (français)
```

---

## Historique des décisions d'architecture (ADR)

Les décisions de conception sont documentées sous forme d'ADR dans
[design-decisions.fr.md](design-decisions.fr.md).

| ADR | Titre | Statut |
|-----|-------|--------|
| ADR-001 | Adoption de Google Skia | Accepté |
| ADR-002 | Adoption de JSON comme modèle de dessin intermédiaire pour les modèles | Accepté |
| ADR-003 | Adoption d'une structure de rapport basée sur des sections | Accepté |
| ADR-004 | La précision WYSIWYG comme principe de conception prioritaire | Accepté |
| ADR-005 | Adoption d'une philosophie d'aperçu sans matériel | Accepté |

English: [design-decisions.md](design-decisions.md) / 日本語: [design-decisions.ja.md](design-decisions.ja.md)

---

## Documentation

- [Spécification](specification.fr.md) — Spécification complète des modèles et du rendu ACR
- [Architecture Decision Records](design-decisions.fr.md) — Justification des choix de conception d'ACR

---

## État du projet

ACR est en développement actif. La spécification est en cours de stabilisation.
Voir [acrossreport.com](https://acrossreport.com) pour les produits basés sur cette spécification.

---

## Licence

Voir [LICENSE](LICENSE) pour plus de détails.
