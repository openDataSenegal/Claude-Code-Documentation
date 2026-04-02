# Systèmes Internes de Claude Code

Ce document décrit les sous-systèmes notables découverts dans le code source, au-delà de l'architecture générale et des feature flags.

---

## Table des matières

1. [Query Pipeline — Le coeur du moteur](#1-query-pipeline)
2. [Compaction — Gestion du contexte](#2-compaction)
3. [Buddy — Le compagnon Tamagotchi](#3-buddy)
4. [Dream — Consolidation de mémoire](#4-dream)
5. [Kairos — Mode assistant persistant](#5-kairos)
6. [Coordinator — Orchestration multi-agents](#6-coordinator)
7. [Chicago — Computer Use (contrôle écran)](#7-chicago)
8. [Tungsten — Multiplexeur terminal tmux](#8-tungsten)
9. [Lodestone — Deep Links (claude-cli://)](#9-lodestone)
10. [UltraPlan — Planification longue durée](#10-ultraplan)
11. [Ink — Renderer React terminal custom](#11-ink)
12. [Système de permissions](#12-permissions)
13. [Hooks — Automatisation événementielle](#13-hooks)
14. [MCP — Model Context Protocol](#14-mcp)
15. [Skills — Extensions par markdown](#15-skills)
16. [Outils — Catalogue complet](#16-outils)
17. [Télémétrie & Analytics](#17-telemetrie)

---

## 1. Query Pipeline

**Fichiers clés** : `query.ts` (1729 lignes), `query/config.ts`, `query/stopHooks.ts`, `query/tokenBudget.ts`

Le query pipeline est le moteur central de Claude Code. C'est un **async generator** qui orchestre chaque échange utilisateur → modèle → outils → réponse.

### Cycle d'une requête

```
Message utilisateur
    ↓
1. Memory prefetch (rappel de mémoires pertinentes)
2. Job classification (dispatch template si TEMPLATES activé)
3. Assemblage du system prompt (buddy, coordinator, computer use...)
4. Requête API Claude avec streaming
    ↓
    ├── Tool execution (interleaved via StreamingToolExecutor)
    ├── Recovery paths :
    │   ├── max_output_tokens → retry avec budget étendu
    │   ├── prompt_too_long  → déclenche compaction
    │   └── API errors       → retry avec backoff
    ↓
5. Post-sampling hooks (persistence état, session)
6. Stop hooks (extractMemories, autoDream, promptSuggestion)
    ↓
7. Analytics (Datadog + 1P logging)
    ↓
Réponse à l'utilisateur
```

### Points notables

- Le loop continue tant que : pas d'interruption utilisateur, budget tokens non dépassé, max turns non atteint
- L'exécution des outils est **streamée et interleaved** avec l'inférence (gate `tengu_streaming_tool_execution2`)
- `QueryConfig` est un snapshot immutable pris une seule fois en entrée de boucle
- Le token budget est optionnel (feature flag `TOKEN_BUDGET`)

---

## 2. Compaction

**Fichiers clés** : `services/compact/compact.ts` (1705 lignes), `autoCompact.ts`, `microCompact.ts`, `sessionMemoryCompact.ts`

Système multi-stratégie pour compresser l'historique de conversation quand le contexte approche la limite API.

### Stratégies

| Stratégie | Description |
|-----------|-------------|
| **Full compaction** | Claude résume tout l'historique pré-boundary en un résumé concis |
| **Microcompaction** | Résumé léger in-API via paramètre `budget_tokens` (5k/skill, 25k total) |
| **Reactive compact** | Détecte `prompt_too_long` mid-stream et déclenche un résumé |
| **Session memory compact** | Extrait les apprentissages conceptuels en fichiers mémoire structurés |
| **Auto-compact** | Proactif : surveille la trajectoire de tokens et compacte avant la limite |
| **Context collapse** | Optimisation supplémentaire du contexte (feature flag) |

### Fonctionnement

- Post-compaction : restaure jusqu'à 5 fichiers (budget 50k tokens) dans le contexte
- `CompactBoundaryMessage` marque la frontière entre historique résumé et messages actifs
- `AutoCompactTrackingState` suit la trajectoire d'utilisation des tokens

---

## 3. Buddy — Le compagnon Tamagotchi

**Fichiers clés** : `buddy/companion.ts`, `buddy/types.ts`, `buddy/CompanionSprite.tsx`, `buddy/useBuddyNotification.tsx`

Un compagnon virtuel personnel, façon Tamagotchi, affiché à côté du terminal.

### Génération déterministe

Le buddy est **généré de façon déterministe** à partir d'un hash de l'`userId` via un PRNG Mulberry32. Pas de RNG, pas de serveur — le même user aura toujours le même buddy.

### Caractéristiques

| Attribut | Détails |
|----------|---------|
| **Espèces** | 18 : duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk |
| **Raretés** | Common (100%), Uncommon, Rare, Epic, Legendary (1%) |
| **Stats** | DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK — 1 stat peaked, 1 dump stat, le reste aléatoire |
| **Shiny** | 1% de chance indépendante |
| **Visuel** | Sprites ASCII 5×12 avec frames d'animation (idle, fidget, blink) |
| **Âme** | Nom + personnalité générés par Claude au premier lancement, stockés en config |

### Comportement

- Quand l'utilisateur s'adresse au buddy par son nom, la bulle du buddy répond (pas Claude)
- Feature gate : `feature('BUDDY')`
- Easter egg prévu pour le 1-7 avril 2026, lancement complet mai 2026

---

## 4. Dream — Consolidation de mémoire

**Fichiers clés** : `services/autoDream/autoDream.ts` (325 lignes), `consolidationPrompt.ts`, `consolidationLock.ts`, `tasks/DreamTask/`

Inspiré de la consolidation mnésique humaine pendant le sommeil, ce système "rêve" en arrière-plan pour organiser les mémoires.

### Déclenchement (3 portes)

1. **Time gate** : 24h minimum depuis la dernière consolidation
2. **Session gate** : au moins 5 sessions depuis la dernière consolidation
3. **Lock gate** : un seul dream à la fois (fichier verrou)

### Processus en 4 phases

```
1. Orient       → Comprendre le contexte actuel
2. Gather Signal → Scanner les sessions récentes
3. Consolidate   → Fusionner les mémoires, convertir dates relatives → absolues
4. Prune/Index   → Supprimer les contradictions, garder MEMORY.md < 200 lignes / ~25KB
```

### Contraintes

- Exécuté via un agent forké avec Bash **lecture seule**
- `querySource: 'auto_dream'`, `skipTranscript: true`
- Désactivé quand KAIROS est actif
- Throttle de 10 min entre les scans si le session gate ne passe pas
- Config via feature config `tengu_onyx_plover`

---

## 5. Kairos — Mode assistant persistant

**Fichiers clés** : `bootstrap/state.ts` (`getKairosActive`/`setKairosActive`), `main.tsx` ~1081

Un mode opératoire spécial où Claude Code devient un **assistant asynchrone persistant** qui n'attend pas l'input utilisateur.

### Caractéristiques

- Envoie des `<tick>` prompts à intervalles réguliers pour action proactive
- Budget de 15 secondes non-bloquant pour le travail proactif
- Outils exclusifs : `SendUserFile`, `PushNotification`, `SubscribePR`
- Journal append-only quotidien pour décisions/observations
- Force le mode brief (`opts.brief = true`)
- Pré-seed d'une équipe in-process (pas besoin de `TeamCreate`)
- Désactive auto-dream pendant son activité

---

## 6. Coordinator — Orchestration multi-agents

**Fichier clé** : `coordinator/coordinatorMode.ts` (370 lignes)

Mode où Claude Code agit comme **coordinateur** déléguant à des agents workers.

### Architecture

```
Coordinateur (Claude principal)
    ├── Worker 1 (recherche)     ← Agent forké, outils complets
    ├── Worker 2 (implémentation) ← Agent forké, Bash/Read/Edit
    └── Worker 3 (vérification)  ← Agent forké, outils complets
```

### Fonctionnement

- Activé via `CLAUDE_CODE_COORDINATOR_MODE` env var
- Workers communiquent via `<task-notification>` XML dans les messages
- Outils du coordinateur : `Agent`, `SendMessage`, `TaskStop`
- Philosophie : *"Parallelism is your superpower. Workers are async. Launch independent workers concurrently whenever possible"*
- Phases : Research → Synthesis → Implementation → Verification
- Scratchpad partagé pour la connaissance inter-workers (feature-gated)

---

## 7. Chicago — Computer Use

**Fichiers clés** : `utils/computerUse/gates.ts`, `mcpServer.ts`, `setup.ts`, `hostAdapter.ts`

Intégration du **contrôle d'écran** (Computer Use) via MCP, permettant à Claude d'interagir avec le bureau.

### Capacités

- Capture d'écran
- Mouvement et clics souris (avec animation optionnelle)
- Saisie clavier avec paste multilignes via clipboard
- Système de coordonnées : pixels ou normalisé (figé au démarrage)
- Détection automatique de l'affichage cible
- Protection clipboard contre les pastes accidentels

### Accès

- Gate : `tengu_malort_pedway` (GrowthBook)
- Réservé aux abonnés Max/Pro (bypass pour employés Anthropic via `USER_TYPE=ant`)
- Serveur MCP in-process créé à la première connexion
- Sous-gates granulaires : `pixelValidation`, `mouseAnimation`, etc.

---

## 8. Tungsten — Multiplexeur terminal

**Fichiers clés** : `tools/TungstenTool/TungstenTool.ts`, `TungstenLiveMonitor.tsx`, `utils/tmuxSocket.ts`

Abstraction de terminal virtuel via **tmux** pour exécuter des shells parallèles. Réservé aux employés Anthropic.

### Fonctionnement

- Singleton : une session tmux par socket
- Sessions persistantes, switchables
- UI en temps réel dans le terminal (pilule tmux + panneau)
- Auto-hide du panneau à la fin d'un tour
- Config sticky : `config.tungstenPanelVisible`
- Le BashTool peut dispatcher vers des sessions tmux

---

## 9. Lodestone — Deep Links

**Fichier clé** : `utils/deepLink/registerProtocol.ts` (349 lignes)

Enregistrement du schéma URI `claude-cli://` au niveau OS.

### Implémentation par plateforme

| OS | Méthode |
|----|---------|
| **macOS** | Crée un bundle `.app` dans `~/Applications`, enregistre via LaunchServices |
| **Linux** | Crée un `.desktop` dans `$XDG_DATA_HOME/applications`, appelle `xdg-mime` |
| **Windows** | Écrit les clés registre sous `HKEY_CURRENT_USER\Software\Classes` |

- Auto-enregistré au premier lancement
- Backoff de 24h sur erreurs persistantes (`EACCES`, `ENOSPC`)
- Gate : `tengu_lodestone_enabled`

---

## 10. UltraPlan — Planification longue durée

Délègue la planification complexe à une **session CCR distante sur Opus 4.6** avec jusqu'à 30 minutes de réflexion.

### Fonctionnement

- UI navigateur pour approbation/rejet du plan
- Sentinel `__ULTRAPLAN_TELEPORT_LOCAL__` pour transférer le résultat
- Polling toutes les 3 secondes depuis le terminal
- Feature flag : `ULTRAPLAN`

---

## 11. Ink — Renderer React terminal custom

**Répertoire** : `ink/` (50 fichiers)

Claude Code n'utilise pas Ink standard — il embarque son **propre renderer React pour le terminal**.

### Composants

- Abstraction DOM-like : `render-node-to-output`, reconciler, screen manager
- Système de sélection avec copy-on-select
- Parsing ANSI couleurs/styles avec support thème
- Parsing keypress avec interaction terminal complète
- Mesure de texte et line wrapping
- Rendu de bordures et composants de layout
- Basé sur le moteur de layout **Yoga** (flexbox en terminal)

---

## 12. Système de permissions

**Fichiers clés** : `utils/permissions/permissions.ts` (52KB), `PermissionMode.ts`, `yoloClassifier.ts`

### Modes

| Mode | Comportement |
|------|-------------|
| `default` | Demande avant opérations risquées |
| `bypassPermissions` | Autorise tout (YOLO) |
| `acceptEdits` | Autorise les edits, demande pour le reste |
| `dontAsk` | Ne demande jamais (deny par défaut) |
| `plan` | Mode plan (restrictions) |
| `auto` | Classifieur ML décide |
| `bubble` | Mode interne pour sous-agents |

### Pipeline de décision

```
1. Vérifier règles explicites (allow/deny)
2. Vérifier le mode de permission
3. Lancer classifieurs (bash classifier, transcript classifier)
4. Lancer hooks (permission_request)
5. Si toujours indécis → prompt utilisateur
```

### Classifieurs

- **Bash Classifier** : analyse les commandes shell pour patterns dangereux (`rm -rf /`, accès DB, credentials)
- **Transcript Classifier** : modèle ML entraîné sur l'historique des commandes
- Sources de règles : user, project, local, CLI, command, session, policy

---

## 13. Hooks — Automatisation événementielle

**Fichiers clés** : `utils/hooks.ts` (600 lignes), `utils/hooks/sessionHooks.ts`, `AsyncHookRegistry.ts`

### Événements disponibles

| Catégorie | Événements |
|-----------|-----------|
| **Session** | `session_start`, `session_end`, `setup` |
| **Outils** | `pre_tool_use`, `post_tool_use`, `post_tool_use_failure` |
| **Fichiers** | `file_changed`, `cwd_changed` |
| **Config** | `config_change`, `instructions_loaded` |
| **Tâches** | `task_created`, `task_completed` |
| **Utilisateur** | `user_prompt_submit` |
| **Notifications** | `notification` |
| **Agents** | `subagent_start`, `subagent_stop` |
| **Permissions** | `permission_denied`, `permission_request` |
| **Compaction** | `pre_compact`, `post_compact` |
| **Coéquipiers** | `teammate_idle` |

### Stratégies d'exécution

- **Shell** : scripts exécutés en subprocess
- **HTTP** : appels à des endpoints
- **Prompt** : fonctions inline

### I/O

- Entrée : `HookInput` (JSON structuré avec détails de l'événement)
- Sortie : `HookJSONOutput` (async boolean, timeout, résultat)
- Timeouts : 10 min (outils), 1.5s (fin de session)

---

## 14. MCP — Model Context Protocol

**Fichiers clés** : `services/mcp/client.ts` (119KB), `config.ts`, `auth.ts` (88KB)

### Sources de configuration

1. `.mcp.json` à la racine du projet
2. `~/.claude/settings.json`
3. Configs enterprise managées

### Transports supportés

- **stdio** (subprocess)
- **SSE** (Server-Sent Events)
- **HTTP** (Streamable HTTP)
- **WebSocket**

### Fonctionnement

- Les outils MCP sont wrappés en `MCPTool` avec nommage `{serveur}__{outil}`
- Ressources exposées via `ListMcpResourcesTool` et `ReadMcpResourceTool`
- Authentification OAuth complète avec refresh de tokens
- Elicitation handler pour step-up auth depuis les serveurs
- Registre officiel MCP supporté (`officialRegistry.ts`)

---

## 15. Skills — Extensions par markdown

**Fichiers clés** : `skills/loadSkillsDir.ts`, `skills/bundledSkills.ts`, `skills/bundled/`

### Sources

| Source | Chemin |
|--------|--------|
| Utilisateur | `~/.claude/skills/*.md` |
| Projet | `.claude/skills/*.md` (versionné git) |
| Plugins | via système de plugins |
| Bundled | compilé dans le binaire (18 skills) |

### Format

```markdown
---
name: mon-skill
description: Ce que fait le skill
whenToUse: Quand l'utiliser automatiquement
allowedTools: [Bash, Read, Edit]
model: opus
---

Instructions du skill en markdown...
```

### Skills bundled (18)

`batch`, `claudeApi`, `claudeApiContent`, `claudeInChrome`, `debug`, `keybindings`, `loop`, `loremIpsum`, `remember`, `scheduleRemoteAgents`, `simplify`, `skillify`, `stuck`, `updateConfig`, `verify`, `verifyContent`

---

## 16. Outils — Catalogue complet

40+ outils répartis par catégorie :

| Catégorie | Outils |
|-----------|--------|
| **Fichiers** | FileReadTool, FileEditTool, FileWriteTool, GlobTool, GrepTool |
| **Exécution** | BashTool, PowerShellTool, REPLTool (Ant-only) |
| **Agents** | AgentTool, SkillTool |
| **Web** | WebFetchTool, WebSearchTool, WebBrowserTool |
| **Planification** | EnterPlanModeTool, ExitPlanModeTool |
| **Tâches** | TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool, TaskStopTool, TaskOutputTool |
| **Worktrees** | EnterWorktreeTool, ExitWorktreeTool |
| **MCP** | MCPTool, ListMcpResourcesTool, ReadMcpResourceTool, McpAuthTool |
| **Config** | ConfigTool, ToolSearchTool, LSPTool |
| **Notebooks** | NotebookEditTool |
| **Équipe** | TeamCreateTool, TeamDeleteTool, SendMessageTool, RemoteTriggerTool |
| **Utilitaires** | AskUserQuestionTool, BriefTool, SleepTool, SyntheticOutputTool, TodoWriteTool |
| **Cron** | CronCreateTool, CronDeleteTool, CronListTool |
| **Terminal** | TerminalCaptureTool (feature-gated), TungstenTool (Ant-only) |

### Exécution

- Chaque outil est une classe avec : `name`, `description`, `input` (JSON schema), `handler()`
- Exécution via `StreamingToolExecutor` avec contrôle de concurrence
- Outils exclusifs (mutex) vs concurrents
- Hooks pre/post exécution intégrés

---

## 17. Télémétrie & Analytics

**Fichiers clés** : `services/analytics/index.ts`, `metadata.ts` (733 lignes), `growthbook.ts`, `datadog.ts`

### Architecture multi-sink

```
logEvent() / logEventAsync()
    ↓
Event Queue (buffer jusqu'à l'attachement du sink)
    ↓
    ├── Datadog          → Métriques générales
    ├── 1P Event Logger  → Événements Anthropic
    ├── Session Tracing  → Enregistrement complet de session
    └── Perfetto         → Profiling performance
```

### Protections

- PII : clés `_PROTO_*` routées vers colonnes BigQuery sécurisées
- Type safety : marqueur `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`
- Sink killswitch pour couper la télémétrie
- Sanitization des métadonnées avant logging

### Métriques OpenTelemetry

Sessions, lignes de code, PRs, coût, tokens, éditions de code, temps actif.

---

## Bonus : Easter eggs et curiosités

| Trouvaille | Détail |
|------------|--------|
| **PRNG Mulberry32** | Utilisé pour le gacha déterministe du buddy |
| **Rendu ANSI → PNG** | 214KB — blit direct de la police Fira Code en bitmap, sans passer par SVG |
| **Undercover mode** | Système de rédaction des noms internes (ironie, vu le leak) |
| **Rainbow shimmer** | Couleurs arc-en-ciel pour l'output de thinking |
| **Sentinel téléport** | `__ULTRAPLAN_TELEPORT_LOCAL__` pour transfert de résultats distants |
| **Âme du buddy** | Générée par Claude lui-même au premier hatch |
| **15s blocking budget** | Concept unique dans Kairos pour le travail proactif |
