# rne_pipeline_files_details.md

## 1. Objectif
Ce document fournit une vue opérationnelle de bout en bout du pipeline RNE. Il reprend la structure de la partie 5 de `2.1_RNE_Architecture.md` et détaille, pour chaque étape, les DAGs Airflow, les tâches, les fichiers exécutés (Python/bash/SQL), les fonctions ou classes clés et l’ordre réel d’appel. L’objectif est de permettre à un non‑développeur de suivre précisément ce qui s’exécute lors de l’acquisition du stock, de l’acquisition quotidienne des flux et de la construction de la base consolidée RNE.

## 2. Vue d’ensemble du pipeline RNE
- **DAGs Airflow impliqués** :
  - `get_rne_stock` (acquisition du stock initial).
  - `get_flux_rne` (acquisition quotidienne des flux via l’API RNE Diff).
  - `fill_rne_database` (construction quotidienne de la base SQLite consolidée et mise à jour des métadonnées).
- **Stockage commun** : MinIO est la source unique de vérité (`rne/stock/`, `rne/flux/`, `rne/database/`).
- **Chaîne globale** : stock (ponctuel) → flux (quotidien) → consolidation (quotidienne, exclut le flux J).

## 3. Détail par étape (partie 5 de 2.1, enrichie)
### 3.1 Acquisition du stock RNE
- **DAG impliqué** : `get_rne_stock` (fichier `workflows/data_pipelines/rne/stock/DAG.py`).
- **Tâches Airflow (ordre)** :
  1. `clean_previous_outputs` (bash) – purge et recrée le dossier temporaire.
  2. `get_rne_latest_stock` – télécharge le ZIP du stock via `download_stock`.
  3. `unzip_files_and_upload_minio` – décompresse le ZIP et envoie chaque JSON dans MinIO `rne/stock/`.
- **Fichiers exécutés** :
  - DAG Airflow : `workflows/data_pipelines/rne/stock/DAG.py`.
  - Script bash de téléchargement FTP : `workflows/data_pipelines/rne/stock/get_stock.sh`.
  - Classe processeur : `workflows/data_pipelines/rne/stock/processor.py`.
  - Configuration source : `workflows/data_pipelines/rne/stock/config.py`.
- **Fonctions / classes clés** :
  - `RneStockProcessor` (hérite de `DataProcessor`) : orchestre le téléchargement et l’upload MinIO (`processor.py`).
  - `download_stock(ftp_url)` : exécute `get_stock.sh` avec `subprocess.run` pour rapatrier `stock_rne.zip` dans le répertoire temporaire.
  - `send_stock_to_minio()` : ouvre le ZIP, extrait chaque JSON puis envoie via `MinIOClient` (classe `File` pour le mapping des chemins) avant de supprimer les fichiers locaux.
- **Chaîne d’appel** :
  - DAG instancie `RneStockProcessor` → `clean_previous_outputs` prépare le dossier → `get_rne_latest_stock` appelle `download_stock` → le script `get_stock.sh` télécharge le ZIP → `unzip_files_and_upload_minio` appelle `send_stock_to_minio` qui extrait et pousse chaque fichier vers MinIO.

### 3.2 Acquisition quotidienne des flux RNE
- **DAG impliqué** : `get_flux_rne` (fichier `workflows/data_pipelines/rne/flux/DAG.py`).
- **Tâches Airflow (ordre)** :
  1. `clean_previous_outputs` (bash) – supprime/recrée le répertoire temporaire `RNE_FLUX_TMP_FOLDER`.
  2. `get_every_day_flux` – boucle de récupération de tous les flux manquants jusqu’à J–1 et upload MinIO.
  3. `clean_outputs` (bash) – supprime les fichiers temporaires locaux.
  4. `send_notification_success_mattermost` – notification Mattermost de succès (en échec, callback `send_notification_failure_mattermost`).
- **Fichiers exécutés** :
  - DAG Airflow : `workflows/data_pipelines/rne/flux/DAG.py`.
  - Tâches Python : `workflows/data_pipelines/rne/flux/flux_tasks.py`.
  - Client API : `workflows/data_pipelines/rne/flux/rne_api.py`.
- **Fonctions / classes clés** :
  - `get_every_day_flux(ti)` : `ti` est l’objet `TaskInstance` fourni par Airflow pour lire/écrire des XCom ; la fonction détermine `start_date` (fichier le plus récent dans MinIO, sinon `RNE_DEFAULT_START_DATE`), fixe `end_date` à J–1, itère jour par jour et appelle `get_and_save_daily_flux_rne`.
  - `get_and_save_daily_flux_rne(start_date, end_date, first_exec, ti)` : crée un fichier JSON local, instancie `ApiRNEClient`, itère sur les pages API, écrit chaque entreprise sur une ligne, gère les erreurs (sauvegarde partielle en `.gz` puis suppression), compresse et envoie le fichier vers MinIO `rne/flux/`, supprime les artefacts locaux.
  - `ApiRNEClient` : gère l’authentification/token (RNE API token), construit l’URL `RNE_API_DIFF_URL`, effectue des retries avec adaptation du `pageSize` si nécessaire.
  - Fonctions de support : `compute_start_date`, `get_last_json_file_date`, `get_last_siren` (pour redémarrer en cas de reprise), `send_notification_success_mattermost`/`send_notification_failure_mattermost` (notifications Mattermost).
- **Chaîne d’appel** :
  - DAG crée les tasks → `clean_previous_outputs` → `get_every_day_flux` : calcule la date de début, puis pour chaque jour appelle `get_and_save_daily_flux_rne` → `get_and_save_daily_flux_rne` instancie `ApiRNEClient.make_api_request` en boucle, écrit puis compresse le fichier, l’upload à MinIO → `clean_outputs` supprime le répertoire temporaire → notification de succès.

### 3.3 Consolidation quotidienne de la base RNE
- **DAG impliqué** : `fill_rne_database` (fichier `workflows/data_pipelines/rne/database/DAG.py`).
- **Tâches Airflow (ordre)** :
  1. `clean_previous_outputs` (bash) – nettoie et recrée le dossier `RNE_DB_TMP_FOLDER`.
  2. `get_start_date` – télécharge `latest_rne_date.json` (si présent) depuis MinIO et pousse `start_date` en XCom.
  3. `create_db` – crée une base SQLite vide (si premier run) et enregistre son chemin dans XCom.
  4. `get_latest_db` – si `start_date` existe, télécharge la base précédente (`rne_<start_date-1>.db.gz`), la décompresse et affiche des comptes de tables.
  5. `process_stock_json_files` – uniquement si `start_date` est `None` : télécharge chaque JSON du stock MinIO, appelle `inject_records_into_db` pour charger les données puis supprime les fichiers locaux.
  6. `process_flux_json_files` – liste les flux MinIO, exclut systématiquement le flux le plus récent, filtre sur `start_date` (ou `0000-00-00` si premier run), décompresse chaque fichier, appelle `inject_records_into_db` pour l’injection, supprime les fichiers locaux, pousse `last_date_processed` en XCom.
  7. `remove_duplicates` – passe sur toutes les tables (`unites_legale`, `siege`, `dirigeant_pp`, `dirigeant_pm`, `immatriculation`) pour supprimer les doublons puis `VACUUM`.
  8. `check_db_count` – contrôle des volumes minimums par table (raises si sous les seuils).
  9. `upload_db_to_minio` – compresse `rne_<start_date>.db` en `.gz`, l’upload vers MinIO `rne/database/` en le renommant `rne_<last_date_processed>.db.gz`, supprime le `.db.gz` local.
  10. `upload_latest_date_rne_minio` – écrit `latest_rne_date.json` avec `latest_date = last_date_processed + 1`, l’upload dans MinIO `rne/database/`, supprime la copie locale, pousse la nouvelle date en XCom.
  11. `clean_outputs` (bash) – supprime le dossier temporaire.
  12. `send_notification_mattermost` – notifie les dates traitées.
- **Fichiers exécutés** :
  - DAG Airflow : `workflows/data_pipelines/rne/database/DAG.py`.
  - Fonctions de tâches : `workflows/data_pipelines/rne/database/task_functions.py`.
  - Connexion SQLite : `workflows/data_pipelines/rne/database/db_connexion.py`.
  - Parsing/mapping/injection : `workflows/data_pipelines/rne/database/process_rne.py`, `map_rne.py`, modèles `rne_model.py` (structure source) et `ul_model.py` (structure cible SQLite).
- **Fonctions / classes clés** :
  - `get_start_date_minio` : lit `latest_rne_date.json` depuis MinIO (`RNE_MINIO_DATA_PATH`), pousse `start_date` (ou `None` si absent).
  - `create_db_path`/`create_db` : détermine le chemin `rne_<start_date>.db`, crée la base et les tables via `create_tables` si pas de `start_date` (premier run).
  - `get_latest_db` : récupère la base précédente (`rne_<start_date-1>.db.gz`), décompresse et compte les tables via `get_tables_count`.
  - `process_stock_json_files` : boucle sur les JSON du stock MinIO (non compressés), injecte dans la base par `inject_records_into_db(file_path, db_path, file_type="stock")` puis supprime les fichiers.
  - `process_flux_json_files` : liste MinIO `rne/flux`, exclut le dernier flux, filtre sur la date, décompresse, appelle `inject_records_into_db(..., file_type="flux")`, supprime les fichiers, pousse `last_date_processed`.
  - `inject_records_into_db` (dans `process_rne.py`) : lit les fichiers (stock complet ou flux ligne à ligne), transforme via `process_records_to_extract_rne_data` qui mappe chaque enregistrement RNE (`RNECompany`) vers `UniteLegale` au moyen de `map_rne_company_to_ul`, puis insère dans SQLite via `insert_unites_legales_into_db`.
  - `map_rne_company_to_ul` (dans `map_rne.py`) : convertit la structure RNE (Pydantic `RNECompany` et objets associés) en objets `UniteLegale`, `Siege`, `Etablissement`, `Activite`, `DirigeantsPP/PM`, `Immatriculation` définis dans `ul_model.py`.
  - `remove_duplicates_from_tables` : supprime les doublons par table, appelé par `remove_duplicates`.
  - `upload_db_to_minio` / `upload_latest_date_rne_minio` : gèrent la compression, l’upload du `.db.gz` et la mise à jour des métadonnées `latest_rne_date.json`.
- **Chaîne d’appel** :
  - DAG crée les tasks → `clean_previous_outputs` → `get_start_date_minio` (XCom `start_date`) → `create_db` (XCom `rne_db_path`, éventuellement création de tables) → `get_latest_db` (récupération base précédente si `start_date` défini) → `process_stock_json_files` (si première exécution) → `process_flux_json_files` (tous les runs, sauf dernier flux) → `remove_duplicates` → `check_db_count` → `upload_db_to_minio` → `upload_latest_date_rne_minio` → `clean_outputs` → `notification_mattermost`.

### 3.4 Mise à disposition
- **DAG impliqué** : `fill_rne_database` (même que l’étape 3.3).
- **Actions finales** :
  - Upload du fichier `rne_<last_date_processed>.db.gz` dans `rne/database/`.
  - Upload/écrasement de `latest_rne_date.json` dans `rne/database/` avec `latest_date = last_date_processed + 1`.
  - Notification Mattermost des dates couvertes.
- **Chaîne d’appel** : tâches `upload_db_to_minio` → `upload_latest_date_rne_minio` → `clean_outputs` → `send_notification_mattermost` dans le DAG `fill_rne_database`.

## 4. Séquence d’exécution de bout en bout
1. **Acquisition du stock initial** (DAG manuel `get_rne_stock`, ponctuel)
   - `clean_previous_outputs` → `get_rne_latest_stock` (exécute `download_stock` → `get_stock.sh`) → `unzip_files_and_upload_minio` (`send_stock_to_minio`).
2. **Acquisition quotidienne des flux** (DAG `get_flux_rne`, 01h)
   - `clean_previous_outputs` → `get_every_day_flux` (`compute_start_date` → boucle jour par jour → `get_and_save_daily_flux_rne` → `ApiRNEClient.make_api_request` → upload MinIO) → `clean_outputs` → `send_notification_success_mattermost` (échec : callback `send_notification_failure_mattermost`).
3. **Construction quotidienne de la base consolidée** (DAG `fill_rne_database`, 02h)
   - `clean_previous_outputs` → `get_start_date` (lit `latest_rne_date.json` si présent) → `create_db` (crée tables si premier run) → `get_latest_db` (récupère base précédente si reprise) → `process_stock_json_files` (premier run) → `process_flux_json_files` (tous les runs, exclut flux J) → `remove_duplicates` → `check_db_count` → `upload_db_to_minio` (compresse, renomme selon `last_date_processed`) → `upload_latest_date_rne_minio` (met à jour métadonnées) → `clean_outputs` → `send_notification_mattermost`.
- **Fonctions réutilisées entre DAGs** :
  - Helpers MinIO (`MinIOClient`, `File`) sont utilisés par le processeur de stock, le collecteur de flux et la consolidation (téléchargement/chargement des fichiers).
  - Notifications Mattermost (`Notification.send_notification_mattermost`, `send_message`) sont utilisées dans les DAGs stock, flux et base.

## 5. Points d’attention
- **Dépendance MinIO** : tous les DAGs lisent/écrivent dans MinIO ; les échecs réseau ou d’authentification bloquent la chaîne (stock → flux → base).
- **Gestion des dates** : `get_flux_rne` détermine `start_date` à partir du dernier flux en MinIO ou `RNE_DEFAULT_START_DATE`; `fill_rne_database` se base uniquement sur `latest_rne_date.json` (pas de fallback) pour décider de recharger le stock et quels flux appliquer.
- **Exclusion du flux le plus récent** : `process_flux_json_files` ignore systématiquement le dernier fichier flux pour éviter un flux incomplet ; la base couvre donc jusqu’à J–1.
- **Volume et cohérence** : `check_db_count` impose des seuils minimaux sur les tables SQLite ; en dessous, le DAG échoue (probable problème de données ou de chargement).
- **Nettoyage** : chaque DAG supprime ses dossiers temporaires en fin d’exécution ; en cas d’échec avant nettoyage, des fichiers peuvent rester mais sont purgés au run suivant (`clean_previous_outputs`).
- **API RNE** : `ApiRNEClient` implémente des retries et peut ajuster `pageSize` en cas d’erreur 500/mémoire ; une absence de token ou des erreurs 401/403/429 forcent la régénération du token.
