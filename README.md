## 📌 Architecture d’automatisation (Data Lineage) <br>
Cette architecture représente la chaîne complète de transformation des données permettant d’automatiser la production :
 - du rapport de chiffre d’affaires (CA par produit + CA total)
 - de la segmentation des vins (premium / ordinaires)<br>
L’ensemble est orchestré par Kestra, qui assure :
 - l’enchaînement des tâches
 - l’exécution des tests de validation
 - la planification mensuelle
 - la gestion des erreurs (logs, retry, alertes, fallback)<br>

## 🟫 1. Zone RAW — Données brutes<br>
* 📂 Sources d’entrée<br>
  - erp.xlsx
  - web.xlsx
  - liaison.xlsx

* 🧪 Tests appliqués<br>
  - Vérification de l’existence des fichiers
  - Vérification que les fichiers ne sont pas vides
  - Vérification du format attendu

* ⚠️ Gestion des erreurs <br>
  - Log d’erreur
  - Notification
  - Retry automatique (3 tentatives)

## 🟧 2. Zone CLEAN — Données nettoyées <br>
* 🏗️ Étape : Ingestion<br>
 - Chargement des fichiers bruts dans l’environnement de traitement (DuckDB + Pandas).<br>
* Tests d’ingestion :<br>
 - Nombre de lignes > 0
 - Colonnes obligatoires présentes
 - Types cohérents (ID, prix, SKU…)

* 🧹 Étape : Nettoyage<br>
  - Suppression des valeurs manquantes sur les colonnes clés
  - Dédoublonnage (ERP, liaison, web)
  - Normalisation des identifiants (minuscules, trim, types)

* Tests de nettoyage :<br>
  - Plus aucun doublon
  - Plus aucune valeur manquante sur les colonnes clés
  - Résultats attendus :(ERP nettoyé : 825 lignes,Liaison nettoyée : 825 lignes,Web nettoyé : 1428 lignes, puis 714 après agrégation)


## 🟩 3. Zone CURATED — Données prêtes pour le métier<br>
* 🔗 Étape : Jointures<br>
* Fusion des données : ERP_clean ↔ liaison_clean ↔ web_agg → full_data
* Tests de jointure :<br>
     - Aucune perte de clés
     - Nombre de lignes final = 714
     - Cohérence des correspondances ERP ↔ Web

* 💰 Étape : Calcul du chiffre d’affaires<br>
   - CA par produit : price × total_sales
   - CA total : somme des CA produits
   - Tests CA :
     *   CA_produit ≥ 0
     *   CA_total > 0
     *   CA_total attendu = 70 568,60 €

* 🥇 Étape : Détection des vins premium
  - Méthode : z-score
  - Vin premium : z-score > 2
  - Vin ordinaire : sinon
  - Tests premium :
     *  Nombre de vins premium = 30
     *  premium + ordinaires = total (714)
     *  Outliers cohérents

## 📊 4. Zone OUTPUT — Fichiers livrables <br>
 * 📦 Fichiers générés(rapport.xlsx,CA par produit,CA total,premium.csv,ordinaire.csv)<br>
 * 🧪 Tests exports:Fichiers générés,Fichiers non vides

CA total dans Excel = 70 568,60 €
-------
## ⚙️ 5. Planification et gestion des erreurs<br>
 * 🕒 Planification<br>
Le workflow est exécuté automatiquement :Tous les 15 du mois à 9h (cron : 0 9 15 * *)

* 🚨 Gestion des erreurs:<br>
    - Retry automatique sur les tâches critiques
    - Logs détaillés
    - Alertes email (succès / échec)
    - Fallback CSV si l’export Excel échoue
    - Arrêt contrôlé du workflow en cas d’anomalie CA

* 🎯 Résultat : une chaîne automatisée, fiable et testée<br>
Cette architecture garantit :
- une qualité de données maîtrisée
- une automatisation complète du reporting
- une détection fiable des vins premium
- une résilience en cas d’erreur (fallback, retry, alertes)
- une traçabilité totale via Kestra <BR>

## 🚨 6. Gestion des erreurs & Alertes
Kestra intègre un système d’alertes à chaque étape critique du pipeline.
L’objectif est simple : détecter, signaler et isoler les erreurs sans bloquer toute la chaîne.

✔ Comment ça fonctionne ?<br>
 * Chaque bloc (CLEAN, FUSION, ANALYSIS, REPORTING) possède un test de validation.
 * Si un test échoue → une alerte dédiée est déclenchée.
 * Le pipeline continue grâce à allowFailure: true.
 * L’erreur apparaît clairement dans l’UI Kestra (carré rouge).
 * Une task d’alerte enregistre un message explicite dans les logs.
