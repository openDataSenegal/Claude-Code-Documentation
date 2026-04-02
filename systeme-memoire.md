# Système de Mémoire de Claude Code — Documentation complète

Comment Claude Code se souvient de tout : le système multi-couches qui persiste les connaissances entre les conversations.

---

## Table des matières

1. [Vue d'ensemble — Les 6 couches de mémoire](#1-vue-densemble)
2. [CLAUDE.md — Instructions projet](#2-claudemd)
3. [Auto-memory — Mémoire persistante](#3-auto-memory)
4. [Recall — Rappel intelligent par Sonnet](#4-recall)
5. [Extraction — Agent d'extraction automatique](#5-extraction)
6. [Dream — Consolidation nocturne](#6-dream)
7. [Session Memory — Notes de session](#7-session-memory)
8. [Team Memory — Mémoire d'équipe](#8-team-memory)
9. [Compaction — Survie au résumé de contexte](#9-compaction)
10. [Skill /remember — Curation manuelle](#10-skill-remember)
11. [Format des fichiers mémoire](#11-format-des-fichiers-mémoire)
12. [Injection dans le system prompt](#12-injection-dans-le-system-prompt)
13. [Limites et contraintes](#13-limites-et-contraintes)
14. [Sécurité](#14-sécurité)
15. [Feature gates](#15-feature-gates)
16. [Diagramme du cycle de vie complet](#16-diagramme-du-cycle-de-vie)
17. [Fichiers clés](#17-fichiers-clés)

---

## 1. Vue d'ensemble

Claude Code utilise un **système de mémoire à 6 couches** pour persister les connaissances entre et pendant les conversations :

```
┌──────────────────────────────────────────────────────────────────┐
│  COUCHE 1 — CLAUDE.md (instructions projet, dans le repo)        │
│  COUCHE 2 — Auto-memory (mémoire persistante, ~/.claude/)        │
│  COUCHE 3 — Team memory (mémoire partagée, optionnelle)          │
│  COUCHE 4 — Session memory (notes de session, post-compaction)   │
│  COUCHE 5 — Recall (rappel intelligent par Sonnet, par requête)  │
│  COUCHE 6 — Dream (consolidation nocturne automatique)           │
└──────────────────────────────────────────────────────────────────┘
```

### Principe fondamental

> **Les instructions vivent sur le disque, pas dans le contexte.**
> Elles sont injectées fraîchement à chaque session et après chaque compaction. Le contexte conversationnel est éphémère ; la mémoire est permanente.

### Structure sur disque

```
~/.claude/
├── CLAUDE.md                              # Instructions utilisateur globales
├── projects/
│   └── <sanitized-git-root>/
│       └── memory/                        # Auto-memory (racine)
│           ├── MEMORY.md                  # Index (≤200 lignes, 25KB)
│           ├── user_role.md               # Fichier mémoire (type: user)
│           ├── feedback_testing.md        # Fichier mémoire (type: feedback)
│           ├── project_deadline.md        # Fichier mémoire (type: project)
│           ├── reference_linear.md        # Fichier mémoire (type: reference)
│           ├── .consolidate-lock          # Verrou Dream (mtime = lastConsolidatedAt)
│           ├── logs/                      # Logs quotidiens (mode KAIROS)
│           │   └── YYYY/MM/YYYY-MM-DD.md
│           └── team/                      # Team memory (optionnel)
│               ├── MEMORY.md
│               └── *.md
├── session-memory/
│   ├── <session-id>.md                    # Notes de session
│   └── config/
│       ├── template.md                    # Template custom
│       └── prompt.md                      # Prompt custom
└── rules/                                 # Règles globales managées

<project-root>/
├── CLAUDE.md                              # Instructions projet (versionné)
├── CLAUDE.local.md                        # Instructions locales (gitignored)
└── .claude/
    ├── CLAUDE.md                          # Alt instructions projet
    └── rules/
        └── *.md                           # Règles projet (conditionnelles ou non)
```

---

## 2. CLAUDE.md — Instructions projet

### Hiérarchie de chargement (4 niveaux)

Les fichiers sont chargés dans cet ordre — les derniers ont la **priorité la plus haute** :

| Niveau | Emplacement | Description |
|--------|-------------|-------------|
| 1. **Managed** | `/etc/claude-code/CLAUDE.md` | Instructions organisation (policy) |
| 2. **User** | `~/.claude/CLAUDE.md` | Instructions utilisateur globales |
| 3. **Project** | `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md` | Instructions projet (versionnées) |
| 4. **Local** | `CLAUDE.local.md` | Instructions locales (gitignored, priorité max) |

### Découverte des fichiers

**Fichier** : `utils/claudemd.ts` → `getMemoryFiles()`

```
1. Remonte l'arbre de répertoires depuis le CWD jusqu'à la racine
2. Pour chaque répertoire : cherche CLAUDE.md, .claude/CLAUDE.md, .claude/rules/*.md
3. Les fichiers plus proches du CWD ont priorité (chargés en dernier)
4. Gère les worktrees git (évite les doublons avec le repo principal)
5. Supporte --add-dir pour des répertoires supplémentaires
```

### Directives @include

Les fichiers CLAUDE.md supportent les inclusions récursives :

```markdown
@./relative/path.md
@~/home/path.md
@/absolute/path.md
@path\ with\ spaces.md
@path.md#heading-name         ← ancre (strippée silencieusement)
```

| Propriété | Valeur |
|-----------|--------|
| Profondeur max | 5 niveaux |
| Protection circulaire | Tracking des chemins résolus |
| Fichiers absents | Ignorés silencieusement |
| Inclusions externes | Nécessitent `hasClaudeMdExternalIncludesApproved` |

### Règles conditionnelles

Les fichiers dans `.claude/rules/` peuvent avoir un frontmatter `paths` :

```markdown
---
paths:
  - "src/api/**/*.ts"
  - "tests/api/**"
---

Règles spécifiques aux fichiers API...
```

- **Sans `paths`** : chargées à chaque session (eagerly)
- **Avec `paths`** : chargées quand Claude accède à un fichier matching (lazily)

### Nettoyage du contenu

- Commentaires HTML (`<!-- -->`) : **strippés** (auteur notes)
- Code blocks : **préservés** (commentaires inline gardés)
- Extensions binaires : **rejetées** (pas d'inclusion de .png, .pdf, etc.)

---

## 3. Auto-memory — Mémoire persistante

### Emplacement

```
~/.claude/projects/<sanitized-git-root>/memory/
```

**Fichier** : `memdir/paths.ts` → `getAutoMemPath()`

Activée par défaut. Désactivation via :
- Env var : `CLAUDE_CODE_AUTO_MEMORY=false`
- Settings : `autoMemory: false`

### Taxonomie — 4 types de mémoire

| Type | Quand sauvegarder | Comment utiliser |
|------|-------------------|-----------------|
| **`user`** | Rôle, préférences, responsabilités, connaissances de l'utilisateur | Adapter le comportement et les explications |
| **`feedback`** | Corrections ET validations de l'approche ("pas comme ça", "parfait, continue") | Guider le comportement sans répéter les mêmes erreurs |
| **`project`** | Objectifs, deadlines, contexte (convertir dates relatives → absolues) | Comprendre la motivation derrière les demandes |
| **`reference`** | Pointeurs vers des systèmes externes (Linear, Grafana, Slack) | Savoir où chercher l'information |

**Fichier** : `memdir/memoryTypes.ts`

### Ce qui ne doit PAS être sauvegardé

- Patterns de code, conventions, architecture, chemins → dérivables du code
- Historique git, qui a changé quoi → `git log` / `git blame`
- Solutions de debugging → le fix est dans le code
- Contenu documenté dans CLAUDE.md
- Détails éphémères de la tâche en cours

### MEMORY.md — L'index

Le fichier `MEMORY.md` est un **index pur**, pas une mémoire. Chaque entrée est un pointeur d'une ligne :

```markdown
- [Rôle utilisateur](user_role.md) — Data scientist focalisé sur l'observabilité
- [Feedback tests](feedback_testing.md) — Tests d'intégration obligatoires, pas de mocks BDD
- [Deadline mobile](project_deadline.md) — Gel des merges le 2026-03-05
```

| Contrainte | Valeur |
|------------|--------|
| Lignes max | 200 |
| Taille max | 25 KB |
| Format par entrée | `- [Titre](fichier.md) — description courte` (~150 chars) |
| Frontmatter | Aucun |
| Troncature | Avertissement ajouté si dépassement |

---

## 4. Recall — Rappel intelligent par Sonnet

### Mécanisme

À chaque requête utilisateur, un **sélecteur Sonnet** choisit les mémoires les plus pertinentes.

**Fichier** : `memdir/findRelevantMemories.ts`

```
1. Scanner les fichiers mémoire (frontmatter : nom, description, type)
2. Envoyer le manifeste + la requête utilisateur à Sonnet
3. Sonnet retourne les ≤5 fichiers les plus pertinents (JSON)
4. Filtrer ceux déjà affichés dans la conversation
5. Lire les fichiers sélectionnés (max 200 lignes chacun)
6. Injecter comme <system-reminder> attachments
```

### Prefetch non-bloquant

```typescript
startRelevantMemoryPrefetch()
// Lancé pendant que le modèle stream + les outils s'exécutent
// Résultat disponible quand la requête suivante arrive
```

### Avertissement de fraîcheur

Pour les mémoires de plus d'un jour :

```xml
<system-reminder>
This memory is 47 days old. Memories are point-in-time observations,
not live state — claims about code behavior or file:line citations
may be outdated. Verify against current code before asserting as fact.
</system-reminder>
```

**Fichier** : `memdir/memoryAge.ts`

| Âge | Affichage |
|-----|-----------|
| 0 jours | "today" |
| 1 jour | "yesterday" |
| N jours | "N days ago" |

### Budget de session

| Contrainte | Valeur |
|------------|--------|
| Mémoires par requête | ≤ 5 |
| Budget cumulatif | 60 KB par session (`MAX_SESSION_BYTES`) |
| Déduplication | Mémoires déjà dans le contexte exclues |

---

## 5. Extraction — Agent d'extraction automatique

### Déclenchement

L'extraction s'exécute comme **stop hook** à la fin de chaque boucle de requête (fire-and-forget).

**Fichier** : `services/extractMemories/extractMemories.ts`

### Conditions de déclenchement

```
1. ✓ Auto-memory activée (feature gate tengu_passport_quail)
2. ✓ Pas en mode remote (CCR)
3. ✓ Agent principal uniquement (pas un sous-agent)
4. ✓ Exclusion mutuelle : si l'agent principal a écrit des mémoires → skip
5. ✓ Throttle : seulement tous les N tours (configurable via tengu_bramble_lintel)
6. ✓ Pas d'extraction déjà en cours
```

### Agent d'extraction (forké)

L'agent d'extraction est un **fork parfait** de la conversation principale, partageant le prompt cache.

| Propriété | Valeur |
|-----------|--------|
| Turns max | 5 |
| Bash | Read-only (ls, find, grep, cat, stat, wc, head, tail) |
| Edit/Write | Uniquement dans le répertoire auto-memory |
| Read/Grep/Glob | Sans restriction |

### Stratégie d'efficacité

```
Tour 1 : Tous les Reads en parallèle (manifeste pré-injecté)
Tour 2 : Tous les Writes en parallèle
```

### Manifeste pré-injecté

Pour éviter que l'agent scanne le répertoire, le manifeste est injecté avant :

```
[user] user_role.md (2026-03-15): Data scientist focused on observability
[feedback] feedback_testing.md (2026-03-20): Integration tests, no mocks
[project] project_deadline.md (2026-03-25): Mobile freeze 2026-03-05
```

**Fichier** : `memdir/memoryScan.ts` → `formatMemoryManifest()`

- Scanne récursivement (excluant MEMORY.md)
- Max 200 fichiers, triés par mtime (plus récent d'abord)
- Lit les 30 premières lignes de chaque fichier (frontmatter)

### Coalescence

Si une nouvelle requête arrive pendant l'extraction :
1. Le contexte est stashé (`pendingContext`)
2. Après la fin du run courant → trailing run avec le contexte stashé
3. Le trailing run saute le throttle gate
4. Seul le dernier contexte stashé est utilisé

---

## 6. Dream — Consolidation nocturne

Inspiré de la consolidation mnésique humaine pendant le sommeil, le système "rêve" en arrière-plan pour organiser les mémoires.

### Déclenchement (5 portes, du moins cher au plus cher)

```
1. Enable gate   : isAutoDreamEnabled() (feature tengu_onyx_plover)
2. Time gate     : heures depuis lastConsolidatedAt ≥ minHours (défaut: 24)
3. Scan throttle : intervalle scan sessions ≥ 10 minutes
4. Session gate  : sessions touchées depuis consolidation ≥ minSessions (défaut: 5)
5. Lock gate     : pas d'autre processus en consolidation
```

**Fichier** : `services/autoDream/autoDream.ts`

### Mécanisme de verrou

**Fichier** : `services/autoDream/consolidationLock.ts`

```
Fichier : ~/.claude/projects/<path>/memory/.consolidate-lock
├── mtime = timestamp de la dernière consolidation (lastConsolidatedAt)
└── Contenu = PID du processus détenteur
```

| Opération | Description |
|-----------|-------------|
| `tryAcquireConsolidationLock()` | Écrit le PID, vérifie, retourne le mtime pré-acquisition |
| `rollbackConsolidationLock()` | Rewind mtime en cas d'échec (permet le retry) |
| `readLastConsolidatedAt()` | Stat du fichier verrou |
| `listSessionsTouchedSince()` | Filtre les sessions avec mtime > sinceMs |
| Dead PID reclaim | Après 1 heure |

### Les 4 phases de consolidation

**Fichier** : `services/autoDream/consolidationPrompt.ts`

```
Phase 1 — ORIENT
├── ls du répertoire mémoire
├── Lire MEMORY.md (l'index)
├── Survoler les fichiers topic existants
└── Examiner les logs quotidiens (si mode assistant)

Phase 2 — GATHER SIGNAL
├── Logs quotidiens (logs/YYYY/MM/YYYY-MM-DD.md)
├── Mémoires dérivées (contradites par le code actuel)
└── Recherche ciblée dans les transcripts (grep étroit)

Phase 3 — CONSOLIDATE
├── Fusionner les nouveaux signaux dans les fichiers existants
├── Convertir les dates relatives → absolues ("hier" → "2026-03-30")
├── Supprimer les faits contredits
└── PAS d'exploration/vérification (uniquement depuis les transcripts)

Phase 4 — PRUNE & INDEX
├── Mettre à jour MEMORY.md (≤200 lignes, 25KB)
├── Supprimer les pointeurs obsolètes/remplacés
├── Résoudre les contradictions entre fichiers
└── Entrées d'index ≤150 chars, une ligne par fichier
```

### Contraintes de l'agent Dream

| Outil | Accès |
|-------|-------|
| Bash | Read-only (ls, find, grep, cat, stat, wc, head, tail) |
| Read/Grep/Glob | Sans restriction |
| Edit/Write | Uniquement dans le répertoire auto-memory |
| querySource | `'auto_dream'` |
| skipTranscript | `true` |

---

## 7. Session Memory — Notes de session

Système optionnel de notes structurées extraites pendant la conversation, utilisées pour la continuité post-compaction.

**Fichier** : `services/SessionMemory/sessionMemory.ts`

### Déclenchement

| Seuil | Valeur |
|-------|--------|
| Initialisation | 10 000 tokens dans le contexte |
| Mise à jour | 5 000 tokens de croissance + 3 appels d'outils depuis la dernière extraction |

### Template (9 sections)

```markdown
# Current State
# Task Specification
# Files and Functions
# Workflow
# Errors and Issues
# Codebase Documentation
# Learnings
# Key Results
# Worklog
```

| Contrainte | Valeur |
|------------|--------|
| Tokens par section | 2 000 max |
| Tokens total | 12 000 max |
| Template custom | `~/.claude/session-memory/config/template.md` |
| Prompt custom | `~/.claude/session-memory/config/prompt.md` |

### Distinction avec auto-memory

| | Auto-memory | Session memory |
|---|---|---|
| **Portée** | Cross-conversations | Intra-conversation |
| **Survie** | Permanente | Liée à la session |
| **Usage** | Rappel intelligent | Restauration post-compaction |
| **Format** | Frontmatter + markdown libre | Template structuré 9 sections |
| **Extraction** | Par turn (stop hook) | Par seuil de tokens |

---

## 8. Team Memory — Mémoire d'équipe

Mémoire partagée entre les membres d'une organisation.

**Fichier** : `memdir/teamMemPaths.ts`, `memdir/teamMemPrompts.ts`

### Emplacement

```
~/.claude/projects/<path>/memory/team/
├── MEMORY.md
└── *.md
```

### Activation

- Feature gate : `tengu_herring_clock`
- Nécessite auto-memory activée (sous-répertoire)

### Fonctionnement

- Synchronisation au démarrage de session (`services/teamMemorySync/`)
- Résolution de conflits : local gagne en cas de merge
- Tags `<scope>` dans les prompts pour distinguer mémoire privée vs team
- Avertissement sur les données sensibles dans les mémoires team

### Prompt combiné

Quand auto-memory ET team memory sont activées :
- Les deux répertoires sont chargés
- Les types de mémoire incluent des tags `<scope>`
- Deux MEMORY.md distincts (privé et team)
- Les mémoires team exclues de l'extraction quand la sync est stale

---

## 9. Compaction — Survie au résumé de contexte

Quand le contexte est compacté (résumé pour libérer des tokens), les mémoires survivent grâce à un mécanisme de réinjection.

### Processus

```
1. PRÉ-COMPACTION
   ├── executePreCompactHooks()
   └── Sauvegarder l'état des mémoires

2. COMPACTION
   ├── Résumer les messages anciens
   └── Vider les caches mémoire :
       ├── resetGetMemoryFilesCache('compact')
       ├── getUserContext.cache.clear()
       └── clearSystemPromptSections()

3. POST-COMPACTION
   ├── Recharger les fichiers CLAUDE.md depuis le disque (frais)
   ├── Recharger MEMORY.md dans le system prompt
   ├── processSessionStartHooks()
   ├── Réattacher les mémoires pertinentes (via budget tokens)
   └── Restaurer skills et plans

4. HOOKS
   └── InstructionsLoaded hook avec load_reason: 'compact'
```

**Fichier** : `services/compact/postCompactCleanup.ts`

### Budget de restauration

| Paramètre | Valeur |
|-----------|--------|
| Fichiers max restaurés | 5 |
| Budget tokens total | 50 000 |
| Tokens max par fichier | 5 000 |
| Tokens max par skill | 5 000 |
| Budget skills total | 25 000 |

### Pourquoi ça marche

Les instructions et mémoires **vivent sur le disque**, pas dans la conversation. Après compaction :
1. Les caches memoize sont vidés
2. `getMemoryFiles()` relit les fichiers depuis le disque
3. Le system prompt est reconstruit avec du contenu frais
4. La conversation continue avec toute la mémoire intacte

---

## 10. Skill /remember — Curation manuelle

**Fichier** : `skills/bundled/remember.ts`

### Fonctionnement

La commande `/remember` lance un agent qui :

```
1. Lit tous les fichiers de mémoire :
   ├── CLAUDE.md (projet et utilisateur)
   ├── CLAUDE.local.md
   ├── Auto-memory (tous les fichiers)
   └── Team memory (si activée)

2. Produit des propositions groupées :
   ├── Promotions (entrées à déplacer, avec justification)
   ├── Nettoyage (doublons, obsolètes, conflits)
   ├── Ambiguës (nécessitent input utilisateur)
   └── Aucune action (entrées qui restent en place)

3. L'utilisateur approuve/refuse avant toute modification
```

Aucune modification automatique — l'utilisateur a le contrôle total.

---

## 11. Format des fichiers mémoire

### Écriture en deux étapes

**Étape 1** — Écrire le fichier mémoire :

```markdown
---
name: Rôle utilisateur
description: Data scientist focalisé sur l'observabilité et le logging
type: user
---

L'utilisateur est un data scientist senior spécialisé dans les pipelines
de données et l'observabilité. Adapter les explications en termes de
flux de données et de métriques.
```

**Étape 2** — Ajouter un pointeur dans MEMORY.md :

```markdown
- [Rôle utilisateur](user_role.md) — Data scientist focalisé sur l'observabilité
```

### Structure du frontmatter

| Champ | Requis | Description |
|-------|--------|-------------|
| `name` | Oui | Nom de la mémoire |
| `description` | Oui | Description d'une ligne (utilisée pour le tri par pertinence) |
| `type` | Oui | `user`, `feedback`, `project`, `reference` |

### Structure du contenu par type

**`feedback`** et **`project`** utilisent une structure spécifique :

```markdown
Règle ou fait principal.

**Why:** Raison donnée par l'utilisateur (souvent un incident passé).

**How to apply:** Quand et où appliquer cette règle.
```

---

## 12. Injection dans le system prompt

### Assemblage du system prompt

**Fichier** : `utils/systemPrompt.ts` → `buildEffectiveSystemPrompt()`

```
1. Override System Prompt (--system-prompt)     ← REMPLACE tout
2. Coordinator Mode Prompt (si activé)
3. Agent System Prompt (si mainThreadAgentDefinition)
4. Custom System Prompt (si --system-prompt)
5. Default System Prompt (getSystemPrompt())
6. ★ Memory Prompt (loadMemoryPrompt())        ← MEMORY.MD + instructions
7. Append System Prompt (--append-system-prompt)
```

### Cache par session

Les sections du system prompt sont **memoizées** via `systemPromptSection()` :

```typescript
systemPromptSection('memory', () => loadMemoryPrompt())
```

| Propriété | Détail |
|-----------|--------|
| Stockage | `STATE.systemPromptSectionCache` (Map) |
| Durée de vie | Toute la session |
| Invalidation | `/clear`, `/compact` |
| Rechargement | Depuis le disque (fichiers frais) |

### Variantes du prompt mémoire

| Mode | Prompt | Contenu |
|------|--------|---------|
| **Auto-only** | `buildMemoryLines()` | Répertoire unique, pas de scope |
| **Team combiné** | `buildMemoryPrompt()` | Privé + team, tags `<scope>` |
| **KAIROS** | `buildAssistantDailyLogPrompt()` | Logs append-only, distillation nocturne |

### Chargement lazy (nested)

Quand Claude accède à un fichier qui déclenche des règles conditionnelles :

```
1. getManagedAndUserConditionalRules() → règles managed/user matching
2. Trouver les répertoires entre CWD et fichier cible
3. Pour chaque répertoire : charger règles unconditionnelles + conditionnelles
4. Charger les règles CWD-level conditionnelles
5. Injecter comme attachments
6. Fire hook InstructionsLoaded avec load_reason: 'nested_traversal'
```

---

## 13. Limites et contraintes

### Index MEMORY.md

| Contrainte | Valeur |
|------------|--------|
| Lignes max | 200 |
| Taille max | 25 KB |
| Caractères par entrée | ~150 |
| Troncature | Avertissement ajouté automatiquement |

### Recall (par requête)

| Contrainte | Valeur |
|------------|--------|
| Mémoires sélectionnées | ≤ 5 par requête |
| Budget cumulatif session | 60 KB |
| Lignes max par fichier | 200 |
| Déduplication | Mémoires déjà en contexte exclues |

### Extraction (par turn)

| Contrainte | Valeur |
|------------|--------|
| Turns max de l'agent | 5 |
| Fichiers scannés | ≤ 200 |
| Throttle | Tous les N turns (configurable) |
| Exclusion mutuelle | Skip si l'agent principal a écrit |

### Consolidation (Dream)

| Contrainte | Valeur |
|------------|--------|
| Intervalle min | 24 heures |
| Sessions min | 5 depuis dernière consolidation |
| Throttle scan | 10 minutes entre les scans |
| Lock | Un seul processus, PID reclaim après 1h |

### Session Memory

| Contrainte | Valeur |
|------------|--------|
| Seuil init | 10 000 tokens |
| Seuil update | 5 000 tokens + 3 tool calls |
| Tokens par section | 2 000 max |
| Tokens total | 12 000 max |
| Sections | 9 |

### Post-compaction

| Contrainte | Valeur |
|------------|--------|
| Fichiers restaurés | ≤ 5 |
| Budget total | 50 000 tokens |
| Par fichier | 5 000 tokens max |

---

## 14. Sécurité

| Mesure | Description |
|--------|-------------|
| **Écriture restreinte** | L'agent d'extraction ne peut écrire que dans `~/.claude/projects/<path>/memory/` |
| **Path traversal** | Normalisation, rejet de `..`, UNC, racines |
| **Null bytes** | Rejetés |
| **Unicode attacks** | Rejetés (backslashes, encodage URL) |
| **HTML comments** | Strippés des CLAUDE.md |
| **Extensions binaires** | Rejetées des @includes |
| **Données sensibles** | Avertissement pour team memory |
| **projectSettings** | CLAUDE.md projet exclu si non trusté (sécurité repo malicieux) |
| **Permissions fichier** | 0o600 (lecture/écriture propriétaire uniquement) |

---

## 15. Feature gates

| Gate | Système |
|------|---------|
| `tengu_passport_quail` | Extraction de mémoires activée |
| `tengu_slate_thimble` | Extraction en sessions non-interactives |
| `tengu_bramble_lintel` | Throttle d'extraction (intervalle de turns) |
| `tengu_herring_clock` | Team memory activée |
| `tengu_onyx_plover` | Auto-dream (seuils minHours, minSessions) |
| `tengu_moth_copse` | Skip MEMORY.md index (mode logs-only, KAIROS) |
| `tengu_session_memory` | Session memory activée |
| `tengu_coral_fern` | Section "Searching past context" |
| `tengu_paper_halyard` | Skip fichiers Project/Local dans CLAUDE.md |

---

## 16. Diagramme du cycle de vie complet

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DÉMARRAGE DE SESSION                              │
├─────────────────────────────────────────────────────────────────────┤
│  ensureMemoryDirExists()                                            │
│  ├── Crée ~/.claude/projects/<path>/memory/ si absent               │
│  loadMemoryPrompt()                                                 │
│  ├── Lit MEMORY.md (tronqué si >200 lignes / 25KB)                 │
│  ├── Construit les instructions mémoire (taxonomie 4 types)        │
│  └── Injecte dans le system prompt (caché pour la session)          │
│  getClaudeMds()                                                     │
│  ├── Charge CLAUDE.md (managed → user → project → local)           │
│  └── Applique filtres, @includes, stripping HTML                    │
└─────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    REQUÊTE UTILISATEUR                               │
├─────────────────────────────────────────────────────────────────────┤
│  startRelevantMemoryPrefetch() [non-bloquant]                       │
│  ├── Scanne le manifeste mémoire (≤200 fichiers, frontmatter)      │
│  ├── Sonnet sélectionne ≤5 mémoires pertinentes                    │
│  ├── Filtre celles déjà dans le contexte                            │
│  ├── Lit les fichiers sélectionnés (max 200 lignes)                │
│  └── Injecte comme <system-reminder> attachments                    │
│       └── Avertissement fraîcheur si >1 jour                       │
│                                                                     │
│  [Modèle génère la réponse + appels d'outils]                      │
│                                                                     │
│  Accès fichier → lazy load de .claude/rules/ conditionnelles        │
└─────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    FIN DE TOUR (Stop Hooks)                          │
├─────────────────────────────────────────────────────────────────────┤
│  executeExtractMemories() [fire-and-forget]                         │
│  ├── Gates : enabled, not remote, main agent, mutual exclusion      │
│  ├── Throttle : skip si < N tours                                   │
│  ├── Fork parfait (partage prompt cache)                            │
│  ├── Manifeste pré-injecté (200 fichiers)                           │
│  ├── Tour 1 : Reads parallèles                                     │
│  ├── Tour 2 : Writes parallèles                                    │
│  ├── Met à jour MEMORY.md (≤200 lignes, 25KB)                      │
│  └── Coalescence si requête pendante                                │
│                                                                     │
│  executeAutoDream() [fire-and-forget, si gates passent]             │
│  ├── Acquiert .consolidate-lock                                     │
│  ├── Phase 1 : Orient                                               │
│  ├── Phase 2 : Gather signal                                        │
│  ├── Phase 3 : Consolidate                                          │
│  ├── Phase 4 : Prune & index                                        │
│  └── Rollback mtime si échec                                       │
└─────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    COMPACTION (si limite tokens)                     │
├─────────────────────────────────────────────────────────────────────┤
│  1. Résumer les messages anciens                                    │
│  2. Vider tous les caches :                                         │
│     ├── resetGetMemoryFilesCache('compact')                         │
│     ├── getUserContext.cache.clear()                                │
│     └── clearSystemPromptSections()                                 │
│  3. Post-compaction :                                               │
│     ├── Recharger CLAUDE.md depuis le disque (frais)                │
│     ├── Recharger MEMORY.md dans le system prompt                   │
│     ├── Restaurer ≤5 fichiers (budget 50K tokens)                   │
│     ├── Restaurer skills et plans                                   │
│     └── Fire hook InstructionsLoaded(load_reason: 'compact')        │
│  4. La conversation continue avec mémoire intacte                   │
└─────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    PROCHAINE SESSION                                 │
├─────────────────────────────────────────────────────────────────────┤
│  MEMORY.md rechargé depuis le disque (100% frais)                   │
│  Mémoires pertinentes injectées par requête via Sonnet              │
│  Le cycle recommence                                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 17. Fichiers clés

### Stockage et index

| Fichier | Rôle |
|---------|------|
| `memdir/memdir.ts` | Orchestrateur principal : prompt building, truncation index |
| `memdir/paths.ts` | Résolution chemins, validation, activation |
| `memdir/memoryScan.ts` | Scan fichiers, parsing frontmatter, formatage manifeste |
| `memdir/memoryTypes.ts` | Taxonomie 4 types, guidelines de sauvegarde |
| `memdir/memoryAge.ts` | Calcul de fraîcheur, caveat staleness |

### CLAUDE.md

| Fichier | Rôle |
|---------|------|
| `utils/claudemd.ts` | Chargement hiérarchique, @includes, règles conditionnelles |
| `context.ts` | Assemblage contexte utilisateur/système, memoization |
| `utils/systemPrompt.ts` | Construction du system prompt effectif |
| `constants/prompts.ts` | Sections système, cache par session |

### Recall

| Fichier | Rôle |
|---------|------|
| `memdir/findRelevantMemories.ts` | Sélecteur Sonnet, manifeste, dédup |
| `utils/attachments.ts` | Prefetch, injection attachments, budget session |

### Extraction

| Fichier | Rôle |
|---------|------|
| `services/extractMemories/extractMemories.ts` | Orchestration extraction, fork, coalescence |
| `services/extractMemories/prompts.ts` | Prompts auto-only et combiné |
| `query/stopHooks.ts` | Déclencheur fire-and-forget |

### Dream

| Fichier | Rôle |
|---------|------|
| `services/autoDream/autoDream.ts` | Orchestration, 5 gates |
| `services/autoDream/consolidationPrompt.ts` | Instructions 4 phases |
| `services/autoDream/consolidationLock.ts` | Verrou fichier, PID, rollback |

### Session Memory

| Fichier | Rôle |
|---------|------|
| `services/SessionMemory/sessionMemory.ts` | Extraction par seuil, 9 sections |
| `services/SessionMemory/prompts.ts` | Template, prompt personnalisable |

### Team Memory

| Fichier | Rôle |
|---------|------|
| `memdir/teamMemPaths.ts` | Chemins team memory, validation |
| `memdir/teamMemPrompts.ts` | Prompt combiné privé + team |
| `services/teamMemorySync/` | Synchronisation au démarrage |

### Compaction

| Fichier | Rôle |
|---------|------|
| `services/compact/postCompactCleanup.ts` | Invalidation caches, rechargement |
| `services/compact/compact.ts` | Flow complet, budget restauration |
| `services/compact/sessionMemoryCompact.ts` | Compaction session memory |

### Curation

| Fichier | Rôle |
|---------|------|
| `skills/bundled/remember.ts` | Skill /remember, promotions, nettoyage |
