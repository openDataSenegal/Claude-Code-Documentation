# Système de Skills — Documentation complète

Les workflows réutilisables de Claude Code : 14+ skills bundled, chargement multi-sources, hooks inline, activation conditionnelle et intégration MCP.

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Catalogue des skills bundled](#2-catalogue-bundled)
3. [Format d'un skill (SKILL.md)](#3-format-skill)
4. [Frontmatter — Référence complète](#4-frontmatter)
5. [Contenu du skill (prompt)](#5-contenu-prompt)
6. [Pipeline de chargement](#6-pipeline-chargement)
7. [Découverte dynamique](#7-découverte-dynamique)
8. [Activation conditionnelle (paths)](#8-activation-conditionnelle)
9. [Détection de changements](#9-détection-changements)
10. [Invocation via SkillTool](#10-invocation)
11. [Contexte inline vs fork](#11-inline-vs-fork)
12. [Hooks dans les skills](#12-hooks)
13. [Arguments et substitution](#13-arguments)
14. [Fichiers embarqués](#14-fichiers-embarqués)
15. [Intégration MCP](#15-intégration-mcp)
16. [Skill listing (annonce au modèle)](#16-skill-listing)
17. [Création de skills (/skillify)](#17-skillify)
18. [Patterns de génération de prompt](#18-patterns-prompt)
19. [Fichiers clés](#19-fichiers-clés)

---

## 1. Vue d'ensemble

Les skills sont des **workflows réutilisables** définis en markdown, invocables comme commandes slash (`/mon-skill`) ou automatiquement par Claude. Ils encapsulent des instructions, des outils autorisés, et optionnellement des hooks de lifecycle.

### Sources de skills (par priorité)

| Source | Emplacement | Portée |
|--------|-------------|--------|
| **Utilisateur** | `~/.claude/skills/` | Personnel, tous projets |
| **Projet** | `.claude/skills/` | Versionné, partagé |
| **Bundled** | Compilé dans le CLI | Toujours disponible |
| **MCP** | Serveurs MCP connectés | Dynamique |
| **Plugin** | Via manifeste plugin | Plugin-scoped |
| **Managed** | Settings managés (enterprise) | Policy-level |

### Principe

```
Skill = Prompt structuré + Outils autorisés + Contexte d'exécution + Hooks optionnels
```

Un skill est plus qu'un prompt : il définit un **environnement d'exécution** avec des permissions spécifiques, un modèle optionnel, et des hooks qui s'exécutent pendant son invocation.

---

## 2. Catalogue des skills bundled

### Skills toujours disponibles

| Skill | Description | User-only | Outils |
|-------|-------------|-----------|--------|
| `/batch` | Refactoring parallèle en masse (5-30 agents worktree, chacun ouvre une PR) | Oui | Plan, Exit, Agent, Ask |
| `/simplify` | Review du code modifié : réutilisation, qualité, efficience (3 agents parallèles) | Non | Tous (via Agent) |
| `/update-config` | Modifier settings.json : permissions, hooks, env vars, MCP, plugins | Non | Read |
| `/debug` | Activer les logs debug, lire le journal de session, diagnostiquer | Oui | Read, Grep, Glob |
| `/keybindings-help` | Personnaliser les raccourcis clavier (~/.claude/keybindings.json) | Caché | Read |
| `/verify` | Vérifier qu'un changement fonctionne end-to-end (Ant-only) | Non | (fichiers embarqués) |
| `/lorem-ipsum` | Générer N tokens de filler text pour tester le long contexte (Ant-only) | Non | Aucun |
| `/remember` | Revoir et organiser la mémoire auto, proposer des promotions (Ant-only) | Non | Aucun |
| `/skillify` | Capturer un processus de session en skill réutilisable (Ant-only) | Oui | Read, Write, Edit, Glob, Grep, Ask |
| `/stuck` | Diagnostiquer les sessions gelées, poster sur Slack (Ant-only) | Oui | (commandes diagnostic) |

### Skills feature-gated

| Skill | Gate | Description |
|-------|------|-------------|
| `/loop` | `AGENT_TRIGGERS` | Exécuter un prompt/commande à intervalle régulier (cron) |
| `/schedule` | `AGENT_TRIGGERS_REMOTE` | Planifier des agents distants récurrents via CCR |
| `/claude-api` | `BUILDING_CLAUDE_APPS` | Aide au développement avec l'API Claude / SDK Anthropic |
| `/claude-in-chrome` | Auto-détecté | Automatiser Chrome : cliquer, remplir, capturer, naviguer |
| `/dream` | `KAIROS` / `KAIROS_DREAM` | Consolidation mémoire (assistant mode) |
| `/hunter` | `REVIEW_ARTIFACT` | Review d'artefacts |
| `/run-skill-generator` | `RUN_SKILL_GENERATOR` | Générer des skills dynamiquement |

---

## 3. Format d'un skill (SKILL.md)

### Structure du répertoire

```
.claude/skills/mon-skill/
├── SKILL.md              # Définition du skill (requis)
└── reference.md          # Fichiers de référence optionnels
```

### Naming

- Nom du skill = nom du répertoire parent
- Convention : kebab-case (`mon-skill`, `code-review`, `deploy-staging`)
- Namespaces : `.claude/skills/team/deploy/SKILL.md` → `/team:deploy`

### Template complet

```markdown
---
name: Mon Skill
description: Description courte pour l'autocomplete et le listing
when_to_use: >
  Utiliser quand l'utilisateur veut faire X.
  Exemples: "fais X", "lance X", "exécute X"
argument-hint: "[branch] [target]"
arguments: [branch, target]
allowed-tools:
  - Bash(git:*)
  - Bash(npm:*)
  - Read
  - Edit
model: sonnet
effort: high
context: fork
agent: general-purpose
user-invocable: true
paths: "**/*.ts,**/*.tsx"
hooks:
  PostToolUse:
    - matcher: Write|Edit
      hooks:
        - type: command
          command: "prettier --write $FILE"
          timeout: 30
---

# Mon Skill

## Objectif
Accomplir X de manière structurée.

## Étapes

### 1. Analyser
Examiner le contexte avec les outils read-only.

### 2. Exécuter
Appliquer les modifications nécessaires.

### 3. Vérifier
!`npm test`

**Succès** : Tous les tests passent.
```

---

## 4. Frontmatter — Référence complète

### Champs requis

| Champ | Type | Description |
|-------|------|-------------|
| `description` | string | Description une ligne (autocomplete, listing, modèle) |

### Champs d'identité

| Champ | Type | Défaut | Description |
|-------|------|--------|-------------|
| `name` | string | Nom répertoire | Override du nom affiché |
| `version` | string | — | Version sémantique du skill |

### Champs d'invocation

| Champ | Type | Défaut | Description |
|-------|------|--------|-------------|
| `user-invocable` | boolean | `true` | Visible dans l'autocomplete `/` |
| `disable-model-invocation` | boolean | `false` | Empêche Claude de l'invoquer (user-only) |
| `when_to_use` | string | — | Quand Claude doit auto-invoquer (trigger phrases) |
| `argument-hint` | string | — | Hint affiché après le nom (`[branch] [target]`) |
| `arguments` | string[] | — | Noms pour substitution `$branch`, `$target` |

### Champs d'exécution

| Champ | Type | Défaut | Description |
|-------|------|--------|-------------|
| `allowed-tools` | string[] | Tous | Outils autorisés (patterns de permission) |
| `model` | string | inherit | `haiku`, `sonnet`, `opus`, `inherit`, ou ID complet |
| `effort` | string/int | — | `low`, `medium`, `high`, `max`, ou entier 1-100 |
| `context` | string | `inline` | `inline` (dans la conversation) ou `fork` (sous-agent) |
| `agent` | string | — | Type d'agent si `context: fork` |
| `shell` | string | `bash` | Shell pour les blocs `!` (`bash` ou `powershell`) |

### Champs de filtrage

| Champ | Type | Défaut | Description |
|-------|------|--------|-------------|
| `paths` | string/string[] | — | Glob gitignore — skill visible seulement après accès aux fichiers matchés |

### Champs de hooks

| Champ | Type | Défaut | Description |
|-------|------|--------|-------------|
| `hooks` | object | — | Hooks de lifecycle (PreToolUse, PostToolUse, etc.) |

---

## 5. Contenu du skill (prompt)

Le corps markdown après le frontmatter constitue les **instructions** que Claude reçoit quand le skill est invoqué.

### Substitutions disponibles

| Variable | Description |
|----------|-------------|
| `$arg1`, `$arg2`, ... | Arguments positionnels fournis par l'utilisateur |
| `$branch`, `$target` | Si `arguments: [branch, target]` |
| `${CLAUDE_SKILL_DIR}` | Chemin absolu du répertoire du skill |
| `${CLAUDE_SESSION_ID}` | UUID de la session courante |

### Blocs shell inline

```markdown
Vérifie le repo :

!`git status && git log --oneline -5`

Ou multi-lignes :

\`\`\`!
npm ci
npm run build
npm test
\`\`\`
```

- Utilise le `shell` du frontmatter (`bash` ou `powershell`)
- **BLOQUÉ pour les skills MCP** (sécurité : MCP est distant, non trusté)

---

## 6. Pipeline de chargement

### Séquence complète

```
1. DÉMARRAGE (initBundledSkills)
   └── Enregistre les skills bundled → getBundledSkills()

2. CHARGEMENT DISQUE
   ├── ~/.claude/skills/ (utilisateur global)
   ├── .claude/skills/ (projet)
   └── Sépare : unconditional (sans paths) vs conditional (avec paths)

3. PENDANT LA SESSION
   ├── Read/Edit/Write sur un fichier
   │   ├── discoverSkillDirsForPaths() → remonte vers le CWD
   │   └── addSkillDirectories() → charge les skills découverts
   ├── activateConditionalSkillsForPaths() → match contre les patterns
   └── skillChangeDetector → clearSkillCaches + resetSentSkillNames

4. AU TOUR SUIVANT
   └── skill_listing attachment inclut tous les skills actifs

5. INVOCATION
   ├── Utilisateur : /mon-skill [args]
   └── Claude : Skill({ skill: "mon-skill", args: "..." })

6. EXÉCUTION
   ├── inline → contenu expandu dans la conversation
   └── fork → sous-agent isolé avec budget tokens séparé

7. TRACKING
   └── STATE.invokedSkills (préservé across compaction)
```

### Déduplication

- Skills avec le même chemin résolu (symlinks) sont dédupliqués
- Premier source trouvé gagne (utilisateur > projet > bundled)
- MCP skills identifiés par `loadedFrom: 'mcp'`

---

## 7. Découverte dynamique

### Pendant les opérations fichier

Quand Claude lit/édite/écrit un fichier, le système découvre les skills dans les répertoires entre le fichier et le CWD :

```
Fichier : /project/src/pages/login/component.tsx
CWD :     /project

Répertoires scannés :
  /project/src/pages/login/.claude/skills/
  /project/src/pages/.claude/skills/
  /project/src/.claude/skills/
  (PAS /project/.claude/skills/ — déjà chargé au démarrage)
```

### Fonctions

```typescript
discoverSkillDirsForPaths(filePaths, cwd)
→ Remonte depuis le parent du fichier jusqu'au CWD
→ Cherche .claude/skills/
→ Ignore les répertoires gitignored
→ Retourne triés par profondeur (le plus profond d'abord)

addSkillDirectories(dirs)
→ Charge les SKILL.md depuis les répertoires découverts
→ Merge dans la map dynamicSkills (profond override shallow)
→ Émet skillsLoaded → clearCommandMemoizationCaches()
```

---

## 8. Activation conditionnelle (paths)

Les skills avec `paths` dans le frontmatter sont chargés mais **dormants** au démarrage. Ils s'activent uniquement quand Claude touche un fichier correspondant.

### Gestion d'état

```typescript
conditionalSkills    // Map<name, Command> — skills dormants avec paths
dynamicSkills        // Map<name, Command> — skills actifs (unconditional + activés)
activatedNames       // Set<string> — record permanent des activations
```

### Pattern matching (style gitignore)

```yaml
paths: "src/**/*.ts"      # Tous les .ts sous src/
paths: "Makefile"          # Makefile à la racine
paths: "docs/**"           # Tout sous docs/
paths:                     # Plusieurs patterns
  - "**/*.ts"
  - "**/*.tsx"
```

### Flux d'activation

```
1. Claude lit /project/src/pages/login.tsx
2. activateConditionalSkillsForPaths() appelé
3. Skill avec paths: "src/**" match → activé
4. Déplacé de conditionalSkills → dynamicSkills
5. skill_listing mis à jour au prochain tour
```

---

## 9. Détection de changements

**Fichier** : `utils/skills/skillChangeDetector.ts`

Utilise Chokidar pour surveiller les modifications de skills :

| Paramètre | Valeur |
|-----------|--------|
| Répertoires surveillés | `~/.claude/skills/`, `.claude/skills/` |
| Profondeur | 2 (skill-name/SKILL.md) |
| Debounce | 300ms |
| Stabilité fichier | 1000ms poll, 500ms intervalle |
| Mode | Polling (Bun, pour éviter deadlock) |

### Sur changement

```
1. clearSkillCaches() → vide la liste memoizée
2. resetSentSkillNames() → force la ré-émission du skill_listing
3. Signal skillsChanged émis
```

---

## 10. Invocation via SkillTool

**Fichier** : `tools/SkillTool/SkillTool.ts`

### Schéma d'entrée

```typescript
{
  skill: string,     // Nom du skill ("commit", "review-pr")
  args?: string,     // Arguments optionnels
}
```

### Flux d'exécution

```
1. Charger le skill : command.getPromptForCommand(args, context)
2. Substituer les variables ($arg, ${CLAUDE_SKILL_DIR}, ${CLAUDE_SESSION_ID})
3. Exécuter les blocs !backtick (sauf MCP — bloqué)
4. Si context: inline → créer message système avec le contenu
5. Si context: fork → créer sous-agent isolé
6. Tracker dans STATE.invokedSkills (préservé across compaction)
7. Retourner les résultats
```

---

## 11. Contexte inline vs fork

| | Inline (défaut) | Fork |
|---|---|---|
| **Contexte** | Même conversation | Sous-agent isolé |
| **Budget tokens** | Partagé avec le parent | Séparé (configurable via effort) |
| **Historique** | Voit tout l'historique | Ne voit que le prompt du skill |
| **Interaction** | Interaction mid-process avec l'utilisateur | Pas d'interaction directe |
| **Vitesse** | Rapide | Plus lent (overhead agent) |
| **Cas d'usage** | Skills interactifs, courts | Tâches autonomes, longues |

### Configuration fork

```yaml
context: fork
agent: general-purpose    # Type d'agent
effort: high              # Budget tokens (~100K)
model: sonnet             # Override modèle optionnel
```

---

## 12. Hooks dans les skills

Les skills peuvent enregistrer des hooks qui s'exécutent pendant leur invocation.

### Événements supportés

| Événement | Description | Peut bloquer |
|-----------|-------------|-------------|
| `PreToolUse` | Avant l'exécution d'un outil | Oui |
| `PostToolUse` | Après exécution réussie | Non |
| `PostToolUseFailure` | Après échec d'un outil | Non |
| `PermissionRequest` | Avant prompt de permission | Oui |
| `Notification` | Sur notification | Non |
| `Stop` | Quand Claude s'arrête | Non |
| `PreCompact` / `PostCompact` | Autour de la compaction | Non |
| `UserPromptSubmit` | Envoi d'un message utilisateur | Non |
| `SessionStart` | Démarrage de session | Non |

### Types de hooks

#### Commande shell

```yaml
hooks:
  PostToolUse:
    - matcher: Write|Edit
      hooks:
        - type: command
          command: "prettier --write $FILE"
          timeout: 30
```

#### Prompt LLM

```yaml
        - type: prompt
          prompt: "Is this change safe? $ARGUMENTS"
          model: claude-haiku
```

#### Agent multi-turn

```yaml
        - type: agent
          prompt: "Verify tests pass after this change"
          timeout: 60
```

---

## 13. Arguments et substitution

### Définition

```yaml
arguments: [branch, target_branch]
argument-hint: "[branch] [target-branch]"
```

### Utilisation dans le contenu

```markdown
Cherry-pick commit from `$branch` to `$target_branch`.

!`git checkout $target_branch && git cherry-pick $branch`
```

### Invocation

```
/mon-skill main release-1.0
```

Substitue : `$branch` → `main`, `$target_branch` → `release-1.0`

---

## 14. Fichiers embarqués

### Skills bundled avec fichiers

Certains skills bundled embarquent des fichiers de référence extraits sur disque à l'invocation :

```typescript
registerBundledSkill({
  name: 'claude-api',
  files: {
    'reference/python-sdk.md': '...contenu...',
    'reference/typescript-sdk.md': '...contenu...',
    'examples/streaming.md': '...contenu...',
  },
  // ...
})
```

### Extraction

```
Emplacement : ~/.claude/.nonce/bundled-skills/<skillName>/
Permissions : 0o700 (nonce par processus)
Protection : O_NOFOLLOW + O_EXCL (symlink/traversal)
Memoizé : une seule extraction par processus
```

### Accès

Le prompt reçoit un préfixe :
```
Base directory for this skill: /Users/x/.claude/.abc123/bundled-skills/claude-api/
```

Le modèle peut ensuite `Read` les fichiers de référence.

---

## 15. Intégration MCP

Les serveurs MCP peuvent fournir des skills via le protocole MCP.

### Caractéristiques MCP skills

| Propriété | Valeur |
|-----------|--------|
| `loadedFrom` | `'mcp'` |
| `source` | `'mcp'` |
| Blocs `!` shell | **BLOQUÉS** (MCP est distant, non trusté) |
| Activation conditionnelle | Non (apparaissent immédiatement) |
| Frontmatter | Même parsing que les skills disque |

### Enregistrement

```typescript
registerMCPSkillBuilders({
  createSkillCommand: typeof createSkillCommand,
  parseSkillFrontmatterFields: typeof parseSkillFrontmatterFields,
})
```

---

## 16. Skill listing (annonce au modèle)

### Attachment type : `skill_listing`

Créé au début de chaque tour avec la liste des skills disponibles. Le modèle utilise cette liste pour décider quand invoquer un skill.

### Budget de listing

| Paramètre | Valeur |
|-----------|--------|
| Budget par défaut | 1% de la fenêtre de contexte (8K chars pour 200K) |
| Cap par entrée | 250 caractères max |
| Skills bundled | Jamais tronqués |
| Skills non-bundled | Tronqués ou noms-only si budget dépassé |

### Cycle de vie

```
1. Créé au début du tour avec les skills actifs
2. Strippé pendant la compaction
3. Ré-injecté post-compaction via resetSentSkillNames()
4. Mis à jour quand skillChangeDetector détecte un changement
5. Enrichi quand de nouveaux skills sont découverts dynamiquement
```

---

## 17. Création de skills (/skillify)

Le skill `/skillify` (Ant-only) capture un processus de la session courante en un skill réutilisable.

### Processus

```
1. ANALYSE DE LA SESSION
   ├── Extraire la mémoire de session
   ├── Extraire tous les messages utilisateur
   └── Identifier le processus répétable (inputs, étapes, critères de succès)

2. INTERVIEW UTILISATEUR (4 rounds via AskUserQuestion)
   ├── Round 1 : Confirmer le processus identifié
   ├── Round 2 : Clarifier les inputs et variations
   ├── Round 3 : Définir les critères de succès
   └── Round 4 : Choisir l'emplacement (projet vs personnel)

3. ÉCRITURE DU SKILL
   ├── Générer le frontmatter (name, description, when_to_use, allowed-tools)
   ├── Rédiger les instructions structurées
   └── Écrire dans .claude/skills/<name>/SKILL.md

4. CONFIRMATION
   └── Présenter le skill créé, demander validation
```

### Emplacement de sortie

| Choix | Chemin |
|-------|--------|
| Projet (partagé) | `.claude/skills/<name>/SKILL.md` |
| Personnel (global) | `~/.claude/skills/<name>/SKILL.md` |

---

## 18. Patterns de génération de prompt

Les skills bundled utilisent 5 patterns distincts pour générer leurs prompts :

### Pattern 1 — Statique + args

```typescript
async getPromptForCommand(args) {
  let prompt = BASE_PROMPT
  if (args) prompt += `\n\n## User Request\n\n${args}`
  return [{ type: 'text', text: prompt }]
}
```

**Skills** : remember, stuck, simplify, claudeInChrome, batch

### Pattern 2 — Parsing d'arguments

```typescript
async getPromptForCommand(args) {
  const count = parseInt(args) || 10000
  const text = generateLoremIpsum(count)
  return [{ type: 'text', text }]
}
```

**Skills** : lorem-ipsum (parse count), loop (parse interval + prompt)

### Pattern 3 — Contexte dynamique

```typescript
async getPromptForCommand(args, context) {
  const env = await detectEnvironment()
  const repos = await listRepos()
  const timezone = detectTimezone()
  return [{ type: 'text', text: buildPrompt(env, repos, timezone, args) }]
}
```

**Skills** : schedule (détecte env, repos, timezone, MCP), debug (tail log, enable logging)

### Pattern 4 — Tables générées

```typescript
async getPromptForCommand(args) {
  const actions = getAvailableActions()
  const contexts = getAvailableContexts()
  const reserved = getReservedShortcuts()
  return [{ type: 'text', text: buildTables(actions, contexts, reserved) }]
}
```

**Skills** : keybindings (tables actions/contextes), update-config (JSON schema généré)

### Pattern 5 — Contenu lazy-loaded

```typescript
async getPromptForCommand(args) {
  const { SKILL_PROMPT, SKILL_FILES } = await import('./claudeApiContent.js')
  const language = detectProjectLanguage()
  const filteredDocs = filterByLanguage(SKILL_FILES, language)
  return [{ type: 'text', text: buildPrompt(SKILL_PROMPT, filteredDocs, args) }]
}
```

**Skills** : claude-api (247KB docs lazy-loaded, filtré par langue), verify (SKILL_MD + examples)

---

## 19. Fichiers clés

### Infrastructure

| Fichier | Rôle | Taille |
|---------|------|--------|
| `skills/loadSkillsDir.ts` | Chargement, parsing, validation, découverte | 1000+ lignes |
| `skills/bundledSkills.ts` | Enregistrement, extraction fichiers, tracking | |
| `skills/bundled/index.ts` | `initBundledSkills()` — registration conditionnelle | |
| `tools/SkillTool/SkillTool.ts` | Invocation inline/fork, substitution, tracking | |
| `tools/SkillTool/prompt.ts` | Formatage du skill listing | |
| `utils/skills/skillChangeDetector.ts` | Chokidar watcher, debounce, signal | |
| `types/command.ts` | Types Command, PromptCommand, CommandBase | |

### Skills bundled

| Fichier | Skill |
|---------|-------|
| `skills/bundled/batch.ts` | `/batch` |
| `skills/bundled/simplify.ts` | `/simplify` |
| `skills/bundled/updateConfig.ts` | `/update-config` |
| `skills/bundled/debug.ts` | `/debug` |
| `skills/bundled/keybindings.ts` | `/keybindings-help` |
| `skills/bundled/verify.ts` | `/verify` |
| `skills/bundled/verifyContent.ts` | Contenu embarqué verify |
| `skills/bundled/loremIpsum.ts` | `/lorem-ipsum` |
| `skills/bundled/remember.ts` | `/remember` |
| `skills/bundled/skillify.ts` | `/skillify` |
| `skills/bundled/stuck.ts` | `/stuck` |
| `skills/bundled/loop.ts` | `/loop` |
| `skills/bundled/scheduleRemoteAgents.ts` | `/schedule` |
| `skills/bundled/claudeApi.ts` | `/claude-api` |
| `skills/bundled/claudeApiContent.ts` | Contenu embarqué claude-api (247KB) |
| `skills/bundled/claudeInChrome.ts` | `/claude-in-chrome` |
