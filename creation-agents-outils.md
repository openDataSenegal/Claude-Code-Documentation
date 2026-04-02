# Créer des Agents, des Outils et des Skills — Guide méthodologique

Ce document est un guide pratique pour créer de nouveaux agents, outils (tools) et skills dans Claude Code. Il couvre les templates, les interfaces, les options de configuration et les patterns recommandés.

---

## Table des matières

**Partie 1 — Agents**
1. [Types d'agents](#1-types-dagents)
2. [Créer un agent custom (markdown)](#2-créer-un-agent-custom)
3. [Référence complète du frontmatter agent](#3-référence-frontmatter-agent)
4. [Filtrage des outils](#4-filtrage-des-outils)
5. [Exemples d'agents](#5-exemples-dagents)
6. [Créer un agent built-in (TypeScript)](#6-créer-un-agent-built-in)

**Partie 2 — Outils (Tools)**
7. [Architecture d'un outil](#7-architecture-dun-outil)
8. [Template minimal d'un outil](#8-template-minimal)
9. [Méthodes requises](#9-méthodes-requises)
10. [Méthodes optionnelles](#10-méthodes-optionnelles)
11. [Enregistrement d'un outil](#11-enregistrement)
12. [Patterns courants pour les outils](#12-patterns-outils)
13. [Exemple complet d'outil](#13-exemple-complet-outil)

**Partie 3 — Skills**
14. [Anatomie d'un skill](#14-anatomie-dun-skill)
15. [Référence complète du frontmatter skill](#15-référence-frontmatter-skill)
16. [Contenu du skill (prompt)](#16-contenu-du-skill)
17. [Hooks dans les skills](#17-hooks-dans-les-skills)
18. [Exemples de skills](#18-exemples-de-skills)
19. [Créer un skill bundled (TypeScript)](#19-skill-bundled)

**Partie 4 — Référence**
20. [Enums et constantes partagées](#20-enums-et-constantes)
21. [Fichiers clés du codebase](#21-fichiers-clés)

---

# Partie 1 — Agents

## 1. Types d'agents

| Type | Source | Emplacement | Format |
|------|--------|-------------|--------|
| **Custom projet** | Développeur | `.claude/agents/*.md` | Markdown + frontmatter YAML |
| **Custom utilisateur** | Personnel | `~/.claude/agents/*.md` | Markdown + frontmatter YAML |
| **Built-in** | Claude Code | `tools/AgentTool/built-in/*.ts` | TypeScript (`BuiltInAgentDefinition`) |
| **Plugin** | Plugin | Manifeste plugin | Via plugin loader |

---

## 2. Créer un agent custom

### Emplacement

- **Projet** (versionné, partagé) : `.claude/agents/mon-agent.md`
- **Personnel** (tous projets) : `~/.claude/agents/mon-agent.md`

### Structure du fichier

```markdown
---
name: mon-agent
description: Description de quand et pourquoi utiliser cet agent
---

# Instructions système de l'agent

Tu es un expert en [domaine].

Tes responsabilités :
- Tâche 1
- Tâche 2

Tes contraintes :
- Ne jamais modifier les fichiers de configuration
- Toujours vérifier avant de supprimer
```

### Utilisation

L'agent est invoqué via l'outil Agent :

```
Agent({
  subagent_type: "mon-agent",
  description: "Audit du code",
  prompt: "Vérifie les patterns de sécurité dans src/"
})
```

---

## 3. Référence frontmatter agent

### Champs requis

| Champ | Type | Description |
|-------|------|-------------|
| `name` | string | Identifiant unique de l'agent (devient `subagent_type`) |
| `description` | string | Quand utiliser cet agent (affiché à Claude) |

### Champs optionnels — Outils

| Champ | Type | Défaut | Description |
|-------|------|--------|-------------|
| `tools` | string[] | Tous | Liste d'outils autorisés (`["Bash", "Read"]` ou `["*"]`) |
| `disallowedTools` | string[] | Aucun | Outils explicitement interdits |

### Champs optionnels — Modèle

| Champ | Type | Défaut | Description |
|-------|------|--------|-------------|
| `model` | string | inherit | `inherit`, `haiku`, `sonnet`, `opus`, ou ID modèle complet |
| `effort` | string/int | — | `low`, `medium`, `high`, `max`, ou entier 1-100 |

### Champs optionnels — Permissions

| Champ | Type | Défaut | Description |
|-------|------|--------|-------------|
| `permissionMode` | string | default | `default`, `acceptEdits`, `dontAsk`, `plan`, `bubble`, `bypassPermissions` |

### Champs optionnels — Exécution

| Champ | Type | Défaut | Description |
|-------|------|--------|-------------|
| `maxTurns` | number | Illimité | Nombre max de tours API |
| `background` | boolean | false | Toujours en arrière-plan |
| `isolation` | string | Aucune | `worktree` (git isolé) ou `remote` (CCR, Ant-only) |

### Champs optionnels — Contexte

| Champ | Type | Défaut | Description |
|-------|------|--------|-------------|
| `memory` | string | — | `user`, `project`, ou `local` (mémoire persistante) |
| `skills` | string[] | — | Skills à précharger (`["commit", "review"]`) |
| `mcpServers` | array | — | Serveurs MCP (par nom ou inline) |
| `hooks` | object | — | Hooks de session |

### Champs optionnels — UI

| Champ | Type | Défaut | Description |
|-------|------|--------|-------------|
| `color` | string | — | `red`, `blue`, `green`, `yellow`, `purple`, `orange`, `pink`, `cyan` |

---

## 4. Filtrage des outils

### Logique de précédence

```
1. Si tools est défini (allowlist) :
   → L'agent n'a QUE ces outils
   → disallowedTools filtre l'allowlist si les deux sont spécifiés

2. Si tools absent et disallowedTools défini :
   → L'agent a tous les outils SAUF ceux listés

3. Si aucun des deux :
   → L'agent a tous les outils disponibles
```

### Exemples

```yaml
# Agent read-only (comme Explore)
disallowedTools: [Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit]
# Résultat : Glob, Grep, Read, Bash (read-only)

# Agent restreint
tools: [Bash, Read]
# Résultat : uniquement Bash et Read

# Agent complet
tools: ["*"]
# Résultat : tous les outils
```

---

## 5. Exemples d'agents

### Agent minimal — Test Runner

```markdown
---
name: test-runner
description: Run tests after implementation work
tools: [Bash, Read]
model: sonnet
maxTurns: 20
---

Tu es un agent spécialisé dans l'exécution de tests.

1. Trouve la suite de tests du projet
2. Exécute les tests
3. Rapporte les résultats avec un statut clair PASS/FAIL
4. Suggère des corrections si les erreurs sont évidentes

Lance toujours : npm test (ou l'équivalent du framework).
```

### Agent complet — Code Architect

```markdown
---
name: code-architect
description: Software architect for design reviews and planning
tools: [Bash, Read, Glob, Grep]
disallowedTools: [Agent, FileWrite, FileEdit]
model: opus
effort: high
permissionMode: dontAsk
maxTurns: 50
color: purple
isolation: worktree
memory: project
skills: [commit, review]
mcpServers:
  - github
hooks:
  session_start:
    - command: echo "Architecture review started"
---

Tu es un architecte logiciel senior spécialisé dans le design système.

Tes responsabilités :
- Revoir les décisions architecturales
- Identifier les design patterns et anti-patterns
- Proposer des améliorations avec trade-offs
- Considérer la scalabilité et la maintenabilité

Processus d'analyse :
1. Comprendre l'architecture actuelle
2. Identifier les contraintes et parties prenantes
3. Proposer des alternatives avec trade-offs
4. Considérer les implications long-terme

Produis un rapport structuré :
- État actuel
- Problèmes identifiés
- Améliorations recommandées
- Priorité d'implémentation
```

### Agent avec serveurs MCP inline

```markdown
---
name: db-inspector
description: Inspect database schema and data
tools: [Bash, Read]
mcpServers:
  - name: postgres
    transport: stdio
    command: npx
    args: ["@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
requiredMcpServers:
  - postgres
---

Tu es un inspecteur de base de données. Utilise le serveur MCP postgres
pour explorer le schéma et les données.
```

---

## 6. Créer un agent built-in (TypeScript)

Pour les agents compilés dans le binaire Claude Code :

### Structure

```typescript
// tools/AgentTool/built-in/monAgent.ts
import type { BuiltInAgentDefinition } from '../loadAgentsDir.js'

export const MON_AGENT: BuiltInAgentDefinition = {
  agentType: 'mon-agent',
  whenToUse: 'Use this agent when the user needs...',
  source: 'built-in',
  baseDir: 'built-in',

  // Outils
  disallowedTools: ['Agent', 'FileWrite'],

  // Modèle
  model: 'haiku',

  // UI
  color: 'blue',

  // Contexte
  omitClaudeMd: true, // Pour les agents read-only

  // Prompt dynamique
  getSystemPrompt: ({ toolUseContext }) => {
    const mcpServers = toolUseContext?.options?.mcpClients
      ?.map(c => c.name).join(', ') || 'none'
    return `You are an expert agent.
Available MCP servers: ${mcpServers}`
  },
}
```

### Enregistrement

```typescript
// tools/AgentTool/builtInAgents.ts
import { MON_AGENT } from './built-in/monAgent.js'

const agents: AgentDefinition[] = [
  // ... agents existants
  MON_AGENT,
]
```

### Pattern avec Critical System Reminder

Pour les agents qui doivent respecter des contraintes strictes à chaque tour :

```typescript
export const AUDIT_AGENT: BuiltInAgentDefinition = {
  agentType: 'audit',
  whenToUse: 'Verify code quality and security',
  source: 'built-in',
  baseDir: 'built-in',
  color: 'red',
  background: true,
  disallowedTools: ['Agent', 'FileEdit', 'FileWrite'],
  model: 'inherit',
  getSystemPrompt: () => AUDIT_PROMPT,
  criticalSystemReminder_EXPERIMENTAL:
    'CRITICAL: This is an AUDIT-ONLY agent. You CANNOT modify files. ' +
    'You MUST end with VERDICT: PASS, FAIL, or PARTIAL.',
}
```

Le `criticalSystemReminder_EXPERIMENTAL` est **réinjecté à chaque tour** — le modèle ne peut pas l'oublier.

---

# Partie 2 — Outils (Tools)

## 7. Architecture d'un outil

### Organisation des fichiers

```
tools/MonOutil/
├── MonOutil.ts          # Implémentation principale (buildTool)
├── prompt.ts            # Constantes DESCRIPTION et PROMPT
├── UI.tsx               # Composants React de rendu
├── constants.ts         # Nom et constantes
└── [utils.ts]           # Utilitaires optionnels
```

### Interface Tool

Chaque outil est créé via `buildTool()` qui fusionne la définition avec `TOOL_DEFAULTS` :

```typescript
const MonOutil = buildTool({
  name: string,                    // REQUIS — Identifiant unique
  maxResultSizeChars: number,      // REQUIS — Seuil de persistance
  inputSchema: ZodSchema,          // REQUIS — Validation entrées
  outputSchema: ZodSchema,         // Optionnel — Validation sorties
  call(...): ToolResult,           // REQUIS — Handler principal
  description(...): string,        // REQUIS — Description pour le modèle
  prompt(...): string,             // REQUIS — Instructions détaillées
  checkPermissions(...): PermRes,  // REQUIS — Vérification permissions
  mapToolResultToToolResultBlockParam(...): ToolResultBlockParam, // REQUIS
  renderToolUseMessage(...): ReactNode, // REQUIS — Rendu UI
  // ... méthodes optionnelles
})
```

---

## 8. Template minimal

```typescript
// tools/MonOutil/MonOutil.ts
import { z } from 'zod/v4'
import { buildTool, type ToolDef } from '../../Tool.js'
import { lazySchema } from '../../utils/lazySchema.js'

const inputSchema = lazySchema(() =>
  z.strictObject({
    query: z.string().describe('The search query'),
  }),
)
type Input = ReturnType<typeof inputSchema>

const outputSchema = lazySchema(() =>
  z.object({
    result: z.string(),
  }),
)
type Output = z.infer<ReturnType<typeof outputSchema>>

export const MonOutil = buildTool({
  name: 'MonOutil',
  maxResultSizeChars: 100_000,

  get inputSchema(): Input { return inputSchema() },
  get outputSchema() { return outputSchema() },

  async description() {
    return 'Searches for something specific'
  },

  async prompt() {
    return 'Use this tool to search. Provide a query string.'
  },

  async call({ query }) {
    const result = `Found: ${query}`
    return { data: { result } }
  },

  checkPermissions(input) {
    return Promise.resolve({ behavior: 'allow', updatedInput: input })
  },

  mapToolResultToToolResultBlockParam(content, toolUseID) {
    return {
      tool_use_id: toolUseID,
      type: 'tool_result',
      content: content.result,
    }
  },

  renderToolUseMessage({ query }) {
    return query ? `Searching: "${query}"` : null
  },

  // Overrides recommandés
  isConcurrencySafe() { return true },
  isReadOnly() { return true },

} satisfies ToolDef<Input, Output>)
```

---

## 9. Méthodes requises

### `call(args, context, canUseTool, parentMessage, onProgress)`

Le handler principal de l'outil.

```typescript
async call(
  args: z.infer<Input>,           // Paramètres validés par Zod
  context: ToolUseContext,         // Contexte d'exécution
  canUseTool: CanUseToolFn,        // Fonction de permission
  parentMessage: AssistantMessage, // Message assistant parent
  onProgress?: ToolCallProgress,   // Callback de progression
): Promise<ToolResult<Output>> {
  // context contient :
  //   .abortController — pour l'annulation
  //   .getAppState() / .setAppState() — gestion d'état
  //   .readFileState — cache de lecture fichiers
  //   .messages — historique de conversation
  //   .options.tools — tous les outils disponibles

  return {
    data: { /* Output */ },
    // Optionnel :
    // newMessages: Message[],
    // contextModifier: (ctx) => ctx,
  }
}
```

### `checkPermissions(input, context)`

Trois comportements possibles :

```typescript
// Autoriser
return { behavior: 'allow', updatedInput: input }

// Demander à l'utilisateur
return { behavior: 'ask', message: 'Autoriser ?', updatedInput: input }

// Refuser
return { behavior: 'deny', message: 'Opération non autorisée' }
```

**Patterns courants** :

```typescript
// Lecture de fichier
import { checkReadPermissionForTool } from '../../utils/permissions/filesystem.js'
return checkReadPermissionForTool(MonOutil, input, appState.toolPermissionContext)

// Écriture de fichier
import { checkWritePermissionForTool } from '../../utils/permissions/filesystem.js'
return checkWritePermissionForTool(MonOutil, input, appState.toolPermissionContext)
```

### `mapToolResultToToolResultBlockParam(content, toolUseID)`

Convertit la sortie en format API :

```typescript
mapToolResultToToolResultBlockParam(content, toolUseID) {
  return {
    tool_use_id: toolUseID,
    type: 'tool_result',
    content: typeof content === 'string'
      ? content
      : JSON.stringify(content, null, 2),
  }
}
```

### `renderToolUseMessage(input, options)`

Rendu dans le terminal quand l'outil est invoqué :

```typescript
renderToolUseMessage(
  input: Partial<z.infer<Input>>,
  options: { theme: ThemeName; verbose: boolean },
): React.ReactNode {
  if (!input.query) return null
  return `Searching: "${input.query}"`
}
```

---

## 10. Méthodes optionnelles

### Comportement et sécurité

| Méthode | Retour | Défaut | Description |
|---------|--------|--------|-------------|
| `isConcurrencySafe()` | boolean | `false` | Exécution parallèle possible ? |
| `isReadOnly()` | boolean | `false` | Lecture seule ? |
| `isDestructive()` | boolean | `false` | Irréversible (delete, overwrite) ? |
| `isEnabled()` | boolean | `true` | Disponible dans l'environnement actuel ? |
| `validateInput()` | ValidationResult | — | Validation pré-permission |
| `interruptBehavior()` | `'cancel'`/`'block'` | `'block'` | Comportement si l'utilisateur envoie un message |

### Rendu UI

| Méthode | Description |
|---------|-------------|
| `renderToolResultMessage()` | Afficher le résultat |
| `renderToolUseProgressMessage()` | Afficher la progression |
| `renderToolUseErrorMessage()` | Afficher les erreurs |
| `renderToolUseRejectedMessage()` | Afficher le refus |
| `renderToolUseQueuedMessage()` | En attente d'exécution |
| `renderGroupedToolUse()` | Pour les tool uses parallèles |

### Métadonnées et recherche

| Méthode/Propriété | Type | Description |
|-------------------|------|-------------|
| `userFacingName()` | string | Nom affiché dans l'UI |
| `getActivityDescription()` | string | Texte du spinner (`"Reading file..."`) |
| `getToolUseSummary()` | string | Résumé compact |
| `searchHint` | string | Mots-clés pour ToolSearch (3-10 mots) |
| `shouldDefer` | boolean | Envoyer avec `defer_loading: true` |
| `aliases` | string[] | Noms dépréciés (backward compat) |
| `toAutoClassifierInput()` | string | Représentation pour le classifieur de sécurité |

---

## 11. Enregistrement

### Dans tools.ts

```typescript
// tools.ts
import { MonOutil } from './tools/MonOutil/MonOutil.js'

export function getAllBaseTools(): Tools {
  return [
    // ... outils existants
    MonOutil,
  ]
}
```

### Enregistrement conditionnel (feature-gated)

```typescript
// Chargement lazy via require()
const MonOutil = feature('MON_FEATURE_FLAG')
  ? require('./tools/MonOutil/MonOutil.js').MonOutil
  : null

// Dans getAllBaseTools() :
...(MonOutil ? [MonOutil] : []),
```

### Enregistrement conditionnel (environnement)

```typescript
// Ant-only
const DebugTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/DebugTool/DebugTool.js').DebugTool
  : null

// Condition runtime
const PowerShellTool = isPowerShellToolEnabled()
  ? require('./tools/PowerShellTool/PowerShellTool.js').PowerShellTool
  : null
```

---

## 12. Patterns courants pour les outils

### Progression

```typescript
async call(input, context, canUseTool, parentMessage, onProgress) {
  onProgress?.({
    toolUseID: context.toolUseId,
    data: { type: 'tool_progress', message: 'Processing step 1...' },
  })

  // ... travail ...

  onProgress?.({
    toolUseID: context.toolUseId,
    data: { type: 'tool_progress', message: 'Processing step 2...' },
  })

  return { data: result }
}
```

### Chemin de fichier et permissions

```typescript
getPath({ file_path }) {
  return expandPath(file_path)
},

async preparePermissionMatcher({ file_path }) {
  return pattern => matchWildcardPattern(pattern, file_path)
},
```

### Schémas Zod — Bonnes pratiques

```typescript
// Toujours wrapper dans lazySchema() (évite les imports circulaires)
const inputSchema = lazySchema(() =>
  z.strictObject({                          // strictObject rejette les champs inconnus
    required: z.string().describe('Description claire'),
    optional: z.number().optional().describe('Optionnel'),
    enumField: z.enum(['a', 'b', 'c']).describe('Valeurs possibles'),
    withDefault: z.boolean().default(false).describe('Avec défaut'),
  }),
)
```

### Outil read-only concurrent

```typescript
isConcurrencySafe() { return true },   // Exécution parallèle OK
isReadOnly() { return true },           // Ne modifie rien
```

### Outil destructif

```typescript
isDestructive(input) {
  return input.action === 'delete' || input.force === true
},
```

---

## 13. Exemple complet d'outil

```typescript
// tools/HealthCheck/HealthCheck.ts
import { z } from 'zod/v4'
import { buildTool, type ToolDef } from '../../Tool.js'
import { lazySchema } from '../../utils/lazySchema.js'

const inputSchema = lazySchema(() =>
  z.strictObject({
    url: z.string().describe('URL to health check'),
    timeout: z.number().optional().describe('Timeout in ms (default: 5000)'),
    expect_status: z.number().optional().describe('Expected HTTP status (default: 200)'),
  }),
)
type Input = ReturnType<typeof inputSchema>

const outputSchema = lazySchema(() =>
  z.object({
    status: z.number(),
    ok: z.boolean(),
    latency_ms: z.number(),
    body_preview: z.string().optional(),
  }),
)
type Output = z.infer<ReturnType<typeof outputSchema>>

export const HealthCheckTool = buildTool({
  name: 'HealthCheck',
  maxResultSizeChars: 10_000,

  get inputSchema(): Input { return inputSchema() },
  get outputSchema() { return outputSchema() },

  async description() {
    return 'Check if a URL is responding correctly'
  },

  async prompt() {
    return `Use this tool to verify that a web endpoint is healthy.
Provide the URL and optionally a timeout and expected status code.`
  },

  isConcurrencySafe() { return true },
  isReadOnly() { return true },

  isEnabled() {
    return process.env.ENABLE_HEALTH_CHECK === 'true'
  },

  async validateInput({ url }) {
    try {
      new URL(url)
      return { result: true }
    } catch {
      return { result: false, message: `Invalid URL: ${url}` }
    }
  },

  async checkPermissions(input) {
    return { behavior: 'allow', updatedInput: input }
  },

  async call({ url, timeout = 5000, expect_status = 200 }, context) {
    const start = Date.now()
    const controller = new AbortController()
    const timer = setTimeout(() => controller.abort(), timeout)

    try {
      const response = await fetch(url, { signal: controller.signal })
      clearTimeout(timer)
      const latency = Date.now() - start
      const body = await response.text()

      return {
        data: {
          status: response.status,
          ok: response.status === expect_status,
          latency_ms: latency,
          body_preview: body.slice(0, 200),
        },
      }
    } catch (err) {
      clearTimeout(timer)
      return {
        data: {
          status: 0,
          ok: false,
          latency_ms: Date.now() - start,
          body_preview: String(err),
        },
      }
    }
  },

  mapToolResultToToolResultBlockParam(content, toolUseID) {
    const status = content.ok ? '✓ OK' : '✗ FAIL'
    return {
      tool_use_id: toolUseID,
      type: 'tool_result',
      content: `${status} — HTTP ${content.status} (${content.latency_ms}ms)\n${content.body_preview || ''}`,
    }
  },

  renderToolUseMessage({ url }) {
    return url ? `Checking: ${url}` : null
  },

  userFacingName() { return 'Health Check' },

  getActivityDescription({ url }) {
    return url ? `Checking ${new URL(url).hostname}` : 'Health check'
  },

  searchHint: 'health check url endpoint status http ping',

} satisfies ToolDef<Input, Output>)
```

---

# Partie 3 — Skills

## 14. Anatomie d'un skill

Un skill est un **workflow réutilisable** défini en markdown, invocable comme commande slash (`/mon-skill`).

### Emplacements

| Source | Chemin | Portée |
|--------|--------|--------|
| Projet | `.claude/skills/mon-skill/SKILL.md` | Versionné, partagé |
| Utilisateur | `~/.claude/skills/mon-skill/SKILL.md` | Personnel, tous projets |
| Bundled | `skills/bundled/monSkill.ts` | Compilé dans le CLI |
| Managed | `managed-settings.json` | Enterprise/policy |

### Convention de nommage

- Nom = nom du répertoire parent
- kebab-case : `mon-skill`, `code-review`, `deploy-staging`
- Namespaces : `.claude/skills/team/deploy/SKILL.md` → `/team:deploy`

---

## 15. Référence frontmatter skill

```yaml
---
# REQUIS
description: One-line description of what this skill does

# IDENTITÉ (optionnels)
name: Override Name                      # Défaut : nom du répertoire
version: "1.0"                           # Version sémantique

# INVOCATION (optionnels)
user-invocable: true                     # Commande slash disponible (défaut: true)
argument-hint: "[branch] [target]"       # Hint après le nom de commande
arguments: [branch, target]              # Noms pour substitution $branch, $target

# ACTIVATION CONDITIONNELLE (optionnel)
when_to_use: >                           # Quand Claude doit auto-invoquer
  Use when the user wants to deploy.
  Examples: "deploy to staging", "push to prod"
paths: "**/*.ts,**/*.tsx"               # Glob — skill visible après accès à ces fichiers

# OUTILS ET MODÈLE (optionnels)
allowed-tools:                           # Outils autorisés (patterns de permission)
  - Bash(git:*)
  - Bash(npm:*)
  - Read
  - Edit
model: sonnet                            # haiku, sonnet, opus, inherit, ou ID complet
effort: high                             # low, medium, high, max, ou entier

# EXÉCUTION (optionnels)
context: inline                          # inline (défaut) ou fork (sous-agent isolé)
agent: general-purpose                   # Type d'agent si context: fork
shell: bash                              # Shell pour !`cmd` blocks (bash ou powershell)

# HOOKS (optionnel)
hooks:
  PostToolUse:
    - matcher: Write|Edit
      hooks:
        - type: command
          command: prettier --write "$FILE"
          timeout: 30
---
```

### Tableau récapitulatif

| Champ | Type | Requis | Défaut | Description |
|-------|------|--------|--------|-------------|
| `description` | string | **Oui** | — | Description une ligne |
| `name` | string | Non | Nom répertoire | Override du nom |
| `when_to_use` | string | Non | — | Triggers d'auto-invocation |
| `user-invocable` | bool | Non | true | Disponible comme `/command` |
| `argument-hint` | string | Non | — | Hint de paramètres |
| `arguments` | string[] | Non | — | Noms pour substitution `$arg` |
| `allowed-tools` | string[] | Non | Tous | Pattern d'outils autorisés |
| `model` | string | Non | Défaut session | Override du modèle |
| `effort` | string/int | Non | — | Effort de raisonnement |
| `context` | string | Non | inline | `inline` ou `fork` |
| `agent` | string | Non | — | Agent si context: fork |
| `paths` | string/string[] | Non | — | Globs d'activation conditionnelle |
| `shell` | string | Non | bash | Shell pour blocs `!` |
| `version` | string | Non | — | Version du skill |
| `hooks` | object | Non | — | Hooks de lifecycle |

---

## 16. Contenu du skill (prompt)

Le corps markdown après le frontmatter est le **prompt du skill** — les instructions que Claude reçoit.

### Substitutions disponibles

| Variable | Description |
|----------|-------------|
| `$arg1`, `$arg2`, ... | Arguments fournis par l'utilisateur |
| `$branch`, `$target` | Si `arguments: [branch, target]` dans le frontmatter |
| `${CLAUDE_SKILL_DIR}` | Chemin vers le répertoire du skill |
| `${CLAUDE_SESSION_ID}` | ID de la session courante |

### Blocs shell inline

Les skills peuvent exécuter des commandes directement :

```markdown
Vérifie l'état du repo :

!`git status && git log --oneline -5`

Ou en bloc multi-lignes :

\`\`\`!
npm ci
npm run build
npm test
\`\`\`
```

### Inline vs Fork

| Mode | Description | Cas d'usage |
|------|-------------|-------------|
| `inline` (défaut) | Le prompt s'insère dans la conversation | Besoin du contexte, pilotage utilisateur |
| `fork` | Sous-agent isolé, propre budget tokens | Tâches autonomes (tests, déploiement, batch) |

---

## 17. Hooks dans les skills

### Événements disponibles

| Événement | Quand |
|-----------|-------|
| `PreToolUse` | Avant l'exécution d'un outil (peut bloquer) |
| `PostToolUse` | Après exécution réussie |
| `PostToolUseFailure` | Après échec d'un outil |
| `PreCompact` / `PostCompact` | Autour de la compaction |
| `Stop` | Quand Claude s'arrête |
| `SessionStart` | Au démarrage de la session |
| `UserPromptSubmit` | Quand l'utilisateur envoie un message |

### Types de hooks

#### Hook commande

```yaml
hooks:
  PostToolUse:
    - matcher: Write|Edit
      hooks:
        - type: command
          command: prettier --write "$FILE"
          timeout: 30
          statusMessage: "Formatting..."
```

#### Hook prompt (évaluation LLM)

```yaml
        - type: prompt
          prompt: "Is this change safe? $ARGUMENTS"
          model: haiku
```

#### Hook agent

```yaml
        - type: agent
          prompt: "Verify tests pass after this change"
          model: haiku
          async: true
```

#### Hook HTTP

```yaml
        - type: http
          url: https://hooks.example.com/notify
          headers:
            Authorization: "Bearer $API_TOKEN"
          allowedEnvVars: [API_TOKEN]
          timeout: 10
```

### Entrée/Sortie des hooks

**Entrée** (stdin JSON) :
```json
{
  "session_id": "abc-123",
  "tool_name": "Write",
  "tool_input": { "file_path": "/path/file.ts", "content": "..." },
  "tool_response": { "success": true }
}
```

**Sortie** (stdout JSON) :
```json
{
  "continue": true,
  "systemMessage": "File formatted successfully",
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow"
  }
}
```

---

## 18. Exemples de skills

### Skill minimal — Simplifier le code

```markdown
---
description: Review changed code for reuse, quality, and efficiency
allowed-tools: [Read, Edit, Glob, Grep, Bash]
---

# Simplify

Review the code that was recently changed. Look for:
1. Duplicated logic that can be extracted
2. Unnecessary complexity
3. Performance issues
4. Missing error handling

Fix any issues found. Keep changes minimal and focused.
```

### Skill avec arguments — Cherry-pick

```markdown
---
description: Cherry-pick a commit to another branch
argument-hint: "[commit-sha] [target-branch]"
arguments: [sha, branch]
allowed-tools: [Bash(git:*)]
---

# Cherry-pick

Cherry-pick commit `$sha` onto branch `$branch`.

!`git fetch origin $branch`
!`git checkout $branch`
!`git cherry-pick $sha`

If conflicts arise, resolve them and commit.
```

### Skill conditionnel avec hooks — Format TypeScript

```markdown
---
description: Format TypeScript files after editing
paths: "**/*.ts,**/*.tsx"
allowed-tools: [Bash, Read, Edit]
hooks:
  PostToolUse:
    - matcher: Write|Edit
      hooks:
        - type: command
          command: 'npx prettier --write "$FILE"'
          if: "Edit(*.ts)|Edit(*.tsx)|Write(*.ts)|Write(*.tsx)"
---

# TypeScript Formatter

After modifying TypeScript files, format them with Prettier.
This skill auto-activates when .ts or .tsx files are touched.
```

### Skill forké — Deploy workflow

```markdown
---
description: Deploy a branch to staging
argument-hint: "[branch-name]"
arguments: [branch]
allowed-tools: [Bash(git:*), Bash(gh:*), Bash(npm:*), Read]
context: fork
agent: general-purpose
model: sonnet
---

# Deploy to Staging

## Steps

### 1. Verify branch
!`git fetch origin $branch && git log --oneline -3 origin/$branch`

### 2. Run tests
!`npm ci && npm test`

### 3. Deploy
!`git push origin $branch:staging --force`

### 4. Verify
!`gh run list --branch staging --limit 1`

**Success**: Staging runs the latest from `$branch`.
```

---

## 19. Créer un skill bundled (TypeScript)

Pour les skills compilés dans le CLI :

```typescript
// skills/bundled/monSkill.ts
import { registerBundledSkill } from '../bundledSkills.js'

const PROMPT = `# Mon Skill

Instructions détaillées du skill...
`

export function registerMonSkill(): void {
  registerBundledSkill({
    name: 'mon-skill',
    description: 'What this skill does',
    whenToUse: 'When to invoke this skill',
    userInvocable: true,
    allowedTools: ['Read', 'Grep', 'Bash(npm:*)'],
    argumentHint: '[args]',
    model: 'haiku',
    context: 'inline',

    async getPromptForCommand(args, context) {
      let prompt = PROMPT
      if (args) {
        prompt += `\n\n## User Input\n\n${args}`
      }
      return [{ type: 'text', text: prompt }]
    },
  })
}
```

Enregistrement dans `skills/bundled/index.ts` :

```typescript
import { registerMonSkill } from './monSkill.js'

export function initBundledSkills(): void {
  // ... autres skills
  registerMonSkill()
}
```

---

# Partie 4 — Référence

## 20. Enums et constantes partagées

### Couleurs d'agent

```
red | blue | green | yellow | purple | orange | pink | cyan
```

### Niveaux d'effort

```
low | medium | high | max | entier (1-100)
```
Note : `max` et les valeurs numériques sont Ant-only.

### Modes de permission

| Mode | Comportement |
|------|-------------|
| `default` | Demande à l'utilisateur |
| `acceptEdits` | Auto-approuve les éditions |
| `dontAsk` | Pas de prompt (safe pour read-only) |
| `plan` | Présente un plan avant exécution |
| `bubble` | Remonte au parent (multi-agents) |
| `bypassPermissions` | Tout autoriser (admin) |
| `auto` | Classifieur ML (Ant-only) |

### Concurrence des outils

| Concurrent (parallèle) | Non-concurrent (exclusif) |
|-------------------------|--------------------------|
| Glob, Grep, Read, WebFetch, WebSearch, Agent | Bash, Edit, Write, NotebookEdit, Skill |

---

## 21. Fichiers clés du codebase

### Agents

| Fichier | Rôle |
|---------|------|
| `tools/AgentTool/loadAgentsDir.ts` | Chargement et parsing des agents custom |
| `tools/AgentTool/builtInAgents.ts` | Registre des agents built-in |
| `tools/AgentTool/built-in/*.ts` | Définitions des agents built-in |
| `tools/AgentTool/constants.ts` | Constantes (ONE_SHOT_BUILTIN_AGENT_TYPES, etc.) |
| `tools/AgentTool/AgentTool.tsx` | Outil Agent (spawn des sous-agents) |

### Outils

| Fichier | Rôle |
|---------|------|
| `Tool.ts` | Interface Tool complète, buildTool(), TOOL_DEFAULTS |
| `tools.ts` | Registre getAllBaseTools(), getTools(), assembleToolPool() |
| `utils/api.ts` | toolToAPISchema() — conversion pour l'API |
| `services/tools/toolExecution.ts` | Pipeline d'exécution runToolUse() |
| `services/tools/StreamingToolExecutor.ts` | File d'attente concurrente/exclusive |
| `services/tools/toolHooks.ts` | Hooks pre/post exécution |

### Skills

| Fichier | Rôle |
|---------|------|
| `skills/loadSkillsDir.ts` | Chargement depuis disque (user, project, managed) |
| `skills/bundledSkills.ts` | Enregistrement des skills compilés |
| `skills/bundled/*.ts` | Implémentations des skills bundled |
| `types/command.ts` | Types Command et PromptCommand |
| `tools/SkillTool/SkillTool.ts` | Outil Skill (invocation par le modèle) |
