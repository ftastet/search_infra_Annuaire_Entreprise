# Traçage des ajouts de champs RNE

Ce document résume les composants à modifier pour ajouter une nouvelle variable issue des JSON RNE (stock ou flux) jusqu'au stockage dans les snapshots `rne_<date>.db.gz` ou `rne.db`.

## Chaîne de transformation

1. **Parsing des JSON (sources RNE)**
   - Fichier : `workflows/data_pipelines/rne/database/rne_model.py`
   - Rôle : modèles Pydantic qui valident et structurent les données brutes (stock ou flux). Ajouter un champ ici permet de parser la variable dès la lecture du JSON.

2. **Modèles cibles UL (objets intermédiaires)**
   - Fichier : `workflows/data_pipelines/rne/database/ul_model.py`
   - Rôle : représentation des entités à insérer dans la base (ex. `UniteLegale`, `Siege`, `Etablissement`, `DirigeantsPP/PM`, etc.). Chaque nouvelle donnée à persister doit avoir un champ correspondant.

3. **Mapping RNE → UL**
   - Fichier : `workflows/data_pipelines/rne/database/map_rne.py`
   - Rôle : copie des champs parsés depuis les modèles RNE vers les modèles UL (ex. `map_rne_company_to_ul`, `map_rne_etablissements_to_ul`). C'est ici que la nouvelle variable doit être affectée au bon objet UL.

4. **Schéma SQLite et insertions**
   - Fichier : `workflows/data_pipelines/rne/database/process_rne.py`
   - Rôle : création des tables (`CREATE TABLE`) et insertions (`INSERT`) dans les fichiers `rne_<date>.db.gz` ou `rne.db`. Ajouter la colonne dans le schéma et l'inclure dans les tuples d'insertion pour stocker la nouvelle variable.

## Étapes pratiques pour ajouter un champ

1. Étendre le modèle Pydantic dans `rne_model.py` pour parser la variable du JSON.
2. Ajouter le champ miroir dans `ul_model.py` pour l'entité concernée.
3. Propager la donnée dans `map_rne.py` (mapping source → cible).
4. Mettre à jour le schéma et les requêtes d'insertion dans `process_rne.py`.

Ces quatre briques constituent le chemin complet pour faire entrer une nouvelle variable RNE depuis le parsing JSON jusqu'au stockage dans les bases `rne_<date>.db.gz` ou `rne.db`.
