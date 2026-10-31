---
plan:
  id: '1'
  name: 1-retro-doc-skill.md
  link: https://github.com/ehouriez/retro-doc/blob/main/doc/roadmap/done/1-retro-doc-skill.md
status: done
date: 2026-10-30
enriched: 2026-10-31
description: >
  Build the retro-doc plugin skill: a reusable, multi-project Diataxis-based
  retro-documentation generator (Reference + Explanation + INDEX), prompt-only,
  no gstack/bun dependency, no auto commit/push.
priority: high
complexity: L
intent: null
scope:
  modules:
    - .claude-plugin/
    - skills/retro-doc/
  patterns:
    - "**/*.md"
    - "**/*.json"
issue:
  id: 1
  url: https://github.com/ehouriez/retro-doc/issues/1
---

# [📚] Plan #1 - retro-doc skill

## Objectif

Construire le skill plugin `retro-doc` (100 % prompt) : un générateur de
rétro-documentation Diataxis ciblée (Reference + Explanation + INDEX) à partir
du code d'un projet, réutilisable multi-projets, sans dépendance gstack/bun ni
commit/push automatique. Il reprend la méthode de `garrytan/document-generate`
(archéologie du code puis écriture structurée) sans sa plomberie.

## Périmètre

### Inclus
- Structure plugin marketplace (manifest marketplace + plugin), calquée sur `roadmap-tracking`.
- `SKILL.md` : moteur Diataxis ciblé (archéologie du code → partition → écriture Reference/Explanation/INDEX).
- Détection de langue de sortie + config projet persistée + confirmation au 1er run + override.
- Logique de sortie : détection base `doc/` vs `docs/`, dossier dédié `retro/`, INDEX généré + lien README.
- Scan secrets avant livraison : grep simple décrit dans le prompt (remplace `gstack-redact`).
- Garde no-commit / no-push : le skill produit les fichiers et s'arrête.
- Références Markdown : templates de doc (Reference / Explanation / INDEX) + extension optionnelle Tutorial/How-to.
- README du repo : install marketplace, usage, exemples.

### Hors scope
- Plomberie gstack/bun, télémétrie, question-tuning, redaction binaire.
- Tutorial / How-to par défaut (restent une option à la demande).
- Publication / commit / push (gérés manuellement par l'utilisateur).

## Étapes

- [x] Étape 1 — Poser la structure du plugin marketplace (manifest marketplace + plugin + arborescence) (S · standard → Sonnet)
- [x] Étape 2 — SKILL.md : préambule, déclencheurs, workflow Diataxis ciblé (archéologie → partition → Reference/Explanation/INDEX) (L · reasoning → Opus)
- [x] Étape 3 — Détection de langue de sortie + config projet persistée + confirmation 1er run + override (M · reasoning → Opus)
- [x] Étape 4 — Logique de sortie : détection base doc/docs, dossier retro/, INDEX + lien README, scan secrets grep, garde no-commit/push (M · standard → Sonnet)
- [x] Étape 5 — Références Markdown : templates Reference / Explanation / INDEX + extension optionnelle Tutorial/How-to (M · standard → Sonnet)
- [x] Étape 6 — README du repo : install marketplace, usage, exemples (S · standard → Sonnet)
- [x] 🧪 Tests — Rédiger et exécuter la procédure de validation (install réelle du plugin + run sur auxitum)
- [x] ✅ Validation — Vérifier les résultats et clôturer

## Décisions techniques

| Décision | Choix retenu | Justification |
|----------|-------------|---------------|
| Packaging | Plugin marketplace (repo = marketplace + plugin unique) | Distribuable, versionné, rejouable multi-projets ; calqué sur roadmap-tracking |
| Implémentation | 100 % prompt (SKILL.md + références Markdown) | Zéro dépendance runtime (KISS) ; évite le lock-in gstack/bun |
| Langue de sortie | Auto-détection docs/README + confirmation 1er run + persistance + override | Vrai multi-projets sans friction répétée |
| Ampleur par défaut | Reference + Explanation + INDEX | Ce qu'une rétro-doc d'archi exige ; Tutorial/How-to ont peu de sens en rétro pur |
| Répertoire de sortie | `<doc\|docs>/retro/` dédié + INDEX + lien README | Frontière nette doc générée/écrite à la main ; régénération sans écrasement ; discoverabilité préservée |
| Git | Aucun commit/push auto ; scan secrets = grep | Règle utilisateur ; remplace le binaire gstack-redact (bun absent) |

## Tests

> Section renseignée à l'étape 🧪 Tests. Procédure, résultats attendus et
> résultats réellement joués seront écrits ici (source de vérité traçable).

### Prérequis pour les tests

- Le repo `ehouriez/retro-doc` doit être **poussé sur GitHub** (branche `main`) : le marketplace tire depuis le remote. Push géré par l'utilisateur.
- Projet cible de validation : `auxitum` (clone local `/mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/auxitum`).
- `claude` CLI disponible dans le terminal ; droit d'installer un plugin (modifie la config Claude Code de l'utilisateur).

```bash
echo "=== Check Claude CLI Available ==="
command -v claude

echo "=== Confirm retro-doc Repo Is Pushed (branch main) ==="
git -C /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/skills/retro-doc log --oneline -1
```

### Procédure de test

```bash
echo "=== Add retro-doc Marketplace ==="
claude plugin marketplace add ehouriez/retro-doc

echo "=== Install retro-doc Plugin ==="
claude plugin install retro-doc@retro-doc

echo "=== List Installed Plugins (expect retro-doc) ==="
claude plugin list
```

Puis, dans une session Claude Code ouverte sur le projet auxitum :

```
/retro-doc documente le module de metering
```

Vérification du résultat après exécution du skill sur auxitum :

```bash
echo "=== Check Generated Output Dir Exists ==="
ls -la /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/auxitum/doc/retro/

echo "=== Check INDEX And Persisted Language ==="
cat /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/auxitum/doc/retro/INDEX.md
cat /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/auxitum/doc/retro/.retro-doc.yml

echo "=== Check README Got A Single Link To The INDEX ==="
grep -n "retro/INDEX.md" /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/auxitum/README*

echo "=== Confirm Skill Did Not Commit (working tree shows new untracked docs only) ==="
git -C /mnt/c/INETUM/AGV/agv-adm/GIT_GITHUB/auxitum status --short
```

**Résultats attendus :**
- `claude plugin list` liste `retro-doc@retro-doc` installé.
- `/retro-doc` confirme le scope, résout la langue (FR détecté sur auxitum) et l'écrit dans `doc/retro/.retro-doc.yml`.
- `doc/retro/` contient au moins un `reference-*.md`, un `explanation-*.md` et `INDEX.md`, en français.
- Le README d'auxitum contient exactement une ligne de lien vers `doc/retro/INDEX.md`.
- `git status` d'auxitum montre les fichiers générés en **untracked** (aucun commit automatique créé par le skill).
- Le scan secrets ne signale aucun secret réel (ou les redige et le signale).

### Résultats joués et vérifiés

| Date | Test | Attendu | Observé | Verdict |
|------|------|---------|---------|---------|
| 2026-10-31 | Install plugin | `retro-doc@retro-doc` installé | Install réussie, skill disponible (`retro-doc:retro-doc`) | ✅ |
| 2026-10-31 | Run sur auxitum | `doc/retro/` avec reference-*, explanation-*, INDEX en FR | 9 reference-*, 4 explanation-*, INDEX.md, langue `fr` persistée | ✅ |
| 2026-10-31 | Lien README | 1 ligne vers `doc/retro/INDEX.md` | README.md:421 lien présent | ✅ |
| 2026-10-31 | Pas de commit auto | fichiers untracked dans auxitum | `git status` → `?? doc/retro/` (untracked) | ✅ |
| 2026-10-31 | Liens cassés | aucun | aucun target `.md` manquant | ✅ |
| 2026-10-31 | Scan secrets | aucun secret réel | 0 match | ✅ |
| 2026-10-31 | Exactitude (anti-hallucination) | `file:line` cités réels | `requiresAuth` bien à `server.js:127`, auth middleware ~146 | ✅ |

## Journal de session

### Session 2026-10-30
- ✅ Fait : cadrage complet, création du repo public `ehouriez/retro-doc`, issue #1, plan créé.
- ✅ Fait : implémentation one-shot des étapes 1-6 (manifests, SKILL.md, references, README, LICENSE).

### Session 2026-10-31
- ✅ Fait : tests finaux OK (install réelle + run sur auxitum), validation vérifiée sur `auxitum/doc/retro/` (exactitude, liens, secrets, absence de commit auto).
- ✅ Clôture : plan `done`, déplacé dans `done/`, roadmap.md mis à jour, issue #1 fermée.
