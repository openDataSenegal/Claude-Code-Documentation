# Constantes — Documentation complète

Inventaire exhaustif de toutes les constantes, limites, seuils, timeouts, enums et valeurs nommées du projet Claude Code.

---

## Table des matières

1. [Limites API et modèles](#1-limites-api)
2. [Tokens et fenêtre de contexte](#2-tokens-contexte)
3. [Compaction et gestion du contexte](#3-compaction)
4. [Limites des outils](#4-limites-outils)
5. [Limites fichiers et médias](#5-limites-fichiers)
6. [Headers beta API](#6-headers-beta)
7. [OAuth et URLs](#7-oauth-urls)
8. [Noms d'outils](#8-noms-outils)
9. [Tags XML](#9-tags-xml)
10. [Timeouts réseau et API](#10-timeouts-réseau)
11. [Retry et backoff](#11-retry-backoff)
12. [Cache et TTL](#12-cache-ttl)
13. [Polling et intervalles](#13-polling)
14. [UI et affichage](#14-ui-affichage)
15. [Mémoire et historique](#15-mémoire)
16. [Voice mode](#16-voice)
17. [Sécurité et permissions](#17-sécurité)
18. [Analytics et télémétrie](#18-analytics)
19. [Enums et types unions](#19-enums)
20. [Symboles Unicode](#20-symboles)
21. [Messages d'erreur](#21-erreurs)
22. [Compilation et macros](#22-macros)
23. [Top 20 des constantes les plus impactantes](#23-top-20)
24. [Fichiers clés](#24-fichiers-clés)

---

## 1. Limites API et modèles

**Fichier** : `constants/apiLimits.ts`

### Images

| Constante | Valeur | Description |
|-----------|--------|-------------|
| `API_IMAGE_MAX_BASE64_SIZE` | 5 MB | Taille max image base64 |
| `IMAGE_TARGET_RAW_SIZE` | 3.75 MB | Cible taille brute (avant base64) |
| `IMAGE_MAX_WIDTH` | 2 000 px | Largeur max côté client |
| `IMAGE_MAX_HEIGHT` | 2 000 px | Hauteur max côté client |
| `API_MAX_MEDIA_PER_REQUEST` | 100 | Max médias (images+PDF) par requête |

### PDF

| Constante | Valeur | Description |
|-----------|--------|-------------|
| `PDF_TARGET_RAW_SIZE` | 20 MB | Taille max PDF pour envoi natif |
| `API_PDF_MAX_PAGES` | 100 | Pages max par PDF (API) |
| `PDF_EXTRACT_SIZE_THRESHOLD` | 3 MB | Seuil extraction par pages vs base64 |
| `PDF_MAX_EXTRACT_SIZE` | 100 MB | Taille max pour extraction de pages |
| `PDF_MAX_PAGES_PER_READ` | 20 | Pages max par appel Read |
| `PDF_AT_MENTION_INLINE_THRESHOLD` | 10 | Seuil inline vs référence |

---

## 2. Tokens et fenêtre de contexte

**Fichiers** : `utils/context.ts`, `services/api/claude.ts`

| Constante | Valeur | Description |
|-----------|--------|-------------|
| `MODEL_CONTEXT_WINDOW_DEFAULT` | 200 000 tokens | Fenêtre de contexte par défaut |
| `CONTEXT_1M` | 1 000 000 tokens | Fenêtre 1M (Sonnet-4, Opus-4-6) |
| `MAX_OUTPUT_TOKENS_DEFAULT` | 32 000 tokens | Output max par défaut |
| `MAX_OUTPUT_TOKENS_UPPER_LIMIT` | 64 000 tokens | Plafond absolu output |
| `CAPPED_DEFAULT_MAX_TOKENS` | 8 000 tokens | Cap par défaut (réservation slot) |
| `ESCALATED_MAX_TOKENS` | 64 000 tokens | Output escaladé sur retry |
| `MAX_NON_STREAMING_TOKENS` | 64 000 tokens | Max réponse non-streaming |
| `COMPACT_MAX_OUTPUT_TOKENS` | 20 000 tokens | Budget output pour résumés |
| `MAX_TOTAL_SESSION_MEMORY_TOKENS` | 12 000 tokens | Budget mémoire de session |
| `BYTES_PER_TOKEN` | 4 | Estimation octets/token |

---

## 3. Compaction et gestion du contexte

**Fichiers** : `services/compact/autoCompact.ts`, `services/compact/compact.ts`, `services/compact/apiMicrocompact.ts`

| Constante | Valeur | Description |
|-----------|--------|-------------|
| `AUTOCOMPACT_BUFFER_TOKENS` | 13 000 | Marge avant auto-compaction |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | 20 000 | Buffer seuil avertissement |
| `ERROR_THRESHOLD_BUFFER_TOKENS` | 20 000 | Buffer seuil erreur |
| `MANUAL_COMPACT_BUFFER_TOKENS` | 3 000 | Buffer compaction manuelle |
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20 000 | Budget output du résumé |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | Circuit breaker auto-compact |
| `MAX_PTL_RETRIES` | 3 | Retries prompt-too-long |
| `MAX_COMPACT_STREAMING_RETRIES` | 2 | Retries streaming compaction |
| `POST_COMPACT_TOKEN_BUDGET` | 50 000 | Budget restauration fichiers |
| `POST_COMPACT_MAX_FILES_TO_RESTORE` | 5 | Fichiers restaurés max |
| `POST_COMPACT_MAX_TOKENS_PER_FILE` | 5 000 | Tokens max par fichier restauré |
| `POST_COMPACT_MAX_TOKENS_PER_SKILL` | 5 000 | Tokens max par skill restauré |
| `POST_COMPACT_SKILLS_TOKEN_BUDGET` | 25 000 | Budget total skills |
| `DEFAULT_MAX_INPUT_TOKENS` (API MC) | 180 000 | Seuil context management server-side |
| `DEFAULT_TARGET_INPUT_TOKENS` (API MC) | 40 000 | Cible keep-alive server-side |

---

## 4. Limites des outils

**Fichier** : `constants/toolLimits.ts`

| Constante | Valeur | Description |
|-----------|--------|-------------|
| `DEFAULT_MAX_RESULT_SIZE_CHARS` | 50 000 | Seuil persistance résultat sur disque |
| `MAX_TOOL_RESULT_TOKENS` | 100 000 | Tokens max par résultat d'outil |
| `MAX_TOOL_RESULT_BYTES` | 400 000 | Octets max par résultat (100K × 4) |
| `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS` | 200 000 | Chars agrégés max par message |
| `TOOL_SUMMARY_MAX_LENGTH` | 50 | Troncature résumé d'outil |
| `BASH_MAX_OUTPUT_DEFAULT` | 30 000 | Sortie Bash par défaut |
| `BASH_MAX_OUTPUT_UPPER_LIMIT` | 150 000 | Sortie Bash maximale |
| `TASK_MAX_OUTPUT_DEFAULT` | 32 000 | Sortie tâche par défaut |
| `TASK_MAX_OUTPUT_UPPER_LIMIT` | 160 000 | Sortie tâche maximale |
| `MAX_TOOL_OUTPUT_BYTES` | 5 GB | Sortie persistée maximale |

### Limites de lecture fichier

**Fichier** : `tools/FileReadTool/limits.ts`

| Constante | Valeur | Description |
|-----------|--------|-------------|
| `DEFAULT_MAX_OUTPUT_TOKENS` | 25 000 | Tokens max par lecture |
| `MAX_OUTPUT_SIZE` | 256 KB | Taille fichier max avant erreur |
| `MAX_LINES_TO_READ` | 2 000 | Lignes max par appel |

### Limites d'édition fichier

| Constante | Valeur | Description |
|-----------|--------|-------------|
| `MAX_EDIT_FILE_SIZE` | 1 GB | Taille max pour édition |
| `MAX_LSP_FILE_SIZE_BYTES` | 10 MB | Taille max pour LSP |

---

## 5. Limites fichiers et médias

| Constante | Valeur | Fichier | Description |
|-----------|--------|---------|-------------|
| `MAX_FILE_SIZE_BYTES` (upload) | 500 MB | `api/filesApi.ts` | Upload fichier |
| `MAX_FILE_SIZE_BYTES` (git) | 500 MB | `utils/git.ts` | Diff git |
| `MAX_FILE_SIZE_BYTES` (sync) | 500 KB | `settingsSync/` | Sync settings |
| `MAX_FILE_SIZE_BYTES` (team mem) | 250 KB | `teamMemorySync/` | Team memory |
| `FAST_PATH_MAX_SIZE` | 10 MB | `readFileInRange.ts` | Seuil lecture rapide |
| `MAX_SCAN_BYTES` | 10 MB | `readEditContext.ts` | Scan contexte max |
| `MAX_TRANSCRIPT_READ_BYTES` | 50 MB | `sessionStorage.ts` | Lecture transcript |
| `MAX_JSONL_READ_BYTES` | 100 MB | `utils/json.ts` | Lecture JSONL |
| `MAX_IMAGE_FILE_SIZE` | 20 MB | `BashTool/utils.ts` | Image dans Bash |
| `MAX_MARKDOWN_LENGTH` | 100 000 | `WebFetchTool/utils.ts` | Markdown converti |
| `BINARY_CHECK_SIZE` | 8 192 | `constants/files.ts` | Buffer détection binaire |

---

## 6. Headers beta API

**Fichier** : `constants/betas.ts`

| Constante | Valeur | Description |
|-----------|--------|-------------|
| `CLAUDE_CODE_20250219_BETA_HEADER` | `claude-code-20250219` | Feature Claude Code |
| `INTERLEAVED_THINKING_BETA_HEADER` | `interleaved-thinking-2025-05-14` | Thinking interleaved |
| `CONTEXT_1M_BETA_HEADER` | `context-1m-2025-08-07` | Fenêtre 1M |
| `CONTEXT_MANAGEMENT_BETA_HEADER` | `context-management-2025-06-27` | Context management |
| `STRUCTURED_OUTPUTS_BETA_HEADER` | `structured-outputs-2025-12-15` | Outputs structurés |
| `WEB_SEARCH_BETA_HEADER` | `web-search-2025-03-05` | Recherche web |
| `TOOL_SEARCH_BETA_HEADER_1P` | `advanced-tool-use-2025-11-20` | Tool search (1P) |
| `TOOL_SEARCH_BETA_HEADER_3P` | `tool-search-tool-2025-10-19` | Tool search (3P) |
| `EFFORT_BETA_HEADER` | `effort-2025-11-24` | Niveau d'effort |
| `TASK_BUDGETS_BETA_HEADER` | `task-budgets-2026-03-13` | Budgets de tâches |
| `PROMPT_CACHING_SCOPE_BETA_HEADER` | `prompt-caching-scope-2026-01-05` | Scope prompt cache |
| `FAST_MODE_BETA_HEADER` | `fast-mode-2026-02-01` | Mode rapide |
| `REDACT_THINKING_BETA_HEADER` | `redact-thinking-2026-02-12` | Redact thinking |
| `TOKEN_EFFICIENT_TOOLS_BETA_HEADER` | `token-efficient-tools-2026-03-28` | Outils token-efficient |
| `ADVISOR_BETA_HEADER` | `advisor-tool-2026-03-01` | Outil advisor |
| `AFK_MODE_BETA_HEADER` | `afk-mode-2026-01-31` | Mode AFK |

---

## 7. OAuth et URLs

**Fichier** : `constants/oauth.ts`, `constants/product.ts`

### URLs de production

| Constante | Valeur |
|-----------|--------|
| `BASE_API_URL` | `https://api.anthropic.com` |
| `CLAUDE_AI_AUTHORIZE_URL` | `https://claude.com/cai/oauth/authorize` |
| `CONSOLE_AUTHORIZE_URL` | `https://platform.claude.com/oauth/authorize` |
| `TOKEN_URL` | `https://platform.claude.com/v1/oauth/token` |
| `API_KEY_URL` | `https://api.anthropic.com/api/oauth/claude_cli/create_api_key` |
| `ROLES_URL` | `https://api.anthropic.com/api/oauth/claude_cli/roles` |
| `MCP_PROXY_URL` | `https://mcp-proxy.anthropic.com` |
| `MCP_CLIENT_METADATA_URL` | `https://claude.ai/oauth/claude-code-client-metadata` |
| `PRODUCT_URL` | `https://claude.com/claude-code` |
| `CLAUDE_AI_BASE_URL` | `https://claude.ai` |
| `DATADOG_LOGS_ENDPOINT` | `https://http-intake.logs.us5.datadoghq.com/api/v2/logs` |

### Client IDs

| Environnement | Client ID |
|---------------|-----------|
| Production | `9d1c250a-e61b-44d9-88ed-5944d1962f5e` |
| Staging | `22422756-60c9-4084-8eb7-27705fd5cf9a` |

### Scopes OAuth

| Constante | Valeurs |
|-----------|---------|
| `CLAUDE_AI_INFERENCE_SCOPE` | `user:inference` |
| `CLAUDE_AI_PROFILE_SCOPE` | `user:profile` |
| `CLAUDE_AI_OAUTH_SCOPES` | `user:profile`, `user:inference`, `user:sessions:claude_code`, `user:mcp_servers`, `user:file_upload` |
| `CONSOLE_OAUTH_SCOPES` | `org:create_api_key`, `user:profile` |

---

## 8. Noms d'outils

**Fichiers** : `tools/*/constants.ts`, `constants/tools.ts`

| Constante | Valeur | Constante | Valeur |
|-----------|--------|-----------|--------|
| `BASH_TOOL_NAME` | `Bash` | `AGENT_TOOL_NAME` | `Agent` |
| `FILE_READ_TOOL_NAME` | `Read` | `SKILL_TOOL_NAME` | `Skill` |
| `FILE_WRITE_TOOL_NAME` | `Write` | `SEND_MESSAGE_TOOL_NAME` | `SendMessage` |
| `FILE_EDIT_TOOL_NAME` | `Edit` | `TEAM_CREATE_TOOL_NAME` | `TeamCreate` |
| `GLOB_TOOL_NAME` | `Glob` | `TEAM_DELETE_TOOL_NAME` | `TeamDelete` |
| `GREP_TOOL_NAME` | `Grep` | `TASK_CREATE_TOOL_NAME` | `TaskCreate` |
| `WEB_FETCH_TOOL_NAME` | `WebFetch` | `TASK_UPDATE_TOOL_NAME` | `TaskUpdate` |
| `WEB_SEARCH_TOOL_NAME` | `WebSearch` | `TASK_STOP_TOOL_NAME` | `TaskStop` |
| `NOTEBOOK_EDIT_TOOL_NAME` | `NotebookEdit` | `CRON_CREATE_TOOL_NAME` | `CronCreate` |
| `ENTER_PLAN_MODE_TOOL_NAME` | `EnterPlanMode` | `REMOTE_TRIGGER_TOOL_NAME` | `RemoteTrigger` |
| `EXIT_PLAN_MODE_TOOL_NAME` | `ExitPlanMode` | `TOOL_SEARCH_TOOL_NAME` | `ToolSearch` |
| `ENTER_WORKTREE_TOOL_NAME` | `EnterWorktree` | `LSP_TOOL_NAME` | `LSP` |
| `EXIT_WORKTREE_TOOL_NAME` | `ExitWorktree` | `BRIEF_TOOL_NAME` | `SendUserMessage` |
| `ASK_USER_QUESTION_TOOL_NAME` | `AskUserQuestion` | `TODO_WRITE_TOOL_NAME` | `TodoWrite` |
| `SLEEP_TOOL_NAME` | `Sleep` | `SYNTHETIC_OUTPUT_TOOL_NAME` | `StructuredOutput` |

### Ensembles d'outils

| Ensemble | Contenu | Usage |
|----------|---------|-------|
| `COORDINATOR_MODE_ALLOWED_TOOLS` | Agent, TaskStop, SendMessage, StructuredOutput | Outils du coordinateur |
| `ASYNC_AGENT_ALLOWED_TOOLS` | Read, WebSearch, Grep, Glob, Bash, Edit, Write, Skill, etc. | Outils des workers |
| `ALL_AGENT_DISALLOWED_TOOLS` | Outils interdits aux agents | Sécurité agents |
| `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` | TaskCreate/Get/List/Update, SendMessage, Cron | Outils teammates |

---

## 9. Tags XML

**Fichier** : `constants/xml.ts`

### Communication agents

| Constante | Valeur | Usage |
|-----------|--------|-------|
| `TASK_NOTIFICATION_TAG` | `task-notification` | Résultats des workers |
| `TASK_ID_TAG` | `task-id` | Identifiant de tâche |
| `STATUS_TAG` | `status` | Statut de la tâche |
| `SUMMARY_TAG` | `summary` | Résumé du résultat |
| `TEAMMATE_MESSAGE_TAG` | `teammate-message` | Message inter-agents |
| `CHANNEL_MESSAGE_TAG` | `channel-message` | Messages de canal |
| `FORK_BOILERPLATE_TAG` | `fork-boilerplate` | Wrapper fork |
| `FORK_DIRECTIVE_PREFIX` | `Your directive: ` | Préfixe directive fork |

### Terminal

| Constante | Valeur | Usage |
|-----------|--------|-------|
| `BASH_INPUT_TAG` | `bash-input` | Entrée terminal |
| `BASH_STDOUT_TAG` | `bash-stdout` | Sortie standard |
| `BASH_STDERR_TAG` | `bash-stderr` | Sortie erreur |

### Skills et commandes

| Constante | Valeur | Usage |
|-----------|--------|-------|
| `COMMAND_NAME_TAG` | `command-name` | Nom du skill |
| `COMMAND_MESSAGE_TAG` | `command-message` | Message du skill |
| `COMMAND_ARGS_TAG` | `command-args` | Arguments du skill |

### Sessions distantes

| Constante | Valeur | Usage |
|-----------|--------|-------|
| `ULTRAPLAN_TAG` | `ultraplan` | Sessions de planification distantes |
| `REMOTE_REVIEW_TAG` | `remote-review` | Review distante |
| `WORKTREE_TAG` | `worktree` | Indicateur worktree |

---

## 10. Timeouts réseau et API

| Constante | Valeur | Fichier | Description |
|-----------|--------|---------|-------------|
| `API_TIMEOUT_MS` | 600s (10 min) | `api/client.ts` | Timeout requête API |
| `STREAM_IDLE_TIMEOUT_MS` | 90s | `api/claude.ts` | Timeout idle streaming |
| `STALL_THRESHOLD_MS` | 30s | `api/claude.ts` | Seuil détection stall |
| `MCP_REQUEST_TIMEOUT_MS` | 60s | `mcp/client.ts` | Timeout requête MCP |
| `DEFAULT_MCP_TOOL_TIMEOUT_MS` | ~27.8 h | `mcp/client.ts` | Timeout outil MCP (quasi illimité) |
| `AUTH_REQUEST_TIMEOUT_MS` | 30s | `mcp/auth.ts` | Timeout auth MCP |
| `IDP_LOGIN_TIMEOUT_MS` | 5 min | `mcp/xaaIdpLogin.ts` | Timeout login IdP |
| `DOWNLOAD_TIMEOUT_MS` | 30s | `bridge/inboundAttachments.ts` | Download attachments |
| `FETCH_TIMEOUT_MS` | 60s | `WebFetchTool/utils.ts` | Timeout WebFetch |
| `DIFF_TIMEOUT_MS` | 5s | `utils/diff.ts` | Timeout calcul diff |
| `COMPACTION_TIMEOUT_MS` | 3 min | `hooks/useRemoteSession.ts` | Timeout compaction |
| `DEFAULT_SESSION_TIMEOUT_MS` | 24h | `bridge/types.ts` | Timeout de session |

---

## 11. Retry et backoff

| Constante | Valeur | Fichier | Description |
|-----------|--------|---------|-------------|
| `DEFAULT_MAX_RETRIES` | 10 | `api/withRetry.ts` | Retries API par défaut |
| `BASE_DELAY_MS` | 500ms | `api/withRetry.ts` | Délai base retry |
| `MAX_529_RETRIES` | 3 | `api/withRetry.ts` | Retries erreur 529 |
| `PERSISTENT_MAX_BACKOFF_MS` | 5 min | `api/withRetry.ts` | Backoff max persistant |
| `PERSISTENT_RESET_CAP_MS` | 6h | `api/withRetry.ts` | Cap reset persistant |
| `RECONNECT_BASE_DELAY_MS` | 1s | SSE transport | Délai reconnexion |
| `RECONNECT_MAX_DELAY_MS` | 30s | SSE transport | Backoff max reconnexion |
| `RECONNECT_GIVE_UP_MS` | 10 min | SSE transport | Abandon reconnexion |
| `POST_MAX_RETRIES` | 10 | SSE transport | Retries POST |
| `MAX_RETRIES` (sync) | 3 | teamMemorySync | Retries sync |
| `MAX_RETRIES` (files) | 3 | filesApi | Retries fichiers |
| `MAX_RETRIES` (session) | 10 | sessionIngress | Retries ingress |
| `MAX_LOCK_RETRIES` | 5 | mcp/auth.ts | Retries verrou OAuth |

---

## 12. Cache et TTL

| Constante | Valeur | Fichier | Description |
|-----------|--------|---------|-------------|
| `KEYCHAIN_CACHE_TTL_MS` | 30s | secureStorage | Cache Keychain macOS |
| `CACHE_TTL` (WebFetch) | 15 min | WebFetchTool | Cache fetch web |
| `CACHE_TTL` (overage) | 1h | overageCreditGrant | Cache crédits overage |
| `CACHE_TTL` (metrics) | 1h | metricsOptOut | Cache opt-out |
| `DISK_CACHE_TTL` (metrics) | 24h | metricsOptOut | Cache disque métriques |
| `CACHE_TTL` (prompt break) | 5min / 1h | promptCacheBreak | Détection cache break |
| `CACHE_TTL` (install counts) | 24h | installCounts | Cache compteurs plugins |
| `GROVE_CACHE_EXPIRATION_MS` | 24h | grove.ts | Cache Grove settings |
| `MCP_AUTH_CACHE_TTL_MS` | 15 min | mcp/client.ts | Cache auth MCP |
| `MCP_FETCH_CACHE_SIZE` | 20 entrées | mcp/client.ts | Taille cache fetch MCP |
| `MAX_CACHE_SIZE_BYTES` (WebFetch) | 50 MB | WebFetchTool | Taille cache web |
| `READ_FILE_STATE_CACHE_SIZE` | 100 entrées | fileStateCache | Cache état fichiers |
| `DEFAULT_MAX_CACHE_SIZE_BYTES` | 25 MB | fileStateCache | Taille max cache fichiers |

---

## 13. Polling et intervalles

| Constante | Valeur | Fichier | Description |
|-----------|--------|---------|-------------|
| `POLL_INTERVAL_MS` (swarm) | 500ms | inProcessRunner | Polling mailbox |
| `PERMISSION_POLL_INTERVAL_MS` | 500ms | inProcessRunner | Polling permissions |
| `INBOX_POLL_INTERVAL_MS` | 1s | useInboxPoller | Polling inbox |
| `POLL_INTERVAL_MS` (tasks) | 1s | task/framework | Polling tâches |
| `POLL_INTERVAL_MS` (bridge idle) | 2s | pollConfigDefaults | Poll travail disponible |
| `POLL_INTERVAL_MS` (bridge busy) | 10 min | pollConfigDefaults | Poll keepalive |
| `POLL_INTERVAL_MS` (CCR) | 3s | ultraplan/ccrSession | Polling session CCR |
| `POLL_INTERVAL_MS` (PR status) | 60s | usePrStatus | Polling statut PR |
| `SESSION_SCAN_INTERVAL_MS` | 10 min | autoDream | Scan sessions Dream |
| `POLLING_INTERVAL_MS` (policy) | 1h | policyLimits | Polling policies admin |
| `POLLING_INTERVAL_MS` (remote settings) | 1h | remoteManagedSettings | Polling settings distants |
| `MDM_POLL_INTERVAL_MS` | 30 min | settings/changeDetector | Polling MDM |
| `SESSION_ACTIVITY_INTERVAL_MS` | 30s | sessionActivity | Tracking activité |

---

## 14. UI et affichage

| Constante | Valeur | Fichier | Description |
|-----------|--------|---------|-------------|
| `FRAME_INTERVAL_MS` | 16ms | `ink/constants.ts` | Rendu terminal (~60fps) |
| `DEFAULT_TIMEOUT_MS` | 8s | notifications | Notification timeout |
| `TOOLTIP_DISPLAY_DURATION_MS` | 5s | useShowFastIconHint | Durée tooltip |
| `MAX_COMMAND_DISPLAY_LINES` | 2 | BashTool/UI | Lignes commande affichées |
| `MAX_COMMAND_DISPLAY_CHARS` | 160 | BashTool/UI | Chars commande affichés |
| `MAX_LINES_TO_RENDER` | 10 | FileWriteTool/UI | Lignes diff affichées |
| `MAX_LINES_PER_FILE` | 400 | useDiffData | Lignes par fichier diff |
| `MAX_DISPLAY_CHARS` | 10 000 | UserPromptMessage | Chars prompt affichés |
| `TRUNCATE_HEAD_CHARS` | 2 500 | UserPromptMessage | Troncature début |
| `TRUNCATE_TAIL_CHARS` | 2 500 | UserPromptMessage | Troncature fin |
| `MIN_COLS_FOR_FULL_SPRITE` | 100 | CompanionSprite | Colonnes min pour buddy |
| `DEFAULT_CHAR_BUDGET` | 8 000 | SkillTool/prompt | Budget chars skill listing |
| `MAX_LISTING_DESC_CHARS` | 250 | SkillTool/prompt | Chars max description skill |

---

## 15. Mémoire et historique

| Constante | Valeur | Fichier | Description |
|-----------|--------|---------|-------------|
| `MAX_ENTRYPOINT_LINES` | 200 | memdir/memdir.ts | Lignes max MEMORY.md |
| `MAX_ENTRYPOINT_BYTES` | 25 000 | memdir/memdir.ts | Octets max MEMORY.md |
| `MAX_MEMORY_CHARACTER_COUNT` | 40 000 | utils/claudemd.ts | Chars max fichier mémoire |
| `MAX_MEMORY_FILES` | 200 | memdir/memoryScan.ts | Fichiers mémoire max scannés |
| `FRONTMATTER_MAX_LINES` | 30 | memdir/memoryScan.ts | Lignes frontmatter max |
| `MAX_INCLUDE_DEPTH` | 5 | utils/claudemd.ts | Profondeur @include max |
| `MAX_HISTORY_ITEMS` | 100 | history.ts | Entrées historique session |
| `HOLDER_STALE_MS` | 1h | consolidationLock.ts | Timeout verrou Dream |
| `TEAMMATE_MESSAGES_UI_CAP` | 50 | InProcessTeammateTask | Messages teammate affichés |

---

## 16. Voice mode

| Constante | Valeur | Fichier | Description |
|-----------|--------|---------|-------------|
| `RECORDING_SAMPLE_RATE` | 16 000 Hz | services/voice.ts | Fréquence d'échantillonnage |
| `SILENCE_DURATION_SECS` | 2.0s | services/voice.ts | Détection silence |
| `SILENCE_THRESHOLD` | 3% | services/voice.ts | Seuil amplitude silence |
| `KEEPALIVE_INTERVAL_MS` | 8s | voiceStreamSTT.ts | Keep-alive WebSocket |
| `FOCUS_SILENCE_TIMEOUT_MS` | 5s | useVoice.ts | Timeout silence focus mode |
| `MAX_KEYTERMS` | 50 | voiceKeyterms.ts | Mots-clés STT max |
| `HOLD_THRESHOLD` | 5 presses | useVoiceIntegration | Seuil activation hold |
| `RAPID_KEY_GAP_MS` | 120ms | useVoiceIntegration | Gap max "rapide" |
| `RELEASE_TIMEOUT_MS` | 200ms | useVoiceIntegration | Gap détection relâchement |

---

## 17. Sécurité et permissions

| Constante | Valeur | Fichier | Description |
|-----------|--------|---------|-------------|
| `MAX_SUBCOMMANDS_FOR_SECURITY_CHECK` | 50 | bashPermissions.ts | Sous-commandes vérifiées |
| `MAX_SUGGESTED_RULES_FOR_COMPOUND` | 5 | bashPermissions.ts | Règles suggérées max |
| `SECURITY_STDIN_LINE_LIMIT` | 4 032 | macOsKeychainStorage | Limite stdin Keychain |
| `maxConsecutive` (denial tracking) | 3 | denialTracking.ts | Refus consécutifs max |
| `maxTotal` (denial tracking) | 20 | denialTracking.ts | Refus total session max |

---

## 18. Analytics et télémétrie

| Constante | Valeur | Fichier | Description |
|-----------|--------|---------|-------------|
| `DEFAULT_FLUSH_INTERVAL_MS` | 15s | datadog.ts | Intervalle flush Datadog |
| `MAX_BATCH_SIZE` | 100 | datadog.ts | Taille batch Datadog |
| `NETWORK_TIMEOUT_MS` | 5s | datadog.ts | Timeout réseau Datadog |
| `DEFAULT_LOGS_EXPORT_INTERVAL_MS` | 10s | firstPartyEventLogger | Export logs 1P |
| `DEFAULT_MAX_EXPORT_BATCH_SIZE` | 200 | firstPartyEventLogger | Batch export 1P |
| `DEFAULT_MAX_QUEUE_SIZE` | 8 192 | firstPartyEventLogger | Taille queue analytics |
| `NUM_USER_BUCKETS` | 30 | datadog.ts | Buckets hashing utilisateur |
| `TOOL_INPUT_STRING_TRUNCATE_AT` | 512 | metadata.ts | Troncature input outil |
| `TOOL_INPUT_STRING_TRUNCATE_TO` | 128 | metadata.ts | Cible troncature |
| `TOOL_INPUT_MAX_JSON_CHARS` | 4 096 | metadata.ts | Chars JSON max |
| `TOOL_INPUT_MAX_DEPTH` | 2 | metadata.ts | Profondeur objet max |
| `MAX_CONTENT_HASH_SIZE` | 100 KB | fileOperationAnalytics | Taille hash max |
| `DATADOG_CLIENT_TOKEN` | `pubbbf48e6d...` | datadog.ts | Token client Datadog |

### Clés GrowthBook SDK

| Environnement | Clé |
|---------------|-----|
| Public (externe) | `sdk-zAZezfDKGoZuXXKe` |
| Production (Ant) | `sdk-xRVcrliHIlrg4og4` |
| Développement (Ant) | `sdk-yZQvlplybuXjYh6L` |

---

## 19. Enums et types unions

### Modes de permission

```typescript
'acceptEdits' | 'bypassPermissions' | 'default' | 'dontAsk' | 'plan' | 'auto' | 'bubble'
```

### Comportements de permission

```typescript
'allow' | 'deny' | 'ask'
```

### Couleurs d'agent

```typescript
'red' | 'blue' | 'green' | 'yellow' | 'purple' | 'orange' | 'pink' | 'cyan'
```

### Thèmes

```typescript
'dark' | 'light' | 'light-daltonized' | 'dark-daltonized' | 'light-ansi' | 'dark-ansi' | 'auto'
```

### Plateformes

```typescript
'macos' | 'windows' | 'wsl' | 'linux' | 'unknown'
```

### États de session

```typescript
'idle' | 'running' | 'requires_action'
```

### Modes Vim

```typescript
'INSERT' | 'NORMAL'
```

### Priorité de queue

```typescript
'now' | 'next' | 'later'
```

### Niveaux d'effort

```typescript
'low' | 'medium' | 'high' | 'max' | number
```

### Sources de settings

```typescript
'userSettings' | 'projectSettings' | 'localSettings' | 'flagSettings' | 'policySettings'
```

### Sources de chargement skills

```typescript
'commands_DEPRECATED' | 'skills' | 'plugin' | 'managed' | 'bundled' | 'mcp'
```

### Buddy — Raretés et espèces

```typescript
// Raretés
'common' | 'uncommon' | 'rare' | 'epic' | 'legendary'

// Espèces (18)
'duck' | 'goose' | 'blob' | 'cat' | 'dragon' | 'octopus' | 'owl' | 'penguin' |
'turtle' | 'snail' | 'ghost' | 'axolotl' | 'capybara' | 'cactus' | 'robot' |
'rabbit' | 'mushroom' | 'chonk'

// Stats
'DEBUGGING' | 'PATIENCE' | 'CHAOS' | 'WISDOM' | 'SNARK'
```

---

## 20. Symboles Unicode

**Fichier** : `constants/figures.ts`

| Constante | Symbole | Usage |
|-----------|---------|-------|
| `BLACK_CIRCLE` | ⏺ / ● | Indicateur plateforme |
| `BULLET_OPERATOR` | ∙ | Puce de liste |
| `TEARDROP_ASTERISK` | ✻ | Emphase |
| `UP_ARROW` / `DOWN_ARROW` | ↑ / ↓ | Merge notice / scroll hint |
| `LIGHTNING_BOLT` | ↯ | Fast mode |
| `EFFORT_LOW/MED/HIGH/MAX` | ○ / ◐ / ● / ◉ | Niveaux d'effort |
| `PLAY_ICON` / `PAUSE_ICON` | ▶ / ⏸ | Média |
| `DIAMOND_OPEN/FILLED` | ◇ / ◆ | Running / completed |
| `FLAG_ICON` | ⚑ | Issue flag |
| `FORK_GLYPH` | ⑂ | Directive fork |
| `BLOCKQUOTE_BAR` | ▎ | Préfixe citation |
| `REFERENCE_MARK` | ※ | Summary recap |

---

## 21. Messages d'erreur

**Fichier** : `services/api/errors.ts`

| Constante | Message |
|-----------|---------|
| `API_ERROR_MESSAGE_PREFIX` | `API Error` |
| `PROMPT_TOO_LONG_ERROR_MESSAGE` | `Prompt is too long` |
| `CREDIT_BALANCE_TOO_LOW_ERROR_MESSAGE` | `Credit balance is too low` |
| `INVALID_API_KEY_ERROR_MESSAGE` | `Not logged in · Please run /login` |
| `TOKEN_REVOKED_ERROR_MESSAGE` | `OAuth token revoked · Please run /login` |
| `CCR_AUTH_ERROR_MESSAGE` | `Authentication error · temporary network issue` |
| `REPEATED_529_ERROR_MESSAGE` | `Repeated 529 Overloaded errors` |
| `API_TIMEOUT_ERROR_MESSAGE` | `Request timed out` |

---

## 22. Compilation et macros

| Macro | Usage |
|-------|-------|
| `MACRO.VERSION` | Version du build (string) |
| `MACRO.VERSION_CHANGELOG` | Notes de version |
| `MACRO.BUILD_DATE` | Date de build |

Injectées au compile-time par Bun. Non définies sans la configuration de build originale.

---

## 23. Top 20 des constantes les plus impactantes

| # | Constante | Valeur | Impact |
|---|-----------|--------|--------|
| 1 | `MODEL_CONTEXT_WINDOW_DEFAULT` | 200K | Plafond absolu du contexte |
| 2 | `AUTOCOMPACT_BUFFER_TOKENS` | 13K | Déclenche la compaction auto |
| 3 | `MAX_OUTPUT_TOKENS_DEFAULT` | 32K | Budget output par requête |
| 4 | `DEFAULT_MAX_RESULT_SIZE_CHARS` | 50K | Seuil persistance outil |
| 5 | `POST_COMPACT_TOKEN_BUDGET` | 50K | Budget restauration post-compact |
| 6 | `BASH_MAX_OUTPUT_DEFAULT` | 30K | Limite sortie Bash |
| 7 | `DEFAULT_MAX_RETRIES` | 10 | Retries API |
| 8 | `API_TIMEOUT_MS` | 600s | Timeout API |
| 9 | `STREAM_IDLE_TIMEOUT_MS` | 90s | Timeout idle streaming |
| 10 | `DEFAULT_SESSION_TIMEOUT_MS` | 24h | Durée de vie session |
| 11 | `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | Circuit breaker |
| 12 | `RECONNECT_GIVE_UP_MS` | 10 min | Abandon reconnexion |
| 13 | `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS` | 200K | Chars agrégés par message |
| 14 | `MAX_ENTRYPOINT_LINES` | 200 | Taille index MEMORY.md |
| 15 | `POLL_INTERVAL_MS` (swarm) | 500ms | Réactivité multi-agents |
| 16 | `KEYCHAIN_CACHE_TTL_MS` | 30s | Fraîcheur credentials |
| 17 | `CAPPED_DEFAULT_MAX_TOKENS` | 8K | Cap output (réservation slot) |
| 18 | `MAX_FILE_SIZE_BYTES` (upload) | 500 MB | Limite upload fichier |
| 19 | `DEFAULT_MCP_TOOL_TIMEOUT_MS` | ~27.8h | Timeout outils MCP (quasi illimité) |
| 20 | `RECORDING_SAMPLE_RATE` | 16 kHz | Qualité audio voice |

---

## 24. Fichiers clés

| Fichier | Contenu |
|---------|---------|
| `constants/apiLimits.ts` | Limites API (images, PDF, médias) |
| `constants/betas.ts` | Headers beta (16+) |
| `constants/tools.ts` | Noms d'outils, ensembles de sécurité |
| `constants/toolLimits.ts` | Limites résultats d'outils |
| `constants/oauth.ts` | URLs OAuth, client IDs, scopes |
| `constants/xml.ts` | Tags XML (30+) |
| `constants/figures.ts` | Symboles Unicode (20+) |
| `constants/product.ts` | URLs produit |
| `constants/files.ts` | Extensions binaires, détection |
| `utils/context.ts` | Fenêtre de contexte, tokens |
| `tools/FileReadTool/limits.ts` | Limites lecture fichier |
| `services/compact/autoCompact.ts` | Seuils compaction |
| `services/compact/compact.ts` | Budgets restauration |
| `services/api/withRetry.ts` | Retry et backoff |
| `services/api/errors.ts` | Messages d'erreur |
| `services/analytics/datadog.ts` | Constantes analytics |
| `utils/settings/constants.ts` | Sources de settings |
