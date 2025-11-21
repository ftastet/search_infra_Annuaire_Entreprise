# 2.1 – Architecture du pipeline RNE

## 1. Objectif
Décrire l’architecture fonctionnelle et technique utilisée pour acquérir, stocker, transformer et exposer les données RNE.

## 2. Vue d’ensemble
(Schéma global du flux : stock → flux → base consolidée → MinIO)

## 3. Composants techniques
- Orchestrateur
- Stockage objets
- Base SQLite
- Librairies et validations
- Gestion des métadonnées

## 4. Artefacts produits
- Stock RNE (ZIP/JSON)
- Flux quotidiens
- Base consolidée `rne_<date>.db.gz`
- Fichier `latest_rne_date.json`

## 5. Processus haute-niveau
- Acquisition du stock
- Acquisition des flux
- Consolidation
- Mise à disposition dans MinIO

## 6. Points forts de l’architecture
- Versionnée
- Quotidienne
- Idempotente
- Basée sur JSON natifs INPI
