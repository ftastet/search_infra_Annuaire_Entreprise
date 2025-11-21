# 4 – Consommateurs de la base sirene.db (Elasticsearch & Data.gouv)

## 1. Objectif de cette section

Cette section décrit les deux principaux consommateurs de la base `sirene.db` générée par l’ETL :
- le pipeline d’indexation **Elasticsearch**,
- le pipeline d’export **Data.gouv**.

L’objectif est d’expliquer **comment** et **quand** ils utilisent `sirene.db`, sans entrer dans la logique interne fine de chaque DAG.

---

## 2. Consommateur 1 – Indexation Elasticsearch

**DAG déclenché par** : `extract_transform_load_db` (ETL SIRENE)  
**Déclenchement** : via `TriggerDagRunOperator` en fin d’ETL  
**Planification** : aucune (`schedule_interval=None`)  
**Source consommée** :  
- dernière archive `sirene_<date>.db.gz` depuis MinIO (`sirene/database/`)

### Vue d’ensemble du rôle Elasticsearch

1. Récupère la base SQLite `sirene_<date>.db.gz` depuis MinIO.  
2. Décompresse l’archive et lit les tables SIRENE, RNE intégrées et enrichissements secondaires.  
3. Construit ou met à jour les index Elasticsearch (entreprises, établissements, dirigeants, etc.).  
4. Met à jour l’alias Elasticsearch pour pointer vers les index fraîchement reconstruits.  
5. Génère puis met à jour le sitemap (tâches `create_sitemap` et `update_sitemap`).  
6. Effectue les actions de post-traitement suivantes :
   - déclenchement d’un DAG de **snapshot** (en environnement distant) ou **flush du cache Redis** selon l’environnement,  
   - synchronisation d’un fichier de suivi/métadonnées des sources,  
   - exécution de tests API E2E,  
   - envoi d’une notification finale.

> Le DAG Elasticsearch n’est jamais planifié directement : il s’exécute uniquement lorsqu’il est déclenché par l’ETL.

---

## 3. Consommateur 2 – Export Data.gouv

**DAG principal** : `publish_files_in_data_gouv`  
**Planification** : `0 17 * * *` (tous les jours à 17h00)  
**Source consommée** :  
- dernière base SQLite publiée dans MinIO (`sirene/database/`)

### Vue d’ensemble du rôle Data.gouv

1. Vérifie l’environnement d’exécution via une tâche `check_if_prod` :
   - si l’environnement n’est **pas** `prod`, le pipeline est court-circuité,
   - si l’environnement est `prod`, le pipeline continue.  
2. Télécharge depuis MinIO la dernière archive `sirene_<date>.db.gz` mise à disposition par l’ETL.  
3. Décompresse la base SQLite.  
4. Génère plusieurs fichiers tabulaires (unités légales, établissements, administrations, etc.).  
5. Comprime les fichiers générés.  
6. Dépose les fichiers dans MinIO.  
7. Publie les fichiers sur **data.gouv.fr** via l’API dédiée.  
8. Nettoie le dossier temporaire.

> En pratique, le DAG Data.gouv ne réalise l’export complet que lorsque l’environnement est `prod`.

---

## 4. Chronologie et dépendances

1. L’ETL SIRENE publie `sirene_<date>.db.gz` dans MinIO.  
2. Le DAG Elasticsearch est déclenché immédiatement après l’upload et reconstruit les index.  
3. Le DAG Data.gouv, exécuté quotidiennement, consomme la **dernière** base publiée, uniquement en environnement de production.

Ces pipelines aval sont donc **totalement dépendants** du succès de l’ETL et de la disponibilité de `sirene.db`.

---

## 5. Points à creuser plus tard

- Structure exacte des index Elasticsearch (mapping, alias, stratégies de rotation)  
- Détail des extractions SQL vers CSV pour Data.gouv  
- Modalités précises de publication Data.gouv (API, dataset, resource-id, refresh policy)  
- Stratégies de versionning dans MinIO pour les exports  
- Conditions d’échec ou de fallback en cas d’indisponibilité de `sirene.db` ou d’échec des tests E2E
