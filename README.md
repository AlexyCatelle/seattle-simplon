# Seattle – Prédiction de la consommation d'énergie et des émissions de CO₂

Projet de groupe (Groupe X : Prénom NOM, Prénom NOM, Prénom NOM).

## Contexte
La ville de Seattle vise la neutralité carbone en 2050. Les relevés de consommation des bâtiments non résidentiels sont coûteux : on cherche à **prédire la consommation totale d'énergie (`SiteEnergyUse(kBtu)`) et les émissions de GES (`TotalGHGEmissions`)** à partir de caractéristiques structurelles (surface, usage, âge, localisation…), et à évaluer l'intérêt de l'**ENERGY STAR Score**.

## Contenu du dépôt
```
.
├── README.md
├── requirements.txt
├── seattle_energy_ml.ipynb      # notebook complet (nettoyage → EDA → modèles → optimisation)
└── data/
    └── 2016-building-energy-benchmarking.csv
```

## Relancer le projet
```bash
git clone <url-du-depot> && cd <dossier>
python -m venv .venv
source .venv/bin/activate          # Windows : .venv\Scripts\activate
pip install -r requirements.txt
jupyter notebook seattle_energy_ml.ipynb   # puis "Run All"
```
Le notebook cherche automatiquement un fichier `*building-energy-benchmarking*.csv` dans le dossier courant ou dans `data/`.

`requirements.txt` :
```
pandas>=2.2
numpy>=1.26
scikit-learn>=1.4
scipy>=1.11
matplotlib>=3.8
seaborn>=0.13
jupyter
```
Durée d'exécution : quelques minutes sur un ordinateur portable récent (la recherche d'hyperparamètres est la partie la plus longue). Tous les tirages aléatoires utilisent `RANDOM_STATE = 42` : les résultats sont reproductibles.

## Démarche
1. **Nettoyage** : restriction aux bâtiments non résidentiels, retrait des relevés signalés aberrants ou non conformes, des cibles nulles/négatives, correction des erreurs de saisie (étages, nombre de bâtiments, quartiers). Chaque décision est justifiée dans le notebook.
2. **Prévention de la fuite de données** : toutes les colonnes de consommation mesurée (EUI, électricité, gaz, vapeur, intensité GES) sont exclues ; seul le raccordement au gaz / à la vapeur (oui/non) est conservé.
3. **EDA** : distributions (cibles passées en log), corrélations de Spearman avec p-values, Kruskal-Wallis, V de Cramér.
4. **Réduction de dimension** : suppression des variables redondantes (|ρ| > 0,9, V de Cramér élevé), regroupement des modalités rares ; PCA testée et comparée.
5. **Modèles comparés** (validation croisée 5 plis) : Dummy, Ridge, KNN, SVR, Random Forest, Gradient Boosting, HistGradientBoosting.
6. **Diagnostic** : écart train/validation, courbes d'apprentissage et de validation.
7. **Optimisation** : `RandomizedSearchCV`, gain mesuré par rapport au modèle par défaut.
8. **ENERGY STAR Score** : comparaison avec/sans sur les mêmes bâtiments et les mêmes plis + test apparié + importance par permutation.
9. **Évaluation finale** sur un jeu de test (20 %) jamais utilisé auparavant.

## Résultats
Scores obtenus avec `RANDOM_STATE = 42`. Les cibles sont modélisées en log(1+y).

| Cible | Modèle retenu | R² CV par défaut → optimisé | R² test (log) | Erreur relative médiane (test) |
|---|---|---|---|---|
| Énergie | Gradient Boosting | 0,744 → 0,752 | 0,745 | 38 % |
| GES | HistGradientBoosting | 0,677 → 0,726 | 0,708 | 47 % |

Les conclusions principales sont les suivantes.

- **Optimisation** : elle réduit fortement le sur-apprentissage du HistGradientBoosting. L'écart entre R² d'entraînement et R² de validation passe de 0,27 à 0,08.
- **ENERGY STAR Score** : sur les bâtiments qui en ont un, il augmente le R² de +0,08 pour l'énergie et de +0,05 pour les GES. Il est cependant coûteux à obtenir, absent pour un tiers des bâtiments et dérivé des consommations mesurées. Nous recommandons donc de ne pas l'exiger.
