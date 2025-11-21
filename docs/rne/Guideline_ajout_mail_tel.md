# Guideline_ajout_contacts_dirigeants_RNE.md

## 1. Objectif

Ajouter des champs de contact (email, téléphone) pour les dirigeants RNE dans les tables SQLite :

- `dirigeant_pp` (personnes physiques)
- `dirigeant_pm` (personnes morales)

en adaptant :

- les modèles Pydantic RNE,
- le schéma SQLite RNE,
- les fonctions de mapping RNE → UL.

---

## 2. Fichiers à modifier

D’après l’analyse du code :

- Modèles JSON RNE (dirigeants) :
  - `workflows/data_pipelines/rne/database/rne_model.py`
    - classes : `DescriptionPersonne`, `PouvoirEntreprise`, `Pouvoir`, `Composition`

- Schéma SQLite RNE :
  - `workflows/data_pipelines/rne/database/process_rne.py`
    - fonction `create_tables` (création de `dirigeant_pp` et `dirigeant_pm`)

- Mapping RNE → UL (dirigeants) :
  - `workflows/data_pipelines/rne/database/map_rne.py`
    - fonctions :
      - `map_rne_dirigeant_pp_to_ul`
      - `map_rne_dirigeant_pm_to_ul`

- Modèles UL cibles (utilisés par le mapping) :
  - `ul_model.py` (vérifier les classes `DirigeantsPP` et `DirigeantsPM`, à étendre si nécessaire)

---

## 3. Étape 1 – Étendre les modèles Pydantic (RNE)

### 3.1 Personnes physiques (dirigeant_pp)

Dans `rne_model.py` :

- Localiser la classe `DescriptionPersonne`.
- Ajouter deux champs optionnels :
  - `email: str | None = None`
  - `telephone: str | None = None`

Ces champs seront accessibles via :

- `composition.pouvoirs[i].individu.descriptionPersonne.email`
- `composition.pouvoirs[i].individu.descriptionPersonne.telephone`

### 3.2 Personnes morales (dirigeant_pm)

Dans `rne_model.py` :

- Localiser la classe `PouvoirEntreprise`.
- Ajouter deux champs optionnels :
  - `email: str | None = None`
  - `telephone: str | None = None`

Ces champs seront accessibles via :

- `composition.pouvoirs[i].entreprise.email`
- `composition.pouvoirs[i].entreprise.telephone`

> Objectif : rendre les contacts disponibles côté Python sans casser le parsing quand ils sont absents.

---

## 4. Étape 2 – Étendre le schéma SQLite (create_tables)

Dans `workflows/data_pipelines/rne/database/process_rne.py`, dans la fonction `create_tables` :

### 4.1 Table `dirigeant_pp`

- Ajouter dans le `CREATE TABLE dirigeant_pp` deux colonnes :

  - `email TEXT`
  - `telephone TEXT`

### 4.2 Table `dirigeant_pm`

- Ajouter dans le `CREATE TABLE dirigeant_pm` deux colonnes :

  - `email TEXT`
  - `telephone TEXT`

Comme la base RNE est reconstruite à chaque run (snapshot), le simple fait de modifier les `CREATE TABLE` suffit :  
pas besoin d’`ALTER TABLE` sur les anciennes bases.

---

## 5. Étape 3 – Étendre les classes UL cibles (dirigeants)

Dans `ul_model.py` (modèle cible UL, utilisé par le mapping) :

- Vérifier les classes :
  - `DirigeantsPP`
  - `DirigeantsPM`

Si ces classes ne possèdent pas encore de champs de contact :

- ajouter :
  - pour `DirigeantsPP` :
    - `email: str | None = None`
    - `telephone: str | None = None`
  - pour `DirigeantsPM` :
    - `email: str | None = None`
    - `telephone: str | None = None`

Objectif : que les objets retournés par `map_rne_dirigeant_pp_to_ul` et `map_rne_dirigeant_pm_to_ul` puissent transporter les contacts jusqu’aux inserts SQLite.

---

## 6. Étape 4 – Adapter les fonctions de mapping (map_rne.py)

Dans `workflows/data_pipelines/rne/database/map_rne.py` :

### 6.1 `map_rne_dirigeant_pp_to_ul`

- Localiser la fonction `map_rne_dirigeant_pp_to_ul(dirigeant_pp_rne, role_entreprise)`.
- Après le mapping des champs existants (nom, prenoms, situation_matrimoniale, etc.), ajouter :

  - `dirigeant_pp_ul.email = getattr(dirigeant_pp_rne, "email", None)`
  - `dirigeant_pp_ul.telephone = getattr(dirigeant_pp_rne, "telephone", None)`

Les valeurs viennent des champs ajoutés dans `DescriptionPersonne`.

### 6.2 `map_rne_dirigeant_pm_to_ul`

- Localiser la fonction `map_rne_dirigeant_pm_to_ul(dirigeant_pm_rne)`.
- Après le mapping des champs existants (denomination, role, forme_juridique, etc.), ajouter :

  - `dirigeant_pm_ul.email = getattr(dirigeant_pm_rne, "email", None)`
  - `dirigeant_pm_ul.telephone = getattr(dirigeant_pm_rne, "telephone", None)`

Les valeurs viennent des champs ajoutés dans `PouvoirEntreprise`.

> Si le JSON RNE contient plusieurs emails / téléphones, définir une stratégie côté modèle ou mapping (ex. ne garder que le premier, concaténer, etc.).

---

## 7. Étape 5 – Tests

### 7.1 Données de test

- Préparer un JSON RNE contenant :
  - au moins un dirigeant personne physique avec `email` et `telephone` dans `descriptionPersonne`,
  - au moins un dirigeant personne morale avec `email` et `telephone` dans `entreprise`.

### 7.2 Exécution

- Lancer le pipeline RNE ou un script ciblé qui :
  - parse ces JSON,
  - crée `rne_<date>.db`,
  - remplit `dirigeant_pp` et `dirigeant_pm`.

### 7.3 Vérifications SQLite

- Ouvrir la base `rne_<date>.db` et vérifier :

  - `SELECT email, telephone FROM dirigeant_pp LIMIT 10;`
  - `SELECT email, telephone FROM dirigeant_pm LIMIT 10;`

- Contrôler que :
  - les colonnes existent,
  - les valeurs sont bien présentes quand le JSON les fournit,
  - elles sont NULL sinon.

---

## 8. Étape 6 – Mise à jour de la documentation

Mettre à jour :

### 8.1 2.4 – Mapping Source → Cible (RNE)

- Ajouter, dans la section `dirigeant_pp` :
  - `descriptionPersonne.email` → `dirigeant_pp.email`
  - `descriptionPersonne.telephone` → `dirigeant_pp.telephone`

- Ajouter, dans la section `dirigeant_pm` :
  - `entreprise.email` → `dirigeant_pm.email`
  - `entreprise.telephone` → `dirigeant_pm.telephone`

### 8.2 2.5 – Modèle de données final RNE (SQLite)

- Ajouter les colonnes dans la description des tables `dirigeant_pp` et `dirigeant_pm` :
  - `email` – TEXT
  - `telephone` – TEXT

### 8.3 2.6 – Consommation de la base RNE

- Mentionner la disponibilité des contacts dirigeants :
  - nouveaux cas d’usage : enrichissement CRM, scoring, conformité, etc.

---

## 9. Checklist synthèse

- [ ] Champs JSON contact identifiés (descriptionPersonne.email / telephone, entreprise.email / telephone).
- [ ] Modèles Pydantic étendus (`DescriptionPersonne`, `PouvoirEntreprise`).
- [ ] Schéma SQLite mis à jour (`dirigeant_pp`, `dirigeant_pm` avec email / telephone).
- [ ] Modèles UL cibles (`DirigeantsPP`, `DirigeantsPM`) étendus.
- [ ] Fonctions de mapping mises à jour (`map_rne_dirigeant_pp_to_ul`, `map_rne_dirigeant_pm_to_ul`).
- [ ] Pipeline testé de bout en bout avec JSON de test.
- [ ] Documentation (2.4, 2.5, 2.6) mise à jour en cohérence.
