---
plan:
  id: '3'
  name: 3-retro-doc-catalogue-first.md
  link: https://github.com/ehouriez/retro-doc/blob/main/doc/roadmap/done/3-retro-doc-catalogue-first.md
status: done
date: 2026-11-03
description: >
  Transposer deux emprunts du skill retrodoc-run de Damien vers retro-doc :
  auto-contrôle du researcher (Lot A) et mode catalogue-first + affinage ciblé
  via --expand (Lot B). Objectif : réduire le coût d'écriture.
priority: high
complexity: L
intent: null
scope:
  modules:
    - SKILL.md
    - references/delegation.md
    - references/cache-schema.md
  patterns:
    - "**/*.md"
issue:
  id: 3
  url: https://github.com/ehouriez/retro-doc/issues/3
---

# [🗂️] Plan #3 - retro-doc catalogue-first mode + researcher self-check

## Objectif

Transposer deux emprunts du skill `retrodoc-run` de Damien (analysé le 2026-11-03)
vers notre `retro-doc`, **strictement orientés compréhension** (Diataxis inchangé) :
réduire le coût d'écriture — poste dominant mesuré au plan #2 (l'écriture de N docs
d'un coup, pas la lecture) — en produisant d'abord un catalogue léger, puis en
n'écrivant que les entités affinées à la demande ; et fiabiliser la recherche via
un auto-contrôle du researcher.

## Contexte de la décision (auto-portant)

Comparatif mené le 2026-11-03 entre `retrodoc-run` (Damien) et `retro-doc` (nous).
Le skill de Damien est un **générateur de dossier de migration/chiffrage** (1
orchestrateur + 7 sous-agents experts séquentiels, structure métier RM/RS/EPIC/FEAT,
chiffrage Excel). Ce n'est **pas** le même produit que `retro-doc` (doc Diataxis de
compréhension). Deux idées seulement sont transposables sans dénaturer `retro-doc` :

1. **Son découpage « catalogue macro → affinage borné à la demande »**
   (`--deep` produit les EPIC, `--features-refine EPIC-nnn` affine un EPIC à la fois).
   C'est l'emprunt structurant : il attaque le coût d'écriture.
2. **Son auto-contrôle de fin d'agent** (checklist de complétude/traçabilité).

Écartés explicitement (dénatureraient le produit) : structure métier RM/RS/EPIC/FEAT,
sous-agents spécialisés par préoccupation, registre d'IDs anti-collision, chiffrage,
modèle figé en frontmatter.

## Décisions techniques

| Décision | Choix retenu | Justification |
|----------|-------------|---------------|
| Déclenchement catalogue | **Catalogue-first systématique** (défaut) | Le skill n'a pas d'existant à casser (testé uniquement sur auxitum). Pas de détection d'intention à maintenir. |
| Déclenchement affinage | Flag `--expand <id>` ; **`--expand all` par défaut** si absent | Explicite et prévisible ; `all` par défaut = comportement « documente tout » historique. |
| Terminologie | `--expand` | Rejeté `--include` (faux-ami : sens universel = filtre de sélection `rsync`/`grep`, pas transformation). `--expand` dit la transformation « ligne de catalogue → fiche complète ». |
| État catalogue | `entities[].status: catalogued \| generated` dans le cache v2 + marqueur INDEX | Source unique et structurée du reste à faire ; survit aux sessions. |
| Rendu INDEX | Lister **toutes** les entités, ✅ générée / ⬜ cataloguée | L'INDEX devient le tableau de bord de progression. |
| Portée produit | Diataxis inchangé (Reference + Explanation + INDEX) | On n'emprunte que le *mode d'exécution*, pas la structure de sortie de Damien. |

## Périmètre

### Inclus
- **Lot A** — Auto-contrôle du researcher dans le prompt de délégation.
- **Lot B** — Mode catalogue-first par défaut ; flag `--expand <id>|all` ; champ de
  statut `catalogued|generated` dans le cache v2 ; rendu INDEX avec marqueurs.

### Hors scope
- Structure métier (RM/RS/EPIC/FEAT), sous-agents spécialisés, chiffrage d'effort,
  registre d'IDs. (Si un besoin « migration/chiffrage » émerge → skill distinct ;
  celui de Damien couvre déjà ce terrain.)
- Modification de la méthode Diataxis elle-même.

## Étapes

- [x] Étape 1 — **Lot A** : ajouter une checklist d'auto-contrôle en fin de prompt researcher — couverture complète du périmètre / chaque claim non-trivial porte `source`+`confidence` / aucune entité ou relation inventée. Dans `SKILL.md` (étape 4, delegation) + `references/delegation.md` (contrat concept-map). (XS · standard → Sonnet)
- [x] Étape 2 — **Lot B** : étendre le schéma de cache v2 avec `entities[].status` ∈ {`catalogued`, `generated`}. Rétrocompat : entité sans `status` dans un cache v2 existant → traitée comme `generated` (comportement actuel). Documenter dans `references/cache-schema.md` + étape « Cache persistence » de `SKILL.md`. (M · reasoning → Opus)
- [x] Étape 3 — **Lot B** : mode **catalogue par défaut**. Un run sans `--expand` produit l'inventaire : cartographie légère (glob/grep, déjà faite en étape 4) → une entité par sous-système avec `id` + une ligne de description, **sans déléguer l'archéologie profonde** et **sans écrire de fiches**. Persister chaque entité `status: catalogued`. Écrire/mettre à jour l'INDEX. Décrire ce branchement dans `SKILL.md` (étape 3 « Regeneration mode » ou nouvelle sous-section amont). (L · reasoning → Opus)
- [x] Étape 4 — **Lot B** : flag `--expand <id>` et `--expand all` (défaut si le flag est absent OU donné sans argument). Pour chaque entité ciblée : déléguer la recherche (researcher Sonnet, inchangé), écrire Reference + Explanation, basculer `status: generated`, mettre à jour le cache et l'INDEX. `--expand all` = affine toutes les entités `catalogued`. Résolution de `<id>` depuis `.retro-doc.yml`. (L · reasoning → Opus)
- [x] Étape 5 — **Lot B** : rendu INDEX listant **toutes** les entités (générées ET cataloguées) avec marqueur ✅ générée / ⬜ cataloguée, par quadrant/sous-système. L'INDEX reste régénéré à chaque run (catalogue comme expand). Dans `SKILL.md` (étape INDEX). (M · standard → Sonnet)
- [x] 🧪 Tests — Rédiger et exécuter la procédure de test (avant-dernière étape, obligatoire)
- [x] ✅ Validation — Vérifier les résultats des tests et clôturer (dernière étape, obligatoire)

> **Lots** : Lot A = étape 1 (livrable immédiat, indépendant). Lot B = étapes 2→5
> (archi catalogue-first ; ordre : cache → catalogue → expand → INDEX).

## Spécification détaillée (auto-portant)

### Lot A — checklist researcher
Ajouter à la fin du prompt du sous-agent researcher (section « Delegation protocol »
de `references/delegation.md` et son reflet dans `SKILL.md` étape 4) un bloc
« Self-check avant de rendre » :
- [ ] Chaque entité du périmètre annoncé est couverte (aucune omise).
- [ ] Chaque claim non-trivial porte `source` (`file:line`) et `confidence`.
- [ ] Aucune entité, fonction, relation ou endpoint inventé (tout est traçable au code).
- [ ] Les recherches récursives ont exclu les arbres vendored/build (hygiène déjà requise).

### Lot B — cycle de vie d'une entité
```
run sans --expand           → toutes entités cataloguées (status: catalogued), INDEX ⬜
run --expand <id>           → entité <id> : recherche + écriture → status: generated, INDEX ✅
run --expand all (défaut)   → toutes les entités catalogued → generated
run incrémental (git diff)  → logique du plan #2 inchangée, s'applique aux entités generated
```
- `--expand all` est le **défaut** quand aucun `--expand` n'est fourni ? **NON** : le
  défaut sans flag = **catalogue seul**. `--expand all` ne s'applique que si l'utilisateur
  écrit `--expand` sans id, ou `--expand all`. (Décision Q3+Q4 : catalogue-first
  systématique ; `all` = défaut *de l'argument de `--expand`*, pas du run.)
- Interaction avec l'incrémental du plan #2 : le `git diff` + invalidation de deps
  ne concerne que les entités déjà `generated`. Une entité `catalogued` n'a rien
  d'écrit à invalider.

## Tests

> Projet cible : `/mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/auxitum`.

### Prérequis pour les tests

```bash
echo "=== Retro-doc Skill Path ==="
SKILL_PATH=/mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/skills/retro-doc/SKILL.md

echo "=== Target Project ==="
TARGET=/mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/auxitum

echo "=== Cache File Path ==="
CACHE=$TARGET/docs/retro/.retro-doc.yml
```

### Procédure de test

**Bloc A — Lot A : self-check researcher (régression)**

```bash
echo "=== Verify Self-check Present In SKILL.md Step 4 ==="
grep -n "Self-check" /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/skills/retro-doc/SKILL.md

echo "=== Verify Self-check In delegation.md ==="
grep -n "Self-check\|self-check" /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/skills/retro-doc/references/delegation.md
```

**Bloc B — Lot B : cache v2 schema (régression)**

```bash
echo "=== Verify Status Field In cache-schema.md ==="
grep -n "status.*catalogued\|catalogued.*generated" /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/skills/retro-doc/references/cache-schema.md

echo "=== Verify Status Field In SKILL.md Step 9 ==="
grep -n "status.*catalogued\|catalogued.*generated" /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/skills/retro-doc/SKILL.md
```

**Bloc C — Lot B : catalogue-first (régression + INDEX ✅/⬜)**

```bash
echo "=== Verify Catalogue Mode In SKILL.md Step 3 ==="
grep -n "CATALOGUE\|catalogue" /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/skills/retro-doc/SKILL.md | head -20

echo "=== Verify expand Flag In SKILL.md ==="
grep -n "\-\-expand" /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/skills/retro-doc/SKILL.md | head -20
```

**Bloc D — Étape 5 : rendu INDEX ✅/⬜**

```bash
echo "=== Verify INDEX Template In templates.md ==="
grep -n "Status\|✅\|⬜" /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/skills/retro-doc/references/templates.md

echo "=== Verify Step 8 Lists All Entities ==="
grep -n "every entity\|generated and\|catalogued\|always fully regenerated" /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/skills/retro-doc/SKILL.md

echo "=== Verify Regeneration Rule Covers All Run Types ==="
grep -A3 "All run types" /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/skills/retro-doc/SKILL.md
```

**Résultats attendus :**
- Bloc A : `grep` retourne au moins une ligne contenant "Self-check" dans SKILL.md Step 4 et dans delegation.md
- Bloc B : `grep` retourne des lignes contenant `status: catalogued` et la valeur `generated` dans les deux fichiers
- Bloc C : `grep` retourne les mots `CATALOGUE` et `--expand` dans SKILL.md
- Bloc D :
  - `templates.md` : lignes contenant `Status`, `✅`, `⬜` dans le template INDEX
  - `SKILL.md` : phrases « every entity », « generated and », « catalogued », « always fully regenerated » présentes dans Step 8
  - `SKILL.md` : bloc « All run types » présent avec explication de la régénération complète

### Résultats joués et vérifiés

| Date | Test | Attendu | Observé | Verdict |
|------|------|---------|---------|---------|
| 2026-11-03 | Bloc A — self-check SKILL.md Step 4 | ligne contenant "Self-check" | l.304 `Self-check before returning (mandatory in the prompt)` | ✅ |
| 2026-11-03 | Bloc A — self-check delegation.md | ligne contenant "Self-check" | l.60 `Self-check before returning (mandatory in the prompt)` | ✅ |
| 2026-11-03 | Bloc B — status field cache-schema.md | `status: catalogued\|generated` présent | l.40 table + l.70 YAML exemple | ✅ |
| 2026-11-03 | Bloc B — status field SKILL.md Step 9 | `status: catalogued\|generated` présent | plusieurs occurrences dont l.470 YAML + l.484 règle catalogue run | ✅ |
| 2026-11-03 | Bloc C — CATALOGUE mode SKILL.md | `CATALOGUE` présent dans Step 3 | l.141 heading, l.155-159 table + description | ✅ |
| 2026-11-03 | Bloc C — --expand flag SKILL.md | `--expand` présent dans Step 3 | l.155-157 table, l.171, l.182 + Hard rules l.582-583 | ✅ |
| 2026-11-03 | Bloc D — template INDEX templates.md | `Status\|✅\|⬜` présents | l.87 légende, l.89 headers, l.91-92 exemples ✅/⬜ | ✅ |
| 2026-11-03 | Bloc D — Step 8 "every entity" SKILL.md | "every entity (generated and catalogued)" + "always fully regenerated" | l.410-414 conforme | ✅ |
| 2026-11-03 | Bloc D — "All run types" SKILL.md | bloc couvrant catalogue + expand full + expand incrémental | l.428-431 présent et correct | ✅ |

## Journal de session

### Session 2026-11-03
- ✅ Fait : analyse comparative retrodoc-run (Damien) vs retro-doc ; cadrage ; création issue #3 + plan.
- 📋 Prochain : implémentation (Lot A puis Lot B, étapes 2→5).

### Session 2026-11-03 (suite)
- ✅ Étape 1 (Lot A) : checklist self-check researcher ajoutée dans `SKILL.md` (Step 4, item 2) et `references/delegation.md` (Delegation protocol, item 3). 4 items : couverture périmètre, `source`+`confidence`, pas d'invention, hygiène vendored/build.
- 📋 Prochain : Étape 2 (Lot B) — schéma cache v2 avec `entities[].status` (M · reasoning → Opus).

### Session 2026-11-03 (étapes 2→4)
- ✅ Étape 2 : champs `status` (`catalogued|generated`) et `description` ajoutés au schéma `entities[]` dans `references/cache-schema.md` (table + exemple + invariants, dont rétrocompat « pas de `status` → `generated` ») et dans `SKILL.md` Step 9 (liste de build + YAML + branchement catalogue/expand full/incrémental).
- ✅ Étape 3 : `SKILL.md` Step 3 renommé « Run mode (catalogue-first, then full vs incremental) » ; nouvelle sous-section « Catalogue vs expand (decided first) » — run sans `--expand` = inventaire seul (cartographie légère, `id` + `description`, `status: catalogued`, INDEX, STOP, aucune délégation ni fiche).
- ✅ Étape 4 : flag `--expand <id>[,<id>…]` / `--expand all` / `--expand` nu (= all) documenté dans la même sous-section ; résolution de `<id>` via `.retro-doc.yml`, bascule `generated`, note interaction avec l'incrémental (plan #2) et cas `<id>` inconnu. Règle dure « Catalogue-first » + ligne `Mode:` du rapport (Step 11) mises à jour.
- ℹ️ `references/delegation.md` volontairement non modifié : le researcher reste inchangé (catalogue ne délègue pas ; expand réutilise le contrat existant).
- 📋 Prochain : Étape 5 (rendu INDEX ✅/⬜, hors de la demande courante), puis 🧪 Tests contre auxitum.

### Session 2026-11-03 (étape 5)
- ✅ Étape 5 : `SKILL.md` Step 8 mis à jour — INDEX liste désormais toutes les entités (générées + cataloguées) avec marqueurs ✅/⬜, colonne « Docs » avec liens pour les générées et `—` pour les cataloguées, régénération complète à chaque run (catalogue comme expand). Note « Incremental runs » remplacée par une note couvrant tous les types de run. `references/templates.md` template INDEX aligné (nouvelle table `Status | Entity | Docs | Description`, légende ✅/⬜ dans le bloc `>`).
- ✅ Plan clôturé : 9/9 tests passés. Tous les lots (A + B étapes 1→5) livrés et validés.
