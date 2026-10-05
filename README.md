# Transport & Logistique — Analyse des retards de livraison

**Projet Data Science — Semaine 17 · Groupe 4**

Analyse de 45 593 livraisons réelles pour identifier les facteurs qui
allongent les délais de livraison.

---

## Le périmètre technique

Le cahier des charges limite les outils à :

> **pandas, statistiques et visualisation de données**

C'est pourquoi ce projet n'utilise **aucun outil de machine learning**.
Les analyses reposent sur des comparaisons de groupes, des corrélations et des
tests statistiques simples — tous utilisés à un niveau compréhensible.

---

## Installation

### Avec conda (recommandé)

```bash
conda env create -f environment.yml
conda activate transport-logistique
jupyter lab
```

### Avec VSCode

1. Installer l'extension **Jupyter** dans VSCode.
2. Ouvrir le dossier du projet.
3. `Ctrl+Shift+P` → **Python: Select Interpreter** → choisir l'environnement.
4. Ouvrir `notebooks/analyse_delais_livraison.ipynb`.
5. **Run All** pour exécuter le notebook entier.

### Sans conda

```bash
pip install -r requirements.txt
```

> Aucune configuration supplémentaire : le notebook trouve seul son fichier de
> données et crée ses dossiers s'ils n'existent pas.

---

## Structure du dépôt

```
transport-logistique-ds/
│
├── notebooks/
│   └── analyse_delais_livraison.ipynb   ← LE LIVRABLE PRINCIPAL
│
├── GUIDE_ANALYSE.md                      ← commentaire cellule par cellule
│
├── data/
│   └── raw/
│       └── train.csv                    ← fichier source (Kaggle, versionné)
│
├── figures/                             ← graphiques produits par le notebook
│
├── outputs/
│   ├── RAPPORT_MIA_FOODS.md              ← rapport pour le propriétaire (langage courant)
│   └── tables/                          ← tableaux de résultats (CSV)
│
├── requirements.txt                      ← installation par pip
├── environment.yml                       ← installation par conda
└── .gitignore
```

**Cette structure est volontairement minimaliste :** tout le travail est dans
le notebook. Il n'y a pas de dossier `src/`, pas de paquet à installer, pas de
fichier de configuration caché. Chaque membre du groupe peut donc lire,
comprendre et modifier le code sans difficulté.

### Les dossiers vides sont-ils un problème ?

Non. Le dossier `figures/` et `outputs/tables/` sont créés automatiquement par
le notebook à la première exécution. Les fichiers `.gitkeep` servent
seulement à ce que Git conserve les dossiers vides.

---

## Le plan des 16 tâches

Le notebook suit l'ordre du cahier des charges. Chaque section a la même forme :
**explication** → **code commenté** → **résultat affiché** (tableau ou graphique).

| Partie | Tâches | Contenu |
|---|---|---|
| **A — Préparer** | 1 à 4 | Lecture, nettoyage, variables, définition du retard |
| **B — Analyser** | 5 à 9 | Temps, Espace, Conditions, Véhicule, Coût |
| **C — Comparer** | 10 à 12 | Corrélations, comparaisons statistiques, tableau de bord |
| **D — Conclure** | 13 à 16 | Scénarios, règles d'interprétation, bilan |

Le notebook compte **66 cellules** : 26 de texte et 40 de code. Les explications
et les commentaires dans le code sont concis, techniques et sans superflu. Le
notebook ne contient aucun emoji et n'affiche que des tableaux et des
graphiques — jamais de `print()`.

### Les figures produites

| Fichier | Question à laquelle il répond |
|---|---|
| `V1_V2_distribution_et_tranches.png` | Quelle est la durée typique, et quand est-elle la plus longue ? |
| `V3_jours_semaine.png` | Le retard change-t-il selon le jour de la semaine ? |
| `V4_V6_trafic_meteo.png` | Quel effet ont le trafic et la météo, et se cumulent-ils ? |
| `V7_V9_distance_et_zones.png` | Quel effet a la distance, et où sont les zones à risque ? |
| `V8_vehicule_par_distance.png` | Quel véhicule choisir selon le type de trajet ? |
| `V10_matrice_correlation.png` | Quelles variables sont associées entre elles ? |
| `V11_ecarts_par_facteur.png` | Quel facteur a le plus d'écart ? |
| `V12_effectifs_groupes.png` | Quels groupes sont assez grands pour conclure ? |
| `V13_tableau_de_bord_kpi.png` | Synthèse visuelle des 12 KPI |
| `V14_gains_par_scenario.png` | Quel levier d'action rapporte le plus ? |

Les tableaux de résultats sont exportés dans `outputs/tables/` :
`kpi.csv`, `scenarios.csv`, `hypotheses.csv`, `effectifs_groupes.csv` et
`recommandations.md`.

---

## Deux précautions pour lire les chiffres

### Les 4 leviers d'action ne sont pas cumulables

Ils se recouvrent : une livraison peut être en congestion **et** par mauvais
temps **et** aux heures de pointe. Additionner les gains de `scenarios.csv`
compterait donc les mêmes livraisons plusieurs fois. Le tableau porte une colonne
`cumulable` qui vaut `non` sur chaque ligne, et `recommandations.md` précise en
bas de page le total réaliste, retenu une seule fois par livraison.

### Les KPI et les statistiques complémentaires

`kpi.csv` contient **12 lignes portant un code** (K1 à K12) : ce sont les
indicateurs du cahier des charges, et eux seuls. Deux lignes supplémentaires
n'ont pas de code — durée moyenne et corrélation distance/durée — : ce sont des
statistiques de confort, utiles à la lecture mais hors décompte.

---

## Note sur le nombre de colonnes

Le fichier `train.csv` contient **20 colonnes**. Après la tâche 2, le tableau
comporte **22 colonnes** : les 20 colonnes d'origine, plus deux colonnes
construites :

- `duree` — la durée de livraison, extraite de `Time_taken(min)` qui la stockait
  avec son suffixe d'unité ;
- `etat_vehicule` — l'état du véhicule, extrait de `Vehicle_condition`.

Ce sont les **seules** deux colonnes ajoutées. Les variables d'analyse de la
tâche 3 (`distance_km`, `attente_min`, `tranche`, etc.) ne sont pas comptées ici,
puisqu'elles servent au calcul et non à la conservation de l'information brute.

---

## Source des données

| Information | Valeur |
|---|---|
| Jeu de données | Food Delivery Dataset (Kaggle) |
| Éditeur | gauravmalik26 |
| Lien | <https://www.kaggle.com/datasets/gauravmalik26/food-delivery-dataset> |
| Fichier | `train.csv` — 45 593 lignes, 20 colonnes |
| Période | février → avril 2022 (8 semaines) |
