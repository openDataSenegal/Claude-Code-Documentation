# Claude Code — Documentation

> Source code complet de **Claude Code**, l'assistant de développement CLI officiel d'Anthropic, exposé via un sourcemap npm le 31 mars 2026.

---

## Vue d'ensemble

Claude Code est un agent de développement logiciel en ligne de commande propulsé par les modèles Claude (Opus, Sonnet, Haiku). Il combine un moteur de conversation, un système d'outils extensible, un rendu terminal React personnalisé et une orchestration multi-agents pour offrir une expérience de coding assistée par IA directement dans le terminal.

### Chiffres clés

| Métrique | Valeur |
|----------|--------|
| Fichiers source | 1 884 TypeScript/TSX |
| Outils intégrés | 56+ |
| Feature flags | 93 |
| Agents built-in | 8 |
| Skills bundled | 14+ |
| Commandes CLI | 87+ |
| Services backend | 38+ |
| Composants UI | 146+ |
| Sous-systèmes documentés | 17 |
| Constantes inventoriées | 200+ |
| **Documents de référence** | **27** |

### Stack technique

- **Langage** : TypeScript / TSX
- **UI Terminal** : Renderer React personnalisé (inspiré d'Ink, moteur de layout Yoga)
- **Bundler** : Bun (avec feature flags compile-time et dead-code elimination)
- **API** : Anthropic Claude API (`@anthropic-ai/sdk`), AWS Bedrock, Google Vertex AI, Azure Foundry
- **Validation** : Zod
- **Observabilité** : OpenTelemetry, Datadog, GrowthBook
- **Protocole** : MCP (Model Context Protocol) — stdio, SSE, HTTP, WebSocket
- **Authentification** : OAuth 2.0 PKCE, XAA Enterprise, macOS Keychain
- **Persistance** : JSONL append-only (pas de base de données)
- **Voice** : Deepgram Nova 2/3 via WebSocket, capture native cpal

---

## Structure de la documentation

### Fondamentaux (1-5)

| # | Document | Description |
|---|----------|-------------|
| 1 | [Installation](installation.md) | Prérequis, installation et premier lancement |
| 2 | [Configuration](configuration.md) | Fichiers de configuration, variables d'environnement, feature flags |
| 3 | [Architecture](architecture.md) | Architecture technique, patterns, diagrammes |
| 4 | [Usage](usage.md) | Commandes, outils, workflows et exemples concrets |
| 5 | [FAQ](faq.md) | Questions fréquentes et dépannage |

### Systèmes en profondeur (6-16)

| # | Document | Description |
|---|----------|-------------|
| 6 | [Outils](outils.md) | 56+ outils, pipeline d'exécution en 7 étapes, concurrence, permissions |
| 7 | [Agents](agents.md) | 8 agents built-in, custom, lifecycle 7 phases, swarm, worktree |
| 8 | [Permissions](permissions.md) | 7 modes, pipeline 3 couches, classifieurs ML, patterns dangereux, 3 handlers |
| 9 | [Query Pipeline](query-pipeline.md) | Moteur central : boucle async, 9 phases, 12 exits, 7 recovery paths |
| 10 | [Hooks](hooks.md) | 27 événements, 4 stratégies (shell/LLM/agent/HTTP), intégration permissions |
| 11 | [Compaction](compaction.md) | 8 stratégies, résumé 9 sections, restauration auto, résilience |
| 12 | [Feature Flags](feature-flags.md) | 93 feature flags compile-time, classés par domaine |
| 13 | [Orchestration & Planification](orchestration-planification.md) | Coordinator mode, Plan mode, Swarm/Team, UltraPlan |
| 14 | [Systèmes Internes](systemes-internes.md) | 17 sous-systèmes : Buddy, Dream, Kairos, Chicago, Tungsten, Ink, etc. |
| 15 | [Skills](skills.md) | 14+ skills bundled, chargement multi-sources, activation conditionnelle, hooks, /skillify |
| 16 | [Constantes](constantes.md) | 200+ constantes : limites API, tokens, timeouts, retry, cache, enums, URLs |
| 17 | [Plugins](plugins.md) | Marketplace, manifest, installation, MCP/hooks/skills/agents, sécurité enterprise |
| 18 | [Contextes](contextes.md) | 12 types de contexte, 41 points d'injection, fenêtre de contexte, isolation agents |
| 19 | [Commandes](commandes.md) | 103+ commandes REPL, sous-commandes CLI, flags globaux, variables d'environnement |

### Flux de données et communication (17-22)

| # | Document | Description |
|---|----------|-------------|
| 20 | [Communication avec l'IA](communication-ia.md) | System prompt, messages normalisés, outils, thinking, caching, streaming |
| 21 | [Système de Mémoire](systeme-memoire.md) | 6 couches : CLAUDE.md, auto-memory, recall Sonnet, extraction, Dream, session |
| 22 | [Lecture de Fichiers](lecture-fichiers.md) | 5 formats, caching 3 couches, déduplication, permissions, sécurité |
| 23 | [Connexions API & Persistance](connexions-api.md) | 4 providers cloud, 3 systèmes OAuth, 25+ services, persistance JSONL |
| 24 | [Facturation](facturation.md) | Pricing par modèle, comptage tokens, abonnements, rate limits, overage |
| 25 | [Voice Mode](voice-mode.md) | Dictée vocale : capture native, STT Deepgram WebSocket, hold-to-talk, 20+ langues |

### Guides pratiques (23-24)

| # | Document | Description |
|---|----------|-------------|
| 26 | [Créer des Agents & Outils](creation-agents-outils.md) | Templates agents/tools/skills, interface `buildTool()`, frontmatter complet |
| 27 | [Outil LinkedIn](linkedin-tool.md) | Cas pratique : serveur MCP (11 outils), agent, 3 skills, OAuth |

---

## Carte de la documentation

```
                              README.md
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
  FONDAMENTAUX              EN PROFONDEUR           GUIDES PRATIQUES
  (1-5)                     (6-22)                  (23-24)
        │                        │                        │
  ┌─────┤              ┌─────────┼─────────┐        ┌─────┤
  │     │              │         │         │        │     │
install config     SYSTÈMES   FLUX DE    REF     création linkedin
archi   usage      (6-16)    DONNÉES   (16)     agents &  tool
faq                  │       (17-22)             outils
                     │           │
            ┌────────┤     ┌─────┤
            │        │     │     │
          outils   query  comm.  mémoire
          agents   hooks  IA     lecture
          perms    compact       fichiers
          orchest. flags         connexions
          skills   systèmes     facturation
          constantes internes   voice
```

---

## Résumé des découvertes clés

### Pas de base de données
Toute la persistance repose sur des fichiers **JSONL append-only** dans `~/.claude/`. Pas de SQL, pas de NoSQL, pas d'ORM.

### 6 couches de mémoire
CLAUDE.md (instructions) → auto-memory (fichiers persistants) → recall Sonnet (sélection intelligente) → extraction (agent forké) → Dream (consolidation nocturne) → session memory (notes post-compaction).

### Pipeline de permissions à 3 niveaux
Règles explicites → mode de permission → classifieurs ML (Bash + Transcript). Le classifieur auto-mode utilise une architecture 2 étages (rapide + réflexion) avec allowlist de 15+ outils sûrs.

### Query pipeline : 9 phases, 12 sorties, 7 recoveries
Boucle async generator `while(true)` avec gestion de prompt_too_long (collapse drain → reactive compact), max_output_tokens (escalade 8K→64K → multi-turn), et circuit breaker sur les échecs de compaction.

### 8 stratégies de compaction
Du plus léger au plus lourd : context collapse → cached microcompact → time-based microcompact → session memory compact → auto-compact → reactive compact → API context management → full compact. Restauration automatique de 5 fichiers (50K tokens) + skills (25K) + plans.

### 27 hooks événementiels
4 stratégies d'exécution (shell, LLM Haiku, agent multi-turn, HTTP REST). Intégration directe avec le système de permissions (deny > ask > allow). Support async avec registry et polling.

### Orchestration multi-agents
Coordinator mode (workers parallèles via XML task-notification), Plan mode (read-only + approbation), Swarm/Team (mailbox fichier, 3 backends, permission delegation). UltraPlan pour planification longue durée (30 min, Opus 4.6 distant).

### 14+ skills bundled
De `/batch` (refactoring parallèle 5-30 agents) à `/claude-api` (247KB docs embarqués). Activation conditionnelle par paths, hooks inline, contexte fork. `/skillify` capture des workflows en skills réutilisables.

### 200+ constantes critiques
Fenêtre contexte 200K tokens, auto-compact à 13K buffer, 10 retries API, 24h session timeout, 500ms polling swarm, 16kHz voice. Top 20 documentées par impact.

### Facturation multi-tier
7 tiers de pricing (Haiku $0.80 → Opus Fast $30/M tokens). 5 fenêtres de rate limit. Overage automatique pour Team/Enterprise avec 11 raisons de désactivation.

### 93 feature flags compile-time
Via `feature('FLAG')` de Bun, permettant le dead-code elimination. Les fonctionnalités désactivées ne sont pas incluses dans le bundle.

### 56+ outils avec concurrence gérée
Pipeline d'exécution en 7 étapes. StreamingToolExecutor pour l'exécution parallèle (concurrent-safe) vs exclusive, avec garantie d'ordre des résultats.

### Voice mode
Capture audio native (cpal C++/NAPI), STT Deepgram Nova 2/3 via WebSocket bidirectionnel, hold-to-talk avec détection auto-repeat, 20+ langues, silent drop replay pour les 1% de sessions impactées.

### Streaming SSE avec prompt caching
Sections statiques du system prompt cachées avec `scope: global` (TTL 5min-1h). Cache editing pour microcompaction sans recalcul. 16+ betas latchées par session.

### Multi-providers cloud
Anthropic (1P), AWS Bedrock, Google Vertex AI, Azure Foundry — avec détection automatique, isolation des credentials, et retry spécifique par provider.

---

## Origine du code source

Ce dépôt contient le code source complet de Claude Code, extrait d'un fichier sourcemap (`.map`) inclus par erreur dans le package npm publié. Le fichier sourcemap contenait l'intégralité du code original dans le champ `sourcesContent` du JSON.

```
npm package
  └── cli.mjs.map
        └── sourcesContent: ["// Tout le code source original..."]
```

> **Note** : Ce code est propriétaire Anthropic. Il ne peut pas être compilé et exécuté de façon autonome sans l'infrastructure interne d'Anthropic.

---

## Navigation rapide

| Je veux... | Document |
|------------|----------|
| Comprendre l'architecture globale | [architecture.md](architecture.md) |
| Voir tous les outils (56+) | [outils.md](outils.md) |
| Comprendre les agents et le multi-agents | [agents.md](agents.md) |
| Comprendre le système de permissions | [permissions.md](permissions.md) |
| Suivre le flux d'une requête de A à Z | [query-pipeline.md](query-pipeline.md) |
| Automatiser avec des hooks (27 événements) | [hooks.md](hooks.md) |
| Comprendre la gestion du contexte (8 stratégies) | [compaction.md](compaction.md) |
| Comprendre l'orchestration multi-agents | [orchestration-planification.md](orchestration-planification.md) |
| Explorer les 14+ skills bundled | [skills.md](skills.md) |
| Étendre Claude Code avec des plugins | [plugins.md](plugins.md) |
| Comprendre les 12 types de contexte | [contextes.md](contextes.md) |
| Trouver une constante / limite / timeout | [constantes.md](constantes.md) |
| Savoir ce qui est envoyé au modèle | [communication-ia.md](communication-ia.md) |
| Comprendre comment Claude Code se souvient | [systeme-memoire.md](systeme-memoire.md) |
| Voir comment les fichiers sont lus | [lecture-fichiers.md](lecture-fichiers.md) |
| Comprendre les connexions API et l'auth | [connexions-api.md](connexions-api.md) |
| Comprendre la facturation et les rate limits | [facturation.md](facturation.md) |
| Comprendre le voice mode (dictée vocale) | [voice-mode.md](voice-mode.md) |
| Créer mon propre agent, outil ou skill | [creation-agents-outils.md](creation-agents-outils.md) |
| Voir un cas pratique complet (LinkedIn) | [linkedin-tool.md](linkedin-tool.md) |
| Lister les 93 feature flags | [feature-flags.md](feature-flags.md) |
| Explorer Buddy, Dream, Kairos, Coordinator... | [systemes-internes.md](systemes-internes.md) |
| Configurer Claude Code | [configuration.md](configuration.md) |
