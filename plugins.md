# Système de Plugins — Documentation complète

L'extensibilité de Claude Code : marketplace, manifest, installation, MCP/hooks/skills/agents, sécurité enterprise et cycle de vie.

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Manifest plugin (plugin.json)](#2-manifest)
3. [Sources et scopes d'installation](#3-sources-scopes)
4. [Marketplace](#4-marketplace)
5. [Cycle de vie d'un plugin](#5-cycle-de-vie)
6. [Capacités des plugins](#6-capacités)
7. [Configuration utilisateur](#7-configuration-utilisateur)
8. [Dépendances et résolution](#8-dépendances)
9. [Sécurité et policies enterprise](#9-sécurité)
10. [Auto-update](#10-auto-update)
11. [Plugins built-in](#11-plugins-built-in)
12. [Commandes CLI](#12-commandes-cli)
13. [Stockage sur disque](#13-stockage-disque)
14. [Gestion des erreurs](#14-gestion-erreurs)
15. [Variables d'environnement](#15-variables-environnement)
16. [Fichiers clés](#16-fichiers-clés)

---

## 1. Vue d'ensemble

Le système de plugins permet d'étendre Claude Code avec des serveurs MCP, des hooks, des skills, des agents, des commandes et des serveurs LSP — le tout distribué via des marketplaces.

### Architecture

```
Marketplace (GitHub / GCS / URL / NPM / Git)
    │
    ▼
Plugin manifest (plugin.json)
    │
    ├── mcpServers      → Serveurs MCP (outils externes)
    ├── hooks           → Hooks lifecycle (PreToolUse, PostToolUse, etc.)
    ├── commands        → Commandes slash (markdown)
    ├── agents          → Agents IA (markdown)
    ├── skills          → Skills réutilisables (SKILL.md)
    ├── outputStyles    → Styles de sortie
    ├── lspServers      → Serveurs Language Server Protocol
    └── userConfig      → Options configurables par l'utilisateur
```

### Identifiant de plugin

Format : `plugin-name@marketplace-name`

```
code-formatter@anthropic-tools
db-assistant@company-internal
my-plugin@builtin                   (built-in)
my-plugin@inline                    (session-only, via --plugin-dir)
```

---

## 2. Manifest plugin (plugin.json)

### Emplacement

```
.claude-plugin/plugin.json          (dans le répertoire du plugin)
```

### Structure complète

```json
{
  "name": "mon-plugin",
  "version": "1.0.0",
  "description": "Description du plugin",
  "author": {
    "name": "Auteur",
    "email": "auteur@example.com",
    "url": "https://example.com"
  },
  "homepage": "https://github.com/org/mon-plugin",
  "repository": "https://github.com/org/mon-plugin",
  "license": "MIT",
  "keywords": ["productivity", "formatting"],

  "dependencies": [
    "autre-plugin",
    "plugin-externe@autre-marketplace"
  ],

  "mcpServers": {
    "mon-serveur": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/server.js"],
      "env": {
        "API_KEY": "${user_config.api_key}"
      }
    }
  },

  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [{
          "type": "command",
          "command": "prettier --write $FILE"
        }]
      }
    ]
  },

  "commands": {
    "format": { "source": "./commands/format.md" },
    "lint": { "source": "./commands/lint.md", "description": "Run linter" }
  },

  "agents": ["./agents/reviewer.md", "./agents/tester.md"],

  "skills": ["./skills/deploy", "./skills/release"],

  "outputStyles": ["./styles/compact.md"],

  "lspServers": {
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"]
    }
  },

  "userConfig": {
    "api_key": {
      "type": "string",
      "title": "API Key",
      "description": "Your API key for the service",
      "required": true,
      "sensitive": true
    },
    "format_on_save": {
      "type": "boolean",
      "title": "Format on Save",
      "description": "Auto-format files after editing",
      "default": true
    }
  },

  "channels": [],
  "settings": {}
}
```

### Champs du manifest

#### Métadonnées

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `name` | string | Oui | Nom kebab-case, pas d'espaces |
| `version` | string | Non | Semver recommandé |
| `description` | string | Non | Description courte |
| `author` | object | Non | `{ name, email?, url? }` |
| `homepage` | string | Non | URL du projet |
| `repository` | string | Non | URL du repo |
| `license` | string | Non | Identifiant SPDX |
| `keywords` | string[] | Non | Pour la découverte |

#### Dépendances

| Champ | Type | Description |
|-------|------|-------------|
| `dependencies` | string[] | `"plugin-name"` ou `"plugin-name@marketplace"` |

#### Composants

| Champ | Type | Description |
|-------|------|-------------|
| `mcpServers` | Record / path / MCPB | Serveurs Model Context Protocol |
| `hooks` | HooksSettings / path | Hooks lifecycle |
| `commands` | string / string[] / Record | Commandes slash (markdown) |
| `agents` | string / string[] | Agents IA (markdown) |
| `skills` | string / string[] | Skills (répertoires avec SKILL.md) |
| `outputStyles` | string / string[] | Styles de sortie |
| `lspServers` | Record / path | Serveurs Language Server Protocol |
| `channels` | array | Canaux de messages (mode assistant) |

#### Configuration

| Champ | Type | Description |
|-------|------|-------------|
| `userConfig` | Record | Options configurables (promptées à l'installation) |
| `settings` | Record | Settings mergés quand le plugin est activé |

### Options userConfig

```json
{
  "option_name": {
    "type": "string | number | boolean | directory | file",
    "title": "Displayed Title",
    "description": "Help text",
    "required": false,
    "default": "value",
    "sensitive": false,
    "multiple": false,
    "min": 0,
    "max": 100
  }
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `type` | string | `string`, `number`, `boolean`, `directory`, `file` |
| `title` | string | Titre affiché à l'utilisateur |
| `description` | string | Texte d'aide |
| `required` | boolean | Validation échoue si vide |
| `default` | mixed | Valeur par défaut |
| `sensitive` | boolean | Stockage Keychain si true |
| `multiple` | boolean | Autorise un tableau (type string) |
| `min` / `max` | number | Bornes (type number) |

---

## 3. Sources et scopes d'installation

### Scopes

| Scope | Emplacement | Persistance | Précédence |
|-------|-------------|------------|------------|
| **managed** | policySettings (MDM) | Permanente (read-only) | Haute |
| **user** | `~/.claude/settings.json` | Permanente | |
| **project** | `.claude/settings.json` | Permanente (versionnable) | |
| **local** | `.claude/settings.local.json` | Permanente (gitignored) | |
| **flag** | `--plugin-dir` CLI | Session uniquement | Basse |

**Précédence** : `local > project > user > managed`

### Sources de plugins

| Source | Format | Description |
|--------|--------|-------------|
| **Relative** | `"./plugins/mon-plugin"` | Local dans le marketplace |
| **NPM** | `{ source: 'npm', package: '@org/pkg', version?: '1.0' }` | Package NPM |
| **Pip** | `{ source: 'pip', package: 'name', version?: '==1.0' }` | Package Python |
| **GitHub** | `{ source: 'github', repo: 'owner/repo', ref?: 'main', sha?: '...' }` | Repo GitHub |
| **Git URL** | `{ source: 'url', url: 'https://...', ref?: 'main' }` | Repo Git générique |
| **Git subdir** | `{ source: 'git-subdir', url: '...', path: 'subdir/', ref?: 'main' }` | Sous-répertoire Git |

---

## 4. Marketplace

### Structure d'un marketplace

```
my-marketplace-repo/
├── .claude-plugin/
│   └── marketplace.json            # Manifeste du marketplace
└── plugins/                        # Sources locales (optionnel)
    ├── plugin-1/
    │   └── .claude-plugin/
    │       └── plugin.json
    └── plugin-2/
        └── .claude-plugin/
            └── plugin.json
```

### Manifest marketplace

```json
{
  "name": "mon-marketplace",
  "owner": { "name": "Organisation" },
  "plugins": [
    {
      "name": "mon-plugin",
      "source": "./plugins/mon-plugin",
      "description": "...",
      "category": "productivity",
      "tags": ["format", "lint"],
      "strict": true
    },
    {
      "name": "plugin-npm",
      "source": { "source": "npm", "package": "@org/plugin", "version": "1.0.0" }
    }
  ],
  "forceRemoveDeletedPlugins": false,
  "allowCrossMarketplaceDependenciesOn": ["trusted-marketplace"]
}
```

### Sources de marketplace (dans settings.json)

| Type | Configuration |
|------|--------------|
| **URL** | `{ source: 'url', url: 'https://...', headers?: {...} }` |
| **GitHub** | `{ source: 'github', repo: 'owner/repo', ref?: 'main', path?: '...' }` |
| **Git** | `{ source: 'git', url: 'https://...', ref?: 'main' }` |
| **NPM** | `{ source: 'npm', package: 'name' }` |
| **File** | `{ source: 'file', path: '/absolute/path/marketplace.json' }` |
| **Directory** | `{ source: 'directory', path: '/absolute/path/' }` |
| **Settings** | `{ source: 'settings', name: 'inline', plugins: [...] }` |
| **Host pattern** | `{ source: 'hostPattern', hostPattern: '^github\\.corp\\.com$' }` |
| **Path pattern** | `{ source: 'pathPattern', pathPattern: '^/opt/approved/' }` |

### Marketplace officiel

```
Nom : claude-code-plugins (réservé Anthropic)
Source : GCS mirror (https://downloads.claude.ai/claude-code-releases/plugins/)
Fallback : GitHub (anthropics/claude-plugins-official)
Auto-update : Activé par défaut
```

**Noms réservés** (uniquement pour l'org `anthropics` sur GitHub) :
```
claude-code-marketplace, claude-code-plugins, claude-plugins-official,
anthropic-marketplace, anthropic-plugins, agent-skills,
life-sciences, knowledge-work-plugins
```

---

## 5. Cycle de vie d'un plugin

### Installation

```
1. RÉSOLUTION
   ├── Charger le marketplace
   ├── Trouver l'entrée du plugin
   ├── Résoudre la source (npm/git/GitHub/...)
   └── Calculer la version

2. DOWNLOAD
   ├── Clone/fetch depuis la source
   ├── Extraire vers le cache versionné :
   │   ~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/
   ├── Parser plugin.json
   └── Valider les chemins des composants

3. INSTALLATION
   ├── Écrire dans installed_plugins.json (V2)
   ├── Ajouter à enabledPlugins dans settings
   ├── Résoudre les dépendances (DFS, détection de cycles)
   └── Prompter la configuration utilisateur (userConfig)

4. ACTIVATION
   ├── enabledPlugins[pluginId] = true
   ├── Charger : commands, agents, skills, hooks, MCP, LSP
   └── Enregistrer dans le système
```

### Mise à jour

```
1. Vérifier la nouvelle version dans le marketplace
2. Télécharger vers un nouveau chemin versionné
3. Mettre à jour installed_plugins.json
4. Recharger les plugins
5. Marquer l'ancienne version comme orpheline (GC)
```

### Désinstallation

```
1. Retirer de enabledPlugins
2. Retirer de installed_plugins.json
3. Si plus aucune installation :
   ├── Supprimer les options de settings
   ├── Supprimer les options sensibles du Keychain
   └── Supprimer le répertoire data : ~/.claude/plugins/data/{plugin-id}/
4. Marquer le cache pour GC
```

### Enable / Disable

```
Toggle enabledPlugins[pluginId] dans settings
Pas de modification sur disque
Chargement/déchargement au prochain reload
```

---

## 6. Capacités des plugins

### 6.1 Serveurs MCP

```json
{
  "mcpServers": {
    "mon-serveur": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/server.js"],
      "env": {
        "DB_URL": "${user_config.database_url}",
        "API_KEY": "${user_config.api_key}"
      }
    }
  }
}
```

- Variables substituées : `${CLAUDE_PLUGIN_ROOT}`, `${CLAUDE_PLUGIN_DATA}`, `${user_config.KEY}`
- Détection de doublons (même commande/URL → suppression)
- Support MCPB (.mcpb, .dxt bundles)

### 6.2 Hooks

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "prettier --write $FILE",
        "timeout": 30
      }]
    }],
    "PreToolUse": [{
      "matcher": "Bash(rm:*)",
      "hooks": [{
        "type": "prompt",
        "prompt": "Is this deletion safe?"
      }]
    }]
  }
}
```

Ou via fichier externe : `hooks/hooks.json`

**Événements supportés** : tous les 27 événements du système de hooks (PreToolUse, PostToolUse, SessionStart, etc.)

### 6.3 Commandes

```json
{
  "commands": {
    "format": { "source": "./commands/format.md", "description": "Format code" },
    "lint": { "source": "./commands/lint.md", "allowedTools": ["Bash"] }
  }
}
```

- Namespace : `plugin-name:command-name`
- Format : fichiers markdown avec frontmatter
- Substitution de variables dans le contenu

### 6.4 Agents

```json
{
  "agents": ["./agents/reviewer.md", "./agents/tester.md"]
}
```

- Namespace : `plugin-name:agent-name`
- Format : markdown avec frontmatter (name, description, tools, color, etc.)
- Chargés comme `AgentDefinition`

### 6.5 Skills

```json
{
  "skills": ["./skills/deploy", "./skills/release"]
}
```

- Chaque répertoire contient un `SKILL.md`
- Namespace : `plugin-name:skill-name`
- Chargement lazy, frontmatter complet

### 6.6 Serveurs LSP

```json
{
  "lspServers": {
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"],
      "env": { "NODE_PATH": "${CLAUDE_PLUGIN_ROOT}/node_modules" }
    }
  }
}
```

- Démarrage automatique, gestion des crashes
- Timeout et tracking des échecs de requêtes
- Substitution de variables dans la configuration

### 6.7 Styles de sortie

```json
{
  "outputStyles": ["./styles/compact.md"]
}
```

- Fichiers markdown avec frontmatter : name, description, force-for-plugin
- Appliqués au formatage de la sortie

---

## 7. Configuration utilisateur

### Flux

```
1. Plugin déclare userConfig dans le manifest
2. À l'installation/enable → prompt interactif
3. Options non-sensibles → settings.json pluginConfigs[pluginId].options
4. Options sensibles (sensitive: true) → Keychain macOS / .credentials.json
5. Disponibles comme :
   ├── ${user_config.KEY} dans MCP/LSP/hooks
   └── CLAUDE_PLUGIN_OPTION_KEY dans l'environnement
```

### Stockage

```json
// ~/.claude/settings.json
{
  "pluginConfigs": {
    "mon-plugin@marketplace": {
      "options": {
        "format_on_save": true,
        "max_line_length": 120
      }
    }
  }
}
```

Les options sensibles sont stockées dans le Keychain macOS (ou `.credentials.json` sur Linux/Windows).

---

## 8. Dépendances et résolution

### Format

```json
{
  "dependencies": [
    "autre-plugin",                     // Résolu dans le même marketplace
    "plugin-externe@autre-marketplace"  // Marketplace qualifié
  ]
}
```

### Résolution (DFS)

```
1. Noms nus → qualifiés avec le marketplace du plugin déclarant
2. Cross-marketplace → bloqué par défaut
   └── Autorisé si allowCrossMarketplaceDependenciesOn contient le marketplace
3. Graphe DFS avec détection de cycles
4. Vérification au chargement : dépendances manquantes → erreur
5. Skip si déjà enabled (pas de surprise dans les settings)
```

### Désinstallation avec dépendants

```
findReverseDependents(pluginId) → liste des plugins qui dépendent de celui-ci
Si dépendants actifs → avertissement avant suppression
```

---

## 9. Sécurité et policies enterprise

### Contrôles de policy

```json
{
  "pluginPolicy": {
    "enabled": true,
    "strictKnownMarketplaces": [
      { "source": "github", "repo": "company/approved-plugins" }
    ],
    "blockedMarketplaces": [
      { "source": "hostPattern", "hostPattern": "^untrusted\\.example\\.com$" }
    ]
  }
}
```

| Control | Description |
|---------|-------------|
| `enabled` | Kill switch global pour les plugins |
| `strictKnownMarketplaces` | Allowlist : uniquement ces marketplaces autorisés |
| `blockedMarketplaces` | Blocklist : ces marketplaces sont interdits |
| `enabledPlugins[id]: false` | Force-disable un plugin spécifique |
| `allowManagedHooksOnly` | Désactive les hooks utilisateur, garde uniquement les managed |

### Validations de sécurité

| Validation | Description |
|------------|-------------|
| Noms réservés | Les noms officiels nécessitent l'org `anthropics` sur GitHub |
| Caractères non-ASCII | Bloqués (attaques homographes) |
| Path traversal | Détection de `../` dans les chemins |
| Cycles de dépendances | DFS avec détection |
| Cross-marketplace | Bloqué par défaut (nécessite allowlist) |
| Delisting | Auto-désinstallation si `forceRemoveDeletedPlugins` |

---

## 10. Auto-update

### Mécanisme

```
1. Au démarrage, vérifier quels marketplaces ont autoUpdate: true
2. Rafraîchir ces marketplaces (git pull / re-download)
3. Comparer les versions installées vs marketplace
4. Installer les nouvelles versions en cache
5. Notification de redémarrage si mises à jour disponibles
```

### Configuration

- Marketplaces officiels : auto-update activé par défaut
- Marketplaces utilisateur : désactivable par marketplace
- Seed directory : `CLAUDE_CODE_PLUGIN_SEED_DIR` pour pré-cache (BYOC)

---

## 11. Plugins built-in

Plugins compilés dans le binaire CLI :

```typescript
BuiltinPluginDefinition {
  name: string,
  description: string,
  version?: string,
  skills?: BundledSkillDefinition[],
  hooks?: HooksSettings,
  mcpServers?: Record<string, McpServerConfig>,
  isAvailable?: () => boolean,
  defaultEnabled?: boolean,
}
```

- Format : `name@builtin`
- Activables/désactivables via l'UI `/plugin`
- Cachés si `isAvailable()` retourne `false`

---

## 12. Commandes CLI

```bash
# Lister les plugins
claude plugin list [--json] [--available]

# Installer
claude plugin install <plugin-id> [--scope user|project|local]

# Désinstaller
claude plugin uninstall <plugin-id>

# Activer / désactiver
claude plugin enable <plugin-id>
claude plugin disable <plugin-id>
claude plugin disable-all

# Mettre à jour
claude plugin update <plugin-id>

# Valider un manifest
claude plugin validate <path>

# Gérer les marketplaces
claude plugin marketplace add <source-config>
claude plugin marketplace remove <name>
claude plugin marketplace list
claude plugin marketplace refresh [<name>]
```

### UI interactive (dans le REPL)

```
/plugin
├── Browse Marketplace     → Découvrir et installer des plugins
├── Manage Plugins         → Enable/disable/uninstall
├── Manage Marketplaces    → Ajouter/supprimer des sources
└── Validate Plugin        → Vérifier un manifest
```

---

## 13. Stockage sur disque

```
~/.claude/
├── settings.json
│   └── enabledPlugins: { "plugin@marketplace": true }
│   └── pluginConfigs: { "plugin@marketplace": { options: {...} } }
│
├── plugins/
│   ├── installed_plugins.json          # Métadonnées V2
│   ├── known_marketplaces.json         # Registre des marketplaces
│   ├── install-counts-cache.json       # Statistiques (24h TTL)
│   │
│   ├── cache/                          # Cache versionné
│   │   └── {marketplace}/
│   │       └── {plugin-name}/
│   │           └── {version}/
│   │               ├── .claude-plugin/
│   │               │   └── plugin.json
│   │               ├── commands/
│   │               ├── agents/
│   │               ├── skills/
│   │               ├── hooks/
│   │               │   └── hooks.json
│   │               └── .mcp.json
│   │
│   ├── marketplaces/                   # Cache des marketplaces
│   │   ├── claude-plugins-official/    # GitHub clone
│   │   └── custom-marketplace.json     # URL fetch
│   │
│   └── data/                           # Données persistantes par plugin
│       └── {plugin-id-sanitized}/      # Survit aux mises à jour
```

### installed_plugins.json (V2)

```json
{
  "version": 2,
  "plugins": {
    "mon-plugin@marketplace": [
      {
        "scope": "user",
        "installPath": "/Users/x/.claude/plugins/cache/marketplace/mon-plugin/1.0.0",
        "version": "1.0.0",
        "installedAt": "2026-04-01T10:00:00Z"
      }
    ]
  }
}
```

---

## 14. Gestion des erreurs

### Types d'erreurs plugin

| Type | Description |
|------|-------------|
| `path-not-found` | Chemin du plugin introuvable |
| `git-auth-failed` | Authentification Git échouée |
| `git-timeout` | Timeout clone/pull Git |
| `network-error` | Erreur réseau |
| `manifest-parse-error` | JSON invalide dans plugin.json |
| `manifest-validation-error` | Schema Zod invalide |
| `plugin-not-found` | Plugin absent du marketplace |
| `marketplace-not-found` | Marketplace introuvable |
| `marketplace-load-failed` | Échec chargement marketplace |
| `mcp-config-invalid` | Config MCP invalide |
| `mcp-server-suppressed-duplicate` | Serveur MCP en doublon |
| `lsp-config-invalid` | Config LSP invalide |
| `lsp-server-start-failed` | Échec démarrage LSP |
| `lsp-server-crashed` | Crash du serveur LSP |
| `marketplace-blocked-by-policy` | Bloqué par policy enterprise |
| `dependency-unsatisfied` | Dépendance manquante |
| `plugin-cache-miss` | Plugin absent du cache |

### Résultat de chargement

```typescript
PluginLoadResult {
  enabled: LoadedPlugin[],    // Plugins activés avec succès
  disabled: LoadedPlugin[],   // Installés mais désactivés
  errors: PluginError[],      // Échecs de chargement
}
```

---

## 15. Variables d'environnement

### Injectées dans les composants du plugin

| Variable | Description |
|----------|-------------|
| `CLAUDE_PLUGIN_ROOT` | Chemin racine du plugin installé |
| `CLAUDE_PLUGIN_DATA` | Répertoire de données persistant |
| `CLAUDE_PLUGIN_OPTION_*` | Valeurs de configuration utilisateur |

### Substitution dans les configs

```json
{
  "command": "${CLAUDE_PLUGIN_ROOT}/bin/server",
  "env": {
    "DATA_DIR": "${CLAUDE_PLUGIN_DATA}",
    "API_KEY": "${user_config.api_key}"
  }
}
```

Fonctionne dans : MCP servers, LSP servers, hooks, contenu de skills/agents.

### Variables système

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_PLUGIN_SEED_DIR` | Répertoire de pré-cache (BYOC, multi-paths `:`) |

---

## 16. Fichiers clés

### Infrastructure

| Fichier | Rôle |
|---------|------|
| `utils/plugins/schemas.ts` | Schemas Zod : PluginManifest, Marketplace, UserConfig |
| `utils/plugins/pluginLoader.ts` | Chargement, validation, résultat LoadedPlugin |
| `utils/plugins/pluginOperations.ts` | Install/uninstall/enable/disable/update |
| `utils/plugins/pluginInstallationHelpers.ts` | Cache, download, extraction |
| `utils/plugins/dependencyResolver.ts` | DFS, cycles, cross-marketplace |
| `utils/plugins/pluginVersioning.ts` | Calcul de version (semver, SHA, etc.) |
| `utils/plugins/pluginAutoupdate.ts` | Vérification et installation des mises à jour |

### Marketplace

| Fichier | Rôle |
|---------|------|
| `utils/plugins/marketplaceManager.ts` | Chargement, cache, refresh |
| `utils/plugins/marketplaceHelpers.ts` | Policy, validation noms réservés |
| `utils/plugins/officialMarketplaceGcs.ts` | Fetch GCS mirror officiel |
| `utils/plugins/installCounts.ts` | Statistiques d'installation |

### Composants

| Fichier | Rôle |
|---------|------|
| `utils/plugins/mcpPluginIntegration.ts` | Serveurs MCP des plugins |
| `utils/plugins/lspPluginIntegration.ts` | Serveurs LSP des plugins |
| `utils/plugins/loadPluginHooks.ts` | Hooks des plugins |
| `utils/plugins/loadPluginCommands.ts` | Commandes des plugins |
| `utils/plugins/loadPluginAgents.ts` | Agents des plugins |
| `utils/plugins/loadPluginOutputStyles.ts` | Styles de sortie |
| `plugins/builtinPlugins.ts` | Plugins built-in |

### CLI et UI

| Fichier | Rôle |
|---------|------|
| `cli/handlers/plugins.ts` | Handlers CLI (`claude plugin *`) |
| `commands/plugin/PluginSettings.tsx` | UI principale `/plugin` |
| `commands/plugin/ManagePlugins.tsx` | Gestion enable/disable/uninstall |
| `commands/plugin/BrowseMarketplace.tsx` | Découverte marketplace |
| `commands/plugin/PluginTrustWarning.tsx` | Avertissements de sécurité |
| `commands/plugin/ValidatePlugin.tsx` | Validation de manifest |
