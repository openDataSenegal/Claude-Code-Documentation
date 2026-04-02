# FAQ

## Questions générales

### Qu'est-ce que Claude Code ?

Claude Code est l'outil CLI officiel d'Anthropic pour le développement assisté par IA. Il permet d'interagir avec les modèles Claude directement depuis le terminal pour lire, écrire, modifier du code, exécuter des commandes et gérer des projets.

### D'où vient ce code source ?

Le code source a été accidentellement exposé le 31 mars 2026 via un fichier sourcemap (`.map`) inclus dans le package npm publié. Le champ `sourcesContent` du sourcemap contenait l'intégralité du code TypeScript original.

### Peut-on compiler et exécuter ce code ?

**Non, pas directement.** Le code dépend de :
- Macros compile-time Bun (`MACRO.VERSION`, `MACRO.BUILD_DATE`)
- Feature flags (`feature('FLAG')`) résolus au build
- Infrastructure interne Anthropic (OAuth, télémétrie, attestation client)
- Clés API et endpoints internes

Le code est exploitable pour l'étude et la compréhension de l'architecture, mais pas pour la compilation autonome.

---

## Architecture

### Pourquoi `main.tsx` fait ~4 700 lignes ?

`main.tsx` est le point d'orchestration central de l'application. Il gère l'enregistrement des 87 commandes, la construction du contexte et la boucle de session. C'est un choix architectural : centraliser l'orchestration plutôt que la fragmenter. Les fonctionnalités métier sont bien séparées dans les dossiers `tools/`, `services/`, `commands/`.

### Pourquoi un renderer React personnalisé au lieu d'Ink ?

Claude Code a développé son propre renderer terminal (dans `ink/`) pour :
- Un contrôle total sur le rendu ANSI et le layout
- Des performances optimisées pour leur cas d'usage spécifique
- L'intégration native du moteur de layout Yoga (Flexbox)
- Des animations et transitions fluides

### Qu'est-ce que le système "Dream" ?

Le Dream system (`services/autoDream/`) est un processus de consolidation mémoire en arrière-plan. Il s'exécute automatiquement (>24h et >5 sessions depuis le dernier run) pour :
- Fusionner les informations redondantes dans `MEMORY.md`
- Supprimer les entrées obsolètes
- Maintenir la mémoire sous 200 lignes et 25KB
- Résoudre les contradictions entre entrées

### Qu'est-ce que KAIROS ?

KAIROS est le mode "assistant toujours actif" (feature flag interne). Contrairement au mode interactif standard où Claude attend une instruction, KAIROS observe, enregistre et agit de façon proactive. Ce mode est réservé aux employés Anthropic.

---

## Fonctionnalités

### Quels modèles sont supportés ?

| Modèle | ID | Contexte |
|--------|----|----------|
| Opus 4.6 | `claude-opus-4-6` | 1M tokens |
| Sonnet 4.6 | `claude-sonnet-4-6` | 1M tokens |
| Haiku 4.5 | `claude-haiku-4-5` | 200K tokens |

### Qu'est-ce que le "Fast Mode" ?

Le Fast Mode (nom interne : Penguin Mode) utilise le même modèle mais avec une inférence accélérée côté serveur. Il réduit la latence au prix d'une légère réduction de qualité sur les tâches très complexes. Activable via `/fast`.

### Comment fonctionne la compaction du contexte ?

Quand la conversation approche la limite de tokens, Claude Code compacte automatiquement :

1. **Résumé** : Les anciens messages sont résumés par le modèle
2. **Snipping** : Les résultats d'outils verbeux sont tronqués
3. **Micro-compaction** : Suppression sélective de contenu non essentiel
4. **Context collapse** : Pour les très grands contextes, réduction agressive

Vous pouvez aussi forcer la compaction manuellement avec `/compact`.

### Comment fonctionne le système de permissions ?

Chaque outil a un niveau de risque :
- **LOW** (Read, Glob, Grep) : Auto-approuvé
- **MEDIUM** (Edit, Write) : Dépend du mode
- **HIGH** (Bash, PowerShell) : Demande toujours confirmation en mode default

Fichiers protégés (`.gitconfig`, `.bashrc`, `.ssh/*`) nécessitent toujours une confirmation explicite.

### Qu'est-ce que MCP ?

Le Model Context Protocol est un standard ouvert pour connecter des outils tiers aux LLMs. Claude Code peut se connecter à des serveurs MCP pour accéder à des bases de données, APIs, services cloud, etc. sans modification du code de Claude Code.

---

## Configuration

### Où sont stockées les données de Claude Code ?

```
~/.claude/
├── config.json          # Configuration globale
├── settings.json        # Paramètres avancés
├── keybindings.json     # Raccourcis clavier
├── skills/              # Skills personnalisés
├── plugins/             # Plugins
├── projects/            # Données par projet
│   └── <hash>/
│       └── memory/      # Mémoire persistante
└── sessions/            # Historique des sessions
```

### Comment personnaliser le comportement de Claude pour un projet ?

Créez un fichier `CLAUDE.md` ou `.claude.md` à la racine du projet :

```markdown
# Mon Projet

## Stack
- Python 3.12, FastAPI, SQLAlchemy 2.0

## Conventions
- Utiliser des dataclasses, pas des dicts
- Nommer les tests test_<module>_<behavior>
- Pas de print(), utiliser logging

## Architecture
- Hexagonal : domain/ ne dépend de rien
- Ports dans domain/ports/, adapters dans infra/
```

### Comment ajouter un serveur MCP ?

Dans `.claude.json` (projet) ou `~/.claude/settings.json` (global) :

```json
{
  "mcpServers": {
    "mon-serveur": {
      "command": "node",
      "args": ["./mcp-server.js"],
      "env": {
        "API_KEY": "${MON_API_KEY}"
      }
    }
  }
}
```

---

## Dépannage

### Claude Code est lent au démarrage

Causes possibles :
1. **Serveurs MCP** : Chaque serveur doit démarrer et faire le handshake. Réduisez le nombre de serveurs MCP configurés.
2. **Grand `CLAUDE.md`** : Gardez-le concis (<500 lignes).
3. **Réseau** : La connexion à l'API est établie au démarrage pour la vérification de version.

### Le contexte est saturé trop vite

- Utilisez `/compact` régulièrement
- Évitez de lire des fichiers entiers quand seule une partie est nécessaire (utilisez `offset` et `limit`)
- Utilisez des agents pour les explorations — ils ont leur propre contexte
- Préférez des instructions ciblées à des demandes vagues

### Claude ne voit pas mes fichiers

- Vérifiez que vous êtes dans le bon répertoire de travail
- Les fichiers dans `.gitignore` sont accessibles mais Claude peut les ignorer par défaut
- Utilisez `/add-dir` pour ajouter des répertoires hors du CWD

### Les outils échouent avec une erreur de permission

En mode `default`, les outils à risque MEDIUM/HIGH demandent confirmation. Options :
- Accepter individuellement chaque demande
- Configurer des règles dans `settings.json` :

```json
{
  "permissions": {
    "allow": ["Bash(npm test)", "Bash(npm run build)"]
  }
}
```

### Comment réduire les coûts ?

1. **Utilisez Sonnet** au lieu d'Opus pour les tâches quotidiennes
2. **Activez Fast Mode** (`/fast`) pour les tâches simples
3. **Compactez régulièrement** pour réduire les tokens d'entrée
4. **Soyez précis** dans vos instructions (moins d'allers-retours)
5. **Utilisez Haiku** pour les tâches triviales (autocomplétion, formatage)

---

## Sécurité

### Claude Code peut-il exécuter des commandes dangereuses ?

Le système de permissions empêche l'exécution non autorisée de commandes. En mode `default` :
- Toute commande shell nécessite votre approbation
- Les fichiers sensibles sont protégés
- Les traversées de chemin sont bloquées

### Les données sont-elles envoyées à Anthropic ?

Oui, les messages de conversation sont envoyés à l'API Claude pour traitement. La télémétrie (usage, erreurs) est également collectée par défaut. Désactivable via :

```bash
export CLAUDE_CODE_DISABLE_TELEMETRY=1
```

### Le code source est-il exposé à l'API ?

Seuls les fichiers que Claude lit explicitement via les outils sont envoyés dans le contexte de l'API. Claude Code ne scanne pas et n'envoie pas l'intégralité du projet automatiquement.
