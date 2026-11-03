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
- [ ] Étape 2 — Step 3 refondu : délégation à un sous-agent `general-purpose` configurable, cartographie légère intégrée, parallélisme adaptatif au scope, contrat de sortie = concept map à provenance, fallback inline si `Agent` indisponible (L · reasoning → Opus)
- [ ] Étape 3 — Persistance du cache enrichi en fin de run (map + SHA courant) dans `.retro-doc.yml`, cohérence avec Step 7 (INDEX) et Step 9 (stop) (S · standard → Sonnet)
- [ ] Étape 4 — Régénération incrémentale : lecture cache + SHA, `git diff <sha>..HEAD`, défaut incrémental si cache présent, mots-clés `full`/`incrémental`, rétrocompat = full forcé si cache v1 (M · reasoning → Opus)
- [ ] Étape 5 — Invalidation par entité + propagation aux dépendants directs (1 niveau) depuis le graphe `deps` du map (M · reasoning → Opus)
- [ ] Étape 6 — Garde-fou qualité : marquage `confidence` par claim par le sous-agent, re-lecture ciblée par le rédacteur (reasoning) sur les claims incertains avant écriture (S · reasoning → Opus)
- [ ] 🧪 Tests — Rédiger et exécuter la procédure de validation (3 scénarios : neuf, rétrocompat, incrémental sur diff)
- [ ] ✅ Validation — Vérifier les résultats et clôturer

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
| Dégradation | Fallback inline (= comportement v1) si tool `Agent` indisponible | Robustesse multi-IDE ; préserve le prompt-only |

## Tests

> Section renseignée à l'étape 🧪 Tests. Procédure, résultats attendus et
> résultats réellement joués seront écrits ici (source de vérité traçable).
> Validation manuelle sur `auxitum` en 3 scénarios (neuf, rétrocompat,
> incrémental sur diff) par renommage de `doc/retro`.

## Journal de session

### Session 2026-11-03
- ✅ Fait : cadrage complet (8 décisions d'architecture + 4 décisions de frontier), issue #2 créée, plan créé.
- 📋 Prochain : implémentation (Lot 1 → Lot 4), puis tests 3 scénarios sur auxitum.
