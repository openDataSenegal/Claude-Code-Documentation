# Système de Hooks — Documentation complète

L'automatisation événementielle de Claude Code : 27 événements, 4 stratégies d'exécution, intégration permissions.

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Événements disponibles (27)](#2-événements)
3. [Stratégies d'exécution (4)](#3-stratégies)
4. [Format d'entrée (input)](#4-format-entrée)
5. [Format de sortie (output)](#5-format-sortie)
6. [Intégration permissions](#6-permissions)
7. [Hooks asynchrones](#7-hooks-asynchrones)
8. [Configuration dans settings.json](#8-configuration)
9. [Matchers et filtrage](#9-matchers)
10. [Hooks de session (runtime)](#10-hooks-session)
11. [Timeouts](#11-timeouts)
12. [Trust et sécurité](#12-trust)
13. [Variables d'environnement](#13-variables)
14. [Diagnostics et télémétrie](#14-diagnostics)
15. [Fichiers clés](#15-fichiers-clés)

---

## 1. Vue d'ensemble

Les hooks permettent d'exécuter du code externe à des moments clés du cycle de vie de Claude Code. Ils peuvent **observer**, **modifier** ou **bloquer** le comportement de l'agent.

```
Événement (ex: PreToolUse)
    │
    ▼
Matching (filtre par nom d'outil / pattern)
    │
    ▼
Vérification trust (workspace trust requis)
    │
    ▼
Exécution parallèle (tous les hooks matchés)
    │         │         │         │
    ▼         ▼         ▼         ▼
  Command   Prompt    Agent     HTTP
  (shell)   (LLM)    (multi-   (REST)
                      turn)
    │         │         │         │
    └────────┬──────────┘─────────┘
             ▼
    Agrégation des résultats
    (deny > ask > allow)
             │
             ▼
    Décision finale
```

---

## 2. Événements disponibles

### Cycle de vie des outils

| Événement | Quand | Peut bloquer |
|-----------|-------|-------------|
| `PreToolUse` | Avant l'exécution d'un outil | Oui (deny/ask) |
| `PostToolUse` | Après exécution réussie | Non |
| `PostToolUseFailure` | Après échec d'un outil | Non |

### Cycle de vie de la session

| Événement | Quand | Peut bloquer |
|-----------|-------|-------------|
| `SessionStart` | Démarrage de session | Non |
| `SessionEnd` | Fin de session | Non |
| `Setup` | Initialisation / maintenance | Non |
| `Stop` | Évaluation de fin de tour | Oui |
| `StopFailure` | Échec du stop hook | Non |

### Interaction utilisateur

| Événement | Quand | Peut bloquer |
|-----------|-------|-------------|
| `UserPromptSubmit` | L'utilisateur envoie un message | Oui (modifier le prompt) |
| `Notification` | Notification système | Non |

### Permissions

| Événement | Quand | Peut bloquer |
|-----------|-------|-------------|
| `PermissionRequest` | Décision de permission en attente | Oui (allow/deny) |
| `PermissionDenied` | Après refus de permission | Non (retry possible) |

### Sous-agents

| Événement | Quand | Peut bloquer |
|-----------|-------|-------------|
| `SubagentStart` | Initialisation d'un sous-agent | Non |
| `SubagentStop` | Fin d'un sous-agent | Non |

### Compaction

| Événement | Quand | Peut bloquer |
|-----------|-------|-------------|
| `PreCompact` | Avant la compaction du transcript | Non |
| `PostCompact` | Après compaction | Non |

### Tâches

| Événement | Quand | Peut bloquer |
|-----------|-------|-------------|
| `TaskCreated` | Création d'une tâche | Non |
| `TaskCompleted` | Achèvement d'une tâche | Non |
| `TeammateIdle` | Teammate en attente | Non |

### MCP / Elicitation

| Événement | Quand | Peut bloquer |
|-----------|-------|-------------|
| `Elicitation` | Requête d'elicitation MCP | Oui (accept/decline/cancel) |
| `ElicitationResult` | Réponse d'elicitation | Non |

### Fichiers et configuration

| Événement | Quand | Peut bloquer |
|-----------|-------|-------------|
| `ConfigChange` | Modification des settings | Non |
| `CwdChanged` | Changement de répertoire de travail | Non |
| `FileChanged` | Modification du filesystem | Non |
| `InstructionsLoaded` | Chargement d'un fichier d'instructions | Non |

### Worktrees

| Événement | Quand | Peut bloquer |
|-----------|-------|-------------|
| `WorktreeCreate` | Création d'un git worktree | Non |
| `WorktreeRemove` | Suppression d'un worktree | Non |

---

## 3. Stratégies d'exécution

### 3.1 Command (Shell)

Exécute un script shell avec JSON sur stdin et stdout.

```yaml
hooks:
  - if: PreToolUse
    matcher: Bash
    command: bash ~/hooks/security-check.sh
    timeout: 30
    shell: bash     # bash (défaut) ou powershell
```

**Flux d'exécution** :
```
1. Sérialise HookInput en JSON
2. Spawn subprocess : /bin/sh -c "command"
3. Écrit JSON sur stdin
4. Lit stdout pour la réponse JSON
5. Interprète le code de sortie :
   ├── 0 → Succès (non-bloquant)
   ├── 1 → Erreur non-bloquante (loguée)
   └── 2 → Erreur bloquante (arrête la continuation)
```

**Plateformes** :
- **Unix** : `/bin/sh` ou bash explicite
- **Windows (Bash)** : Git Bash, chemins convertis en POSIX
- **Windows (PowerShell)** : `-NoProfile -NonInteractive -Command`

### 3.2 Prompt (LLM)

Évalue une condition via un modèle rapide (Haiku).

```yaml
hooks:
  - if: PreToolUse
    matcher: "Bash(rm:*)"
    prompt: "Is this deletion safe? $ARGUMENTS"
    model: claude-haiku
    timeout: 5
```

**Flux** :
```
1. Remplace $ARGUMENTS dans le prompt
2. Appel API vers Haiku avec system prompt structuré
3. Attend une réponse JSON :
   ├── { "ok": true }  → succès
   └── { "ok": false, "reason": "..." } → bloquant
```

### 3.3 Agent (Multi-turn LLM)

Lance un agent complet avec accès aux outils pour évaluer une condition.

```yaml
hooks:
  - if: Stop
    matcher: "*"
    agent:
      prompt: "Is the task complete? $ARGUMENTS"
    timeout: 30
```

**Flux** :
```
1. Crée un agent temporaire avec ID unique
2. Filtre les outils (pas d'Agent, pas de PlanMode)
3. Remplace StructuredOutput par le schema ok/reason
4. Max 50 tours avant timeout
5. Résultat :
   ├── ok: true → succès
   └── ok: false, reason: "..." → bloquant
```

**Outils** : Tous sauf Agent, PlanMode, StructuredOutput (remplacé). Mode : `dontAsk`.

### 3.4 HTTP (REST endpoint)

POST vers un endpoint externe.

```yaml
hooks:
  - if: PostToolUse
    matcher: Bash
    url: https://hooks.example.com/validate
    headers:
      Authorization: "Bearer $API_TOKEN"
    allowedEnvVars: [API_TOKEN]
    timeout: 60
```

**Flux** :
```
1. Résout les $VAR dans les headers (allowlist uniquement)
2. Vérifie l'URL contre allowedHttpHookUrls
3. Guard SSRF (bloque IPs privées/link-local)
4. POST JSON sur l'URL
5. Parse la réponse :
   ├── HTTP 200-299 → succès
   └── Autre → erreur
```

**Sécurité** :
- URL allowlist : `settings.allowedHttpHookUrls` (patterns avec `*`)
- Variables d'environnement : uniquement celles dans `allowedEnvVars`
- Injection CR/LF/NUL : strippée des headers
- SSRF : `ssrfGuardedLookup()` bloque les IPs privées
- Sandbox : routé via proxy sandbox si sandboxing activé

---

## 4. Format d'entrée

### Base commune (tous les événements)

```json
{
  "session_id": "uuid-session",
  "transcript_path": "/path/to/session.jsonl",
  "cwd": "/path/to/project",
  "permission_mode": "default",
  "agent_id": "researcher@team",
  "agent_type": "Explore"
}
```

### Événements spécifiques

**PreToolUse** :
```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "rm -rf /tmp/old" },
  "tool_use_id": "toolu_xxx"
}
```

**PostToolUse** :
```json
{
  "hook_event_name": "PostToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "npm test" },
  "tool_response": { "exitCode": 0, "stdout": "..." },
  "tool_use_id": "toolu_xxx"
}
```

**PostToolUseFailure** :
```json
{
  "hook_event_name": "PostToolUseFailure",
  "tool_name": "Bash",
  "tool_input": { "command": "npm build" },
  "tool_use_id": "toolu_xxx",
  "error": "exit code 1",
  "is_interrupt": false
}
```

**UserPromptSubmit** :
```json
{
  "hook_event_name": "UserPromptSubmit",
  "prompt": "Déploie en staging"
}
```

**SessionStart** :
```json
{
  "hook_event_name": "SessionStart",
  "source": "startup",
  "agent_type": "general-purpose",
  "model": "claude-opus-4-6"
}
```

**PermissionRequest** :
```json
{
  "hook_event_name": "PermissionRequest",
  "tool_name": "Bash",
  "tool_input": { "command": "docker build ." },
  "permission_suggestions": [
    { "type": "addRules", "rules": [{"toolName": "Bash", "ruleContent": "docker:*"}], "behavior": "allow" }
  ]
}
```

**Elicitation** :
```json
{
  "hook_event_name": "Elicitation",
  "mcp_server_name": "slack",
  "message": "Please authenticate",
  "mode": "url",
  "url": "https://slack.com/oauth/authorize?..."
}
```

---

## 5. Format de sortie

### Réponse synchrone

```json
{
  "continue": true,
  "suppressOutput": false,
  "stopReason": "Blocked because...",
  "systemMessage": "Warning to user",

  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",

    "permissionDecision": "allow",
    "permissionDecisionReason": "Safe command",
    "updatedInput": { "command": "rm -rf /tmp/old --dry-run" },
    "additionalContext": "This command was verified safe"
  }
}
```

### Champs communs

| Champ | Type | Description |
|-------|------|-------------|
| `continue` | boolean | `false` arrête l'exécution |
| `suppressOutput` | boolean | Masque stdout du transcript |
| `stopReason` | string | Message affiché quand `continue: false` |
| `systemMessage` | string | Avertissement à l'utilisateur |

### Champs spécifiques par événement

**PreToolUse** :

| Champ | Type | Description |
|-------|------|-------------|
| `permissionDecision` | `allow`/`deny`/`ask` | Décision de permission |
| `permissionDecisionReason` | string | Explication |
| `updatedInput` | object | Input modifié de l'outil |
| `additionalContext` | string | Contexte injecté comme message système |

**PostToolUse** :

| Champ | Type | Description |
|-------|------|-------------|
| `additionalContext` | string | Contexte injecté |
| `updatedMCPToolOutput` | unknown | Remplace la sortie MCP |

**UserPromptSubmit / SessionStart** :

| Champ | Type | Description |
|-------|------|-------------|
| `additionalContext` | string | Contexte système injecté |
| `initialUserMessage` | string | (SessionStart) Prompt initial |
| `watchPaths` | string[] | Chemins à surveiller pour FileChanged |

**PermissionRequest** :

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow",
      "updatedInput": { "command": "..." },
      "updatedPermissions": [
        {
          "type": "addRules",
          "rules": [{"toolName": "Bash", "ruleContent": "docker:*"}],
          "behavior": "allow",
          "destination": "session"
        }
      ]
    }
  }
}
```

**Elicitation** :

| Champ | Type | Description |
|-------|------|-------------|
| `action` | `accept`/`decline`/`cancel` | Réponse à l'elicitation |
| `content` | object | Contenu de la réponse |

### Réponse asynchrone

```json
{
  "async": true,
  "asyncTimeout": 15000
}
```

Émis en **première ligne** de stdout. Le hook passe en background, le résultat est délivré plus tard via message queue.

---

## 6. Intégration permissions

### PreToolUse → Modification de permission

Les hooks PreToolUse peuvent **modifier** la décision de permission :

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow"
  }
}
```

### Précédence avec plusieurs hooks

```
deny > ask > allow > (non défini)
```

- Si **un** hook retourne `deny` → DENY (toujours)
- Si **un** hook retourne `ask` et aucun `deny` → ASK
- Si **tous** retournent `allow` → ALLOW
- Si aucun ne décide → passthrough vers le pipeline normal

### Modification d'input

Les hooks peuvent modifier les arguments de l'outil **sans prendre de décision** :

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "updatedInput": { "command": "ls -la --color=never" }
  }
}
```

Fonctionne avec `allow` et `ask`. Les modifications sont loguées avec les noms de clés.

### PermissionRequest → Décision complète

Les hooks PermissionRequest peuvent prendre une décision avec mise à jour des règles :

```json
{
  "hookSpecificOutput": {
    "decision": {
      "behavior": "allow",
      "updatedPermissions": [
        {
          "type": "addRules",
          "rules": [{"toolName": "Bash", "ruleContent": "docker:*"}],
          "behavior": "allow",
          "destination": "session"
        }
      ]
    }
  }
}
```

---

## 7. Hooks asynchrones

### Protocole

1. Le hook émet `{"async": true, "asyncTimeout": 15000}` comme **première ligne** stdout
2. Immédiatement backgroundé via `AsyncHookRegistry`
3. Le thread parent continue sans attendre
4. Le résultat est délivré plus tard via message queue

### Événements supportés

Seuls les hooks `PreToolUse`, `PostToolUse`, et `Stop` supportent le mode async.

### AsyncHookRegistry

```typescript
PendingAsyncHook {
  processId: string,
  hookId: string,
  hookEvent: HookEvent,
  startTime: number,
  timeout: number,
  responseAttachmentSent: boolean,
}
```

| Fonction | Description |
|----------|-------------|
| `registerPendingAsyncHook()` | Enregistre le hook backgroundé |
| `checkForAsyncHookResponses()` | Poll les résultats (fréquent) |
| `removeDeliveredAsyncHooks()` | Nettoie les résultats livrés |
| `finalizePendingAsyncHooks()` | Kill les hooks restants au shutdown |

### Progression

`startHookProgressInterval()` émet un message de progression toutes les 500ms.

---

## 8. Configuration dans settings.json

### Exemple complet

```json
{
  "hooks": [
    {
      "if": "PreToolUse",
      "matcher": "Bash|Write|Edit",
      "command": "bash ~/hooks/security-check.sh",
      "timeout": 30,
      "shell": "bash"
    },
    {
      "if": "UserPromptSubmit",
      "matcher": "*",
      "prompt": "Check for PII in: $ARGUMENTS",
      "model": "claude-haiku",
      "timeout": 5
    },
    {
      "if": "Stop",
      "matcher": "*",
      "agent": {
        "prompt": "Is the task complete? Verify with tests."
      },
      "timeout": 30
    },
    {
      "if": "PostToolUse",
      "matcher": "Bash",
      "url": "https://hooks.example.com/validate",
      "headers": { "Authorization": "Bearer $API_TOKEN" },
      "allowedEnvVars": ["API_TOKEN"],
      "timeout": 60
    },
    {
      "if": "SessionStart",
      "matcher": "*",
      "command": "echo '{\"hookSpecificOutput\":{\"watchPaths\":[\"/src/**/*.ts\"]}}'",
      "timeout": 5
    }
  ]
}
```

### Champs de configuration par hook

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `if` | string | Oui | Événement déclencheur |
| `matcher` | string | Oui | Pattern de matching |
| `command` | string | * | Script shell à exécuter |
| `prompt` | string | * | Prompt LLM (Haiku) |
| `agent` | object | * | Configuration agent multi-turn |
| `url` | string | * | Endpoint HTTP |
| `timeout` | number | Non | Timeout en secondes |
| `shell` | string | Non | `bash` ou `powershell` |
| `async` | boolean | Non | Exécution en background |
| `asyncRewake` | boolean | Non | Réveiller l'agent après async |
| `headers` | object | Non | Headers HTTP (avec $VAR) |
| `allowedEnvVars` | string[] | Non | Variables autorisées dans headers |
| `model` | string | Non | Modèle pour prompt hooks |

\* Un seul parmi `command`, `prompt`, `agent`, `url` requis.

---

## 9. Matchers et filtrage

### Types de patterns

| Pattern | Exemple | Matching |
|---------|---------|---------|
| Exact | `Write` | Outil Write uniquement |
| Pipe-séparé | `Write\|Edit` | Write OU Edit |
| Regex | `^Write.*` | Commence par Write |
| Wildcard | `.*` ou `*` | Tous les outils |

### Condition `if` supplémentaire

```yaml
- if: PreToolUse
  matcher: Bash
  command: ...
  # Condition optionnelle :
  if_condition: "Bash(rm:*)"  # Seulement pour rm
```

### Normalisation

Les noms d'outils legacy (`Task`, `Command`) sont normalisés vers les noms modernes.

---

## 10. Hooks de session (runtime)

Des hooks peuvent être ajoutés dynamiquement pendant la session.

### Hooks commande (persistés)

```typescript
addSessionHook(setAppState, sessionId, event, matcher, hook, onHookSuccess?, skillRoot?)
removeSessionHook(setAppState, sessionId, event, hook)
```

### Hooks fonction (in-memory)

```typescript
addFunctionHook(setAppState, sessionId, event, matcher, callback, errorMessage, { timeout?, id? })
removeFunctionHook(setAppState, sessionId, event, hookId)
clearSessionHooks(setAppState, sessionId)
```

**Signature du callback** :
```typescript
(messages: Message[], signal?: AbortSignal) => boolean | Promise<boolean>
// true = pass, false = block
```

**Timeout par défaut** : 5000 ms

---

## 11. Timeouts

| Type de hook | Défaut | Configuration |
|-------------|--------|---------------|
| Command (outil) | 10 min | `hook.timeout` (sec) |
| Command (SessionEnd) | 1.5 sec | `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` |
| Prompt | 30 sec | `hook.timeout` (sec) |
| Agent | 60 sec | `hook.timeout` (sec) |
| HTTP | 10 min | `hook.timeout` (sec) |
| Fonction (session) | 5 sec | paramètre `timeout` |

**Gestion du signal** : `createCombinedAbortSignal()` combine le signal parent + timeout via AbortController.

---

## 12. Trust et sécurité

### Workspace trust

```
shouldSkipHookDueToTrust() :
  ├── Session non-interactive (SDK) → trust implicite
  ├── Session interactive + trust accepté → exécuter
  └── Session interactive + pas de trust → skip
```

Tous les types de hooks sont soumis à la vérification de trust.

### Sécurité HTTP

| Mesure | Description |
|--------|-------------|
| URL allowlist | `settings.allowedHttpHookUrls` |
| Env var allowlist | `hook.allowedEnvVars` |
| Injection header | CR/LF/NUL strippés |
| SSRF guard | Bloque IPs privées/link-local |
| Proxy sandbox | Routé via proxy si sandbox actif |

---

## 13. Variables d'environnement

Variables injectées dans l'environnement des hooks commande :

| Variable | Description |
|----------|-------------|
| `CLAUDE_PROJECT_DIR` | Racine du projet (stable, pas worktree) |
| `CLAUDE_PLUGIN_ROOT` | Répertoire du plugin |
| `CLAUDE_PLUGIN_DATA` | Répertoire de données du plugin |
| `CLAUDE_ENV_FILE` | Chemin vers fichier .sh pour export (SessionStart/Setup/CwdChanged/FileChanged) |
| `CLAUDE_PLUGIN_OPTION_*` | Valeurs de config plugin (clés sanitisées) |

---

## 14. Diagnostics et télémétrie

### Messages de progression

```typescript
{
  type: 'progress',
  data: {
    type: 'hook_progress',
    hookEvent: 'PreToolUse',
    hookName: 'security-check',
    command: 'bash ~/hooks/security-check.sh',
    promptText?: 'Is this safe?',
    statusMessage?: 'Checking...'
  }
}
```

### Événements analytics

| Événement | Description |
|-----------|-------------|
| `tengu_run_hook` | Batch de hooks démarré |
| `tengu_repl_hook_finished` | Batch terminé (success/blocking/error/cancelled) |
| `tengu_agent_stop_hook_success` | Agent hook réussi (durée, tours) |
| `tengu_agent_stop_hook_error` | Agent hook échoué |

---

## 15. Fichiers clés

| Fichier | Rôle |
|---------|------|
| `utils/hooks.ts` | Moteur d'exécution principal (600+ lignes) |
| `utils/hooks/sessionHooks.ts` | Hooks de session (runtime) |
| `utils/hooks/AsyncHookRegistry.ts` | Registre hooks asynchrones |
| `utils/hooks/execPromptHook.ts` | Exécution hooks prompt (LLM) |
| `utils/hooks/execAgentHook.ts` | Exécution hooks agent (multi-turn) |
| `utils/hooks/execHttpHook.ts` | Exécution hooks HTTP |
| `entrypoints/sdk/coreSchemas.ts` | Schemas Zod input/output |
| `entrypoints/sdk/coreTypes.ts` | Types TypeScript + HOOK_EVENTS |
| `types/hooks.ts` | Validation et type guards |
