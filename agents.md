# Agents de Claude Code — Documentation complète

Ce document décrit l'ensemble du système d'agents de Claude Code : les types d'agents disponibles, leur cycle de vie, le modèle de communication multi-agents (swarm), et les mécanismes d'isolation et de permissions.

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Agents built-in](#2-agents-built-in)
3. [Agents conditionnels et spéciaux](#3-agents-conditionnels-et-spéciaux)
4. [Agents custom (utilisateur)](#4-agents-custom)
5. [Cycle de vie d'un agent](#5-cycle-de-vie-dun-agent)
6. [Communication multi-agents (Swarm)](#6-communication-multi-agents)
7. [Isolation et worktrees](#7-isolation-et-worktrees)
8. [Héritage et permissions](#8-héritage-et-permissions)
9. [Agents distants (CCR)](#9-agents-distants)
10. [Définition d'un agent — Structure complète](#10-définition-dun-agent)
11. [Fichiers clés](#11-fichiers-clés)

---

## 1. Vue d'ensemble

Claude Code utilise un système d'agents hiérarchique : un agent principal (le "leader") peut spawner des sous-agents spécialisés via l'outil `Agent`. Chaque agent s'exécute dans un contexte isolé (AsyncLocalStorage) avec ses propres outils, permissions et état.

### Mécanismes de spawn

| Mécanisme | Description |
|-----------|-------------|
| **In-process** | Sous-agent dans le même processus, contexte AsyncLocalStorage |
| **Fork** | Clone le contexte conversationnel complet du parent |
| **Worktree** | Copie isolée du repo git dans un répertoire temporaire |
| **Remote (CCR)** | Exécution distante via Claude Code Remote (HTTP/WebSocket) |

### Sélection par défaut

- Si `subagent_type` est omis et fork gate OFF → agent `general-purpose`
- Si `subagent_type` est omis et fork gate ON → fork implicite (hérite du contexte complet)
- Si `subagent_type` est spécifié → agent correspondant

---

## 2. Agents built-in

### 2.1 general-purpose

| | |
|---|---|
| **Type** | `general-purpose` |
| **Fichier** | `tools/AgentTool/built-in/generalPurposeAgent.ts` |
| **Outils** | Tous (`['*']`) |
| **Modèle** | Hérite du parent (`getDefaultSubagentModel()`) |
| **Disponibilité** | Toujours disponible |

**Description** : Agent polyvalent pour la recherche complexe, la recherche de code et l'exécution de tâches multi-étapes. Utilisé par défaut quand aucun `subagent_type` n'est spécifié (et que le fork gate est OFF).

---

### 2.2 Explore

| | |
|---|---|
| **Type** | `Explore` |
| **Fichier** | `tools/AgentTool/built-in/exploreAgent.ts` |
| **Outils autorisés** | Glob, Grep, Read (+ Bash embedded sur builds Ant) |
| **Outils interdits** | Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit |
| **Modèle** | `haiku` (externe) / `inherit` (Ant, via gate `tengu_explore_agent`) |
| **Disponibilité** | Feature flag `BUILTIN_EXPLORE_PLAN_AGENTS` |
| **One-shot** | Oui |

**Description** : Agent rapide spécialisé dans l'exploration de codebase. Lecture seule. Supporte trois niveaux de profondeur : `quick`, `medium`, `very thorough`.

**Particularités** :
- Omet CLAUDE.md du contexte (économie de tokens, pas besoin de règles commit/PR)
- Minimum 3 requêtes par invocation (`EXPLORE_AGENT_MIN_QUERIES`)

---

### 2.3 Plan

| | |
|---|---|
| **Type** | `Plan` |
| **Fichier** | `tools/AgentTool/built-in/planAgent.ts` |
| **Outils autorisés** | Glob, Grep, Read (+ Bash embedded sur builds Ant) |
| **Outils interdits** | Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit |
| **Modèle** | `inherit` |
| **Disponibilité** | Feature flag `BUILTIN_EXPLORE_PLAN_AGENTS` |
| **One-shot** | Oui |

**Description** : Agent architecte pour la conception de plans d'implémentation. Lecture seule. Identifie les fichiers critiques, considère les trade-offs architecturaux.

**Particularités** :
- Doit produire une section "Critical Files for Implementation" en sortie
- Omet CLAUDE.md (peut le lire directement si besoin)

---

### 2.4 claude-code-guide

| | |
|---|---|
| **Type** | `claude-code-guide` |
| **Fichier** | `tools/AgentTool/built-in/claudeCodeGuideAgent.ts` |
| **Outils autorisés** | Glob, Grep, Read, WebFetch, WebSearch (externe) / Bash, Read, WebFetch, WebSearch (Ant) |
| **Modèle** | `haiku` |
| **Mode permission** | `dontAsk` |
| **Disponibilité** | Toujours (sauf mode SDK) |

**Description** : Agent guide pour les questions sur Claude Code, l'Agent SDK et l'API Claude. Répond aux questions "Comment faire...", "Est-ce que Claude peut...", etc.

**Particularités** :
- System prompt dynamique incluant : skills custom, agents configurés, serveurs MCP, plugins, settings
- Fetche la documentation depuis :
  - `code.claude.com/docs/en/claude_code_docs_map.md`
  - `platform.claude.com/llms.txt`
- Réutilisable : avant de spawner un nouveau guide, vérifier si un précédent est encore actif

---

### 2.5 verification

| | |
|---|---|
| **Type** | `verification` |
| **Fichier** | `tools/AgentTool/built-in/verificationAgent.ts` |
| **Outils interdits** | Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit |
| **Modèle** | `inherit` |
| **Couleur** | Rouge |
| **Background** | Toujours |
| **Disponibilité** | Feature flag `VERIFICATION_AGENT` + gate `tengu_hive_evidence` |

**Description** : Agent de vérification post-implémentation. Lance builds, tests, linters et produit un verdict PASS/FAIL/PARTIAL avec preuves.

**Particularités** :
- Lecture seule (sauf `/tmp` pour scripts de test éphémères)
- Doit inclure au moins une sonde adversariale (concurrence, boundary, idempotence, orphan op)
- Critical System Reminder réinjecté à chaque tour :
  > *"CRITICAL: This is a VERIFICATION-ONLY task. You CANNOT edit, write, or create files IN THE PROJECT DIRECTORY. You MUST end with VERDICT: PASS, VERDICT: FAIL, or VERDICT: PARTIAL."*
- Invoqué automatiquement après : 3+ éditions de fichiers, changements backend/API, changements infra

---

### 2.6 statusline-setup

| | |
|---|---|
| **Type** | `statusline-setup` |
| **Fichier** | `tools/AgentTool/built-in/statuslineSetup.ts` |
| **Outils autorisés** | Read, Edit |
| **Modèle** | `sonnet` |
| **Couleur** | Orange |
| **Disponibilité** | Toujours |

**Description** : Agent spécialisé dans la configuration de la status line de Claude Code. Convertit les configurations PS1 shell en commandes statusLine.

**Particularités** :
- Met à jour `~/.claude/settings.json`
- Peut créer des scripts shell dans `~/.claude/statusline-command.sh`

---

## 3. Agents conditionnels et spéciaux

### 3.1 fork (implicite)

| | |
|---|---|
| **Type** | `fork` (synthétique, pas sélectionnable) |
| **Fichier** | `tools/AgentTool/forkSubagent.ts` |
| **Outils** | Tous (pool exact du parent pour l'identité cache) |
| **Modèle** | `inherit` |
| **Mode permission** | `bubble` (remonte au parent) |
| **Max turns** | 200 |
| **Disponibilité** | Feature flag `FORK_SUBAGENT` |

**Description** : Fork implicite qui hérite du contexte conversationnel complet du parent. Non sélectionnable par nom — déclenché quand `subagent_type` est omis et le fork gate est actif.

**Particularités** :
- Toujours en arrière-plan (async)
- Utilise `buildForkedMessages()` pour cloner le message assistant + résultats d'outils placeholder
- Protection contre le forking récursif
- Pas de mode coordinateur ni sessions non-interactives

---

### 3.2 worker (Coordinator Mode)

| | |
|---|---|
| **Type** | `worker` |
| **Fichier** | `coordinator/coordinatorMode.ts` (chargé via `getCoordinatorAgents()`) |
| **Disponibilité** | Feature flag `COORDINATOR_MODE` + env `CLAUDE_CODE_COORDINATOR_MODE` |

**Description** : Agent worker autonome dans le mode coordinateur. Le coordinateur orchestre plusieurs workers pour des workflows parallèles.

**Utilisation** :
```
Coordinateur
  ├── Worker 1 (recherche)      ← subagent_type: "worker"
  ├── Worker 2 (implémentation) ← subagent_type: "worker"
  └── Worker 3 (vérification)   ← subagent_type: "worker"
```

**Particularités** :
- Résultats transmis via `<task-notification>` XML
- Continuable via `SendMessage`
- Stoppable via `TaskStop`
- Phases : Research → Synthesis → Implementation → Verification

---

## 4. Agents custom

Les utilisateurs peuvent définir leurs propres agents via des fichiers markdown.

### Sources de découverte

| Source | Chemin | Portée |
|--------|--------|--------|
| Projet | `.claude/agents/*.md` | Versionné git, partagé avec l'équipe |
| Utilisateur | `~/.claude/agents/*.md` | Personnel, tous projets |
| Plugins | Manifeste plugin | Via système de plugins |

### Format de définition

```markdown
---
name: mon-agent
description: Description courte
whenToUse: Quand utiliser cet agent automatiquement
tools:
  - Bash
  - Read
  - Edit
disallowedTools:
  - Agent
model: sonnet
effort: high
permissionMode: acceptEdits
maxTurns: 50
color: blue
background: false
isolation: worktree
memory: project
skills:
  - commit
  - review
mcpServers:
  - name: my-server
    transport: stdio
    command: node
    args: [server.js]
hooks:
  session_start:
    - command: echo "Agent started"
---

Instructions système de l'agent en markdown...

Tu es un agent spécialisé dans [domaine].
Tes responsabilités sont :
- ...
- ...
```

### Exemples typiques

| Agent custom | Rôle |
|-------------|------|
| `test-runner` | Exécute les tests après modification de code |
| `api-docs-writer` | Génère la documentation API |
| `code-formatter` | Formate le code selon les conventions |
| `researcher` | Recherche approfondie dans la codebase |

---

## 5. Cycle de vie d'un agent

### Diagramme complet

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CYCLE DE VIE D'UN AGENT                     │
└─────────────────────────────────────────────────────────────────────┘

[1. SPAWN]
  │
  ├── formatAgentId(name, team) → "researcher@my-team"
  ├── generateTaskId('in_process_teammate') → ID unique
  ├── createAbortController() → abort indépendant du parent
  ├── createTeammateContext() → isolation AsyncLocalStorage
  ├── InProcessTeammateTaskState → enregistrement dans AppState
  ├── registerTask() → événement SDK task_started
  └── startInProcessTeammate() → fire-and-forget
      │
      ▼
[2. INITIALISATION]
  │
  ├── runWithTeammateContext(context, async () => {
  │     runWithAgentContext(agentContext, async () => {
  │       runAgent(config)
  │     })
  │   })
  ├── Héritage : permissions, outils, file state, clients MCP
  ├── Clone du cache d'état fichiers (isolé)
  └── Construction du system prompt (custom ou hérité)
      │
      ▼
[3. EXÉCUTION]
  │
  ├── while (!abortController.aborted) {
  │     query(messages, tools, system)  ← boucle API
  │     if (permission prompt) → Leader UI ou mailbox
  │     if (tool use) → exécution avec contexte isolé
  │     append message → allMessages
  │     update AppState.messages (plafonné à 50)
  │     update progress tracker
  │   }
  │
  ├── Transcript persisté : ~/.claude/sessions/{id}/sidechains/{agentId}/
  └── Progression : toolCount, tokenCount, spinnerVerb
      │
      ▼
[4. COMMUNICATION] (si multi-agents)
  │
  ├── Teammate → Teammate : writeToMailbox() + readMailbox()
  ├── Permission requests : SwarmPermissionRequest (fichier)
  ├── Idle notifications : isIdle = true → onIdleCallbacks
  └── Team file updates : membres, modes, statut actif
      │
      ▼
[5. COLLECTE DES RÉSULTATS]
  │
  ├── recordSidechainTranscript(agentId, allMessages)
  ├── writeAgentMetadata(agentId, { duration, usage, error })
  └── AgentToolResult { type, output, usage, durationMs, worktreePath? }
      │
      ▼
[6. TERMINAISON]
  │
  ├── Trois chemins de sortie :
  │   ├── A. Complétion normale → status: 'completed'
  │   ├── B. Abort (user interrupt) → status: 'stopped'
  │   └── C. Kill via backend → status: 'stopped'
  │
  ├── emitTaskTerminatedSdk(taskId, status, summary)
  ├── evictTerminalTask(taskId) → suppression de AppState (après délai)
  └── notified: true
      │
      ▼
[7. NETTOYAGE]
  │
  ├── removeMemberByAgentId(team, agentId)
  ├── destroyWorktree() si utilisé
  ├── unregisterCleanup()
  ├── unregisterPerfettoAgent(agentId)
  └── Sur fin de session : cleanupSessionTeams()
        ├── killOrphanedTeammatePanes()
        ├── destroyWorktree()
        ├── rm ~/.claude/teams/{team}
        └── rm ~/.claude/tasks/{taskId}
```

### États d'un agent

```
spawned → running → idle ⇄ running → completed | stopped | failed
```

| État | Description |
|------|-------------|
| `spawned` | Créé, pas encore démarré |
| `running` | En cours d'exécution (tour API actif) |
| `idle` | En attente de message ou d'input |
| `completed` | Terminé avec succès |
| `stopped` | Arrêté (abort ou kill) |
| `failed` | Erreur non récupérable |

---

## 6. Communication multi-agents

### Architecture Swarm

```
Leader (agent principal)
  │
  ├── Teammate A ◄──── mailbox ────► Teammate B
  │     │                                 │
  │     └──── permission request ────► Leader UI
  │                                       │
  └──── team file ◄──────────────────────┘
```

### Mailbox (fichier)

**Emplacement** : `~/.claude/teams/{teamName}/inboxes/{agentName}.json`

```typescript
TeammateMessage {
  from: string        // ID de l'expéditeur
  text: string        // Contenu
  timestamp: string   // ISO 8601
  read: boolean       // Lu/non lu
  color?: string      // Couleur UI de l'expéditeur
  summary?: string    // Aperçu
}
```

**API** :

| Fonction | Description |
|----------|-------------|
| `writeToMailbox(recipient, message, team)` | Écriture avec verrouillage fichier |
| `readMailbox(agent, team)` | Lecture de tous les messages |
| `readUnreadMessages(agent, team)` | Filtrage des non-lus |
| `markMessageAsReadByIndex(agent, team, index)` | Marquer comme lu |

### Team File

**Emplacement** : `~/.claude/teams/{teamName}/config.json`

```typescript
TeamFile {
  name: string
  createdAt: number
  leadAgentId: string
  leadSessionId?: string
  members: Array<{
    agentId: string
    name: string
    model?: string
    color?: string
    worktreePath?: string
    isActive?: boolean
    mode?: PermissionMode
  }>
  teamAllowedPaths?: Array<{
    path: string
    toolName: string
    addedBy: string
    addedAt: number
  }>
}
```

### Requêtes de permission (async)

**Stockage** :
```
~/.claude/teams/{team}/permissions/pending/{requestId}.json
~/.claude/teams/{team}/permissions/resolved/{requestId}.json
```

```typescript
SwarmPermissionRequest {
  id: string                    // "permission-{timestamp}@{agentId}"
  workerId: string              // Demandeur
  workerName: string
  toolName: string              // "Bash", "Edit", etc.
  toolUseId: string
  description: string
  input: Record<string, unknown>
  status: 'pending' | 'approved' | 'rejected'
  resolvedBy?: 'worker' | 'leader'
  feedback?: string
  updatedInput?: Record<string, unknown>
  permissionUpdates?: PermissionUpdate[]
}
```

### Deux chemins de résolution

**Chemin A — UI Queue (préféré)** :
```
1. Teammate rencontre un prompt de permission
2. Vérifie la queue ToolUseConfirm du leader
3. Ajoute la requête avec workerBadge (couleur)
4. Attend callback onAllow/onReject
5. Reprend l'exécution avec la décision
```

**Chemin B — Mailbox (fallback)** :
```
1. Crée SwarmPermissionRequest
2. writeToMailbox(leadAgentId, request)
3. Poll la mailbox (toutes les 500ms)
4. processMailboxPermissionResponse()
5. Reprend l'exécution
```

---

## 7. Isolation et worktrees

### Mode worktree

Chaque agent peut s'exécuter dans un worktree git isolé, une copie légère du repo.

```
Repo parent (/project)
    │
    ├── Agent 1 → /tmp/worktrees/feature-abc
    ├── Agent 2 → /tmp/worktrees/feature-def
    └── Leader  → /project (original)
```

### Cycle de vie du worktree

**Création** :
```
1. git worktree add .claude/worktrees/{name} {branch}
2. Symlink des répertoires lourds (node_modules) → économie d'espace
3. recordWorktreeSession() → enregistrement dans bootstrap state
4. chdir vers le worktree
```

**Nettoyage** :
```
1. git worktree remove --force {path}
2. Fallback : rm -rf {path}
3. Enregistré dans le team file pour le nettoyage de session
```

**Traduction de chemins** :
- Les chemins hérités du contexte parent sont transparents
- `buildWorktreeNotice()` informe l'agent que les chemins ont été traduits
- L'agent doit relire les fichiers hérités dans son propre worktree

### Mode remote (CCR)

Pour les agents Ant-only, exécution dans un environnement Claude Code Remote distant.

```
CLI local ──── HTTP POST ────► CCR (envoi message)
           ◄── WebSocket ──── CCR (réception messages & événements)
```

Communication bidirectionnelle :
- `SDKMessage` — output régulier
- `SDKControlRequest` — demandes de permission depuis CCR
- `SDKControlResponse` — réponses depuis le CLI local
- `SDKControlCancelRequest` — signal d'interruption

---

## 8. Héritage et permissions

### Cascade des modes de permission

```
1. Teammate spawné avec planModeRequired?
   → Force plan mode (priorité maximale)

2. Parent en mode bypass?
   → Hérite SAUF si planModeRequired

3. Sinon : hérite du mode du parent
   (acceptEdits, default, etc.)
```

### Héritage du contexte

| Élément | Comportement |
|---------|-------------|
| System prompt | Hérité ou override par définition d'agent |
| Outils | Filtrés par `tools`/`disallowedTools` de l'agent |
| Permissions | Mode hérité, règles partagées |
| MCP clients | Connexions héritées |
| File state cache | Cloné (isolation) |
| CLAUDE.md | Inclus sauf si `omitClaudeMd: true` |
| Variables d'env | Héritées (API provider, proxy, etc.) |

### Synchronisation des permissions

Quand un teammate approuve une règle "always allow" :
```typescript
const updates: PermissionUpdate[] = [
  { type: 'allow', toolName: 'Edit', pattern: '*.ts' }
]
// Persisté sur disque → partagé par tous les teammates
persistPermissionUpdates(updates)
// Propagé au leader
applyPermissionUpdates(context, updates)
```

---

## 9. Agents distants

### RemoteSessionManager

**Fichier** : `remote/RemoteSessionManager.ts`

Gère les sessions d'agents distants via Claude Code Remote (CCR).

### Flux de permission distant

```
1. Agent CCR a besoin d'une permission
2. Envoie SDKControlPermissionRequest
3. CLI local affiche PermissionRequest UI
4. Utilisateur approuve/refuse
5. Envoie SDKControlResponse
6. Agent CCR reprend l'exécution
```

### Types de tâches distantes

| Type | Fichier | Description |
|------|---------|-------------|
| `remote_agent` | `tasks/RemoteAgentTask/` | Agent sur CCR avec SSH tunnel |
| `dream` | `tasks/DreamTask/` | Consolidation mémoire en arrière-plan |
| `local_agent` | `tasks/AgentTask/` | Agent local (in-process ou forké) |
| `local_bash` | Inline | Commande bash en background |
| `in_process_teammate` | `tasks/InProcessTeammateTask/` | Teammate swarm |
| `monitor_mcp` | Feature-gated | Monitoring MCP |

---

## 10. Définition d'un agent — Structure complète

```typescript
BaseAgentDefinition {
  // Identité
  agentType: string              // Identifiant unique (ex: "Explore", "worker")
  whenToUse: string              // Description pour le modèle (quand l'utiliser)
  color?: AgentColorName         // Couleur UI (red, blue, green, orange, etc.)

  // Outils
  tools?: string[]               // Outils autorisés (["*"] = tous)
  disallowedTools?: string[]     // Outils explicitement interdits

  // Modèle
  model?: string                 // "inherit" | "sonnet" | "opus" | "haiku"
  effort?: EffortValue           // "trivial" | "standard" | "moderate" | "high" | number

  // Permissions
  permissionMode?: PermissionMode // "acceptEdits" | "plan" | "dontAsk" | "bubble"

  // Exécution
  maxTurns?: number              // Nombre max de tours agentiques
  background?: boolean           // Toujours en arrière-plan
  isolation?: 'worktree' | 'remote'  // Mode d'isolation

  // Contexte
  memory?: 'user' | 'project' | 'local'  // Portée mémoire persistante
  skills?: string[]              // Skills à précharger
  mcpServers?: AgentMcpServerSpec[]      // Serveurs MCP spécifiques
  hooks?: HooksSettings          // Hooks de session
  requiredMcpServers?: string[]  // Serveurs MCP requis (patterns)
  omitClaudeMd?: boolean         // Exclure CLAUDE.md du contexte

  // Prompt
  initialPrompt?: string         // Prepended au premier tour
  getSystemPrompt: (params) => string   // Génération dynamique du prompt
  criticalSystemReminder_EXPERIMENTAL?: string  // Réinjecté à chaque tour
}
```

---

## 11. Fichiers clés

### Définitions

| Fichier | Contenu |
|---------|---------|
| `tools/AgentTool/AgentTool.tsx` | Outil Agent — schema, validation, logique de spawn |
| `tools/AgentTool/builtInAgents.ts` | Registre central des agents built-in |
| `tools/AgentTool/built-in/generalPurposeAgent.ts` | Agent general-purpose |
| `tools/AgentTool/built-in/exploreAgent.ts` | Agent Explore |
| `tools/AgentTool/built-in/planAgent.ts` | Agent Plan |
| `tools/AgentTool/built-in/claudeCodeGuideAgent.ts` | Agent claude-code-guide |
| `tools/AgentTool/built-in/verificationAgent.ts` | Agent verification |
| `tools/AgentTool/built-in/statuslineSetup.ts` | Agent statusline-setup |
| `tools/AgentTool/forkSubagent.ts` | Agent fork (implicite) |
| `tools/AgentTool/loadAgentsDir.ts` | Chargement des agents custom (`.claude/agents/`) |
| `tools/AgentTool/constants.ts` | Constantes (`ONE_SHOT_BUILTIN_AGENT_TYPES`, etc.) |
| `coordinator/coordinatorMode.ts` | Agent worker (mode coordinateur) |

### Exécution et lifecycle

| Fichier | Contenu |
|---------|---------|
| `utils/swarm/spawnInProcess.ts` | `spawnInProcessTeammate()` — spawn |
| `utils/swarm/backends/InProcessBackend.ts` | Backend in-process |
| `utils/swarm/inProcessRunner.ts` | `runInProcessTeammate()` — boucle d'exécution |
| `utils/teammateContext.ts` | `runWithTeammateContext()` — isolation AsyncLocalStorage |
| `utils/forkedAgent.ts` | `createSubagentContext()` — contexte hérité |
| `utils/agentId.ts` | `formatAgentId()`, `parseAgentId()` — identité |

### Communication

| Fichier | Contenu |
|---------|---------|
| `utils/teammateMailbox.ts` | Mailbox fichier (lecture/écriture/marquage) |
| `utils/swarm/teamHelpers.ts` | Gestion du team file, nettoyage |
| `utils/swarm/permissionSync.ts` | Requêtes de permission async |
| `utils/swarm/leaderPermissionBridge.ts` | Bridge UI pour permissions |

### Remote

| Fichier | Contenu |
|---------|---------|
| `remote/RemoteSessionManager.ts` | Gestion des sessions distantes CCR |
| `tasks/RemoteAgentTask/` | Tâche d'agent distant |
| `tasks/DreamTask/` | Tâche de consolidation mémoire |
| `tasks/InProcessTeammateTask/` | Tâche de teammate in-process |

---

## Récapitulatif des agents

| Agent | Outils | Modèle | Mode | Disponibilité |
|-------|--------|--------|------|---------------|
| `general-purpose` | Tous | inherit | default | Toujours |
| `Explore` | Glob, Grep, Read | haiku/inherit | read-only | `BUILTIN_EXPLORE_PLAN_AGENTS` |
| `Plan` | Glob, Grep, Read | inherit | read-only | `BUILTIN_EXPLORE_PLAN_AGENTS` |
| `claude-code-guide` | Read, Glob, Grep, WebFetch, WebSearch | haiku | dontAsk | Toujours (hors SDK) |
| `verification` | Tous sauf write | inherit | read-only | `VERIFICATION_AGENT` + gate |
| `statusline-setup` | Read, Edit | sonnet | default | Toujours |
| `fork` | Tous (parent) | inherit | bubble | `FORK_SUBAGENT` |
| `worker` | Variable | inherit | variable | `COORDINATOR_MODE` |
| **Custom** | Configurables | Configurable | Configurable | `.claude/agents/*.md` |
