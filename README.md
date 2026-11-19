# E-Commerce Customer Analytics & Markovian Churn Pipeline

## Présentation du Projet
Ce projet met en œuvre une architecture analytique de bout en bout appliquée aux données transactionnelles du secteur de la vente au détail en ligne. L'objectif principal est de structurer un pipeline ELT (Extract, Load, Transform) robuste, d'évaluer la valeur vie client à travers une segmentation comportementale RFM (Récence, Fréquence, Montant), et de modéliser les dynamiques de rétention et d'attrition (churn) à l'aide de chaînes de Markov à temps discret et de la théorie des graphes.

---

## Source des Données
Les analyses reposent sur le jeu de données public **Online Retail II** (issu de l'UCI Machine Learning Repository). Ce jeu de données regroupe l'historique des transactions d'une entreprise de commerce électronique basée au Royaume-Uni, spécialisée dans la vente de cadeaux et d'articles de décoration (clientèle mixte B2C et grossistes B2B).

* **Période couverte :** Du 1er décembre 2009 au 9 décembre 2011.
* **Volumétrie initiale :** 1 067 371 enregistrements et 8 variables (`Invoice`, `StockCode`, `Description`, `Quantity`, `InvoiceDate`, `Price`, `Customer ID`, `Country`).

---

## Architecture Méthodologique

### 1. Pipeline ELT et Préparation des Données
Le traitement initial vise à garantir l'intégrité statistique des indicateurs financiers et comportementaux :
* **Gestion des valeurs manquantes :** Exclusion des transactions non rattachées à un identifiant client unique (`Customer ID` manquant sur 243 007 lignes) ainsi que des descriptions absentes (4 382 lignes).
* **Filtrage des anomalies et annulations :** Suppression des quantités négatives correspondant aux retours de marchandises et annulations de commandes, ainsi que l'examen des lignes à prix unitaire nul (71 enregistrements).
* **Standardisation temporelle et création de variables :** Conversion des horodatages (`InvoiceDate`) au format `datetime64`, extraction temporelle mensuelle/semestrielle et calcul du chiffre d'affaires par ligne (`CA = Quantity * Price`).
* **Volumétrie consolidée :** 805 620 transactions valides retenues, exportées dans `online_retail_cleaned.csv`.

### 2. Analyse Exploratoire des Données (EDA)
L'analyse descriptive met en évidence la structure macroéconomique de l'activité :
* **Chiffre d'affaires global :** 17 743 429,18 £ générés sur l'ensemble de la période nettoyée.
* **Saisonnalité :** Concentration marquée des volumes de ventes et du chiffre d'affaires au second semestre (pics d'activité en juillet, septembre et novembre) préparant les fêtes de fin d'année.
* **Répartition géographique :** Forte concentration sur le marché domestique (Royaume-Uni avec ~14,72 M£), suivi par l'Irlande (EIRE avec ~621,6 k£), les Pays-Bas (~554,2 k£), l'Allemagne (~431,3 k£) et la France (~355,3 k£).

### 3. Segmentation Comportementale RFM
Agrégation des 805 620 transactions autour des **5 881 clients uniques** (date de référence fixée au 11 décembre 2011). Chaque dimension est discrétisée en quintiles (scores de 1 à 5) afin d'établir une typologie en 6 segments métiers :

| Segment | Nombre de clients | Part du portefeuille | Récence moyenne (jours) | Fréquence moyenne (achats) | Panier total moyen (£) |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Champions** | 1 515 | 25,8 % | 26,6 | 15,4 | 8 125,3 £ |
| **Clients fidèles** | 1 193 | 20,3 % | 92,6 | 5,6 | 2 216,2 £ |
| **Clients à risque** | 820 | 13,9 % | 368,4 | 4,8 | 1 857,2 £ |
| **Clients prometteurs** | 395 | 6,7 % | 131,4 | 1,4 | 973,3 £ |
| **Nouveaux clients** | 431 | 7,3 % | 36,6 | 1,4 | 523,9 £ |
| **Clients perdus** | 1 527 | 26,0 % | 475,4 | 1,2 | 430,0 £ |

### 4. Modélisation des Transitions par Chaînes de Markov et Théorie des Graphes
Pour dépasser la vision statique du modèle RFM, le portefeuille est évalué de manière séquentielle par semestre calendaire. Une chaîne de Markov modélise la probabilité $P(X_{t+1} = j \mid X_t = i)$ qu'un client actif dans le segment $i$ au semestre $t$ transite vers le segment $j$ ou devienne inactif (`Churn`) au semestre $t+1$.

#### Matrice des probabilités de transition observées :

| Segment de départ ($t$) \ État suivant ($t+1$) | Champions | Clients fidèles | Nouveaux clients | Clients à risque | Clients perdus | Clients prometteurs | Churn (Inactivité) |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Champions** | 28,0 % | 12,0 % | 4,1 % | 10,9 % | 7,3 % | 3,1 % | **34,6 %** |
| **Clients fidèles** | 15,7 % | 11,1 % | 2,5 % | 12,2 % | 4,4 % | 1,9 % | **52,1 %** |
| **Nouveaux clients** | 7,9 % | 3,3 % | 5,4 % | 3,6 % | 9,2 % | 4,1 % | **66,5 %** |
| **Clients à risque** | 13,8 % | 11,1 % | 2,4 % | 11,7 % | 3,5 % | 1,3 % | **56,2 %** |
| **Clients perdus** | 7,3 % | 3,8 % | 5,8 % | 3,7 % | 10,5 % | 3,6 % | **65,3 %** |
| **Clients prometteurs** | 8,3 % | 4,6 % | 3,5 % | 3,7 % | 12,3 % | 4,6 % | **63,1 %** |

La représentation sous forme de graphe orienté pondéré (`NetworkX`) permet d'isoler visuellement les sous-réseaux de fidélisation (échanges récurrents au sein de la triade *Champions – Clients fidèles – Clients à risque*) et de quantifier l'attraction exercée par l'état d'inactivité.

---

## Principaux Enseignements et Recommandations Métier

1. **Vulnérabilité critique de la phase d'acquisition (Onboarding) :**
   * *Constat :* 66,5 % des *Nouveaux clients* et 63,1 % des *Clients prometteurs* deviennent inactifs dès le semestre suivant leur premier achat.
   * *Recommandation :* Déployer des séquences d'activation automatisées post-premier achat (incitations financières sur la deuxième commande sous 30 jours, recommandations personnalisées) afin de favoriser la transition vers le segment des *Clients fidèles*.

2. **Potentiel de réactivation des Clients à risque :**
   * *Constat :* Bien que 56,2 % des *Clients à risque* basculent vers le churn au semestre suivant, près d'un quart d'entre eux (24,9 %) rebondit positivement vers les segments *Champions* (13,8 %) ou *Clients fidèles* (11,1 %).
   * *Recommandation :* Concentrer le budget des campagnes de rétention (offres de reconquête ciblées) sur ce segment spécifique dont le panier historique moyen (1 857,2 £) garantit un retour sur investissement élevé.

3. **Protection du noyau de valeur (Champions) :**
   * *Constat :* Les *Champions* génèrent en moyenne 8 125,3 £ par client et présentent le taux de rétention directe le plus fort (28,0 % restent Champions, 12,0 % deviennent Clients fidèles), mais subissent néanmoins une érosion de 34,6 % par semestre.
   * *Recommandation :* Instaurer un programme relationnel privilégié (conditions commerciales B2B dédiées, alertes préventives dès que la récence dépasse 30 jours) pour verrouiller ce segment stratégique.

---

## Structure du Répertoire

```text
├── data/
│   └── online_retail_II.csv          # Jeu de données transactionnel brut (source)
├── analyse_e_commerce.ipynb          # Notebook principal (Pipeline ELT, EDA, RFM, Markov & Graphes)
├── online_retail_cleaned.csv         # Données transactionnelles nettoyées exportées
├── rfm_segmentation.csv              # Table d'agrégation et de scoring RFM par client
└── README.md                         # Documentation technique et métier du projet
```

---

## Environnement Technique et Dépendances
* **Langage :** Python 3.x
* **Bibliothèques principales :**
  * `pandas` & `numpy` : Manipulation matricielle, nettoyage et agrégation temporelle.
  * `matplotlib` & `seaborn` : Visualisations statistiques et matrices de chaleur (heatmaps).
  * `networkx` : Modélisation mathématique et représentation spatiale des graphes de transition.
* **Outils :** Visual Studio Code, Jupyter Notebook, Git.

---

## Instructions d'Exécution

1. Cloner le dépôt localement :
   ```bash
   git clone https://github.com/dieuhuongdoan/ecommerce-customer-analytics.git
   cd ecommerce-customer-analytics
   ```

2. Installer les dépendances Python requises :
   ```bash
   pip install pandas numpy matplotlib seaborn networkx
   ```

3. Assurer la présence du fichier `online_retail_II.csv` dans le répertoire `data/`, puis exécuter le notebook `analyse_e_commerce.ipynb` via Jupyter ou VS Code.

---

## Auteure
**Dieu Huong DOAN**  
Master 1 Data Science & Management de l'Innovation  
Université Jean Monnet — Saint-Étienne, France
