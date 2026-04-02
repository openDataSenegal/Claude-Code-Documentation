# Contrôler LinkedIn depuis Claude Code — Guide de création

Ce document décrit comment créer un outil permettant de contrôler son compte LinkedIn en utilisant l'architecture de Claude Code. Il couvre le choix architectural, l'implémentation d'un serveur MCP, la création d'agents et de skills spécialisés.

---

## Table des matières

1. [Choix architectural](#1-choix-architectural)
2. [Architecture cible](#2-architecture-cible)
3. [Prérequis — API LinkedIn](#3-prérequis)
4. [Étape 1 — Serveur MCP LinkedIn](#4-serveur-mcp)
5. [Étape 2 — Configuration dans Claude Code](#5-configuration)
6. [Étape 3 — Agent LinkedIn](#6-agent-linkedin)
7. [Étape 4 — Skills LinkedIn](#7-skills-linkedin)
8. [Étape 5 — Sécurité et permissions](#8-sécurité)
9. [Étape 6 — Approche alternative avec Browser](#9-approche-browser)
10. [Tests et validation](#10-tests)
11. [Structure complète du projet](#11-structure)
12. [Limitations et considérations](#12-limitations)

---

## 1. Choix architectural

Claude Code offre trois points d'extension pour intégrer LinkedIn :

| Approche | Avantages | Inconvénients | Recommandation |
|----------|-----------|---------------|----------------|
| **Serveur MCP** | Standard MCP, réutilisable, découverte auto des outils | Nécessite un process séparé | **Recommandée** |
| **Tool custom (TypeScript)** | Intégré au binaire, pas de process externe | Nécessite de modifier le source de Claude Code | Possible mais fragile |
| **Agent + Bash** | Zéro code, utilise `curl` | Pas de structure, gestion tokens manuelle | Prototypage uniquement |

**Verdict** : Le **serveur MCP** est la meilleure approche. C'est le pattern standard de Claude Code pour intégrer des services externes, et il supporte l'authentification OAuth nativement.

---

## 2. Architecture cible

```
Claude Code (CLI)
    │
    ├── .mcp.json                    ← Configuration du serveur LinkedIn
    │
    ├── .claude/agents/linkedin.md   ← Agent spécialisé LinkedIn
    │
    ├── .claude/skills/              ← Skills LinkedIn
    │   ├── linkedin-post/SKILL.md
    │   ├── linkedin-network/SKILL.md
    │   └── linkedin-analytics/SKILL.md
    │
    └── Connexion MCP (stdio) ──────► linkedin-mcp-server (Node.js)
                                         │
                                         ├── LinkedIn REST API v2
                                         │   ├── /me (profil)
                                         │   ├── /ugcPosts (publications)
                                         │   ├── /connections (réseau)
                                         │   ├── /socialActions (réactions)
                                         │   └── /organizationalEntityShareStatistics
                                         │
                                         └── OAuth 2.0 (3-legged)
                                             ├── Access token
                                             └── Refresh token
```

---

## 3. Prérequis

### 3.1 Application LinkedIn

1. Créer une application sur [LinkedIn Developer Portal](https://www.linkedin.com/developers/)
2. Configurer les produits API nécessaires :
   - **Share on LinkedIn** — publier du contenu
   - **Sign In with LinkedIn using OpenID Connect** — authentification
   - **Marketing Developer Platform** — analytics (optionnel, approbation requise)
3. Récupérer :
   - `Client ID`
   - `Client Secret`
   - `Redirect URI` : `http://localhost:3456/callback`

### 3.2 Scopes OAuth requis

| Scope | Accès |
|-------|-------|
| `openid` | Authentification OpenID Connect |
| `profile` | Informations de profil |
| `email` | Adresse email |
| `w_member_social` | Publier, commenter, réagir |
| `r_liteprofile` | Profil simplifié |
| `r_organization_social` | Publications d'organisation (si admin page) |

### 3.3 Dépendances

```bash
mkdir linkedin-mcp-server && cd linkedin-mcp-server
npm init -y
npm install @modelcontextprotocol/sdk axios open
npm install -D typescript @types/node
```

---

## 4. Étape 1 — Serveur MCP LinkedIn

### 4.1 Structure du serveur

```
linkedin-mcp-server/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts              # Point d'entrée MCP
│   ├── server.ts             # Définition du serveur et des outils
│   ├── linkedin-api.ts       # Client API LinkedIn
│   ├── auth.ts               # OAuth 2.0 flow
│   └── types.ts              # Types TypeScript
└── .env                      # Credentials (jamais commité)
```

### 4.2 Point d'entrée MCP

```typescript
// src/index.ts
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'
import { createLinkedInServer } from './server.js'

async function main() {
  const server = createLinkedInServer()
  const transport = new StdioServerTransport()
  await server.connect(transport)
  console.error('LinkedIn MCP server started')
}

main().catch(console.error)
```

### 4.3 Définition du serveur et des outils

```typescript
// src/server.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js'
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js'
import { LinkedInAPI } from './linkedin-api.js'

export function createLinkedInServer(): Server {
  const api = new LinkedInAPI()

  const server = new Server(
    { name: 'linkedin', version: '1.0.0' },
    { capabilities: { tools: {} } },
  )

  // ── Liste des outils ──────────────────────────────────

  server.setRequestHandler(ListToolsRequestSchema, async () => ({
    tools: [
      {
        name: 'get_profile',
        description: "Récupère le profil LinkedIn de l'utilisateur connecté",
        inputSchema: {
          type: 'object' as const,
          properties: {},
        },
      },
      {
        name: 'create_post',
        description: 'Publie un post texte sur LinkedIn',
        inputSchema: {
          type: 'object' as const,
          properties: {
            text: {
              type: 'string',
              description: 'Contenu du post (max 3000 caractères)',
            },
            visibility: {
              type: 'string',
              enum: ['PUBLIC', 'CONNECTIONS'],
              description: 'Visibilité du post (défaut: PUBLIC)',
            },
          },
          required: ['text'],
        },
      },
      {
        name: 'create_article_post',
        description: 'Publie un post avec un lien article sur LinkedIn',
        inputSchema: {
          type: 'object' as const,
          properties: {
            text: {
              type: 'string',
              description: 'Texte accompagnant le lien',
            },
            article_url: {
              type: 'string',
              description: "URL de l'article à partager",
            },
            title: {
              type: 'string',
              description: "Titre de l'article (optionnel)",
            },
          },
          required: ['text', 'article_url'],
        },
      },
      {
        name: 'get_feed',
        description: 'Récupère les derniers posts du feed LinkedIn',
        inputSchema: {
          type: 'object' as const,
          properties: {
            count: {
              type: 'number',
              description: 'Nombre de posts à récupérer (défaut: 10, max: 50)',
            },
          },
        },
      },
      {
        name: 'react_to_post',
        description: 'Ajoute une réaction à un post LinkedIn',
        inputSchema: {
          type: 'object' as const,
          properties: {
            post_urn: {
              type: 'string',
              description: 'URN du post (ex: urn:li:share:12345)',
            },
            reaction_type: {
              type: 'string',
              enum: ['LIKE', 'CELEBRATE', 'SUPPORT', 'LOVE', 'INSIGHTFUL', 'FUNNY'],
              description: 'Type de réaction',
            },
          },
          required: ['post_urn', 'reaction_type'],
        },
      },
      {
        name: 'comment_on_post',
        description: 'Ajoute un commentaire à un post LinkedIn',
        inputSchema: {
          type: 'object' as const,
          properties: {
            post_urn: {
              type: 'string',
              description: 'URN du post',
            },
            text: {
              type: 'string',
              description: 'Texte du commentaire',
            },
          },
          required: ['post_urn', 'text'],
        },
      },
      {
        name: 'get_connections',
        description: 'Récupère la liste des connexions',
        inputSchema: {
          type: 'object' as const,
          properties: {
            count: {
              type: 'number',
              description: 'Nombre de connexions (défaut: 20)',
            },
            start: {
              type: 'number',
              description: 'Offset de pagination (défaut: 0)',
            },
          },
        },
      },
      {
        name: 'search_people',
        description: 'Recherche des personnes sur LinkedIn',
        inputSchema: {
          type: 'object' as const,
          properties: {
            keywords: {
              type: 'string',
              description: 'Mots-clés de recherche',
            },
            count: {
              type: 'number',
              description: 'Nombre de résultats (défaut: 10)',
            },
          },
          required: ['keywords'],
        },
      },
      {
        name: 'send_message',
        description: 'Envoie un message privé à une connexion',
        inputSchema: {
          type: 'object' as const,
          properties: {
            recipient_urn: {
              type: 'string',
              description: 'URN du destinataire (ex: urn:li:person:ABC123)',
            },
            text: {
              type: 'string',
              description: 'Contenu du message',
            },
          },
          required: ['recipient_urn', 'text'],
        },
      },
      {
        name: 'get_post_analytics',
        description: "Récupère les statistiques d'un post",
        inputSchema: {
          type: 'object' as const,
          properties: {
            post_urn: {
              type: 'string',
              description: 'URN du post',
            },
          },
          required: ['post_urn'],
        },
      },
      {
        name: 'delete_post',
        description: 'Supprime un post LinkedIn',
        inputSchema: {
          type: 'object' as const,
          properties: {
            post_urn: {
              type: 'string',
              description: 'URN du post à supprimer',
            },
          },
          required: ['post_urn'],
        },
      },
    ],
  }))

  // ── Exécution des outils ──────────────────────────────

  server.setRequestHandler(CallToolRequestSchema, async (request) => {
    const { name, arguments: args } = request.params

    try {
      switch (name) {
        case 'get_profile':
          return formatResult(await api.getProfile())

        case 'create_post':
          return formatResult(await api.createPost(
            args.text as string,
            (args.visibility as string) || 'PUBLIC',
          ))

        case 'create_article_post':
          return formatResult(await api.createArticlePost(
            args.text as string,
            args.article_url as string,
            args.title as string | undefined,
          ))

        case 'get_feed':
          return formatResult(await api.getFeed(
            (args.count as number) || 10,
          ))

        case 'react_to_post':
          return formatResult(await api.reactToPost(
            args.post_urn as string,
            args.reaction_type as string,
          ))

        case 'comment_on_post':
          return formatResult(await api.commentOnPost(
            args.post_urn as string,
            args.text as string,
          ))

        case 'get_connections':
          return formatResult(await api.getConnections(
            (args.count as number) || 20,
            (args.start as number) || 0,
          ))

        case 'search_people':
          return formatResult(await api.searchPeople(
            args.keywords as string,
            (args.count as number) || 10,
          ))

        case 'send_message':
          return formatResult(await api.sendMessage(
            args.recipient_urn as string,
            args.text as string,
          ))

        case 'get_post_analytics':
          return formatResult(await api.getPostAnalytics(
            args.post_urn as string,
          ))

        case 'delete_post':
          return formatResult(await api.deletePost(
            args.post_urn as string,
          ))

        default:
          return { content: [{ type: 'text', text: `Unknown tool: ${name}` }], isError: true }
      }
    } catch (error) {
      return {
        content: [{ type: 'text', text: `Error: ${error instanceof Error ? error.message : String(error)}` }],
        isError: true,
      }
    }
  })

  return server
}

function formatResult(data: unknown) {
  return {
    content: [{
      type: 'text' as const,
      text: JSON.stringify(data, null, 2),
    }],
  }
}
```

### 4.4 Client API LinkedIn

```typescript
// src/linkedin-api.ts
import axios, { type AxiosInstance } from 'axios'
import { getAccessToken } from './auth.js'

export class LinkedInAPI {
  private client: AxiosInstance

  constructor() {
    this.client = axios.create({
      baseURL: 'https://api.linkedin.com/v2',
      headers: { 'X-Restli-Protocol-Version': '2.0.0' },
    })

    this.client.interceptors.request.use(async (config) => {
      const token = await getAccessToken()
      config.headers.Authorization = `Bearer ${token}`
      return config
    })
  }

  async getProfile() {
    const { data } = await this.client.get('/me', {
      params: { projection: '(id,localizedFirstName,localizedLastName,localizedHeadline,profilePicture)' },
    })
    return data
  }

  async createPost(text: string, visibility: string) {
    const profile = await this.getProfile()
    const authorUrn = `urn:li:person:${profile.id}`

    const { data } = await this.client.post('/ugcPosts', {
      author: authorUrn,
      lifecycleState: 'PUBLISHED',
      specificContent: {
        'com.linkedin.ugc.ShareContent': {
          shareCommentary: { text },
          shareMediaCategory: 'NONE',
        },
      },
      visibility: {
        'com.linkedin.ugc.MemberNetworkVisibility': visibility,
      },
    })
    return { success: true, postUrn: data.id, message: 'Post published successfully' }
  }

  async createArticlePost(text: string, articleUrl: string, title?: string) {
    const profile = await this.getProfile()
    const authorUrn = `urn:li:person:${profile.id}`

    const { data } = await this.client.post('/ugcPosts', {
      author: authorUrn,
      lifecycleState: 'PUBLISHED',
      specificContent: {
        'com.linkedin.ugc.ShareContent': {
          shareCommentary: { text },
          shareMediaCategory: 'ARTICLE',
          media: [{
            status: 'READY',
            originalUrl: articleUrl,
            ...(title && { title: { text: title } }),
          }],
        },
      },
      visibility: {
        'com.linkedin.ugc.MemberNetworkVisibility': 'PUBLIC',
      },
    })
    return { success: true, postUrn: data.id }
  }

  async getFeed(count: number) {
    const { data } = await this.client.get('/ugcPosts', {
      params: { q: 'authors', authors: 'List(urn:li:person:me)', count },
    })
    return data.elements?.map((post: any) => ({
      urn: post.id,
      text: post.specificContent?.['com.linkedin.ugc.ShareContent']?.shareCommentary?.text,
      created: new Date(post.created?.time).toISOString(),
      lifecycleState: post.lifecycleState,
    })) || []
  }

  async reactToPost(postUrn: string, reactionType: string) {
    const profile = await this.getProfile()
    const actorUrn = `urn:li:person:${profile.id}`

    await this.client.post(`/socialActions/${encodeURIComponent(postUrn)}/likes`, {
      actor: actorUrn,
      object: postUrn,
    })
    return { success: true, message: `Reacted with ${reactionType}` }
  }

  async commentOnPost(postUrn: string, text: string) {
    const profile = await this.getProfile()
    const actorUrn = `urn:li:person:${profile.id}`

    const { data } = await this.client.post(
      `/socialActions/${encodeURIComponent(postUrn)}/comments`,
      { actor: actorUrn, message: { text } },
    )
    return { success: true, commentUrn: data.id }
  }

  async getConnections(count: number, start: number) {
    const { data } = await this.client.get('/connections', {
      params: { q: 'viewer', start, count },
    })
    return data
  }

  async searchPeople(keywords: string, count: number) {
    const { data } = await this.client.get('/search/blended', {
      params: { q: 'all', keywords, count, origin: 'GLOBAL_SEARCH_HEADER' },
    })
    return data
  }

  async sendMessage(recipientUrn: string, text: string) {
    const { data } = await this.client.post('/messaging/conversations', {
      recipients: [recipientUrn],
      body: { text },
    })
    return { success: true, conversationUrn: data.id }
  }

  async getPostAnalytics(postUrn: string) {
    const { data } = await this.client.get(
      `/socialActions/${encodeURIComponent(postUrn)}`,
    )
    return {
      likes: data.likesSummary?.totalLikes || 0,
      comments: data.commentsSummary?.totalFirstLevelComments || 0,
      shares: data.sharesSummary?.totalShares || 0,
    }
  }

  async deletePost(postUrn: string) {
    await this.client.delete(`/ugcPosts/${encodeURIComponent(postUrn)}`)
    return { success: true, message: 'Post deleted' }
  }
}
```

### 4.5 Authentification OAuth

```typescript
// src/auth.ts
import axios from 'axios'
import fs from 'fs'
import path from 'path'
import { fileURLToPath } from 'url'

const __dirname = path.dirname(fileURLToPath(import.meta.url))
const TOKEN_PATH = path.join(__dirname, '..', '.tokens.json')

const CLIENT_ID = process.env.LINKEDIN_CLIENT_ID!
const CLIENT_SECRET = process.env.LINKEDIN_CLIENT_SECRET!
const REDIRECT_URI = process.env.LINKEDIN_REDIRECT_URI || 'http://localhost:3456/callback'

interface TokenData {
  access_token: string
  refresh_token?: string
  expires_at: number
}

function loadTokens(): TokenData | null {
  try {
    if (fs.existsSync(TOKEN_PATH)) {
      return JSON.parse(fs.readFileSync(TOKEN_PATH, 'utf-8'))
    }
  } catch { /* ignore */ }
  return null
}

function saveTokens(tokens: TokenData): void {
  fs.writeFileSync(TOKEN_PATH, JSON.stringify(tokens, null, 2), { mode: 0o600 })
}

async function refreshAccessToken(refreshToken: string): Promise<TokenData> {
  const { data } = await axios.post('https://www.linkedin.com/oauth/v2/accessToken', null, {
    params: {
      grant_type: 'refresh_token',
      refresh_token: refreshToken,
      client_id: CLIENT_ID,
      client_secret: CLIENT_SECRET,
    },
  })

  const tokens: TokenData = {
    access_token: data.access_token,
    refresh_token: data.refresh_token || refreshToken,
    expires_at: Date.now() + data.expires_in * 1000,
  }
  saveTokens(tokens)
  return tokens
}

export async function getAccessToken(): Promise<string> {
  const tokens = loadTokens()

  if (!tokens) {
    throw new Error(
      'No LinkedIn tokens found. Run the auth flow first:\n' +
      `  Open: https://www.linkedin.com/oauth/v2/authorization?` +
      `response_type=code&client_id=${CLIENT_ID}&redirect_uri=${encodeURIComponent(REDIRECT_URI)}` +
      `&scope=${encodeURIComponent('openid profile email w_member_social')}` +
      `&state=claude-code-linkedin\n` +
      '  Then exchange the code with: npm run auth -- <code>'
    )
  }

  // Refresh 5 minutes before expiry
  if (Date.now() > tokens.expires_at - 300_000) {
    if (tokens.refresh_token) {
      const refreshed = await refreshAccessToken(tokens.refresh_token)
      return refreshed.access_token
    }
    throw new Error('Token expired and no refresh token available. Re-authenticate.')
  }

  return tokens.access_token
}

// Script d'échange de code (npm run auth -- <code>)
export async function exchangeCode(code: string): Promise<void> {
  const { data } = await axios.post('https://www.linkedin.com/oauth/v2/accessToken', null, {
    params: {
      grant_type: 'authorization_code',
      code,
      client_id: CLIENT_ID,
      client_secret: CLIENT_SECRET,
      redirect_uri: REDIRECT_URI,
    },
  })

  const tokens: TokenData = {
    access_token: data.access_token,
    refresh_token: data.refresh_token,
    expires_at: Date.now() + data.expires_in * 1000,
  }
  saveTokens(tokens)
  console.log('Tokens saved successfully.')
}
```

### 4.6 package.json

```json
{
  "name": "linkedin-mcp-server",
  "version": "1.0.0",
  "type": "module",
  "bin": {
    "linkedin-mcp": "./dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "auth": "node dist/auth-cli.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0",
    "axios": "^1.7.0",
    "open": "^10.0.0"
  },
  "devDependencies": {
    "typescript": "^5.4.0",
    "@types/node": "^20.0.0"
  }
}
```

### 4.7 Build et test

```bash
npm run build
# Test le serveur en mode stdio (Ctrl+C pour quitter)
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | node dist/index.js
```

---

## 5. Étape 2 — Configuration dans Claude Code

### 5.1 Fichier `.mcp.json` (niveau projet)

```json
{
  "mcpServers": {
    "linkedin": {
      "command": "node",
      "args": ["/chemin/absolu/vers/linkedin-mcp-server/dist/index.js"],
      "env": {
        "LINKEDIN_CLIENT_ID": "${LINKEDIN_CLIENT_ID}",
        "LINKEDIN_CLIENT_SECRET": "${LINKEDIN_CLIENT_SECRET}",
        "LINKEDIN_REDIRECT_URI": "http://localhost:3456/callback"
      }
    }
  }
}
```

### 5.2 Configuration globale (~/.claude/config.json)

Pour rendre le serveur disponible dans tous les projets :

```json
{
  "mcpServers": {
    "linkedin": {
      "command": "node",
      "args": ["/chemin/absolu/vers/linkedin-mcp-server/dist/index.js"],
      "env": {
        "LINKEDIN_CLIENT_ID": "${LINKEDIN_CLIENT_ID}",
        "LINKEDIN_CLIENT_SECRET": "${LINKEDIN_CLIENT_SECRET}"
      }
    }
  }
}
```

### 5.3 Variables d'environnement

```bash
# Dans ~/.zshrc ou ~/.bashrc
export LINKEDIN_CLIENT_ID="votre_client_id"
export LINKEDIN_CLIENT_SECRET="votre_client_secret"
```

### 5.4 Vérification

```bash
# Depuis Claude Code
claude
> /mcp
# Devrait afficher : linkedin (connected) avec les outils listés
```

---

## 6. Étape 3 — Agent LinkedIn

### 6.1 Agent spécialisé

Créer `.claude/agents/linkedin.md` :

```markdown
---
name: linkedin
description: >
  Manage LinkedIn presence: create posts, engage with content,
  analyze performance, and manage connections.
  Use when the user wants to publish on LinkedIn, check analytics,
  or interact with their network.
tools:
  - Bash
  - Read
  - Edit
  - Write
  - Glob
  - Grep
mcpServers:
  - linkedin
requiredMcpServers:
  - linkedin
model: sonnet
color: blue
maxTurns: 30
---

# LinkedIn Manager Agent

Tu es un gestionnaire de présence LinkedIn expert. Tu maîtrises la stratégie
de contenu, l'engagement et l'analytics LinkedIn.

## Principes

1. **Ton** : Professionnel mais authentique. Adapte le ton au contexte.
2. **Contenu** : Privilégie la valeur ajoutée. Pas de spam, pas de clickbait.
3. **Engagement** : Interactions pertinentes et constructives.
4. **Timing** : Suggère les meilleurs moments pour publier (mardi-jeudi 8h-10h ou 17h-18h).

## Capacités

### Publication
- Rédiger des posts optimisés (hooks, structure, hashtags, CTA)
- Partager des articles avec commentaire éditorial
- Planifier une série de posts sur un thème

### Engagement
- Réagir aux posts pertinents du feed
- Rédiger des commentaires à valeur ajoutée
- Identifier les posts nécessitant une réponse

### Analytics
- Analyser la performance des posts (likes, commentaires, partages)
- Identifier les tendances de contenu
- Recommander des ajustements de stratégie

### Réseau
- Explorer les connexions et identifier des opportunités
- Rédiger des messages personnalisés de prise de contact
- Rechercher des profils par mots-clés

## Règles strictes

- TOUJOURS demander confirmation avant de publier, commenter ou envoyer un message
- JAMAIS de contenu controversé, politique ou religieux sans accord explicite
- LIMITER à 5 hashtags par post
- RESPECTER la limite de 3000 caractères par post
- VÉRIFIER les URLs avant de les partager
```

### 6.2 Utilisation

```bash
claude
> Utilise l'agent linkedin pour publier un post sur les tendances IA en 2026
# Claude invoque : Agent(subagent_type: "linkedin", prompt: "...")
```

---

## 7. Étape 4 — Skills LinkedIn

### 7.1 Skill — Publier un post

Créer `.claude/skills/linkedin-post/SKILL.md` :

```markdown
---
description: Rédige et publie un post LinkedIn optimisé
argument-hint: "[sujet du post]"
arguments: [topic]
when_to_use: >
  Use when the user wants to publish a LinkedIn post, create content
  for LinkedIn, or write a professional update.
  Examples: "publie sur LinkedIn", "post LinkedIn", "write a LinkedIn post"
allowed-tools:
  - Read
  - Edit
  - Write
---

# Publier sur LinkedIn

## Objectif
Rédiger un post LinkedIn engageant sur le sujet : **$topic**

## Processus

### 1. Recherche de contexte
Si le sujet touche au projet courant, parcours le code pour ajouter
des insights techniques concrets.

### 2. Rédaction du post
Structure recommandée :
- **Hook** (1ère ligne) : Question provocante ou stat surprenante
- **Corps** (3-5 paragraphes courts) : Développe avec des exemples concrets
- **CTA** (dernière ligne) : Question ouverte pour engager

### 3. Optimisation
- Ajoute 3-5 hashtags pertinents
- Vérifie : < 3000 caractères
- Pas d'emojis excessifs (max 3-4 par post)
- Aère le texte (sauts de ligne entre les paragraphes)

### 4. Validation
Présente le post formaté à l'utilisateur et demande confirmation
avant de publier via l'outil `create_post`.

### 5. Publication
Après confirmation, publie le post et rapporte le lien/URN.
```

### 7.2 Skill — Analyser le réseau

Créer `.claude/skills/linkedin-network/SKILL.md` :

```markdown
---
description: Analyse le réseau LinkedIn et recommande des actions
when_to_use: >
  Use when the user wants to analyze their LinkedIn network,
  find people, or explore connection opportunities.
allowed-tools:
  - Read
  - Write
---

# Analyse Réseau LinkedIn

## Processus

### 1. État du réseau
Récupère les connexions récentes et le profil.

### 2. Analyse
- Nombre de connexions
- Domaines représentés
- Connexions récentes

### 3. Recommandations
- Profils à contacter
- Messages personnalisés suggérés
- Groupes à rejoindre

Présente un rapport structuré avec des actions concrètes.
```

### 7.3 Skill — Analytics de posts

Créer `.claude/skills/linkedin-analytics/SKILL.md` :

```markdown
---
description: Analyse la performance des posts LinkedIn récents
when_to_use: >
  Use when the user wants LinkedIn analytics, post performance,
  or content strategy insights.
allowed-tools:
  - Read
  - Write
---

# Analytics LinkedIn

## Processus

### 1. Récupération des posts
Récupère les 10 derniers posts via `get_feed`.

### 2. Analyse par post
Pour chaque post, récupère les statistiques via `get_post_analytics` :
- Likes, commentaires, partages
- Taux d'engagement estimé

### 3. Rapport
Génère un rapport avec :
- Tableau des performances
- Meilleur post (par engagement)
- Pire post (par engagement)
- Tendances (sujets qui fonctionnent)
- Recommandations pour les prochains posts
```

---

## 8. Étape 5 — Sécurité et permissions

### 8.1 Règles de permission dans settings.json

```json
{
  "permissions": {
    "deny": [],
    "ask": [
      "mcp:linkedin:create_post",
      "mcp:linkedin:delete_post",
      "mcp:linkedin:send_message",
      "mcp:linkedin:comment_on_post",
      "mcp:linkedin:react_to_post"
    ],
    "allow": [
      "mcp:linkedin:get_profile",
      "mcp:linkedin:get_feed",
      "mcp:linkedin:get_connections",
      "mcp:linkedin:search_people",
      "mcp:linkedin:get_post_analytics"
    ]
  }
}
```

**Logique** :
- Les opérations de **lecture** sont auto-autorisées
- Les opérations d'**écriture** (publier, supprimer, envoyer, commenter, réagir) demandent confirmation
- Aucune opération n'est bloquée (l'utilisateur décide)

### 8.2 Protection des credentials

| Mesure | Description |
|--------|-------------|
| `.env` jamais commité | Ajouter `.env` et `.tokens.json` au `.gitignore` |
| Permissions fichier | Tokens stockés en 0o600 (propriétaire uniquement) |
| Variables d'environnement | Credentials dans l'environnement shell, pas dans le code |
| Refresh automatique | Token rafraîchi 5 min avant expiration |
| Scope minimal | Demander uniquement les scopes nécessaires |

### 8.3 .gitignore

```gitignore
# LinkedIn MCP server
linkedin-mcp-server/.env
linkedin-mcp-server/.tokens.json
linkedin-mcp-server/dist/
```

---

## 9. Étape 6 — Approche alternative avec Browser

Si l'API LinkedIn est trop restrictive (accès aux endpoints limité), une approche **browser automation** via le feature flag `WEB_BROWSER_TOOL` ou un serveur MCP Puppeteer est possible.

### 9.1 MCP Puppeteer pour LinkedIn

```json
{
  "mcpServers": {
    "linkedin-browser": {
      "command": "npx",
      "args": ["@anthropic/mcp-puppeteer"],
      "env": {
        "PUPPETEER_HEADLESS": "false"
      }
    }
  }
}
```

### 9.2 Agent browser LinkedIn

```markdown
---
name: linkedin-browser
description: Control LinkedIn through browser automation
tools: [Bash, Read]
mcpServers:
  - linkedin-browser
color: orange
---

Tu contrôles LinkedIn via un navigateur automatisé.

## Flux d'interaction
1. Naviguer vers linkedin.com
2. Vérifier que l'utilisateur est connecté
3. Effectuer les actions demandées via les sélecteurs DOM

## Sélecteurs clés (à adapter selon les mises à jour LinkedIn)
- Feed : `.feed-shared-update-v2`
- Bouton publier : `.share-box-feed-entry__trigger`
- Éditeur de post : `.ql-editor`
- Bouton poster : `.share-actions__primary-action`

## Avertissements
- LinkedIn peut détecter l'automatisation et bloquer le compte
- Les sélecteurs CSS peuvent changer sans préavis
- Toujours demander confirmation avant toute action
```

**Avertissement** : L'automatisation par browser enfreint potentiellement les conditions d'utilisation de LinkedIn. L'approche API officielle est toujours préférable.

---

## 10. Tests et validation

### 10.1 Test du serveur MCP isolé

```bash
cd linkedin-mcp-server
npm run build

# Test de listing des outils
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | node dist/index.js

# Test d'appel d'outil
echo '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"get_profile","arguments":{}}}' | node dist/index.js
```

### 10.2 Test dans Claude Code

```bash
claude

# Vérifier la connexion MCP
> /mcp

# Test lecture (auto-autorisé)
> Quel est mon profil LinkedIn ?

# Test écriture (demande confirmation)
> Publie "Hello World" sur LinkedIn

# Test agent
> Utilise l'agent linkedin pour analyser mes 5 derniers posts

# Test skill
> /linkedin-post "Les 3 leçons que j'ai apprises en développant avec l'IA"
```

### 10.3 Checklist de validation

- [ ] Le serveur MCP démarre sans erreur
- [ ] `/mcp` affiche "linkedin (connected)"
- [ ] `get_profile` retourne les informations du profil
- [ ] `create_post` demande confirmation avant publication
- [ ] `delete_post` demande confirmation
- [ ] Les tokens se rafraîchissent automatiquement
- [ ] L'agent linkedin fonctionne via `subagent_type`
- [ ] Les skills `/linkedin-post` et `/linkedin-analytics` fonctionnent
- [ ] Les permissions read/write sont correctement séparées

---

## 11. Structure complète du projet

```
mon-projet/
├── .mcp.json                              # Config MCP (linkedin server)
├── .gitignore                             # Exclut .env, .tokens.json
│
├── .claude/
│   ├── agents/
│   │   └── linkedin.md                    # Agent LinkedIn
│   ├── skills/
│   │   ├── linkedin-post/
│   │   │   └── SKILL.md                   # Skill publication
│   │   ├── linkedin-network/
│   │   │   └── SKILL.md                   # Skill réseau
│   │   └── linkedin-analytics/
│   │       └── SKILL.md                   # Skill analytics
│   └── settings.json                      # Permissions MCP
│
└── linkedin-mcp-server/                   # Serveur MCP
    ├── package.json
    ├── tsconfig.json
    ├── .env                               # Credentials (JAMAIS commité)
    ├── .tokens.json                       # Tokens OAuth (JAMAIS commité)
    └── src/
        ├── index.ts                       # Point d'entrée stdio
        ├── server.ts                      # Définition serveur + outils
        ├── linkedin-api.ts                # Client API REST LinkedIn
        ├── auth.ts                        # OAuth 2.0 flow + refresh
        └── types.ts                       # Types TypeScript
```

---

## 12. Limitations et considérations

### API LinkedIn

| Limitation | Détail |
|------------|--------|
| **Rate limiting** | 100 requêtes/jour pour la plupart des endpoints |
| **Accès restreint** | Marketing API nécessite une approbation LinkedIn |
| **Scope limité** | `w_member_social` requis pour publier (nécessite approbation app) |
| **Pas de feed public** | L'API ne permet pas de lire le feed d'autres utilisateurs |
| **Recherche limitée** | L'endpoint `/search` n'est plus accessible aux apps communautaires |
| **Messaging** | L'API messaging est restreinte aux apps approuvées |

### Recommandations

| Aspect | Recommandation |
|--------|----------------|
| **Rate limits** | Implémenter un cache local et un throttle dans le serveur MCP |
| **Tokens** | Ne jamais exposer les tokens dans les logs ou la conversation |
| **Contenu** | Toujours valider avec l'utilisateur avant publication |
| **Automatisation** | Respecter les CGU LinkedIn — pas de mass actions |
| **Fallback** | Prévoir un mode dégradé si l'API est indisponible |
| **Monitoring** | Logger les appels API pour détecter les rate limits |

### Évolutions possibles

- Planification de posts (via CronCreate + agent background)
- Templates de posts par catégorie (via CLAUDE.md ou mémoire)
- Analytics automatiques quotidiens (via KAIROS + PushNotification)
- Intégration avec d'autres réseaux (même pattern MCP)
- A/B testing de posts (publier des variantes, comparer les analytics)
