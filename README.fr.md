# 📊 Tableau de Bord de Santé Financière — Power BI (PME)

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-217346?style=for-the-badge&logo=microsoft&logoColor=white)
![Power Query](https://img.shields.io/badge/Power%20Query-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)
![Star Schema](https://img.shields.io/badge/Data%20Modeling-Star%20Schema-blue?style=for-the-badge)
![ETL](https://img.shields.io/badge/ETL-Data%20Cleaning-orange?style=for-the-badge)
![Forecasting](https://img.shields.io/badge/Forecasting-Time%20Series-9cf?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-lightgrey?style=for-the-badge)

Tableau de bord Power BI complet permettant à une PME de suivre sa santé financière (états financiers, rentabilité, prévisions budgétaires) à partir d'un jeu de données trimestriel multi-entreprises.

---

## 🎯 Objectif du projet

Ce projet répond à un besoin métier concret : donner à un dirigeant de PME une vision claire et actionnable de sa situation financière, répartie sur **trois axes** :

1. **États financiers** — les chiffres clés (revenu, dépenses, résultat net, trésorerie) et leur évolution.
2. **Rentabilité et performance** — marges, ROE, ROA, comparaison entre entreprises.
3. **Prévision et budgétisation** — anticipation du revenu du trimestre suivant pour préparer les décisions budgétaires.

## 🗂️ Jeu de données

- **2 000 lignes** × **30 colonnes**
- **50 entreprises** (`Company_ID`), suivies **trimestriellement de 2015 à 2024** (`Year`, `Quarter`)
- Aucune valeur manquante

Le fichier source contient, au-delà des champs financiers classiques, des colonnes destinées à un tout autre type de tâche (scores de sentiment, buzz réseaux sociaux, indicateurs macroéconomiques, flags de fraude/anomalie). Une partie de ce projet a donc consisté à **identifier ces colonnes hors périmètre et à justifier leur exclusion**, plutôt qu'à les garder par défaut.

Un contrôle statistique rapide a notamment montré que `Inflation_Rate`, `Interest_Rate`, `Exchange_Rate` et `Global_Economic_Score` varient de façon aléatoire **par entreprise**, y compris pour une même année/trimestre (écart-type ≈ 2 au sein d'un même trimestre). Ce ne sont donc pas de vrais indicateurs macroéconomiques partagés, mais du bruit par ligne — d'où leur exclusion.

## 🧹 Nettoyage des données (Power Query)

### Colonnes supprimées (13)
| Catégorie | Colonnes |
|---|---|
| Données de marché boursier | `Stock_Price`, `Volume_Traded` |
| Données alternatives / sentiment | `News_Sentiment_Score`, `Social_Media_Buzz`, `Sector_Trend_Index` |
| Bruit statistique (validé par analyse) | `Global_Economic_Score`, `Inflation_Rate`, `Exchange_Rate`, `Interest_Rate` |
| Hors périmètre (fraude/anomalie) | `Audit_Flag`, `Fraud_Flag`, `Market_Shock_Flag`, `Policy_Change_Flag`, `Target_Anomaly_Class` |

### Colonnes conservées
`Year`, `Quarter`, `Company_ID`, `Revenue`, `Expenses`, `Operating_Income`, `Net_Income`, `Assets`, `Liabilities`, `Equity`, `Cash_Flow`, `EPS`, `ROE`, `ROA`, `Debt_to_Equity`, `Target_Revenue_Next_Qtr` (réservée à l'onglet prévision).

### Points d'attention sur le typage
- `ROE` et `ROA` sont **déjà** exprimés en points de pourcentage (`ROE = Net_Income / Equity × 100`). Le type *Pourcentage* natif de Power BI aurait multiplié la valeur par 100 une seconde fois → utilisation d'un **Nombre décimal** avec le format personnalisé `0.00"%"`.
- `Debt_to_Equity` : nombre décimal, 2 décimales, suffixe optionnel `x`.
- Champs monétaires en **Nombre décimal fixe**, format Devise.

## 🧩 Modélisation — Schéma en étoile

Une table de faits (`FactFinancials`) et deux dimensions (`DimDate`, `DimCompany`) : la structure minimale suffisante pour un vrai schéma en étoile, adaptée au contenu réel du jeu de données.

**Étapes clés :**
- Création d'une colonne `DateKey = #date(Year, (Quarter-1)*3+1, 1)` pour transformer `Year` + `Quarter` en date exploitable par le *time intelligence* et le moteur de prévision natif.
- `DimDate` : marquée comme table de dates, avec `Quarter Label` (`"Q" & Quarter & " " & Year`) et `YearQuarterSort` pour un tri correct des axes.
- `DimCompany` : dimension entreprise dédiée.
- Relations 1 → plusieurs, sens unique, entre chaque dimension et la table de faits.

**Point de modélisation important :** `Assets`, `Liabilities` et `Equity` sont des **photographies de bilan**, pas des flux additifs — les sommer sur plusieurs trimestres n'a pas de sens. Des mesures basées sur `LASTNONBLANK` ont été utilisées pour obtenir la valeur du **dernier trimestre disponible** plutôt qu'un `SUM` brut.

## 🧮 Mesures DAX principales

```dax
Total Revenue = SUM(FactFinancials[Revenue])
Net Income = SUM(FactFinancials[Net_Income])
Net Profit Margin % = DIVIDE([Net Income], [Total Revenue])
Operating Margin % = DIVIDE(SUM(FactFinancials[Operating_Income]), [Total Revenue])

Latest Assets =
CALCULATE(SUM(FactFinancials[Assets]), LASTNONBLANK(DimDate[DateKey], [Total Revenue]))

Latest Equity =
CALCULATE(SUM(FactFinancials[Equity]), LASTNONBLANK(DimDate[DateKey], [Total Revenue]))

Forecast Revenue (3Q Avg) =
AVERAGEX(
    DATESINPERIOD(DimDate[DateKey], LASTDATE(DimDate[DateKey]), -2, QUARTER),
    [Total Revenue]
)

Forecast Variance % =
DIVIDE(
    SUM(FactFinancials[Target_Revenue_Next_Qtr]) - [Forecast Revenue (3Q Avg)],
    SUM(FactFinancials[Target_Revenue_Next_Qtr])
)
```

`Target_Revenue_Next_Qtr` a été délibérément conservée pour construire une vraie comparaison prévision-vs-réel du trimestre suivant, sans devoir inventer un objectif artificiel.

## 📑 Structure du rapport — 3 onglets

### Onglet 1 · États financiers
Cartes KPI (Revenu, Dépenses, Résultat net, Trésorerie), graphique en cascade Revenu → Résultat net, histogramme empilé Actif vs Passif/Capitaux propres, graphique en aires de trésorerie avec ligne zéro, table détaillée avec barres de données.

### Onglet 2 · Rentabilité & performance
Cartes KPI (marge, marge opérationnelle, ROE, ROA), nuage de points ROE vs ROA (bulle = revenu, couleur = entreprise), jauge marge vs objectif de référence, classement des entreprises par résultat net, matrice thermique ROE par entreprise × année.

### Onglet 3 · Prévision & budgétisation
Cartes KPI (prévision, réel, écart), courbe de revenu avec **fonction de prévision native Power BI** (Analyse → Prévision), graphique combiné réel vs `Target_Revenue_Next_Qtr`, histogramme d'écart de prévision (%), table exportable pour le suivi budgétaire.

Chaque onglet est complété par des **infobulles** (au niveau des champs et en page de rapport) pour enrichir le contexte sans surcharger les visuels principaux.

## ⏱️ Répartition du temps (≈ 12–15 h)

| Tâche | Temps |
|---|---|
| Power Query + schéma en étoile + relations | 2–3 h |
| Mesures DAX (marges, ratios, prévision, écart) | 2 h |
| Onglet 1 (états financiers) | 2–3 h |
| Onglet 2 (rentabilité) | 2–3 h |
| Onglet 3 (prévision) + configuration prévision native | 2 h |
| Infobulles + mise en forme + relecture | 2 h |

## 🛠️ Compétences démontrées

- Nettoyage et sélection critique de données (Power Query / ETL)
- Modélisation dimensionnelle (schéma en étoile, relations, table de dates)
- Écriture de mesures DAX (agrégations, `LASTNONBLANK`, `DATESINPERIOD`, `DIVIDE`)
- Choix de visuels adaptés au message (cascade, heatmap, jauge, combo chart)
- Conception UX de rapport multi-onglets orientée décision métier
- Utilisation de la fonction de prévision native de Power BI

## 🚀 Utilisation

1. Ouvrir le fichier `.pbix` dans Power BI Desktop.
2. Rafraîchir les données si nécessaire (Accueil → Actualiser).
3. Naviguer entre les trois onglets via la barre de navigation en bas du rapport.

## 📄 Licence

Projet réalisé à des fins de démonstration / portfolio.
