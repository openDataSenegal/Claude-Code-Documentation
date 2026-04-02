# Outils de Claude Code — Documentation complète

Ce document recense l'ensemble des outils (tools) disponibles dans Claude Code, leur fonctionnement, leurs paramètres et le système d'exécution sous-jacent.

---

## Table des matières

1. [Architecture du système d'outils](#1-architecture-du-système-doutils)
2. [Outils core — Toujours disponibles](#2-outils-core)
3. [Outils de gestion de tâches (v2)](#3-outils-de-gestion-de-tâches)
4. [Outils Worktree](#4-outils-worktree)
5. [Outils MCP](#5-outils-mcp)
6. [Outils Team / Multi-agents](#6-outils-team--multi-agents)
7. [Outils Cron / Scheduling](#7-outils-cron--scheduling)
8. [Outils Kairos (assistant persistant)](#8-outils-kairos)
9. [Outils développeur / Ant-only](#9-outils-développeur--ant-only)
10. [Outils feature-gated divers](#10-outils-feature-gated-divers)
11. [Outils internes / Testing](#11-outils-internes--testing)
12. [Pipeline d'exécution](#12-pipeline-dexécution)
13. [Modèle de concurrence](#13-modèle-de-concurrence)
14. [Système de permissions](#14-système-de-permissions)
15. [Hooks pre/post outil](#15-hooks-prepost-outil)

---

## 1. Architecture du système d'outils

### Fichiers clés

| Fichier | Rôle |
|---------|------|
| `Tool.ts` | Interface `Tool<Input, Output>`, factory `buildTool()`, constantes par défaut |
| `tools.ts` | Registre central — `getAllBaseTools()`, `getTools()`, `assembleToolPool()` |
| `services/tools/toolExecution.ts` | Pipeline d'exécution — `runToolUse()` |
| `services/tools/StreamingToolExecutor.ts` | File d'attente concurrente/exclusive |
| `services/tools/toolHooks.ts` | Hooks pre/post exécution |

### Contrat d'un outil

Chaque outil est créé via `buildTool()` et implémente :

```typescript
interface Tool<Input, Output> {
  name: string                           // Identifiant unique
  aliases?: string[]                     // Noms dépréciés (ex: "KillShell" → "TaskStop")
  description(): Promise<string>         // Description pour le modèle
  prompt(): Promise<string>              // Instructions détaillées / schema
  inputSchema: ZodSchema                 // Validation des entrées (Zod)

  call(input, context, canUseTool, assistantMessage, onProgress): Promise<ToolResult>

  isEnabled(): boolean                   // Disponibilité runtime
  isConcurrencySafe(input): boolean      // true = exécution parallèle possible
  isReadOnly(input): boolean             // true = lecture seule
  isDestructive(input): boolean          // true = irréversible
  checkPermissions(input, context): Promise<PermissionResult>
  validateInput(input, context): Promise<ValidationResult>

  userFacingName(input): string          // Nom affiché dans l'UI
  getActivityDescription(input): string  // Texte du spinner
  shouldDefer: boolean                   // Masqué du listing initial (chargement à la demande)
  maxResultSizeChars: number             // Seuil de persistance du résultat
  interruptBehavior(): 'cancel' | 'block'  // Comportement si l'utilisateur envoie un message
}
```

### Modes de chargement

| Mode | Outils |
|------|--------|
| **Import direct** | Agent, Bash, Read, Edit, Write, WebFetch, WebSearch, Skill, etc. |
| **Import conditionnel** | Config (Ant-only), LSP (env), Glob/Grep (si pas embedded) |
| **`require()` lazy** | Tous les feature-gated (Sleep, Cron, WebBrowser, Monitor, etc.) |
| **Getter lazy** | TeamCreate, TeamDelete, SendMessage (évite les imports circulaires) |

---

## 2. Outils core

### 2.1 Bash

| | |
|---|---|
| **Nom** | `Bash` |
| **Fichier** | `tools/BashTool/BashTool.tsx` |
| **Read-only** | Dynamique (analyse la commande) |
| **Concurrent** | Non |

**Paramètres**

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `command` | string | oui | Commande shell à exécuter |
| `timeout` | number | non | Timeout en ms (max 600 000) |
| `cwd` | string | non | Répertoire de travail |

**Comportement** : Exécute via shell, auto-background après seuil de timeout, détecte les opérations destructrices, valide le mode sandbox, trace les opérations git.

---

### 2.2 Read

| | |
|---|---|
| **Nom** | `Read` |
| **Fichier** | `tools/FileReadTool/FileReadTool.ts` |
| **Read-only** | Oui |
| **Concurrent** | Oui |

**Paramètres**

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `file_path` | string | oui | Chemin absolu du fichier |
| `limit` | number | non | Nombre de lignes à lire |
| `offset` | number | non | Ligne de départ |
| `pages` | string | non | Pages PDF (ex: "1-5") |

**Comportement** : Supporte texte, images (resize auto), PDF (extraction de pages), notebooks Jupyter. Détecte l'encodage, bloque les fichiers device.

---

### 2.3 Edit

| | |
|---|---|
| **Nom** | `Edit` |
| **Fichier** | `tools/FileEditTool/FileEditTool.ts` |
| **Read-only** | Non |
| **Concurrent** | Non |

**Paramètres**

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `file_path` | string | oui | Chemin absolu du fichier |
| `old_string` | string | oui | Texte à remplacer |
| `new_string` | string | oui | Texte de remplacement |
| `replace_all` | boolean | non | Remplacer toutes les occurrences (défaut: false) |

**Comportement** : Remplacement exact de chaînes. Échoue si `old_string` n'est pas unique (sauf `replace_all`). Trace les diffs git, valide les fichiers settings, limite à 1 GiB.

---

### 2.4 Write

| | |
|---|---|
| **Nom** | `Write` |
| **Fichier** | `tools/FileWriteTool/FileWriteTool.ts` |
| **Read-only** | Non |
| **Concurrent** | Non |

**Paramètres**

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `file_path` | string | oui | Chemin absolu du fichier |
| `content` | string | oui | Contenu complet à écrire |

**Comportement** : Crée ou écrase le fichier. Retourne un diff patch.

---

### 2.5 Glob

| | |
|---|---|
| **Nom** | `Glob` |
| **Fichier** | `tools/GlobTool/GlobTool.ts` |
| **Read-only** | Oui |
| **Concurrent** | Oui |

**Paramètres**

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `pattern` | string | oui | Pattern glob (ex: `**/*.ts`) |
| `path` | string | non | Répertoire de recherche (défaut: cwd) |

**Comportement** : Limite à 100 résultats, triés par date de modification. Chemins relatifs.

---

### 2.6 Grep

| | |
|---|---|
| **Nom** | `Grep` |
| **Fichier** | `tools/GrepTool/GrepTool.ts` |
| **Read-only** | Oui |
| **Concurrent** | Oui |

**Paramètres**

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `pattern` | string | oui | Regex ripgrep |
| `path` | string | non | Fichier ou répertoire |
| `glob` | string | non | Filtre glob (`*.js`) |
| `type` | string | non | Type de fichier (`js`, `py`, etc.) |
| `output_mode` | enum | non | `content`, `files_with_matches` (défaut), `count` |
| `-A` | number | non | Lignes après le match |
| `-B` | number | non | Lignes avant le match |
| `-C` | number | non | Lignes de contexte |
| `-n` | boolean | non | Numéros de ligne (défaut: true) |
| `-i` | boolean | non | Insensible à la casse |
| `multiline` | boolean | non | Mode multiligne |
| `head_limit` | number | non | Limite de résultats (défaut: 250) |
| `offset` | number | non | Passer N premiers résultats |

**Comportement** : Basé sur ripgrep. Parsing sémantique des nombres/booléens.

---

### 2.7 Agent

| | |
|---|---|
| **Nom** | `Agent` (alias legacy: `Task`) |
| **Fichier** | `tools/AgentTool/AgentTool.tsx` |
| **Read-only** | Non |
| **Concurrent** | Oui |

**Paramètres**

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `description` | string | oui | Description courte (3-5 mots) |
| `prompt` | string | oui | Tâche à accomplir |
| `subagent_type` | string | non | Type d'agent spécialisé |
| `model` | string | non | Override du modèle (`sonnet`, `opus`, `haiku`) |
| `run_in_background` | boolean | non | Exécution en arrière-plan |
| `name` | string | non | Nom de l'agent |
| `team_name` | string | non | Nom de l'équipe |
| `mode` | string | non | Mode de permission |
| `isolation` | enum | non | `worktree` pour isolation git |
| `cwd` | string | non | Répertoire de travail |

**Comportement** : Crée des sous-agents isolés. Supporte worktree, background, équipes, override de modèle et permissions.

---

### 2.8 WebFetch

| | |
|---|---|
| **Nom** | `WebFetch` |
| **Fichier** | `tools/WebFetchTool/WebFetchTool.ts` |
| **Read-only** | Oui |
| **Concurrent** | Oui |

**Paramètres**

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `url` | string | oui | URL à récupérer |
| `prompt` | string | oui | Prompt appliqué au contenu |

**Comportement** : Convertit HTML → markdown, applique le prompt via modèle rapide, cache 15 min. Liste de domaines pré-approuvés.

---

### 2.9 WebSearch

| | |
|---|---|
| **Nom** | `WebSearch` |
| **Fichier** | `tools/WebSearchTool/WebSearchTool.ts` |
| **Read-only** | Oui |
| **Concurrent** | Oui |

**Paramètres**

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `query` | string | oui | Requête de recherche (min 2 chars) |
| `allowed_domains` | string[] | non | Domaines autorisés |
| `blocked_domains` | string[] | non | Domaines bloqués |

**Comportement** : Recherche web native Claude. Retourne titres et URLs.

---

### 2.10 NotebookEdit

| | |
|---|---|
| **Nom** | `NotebookEdit` |
| **Fichier** | `tools/NotebookEditTool/NotebookEditTool.ts` |
| **Read-only** | Non |
| **Concurrent** | Non |

**Paramètres**

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `notebook_path` | string | oui | Chemin du notebook `.ipynb` |
| `cell_id` | string | non | ID de la cellule cible |
| `new_source` | string | oui | Nouveau contenu de la cellule |
| `cell_type` | enum | non | `code` ou `markdown` |
| `edit_mode` | enum | non | `replace`, `insert`, `delete` |

**Comportement** : Édition de cellules Jupyter. Préserve la structure du notebook, supporte insertion/suppression/remplacement.

---

### 2.11 Skill

| | |
|---|---|
| **Nom** | `Skill` |
| **Fichier** | `tools/SkillTool/SkillTool.ts` |
| **Read-only** | Non |
| **Concurrent** | Non |

**Paramètres**

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `skill` | string | oui | Nom du skill |
| `args` | string | non | Arguments du skill |

**Comportement** : Charge depuis `~/.claude/skills/`, `.claude/skills/`, plugins ou skills bundled. Supporte override de modèle et exécution agent.

---

### 2.12 AskUserQuestion

| | |
|---|---|
| **Nom** | `AskUserQuestion` |
| **Fichier** | `tools/AskUserQuestionTool/AskUserQuestionTool.tsx` |
| **Read-only** | Oui |
| **Concurrent** | Non |

**Paramètres**

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `questions` | array | oui | 1-4 questions, chacune avec `question`, `header`, `options` (2-4) |
| `multiSelect` | boolean | non | Sélection multiple |
| `preview` | object | non | Aperçu optionnel |

**Comportement** : Affiche des questions à choix multiples dans l'UI. Capture les annotations utilisateur.

---

### 2.13 EnterPlanMode / ExitPlanMode

| | |
|---|---|
| **Noms** | `EnterPlanMode`, `ExitPlanMode` |
| **Fichiers** | `tools/EnterPlanModeTool/`, `tools/ExitPlanModeTool/` |
| **Read-only** | Oui (enter) / Non (exit) |

**EnterPlanMode** : Aucun paramètre. Transition vers le mode plan (lecture seule).

**ExitPlanMode** : Paramètres optionnels pour les demandes de permissions. Persiste le plan sur disque.

---

### 2.14 ToolSearch

| | |
|---|---|
| **Nom** | `ToolSearch` |
| **Fichier** | `tools/ToolSearchTool/ToolSearchTool.ts` |
| **Read-only** | Oui |
| **Concurrent** | Oui |

**Paramètres**

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `query` | string | oui | `"select:Read,Edit"` ou mots-clés |
| `max_results` | number | non | Maximum de résultats (défaut: 5) |

**Comportement** : Recherche parmi les outils différés (deferred). Mode sélection directe ou recherche par mots-clés. Gate: `isToolSearchEnabledOptimistic()`.

---

### 2.15 SendUserMessage / Brief

| | |
|---|---|
| **Nom** | `SendUserMessage` (alias: `Brief`) |
| **Fichier** | `tools/BriefTool/BriefTool.ts` |
| **Read-only** | Oui |
| **Concurrent** | Non |

**Paramètres**

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `message` | string | oui | Message markdown |
| `attachments` | string[] | non | Chemins de fichiers joints |
| `status` | enum | non | `normal` ou `proactive` |

**Comportement** : Envoie un message structuré à l'utilisateur. Résout les pièces jointes, détecte les images, inclut les timestamps ISO.

---

### 2.16 TaskOutput

| | |
|---|---|
| **Nom** | `TaskOutput` |
| **Fichier** | `tools/TaskOutputTool/TaskOutputTool.tsx` |
| **Read-only** | Oui |
| **Concurrent** | Oui |

**Paramètres**

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `task_id` | string | oui | ID de la tâche |
| `block` | boolean | non | Bloquant (défaut: true) |
| `timeout` | number | non | Timeout en secondes (défaut: 30, max: 600) |

---

### 2.17 TaskStop

| | |
|---|---|
| **Nom** | `TaskStop` (alias: `KillShell`) |
| **Fichier** | `tools/TaskStopTool/TaskStopTool.ts` |
| **Read-only** | Non |
| **Concurrent** | Non |

**Paramètres**

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `task_id` | string | non | ID de la tâche (au moins un requis) |
| `shell_id` | string | non | ID shell (déprécié) |

---

### 2.18 TodoWrite (legacy)

| | |
|---|---|
| **Nom** | `TodoWrite` |
| **Fichier** | `tools/TodoWriteTool/TodoWriteTool.ts` |
| **Read-only** | Non |
| **Concurrent** | Non |
| **Condition** | `isTodoV2Enabled() === false` |

**Paramètres**

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `todos` | array | oui | Liste complète de todos |

---

## 3. Outils de gestion de tâches

**Condition** : `isTodoV2Enabled() === true`

### 3.1 TaskCreate

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `subject` | string | oui | Titre de la tâche |
| `description` | string | oui | Description détaillée |
| `activeForm` | string | non | Formulaire actif |
| `metadata` | object | non | Métadonnées |

Retourne un `task_id`.

### 3.2 TaskGet

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `taskId` | string | oui | ID de la tâche |

Retourne la tâche complète avec `blocks`/`blockedBy`.

### 3.3 TaskList

Aucun paramètre. Retourne toutes les tâches (filtre les internes).

### 3.4 TaskUpdate

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `taskId` | string | oui | ID de la tâche |
| `subject` | string | non | Nouveau titre |
| `description` | string | non | Nouvelle description |
| `status` | enum | non | `pending`, `in_progress`, `completed`, `deleted` |
| `addBlocks` | string[] | non | IDs des tâches bloquées |
| `addBlockedBy` | string[] | non | IDs des tâches bloquantes |
| `owner` | string | non | Propriétaire |
| `metadata` | object | non | Métadonnées |

---

## 4. Outils Worktree

**Condition** : `isWorktreeModeEnabled()`

### 4.1 EnterWorktree

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `name` | string | non | Nom slug (max 64 chars) |

Crée un worktree git isolé et change le cwd.

### 4.2 ExitWorktree

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `action` | enum | oui | `keep` ou `remove` |
| `discard_changes` | boolean | non | Requis pour suppression sûre |

Compte les fichiers/commits modifiés, tue les sessions tmux, restaure le cwd original.

---

## 5. Outils MCP

### 5.1 MCPTool (dynamique)

Template dynamique — instancié par serveur MCP connecté. Nommage : `{serveur}__{outil}`.

Le schéma d'entrée est passthrough vers le serveur MCP. Les permissions sont par outil.

### 5.2 ListMcpResources

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `server` | string | non | Filtrer par nom de serveur |

Retourne URI, nom, mimeType, description. Cache LRU.

### 5.3 ReadMcpResource

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `server` | string | oui | Nom du serveur |
| `uri` | string | oui | URI de la ressource |

Gère le texte et les blobs binaires (persistance fichier).

### 5.4 McpAuth (dynamique)

Nom : `mcp__{serveur}__authenticate`. Initie un flow OAuth, retourne l'URL d'auth, remplace l'outil une fois complété.

---

## 6. Outils Team / Multi-agents

**Condition** : `isAgentSwarmsEnabled()`

### 6.1 TeamCreate

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `team_name` | string | oui | Nom de l'équipe |
| `description` | string | non | Description |
| `agent_type` | string | non | Type d'agent |

Génère un nom unique, écrit `team.json`, assigne les couleurs.

### 6.2 TeamDelete

Aucun paramètre. Supprime `team.json`, nettoie les répertoires, libère les couleurs.

### 6.3 SendMessage

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `to` | string | oui | Destinataire (nom, `*` pour broadcast, adresse peer) |
| `message` | string/object | oui | Contenu du message |
| `summary` | string | non | Résumé |
| `attachments` | array | non | Pièces jointes |

Route vers la mailbox du coéquipier. Supporte shutdown, approbation de plans, broadcast.

---

## 7. Outils Cron / Scheduling

**Condition** : `feature('AGENT_TRIGGERS')`

### 7.1 CronCreate

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `cron` | string | oui | Expression cron 5 champs |
| `prompt` | string | oui | Prompt à exécuter |
| `recurring` | boolean | non | Récurrent (défaut: true) |
| `durable` | boolean | non | Persistant sur disque |

Max 50 jobs. Auto-expiration après 7 jours. Parse en description humainement lisible.

### 7.2 CronDelete

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `id` | string | oui | ID du job |

### 7.3 CronList

Aucun paramètre. Filtre par contexte d'équipe.

---

## 8. Outils Kairos

**Condition** : `feature('KAIROS')` et sous-flags associés

### 8.1 Sleep

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `duration` | number | oui | Durée en ms |

**Condition** : `feature('PROACTIVE')` ou `feature('KAIROS')`

### 8.2 SendUserFile

Envoie des fichiers à l'utilisateur en mode assistant. **Condition** : `feature('KAIROS')`.

### 8.3 PushNotification

Envoie des notifications push vers l'appareil de l'utilisateur. **Condition** : `feature('KAIROS')` ou `feature('KAIROS_PUSH_NOTIFICATION')`.

### 8.4 SubscribePR

S'abonne aux événements webhook d'une PR GitHub. **Condition** : `feature('KAIROS_GITHUB_WEBHOOKS')`.

---

## 9. Outils développeur / Ant-only

Réservés aux employés Anthropic (`process.env.USER_TYPE === 'ant'`).

### 9.1 Config

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `setting` | string | oui | Nom du paramètre (`theme`, `model`, etc.) |
| `value` | string | non | Valeur (omis = lecture) |

Lecture/écriture de la configuration globale et `settings.json`.

### 9.2 REPL

Exécution de code dans une VM isolée (Python/Node.js/Shell). Masque les outils primitifs (Bash/Read/Edit) quand activé.

### 9.3 Tungsten

Multiplexeur terminal tmux. Sessions persistantes, panneau live dans l'UI.

### 9.4 SuggestBackgroundPR

Suggestion de création de PR en arrière-plan.

---

## 10. Outils feature-gated divers

| Outil | Flag | Description |
|-------|------|-------------|
| `WebBrowser` | `WEB_BROWSER_TOOL` | Navigateur web automatisé (Bun.WebView) |
| `Monitor` | `MONITOR_TOOL` | Monitoring |
| `RemoteTrigger` | `AGENT_TRIGGERS_REMOTE` | Gestion de triggers distants via API CCR |
| `ListPeers` | `UDS_INBOX` | Liste des peers UDS connectés |
| `Workflow` | `WORKFLOW_SCRIPTS` | Exécution de scripts workflow |
| `Snip` | `HISTORY_SNIP` | Snippets d'historique |
| `CtxInspect` | `CONTEXT_COLLAPSE` | Inspection du contexte/tokens |
| `TerminalCapture` | `TERMINAL_PANEL` | Capture du panneau terminal |
| `PowerShell` | `isPowerShellToolEnabled()` | Commandes PowerShell (Windows) |
| `OverflowTest` | `OVERFLOW_TEST_TOOL` | Test de dépassement de contexte |
| `VerifyPlanExecution` | env `CLAUDE_CODE_VERIFY_PLAN` | Vérification de sécurité des plans |
| `LSP` | env `ENABLE_LSP_TOOL` | Language Server Protocol (go-to-def, references, hover, symbols) |

### LSP — Détail des opérations

| Opération | Description |
|-----------|-------------|
| `goToDefinition` | Aller à la définition |
| `findReferences` | Trouver les références |
| `hover` | Information au survol |
| `documentSymbol` | Symboles du document |
| `workspaceSymbol` | Symboles du workspace |
| `goToImplementation` | Aller à l'implémentation |
| `prepareCallHierarchy` | Préparer la hiérarchie d'appels |
| `incomingCalls` | Appels entrants |
| `outgoingCalls` | Appels sortants |

---

## 11. Outils internes / Testing

| Outil | Condition | Description |
|-------|-----------|-------------|
| `SyntheticOutputTool` | Filtré à l'assemblage | Output synthétique (jamais exposé au modèle) |
| `TestingPermissionTool` | `NODE_ENV === 'test'` | Outil de test interne |

---

## 12. Pipeline d'exécution

Quand le modèle émet un `tool_use` block, voici le chemin complet :

```
API Response: { name: "Bash", input: { command: "ls" } }
    │
    ▼
1. LOOKUP — Trouver l'outil dans toolUseContext.options.tools
    │         Fallback: getAllBaseTools() pour les alias dépréciés
    ▼
2. VALIDATION — inputSchema.safeParse(input)
    │             tool.validateInput?(input, context)
    ▼
3. PRE-HOOKS — runPreToolUseHooks()
    │            Peuvent : modifier l'input, refuser, demander permission, bloquer
    ▼
4. PERMISSIONS — resolveHookPermissionDecision()
    │              hook result → règles → canUseTool() (interactif)
    │              Résultat : { behavior: 'allow'|'deny'|'ask' }
    ▼
5. EXÉCUTION — tool.call(input, context, canUseTool, assistantMessage, onProgress)
    │            Via StreamingToolExecutor (file d'attente concurrente)
    ▼
6. POST-HOOKS — runPostToolUseHooks()
    │              Peuvent : modifier la sortie MCP, ajouter du contexte, bloquer
    ▼
7. SÉRIALISATION — mapToolResultToToolResultBlockParam()
    │                 Intégré dans le ToolResultBlockParam envoyé à l'API
    ▼
Résultat injecté dans la conversation
```

---

## 13. Modèle de concurrence

Géré par `StreamingToolExecutor` :

### États d'un outil

```
queued → executing → completed → yielded
```

### Règles de concurrence

| Scénario | Comportement |
|----------|-------------|
| Outil non-concurrent + file vide | Exécution immédiate |
| Outil non-concurrent + outils en cours | Attend la fin de tous |
| Outil concurrent + file vide | Exécution immédiate |
| Outil concurrent + outils concurrents en cours | Exécution parallèle |
| Outil concurrent + outil non-concurrent en cours | Attend |

```typescript
canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executing = this.tools.filter(t => t.status === 'executing')
  return (
    executing.length === 0 ||
    (isConcurrencySafe && executing.every(t => t.isConcurrencySafe))
  )
}
```

**Les résultats sont toujours émis dans l'ordre de réception**, pas d'exécution.

### Classification par outil

| Concurrent (parallèle) | Non-concurrent (exclusif) |
|------------------------|--------------------------|
| Glob, Grep, Read, WebFetch, WebSearch, TaskOutput, ToolSearch, Agent | Bash, Edit, Write, NotebookEdit, Skill, AskUserQuestion, tous les Task*, Team*, Cron* |

---

## 14. Système de permissions

### Modes

| Mode | Description |
|------|-------------|
| `default` | Demande avant les opérations risquées |
| `bypassPermissions` | Tout autoriser (YOLO) |
| `acceptEdits` | Autorise les éditions, demande pour le reste |
| `dontAsk` | Ne demande jamais (deny par défaut) |
| `plan` | Mode plan (restrictions) |
| `auto` | Classifieur ML décide |
| `bubble` | Interne pour sous-agents |

### Chaîne de décision

```
1. Règles explicites (allow/deny lists dans settings.json)
2. Mode de permission global
3. Classifieurs ML :
   ├── Bash Classifier → analyse les patterns dangereux
   └── Transcript Classifier → ML sur l'historique
4. Hooks permission_request
5. Si indécis → prompt interactif utilisateur
```

### Sources de règles (par priorité)

`policy` > `user` > `project` > `local` > `CLI` > `command` > `session`

---

## 15. Hooks pre/post outil

### Pre-Tool Hooks

Déclencheur : `pre_tool_use`

**Peuvent** :
- Inspecter et modifier l'input avant exécution
- Refuser l'exécution (deny)
- Demander confirmation à l'utilisateur (ask)
- Empêcher la continuation

**Yield** :
```typescript
{ type: 'message' | 'hookPermissionResult' | 'hookUpdatedInput' | 'preventContinuation' | 'stop' }
```

### Post-Tool Hooks

Déclencheur : `post_tool_use`, `post_tool_use_failure`

**Peuvent** :
- Modifier la sortie (surtout MCP)
- Ajouter du contexte ou des corrections
- Empêcher la continuation
- S'exécutent **même si l'outil a échoué**

### Stratégies d'exécution

| Stratégie | Mécanisme |
|-----------|-----------|
| Shell | Script subprocess avec JSON I/O |
| HTTP | Appel endpoint REST |
| Prompt | Fonction inline |

### Timeouts

- Outils : **10 minutes**
- Fin de session : **1.5 secondes**

---

## Récapitulatif — Nombre total d'outils

| Catégorie | Nombre |
|-----------|--------|
| Core (toujours disponibles) | 18 |
| Gestion de tâches (v2) | 4 |
| Worktree | 2 |
| MCP (statiques) | 4 |
| MCP (dynamiques par serveur) | N |
| Team / Multi-agents | 3 |
| Cron / Scheduling | 3 |
| Kairos | 4 |
| Ant-only | 4 |
| Feature-gated divers | 12 |
| Internes / Testing | 2 |
| **Total (statiques)** | **~56** |
| **+ outils MCP dynamiques** | **variable** |
