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

- [x] Étape 1 — Schéma `.retro-doc.yml` étendu (`language`, `source_sha`, `entities[]{sources[file:line], deps[], confidence, quadrants}`) + extension `allowed-tools` (`Agent`/`Task`) + doc du format cache dans `references/` (M · reasoning → Opus)
- [x] Étape 2 — Step 3 refondu : délégation à un sous-agent `general-purpose` configurable (`Agent` frais avec override `model` vers le tier `standard`, **jamais un `fork`** qui hériterait du contexte et ignorerait l'override), cartographie légère intégrée, parallélisme adaptatif au scope, contrat de sortie = concept map à provenance, fallback inline si `Agent` indisponible. Documenter les **deux rôles** (rédacteur / chercheur) et émettre une **recommandation non bloquante** : rédacteur en tier `reasoning` (L · reasoning → Opus)
- [x] Étape 3 — Persistance du cache enrichi en fin de run (map + SHA courant) dans `.retro-doc.yml`, cohérence avec Step 7 (INDEX) et Step 9 (stop) (S · standard → Sonnet)
- [x] Étape 4 — Régénération incrémentale : lecture cache + SHA, `git diff <sha>..HEAD`, défaut incrémental si cache présent, mots-clés `full`/`incrémental`, rétrocompat = full forcé si cache v1 (M · reasoning → Opus)
- [x] Étape 5 — Invalidation par entité + propagation aux dépendants directs (1 niveau) depuis le graphe `deps` du map (M · reasoning → Opus)
- [x] Étape 6 — Garde-fou qualité : marquage `confidence` par claim par le sous-agent, re-lecture ciblée par le **rédacteur lui-même** (le modèle appelant, pas une gate ni un sous-agent) via ses propres `Read`/`Grep` sur les `file:line` des claims `confidence: low`, avant écriture. Interne au run, silencieux, sans interaction utilisateur (S · reasoning → Opus)
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

> Validation manuelle sur `auxitum` en 3 scénarios. Les exécutions `/retro-doc`
> se font dans une session Claude Code ouverte sur `auxitum` (rédacteur en tier
> `reasoning` recommandé). Les renommages et vérifications sont des commandes
> opérateur (bloc bash). Source de vérité : cette section.

### Prérequis pour les tests

- Skill `retro-doc` **v2 implémenté et installé** (plan #2 terminé) avant de tester.
- `auxitum` cloné en `/mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/auxitum`, branche `dev`, working tree propre.
- `auxitum/doc/retro/` v1.0.0 **présent et commité** (généré par le run v1, poussé sur `dev`).
- `claude` CLI disponible ; scope de doc identique au run v1 pour comparabilité : **module metering**.
- ✅ **Décision tranchée (étape 4)** : base de diff incrémental = `source_sha..HEAD` (**commité seul**, working-tree non couvert). Le scénario 3 **commite** le marqueur pour que HEAD avance — conforme à l'implémentation. Aucune adaptation nécessaire.

### Procédure de test

**Scénario 1 — projet neuf (pas de cache) :**

```bash
echo "=== Go To auxitum ==="
cd /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/auxitum

echo "=== Backup The v1 retro-doc Output ==="
mv doc/retro doc/retro.bak
```

Puis, dans une session Claude Code sur `auxitum` :

```
# session: 35110636-cbc5-452e-9998-a4c25c1d6bad
/retro-doc documente le module de metering
```

```bash
echo "=== Check Fresh Output And v2 Cache ==="
ls doc/retro/
cat doc/retro/.retro-doc.yml
```

**Scénario 2 — projet déjà documenté par la v1 (rétrocompat) :**

```bash
echo "=== Keep The Fresh v2 Output Aside ==="
mv doc/retro doc/retro.new.metering

echo "=== Restore The v1 Output As Current ==="
mv doc/retro.bak doc/retro
```

Puis, dans la session Claude Code :

```
/retro-doc documente le module de metering
```

```bash
echo "=== Check The v1 Cache Was Upgraded To v2 ==="
cat doc/retro/.retro-doc.yml
```

**Scénario 3 — incrémental réel (sur un cache v2) :**

```bash
echo "=== Pick One Documented Source File (adapt FILE to a path listed under entities[].sources of doc/retro/.retro-doc.yml; server.js was documented by the v1 run) ==="
FILE="server.js"

echo "=== Append A Revertible Marker And Commit It So HEAD Advances ==="
printf '\n// retro-doc incremental test marker\n' >> "$FILE"
git add "$FILE"
git commit -m "test: retro-doc incremental marker"
```

Puis, dans la session Claude Code :

```
/retro-doc documente le module de metering
```

```bash
echo "=== Check Only The Touched Entity Was Regenerated And source_sha Advanced ==="
git -C /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/auxitum status --short doc/retro/
cat doc/retro/.retro-doc.yml

echo "=== Revert The Marker ==="
git reset --hard HEAD~1
```

**Re-test délégation (transverse, après correctif session 2026-11-03) :**
Sur **chaque** run (y compris S1 sur le petit scope metering), vérifier dans l'UI
Claude Code qu'**au moins un sous-agent `Task` est lancé** (modèle Sonnet) pour
l'archéologie — le writer Opus ne doit **jamais** lire le code source lui-même
(hors re-lecture ciblée des claims `confidence: low`). Indices d'un échec : aucun
`Task` visible, le writer enchaîne des `Read`/`Grep` sur le code source, coût
writer élevé. Vérifier aussi que `deps` du cache se re-peuple si le module a des
dépendances réelles.

**Résultats attendus :**
- **S1** : `doc/retro/` recréé avec `reference-*` / `explanation-*` / `INDEX.md` ; `.retro-doc.yml` en `schema_version: 2` avec `source_sha` et `entities[]` peuplé. **Un sous-agent Sonnet est lancé** (délégation effective).
- **S2** : le skill **signale un cache v1** (pas de `schema_version`) → **run complet forcé**, puis `.retro-doc.yml` réécrit en `schema_version: 2` avec `source_sha` + `entities[]`.
- **S3** : le skill annonce un **run incrémental** ne re-recherchant que l'entité dont `FILE` est une source (+ ses dépendants directs) ; seules les docs de cette entité sont modifiées (les autres intactes) ; `source_sha` = nouveau HEAD. Après revert, working tree propre.
- Transverses : aucun commit auto par le skill ; scan secrets propre ; exactitude `file:line` ≥ v1.

### Procédure de test — gros scope multi-entités (S4, S5)

> Objectif : combler les 2 trous laissés par le scope mono-entité `metering` —
> (a) **exercer l'étape 5** (propagation 1-niveau, impossible à une seule entité),
> (b) **mesurer la délégation sur un cas favorable** (volume de lecture élevé, moins
> cache-dominé). Scope choisi : le **sous-système LLM** d'`auxitum`, qui se
> décompose naturellement en plusieurs entités avec dépendances réelles
> (`agenticOrchestrator.js` 52 Ko, `agenticLoop.js`, `streamProxy.js`, `verifier.js`,
> `toolCallParser.js`, `confirmation-guard.js`, `routes/llm-proxy.js`).

#### Prérequis S4/S5

> ⚠️ **Isoler le test.** Le `doc/retro` committé contient la rétro-doc **v1 de
> tout auxitum** (scope large). Lancer S4 (scope = sous-système LLM) par-dessus
> mélangerait anciennes et nouvelles entités et fausserait les assertions (N
> entités, `deps`). On part donc d'un `doc/retro` **vide**, puis on **restaure**
> l'original à la fin. On **conserve** la v1 (backup), on ne la supprime pas.

```bash
echo "=== Go To auxitum ==="
cd /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/auxitum

echo "=== Restore The Committed doc/retro (undo S1-S3 working-tree manips) ==="
git checkout -- doc/retro

echo "=== Remove Any Leftover Test Backups From S1-S3 ==="
rm -rf doc/retro.bak doc/retro.new doc.bak

echo "=== Back Up The v1 doc/retro Aside (preserve, do not delete) ==="
mv doc/retro doc/retro.v1.bak

echo "=== Confirm doc/retro Is Absent So S4 Starts Fresh ==="
ls doc/retro 2>/dev/null || echo "doc/retro absent — ready for a clean full run"
```

> **Après S4/S5** (une fois les résultats relevés), restaurer l'état d'origine :
>
> ```bash
> echo "=== Go To auxitum ==="
> cd /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/auxitum
>
> echo "=== Drop The Test Output ==="
> rm -rf doc/retro
>
> echo "=== Restore The v1 doc/retro ==="
> mv doc/retro.v1.bak doc/retro
>
> echo "=== Confirm Working Tree Is Clean ==="
> git status --short doc/retro
> ```

**Scénario 4 — gros scope full run (délégation + multi-entités) :**

Dans une session Claude Code sur `auxitum` (rédacteur Opus recommandé) :

```
/retro-doc documente le sous-système LLM : orchestration agentique, proxy streaming, parsing des tool calls, vérification, et la route llm-proxy — full
```

```bash
echo "=== Go To auxitum ==="
cd /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/auxitum

echo "=== List Generated Docs (expect several reference-/explanation- files) ==="
ls doc/retro/

echo "=== Inspect The Cache: Expect Multiple Entities With Non-Empty deps ==="
cat doc/retro/.retro-doc.yml
```

**Scénario 5 — incrémental avec propagation 1-niveau (le test clé de l'étape 5) :**

Choisir une source appartenant à une entité **dont d'autres dépendent** (ex.
`streamProxy.js`, utilisé par l'orchestration agentique et la route proxy).

```bash
echo "=== Go To auxitum ==="
cd /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/auxitum

echo "=== Append A Revertible Marker On A Depended-Upon Source ==="
printf '\n// retro-doc propagation test marker\n' >> auxitum-orchestrator/lib/llm/streamProxy.js

echo "=== Commit So HEAD Advances ==="
git add auxitum-orchestrator/lib/llm/streamProxy.js
git commit -m "test: retro-doc propagation marker"
```

Dans la session Claude Code :

```
/retro-doc documente le sous-système LLM : orchestration agentique, proxy streaming, parsing des tool calls, vérification, et la route llm-proxy
```

```bash
echo "=== Go To auxitum ==="
cd /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/auxitum

echo "=== Check Which Docs Were Rewritten (expect the touched entity + its direct dependents only) ==="
git status --short doc/retro/

echo "=== Inspect Cache source_sha Advanced ==="
cat doc/retro/.retro-doc.yml

echo "=== Revert The streamProxy Marker Commit (leaves untracked doc/retro test output in place) ==="
git reset --hard HEAD~1
```

> `doc/retro` est ici une sortie de test **non trackée** → `git reset --hard` ne
> la touche pas. Sa suppression + la restauration de la v1 sont faites par le
> bloc « Après S4/S5 » du prérequis ci-dessus.

**Résultats attendus S4/S5 :**
- **S4** : `doc/retro/` contient **plusieurs** `reference-*` / `explanation-*` (une paire par entité) ; `.retro-doc.yml` liste **≥ 3-4 entités** dont au moins une avec `deps` **non vide** pointant vers les **ids d'autres entités documentées** (preuve que `dependencies` → `deps` se peuple quand il y a des pairs). Observer si **plusieurs sous-agents Sonnet** sont lancés en parallèle (parallélisme adaptatif, > ~3 entités). Noter coût + contexte d'entrée pour comparer au mono-entité.
- **S5** : le skill annonce un run **incrémental** ; **seules** les docs de l'entité dont `streamProxy.js` est une source **ET de ses dépendants directs** sont réécrites (`git status` doit montrer ce sous-ensemble précis, pas toutes les entités) ; les entités sans lien restent intactes ; `source_sha` = nouveau HEAD. Après revert + checkout, working tree propre.
- **Mesure délégation (transverse)** : sur S4 (gros scope), comparer le coût/contexte au mono-entité. Si le gain délégation existe, il doit apparaître ici (lecture fraîche moins cache-dominée). Sinon, l'hypothèse #2 est infirmée même sur gros scope → trancher l'archi.

### Résultats joués et vérifiés

| Date | Test | Attendu | Observé | Verdict |
|------|------|---------|---------|---------|
| 2026-11-03 | S1 neuf (pré-fix) | cache v2 peuplé | cache v2 écrit (sources + deps), **mais chercheur hang** sur greps non filtrés (31088 fichiers) — $5.73 / 12m55 | ⚠️ |
| 2026-11-03 | S1 neuf (post-fix hygiene) | cache v2, pas de hang | pas de hang — $3.41 / 8m58 ; `deps: []` (était peuplé avant fix) | ⚠️ |
| 2026-11-03 | S2 rétrocompat v1→v2 | v1 détecté → full forcé → réécrit v2 | `schema_version: 2` + `source_sha` + `entities[]` réécrits — $3.66 / 7m58 | ✅ |
| 2026-11-03 | S3 incrémental réel | run incrémental ; entité non invalidée si source inchangée | diff `89ac41b..HEAD` → aucune source metering touchée (seul `server.js` marker) → **no-op correct** — $1.27 / 1m23 | ✅ |
| 2026-11-03 | Délégation (transverse, bloquant) | archéologie déléguée à un sous-agent Sonnet, writer Opus = orchestration/écriture seule | writer **Opus a fait l'archéologie INLINE** (« petit périmètre cohérent → lecture directe, pas de sous-agent ») → économie de tokens défaite | ❌ |
| 2026-11-03 | Délégation (re-test post-fix) | sous-agent Sonnet lancé même sur petit scope ; writer ne lit pas le code | `Agent(Research metering) Sonnet 4.6` **lancé** ; writer Opus : orchestration + re-lecture ciblée de 5 claims + écriture (a détecté du dead code) — $3.41 / 6m59 | ✅ |
| 2026-11-03 | Gain de coût réel (full, post-fix) | coût writer full ↓ vs baseline inline | **coût inchangé** ($3.41 vs $3.41/$3.66) malgré délégation effective — hypothèse du gain non démontrée (voir Observation 5) | ❌ |

### Analyse comparative des runs (2026-11-03)

Tous les runs portent sur le **même scope** (module metering d'`auxitum`) et, pour
S1/S2, le **même SHA** (`89ac41b`) — ils sont donc directement comparables.

| Run | Durée | Coût | Contexte in → out (∑ cache) | Sources | deps | Sous-agent ? |
|---|---|---|---|---|---|---|
| S1 hang (pré-fix) | 12m55 | **$5.73** | 5 808 119 → 41 294 (∑ 5 562 870) | 26 | `[db, streamProxy, config, auth]` | oui (a hang) |
| S1 fix hygiene | 8m58 | $3.41 | 3 153 846 → 22 438 (∑ 3 077 012) | 13 | `[]` | ❓ non confirmé |
| S2 rétrocompat | 7m58 | $3.66 | 3 247 252 → 27 758 (∑ 3 171 850) | 12 | `[]` | ❓ non confirmé |
| S3 incrémental | **1m23** | **$1.27** | 621 881 → 5 003 (∑ 507 723) | 13 (inchangé) | `[]` | non (no-op) |
| Re-test S1 (post-fix délég.) | 6m59 | $3.41 | 3 030 599 → 28 644 (∑ 2 858 967) | 14 | `[]` | **oui, Sonnet (confirmé)** |

> 🤔 **Hypothèse** sur le format `X → Y (∑ Z)` : `X` = contexte cumulé en entrée,
> `Y` = tokens générés, `Z` = part lue depuis le cache (sous-ensemble de `X`).
> Sémantique exacte non vérifiée (pas d'accès au champ `usage` brut), mais `Z < X`
> exclut que `Z` soit le total.

**Observation 1 — le hang a coûté ~$2,3 et ~4 min pour rien.** Le `grep -rn` sans
exclusion a renvoyé un volume massif réinjecté dans le contexte : l'entrée bondit
à **5,8 M** vs **3,15 M** après fix (**+84 %**). Le correctif « search hygiene »
est validé par les chiffres : −45 % de contexte, −$2,32, −4 min.

**Observation 2 — le gain incrémental est réel et important (S3).** Sur un module
inchangé, le run incrémental coûte **$1,27 vs ~$3,4–3,66** pour un full
(**−63 %**) et **1m23 vs ~8 min** (**−83 %**). Le contexte d'entrée tombe à
**0,62 M** (−80 %) : le run lit seulement le diff, le cache et l'existence des docs,
sans archéologie. C'est la preuve que les étapes 4-5 remplissent leur objectif.

**Observation 3 — mais le coût d'un full reste anormalement haut (~$3,5 pour 3
fichiers).** Un contexte d'entrée de ~3,2 M sur un module de ~660 lignes est le
**symptôme direct de la régression délégation** : le writer Opus lit tout le code
lui-même au lieu de déporter sur un sous-agent Sonnet. Le baseline $3,4–3,66
**n'est donc pas la cible optimisée** — il est gonflé par la lecture inline. Une
fois la délégation forcée, le contexte (et le coût) du writer doit chuter
nettement. À re-mesurer au re-test.

**Observation 4 — couverture non déterministe à SHA constant, et `deps: []` est
en fait correct ici.** À code identique (`89ac41b`), le nombre de sources varie
(**26 → 13 → 12 → 14**) : variance intrinsèque de l'archéologie LLM, acceptable
tant que les claims restent exacts. En revanche, le `deps: [db, streamProxy,
config, auth]` du run S1-hang était **erroné** : `deps` doit contenir des **ids
d'entités documentées**, pas des noms de modules bruts. Le scope de test ne
documente qu'**une** entité (`metering`) → `deps: []` est le **résultat correct**
(il n'y a aucune autre entité documentée dont dépendre). Le run hang a donc
sur-peuplé `deps` à tort ; le comportement post-fix est le bon.

> **Conséquence test-coverage** : la **propagation 1-niveau de l'étape 5 n'est
> pas exerçable** sur un scope mono-entité — il faut un scope **multi-entités**
> pour la tester. Lacune de couverture à combler (voir Diagnostic), pas un bug.

**Observation 5 — la délégation marche mais le coût ne baisse pas (challenge de
l'hypothèse #2).** Re-test post-fix : un sous-agent Sonnet est bien lancé et le
writer Opus ne lit plus le code — pourtant le coût est **identique** ($3.41,
3,03 M de contexte) aux runs pré-fix. Lecture honnête :

- **94 % du contexte est du cache-read** (∑ 2,86 M / 3,03 M). Les lectures de code
  sont donc majoritairement facturées au tarif cache (~10 % du prix input) : le
  **tier du lecteur (Opus vs Sonnet) pèse peu sur le coût** quand la lecture est
  cache-dominée. Déplacer la lecture sur Sonnet ne change presque rien au total.
- ❓ **À confirmer de ton côté** : la ligne `Total Cost` / `Usage` de Claude Code
  **inclut-elle les tokens du sous-agent**, ou est-elle main-session seule ? Si
  main-seule, alors Opus consomme encore 3 M de contexte malgré la délégation
  (contre-intuitif → à investiguer). Si agrégée, le wash s'explique par le point
  cache ci-dessus. Je ne peux pas trancher sans accès au `usage` brut (hors scope
  du plan, cf. « Hors scope : Mesure runtime du gain token »).

**Synthèse** : les trois mécanismes **fonctionnent** (incrémental ✅ S3,
search-hygiene ✅, délégation ✅ re-test). Le **gain de coût sur les full runs
n'est toutefois pas démontré** — soit parce qu'il est réel mais invisible
(facturation cache-dominée / attribution sous-agent inconnue), soit parce que
l'architecture déplace le travail sans réduire le coût facturé dans ce modèle de
billing. Le vrai gain mesurable et incontestable est **l'incrémental** (S3 : −63 %
sur un module inchangé), pas la délégation.

## Diagnostic en cours

### Bloc — Délégation non forcée (régression critique, remontée aux tests 🧪)

**Symptôme** : sur un scope jugé « petit » (3 fichiers ≈ 660 l.), le writer Opus
lit le code lui-même au lieu de déléguer à un sous-agent Sonnet. L'objectif
primaire de #2 (déporter l'archéologie — poste de tokens dominant — sur le tier
`standard`) est annulé dès que le writer estime la tâche triviale.

**Cause racine (vérifiée dans `SKILL.md` Step 4 + `references/delegation.md`)** :
le wording n'impose nulle part la délégation inconditionnelle. Trois portes de sortie :
1. Step 4 item 1 : « Read the entry points inline » → le modèle étend la lecture inline au module.
2. Step 4 item 2 / delegation.md item 2 : « ≤ ~3 entities → one sub-agent » → interprété comme « petit → pas de sous-agent » au lieu de « petit → un seul sous-agent ».
3. Step 4 item 3 / delegation.md item 4 : fallback inline déclenché sur un **jugement de trivialité**, alors qu'il ne doit l'être que sur une **indisponibilité réelle du tool `Agent`**.

**Correctif appliqué (session 2026-11-03, re-test requis)** : délégation rendue
**obligatoire quelle que soit la taille du scope** ; le parallélisme adaptatif ne
pilote que le *nombre* de chercheurs (≥ 1), jamais le *si* ; fallback inline
restreint à l'indisponibilité avérée du tool ; item 1 clarifié (carto ≠
archéologie) ; nouvelle Hard rule. Fichiers : `SKILL.md` Step 4 (items 1-3 +
Hard rules), `references/delegation.md` (items 2, 4).

**✅ RÉSOLU (re-test 2026-11-03, run `f8058b23`)** : sous-agent Sonnet
(`Agent(Research metering module) Sonnet 4.6`) bien lancé sur le petit scope ;
writer Opus n'a pas lu le code (orchestration + re-lecture ciblée + écriture). La
régression bloquante est levée. Verdict délégation : ✅.

**Pistes restées ouvertes (ne bloquent pas la délégation, mais à trancher avant clôture)** :
- **Gain de coût non démontré** : le coût full est resté identique malgré la délégation (voir `### Analyse comparative § Observation 5`). Cause probable : facturation cache-dominée (94 % cache-read) rendant le tier du lecteur quasi neutre. ❓ À confirmer : la ligne `Total Cost` de Claude Code inclut-elle le sous-agent ?
- **`deps: []` est correct** sur scope mono-entité (deps = ids d'entités documentées ; une seule entité ⇒ vide). Le `deps` peuplé du run hang était erroné (noms de modules). → voir Observation 4.
- **Étape 5 (propagation 1-niveau) non testée** : le scope mono-entité `metering` ne peut pas l'exercer. Lacune de couverture — nécessite un **scope multi-entités**.

## Journal de session

### Session 2026-11-03
- ✅ Fait : cadrage complet (8 décisions d'architecture + 4 décisions de frontier), issue #2 créée, plan créé.
- ✅ Fait : section « Rôles et modèles » ajoutée (rédacteur appelant `reasoning` recommandé / chercheur sous-agent `standard`), clarification du garde-fou #6 (re-lecture par le rédacteur lui-même, sans gate), décisions `Modèle du rédacteur` et `Mécanisme de délégation` ajoutées.
- ✅ Fait : section « Contexte d'implémentation (auto-portant) » ajoutée (état de départ, invariants, schéma `.retro-doc.yml` v2, contrat du concept map, protocole de délégation, algo de régénération, mapping steps, reprise) — le plan est désormais implémentable par un agent frais post-`/clear`.
- ✅ Fait : procédure de tests écrite (section `## Tests`) — 3 scénarios sur auxitum (neuf / rétrocompat / incrémental réel avec commit du marqueur + revert). Décision « base de diff incrémental » signalée à confirmer à l'étape 4.
- 📋 Prochain : implémentation (Lot 1 → Lot 4), puis exécution des tests 3 scénarios sur auxitum.

### Session 2026-11-03 (implémentation étapes 1-2)
- ✅ Étape 1 : `allowed-tools` étendu (`Agent`/`Task`) dans `SKILL.md` ; schéma cache v2 documenté dans `references/cache-schema.md` (`schema_version`, `source_sha`, `entities[]{quadrants, docs, sources[file:line], deps, confidence}`, rétrocompat v1→full) ; pointeur ajouté au Step 1.
- ✅ Étape 2 : Step 3 de `SKILL.md` refondu (archéologie → délégation) — carto légère inline, parallélisme adaptatif, `Agent general-purpose` + `model: sonnet` (jamais `fork`), fallback inline, reco non bloquante rédacteur `reasoning` ; contrat concept map + 2 rôles + mapping tier→modèle dans `references/delegation.md`.
- ✅ Étape 3 : nouveau step `SKILL.md` **Step 8 — Cache persistence** inséré entre INDEX et Quality review ; steps 8→9 et 9→10 renommés ; note incremental ajoutée à Step 7 ; note scan scope ajoutée à Step 10.
- 📋 Prochain : étapes 4→6 (régén incrémentale, invalidation, garde-fou qualité), puis 🧪 Tests + ✅ Validation sur auxitum.

### Session 2026-11-03 (implémentation étapes 4-6, one-shot)
- ✅ Étape 4 : nouveau **Step 3 — Regeneration mode (full vs incremental)** inséré **avant** l'archéologie dans `SKILL.md` ; décide full/incrémental (cache absent/v1 → full forcé ; mot-clé `full` → full ; cache v2 par défaut ou `incrémental` → incrémental) ; diff `git diff --name-only <source_sha>..HEAD` ; fallback full si pas de repo / SHA inconnu.
- ✅ Étape 5 : algo d'invalidation dans Step 3 — `invalidated = {e | source de e ∈ changed}` puis propagation **1 niveau** via `e.deps ∩ invalidated` ; le reste du run (archéologie + écriture docs + INDEX + scan) est borné aux entités invalidées ; entités non touchées conservées telles quelles.
- ✅ Étape 6 : garde-fou qualité = item 5 du Step 4 (archéologie) — re-lecture ciblée par le **rédacteur lui-même** des claims `confidence: low` via `Read`/`Grep` sur leurs `file:line` (lignes citées seulement), interne/silencieux, aucune interaction ; ajout d'une checkbox dédiée à la gate Accuracy (Step 10).
- 🔧 Renumérotation : insertion du Step 3 a décalé l'archéologie → Step 4, puis 4→5…10→11 ; toutes les cross-refs internes et le résumé final (`Mode: full | incremental`) mis à jour ; version metadata bumpée `1.0.0` → `2.0.0`.
- 🔒 Décision étape 4 tranchée : base de diff = `source_sha..HEAD` (commité seul) ; limite documentée dans le Step 3 (editions working-tree non détectées → commiter ou `full`). Conforme au scénario de test 3 qui commite le marqueur.
- 🐛 Correctif (remonté par le 1er test réel) : le chercheur Sonnet a hang sur des `grep -rn …/auxitum-orchestrator/` sans exclusion → 31088 fichiers scannés (×65) sur mount `/mnt/c`, 2 tasks `stopped`, `dependents[]` incomplet. Le `find` de carto excluait déjà les vendored dirs mais l'exclusion n'était **pas transmise au chercheur**. Ajout d'une consigne « search hygiene » (`--exclude-dir={node_modules,dist,build,.next,vendor,.git}` ou `rg`) dans le prompt de délégation : `SKILL.md` Step 4 item 2, fallback item 3, et `references/delegation.md` protocole item 3.
- 🧪 Tests joués (voir tableau `## Tests`) : S2 ✅, S3 ✅ (incrémental no-op correct, $1.27), S1 ⚠️ (fonctionnel mais `deps: []`), **Délégation ❌ bloquant**.
- 🐛 Régression critique (remontée par l'utilisateur au test délégation) : writer Opus fait l'archéologie **inline** sur scope « petit » au lieu de déléguer → objectif #2 défait. Cause : wording non impératif (3 portes de sortie). Correctif : délégation rendue **obligatoire quelle que soit la taille** (`SKILL.md` Step 4 items 1-3 + nouvelle Hard rule ; `references/delegation.md` items 2, 4). Détail et pistes ouvertes dans `## Diagnostic en cours`.
- 📋 Prochain : **re-test délégation** (vérifier qu'un sous-agent Sonnet est bien lancé sur petit scope + que `deps` se re-peuple), puis reprise de 🧪 Tests → ✅ Validation.
- ✅ Re-test délégation OK (run `f8058b23`) : sous-agent Sonnet lancé sur petit scope, writer Opus ne lit plus le code, garde-fou étape 6 a détecté du dead code. Régression bloquante **levée**.
- ⚠️ Finding stratégique (Observation 5) : coût full **inchangé** ($3.41) malgré délégation effective → facturation cache-dominée (94 %) neutralise le tier du lecteur. Le gain mesurable réel = **l'incrémental** (S3, −63 %), pas la délégation. Le scope de test (3 fichiers) est trop petit pour que la délégation paie. 2 points ouverts avant clôture : mesure délégation sur gros scope + test étape 5 (propagation, nécessite multi-entités).
- 📋 Prochain : décision utilisateur — test multi-entités/gros scope, ou clôture avec caveats.
