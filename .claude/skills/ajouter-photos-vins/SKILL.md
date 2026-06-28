---
name: ajouter-photos-vins
description: >-
  Ajoute des photos de vins au repo GitHub `photo` (parlezmoidevin-arch/photo,
  branche main) et relie leur lien raw dans la table Supabase `wines` du projet
  Parlez-moi de Vin. Utiliser dès que l'utilisateur veut ajouter / importer /
  uploader des photos de bouteilles, domaines, propriétaires, cartes ou parcelles
  et les rattacher à un vin, surtout quand les fichiers sont nommés au format
  `categorie__domaine__cuvee.ext`.
---

# Ajouter des photos de vins (GitHub + Supabase)

Workflow pour publier des photos dans le dépôt `photo` et écrire leur lien raw
dans la base de données du site parlezmoidevin.fr.

## Contexte fixe du projet

- **Repo GitHub** : `parlezmoidevin-arch/photo`, branche **`main`** (les liens raw
  pointent vers `main` — c'est là que les photos doivent atterrir).
- **Projet Supabase** : `Parlez-moi de Vin`, project_id **`fybwclwkxzvmfyksixct`**
  (région eu-west-3). Table cible : **`public.wines`**.
- **Base URL des liens raw** :
  `https://raw.githubusercontent.com/parlezmoidevin-arch/photo/main/[Dossier]/[fichier]`

## Convention de nommage des fichiers

Chaque fichier est nommé `categorie__domaine__cuvee.ext` (double underscore `__`
comme séparateur).

- `categorie` : un de `bouteille`, `domaine`, `proprietaires`, `carte`, `parcelle`.
- `domaine` : nom du domaine (pour le rapprochement).
- `cuvee` : nom de la cuvée — **obligatoire pour `bouteille` et `parcelle`**,
  facultatif/ignoré pour `domaine`, `proprietaires`, `carte`.
- `ext` : extension d'origine (`.png`, `.jpg`, `.jpeg`, `.webp`, …).

## Mapping catégorie → dossier → colonne

| Catégorie       | Dossier (nom exact, **accents inclus**) | Colonne `wines`            | Type        |
|-----------------|------------------------------------------|----------------------------|-------------|
| `bouteille`     | `Bouteilles`                             | `image_url`                | écrase      |
| `domaine`       | `Domaines`                               | `photo_domaine_url`        | écrase      |
| `proprietaires` | `Propriétaires` ⚠️ accent é              | `photo_proprietaires_url`  | écrase      |
| `carte`         | `Cartes` (créer le dossier s'il manque)  | `carte_france_url`         | écrase      |
| `parcelle`      | `Parcelles`                              | `photos_parcelles` (array) | **ajoute**  |

⚠️ Le dossier propriétaires s'écrit **`Propriétaires`** (avec accent). Les liens
en base utilisent ce nom littéral.

## Convention des liens raw

- Le chemin reproduit **littéralement** le nom de dossier et le nom de fichier,
  **sans encodage d'URL** : les espaces et les accents restent tels quels.
  Exemple réel en base : `.../main/Bouteilles/Photo gaby.png`.
- Conserve le nom de fichier d'origine (n'invente pas, ne renomme pas sans
  raison). Si tu renommes, garde une trace dans le récap.

## Procédure (pour chaque fichier)

1. **Parser le nom** `categorie__domaine__cuvee.ext`.
   - Nom mal formé (pas de `__`, catégorie inconnue, cuvée manquante pour
     `bouteille`/`parcelle`) → **ne devine pas**, marque ⚠️ et demande à
     l'utilisateur.

2. **Placer le fichier** dans le bon dossier du repo (voir mapping). Crée le
   dossier `Cartes` s'il n'existe pas encore.

3. **Construire le lien raw** :
   `https://raw.githubusercontent.com/parlezmoidevin-arch/photo/main/[Dossier]/[fichier]`

4. **Trouver le(s) vin(s)** dans `wines` — accents et casse ignorés :
   - Récupère d'abord les candidats :
     `SELECT id, domaine_name, cuvee FROM wines ORDER BY domaine_name;`
   - `bouteille` / `parcelle` → rapprochement sur **`domaine_name` + `cuvee`**
     (la cuvée du nom de fichier vs `wines.cuvee`).
   - `domaine` / `proprietaires` / `carte` → rapprochement sur **`domaine_name`
     seul** ; ces photos sont au niveau du domaine et concernent donc **toutes
     les lignes** de ce domaine (en base, un même domaine partage la même valeur
     sur ses cuvées).
   - Fais le rapprochement toi-même (insensible aux accents/casse). **Si aucune
     correspondance, ou plusieurs candidats ambigus → marque ⚠️ et DEMANDE.
     Ne devine jamais.**

5. **Écrire le lien** dans la bonne colonne via `execute_sql` :
   - `bouteille` → `UPDATE wines SET image_url = '<lien>' WHERE id = '<id>';`
   - `domaine` → `UPDATE wines SET photo_domaine_url = '<lien>' WHERE domaine_name = '<domaine>';`
   - `proprietaires` → `UPDATE wines SET photo_proprietaires_url = '<lien>' WHERE domaine_name = '<domaine>';`
   - `carte` → `UPDATE wines SET carte_france_url = '<lien>' WHERE domaine_name = '<domaine>';`
   - `parcelle` → **ajouter sans écraser** l'existant :
     ```sql
     UPDATE wines
     SET photos_parcelles = array_append(coalesce(photos_parcelles, '{}'), '<lien>')
     WHERE id = '<id>' AND NOT ('<lien>' = ANY(coalesce(photos_parcelles, '{}')));
     ```
     (le garde-fou `NOT ... = ANY` évite les doublons si on relance).

## Publier sur GitHub

Les fichiers doivent arriver sur la branche `main` du repo `photo` :
- Si tu travailles dans une copie locale du repo `photo` : copie les fichiers
  dans les bons dossiers, `git add`, commit, puis `git push` vers `main`.
- Sinon, utilise les outils MCP GitHub (`mcp__github__push_files` /
  `mcp__github__create_or_update_file`) en ciblant `parlezmoidevin-arch/photo`
  branche `main`.
- Un lien raw n'est valide qu'une fois le fichier réellement commité sur `main`.

## Récapitulatif final (obligatoire)

Termine **toujours** par un tableau récap, une ligne par fichier :

| Fichier | Dossier | Vin trouvé | Colonne mise à jour |
|---------|---------|------------|---------------------|

- Signale en **⚠️** tout fichier dont le nom est mal formé ou dont le vin n'a
  pas été trouvé, et **pose la question** à l'utilisateur — ne devine jamais.
- Indique aussi les fichiers ignorés (doublon déjà présent) le cas échéant.

## Règle d'or

**Ne devine jamais.** En cas de doute sur la catégorie, le domaine, la cuvée ou
le vin à mettre à jour, arrête-toi et demande à l'utilisateur avant d'écrire en
base ou de committer.
