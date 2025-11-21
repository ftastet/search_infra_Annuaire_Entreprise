# 1 – Acquisition SIRENE (stock + flux)

## 1. Objectif du module SIRENE

L’objectif est de récupérer les données SIRENE (stock complet + flux quotidiens) et de les déposer dans MinIO (`insee/stock/` et `insee/flux/`) pour alimenter l’ETL, l’indexation Elasticsearch et les exports Data.gouv.

---

## 2. DAG `data_processing_sirene_stock`

**Chemin** : `workflows/data_pipelines/sirene/stock/`  
**dag_id** : `data_processing_sirene_stock`  
**Schedule** : `0 0 * * *`  
**Source** : ZIP SIRENE (stock complet)  
**Sortie** : MinIO `insee/stock/`

### Vue d’ensemble
1. Préparation du dossier temporaire  
2. Téléchargement du stock SIRENE  
3. Upload MinIO  
4. Nettoyage  
5. Notification

---

## 3. DAG `data_processing_sirene_flux`

**Chemin** : `workflows/data_pipelines/sirene/flux/`  
**dag_id** : `data_processing_sirene_flux`  
**Schedule** : `0 4 * * *`  
**Source** : API INSEE (flux unitaires / établissements)  
**Sorties** : MinIO `insee/flux/`, métadonnées de dernière modification

### Vue d’ensemble
1. Appel API flux  
2. Écriture fichiers du jour  
3. Upload MinIO  
4. Mise à jour métadonnées  
5. Nettoyage  
6. Notification

---

## 4. Chronologie et dépendances

1. `data_processing_sirene_stock` → stock complet en MinIO  
2. `data_processing_sirene_flux` → flux du jour en MinIO  
3. ETL `extract_transform_load_db` lit :
   - `insee/stock/`
   - `insee/flux/`

Le stock + flux servent de socle à la reconstruction SIRENE avant intégration RNE.

---

## 5. Points à creuser plus tard

- Gestion avancée des dates flux  
- Structure exacte des dossiers MinIO  
- Gestion d’erreurs (API INSEE, fichiers incomplets)  
- Alignement des horaires avec RNE et l’ETL  
- Détails sur les métadonnées générées
