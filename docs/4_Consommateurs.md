# 4 – Consommateurs de la base sirene.db (Elasticsearch & Data.gouv)

## 1. Objectif de cette section

Cette section décrit les deux principaux consommateurs de la base `sirene.db` générée par l’ETL :
- le pipeline d’indexation **Elasticsearch**,
- le pipeline d’export **Data.gouv**.

L’objectif est d’expliquer **comment** et **quand** ils utilisent `sirene.db`, sans entrer dans la logique interne de leurs propres DAGs (qui feront l’objet d’un fichier dédié si nécessaire).

---

## 2. Consommateur 1 – Indexation Elasticsearch

**DAG déclenché par** : `extract_transform_load_db` (ETL SIRENE)  
**Déclenchement** : via `TriggerDagRunOperator` en fin d’ETL  
**Source consommée** :  
- `sirene_<date>.db.gz` depuis MinIO (`sirene/database/`)

### Vue d’ensemble du rôle Elasticsearch
1. Récupère la base SQLite `sirene.db` compressée depuis MinIO.  
2. Décompresse puis lit les tables SIRENE, RNE intégrées et enrichissements secondaires.  
3. Alimente les index Elasticsearch (entreprises, établissements, dirigeants, attributs dérivés).  
4. Met à jour les index existants ou les reconstruit selon la politique définie.  

> Le DAG Elasticsearch n’est jamais planifié directement : il dépend exclusivement du succès de l’ETL.

---

## 3. Consommateur 2 – Export Data.gouv

**DAG principal** : `publish_files_in_data_gouv`  
**Planification** : quotidienne (selon son propre cron)  
**Source consommée** :  
- dernière base SQLite publiée dans MinIO (`sirene/database/`)

### Vue d’ensemble du rôle Data.gouv
1. Télécharge depuis MinIO la dernière archive `sirene_<date>.db.gz` mise à disposition par l’ETL.  
2. Décompresse la base SQLite.  
3. Génère plusieurs fichiers tabulaires (UL, établissements, administrations, etc.).  
4. Comprime les fichiers générés.  
5. Dépose les fichiers dans MinIO.  
6. Publie les fichiers sur **data.gouv.fr** via l’API dédiée.  

> Data.gouv ne consomme jamais les données brutes SIRENE/RNE directement, uniquement la base consolidée `sirene.db`.

---

## 4. Chronologie et dépendances

1. L’ETL SIRENE publie `sirene_<date>.db.gz` dans MinIO.  
2. Le DAG Elasticsearch est déclenché immédiatement après l’upload.  
3. Le DAG Data.gouv exécuté quotidiennement consomme la **dernière** base publiée.  

Ces pipelines aval sont donc **totalement dépendants** du succès de l’ETL.

---

## 5. Points à creuser plus tard

- Structure exacte des index Elasticsearch  
- Détail des extractions SQL vers CSV pour Data.gouv  
- Modalités de publication Data.gouv (API, dataset, resource-id, refresh policy)  
- Stratégies de versionning dans MinIO pour les exports  
- Conditions d’échec ou de fallback en cas d’indisponibilité de `sirene.db`
