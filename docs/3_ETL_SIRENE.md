# 3 – ETL SIRENE (construction de la base sirene.db)

## 1. Objectif du module ETL SIRENE

L’ETL reconstruit une base SQLite `sirene.db` en combinant :
- les données SIRENE (stock + flux),
- la base consolidée RNE,
- les jeux de données secondaires (liste non exhaustive),
- des contrôles de volumétrie et validations internes.

Le DAG est planifié tous les jours à 05h00 **à partir du 20/08/2025**, avec le rattrapage (catchup) désactivé.  
Le résultat final est déposé dans MinIO (`sirene/database/`) et alimente :
- l’index Elasticsearch,
- les exports Data.gouv,
- les usages internes.

---

## 2. DAG `extract_transform_load_db` (ETL principal)

**Chemin** : `workflows/data_pipelines/etl/`  
**dag_id** : valeur de la variable Airflow `AIRFLOW_ETL_DAG_NAME`  
**Schedule** : `0 5 * * *` (tous les jours à 05h00, sans catchup, start_date=2025-08-20)

### Sources utilisées
- SIRENE stock (`insee/stock/`)
- SIRENE flux (`insee/flux/`)
- Base RNE consolidée (`rne/database/`)
- Jeux de données secondaires (via `DatabaseTableConstructor`)
- Métadonnées internes (ex. date courante / précédente via `determine_sirene_date`)

### Sorties
- Base SQLite compressée `sirene_<date>.db.gz` déposée dans MinIO (`sirene/database/`)
- Fichier de métadonnées `data_source_updates.json` dans `metadata/updates/new/`
- Déclenchement du DAG d’indexation Elasticsearch

---

## 3. Vue d’ensemble du traitement ETL

1. Préparation du répertoire temporaire  
2. Sélection du mois SIRENE à traiter via `determine_sirene_date`  
   - tentative sur le mois courant  
   - bascule automatique sur le mois précédent si les fichiers ne sont pas disponibles  
   - échec si aucun des deux mois n’est disponible  
3. Chargement SIRENE stock (pour le mois choisi)  
4. Application des flux SIRENE  
5. Intégration de la base RNE, incluant :
   - la création des tables de dirigeants (personnes physiques / personnes morales),
   - la copie/ajout de la table d’immatriculation après enrichissement des tables SIRENE  
6. Contrôles de cohérence / volumétrie (SIRENE seule puis SIRENE + RNE)  
7. Enrichissements via les jeux de données secondaires (via `DatabaseTableConstructor`)  
8. Génération de la base SQLite `sirene.db`  
9. Compression et upload vers MinIO (`sirene/database/`), **avec suppression locale de la base et de l’archive** une fois l’envoi terminé  
10. Génération du fichier de métadonnées `data_source_updates.json` dans `metadata/updates/new/`  
11. Nettoyage du répertoire temporaire  
12. Déclenchement du DAG d’indexation Elasticsearch (via `TriggerDagRunOperator`)

> Les notifications, logs et callbacks Airflow sont gérés par la configuration Airflow et ne sont pas détaillés ici.

---

## 4. Rôle des enrichissements secondaires

Les datasets secondaires apportent des attributs complémentaires dans des tables annexes (ex. labels, secteurs spécifiques, jeux administratifs).  
Ils ne modifient pas la mécanique centrale **SIRENE + RNE**, mais enrichissent la base finale.

L’intégration est orchestrée par `DatabaseTableConstructor` dans un bloc dédié de l’ETL, après la consolidation SIRENE/RNE et avant la génération/export de `sirene.db`.

---

## 5. Chronologie et dépendances

1. Acquisition SIRENE (stock + flux)  
2. Acquisition RNE (stock + flux + base consolidée)  
3. **ETL SIRENE** :
   - sélectionne le mois SIRENE courant ou précédent disponible,
   - lit SIRENE stock + flux et la base RNE consolidée,
   - enrichit avec les datasets secondaires,
   - construit la base SQLite `sirene_<date>.db.gz` dans `sirene/database/`,
   - génère `data_source_updates.json` pour tracer les mises à jour,
4. Déclenchement du DAG Elasticsearch (indexation à partir de la base publiée)  
5. La base publiée est ensuite utilisée par Data.gouv et d’autres consommateurs en aval.

---

## 6. Points à creuser plus tard

- Structure exacte des tables créées (SIRENE, RNE, dirigeants, immatriculation, enrichissements)  
- Détails des validations (volumétrie, cohérence unités légales / établissements, cohérence SIRENE/RNE)  
- Mécanique interne de `DatabaseTableConstructor`  
- Format exact de `data_source_updates.json` et son usage aval  
- Gestion des erreurs / retries Airflow (API, MinIO, volumétrie inattendue)  
- Détails de la logique `determine_sirene_date` (gestion des cas limites de disponibilité)
