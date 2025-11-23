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
**first run** : première exécution historique (inclut le stock).  

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
1) `clean_previous_outputs`  
2) `get_rne_latest_stock`  
3) `unzip_files_and_upload_minio`

#### Fichiers utilisés
- `DAG.py`  
- `processor.py`  
- `config.py`  
- `get_stock.sh`

#### Fonctions clés
- `download_stock` → appelle `get_stock.sh`
- `send_stock_to_minio`
- `RneStockProcessor`

#### Fonctionnellement
- Télécharge le ZIP officiel INPI.
- Extrait tous les JSON.
- Envoie chaque entreprise dans `rne/stock/`.
- Cette étape n’est refaite qu’en cas de réinitialisation.

---

### 5.2 Acquisition quotidienne des flux RNE
- **DAG :** `get_flux_rne`  
  (`workflows/data_pipelines/rne/flux/DAG.py`)

#### Ordre des tâches
1) `clean_previous_outputs`  
2) `get_every_day_flux`  
3) `clean_outputs`  
4) `send_notification_success_mattermost`

#### Fichiers utilisés
- `flux_tasks.py`
- `rne_api.py`

#### Fonctions clés
- `compute_start_date`
- `get_and_save_daily_flux_rne`
- `ApiRNEClient.make_api_request`
- `get_last_siren` (pour reprise)
- `send_notification*_mattermost`

#### Fonctionnellement
- Détermine automatiquement la dernière journée récupérée.
- Télécharge les flux jusqu’à J–1.
- Gère pagination, erreurs API, sauvegardes partielles.
- Compresse et envoie chaque journée dans MinIO.

---

### 5.3 Construction quotidienne de la base consolidée
- **DAG :** `fill_rne_database`  
  (`workflows/data_pipelines/rne/database/DAG.py`)

#### Ordre complet des tâches
1) `clean_previous_outputs`  
2) `get_start_date`  
3) `create_db`  
4) `get_latest_db`  
5) `process_stock_json_files` (si start_date = None)  
6) `process_flux_json_files`  
7) `remove_duplicates`  
8) `check_db_count`  
9) `upload_db_to_minio`  
10) `upload_latest_date_rne_minio`  
11) `clean_outputs`  
12) `send_notification_mattermost`

#### Fichiers utilisés
- `task_functions.py`
- `db_connexion.py`
- `process_rne.py`
- `map_rne.py`
- `rne_model.py`
- `ul_model.py`

#### Fonctions clés
- `create_tables`
- `inject_records_into_db`
- `process_records_to_extract_rne_data`
- `map_rne_company_to_ul`
- `remove_duplicates_from_tables`
- `upload_db_to_minio`

#### Fonctionnellement
- Récupère la base précédente (si existante).
- Recharge le stock uniquement la toute première fois.
- Charge tous les flux utiles (sauf flux J).
- Transforme tous les JSON → tables UL.
- Déduplique.
- Valide la cohérence.
- Versionne la base + met à jour latest_rne_date.json.

---

## 6. Séquence d’exécution de bout en bout

[Stock - manuel]
↓ download_stock
↓ unzip JSON
↓ upload MinIO rne/stock/

[Flux - chaque jour à 01:00]
↓ clean_previous_outputs
↓ compute_start_date (MinIO)
↓ get_and_save_daily_flux_rne (API RNE → JSON.gz)
↓ upload MinIO rne/flux/
↓ clean_outputs
↓ notification

[Base consolidée - chaque jour à 02:00]
↓ clean_previous_outputs
↓ lire latest_rne_date.json (MinIO)
↓ récupérer base précédente si reprise
↓ charger stock (si première fois)
↓ charger flux (sauf flux J)
↓ mapping complet RNE → UL
↓ injection SQLite
---

## 7. Points d’attention

### 7.1 Dépendance forte à MinIO
Tous les DAGs lisent et écrivent dans MinIO.  
▶ Panne MinIO = pipeline bloqué.

### 7.2 Date logic double (important)
- **Flux** : start_date déterminée depuis les fichiers MinIO.  
- **Base** : start_date déterminée via `latest_rne_date.json`.  
Les deux systèmes doivent rester cohérents.

### 7.3 Flux J exclu
Pour éviter un snapshot basé sur des données incomplètes.

### 7.4 Volume croissant
La base SQLite grossit chaque jour → attention au coût des opérations (VACUUM, upload).

### 7.5 Risques d’incohérence si un DAG échoue
Si `get_flux_rne` échoue :
- La base ne sera pas avancée le lendemain.

Si `fill_rne_database` échoue :
- `latest_rne_date.json` ne bouge pas → reprise au même point au prochain run.

---

## 8. TL;DR (résumé ultra-court)

- Le **stock** initialise le registre.  
- Les **flux** amènent les changements quotidiens.  
- La **consolidation** produit une base SQLite complète et exploitable.  
- Tout transite via **MinIO**.  
- La base quotidienne est versionnée et consommable immédiatement.  
- Le pipeline est **séquentiel et strictement ordonné** :
  stock → flux → consolidation.


↓ déduplication
↓ validation
↓ compression + upload rne_<date>.db.gz
↓ maj latest_rne_date.json
↓ notification finale
↓ génère rne_<date>.db.gz
↓ met à jour latest_rne_date.json

