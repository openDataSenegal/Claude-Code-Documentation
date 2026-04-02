# Installation

## Prérequis

| Outil | Version minimale | Usage |
|-------|-----------------|-------|
| Node.js | 18+ | Runtime JavaScript |
| Bun | 1.x | Bundler et runtime |
| Git | 2.x | Gestion de version |
| Clé API Anthropic | — | Authentification API Claude |

## Installation officielle (package npm)

Claude Code est distribué via npm en tant que CLI :

```bash
# Installation globale
npm install -g @anthropic-ai/claude-code

# Vérification
claude --version
```

## Installation depuis les sources (ce dépôt)

> **Avertissement** : Ce dépôt contient le code source extrait d'un sourcemap. La compilation complète nécessite l'infrastructure de build interne d'Anthropic (macros Bun, feature flags compile-time, attestation client).

### Cloner le dépôt

```bash
git clone https://github.com/kuberwastaken/claude-code.git
cd claude-code
```

### Structure des fichiers d'entrée

```
entrypoints/
├── cli.tsx          # Point d'entrée CLI principal
├── init.ts          # Initialisation (config, telemetry, OAuth)
├── mcp.ts           # Serveur MCP
└── sdk/             # Schémas SDK
```

Le point d'entrée principal est `entrypoints/cli.tsx`, qui :
1. Vérifie la version de Node.js
2. Initialise la télémétrie et l'authentification
3. Lance le worker daemon si nécessaire
4. Démarre la boucle REPL interactive via `main.tsx`

### Build théorique

```bash
# Installer les dépendances
bun install

# Build avec le bundler Bun
bun build entrypoints/cli.tsx --outdir dist/

# Lancer
node dist/cli.js
```

> Les macros `MACRO.VERSION`, `MACRO.BUILD_DATE` et les feature flags `feature('FLAG')` sont résolus au compile-time par Bun. Sans la configuration de build originale, ces valeurs resteront non définies.

---

## Authentification

Claude Code supporte plusieurs méthodes d'authentification :

### 1. Clé API directe

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
claude
```

### 2. OAuth (Claude Pro/Teams)

```bash
claude login
# Ouvre le navigateur pour le flow OAuth
```

### 3. AWS Bedrock

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=us-east-1
claude
```

### 4. Google Vertex AI

```bash
export CLAUDE_CODE_USE_VERTEX=1
export CLOUD_ML_REGION=us-central1
claude
```

---

## Vérification de l'installation

```bash
# Vérifier la version
claude --version

# Lancer en mode interactif
claude

# Exécuter une commande unique
claude -p "Explique ce fichier" main.tsx
```

---

## Désinstallation

```bash
# Supprimer le package
npm uninstall -g @anthropic-ai/claude-code

# Supprimer les données utilisateur (optionnel)
rm -rf ~/.claude
```
