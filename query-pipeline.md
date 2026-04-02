# Query Pipeline — Documentation complète

Le moteur central de Claude Code : comment un message utilisateur devient une réponse, tour après tour, avec gestion des erreurs, compaction et exécution d'outils.

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Architecture : config immutable + state mutable](#2-architecture)
3. [Phase 1 — Préparation des messages](#3-phase-1)
4. [Phase 2 — Appel API et streaming](#4-phase-2)
5. [Phase 3 — Post-sampling hooks](#5-phase-3)
6. [Phase 4 — Vérification d'interruption](#6-phase-4)
7. [Phase 5 — Chemins de récupération d'erreur](#7-phase-5)
8. [Phase 6 — Exécution des outils](#8-phase-6)
9. [Phase 7 — Attachments et mémoire](#9-phase-7)
10. [Phase 8 — Vérifications de terminaison](#10-phase-8)
11. [Phase 9 — Continuation](#11-phase-9)
12. [Conditions de terminaison](#12-conditions-terminaison)
13. [Arbre de décision des récupérations](#13-arbre-récupération)
14. [Invariants critiques](#14-invariants)
15. [Fichiers clés](#15-fichiers-clés)

---

## 1. Vue d'ensemble

Le query pipeline est un **async generator** `while(true)` qui itère jusqu'à une condition de terminaison. Chaque itération est un "tour" comprenant : préparation → appel API → exécution d'outils → récupération d'erreurs → continuation.

```
query(params)
  └── queryLoop() ← async generator, while(true)
        │
        ├── Phase 1 : Préparation messages (snip, microcompact, autocompact)
        ├── Phase 2 : Appel API + streaming (avec fallback modèle)
        ├── Phase 3 : Post-sampling hooks (fire-and-forget)
        ├── Phase 4 : Check interruption utilisateur
        ├── Phase 5 : Récupération erreurs (si pas de tool calls)
        │   ├── prompt_too_long → collapse drain → reactive compact
        │   ├── max_output_tokens → escalade 8K→64K → multi-turn
        │   ├── Stop hooks (mémoire, dream, suggestions)
        │   └── Token budget check
        ├── Phase 6 : Exécution outils (streaming ou séquentiel)
        ├── Phase 7 : Attachments, mémoire, skills
        ├── Phase 8 : Max turns, task summary
        └── Phase 9 : Construction nouvel état → retour Phase 1
```

**Fichier** : `query.ts` (1729 lignes)

---

## 2. Architecture

### QueryConfig (snapshot immutable)

Capturé **une seule fois** à l'entrée de `query()` :

```typescript
QueryConfig {
  sessionId: string,
  gates: {
    streamingToolExecution: boolean,  // Exécution parallèle d'outils
    emitToolUseSummaries: boolean,    // Résumés Haiku entre tours
    isAnt: boolean,                   // Télémétrie Ant-only
    fastModeEnabled: boolean,         // Fast mode activé
  }
}
```

### State (mutable, reconstruit à chaque continuation)

```typescript
State {
  messages: Message[],                          // Historique complet
  toolUseContext: ToolUseContext,               // Contexte d'exécution outils
  autoCompactTracking: AutoCompactTrackingState,
  maxOutputTokensRecoveryCount: number,         // Tentatives de récupération
  hasAttemptedReactiveCompact: boolean,
  maxOutputTokensOverride: number | undefined,  // 8K → 64K escalade
  pendingToolUseSummary: Promise | undefined,
  stopHookActive: boolean | undefined,
  turnCount: number,                            // Compteur d'itérations
  transition: Continue | undefined,             // Raison de la continuation
}
```

**7 sites de continuation** construisent explicitement un nouveau `State` :
1. Collapse drain recovery
2. Reactive compact recovery
3. Max output tokens escalade
4. Max output tokens recovery (multi-turn)
5. Stop hook blocking retry
6. Token budget continuation
7. Tour suivant avec résultats d'outils

---

## 3. Phase 1 — Préparation des messages

### 1a. Budget de contenu par message

```
applyToolResultBudget(messages)
→ Remplace les résultats d'outils surdimensionnés par des résumés
→ Persiste en fichier session uniquement pour le thread principal/agent
```

### 1b. Snip (HISTORY_SNIP)

```
snipModule.snipCompactIfNeeded()
→ Supprime les vieux messages
→ Retourne snipTokensFreed pour ajuster le seuil autocompact
→ Yield du boundary message si snip effectué
```

### 1c. Microcompaction (CACHED_MICROCOMPACT)

```
deps.microcompact()
→ Supprime les résultats d'outils dupliqués via cache editing
→ Capture pendingCacheEdits pour le boundary message post-API
→ Retourne cache_deleted_input_tokens
```

### 1d. Context collapse (CONTEXT_COLLAPSE)

```
contextCollapse.applyCollapsesIfNeeded()
→ Projette la vue collapsed
→ Peut committer des collapses staged
→ Projection read-time ; les summary messages persistent dans le store
```

### 1e. Autocompaction

```
deps.autocompact()
→ Circuit breaker : skip si consecutiveFailures ≥ 3
→ Supprimé si : reactive compact activé OU context collapse activé
→ Yield post-compact messages si compaction réussie
→ Reset tracking avec nouveau turnId
```

### 1f. Vérification limite bloquante

```
calculateTokenWarningState()
→ Si tokens ≥ blocking limit :
   → EXIT { reason: 'blocking_limit' }
```

---

## 4. Phase 2 — Appel API et streaming

### Boucle de fallback

```
while (attemptWithFallback) {
  currentModel = getRuntimeMainLoopModel()
  streamingToolExecutor = new StreamingToolExecutor() // si gate

  for await (message of deps.callModel(...)) {
    // Traitement de chaque chunk/message
  }
}
```

### Traitement des messages streamés

Pour chaque message du stream :

**i. Streaming fallback** (503 → modèle de fallback)
```
Si onStreamingFallback() déclenché :
→ Tombstone les messages assistant orphelins
→ Clear assistantMessages, toolResults, toolUseBlocks
→ Discard et recréer streamingToolExecutor
→ Retry avec fallbackModel
```

**ii. Backfill des inputs observables**
```
Pour les messages avec tool_use :
→ Backfill observable input fields
→ Yield message cloné si champs ajoutés
```

**iii. Withholding (rétention)**

Certains messages d'erreur sont **retenus** (pas envoyés à l'utilisateur) pour permettre la récupération :

| Condition | Garde |
|-----------|-------|
| Context collapse prompt_too_long | `contextCollapse.isWithheldPromptTooLong()` |
| Reactive compact prompt_too_long | `reactiveCompact.isWithheldPromptTooLong()` |
| Media recovery | `reactiveCompact.isWithheldMediaSizeError()` |
| Max output tokens | `isWithheldMaxOutputTokens()` |

**iv. Collection des messages assistant**
```
assistantMessages.push(message)
Extraction des blocs tool_use → needsFollowUp = true
Si streamingToolExecutor :
  → streamingToolExecutor.addTool(block, message) (queue)
  → Yield des résultats complétés (streaming)
```

---

## 5. Phase 3 — Post-sampling hooks

```
Fire post-sampling hooks (fire-and-forget)
→ Asynchrone, ne bloque pas l'itération
→ Seulement si assistantMessages existent
```

---

## 6. Phase 4 — Vérification d'interruption

```
Si abortController.signal.aborted :
  ├── StreamingToolExecutor : consomme getRemainingResults()
  │   (génère des tool_results synthétiques)
  ├── runTools : yield missing tool_results "Interrupted by user"
  ├── Computer Use cleanup (CHICAGO_MCP)
  └── EXIT { reason: 'aborted_streaming' }
```

---

## 7. Phase 5 — Chemins de récupération d'erreur

**Condition** : `needsFollowUp === false` (pas de tool_use dans la réponse)

### 5a. Récupération prompt_too_long

```
Si erreur 413 retenue :
  │
  ├── Context collapse drain disponible ?
  │   ├── OUI → drain() → CONTINUE (collapse_drain_retry)
  │   └── NON ↓
  │
  ├── Reactive compact ?
  │   ├── COMPACTÉ → CONTINUE (reactive_compact_retry)
  │   └── ÉCHEC → EXIT { reason: 'prompt_too_long' }
  │
  └── Pas de recovery → EXIT { reason: 'prompt_too_long' }
```

### 5b. Récupération max_output_tokens

```
Si erreur max_output_tokens retenue :
  │
  ├── Escalade disponible (8K → 64K) ?
  │   ├── OUI → CONTINUE (max_output_tokens_escalate)
  │   └── NON ↓
  │
  ├── Compteur recovery < 3 ?
  │   ├── OUI → Message "Resume directly" → CONTINUE
  │   └── NON → Yield erreur retenue, fall through
```

### 5c. Stop hooks

```
Si message d'erreur API :
  → executeStopFailureHooks()
  → EXIT { reason: 'completed' }

Sinon :
  → handleStopHooks() :
    ├── Job classification (TEMPLATES)
    ├── Prompt suggestion (fire-and-forget)
    ├── Memory extraction (fire-and-forget)
    ├── Auto-dream (fire-and-forget)
    ├── Chicago MCP cleanup
    ├── Execute Stop hooks
    ├── TaskCompleted hooks
    └── TeammateIdle hooks

  Si preventContinuation → EXIT { reason: 'stop_hook_prevented' }
  Si blockingErrors → CONTINUE (stop_hook_blocking_retry)
```

### 5d. Token budget (TOKEN_BUDGET)

```
checkTokenBudget(tracker, budget)
  ├── action: 'continue' → Message nudge → CONTINUE
  ├── Diminishing returns (3+ continuations, delta < 500 tokens) → Stop
  └── Sinon → EXIT { reason: 'completed' }
```

---

## 8. Phase 6 — Exécution des outils

**Condition** : `needsFollowUp === true`

### Exécution

```
Pour chaque résultat de streamingToolExecutor.getRemainingResults()
OU runTools() :
  ├── yield update.message (tool_result, progress)
  ├── Si hook_stopped_continuation → shouldPreventContinuation = true
  ├── Normalise et push dans toolResults
  └── Si update.newContext → maj toolUseContext
```

### StreamingToolExecutor

Exécute les outils au fur et à mesure du streaming :

| Propriété | Comportement |
|-----------|-------------|
| Outils concurrent-safe | Exécution parallèle |
| Outils non-concurrent | Exécution exclusive |
| Ordre des résultats | Toujours dans l'ordre de réception |
| Sibling abort | Une erreur Bash annule les frères |

### Résumé de tour (tool use summary)

Si `emitToolUseSummaries` gate activé :
- Génère un résumé via Haiku (~1s) pendant l'itération suivante
- Collecte les paires input/output d'outils
- Promise await en début de tour suivant

---

## 9. Phase 7 — Attachments et mémoire

### 7a. Commandes en queue
```
Drain des commandes par priorité max
Filtre : non-slash, bon agentId
```

### 7b. Messages d'attachments
```
getAttachmentMessages()
→ Fichiers édités, interruptions utilisateur, mémoire
→ Yield et push dans toolResults
```

### 7c. Consommation du memory prefetch
```
Si prefetch terminé et pas encore consommé ce tour :
→ Await promise
→ Filtre doublons
→ Yield et push
```

### 7d. Découverte de skills
```
Collecte les résultats de skill discovery prefetch
→ Yield et push
```

### 7e. Dequeue des commandes
```
Retire les commandes consommées de la queue
Notifie le cycle de vie ('started' → 'completed')
```

---

## 10. Phase 8 — Vérifications de terminaison

### Max turns
```
nextTurnCount = turnCount + 1
Si maxTurns && nextTurnCount > maxTurns :
  → Yield max_turns_reached attachment
  → EXIT { reason: 'max_turns' }
```

### Task summary (BG_SESSIONS)
```
Génère un résumé périodique pour `claude ps`
→ Pour les agents long-running (mid-turn)
```

### Refresh des outils
```
Si nouveaux serveurs MCP connectés :
→ refreshTools()
→ Maj toolUseContext.options.tools
```

---

## 11. Phase 9 — Continuation

Construction du nouvel état :

```typescript
state = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  toolUseContext: updated,
  autoCompactTracking: tracking,
  turnCount: nextTurnCount,
  maxOutputTokensRecoveryCount: 0,    // Reset
  hasAttemptedReactiveCompact: false,  // Reset
  maxOutputTokensOverride: undefined,  // Reset
  pendingToolUseSummary: next,
  stopHookActive,
  transition: { reason: 'next_turn' },
}
```

Retour à la Phase 1.

---

## 12. Conditions de terminaison

| Raison | Description | Phase |
|--------|-------------|-------|
| `completed` | Terminaison normale (pas de tool calls) | 5c/5d |
| `max_turns` | `turnCount > maxTurns` | 8 |
| `blocking_limit` | Tokens au hard limit | 1f |
| `prompt_too_long` | Récupération 413 épuisée | 5a |
| `image_error` | Erreur média non récupérable | 2/5a |
| `model_error` | Erreur API non rattrapée | 2 |
| `aborted_streaming` | Ctrl+C pendant le streaming | 4 |
| `aborted_tools` | Ctrl+C pendant l'exécution d'outils | 6 |
| `hook_stopped` | Hook a bloqué la continuation | 6 |
| `stop_hook_prevented` | Stop hook a interdit la continuation | 5c |

---

## 13. Arbre de décision des récupérations

```
needsFollowUp === false (pas de tool calls)
│
├── Erreur API ?
│   ├── prompt_too_long (413 retenu) ?
│   │   ├── Collapse drain → CONTINUE
│   │   ├── Reactive compact → CONTINUE
│   │   └── EXIT prompt_too_long
│   │
│   ├── max_output_tokens (retenu) ?
│   │   ├── Escalade 8K→64K → CONTINUE
│   │   ├── Recovery count < 3 → CONTINUE
│   │   └── Yield erreur, fall through
│   │
│   └── Autre erreur API
│       → executeStopFailureHooks()
│       → EXIT completed
│
├── Stop hooks
│   ├── preventContinuation → EXIT stop_hook_prevented
│   ├── blockingErrors → CONTINUE stop_hook_blocking_retry
│   └── OK → fall through
│
├── Token budget check
│   ├── Continue → CONTINUE token_budget_continuation
│   ├── Diminishing returns → stop
│   └── Fall through
│
└── EXIT completed
```

---

## 14. Invariants critiques

| Invariant | Description |
|-----------|-------------|
| **Appariement tool_use/tool_result** | Chaque bloc `tool_use` doit avoir un `tool_result` correspondant dans le message user suivant |
| **Withholding/Recovery** | Si un message est retenu, le chemin de récupération doit le traiter |
| **Propagation abort** | Les controllers enfants remontent ; abort ne quitte pas la boucle interne |
| **Construction d'état** | Chaque site de continuation construit explicitement un nouveau State |
| **Stop hooks** | Exécutés UNIQUEMENT après une réponse modèle valide (pas sur erreur API) |
| **Accumulation messages** | Chaque itération ajoute assistantMessages + toolResults au tour suivant |
| **Immutabilité config** | QueryConfig ne change jamais ; les feature gates sont inline |

---

## 15. Fichiers clés

| Fichier | Rôle | Taille |
|---------|------|--------|
| `query.ts` | Boucle principale, toutes les phases | 1729 lignes |
| `query/config.ts` | QueryConfig snapshot | |
| `query/stopHooks.ts` | Stop hooks (mémoire, dream, classification) | |
| `query/tokenBudget.ts` | Système de budget tokens | |
| `query/deps.ts` | Injection de dépendances | |
| `services/api/claude.ts` | Appel API, streaming, retry | 2700+ lignes |
| `services/tools/StreamingToolExecutor.ts` | Exécution parallèle/exclusive | |
| `services/compact/autoCompact.ts` | Déclenchement auto-compaction | |
