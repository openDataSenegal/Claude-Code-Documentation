# Orchestration et Planification — Documentation complète

Les systèmes multi-agents et de planification de Claude Code : Coordinator mode, Plan mode, Swarm/Team, et UltraPlan.

---

## Table des matières

**Partie 1 — Coordinator Mode (Orchestrateur)**
1. [Activation et détection](#1-activation)
2. [System prompt du coordinateur](#2-system-prompt-coordinateur)
3. [Outils du coordinateur](#3-outils-coordinateur)
4. [Workers : spawn, outils et contraintes](#4-workers)
5. [Communication : task-notification XML](#5-task-notification)
6. [Workflow type : 4 phases](#6-workflow-coordinateur)
7. [Scratchpad : connaissance partagée](#7-scratchpad)
8. [Permissions en mode coordinateur](#8-permissions-coordinateur)

**Partie 2 — Plan Mode (Planification)**
9. [EnterPlanMode : entrer en mode plan](#9-enter-plan-mode)
10. [ExitPlanMode : sortie et approbation](#10-exit-plan-mode)
11. [Stockage des plans sur disque](#11-stockage-plans)
12. [Agent Plan (built-in)](#12-agent-plan)
13. [Permissions en mode plan](#13-permissions-plan)
14. [Plans et compaction](#14-plans-compaction)
15. [Plans et sessions (resume/fork)](#15-plans-sessions)
16. [UltraPlan : planification longue durée](#16-ultraplan)

**Partie 3 — Swarm / Team (Multi-agents)**
17. [Création d'équipe](#17-création-équipe)
18. [Identité des agents (name@team)](#18-identité-agents)
19. [Backends d'exécution](#19-backends)
20. [Boucle d'exécution d'un teammate](#20-boucle-exécution)
21. [Communication inter-agents (mailbox)](#21-mailbox)
22. [Messages structurés](#22-messages-structurés)
23. [Permissions et délégation](#23-permissions-swarm)
24. [Shutdown et nettoyage](#24-shutdown)

**Partie 4 — Référence**
25. [Diagramme d'architecture globale](#25-diagramme)
26. [Fichiers clés](#26-fichiers-clés)

---

# Partie 1 — Coordinator Mode

## 1. Activation

### Prérequis

- Feature flag : `COORDINATOR_MODE` (compile-time)
- Variable d'environnement : `CLAUDE_CODE_COORDINATOR_MODE=true`

### Détection

```typescript
function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

### Persistance de session

Quand une session est reprise (`--resume`), la fonction `matchSessionMode()` ajuste automatiquement la variable d'environnement pour correspondre au mode stocké dans la session. Cela empêche les incohérences entre le démarrage et la reprise.

---

## 2. System prompt du coordinateur

Le coordinateur reçoit un system prompt de ~370 lignes qui définit son rôle, ses outils, et le workflow. Voici les sections clés :

### Rôle

```
You are Claude Code, an AI assistant that orchestrates software engineering
tasks across multiple workers.

Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible — don't delegate trivially
```

### Règles fondamentales

- **Ne jamais fabriquer de résultats** — les résultats workers arrivent comme messages séparés
- **Ne jamais remercier les workers** — ce sont des signaux internes, pas des interlocuteurs
- **Synthétiser, toujours** — comprendre les résultats avant de diriger le travail suivant
- **Parallélisme maximal** — lancer les workers indépendants simultanément

### Format des résultats workers

```xml
<task-notification>
  <task-id>{agentId}</task-id>
  <status>completed|failed|killed</status>
  <summary>Agent "description" completed</summary>
  <result>Detailed findings...</result>
  <usage>
    <total_tokens>N</total_tokens>
    <tool_uses>N</tool_uses>
    <duration_ms>N</duration_ms>
  </usage>
</task-notification>
```

Les `<task-notification>` arrivent comme messages **user-role** mais ne sont PAS de l'utilisateur — ils sont identifiés par le tag d'ouverture `<task-notification>`.

---

## 3. Outils du coordinateur

Le coordinateur a un ensemble d'outils **restreint** — il ne code pas lui-même, il orchestre.

### Outils autorisés

| Outil | Usage |
|-------|-------|
| `Agent` | Spawner un nouveau worker |
| `SendMessage` | Continuer un worker existant (via son `to` agent ID) |
| `TaskStop` | Arrêter un worker en cours |
| `subscribe_pr_activity` | S'abonner aux événements PR GitHub (si MCP disponible) |
| `unsubscribe_pr_activity` | Se désabonner des événements PR |

### Filtrage

```typescript
function applyCoordinatorToolFilter(tools: Tools): Tools {
  return tools.filter(t =>
    COORDINATOR_MODE_ALLOWED_TOOLS.has(t.name) ||
    isPrActivitySubscriptionTool(t.name)
  )
}
```

Appliqué dans deux chemins :
- **REPL** : `useMergedTools.ts`
- **Headless** : `main.tsx`

---

## 4. Workers

### Spawn

Les workers sont créés via l'outil Agent avec `subagent_type: "worker"` :

```typescript
Agent({
  description: "Investigate auth bug",
  subagent_type: "worker",
  prompt: "Find null pointer exceptions in src/auth/..."
})
```

### Outils des workers

**Mode standard** (par défaut) :
- Tous les outils de `ASYNC_AGENT_ALLOWED_TOOLS` :
  Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch, NotebookEdit, Skill, ToolSearch, EnterWorktree, ExitWorktree, TodoWrite
- Plus : outils MCP des serveurs connectés
- Moins : outils internes (TeamCreate, TeamDelete, SendMessage, SyntheticOutput)

**Mode simple** (`CLAUDE_CODE_SIMPLE=true`) :
- Uniquement : Bash, Read, Edit

### Contexte injecté aux workers

Le coordinateur informe le modèle des outils disponibles pour les workers :

```
Workers have access to: Bash, Read, Edit, Glob, Grep, WebSearch, WebFetch, ...
Workers also have access to MCP tools from: slack, github
Scratchpad directory: /tmp/scratchpad (workers can read/write without prompts)
```

### Règles de prompts workers

Le system prompt du coordinateur contient des guidelines précises :

**Bon prompt worker** (synthétisé, spécifique) :
```
"Fix the null pointer in src/auth/validate.ts:42. The user field on
Session is undefined when sessions expire. Add a null check before
user.id access — if null, return 401 with 'Session expired'.
Commit and report the hash."
```

**Mauvais prompt worker** (vague, délégation paresseuse) :
```
"Based on your findings, fix the auth bug"
"Something went wrong with the tests, can you look?"
```

---

## 5. Communication : task-notification

### Flux

```
Coordinateur envoie Agent({ subagent_type: "worker", ... })
    ↓
Agent tool assemble le pool d'outils du worker
    ↓
Worker s'exécute (LocalAgentTask, async)
    ↓
Worker termine / échoue / est stoppé
    ↓
enqueueAgentNotification() construit le XML <task-notification>
    ↓
Notification enqueued (mode: 'task-notification', non-éditable)
    ↓
Query loop auto-dequeue la notification
    ↓
<task-notification> délivré comme message user au coordinateur
    ↓
Coordinateur parse <task-id>, <status>, <summary>, <result>
    ↓
Coordinateur synthétise et dirige le travail suivant
```

### Continuer un worker

```typescript
SendMessage({
  to: "agent-a1b",  // task-id du worker
  message: "Fix the null pointer at line 42..."
})
```

### Stopper un worker

```typescript
TaskStop({ task_id: "agent-x7q" })
```

Le worker stoppé peut être continué avec `SendMessage`.

---

## 6. Workflow type : 4 phases

| Phase | Qui | But |
|-------|-----|-----|
| **1. Research** | Workers (parallèles) | Explorer le codebase, trouver les fichiers, comprendre le problème |
| **2. Synthesis** | **Coordinateur** | Lire les résultats, comprendre, créer des specs d'implémentation |
| **3. Implementation** | Workers | Modifications ciblées selon la spec, commit |
| **4. Verification** | Workers | Prouver que les changements fonctionnent |

### Règles de concurrence

| Type de tâche | Concurrence |
|--------------|-------------|
| Read-only (recherche) | Parallèle librement |
| Write-heavy (implémentation) | Un worker à la fois par ensemble de fichiers |
| Verification | Peut coexister avec implémentation sur des zones différentes |

### Continue vs. Spawn

| Situation | Mécanisme | Raison |
|-----------|-----------|--------|
| Recherche a trouvé les fichiers exacts à éditer | **Continue** (SendMessage) | Worker a déjà le contexte |
| Recherche large, implémentation étroite | **Spawn** (Agent) | Contexte propre, pas de bruit |
| Correction d'un échec | **Continue** | Worker a le contexte d'erreur |
| Vérification de code écrit par un autre worker | **Spawn** | Yeux frais, pas de biais |
| Approche précédente totalement mauvaise | **Spawn** | Évite l'ancrage sur l'échec |
| Tâche complètement différente | **Spawn** | Aucun contexte réutilisable |

---

## 7. Scratchpad

### Activation

Gate : `tengu_scratch` (GrowthBook)

### Fonctionnement

Quand activé, un répertoire scratchpad est partagé entre tous les workers :

```
Scratchpad directory: /tmp/claude-{uid}/{cwd}/scratchpad/
Workers can read and write here without permission prompts.
```

- Accès **automatique** en lecture/écriture (pas de prompt de permission)
- Structure libre ("however fits the work")
- Persiste pendant toute la session coordinateur
- Permet le partage de connaissance durable entre workers

---

## 8. Permissions en mode coordinateur

### Handler coordinateur

```
1. Essaie les hooks PermissionRequest (rapide, local)
2. Essaie le classifieur Bash (lent, inférence — Bash uniquement)
3. Fall through vers le dialogue interactif si les deux échouent
```

L'exécution est **séquentielle** (pas de course hooks/classifieur comme en interactif).

### Permissions des workers

- Chaque worker reçoit son propre `permissionContext`
- Mode par défaut : `acceptEdits`
- Les permissions bubblent vers le leader pour les outils non autorisés

---

# Partie 2 — Plan Mode

## 9. EnterPlanMode

**Fichier** : `tools/EnterPlanModeTool/EnterPlanModeTool.ts`

### Caractéristiques

| Propriété | Valeur |
|-----------|--------|
| Nom | `EnterPlanMode` |
| Paramètres | Aucun |
| Read-only | Oui |
| Concurrent-safe | Oui |
| Deferred | Oui (annoncé au démarrage) |

### Flux d'exécution

```
1. Validation : rejet si en contexte agent
2. handlePlanModeTransition(fromMode, 'plan')
3. prepareContextForPlanMode(context) :
   ├── Sauvegarde prePlanMode (mode avant plan)
   ├── Si auto-mode + shouldPlanUseAutoMode() → garde auto
   ├── Si bypass → sauvegarde, entre en plan
   ├── Sinon → passe en mode plan, strip perms dangereuses
4. Retourne confirmation + instructions workflow
```

### Instructions après entrée

Le modèle reçoit un workflow en 5 phases :
1. **Compréhension** — explorer avec des agents Explore
2. **Design** — utiliser des agents Plan
3. **Revue** — lire les fichiers critiques, poser des questions
4. **Plan final** — rédiger le plan avec structure
5. **Sortie** — appeler ExitPlanMode

**Règle stricte** : "DO NOT write or edit any files except the plan file"

---

## 10. ExitPlanMode

**Fichier** : `tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`

### Paramètres

```typescript
{
  allowedPrompts?: Array<{
    tool: string,     // "Bash"
    prompt: string,   // "run tests"
  }>
}
```

### Deux chemins d'exécution

#### Chemin A — Teammate avec plan_mode_required

```
1. Valider que le fichier plan existe
2. Construire plan_approval_request :
   { type, from, timestamp, planFilePath, planContent, requestId }
3. Écrire dans la mailbox du team lead
4. Mettre à jour le task state → "awaiting approval"
5. Retourner { awaitingLeaderApproval: true }
6. Attendre la réponse dans la mailbox
```

#### Chemin B — Utilisateur local

```
1. Lire le plan depuis le disque
2. Syncer les éditions si l'utilisateur a modifié (Ctrl+G)
3. Restaurer le mode de permission :
   ├── Restaurer prePlanMode
   ├── Si auto-mode gate off → fallback 'default'
   ├── Restaurer les permissions dangereuses strippées
4. Retourner le contenu du plan approuvé
```

### Restauration du mode

```typescript
let restoreMode = prePlanMode ?? 'default'

// Si le gate auto-mode n'est plus disponible
if (restoreMode === 'auto' && !isAutoModeGateEnabled()) {
  restoreMode = 'default'
}

// Restaurer les permissions dangereuses si on ne reste pas en auto
if (!restoringToAuto && strippedDangerousRules) {
  restoreDangerousPermissions()
}
```

---

## 11. Stockage des plans

### Emplacement

```
~/.claude/plans/{slug}.md                    # Plan principal
~/.claude/plans/{slug}-agent-{agentId}.md    # Plan de sous-agent
```

Le slug est généré via `generateWordSlug()` (mots aléatoires), caché par session dans `STATE.planSlugCache`.

### Personnalisation

Via `settings.json` → `plansDirectory` (relatif à la racine du projet). Validation : le chemin doit rester dans la racine du projet.

### Opérations

| Fonction | Description |
|----------|-------------|
| `getPlanFilePath(agentId?)` | Chemin du fichier plan |
| `getPlan(agentId?)` | Lire le plan depuis le disque |
| `clearPlanSlug()` | Reset le slug de la session |
| `generateWordSlug()` | Nouveau slug (retry ×10 si collision) |

---

## 12. Agent Plan (built-in)

**Fichier** : `tools/AgentTool/built-in/planAgent.ts`

### Définition

```typescript
PLAN_AGENT = {
  agentType: 'Plan',
  whenToUse: 'Software architect agent for designing implementation plans',
  disallowedTools: [Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit],
  tools: [/* mêmes que Explore : Glob, Grep, Read, Bash read-only */],
  model: 'inherit',
  omitClaudeMd: true,
  oneShot: true,
}
```

### System prompt

- **Mode** : LECTURE SEULE stricte
- **Outils** : find/grep/cat via Bash, Glob, Grep, Read
- **Processus** : Comprendre → Explorer → Designer → Détailler
- **Output requis** : Section "Critical Files for Implementation" (3-5 fichiers)

---

## 13. Permissions en mode plan

### Hiérarchie des modes

```
bypassPermissions (plus permissif)
    ↓
acceptEdits
    ↓
default
    ↓
plan (lecture seule focus)     ← Mode plan
    ↓
auto (classifieur ML)
    ↓
dontAsk (plus restrictif)
```

### Entrée en mode plan

```
prepareContextForPlanMode(context) :
  1. Sauvegarde le mode courant dans prePlanMode
  2. Si auto-mode actif et shouldPlanUseAutoMode() :
     → Garde auto-mode, sauvegarde prePlanMode='auto'
  3. Si PAS en bypass et shouldPlanUseAutoMode() :
     → Active auto-mode, strip les permissions dangereuses
  4. Sinon : simple passage en mode plan
```

### Outils en mode plan

| Catégorie | Disponibilité |
|-----------|--------------|
| Read-only (Read, Glob, Grep, Bash read) | Disponibles |
| EnterPlanMode | Désactivé (déjà en plan) |
| ExitPlanMode | Disponible |
| Write tools (Edit, Write, Bash write) | Bloqués |
| **Exception** : écriture du fichier plan | Autorisée |

---

## 14. Plans et compaction

### Attachments créés lors de la compaction

**Plan file reference** :
```typescript
{
  type: 'plan_file_reference',
  planFilePath: string,
  planContent: string    // Contenu complet du plan
}
```

**Plan mode attachment** (si en mode plan pendant la compaction) :
```typescript
{
  type: 'plan_mode',
  reminderType: 'full' | 'sparse',
  planFilePath: string,
  planExists: boolean
}
```

### Restauration post-compaction

1. Le fichier plan est relu depuis le disque
2. Le contenu est injecté comme attachment
3. Si en mode plan → instructions fraîches réinjectées
4. Si plan existant + nouvelle session → instructions de ré-entrée

---

## 15. Plans et sessions

### Resume

```
copyPlanForResume(log, targetSessionId) :
  1. Extraire le slug depuis l'historique des messages
  2. Tenter de lire le fichier plan sur disque
  3. Si fichier absent → recovery :
     ├── Snapshot incrémental dans le transcript
     ├── tool_use input de ExitPlanMode
     ├── planContent dans les messages user
     └── Attachment plan_file_reference
  4. Réécrire le plan sur disque
```

### Fork

```
copyPlanForFork(log, targetSessionId) :
  1. Lire le plan original depuis le disque
  2. Générer un NOUVEAU slug (évite l'écrasement)
  3. Copier le contenu vers le nouveau chemin
```

---

## 16. UltraPlan

### Fonctionnement

UltraPlan délègue la planification complexe à une **session CCR distante sur Opus 4.6** avec jusqu'à 30 minutes de réflexion.

```
Terminal local
    │
    ├── Envoie la tâche à CCR distant
    ├── Polling toutes les 3 secondes
    │
    └── CCR distant (Opus 4.6, jusqu'à 30 min)
         ├── Explore le codebase
         ├── Conçoit le plan
         └── Retourne via __ULTRAPLAN_TELEPORT_LOCAL__
              │
              ▼
         Plan approuvé/rejeté via UI navigateur
```

### Feature flag : `ULTRAPLAN`

---

# Partie 3 — Swarm / Team

## 17. Création d'équipe

### TeamCreateTool

```typescript
// Crée une équipe avec un nom unique
TeamCreate({ team_name: "auth-fix", description: "Fix auth module" })
```

**Flux** :
```
1. Générer un nom unique déterministe
2. Créer ~/.claude/teams/{team-name}/config.json
3. Créer le répertoire de tâches
4. Enregistrer pour le nettoyage de session
5. Définir le team context dans AppState :
   { teamName, teamFilePath, leadAgentId: "team-lead@{teamName}" }
6. Retourner { team_name, team_file_path, lead_agent_id }
```

### Team File (config.json)

```typescript
{
  name: string,
  description?: string,
  createdAt: number,
  leadAgentId: string,            // "team-lead@auth-fix"
  leadSessionId: string,
  members: [{
    agentId: string,              // "researcher@auth-fix"
    name: string,
    model?: string,
    color?: string,
    planModeRequired?: boolean,
    cwd: string,
    worktreePath?: string,
    isActive?: boolean,
    mode?: PermissionMode,
  }],
  teamAllowedPaths?: [{
    path: string,
    toolName: string,
    addedBy: string,
  }],
}
```

---

## 18. Identité des agents

### Format : `name@team`

```typescript
formatAgentId("researcher", "auth-fix")  → "researcher@auth-fix"
parseAgentId("researcher@auth-fix")      → { agentName: "researcher", teamName: "auth-fix" }
```

- **Déterministe** : même nom + même équipe = même ID (toujours)
- **Lisible** : visible dans les logs, transcripts, fichiers d'équipe
- **Prédictible** : le leader peut calculer les IDs sans lookup

### IDs de requêtes

```typescript
generateRequestId("shutdown", "researcher@auth-fix")
→ "shutdown-1702500000000@researcher@auth-fix"
```

---

## 19. Backends d'exécution

| Backend | Mécanisme | Isolation | Disponibilité |
|---------|-----------|-----------|---------------|
| **in-process** | AsyncLocalStorage dans le même processus | Contexte isolé | Toujours |
| **tmux** | Panes tmux via socket | Processus séparé | Si tmux installé |
| **iterm2** | Split panes iTerm2 natif | Processus séparé | macOS + iTerm2 |

### Interface TeammateExecutor

```typescript
interface TeammateExecutor {
  type: BackendType
  isAvailable(): Promise<boolean>
  spawn(config): Promise<TeammateSpawnResult>
  sendMessage(agentId, message): Promise<void>
  terminate(agentId, reason?): Promise<boolean>
  kill(agentId): Promise<boolean>
  isActive(agentId): Promise<boolean>
}
```

### InProcessBackend

Le plus utilisé — exécute les teammates dans le même processus Node.js avec isolation AsyncLocalStorage :

```typescript
TeammateContext {
  agentId: string,              // "researcher@auth-fix"
  agentName: string,
  teamName: string,
  color?: string,
  planModeRequired: boolean,
  parentSessionId: string,
  isInProcess: true,
  abortController: AbortController,
}
```

---

## 20. Boucle d'exécution d'un teammate

**Fichier** : `utils/swarm/inProcessRunner.ts`

```
while (!abort && !shouldExit) {
    │
    ├── 1. Exécuter le prompt initial (ou message reçu)
    │      → runAgent() avec contexte AsyncLocalStorage isolé
    │      → createInProcessCanUseTool() pour les permissions
    │
    ├── 2. Mettre à jour la progression (tools, tokens, spinner)
    │
    ├── 3. Marquer le task comme idle (PAS completed)
    │
    ├── 4. Envoyer idle_notification au leader via mailbox
    │
    ├── 5. waitForNextPromptOrShutdown() :
    │      ├── Poll mailbox toutes les 500ms
    │      │   ├── Shutdown requests (priorité haute)
    │      │   ├── Messages du leader
    │      │   └── Messages des peers
    │      ├── Poll task list (tâches non assignées)
    │      └── Listen abort signal
    │
    ├── 6. Traiter le message reçu :
    │      ├── Shutdown → passer au modèle pour décision
    │      ├── Nouveau message → wrapper en XML si d'un teammate
    │      └── Assignation de tâche → formater comme prompt
    │
    └── 7. Loop ou exit
```

### Niveaux d'AbortController

| Controller | Portée | Déclencheur |
|-----------|--------|-------------|
| `task.abortController` | Tue tout le teammate | Kill, shutdown approuvé |
| `task.currentWorkAbortController` | Abort le tour courant | Touche Escape |

---

## 21. Communication inter-agents (mailbox)

### Structure fichier

```
~/.claude/teams/{team-name}/inboxes/{agent-name}.json
```

Format : tableau JSON de `TeammateMessage` :

```typescript
TeammateMessage {
  from: string,        // Nom de l'expéditeur
  text: string,        // Contenu (texte ou JSON structuré)
  timestamp: string,   // ISO 8601
  read: boolean,
  color?: string,      // Couleur UI
  summary?: string,    // Aperçu 5-10 mots
}
```

### Opérations (concurrent-safe via file locking)

| Fonction | Description |
|----------|-------------|
| `writeToMailbox(recipient, message, team)` | Écriture atomique (lock → read → modify → write → unlock) |
| `readMailbox(agent, team)` | Lecture complète |
| `readUnreadMessages(agent, team)` | Filtrage non-lus |
| `markMessageAsReadByIndex(agent, team, index)` | Marquage individuel |
| `clearMailbox(agent, team)` | Vider la boîte |

### Polling dans la boucle

```
1. Vérifier les messages in-memory pending (UI transcript)
2. Poll le fichier mailbox (500ms)
3. Scanner les shutdown requests (priorité maximale)
4. Prioriser les messages du team-lead vs peer-to-peer
5. Fallback FIFO pour les messages peers
6. Vérifier la task list pour les tâches non assignées
7. Marquer comme lu pour éviter le retraitement
```

---

## 22. Messages structurés

### Shutdown request (leader → teammate)

```json
{
  "type": "shutdown_request",
  "request_id": "shutdown-{timestamp}@{agentId}",
  "from": "team-lead",
  "reason": "Task completed, shutting down team"
}
```

### Idle notification (teammate → leader)

```json
{
  "type": "idle_notification",
  "from": "researcher",
  "timestamp": "2026-04-01T10:30:00Z",
  "idleReason": "available",
  "summary": "Finished investigating auth module",
  "completedTaskId": "task-123",
  "completedStatus": "resolved"
}
```

### Permission request (teammate → leader)

```json
{
  "type": "permission_request",
  "request_id": "permission-{timestamp}@{agentId}",
  "agent_id": "researcher@auth-fix",
  "tool_name": "Bash",
  "tool_use_id": "tooluse-xyz",
  "description": "Execute npm test",
  "input": { "command": "npm test" }
}
```

### Permission response (leader → teammate)

```json
{
  "type": "permission_response",
  "request_id": "permission-{timestamp}@{agentId}",
  "subtype": "success",
  "response": {
    "updated_input": { "command": "npm test -- --coverage" },
    "permission_updates": [
      { "type": "addRules", "rules": [{"toolName": "Bash", "ruleContent": "npm:*"}], "behavior": "allow" }
    ]
  }
}
```

### Plan approval (teammate → leader → teammate)

```json
{
  "type": "plan_approval_request",
  "from": "implementer",
  "planFilePath": "/path/to/plan.md",
  "planContent": "...full plan...",
  "requestId": "plan_approval__implementer@team"
}
```

---

## 23. Permissions et délégation

### Deux chemins de résolution

**Chemin A — UI Queue du leader (préféré)** :

```
1. Teammate rencontre un 'ask' de permission
2. registerLeaderToolUseConfirmQueue() → ajoute à la queue UI du leader
3. Affiche dans l'UI du leader avec badge worker (nom + couleur)
4. Leader approuve/refuse via l'interface standard
5. Callback onAllow/onReject déclenché
6. Teammate reprend avec la décision
```

**Chemin B — Mailbox (fallback)** :

```
1. Teammate crée permission_request JSON
2. writeToMailbox(leader, request)
3. Teammate poll sa propre mailbox (500ms)
4. Leader détecte la requête, prompt l'utilisateur
5. Réponse écrite dans la mailbox du teammate
6. Teammate traite la réponse, reprend
```

### Write-back des permissions

Quand un teammate reçoit un 'allow', les mises à jour de permissions sont persistées via `getLeaderSetToolPermissionContext()` avec `preserveMode: true` (empêche les transformations du worker de fuiter vers le coordinateur).

---

## 24. Shutdown et nettoyage

### TeamDeleteTool

```
1. Vérifier qu'aucun membre n'est actif (refuser si oui)
2. cleanupTeamDirectories(teamName) :
   ├── Lire le team file pour les worktree paths
   ├── git worktree remove --force (chaque worktree)
   ├── Fallback rm -rf si git échoue
   ├── Supprimer ~/.claude/teams/{team-name}/
   └── Supprimer ~/.claude/tasks/{team-name}/
3. Désenregistrer du tracking de session
4. Libérer les couleurs des teammates
5. Clear AppState.teamContext
```

### Nettoyage de fin de session

```typescript
cleanupSessionTeams() :
  Pour chaque équipe créée cette session :
    ├── Kill les panes tmux orphelins
    ├── Destroy les worktrees git
    ├── Supprimer les répertoires d'équipe
    └── Supprimer les répertoires de tâches
```

### Kill d'un teammate in-process

```
killInProcessTeammate(taskId) :
  1. Abort le AbortController
  2. Appeler le cleanup handler
  3. Appeler les idle callbacks
  4. Retirer de AppState.teamContext.teammates
  5. Retirer du team file
  6. Mettre à jour le task status → 'killed'
  7. Émettre l'événement SDK task_terminated
  8. Planifier evictTerminalTask (après délai UI)
```

---

# Partie 4 — Référence

## 25. Diagramme d'architecture globale

```
┌─────────────────────────────────────────────────────────────────────┐
│                        UTILISATEUR                                   │
│                            │                                         │
│                    ┌───────┴───────┐                                 │
│                    │  Claude Code  │                                  │
│                    │   (Leader)    │                                  │
│                    └───────┬───────┘                                 │
│                            │                                         │
│              ┌─────────────┼─────────────┐                          │
│              │             │             │                           │
│     ┌────────┴──┐   ┌─────┴─────┐  ┌────┴────────┐                │
│     │COORDINATOR│   │ PLAN MODE │  │   SWARM     │                 │
│     │   MODE    │   │           │  │   /TEAM     │                 │
│     └────┬──────┘   └─────┬─────┘  └────┬────────┘                │
│          │                │              │                          │
│    ┌─────┼─────┐     ┌────┤         ┌────┼────┐                    │
│    │     │     │     │    │         │    │    │                     │
│  Worker Worker Worker Plan Agent  Team  Team  Team                 │
│    A     B     C    (read-only)  mate  mate  mate                  │
│    │     │     │        │        A     B     C                     │
│    ▼     ▼     ▼        ▼        │     │     │                     │
│  <task-   <task-  <task-  Plan    ▼     ▼     ▼                    │
│  notif>   notif>  notif>  File  mailbox mailbox mailbox            │
│                                                                     │
│  Communication:     Communication:     Communication:               │
│  XML task-notif     Plan file +        File-based                   │
│  via message queue  approval flow      mailbox (JSON)               │
│                                                                     │
│  Outils coord:      Outils plan:       Outils team:                │
│  Agent, SendMsg,    Read, Glob, Grep   Tous (selon config)         │
│  TaskStop           Bash (read-only)   + permissions bubblées       │
│                     ExitPlanMode                                    │
│                                                                     │
│  Permissions:        Permissions:       Permissions:                 │
│  Sequential          Plan mode          Leader UI queue             │
│  (hooks→classifier   (read-only +       ou mailbox                  │
│   →dialog)           plan file write)   fallback                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 26. Fichiers clés

### Coordinator Mode

| Fichier | Rôle |
|---------|------|
| `coordinator/coordinatorMode.ts` | System prompt (~370 lignes), activation, contexte workers, scratchpad |
| `constants/tools.ts` | COORDINATOR_MODE_ALLOWED_TOOLS, ASYNC_AGENT_ALLOWED_TOOLS, INTERNAL_WORKER_TOOLS |
| `utils/toolPool.ts` | applyCoordinatorToolFilter(), mergeAndFilterTools() |
| `hooks/toolPermission/handlers/coordinatorHandler.ts` | Pipeline permissions coordinateur |
| `tasks/LocalAgentTask/LocalAgentTask.tsx` | enqueueAgentNotification(), XML task-notification |

### Plan Mode

| Fichier | Rôle |
|---------|------|
| `tools/EnterPlanModeTool/EnterPlanModeTool.ts` | Entrée en mode plan |
| `tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts` | Sortie + flux d'approbation |
| `utils/plans.ts` | Stockage fichier, slug, recovery |
| `utils/planModeV2.ts` | Configuration (interview, pewter ledger variants) |
| `tools/AgentTool/built-in/planAgent.ts` | Agent Plan (read-only) |
| `utils/permissions/permissionSetup.ts` | prepareContextForPlanMode(), restore |
| `services/compact/compact.ts` | Attachments plan pour compaction |

### Swarm / Team

| Fichier | Rôle |
|---------|------|
| `tools/TeamCreateTool/TeamCreateTool.ts` | Création d'équipe |
| `tools/TeamDeleteTool/TeamDeleteTool.ts` | Suppression + nettoyage |
| `tools/SendMessageTool/SendMessageTool.ts` | Communication inter-agents |
| `utils/swarm/inProcessRunner.ts` | Boucle d'exécution teammate |
| `utils/swarm/spawnInProcess.ts` | Spawn in-process |
| `utils/swarm/backends/InProcessBackend.ts` | Backend in-process |
| `utils/swarm/teamHelpers.ts` | Team file management, cleanup |
| `utils/teammateMailbox.ts` | Mailbox fichier (concurrent-safe) |
| `utils/swarm/permissionSync.ts` | Synchronisation permissions |
| `utils/swarm/leaderPermissionBridge.ts` | Bridge UI leader |
| `utils/teammateContext.ts` | AsyncLocalStorage isolation |
| `utils/agentId.ts` | Format name@team, parsing |
