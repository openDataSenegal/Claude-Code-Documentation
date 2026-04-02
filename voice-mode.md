# Voice Mode — Documentation complète

Le système de dictée vocale de Claude Code : capture audio native, STT en temps réel via WebSocket, hold-to-talk, et intégration prompt.

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Architecture](#2-architecture)
3. [Capture audio](#3-capture-audio)
4. [Speech-to-Text (WebSocket)](#4-speech-to-text)
5. [Keyterms — Vocabulaire contextuel](#5-keyterms)
6. [Hold-to-talk — Détection de touche](#6-hold-to-talk)
7. [Cycle de vie d'un enregistrement](#7-cycle-de-vie)
8. [Intégration dans le prompt](#8-intégration-prompt)
9. [Focus mode](#9-focus-mode)
10. [Silent drop et replay](#10-silent-drop)
11. [Visualisation audio](#11-visualisation)
12. [Composants UI](#12-composants-ui)
13. [Commande /voice](#13-commande-voice)
14. [Configuration](#14-configuration)
15. [Feature gates et auth](#15-feature-gates)
16. [Langues supportées](#16-langues)
17. [Limitations et edge cases](#17-limitations)
18. [Analytics](#18-analytics)
19. [Fichiers clés](#19-fichiers-clés)

---

## 1. Vue d'ensemble

Le voice mode est un système **push-to-talk** (hold-to-talk) complet qui permet de dicter du texte dans le prompt de Claude Code. L'utilisateur maintient une touche (par défaut : Espace), parle, puis relâche pour insérer le texte transcrit.

```
Maintien touche → Capture micro → Stream WebSocket → STT Deepgram → Texte dans le prompt
```

### Caractéristiques

| Propriété | Valeur |
|-----------|--------|
| **STT Provider** | Deepgram Nova 2 / Nova 3 (via Anthropic) |
| **Audio** | 16 kHz, 16-bit PCM, mono |
| **Latence** | ~300ms transcription finale |
| **Langues** | 20+ (BCP-47) |
| **Auth** | OAuth Claude.ai uniquement (pas API keys) |
| **Touche par défaut** | Espace (configurable) |
| **Feature flag** | `VOICE_MODE` |

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        UTILISATEUR                           │
│                     Maintient Espace                         │
└─────────────────────┬───────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  useVoiceIntegration (hooks/useVoiceIntegration.tsx)         │
│  ├── Détection hold-to-talk (5 auto-repeats)                │
│  ├── Strip des caractères de warmup                          │
│  └── Capture prefix/suffix autour du curseur                 │
└─────────────────────┬───────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  useVoice (hooks/useVoice.ts, ~1000 lignes)                  │
│  ├── startRecordingSession()                                 │
│  │   ├── voice.startRecording() → chunks audio PCM          │
│  │   └── voiceStreamSTT.connect() → WebSocket               │
│  ├── Accumulation des transcripts (interim + final)          │
│  ├── Gestion des erreurs (retry, silent drop)                │
│  └── finishRecording() → texte final                         │
└──────────┬──────────────────────────┬───────────────────────┘
           ↓                          ↓
┌──────────────────────┐   ┌──────────────────────────────────┐
│ services/voice.ts    │   │ services/voiceStreamSTT.ts       │
│ (Capture audio)      │   │ (WebSocket STT)                  │
│                      │   │                                   │
│ Backends :           │   │ wss://api.anthropic.com           │
│ 1. Native (cpal)     │   │   /api/ws/speech_to_text/        │
│ 2. arecord (ALSA)    │   │   voice_stream                   │
│ 3. SoX rec           │   │                                   │
│                      │   │ Bearer: OAuth token               │
│ 16kHz 16-bit mono    │   │ Provider: Deepgram Nova 2/3      │
└──────────────────────┘   └──────────────────────────────────┘
```

---

## 3. Capture audio

**Fichier** : `services/voice.ts` (526 lignes)

### Spécifications audio

| Paramètre | Valeur |
|-----------|--------|
| Sample rate | 16 000 Hz |
| Bit depth | 16-bit signed |
| Encodage | Linear PCM (raw, pas de header WAV) |
| Canaux | 1 (mono) |
| Byte order | Little-endian |
| Bit rate | ~256 kbps |
| Buffer max | ~2 MB (~60 secondes) |

### Backends de capture (par ordre de priorité)

#### 1. Native (cpal) — `audio-capture-napi`

- Module C++ via NAPI, in-process
- **Plateformes** : macOS, Linux, Windows
- **Dépendances** : CoreAudio.framework + AudioUnit.framework (macOS)
- **Chargement** : Lazy (au premier keypress voice, pas au démarrage)
- **Latence de chargement** : 1-8 secondes (dlopen CoreAudio)
- **Avantage** : Pas de dépendance externe

#### 2. arecord (ALSA) — Fallback Linux

- Commande système `arecord -f S16_LE -r 16000 -c 1 -t raw`
- Vérifié par probe (150ms timeout)
- **Fonctionne** : Linux natif, WSL2+WSLg (Win11 avec PulseAudio)
- **Ne fonctionne pas** : WSL1, Win10-WSL2 (pas de carte ALSA)

#### 3. SoX rec — Dernier recours

- Commande `rec -q -t raw -r 16000 -e signed -b 16 -c 1 --buffer 1024 -`
- Détection de silence intégrée : 2s silence, seuil 3% amplitude
- Filtre SoX : `silence 1 0.1 3% 1 2.0 3%`
- **Avantage** : Cross-platform
- **Inconvénient** : Nécessite installation (`brew install sox`, `apt install sox`)

### Fonctions principales

| Fonction | Description |
|----------|-------------|
| `startRecording(onData, onEnd, options)` | Démarre l'enregistrement, callback par chunk |
| `stopRecording()` | Arrête l'enregistrement (SIGTERM) |
| `checkRecordingAvailability()` | Teste l'accès au micro |
| `checkVoiceDependencies()` | Vérifie les outils installés |
| `requestMicrophonePermission()` | Déclenche le dialogue TCC (macOS) |

---

## 4. Speech-to-Text (WebSocket)

**Fichier** : `services/voiceStreamSTT.ts` (545 lignes)

### Connexion WebSocket

```
Endpoint : wss://api.anthropic.com/api/ws/speech_to_text/voice_stream
Auth     : Authorization: Bearer {accessToken}
Override : VOICE_STREAM_BASE_URL (env var)
```

### Paramètres de query

| Paramètre | Valeur | Description |
|-----------|--------|-------------|
| `encoding` | `linear16` | Format audio PCM 16-bit |
| `sample_rate` | `16000` | Fréquence d'échantillonnage |
| `channels` | `1` | Mono |
| `endpointing_ms` | `300` | Détection fin de phrase (300ms) |
| `utterance_end_ms` | `1000` | Gap inter-phrases (1s) |
| `language` | `en` | Code BCP-47 (configurable) |
| `keyterms` | `...` | Vocabulaire contextuel (max 50) |
| `stt_provider` | `deepgram-nova3` | Si gate `tengu_cobalt_frost` activé |
| `use_conversation_engine` | `true` | Pour Nova 3 |

### Protocole de messages

**Client → Serveur** :

| Type | Format | Description |
|------|--------|-------------|
| Audio | Binaire | Frames PCM 16-bit |
| `KeepAlive` | JSON | Envoyé toutes les 8s |
| `CloseStream` | JSON | Fin d'enregistrement |

**Serveur → Client** :

| Type | Description |
|------|-------------|
| `TranscriptText` | Texte transcrit (interim ou final) |
| `TranscriptEndpoint` | Marque la fin d'une phrase |
| `TranscriptError` | Erreur STT |
| `error` | Erreur de protocole |

### Machine à états de finalisation

```
Enregistrement terminé → CloseStream envoyé
    │
    ├── post_closestream_endpoint (~300ms) — Chemin rapide normal
    │
    ├── no_data_timeout (1.5s) — Pas de transcript reçu
    │   └── Possible silent drop → replay (voir section 10)
    │
    ├── safety_timeout (5s) — Timeout de sécurité
    │
    ├── ws_close — WebSocket fermé par le serveur
    │
    └── ws_already_closed — Déjà finalisé
```

---

## 5. Keyterms — Vocabulaire contextuel

**Fichier** : `services/voiceKeyterms.ts` (107 lignes)

### Sources de keyterms

| Source | Exemples |
|--------|---------|
| **Global** | TypeScript, JSON, OAuth, MCP, symlink, grep, regex, localhost, codebase, webhook, gRPC, dotfiles, worktree, subagent |
| **Projet** | Nom de la racine du projet (basename) |
| **Git** | Mots de la branche courante (split par `-_./\s`) |
| **Fichiers récents** | Noms de fichiers sans extension (split camelCase/snake_case) |

### Contraintes

- **Maximum** : 50 keyterms par session
- **Filtrage** : Uniquement les mots de 3-20 caractères
- **But** : Améliorer la précision STT sur le vocabulaire technique

---

## 6. Hold-to-talk — Détection de touche

**Fichier** : `hooks/useVoiceIntegration.tsx` (~300 lignes)

### Deux modes de touche

#### Mode A — Touche modificateur + lettre (ex: Meta+K, Ctrl+X)

- Active au **premier appui** (intention non ambiguë)
- Pas de seuil de warmup nécessaire
- Pas de caractère injecté dans l'input

#### Mode B — Touche seule (Espace, V, X)

```
Appuis 1-2 : caractères insérés normalement (warmup)
Appui 3-4 : détection de hold en cours
Appui 5+   : ACTIVATION → strip des caractères warmup → enregistrement
```

### Constantes de timing

| Constante | Valeur | Description |
|-----------|--------|-------------|
| `HOLD_THRESHOLD` | 5 | Appuis rapides pour activer (touche seule) |
| `WARMUP_THRESHOLD` | 2 | Premiers appuis qui passent dans l'input |
| `RAPID_KEY_GAP_MS` | 120ms | Gap max entre appuis "rapides" |
| `RELEASE_TIMEOUT_MS` | 200ms | Gap pour détecter le relâchement |
| `REPEAT_FALLBACK_MS` | 600ms | Fallback si pas d'auto-repeat détecté |
| `FIRST_PRESS_FALLBACK_MS` | 2000ms | Couvre le délai initial macOS |

### Keybinding par défaut

```typescript
// keybindings/defaultBindings.ts
feature('VOICE_MODE') ? { space: 'voice:pushToTalk' } : {}
```

Action : `voice:pushToTalk` — contexte Chat uniquement.

Personnalisable via `~/.claude/keybindings.json`.

---

## 7. Cycle de vie d'un enregistrement

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. WARMUP (~120ms)                                               │
│    ├── 2 premiers espaces insérés dans l'input                   │
│    ├── VoiceWarmupHint affiché : "keep holding…"                │
│    └── Attente du seuil HOLD_THRESHOLD (5 appuis)                │
└─────────────────────────────────┬───────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. ACTIVATION                                                    │
│    ├── stripTrailing() → retire les 5 espaces du warmup         │
│    ├── Capture prefix/suffix autour du curseur                   │
│    ├── startRecordingSession()                                   │
│    │   ├── voice.startRecording() → chunks audio bufferisés     │
│    │   └── connectVoiceStream() → WebSocket                     │
│    ├── voiceState → 'recording'                                  │
│    └── VoiceIndicator : "listening…"                            │
└─────────────────────────────────┬───────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. ENREGISTREMENT (~1-60s)                                       │
│    ├── Audio PCM capturé et streamé via WebSocket                │
│    ├── Chunks bufferisés pendant le handshake WS (~1-2s)        │
│    ├── onReady() → flush du buffer                               │
│    ├── TranscriptText (interim) arrive toutes les ~200-500ms    │
│    ├── voiceInterimTranscript mis à jour en temps réel           │
│    ├── Histogramme audio (16 barres) mis à jour par chunk       │
│    └── Curseur affiche la waveform                               │
└─────────────────────────────────┬───────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────┐
│ 4. RELÂCHEMENT                                                   │
│    ├── Pas de keypress pendant 200ms (RELEASE_TIMEOUT_MS)       │
│    ├── finishRecording()                                         │
│    │   ├── stopRecording() → SIGTERM au processus rec/arecord   │
│    │   └── connection.finalize() → CloseStream                  │
│    ├── voiceState → 'processing'                                 │
│    └── VoiceIndicator : shimmer "processing…"                   │
└─────────────────────────────────┬───────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────┐
│ 5. FINALISATION (~300ms à 5s)                                    │
│    ├── Serveur flush le transcript final                         │
│    ├── TranscriptEndpoint reçu → onTranscript(final)            │
│    ├── accumulatedRef = texte final complet                      │
│    ├── WebSocket fermé                                           │
│    └── Si no_data_timeout → silent drop replay (voir section 10)│
└─────────────────────────────────┬───────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────┐
│ 6. INJECTION                                                     │
│    ├── Texte inséré à la position du curseur                     │
│    ├── Espacement géré (prefix + espace + texte + espace + suffix)│
│    ├── voiceState → 'idle'                                       │
│    └── VoiceIndicator masqué                                     │
│                                                                  │
│    L'utilisateur appuie Entrée pour soumettre                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. Intégration dans le prompt

**Fichier** : `hooks/useVoiceIntegration.tsx`

### Modèle d'insertion de texte

```
[prefix existant] + [espace si nécessaire] + [transcript] + [espace si nécessaire] + [suffix existant]
```

### Logique d'espacement

```typescript
const needsSpace = prefix.length > 0 && !/\s$/.test(prefix) && interim.length > 0
const needsTrailingSpace = suffix.length > 0 && !/^\s/.test(suffix)
```

### Tracking de la plage interim

```typescript
interimRange = { start, end }  // Indices du texte interim dans l'input
```

Utilisé pour : atténuer visuellement le texte interim (pas encore final) dans l'UI.

### Gardes de cohérence

- **Submit guard** : Si l'utilisateur soumet avant la fin de la voix → abandon (pas d'écrasement)
- **Edit guard** : Si l'utilisateur édite pendant la voix → bail (ancre stale)
- **Session staleness** : `sessionGenRef` vérifié dans chaque callback async

---

## 9. Focus mode

### Fonctionnement

- Le terminal gagne le focus → enregistrement démarre automatiquement
- Le terminal perd le focus → finalisation
- Timeout de silence : 5 secondes (`FOCUS_SILENCE_TIMEOUT_MS`)

### Différences avec hold-to-talk

| | Hold-to-talk | Focus mode |
|---|---|---|
| Déclenchement | Maintien de touche | Focus terminal |
| Accumulation | Tout le texte accumulé, injecté à la fin | Chaque phrase injectée immédiatement |
| Durée | Jusqu'au relâchement | Jusqu'au blur ou 5s silence |
| Silent drop replay | Oui | Non (pas de fullAudioRef) |

---

## 10. Silent drop et replay

### Problème

~1% des sessions STT subissent un "silent drop" — le serveur ne renvoie aucun transcript malgré de l'audio valide. Causé par un bug session-sticky dans les pods Conversation Engine.

### Détection

```
finalize() → no_data_timeout (1.5s)
  ET hadAudioSignal = true     (le micro a capté du son)
  ET wsConnected = true        (le WebSocket était connecté)
  ET !focusTriggered           (pas en focus mode)
```

### Recovery

```
1. Backoff 250ms (évite le re-connect sur le même pod)
2. connectVoiceStream() sur un nouveau pod
3. Replay du buffer audio complet (fullAudioRef, ~32KB chunks)
4. Envoi CloseStream → attente du transcript
5. Transcript re-accumulé
```

### Limite

Une seule tentative de replay par session (`silentDropRetriedRef`).

### Early-error retry

Si une erreur survient AVANT tout transcript :
- Backoff 250ms
- Reconnexion automatique
- Tracking via `attemptGenRef` (vérifie la fraîcheur)

---

## 11. Visualisation audio

### Histogramme (16 barres)

```typescript
function computeLevel(chunk: Buffer): number {
  // PCM 16-bit → amplitude RMS → normalisé 0-1 → courbe sqrt
  const samples = chunk.length >> 1
  let sumSq = 0
  for (let i = 0; i < chunk.length - 1; i += 2) {
    const sample = ((chunk[i]! | (chunk[i + 1]! << 8)) << 16) >> 16
    sumSq += sample * sample
  }
  return Math.sqrt(Math.sqrt(sumSq / samples) / 2000)
}
```

| Propriété | Valeur |
|-----------|--------|
| Nombre de barres | 16 (`AUDIO_LEVEL_BARS`) |
| Mise à jour | Chaque chunk audio (~50ms) |
| Normalisation | RMS → 0-1 → courbe sqrt (étale les niveaux faibles) |
| Seuil de détection | `> 0.01` (distingue micro silencieux de "pas de parole") |

### Waveform au curseur

Pendant l'enregistrement, le curseur standard est remplacé par une mini-waveform :
- 16 niveaux lissés (moving average)
- Taux de rafraîchissement : 50ms
- Respecte `prefersReducedMotion` (curseur standard si activé)

---

## 12. Composants UI

### VoiceIndicator

| État | Affichage |
|------|-----------|
| `idle` | Rien |
| `recording` | Texte "listening…" (atténué) |
| `processing` | Shimmer pulsant "Voice: processing…" (période 2s) |

### VoiceWarmupHint

```
Texte : "keep holding…" (atténué)
Durée : ~120ms (trop bref pour animation, reste statique)
Position : Footer du prompt input
```

### VoiceModeNotice (bannière d'onboarding)

```
"Voice mode is now available · /voice to enable"
```

| Condition | Affichage |
|-----------|-----------|
| Voice activable + pas encore activé + vu < 3 fois | Affiché |
| Voice activé | Masqué |
| Vu 3 fois | Masqué définitivement |

### Notifications d'erreur

| Message | Cause |
|---------|-------|
| "No audio detected…" | Micro silencieux (`hadAudioSignal = false`) |
| "No speech detected." | Micro OK mais pas de parole reconnue |
| "Voice connection failed…" | Échec WebSocket (réseau/OAuth) |

Timeout : 10 secondes. Dédupliqué par clé `voice-error`.

---

## 13. Commande /voice

**Fichier** : `commands/voice/voice.ts` (151 lignes)

### Vérifications pré-activation (dans l'ordre)

```
1. GrowthBook + auth activé ?
2. Microphone accessible ?
3. API voice_stream disponible ?
4. Dépendances audio installées ?
5. Permission microphone accordée ?
```

### Messages d'erreur

| Check | Message |
|-------|---------|
| Auth manquante | "Voice mode requires a Claude.ai account. Please run /login" |
| Micro refusé | Guidance spécifique plateforme (System Settings sur macOS, etc.) |
| Outils manquants | Auto-détection package manager (brew/apt/dnf/pacman) |

### On enable

1. Sauvegarde `voiceEnabled: true` dans settings
2. Affiche le raccourci (`getShortcutDisplay('voice:pushToTalk', 'Chat', 'Space')`)
3. Affiche le hint de langue (max 2 fois par langue)
4. Log `tengu_voice_toggled`

---

## 14. Configuration

### Settings utilisateur

```json
{
  "voiceEnabled": true
}
```

Persisté dans `~/.claude/settings.json`. Survit au redémarrage.

### Config globale (tracking interne)

| Clé | Type | Description |
|-----|------|-------------|
| `voiceLangHintShownCount` | number | Compteur d'affichages du hint langue |
| `voiceLangHintLastLanguage` | string | Dernière langue (reset compteur si changée) |
| `voiceNoticeSeenCount` | number | Impressions de la bannière VoiceModeNotice (max 3) |

### Variables d'environnement

| Variable | Description |
|----------|-------------|
| `VOICE_STREAM_BASE_URL` | Override de l'endpoint voice_stream |
| `CLAUDE_CODE_REMOTE` | Désactive la voix en environnement distant |

### Keybinding custom

```json
// ~/.claude/keybindings.json
{
  "voice:pushToTalk": "meta+k"
}
```

---

## 15. Feature gates et auth

### Gates

| Gate | Type | Description |
|------|------|-------------|
| `VOICE_MODE` | Feature flag (compile-time) | Dead-code elimination dans les builds sans voix |
| `tengu_amber_quartz_disabled` | GrowthBook (runtime) | Kill-switch d'urgence (inversé : false = activé) |
| `tengu_cobalt_frost` | GrowthBook (runtime) | Active Deepgram Nova 3 (au lieu de Nova 2) |

### Auth

```typescript
hasVoiceAuth() :
  ├── Requiert OAuth Anthropic (pas API key)
  ├── Requiert access token valide
  └── Pas de support Bedrock/Vertex/Foundry

isVoiceModeEnabled() = hasVoiceAuth() + isVoiceGrowthBookEnabled()
```

L'auth est memoizée sur `authVersion` (incrémenté uniquement sur `/login`).

---

## 16. Langues supportées

| Code | Langue | Code | Langue |
|------|--------|------|--------|
| `en` | English | `ru` | Russian |
| `es` | Spanish | `pl` | Polish |
| `fr` | French | `tr` | Turkish |
| `ja` | Japanese | `nl` | Dutch |
| `de` | German | `uk` | Ukrainian |
| `pt` | Portuguese | `el` | Greek |
| `it` | Italian | `cs` | Czech |
| `ko` | Korean | `da` | Danish |
| `hi` | Hindi | `sv` | Swedish |
| `id` | Indonesian | `no` | Norwegian |

### Fallback

- Langue non supportée → English (`DEFAULT_STT_LANGUAGE`)
- `settings.language` peut être un nom complet ("spanish") ou un code ("es")
- Avertissement affiché max 2 fois si la langue ne correspond pas

---

## 17. Limitations et edge cases

| Plateforme | Limitation |
|-----------|------------|
| **Windows natif** | Nécessite le module natif (pas de fallback arecord/SoX) |
| **WSL1 / Win10-WSL2** | Pas de device audio (message d'erreur gracieux) |
| **WSL2+WSLg (Win11)** | Fonctionne via PulseAudio RDP (nécessite SoX ou arecord) |
| **Linux headless** | Probe arecord runtime ; vérifie `/proc/asound/cards` |
| **macOS Arm64** | Préfère native cpal (pas de dépendance SoX) |
| **API Key auth** | Voice non supporté (OAuth Claude.ai uniquement) |
| **Langue inconnue** | WebSocket fermé avec code 1008 |
| **Nova 3** | Désactive l'auto-finalize (interims cumulatifs) |

---

## 18. Analytics

| Événement | Dimensions |
|-----------|-----------|
| `tengu_voice_toggled` | `enabled` |
| `tengu_voice_recording_started` | `sttLanguage`, `focusTriggered`, `systemLocaleLanguage` |
| `tengu_voice_recording_completed` | `transcriptChars`, `recordingDurationMs`, `hadAudioSignal`, `retried`, `silentDropRetried`, `wsConnected`, `focusTriggered` |
| `tengu_voice_silent_drop_replay` | `recordingDurationMs`, `chunkCount` |
| `tengu_voice_stream_early_retry` | (vide) |

---

## 19. Fichiers clés

### Services core

| Fichier | Rôle | Taille |
|---------|------|--------|
| `services/voice.ts` | Capture audio (native + fallbacks) | 526 lignes |
| `services/voiceStreamSTT.ts` | Client WebSocket STT | 545 lignes |
| `services/voiceKeyterms.ts` | Vocabulaire contextuel | 107 lignes |

### Hooks React

| Fichier | Rôle | Taille |
|---------|------|--------|
| `hooks/useVoice.ts` | Orchestration enregistrement + streaming | ~1000 lignes |
| `hooks/useVoiceIntegration.tsx` | Keybinding + intégration input | ~300 lignes |
| `hooks/useVoiceEnabled.ts` | Check auth + disponibilité | 26 lignes |
| `context/voice.tsx` | Store état voice (Redux-like) | 88 lignes |

### UI

| Fichier | Rôle |
|---------|------|
| `components/PromptInput/VoiceIndicator.tsx` | Indicateurs recording/processing |
| `components/LogoV2/VoiceModeNotice.tsx` | Bannière d'onboarding |
| `components/TextInput.tsx` | Waveform au curseur |
| `components/PromptInput/Notifications.tsx` | Messages d'erreur voice |

### Commande et config

| Fichier | Rôle |
|---------|------|
| `commands/voice/voice.ts` | Commande `/voice` |
| `voice/voiceModeEnabled.ts` | Auth + GrowthBook checks |
| `keybindings/defaultBindings.ts` | Binding par défaut (Space) |
