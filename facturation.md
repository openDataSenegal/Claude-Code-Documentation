# Système de Facturation — Documentation complète

Tout ce que Claude Code sait sur les coûts : pricing par modèle, comptage de tokens, abonnements, rate limits, overage, et affichage.

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Abonnements et tiers](#2-abonnements)
3. [Pricing par modèle](#3-pricing)
4. [Pipeline de comptage des tokens](#4-comptage-tokens)
5. [Calcul des coûts en USD](#5-calcul-coûts)
6. [Accumulation et persistance](#6-accumulation)
7. [Rate limits et quotas](#7-rate-limits)
8. [Système d'overage (extra usage)](#8-overage)
9. [Avertissements précoces](#9-avertissements)
10. [Messages d'erreur rate limit](#10-messages-erreur)
11. [Commandes utilisateur](#11-commandes)
12. [Contrôle d'accès à la facturation](#12-controle-accès)
13. [Policy limits (admin)](#13-policy-limits)
14. [Referral et guest passes](#14-referral)
15. [Télémétrie et analytics](#15-télémétrie)
16. [Fichiers clés](#16-fichiers-clés)

---

## 1. Vue d'ensemble

Claude Code implémente un système de facturation à 5 étages :

```
Réponse API
    ↓
[1] Extraction des tokens (input, output, cache read, cache write, web search)
    ↓
[2] Calcul du coût USD (pricing par modèle × tokens)
    ↓
[3] Accumulation dans l'état global (par modèle, par session)
    ↓
[4] Persistance sur disque (~/.claude/config.json)
    ↓
[5] Affichage (commande /cost, status line, warnings)
```

### Qui paie quoi

| Utilisateur | Facturation | Voit les coûts | Rate limits |
|-------------|-------------|----------------|-------------|
| **API Key** (Console) | Pay-per-use | Oui (détaillé) | API rate limits |
| **Claude Pro** | Abonnement $20/mois | Non (abonnement) | 5h + 7j |
| **Claude Max** | Abonnement $200/mois | Non (abonnement) | 5h + 7j (plus élevés) |
| **Team** | Abonnement organisation | Non (abonnement) | 5h + 7j + policy |
| **Enterprise** | Contrat | Non (abonnement) | 5h + 7j + policy |
| **Ant** (interne) | Interne | Oui (forcé) | Interne |

---

## 2. Abonnements et tiers

### Types d'abonnement

| Tier | Valeur interne | Billing type | Overage |
|------|---------------|--------------|---------|
| **Free** | `null` | API key uniquement | Non |
| **Pro** | `'pro'` | `stripe_subscription` | Non |
| **Max** | `'max'` | `stripe_subscription` | Non |
| **Team** | `'team'` | `stripe_subscription` / `stripe_subscription_contracted` | Oui (si provisionné) |
| **Enterprise** | `'enterprise'` | Contrat | Oui (si provisionné) |

### Détection du tier

```typescript
// services/oauth/client.ts
organization.organization_type mapping:
  'claude_max'        → 'max'
  'claude_pro'        → 'pro'
  'claude_team'       → 'team'
  'claude_enterprise' → 'enterprise'
```

### Profil compte

```typescript
AccountInfo {
  accountUuid: string,
  emailAddress: string,
  organizationUuid?: string,
  organizationRole?: string,      // 'admin' | 'member' | 'billing' | 'owner'
  workspaceRole?: string,
  billingType?: BillingType,
  subscriptionCreatedAt?: string,
  hasExtraUsageEnabled?: boolean,  // Overage provisionné au niveau org
}
```

Stocké dans `~/.claude/config.json` sous `oauthAccount`.

---

## 3. Pricing par modèle

### Grille tarifaire (USD par million de tokens)

| Modèle | Input | Output | Cache Write | Cache Read | Web Search |
|--------|-------|--------|-------------|------------|------------|
| **Haiku 3.5** | $0.80 | $4.00 | $1.00 | $0.08 | $0.01/req |
| **Haiku 4.5** | $1.00 | $5.00 | $1.25 | $0.10 | $0.01/req |
| **Sonnet** (toutes versions) | $3.00 | $15.00 | $3.75 | $0.30 | $0.01/req |
| **Opus 4 / 4.1** | $15.00 | $75.00 | $18.75 | $1.50 | $0.01/req |
| **Opus 4.5** | $5.00 | $25.00 | $6.25 | $0.50 | $0.01/req |
| **Opus 4.6** | $5.00 | $25.00 | $6.25 | $0.50 | $0.01/req |
| **Opus 4.6 (Fast)** | $30.00 | $150.00 | $37.50 | $3.00 | $0.01/req |

**Fichier** : `utils/modelCost.ts`

### Rapport cache write / cache read

| Opération | Rapport vs input normal |
|-----------|------------------------|
| Cache write (création) | 125% du prix input (surcoût 25%) |
| Cache read (lecture) | 10% du prix input (économie 90%) |

### Fast mode

Le fast mode ne s'applique qu'à **Opus 4.6** et multiplie les coûts par **6× l'input** et **6× l'output** par rapport au mode normal.

### Modèles inconnus

Si un modèle n'est pas dans la grille tarifaire :
- Fallback vers le tier Opus 4.5 ($5/$25)
- Flag `hasUnknownModelCost` activé
- Warning affiché dans l'UI

---

## 4. Pipeline de comptage des tokens

### Étape 1 — Extraction depuis la réponse API

À chaque réponse streamée, les tokens sont extraits des événements SSE :

| Événement | Données extraites |
|-----------|------------------|
| `message_start` | `input_tokens`, `cache_read_input_tokens`, `cache_creation_input_tokens` |
| `content_block_delta` | `output_tokens` (incrémental) |
| `message_delta` | Compteurs finaux, `web_search_requests` |

**Fonction** : `updateUsage()` dans `services/api/claude.ts` — merge les mises à jour incrémentales, protège contre les écrasements par zéro.

### Étape 2 — Types de tokens trackés

| Type | Description | Coût |
|------|-------------|------|
| `input_tokens` | Tokens d'entrée normaux | Prix input |
| `output_tokens` | Tokens générés par le modèle | Prix output |
| `cache_read_input_tokens` | Tokens lus depuis le cache prompt | 10% du prix input |
| `cache_creation_input_tokens` | Tokens écrits dans le cache prompt | 125% du prix input |
| `web_search_requests` | Nombre de recherches web | $0.01/requête |
| `speed` | `'fast'` ou `'normal'` | ×6 si fast (Opus 4.6) |

### Étape 3 — Estimation de tokens (pré-API)

Avant l'appel API, le système estime le nombre de tokens pour la gestion du contexte :

| Méthode | Précision | Coût | Usage |
|---------|-----------|------|-------|
| `roughTokenCountEstimation()` | Basse (content.length / 4) | Zéro | Décisions rapides |
| `tokenCountWithEstimation()` | Moyenne (usage API + estimation delta) | Zéro | Autocompact, warnings |
| `countMessagesTokensWithAPI()` | Haute (API `countTokens`) | 1 call API | Vérification précise |
| `countTokensViaHaikuFallback()` | Haute (via Haiku) | 1 call Haiku | Fallback |

**Fonction canonique** : `tokenCountWithEstimation()` dans `utils/tokens.ts` — combine l'usage réel de la dernière réponse + estimation rough pour les nouveaux messages.

---

## 5. Calcul des coûts en USD

### Formule

```
USD = (input_tokens / 1,000,000) × inputPrice
    + (output_tokens / 1,000,000) × outputPrice
    + (cache_read_input_tokens / 1,000,000) × cacheReadPrice
    + (cache_creation_input_tokens / 1,000,000) × cacheWritePrice
    + web_search_requests × $0.01
```

**Fonction** : `calculateUSDCost(model, usage)` dans `utils/modelCost.ts`

### Exemple

Pour un appel Opus 4.6 avec :
- 50,000 input tokens
- 2,000 output tokens
- 40,000 cache read tokens
- 10,000 cache write tokens
- 1 web search

```
Coût = (50,000 / 1M × $5.00)      = $0.250
     + (2,000 / 1M × $25.00)      = $0.050
     + (40,000 / 1M × $0.50)      = $0.020
     + (10,000 / 1M × $6.25)      = $0.063
     + (1 × $0.01)                 = $0.010
     ─────────────────────────────
     Total                         = $0.393
```

### Advisor tool

Les coûts de l'advisor (outil server-side) sont calculés récursivement depuis `usage.server_tool_use.web_search_requests`.

---

## 6. Accumulation et persistance

### État global

**Fichier** : `bootstrap/state.ts`

```typescript
STATE = {
  totalCostUSD: number,              // Coût cumulé de la session
  totalAPIDuration: number,          // Durée API cumulée
  totalAPIDurationWithoutRetries: number,
  totalToolDuration: number,         // Durée des outils
  totalLinesAdded: number,
  totalLinesRemoved: number,
  modelUsage: {
    [modelName: string]: {
      inputTokens: number,
      outputTokens: number,
      cacheReadInputTokens: number,
      cacheCreationInputTokens: number,
      webSearchRequests: number,
      costUSD: number,
      contextWindow: number,
      maxOutputTokens: number,
    }
  }
}
```

### Accumulation par appel API

**Fonction** : `addToTotalSessionCost()` dans `cost-tracker.ts`

```
1. calculateUSDCost(model, usage) → coût de cet appel
2. addToTotalModelUsage(cost, usage, model) → met à jour le modèle
3. addToTotalCostState(cost, modelUsage, model) → met à jour l'état global
4. Log OpenTelemetry :
   ├── costCounter.add(cost, { model, speed })
   ├── tokenCounter.add(input, { type: 'input' })
   ├── tokenCounter.add(output, { type: 'output' })
   ├── tokenCounter.add(cache_read, { type: 'cacheRead' })
   └── tokenCounter.add(cache_creation, { type: 'cacheCreation' })
```

### Persistance sur disque

Sauvegarde dans `~/.claude/config.json` :

```json
{
  "lastCost": 1.234,
  "lastAPIDuration": 45000,
  "lastSessionId": "uuid-session",
  "lastModelUsage": {
    "claude-opus-4-6": {
      "inputTokens": 150000,
      "outputTokens": 8000,
      "costUSD": 1.234
    }
  }
}
```

### Restauration

`restoreCostStateForSession(sessionId)` restaure les coûts quand une session est reprise (`--resume`).

---

## 7. Rate limits et quotas

### Fenêtres de rate limit

| Type | Fenêtre | Nom affiché | Qui est affecté |
|------|---------|-------------|-----------------|
| `five_hour` | 5 heures | "session limit" | Tous les abonnés |
| `seven_day` | 7 jours | "weekly limit" | Tous les abonnés |
| `seven_day_opus` | 7 jours | "Opus limit" | Max, Pro, Enterprise |
| `seven_day_sonnet` | 7 jours | "Sonnet limit" | Pro, Enterprise |
| `overage` | Cycle mensuel | "extra usage limit" | Team, Enterprise (si provisionné) |

### Statuts de quota

| Statut | Description | Action |
|--------|-------------|--------|
| `allowed` | OK, pas de restriction | Requêtes normales |
| `allowed_warning` | Approche de la limite | Avertissement affiché |
| `rejected` | Limite atteinte | 429, requêtes bloquées |

### Headers de réponse API

Chaque réponse API inclut les headers de rate limit :

```
anthropic-ratelimit-unified-status: allowed|allowed_warning|rejected
anthropic-ratelimit-unified-reset: <unix-timestamp>
anthropic-ratelimit-unified-representative-claim: five_hour|seven_day|...
anthropic-ratelimit-unified-5h-utilization: 0.45
anthropic-ratelimit-unified-5h-reset: 1711900800
anthropic-ratelimit-unified-7d-utilization: 0.23
anthropic-ratelimit-unified-7d-reset: 1712505600
anthropic-ratelimit-unified-overage-status: allowed|allowed_warning|rejected
anthropic-ratelimit-unified-overage-disabled-reason: <reason>
anthropic-ratelimit-unified-fallback: available|unavailable
```

### Fallback Opus → Sonnet

Quand la limite Opus est atteinte (`seven_day_opus: rejected`) :
- Si `unifiedRateLimitFallbackAvailable: true`
- Le système peut automatiquement utiliser Sonnet
- L'utilisateur est notifié du fallback

---

## 8. Système d'overage (extra usage)

### Qui peut l'utiliser

Conditions :
1. Abonné Claude (subscriber)
2. Billing type compatible : `stripe_subscription`, `stripe_subscription_contracted`, `apple_subscription`, `google_play_subscription`
3. Provisionné au niveau organisation

### Données d'overage

```typescript
ExtraUsage {
  is_enabled: boolean,        // Overage activé ?
  monthly_limit: number,      // Plafond mensuel (null = illimité)
  used_credits: number,       // Déjà dépensé ce mois
  utilization: number,        // Pourcentage (0-100)
}
```

### Raisons de désactivation

| Raison | Description |
|--------|-------------|
| `overage_not_provisioned` | Pas configuré pour ce tier |
| `org_level_disabled` | Admin a désactivé |
| `org_level_disabled_until` | Temporairement désactivé (date) |
| `out_of_credits` | Budget organisation épuisé |
| `seat_tier_level_disabled` | Tier ne supporte pas |
| `member_level_disabled` | Utilisateur spécifiquement désactivé |
| `seat_tier_zero_credit_limit` | Limite du tier = $0 |
| `group_zero_credit_limit` | Limite du groupe = $0 |
| `member_zero_credit_limit` | Limite de l'utilisateur = $0 |
| `org_service_level_disabled` | Service désactivé |
| `org_service_zero_credit_limit` | Limite service = $0 |
| `no_limits_configured` | Rien configuré |

### Crédit grant (promotionnel)

```typescript
OverageCreditGrantInfo {
  available: boolean,          // Disponible ?
  eligible: boolean,           // Rôle autorise ?
  granted: boolean,            // Déjà réclamé ?
  amount_minor_units: number,  // Montant en centimes
  currency: string,            // 'USD'
}
```

- **API** : `/api/oauth/organizations/{orgUUID}/overage_credit_grant`
- **Cache** : 1 heure
- **Affichage** : `formatGrantAmount(info)` → "$X.XX"

### Flux overage

```
Limite standard épuisée (rejected)
    ↓
Vérification overage disponible ?
    ├── NON → 429, utilisateur bloqué
    └── OUI → Basculement automatique
         ├── isUsingOverage = true
         ├── Notification : "You're now using extra usage"
         ├── Requêtes continuent avec crédits overage
         └── Quand overage épuisé :
              └── overageStatus: rejected → utilisateur bloqué
```

---

## 9. Avertissements précoces

### Seuils d'avertissement

Le système avertit l'utilisateur **avant** d'atteindre les limites, basé sur le ratio utilisation/temps écoulé :

#### Limite 5 heures

| Utilisation | Temps écoulé max | Message |
|-------------|-----------------|---------|
| ≥ 90% | ≤ 72% de la fenêtre | "Approaching session limit" |

#### Limite 7 jours

| Utilisation | Temps écoulé max | Message |
|-------------|-----------------|---------|
| ≥ 75% | ≤ 60% de la fenêtre | "You've used 75% of your weekly limit" |
| ≥ 50% | ≤ 35% | "You've used 50%..." |
| ≥ 25% | ≤ 15% | "You've used 25%..." |

**Logique** : Un avertissement n'est émis que si l'utilisation est disproportionnée par rapport au temps restant. Utiliser 75% en 6 jours sur 7 est normal — utiliser 75% en 1 jour ne l'est pas.

**Fichier** : `services/claudeAiLimits.ts`

---

## 10. Messages d'erreur rate limit

### Messages par état

**Avertissement (allowed_warning)** :
```
"You've used 75% of your weekly limit · resets in 2 days"
"Approaching session limit · resets in 1h 30m"
```

**Rejeté (rejected)** :
```
"You've hit your weekly limit · resets in 2 days"
"You've hit your Opus limit · resets in 5 days"
```

**Transition overage** :
```
"You're now using extra usage · Your weekly limit resets in 2 days"
```

**Approche du plafond overage** :
```
"You're close to your extra usage spending limit"
```

**Standard ET overage épuisés** :
```
"You're out of extra usage · resets in 2 days"
```

### Upsell selon le tier

| Tier | Message d'upsell |
|------|-----------------|
| Pro | `/upgrade to keep using Claude Code` |
| Max | (pas d'upsell, tier max) |
| Team/Enterprise sans overage | `/extra-usage to request more` |
| Team/Enterprise avec overage | (notification de transition) |

**Fichier** : `services/rateLimitMessages.ts`

---

## 11. Commandes utilisateur

### `/cost` — Afficher les coûts

**Pour les utilisateurs API key** :
```
Total cost:            $1.23
Total duration (API):  2m 15s
Total duration (wall): 5m 30s
Total code changes:    45 lines added, 12 lines removed

Usage by model:
  claude-opus-4-6:     150K input, 8K output, 120K cache read, 30K cache write ($1.23)
```

**Pour les abonnés Claude** :
```
You are currently using your subscription to power your Claude Code usage.
```
Ou si en overage :
```
You are currently using your overages to power your Claude Code usage.
We will automatically switch you back when your subscription rate limits reset.
```

### `/extra-usage` — Demander plus d'usage

**Pour les non-billing Team/Enterprise** :
- Vérifie si overage déjà illimité
- Vérifie l'éligibilité admin
- Crée une requête admin pour `limit_increase`

**Pour les billing users** :
- Ouvre le navigateur vers `claude.ai/admin-settings/usage` (teams) ou `claude.ai/settings/usage` (consumer)

### `/rate-limit-options` — Menu interactif

Affiché quand un rate limit est atteint :
- "Stop and wait for limit to reset"
- "Upgrade to Claude Max" (si Pro)
- "/extra-usage to request more" (si Team/Enterprise)

### `/usage` — Barres d'utilisation

Affiche des barres de progression visuelles pour chaque fenêtre de rate limit active.

---

## 12. Contrôle d'accès à la facturation

### Qui voit les coûts

**Fonction** : `hasConsoleBillingAccess()` dans `utils/billing.ts`

**Utilisateurs Console/API Key** :
- Requiert rôle `admin` ou `billing` au niveau organisation
- OU rôle `workspace_admin` ou `workspace_billing` au niveau workspace

**Abonnés Claude.ai** (`hasClaudeAiBillingAccess()`) :
- Max/Pro : toujours accès
- Team/Enterprise : requiert `admin`, `billing`, `owner`, ou `primary_owner`

### Désactivation

Variable d'environnement : `DISABLE_COST_WARNINGS=true` → désactive tout l'affichage des coûts.

### Formatage des coûts

| Montant | Précision | Exemple |
|---------|-----------|---------|
| ≥ $0.50 | 2 décimales | $1.23 |
| < $0.50 | 4 décimales | $0.0042 |

Arrondi bancaire (banker's rounding) au centième le plus proche.

---

## 13. Policy limits (admin)

### Qui peut imposer des policies

- **Team** et **Enterprise** uniquement (OAuth)
- Pas les Free/Pro/Max
- Pas les utilisateurs Console/API Key

### Format des policies

```typescript
PolicyLimitsResponse {
  restrictions: {
    "allow_remote_sessions": { allowed: false },
    "allow_feedback": { allowed: true },
    // ...
  }
}
```

### Vérification

```typescript
isPolicyAllowed('allow_remote_sessions')
// true si pas de restriction ou explicitement autorisé
// false si restriction active
```

### Comportement

- **API** : `GET /api/claude_code/policy_limits`
- **Cache** : ETag HTTP + fichier local `~/.claude/policy-limits.json`
- **Polling** : Toutes les heures en arrière-plan
- **Fail-open** : Si le fetch échoue et pas de cache → autorisé par défaut

---

## 14. Referral et guest passes

### Système de parrainage (Claude Max)

```typescript
ReferralEligibilityResponse {
  eligible: boolean,          // Peut générer des passes ?
  remaining_passes: number,   // Combien restent ce mois ?
  referrer_reward: number,    // Crédit par parrainage réussi
  currency: string,           // Devise de la récompense
}
```

- **API** : `/api/oauth/organizations/{orgUUID}/referral/eligibility`
- **Campagne** : `claude_code_guest_pass`
- **Éligibilité** : Claude Max uniquement, scope `user:profile`
- **Cache** : 24 heures par organisation

---

## 15. Télémétrie et analytics

### Compteurs OpenTelemetry

| Compteur | Dimensions | Description |
|----------|-----------|-------------|
| `costCounter` | model, speed | Coût en USD par appel |
| `tokenCounter` | type (input/output/cacheRead/cacheCreation) | Tokens par type |
| `sessionCounter` | — | Métriques de session |
| `locCounter` | — | Lignes de code modifiées |
| `prCounter` | — | PRs créées |
| `commitCounter` | — | Commits effectués |

### Événements analytics

| Événement | Quand |
|-----------|-------|
| `tengu_api_success` | Chaque appel API réussi |
| `tengu_advisor_tool_token_usage` | Usage advisor avec coût en micros |
| `tengu_unknown_model_cost` | Modèle sans pricing connu |
| `tengu_auto_mode_decision` | Décision du classifieur (inclut usage tokens) |

---

## 16. Fichiers clés

### Coûts et tokens

| Fichier | Rôle |
|---------|------|
| `utils/modelCost.ts` | Grille tarifaire, `calculateUSDCost()` |
| `utils/tokens.ts` | Extraction tokens, `tokenCountWithEstimation()` |
| `cost-tracker.ts` | Accumulation, formatage, persistance session |
| `bootstrap/state.ts` | État global (totalCostUSD, modelUsage) |
| `services/tokenEstimation.ts` | Estimation pré-API (rough, API, Haiku fallback) |
| `costHook.ts` | React hook pour affichage coûts en sortie |

### Rate limits et quotas

| Fichier | Rôle |
|---------|------|
| `services/claudeAiLimits.ts` | État quotas, seuils d'avertissement, headers |
| `services/rateLimitMessages.ts` | Messages utilisateur, upsell |
| `services/api/usage.ts` | Client API usage (`/api/oauth/usage`) |
| `services/api/overageCreditGrant.ts` | Crédits overage promotionnels |
| `services/mockRateLimits.ts` | Mock rate limits (Ant testing) |

### Abonnements et comptes

| Fichier | Rôle |
|---------|------|
| `utils/billing.ts` | Contrôle d'accès facturation |
| `utils/auth.ts` | Détection subscription, éligibilité overage |
| `services/oauth/client.ts` | Profil OAuth, mapping tiers |
| `services/policyLimits/index.ts` | Policies admin Team/Enterprise |
| `services/api/referral.ts` | Guest passes et parrainages |
| `services/api/firstTokenDate.ts` | Date de premier usage |

### Commandes et UI

| Fichier | Rôle |
|---------|------|
| `commands/cost/cost.ts` | Commande `/cost` |
| `commands/extra-usage/extra-usage-core.ts` | Commande `/extra-usage` |
| `commands/rate-limit-options/rate-limit-options.tsx` | Menu interactif rate limit |
| `components/CostThresholdDialog.tsx` | Dialogue d'avertissement de coût |
| `components/Settings/Usage.tsx` | Barres d'utilisation visuelles |
