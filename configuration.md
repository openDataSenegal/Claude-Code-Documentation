# Configuration

Claude Code utilise un système de configuration à plusieurs niveaux : fichiers locaux au projet, fichiers globaux utilisateur, variables d'environnement et feature flags compile-time.

---

## Fichiers de configuration

### Hiérarchie (du plus spécifique au plus général)

```
Projet (priorité haute)
├── .claude.json              # Configuration projet
├── .claude.md / CLAUDE.md    # Contexte et instructions projet
│
Utilisateur (priorité moyenne)
├── ~/.claude/config.json     # Configuration globale
├── ~/.claude/settings.json   # Paramètres avancés
├── ~/.claude/keybindings.json # Raccourcis clavier
├── ~/.claude/skills/         # Skills personnalisés
│
Système (priorité basse)
└── Valeurs par défaut intégrées
```

### `.claude.md` / `CLAUDE.md`

Fichier de contexte projet, chargé automatiquement dans le prompt système. Utilisé pour :

- Décrire le stack technique
- Définir les conventions de code
- Spécifier les patterns requis ou interdits
- Fournir des instructions spécifiques au dépôt

```markdown
# CLAUDE.md

## Stack
- TypeScript 5.x, React 19, PostgreSQL 16
- Tests: Vitest + Testing Library

## Conventions
- Pas de `any` TypeScript
- Tous les composants en functional components
- Tests requis pour toute nouvelle feature
```

### `~/.claude/settings.json`

Configuration avancée avec hooks, permissions et outils personnalisés :

```json
{
  "permissions": {
    "allow": ["Read", "Glob", "Grep"],
    "deny": ["Bash(rm *)"]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "echo 'Bash tool invoked'" }
        ]
      }
    ]
  },
  "env": {
    "ANTHROPIC_MODEL": "claude-sonnet-4-6"
  }
}
```

### `~/.claude/keybindings.json`

```json
[
  { "key": "ctrl+s", "command": "submit" },
  { "key": "ctrl+k ctrl+c", "command": "clear" }
]
```

---

## Variables d'environnement

### Authentification

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | Clé API Anthropic |
| `CLAUDE_CODE_USE_BEDROCK` | Utiliser AWS Bedrock comme backend |
| `CLAUDE_CODE_USE_VERTEX` | Utiliser Google Vertex AI comme backend |
| `AWS_REGION` | Région AWS pour Bedrock |
| `CLOUD_ML_REGION` | Région GCP pour Vertex |

### Modèle et comportement

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_MODEL` | Modèle par défaut (`claude-opus-4-6`, `claude-sonnet-4-6`, `claude-haiku-4-5`) |
| `ANTHROPIC_BASE_URL` | URL de base de l'API (défaut : `https://api.anthropic.com`) |
| `CLAUDE_CODE_MAX_TOKENS` | Limite de tokens par requête |

---

## Feature Flags (compile-time)

Les feature flags contrôlent l'activation de fonctionnalités via le système `feature('FLAG')` de Bun. Ils sont résolus au compile-time et permettent le dead-code elimination.

### Flags publics

| Flag | Description |
|------|-------------|
| `VOICE_MODE` | Support de la saisie vocale (STT) |
| `WORKFLOW_SCRIPTS` | Automatisation via scripts YAML |

### Flags internes (Anthropic uniquement)

| Flag | Description |
|------|-------------|
| `KAIROS` / `PROACTIVE` | Mode assistant toujours actif |
| `KAIROS_BRIEF` | Mode sortie concise pour KAIROS |
| `BRIDGE_MODE` | Contrôle distant via claude.ai |
| `DAEMON` | Mode daemon en arrière-plan |
| `COORDINATOR_MODE` | Orchestration multi-agents |
| `BUDDY` | Compagnon Tamagotchi terminal |
| `TRANSCRIPT_CLASSIFIER` | Mode AFK (approbation ML automatique) |
| `CHICAGO_MCP` | Computer Use (screenshot/click) |
| `ULTRAPLAN` | Sessions de planification de 30 min |
| `NATIVE_CLIENT_ATTESTATION` | Vérification de l'attestation client |
| `HISTORY_SNIP` | Compaction de l'historique |
| `ABLATION_BASELINE` | Tests A/B baseline |

### Vérification d'un flag dans le code

```typescript
import { feature } from 'bun:bundle';

if (feature('BUDDY')) {
  // Code inclus uniquement si le flag est activé au build
  const buddy = await import('./buddy/companion.js');
}
```

---

## Modes de permission

Claude Code offre plusieurs modes de contrôle des permissions d'outils :

| Mode | Comportement |
|------|-------------|
| `default` | Interactif — demande confirmation pour les outils à risque |
| `auto` | Approbation automatique basée sur le niveau de risque |
| `bypass` | Autorise tous les outils sans confirmation |
| `yolo` | Refuse toutes les opérations (mode lecture seule) |

### Classification des risques

```
LOW    → Read, Glob, Grep (lecture seule)
MEDIUM → Edit, Write, NotebookEdit (modification de fichiers)
HIGH   → Bash, PowerShell (exécution de commandes)
```

---

## Sélection du modèle

```bash
# Via variable d'environnement
export ANTHROPIC_MODEL=claude-sonnet-4-6

# Via argument CLI
claude --model claude-opus-4-6

# Basculer en mode rapide (même modèle, inférence accélérée)
# Dans une session : taper /fast
```

### Modèles disponibles

| Modèle | ID | Usage recommandé |
|--------|----|-----------------|
| Opus 4.6 | `claude-opus-4-6` | Tâches complexes, architecture |
| Sonnet 4.6 | `claude-sonnet-4-6` | Usage quotidien, bon équilibre |
| Haiku 4.5 | `claude-haiku-4-5` | Tâches simples, faible latence |

---

## Thèmes

```bash
# Changer de thème
claude config set theme dark   # dark, light, ou custom
```

---

## Plugins

Les plugins étendent les capacités de Claude Code :

```
~/.claude/plugins/
└── my-plugin/
    ├── manifest.json    # Métadonnées et configuration
    └── index.ts         # Point d'entrée
```

## Skills

Les skills sont des prompts réutilisables invocables via `/nom-du-skill` :

```
~/.claude/skills/
└── review.md            # Invocable via /review
```

```markdown
---
name: review
description: Code review approfondie
---

Effectue une code review du fichier suivant en vérifiant :
- Clarté et lisibilité
- Gestion des erreurs
- Performance
- Sécurité
```
