# Système de Contextes — Documentation complète

Toutes les formes de contexte dans Claude Code : 12 types de contexte, 41 points d'injection, gestion de la fenêtre de contexte, isolation multi-agents et survie à la compaction.

---

## Table des matières

1. [Vue d'ensemble — Les 12 types de contexte](#1-vue-densemble)
2. [ToolUseContext — Le contexte central d'exécution](#2-toolusecontext)
3. [AppState — L'état runtime partagé](#3-appstate)
4. [SystemContext & UserContext — Contexte de session](#4-session-context)
5. [System Prompt — Le prompt assemblé](#5-system-prompt)
6. [QueryConfig — Snapshot immutable de requête](#6-queryconfig)
7. [TeammateContext & AgentContext — Isolation AsyncLocalStorage](#7-isolation)
8. [FileStateCache — Contexte de lecture fichier](#8-filestacecache)
9. [ContentReplacementState — Budget des résultats d'outils](#9-content-replacement)
10. [Points d'injection du contexte (41)](#10-points-injection)
11. [Gestion de la fenêtre de contexte](#11-fenêtre-contexte)
12. [Optimisation du contexte](#12-optimisation)
13. [Survie du contexte à la compaction](#13-compaction)
14. [Isolation du contexte pour la concurrence](#14-isolation-concurrence)
15. [Mutabilité et immutabilité](#15-mutabilité)
16. [Fichiers clés](#16-fichiers-clés)

---

## 1. Vue d'ensemble

Claude Code gère **12 types de contexte** distincts qui collaborent pour fournir au modèle toute l'information nécessaire à chaque tour de conversation.

| Type | Scope | Mutable | Isolé pour agents | Cache |
|------|-------|---------|-------------------|-------|
| **ToolUseContext** | Exécution de requête | Partiel | Cloné | readFileState LRU |
| **AppState** | Session | Oui | Wrappé (read-only) | Non |
| **SystemContext** | Session | Non | N/A | Memoizé |
| **UserContext** | Session | Non | N/A | Memoizé |
| **SystemPrompt** | Requête | Non | N/A | Par requête |
| **QueryConfig** | Requête | Non | N/A | Snapshot unique |
| **TeammateContext** | In-process | Non | AsyncLocalStorage | Non |
| **AgentContext** | In-process | Partiel | AsyncLocalStorage | Non |
| **FileStateCache** | Chaîne d'exécution | Oui | Cloné | LRU + taille |
| **ContentReplacementState** | Requête | Oui | Cloné pour forks | Reconstruit au resume |
| **QueryTracking** | Requête | Oui | Via ToolUseContext | Non |
| **REPLHookContext** | Par invocation hook | Non | N/A | Non |

---

## 2. ToolUseContext — Le contexte central d'exécution

Le coeur du système — passé à chaque outil, chaque hook, chaque sous-agent.

### Contenu

```typescript
ToolUseContext {
  // Configuration
  options: {
    tools: Tool[],              // Outils disponibles
    mcpClients: MCPClient[],    // Serveurs MCP connectés
    agentDefinitions: Agent[],  // Agents définis
    model: string,              // Modèle en cours
    thinkingConfig: ThinkingConfig,
    debug: boolean,
  },

  // Lifecycle
  abortController: AbortController,
  messages: Message[],          // Historique de conversation (ref mutable)

  // Caches
  readFileState: FileStateCache,          // LRU 100 entrées / 25 MB
  contentReplacementState: ContentReplacementState,

  // État mutable
  nestedMemoryAttachmentTriggers: Set<string>,  // CLAUDE.md injectés
  loadedNestedMemoryPaths: Set<string>,         // Fichiers mémoire chargés
  dynamicSkillDirTriggers: Set<string>,         // Skills découverts
  discoveredSkillNames: Set<string>,
  toolDecisions: Map<string, Decision>,         // Historique approbations

  // Callbacks (no-op pour sous-agents sauf opt-in)
  getAppState(): AppState,
  setAppState(updater): void,
  setAppStateForTasks(updater): void,   // Toujours atteint la racine
  handleElicitation: ElicitationHandler,
  requestPrompt: PromptFactory,

  // Tracking
  queryTracking: { chainId: string, depth: number },
  toolUseId: string,
  criticalSystemReminder_EXPERIMENTAL?: string,

  // Rendu (undefined pour agents background)
  setToolJSX?: Function,
  addNotification?: Function,
  renderedSystemPrompt?: string[],      // Figé au début du tour (cache-sharing)
}
```

### Flux

```
1. Créé dans les params de query()
2. Porté à travers chaque itération du loop
3. Mis à jour via queryTracking mid-itération
4. Passé aux outils via StreamingToolExecutor
5. Modifiable par contextModifier dans les résultats d'outils
6. Cloné pour les sous-agents via createSubagentContext()
```

---

## 3. AppState — L'état runtime partagé

Store unique partagé par toute la session, géré via `getAppState()` / `setAppState()`.

### Sections principales

| Section | Contenu |
|---------|---------|
| **Settings & Model** | settings, verbose, mainLoopModel |
| **Permissions** | toolPermissionContext (rules, mode, bypass) |
| **UI** | expandedView, footerSelection, statusLine |
| **Tasks** | tasks (Map), foregroundedTaskId, agentNameRegistry |
| **MCP & Plugins** | clients, tools, commands, resources, enabled/disabled plugins |
| **Team** | teamContext (name, leader, teammates), inbox |
| **Thinking** | thinkingEnabled, promptSuggestionEnabled |
| **Fast Mode** | fastMode, advisorModel, effortValue |
| **File History** | fileHistory, attribution |
| **Notifications** | notifications queue, elicitation queue |
| **Hooks** | sessionHooks |

### Isolation

- **Agent principal** : accès lecture/écriture complet
- **Sous-agents** : `getAppState()` wrappé (read-only, `shouldAvoidPermissionPrompts=true`)
- **setAppStateForTasks()** : toujours atteint la racine (pour l'enregistrement d'infrastructure)

---

## 4. SystemContext & UserContext — Contexte de session

### SystemContext

```typescript
getSystemContext() → {
  gitStatus: string,    // Branche, status, commits récents (snapshot au démarrage)
  cacheBreaker?: string // Debug ant-only
}
```

- **Memoizé** une fois par session
- **Appendé** au system prompt via `appendSystemContext()`
- Skippé en mode CCR (remote)

### UserContext

```typescript
getUserContext() → {
  claudeMd: string,     // Contenu CLAUDE.md (managed → user → project → local)
  currentDate: string   // "Today's date is 2026-04-02."
}
```

- **Memoizé** une fois par session
- **Prepended** aux messages via `prependUserContext()`
- Wrappé dans `<system-reminder>`
- Invalidé sur `/clear` ou `/compact` → rechargé depuis le disque

---

## 5. System Prompt — Le prompt assemblé

### Priorité d'assemblage

```
1. Override (loop mode)          ← REMPLACE tout
2. Coordinator mode              ← Si CLAUDE_CODE_COORDINATOR_MODE
3. Agent system prompt           ← Si mainThreadAgentDefinition
4. Custom (--system-prompt)
5. Default (getSystemPrompt())   ← Standard
6. Append (--append-system-prompt)
```

### Sections

| Zone | Contenu | Cache |
|------|---------|-------|
| **Statique** (avant DYNAMIC_BOUNDARY) | Identity, tools, tasks, actions, tone, output | `scope: global` (cross-sessions) |
| **Dynamique** (après boundary) | Session guidance, memory, environment, language, MCP, scratchpad, brief | `scope: ephemeral` (par session) |

Chaque section est memoizée via `systemPromptSection(name, compute)` dans `STATE.systemPromptSectionCache`. Invalidé sur `/clear` ou `/compact`.

---

## 6. QueryConfig — Snapshot immutable

Capturé **une seule fois** à l'entrée de `query()` :

```typescript
QueryConfig {
  sessionId: string,
  gates: {
    streamingToolExecution: boolean,
    emitToolUseSummaries: boolean,
    isAnt: boolean,
    fastModeEnabled: boolean,
  }
}
```

Ne change jamais pendant la boucle de requête. Les feature gates inline (`feature('FLAG')`) restent pour le tree-shaking Bun.

---

## 7. TeammateContext & AgentContext — Isolation

### Pourquoi AsyncLocalStorage

Quand des agents sont en arrière-plan (Ctrl+B), plusieurs agents s'exécutent en concurrence dans le même processus. AppState est un store unique partagé → serait écrasé. AsyncLocalStorage isole chaque chaîne async.

### TeammateContext

```typescript
TeammateContext {
  agentId: "researcher@my-team",
  agentName: "researcher",
  teamName: "my-team",
  color: "blue",
  planModeRequired: boolean,
  parentSessionId: string,
  isInProcess: true,
  abortController: AbortController,
}
```

Accès : `getTeammateContext()`, `runWithTeammateContext(ctx, fn)`

### AgentContext (union discriminée)

**SubagentContext** :
```typescript
{ agentType: 'subagent', agentId, parentSessionId, subagentName, isBuiltIn, invokingRequestId }
```

**TeammateAgentContext** :
```typescript
{ agentType: 'teammate', agentId, agentName, teamName, isTeamLead, invokingRequestId }
```

Accès : `getAgentContext()`, `runWithAgentContext(ctx, fn)`

---

## 8. FileStateCache — Contexte de lecture

Cache LRU qui suit les fichiers déjà lus dans la session.

```typescript
FileState {
  content: string,         // Contenu brut du disque
  timestamp: number,       // mtime
  offset?: number,
  limit?: number,
  isPartialView?: boolean, // True si le modèle a vu une version tronquée
}
```

| Propriété | Valeur |
|-----------|--------|
| Max entrées | 100 |
| Max taille | 25 MB |
| Clé | Chemin absolu normalisé |
| Éviction | LRU |
| Isolation | Cloné pour chaque sous-agent |

**Opérations** : `cloneFileStateCache()`, `mergeFileStateCaches()`, `cacheToObject()` (pour compaction)

---

## 9. ContentReplacementState — Budget résultats

Gère le budget de taille des résultats d'outils par conversation.

```typescript
ContentReplacementState {
  seenIds: Set<string>,                    // Tool use IDs déjà traités
  replacements: Map<string, string>,       // ID → texte de remplacement
}
```

- **Appliqué** par `applyToolResultBudget()` à chaque tour
- **Cloné** pour les forks cache-sharing (mêmes décisions → cache hit)
- **Reconstruit** au resume depuis les enregistrements sidechain
- **Persisté** uniquement pour `agent:*` et `repl_main_thread`

---

## 10. Points d'injection du contexte (41)

### Phase 1 — Initialisation de session

| # | Injection | Quand | Wrapping |
|---|-----------|-------|----------|
| 1 | System prompt (statique + dynamique) | Chaque tour | Paramètre `system` API |
| 2 | User context (CLAUDE.md + date) | Chaque tour | `<system-reminder>` prepended |
| 3 | System context (git status) | Session start | Appendé au system prompt |

### Phase 2 — Input utilisateur

| # | Injection | Trigger | Type |
|---|-----------|---------|------|
| 4 | @mentioned files | `@filename` | file / pdf_reference / directory |
| 5 | MCP resources | `@server:resource` | mcp_resource |
| 6 | Agent mentions | `@agent-type` | agent_mention |
| 7 | Skill discovery (tour 0) | Premier input | skill_discovery |

### Phase 3 — Attachments par tour

| # | Injection | Trigger | Type |
|---|-----------|---------|------|
| 8 | Queued commands | Message queue drain | queued_command |
| 9 | Date change | Changement de jour | date_change |
| 10 | Nested memory (CLAUDE.md lazy) | Accès fichier | nested_memory |
| 11 | Relevant memories | Prefetch Sonnet | relevant_memories |
| 12 | Skill listing | Changement skills | skill_listing |
| 13 | Deferred tools delta | Changement outils | deferred_tools_delta |
| 14 | Agent listing delta | Changement agents | agent_listing_delta |
| 15 | MCP instructions delta | Connexion MCP | mcp_instructions_delta |
| 16 | Diagnostics (IDE/LSP) | Nouveaux diagnostics | diagnostics |
| 17 | Plan mode / Auto mode | Entrée/sortie mode | plan_mode / auto_mode |
| 18 | Todo / Task reminders | 10+ tours sans update | todo_reminder / task_reminder |
| 19 | Unified task status | Complétion tâche async | task_status |
| 20 | Token / budget usage | Chaque tour | token_usage / budget_usd |
| 21 | Changed files (git delta) | Accès fichier | changed_files |

### Phase 4 — Post-exécution outils

| # | Injection | Mécanisme |
|---|-----------|-----------|
| 22 | Context modifiers | `tool.call()` retourne `contextModifier` → appliqué par StreamingToolExecutor |
| 23 | Async hook responses | `AsyncHookRegistry` → collecté au tour suivant |

### Phase 5 — Main thread uniquement

| # | Injection | Type |
|---|-----------|------|
| 24 | IDE selection | selected_lines_in_ide |
| 25 | IDE opened file | opened_file_in_ide |
| 26 | Output style | output_style |
| 27 | Verify plan reminder | critical_system_reminder |
| 28 | Companion intro (Buddy) | companion_intro |

### Phase 6 — Swarm

| # | Injection | Type |
|---|-----------|------|
| 29 | Teammate mailbox | teammate_mailbox |
| 30 | Team context | team_context |

### Phase 7 — Restauration post-compaction

| # | Injection | Budget |
|---|-----------|--------|
| 31 | Recent files | 5 fichiers, 50K tokens |
| 32 | Invoked skills | 25K tokens |
| 33 | Active plan | Contenu complet |
| 34 | Plan mode instructions | Si en mode plan |
| 35 | Async agent status | Par agent |
| 36 | Tool/Agent/MCP deltas | Diff vs preserved |
| 37 | Session start hooks | CLAUDE.md refresh |
| 38 | Compact boundary marker | Métadonnées compaction |

### Phase 8 — Spécial

| # | Injection | Quand |
|---|-----------|-------|
| 39 | Hook initial message | Session start (hooks emettent `initialUserMessage`) |
| 40 | Instructions loaded hooks | Audit (fire-and-forget) |
| 41 | Context modifiers post-tool | Outils non-concurrents uniquement |

---

## 11. Gestion de la fenêtre de contexte

### Détermination de la taille

```
Context window :
  1. Override env : CLAUDE_CODE_MAX_CONTEXT_TOKENS
  2. Suffixe modèle [1m] → 1,000,000
  3. Registre modèle max_input_tokens
  4. Beta header CONTEXT_1M
  5. Défaut : 200,000 tokens
```

### Allocation de l'espace

```
Fenêtre totale :           200,000 tokens
├── System prompt :        ~15-30K (statique + dynamique)
├── Messages :             Variable (historique + tool results)
├── Réserve output :       -20,000 (MAX_OUTPUT_TOKENS_FOR_SUMMARY)
└── Buffer auto-compact :  -13,000 (AUTOCOMPACT_BUFFER_TOKENS)
                          ─────────
    Seuil auto-compact :   ~167,000 tokens
```

### Seuils

| Seuil | Valeur (200K) | Action |
|-------|---------------|--------|
| Auto-compact | 167K | Déclenche la compaction proactive |
| Warning | 147K | Avertissement utilisateur |
| Error | 147K | État d'erreur |
| Blocking limit | 177K | Bloque les requêtes (compaction manuelle requise) |

---

## 12. Optimisation du contexte

### Comptage de tokens

| Méthode | Précision | Coût | Usage |
|---------|-----------|------|-------|
| `roughTokenCountEstimation()` | Basse (length/4) | 0 | Décisions rapides |
| `tokenCountWithEstimation()` | Moyenne (API + delta) | 0 | Auto-compact, warnings |
| `countMessagesTokensWithAPI()` | Haute (API countTokens) | 1 call | Vérification précise |

### Microcompaction (3 variantes)

| Variante | Trigger | Cache impact | Mécanisme |
|----------|---------|-------------|-----------|
| Cached MC | Nombre d'outils | Préserve | `cache_edits` API |
| Time-based MC | Gap > 60 min | Aucun (expiré) | Mutation directe |
| API context mgmt | 180K tokens | Server-side | Paramètre `context_management` |

### Suggestions d'optimisation

```
generateContextSuggestions(data) :
  ├── ≥ 80% capacité → "Near capacity"
  ├── ≥ 15% ou ≥ 10K tokens en tool results → "Large tool results"
  ├── ≥ 5% en reads → "Read result bloat"
  ├── ≥ 5% ou ≥ 5K en mémoire → "Memory bloat"
  └── 50-80% + auto-compact désactivé → "Enable auto-compact"
```

---

## 13. Survie du contexte à la compaction

### Ce qui survit

| Élément | Mécanisme |
|---------|-----------|
| Messages récents | Préservés verbatim (10-40K tokens) |
| CLAUDE.md | Rechargé depuis le disque (processSessionStartHooks) |
| MEMORY.md | Rechargé (section system prompt) |
| Fichiers récents | Relus (top 5, budget 50K tokens) |
| Skills invoqués | Réinjectés (budget 25K) |
| Plan actif | Relu depuis le disque |
| Mode plan | Instructions réinjectées |
| Agents async | Statut réinjecté |
| Outils/agents/MCP | Deltas recalculés |
| Outils découverts | `preCompactDiscoveredTools` dans le boundary marker |

### Ce qui est perdu

| Élément | Raison |
|---------|--------|
| Images/PDF dans les messages | Strippés avant résumé (évite prompt_too_long) |
| Tool results au-delà de la fenêtre | Résumés, pas préservés |
| Attachments non listés | Non dans la liste de réinjection |
| Skill discovery | Non ré-annoncé post-compact |

---

## 14. Isolation du contexte pour la concurrence

### Sous-agents (Agent tool)

```
createSubagentContext() :
  ├── readFileState → CLONÉ (isolation LRU)
  ├── contentReplacementState → CLONÉ (cache-sharing)
  ├── abortController → NOUVEAU enfant (lié au parent)
  ├── getAppState → WRAPPÉ (read-only, shouldAvoidPermissionPrompts=true)
  ├── setAppState → NO-OP (sauf opt-in explicite)
  ├── nestedMemoryAttachmentTriggers → FRAIS (Set vide)
  ├── loadedNestedMemoryPaths → FRAIS
  ├── dynamicSkillDirTriggers → FRAIS
  ├── toolDecisions → FRAIS (Map vide)
  └── renderedSystemPrompt → FIGÉ (du tour parent)
```

### Teammates swarm (processus séparés via tmux)

```
Isolation OS complète :
  ├── CLAUDE_CODE_AGENT_ID env var
  ├── CLAUDE_CODE_PARENT_SESSION_ID env var
  ├── dynamicTeamContext global
  └── Processus séparé → mémoire isolée
```

### Forks cache-sharing (speculation, agent_summary)

```
Mêmes params cache-critiques (system prompt, tools, model, messages prefix)
  ├── readFileState → CLONÉ (contenu identique, mutations séparées)
  ├── contentReplacementState → CLONÉ (mêmes décisions)
  └── Résultat : préfixe wire identique → prompt cache hit
```

---

## 15. Mutabilité et immutabilité

### Immutable

| Contexte | Raison |
|----------|--------|
| QueryConfig | Snapshotté une fois |
| SystemPrompt | Figé par requête |
| UserContext | Memoizé, cachable |
| SystemContext | Snapshot au démarrage |
| toolUseContext.options | Read-only |
| renderedSystemPrompt | Figé pour cache-sharing |

### Mutable

| Contexte | Mutations |
|----------|-----------|
| readFileState | LRU set/delete par les outils Read |
| contentReplacementState | seenIds, replacements par applyToolResultBudget |
| AppState | Via setAppState (UI, tools, hooks) |
| toolDecisions | Historique d'approbations |
| nestedMemoryAttachmentTriggers | Chemins CLAUDE.md injectés |
| discoveredSkillNames | Skills trouvés dynamiquement |
| messages (ref) | Liste mutable via accumulation |

---

## 16. Fichiers clés

### Contexte central

| Fichier | Rôle |
|---------|------|
| `Tool.ts` | Type ToolUseContext, contextModifier |
| `context.ts` | getUserContext(), getSystemContext() |
| `utils/systemPrompt.ts` | buildEffectiveSystemPrompt() |
| `constants/systemPromptSections.ts` | systemPromptSection(), cache |
| `query/config.ts` | QueryConfig snapshot |
| `state/AppStateStore.ts` | AppState store |

### Isolation

| Fichier | Rôle |
|---------|------|
| `utils/teammateContext.ts` | TeammateContext AsyncLocalStorage |
| `utils/agentContext.ts` | AgentContext AsyncLocalStorage |
| `utils/forkedAgent.ts` | createSubagentContext() |

### Caches

| Fichier | Rôle |
|---------|------|
| `utils/fileStateCache.ts` | FileStateCache LRU |
| `utils/toolResultStorage.ts` | ContentReplacementState |
| `bootstrap/state.ts` | SystemPromptSectionCache |

### Injection

| Fichier | Rôle |
|---------|------|
| `utils/attachments.ts` | 41 types d'attachments, assembly pipeline |
| `utils/api.ts` | prependUserContext(), appendSystemContext() |
| `utils/messages.ts` | normalizeAttachmentForAPI() |
| `services/compact/compact.ts` | Restauration post-compaction |

### Fenêtre de contexte

| Fichier | Rôle |
|---------|------|
| `utils/context.ts` | getContextWindowForModel(), getMaxOutputTokens() |
| `utils/tokens.ts` | tokenCountWithEstimation() (canonical) |
| `utils/analyzeContext.ts` | Breakdown et visualisation |
| `utils/contextSuggestions.ts` | Suggestions d'optimisation |
| `services/compact/autoCompact.ts` | Seuils et circuit breaker |
