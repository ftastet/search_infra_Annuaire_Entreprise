# Guideline_ajout_contacts_dirigeants_RNE.md

## 1. Objectif

Ajouter des champs de contact (email, téléphone) pour les dirigeants RNE dans les tables :
- `dirigeant_pp` (personnes physiques),
- `dirigeant_pm` (personnes morales),

en modifiant uniquement le pipeline RNE (modèles + mapping + schéma SQLite).

---

## 2. Fichiers à modifier (ciblage)

1. `rne_model.py`
   - Contient les modèles Pydantic décrivant le JSON RNE, notamment :
     - les blocs `composition.pouvoirs` pour les dirigeants,
     - `descriptionPersonne` pour les personnes physiques,
     - les attributs de l’entreprise pour les dirigeants personnes morales.

2. `process_rne.py`
   - Crée les tables SQLite RNE (dont `dirigeant_pp` et `dirigeant_pm`).
   - Contient les fonctions de mapping :
     - `map_rne_dirigeant_pp_to_ul` → insertion dans `dirigeant_pp`,
     - `map_rne_dirigeant_pm_to_ul` → insertion dans `dirigeant_pm`,
     - et la logique d’insertion globale (boucles qui parcourent les JSON).

3. (Optionnel, si propagation vers SIRENE)
   - Script ETL SIRENE (`3_ETL_SIRENE.py` ou équivalent) qui :
     - attache `rne.db` en `db_rne`,
     - relit `db_rne.dirigeant_pp` et `db_rne.dirigeant_pm`,
     - alimente les tables dirigeant_pp / dirigeant_pm de `sirene.db`.

---

## 3. Étape 1 – Ajouter les champs dans `rne_model.py`

Objectif : rendre les champs `email` et `telephone` accessibles dans le code Python.

1. Localiser dans `rne_model.py` :
   - la classe qui représente `composition.pouvoirs[].individu.descriptionPersonne`,
   - la classe qui représente `composition.pouvoirs[].entreprise` (dirigeant PM).

2. Ajouter dans ces classes des attributs optionnels pour les contacts, par exemple :
   - `email: Optional[str]`
   - `telephone: Optional[str]`
   - ou toute autre structure cohérente avec le JSON réel (ex. `emails: List[str]`).

3. Vérifier :
   - que les champs sont bien déclarés en optionnels,
   - que le parsing fonctionne même si les champs sont absents dans certains JSON.

---

## 4. Étape 2 – Étendre le schéma SQLite dans `process_rne.py`

Objectif : ajouter des colonnes de contact dans les tables RNE.

1. Dans `process_rne.py`, localiser le bloc qui crée les tables :
   - `CREATE TABLE dirigeant_pp (...)`
   - `CREATE TABLE dirigeant_pm (...)`

2. Ajouter les colonnes suivantes (noms à adapter si besoin) :
   - Dans `dirigeant_pp` :
     - `email_dirigeant TEXT`
     - `telephone_dirigeant TEXT`
   - Dans `dirigeant_pm` :
     - `email_dirigeant TEXT`
     - `telephone_dirigeant TEXT`

3. Si le schéma RNE est recréé à chaque run (ce qui est le cas pour la base consolidée), il suffit de modifier les `CREATE TABLE` :
   - pas besoin d’ALTER TABLE, la base repart de zéro pour chaque snapshot.

---

## 5. Étape 3 – Adapter `map_rne_dirigeant_pp_to_ul` et `map_rne_dirigeant_pm_to_ul`

Objectif : remplir effectivement les colonnes de contact.

1. Dans `process_rne.py`, localiser :
   - `map_rne_dirigeant_pp_to_ul` (mapping RNE → `dirigeant_pp`),
   - `map_rne_dirigeant_pm_to_ul` (mapping RNE → `dirigeant_pm`).

2. Étendre les dictionnaires retournés par ces fonctions :

   - Pour `dirigeant_pp` :
     - Lire l’email / téléphone dans l’objet Pydantic (ex. `descriptionPersonne.email`, `descriptionPersonne.telephone` ou chemin réel équivalent).
     - Ajouter les clés :
       - `"email_dirigeant": <valeur source>`
       - `"telephone_dirigeant": <valeur source>`

   - Pour `dirigeant_pm` :
     - Lire les contacts dans l’objet entreprise du dirigeant (ex. `entreprise.email`, `entreprise.telephone`).
     - Ajouter les mêmes clés dans le dict retourné.

3. Si plusieurs emails / téléphones sont possibles :
   - décider d’une règle simple :
     - soit garder le premier,
     - soit concaténer avec un séparateur (ex. `";"`),
     - soit sérialiser en JSON (champ texte).

---

## 6. Étape 4 – Tests rapides

1. Préparer un JSON RNE d’exemple avec :
   - au moins un dirigeant personne physique avec email / téléphone,
   - au moins un dirigeant personne morale avec email / téléphone.

2. Lancer le script / DAG qui :
   - parse ce JSON,
   - alimente `rne.db`,
   - crée la table `dirigeant_pp` et `dirigeant_pm`.

3. Vérifier dans SQLite :
   - `SELECT email_dirigeant, telephone_dirigeant FROM dirigeant_pp LIMIT 10;`
   - `SELECT email_dirigeant, telephone_dirigeant FROM dirigeant_pm LIMIT 10;`

4. Confirmer que :
   - les colonnes existent,
   - les valeurs sont correctement remplies (ou NULL quand absentes).

---

## 7. (Optionnel) Étape 5 – Propagation dans `sirene.db`

Si tu veux utiliser ces contacts dans les tables dirigeants de `sirene.db` :

1. Identifier dans l’ETL Sirene :
   - où `db_rne.dirigeant_pp` et `db_rne.dirigeant_pm` sont lus,
   - où les colonnes sont projetées vers les tables SIRENE `dirigeant_pp` / `dirigeant_pm`.

2. Ajouter les colonnes contact dans :
   - les définitions des tables SIRENE,
   - les requêtes `INSERT` / `SELECT` qui copient depuis `db_rne.dirigeant_pp` / `db_rne.dirigeant_pm`.

3. Mettre à jour la documentation SIRENE si nécessaire.

---

## 8. Mise à jour documentaire

Après implémentation :

1. Mettre à jour `2.4_RNE_Mapping.md` :
   - ajouter les lignes :
     - `descriptionPersonne.email` → `dirigeant_pp.email_dirigeant`
     - `descriptionPersonne.telephone` → `dirigeant_pp.telephone_dirigeant`
     - `entreprise.email` → `dirigeant_pm.email_dirigeant`
     - `entreprise.telephone` → `dirigeant_pm.telephone_dirigeant`

2. Mettre à jour `2.5_RNE_Tables_Finales.md` :
   - ajouter les colonnes correspondantes dans la description des tables `dirigeant_pp` et `dirigeant_pm`.

3. Mettre à jour `2.6_RNE_ConsommationEntreprise.md` :
   - signaler la présence de ces contacts comme nouvel usage potentiel (ex. enrichissement CRM).

---
