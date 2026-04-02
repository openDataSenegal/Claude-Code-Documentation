# Système de Compaction — Documentation complète

Comment Claude Code gère la fenêtre de contexte : 8 stratégies de compression, restauration automatique et résilience.

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Les 8 stratégies de compaction](#2-stratégies)
3. [Auto-compaction (proactive)](#3-auto-compaction)
4. [Session memory compaction (légère)](#4-session-memory-compaction)
5. [Full compaction (complète)](#5-full-compaction)
6. [Partial compaction (manuelle)](#6-partial-compaction)
7. [Microcompaction (in-API)](#7-microcompaction)
8. [Reactive compaction (urgence)](#8-reactive-compaction)
9. [API context management (server-side)](#9-api-context-management)
10. [Context collapse](#10-context-collapse)
11. [Processus de résumé](#11-processus-résumé)
12. [Restauration post-compaction](#12-restauration)
13. [Gestion des erreurs et résilience](#13-résilience)
14. [Constantes et seuils](#14-constantes)
15. [Fichiers clés](#15-fichiers-clés)

---

## 1. Vue d'ensemble

La compaction est le mécanisme qui permet à Claude Code de gérer des conversations plus longues que la fenêtre de contexte. Elle résume les messages anciens tout en préservant le contexte critique.

```
Fenêtre de contexte (~200K tokens)
┌──────────────────────────────────────────────┐
│ System prompt (statique + dynamique)          │
│ ─── Compact boundary ───                      │
│ [Résumé des messages anciens]                 │
│ [Attachments restaurés]                       │
│ ─── Messages préservés ───                    │
│ [Messages récents (verbatim)]                 │
│ [Tool results]                                │
│ [Réponse en cours]                            │
│ ─── Buffer (13K tokens) ───                   │
└──────────────────────────────────────────────┘
```

### Ordre de priorité des stratégies

```
1. Context collapse      (projection read-time, pas d'API call)
2. Microcompaction        (cache editing, pas d'API call de compaction)
3. Session memory compact (lightweight, pas d'API call de compaction)
4. Auto-compact           (proactif, avant que ça devienne urgent)
5. Reactive compact       (urgence, sur erreur prompt_too_long)
6. API context management (server-side, transparent)
7. Full compact           (API call dédié, résumé complet)
8. Partial compact        (sélection manuelle par l'utilisateur)
```

---

## 2. Les 8 stratégies

| Stratégie | Déclencheur | API call | Impact cache | Coût |
|-----------|------------|---------|-------------|------|
| **Context collapse** | Feature flag | Non | Aucun | Nul |
| **Cached microcompact** | Nombre d'outils | Non (cache_edits) | Préserve | Minimal |
| **Time-based microcompact** | Gap > 60 min | Non (mutation) | Aucun (cache expiré) | Nul |
| **Session memory compact** | Seuil tokens | Non | Reset partiel | Minimal |
| **Auto-compact** | Seuil tokens | Oui (résumé) | Reset complet | Moyen |
| **Reactive compact** | Erreur 413 | Oui (résumé) | Reset complet | Moyen |
| **API context management** | Seuil tokens | Non (paramètre API) | Géré server-side | Nul |
| **Full compact** | Manuel `/compact` | Oui (résumé) | Reset complet | Moyen |
| **Partial compact** | Sélection UI | Oui (résumé) | Partiel | Moyen |

---

## 3. Auto-compaction (proactive)

**Fichier** : `services/compact/autoCompact.ts`

### Déclenchement

```
effectiveWindow = contextWindow - 20,000 (réservé pour la sortie)
autoCompactThreshold = effectiveWindow - 13,000 (AUTOCOMPACT_BUFFER_TOKENS)

Exemple : 200K context → effectiveWindow = 180K → seuil = 167K tokens
```

### Conditions d'activation

```
✓ Pas de DISABLE_COMPACT ou DISABLE_AUTO_COMPACT env var
✓ Pas désactivé dans les settings
✓ Circuit breaker OK (consecutiveFailures < 3)
✓ Pas supprimé par reactive compact (tengu_cobalt_raccoon)
✓ Pas supprimé par context collapse
✓ Pas un querySource de type compact/session_memory
```

### Tracking

```typescript
AutoCompactTrackingState {
  turnId: string,
  turnCounter: number,
  consecutiveFailures: number,  // Circuit breaker à 3
  compacted: boolean,
}
```

### Flux

```
1. Vérifie le seuil de tokens
2. Tente session memory compaction d'abord (si activé)
3. Si pas de session memory ou échec → full compaction
4. Yield post-compact messages
5. Reset tracking avec nouveau turnId
```

---

## 4. Session memory compaction (légère)

**Fichier** : `services/compact/sessionMemoryCompact.ts`

### Prérequis

- Feature flags `tengu_session_memory` ET `tengu_sm_compact` activés
- Fichier session memory existant avec contenu réel (pas le template vide)

### Processus (pas d'API call)

```
1. Lire le fichier session memory (~/.claude/session-memory/<sessionId>.md)
2. Trouver le lastSummarizedMessageId (frontière)
3. Calculer les messages à garder :
   ├── Expandre en arrière depuis la frontière
   ├── Min tokens : 10,000
   ├── Min messages texte : 5
   └── Max tokens : 40,000
4. Préserver les paires tool_use/tool_result (jamais couper au milieu)
5. Filtrer les anciens compact boundary messages
6. Re-lancer les session start hooks (CLAUDE.md, etc.)
```

### Avantage

Beaucoup plus rapide que la full compaction — pas d'appel API de résumé. Utilise le résumé pré-extrait de la session memory.

---

## 5. Full compaction (complète)

**Fichier** : `services/compact/compact.ts` (1705 lignes)

### Fonction : `compactConversation()`

```
1. Exécuter pre-compact hooks
2. Stripper images/documents (causeraient prompt_too_long)
3. Stripper les attachments ré-injectés (skill_discovery, skill_listing)
4. Envoyer messages + prompt de résumé à l'API Claude
5. Gérer prompt-too-long retry (truncateHeadForPTLRetry, max 3 retries)
6. Parser + formater le résumé :
   ├── Stripper le bloc <analysis> (brouillon)
   └── Garder le bloc <summary>
7. Vider le cache read-file
8. Générer les attachments post-compact
9. Créer le compact boundary marker
10. Lancer post-compact hooks
```

### Cache sharing

La compaction tente de réutiliser le **prompt cache** du thread principal via un agent forké :
- Cible : 80% de cache hit par défaut
- Fallback : streaming régulier si le fork échoue
- Analytics : `tengu_compact_cache_sharing_success/fallback`

### Appel API

| Paramètre | Valeur |
|-----------|--------|
| Modèle | Modèle principal (Claude 4.5 typiquement) |
| Outils | FileReadTool (+ ToolSearchTool + MCP si tool-search activé) |
| Thinking | Désactivé |
| Max output tokens | 20,000 (`MAX_OUTPUT_TOKENS_FOR_SUMMARY`) |
| System prompt | "You are a helpful AI assistant tasked with summarizing conversations." |

---

## 6. Partial compaction (manuelle)

**Fichier** : `services/compact/compact.ts` → `partialCompactConversation()`

### Deux directions

| Direction | Action | Cache |
|-----------|--------|-------|
| `from` | Résume les messages APRÈS l'index, garde les précédents | Préserve |
| `up_to` | Résume les messages AVANT l'index, garde les suivants | Invalide |

### Usage

L'utilisateur sélectionne un message dans l'UI pour résumer autour de celui-ci.

---

## 7. Microcompaction (in-API)

**Fichier** : `services/compact/microCompact.ts`

### 7a. Cached microcompact (cache_edits)

Utilise le paramètre `cache_edits` de l'API pour supprimer des résultats d'outils anciens **sans invalider le prompt cache**.

```
Trigger : Nombre d'outils depuis la dernière microcompaction
          (seuil configurable via GrowthBook)

Action :
1. Tracker les tool results par message user
2. Queuer un cache_edit block au niveau API
3. L'API supprime les anciens résultats côté serveur
4. Les messages locaux restent INCHANGÉS (cache_edits séparé)
```

### 7b. Time-based microcompact (mutation directe)

Se déclenche quand le cache serveur a probablement expiré.

```
Trigger : Gap > 60 minutes depuis le dernier message assistant
          (gapThresholdMinutes configurable, défaut 60)

Hypothèse : Le cache prompt serveur a expiré

Action :
1. Content-clear tous les tool results sauf les N derniers
2. keepRecent = 5 (défaut)
3. Outils ciblés : FileRead, Bash, Grep, Glob, WebSearch, WebFetch, FileEdit, FileWrite
4. Mutation directe des messages (pas de cache_edits)
```

---

## 8. Reactive compaction (urgence)

### Déclencheur

L'API retourne une erreur `prompt_too_long` (413).

### Mécanisme

Géré dans le query pipeline (Phase 5a) :

```
Erreur 413 retenue (withheld)
    │
    ├── Context collapse drain disponible ?
    │   └── OUI → drain commits → retry
    │
    ├── Reactive compact disponible ?
    │   ├── tryReactiveCompact() → compaction d'urgence
    │   └── Retry avec messages compactés
    │
    └── Aucune recovery → EXIT prompt_too_long
```

### Feature flag

`REACTIVE_COMPACT` — quand activé, **supprime** l'auto-compaction proactive (laisse l'erreur API déclencher la recovery).

---

## 9. API context management (server-side)

**Fichier** : `services/compact/apiMicrocompact.ts`

Envoyé dans le paramètre `context_management` de la requête API :

```typescript
context_management: {
  edits: [
    {
      type: "clear_tool_uses_20250919",
      threshold_tokens: 180000  // Déclenche quand input_tokens dépasse
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
| `clear_tool_uses` | Efface les résultats/appels d'outils au-delà du seuil |
| `clear_thinking` | Efface les blocs de réflexion, garde le dernier tour |

### Paramètres

| Paramètre | Défaut | Description |
|-----------|--------|-------------|
| `threshold_tokens` | 180,000 | Seuil de déclenchement |
| `target_tokens` | 40,000 | Tokens à conserver |

---

## 10. Context collapse

**Feature flag** : `CONTEXT_COLLAPSE`

### Principe

Projection read-time des messages — les messages anciens sont **collapsés** (remplacés par des résumés) sans appel API de compaction.

### Interaction avec auto-compact

Quand activé, context collapse **supprime** l'auto-compaction proactive :
- Utilise un flow commit/blocking à 90%/95% au lieu du seuil 93% d'auto-compact
- Évite la collision entre les deux systèmes

### Recovery

```
contextCollapse.recoverFromOverflow(messages, querySource)
→ Si drained.committed > 0 : retry (collapse_drain_retry)
→ Sinon : fall through vers reactive compact
```

---

## 11. Processus de résumé

### Prompt de résumé

**Fichier** : `services/compact/prompt.ts`

Le prompt envoie les messages à résumer à Claude avec ces instructions :

```
1. PREAMBLE : "CRITICAL: Respond with TEXT ONLY. Do NOT call any tools."

2. ANALYSIS BLOCK : Instructions pour un brouillon <analysis>

3. SECTIONS DU RÉSUMÉ (9 sections) :
   ├── Primary Request and Intent
   ├── Key Technical Concepts
   ├── Files and Code Sections (avec snippets de code)
   ├── Errors and Fixes
   ├── Problem Solving
   ├── All User Messages (non-tool-results)
   ├── Pending Tasks
   ├── Current Work / Work Completed
   └── Optional Next Step / Context for Continuing
```

### Post-traitement

```
formatCompactSummary(response)
├── Stripper le bloc <analysis> (brouillon, pas envoyé au modèle)
├── Remplacer les tags <summary> par un header "Summary:"
└── Retourner le texte nettoyé
```

### Message de résumé injecté

```
"This session is being continued from a previous conversation..."
├── Lien vers le transcript complet
├── "Recent messages preserved verbatim" (pour partial)
├── Suppression des questions de suivi (auto-compact)
└── Notice mode proactif (continuation autonome)
```

### Variantes de prompt

| Variante | Usage |
|----------|-------|
| `BASE_COMPACT_PROMPT` | Résumé de conversation complète |
| `PARTIAL_COMPACT_PROMPT` | Résumé de messages récents uniquement |
| `PARTIAL_COMPACT_UP_TO_PROMPT` | Résumé du préfixe (préserve le cache) |

---

## 12. Restauration post-compaction

Après compaction, plusieurs éléments sont restaurés automatiquement pour que le modèle ne perde pas le contexte.

### 12a. Fichiers récemment lus

```
createPostCompactFileAttachments()
├── Sélectionne les 5 fichiers les plus récents (par timestamp)
├── Exclut ceux déjà dans les messages préservés
├── Exclut les fichiers plan et mémoire (claude.md)
├── Relit chaque fichier via FileReadTool (contenu frais)
└── Injecte comme attachments
```

| Contrainte | Valeur |
|------------|--------|
| Fichiers max | 5 |
| Budget tokens total | 50,000 |
| Tokens par fichier | 5,000 max |

### 12b. Skills invoqués

```
createSkillAttachmentIfNeeded()
├── Récupère les skills utilisés dans la session
├── Plus récemment invoqué en premier
├── Tronque si nécessaire avec marqueur
└── Injecte le contenu complet du skill
```

| Contrainte | Valeur |
|------------|--------|
| Budget tokens total | 25,000 |
| Tokens par skill | 5,000 max |

### 12c. Plan actif

```
createPlanAttachmentIfNeeded()
└── Contenu du fichier plan + chemin
```

### 12d. Mode plan

```
createPlanModeAttachmentIfNeeded()
└── Si en mode plan → ré-injecte les instructions plan_mode
```

### 12e. Agents asynchrones

```
createAsyncAgentAttachmentsIfNeeded()
├── Agents en cours d'exécution
├── Agents terminés mais pas récupérés
└── Attachment task_status par agent
```

### 12f. Deltas (outils, agents, MCP)

```
getDeferredToolsDeltaAttachment()    → Outils nouvellement chargés
getAgentListingDeltaAttachment()     → Agents disponibles
getMcpInstructionsDeltaAttachment()  → Instructions MCP
```

Diff contre les messages préservés — n'injecte que ce qui n'est pas déjà visible.

### 12g. Session start hooks

```
processSessionStartHooks('compact', { model })
→ Recharge CLAUDE.md, instructions, etc.
```

### 12h. Compact boundary marker

```typescript
CompactBoundaryMessage {
  type: 'system',
  subtype: 'compact_boundary',
  content: 'Conversation compacted',
  compactMetadata: {
    trigger: 'auto' | 'manual',
    preTokens: number,
    messagesSummarized: number,
    preCompactDiscoveredTools: string[],  // Outils chargés avant compaction
    preservedSegment?: {                  // UUIDs pour le linking
      headUuid, anchorUuid, tailUuid
    }
  }
}
```

---

## 13. Gestion des erreurs et résilience

### Prompt-too-long retry

```
truncateHeadForPTLRetry()
├── Trigger : réponse de résumé contient "API Error: prompt_too_long"
├── Max retries : 3
├── Stratégie : grouper par API-round, supprimer les plus anciens
├── Fallback : si gap non parseable, supprimer 20% des groupes
├── Message retry : "[earlier conversation truncated for compaction retry]"
└── Dernier recours : ERROR_MESSAGE_PROMPT_TOO_LONG
```

### Streaming retry

```
├── Max tentatives : 2 (si tengu_compact_streaming_retry activé)
├── Condition : pas de réponse après début du streaming
├── Délai : getRetryDelay(attempt)
└── Erreur fallback : ERROR_MESSAGE_INCOMPLETE_RESPONSE
```

### Cache sharing fallback

```
Primaire : agent forké avec réutilisation du prompt cache
Fallback : streaming régulier si :
  ├── Cache sharing désactivé
  ├── Erreur de fork
  └── Pas de texte dans la réponse
```

### Circuit breaker (auto-compact)

```
consecutiveFailures >= 3 → arrête les tentatives sur ce tour
Reset sur compaction réussie
```

### User abort

```
AbortController.signal → abort compaction (main + agent forké)
ERROR_MESSAGE_USER_ABORT
```

---

## 14. Constantes et seuils

| Constante | Valeur | Description |
|-----------|--------|-------------|
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | Marge avant auto-compact |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | 20,000 | Seuil d'avertissement tokens |
| `ERROR_THRESHOLD_BUFFER_TOKENS` | 20,000 | Seuil d'erreur tokens |
| `MANUAL_COMPACT_BUFFER_TOKENS` | 3,000 | Limite bloquante pour `/compact` |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | Circuit breaker |
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | Budget output du résumé |
| `POST_COMPACT_MAX_FILES_TO_RESTORE` | 5 | Fichiers restaurés max |
| `POST_COMPACT_TOKEN_BUDGET` | 50,000 | Budget total attachments fichiers |
| `POST_COMPACT_MAX_TOKENS_PER_FILE` | 5,000 | Tokens max par fichier |
| `POST_COMPACT_MAX_TOKENS_PER_SKILL` | 5,000 | Tokens max par skill |
| `POST_COMPACT_SKILLS_TOKEN_BUDGET` | 25,000 | Budget total skills |
| `MAX_PTL_RETRIES` | 3 | Retries prompt-too-long |
| `MAX_COMPACT_STREAMING_RETRIES` | 2 | Retries streaming |
| `DEFAULT_MAX_INPUT_TOKENS` (API MC) | 180,000 | Seuil context management |
| `DEFAULT_TARGET_INPUT_TOKENS` (API MC) | 40,000 | Cible keep-alive |

### Seuils calculés (exemple 200K context)

```
Context window :           200,000 tokens
Réservé output :          - 20,000
Effective window :         180,000
Auto-compact buffer :     - 13,000
─────────────────────────────────
Auto-compact threshold :   167,000 tokens
```

---

## 15. Fichiers clés

| Fichier | Rôle | Taille |
|---------|------|--------|
| `services/compact/compact.ts` | Full + partial compaction | 1,705 lignes |
| `services/compact/autoCompact.ts` | Déclenchement proactif | 352 lignes |
| `services/compact/sessionMemoryCompact.ts` | Compaction légère session memory | 631 lignes |
| `services/compact/microCompact.ts` | Cached + time-based microcompaction | 531 lignes |
| `services/compact/apiMicrocompact.ts` | Context management server-side | 154 lignes |
| `services/compact/prompt.ts` | Prompt de résumé (9 sections) | 375 lignes |
| `services/compact/postCompactCleanup.ts` | Nettoyage et invalidation caches | 78 lignes |
| `services/compact/grouping.ts` | Groupement par API-round | 64 lignes |
| `services/compact/timeBasedMCConfig.ts` | Config time-based microcompact | 44 lignes |
