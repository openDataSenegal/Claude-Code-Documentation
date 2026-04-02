# Système de Permissions — Documentation complète

Le gardien de sécurité de Claude Code : comment chaque action est autorisée, refusée ou soumise à validation.

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Modes de permission](#2-modes-de-permission)
3. [Règles de permission](#3-règles-de-permission)
4. [Pipeline de décision](#4-pipeline-de-décision)
5. [Classifieur auto-mode (ML)](#5-classifieur-auto-mode)
6. [Détection de patterns dangereux](#6-patterns-dangereux)
7. [Handlers de permission](#7-handlers)
8. [Chargement et persistance des règles](#8-chargement-et-persistance)
9. [Fichiers et chemins protégés](#9-fichiers-protégés)
10. [Suivi des refus](#10-suivi-des-refus)
11. [Types de décision](#11-types-de-décision)
12. [Fichiers clés](#12-fichiers-clés)

---

## 1. Vue d'ensemble

Chaque invocation d'outil passe par un pipeline de permission en 3 couches :

```
Requête d'outil
    │
    ▼
[1] RÈGLES EXPLICITES (deny/ask/allow lists)
    │
    ▼
[2] MODE DE PERMISSION (default/bypass/acceptEdits/dontAsk/plan/auto)
    │
    ▼
[3] CLASSIFIEURS ML (Bash classifier + Transcript classifier)
    │
    ▼
[4] HOOKS (permission_request)
    │
    ▼
[5] INTERACTION UTILISATEUR (si encore indécis)
    │
    ▼
DÉCISION : allow | deny | ask
```

---

## 2. Modes de permission

### Modes externes (utilisateur)

| Mode | Comportement | Cas d'usage |
|------|-------------|-------------|
| `default` | Demande avant les opérations risquées | Usage standard |
| `acceptEdits` | Auto-approuve les éditions de fichiers dans le working dir | Confiance modérée |
| `plan` | Pause et présentation du plan avant exécution | Revue avant action |
| `bypassPermissions` | Tout autoriser sans demander | YOLO / scripts CI |
| `dontAsk` | Convertit tous les `ask` en `deny` | Mode headless strict |

### Modes internes

| Mode | Comportement | Contexte |
|------|-------------|---------|
| `auto` | Classifieur ML décide (Ant-only, `TRANSCRIPT_CLASSIFIER`) | Automatisation avancée |
| `bubble` | Remonte les demandes au parent | Sous-agents en swarm |

---

## 3. Règles de permission

### Structure d'une règle

```typescript
PermissionRule {
  source: PermissionRuleSource,     // D'où vient la règle
  ruleBehavior: 'allow' | 'deny' | 'ask',
  ruleValue: {
    toolName: string,               // "Bash", "mcp__server__tool"
    ruleContent?: string,           // Pattern optionnel "python:*"
  }
}
```

### Sources de règles (par priorité)

| Source | Emplacement | Persistance | Priorité |
|--------|-------------|------------|----------|
| `policySettings` | Managed settings (read-only) | Permanente | Haute |
| `userSettings` | `~/.claude/settings.json` | Permanente | |
| `projectSettings` | `.claude/settings.json` | Permanente | |
| `localSettings` | `.claude.local.json` | Permanente | |
| `flagSettings` | GrowthBook feature flags | Dynamique | |
| `cliArg` | `--allowed-tools` / `--denied-tools` | Session | |
| `command` | `/permissions` command | Session | |
| `session` | Règles runtime in-memory | Session | Basse |

### Matching des noms d'outils

| Pattern | Exemple | Matching |
|---------|---------|---------|
| Nom direct | `Bash` | Outil Bash uniquement |
| Serveur MCP | `mcp__slack` | Tous les outils du serveur slack |
| Wildcard MCP | `mcp__slack__*` | Tous les outils du serveur slack |
| Contenu spécifique | `Bash(python:*)` | Bash avec commandes python |
| Chemin spécifique | `FileEdit(/.claude/*)` | Édition dans .claude/ |

---

## 4. Pipeline de décision

### Fonction principale : `hasPermissionsToUseTool()`

**Fichier** : `utils/permissions/permissions.ts`

```
ÉTAPE 1 — VÉRIFICATIONS BASÉES SUR LES RÈGLES (bypass-immune)

  1a. Outil entièrement refusé ?
      → Vérifier alwaysDenyRules pour le tool match
      → Si oui : DENY

  1b. Outil a une règle ask ?
      → Vérifier alwaysAskRules
      → Exception : Bash sandboxé avec autoAllowBashIfSandboxed
      → Si oui : ASK

  1c. Vérification spécifique à l'outil
      → tool.checkPermissions(input, context)
      → Chaque outil (Bash, FileEdit, etc.) a sa propre logique

  1d. Outil refusé par l'outil ?
      → Si behavior: 'deny' retourné : DENY

  1e. Interaction utilisateur requise ?
      → Si requiresUserInteraction() ET behavior: 'ask' : ASK

  1f. Règle ask sur le contenu spécifique ?
      → Si decisionReason: {type: 'rule', ruleBehavior: 'ask'} : ASK

  1g. Vérifications de sécurité (bypass-immune)
      → Chemins sensibles (.git/, .claude/, shell configs)
      → Si safetyCheck retourné : ASK

ÉTAPE 2 — VÉRIFICATIONS BASÉES SUR LE MODE

  2a. Mode bypass ?
      → Si bypassPermissions ou plan+bypass disponible : ALLOW

  2b. Règle allow sur l'outil entier ?
      → Vérifier alwaysAllowRules : ALLOW

ÉTAPE 3 — CONVERSION passthrough → ask

POST-TRAITEMENT

  ├── Reset du compteur de refus si allow (auto mode)
  ├── Transformation dontAsk : ask → DENY
  ├── Auto mode classifier (si activé)
  │   ├── Fast path : acceptEdits autorise ?
  │   ├── Safe tool allowlist ?
  │   ├── YOLO classifier (inférence ML)
  │   └── Gestion limite de refus (fallback prompting)
  └── Agent asynchrone : hooks ou auto-deny
```

---

## 5. Classifieur auto-mode

### Gate : `feature('TRANSCRIPT_CLASSIFIER')` (Ant-only)

### Architecture à 2 étages

```
Étage 1 (rapide) : oui/non en max_tokens=64
    │
    ├── Non bloqué → ALLOW
    │
    └── Bloqué → Étage 2
                    │
                    Étage 2 (réflexion) : chain-of-thought
                    │
                    ├── Non bloqué → ALLOW
                    └── Bloqué → DENY (avec raison)
```

### Entrée du classifieur

```
System Prompt :
├── Règles de base (auto_mode_system_prompt.txt)
├── Permissions template (externe ou Anthropic)
├── Règles utilisateur :
│   ├── allow (settings.autoMode.allow)
│   ├── soft_deny (settings.autoMode.soft_deny)
│   └── environment (settings.autoMode.environment)
└── Règles Bash/PowerShell spécifiques

Transcript :
├── Messages user (texte uniquement)
├── Blocs tool_use assistant (pas le texte — prévient la manipulation)
├── Commandes en queue
└── Projection toAutoClassifierInput() par outil

Action courante :
└── Sérialisation de l'outil et de son input
```

### Outils sûrs (skip le classifieur)

Outils read-only ou sans effet de bord qui ne passent jamais par le classifieur :

```
FileRead, Grep, Glob, LSP, ToolSearch,
TaskCreate, TaskUpdate, TaskList, TaskStop,
AskUserQuestion, EnterPlanMode, ExitPlanMode,
TeamCreate, TeamDelete, SendMessage,
Sleep, Workflow
```

### Indisponibilité du classifieur

Quand l'API du classifieur échoue :
- `tengu_iron_gate_closed = true` → **DENY** (fail closed) avec guidance retry
- `tengu_iron_gate_closed = false` → **fallback** vers le prompting normal (fail open)

---

## 6. Détection de patterns dangereux

### Commandes Bash dangereuses

**Exécuteurs de code** (cross-platform) :
```
python, python3, node, deno, tsx, ruby, perl, php, lua
npx, bunx, npm run, yarn run, pnpm run, bun run
bash, sh, ssh
```

**Unix spécifique** :
```
zsh, fish, eval, exec, env, xargs, sudo
```

**Ant-only** (outils à haut risque) :
```
fa run, coo, gh, gh api, curl, wget, git, kubectl, aws, gcloud, gsutil
```

### Détection (`isDangerousBashPermission()`)

Un allow rule Bash est dangereux si :
- Allow sur tout Bash : `Bash` ou `Bash(*)`
- Wildcard standalone : `Bash(*pattern*)`
- Préfixe : `Bash(python:*)`
- Suffixe wildcard : `Bash(python*)`
- Espace wildcard : `Bash(python *)`
- Flag wildcard : `Bash(python -*)`

### Patterns PowerShell dangereux

```
iex (Invoke-Expression), pwsh, powershell,
Start-Process, Add-Type, New-Object,
Register-ScheduledJob, Register-WmiEvent
```

### Entrée en auto mode

Quand l'utilisateur active le mode auto :
1. `findDangerousClassifierPermissions()` scanne les règles
2. Les règles wildcard Bash/PowerShell sont **strippées**
3. Les règles Agent allow sont **strippées**
4. Avertissements loggés

---

## 7. Handlers de permission

Trois implémentations selon le contexte d'exécution :

### Handler interactif (terminal principal)

```
1. Push ToolUseConfirm dans la queue UI avec callbacks
2. Lance hooks + classifieur en arrière-plan (async)
3. Course entre : hooks/classifieur vs interaction utilisateur
4. Garde resolve-once + flag userInteracted
5. Transition checkmark visuelle
```

### Handler coordinateur (workers)

```
1. Essaie les hooks PermissionRequest d'abord (rapide, local)
2. Essaie le classifieur (lent, inférence — Bash uniquement)
3. Fall through vers le dialogue interactif si les deux échouent
```

### Handler swarm worker (agents background)

```
1. Essaie le classifieur auto-approval en premier
2. Forward la requête au leader via mailbox
3. Enregistre des callbacks pour la réponse du leader
4. Affiche un indicateur "pending" en attendant
```

---

## 8. Chargement et persistance des règles

### Chargement initial

```typescript
loadAllPermissionRulesFromDisk()
├── policySettings (read-only, haute priorité)
├── userSettings (~/.claude/settings.json)
├── projectSettings (.claude/settings.json)
└── localSettings (.claude.local.json)
```

### Synchronisation en cours de session

```typescript
syncPermissionRulesFromDisk()
├── Vide toutes les sources disk avant rechargement
├── Respecte allowManagedPermissionRulesOnly (policy)
└── Garantit que les règles supprimées disparaissent
```

### Types de mise à jour

| Type | Action |
|------|--------|
| `addRules` | Ajouter aux règles existantes |
| `replaceRules` | Remplacer toutes les règles source:behavior |
| `removeRules` | Supprimer des règles spécifiques |
| `setMode` | Changer le mode de permission |
| `addDirectories` | Ajouter des répertoires de travail |
| `removeDirectories` | Retirer des répertoires |

---

## 9. Fichiers et chemins protégés

### Fichiers dangereux (permission requise)

```
.gitconfig, .gitmodules, .bashrc, .bash_profile, .zshrc,
.zprofile, .profile, .ripgreprc, .mcp.json, .claude.json
```

### Répertoires dangereux

```
.git/, .vscode/, .idea/, .claude/
```

### Patterns Windows dangereux (toutes plateformes)

| Pattern | Risque |
|---------|--------|
| NTFS Alternate Data Streams | Accès à des flux cachés |
| Noms courts 8.3 (`~\d`) | Contournement de matching |
| Préfixes longs (`\\?\`, `//?/`) | Bypass de sécurité |
| Points/espaces finaux | Normalisation plateforme |
| Noms device DOS (`.CON`, `.PRN`) | Accès devices système |

### Normalisation

Toutes les vérifications utilisent `normalizeCaseForComparison()` (minuscules) pour empêcher les contournements comme `.cLauDe/Settings.json`.

---

## 10. Suivi des refus

### Limites du mode auto

| Limite | Valeur | Action |
|--------|--------|--------|
| Refus consécutifs | 3 | Fallback vers prompting manuel |
| Refus total session | 20 | Fallback vers prompting + reset state |

### En mode headless

Si la limite est atteinte : `AbortError` (arrêt de l'agent).

### Analytics

Événement `tengu_auto_mode_denial_limit_exceeded` loggé avec compteurs.

---

## 11. Types de décision

### Structure PermissionDecision

```typescript
// ALLOW
{
  behavior: 'allow',
  updatedInput?: Input,           // Input potentiellement modifié
  userModified?: boolean,         // L'utilisateur a modifié l'input
  decisionReason?: PermissionDecisionReason,
  acceptFeedback?: string,        // Message de feedback
}

// ASK
{
  behavior: 'ask',
  message: string,                // Question à poser
  updatedInput?: Input,
  suggestions?: PermissionUpdate[], // Règles suggérées
  blockedPath?: string,           // Chemin bloqué
  pendingClassifierCheck?: {...},  // Classifieur en attente
}

// DENY
{
  behavior: 'deny',
  message: string,                // Raison du refus
  decisionReason: PermissionDecisionReason,
}
```

### Raisons de décision

| Type | Description |
|------|-------------|
| `rule` | Règle explicite (allow/deny/ask) |
| `mode` | Mode de permission actif |
| `classifier` | Décision du classifieur ML |
| `safetyCheck` | Chemin/fichier protégé |
| `hook` | Décision d'un hook |
| `workingDir` | Hors du répertoire de travail |
| `sandboxOverride` | Override de sandbox |
| `asyncAgent` | Décision d'agent asynchrone |
| `subcommandResults` | Résultats de sous-commandes |
| `other` | Autre raison |

Le champ `classifierApprovable` indique si le classifieur auto-mode peut évaluer cette décision (`true` pour les chemins sensibles, `false` pour les tentatives de bypass Windows).

---

## 12. Fichiers clés

### Moteur de permissions

| Fichier | Rôle | Taille |
|---------|------|--------|
| `utils/permissions/permissions.ts` | Pipeline de décision principal | 52 KB |
| `utils/permissions/yoloClassifier.ts` | Classifieur ML auto-mode | 1500+ lignes |
| `utils/permissions/permissionSetup.ts` | Chargement des règles, détection patterns dangereux | |
| `utils/permissions/filesystem.ts` | Permissions fichier, chemins dangereux, symlinks | |
| `utils/permissions/pathValidation.ts` | Validation de chemins, expansion tilde, UNC | |
| `utils/permissions/denialTracking.ts` | Suivi des limites de refus | |
| `utils/permissions/autoModeState.ts` | État du mode auto | |
| `utils/permissions/classifierDecision.ts` | Allowlist d'outils sûrs | |
| `utils/permissions/dangerousPatterns.ts` | Listes de patterns | |

### Handlers

| Fichier | Rôle |
|---------|------|
| `hooks/useCanUseTool.tsx` | Hook React intégrateur |
| `hooks/toolPermission/handlers/interactiveHandler.ts` | Terminal interactif |
| `hooks/toolPermission/handlers/coordinatorHandler.ts` | Workers coordinateur |
| `hooks/toolPermission/handlers/swarmWorkerHandler.ts` | Agents swarm |

### Types et prompts

| Fichier | Rôle |
|---------|------|
| `types/permissions.ts` | Définitions de types |
| `utils/permissions/yolo-classifier-prompts/` | Prompts du classifieur ML |
