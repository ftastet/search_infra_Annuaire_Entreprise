# 2 – Acquisition RNE (flux + stock + base quotidienne)

## 1. Objectif du module RNE

Le module RNE récupère et consolide :
- le **stock complet** RNE (archive annuelle),
- les **flux quotidiens** (SIREN modifiés),
- et produit une **base SQLite consolidée** (stock + tous les flux applicables).

Toutes les données sont déposées dans MinIO (`rne/stock/`, `rne/flux/`, `rne/database/`) pour alimenter l’ETL Sirene et la création de `sirene.db`.

---

## 2. DAG `get_rne_stock` (téléchargement du stock annuel)

**Chemin** : `workflows/data_pipelines/rne/stock/`  
**dag_id** : `get_rne_stock`  
**Schedule** : manuel (pas de cron)  
**Source** : archive RNE (stock complet)  
**Sortie** : MinIO `rne/stock/`

### Vue d’ensemble
1. Préparation du dossier temporaire  
2. Téléchargement et extraction du stock  
3. Upload vers MinIO (`rne/stock/`)  
4. Nettoyage

> Le stock n’est pas téléchargé automatiquement : il est lancé manuellement lors des nouvelles publications INPI.

---

## 3. DAG `get_flux_rne` (flux quotidiens)

**Chemin** : `workflows/data_pipelines/rne/flux/`  
**dag_id** : `get_flux_rne`  
**Schedule** : `0 1 * * *` (tous les jours à 01h00)  
**Source** : API RNE (SIREN modifiés du jour)  
**Sortie** : MinIO `rne/flux/`

### Vue d’ensemble
1. Préparation / nettoyage du dossier temporaire  
2. Appel API RNE (récupération des SIREN modifiés)  
3. Enregistrement des fichiers du jour  
4. Upload vers MinIO (`rne/flux/`)  
5. Nettoyage

> Les notifications sont gérées via les callbacks Airflow.

---

## 4. DAG `fill_rne_database` (construction quotidienne de la base RNE)

**Chemin** : `workflows/data_pipelines/rne/database/`  
**dag_id** : `fill_rne_database`  
**Schedule** : `0 2 * * *` (tous les jours à 02h00)  
**Sources** :
- stock RNE (MinIO `rne/stock/`)
- flux RNE (MinIO `rne/flux/`)

**Sorties** :
- base consolidée : `rne_<date>.db.gz` (MinIO `rne/database/`)  
- fichier `latest_rne_date.json`

### Vue d’ensemble
1. Préparation du dossier temporaire  
2. Téléchargement du stock et des flux depuis MinIO  
3. Agrégation des JSON → création de la base SQLite  
4. Déduplication interne  
5. Validation de volumétrie  
6. Upload vers MinIO (`rne/database/`) avec nommage par date  
7. Génération / mise à jour de `latest_rne_date.json`  
8. Nettoyage

> Le dernier flux disponible (jour même) peut être ignoré pour éviter un flux incomplet.

---

## 5. Chronologie et dépendances

1. `get_rne_stock` (manuel) → fournit le stock complet  
2. `get_flux_rne` (quotidien) → fournit les flux journaliers  
3. `fill_rne_database` (quotidien) :
   - lit stock + flux  
   - génère la base consolidée dans `rne/database/`  
   - fournit `latest_rne_date.json`

Cette base RNE est ensuite consommée par l’ETL Sirene dans `3_ETL_SIRENE.md`.

---

## 6. Points à creuser plus tard

- Structure exacte de `latest_rne_date.json`  
- Gestion de l’exclusion du flux N (jour même)  
- Nommage des bases RNE (`rne_<date>.db.gz`)  
- Détails des règles de déduplication interne  
- Volumétrie attendue des flux et du stock  
