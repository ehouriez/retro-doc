---
plan:
  id: '2'
  name: 2-retro-doc-token-optimization.md
  link: https://github.com/ehouriez/retro-doc/blob/main/doc/roadmap/2-retro-doc-token-optimization.md
status: active
date: 2026-11-03
description: >
  Reduce retro-doc token cost by delegating code archaeology to a configurable
  general-purpose sub-agent returning a provenance-tagged concept map, persisting
  that map plus the Git SHA in an extended .retro-doc.yml, and regenerating
  incrementally via git diff with one-level dependent invalidation. No quality
  regression, stays prompt-only.
priority: high
complexity: L
intent: null
scope:
  modules:
    - SKILL.md
    - references/
  patterns:
    - "**/*.md"
issue:
  id: 2
  url: https://github.com/ehouriez/retro-doc/issues/2
---

# [⚡] Plan #2 - retro-doc token optimization

## Objectif

Réduire le coût en tokens de `retro-doc` en déléguant l'archéologie du code (le
poste de tokens dominant) à un sous-agent `general-purpose` configurable qui
renvoie un concept map à provenance `file:line`, en persistant ce map + le SHA
Git dans un `.retro-doc.yml` étendu, et en régénérant en incrémental via diff
Git. Objectif secondaire non négociable : aucune régression de qualité, le skill
reste 100 % prompt-only.

## Périmètre

### Inclus
- `SKILL.md` : refonte du Step 3 (archéologie → délégation) + nouveaux steps
  régénération incrémentale, persistance du cache et garde-fou qualité.
- `references/` : contrat du concept map (format, provenance, confidence, deps)
  et protocole de délégation au sous-agent.
- Schéma `.retro-doc.yml` étendu : `language` (existant) + `source_sha` +
  `entities[]` (sources `file:line`, `deps`, `confidence`, quadrants produits).
- `allowed-tools` : ajout de `Agent`/`Task`.

### Hors scope
- Packaging / marketplace, détection de langue, logique de sortie `retro/`,
  scan secrets, garde no-commit/push — tous **inchangés**.
- Mesure runtime du gain token (aucun outil de comptage accessible).

## Étapes

- [ ] Étape 1 — Schéma `.retro-doc.yml` étendu (`language`, `source_sha`, `entities[]{sources[file:line], deps[], confidence, quadrants}`) + extension `allowed-tools` (`Agent`/`Task`) + doc du format cache dans `references/` (M · reasoning → Opus)
- [ ] Étape 2 — Step 3 refondu : délégation à un sous-agent `general-purpose` configurable (`Agent` frais avec override `model` vers le tier `standard`, **jamais un `fork`** qui hériterait du contexte et ignorerait l'override), cartographie légère intégrée, parallélisme adaptatif au scope, contrat de sortie = concept map à provenance, fallback inline si `Agent` indisponible. Documenter les **deux rôles** (rédacteur / chercheur) et émettre une **recommandation non bloquante** : rédacteur en tier `reasoning` (L · reasoning → Opus)
- [ ] Étape 3 — Persistance du cache enrichi en fin de run (map + SHA courant) dans `.retro-doc.yml`, cohérence avec Step 7 (INDEX) et Step 9 (stop) (S · standard → Sonnet)
- [ ] Étape 4 — Régénération incrémentale : lecture cache + SHA, `git diff <sha>..HEAD`, défaut incrémental si cache présent, mots-clés `full`/`incrémental`, rétrocompat = full forcé si cache v1 (M · reasoning → Opus)
- [ ] Étape 5 — Invalidation par entité + propagation aux dépendants directs (1 niveau) depuis le graphe `deps` du map (M · reasoning → Opus)
- [ ] Étape 6 — Garde-fou qualité : marquage `confidence` par claim par le sous-agent, re-lecture ciblée par le **rédacteur lui-même** (le modèle appelant, pas une gate ni un sous-agent) via ses propres `Read`/`Grep` sur les `file:line` des claims `confidence: low`, avant écriture. Interne au run, silencieux, sans interaction utilisateur (S · reasoning → Opus)
- [ ] 🧪 Tests — Rédiger et exécuter la procédure de validation (3 scénarios : neuf, rétrocompat, incrémental sur diff)
- [ ] ✅ Validation — Vérifier les résultats et clôturer

## Rôles et modèles

Le skill fait collaborer **deux rôles** portés par **deux modèles distincts** :

| Rôle | Qui l'exécute | Tier | Responsabilité |
|---|---|---|---|
| **Rédacteur** | Le modèle qui exécute `/retro-doc` (le modèle **appelant**, déjà actif) | `reasoning` recommandé | Partition Diataxis, arbitrage des claims `confidence: low`, prose Explanation (trade-offs, rationale), écriture finale |
| **Chercheur** | Sous-agent `general-purpose` lancé via le tool `Agent` (modèle surchargé) | `standard` | Cartographie + lecture lourde du code, production du concept map à provenance `file:line` |

**Pourquoi le rédacteur devrait très souvent être `reasoning`** : la lecture
exhaustive du code (poste de tokens dominant) est **déjà déportée** sur le
chercheur `standard`. Garder le rédacteur en `reasoning` ne pèse donc que sur
l'**output** (poste mineur), précisément là où le jugement paie :

1. Le chercheur renvoie le map ; chaque claim porte un `confidence` et sa provenance `file:line`.
2. Le rédacteur parcourt le map et, pour les seuls claims `confidence: low`, relit lui-même quelques lignes via `Read`/`Grep` sur les `file:line` concernés (re-lecture **ciblée**, pas le fichier entier).
3. Il écrit la doc à partir du map corrigé.

Tout est **interne au run**, automatique et silencieux : **aucune gate, aucun
`/model`, aucune `AskUserQuestion`**. Le skill ne peut pas forcer un changement
de modèle (action IDE) — il **recommande** `reasoning` pour le rédacteur sans
bloquer, et reste fonctionnel si le rédacteur est `standard` (qualité moindre
sur l'arbitrage des claims douteux).

## Contexte d'implémentation (auto-portant)

> Tout ce qu'un agent frais (post-`/clear`) doit savoir pour implémenter sans le
> contexte de la session de cadrage. **Lire `SKILL.md` + `references/` avant toute
> édition** (leurs numéros de ligne bougeront — se repérer par titres de section).

### État de départ du skill (v1.0.0)

| Fichier | Rôle actuel |
|---|---|
| `SKILL.md` (~320 l.) | Moteur prompt-only en 10 steps (0→9). `allowed-tools`: Bash, Read, Write, Edit, Grep, Glob, AskUserQuestion |
| `references/diataxis.md` | Matrice de partition Diataxis (quel quadrant par type d'entité) |
| `references/templates.md` | Templates Reference / Explanation / INDEX / How-to / Tutorial |

Steps actuels de `SKILL.md` : **0** Scope & intent · **1** Output language
(detect/persist `.retro-doc.yml`) · **2** Output location (`<base>/retro/`) ·
**3** Codebase archaeology (concept map interne) · **4** Diataxis partition ·
**5** Reference · **6** Explanation · **7** INDEX & README link · **8** Quality
self-review · **9** Secret scan + STOP.

### Invariants — à NE PAS casser

- **Prompt-only** : aucun script runtime, aucune dépendance externe. Le tool `Agent` est un outil de l'agent, pas une dépendance → acceptable.
- Sortie **isolée** sous `<base>/retro/` ; un seul lien ajouté au README ; **jamais de commit/push**.
- Détection de langue + `.retro-doc.yml` existant (clé `language`) : **rétrocompat totale**.
- Scan secrets (Step 9) conservé ; exactitude `file:line` (déjà validée v1) préservée.

### Schéma cible `.retro-doc.yml` (v2)

```yaml
# <base>/retro/.retro-doc.yml
language: fr                 # v1 — inchangé
schema_version: 2            # absent ou 1 = cache v1 (language seul) → full forcé
source_sha: a1b2c3d          # SHA du dernier run réussi (full ou incrémental)
entities:
  - id: metering-service
    quadrants: [reference, explanation]
    docs:
      reference: reference-metering-service.md
      explanation: explanation-metering-service.md
    sources:                 # fichiers/ancres qui alimentent cette entité
      - server.js:127
      - metering/usage.js:1-88
    deps: [auth-middleware]  # entités dont celle-ci dépend (invalidation 1 niveau)
    confidence: high
```

### Contrat du concept map (sortie du sous-agent chercheur)

Pour **chaque entité en scope**, le sous-agent retourne (en plus de la prose, un
objet structuré parsable) :

```
id · purpose (1 phrase)
key_concepts[]
public_surface[]     chaque item : { text, source "file:line", confidence: high|low }
dependencies[] / dependents[]   → alimentent `deps`
edge_cases[]         { text, source }
design_decisions[]   { text, source, confidence }
suggested_quadrants  (reference / explanation, selon references/diataxis.md)
```

Règle : **tout claim non trivial porte `source` (`file:line`) et `confidence`**.
Les claims `confidence: low` sont ceux que le rédacteur re-lira (étape 6).

### Protocole de délégation (étape 2)

1. Après résolution base + langue, cartographie légère (`glob`/`grep`) pour estimer les entités en scope.
2. **Parallélisme adaptatif** : ≤ ~3 entités → 1 sous-agent ; sinon 1 sous-agent par groupe cohérent d'entités.
3. Lancer `Agent(subagent_type: "general-purpose", model: <tier standard résolu>)` — prompt = instructions d'archéologie + **contrat de sortie ci-dessus** (exiger `file:line` + `confidence`). Jamais `fork`.
4. **Fallback** : tool `Agent` ou modèle cible indisponible → le rédacteur fait l'archéologie inline (= Step 3 v1). Le reste du pipeline est identique.

### Algorithme de régénération (nouveau step, avant l'archéologie)

```
Lire <base>/retro/.retro-doc.yml
SI absent OU schema_version ∈ {absent, 1}      → FULL (puis écrire cache v2)
SINON SI mot-clé "full" dans le prompt         → FULL
SINON (défaut si cache v2, ou mot-clé "incrémental") → INCRÉMENTAL

INCRÉMENTAL :
  changed = git diff --name-only <source_sha>..HEAD
  SI pas de repo git / SHA introuvable         → FULL + signaler
  invalides = { e ∈ entities | un fichier de e.sources ∈ changed }
  invalides += { e | e.deps ∩ invalides ≠ ∅ }   # propagation 1 niveau
  Re-déléguer la recherche UNIQUEMENT sur `invalides` ; réécrire leurs docs ;
  conserver les docs des entités non invalidées.
  Réécrire INDEX (entrées touchées), re-scan secrets (fichiers réécrits).

FIN (full ou incrémental) :
  source_sha = HEAD courant ; réécrire entities[] ; persister .retro-doc.yml v2.
```

### Mapping steps `SKILL.md` → modifications

| Step actuel | Modification |
|---|---|
| (nouveau) Régénération | Insérer **avant** l'archéologie : décide full/incrémental, calcule `invalides` |
| 3 Archaeology | Remplacé par **délégation** (protocole ci-dessus) ; map = contrat de sortie du sous-agent |
| 4 Partition | Inchangé (s'appuie sur `suggested_quadrants`) |
| 5/6 Reference/Explanation | Inchangés, mais partent du map + **re-lecture ciblée** des claims `confidence: low` (étape 6) |
| 7 INDEX | En incrémental, ne régénère que les entrées touchées |
| (nouveau) Persistance cache | Écrire `.retro-doc.yml` v2 (`source_sha`, `entities[]`) |
| 9 Secret scan | Limité aux fichiers générés/réécrits de ce run |
| front matter `allowed-tools` | + `Agent` (+ `Task` si exposé par l'IDE) |

### Reprise (agent frais)

- `doc/roadmap/` présent → **roadmap-tracking s'appliquera au démarrage** : reprendre le plan `#2`, suivre les phases jusqu'à l'implémentation.
- Projet de validation : `auxitum` (clone `/mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/auxitum`), déjà documenté par la v1 → sert au scénario rétrocompat.
- Prompt de reprise : `Reprends le plan 2-retro-doc-token-optimization.md`.

## Décisions techniques

| Décision | Choix retenu | Justification |
|----------|-------------|---------------|
| Agent de recherche | `general-purpose` (modèle configurable) + cartographie légère intégrée (pas d'agent `Explore` séparé) | La carto (`find`/`grep`) n'est pas le poste cher ; un agent séparé ajoute un handoff séquentiel et un risque de complétude pour un gain marginal |
| Parallélisme | Adaptatif au scope (1 agent si petit, 1 par groupe d'entités si gros) | Évite de payer N coûts de démarrage sur un petit projet |
| Emplacement du cache | Committé sous `<base>/retro/` dans `.retro-doc.yml` | L'incrémental bénéficie à toute l'équipe et à la CI |
| Fichiers de méta | Un seul `.retro-doc.yml` étendu (langue + SHA + map) | Un seul fichier à lire/écrire, cohérent avec l'existant v1 |
| Provenance | `file:line` par claim dans le map | Double usage : invalidation fine (#5) ET re-vérification ciblée (#6) ; prolonge l'exactitude déjà validée en v1 |
| Invalidation | Par entité + propagation aux dépendants directs (1 niveau) | Borné et prévisible ; la transitive complète ré-invaliderait tout et annulerait le gain |
| Déclenchement régén | Défaut incrémental si cache présent ; mots-clés `full` / `incrémental` dans le prompt | Pas d'interruption systématique ; pilotable |
| Rétrocompat | Cache v1 (langue seule) → full forcé + écriture du cache enrichi | Transparent, aucune casse sur un projet déjà documenté |
| Garde-fou qualité | Confiance + re-lecture ciblée du rédacteur sur les claims incertains | Équilibre coût/fiabilité ; qualité ≥ v1 |
| Modèle du rédacteur | Recommandé `reasoning` (non bloquant) ; le skill reste agnostique | Le rédacteur arbitre les claims douteux + produit la prose Explanation et la partition Diataxis — jugement où `reasoning` paie ; la lecture lourde étant déportée sur le chercheur `standard`, ça ne pèse que sur l'output (poste mineur) |
| Mécanisme de délégation | `Agent` frais `general-purpose` + override `model`, jamais un `fork` | Un `fork` hérite du contexte du parent (ruine l'économie) et ignore l'override `model` |
| Dégradation | Fallback inline (= comportement v1) si tool `Agent` indisponible | Robustesse multi-IDE ; préserve le prompt-only |

## Tests

> Section renseignée à l'étape 🧪 Tests. Procédure, résultats attendus et
> résultats réellement joués seront écrits ici (source de vérité traçable).
> Validation manuelle sur `auxitum` en 3 scénarios (neuf, rétrocompat,
> incrémental sur diff) par renommage de `doc/retro`.

## Journal de session

### Session 2026-11-03
- ✅ Fait : cadrage complet (8 décisions d'architecture + 4 décisions de frontier), issue #2 créée, plan créé.
- ✅ Fait : section « Rôles et modèles » ajoutée (rédacteur appelant `reasoning` recommandé / chercheur sous-agent `standard`), clarification du garde-fou #6 (re-lecture par le rédacteur lui-même, sans gate), décisions `Modèle du rédacteur` et `Mécanisme de délégation` ajoutées.
- ✅ Fait : section « Contexte d'implémentation (auto-portant) » ajoutée (état de départ, invariants, schéma `.retro-doc.yml` v2, contrat du concept map, protocole de délégation, algo de régénération, mapping steps, reprise) — le plan est désormais implémentable par un agent frais post-`/clear`.
- 📋 Prochain : implémentation (Lot 1 → Lot 4), puis tests 3 scénarios sur auxitum.
