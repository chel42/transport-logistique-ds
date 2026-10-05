# Guide de l'Analyse — Transport & Logistique

Ce document constitue le guide de référence technique et fonctionnel du projet d'analyse des retards de livraison. Il détaille l'organisation de l'environnement de travail, la logique de traitement de chaque cellule du notebook principal et l'interprétation méthodique de chaque graphique avec ses **valeurs numériques exactes issues des calculs**.

---

## 1. Structure Actuelle du Dépôt

Le projet est organisé selon une arborescence modulaire qui sépare strictement les données brutes, le code d'analyse, les figures générées et les livrables d'aide à la décision.

```text
transport-logistique-ds/
│
├── notebooks/
│   └── analyse_delais_livraison.ipynb   # Document principal de calcul (66 cellules)
│
├── data/
│   └── raw/
│       └── train.csv                    # Données brutes Kaggle (45 593 lignes, 20 colonnes)
│
├── figures/                             # 10 visualisations exportées en haute résolution
│   ├── V1_V2_distribution_et_tranches.png
│   ├── V3_jours_semaine.png
│   ├── V4_V6_trafic_meteo.png
│   ├── V7_V9_distance_et_zones.png
│   ├── V8_vehicule_par_distance.png
│   ├── V10_matrice_correlation.png
│   ├── V11_ecarts_par_facteur.png
│   ├── V12_effectifs_groupes.png
│   ├── V13_tableau_de_bord_kpi.png
│   └── V14_gains_par_scenario.png
│
├── outputs/
│   ├── RAPPORT_MIA_FOODS.md              # Synthèse managériale vulgarisée pour les décideurs
│   └── tables/                          # Données tabulaires exportées au format CSV et Markdown
│       ├── effectifs_groupes.csv        # Contrôle des tailles d'échantillons (KPI K12)
│       ├── hypotheses.csv               # Bilan de validation des 6 hypothèses statistiques
│       ├── kpi.csv                      # Synthèse chiffrée des 12 KPI (K1 à K12)
│       ├── recommandations.md           # Tableau de synthèse des actions prioritaires
│       └── scenarios.csv                # Chiffrage des gains potentiels par levier d'action
│
├── requirements.txt                      # Spécification des dépendances Python (Pip)
├── environment.yml                       # Configuration de l'environnement Conda
├── README.md                             # Documentation d'accueil et protocole d'installation
└── GUIDE_ANALYSE.md                      # Présent document de référence
```

### Principes d'architecture
- **Immuabilité des données sources :** Le fichier `data/raw/train.csv` n'est jamais écrasé ni modifié directement.
- **Résolution dynamique des chemins :** Le notebook localise la racine du projet de manière relative (`Path.cwd()`), permettant une exécution portable sur n'importe quel système d'exploitation.
- **Double niveau de restitution :** Les calculs scientifiques sont concentrés dans le notebook, tandis que les conclusions opérationnelles sont formalisées dans `RAPPORT_MIA_FOODS.md` dans un langage orienté gestion d'entreprise.

---

## 2. Explication Détaillée des Cellules du Notebook

Le notebook `analyse_delais_livraison.ipynb` comprend **66 cellules** (26 de texte explicatif et 40 de code exécutable), ordonnées selon le plan de 16 tâches du cahier des charges.

### Partie A — Préparation et Ingénierie des Données (Tâches 1 à 4)

#### Cellule 03 (Code) — Initialisation de l'environnement
- **Rôle :** Importe les bibliothèques requises (`pandas`, `numpy`, `matplotlib`, `seaborn`, `scipy`), configure la résolution automatique des répertoires du projet et définit la charte graphique globale (`whitegrid`, dimensions des figures).
- **Paramètre clé :** Fixe le seuil de fiabilité statistique `SEUIL_MIN_LIVRAISONS = 100` (KPI K12), qui servira de garde-fou contre les conclusions tirées d'échantillons trop réduits.

#### Cellule 05 (Code) — Ingestion et audit de structure (Tâche 1)
- **Rôle :** Charge le fichier brut `train.csv` et construit un profil structurel de chaque colonne (type de donnée, cardinalité des modalités uniques, exemple de valeur non nulle).
- **Enseignement technique :** Identifie la présence d'espaces superflus autour des chaînes de caractères, des valeurs manquantes stockées sous forme de texte `"NaN "` et du suffixe `(min)` dans la variable cible `Time_taken(min)`.

#### Cellule 07 (Code) — Nettoyage, typage et normalisation (Tâche 2)
- **Rôle :**
  1. Élimine les espaces périphériques sur l'ensemble des 12 colonnes textuelles.
  2. Remplace explicitement les représentations textuelles de l'absence (`"NaN"`, `"nan"`, `"None"`, `""`) par l'objet standard `pd.NA`.
  3. Extrait la valeur numérique de `Time_taken(min)` dans la nouvelle colonne `duree` (`float`).
  4. Convertit les identifiants d'âge, de notation, d'état du véhicule et de coordonnées GPS en types numériques exploitables.
- **Sortie :** Tableau récapitulatif des 22 colonnes résultantes (les 20 colonnes d'origine conservées pour audit, plus `duree` et `etat_vehicule`).

#### Cellule 08 (Code) — Filtrage des aberrations et imputation (Tâche 2)
- **Rôle :**
  1. Supprime les doublons éventuels sur l'identifiant unique de livraison `ID`.
  2. Élimine les coordonnées géographiques situées hors du territoire national de l'étude (notamment le point `(0, 0)` au milieu de l'océan Atlantique, correspondant à un défaut de géolocalisation GPS).
  3. Filtre les durées impossibles ou non plausibles (conservation de l'intervalle 1 à 200 minutes).
  4. Impute la modalité explicite `"Inconnu"` aux variables catégorielles incomplètes (`Weatherconditions`, `Road_traffic_density`, `City`, `Type_of_vehicle`) afin de conserver ces observations réelles dans l'analyse.
- **Sortie :** Journal de nettoyage quantifiant les lignes retirées par opération (taux de conservation global de 91,07 %, soit 41 522 livraisons valides sur 45 593 initiales).

#### Cellule 09 (Code) — Bilan de volumétrie du nettoyage
- **Rôle :** Produit un tableau de synthèse consolidé affichant le volume initial lu (45 593), le volume final analysé (41 522), le nombre de rejets (4 071) et le nombre de colonnes (22).

#### Cellule 11 (Code) — Calcul de la distance géodésique (Tâche 3)
- **Rôle :** Implémente la formule mathématique d'Haversine de manière vectorisée sur les coordonnées du restaurant et du client.
- **Raisonnement :** Calcule la distance à vol d'oiseau (`distance_km`) qui représente la borne minimale physique du trajet. Le code valide également la précision de la formule sur deux trajets de référence connus (Paris-Londres : 343,6 km calculés pour 344 km réels ; Bogota-Medellín : 238,7 km calculés pour 239 km réels).

#### Cellule 12 (Code) — Traitement temporel et temps d'attente (Tâche 3)
- **Rôle :**
  1. Vérifie le taux de valeurs temporelles illisibles sur `Time_Orderd` et `Time_Order_picked`.
  2. Assemble la date et l'heure pour former deux horodatages complets `datetime64`.
  3. Calcule la durée d'attente en restaurant `attente_min` (délai entre commande et prise en charge par le livreur).
  4. **Correction du franchissement de minuit :** Lorsqu'une commande est passée avant minuit (ex. 23h50) et collectée après (ex. 00h10), l'écart brut est négatif (760 occurrences identifiées) ; le code ajoute 24 heures (1 440 minutes) pour rétablir la réalité chronologique.
- **Contrôle :** Attente médiane de 10,0 minutes, attente maximale plafonnée à 15,0 minutes, et zéro valeur supérieure à 90 minutes après correction.

#### Cellule 13 (Code) — Découpage calendaire et discrétisation (Tâche 3)
- **Rôle :**
  1. Extrait l'heure de commande (`heure`) et le numéro du jour de la semaine pour calculer l'indicateur binaire `est_weekend` (samedi/dimanche) et la variable textuelle `jour_nom` (Lundi à Dimanche).
  2. Identifie les heures de pointe conventionnelles (`est_pointe` : 8h, 9h, 18h, 19h, 20h).
  4. Discrétise l'heure en 5 tranches : Nuit (0h-6h), Matin (6h-11h), Midi (11h-15h), Après-midi (15h-18h), Soir (18h-23h).
  5. Segmente la distance en 5 classes kilométriques : `< 2 km`, `2-5 km`, `5-10 km`, `10-20 km`, `> 20 km`.

#### Cellule 14 (Code) — Normalisation des étiquettes et variables ordinales (Tâche 3)
- **Rôle :** Traduit et harmonise les modalités de la météo, des types de ville et des véhicules. Attribue des encodages ordinaux à la densité du trafic (1: Low à 4: Jam) et aux conditions météo (1: Sunny à 6: Sandstorms) pour les calculs de corrélation.
- **Hypothèse à connaître :** l'ordre du trafic est naturel, mais l'ordre de la météo est une convention discutable (Sandstorms > Stormy n'est pas une évidence). Ces codages ne servent qu'à la matrice de corrélation (Figure 7) ; toutes les comparaisons de groupes utilisent les catégories brutes, non encodées.

#### Cellule 16 (Code) — Définition du retard et valorisation financière (Tâche 4)
- **Rôle :**
  1. Fixe le seuil de retard à la médiane de l'échantillon (`26,0 minutes`).
  2. Crée l'indicateur binaire `en_retard` (`duree > 26 min`) et calcule les `minutes_perdues` pour chaque course au-delà de cette borne.
  3. Introduit un barème de valorisation horaire **illustratif** (aucun montant dans le jeu de données ; scénario central à 1 200 par heure, soit 20 par minute) pour quantifier l'impact financier théorique.
- **Sortie :** Taux de retard de 45,57 %, moyenne de 3,94 minutes perdues par livraison, et volume global de 163 579 minutes perdues sur l'ensemble de la période.

#### Cellule 17 (Code) — Analyse de sensibilité du seuil de retard (Tâche 4)
- **Rôle :** Compare le taux de retard et le volume de temps perdu selon trois définitions alternatives :
  - Médiane (26,0 min) : 45,57 % de retard, 163 579 min perdues.
  - 75e percentile (32,0 min) : 24,81 % de retard, 74 878 min perdues.
  - Seuil fixe (40,0 min) : 8,86 % de retard, 18 636 min perdues.
- **Conclusion :** Démontre que le classement des facteurs d'influence reste rigoureusement inchangé quel que soit le seuil retenu.

#### Cellule 18 (Code) — Analyse de sensibilité du coût horaire (Tâche 4)
- **Rôle :** Teste trois hypothèses de valorisation économique (basse à 500/h : 1 363 158 ; centrale à 1 200/h : 3 271 580 ; haute à 2 500/h : 6 815 792) confirmant que le choix tarifaire n'impacte que l'échelle monétaire sans fausser les priorités opérationnelles.

---

### Partie B — Analyses Univariées et Bivariées (Tâches 5 à 9)

#### Cellule 21 (Code) — Fonction d'agrégation et synthèse par tranche horaire (Tâche 5)
- **Rôle :** Définit la fonction réutilisable `par_categorie()` qui calcule systématiquement l'effectif, la durée moyenne, la durée médiane, le taux de retard et les minutes perdues moyennes pour chaque modalité. Génère la synthèse pour les 5 tranches de la journée.

#### Cellule 22 (Code) — Visualisation V1 et V2 (Axe Temps)
- **Rôle :** Construit et exporte la figure `V1_V2_distribution_et_tranches.png`.
- **Graphiques générés :** Histogramme des fréquences des durées de livraison (seuil médian à 26 min) et diagramme en barres de la durée moyenne par tranche horaire.

#### Cellule 23 (Code) — Synthèse par jour de la semaine (Tâche 5)
- **Rôle :** Agrège les métriques de livraison pour chaque jour de la semaine (Lundi à Dimanche) via `par_categorie("jour_nom")`. Résultats :

| Jour | Livraisons | Durée moy. (min) | Taux retard (%) | Min. perdues moy. |
|---|---:|---:|---:|---:|
| Lundi | 5 490 | 26,24 | 44,72 | 3,88 |
| Mardi | 5 612 | 25,45 | 42,57 | 3,39 |
| Mercredi | 6 285 | 27,76 | 50,93 | 4,85 |
| Jeudi | 5 607 | 25,31 | 41,82 | 3,28 |
| Vendredi | 6 125 | 26,83 | 47,80 | 4,27 |
| Samedi | 5 575 | 26,00 | 44,61 | 3,75 |
| Dimanche | 5 503 | 26,43 | 45,54 | 4,01 |

- **Constat :** Le mercredi est le jour le plus critique (27,76 min, 50,93 % de retard), le jeudi est le plus performant (25,31 min, 41,82 % de retard). L'écart maximal entre jours est de 2,45 minutes, ce qui confirme que le jour de la semaine est un facteur d'influence modéré mais mesurable.

#### Cellule 24 (Code) — Visualisation V3 (Jours de la semaine)
- **Rôle :** Construit et exporte la figure `V3_jours_semaine.png`.
- **Graphiques générés :** Deux diagrammes en barres côte à côte représentant la durée moyenne et le taux de retard par jour de la semaine (Lundi à Dimanche), avec un code couleur distinguant les jours de semaine (bleu foncé) du week-end (bleu clair).

#### Cellule 25 (Code) — Test statistique sur les heures de pointe et week-end (Tâche 5)
- **Rôle :**
  - Compare la durée en heures de pointe (moyenne 27,58 min, retard 50,96 %) vs autres heures (moyenne 25,52 min, retard 42,20 %) via un test de Mann-Whitney U ($p = 4 \times 10^{-91}$).
  - Compare les jours de semaine (moyenne 26,35 min, retard 45,75 %) vs week-end (moyenne 26,21 min, retard 45,07 %), confirmant que le week-end n'a pas d'impact significatif sur les délais.

#### Cellule 27 (Code) — Indicateurs de l'axe spatial (Tâche 6)
- **Rôle :** Calcule le coefficient de corrélation linéaire entre distance et durée ($r = 0{,}321$) ainsi que la pente par régression linéaire ($0{,}538\text{ min/km}$, KPI K9).

#### Cellule 28 (Code) — Contrôle d'effectif par typologie urbaine (Tâche 6)
- **Rôle :** Ventile les performances par type de ville (`Metropolitaine` : 27,3 min, n = 34 093 ; `Urbaine` : 22,5 min, n = 6 364 ; `Semi-urbaine` : 49,7 min, n = 164 ; `Inconnu` : 26,9 min, n = 901) et valide le dépassement du seuil $n \ge 100$.

#### Cellule 29 (Code) — Visualisation V7 et V9 (Axe Espace)
- **Rôle :** Construit et exporte la figure `V7_V9_distance_et_zones.png`.
- **Graphiques générés :** Nuage de points durée/distance avec courbe des moyennes par classe kilométrique, et cartographie géographique (densité de flux en hexagones bleus et taux de retard localisé).

#### Cellule 31 (Code) — Synthèse par niveau de trafic (Tâche 7)
- **Rôle :** Agrège les métriques de livraison selon les quatre paliers de densité routière (`Low`, `Medium`, `High`, `Jam`).

#### Cellule 32 (Code) — Regroupement et synthèse météo (Tâche 7)
- **Rôle :** Regroupe les conditions climatiques en deux macro-familles comparables : favorable (`Sunny`, `Windy` : moyenne 24,03 min) et sévère (`Fog`, `Stormy`, `Sandstorms` : moyenne 26,93 min).

#### Cellule 33 (Code) — Tests statistiques trafic et météo (Tâche 7)
- **Rôle :** Calcule les écarts moyens nets et teste la significativité statistique via Mann-Whitney U pour l'impact du trafic fluide vs embouteillé ($+9{,}91\text{ min}$, $p = 0{,}0$) et pour la météo favorable vs sévère ($+2{,}89\text{ min}$, $p = 1{,}8 \times 10^{-79}$).

#### Cellule 34 (Code) — Visualisation V4 et V6 (Axe Conditions)
- **Rôle :** Construit et exporte la figure `V4_V6_trafic_meteo.png`.
- **Graphiques générés :** Boîtes à moustaches de la durée selon les niveaux de trafic avec ligne de tendance des moyennes, et matrice thermique (heatmap) croisant la densité du trafic et les conditions météo.

#### Cellule 35 (Code) — Test de super-additivité du trafic et de la météo (Tâche 7 / Hypothèse H6)
- **Rôle :** Mesure si les effets conjoints du trafic dense et du mauvais temps s'additionnent simplement ou se renforcent.
- **Résultat mathématique :** La durée de référence (Low + Favorable) est de 21,10 min. L'effet isolé du trafic dense est de 26,37 min (+5,27 min). L'effet isolé de la météo sévère est de 21,08 min (-0,02 min). La somme prévisionnelle additive est de 26,36 min. En situation réelle combinée, la durée observée atteint 31,27 min, soit une amplification super-additive de **+4,91 minutes**.

#### Cellule 37 (Code) — Analyse de l'efficacité par type de véhicule (Tâche 8)
- **Rôle :** Crée le ratio `minutes_par_km` pour évaluer la vitesse opérationnelle relative de chaque véhicule indépendamment de la distance parcourue.

#### Cellule 38 (Code) — Visualisation V8 (Véhicule par classe de distance)
- **Rôle :** Construit et exporte la figure `V8_vehicule_par_distance.png`.
- **Graphique généré :** Carte thermique croisant le type de véhicule et les classes de distance avec les durées moyennes réelles calculées.

#### Cellule 40 (Code) — Concentration des retards et du coût (Tâche 9)
- **Rôle :** Identifie et classe les combinaisons trafic/météo qui génèrent le plus fort volume cumulé de minutes perdues (ex. Jam x Severe concentre 47 706 minutes perdues pour 5 801 livraisons).

#### Cellule 41 (Code) — Visualisation V11 (Hiérarchie des facteurs de retard)
- **Rôle :** Construit et exporte la figure `V11_ecarts_par_facteur.png`.
- **Graphique généré :** Diagramme en barres horizontales ordonnant les 4 écarts majeurs : Météo (+2,89 min), Distance (+8,40 min), Heure (+9,19 min) et Trafic (+9,91 min).

---

### Partie C — Croiser, Comparer et Valider (Tâches 10 à 12)

#### Cellule 44 (Code) — Matrice de corrélation et Visualisation V10 (Tâche 10)
- **Rôle :**
  1. Exclut les modalités `Inconnu` pour éviter d'introduire un biais artificiel dans les calculs.
  2. Calcule la matrice de corrélation de Pearson entre les variables numériques et ordinales.
  3. Construit et exporte la figure `V10_matrice_correlation.png`.

#### Cellule 46 (Code) — Contrôle d'échantillonnage K12 et Visualisation V12 (Tâche 11)
- **Rôle :**
  1. Évalue la taille d'échantillon de chaque sous-groupe comparé par rapport au seuil `SEUIL_MIN_LIVRAISONS = 100`.
  2. Construit et exporte la figure `V12_effectifs_groupes.png`, mettant en évidence que seul le véhicule `bicycle` ($n = 43$) est sous le seuil critique.

#### Cellule 47 (Code) — Bootstrap et utilité opérationnelle (Tâche 11)
- **Rôle :**
  1. Génère un intervalle de confiance à 95% par ré-échantillonnage Bootstrap (1 000 itérations) sur l'écart de trafic fluide vs embouteillé : `[9,71 min ; 10,12 min]`.
  2. Démontre que la corrélation entre temps d'attente en restaurant et durée de livraison est nulle ($r = -0{,}010$), invalidant ce levier d'action.

#### Cellule 49 (Code) — Consolidation du tableau des 12 KPI (Tâche 12)
- **Rôle :** Compile dans un DataFrame unique les **12 KPI portant un code (K1 à K12)**, avec une colonne `Code` qui permet de les comparer ligne à ligne au cahier des charges. Deux statistiques complémentaires (durée moyenne, corrélation distance/durée) figurent dans la même table avec le code `-` : elles ne comptent pas dans le décompte des KPI.

#### Cellule 50 (Code) — Visualisation V13 (Tableau de bord visuel des KPI)
- **Rôle :** Construit et exporte la figure `V13_tableau_de_bord_kpi.png`.
- **Graphique généré :** Tableau de bord sous forme de barres horizontales classées par magnitude croissante et colorées par catégorie.

---

### Partie D — Scénarios d'Action et Recommandations (Tâches 13 à 16)

#### Cellule 53 (Code) — Chiffrage des scénarios d'optimisation (Tâche 13)
- **Rôle :** Implémente la fonction `gain_par_livraison()` qui attribue à **chaque livraison** le gain d'un levier, mesuré à classe de distance comparable. Les livraisons qu'un levier ne concerne pas reçoivent `NaN`, ce qui permet de savoir quels leviers se recouvrent. Une colonne `cumulable` vaut `non` sur chaque ligne : les 4 gains ne peuvent pas être additionnés.
  - *Planifier hors des zones congestionnées :* 12 950 livraisons concernées, durée actuelle 31,19 min, gain de +8,78 min/livraison, **gain total de 113 641 min**.
  - *Adapter le véhicule au type de trajet :* 41 522 livraisons concernées, durée actuelle 26,32 min, gain de +2,39 min/livraison, **gain total de 99 173 min**.
  - *Prévoir une marge par mauvais temps :* 20 747 livraisons concernées, durée actuelle 26,93 min, gain de +2,88 min/livraison, **gain total de 59 733 min**.
  - *Décaler les livraisons hors des pointes :* 15 966 livraisons concernées, durée actuelle 27,58 min, gain de +1,88 min/livraison, **gain total de 29 994 min**.
  - **Attention :** le levier « véhicule » porte sur les 41 522 livraisons, c'est-à-dire la totalité du jeu. Il englobe donc les livraisons déjà comptées par les trois autres leviers. C'est la raison principale pour laquelle les lignes ne s'additionnent pas.

#### Cellule 54 (Code) — Visualisation V14 (Gains par scénario d'action)
- **Rôle :** Construit et exporte la figure `V14_gains_par_scenario.png`.
- **Graphique généré :** Diagramme en barres horizontales classant les scénarios selon le volume total de minutes économisées.

#### Cellule 57 (Code) — Validation formelle des hypothèses (Tâche 15)
- **Rôle :** Formalise l'état de validation des 6 hypothèses de travail :
  - H1 (Distance) : confirmée ($r = 0{,}32$ ; $0{,}54\text{ min/km}$, portée modérée).
  - H2 (Trafic) : confirmée (écart $9{,}9\text{ min}$ ; $p = 0$, portée forte).
  - H3 (Pointes) : confirmée (écart $2{,}1\text{ min}$ ; $p = 4 \times 10^{-91}$, portée modérée).
  - H4 (Véhicule) : confirmée (arbitrage selon la distance).
  - H5 (Attente) : **infirmée** (corrélation $-0{,}010$, portée négligeable).
  - H6 (Trafic x Météo) : confirmée (surcoût d'interaction $+4{,}9\text{ min}$, effet super-additif).

#### Cellule 61 (Code) — Priorisation des recommandations (Tâche 16)
- **Rôle :** Met en forme le tableau de restitution opérationnelle classant les 4 actions par priorité décroissante de gain total.

#### Cellule 62 (Code) — Contrôle de cohérence et export Markdown
- **Rôle :** Écrit `outputs/tables/recommandations.md` en y ajoutant un avertissement de non-cumulabilité, et calcule le **total réaliste** : pour chaque livraison, seul le meilleur gain parmi les leviers qui la concernent est retenu. Le total non cumulable ressort à **241 279 min**, contre **302 541 min** si l'on additionne les quatre lignes à la main — soit 61 262 minutes de gain fantôme dues au double comptage.

#### Cellule 65 (Code) — Exportation définitive et audit final du livrable
- **Rôle :** Sauvegarde les 4 fichiers CSV de référence (`kpi.csv`, `scenarios.csv`, `hypotheses.csv`, `effectifs_groupes.csv`) et affiche le bilan de clôture (41 522 lignes, 91,07 % de conservation, 12 KPI du cahier des charges + 2 statistiques complémentaires, 10 figures, 4 tables).

---

## 3. Catalogue et Analyse des Graphiques du Projet

Les 10 visualisations enregistrées dans le dossier `figures/` présentent les **valeurs numériques réelles** suivantes :

```text
figures/
├── V1_V2_distribution_et_tranches.png
├── V3_jours_semaine.png
├── V7_V9_distance_et_zones.png
├── V4_V6_trafic_meteo.png
├── V8_vehicule_par_distance.png
├── V11_ecarts_par_facteur.png
├── V10_matrice_correlation.png
├── V12_effectifs_groupes.png
├── V13_tableau_de_bord_kpi.png
└── V14_gains_par_scenario.png
```

---

### Figure 1 : `V1_V2_distribution_et_tranches.png`

```text
+---------------------------------------+---------------------------------------+
| V1 - Distribution des durées          | V2 - Durée moyenne par tranche        |
|                                       |                                       |
|      ▲                                |  Durée (min)                          |
|  Nb  |      _/\_                      |   30 |                 [26.3] [28.8]  |
|  de  |     /    \                     |   25 |          [26.9]   ■      ■     |
| liv. |    /  ||  \                    |   20 |   [19.6]   ■      ■      ■     |
|      |   /   ||   \                   |   15 |     ■      ■      ■      ■     |
|      +---/---||----\--------►         |    0 +-----+------+------+------+     |
|             26 min (Seuil médian)     |          Matin   Midi   A-Midi  Soir  |
+---------------------------------------+---------------------------------------+
```

- **Questions traitées :** Quelle est la distribution réelle des délais ? Quand les retards surviennent-ils ?
- **Valeurs réelles affichées sur le graphique :**
  - Seuil médian de retard (V1) : **26,0 minutes** (45,57 % de livraisons au-delà).
  - Durées moyennes par tranche (V2) :
    - **Matin :** 19,56 min (19,6 affiché sur la barre ; 5 272 livraisons ; 13,35 % de retard).
    - **Nuit :** 21,97 min (22,0 affiché ; 395 livraisons ; 25,57 % de retard).
    - **Après-midi :** 26,29 min (26,3 affiché ; 5 377 livraisons ; 48,73 % de retard).
    - **Midi :** 26,87 min (26,9 affiché ; 4 056 livraisons ; 52,37 % de retard).
    - **Soir :** 28,75 min (28,8 affiché ; 20 991 livraisons ; 55,80 % de retard).
- **Interprétation :** Le soir concentre à la fois le plus fort volume d'activité (plus de 50 % des courses) et la pire moyenne (28,8 min), générant un écart de **9,19 minutes** par rapport aux livraisons du matin.

---

### Figure 2 : `V3_jours_semaine.png`

```text
+-------------------------------------------------------------------------------+
| Durée moyenne par jour        |  Taux de retard par jour                      |
|                               |                                               |
|  28 |        [27.8]            |   51 |        [50.9]                           |
|  27 |  [26.2]  ■   [26.8]      |   48 |  [44.7]  ■   [47.8]                      |
|  26 |  ■   [25.5]  ■   ■  [26.4]|   45 |  ■   [42.6]  ■   ■  [45.5]              |
|  25 |      ■        ■          |   42 |      ■        ■                          |
|      Lun Mar Mer Jeu Ven Sam Dim|       Lun Mar Mer Jeu Ven Sam Dim             |
|      ---- seuil 26 min ----     |                                               |
+-------------------------------------------------------------------------------+
```

- **Question traitée :** Le retard change-t-il selon le jour de la semaine ?
- **Valeurs réelles affichées (issues de `par_jour`) :**
  - **Mercredi :** 27,76 min de durée moyenne, 50,93 % de taux de retard (6 285 livraisons) — le pire jour.
  - **Jeudi :** 25,31 min, 41,82 % (5 607 livraisons) — le meilleur jour.
  - **Vendredi :** 26,83 min, 47,80 % (6 125 livraisons).
  - **Lundi :** 26,24 min, 44,72 % ; **Mardi :** 25,45 min, 42,57 % ; **Samedi :** 26,00 min, 44,61 % ; **Dimanche :** 26,43 min, 45,54 %.
  - Écart max / min : **2,45 min** de durée moyenne et **9,11 points** de taux de retard (Mercredi vs Jeudi).
  - Code couleur : jours de semaine en bleu foncé, week-end en bleu clair ; trait pointillé = seuil médian de 26 min.
- **Interprétation :** L'écart entre jours est réel mais modéré (moins de 2,5 min de durée moyenne), et il ne suit pas le schéma « week-end = pire jours » : le mercredi est le point haut et le jeudi le point bas. Le jour de la semaine est un facteur secondaire comparé au trafic (+9,91 min) ou à l'horaire (+9,19 min). Les effectifs de tous les jours (5 490 à 6 285) dépassent le seuil de 100 (K12).

---

### Figure 3 : `V7_V9_distance_et_zones.png`

```text
+---------------------------------------+---------------------------------------+
| V7 - Durée selon la distance          | V9 - Carte des zones à risque         |
|                                       |                                       |
|  Durée                                |   Lat ▲                               |
|  (min)                                |       |       . . : : █ █ : .         |
|   35 |               • 30.7 min       |       |     . : █ █ █ █ █ █ :         |
|   30 |        • 24.3 min              |       |     : █ █ [CRITIQUE] █        |
|   20 |    • 20.9 min (Pente: +0.54)   |       |       . : : █ █ : .           |
|      +----+-------+-------+----►      |       +-----------------------►       |
|          2 km    10 km   20 km        |                               Lon     |
+---------------------------------------+---------------------------------------+
```

- **Questions traitées :** Quel surcoût temporel induit la distance ? Où sont situées les zones de congestion ?
- **Valeurs réelles calculées et représentées :**
  - Pente de régression linéaire : **+0,54 min/km** ($0{,}538\text{ min/km}$).
  - Corrélation distance / durée : **0,32**.
  - Durée moyenne par classe de distance (V7) :
    - `< 2 km` : **20,89 min** (distance moyenne 1,48 km).
    - `2-5 km` : **22,38 min** (distance moyenne 3,07 km).
    - `5-10 km` : **24,32 min** (distance moyenne 7,91 km).
    - `10-20 km` : **29,29 min** (distance moyenne 13,42 km).
    - `> 20 km` : **30,68 min** (distance moyenne 20,52 km).
  - Écart `< 2 km` vs `10-20 km` : **8,40 minutes**.
- **Interprétation :** La distance augmente la durée de façon linéaire et modérée. La cartographie spatiale (V9) montre que les retards élevés (> 60 %) ne sont pas uniformément répartis mais concentrés dans des mailles urbaines spécifiques à forte densité d'arrêts.

---

### Figure 4 : `V4_V6_trafic_meteo.png`

```text
+---------------------------------------+---------------------------------------+
| V4 - Dispersion selon le trafic       | V6 - Matrice croisée Trafic x Météo   |
|                                       |                                       |
|  Durée                                |  Trafic                               |
|   50 |             ┬                  |   Jam  |  26.8   36.7   [32.3]        |
|   40 |      ┬      ┼                  |   High |  25.2   29.0    28.0         |
|   30 |      ┼      ┴                  |   Med  |  24.1   28.5    27.9         |
|   20 |      ┴    [Jam: 31.2 min]      |   Low  |  21.1   22.2    21.1         |
|      +------+------+-----------►      |        +-------+-------+------        |
|            Low    High   Jam          |               Fav     Neutre  Sévère  |
+---------------------------------------+---------------------------------------+
```

- **Questions traitées :** Quel impact direct a le trafic ? Comment s'exprime l'effet combiné avec la météo ?
- **Valeurs réelles affichées :**
  - Durées moyennes par niveau de trafic (V4) :
    - `Low` : **21,28 min** (médiane 20,0 min ; taux de retard 20,81 % ; n = 14 095).
    - `Medium` : **26,74 min** (médiane 27,0 min ; taux de retard 50,20 % ; n = 10 023).
    - `High` : **27,21 min** (médiane 27,0 min ; taux de retard 54,13 % ; n = 4 046).
    - `Jam` : **31,19 min** (médiane 31,0 min ; taux de retard 66,19 % ; n = 12 950).
    - Écart de durée moyen fluide vs embouteillé : **9,91 minutes** (9,9 affiché dans le titre).
  - Matrice thermique croisée Trafic x Météo (V6) :
    - Low x Favorable : **21,10 min** | Low x Neutre : **22,22 min** | Low x Sévère : **21,08 min**
    - Medium x Favorable : **24,09 min** | Medium x Neutre : **28,54 min** | Medium x Sévère : **27,89 min**
    - High x Favorable : **25,18 min** | High x Neutre : **28,96 min** | High x Sévère : **27,98 min**
    - Jam x Favorable : **26,75 min** | Jam x Neutre : **36,65 min** | Jam x Sévère : **32,28 min**
- **Interprétation :** La météo sévère seule en trafic fluide ne dégrade pas les délais (21,08 min vs 21,10 min). En revanche, lorsque le trafic est saturé (`Jam`), les intempéries entraînent une sur-dégradation atteignant 32,28 min (et 36,65 min en conditions neutres brumeuses), validant l'effet d'amplification super-additif (+4,91 min).

---

### Figure 5 : `V8_vehicule_par_distance.png`

```text
+-------------------------------------------------------------------------------+
| V8 - Durée selon le véhicule et la classe de distance (Valeurs réelles)       |
|                                                                               |
|  Véhicule                                                                     |
|  bicycle          |   [14.1]       25.2        22.8        32.8        30.0   |
|  electric scooter |    19.6       [20.9]      [22.6]      [27.8]       28.0   |
|  motorcycle       |    22.7        23.7        25.7        31.2        31.2   |
|  scooter          |    19.8       [20.9]       22.5        28.0       [27.7]  |
|                   +------------+-----------+-----------+-----------+--------- |
|                       < 2 km      2-5 km     5-10 km    10-20 km    > 20 km   |
+-------------------------------------------------------------------------------+
```

- **Question traitée :** Quel véhicule offre la meilleure performance réelle selon la distance ?
- **Valeurs réelles calculées dans le tableau croisé :**
  - `< 2 km` : **bicycle (14,1 min)**, electric scooter (19,6 min), scooter (19,8 min), motorcycle (22,7 min).
  - `2-5 km` : **electric scooter (20,9 min)**, **scooter (20,9 min)**, motorcycle (23,7 min), bicycle (25,2 min).
  - `5-10 km` : **scooter (22,5 min)**, electric scooter (22,6 min), bicycle (22,8 min), motorcycle (25,7 min).
  - `10-20 km` : **electric scooter (27,8 min)**, scooter (28,0 min), motorcycle (31,2 min), bicycle (32,8 min).
  - `> 20 km` : **scooter (27,7 min)**, electric scooter (28,0 min), bicycle (30,0 min), motorcycle (31,2 min).
- **Interprétation :** Sur les segments courts et intermédiaires (`< 10 km`), les deux-roues légers (trottinettes et scooters) sont systématiquement plus rapides de 2 à 3 minutes que les motos. Les motos souffrent des contraintes de manœuvre et de stationnement en milieu dense.

---

### Figure 6 : `V11_ecarts_par_facteur.png`

```text
+-------------------------------------------------------------------------------+
| V11 - Synthèse ordonnée des écarts de durée moyenne par facteur               |
|                                                                               |
|  Facteur                                                                      |
|  Trafic (fluide vs embouteillé)    | █████████████████████████████ 9.91 min  |
|  Heure (matin vs soir)             | ████████████████████████ 9.19 min       |
|  Distance (< 2 km vs 10-20 km)     | ██████████████████ 8.40 min             |
|  Météo (favorable vs sévère)       | ████████ 2.89 min                       |
|                                    +----------------------------------------- |
|                                    0         2         4         6    8   10 min |
+-------------------------------------------------------------------------------+
```

- **Question traitée :** Quelle est la hiérarchie exacte des leviers de retard ?
- **Valeurs réelles affichées sur les barres :**
  - Météo (favorable vs sévère) : **2,89 min** (barre bleue, inférieur à 5 min).
  - Distance (< 2 km vs 10-20 km) : **8,40 min** (barre rouge, supérieur à 5 min).
  - Heure (matin vs soir) : **9,19 min** (barre rouge, supérieur à 5 min).
  - Trafic (fluide vs embouteillé) : **9,91 min** (barre rouge, supérieur à 5 min).
- **Interprétation :** Le trafic urbain (+9,91 min) et l'horaire vespéral (+9,19 min) sont les causes dominantes d'allongement des trajets, surpassant nettement l'impact de la distance (+8,40 min) et de la météo (+2,89 min).

---

### Figure 7 : `V10_matrice_correlation.png`

```text
+-------------------------------------------------------------------------------+
| V10 - Matrice de corrélation linéaire de Pearson                              |
|                                                                               |
|  Variable      | Corrélation avec la durée de livraison (duree)               |
|  niveau_trafic | +0.42 (Forte association positive)                           |
|  note_livreur  | -0.35 (Association négative / facteur de confusion)          |
|  distance_km   | +0.32 (Association positive modérée)                         |
|  age_livreur   | +0.30 (Association positive)                                 |
|  heure         | +0.19 (Association positive modérée)                         |
|  niveau_meteo  | +0.10 (Association faible)                                   |
|  attente_min   | -0.01 (Association rigoureusement nulle)                     |
+-------------------------------------------------------------------------------+
```

- **Question traitée :** Quelles variables sont corrélées à la durée de livraison ?
- **Note de lecture sur l'affichage :** les coefficients de la matrice sont arrondis à 2 décimales. Une cellule affichée **« -0.00 »** n'est pas un zéro négatif : elle signifie une corrélation légèrement négative dont la valeur absolue est inférieure à 0,005 (par exemple `attente_min` / `niveau_trafic` : -0,0042). Le signe moins est conservé pendant l'arrondi alors que la magnitude devient 0,00. En pratique, ces cellules se lisent comme une **corrélation nulle** (|r| < 0,005).
- **Valeurs réelles calculées dans la matrice :**
  - `niveau_trafic` : **+0,42** (première corrélation explicative).
  - `note_livreur` : **-0,35** (les livreurs chevronnés ont de meilleures notes et reçoivent les trajets complexes).
  - `distance_km` : **+0,32** (pente modérée).
  - `age_livreur` : **+0,30**.
  - `heure` : **+0,19** (également corrélée à la distance à **+0,58**, car les livraisons lointaines sont commandées plus tard).
  - `niveau_meteo` : **+0,10**.
  - `attente_min` : **-0,01** (corrélation nulle).
- **Interprétation :** La corrélation quasi nulle entre l'attente en cuisine (`attente_min`) et la durée totale prouve que le temps de préparation n'a aucune incidence sur la ponctualité finale.

---

### Figure 8 : `V12_effectifs_groupes.png`

```text
+-------------------------------------------------------------------------------+
| V12 - Effectifs des groupes et contrôle de puissance statistique (K12)        |
|                                                                               |
|  Ligne seuil : 100 observations minimum                                       |
|  Groupes conformes (Bleu)    : 17 modalités (de 164 à 34 093 observations)   |
|  Groupe non conforme (Rouge) : 1 modalité -> Véhicule / bicycle (n = 43)     |
+-------------------------------------------------------------------------------+
```

- **Question traitée :** Quels groupes possèdent un volume d'observations suffisant pour conclure ?
- **Valeurs réelles contrôlées :**
  - Seuil d'admissibilité : **$n \ge 100$**.
  - Échantillon des vélos (`bicycle`) : **43 livraisons** (seul groupe sous le seuil, classé `conclusion possible: NON`).
  - Échantillons des autres véhicules : `electric scooter` ($n = 3\ 794$), `motorcycle` ($n = 26\ 285$), `scooter` ($n = 11\ 400$).
  - Échantillons de trafic : `Low` ($n = 14\ 095$), `Medium` ($n = 10\ 023$), `High` ($n = 4\ 046$), `Jam` ($n = 12\ 950$).
- **Interprétation :** Les conclusions relatives aux vélos doivent être considérées avec réserve jusqu'à l'acquisition d'un historique plus étoffé.

---

### Figure 9 : `V13_tableau_de_bord_kpi.png`

```text
+-------------------------------------------------------------------------------+
| V13 - Tableau de bord de synthèse des 12 KPI clés                             |
|                                                                               |
|  Conservation des données (K11) | ██████████████████████████████████ 91.1 %   |
|  Temps perdu total (K4)         | ████████████████████████████ 114 jours      |
|  Taux de retard (K2)            | ████████████████ 45.6 %                     |
|  Durée médiane (K1)             | █████████ 26.0 min                          |
|  Durée moyenne                  | █████████ 26.3 min                          |
|  Trafic dense vs fluide (K7)    | ███ 9.9 min                                 |
|  Écart pire / meilleur crén.(K6)| ███ 9.2 min                                 |
|  Minutes perdues / livr. (K3)   | █ 3.94 min                                  |
|  Météo sévère vs favorable (K8) | █ 2.9 min                                   |
|  Groupes non concluants (K12)   | ▏ 1 groupes                                 |
|  Minutes ajoutées par km (K9)   | ▏ 0.54 min/km                               |
|  Corrélation distance/durée     | ▏ 0.32 coef.                                |
+-------------------------------------------------------------------------------+
```

- **Question traitée :** Quelle est la synthèse consolidée des métriques du réseau ?
- **Valeurs réelles représentées (issues de `kpi.csv`) :**
  1. *Conservation des données (K11) :* **91,1 %**
  2. *Temps perdu total (K4) :* **114 jours** ($163\ 579\text{ min} / 1\ 440$)
  3. *Taux de retard (K2) :* **45,6 %**
  4. *Durée médiane (K1) :* **26,0 min**
  5. *Durée moyenne :* **26,3 min**
  6. *Trafic dense vs fluide (K7) :* **9,9 min**
  7. *Écart pire / meilleur créneau (K6) :* **9,2 min**
  8. *Minutes perdues par livraison (K3) :* **3,94 min**
  9. *Météo sévère vs favorable (K8) :* **2,9 min**
  10. *Groupes non concluants (K12) :* **1 groupe**
  11. *Minutes ajoutées par km (K9) :* **0,54 min/km**
  12. *Corrélation distance / durée :* **0,32**

---

### Figure 10 : `V14_gains_par_scenario.png`

```text
+-------------------------------------------------------------------------------+
| V14 - Gain potentiel estimé par scénario d'action (KPI K10)                   |
|                                                                               |
|  Planifier hors congestion      | ████████████████████ 113 641 min (+8.78/liv)|
|  Adapter véhicule à distance    | █████████████████ 99 173 min (+2.39/liv)    |
|  Marge par mauvais temps        | ██████████ 59 733 min (+2.88/liv)           |
|  Décaler hors heures de pointe  | █████ 29 994 min (+1.88/liv)                |
|                                 +-------------------------------------------- |
|                                 0         30 000    60 000    90 000   120 000|
+-------------------------------------------------------------------------------+
```

- **Question traitée :** Quels sont les gains mesurés pour chaque décision opérationnelle ?
- **Valeurs réelles représentées (issues de `scenarios.csv`) :**
  1. **Planifier hors des zones congestionnées :**
     - Livraisons concernées : **12 950**
     - Durée actuelle : **31,19 min**
     - Gain unitaire : **+8,78 min / livraison**
     - Gain total cumulé : **113 641 minutes**
  2. **Adapter le véhicule au type de trajet :**
     - Livraisons concernées : **41 522**
     - Durée actuelle : **26,32 min**
     - Gain unitaire : **+2,39 min / livraison**
     - Gain total cumulé : **99 173 minutes**
  3. **Prévoir une marge par mauvais temps :**
     - Livraisons concernées : **20 747**
     - Durée actuelle : **26,93 min**
     - Gain unitaire : **+2,88 min / livraison**
     - Gain total cumulé : **59 733 minutes**
  4. **Décaler les livraisons hors des pointes :**
     - Livraisons concernées : **15 966**
     - Durée actuelle : **27,58 min**
     - Gain unitaire : **+1,88 min / livraison**
     - Gain total cumulé : **29 994 minutes**

---

## 4. Synthèse et Cohérence Globale

### Alignement des sources et des chiffres
- Toutes les figures exportées dans `figures/` correspondent rigoureusement aux tableaux exportés dans `outputs/tables/` (`kpi.csv`, `scenarios.csv`, `hypotheses.csv`, `effectifs_groupes.csv`).
- La variable de temps d'attente en cuisine (`attente_min`) présente une corrélation nulle ($r = -0{,}010$) : l'hypothèse H5 est formellement déclarée **infirmée** dans l'ensemble des livrables.
- L'analyse temporelle repose sur les tranches horaires de la journée et l'indicateur binaire de week-end, sans segmentation par jour calendaire individuel.
