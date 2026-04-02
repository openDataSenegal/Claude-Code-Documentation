# Commandes — Documentation complète

Toutes les commandes de Claude Code : 103+ commandes REPL (slash), sous-commandes CLI, flags globaux et variables d'environnement.

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Commandes REPL — Interaction](#2-commandes-interaction)
3. [Commandes REPL — Session & conversation](#3-commandes-session)
4. [Commandes REPL — Modèle & configuration](#4-commandes-config)
5. [Commandes REPL — Développement & code](#5-commandes-dev)
6. [Commandes REPL — Contexte & performance](#6-commandes-contexte)
7. [Commandes REPL — IDE & intégrations](#7-commandes-ide)
8. [Commandes REPL — Compte & auth](#8-commandes-auth)
9. [Commandes REPL — Debug & diagnostics](#9-commandes-debug)
10. [Commandes REPL — Feature-gated](#10-commandes-gated)
11. [CLI — Flags globaux](#11-flags-globaux)
12. [CLI — Sous-commande auth](#12-cli-auth)
13. [CLI — Sous-commande mcp](#13-cli-mcp)
14. [CLI — Sous-commande plugin](#14-cli-plugin)
15. [CLI — Autres sous-commandes](#15-cli-autres)
16. [Architecture du système de commandes](#16-architecture)
17. [Fichiers clés](#17-fichiers-clés)

---

## 1. Vue d'ensemble

Claude Code a deux niveaux de commandes :

| Niveau | Accès | Exemples |
|--------|-------|----------|
| **REPL** (slash commands) | `/commande` dans la conversation | `/help`, `/compact`, `/model`, `/diff` |
| **CLI** (terminal) | `claude sous-commande` dans le shell | `claude auth login`, `claude mcp add`, `claude plugin install` |

### Types de commandes REPL

| Type | Comportement |
|------|-------------|
| `prompt` | Expandé en texte envoyé au modèle (skills, agents) |
| `local` | Exécuté localement, retourne du texte |
| `local-jsx` | Exécuté localement, rendu Ink (React terminal) |

### Disponibilité

| Flag | Signification |
|------|--------------|
| `availability: ['claude-ai']` | Abonnés Claude.ai uniquement |
| `availability: ['console']` | Utilisateurs API key uniquement |
| Pas de flag | Universel |
| `isHidden: true` | Caché de l'autocomplete et /help |
| `isEnabled()` | Gate dynamique (feature flag, env, etc.) |
| `immediate: true` | S'exécute immédiatement sans attendre un stop point |

---

## 2. Commandes REPL — Interaction

| Commande | Alias | Type | Description |
|----------|-------|------|-------------|
| `/help` | — | local-jsx | Affiche l'aide et les commandes disponibles |
| `/exit` | `/quit` | local-jsx | Quitte le REPL |
| `/clear` | `/reset`, `/new` | local | Efface l'historique et libère le contexte |
| `/compact` | — | local | Résume la conversation pour libérer du contexte |
| `/copy` | — | local-jsx | Copie la dernière réponse dans le presse-papiers |
| `/btw` | — | local-jsx | Question rapide sans interrompre la conversation (immédiat) |
| `/feedback` | `/bug` | local-jsx | Soumettre un feedback |
| `/rewind` | `/checkpoint` | local | Restaurer le code/conversation à un point précédent |
| `/export` | — | local-jsx | Exporter la conversation en fichier ou presse-papiers |

---

## 3. Commandes REPL — Session & conversation

| Commande | Alias | Type | Description |
|----------|-------|------|-------------|
| `/resume` | `/continue` | local-jsx | Reprendre une conversation précédente |
| `/rename` | — | local-jsx | Renommer la conversation (immédiat) |
| `/session` | `/remote` | local-jsx | Afficher l'URL de session distante et QR code |
| `/mobile` | `/ios`, `/android` | local-jsx | QR code pour l'app mobile Claude |
| `/tasks` | `/bashes` | local-jsx | Lister et gérer les tâches en arrière-plan |
| `/plan` | — | local-jsx | Activer le mode plan ou voir le plan de session |
| `/branch` | `/fork` (conditionnel) | local-jsx | Créer une branche de la conversation |
| `/stickers` | — | local | Commander des stickers Claude Code |

---

## 4. Commandes REPL — Modèle & configuration

| Commande | Alias | Type | Description |
|----------|-------|------|-------------|
| `/model` | — | local-jsx | Changer le modèle IA (immédiat) |
| `/fast` | — | local-jsx | Toggle fast mode (gate: `isFastModeEnabled`) |
| `/effort` | — | local-jsx | Définir le niveau d'effort (immédiat) |
| `/config` | `/settings` | local-jsx | Ouvrir le panneau de configuration |
| `/theme` | — | local-jsx | Changer le thème |
| `/color` | — | local-jsx | Couleur de la barre de prompt (immédiat) |
| `/vim` | — | local | Toggle entre modes Vim et Normal |
| `/statusline` | — | prompt | Configurer la status line |
| `/advisor` | — | local | Configurer le modèle advisor |
| `/output-style` | — | local-jsx | Changer le style de sortie (caché, déprécié) |

---

## 5. Commandes REPL — Développement & code

| Commande | Alias | Type | Description |
|----------|-------|------|-------------|
| `/diff` | — | local-jsx | Voir les changements non commités et diffs par tour |
| `/review` | — | prompt | Revoir une pull request |
| `/ultrareview` | — | local-jsx | Review PR avancée avec analyse distante |
| `/security-review` | — | prompt | Review de sécurité des changements en attente |
| `/commit` | — | prompt | Créer un commit git (interne) |
| `/commit-push-pr` | — | prompt | Commit + push + création PR (interne) |
| `/pr-comments` | — | prompt | Récupérer les commentaires d'une PR GitHub |
| `/add-dir` | — | local-jsx | Ajouter un répertoire de travail |
| `/init` | — | prompt | Configurer CLAUDE.md et skills pour le repo |

---

## 6. Commandes REPL — Contexte & performance

| Commande | Alias | Type | Description |
|----------|-------|------|-------------|
| `/context` | — | local-jsx | Visualiser l'utilisation du contexte (grille colorée) |
| `/cost` | — | local | Coût et durée de la session (caché pour abonnés) |
| `/usage` | — | local-jsx | Limites d'utilisation du plan (claude-ai) |
| `/extra-usage` | — | local-jsx | Configurer l'usage extra quand les limites sont atteintes |
| `/rate-limit-options` | — | local-jsx | Options quand le rate limit est atteint (caché) |
| `/files` | — | local | Lister les fichiers dans le contexte (ant-only) |
| `/stats` | — | local-jsx | Statistiques d'utilisation |
| `/release-notes` | — | local | Notes de version |

---

## 7. Commandes REPL — IDE & intégrations

| Commande | Alias | Type | Description |
|----------|-------|------|-------------|
| `/ide` | — | local-jsx | Gérer les intégrations IDE |
| `/desktop` | `/app` | local-jsx | Continuer dans Claude Desktop (claude-ai) |
| `/install-github-app` | — | local-jsx | Installer Claude GitHub Actions |
| `/install-slack-app` | — | local | Installer Claude Slack app (claude-ai) |
| `/plugin` | `/plugins`, `/marketplace` | local-jsx | Gérer les plugins (immédiat) |
| `/reload-plugins` | — | local | Activer les changements de plugins en attente |
| `/mcp` | — | local-jsx | Gérer les serveurs MCP (immédiat) |
| `/chrome` | — | local-jsx | Paramètres Claude in Chrome (claude-ai) |

---

## 8. Commandes REPL — Compte & auth

| Commande | Alias | Type | Description |
|----------|-------|------|-------------|
| `/login` | — | local-jsx | Se connecter / changer de compte Anthropic |
| `/logout` | — | local-jsx | Se déconnecter |
| `/voice` | — | local | Toggle voice mode (claude-ai, `VOICE_MODE`) |
| `/upgrade` | — | local-jsx | Passer à l'abonnement Max (claude-ai) |
| `/privacy-settings` | — | local-jsx | Paramètres de confidentialité (abonnés consumer) |
| `/passes` | — | local-jsx | Partager des passes d'essai (Max, conditionnel) |

---

## 9. Commandes REPL — Debug & diagnostics

| Commande | Alias | Type | Description |
|----------|-------|------|-------------|
| `/doctor` | — | local-jsx | Diagnostic de l'installation Claude Code |
| `/status` | — | local-jsx | Statut (version, modèle, connectivité) (immédiat) |
| `/version` | — | local | Version de la session (ant-only) |
| `/tag` | — | local-jsx | Toggle un tag sur la session (ant-only) |
| `/heapdump` | — | local | Dump du heap JS sur ~/Desktop (caché) |
| `/keybindings` | — | local | Ouvrir la config des raccourcis |
| `/hooks` | — | local-jsx | Voir les hooks configurés (immédiat) |
| `/permissions` | `/allowed-tools` | local-jsx | Gérer les règles allow/deny |
| `/memory` | — | local-jsx | Éditer les fichiers mémoire |
| `/skills` | — | local-jsx | Lister les skills disponibles |
| `/agents` | — | local-jsx | Gérer les agents |
| `/sandbox` | — | local-jsx | Paramètres sandbox |
| `/terminal-setup` | — | local-jsx | Configurer les keybindings terminal (caché) |

---

## 10. Commandes REPL — Feature-gated

| Commande | Gate | Type | Description |
|----------|------|------|-------------|
| `/brief` | `KAIROS` / `KAIROS_BRIEF` | prompt | Aperçu bref |
| `/proactive` | `PROACTIVE` / `KAIROS` | prompt | Mode proactif autonome |
| `/assistant` | `KAIROS` | prompt | Mode assistant Kairos |
| `/ultraplan` | `ULTRAPLAN` | prompt | Exploration multi-agents avec approbation de plan |
| `/torch` | `TORCH` | prompt | Outil Torch |
| `/workflows` | `WORKFLOW_SCRIPTS` | prompt | Scripts de workflow |
| `/force-snip` | `HISTORY_SNIP` | prompt | Snipping d'historique |
| `/subscribe-pr` | `KAIROS_GITHUB_WEBHOOKS` | prompt | Abonnement PR |
| `/peers` | `UDS_INBOX` | local-jsx | Peers UDS |
| `/fork` | `FORK_SUBAGENT` | local-jsx | Fork de sous-agent |
| `/buddy` | `BUDDY` | prompt | Compagnon Tamagotchi |
| `/remote-control` | `BRIDGE_MODE` | local-jsx | Contrôle distant (immédiat) |
| `/remote-env` | — | local-jsx | Environnement distant (claude-ai, policy) |
| `/web-setup` | GB flag | local-jsx | Setup web (claude-ai, policy) |
| `/think-back` | `tengu_thinkback` | local-jsx | Rétrospective 2025 |

---

## 11. CLI — Flags globaux

### Modèle et sortie

| Flag | Description |
|------|-------------|
| `--model <model>` | Modèle pour la session (alias ou nom complet) |
| `--effort <level>` | Niveau d'effort : low, medium, high, max |
| `-p, --print` | Mode non-interactif : imprime et quitte |
| `--output-format <format>` | `text` (défaut), `json`, `stream-json` (avec --print) |
| `--input-format <format>` | `text` (défaut), `stream-json` (avec --print) |
| `--json-schema <schema>` | Schema JSON pour output structuré |
| `--fallback-model <model>` | Modèle de fallback si surchargé (avec --print) |

### Session et persistance

| Flag | Description |
|------|-------------|
| `-c, --continue` | Continuer la conversation la plus récente |
| `-r, --resume [value]` | Reprendre par ID session ou picker interactif |
| `--fork-session` | Créer un nouvel ID au lieu de réutiliser l'original |
| `-n, --name <name>` | Nom d'affichage de la session |
| `--session-id <uuid>` | Utiliser un UUID spécifique |
| `--no-session-persistence` | Désactiver la persistance (avec --print) |

### System prompt

| Flag | Description |
|------|-------------|
| `--system-prompt <prompt>` | System prompt pour la session |
| `--system-prompt-file <file>` | Lire le system prompt depuis un fichier |
| `--append-system-prompt <prompt>` | Ajouter au system prompt par défaut |
| `--append-system-prompt-file <file>` | Lire et ajouter depuis un fichier |

### Outils et permissions

| Flag | Description |
|------|-------------|
| `--tools <tools...>` | Outils disponibles : `""` (aucun), `default`, ou noms spécifiques |
| `--allowed-tools <tools...>` | Liste d'outils autorisés |
| `--disallowed-tools <tools...>` | Liste d'outils refusés |
| `--dangerously-skip-permissions` | Bypass toutes les vérifications (sandbox recommandé) |
| `--permission-mode <mode>` | Mode de permission pour la session |
| `--permission-prompt-tool <tool>` | Outil MCP pour les prompts de permission (avec --print) |

### MCP et plugins

| Flag | Description |
|------|-------------|
| `--mcp-config <configs...>` | Charger des serveurs MCP depuis JSON |
| `--strict-mcp-config` | Uniquement les serveurs de --mcp-config |
| `--plugin-dir <path>` | Charger des plugins depuis un répertoire (répétable) |
| `--agents <json>` | Agents custom en JSON |
| `--agent <agent>` | Agent pour la session |

### Divers

| Flag | Description |
|------|-------------|
| `-d, --debug [filter]` | Mode debug avec filtre optionnel |
| `--bare` | Mode minimal (skip hooks, LSP, plugins, attributions, mémoire auto) |
| `-w, --worktree [name]` | Créer un git worktree isolé |
| `--add-dir <dirs...>` | Répertoires supplémentaires autorisés |
| `--settings <file-or-json>` | Fichier ou JSON de settings |
| `--max-turns <turns>` | Tours max en mode non-interactif |
| `--max-budget-usd <amount>` | Budget max en USD (avec --print) |
| `--ide` | Auto-connecter à l'IDE au démarrage |
| `--chrome` / `--no-chrome` | Activer/désactiver Claude in Chrome |
| `--disable-slash-commands` | Désactiver tous les skills |
| `--verbose` | Mode verbeux |
| `-v, --version` | Numéro de version |
| `-h, --help` | Aide |

---

## 12. CLI — Sous-commande auth

```bash
# Se connecter
claude auth login [--email <email>] [--sso] [--console|--claudeai]

# Se déconnecter
claude auth logout

# Vérifier le statut
claude auth status [--json|--text]
```

| Flag | Description |
|------|-------------|
| `--email <email>` | Pré-remplir l'email sur la page de login |
| `--sso` | Forcer le flow SSO |
| `--console` | Console Anthropic (facturation API) |
| `--claudeai` | Abonnement Claude (défaut) |

---

## 13. CLI — Sous-commande mcp

```bash
# Serveur MCP Claude Code
claude mcp serve [-d, --debug] [--verbose]

# Lister les serveurs configurés
claude mcp list

# Détails d'un serveur
claude mcp get <name>

# Ajouter un serveur
claude mcp add <name> <config>
claude mcp add-json <name> <json> [-s, --scope <scope>] [--client-secret]
claude mcp add-from-claude-desktop [-s, --scope <scope>]

# Supprimer un serveur
claude mcp remove <name> [-s, --scope <scope>]

# Réinitialiser les choix projet
claude mcp reset-project-choices
```

| Flag | Description |
|------|-------------|
| `-s, --scope <scope>` | Scope : `local`, `user`, ou `project` |
| `--client-secret` | Prompt pour le client secret OAuth |

---

## 14. CLI — Sous-commande plugin

```bash
# Lister les plugins
claude plugin list [--json] [--available]

# Installer / Désinstaller
claude plugin install <plugin> [-s, --scope <scope>]
claude plugin uninstall <plugin> [-s, --scope <scope>] [--keep-data]

# Activer / Désactiver
claude plugin enable <plugin> [-s, --scope <scope>]
claude plugin disable [plugin] [-a, --all] [-s, --scope <scope>]

# Mettre à jour
claude plugin update <plugin> [-s, --scope <scope>]

# Valider un manifest
claude plugin validate <path>

# Gérer les marketplaces
claude plugin marketplace add <source> [--sparse <paths...>] [--scope <scope>]
claude plugin marketplace list [--json]
claude plugin marketplace remove <name>
claude plugin marketplace update [name]
```

| Flag | Description |
|------|-------------|
| `-s, --scope` | `user` (défaut), `project`, ou `local` |
| `--keep-data` | Préserver les données persistantes du plugin |
| `-a, --all` | Désactiver tous les plugins |
| `--sparse` | Limiter le checkout git (sparse-checkout) |

---

## 15. CLI — Autres sous-commandes

### Système

```bash
claude doctor                    # Diagnostic auto-updater
claude update|upgrade            # Vérifier et installer les mises à jour
claude install [target] [--force] # Installer une version native
claude setup-token               # Configurer un token long-lived
claude completion <shell>        # Script d'autocomplétion (ant-only)
```

### Agents

```bash
claude agents [--setting-sources <sources>]  # Lister les agents configurés
```

### Auto mode (conditionnel)

```bash
claude auto-mode defaults                     # Règles par défaut en JSON
claude auto-mode config                       # Config effective en JSON
claude auto-mode critique [--model <model>]   # Feedback IA sur les règles custom
```

### Sessions et logs (ant-only)

```bash
claude log [number|sessionId]     # Gérer les logs
claude error [number]             # Voir les logs d'erreur
claude export <source> <output>   # Exporter une conversation
claude ps                         # Lister les processus Claude (implicite)
```

### Tâches (ant-only, conditionnel)

```bash
claude task create <subject> [-d <desc>] [-l <list>]
claude task list [-l <list>] [--pending] [--json]
claude task get <id> [-l <list>]
claude task update <id> [-s <status>] [--subject <text>] [-d <desc>]
claude task dir [-l <list>]
```

### Remote et serveur (ant-only)

```bash
claude remote-control [name]         # Contrôle distant via claude.ai/code
claude ssh <host> [dir] [opts]       # Claude Code sur un hôte distant
claude server [--port N] [--host H]  # Démarrer un serveur Claude Code
claude open <cc-url>                 # Se connecter à un serveur (interne)
```

---

## 16. Architecture du système de commandes

### Pipeline de chargement

```
1. COMMANDS() — Tableau memoizé des commandes built-in
2. loadAllCommands(cwd) — Charge : skills + plugins + workflows + built-in
3. getCommands(cwd) — Filtre par availability + isEnabled
4. getSkillToolCommands() — Filtre pour les commandes invocables par le modèle
```

### Résolution de commande

```typescript
findCommand(name, commands) :
  1. Recherche par cmd.name exact
  2. Recherche par getCommandName(cmd) (nom user-facing)
  3. Recherche dans cmd.aliases[]
```

### Propriétés communes

```typescript
CommandBase {
  name: string,
  aliases?: string[],
  description: string,
  type: 'prompt' | 'local' | 'local-jsx',
  isEnabled?: () => boolean,
  isHidden?: boolean,
  availability?: ('claude-ai' | 'console')[],
  argumentHint?: string,
  immediate?: boolean,
  isSensitive?: boolean,    // Masque les args de l'historique
  source?: string,          // 'builtin', 'skills', 'plugin', etc.
  whenToUse?: string,       // Pour l'auto-invocation par le modèle
}
```

### Commandes remote-safe

Sûres en mode `--remote` (pas d'accès filesystem local) :
```
session, exit, clear, help, theme, color, vim, cost, usage,
copy, btw, feedback, plan, keybindings, statusline, stickers, mobile
```

### Commandes bridge-safe

Sûres depuis un client mobile/web :
```
compact, clear, cost, summary, releaseNotes, files
```

---

## 17. Fichiers clés

### Registre et types

| Fichier | Rôle |
|---------|------|
| `commands.ts` | Registre central, getCommands(), loading pipeline |
| `types/command.ts` | CommandBase, PromptCommand, LocalCommand, LocalJSXCommand |

### Répertoires de commandes

| Répertoire | Exemples |
|-----------|----------|
| `commands/help/` | /help |
| `commands/compact/` | /compact |
| `commands/context/` | /context |
| `commands/cost/` | /cost |
| `commands/diff/` | /diff |
| `commands/model/` | /model |
| `commands/plan/` | /plan |
| `commands/plugin/` | /plugin |
| `commands/resume/` | /resume |
| `commands/voice/` | /voice |
| `commands/mcp/` | /mcp |
| `commands/login/` | /login, /logout |
| `commands/doctor/` | /doctor |
| `commands/config/` | /config |
| `commands/permissions/` | /permissions |
| `commands/extra-usage/` | /extra-usage |
| `commands/rate-limit-options/` | /rate-limit-options |
| `commands/branch/` | /branch |
| `commands/review/` | /review |
| `commands/commit/` | /commit |

### CLI

| Fichier | Rôle |
|---------|------|
| `main.tsx` | Point d'entrée, parsing args, flags |
| `utils/cliArgs.ts` | Parsing des arguments CLI |
| `cli/handlers/auth.ts` | `claude auth *` |
| `cli/handlers/mcp.tsx` | `claude mcp *` |
| `cli/handlers/plugins.ts` | `claude plugin *` |
| `cli/handlers/autoMode.ts` | `claude auto-mode *` |
| `cli/handlers/agents.ts` | `claude agents` |
| `cli/exit.ts` | Gestion de sortie |

### Exécution

| Fichier | Rôle |
|---------|------|
| `utils/processUserInput/processSlashCommand.tsx` | Exécution des slash commands |
| `hooks/useSlashCommand.ts` | Hook React pour les slash commands |
| `utils/commandAutocomplete.ts` | Autocomplete des commandes |
