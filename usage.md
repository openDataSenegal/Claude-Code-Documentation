# Usage

## Démarrage rapide

### Mode interactif (REPL)

```bash
claude
```

Lance une session conversationnelle. Tapez vos instructions en langage naturel, Claude analyse le contexte du projet et agit.

### Mode commande unique

```bash
claude -p "Explique l'architecture de ce projet"
```

Exécute une instruction unique et retourne le résultat sans session interactive.

### Mode pipe

```bash
cat error.log | claude -p "Analyse ces erreurs et propose des corrections"
git diff | claude -p "Review ce diff"
```

---

## Commandes intégrées

Les commandes sont préfixées par `/` dans le mode interactif.

### Navigation et session

| Commande | Description |
|----------|-------------|
| `/help` | Aide générale |
| `/clear` | Efface la conversation |
| `/compact` | Compacte le contexte pour libérer des tokens |
| `/history` | Historique des sessions |
| `/resume` | Reprend la dernière session |
| `/quit` | Quitte Claude Code |

### Développement

| Commande | Description |
|----------|-------------|
| `/commit` | Crée un commit avec message généré |
| `/review` | Review de code (PR ou diff) |
| `/simplify` | Analyse le code modifié pour qualité et réutilisabilité |
| `/add-dir <path>` | Ajoute un répertoire au contexte |

### Configuration

| Commande | Description |
|----------|-------------|
| `/config` | Affiche/modifie la configuration |
| `/settings` | Paramètres avancés |
| `/theme` | Change le thème |
| `/fast` | Bascule le mode rapide (latence réduite) |
| `/model` | Change le modèle |

### Mémoire

| Commande | Description |
|----------|-------------|
| `/memory` | Gestion de la mémoire persistante |
| `/context` | Affiche le contexte chargé |

### Agents et tâches

| Commande | Description |
|----------|-------------|
| `/plan` | Entre en mode planification |
| `/schedule` | Gère les agents planifiés (cron) |
| `/loop <interval> <cmd>` | Exécute une commande en boucle |

---

## Outils disponibles

Claude utilise les outils de façon autonome en fonction de vos instructions. Voici les principaux :

### Lecture de fichiers

```
Utilisateur : "Lis le fichier main.tsx"
→ Claude utilise Read pour lire le fichier et l'afficher
```

- **Read** : Lecture de fichiers (texte, images, PDF, notebooks)
- **Glob** : Recherche de fichiers par pattern (`**/*.ts`)
- **Grep** : Recherche de contenu dans les fichiers (regex)

### Modification de fichiers

```
Utilisateur : "Renomme la variable userId en user_id dans auth.ts"
→ Claude utilise Edit pour effectuer le remplacement
```

- **Edit** : Remplacement de texte dans un fichier existant
- **Write** : Création d'un nouveau fichier
- **NotebookEdit** : Modification de notebooks Jupyter

### Exécution de commandes

```
Utilisateur : "Lance les tests"
→ Claude utilise Bash pour exécuter la commande de test
```

- **Bash** : Exécution de commandes shell (Linux/macOS)
- **PowerShell** : Exécution de commandes (Windows)

### Web

```
Utilisateur : "Vérifie la doc de l'API Stripe pour les webhooks"
→ Claude utilise WebSearch puis WebFetch
```

- **WebSearch** : Recherche web
- **WebFetch** : Récupération du contenu d'une URL

### Agents

```
Utilisateur : "Explore le codebase pour comprendre comment fonctionne l'auth"
→ Claude spawne un agent Explorer
```

- **Agent** : Spawne un sous-agent spécialisé pour des tâches complexes

Types d'agents disponibles :
| Type | Usage |
|------|-------|
| `general-purpose` | Recherche et tâches multi-étapes |
| `Explore` | Exploration rapide du codebase |
| `Plan` | Planification d'implémentation |
| `claude-code-guide` | Questions sur Claude Code |

### Tâches en arrière-plan

```
Utilisateur : "Lance les tests en arrière-plan pendant que je continue"
→ Claude crée une tâche background
```

- **TaskCreate** : Crée une tâche en arrière-plan
- **TaskGet** / **TaskList** : Consulte l'état des tâches
- **TaskUpdate** : Met à jour une tâche
- **TaskStop** : Arrête une tâche

### Git Worktrees

```
Utilisateur : "Travaille sur ce refactoring dans un worktree isolé"
→ Claude crée un worktree Git temporaire
```

- **EnterWorktree** : Crée un worktree Git isolé
- **ExitWorktree** : Sort du worktree et propose de merger

---

## Workflows courants

### 1. Corriger un bug

```
> Il y a un bug dans le parsing des dates quand le timezone est UTC+0.
> Le test test_parse_date_utc échoue.
```

Claude va :
1. Lire les tests pour comprendre le comportement attendu
2. Localiser le code de parsing
3. Identifier le bug
4. Proposer et appliquer le fix
5. Relancer les tests pour vérifier

### 2. Ajouter une feature

```
> Ajoute un endpoint GET /api/users/:id/activity qui retourne
> les 50 dernières actions de l'utilisateur, paginé.
```

Claude va :
1. Explorer les endpoints existants pour comprendre les conventions
2. Vérifier le modèle de données
3. Implémenter le endpoint
4. Ajouter la validation d'entrée
5. Suggérer les tests à écrire

### 3. Code review

```bash
# Via commande slash
/review

# Ou via instruction
> Review la PR #42 sur ce repo
```

### 4. Refactoring

```
> Refactore le module auth pour utiliser le pattern Repository
> au lieu d'accéder directement à la base de données.
```

Claude va :
1. Analyser le code actuel
2. Entrer en mode planification pour aligner sur l'approche
3. Implémenter le refactoring de façon incrémentale
4. Vérifier que les tests passent toujours

### 5. Commit et PR

```
> Commit les changements et crée une PR
```

Claude va :
1. Analyser les diffs
2. Rédiger un message de commit descriptif
3. Créer la branche si nécessaire
4. Pousser et créer la PR via `gh`

---

## Mode Vim

Claude Code intègre un mode Vim complet pour l'édition dans le prompt :

```bash
# Activer le mode Vim
claude config set vim true
```

Supporte :
- Modes normal, insertion, visuel
- Motions (`w`, `b`, `e`, `0`, `$`, `gg`, `G`)
- Opérateurs (`d`, `c`, `y`, `p`)
- Text objects (`iw`, `aw`, `i"`, `a(`)

---

## Raccourcis clavier

| Raccourci | Action |
|-----------|--------|
| `Enter` | Envoyer le message |
| `Ctrl+C` | Annuler l'opération en cours |
| `Ctrl+D` | Quitter |
| `Tab` | Autocomplétion |
| `↑` / `↓` | Navigation dans l'historique |
| `Escape` | Annuler la saisie en cours |

Personnalisables via `~/.claude/keybindings.json`.

---

## Bonnes pratiques

1. **Utilisez `CLAUDE.md`** pour donner du contexte projet — Claude le charge automatiquement
2. **Soyez spécifique** dans vos instructions — "corrige le bug de parsing dans `date.ts` ligne 42" > "corrige le bug"
3. **Utilisez `/compact`** quand la conversation devient longue
4. **Exploitez les agents** pour les tâches d'exploration complexes
5. **Utilisez le mode plan** (`/plan`) avant les gros refactorings
6. **Laissez Claude lire avant d'écrire** — il produit de meilleur code quand il comprend le contexte existant
