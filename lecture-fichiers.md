# Système de Lecture de Fichiers — Documentation complète

Ce document décrit en profondeur le système de lecture de fichiers de Claude Code : le FileReadTool, les formats supportés, les pipelines de traitement, le caching, les permissions, la sécurité et la restauration post-compaction.

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Formats supportés et pipelines](#2-formats-supportés-et-pipelines)
3. [Lecture de fichiers texte](#3-lecture-de-fichiers-texte)
4. [Lecture d'images](#4-lecture-dimages)
5. [Lecture de PDF](#5-lecture-de-pdf)
6. [Lecture de notebooks Jupyter](#6-lecture-de-notebooks-jupyter)
7. [Limites et contraintes](#7-limites-et-contraintes)
8. [Formatage de la sortie](#8-formatage-de-la-sortie)
9. [Détection d'encodage](#9-détection-dencodage)
10. [Caching et déduplication](#10-caching-et-déduplication)
11. [Permissions et contrôle d'accès](#11-permissions-et-contrôle-daccès)
12. [Sécurité — Protections et validations](#12-sécurité)
13. [Restauration post-compaction](#13-restauration-post-compaction)
14. [Cas spéciaux](#14-cas-spéciaux)
15. [Analytics et logging](#15-analytics-et-logging)
16. [Fichiers clés](#16-fichiers-clés)

---

## 1. Vue d'ensemble

Le `FileReadTool` (nom enregistré : `Read`) est l'outil principal de lecture de fichiers. Il gère cinq formats distincts via un pipeline unifié avec validation, permissions, caching et formatage.

### Caractéristiques

| Propriété | Valeur |
|-----------|--------|
| **Nom** | `Read` |
| **Fichier** | `tools/FileReadTool/FileReadTool.ts` |
| **Read-only** | Oui |
| **Concurrent-safe** | Oui |
| **Formats** | Texte, Images, PDF, Notebooks Jupyter, Pages PDF extraites |

### Schéma d'entrée

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `file_path` | string | oui | Chemin absolu du fichier |
| `offset` | number | non | Ligne de départ (1-indexé) |
| `limit` | number | non | Nombre de lignes à lire |
| `pages` | string | non | Plage de pages PDF (ex: `"1-5"`, `"3"`, `"10-"`) |

### Pipeline général

```
file_path
    │
    ▼
1. VALIDATION
   ├── Chemin absolu requis
   ├── Blocage fichiers device (/dev/zero, /dev/stdin, etc.)
   ├── Validation pages PDF (max 20)
   └── Résolution macOS screenshot paths
    │
    ▼
2. PERMISSIONS
   ├── checkReadPermissionForTool()
   ├── Résolution complète de la chaîne de symlinks
   └── Vérification contre deny/allow rules + working directories
    │
    ▼
3. DÉTECTION DE TYPE
   ├── .png, .jpg, .jpeg, .gif, .webp → Image
   ├── .pdf → PDF
   ├── .ipynb → Notebook
   └── Tout le reste → Texte
    │
    ▼
4. LECTURE SPÉCIALISÉE
   ├── Texte : readFileInRange() avec offset/limit
   ├── Image : readImageWithTokenBudget()
   ├── PDF : readPDF() ou extractPDFPages()
   └── Notebook : readNotebook()
    │
    ▼
5. POST-TRAITEMENT
   ├── Déduplication (mtime check)
   ├── Mise à jour du fileStateCache
   ├── Notification des listeners
   ├── Découverte dynamique de skills
   └── Analytics
    │
    ▼
6. FORMATAGE DE SORTIE
   └── mapToolResultToToolResultBlockParam()
```

---

## 2. Formats supportés et pipelines

| Format | Extensions | Offset/Limit | Compression | Sortie |
|--------|-----------|-------------|-------------|--------|
| **Texte** | `*` (par défaut) | Oui | Non | Lignes numérotées |
| **Image** | `.png`, `.jpg`, `.jpeg`, `.gif`, `.webp` | Non | Token-based | Base64 |
| **PDF natif** | `.pdf` | Non | Non | Base64 du PDF complet |
| **PDF pages** | `.pdf` (avec `pages`) | Pages | JPEG 100 DPI | Base64 par page |
| **Notebook** | `.ipynb` | Non | Troncature outputs | Cellules structurées |

---

## 3. Lecture de fichiers texte

### Deux chemins d'exécution

Le système choisit dynamiquement entre deux stratégies selon la taille du fichier :

#### Chemin rapide (< 10 MB)

```
readFile() → buffer complet → split en mémoire → extraction de la plage
```

- Charge le fichier entier avec `readFile()`
- Split en lignes, accès O(1) à n'importe quelle plage
- ~2x plus rapide pour les fichiers typiques

#### Chemin streaming (≥ 10 MB, pipes, devices)

```
createReadStream() → scan ligne par ligne → accumulation de la plage demandée
```

- Utilise `createReadStream()` avec scan manuel
- N'accumule que les lignes dans la plage demandée
- Compte les lignes hors plage mais les discard
- Prévient les OOM sur les fichiers volumineux

### Fichier clé : `utils/readFileInRange.ts`

```typescript
readFileInRange(
  filePath: string,
  offset: number,        // Ligne de départ (0-indexé interne)
  maxLines?: number,     // undefined = jusqu'à EOF
  maxBytes?: number,     // Limite en octets
  signal?: AbortSignal,
  options?: { truncateOnByteLimit?: boolean }
)
→ {
  content: string,
  lineCount: number,     // Lignes retournées
  totalLines: number,    // Total de lignes dans le fichier
  totalBytes: number,    // Taille totale du fichier
  readBytes: number,     // Octets effectivement lus
  mtimeMs: number,       // Timestamp de modification
  truncatedByBytes?: boolean
}
```

### Normalisation

| Opération | Détail |
|-----------|--------|
| **BOM UTF-8** | Détecté et supprimé (U+FEFF) |
| **CRLF → LF** | `\r\n` converti en `\n` |
| **\r isolés** | Supprimés |

---

## 4. Lecture d'images

### Pipeline de compression

```
1. Lecture du buffer fichier
2. Détection format par magic bytes (fallback sur extension)
3. Resize standard (max 12000×12000 px)
4. Vérification budget tokens
    │
    ├── OK → retour base64
    │
    └── Dépassement → compression agressive
         ├── PNG : palette + réduction couleurs
         ├── JPEG : qualité progressive (80 → 60 → 40 → 20)
         └── Dernier recours : 400×400 px, JPEG qualité 20
              │
              ├── OK → retour base64
              └── Échec → ImageResizeError
```

### Fichier clé : `utils/imageResizer.ts`

**Fonctions principales** :
- `readImageWithTokenBudget()` — Orchestrateur principal
- `detectImageFormatFromBuffer()` — Détection PNG/JPEG/GIF/WebP par magic bytes
- `maybeResizeAndDownsampleImageBuffer()` — Resize avec contraintes de dimensions
- `compressImageBufferWithTokenLimit()` — Compression itérative jusqu'au budget

### Dimensions retournées

```typescript
{
  originalWidth: number,   // Dimensions originales
  originalHeight: number,
  displayWidth: number,    // Dimensions après resize
  displayHeight: number
}
```

### Limites images

| Limite | Valeur |
|--------|--------|
| Base64 max | 5 MB (`API_IMAGE_MAX_BASE64_SIZE`) |
| Pixels max | 12000 × 12000 |
| Fallback | 400 × 400, JPEG qualité 20 |

### Classification des erreurs

| Type | Description |
|------|-------------|
| `MODULE_LOAD` | Échec chargement sharp/napi |
| `PROCESSING` | Erreur de traitement image |
| `PIXEL_LIMIT` | Image trop grande en pixels |
| `MEMORY` | Dépassement mémoire |
| `TIMEOUT` | Timeout de traitement |
| `VIPS` | Erreur spécifique libvips |
| `PERMISSION` | Accès fichier refusé |

---

## 5. Lecture de PDF

### Deux modes de lecture

#### Mode A — PDF natif (complet)

Envoie le PDF entier en base64 dans un bloc document supplémentaire.

**Conditions** :
- Modèle compatible (Sonnet 3.5 v2+, pas Haiku 3)
- Fichier < 20 MB (`PDF_TARGET_RAW_SIZE`)
- Pas de paramètre `pages`
- Magic bytes `%PDF-` présents

**Fichier** : `utils/pdf.ts` → `readPDF()`

#### Mode B — Extraction de pages (JPEG)

Extrait les pages spécifiées en images JPEG via `pdftoppm` (poppler-utils).

**Conditions de déclenchement** :
- Paramètre `pages` explicite
- Modèle ne supportant pas les PDF natifs (Haiku 3)
- Fichier > 25 MB (`PDF_EXTRACT_SIZE_THRESHOLD`)

**Processus** :
```
1. Valider la plage de pages (max 20)
2. Créer répertoire temporaire
3. pdftoppm -jpeg -r 100 -f {first} -l {last} input.pdf output/page
4. Retourner page-01.jpg, page-02.jpg, ...
```

**Fichier** : `utils/pdf.ts` → `extractPDFPages()`

### Utilitaires PDF

| Fonction | Fichier | Description |
|----------|---------|-------------|
| `parsePDFPageRange()` | `utils/pdfUtils.ts` | Parse `"1-5"`, `"3"`, `"10-"` en `{firstPage, lastPage}` |
| `isPDFSupported()` | `utils/pdfUtils.ts` | Vérifie compatibilité modèle |
| `getPDFPageCount()` | `utils/pdf.ts` | Compte les pages via `pdfinfo` |
| `isPdftoppmAvailable()` | `utils/pdf.ts` | Vérifie l'installation de poppler-utils (caché) |

### Erreurs PDF

| Type | Description |
|------|-------------|
| `empty` | Fichier PDF vide (0 octets) |
| `too_large` | Dépasse les limites de taille |
| `password_protected` | PDF chiffré |
| `corrupted` | PDF invalide ou magic bytes `%PDF-` absents |
| `unavailable` | poppler-utils non installé |
| `unknown` | Autre erreur |

---

## 6. Lecture de notebooks Jupyter

### Fichier clé : `utils/notebook.ts`

### Pipeline

```
1. Lecture du .ipynb en UTF-8 JSON
2. Validation de la taille
3. Pour chaque cellule :
   ├── Combiner les tableaux source en string unique
   ├── Traiter les outputs :
   │   ├── stream → extraction texte
   │   ├── execute_result / display_data → texte + images (PNG/JPEG)
   │   └── error → ename + evalue + traceback
   ├── Tronquer les outputs volumineux
   └── Compresser les images larges
4. Fusionner les blocs texte adjacents
5. Retourner NotebookCellSource[]
```

### Gestion des outputs volumineux

| Seuil | Comportement |
|-------|-------------|
| < 10 KB | Output inclus intégralement |
| ≥ 10 KB | Tronqué avec instructions `jq` pour accès spécifique |

### Structure de sortie

```typescript
NotebookCellSource {
  cellId: string,
  cellType: 'code' | 'markdown',
  source: string,
  outputs?: Array<{
    type: 'text' | 'image',
    content: string   // texte ou base64
  }>
}
```

---

## 7. Limites et contraintes

### Double système de limites

La lecture de fichiers texte applique **deux limites indépendantes** :

| Limite | Défaut | Vérification | Coût | Action |
|--------|--------|-------------|------|--------|
| `maxSizeBytes` | 256 KB | Pré-lecture (`stat()`) | 1 syscall | `FileTooLargeError` |
| `maxTokens` | 25 000 | Post-lecture (API) | 1 round-trip API | `MaxFileReadTokenExceededError` |

### Précédence de configuration

```
1. Variable d'environnement : CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS
2. GrowthBook experiment : tengu_amber_wren
3. Override par ToolUseContext : context.fileReadingLimits
4. Défaut hardcodé : DEFAULT_MAX_OUTPUT_TOKENS = 25000
```

### Fichier clé : `tools/FileReadTool/limits.ts`

```typescript
FileReadingLimits {
  maxTokens: number,
  maxSizeBytes: number,
  includeMaxSizeInPrompt?: boolean,
  targetedRangeNudge?: boolean
}
```

### Limites par format

| Format | Limite | Valeur |
|--------|--------|--------|
| Texte | Tokens max | 25 000 |
| Texte | Taille fichier max | 256 KB |
| Image | Base64 max | 5 MB |
| Image | Pixels max | 12000 × 12000 |
| PDF natif | Taille max | ~20 MB |
| PDF pages | Pages max | 20 par requête |
| Notebook | Seuil output | 10 KB avant troncature |

---

## 8. Formatage de la sortie

### Numéros de ligne (fichiers texte)

Deux formats selon le feature gate `tengu_compact_line_prefix_killswitch` :

#### Format compact (par défaut)

```
1	line content here
2	next line
999	line 999 content
```

Tab comme séparateur. Économise ~9 octets/ligne.

#### Format étendu (legacy)

```
     1→line content here
     2→next line
   999→line 999 content
```

Padding à 6 caractères avec flèche séparatrice.

### Fonctions

| Fonction | Fichier | Description |
|----------|---------|-------------|
| `addLineNumbers()` | `utils/file.ts` | Ajoute les préfixes de ligne |
| `stripLineNumberPrefix()` | `utils/file.ts` | Supprime les préfixes (les deux formats) |
| `isCompactLinePrefixEnabled()` | `utils/file.ts` | Vérifie le feature gate |

### Schéma de sortie par type

#### Texte

```typescript
{
  type: 'text',
  file: {
    filePath: string,
    content: string,       // Vide si fichier plus court que offset
    numLines: number,      // Lignes retournées
    startLine: number,     // Ligne de départ (offset)
    totalLines: number     // Total de lignes dans le fichier
  }
}
```

#### Image

```typescript
{
  type: 'image',
  file: {
    base64: string,
    type: 'image/jpeg' | 'image/png' | 'image/gif' | 'image/webp',
    originalSize: number,
    dimensions?: {
      originalWidth: number,
      originalHeight: number,
      displayWidth: number,
      displayHeight: number
    }
  }
}
```

#### PDF natif

```typescript
{
  type: 'pdf',
  file: {
    filePath: string,
    base64: string,
    originalSize: number
  }
}
```

#### Pages PDF extraites

```typescript
{
  type: 'parts',
  file: {
    filePath: string,
    originalSize: number,
    count: number,        // Nombre de pages extraites
    outputDir: string     // Répertoire temporaire avec page-NN.jpg
  }
}
```

#### Notebook

```typescript
{
  type: 'notebook',
  file: {
    filePath: string,
    cells: NotebookCellSource[]
  }
}
```

#### Fichier inchangé (dédup)

```typescript
{
  type: 'file_unchanged',
  file: { filePath: string }
}
```

---

## 9. Détection d'encodage

### Fichier clé : `utils/fileRead.ts`

### Stratégie de détection

```
1. Lire les 4096 premiers octets
2. Vérifier les BOM (Byte Order Mark) :
   ├── 0xFF 0xFE → UTF-16LE
   ├── 0xEF 0xBB 0xBF → UTF-8 (BOM)
   └── Absent → UTF-8 (défaut)
3. Heuristiques supplémentaires si BOM absent
4. Retour : encodage détecté
```

### Fonctions

| Fonction | Description |
|----------|-------------|
| `detectEncodingForResolvedPath()` | Détection d'encodage par lecture des premiers octets |
| `detectLineEndingsForString()` | Détection CRLF vs LF par scan du contenu |
| `readFileSyncWithMetadata()` | Lecture + encodage + line endings en un seul passage |

### Normalisation des fins de ligne

| Entrée | Sortie | Action |
|--------|--------|--------|
| `\r\n` (CRLF) | `\n` (LF) | Conversion automatique |
| `\r` isolé | supprimé | Nettoyage |
| `\n` (LF) | `\n` (LF) | Inchangé |

Le type de fin de ligne original (`LineEndingType: 'LF' | 'CRLF'`) est préservé dans les métadonnées pour la réécriture fidèle lors des éditions.

---

## 10. Caching et déduplication

### Trois couches de cache

#### A. FileStateCache — Suivi d'état des fichiers lus

**Fichier** : `utils/fileStateCache.ts`

Cache LRU qui suit les fichiers récemment lus pour la déduplication et la restauration post-compaction.

```typescript
FileState {
  content: string,         // Contenu brut du disque
  timestamp: number,       // mtime du fichier
  offset?: number,         // Ligne de départ (undefined = injection auto)
  limit?: number,          // Nombre de lignes
  isPartialView?: boolean  // True si le contenu est tronqué (ex: MEMORY.md injecté)
}
```

| Propriété | Valeur |
|-----------|--------|
| Max entrées | 100 (`READ_FILE_STATE_CACHE_SIZE`) |
| Taille max | 25 MB (`DEFAULT_MAX_CACHE_SIZE_BYTES`) |
| Clé | Chemin absolu normalisé |
| Éviction | LRU |

**Opérations** :
- `cloneFileStateCache()` — Clone pour isolation des sous-agents
- `mergeFileStateCaches()` — Fusion avec entrées les plus récentes gagnantes
- `cacheToObject()` — Sérialisation pour la compaction

#### B. FileReadCache — Cache de contenu pour les éditions

**Fichier** : `utils/fileReadCache.ts`

Cache singleton utilisé par FileEditTool pour éviter les relectures.

| Propriété | Valeur |
|-----------|--------|
| Max entrées | 1 000 |
| Éviction | FIFO |
| Invalidation | Basée sur mtime |

#### C. Déduplication de lecture

**Mécanisme** :
```
1. Vérifier si le fichier+plage a déjà été lu (via readFileState)
2. Si mtime identique ET offset/limit identiques :
   → Retourner FILE_UNCHANGED_STUB au lieu du contenu complet
3. Sinon : lecture normale
```

**Conditions** :
- Uniquement pour les lectures texte/notebook (pas images/PDF)
- Uniquement si `offset` est défini (indiquant un Read tool antérieur, pas une injection auto)
- Contrôlé par killswitch : `tengu_read_dedup_killswitch`
- Défaut 3P : dédup activée

---

## 11. Permissions et contrôle d'accès

### Pipeline de vérification

**Fichier** : `utils/permissions/filesystem.ts` → `checkReadPermissionForTool()`

```
1. DENY RULES     → Refus explicites (priorité maximale)
2. ASK RULES      → Demande de permission explicite
3. EDIT ACCESS     → Si l'écriture est autorisée, la lecture l'est aussi
4. WORKING DIR     → Fichiers dans cwd ou additionalWorkingDirectories → auto-allow
5. INTERNAL PATHS  → Session memory, plans, tool results → auto-allow
6. ALLOW RULES     → Autorisations explicites
7. DEFAULT DENY    → Demande à l'utilisateur
```

### Résolution de symlinks

Critique pour la sécurité : `getPathsForPermissionCheck()` résout la **chaîne complète** de symlinks.

```
/project/link → /tmp/intermediate → /etc/passwd
        │              │                │
        └── check ─────┘── check ──────┘── check (TOUS vérifiés)
```

Prévient :
- Attaques symlink→fichier protégé
- Redirections de répertoire parent
- Divergences de chemin canonique (macOS `/tmp` → `/private/tmp`)

### Chemins internes auto-autorisés

| Chemin | Description |
|--------|-------------|
| `/tmp/claude-{uid}/{cwd}/` | Répertoire temporaire du projet |
| `~/.claude/projects/{cwd}/{session}/session-memory/` | Mémoire de session |
| Fichiers auto-memory | Détectés via `isAutoMemFile()` |
| Tool result storage | Résultats d'outils persistés |
| Agent memory paths | Mémoire d'agent |

---

## 12. Sécurité

### Fichiers device bloqués

Chemins bloqués statiquement pour prévenir les lectures infinies ou bloquantes :

| Catégorie | Chemins |
|-----------|---------|
| **Output infini** | `/dev/zero`, `/dev/random`, `/dev/urandom`, `/dev/full` |
| **Bloque l'input** | `/dev/stdin`, `/dev/tty`, `/dev/console` |
| **Sans sens** | `/dev/stdout`, `/dev/stderr` |
| **Alias FD** | `/dev/fd/0`, `/dev/fd/1`, `/dev/fd/2` |
| **Alias Linux** | `/proc/<pid>/fd/0`, `/proc/<pid>/fd/1`, `/proc/<pid>/fd/2` |

Vérification : `isBlockedDevicePath()` — check sur le chemin uniquement, pas d'I/O.

### Fichiers dangereux (permission requise)

**Fichiers** :
```
.gitconfig, .gitmodules, .bashrc, .bash_profile, .zshrc, .zprofile,
.profile, .ripgreprc, .mcp.json, .claude.json
```

**Répertoires** :
```
.git/, .vscode/, .idea/, .claude/
```

Exception : `.claude/worktrees/` est autorisé (structural pour git worktrees).

### Patterns Windows dangereux (vérifiés sur toutes les plateformes)

| Pattern | Risque |
|---------|--------|
| NTFS Alternate Data Streams (`:` après position 2) | Accès à des flux cachés |
| Noms courts 8.3 (`~\d`) | Contournement de pattern matching |
| Préfixes chemins longs (`\\?\`, `\\.`, `//?/`, `//.`) | Contournement de sécurité |
| Points/espaces finaux | Bypass de normalisation plateforme |
| Noms de device DOS (`.CON`, `.PRN`, `.AUX`) | Accès aux devices système |
| Trois+ points consécutifs (`/.../`) | Traversée de répertoire non standard |

### Normalisation de casse

Toutes les vérifications de sécurité utilisent `normalizeCaseForComparison()` qui met en minuscules :
```
.cLauDe/Settings.locaL.json → détecté comme dangereux
.GIT/config                 → détecté comme dangereux
```

### Blocage de chemins UNC

Les chemins réseau (`\\server\share`, `//server/share`) sont bloqués pour prévenir les requêtes réseau non autorisées.

### Blocage de l'expansion shell

Rejeté dans les chemins :
- `$VAR`, `${VAR}` — expansion de variable
- `$(cmd)` — substitution de commande
- `%VAR%` — expansion Windows
- `=cmd` — expansion zsh

---

## 13. Restauration post-compaction

Quand le contexte est compacté (résumé pour libérer des tokens), les fichiers récemment lus sont restaurés automatiquement.

### Fichier clé : `services/compact/compact.ts`

### Processus

```
1. PRÉ-COMPACTION
   └── Sauvegarde du readFileState : preCompactReadFileState = cacheToObject(readFileState)

2. COMPACTION
   └── readFileState.clear()  ← le cache est vidé

3. POST-COMPACTION
   └── createPostCompactFileAttachments()
       ├── Sélectionner les 5 fichiers les plus récents (par timestamp)
       ├── Exclure ceux déjà présents dans les messages préservés
       ├── Relire chaque fichier via FileReadTool (contenu frais, validation)
       └── Injecter comme attachments (pas comme invocations d'outils)
```

### Contraintes de restauration

| Paramètre | Valeur |
|-----------|--------|
| Fichiers max | 5 (`POST_COMPACT_MAX_FILES_TO_RESTORE`) |
| Budget tokens total | 50 000 (`POST_COMPACT_TOKEN_BUDGET`) |
| Tokens max par fichier | 5 000 (`POST_COMPACT_MAX_TOKENS_PER_FILE`) |

**Point important** : Les fichiers restaurés sont injectés comme **attachments** et non comme invocations d'outils, ce qui contourne les vérifications de permissions (le fichier a déjà été autorisé lors de la lecture initiale).

---

## 14. Cas spéciaux

### Chemins de screenshot macOS

Les noms de fichiers de captures d'écran macOS varient selon la version de l'OS :
- Espace normal (U+0020) : `Screenshot 2024-01-01 at 10.30.00 AM.png`
- Espace fine (U+202F) : `Screenshot 2024-01-01 at 10.30.00 AM.png`

La fonction `getAlternateScreenshotPath()` tente le caractère alternatif si le fichier n'est pas trouvé.

### Fichiers auto-memory

Les fichiers de mémoire de session (`~/.claude/session-memory/`) reçoivent un traitement spécial :
- Tracking via `memoryFileMtimes` WeakMap (évite les syscalls sync dans le mapper)
- Préfixe de fraîcheur via `memoryFileFreshnessPrefix()`
- Détection via `isAutoMemFile()`

### Découverte dynamique de skills

La lecture d'un fichier peut déclencher la découverte de répertoires de skills :
- `discoverSkillDirsForPaths()` scanne les chemins lus
- Les répertoires trouvés sont ajoutés à `context.dynamicSkillDirTriggers`
- Le chargement des skills est asynchrone (fire-and-forget)

### Fichiers de session

La lecture de fichiers liés aux sessions est trackée via analytics :
- `detectSessionFileType()` retourne `'session_memory'` ou `'session_transcript'`
- Événement : `tengu_session_file_read`

### Rappel de cybersécurité

Un rappel système est ajouté à toutes les lectures de fichiers texte (sauf sur Opus 4.6) pour prévenir l'analyse de malware.

### Fichier non trouvé — Suggestions

En cas de fichier introuvable :
1. Suggestion de noms similaires avec extensions différentes (`findSimilarFile()`)
2. Suggestion de chemin corrigé sous le cwd (`suggestPathUnderCwd()` — gère le pattern "dossier repo droppé")
3. Inclusion du cwd dans le message d'erreur (`FILE_NOT_FOUND_CWD_NOTE`)

---

## 15. Analytics et logging

### Événements trackés

| Événement | Quand |
|-----------|-------|
| `tengu_file_read_dedup` | Fichier inchangé depuis la dernière lecture |
| `tengu_session_file_read` | Lecture de fichier texte (lignes, octets, offset, limit, extension, type) |
| `tengu_pdf_page_extraction` | Extraction PDF succès/échec |
| `tengu_image_resize_failed` | Erreur de traitement image (type, hash du message) |
| `tengu_image_compress_failed` | Échec de compression image |
| `tengu_file_read_limits_override` | Override des limites de lecture via contexte |

### Privacy

| Mesure | Détail |
|--------|--------|
| Hash des chemins | SHA256 tronqué à 16 caractères |
| Hash du contenu | SHA256 complet (max 100 KB) |
| Pas de contenu brut | Jamais envoyé aux analytics |

**Fichier** : `utils/fileOperationAnalytics.ts` → `logFileOperation()`

---

## 16. Fichiers clés

### Outil principal

| Fichier | Rôle |
|---------|------|
| `tools/FileReadTool/FileReadTool.ts` | Implémentation principale, orchestration de tous les types |
| `tools/FileReadTool/limits.ts` | Configuration maxTokens / maxSizeBytes avec GrowthBook |
| `tools/FileReadTool/prompt.ts` | Template de description et prompt pour le modèle |
| `tools/FileReadTool/imageProcessor.ts` | Chargement de sharp / image-processor-napi |
| `tools/FileReadTool/UI.tsx` | Composants React pour le rendu des résultats |

### Utilitaires de lecture

| Fichier | Rôle |
|---------|------|
| `utils/readFileInRange.ts` | Lecture ligne par ligne avec chemin rapide/streaming |
| `utils/fileRead.ts` | Détection d'encodage, fins de ligne, lecture sync avec métadonnées |
| `utils/file.ts` | Numéros de ligne, chemins d'affichage, suggestions, constantes |
| `utils/pdf.ts` | Lecture PDF et extraction de pages via poppler-utils |
| `utils/pdfUtils.ts` | Parsing de plages de pages, compatibilité modèle |
| `utils/notebook.ts` | Parsing et rendu de notebooks Jupyter |
| `utils/imageResizer.ts` | Pipeline de compression et resize d'images |

### Caching et état

| Fichier | Rôle |
|---------|------|
| `utils/fileStateCache.ts` | Cache LRU d'état fichier (100 entrées, 25 MB) |
| `utils/fileReadCache.ts` | Cache de contenu pour FileEditTool (1000 entrées) |
| `utils/readEditContext.ts` | Recherche contextuelle dans les fichiers pour les éditions |

### Permissions et sécurité

| Fichier | Rôle |
|---------|------|
| `utils/permissions/filesystem.ts` | Vérification permissions lecture, fichiers dangereux, symlinks |
| `utils/permissions/pathValidation.ts` | Validation de chemins, expansion tilde, détection UNC/shell |
| `utils/fsOperations.ts` | Abstraction FS, résolution de symlinks, détection de duplicats |

### Analytics

| Fichier | Rôle |
|---------|------|
| `utils/fileOperationAnalytics.ts` | Logging privacy-preserving des opérations fichier |
