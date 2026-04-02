# Connexions aux APIs et Persistance des Données — Documentation complète

Ce document décrit l'ensemble des connexions externes, des mécanismes d'authentification et du système de persistance des données dans Claude Code.

> **Point clé** : Claude Code n'utilise **aucune base de données traditionnelle** (SQL, NoSQL, ORM). Toute la persistance repose sur un système de fichiers JSON/JSONL append-only.

---

## Table des matières

1. [Architecture de persistance (pas de BDD)](#1-architecture-de-persistance)
2. [API Claude — Connexion principale](#2-api-claude)
3. [Providers cloud — Bedrock, Vertex, Foundry](#3-providers-cloud)
4. [Système OAuth](#4-système-oauth)
5. [Gestion des credentials](#5-gestion-des-credentials)
6. [APIs internes Anthropic](#6-apis-internes-anthropic)
7. [MCP — Connexions aux serveurs externes](#7-mcp)
8. [Outils web — WebFetch et WebSearch](#8-outils-web)
9. [Télémétrie — GrowthBook et Datadog](#9-télémétrie)
10. [Teleport et sessions distantes (CCR)](#10-teleport-et-ccr)
11. [Bridge — Intégrations IDE/Chrome](#11-bridge)
12. [Proxy et réseau](#12-proxy-et-réseau)
13. [Retry et rate limiting](#13-retry-et-rate-limiting)
14. [Sécurité des credentials](#14-sécurité-des-credentials)
15. [Variables d'environnement — Référence complète](#15-variables-denvironnement)
16. [Fichiers clés](#16-fichiers-clés)

---

## 1. Architecture de persistance

### Pas de base de données

Claude Code **ne contient aucun** :
- Import de bibliothèque BDD (Prisma, Sequelize, TypeORM, Knex, pg, mysql2, mongoose, better-sqlite)
- Connection string ou credential BDD
- Migration de schéma
- Connection pool ou requête SQL

### Stockage fichier — JSONL append-only

Toute la persistance utilise des fichiers JSON Lines (JSONL) sur le filesystem local.

#### Structure des répertoires

```
~/.claude/
├── config.json                          # Configuration globale
├── settings.json                        # Settings utilisateur
├── keybindings.json                     # Raccourcis clavier
├── history.jsonl                        # Historique global des commandes
├── managed/
│   ├── managed-settings.json            # Settings admin/MDM
│   └── managed-settings.d/             # Snippets de config drop-in
├── teams/                               # Mémoire d'équipe multi-agents
│   └── {teamName}/
│       ├── config.json                  # Configuration d'équipe
│       ├── inboxes/{agentName}.json     # Mailboxes inter-agents
│       └── permissions/                 # Requêtes de permissions
└── projects/
    └── {sanitized-cwd}/
        ├── {sessionId}.jsonl            # Transcript (append-only)
        ├── {sessionId}/
        │   └── tool-results/            # Sorties d'outils volumineuses
        │       └── {toolUseId}.txt
        ├── memory/                      # Mémoire de session (markdown)
        └── .claude/
            └── settings.json            # Config niveau projet
```

#### Mécanismes de stockage

| Système | Emplacement | Format | Mode d'écriture |
|---------|-------------|--------|-----------------|
| **Transcripts** | `{sessionId}.jsonl` | JSON Lines | Append-only, batch 100ms, max 100MB/chunk |
| **Historique** | `history.jsonl` | JSON Lines | Append, lecture inversée |
| **Tool results** | `tool-results/{id}.txt` | Texte brut | Écriture unique (seuil 50KB) |
| **Mémoire** | `memory/*.md` | Markdown | Mise à jour via sous-agent |
| **Config** | `*.json` | JSON | Read-modify-write |

#### Écriture batch

```typescript
// Classe Project — sessionStorage.ts
1. Messages mis en file : pendingEntries[]
2. Drain planifié : scheduleDrain() (intervalle 100ms)
3. Files d'écriture par fichier : writeQueues Map
4. Création auto des répertoires au premier write
5. Hook de shutdown : flush garanti en sortie
6. Permissions fichier : 0o600 (lecture/écriture propriétaire uniquement)
```

#### Types d'entrées dans les transcripts

| Type | Description |
|------|-------------|
| `user` | Messages utilisateur |
| `assistant` | Réponses Claude |
| `attachment` | Contenu attaché |
| `system` | Messages système |
| `contextCollapseSectionEntry` | Métadonnées de compression |
| `fileHistorySnapshot` | Tracking d'historique fichiers |
| `contentReplacement` | Éditions de contenu |
| `attributionSnapshot` | Données d'attribution git |

#### State management in-memory

| Composant | Fichier | Description |
|-----------|---------|-------------|
| **AppState** | `state/AppStateStore.ts` | Store Zustand, session-scoped |
| **Bootstrap state** | `bootstrap/state.ts` | Singleton global (56KB), coûts/tokens/session |

---

## 2. API Claude — Connexion principale

### Client Anthropic

**Fichier** : `services/api/client.ts`

```
Claude Code ──── @anthropic-ai/sdk ────► api.anthropic.com/api/messages
```

| Propriété | Détail |
|-----------|--------|
| **SDK** | `@anthropic-ai/sdk` |
| **Transport** | HTTP POST avec streaming SSE |
| **Endpoint** | `/api/messages` (configurable via `ANTHROPIC_BASE_URL`) |
| **Headers custom** | Via `ANTHROPIC_CUSTOM_HEADERS` |
| **Timeout** | Configurable via `API_TIMEOUT_MS` |

### Authentification (par priorité)

```
1. OAuth Token      → CLAUDE_CODE_OAUTH_TOKEN ou token file descriptor
2. Auth Token       → ANTHROPIC_AUTH_TOKEN (Bearer)
3. API Key Helper   → Commande externe avec TTL 5min
4. API Key          → ANTHROPIC_API_KEY (env) ou Keychain
5. API Key FD       → CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR (CI/sandbox)
```

### Initialisation du client

```typescript
getAnthropicClient() {
  // Détecte le provider
  switch (getAPIProvider()) {
    case 'bedrock':  return new AnthropicBedrock(bedrockConfig)
    case 'vertex':   return new AnthropicVertex(vertexConfig)
    case 'foundry':  return new AnthropicFoundry(foundryConfig)
    case 'firstParty': return new Anthropic(anthropicConfig)
  }
}
```

---

## 3. Providers cloud

### 3.1 AWS Bedrock

| | |
|---|---|
| **SDK** | `@anthropic-ai/bedrock-sdk` |
| **Activation** | `CLAUDE_CODE_USE_BEDROCK=true` |
| **Skip auth** | `CLAUDE_CODE_SKIP_BEDROCK_AUTH=true` (proxy) |

**Authentification (par priorité)** :

| Source | Mécanisme |
|--------|-----------|
| Bearer Token | `AWS_BEARER_TOKEN_BEDROCK` (API key auth) |
| STS Credentials | `awsCredentialExport` commande + cache |
| AWS SDK | Fichier credentials, env vars, instance metadata |
| IAM | `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` + `AWS_SESSION_TOKEN` |

**Configuration région** :

| Variable | Usage |
|----------|-------|
| `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION` | Override pour Haiku |
| `AWS_REGION` / `AWS_DEFAULT_REGION` | Région globale (défaut: us-east-1) |

**Refresh credentials** :
- TTL par défaut : 1 heure (expiration STS)
- Invalidation automatique sur 403 "security token invalid"
- Cache vidé avant chaque appel API si nécessaire

---

### 3.2 Google Vertex AI

| | |
|---|---|
| **SDK** | `@anthropic-ai/vertex-sdk` |
| **Activation** | `CLAUDE_CODE_USE_VERTEX=true` |
| **Skip auth** | `CLAUDE_CODE_SKIP_VERTEX_AUTH=true` (proxy) |

**Authentification (par priorité)** :

| Source | Mécanisme |
|--------|-----------|
| Service Account | `GOOGLE_APPLICATION_CREDENTIALS` (fichier JSON) |
| ADC | Application Default Credentials (gcloud config, metadata) |
| Projet | `GCLOUD_PROJECT` / `GOOGLE_CLOUD_PROJECT` |
| Fallback | `ANTHROPIC_VERTEX_PROJECT_ID` (évite timeout metadata 12s) |

**Configuration région** :

| Variable | Usage |
|----------|-------|
| `VERTEX_REGION_CLAUDE_3_5_HAIKU` | Région spécifique Haiku 3.5 |
| `VERTEX_REGION_CLAUDE_HAIKU_4_5` | Région spécifique Haiku 4.5 |
| `VERTEX_REGION_CLAUDE_3_5_SONNET` | Région spécifique Sonnet 3.5 |
| `VERTEX_REGION_CLAUDE_3_7_SONNET` | Région spécifique Sonnet 3.7 |
| `CLOUD_ML_REGION` | Région globale (défaut: us-east5) |

---

### 3.3 Azure Foundry

| | |
|---|---|
| **SDK** | `@anthropic-ai/foundry-sdk` |
| **Activation** | `CLAUDE_CODE_USE_FOUNDRY=true` |
| **Skip auth** | `CLAUDE_CODE_SKIP_FOUNDRY_AUTH=true` (proxy) |

**Authentification (par priorité)** :

| Source | Mécanisme |
|--------|-----------|
| API Key | `ANTHROPIC_FOUNDRY_API_KEY` |
| Azure AD | DefaultAzureCredential (env, managed identity, Azure CLI) |

**Configuration** :

| Variable | Usage |
|----------|-------|
| `ANTHROPIC_FOUNDRY_RESOURCE` | Nom de la ressource (ex: "my-resource") |
| `ANTHROPIC_FOUNDRY_BASE_URL` | URL de base alternative |

---

### Résumé multi-providers

```
                    ┌─── firstParty ──► api.anthropic.com
                    │
getAPIProvider() ───┼─── bedrock ─────► AWS Bedrock (SigV4)
                    │
                    ├─── vertex ──────► Google Vertex AI (OAuth2)
                    │
                    └─── foundry ─────► Azure Foundry (Azure AD)
```

Quand un provider 3P est actif :
- OAuth Anthropic est **désactivé** (`isAnthropicAuthEnabled() → false`)
- Les credentials du provider sont utilisées exclusivement
- La télémétrie Datadog est désactivée

---

## 4. Système OAuth

### 4.1 OAuth first-party (Claude)

**Fichier** : `services/oauth/client.ts`

**Flow OAuth 2.0 avec PKCE** :

```
1. Génération code_verifier + code_challenge (S256)
2. Redirection navigateur → claude.com/cai/oauth/authorize
3. Utilisateur approuve
4. Callback localhost avec authorization code
5. POST /v1/oauth/token (code + code_verifier)
6. Réception : access_token, refresh_token, expires_in
```

**Endpoints** :

| Endpoint | URL (production) |
|----------|-----------------|
| Authorization | `https://claude.com/cai/oauth/authorize` |
| Token | `https://platform.claude.com/v1/oauth/token` |
| Profile | `{BASE_URL}/api/oauth/profile` |
| Roles | `{BASE_URL}/api/oauth/roles` |
| API Key | `{BASE_URL}/api/oauth/api_key` |
| Usage | `{BASE_URL}/api/oauth/usage` |

**Scopes** :

| Scope | Accès |
|-------|-------|
| `user:profile` | Profil / compte |
| `user:inference` | API Claude (tokens long-lived) |
| `user:sessions:claude_code` | Gestion de sessions |
| `user:mcp_servers` | Connecteurs MCP |
| `user:file_upload` | Upload de fichiers |

**Refresh automatique** :
- Détecte l'expiration imminente
- POST avec `grant_type: refresh_token`
- Cache le profil (billing, subscription) dans config globale
- Normalise les codes d'erreur non-standard (`invalid_refresh_token` → `invalid_grant`)

---

### 4.2 OAuth MCP (serveurs externes)

**Fichier** : `services/mcp/auth.ts` (2400+ lignes)

Flow complet pour l'authentification des serveurs MCP :

```
1. RFC 8414/9728 — Découverte des métadonnées OAuth du serveur
2. Dynamic Client Registration (DCR) ou client ID pré-configuré
3. State CSRF (32 octets aléatoires)
4. Construction URL d'autorisation avec scopes du WWW-Authenticate
5. Serveur callback local sur port éphémère
6. Redirection navigateur → URL d'autorisation PKCE
7. Échange code → tokens (code_verifier PKCE)
8. Refresh automatique avec verrou inter-processus (5 retries max)
9. Stockage tokens dans le Keychain avec suivi d'expiration
10. Révocation RFC 7009 sur déconnexion
```

**Timeout** : 5 minutes pour le flow complet (avec `unref()` pour ne pas bloquer l'event loop)

---

### 4.3 XAA Enterprise (Cross-App Access)

**Fichier** : `services/mcp/xaaIdpLogin.ts`

Authentification enterprise via OIDC avec cache d'id_token :

```
1. Login OIDC auprès de l'IdP enterprise → id_token
2. Cache id_token dans le Keychain (clé: issuerKey)
3. Pour chaque serveur MCP :
   RFC 8693 Token Exchange (id_token → ID-JAG)
   RFC 7523 JWT Bearer Grant (ID-JAG → access_token)
4. Résultat : auth transparente N serveurs, 1 seul login navigateur
```

| Config | Description |
|--------|-------------|
| `settings.xaaIdp.issuer` | URL de l'IdP (normalisée) |
| `settings.xaaIdp.clientId` | Client ID de l'IdP |
| `settings.xaaIdp.callbackPort` | Port fixe si l'IdP requiert un redirect URI spécifique |
| `CLAUDE_CODE_ENABLE_XAA` | Activation (`=1`) |

**Buffer d'expiration** : 60 secondes (ré-acquisition si ≤60s restant)

---

## 5. Gestion des credentials

### 5.1 Stockage sécurisé

**macOS Keychain** (`utils/secureStorage/macOsKeychainStorage.ts`) :

| Propriété | Détail |
|-----------|--------|
| Outil | `security add-generic-password` / `find-generic-password` |
| Encodage | Hexadécimal (flag `-X`) — empêche les leaks dans le monitoring de processus |
| Service | `Claude Code {username} Credentials` |
| Cache TTL | 30 secondes par lecture Keychain |
| Invalidation | Tracking de génération pour détecter les modifications cross-processus |
| Fallback | Stockage plaintext si lecture échoue (stale-while-error) |

**Structure stockée** :

```typescript
SecureStorageData {
  mcpOAuth: {
    "{serverName|configHash}": {
      accessToken: string,
      refreshToken: string,
      expiresAt: number,
      clientId: string,
      clientSecret: string,
      discoveryState?: object  // pour XAA
    }
  },
  mcpXaaIdp: {
    "{issuerKey}": {
      idToken: string,
      expiresAt: number
    }
  },
  mcpOAuthClientConfig: {
    "{serverName|configHash}": {
      clientSecret: string
    }
  }
}
```

**Linux** : Stockage fichier plaintext (support libsecret prévu).

---

### 5.2 API Key Helper

Commande externe dynamique pour récupérer les clés API :

```
apiKeyHelper: "vault read -field=key secret/anthropic"
```

| Propriété | Détail |
|-----------|--------|
| TTL | 5 min (configurable via `CLAUDE_CODE_API_KEY_HELPER_TTL_MS`) |
| Stale-while-revalidate | Retourne la clé périmée pendant le refresh en arrière-plan |
| Déduplication | Les appels concurrents sur cache froid sont dédupliqués |
| Trust | Bloqué jusqu'à acceptation du workspace trust (settings projet) |
| Timeout | 3 min pour `awsAuthRefresh` / `gcpAuthRefresh` |

---

### 5.3 Invalidation des caches

| Cache | Fonction | Déclencheur |
|-------|----------|-------------|
| OAuth | `clearOAuthTokenCache()` | Force relecture Keychain |
| API Key Helper | `clearApiKeyHelperCache()` | Bump d'epoch, orpheline les exécutions pendantes |
| AWS | `clearAwsCredentialsCache()` | 403 "expired STS" |
| GCP | `clearGcpCredentialsCache()` | "Could not refresh access token" |
| Keychain | `clearKeychainCache()` | Invalidation MCP/XAA/legacy |

---

## 6. APIs internes Anthropic

### 6.1 Bootstrap API

```
GET {BASE_URL}/api/claude_cli/bootstrap
```

Récupère la configuration distante (modèles, options client). Auth : OAuth Bearer ou API Key.

### 6.2 Usage & Billing API

```
GET {BASE_URL}/api/oauth/usage
```

Rate limits (5h et 7j), statut overage, limites par modèle. Auth : OAuth (`user:profile`).

### 6.3 Grove Privacy Settings

```
GET/POST {BASE_URL}/api/oauth/account/settings
```

Paramètres de confidentialité (grove_enabled, notice_viewed_at). Auth : OAuth. Cache 24h.

### 6.4 Session Ingress (logging distant)

```
PUT {INGRESS_URL}/sessions/{sessionId}/{resourceType}
```

Streaming des transcripts vers le serveur distant (mode CCR). Auth : JWT session-specific. Écriture séquentielle avec UUIDs idempotents.

### 6.5 Public Files API

```
GET https://api.anthropic.com/api/files/{file_id}/content
```

Téléchargement de fichiers attachés. Auth : OAuth (`user:inference`). Retry avec backoff exponentiel.

### 6.6 Autres services

| Service | Endpoint | Description |
|---------|----------|-------------|
| Admin Requests | `api/admin/*` | Opérations admin (Ant-only) |
| Metrics Opt-Out | `api/oauth/metrics-opt-out` | Préférences de télémétrie |
| Referral | `api/oauth/referral` | Programme de parrainage |
| UltraReview Quota | `api/ultrareview/quota` | Quotas UltraReview |
| Overage Credit | `api/oauth/overage-credit` | Gestion des crédits extra |
| First Token Date | `api/oauth/first-token` | Date du premier token |

Tous utilisent OAuth + axios.

---

## 7. MCP — Connexions aux serveurs externes

### Transports supportés

| Transport | SDK | Usage |
|-----------|-----|-------|
| **stdio** | `StdioClientTransport` | Subprocess local (commande + args) |
| **SSE** | `SSEClientTransport` | HTTP long-polling bidirectionnel |
| **HTTP** | `StreamableHTTPClientTransport` | HTTP/1.1 streaming |
| **WebSocket** | `WebSocketTransport` | WS/WSS avec support TLS/mTLS |

### Authentification MCP

| Méthode | Description |
|---------|-------------|
| **OAuth standard** | Flow PKCE complet avec DCR |
| **XAA** | RFC 8693/7523 via IdP enterprise |
| **Custom headers** | `getMcpServerHeaders()` par serveur |
| **Session token** | `getSessionIngressAuthToken()` pour CCR |

### Gestion des outils MCP

- Nommage : `{serveur}__{outil}` (double underscore)
- Chargement lazy via `defer_loading`
- Intégration tool search
- Cache et troncature des résultats
- Persistance du contenu binaire

**Fichier principal** : `services/mcp/client.ts` (1000+ lignes)

---

## 8. Outils web

### 8.1 WebFetch

| | |
|---|---|
| **Transport** | `fetch()` natif |
| **User-Agent** | `Claude-User` |
| **Cache** | 15 min |

- Conversion HTML → Markdown
- Liste de domaines pré-approuvés (github.com/anthropics, anthropic.com, etc.)
- Protection SSRF (bloque accès réseau local)
- Permission utilisateur requise par domaine

### 8.2 WebSearch

| | |
|---|---|
| **Transport** | Via API Claude (server-side) |
| **Type** | `web_search_20250305` |
| **Max recherches** | 8 par requête |

- Recherche déléguée à l'API (pas d'appel direct externe)
- Filtrage domaines autorisés/bloqués
- Intégré dans la complétion de message Claude

---

## 9. Télémétrie

### 9.1 GrowthBook (Feature Flags)

**Fichier** : `services/analytics/growthbook.ts`

```
Claude Code ──── @growthbook/growthbook ────► GrowthBook API
```

| Propriété | Détail |
|-----------|--------|
| Auth | SDK Client Key (publique, pas de secret) |
| Clé Ant | `sdk-xRVcrliHIlrg4og4` (prod) / `sdk-yZQvlplybuXjYh6L` (dev) |
| Clé externe | `sdk-zAZezfDKGoZuXXKe` |
| Refresh | Périodique avec cache |
| Ré-init | Sur changement d'auth |

**Attributs utilisateur envoyés** : userId (hash), sessionId, deviceID, platform, accountUUID, organizationUUID, subscription type, rate limit tier, user type, app version, email, GitHub Actions metadata.

### 9.2 Datadog (Logging)

**Fichier** : `services/analytics/datadog.ts`

```
Claude Code ──── axios POST ────► https://http-intake.logs.us5.datadoghq.com/api/v2/logs
```

| Propriété | Détail |
|-----------|--------|
| Auth | Client token public : `pubbbf48e6d78dae54bceaa4acf463299bf` |
| Batch | Max 100 événements, flush 15 secondes |
| Whitelist | Événements autorisés uniquement (erreurs API, OAuth, usage outils) |
| Tags | event, tool_name, model, version, etc. |
| Privacy | Hash bucket utilisateur, noms MCP normalisés à "mcp" |
| Désactivé pour | Providers 3P (Bedrock, Vertex, Foundry) |

### 9.3 First-Party Event Logging

**Fichier** : `services/analytics/firstPartyEventLoggingExporter.ts`

Infrastructure propriétaire Anthropic. Sampling configuré via GrowthBook. Batch processing.

---

## 10. Teleport et sessions distantes (CCR)

### API Teleport

**Fichier** : `utils/teleport/api.ts`

```
Claude Code ──── axios ────► Teleport API (environnements, sessions)
```

| Propriété | Détail |
|-----------|--------|
| Auth | OAuth Bearer + Organization UUID |
| Retry | Backoff exponentiel (2s, 4s, 8s, 16s) |
| Opérations | List/create environments, manage sessions, git bundle |

### Session Ingress Auth

**Fichier** : `utils/sessionIngressAuth.ts`

**Sources de token (par priorité)** :

```
1. CLAUDE_CODE_SESSION_ACCESS_TOKEN  → env var (mis à jour par bridge)
2. CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR → file descriptor (lecture unique)
3. Fichier well-known → CLAUDE_SESSION_INGRESS_TOKEN_FILE ou ~/.claude/remote/.session_ingress_token
```

**Types de token** :

| Format | Auth Header |
|--------|-------------|
| `sk-ant-sid-*` (Session Key) | `Cookie: sessionKey={token}` + `X-Organization-Uuid` |
| JWT | `Authorization: Bearer {token}` |

### Contexte OAuth managé

Quand `CLAUDE_CODE_REMOTE=1` ou entrypoint `claude-desktop` :
- Ignore le `apiKeyHelper` de l'utilisateur
- Ignore `ANTHROPIC_API_KEY`
- Ignore les clés Keychain gérées par `/login`
- Utilise exclusivement les tokens fournis par le launcher

---

## 11. Bridge — Intégrations IDE/Chrome

**Fichier** : `bridge/bridgeApi.ts`, `bridge/codeSessionApi.ts`

```
Claude Code ◄──── polling HTTP + WebSocket ────► IDE/Chrome Extension
```

| Propriété | Détail |
|-----------|--------|
| Auth | Bearer token + `X-Trusted-Device-Token` |
| Headers | `anthropic-beta`, `x-environment-runner-version` |
| Opérations | Poll work items, submit results, permission requests, stream files |

---

## 12. Proxy et réseau

### Support proxy

**Fichier** : `utils/proxy.ts`, `upstreamproxy/upstreamproxy.ts`

| Variable | Usage |
|----------|-------|
| `HTTP_PROXY` | Proxy HTTP |
| `HTTPS_PROXY` | Proxy HTTPS |
| `ALL_PROXY` | Proxy universel |
| `NO_PROXY` | Exclusions (localhost, 127.0.0.1, etc.) |
| `CLAUDE_CODE_CA_CERTS` | Certificats CA custom |

**Fonctionnalités** :
- Tunneling CONNECT pour HTTPS
- Support proxy WebSocket
- Authentification proxy (credentials dans l'URL)
- Liste de bypass configurable

---

## 13. Retry et rate limiting

### Fichier clé : `services/api/withRetry.ts`

### Stratégie de retry

```
Base delay : 500ms
Backoff : exponentiel × 2^(attempt-1)
Max delay : 32 secondes
Jitter : 25%
Max retries : 10 (configurable via CLAUDE_CODE_MAX_RETRIES)
```

### Codes HTTP gérés

| Code | Action |
|------|--------|
| **401** | Force refresh OAuth, nouveau client |
| **403** | Idem 401 ; aussi "OAuth token revoked", STS expired (Bedrock) |
| **408/409** | Retry standard |
| **429** | Rate limit → backoff avec `Retry-After` |
| **500+** | Retry (sauf `x-should-retry: false`) |
| **529** | Overloaded → backoff ; fallback modèle après N échecs (Opus → Sonnet) |

### Modes spéciaux

| Mode | Comportement |
|------|-------------|
| **Fast mode** | Retries courts pour préserver le cache de prompt |
| **Persistent retry** | Sessions non-assistées : retry 429/529 pendant 6h max |
| **Ant mode** | Retry 5xx même avec `x-should-retry: false` |
| **CCR mode** | Retry 401/403 (flap du service auth, pas de mauvaises credentials) |

### Headers exploités

| Header | Usage |
|--------|-------|
| `Retry-After` | Délai d'attente (RFC 7231) |
| `anthropic-ratelimit-unified-reset` | Reset de fenêtre (Unix seconds) |
| `x-should-retry` | Indication du serveur sur la possibilité de retry |

---

## 14. Sécurité des credentials

### Protection des credentials

| Mesure | Détail |
|--------|--------|
| Encodage hex (Keychain) | Masque les credentials dans le monitoring de processus |
| stdin (`security -i`) | Évite l'exposition argv pour les gros payloads (compatibilité CrowdStrike) |
| Pas de logging | Les API keys ne sont jamais loguées |
| File mode 0o600 | Lecture/écriture propriétaire uniquement |

### Sécurité OAuth

| Mesure | Détail |
|--------|--------|
| PKCE | `code_challenge` / `code_verifier` pour clients publics |
| State CSRF | Paramètre state de 32 octets aléatoires |
| RFC 7009 | Révocation de tokens conforme |
| Rédaction | Paramètres sensibles masqués dans les logs (state, nonce, code, code_challenge) |

### Modèle de trust

| Mesure | Détail |
|--------|--------|
| Workspace trust | `apiKeyHelper`/`awsAuthRefresh` bloqué jusqu'à acceptation |
| Allowlist OAuth | Endpoints custom limités (FedStart/PubSec uniquement) |
| Isolation provider | Bedrock/Vertex/Foundry désactivent OAuth Anthropic (prévention fuite de tokens) |
| Validation API Key | Alphanumériques, tirets, underscores uniquement |

---

## 15. Variables d'environnement — Référence complète

### API Anthropic

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | Clé API directe |
| `ANTHROPIC_AUTH_TOKEN` | Bearer token |
| `ANTHROPIC_BASE_URL` | URL de base de l'API |
| `ANTHROPIC_CUSTOM_HEADERS` | Headers custom (JSON) |
| `API_TIMEOUT_MS` | Timeout des requêtes API |

### AWS Bedrock

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_USE_BEDROCK` | Activer Bedrock |
| `AWS_REGION` / `AWS_DEFAULT_REGION` | Région AWS |
| `AWS_ACCESS_KEY_ID` | ID de clé d'accès |
| `AWS_SECRET_ACCESS_KEY` | Clé secrète |
| `AWS_SESSION_TOKEN` | Token de session STS |
| `AWS_BEARER_TOKEN_BEDROCK` | Bearer token Bedrock |
| `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION` | Région pour le petit modèle rapide |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | Skip auth (proxy) |

### Google Vertex AI

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_USE_VERTEX` | Activer Vertex |
| `ANTHROPIC_VERTEX_PROJECT_ID` | ID projet |
| `CLOUD_ML_REGION` | Région globale |
| `GOOGLE_APPLICATION_CREDENTIALS` | Fichier service account |
| `GCLOUD_PROJECT` / `GOOGLE_CLOUD_PROJECT` | Projet GCP |
| `VERTEX_REGION_CLAUDE_*` | Régions spécifiques par modèle |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH` | Skip auth (proxy) |

### Azure Foundry

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_USE_FOUNDRY` | Activer Foundry |
| `ANTHROPIC_FOUNDRY_API_KEY` | Clé API Foundry |
| `ANTHROPIC_FOUNDRY_RESOURCE` | Nom de la ressource |
| `ANTHROPIC_FOUNDRY_BASE_URL` | URL de base |
| `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` | Skip auth (proxy) |

### OAuth

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_OAUTH_TOKEN` | Token OAuth direct |
| `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` | Token via file descriptor |
| `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` | Refresh token |
| `CLAUDE_CODE_OAUTH_SCOPES` | Scopes du flow |
| `CLAUDE_CODE_OAUTH_CLIENT_ID` | Override client ID |
| `CLAUDE_CODE_CUSTOM_OAUTH_URL` | URL OAuth custom |
| `USE_STAGING_OAUTH` | Utiliser les serveurs staging |
| `MCP_OAUTH_CALLBACK_PORT` | Port callback OAuth MCP |
| `CLAUDE_CODE_ENABLE_XAA` | Activer XAA enterprise |

### Proxy et réseau

| Variable | Description |
|----------|-------------|
| `HTTP_PROXY` | Proxy HTTP |
| `HTTPS_PROXY` | Proxy HTTPS |
| `ALL_PROXY` | Proxy universel |
| `NO_PROXY` | Exclusions proxy |
| `CLAUDE_CODE_CA_CERTS` | Certificats CA custom |

### Sessions distantes

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_REMOTE` | Flag session distante |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | ID de session distante |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | Token d'accès session |
| `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR` | Token via FD WebSocket |
| `CLAUDE_SESSION_INGRESS_TOKEN_FILE` | Fichier de token session |
| `CLAUDE_TRUSTED_DEVICE_TOKEN` | Token de device trusted |

### Retry et télémétrie

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_MAX_RETRIES` | Max retries (défaut: 10) |
| `CLAUDE_CODE_UNATTENDED_RETRY` | Mode retry persistant (6h) |
| `ENABLE_GROWTHBOOK_DEV` | GrowthBook mode dev |
| `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS` | Intervalle flush Datadog |

---

## 16. Fichiers clés

### Connexion API

| Fichier | Rôle |
|---------|------|
| `services/api/client.ts` | Initialisation client, config Bedrock/Vertex/Foundry |
| `services/api/withRetry.ts` | Retry, rate limiting, recovery auth |
| `services/api/bootstrap.ts` | Bootstrap API (config distante) |
| `services/api/usage.ts` | Usage & billing API |
| `services/api/grove.ts` | Privacy settings API |
| `services/api/sessionIngress.ts` | Streaming transcripts (CCR) |
| `services/api/filesApi.ts` | Public Files API |

### Authentification

| Fichier | Rôle |
|---------|------|
| `services/oauth/client.ts` | OAuth first-party (token exchange, refresh, profile) |
| `services/mcp/auth.ts` | OAuth MCP (2400+ lignes : flow PKCE, DCR, refresh, révocation) |
| `services/mcp/xaaIdpLogin.ts` | XAA OIDC enterprise |
| `utils/auth.ts` | Détection source auth, cache AWS/GCP, API key helper |
| `utils/secureStorage/macOsKeychainStorage.ts` | Keychain macOS |
| `utils/sessionIngressAuth.ts` | Auth sessions distantes CCR |
| `constants/oauth.ts` | URLs OAuth, client IDs, config prod/staging/local |

### Persistance

| Fichier | Rôle |
|---------|------|
| `utils/sessionStorage.ts` | Transcripts JSONL, batch writes, classe Project |
| `utils/toolResultStorage.ts` | Stockage résultats d'outils volumineux |
| `utils/config.ts` | Configuration globale (~/.claude/config.json) |
| `utils/settings/settings.ts` | Settings utilisateur/projet/managed |
| `history.ts` | Historique des commandes |

### Réseau

| Fichier | Rôle |
|---------|------|
| `utils/proxy.ts` | Support proxy HTTP/HTTPS |
| `upstreamproxy/upstreamproxy.ts` | Tunneling CONNECT |
| `utils/teleport/api.ts` | API Teleport (sessions distantes) |
| `bridge/bridgeApi.ts` | API Bridge (IDE/Chrome) |

### Télémétrie

| Fichier | Rôle |
|---------|------|
| `services/analytics/growthbook.ts` | Feature flags GrowthBook |
| `services/analytics/datadog.ts` | Logging Datadog |
| `services/analytics/firstPartyEventLoggingExporter.ts` | Événements 1P Anthropic |
