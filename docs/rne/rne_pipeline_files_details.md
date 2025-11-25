# rne_pipeline_files_details.md

## 1. Objectif
Ce document fournit une vision complète, technique et fonctionnelle, de l’ensemble du pipeline RNE.  
Il détaille les DAGs Airflow, les fichiers exécutés, les fonctions, les modèles et les séquences d’appel, mais aussi la logique métier sous-jacente (pourquoi chaque étape existe, ce qu’elle produit, ce qu’elle consomme).  

L’objectif est que même un non-développeur puisse comprendre :
- ce qui se passe réellement dans le pipeline,
- dans quel ordre,
- quels fichiers sont impliqués,
- et comment la base consolidée RNE est produite chaque jour.

Le document reprend la partie 5 du fichier `2.1_RNE_Architecture.md` et l’étend en une vue opérationnelle détaillée.

---

## 2. Vue d’ensemble du pipeline RNE

### 2.1 Vue simple (pour non-développeur)

STOCK (run manuel une seule fois)
    ↓ écrit dans MinIO rne/stock/

FLUX (tous les jours à 01h00)
    ↓ écrit dans MinIO rne/flux/

BASE CONSOLIDÉE RNE (tous les jours à 02h00)
    ↓ lit rne/stock + rne/flux (sauf flux J)
    ↓ génère rne_<date>.db.gz
    ↓ met à jour latest_rne_date.json

### 2.2 Vue technique
DAGs Airflow impliqués :
- **get_rne_stock**  
  Rapatrie le ZIP du stock et dépose les JSON en `rne/stock/`.

- **get_flux_rne**  
  Télécharge les flux RNE Diff depuis l’API INPI, un fichier `.json.gz` par jour, stockés en `rne/flux/`.

- **fill_rne_database**  
  Construit une base SQLite consolidée contenant tout l’historique : stock + flux jusqu’à J–1.

Stockage centralisé :
- MinIO est la source unique de vérité :
  - `rne/stock/`
  - `rne/flux/`
  - `rne/database/`

Rythme :
- Stock : ponctuel.
- Flux : quotidien.
- Consolidation : quotidienne (exclut J par sécurité).

---

## 3. Glossaire (important pour non-dev)

**DAG** : un workflow Airflow composé de tâches ordonnées.  
**Task** : une étape individuelle d’un DAG.  
**MinIO** : stockage objet compatible S3.  
**Stock RNE** : photographie complète du registre INPI à un instant T.  
**Flux RNE** : mises à jour quotidiennes du registre.  
**Snapshot SQLite** : base générée chaque jour pour exploitation interne.  
**XCom** : mécanisme Airflow pour transmettre des données entre tâches.  
**Flux J** : flux du jour en cours, ignoré par sécurité (souvent incomplet).  
**First run** : première exécution historique (inclut le stock).  

---

## 4. Logique fonctionnelle du pipeline (la raison de chaque étape)

### 4.1 Pourquoi un stock ?
Il donne un **état initial complet** du registre RNE.  
Sans ce point de départ, impossible de reconstruire un historique fiable.

### 4.2 Pourquoi des flux ?
Ils amènent **tous les changements quotidiens** :
- créations,
- radiations,
- modifications,
- dirigeants,
- adresses,
- activités.

### 4.3 Pourquoi une base consolidée ?
Les JSON RNE sont bruts, volumineux et difficiles à manipuler.  
La consolidation produit :
- une structure tabulaire exploitable,
- une vue unifiée,
- un référentiel interne fiable,
- une base versionnée par jour.

### 4.4 Pourquoi ignorer le flux J ?
Le flux du jour peut être :
- incomplet,
- en cours de constitution,
- corrompu lors d’une récupération partielle.

Pour garantir la fiabilité :
→ le pipeline intègre les flux **jusqu’à J–1**.

### 4.5 Pourquoi des répertoires temporaires ?
Pour éviter que des fichiers partiellement écrits ou corrompus n’influencent les runs suivants.

---

## 5. Détail complet par étape (partie 5 enrichie)

### 5.1 Acquisition du stock RNE
- **DAG :** `get_rne_stock`  
  (`workflows/data_pipelines/rne/stock/DAG.py`)

#### Ordre des tâches
1. `clean_previous_outputs`  
2. `get_rne_latest_stock`  
3. `unzip_files_and_upload_minio`

#### Fichiers utilisés
- `workflows/data_pipelines/rne/stock/DAG.py`
- `workflows/data_pipelines/rne/stock/processor.py`
- `workflows/data_pipelines/rne/stock/config.py`
- `workflows/data_pipelines/rne/stock/get_stock.sh`

#### Fonctions / classes clés
- `RneStockProcessor` : orchestre les étapes de téléchargement et d’upload.
- `download_stock` : exécute `get_stock.sh` et télécharge le ZIP du stock RNE.
- `send_stock_to_minio` : lit le ZIP, extrait chaque JSON et l’envoie vers MinIO.

#### Fonctionnellement
- Télécharge le ZIP officiel INPI.
- Extrait tous les JSON.
- Envoie chaque fichier dans `rne/stock/`.
- Cette étape n’est normalement exécutée qu’une seule fois (ou lors d’une réinitialisation).

---

### 5.2 Acquisition quotidienne des flux RNE
- **DAG :** `get_flux_rne`  
  (`workflows/data_pipelines/rne/flux/DAG.py`)

#### Ordre des tâches
1. `clean_previous_outputs`  
2. `get_every_day_flux`  
3. `clean_outputs`  
4. `send_notification_success_mattermost` (en cas d’échec, callback `send_notification_failure_mattermost`)

#### Fichiers utilisés
- `workflows/data_pipelines/rne/flux/flux_tasks.py`
- `workflows/data_pipelines/rne/flux/rne_api.py`

#### Fonctions / classes clés
- `compute_start_date` : Inspecte les fichiers flux déjà présents dans MinIO et renvoie **la dernière date trouvée**, pas le jour suivant. La boucle `get_every_day_flux` redémarre donc **sur cette journée déjà existante (retraitement inclus)** avant de poursuivre jusqu’à J–1.
- `get_every_day_flux` : pilote la boucle sur les jours à traiter jusqu’à J–1.
- `get_and_save_daily_flux_rne` : appelle l’API RNE Diff pour une journée donnée, écrit les enregistrements JSON ligne à ligne, compresse en `.json.gz`, envoie dans MinIO, gère les erreurs et la reprise.
- `ApiRNEClient` : encapsule l’appel à l’API (token, URL de base, retries, adaptation du pageSize).
- `get_last_siren` : récupère le dernier SIREN traité dans un fichier flux existant pour reprendre en cas de coupure.
- `send_notification_success_mattermost` / `send_notification_failure_mattermost` : journalisent les résultats dans Mattermost.

#### Fonctionnellement
- La dernière journée présente dans MinIO est systématiquement retraitée pour garantir la complétude d’un flux qui aurait pu être partiellement téléchargé lors d’un run précédent.
- Détermine automatiquement à partir du contenu MinIO la première date de flux à récupérer.
- Télécharge tous les flux manquants jusqu’à J–1.
- Assure la résilience en cas d’erreurs API (reprise, réduction du pageSize, sauvegarde partielle).
- Produit un fichier compressé `.json.gz` par journée, stocké dans `rne/flux/`.

---

### 5.3 Construction quotidienne de la base consolidée RNE
- **DAG :** `fill_rne_database`  
  (`workflows/data_pipelines/rne/database/DAG.py`)

#### Ordre des tâches
1. `clean_previous_outputs`  
2. `get_start_date`  
3. `create_db`  
4. `get_latest_db`  
5. `process_stock_json_files`  
6. `process_flux_json_files`  
7. `remove_duplicates`  
8. `check_db_count`  
9. `upload_db_to_minio`  
10. `upload_latest_date_rne_minio`  
11. `clean_outputs`  
12. `send_notification_mattermost`

#### Chemin technique global

```
DAG fill_rne_database                              (rne/database/DAG.py)
    │
    ├── clean_previous_outputs (BashOperator – DAG.py)
    │       → bash_command: rm -rf ${RNE_DB_TMP_FOLDER} && mkdir -p ${RNE_DB_TMP_FOLDER}
    │
    ├── get_start_date (PythonOperator – DAG.py)
    │       → get_start_date_minio(**kwargs)             (rne/database/task_functions.py)
    │               → MinIOClient().get_files(...)       (helpers/minio_helpers.py)
    │                     - source_path: RNE_MINIO_DATA_PATH
    │                     - source_name: RNE_LATEST_DATE_FILE (latest_rne_date.json)
    │                     - dest_path:   RNE_DB_TMP_FOLDER
    │               → ouvre RNE_DB_TMP_FOLDER/latest_rne_date.json
    │               → lit data["latest_date"]
    │               → parse en datetime, reformat YYYY-MM-DD
    │               → ti.xcom_push("start_date", value=start_date)
    │               → en cas de S3Error code "NoSuchKey" :
    │                     ti.xcom_push("start_date", value=None)
    │
    ├── create_db (PythonOperator – DAG.py)
    │       → create_db(**kwargs)                        (rne/database/task_functions.py)
    │               → start_date = ti.xcom_pull("start_date", task_ids="get_start_date")
    │               → rne_db_path = create_db_path(start_date)   (task_functions.py)
    │                     - construit: RNE_DB_TMP_FOLDER + f"rne_{start_date}.db"
    │               → ti.xcom_push("rne_db_path", rne_db_path)
    │               → si start_date est non nul:
    │                     return (on ne recrée pas la DB)
    │               → si fichier rne_db_path existe: os.remove(...)
    │               → connect_to_db(rne_db_path)         (rne/database/db_connexion.py)
    │               → create_tables(cursor)              (rne/database/process_rne.py)
    │               → connection.commit() / close()
    │
    ├── get_latest_db (PythonOperator – DAG.py)
    │       → get_latest_db(**kwargs)                    (rne/database/task_functions.py)
    │               → start_date = ti.xcom_pull("start_date", "get_start_date")
    │               → si start_date est non nul:
    │                     previous_start_date = (start_date - 1 jour)
    │                     MinIOClient().get_files(...)   (helpers/minio_helpers.py)
    │                         - source_name: rne_{previous_start_date}.db.gz
    │                         - dest_name:   rne_{start_date}.db.gz
    │                         - paths: RNE_MINIO_DATA_PATH → RNE_DB_TMP_FOLDER
    │                     db_path = RNE_DB_TMP_FOLDER + f"rne_{start_date}.db"
    │                     gzip.open(db_path + ".gz") → open(db_path) (shutil.copyfileobj)
    │                     os.remove(db_path + ".gz")
    │               → get_tables_count(RNE_DB_TMP_FOLDER + f"rne_{start_date}.db")
    │                     (rne/database/process_rne.py)
    │               → logging des counts (UL, siège, dirigeants, immat)
    │
    ├── process_stock_json_files (PythonOperator – DAG.py)
    │       → process_stock_json_files(**kwargs)         (rne/database/task_functions.py)
    │               → start_date = ti.xcom_pull("start_date", "get_start_date")
    │               → rne_db_path = ti.xcom_pull("rne_db_path", "create_db")
    │               → minio_client = MinIOClient()
    │               → si start_date est non nul:
    │                     return  (stock traité uniquement si pas encore de date)
    │               → json_stock_rne_files = minio_client.get_files_from_prefix(
    │                     prefix=RNE_MINIO_STOCK_DATA_PATH
    │                 )
    │               → si liste vide: raise Exception("No RNE stock files found!!!")
    │               → pour chaque file_path dans json_stock_rne_files:
    │                     minio_client.get_files(
    │                         list_files=[{
    │                             "source_path": "",
    │                             "source_name": file_path,
    │                             "dest_path": "",
    │                             "dest_name": file_path,
    │                         }]
    │                     )
    │                     → inject_records_into_db(file_path, rne_db_path, "stock")
    │                           (rne/database/process_rne.py)
    │                     → os.remove(file_path)
    │
    ├── process_flux_json_files (PythonOperator – DAG.py)
    │       → process_flux_json_files(**kwargs)          (rne/database/task_functions.py)
    │               → start_date  = ti.xcom_pull("start_date", "get_start_date")
    │               → rne_db_path = ti.xcom_pull("rne_db_path", "create_db")
    │               → minio_client = MinIOClient()
    │               → json_daily_flux_files = minio_client.get_files_from_prefix(
    │                     prefix=RNE_MINIO_FLUX_DATA_PATH
    │                 )
    │               → si liste vide: return
    │               → si start_date est None: start_date = "0000-00-00"
    │               → filtre des fichiers flux par date ≥ start_date
    │               → pour chaque file_date retenue:
    │                     minio_client.get_files(
    │                         list_files=[{
    │                             "source_path": RNE_MINIO_FLUX_DATA_PATH,
    │                             "source_name": f"rne_flux_{file_date}.json.gz",
    │                             "dest_path": RNE_DB_TMP_FOLDER,
    │                             "dest_name": f"rne_flux_{file_date}.json.gz",
    │                         }]
    │                     )
    │                     json_path = RNE_DB_TMP_FOLDER + f"rne_flux_{file_date}.json"
    │                     → gzip.open(json_path + ".gz") → open(json_path)
    │                     → os.remove(json_path + ".gz")
    │                     → inject_records_into_db(json_path, rne_db_path, "flux")
    │                           (rne/database/process_rne.py)
    │                     → os.remove(json_path)
    │               → dates = tri des dates "rne_flux_YYYY-MM-DD" dans json_daily_flux_files
    │               → last_date_processed = dernière date ou None
    │               → ti.xcom_push("last_date_processed", last_date_processed)
    │
    │       [Détail interne inject_records_into_db / flux]        (rne/database/process_rne.py)
    │               → pour chaque ligne JSON:
    │                     data = json.loads(line)
    │                     unites_legales_temp = process_records_to_extract_rne_data(
    │                           data, "flux")
    │                     unites_legales += unites_legales_temp
    │                     si len(unites_legales) >= 100000:
    │                           insert_unites_legales_into_db(unites_legales, file_path, db_path)
    │                           unites_legales = []
    │               → flush final via insert_unites_legales_into_db(...)
    │
    │       [Détail interne mapping / Pydantic]                   (rne/database/process_rne.py)
    │               process_records_to_extract_rne_data(data,"flux")
    │                     → extract_rne_data(entity, "flux")
    │                           → company = entity["company"]
    │                           → rne_company = RNECompany.model_validate(company)
    │                                 (rne/database/rne_model.py – Pydantic)
    │                           → unite_legale = UniteLegale()        (rne/database/ul_model.py)
    │                           → unite_legale = map_rne_company_to_ul(
    │                                 rne_company, unite_legale)      (rne/database/map_rne.py)
    │                           → retourne UniteLegale rempli
    │               insert_unites_legales_into_db(...)
    │                     → connect_to_db(db_path)                   (db_connexion.py)
    │                     → find_and_delete_same_siren(...)          (process_rne.py)
    │                     → INSERT INTO: unite_legale, siege, dirigeant_pp,
    │                                       dirigeant_pm, immatriculation,
    │                                       etablissement, activite
    │                     → commit / close
    │
    ├── remove_duplicates (PythonOperator – DAG.py)
    │       → remove_duplicates(**kwargs)                  (rne/database/task_functions.py)
    │               → rne_db_path = ti.xcom_pull("rne_db_path", "create_db")
    │               → connection, cursor = connect_to_db(rne_db_path)
    │               → tables = ["unites_legale","siege","dirigeant_pp",
    │                            "dirigeant_pm","immatriculation"]
    │               → pour chaque table:
    │                     remove_duplicates_from_tables(cursor, table)
    │                           (rne/database/process_rne.py)
    │                             - crée table temporaire
    │                             - INSERT DISTINCT
    │                             - DROP ancienne table
    │                             - RENAME temp → table
    │               → VACUUM
    │               → commit, close (rollback + close en cas d’exception)
    │
    ├── check_db_count (PythonOperator – DAG.py)
    │       → check_db_count(ti, min_*_table_count=...)    (rne/database/task_functions.py)
    │               → rne_db_path = ti.xcom_pull("rne_db_path", "create_db")
    │               → count_ul, count_siege, count_pp, count_pm, count_immat = \
    │                     get_tables_count(rne_db_path)    (rne/database/process_rne.py)
    │               → logging des counts
    │               → si un count < min correspondant:
    │                     raise Exception("Minimum threshold not met...")
    │
    ├── upload_db_to_minio (PythonOperator – DAG.py)
    │       → upload_db_to_minio(**kwargs)                 (rne/database/task_functions.py)
    │               → start_date         = ti.xcom_pull("start_date", "get_start_date")
    │               → last_date_processed= ti.xcom_pull("last_date_processed", "process_flux_json_files")
    │               → database_file_path     = RNE_DB_TMP_FOLDER + f"rne_{start_date}.db"
    │               → database_zip_file_path = RNE_DB_TMP_FOLDER + f"rne_{start_date}.db.gz"
    │               → gzip de rne_{start_date}.db → rne_{start_date}.db.gz
    │               → os.remove(database_file_path)
    │               → send_to_minio([...])                  (task_functions.py)
    │                     → MinIOClient().send_files(...)   (helpers/minio_helpers.py)
    │                           - source_name: rne_{start_date}.db.gz
    │                           - dest_name:   rne_{last_date_processed}.db.gz
    │               → os.remove(database_zip_file_path) (si présent)
    │
    ├── upload_latest_date_rne_minio (PythonOperator – DAG.py)
    │       → upload_latest_date_rne_minio(ti)             (rne/database/task_functions.py)
    │               → last_date_processed = ti.xcom_pull("last_date_processed",
    │                                                    "process_flux_json_files")
    │               → last_date_processed = datetime.strptime(..., "%Y-%m-%d")
    │               → latest_date = (last_date_processed + 1 jour).strftime("%Y-%m-%d")
    │               → écrit RNE_DB_TMP_FOLDER + "latest_rne_date.json"
    │                     {"latest_date": latest_date}
    │               → send_to_minio([...])                  (task_functions.py)
    │                     - source_path: RNE_DB_TMP_FOLDER
    │                     - source_name: RNE_LATEST_DATE_FILE
    │                     - dest_path:   RNE_MINIO_DATA_PATH
    │                     - dest_name:   RNE_LATEST_DATE_FILE
    │
    ├── clean_outputs (BashOperator – DAG.py)
    │       → bash_command: rm -rf ${RNE_DB_TMP_FOLDER}
    │
    └── send_notification_mattermost (PythonOperator – DAG.py)
            → notification_mattermost(ti)                 (rne/database/task_functions.py)
                    → start_date = ti.xcom_pull("start_date", "get_start_date")
                    → last_date_processed = ti.xcom_pull("last_date_processed",
                    │                                     "process_flux_json_files")
                    → send_message("🟢 Données RNE traitées de {start_date} à {last_date_processed}.")
                          (helpers/mattermost.py)
```

#### Fichiers utilisés
- `workflows/data_pipelines/rne/database/DAG.py`
- `workflows/data_pipelines/rne/database/task_functions.py`
- `workflows/data_pipelines/rne/database/db_connexion.py`
- `workflows/data_pipelines/rne/database/process_rne.py`
- `workflows/data_pipelines/rne/database/map_rne.py`
- `workflows/data_pipelines/rne/database/rne_model.py`
- `workflows/data_pipelines/rne/database/ul_model.py`

#### Fonctions / classes clés
- `get_start_date_minio` : lit `latest_rne_date.json` dans MinIO, en extrait `latest_date` et pousse `start_date` en XCom (ou `None` si fichier absent).
- `create_db` / `create_tables` : Crée la base SQLite et initialise toutes les tables **uniquement lorsque `start_date` est `None` (premier run)**. Lorsque `start_date` est défini, la base n’est pas recréée : elle est récupérée par `get_latest_db`, décompressée, puis réutilisée telle quelle pour l’injection des nouveaux flux.
- `get_latest_db` : télécharge la base précédente depuis MinIO (snapshot `rne_<date>.db.gz` de la veille, i.e. `start_date – 1`), la décompresse sous `rne_<start_date>.db` et permet ainsi une reprise à partir d’un état consolidé.
- `process_stock_json_files` : ne s’exécute que si `start_date` est `None` (premier run). Télécharge les JSON du stock depuis MinIO, les injecte via `inject_records_into_db` puis supprime les fichiers locaux.
- `process_flux_json_files` : liste les flux `rne/flux`, exclut le flux le plus récent, filtre les fichiers à partir de `start_date` (ou de la date la plus ancienne si première exécution), décompresse puis injecte chaque fichier via `inject_records_into_db`, et pousse `last_date_processed` en XCom.
- `inject_records_into_db` : lit un fichier JSON (stock ou flux), transforme les enregistrements RNE en objets UL (via `process_records_to_extract_rne_data` + `map_rne_company_to_ul`) et insère dans les tables SQLite.
- `map_rne_company_to_ul` : convertit les modèles Pydantic de `rne_model.py` en objets UL cibles (`UniteLegale`, `Siege`, `Etablissement`, `Activite`, `DirigeantsPP`, `DirigeantsPM`, `Immatriculation`) définis dans `ul_model.py`.
- `remove_duplicates_from_tables` : supprime les doublons sur les tables clés.
- `check_db_count` : vérifie un nombre minimum de lignes par table ; en cas d’anomalie, la tâche échoue.
- `upload_db_to_minio` : compresse le fichier `.db` en `.db.gz`, l’upload dans `rne/database/` sous le nom `rne_<last_date_processed>.db.gz`, supprime le fichier temporaire local.
- `upload_latest_date_rne_minio` : écrit localement `latest_rne_date.json` en mettant `latest_date = last_date_processed + 1`, l’upload dans `rne/database/` puis supprime la copie locale.
- `send_notification_mattermost` : envoie un récapitulatif des dates traitées.

#### Fonctionnellement
- Récupère l’état antérieur (base précédente) si on n’est pas au tout premier run.
- Charge le stock complet uniquement une fois (première exécution historique).
- Applique les flux manquants pour amener la base à jour jusqu’à J–1.
- Garantit la cohérence et l’absence de doublons.
- Produit un snapshot daté exploitable (`rne_<date>.db.gz`) et une métadonnée `latest_rne_date.json` qui sert de point de reprise.

---

## 6. Séquence d’exécution de bout en bout

### 6.1 Résumé séquentiel

1. **Acquisition du stock (ponctuel, manuel)**  
   - `get_rne_stock`  
   - Téléchargement stock RNE → extraction → stockage JSON bruts dans `rne/stock/`.

2. **Acquisition des flux (quotidien à 01h)**  
   - `get_flux_rne`  
   - Calcul de la date de départ → appels API journaliers → fichiers `.json.gz` en `rne/flux/`.

3. **Consolidation de la base (quotidien à 02h)**  
   - `fill_rne_database`  
   - Récupération éventuelle de la base précédente → (stock si premier run) → flux jusqu’à J–1 → consolidation en SQLite → upload du snapshot + mise à jour de la date.

### 6.2 Vue textuelle simplifiée

STOCK → FLUX → BASE CONSOLIDÉE

- Le stock sert de fondation.
- Les flux rejouent l’histoire jour après jour.
- La base consolidée assemble le tout en une vision unique.

---

## 7. Points d’attention

### 7.1 Dépendance à MinIO
- Tous les DAGs lisent et écrivent dans MinIO.
- Une indisponibilité ou une erreur d’authentification perturbe la chaîne complète.

### 7.2 Gestion des dates (logique double)
- `get_flux_rne` se base sur les fichiers flux présents dans MinIO pour décider des jours à traiter.
- `fill_rne_database` se base sur `latest_rne_date.json` pour savoir jusqu’où les flux ont été intégrés dans la base.
- Une incohérence entre les deux (suppression manuelle d’un fichier, rollback, etc.) peut nécessiter une reprise contrôlée.

### 7.3 Flux J exclu systématiquement
- La base RNE du jour N reflète l’état des flux jusqu’à N–1.
- Toute consommation downstream doit garder en tête ce décalage d’un jour.

### 7.4 Volume et performance
- La base SQLite grandit avec le temps (plus de flux intégrés).
- Les opérations de déduplication et de VACUUM peuvent devenir plus coûteuses.
- L’upload des `.db.gz` peut augmenter en taille.

### 7.5 Reprise sur erreur
- Si `get_flux_rne` échoue un jour, les flux de cette journée seront rattrapés au prochain run (car absents de MinIO).
- Si `fill_rne_database` échoue, `latest_rne_date.json` n’est pas mis à jour ; le run suivant reprendra au même `start_date`.

---

## 8. TL;DR (résumé ultra-court)

- Le pipeline RNE tourne en trois temps :
  1. Stock (initial)  
  2. Flux (quotidien)  
  3. Base consolidée (quotidienne)

- Tout transite par MinIO (stock, flux, snapshots, métadonnées).
- La base consolidée `rne_<date>.db.gz` est le produit final, exploitable en interne.
- Le pipeline ignore volontairement le flux du jour (J) pour garantir la complétude.
- La reprise après incident est gérée par la combinaison :
  - fichiers présents en `rne/flux/`
  - `latest_rne_date.json`
