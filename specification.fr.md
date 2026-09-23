# Spécification ACR

**ACR (Across Report)** est une spécification de rapports indépendante des imprimantes, conçue pour les systèmes logiciels modernes.

> *Render once. Output everywhere.*

ACR sépare la description de la mise en page, le rendu et la génération de la sortie —  
ce qui permet de rendre un même modèle sur plusieurs plateformes et dans plusieurs formats de sortie.

---

## Pourquoi ACR

Les systèmes de rapports traditionnels sont étroitement liés à des imprimantes, des moteurs de rendu ou des frameworks spécifiques.

ACR introduit une architecture claire, organisée en couches :

```
Modèle → Moteur de mise en page → Moteur de dessin → Sortie
```

Chaque couche a une seule responsabilité :

| Couche                        | Rôle                                                                        |
|-------------------------------|-----------------------------------------------------------------------------|
| **Modèle**                    | Définition du rapport en JSON. Indépendante de la plateforme.               |
| **Moteur de mise en page**    | Résout les sections, lie les données, calcule les positions.                |
| **Moteur de dessin**          | Effectue le rendu en pixels avec Google Skia. Précision au point près.      |
| **Sortie**                    | PDF, PNG, SVG ou tout autre format cible.                                   |

Cette séparation signifie que :

- Le même modèle fonctionne sous Windows, macOS et Linux sans modification.
- L'aperçu dans le concepteur est **identique au pixel près** à la sortie imprimée finale (WYSIWYG).
- Le format de sortie est un détail — pas une contrainte.

---

## Principes de conception

**Indépendance vis-à-vis des imprimantes**  
ACR ne dépend ni des pilotes d'imprimante ni des sous-systèmes d'impression du système d'exploitation. La mise en page est calculée en unités indépendantes du périphérique, puis rendue à la résolution cible.

**JSON comme modèle intermédiaire**  
Les modèles sont définis en JSON lisible par l'humain. Le moteur de mise en page produit un modèle de dessin — lui aussi en JSON — qui peut être inspecté, mis en cache ou transmis indépendamment du rendu.

**Rendu basé sur Skia**  
Le moteur de dessin utilise Google Skia, la bibliothèque graphique utilisée par Chrome et Android. Cela garantit une sortie cohérente et fidèle sur toutes les plateformes.

**Modèle de rapport par sections**  
ACR adopte une structure par sections bien connue (En-tête de rapport / En-tête de page / En-tête de groupe / Détail / Pied de groupe / Pied de page / Pied de rapport), largement utilisée dans les rapports d'entreprise, ce qui réduit au minimum l'effort de migration des définitions de rapports existantes.

**Aperçu sans matériel**  
Un aperçu complet et précis au pixel près est disponible sans imprimante physique, sans pilote ni matériel spécialisé. Ce que vous voyez est exactement ce qui sera imprimé.

---

## Structure du dépôt

```
acr-spec/
├── README.md                  ← Version anglaise
├── README.ja.md               ← Version japonaise
├── README.fr.md               ← Ce fichier
└── specification.md           ← Spécification complète d'ACR
```

---

## Documentation

- [Spécification](specification.md) — Spécification complète des modèles et du rendu ACR

---

## État

ACR est en cours de développement actif. La spécification est en voie de stabilisation.  
Consultez [acrossreport.com](https://acrossreport.com) pour découvrir les produits basés sur cette spécification.

---

## Licence

Voir [LICENSE](LICENSE) pour plus de détails.
