# 2 – Acquisition RNE (flux + stock + base quotidienne)

## 1. Objectif du module RNE

Le module RNE récupère et consolide :
- le **stock complet** RNE (archive annuelle),
- les **flux quotidiens** (SIREN modifiés),
- et produit une **base SQLite consolidée** (stock + flux applicables).

Toutes les données sont déposées dans MinIO (`rne/stock/`, `rne/flux/`, `rne/database/`) pour alimenter l’ETL Sirene et la construction de `sirene.db`.

---

## 2. DAG `get_rne_stock` (téléchargement du stock annuel)

**Chemin** : `workflows/data_pipelines/rne/stock/`  
**dag_id** : `get_rne_stock`  
**Schedule** : manuel (pas de cron)  
**Source** : archive du stock RNE  
**Sortie** : MinIO `rne/stock/`

### Vue d’ensemble
1. Préparation / nettoyage du dossier temporaire
2. Téléchargement et extraction du stock
3. Upload vers MinIO (`rne/stock/`)
4. Pas de nettoyage final (le DAG s’arrête après l’upload)

> Le stock n’est pas téléchargé automatiquement : il est exécuté manuellement à chaque nouvelle publication INPI.

---

## 3. DAG `get_flux_rne` (flux quotidiens)

**Chemin** : `workflows/data_pipelines/rne/flux/`  
**dag_id** : `get_flux_rne`  
**Schedule** : `0 1 * * *` (tous les jours à 01h00)  
**Source** : API RNE (SIREN modifiés du jour)  
**Sortie** : MinIO `rne/flux/`

### Vue d’ensemble
1. Préparation / nettoyage du dossier temporaire  
2. Appel API RNE  
3. Enregistrement des fichiers du jour  
4. Upload vers MinIO (`rne/flux/`)  
5. Nettoyage

> Les notifications sont gérées via les callbacks Airflow.

---

## 4. DAG `fill_rne_database` (construction quotidienne de la base consolidée)

**Chemin** : `workflows/data_pipelines/rne/database/`  
**dag_id** : `fill_rne_database`  
**Schedule** : `0 2 * * *`  
**Sources** :
- stock RNE (MinIO `rne/stock/`)
- flux RNE (MinIO `rne/flux/`)
- dernière date traitée via `latest_rne_date.json` (MinIO `rne/database/`)

**Sorties** :
- `rne_<date>.db.gz` dans `rne/database/`  
- fichier `latest_rne_date.json`

### Vue d’ensemble
1. Préparation du dossier temporaire  
2. Récupération de la date la plus récente (`latest_rne_date.json`) et
   téléchargement de la base précédente si besoin
3. Téléchargement du stock et des flux depuis MinIO (le stock est ignoré si une
   date existe déjà)
4. Agrégation des JSON → création de la base SQLite
5. Déduplication interne
6. Validation de volumétrie
7. Upload vers MinIO (`rne/database/`) avec nommage selon la dernière date de
   flux traitée
8. Génération / mise à jour de `latest_rne_date.json` (date = dernier flux + 1
   jour)
9. Nettoyage

> Le dernier flux disponible (jour même) peut être ignoré pour éviter un flux incomplet.

---

## 5. Chronologie et dépendances

1. `get_rne_stock` (manuel) → fournit le stock RNE  
2. `get_flux_rne` (quotidien) → fournit les flux journaliers  
3. `fill_rne_database` (quotidien) :
   - lit stock + flux (saute le stock après la première exécution si une date existe déjà)
   - génère la base consolidée dans `rne/database/`  
   - met à jour `latest_rne_date.json`

> **Important :** après la première exécution, `fill_rne_database` réutilise la date enregistrée dans `latest_rne_date.json`.  
> Il **ignore complètement le stock** et ne traite **que les flux depuis cette date**.

Cette base RNE est ensuite consommée par l’ETL Sirene dans `3_ETL_SIRENE.md`.

---

## 6. Points à creuser plus tard

- Logique exacte d’utilisation de `latest_rne_date.json`  
- Gestion de l’exclusion du flux du jour  
- Nommage et versionning des archives RNE  
- Détails des règles de déduplication interne  
- Volumétrie attendue (stock, flux)  
