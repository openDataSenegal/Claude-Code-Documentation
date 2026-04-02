# Communication avec l'IA — Documentation complète

Ce document décrit exactement **quelles informations** sont envoyées au modèle Claude et **comment** elles sont assemblées, formatées et transmises via l'API.

---

## Table des matières

1. [Vue d'ensemble du pipeline](#1-vue-densemble-du-pipeline)
2. [System prompt — Construction et contenu](#2-system-prompt)
3. [Contexte utilisateur — CLAUDE.md et date](#3-contexte-utilisateur)
4. [Contexte système — Git et environnement](#4-contexte-système)
5. [Messages — Normalisation et formatage](#5-messages)
6. [Attachments — Injection de contexte](#6-attachments)
7. [Outils — Schémas envoyés au modèle](#7-outils)
8. [Thinking — Configuration de la réflexion](#8-thinking)
9. [Prompt caching — Optimisation du cache](#9-prompt-caching)
10. [Paramètres API complets](#10-paramètres-api-complets)
11. [Headers et métadonnées](#11-headers-et-métadonnées)
12. [Betas — Fonctionnalités activées](#12-betas)
13. [Streaming — Réception de la réponse](#13-streaming)
14. [Sélection du modèle](#14-sélection-du-modèle)
15. [Fast mode](#15-fast-mode)
16. [Context management — Microcompaction](#16-context-management)
17. [Pipeline complet — Du clavier à l'API](#17-pipeline-complet)
18. [Fichiers clés](#18-fichiers-clés)

---

## 1. Vue d'ensemble du pipeline

```
Saisie utilisateur
    ↓
[1] Construction du system prompt (sections statiques + dynamiques)
    ↓
[2] Assemblage du contexte (CLAUDE.md, date, git, environnement)
    ↓
[3] Normalisation des messages (historique conversation)
    ↓
[4] Conversion des attachments (fichiers, mémoires, tâches, skills)
    ↓
[5] Construction des schémas d'outils (built-in + MCP + deferred)
    ↓
[6] Configuration thinking + caching + betas
    ↓
[7] Appel API : anthropic.beta.messages.create({ stream: true })
    ↓
[8] Streaming : réception chunk par chunk
    ↓
[9] Normalisation de la réponse → AssistantMessage
```

**Fichier principal** : `services/api/claude.ts` → `queryModel()` (lignes 1017-2700+)

---

## 2. System prompt — Construction et contenu

### Hiérarchie de priorité

**Fichier** : `utils/systemPrompt.ts` → `buildEffectiveSystemPrompt()`

```
1. Override system prompt (--system-prompt flag)     ← REMPLACE tout
2. Coordinator mode prompt (si COORDINATOR_MODE)
3. Agent system prompt (si agent actif) :
   ├── Mode proactif → appendé au défaut
   └── Sinon → remplace le défaut
4. Custom system prompt (si --system-prompt)
5. ★ Default system prompt (getSystemPrompt())       ← Standard
6. Memory prompt (loadMemoryPrompt())
7. Append system prompt (--append-system-prompt)
```

### Sections du system prompt par défaut

**Fichier** : `constants/prompts.ts` → `getSystemPrompt()`

Le system prompt est un tableau `string[]` où chaque élément est une section :

#### Sections statiques (cachables, cross-organisation)

| Section | Contenu |
|---------|---------|
| **Intro** | Identité Claude Code, instruction cyber risk |
| **System** | Outils, permissions, résultats d'outils, hooks, compression |
| **Doing tasks** | Style de code, comportement tâches, guidelines software engineering |
| **Actions** | Réversibilité, blast radius, confirmation utilisateur |
| **Using tools** | Outils dédiés vs bash, exécution parallèle, outils de tâche |
| **Tone and style** | Pas d'emojis, markdown, références GitHub, pas de ":" avant tool calls |
| **Output efficiency** | Brief/direct, milestones uniquement |

#### Marqueur de frontière dynamique

```
SYSTEM_PROMPT_DYNAMIC_BOUNDARY
```

Tout **avant** ce marqueur peut être caché avec `scope: 'global'`. Tout **après** est dynamique et session-specific.

#### Sections dynamiques (memoizées, par session)

| Section | Contenu |
|---------|---------|
| **Session guidance** | AskUserQuestion, syntaxe `!command`, agents, skills, verification |
| **Memory** | MEMORY.md + instructions de mémoire (via `loadMemoryPrompt()`) |
| **Environment** | cwd, git status, plateforme, shell, OS, modèle, date de cutoff |
| **Language preference** | Langue de l'utilisateur détectée |
| **Output style** | Configuration du style de sortie |
| **MCP instructions** | Instructions pour les serveurs MCP connectés |
| **Scratchpad** | Instructions scratchpad (si activé) |
| **Token budget** | Guidance budget tokens (si feature activée) |
| **Brief** | Mode KAIROS/brief |

### Cache par session

Chaque section est memoizée via `systemPromptSection(name, computeFn)` :

```typescript
systemPromptSection('memory', () => loadMemoryPrompt())
```

- Stocké dans `STATE.systemPromptSectionCache`
- Invalidé sur `/clear` ou `/compact`
- Rechargé depuis le disque (contenu frais)

---

## 3. Contexte utilisateur — CLAUDE.md et date

**Fichier** : `context.ts` → `getUserContext()`

Le contexte utilisateur est **prepended aux messages** (pas dans le system prompt).

### Contenu injecté

```xml
<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
Codebase and user instructions are shown below. Be sure to adhere to
these instructions. IMPORTANT: These instructions OVERRIDE any default
behavior and you MUST follow them exactly as written.

Contents of /project/CLAUDE.md (project instructions):
[contenu du CLAUDE.md projet]

Contents of ~/.claude/CLAUDE.md (user instructions):
[contenu du CLAUDE.md utilisateur]

# currentDate
Today's date is 2026-04-01.
</system-reminder>
```

### Chargement des CLAUDE.md

**Fichier** : `utils/claudemd.ts` → `getClaudeMds()`

Ordre de chargement (dernier = priorité max) :

```
1. Managed  → /etc/claude-code/CLAUDE.md
2. User     → ~/.claude/CLAUDE.md
3. Project  → CLAUDE.md, .claude/CLAUDE.md, .claude/rules/*.md
4. Local    → CLAUDE.local.md
```

Chaque fichier est labelé avec son type et son chemin :
```
Contents of /project/CLAUDE.md (project instructions, checked into the codebase):
```

### Memoization

- `getUserContext()` est memoizé une fois par session
- Invalidé sur `/clear` ou `/compact`
- Les fichiers sont relus depuis le disque après invalidation

---

## 4. Contexte système — Git et environnement

**Fichier** : `context.ts` → `getSystemContext()`

Appendé au system prompt (pas aux messages).

### Contenu

```
gitStatus: This is the git status at the start of the conversation.
Current branch: main
Main branch: main
Status:
M src/app.ts
?? new-file.ts

Recent commits:
abc1234 fix: resolve race condition in auth
def5678 feat: add user profile page
```

- Caché une fois par session
- Inclut : branche courante, branche principale, utilisateur git, status, commits récents
- Cache breaker injection possible (ant-only, debug)

### Informations d'environnement (dans le system prompt)

```
- Primary working directory: /path/to/project
  - Is a git repository: true
- Platform: darwin
- Shell: zsh
- OS Version: Darwin 22.1.0
- Model: claude-opus-4-6[1m]
- Knowledge cutoff: May 2025
- Claude Code available as: CLI, desktop app, web app, IDE extensions
```

---

## 5. Messages — Normalisation et formatage

### Pipeline de normalisation

**Fichier** : `utils/messages.ts` → `normalizeMessagesForAPI()`

Avant d'être envoyés à l'API, les messages traversent un pipeline de 12 étapes :

```
Messages internes
    │
    ▼
1.  Réordonnement des attachments (bubble-up jusqu'aux tool results)
2.  Filtrage des messages virtuels (internes REPL)
3.  Suppression des blocs ayant causé des erreurs API
4.  Normalisation des inputs d'outils (strip champs non-standard)
5.  Merge des messages user consécutifs (requis Bedrock)
6.  Suppression des tool_reference blocks (si tool search désactivé)
7.  Injection de turn boundary (évite deux messages humains consécutifs)
8.  Filtrage des messages thinking orphelins (sans contenu non-thinking)
9.  Merge final des messages user adjacents
10. Sanitization des tool results en erreur
11. Ajout des tags message ID (si HISTORY_SNIP activé)
12. Validation de la taille des images
    │
    ▼
Messages API-ready (UserMessage | AssistantMessage)
```

### Types de blocs dans les messages

| Type de bloc | Usage |
|-------------|-------|
| `TextBlockParam` | Texte brut avec `cache_control` optionnel |
| `ImageBlockParam` | Image base64 (PNG, JPEG, GIF, WebP) |
| `DocumentBlockParam` | PDF base64 |
| `ToolUseBlockParam` | Appel d'outil (name, id, input) |
| `ToolResultBlockParam` | Résultat d'outil (tool_use_id, content, is_error) |
| `ThinkingBlockParam` | Bloc de réflexion (redacted ou visible) |
| `ToolReferenceBlockParam` | Référence à un outil deferred (tool search) |

### Limite média

Maximum **100 items média** (images + PDFs) par requête. Au-delà, les items excédentaires sont supprimés.

---

## 6. Attachments — Injection de contexte

Les attachments sont des informations contextuelles converties en blocs de messages avant envoi.

**Fichier** : `utils/attachments.ts`, `utils/messages.ts` → `normalizeAttachmentForAPI()`

### Types d'attachments et leur conversion

| Type d'attachment | Conversion API | Wrapping |
|-------------------|---------------|----------|
| `file` (texte) | Tool use → Tool result (simule un Read) | Non |
| `file` (image) | Image block base64 | Non |
| `file` (PDF) | Document block base64 | Non |
| `file` (notebook) | Tool use → Tool result | Non |
| `directory` | Simule `ls` via Bash → résultat | Non |
| `edited_text_file` | Message texte avec diff | Non |
| `compact_file_reference` | Message texte (fichier trop gros) | Non |
| `pdf_reference` | Message texte avec nombre de pages | Non |
| `selected_lines_in_ide` | Message texte avec sélection | Non |
| `queued_command` | Message user avec images optionnelles | Non |
| `relevant_memories` | Contenu des fichiers mémoire | `<system-reminder>` |
| `todo_reminder` | Liste de todos | `<system-reminder>` |
| `task_reminder` | Liste de tâches | `<system-reminder>` |
| `skill_listing` | Skills disponibles | `<system-reminder>` |
| `plan_mode` | Instructions mode plan | `<system-reminder>` |
| `diagnostic` | Diagnostics IDE | `<new-diagnostics>` |
| `teammate_mailbox` | Messages d'équipe | `<system-reminder>` |

### Wrapping `<system-reminder>`

La plupart des attachments contextuels sont wrappés dans des balises système :

```xml
<system-reminder>
[contenu de l'attachment]
</system-reminder>
```

Ces balises indiquent au modèle que le contenu provient du système, pas de l'utilisateur.

---

## 7. Outils — Schémas envoyés au modèle

### Construction des schémas

**Fichier** : `utils/api.ts` → `toolToAPISchema()`

Chaque outil est converti en schéma JSON pour l'API :

```typescript
{
  name: "Bash",
  description: "Execute shell commands...",
  input_schema: {
    type: "object",
    properties: {
      command: { type: "string", description: "The command to execute" },
      timeout: { type: "number", description: "Optional timeout in ms" }
    },
    required: ["command"]
  }
}
```

### Types d'outils envoyés

| Source | Nombre | Chargement |
|--------|--------|-----------|
| Built-in | ~56 | Eager (schémas complets) |
| MCP | Variable | Eager ou deferred (`defer_loading: true`) |
| Advisor | 0-1 | Si advisor activé (type: `advisor_20260301`) |
| Deferred | Variable | Découverts via `tool_reference` blocks |

### Tool search et deferred loading

Quand le tool search est activé :
1. Les outils MCP sont marqués `defer_loading: true`
2. Seuls les outils **découverts** par le modèle (via `tool_reference`) sont déployés
3. Un préambule liste les outils disponibles en deferred
4. Le modèle peut demander à charger un outil via `ToolSearch`

### Filtrage

- Outils filtrés par deny rules (permissions)
- Outils filtrés par `isEnabled()` runtime
- ToolSearchTool retiré si tool search désactivé
- Schémas adaptés selon les capacités du modèle

---

## 8. Thinking — Configuration de la réflexion

### Modes

**Fichier** : `utils/thinking.ts`

| Mode | Paramètre API | Description |
|------|--------------|-------------|
| `adaptive` | `{ type: 'adaptive' }` | Le modèle choisit quand réfléchir |
| `enabled` | `{ type: 'enabled', budget_tokens: N }` | Réflexion avec budget fixe |
| `disabled` | Omis de la requête | Pas de thinking |

### Détermination du mode

```
1. Modèle supporte thinking ? (Opus/Sonnet)
2. CLAUDE_CODE_DISABLE_THINKING env var ?
3. CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING env var ?
4. Feature flags et settings
→ ThinkingConfig { type, budget_tokens? }
```

### Contrainte

- `budget_tokens` plafonné à `maxOutputTokens - 1`
- Si thinking activé → `temperature` **non envoyée** (incompatible)
- Si thinking désactivé → `temperature: 1` (défaut)

### Blocs thinking dans l'historique

- Les blocs thinking de l'assistant précédent sont **strippés** du dernier message (requis par l'API)
- Les messages assistant ne contenant que du thinking (orphelins) sont filtrés
- Les signatures thinking sont préservées pour la stabilité du cache

---

## 9. Prompt caching — Optimisation du cache

### Mécanisme

**Fichier** : `services/api/claude.ts` → `addCacheBreakpoints()`

Le prompt caching permet de réutiliser des préfixes de contexte déjà calculés par l'API, économisant des tokens d'entrée.

### Placement des marqueurs `cache_control`

```typescript
{
  type: "text",
  text: "...",
  cache_control: {
    type: "ephemeral"  // ou "session" (TTL 1h si éligible)
  }
}
```

**Blocs marqués** :
- Dernier bloc du system prompt
- Blocs de contenu user (messages)
- Blocs de contenu assistant (hors thinking/connector)
- Tool results dans le préfixe caché

### Scope global

Le system prompt **avant** `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` est caché avec `scope: 'global'` :
- Partagé entre toutes les sessions du même utilisateur
- Persistance jusqu'à 5 min sans activité (1h pour subscribers)
- Réduit massivement les coûts sur les sections statiques

### Cache editing (microcompact)

Quand activé (`CACHE_EDITING_BETA_HEADER`) :
- Insère des `cache_edit` après les tool_result blocks
- Ajoute des `cache_reference` aux tool_results dans le préfixe caché
- Permet de modifier le cache sans le recalculer entièrement

### Latching des betas

Les headers de beta sont **latchés** une fois activés par session :
- Évite les cache busts si un toggle mid-session change
- Reset sur `/clear` ou `/compact`

---

## 10. Paramètres API complets

### Structure de la requête

```typescript
anthropic.beta.messages.create({
  // Modèle
  model: "claude-opus-4-6",

  // System prompt (blocs avec cache_control)
  system: [
    { type: "text", text: "Attribution header...", cache_control: {...} },
    { type: "text", text: "CLI prefix..." },
    { type: "text", text: "Main system prompt...", cache_control: {...} },
    // ... sections dynamiques
  ],

  // Messages (historique de conversation)
  messages: [
    { role: "user", content: [
      { type: "text", text: "<system-reminder>CLAUDE.md content...</system-reminder>" },
      { type: "text", text: "Question de l'utilisateur" }
    ]},
    { role: "assistant", content: [
      { type: "text", text: "Réponse..." },
      { type: "tool_use", id: "toolu_xxx", name: "Bash", input: { command: "ls" } }
    ]},
    { role: "user", content: [
      { type: "tool_result", tool_use_id: "toolu_xxx", content: "file1.ts\nfile2.ts" }
    ]},
    // ... suite de la conversation
  ],

  // Outils (schémas JSON)
  tools: [
    { name: "Bash", description: "...", input_schema: {...} },
    { name: "Read", description: "...", input_schema: {...} },
    // ... 40+ outils
  ],

  // Streaming
  stream: true,

  // Tokens de sortie
  max_tokens: 16384,

  // Thinking (réflexion étendue)
  thinking: { type: "adaptive" },

  // Betas activées
  betas: [
    "context-1m-2025-08-07",
    "effort-2025-11-24",
    "fast-mode-2026-02-01",
    // ...
  ],

  // Métadonnées
  metadata: {
    user_id: {
      device_id: "uuid-device",
      account_uuid: "uuid-account",
      session_id: "uuid-session"
    }
  },

  // Température (uniquement si thinking désactivé)
  // temperature: 1,

  // Tool choice (optionnel)
  // tool_choice: { type: "auto" },

  // Context management (microcompaction)
  context_management: {
    edits: [
      { type: "clear_tool_uses_20250919", threshold_tokens: N },
      { type: "clear_thinking_20251015" }
    ]
  },

  // Output config (optionnel)
  output_config: {
    effort: "high",
    // task_budget: {...},
    // format: { type: "json_schema", ... }
  },

  // Fast mode
  speed: "fast",  // ou undefined

  // Extra body params (env var override)
  // ...extraBodyParams
})
```

---

## 11. Headers et métadonnées

### Headers HTTP envoyés

| Header | Valeur | Description |
|--------|--------|-------------|
| `x-app` | `'cli'` | Identifiant application |
| `User-Agent` | Custom | Version Claude Code, OS, etc. |
| `X-Claude-Code-Session-Id` | UUID | ID de la session courante |
| `x-client-request-id` | UUID | Corrélation requête (1P uniquement) |
| `x-anthropic-additional-protection` | `'true'` | Si protection supplémentaire activée |
| `x-claude-remote-container-id` | ID | Si session CCR |
| `x-claude-remote-session-id` | ID | Si session CCR |
| `x-client-app` | ID | Identifiant app client |
| `anthropic-beta` | Liste | Betas activées (voir section 12) |
| Custom headers | JSON | Via `ANTHROPIC_CUSTOM_HEADERS` env var |

### Métadonnées dans la requête

```typescript
metadata: {
  user_id: JSON.stringify({
    device_id: "uuid",          // Unique par device
    account_uuid: "uuid",       // UUID OAuth (si authentifié)
    session_id: "uuid",         // Session courante
    ...extraMetadata            // Via CLAUDE_CODE_EXTRA_METADATA
  })
}
```

### Configuration client

| Paramètre | Valeur |
|-----------|--------|
| `maxRetries` | 0 (retry manuel) |
| `timeout` | `API_TIMEOUT_MS` ou 600s |
| `dangerouslyAllowBrowser` | true |
| `fetch` | Override possible (proxy) |

---

## 12. Betas — Fonctionnalités activées

### Headers beta conditionnels

| Beta | Header | Condition |
|------|--------|-----------|
| **Context 1M** | `context-1m-2025-08-07` | Modèle 1M tokens |
| **Context management** | `context-management-2025-06-27` | Microcompact activé |
| **Effort** | `effort-2025-11-24` | Modèle supporte effort |
| **Task budgets** | `task-budgets-2026-03-13` | Token budget activé |
| **Structured outputs** | `structured-outputs-2025-12-15` | Format JSON schema |
| **Prompt caching scope** | (variable) | Cache global activé |
| **Tool search** | (1P ou 3P) | Tool search activé |
| **Fast mode** | `fast-mode-2026-02-01` | Fast mode actif |
| **AFK mode** | (variable) | Transcript classifier |
| **Cache editing** | (variable) | Cached microcompact |
| **Advisor** | `advisor-tool-2026-03-01` | Advisor activé |
| **Redact thinking** | `redact-thinking-2026-02-12` | Thinking redacté |

### Latching

Les headers beta sont **latchés** (`true` une fois activés) pour la durée de la session. Cela empêche les cache busts si un toggle change mid-session.

Reset sur `/clear` ou `/compact`.

---

## 13. Streaming — Réception de la réponse

### Événements du stream

**Fichier** : `services/api/claude.ts` (lignes 1940-2400+)

```
message_start          → Initialise le message, capture l'usage
    ↓
content_block_start    → Commence l'accumulation (text/tool_use/thinking)
    ↓
content_block_delta    → Accumule les deltas (text/input_json/thinking)
    ↓
content_block_stop     → Finalise le bloc, normalise, yield message
    ↓
message_delta          → Met à jour l'usage
    ↓
message_stop           → Finalise la réponse
```

### Normalisation du contenu

`normalizeContentFromAPI()` traite chaque bloc :
- Parse les inputs JSON des tool_use (gère le JSON stringifié imbriqué)
- Applique la normalisation par outil
- Parse les inputs server_tool_use
- Préserve text/thinking/connector_text tel quel

### Timeouts et détection de stall

| Paramètre | Valeur | Description |
|-----------|--------|-------------|
| Idle timeout | 90s (défaut) | Configurable via `CLAUDE_STREAM_IDLE_TIMEOUT_MS` |
| Stall detection | 30s | Entre deux chunks |
| Watchdog | Reset à chaque chunk | Timer auto-reset |

### Checkpoints de performance

| Checkpoint | Moment |
|-----------|--------|
| `query_api_streaming_start` | Avant le début du stream |
| `query_api_request_sent` | Après dispatch de la requête |
| `query_response_headers_received` | Réception des headers |
| `query_first_chunk_received` | Premier chunk de données |
| `query_api_streaming_end` | Fin du stream |

---

## 14. Sélection du modèle

### Hiérarchie de sélection

**Fichier** : `utils/model/model.ts`

```
1. Override utilisateur (settings / UI / --model flag)
2. Défaut par abonnement :
   ├── Max subscribers → Opus 4.6 [1m]
   └── Autres → Sonnet 4.6
3. Fallback provider 3P (Bedrock, Vertex, Foundry)
```

### Normalisation pour l'API

`normalizeModelStringForAPI()` :
- Supprime les suffixes `[1m]` et `[2m]`
- Résout les inference profiles sur Bedrock
- Retourne l'identifiant modèle brut pour l'API

---

## 15. Fast mode

### Conditions d'activation

```
1. Feature activée (isFastModeEnabled())
2. Disponible en quota (isFastModeAvailable())
3. Pas en cooldown (isFastModeCooldown())
4. Modèle le supporte (isFastModeSupportedByModel())
5. Explicitement demandé (options.fastMode)
```

### Paramètres envoyés

- Header beta **latché** pour la session
- Paramètre `speed: 'fast'` envoyé dynamiquement
- Même modèle (Opus 4.6) avec sortie plus rapide
- Fallback non-streaming si nécessaire

---

## 16. Context management — Microcompaction

### Mécanisme

**Fichier** : `services/compact/apiMicrocompact.ts`

Quand activé, envoie des instructions de gestion de contexte à l'API :

```typescript
context_management: {
  edits: [
    {
      type: "clear_tool_uses_20250919",
      threshold_tokens: N  // Seuil de déclenchement
    },
    {
      type: "clear_thinking_20251015"
      // Efface les blocs thinking, préserve le dernier tour
    }
  ]
}
```

### Stratégies

| Stratégie | Action |
|-----------|--------|
| `clear_tool_uses` | Efface les résultats/appels d'outils au-delà du seuil de tokens |
| `clear_thinking` | Efface les blocs de réflexion, préserve le dernier tour |

---

## 17. Pipeline complet — Du clavier à l'API

```
┌─────────────────────────────────────────────────────────────────────┐
│ SAISIE UTILISATEUR                                                   │
│ "Explique-moi comment fonctionne l'auth dans ce projet"             │
└─────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│ ASSEMBLAGE DU SYSTEM PROMPT                                          │
│                                                                      │
│ buildEffectiveSystemPrompt()                                         │
│ ├── getSystemPrompt() → sections statiques + dynamiques             │
│ │   ├── [STATIC] Intro, System, Tasks, Actions, Tools, Tone, Output│
│ │   ├── ─── SYSTEM_PROMPT_DYNAMIC_BOUNDARY ───                      │
│ │   ├── [DYNAMIC] Session guidance, Environment, Language           │
│ │   ├── [DYNAMIC] Memory (MEMORY.md + instructions)                 │
│ │   └── [DYNAMIC] MCP, Scratchpad, Token budget, Brief              │
│ └── appendSystemContext(systemPrompt, gitStatus)                     │
│                                                                      │
│ buildSystemPromptBlocks()                                            │
│ ├── Attribution header (fingerprint)                                 │
│ ├── CLI prefix                                                       │
│ ├── Sections du system prompt                                        │
│ ├── cache_control sur le dernier bloc statique (scope: global)      │
│ └── cache_control sur le dernier bloc dynamique (scope: ephemeral)  │
└─────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│ ASSEMBLAGE DU CONTEXTE UTILISATEUR                                   │
│                                                                      │
│ getUserContext()                                                      │
│ ├── getClaudeMds() → CLAUDE.md (managed → user → project → local)  │
│ └── currentDate → "Today's date is 2026-04-01."                     │
│                                                                      │
│ → Prepended aux messages comme <system-reminder>                    │
└─────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│ ATTACHMENTS → BLOCS DE MESSAGES                                      │
│                                                                      │
│ getAttachments()                                                     │
│ ├── relevant_memories → <system-reminder> (≤5 fichiers Sonnet)      │
│ ├── task_reminder → <system-reminder> (liste de tâches)             │
│ ├── skill_listing → <system-reminder> (skills disponibles)          │
│ ├── file attachments → tool_use/tool_result pairs                   │
│ └── diagnostic → <new-diagnostics> (IDE)                            │
│                                                                      │
│ normalizeAttachmentForAPI() → UserMessage[]                          │
└─────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│ NORMALISATION DES MESSAGES                                           │
│                                                                      │
│ normalizeMessagesForAPI(messages, tools)                              │
│ ├── Réordonnement attachments (bubble-up)                            │
│ ├── Filtrage messages virtuels                                       │
│ ├── Suppression blocs ayant causé des erreurs                        │
│ ├── Normalisation inputs d'outils                                    │
│ ├── Merge messages user consécutifs                                  │
│ ├── Suppression tool_reference (si tool search off)                  │
│ ├── Filtrage thinking orphelins                                      │
│ ├── Validation taille images                                         │
│ └── Limite 100 items média                                           │
└─────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│ CONSTRUCTION DES SCHÉMAS D'OUTILS                                    │
│                                                                      │
│ toolToAPISchema() pour chaque outil                                  │
│ ├── Built-in tools (~56 schémas complets)                            │
│ ├── MCP tools (eager ou defer_loading: true)                         │
│ ├── Advisor tool (si activé)                                         │
│ └── Filtrage par deny rules et isEnabled()                           │
└─────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│ CONFIGURATION FINALE                                                 │
│                                                                      │
│ ├── Thinking: { type: 'adaptive' }                                   │
│ ├── Betas: ["context-1m-...", "effort-...", "fast-mode-...", ...]    │
│ ├── Metadata: { user_id: { device_id, account_uuid, session_id } }  │
│ ├── max_tokens: 16384                                                │
│ ├── speed: "fast" (si fast mode)                                     │
│ ├── context_management: { edits: [...] }                             │
│ └── addCacheBreakpoints(messages)                                    │
└─────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│ APPEL API                                                            │
│                                                                      │
│ anthropic.beta.messages.create({                                     │
│   model, system, messages, tools, stream: true,                      │
│   max_tokens, thinking, betas, metadata,                             │
│   context_management, output_config, speed                           │
│ }, { signal, headers })                                              │
│                                                                      │
│ Headers : x-app, User-Agent, Session-Id, x-client-request-id       │
└─────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STREAMING DE LA RÉPONSE                                              │
│                                                                      │
│ for await (chunk of stream):                                         │
│   ├── message_start → init message, capture usage                    │
│   ├── content_block_start → accumulation (text/tool_use/thinking)   │
│   ├── content_block_delta → deltas (text/json/thinking)             │
│   ├── content_block_stop → normalise, yield                         │
│   ├── message_delta → update usage                                   │
│   └── message_stop → finalise                                        │
│                                                                      │
│ normalizeContentFromAPI() → AssistantMessage                         │
│ ├── Parse tool_use JSON inputs                                       │
│ ├── Normalisation par outil                                          │
│ └── Préservation text/thinking                                       │
└─────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│ POST-TRAITEMENT                                                      │
│                                                                      │
│ Si tool_use dans la réponse :                                        │
│   → StreamingToolExecutor exécute les outils                         │
│   → Résultats formatés en tool_result                                │
│   → Nouveau tour API avec l'historique enrichi                       │
│                                                                      │
│ Si fin de tour :                                                     │
│   → Stop hooks (extractMemories, autoDream, promptSuggestion)       │
│   → Transcript enregistré                                            │
│   → Affichage à l'utilisateur                                        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 18. Fichiers clés

### Construction de la requête

| Fichier | Rôle |
|---------|------|
| `services/api/claude.ts` | Assemblage complet de la requête API, streaming, retry |
| `services/api/client.ts` | Création du client Anthropic, headers, providers |
| `constants/prompts.ts` | Sections du system prompt, contenu statique et dynamique |
| `utils/systemPrompt.ts` | Construction du system prompt effectif, priorités |
| `constants/betas.ts` | Headers beta, gestion du latching |

### Contexte et mémoire

| Fichier | Rôle |
|---------|------|
| `context.ts` | getUserContext(), getSystemContext(), memoization |
| `utils/claudemd.ts` | Chargement CLAUDE.md, @includes, règles conditionnelles |
| `memdir/memdir.ts` | Prompt mémoire, MEMORY.md |
| `utils/attachments.ts` | Construction des attachments, mémoires, prefetch |

### Messages et normalisation

| Fichier | Rôle |
|---------|------|
| `utils/messages.ts` | Normalisation messages, conversion attachments, blocs contenu |
| `utils/api.ts` | toolToAPISchema(), utilitaires API |
| `query.ts` | Boucle principale, assemblage query, appel model |

### Optimisation

| Fichier | Rôle |
|---------|------|
| `services/compact/apiMicrocompact.ts` | Context management, cache editing |
| `utils/thinking.ts` | Configuration thinking/réflexion |
| `utils/model/model.ts` | Sélection et normalisation du modèle |
