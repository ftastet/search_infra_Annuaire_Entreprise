# CONTEXT_RNE_PROJECT.md
Contexte global du projet de rétro-ingénierie et documentation du pipeline RNE/SIRENE.

## 1. Objectif général du projet
Le but du projet est de documenter, comprendre et reproduire les pipelines d’acquisition et de traitement des données d’entreprise (SIRENE, RNE) utilisés dans le dépôt GitHub analysé.

Cette documentation doit être :
- fidèle au code réel (aucune inférence, aucune invention),
- organisée en fichiers Markdown,
- contrôlée par audits Web-Codex,
- utilisable pour reconstruire ou re-implementer les pipelines.

## 2. Priorité stratégique (Lot 1 vs Lot 2)
### Lot 1 — **Pipeline RNE uniquement**
Le périmètre prioritaire pour la société est :
- acquisition du stock RNE,
- acquisition quotidienne des flux RNE,
- génération de la base consolidée `rne_<date>.db.gz`,
- mise à disposition de cette base (référentiel interne).

Le pipeline RNE est **totalement autonome** et ne dépend pas du pipeline SIRENE.

### Lot 2 — **Fusion RNE + SIRENE et ETL global**
En second temps uniquement, les pipelines SIRENE et ETL (construction de `sirene.db`) seront étudiés.
Le Lot 1 n’a **aucune dépendance** directe au pipeline SIRENE.

---

## 3. Règles strictes d'écriture pour la documentation
Ces règles sont imposées par l’utilisateur et doivent être respectées :
- **Ne jamais inventer, déduire ou extrapoler** : toute information non vérifiable doit être explicitement marquée `[Non vérifié]` ou exclue.
- Utiliser les audits Codex pour valider la fidélité du contenu Markdown.
- Ne pas alourdir les documents : rester sur des vues d’ensemble (overview) pour les sections correspondantes.
- Pour les vues détaillées : n’ajouter que des faits visibles dans le code réel.
- Ne jamais reformuler les messages de l’utilisateur.
- En cas d’erreur : ajouter la mention “Correction : ...”.

---

## 4. Structure documentaire validée


## Fichiers RNE (dossier `/docs/rne/`) :
- **2.1_RNE_Architecture.md** ✔ (validé Codex)
- **2.2_RNE_Acquisition.md** ⏳
- **2.3_RNE_Database.md** ⏳
- **2.4_RNE_Mapping.md** ⏳
- **2.5_RNE_Tables_Finales.md** ⏳
- **2.6_RNE_ConsommationEntreprise.md** ⏳

## Autres fichiers liés au pipeline global :
- `0_overview_pipeline_sirene_rne.md` ✔ (validé Codex)
- `1_Acquisition_SIRENE.md`✔ (validé Codex)
- `3_ETL_SIRENE.md` ✔ (validé Codex)
- `4_Consommateurs.md` ✔ (validé Codex)

---

## 5. Architecture générale du pipeline RNE (résumé)
Basée sur les DAGs présents dans :
- workflows/data_pipelines/rne/stock/**
- workflows/data_pipelines/rne/flux/**
- workflows/data_pipelines/rne/database/**

### DAG 1 : get_rne_stock
- Téléchargement automatique du ZIP INPI via `get_stock.sh`.
- Extraction des JSON.
- Dépôt MinIO : `rne/stock/`.

### DAG 2 : get_flux_rne
- Appel quotidien à 01h00 à l’API RNE Diff.
- Dépôt MinIO : `rne/flux/`.

### DAG 3 : fill_rne_database
- Lecture de `latest_rne_date.json`.
- Fallback si absent : `RNE_DEFAULT_START_DATE` = `2025-04-03`.
- Ignorance volontaire du flux le plus récent (flux J).
- Parsing, validation, déduplication.
- Construction SQLite → `rne_<date>.db.gz`.
- Upload dans MinIO : `rne/database/`.
- Mise à jour de `latest_rne_date.json`.

---

## 6. Artefacts produits (Lot 1)
- **rne/stock/**  
  JSON du stock annuel.

- **rne/flux/**  
  JSON des flux quotidiens.

- **rne/database/rne_<date>.db.gz**  
  Base SQLite consolidée contenant les tables :
  - unite_legale  
  - siege  
  - etablissement  
  - activite  
  - immatriculation  
  - dirigeant_pp  
  - dirigeant_pm  

- **rne/database/latest_rne_date.json**  
  Date du dernier flux traité (J-1).

---

## 7. Contraintes techniques déjà identifiées
- Orchestreur : Airflow (DAGs stock, flux, database).
- Stockage : MinIO (S3-compatible).
- Base intermédiaire : SQLite (compressée en `.gz`).
- Validation et parsing : Pydantic + Python.
- Multiples structures JSON profondes.
- Flux API RNE Diff potentiellement incomplets à J0.
- Versionnage implicite par date (flux J-1 utilisé pour nommage base).

---

## 8. Schémas validés

### Schéma général du pipeline RNE
FTP INPI ----------------------> MinIO rne/stock/
API RNE companies/diff -------> MinIO rne/flux/
|
v
fill_rne_database
|
v
SQLite rne.db
|
v
MinIO rne/database/


---

## 9. Décisions actées
- Le flux RNE incomplet du jour J est **toujours ignoré**.
- Le stock n’est chargé qu’à la première exécution.
- La base consolidée couvre **jusqu’à J-1**.
- Le Lot 1 ne doit pas référencer SIRENE ou l’ETL global.
- Les documents doivent être validés à chaque étape via audit Codex.

---

## 10. Comment utiliser ce fichier comme contexte
À chaque nouvelle discussion ChatGPT :

1. Charger ce fichier intégral.
2. Demander explicitement :
   “Charge tout le contenu de CONTEXT_RNE_PROJECT.md comme contexte permanent
    pour cette session.”
3. Continuer le travail sur les fichiers `/docs/rne/*.md`.



