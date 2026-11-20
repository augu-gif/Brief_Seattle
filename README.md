# Brief Seattle — Consommation d'énergie des bâtiments

Projet réalisé dans le cadre de la formation Data Engineer (Simplon).

## Objectif

À partir des relevés énergétiques de la ville de Seattle pour 2016, l'objectif est de prédire, pour les bâtiments **non résidentiels** :

- leur **consommation d'énergie** (`SiteEnergyUse(kBtu)`) ;
- leurs **émissions de CO₂** (`TotalGHGEmissions`) ;

puis d'évaluer l'intérêt de l'**ENERGY STAR Score** pour cette prédiction.

Ce dépôt couvre la préparation des données : l'**analyse exploratoire** et le **nettoyage**.

## Les données

Données ouvertes de la ville de Seattle (*2016 Building Energy Benchmarking*) : **3 376 bâtiments**, **46 colonnes** (localisation, surfaces, usages, année de construction, consommations mesurées, statut de conformité…).

## Partie 1 — Analyse exploratoire

Cette partie observe les données **sans les modifier** et aboutit à une liste de décisions pour le nettoyage.

| Constat | Détail |
|---|---|
| Périmètre | 1 711 bâtiments sont résidentiels (`Multifamily`), hors sujet. Ils consomment nettement moins au pied² (32,0 kBtu contre 51,9) |
| Valeurs manquantes | `Comments` vide à 100 %. Les 2ᵉ et 3ᵉ usages manquent souvent, mais c'est normal : beaucoup de bâtiments n'ont qu'un usage. L'ENERGY STAR Score manque pour 25 % du fichier |
| Lignes signalées par la ville | 165 bâtiments non conformes, 32 marqués `Outlier` |
| Erreurs de saisie | 19 libellés pour 13 quartiers réels (casse différente), 16 bâtiments non résidentiels à 0 étage, 21 consommations ou émissions nulles ou négatives |
| Forme des cibles | Très asymétrique : médiane de 2,6 millions de kBtu, moyenne de 8,5 millions, maximum de 874 millions. Un passage au logarithme sera nécessaire |
| **Fuite de données** | Électricité + gaz + vapeur = consommation totale (rapport médian de 1,000). Ces colonnes et leurs dérivées contiennent la réponse et doivent être retirées |
| Facteurs de consommation | La surface et le type d'usage sont les deux grands facteurs (un hôpital consomme environ 150 fois plus qu'un garde-meubles) |
| ENERGY STAR Score | Disponible pour 1 090 des 1 645 bâtiments non résidentiels. Il est calculé à partir de la consommation mesurée, donc il contient en partie la réponse |

## Partie 2 — Nettoyage

Les décisions de la partie 1 sont appliquées dans l'ordre, sur une copie des données brutes.

### Filtrage des lignes

| Étape | Bâtiments restants |
|---|---|
| Données brutes | 3 376 |
| Retrait des bâtiments résidentiels | 1 665 |
| Retrait des lignes non conformes ou signalées `Outlier` | 1 546 |
| Retrait des cibles manquantes, nulles ou négatives | **1 544** |

### Corrections

- **Quartiers** : mise en majuscules et fusion des variantes (19 libellés → 13 quartiers).
- **Nombre de bâtiments** : 0 remplacé par 1 (52 sites ; 93,3 % des sites ont exactement un bâtiment).
- **Nombre d'étages** : les 16 valeurs à 0 sont remplacées par la **médiane des étages du même type de bâtiment**. La médiane plutôt que la moyenne, pour ne pas être influencé par les gratte-ciel.

### Nouvelles variables (*feature engineering*)

| Variable | Calcul |
|---|---|
| `Age` | 2016 − année de construction |
| `NbUsages` | Nombre d'usages déclarés |
| `PartParking` | Surface de parking ÷ surface totale |
| `PartUsagePrincipal` | Surface du 1ᵉʳ usage ÷ surface totale, plafonnée à 1 (180 incohérences corrigées) |
| `LogSurface` | Logarithme de la surface totale |

### Colonnes retirées

- **11 colonnes de fuite** : électricité, gaz, vapeur, intensités énergétiques, versions corrigées de la météo, intensité de CO₂.
- **20 colonnes inutiles** : identifiants administratifs, adresse, colonnes vides, colonnes brutes remplacées par une nouvelle variable.
- **3 surfaces brutes redondantes** : surface totale, surface des bâtiments et surface du 1ᵉʳ usage disent presque la même chose (les deux premières sont identiques pour 78,7 % des bâtiments). La taille est portée par `LogSurface`, la composition par les proportions.

### Ce qui n'est volontairement pas fait ici

La **standardisation** et l'**encodage** des catégories apprennent des paramètres à partir des données. Les faire avant la séparation entraînement / test ferait fuiter le jeu de test dans l'apprentissage. Ces étapes sont donc laissées à la phase de modélisation, dans un `Pipeline` scikit-learn.

## Résultat

Un jeu de données propre de **1 544 bâtiments** avec **12 variables explicatives** :

- 9 numériques : `LogSurface`, `PartParking`, `PartUsagePrincipal`, `NumberofBuildings`, `NumberofFloors`, `Age`, `NbUsages`, `Latitude`, `Longitude` ;
- 3 catégorielles : `PrimaryPropertyType`, `Neighborhood`, `BuildingType` ;

ainsi que les deux cibles et l'ENERGY STAR Score (vide pour 550 bâtiments, conservé pour être étudié à part).

## Outils

Python 3.12, pandas, NumPy, Matplotlib, seaborn, Jupyter.
