# Feature Flags

Les feature flags contrôlent l'activation de fonctionnalités via le système `feature('FLAG')` de Bun. Ils sont résolus au **compile-time** et permettent le **dead-code elimination** — le code derrière un flag désactivé n'est pas inclus dans le bundle final.

```typescript
import { feature } from 'bun:bundle'

if (feature('MON_FLAG')) {
  // Code inclus uniquement si le flag est activé au build
}
```

---

## Agent & Coordination

| Flag | Rôle |
|------|------|
| `COORDINATOR_MODE` | Mode multi-agents coordonné |
| `FORK_SUBAGENT` | Fork de sous-agents |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | Agents Explore/Plan intégrés |
| `VERIFICATION_AGENT` | Agent de vérification automatique |
| `PROACTIVE` | Comportement proactif de l'agent |
| `BRIDGE_MODE` | Mode pont (communication inter-agents) |

## Sessions & Background

| Flag | Rôle |
|------|------|
| `BG_SESSIONS` | Sessions en arrière-plan |
| `DAEMON` | Process daemon persistant |
| `AGENT_TRIGGERS` | Déclencheurs d'agents locaux |
| `AGENT_TRIGGERS_REMOTE` | Déclencheurs d'agents distants |
| `AWAY_SUMMARY` | Résumé des actions pendant l'absence |

## Mémoire & Contexte

| Flag | Rôle |
|------|------|
| `EXTRACT_MEMORIES` | Extraction automatique de mémoires |
| `AGENT_MEMORY_SNAPSHOT` | Snapshot mémoire d'agent |
| `MEMORY_SHAPE_TELEMETRY` | Télémétrie sur la forme des mémoires |
| `CONTEXT_COLLAPSE` | Compression/collapse de contexte |
| `COMPACTION_REMINDERS` | Rappels de compaction |
| `REACTIVE_COMPACT` | Compaction réactive |
| `FILE_PERSISTENCE` | Persistance fichier |

## UI & Expérience

| Flag | Rôle |
|------|------|
| `BUDDY` | Compagnon/mascotte animé |
| `AUTO_THEME` | Thème automatique |
| `HISTORY_PICKER` | Sélecteur d'historique |
| `HISTORY_SNIP` | Extrait d'historique |
| `MESSAGE_ACTIONS` | Actions sur les messages |
| `STREAMLINED_OUTPUT` | Sortie simplifiée |
| `CONNECTOR_TEXT` | Texte de connexion UI |
| `VOICE_MODE` | Mode vocal |
| `NATIVE_CLIPBOARD_IMAGE` | Images clipboard natif |

## Outils & Skills

| Flag | Rôle |
|------|------|
| `WEB_BROWSER_TOOL` | Navigateur web intégré (via `Bun.WebView`) |
| `TERMINAL_PANEL` | Panneau terminal |
| `MONITOR_TOOL` | Outil de monitoring |
| `MCP_SKILLS` | Skills MCP |
| `MCP_RICH_OUTPUT` | Sortie riche MCP |
| `EXPERIMENTAL_SKILL_SEARCH` | Recherche de skills expérimentale |
| `SKILL_IMPROVEMENT` | Amélioration de skills |
| `RUN_SKILL_GENERATOR` | Générateur de skills |
| `QUICK_SEARCH` | Recherche rapide |
| `OVERFLOW_TEST_TOOL` | Outil de test overflow |

## Modèle & Inférence

| Flag | Rôle |
|------|------|
| `ULTRATHINK` | Réflexion étendue |
| `ULTRAPLAN` | Planification étendue |
| `KAIROS` | Système Kairos (base) |
| `KAIROS_BRIEF` | Kairos — mode brief |
| `KAIROS_CHANNELS` | Kairos — canaux |
| `KAIROS_DREAM` | Kairos — mode dream |
| `KAIROS_GITHUB_WEBHOOKS` | Kairos — webhooks GitHub |
| `KAIROS_PUSH_NOTIFICATION` | Kairos — notifications push |
| `ABLATION_BASELINE` | Baseline d'ablation (A/B testing) |
| `TOKEN_BUDGET` | Budget de tokens |
| `CACHED_MICROCOMPACT` | Micro-compaction cachée |
| `PROMPT_CACHE_BREAK_DETECTION` | Détection de rupture de cache prompt |
| `BREAK_CACHE_COMMAND` | Commande de rupture de cache |

## Télémétrie & Analytics

| Flag | Rôle |
|------|------|
| `ENHANCED_TELEMETRY_BETA` | Télémétrie enrichie (beta) |
| `COWORKER_TYPE_TELEMETRY` | Télémétrie type coworker |
| `SHOT_STATS` | Statistiques de shots |
| `SLOW_OPERATION_LOGGING` | Logging des opérations lentes |
| `PERFETTO_TRACING` | Tracing Perfetto |
| `TRANSCRIPT_CLASSIFIER` | Classification de transcripts |
| `BASH_CLASSIFIER` | Classification de commandes bash |

## Infra & Déploiement

| Flag | Rôle |
|------|------|
| `SSH_REMOTE` | Connexion SSH remote |
| `CCR_AUTO_CONNECT` | Claude Code Remote — auto-connexion |
| `CCR_MIRROR` | Claude Code Remote — miroir |
| `CCR_REMOTE_SETUP` | Claude Code Remote — setup |
| `DIRECT_CONNECT` | Connexion directe |
| `SELF_HOSTED_RUNNER` | Runner auto-hébergé |
| `BYOC_ENVIRONMENT_RUNNER` | Runner BYOC (Bring Your Own Cloud) |
| `UDS_INBOX` | Inbox Unix Domain Socket |
| `IS_LIBC_GLIBC` | Détection libc glibc |
| `IS_LIBC_MUSL` | Détection libc musl |

## Sécurité & Compliance

| Flag | Rôle |
|------|------|
| `ANTI_DISTILLATION_CC` | Anti-distillation |
| `NATIVE_CLIENT_ATTESTATION` | Attestation client natif |
| `HARD_FAIL` | Échec strict (pas de fallback silencieux) |

## Divers

| Flag | Rôle |
|------|------|
| `ALLOW_TEST_VERSIONS` | Autorise les versions de test |
| `BUILDING_CLAUDE_APPS` | Aide à la construction d'apps Claude |
| `CHICAGO_MCP` | MCP "Chicago" |
| `COMMIT_ATTRIBUTION` | Attribution de commits |
| `DOWNLOAD_USER_SETTINGS` | Téléchargement des settings utilisateur |
| `UPLOAD_USER_SETTINGS` | Upload des settings utilisateur |
| `DUMP_SYSTEM_PROMPT` | Dump du system prompt (debug) |
| `HOOK_PROMPTS` | Prompts pour hooks |
| `LODESTONE` | Système "Lodestone" |
| `NEW_INIT` | Nouveau flow d'initialisation |
| `POWERSHELL_AUTO_MODE` | Mode auto PowerShell |
| `REVIEW_ARTIFACT` | Review d'artefacts |
| `TEAMMEM` | Mémoire d'équipe |
| `TEMPLATES` | Système de templates |
| `TORCH` | Système "Torch" |
| `TREE_SITTER_BASH` | Parsing Bash via Tree-sitter |
| `TREE_SITTER_BASH_SHADOW` | Parsing Bash via Tree-sitter (shadow mode) |
| `UNATTENDED_RETRY` | Retry en mode non-assisté |
| `WORKFLOW_SCRIPTS` | Scripts de workflow |

---

**Total : 93 feature flags**
