# Secure Local Personal Bot — Architecture Plan

## Vision

A **privacy-first personal AI assistant** that runs entirely on your own hardware using LM Studio as the local inference engine. No cloud LLM dependency. A polished onboarding UI (not CLI), native mobile apps, WhatsApp + email integration, and full voice capabilities (TTS, STT, real-time voice conversations).

---

## 1. Components We Already Have (from clawdbot)

The existing codebase is a strong foundation. Here's what we keep and leverage:

| Component | Location | Status |
|---|---|---|
| **WhatsApp channel** | `src/whatsapp/` (Baileys) | ✅ Ready — already built |
| **Gateway control plane** | `src/gateway/` | ✅ Ready — local WebSocket server, sessions, presence |
| **iOS native app** | `apps/ios/` (SwiftUI) | ✅ Ready — voice wake, talk mode, camera, Bonjour pairing |
| **Android native app** | `apps/android/` (Kotlin) | ✅ Ready — talk mode, camera, canvas |
| **macOS menu bar app** | `apps/macos/` (SwiftUI) | ✅ Ready |
| **TTS engine** | `src/tts/` + `node-edge-tts` | ✅ Ready — Edge TTS fallback |
| **Plugin system** | `src/plugins/` | ✅ Ready — manifest, HTTP registry, hooks |
| **Security framework** | `src/security/`, SECURITY.md | ✅ Ready — DM pairing, allowlisting, sandbox, secret detection |
| **Onboarding wizard** | `src/wizard/` | ⚠️ Exists but CLI-based — needs UI replacement |
| **Config system** | `src/config/` (Zod schemas) | ✅ Ready — schema-validated, env-var substitution |
| **Session/memory** | `src/memory/`, `src/sessions/` | ✅ Ready |
| **Media pipeline** | `src/media/`, `src/media-understanding/` | ✅ Ready — image/video/audio processing |
| **Email skill** | `skills/himalaya/` | ⚠️ Exists — needs hardening for local-only mode |
| **Voice call extension** | `extensions/voice-call/` (Twilio) | ⚠️ Exists — needs local STT/TTS path |
| **Browser automation** | `src/browser/` (Playwright) | ✅ Ready |
| **Cron scheduling** | `src/cron/` | ✅ Ready |
| **Docker sandbox** | `Dockerfile`, `Dockerfile.sandbox` | ✅ Ready |

---

## 2. New Components Required

### 2.1 LM Studio Integration Provider

**What:** A new provider module that connects to LM Studio's local API as the primary inference backend.

**Why:** Replace cloud LLM dependency with a local model running on LM Studio.

**Components:**

```
src/providers/lmstudio/
├── index.ts              # Provider entry & registration
├── client.ts             # OpenAI-compatible client → localhost:1234
├── health.ts             # Health check & model availability monitor
├── model-manager.ts      # Auto-load/unload via lms CLI or /api/v1/load
├── config.schema.ts      # Zod schema for LM Studio provider config
└── __tests__/
    └── client.test.ts
```

**Key design decisions:**
- Use LM Studio's **OpenAI-compatible API** (`POST /v1/chat/completions`) so the existing agent runtime works with zero changes
- Use the **native REST API** (`/api/v1/`) for model lifecycle (load, unload, download, health)
- Support **Just-In-Time (JIT) loading** — model loads automatically on first request
- Support **TTL auto-eviction** — idle models free VRAM after configurable timeout
- Health endpoint polls `lms server status` / `GET /v1/models` to show model status in UI
- **Speculative decoding** support via `draft_model` parameter for faster responses
- **Tool/function calling** pass-through (LM Studio supports this natively)

**Config schema:**
```typescript
lmstudio: {
  endpoint: "http://127.0.0.1:1234",   // Local only, never exposed
  apiToken: "${LM_STUDIO_TOKEN}",       // Optional auth token
  defaultModel: "qwen3-8b",            // Model identifier
  draftModel: "qwen3-1.7b",            // For speculative decoding
  ttl: 300,                             // Auto-unload after 5min idle
  jit: true,                            // Just-in-time loading
  gpu: "max",                           // GPU offload setting
  contextLength: 8192,                  // Context window
  maxTokens: 4096,                      // Max generation length
}
```

**Systemd / daemon setup:**
- Provide a systemd unit file template for headless LM Studio operation
- `lms daemon up` → `lms load <model> --yes` → `lms server start`
- Auto-restart on crash

---

### 2.2 Onboarding Web UI

**What:** Replace the CLI-based `src/wizard/` with a local web-based onboarding experience.

**Why:** "Too tech niche" — users need a visual, step-by-step setup flow with toggles, not terminal prompts.

**Components:**

```
ui/onboarding/
├── package.json
├── vite.config.ts
├── src/
│   ├── App.tsx
│   ├── main.tsx
│   ├── pages/
│   │   ├── Welcome.tsx           # Intro & system check
│   │   ├── ModelSetup.tsx        # LM Studio detection & model selection
│   │   ├── ChannelPicker.tsx     # Toggle WhatsApp, Email, Voice, etc.
│   │   ├── WhatsAppSetup.tsx     # QR scan flow
│   │   ├── EmailSetup.tsx        # IMAP/SMTP credentials
│   │   ├── VoiceSetup.tsx        # Mic test, TTS preview, STT engine pick
│   │   ├── SecuritySetup.tsx     # DM pairing, allowlist, sandbox toggle
│   │   ├── MobileSetup.tsx       # QR code for iOS/Android pairing
│   │   ├── PersonalitySetup.tsx  # Bot name, system prompt, behavior
│   │   └── Review.tsx            # Summary & launch
│   ├── components/
│   │   ├── FeatureCard.tsx       # Toggle card with icon + description
│   │   ├── StepIndicator.tsx     # Progress stepper
│   │   ├── SystemCheck.tsx       # GPU, RAM, LM Studio status
│   │   └── QRScanner.tsx         # Reusable QR display
│   └── hooks/
│       ├── useGatewayStatus.ts   # WebSocket to gateway
│       └── useLMStudioHealth.ts  # Poll LM Studio availability
├── public/
│   └── index.html
└── tsconfig.json
```

**Key design decisions:**
- **React + Vite** (consistent with existing `ui/` workspace)
- Served by the gateway on `http://localhost:<port>/onboard`
- Communicates with gateway via WebSocket for real-time status
- Writes to `~/.clawdbot/config.json` via gateway API (never direct file access from browser)
- Each page is a self-contained step — users can skip channels they don't want
- **System check page** validates: Node.js version, LM Studio running, GPU detected, available VRAM, disk space
- Mobile-responsive so you can finish setup from your phone after initial pairing

**Flow:**
```
Welcome → System Check → Model Setup → Channel Picker
  → [WhatsApp QR] → [Email Config] → [Voice Test]
  → Security Settings → Mobile Pairing → Review → Launch
```

---

### 2.3 Voice System (TTS + STT + Real-time)

**What:** Full voice pipeline — text-to-speech, speech-to-text, and real-time bidirectional voice conversations.

**Why:** User explicitly wants voice support including real-time voice talks.

**Components:**

```
src/voice/
├── index.ts              # Voice system orchestrator
├── tts/
│   ├── engine.ts         # TTS engine interface
│   ├── local-piper.ts    # Piper TTS (fully local, fast)
│   ├── local-kokoro.ts   # Kokoro TTS (high quality local)
│   ├── edge-tts.ts       # Existing Edge TTS (fallback, needs internet)
│   └── voice-registry.ts # Available voices & language support
├── stt/
│   ├── engine.ts         # STT engine interface
│   ├── local-whisper.ts  # whisper.cpp / faster-whisper (fully local)
│   ├── vad.ts            # Voice Activity Detection (Silero VAD)
│   └── stream-stt.ts     # Streaming STT for real-time mode
├── realtime/
│   ├── session.ts        # Real-time voice session manager
│   ├── duplex-stream.ts  # Full-duplex audio stream handling
│   ├── turn-detection.ts # End-of-turn detection
│   └── protocol.ts       # WebSocket audio streaming protocol
└── __tests__/
```

**Key design decisions:**

| Layer | Local Option | Why |
|-------|-------------|-----|
| **TTS** | **Piper TTS** (primary) | Runs on CPU, <50ms latency, 40+ languages, MIT licensed |
| **TTS** | **Kokoro TTS** (high quality) | Better prosody, needs GPU, Apache 2.0 |
| **TTS** | Edge TTS (fallback) | Already integrated, needs internet |
| **STT** | **whisper.cpp** / **faster-whisper** | Local Whisper inference, no cloud, great accuracy |
| **VAD** | **Silero VAD** | Lightweight voice activity detection, runs on CPU |
| **Real-time** | WebSocket audio streaming | Full-duplex via gateway, works with mobile apps |

**Real-time voice conversation flow:**
```
Mobile App / Web UI
  → WebSocket audio stream (opus/pcm)
  → VAD (detect speech boundaries)
  → STT (whisper.cpp transcription)
  → LM Studio (generate response)
  → TTS (Piper/Kokoro synthesis)
  → WebSocket audio stream back
```

**Integration points:**
- iOS app already has `VoiceWakeMode` and `TalkMode` — extend with local STT/TTS
- Android app already has `TalkMode` — extend similarly
- WhatsApp voice messages: receive audio → STT → process → TTS → send audio reply
- Web UI: WebRTC or WebSocket audio for browser-based voice chat

---

### 2.4 Email Channel

**What:** Full email integration as a first-class channel (not just a skill).

**Why:** User explicitly wants email support alongside WhatsApp and voice.

**Components:**

```
src/email/
├── index.ts              # Channel registration
├── imap-client.ts        # IMAP listener (incoming mail)
├── smtp-client.ts        # SMTP sender (outgoing mail)
├── parser.ts             # Email body parsing (HTML→text, attachments)
├── composer.ts           # Email response formatting
├── thread-tracker.ts     # Track conversation threads via Message-ID/References
├── security.ts           # SPF/DKIM verification, phishing detection
├── config.schema.ts      # Zod schema for email config
└── __tests__/
```

**Key design decisions:**
- Use **IMAP IDLE** for real-time email push (no polling)
- SMTP for sending replies
- Thread tracking via standard email headers (`In-Reply-To`, `References`, `Message-ID`)
- Parse HTML emails to plain text for LLM processing
- Attachment handling via existing media pipeline
- **Security:** SPF/DKIM verification on incoming, DKIM signing on outgoing
- Allowlist filtering — only process emails from approved senders
- Rate limiting to prevent abuse

**Config schema:**
```typescript
email: {
  imap: { host, port, user, password, tls: true },
  smtp: { host, port, user, password, tls: true },
  allowedSenders: ["*@yourdomain.com"],
  autoReply: true,
  signature: "— Sent by your personal AI assistant",
}
```

---

### 2.5 Mobile App Enhancements

**What:** Upgrade existing iOS and Android apps for the local-first model.

**Why:** User wants proper mobile app integration.

**Existing foundation:**
- iOS: SwiftUI, Bonjour pairing, voice wake, talk mode, camera
- Android: Kotlin, talk mode, camera, canvas

**New additions:**

```
apps/shared/ClawdbotKit/
├── LocalVoiceSession.swift     # On-device STT/TTS for offline use
├── LMStudioStatus.swift        # Show model status (loaded, VRAM, etc.)
└── OnboardingFlow.swift        # Native onboarding that mirrors web UI

apps/ios/Clawdbot/
├── Views/
│   ├── OnboardingView.swift    # Native iOS onboarding
│   ├── VoiceCallView.swift     # Real-time voice conversation UI
│   ├── ModelStatusView.swift   # LM Studio health dashboard
│   └── ChannelStatusView.swift # WhatsApp/Email/Voice status
├── Services/
│   ├── LocalSTT.swift          # On-device Speech framework STT
│   ├── LocalTTS.swift          # On-device AVSpeechSynthesizer TTS
│   └── AudioStreamManager.swift # WebSocket audio streaming
└── ...

apps/android/app/src/main/java/.../
├── ui/
│   ├── OnboardingActivity.kt   # Native Android onboarding
│   ├── VoiceCallFragment.kt    # Real-time voice UI
│   └── StatusDashboard.kt      # System status view
├── services/
│   ├── LocalSTTService.kt      # Android SpeechRecognizer
│   ├── LocalTTSService.kt      # Android TextToSpeech
│   └── AudioStreamService.kt   # WebSocket audio streaming
└── ...
```

**Key design decisions:**
- **Bonjour/mDNS** discovery for zero-config LAN pairing (already exists)
- **Tailscale** for secure remote access when not on local network
- Mobile apps connect to gateway via WebSocket (existing pattern)
- On-device STT/TTS for offline fallback (iOS Speech framework, Android SpeechRecognizer)
- Primary STT/TTS runs server-side (whisper.cpp + Piper) for better quality
- Push notifications via gateway webhook → APNs/FCM
- Show real-time model status (loaded model, VRAM usage, inference speed)

---

### 2.6 Security Hardening for Local-Only Mode

**What:** Ensure the entire system operates securely without any cloud dependency.

**Components to harden:**

```
src/security/
├── local-only-guard.ts    # Block all outbound LLM API calls
├── network-policy.ts      # Enforce localhost-only for LM Studio
├── tls-local.ts           # Self-signed TLS for gateway ↔ app communication
├── vault.ts               # Encrypted local credential storage
└── audit-log.ts           # Local audit trail for all bot actions
```

**Security architecture:**

| Layer | Measure |
|-------|---------|
| **Network** | LM Studio API bound to `127.0.0.1` only — never exposed externally |
| **Network** | Gateway binds to LAN only (or Tailscale for remote) |
| **Auth** | LM Studio API token required for all requests |
| **Auth** | Gateway requires pairing code for new devices |
| **Auth** | PAM authentication for local admin access (already exists) |
| **Encryption** | All credentials encrypted at rest in `~/.clawdbot/vault/` |
| **Encryption** | TLS for all gateway ↔ mobile app communication |
| **Sandbox** | Tool execution in Docker sandbox (already exists) |
| **Allowlist** | DM pairing for WhatsApp (already exists) |
| **Allowlist** | Email sender allowlist |
| **Audit** | Local audit log of all actions, queries, and tool executions |
| **Data** | No telemetry, no cloud analytics, no external data sharing |
| **Data** | Conversation history encrypted at rest |
| **Updates** | Model files verified via SHA256 checksums |

---

## 3. Full Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        YOUR MACHINE                             │
│                                                                 │
│  ┌──────────────┐     ┌──────────────────────────────────────┐  │
│  │  LM Studio   │     │         Clawdbot Gateway             │  │
│  │  (Headless)   │     │                                      │  │
│  │              │     │  ┌────────┐  ┌──────┐  ┌──────────┐  │  │
│  │  Local LLM   │◄───►│  │Provider│  │Router│  │ Plugins  │  │  │
│  │  (Qwen3,     │     │  │(LMStudio)│ │      │  │          │  │  │
│  │   DeepSeek,  │     │  └────────┘  └──┬───┘  └──────────┘  │  │
│  │   Gemma3...) │     │                 │                     │  │
│  │              │     │  ┌──────────────┼──────────────────┐  │  │
│  │  API:1234    │     │  │         Channels                │  │  │
│  └──────────────┘     │  │                                 │  │  │
│                       │  │ ┌─────────┐ ┌───────┐ ┌──────┐ │  │  │
│  ┌──────────────┐     │  │ │WhatsApp │ │ Email │ │Voice │ │  │  │
│  │  Voice Engine │     │  │ │(Baileys)│ │(IMAP/ │ │(Piper│ │  │  │
│  │              │     │  │ │         │ │ SMTP) │ │Whisper)│ │  │
│  │ Piper TTS   │◄───►│  │ └─────────┘ └───────┘ └──────┘ │  │  │
│  │ whisper.cpp  │     │  └─────────────────────────────────┘  │  │
│  │ Silero VAD   │     │                                      │  │
│  └──────────────┘     │  ┌──────────┐  ┌──────────────────┐  │  │
│                       │  │ Security │  │   Onboarding UI  │  │  │
│                       │  │ Vault,   │  │   (React/Vite)   │  │  │
│                       │  │ Audit,   │  │   localhost:port  │  │  │
│                       │  │ Sandbox  │  │   /onboard        │  │  │
│                       │  └──────────┘  └──────────────────┘  │  │
│                       └──────────────────────────────────────┘  │
│                              ▲              ▲                    │
│                              │ WebSocket    │ WebSocket          │
│                              │ (TLS)        │ (TLS)              │
└──────────────────────────────┼──────────────┼───────────────────┘
                               │              │
                    ┌──────────┴──┐    ┌──────┴──────┐
                    │  iOS App    │    │ Android App │
                    │  (SwiftUI)  │    │  (Kotlin)   │
                    │             │    │             │
                    │ Voice Chat  │    │ Voice Chat  │
                    │ Status      │    │ Status      │
                    │ Onboarding  │    │ Onboarding  │
                    └─────────────┘    └─────────────┘
```

---

## 4. Technology Stack Summary

| Layer | Technology | Local? | Notes |
|-------|-----------|--------|-------|
| **LLM Inference** | LM Studio 0.3.x+ (headless) | ✅ 100% | OpenAI-compatible API on localhost:1234 |
| **Models** | Qwen3, DeepSeek, Gemma3, Llama | ✅ 100% | User selects during onboarding |
| **Runtime** | Node.js 22+ / TypeScript | ✅ | Existing stack |
| **Gateway** | Express + WebSocket | ✅ | Existing control plane |
| **WhatsApp** | Baileys (WhatsApp Web protocol) | ✅ | Already built |
| **Email** | IMAP IDLE + SMTP | ✅ | New channel |
| **TTS** | Piper TTS (primary) | ✅ 100% | CPU, <50ms, 40+ languages |
| **TTS fallback** | Kokoro TTS | ✅ 100% | GPU, better quality |
| **STT** | whisper.cpp / faster-whisper | ✅ 100% | Local Whisper inference |
| **VAD** | Silero VAD | ✅ 100% | Voice activity detection |
| **Onboarding UI** | React + Vite | ✅ | Served from gateway |
| **iOS App** | SwiftUI | ✅ | Existing + enhancements |
| **Android App** | Kotlin | ✅ | Existing + enhancements |
| **Security** | Encrypted vault, DM pairing, sandbox | ✅ | Existing + hardened |
| **Database** | SQLite + sqlite-vec | ✅ 100% | Vector memory, sessions |
| **Remote Access** | Tailscale (optional) | ✅ | Encrypted tunnel, no port forwarding |

---

## 5. Implementation Phases

### Phase 1 — LM Studio Provider + Local Security (Foundation)

**Goal:** Bot works with local model, no cloud dependency.

1. Implement `src/providers/lmstudio/` provider module
2. OpenAI-compatible client connecting to `localhost:1234`
3. Health monitoring and model lifecycle management via `lms` CLI
4. Systemd service template for headless LM Studio
5. `local-only-guard.ts` — block outbound LLM API calls
6. Encrypted credential vault
7. Configuration schema additions

**Deliverable:** `clawdbot gateway` starts and uses LM Studio for all inference.

### Phase 2 — Onboarding Web UI

**Goal:** Non-technical users can set up the bot visually.

1. Scaffold `ui/onboarding/` React+Vite app
2. System check page (Node, LM Studio, GPU, RAM, disk)
3. Model selection page (browse available models, download)
4. Channel picker with toggle cards
5. WhatsApp QR scan flow
6. Security settings page
7. Mobile pairing QR page
8. Review & launch page
9. Gateway serves onboarding UI at `/onboard`
10. Write config via gateway API

**Deliverable:** Open browser → step through wizard → bot is configured and running.

### Phase 3 — Voice System

**Goal:** Full voice: TTS, STT, real-time conversations.

1. Integrate Piper TTS as primary local TTS engine
2. Integrate whisper.cpp as primary local STT engine
3. Implement Silero VAD for speech boundary detection
4. WebSocket audio streaming protocol
5. Real-time voice session manager (full-duplex)
6. Turn detection for natural conversation flow
7. WhatsApp voice message support (receive audio → STT → respond → TTS → send)
8. Wire into iOS/Android talk mode

**Deliverable:** Voice conversations via mobile app and WhatsApp voice messages.

### Phase 4 — Email Channel

**Goal:** Bot processes and responds to emails.

1. IMAP IDLE listener for incoming email
2. SMTP client for sending replies
3. Email parser (HTML→text, attachment extraction)
4. Thread tracking via email headers
5. Sender allowlist and SPF/DKIM verification
6. Register as first-class channel in channel registry
7. Onboarding UI email setup page

**Deliverable:** Bot receives email, processes with local LLM, sends reply.

### Phase 5 — Mobile App Enhancements

**Goal:** Native apps are full-featured companions.

1. Native onboarding flow (mirrors web UI)
2. Real-time voice conversation UI
3. LM Studio status dashboard (model, VRAM, speed)
4. Channel status view (WhatsApp connected, email active, etc.)
5. WebSocket audio streaming for voice
6. Push notification support via APNs/FCM
7. On-device STT/TTS fallback for offline use

**Deliverable:** Full-featured iOS and Android apps paired to local gateway.

---

## 6. Model Recommendations for Local Use

| Use Case | Recommended Model | VRAM | Notes |
|----------|------------------|------|-------|
| **General assistant** | Qwen3-8B | ~6GB | Best balance of quality and speed |
| **Advanced reasoning** | DeepSeek-R1-14B | ~10GB | Stronger reasoning, slower |
| **Fast responses** | Qwen3-1.7B (draft) | ~2GB | Speculative decoding draft model |
| **Coding** | Qwen2.5-Coder-7B | ~5GB | Specialized for code |
| **Low VRAM** | Gemma3-4B | ~3GB | Good quality at small size |
| **High quality** | Llama-3.3-70B (Q4) | ~40GB | Needs high-end GPU |

Users select their model during onboarding based on their hardware.

---

## 7. Prerequisites & System Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| **RAM** | 16GB | 32GB+ |
| **VRAM** | 4GB (GTX 1060 / M1) | 8GB+ (RTX 3070 / M1 Pro) |
| **Disk** | 20GB free | 50GB+ (for multiple models) |
| **OS** | macOS 13+, Ubuntu 22.04+, Win10 WSL2 | macOS 14+, Ubuntu 24.04 |
| **Node.js** | 22.12.0 | Latest LTS |
| **LM Studio** | 0.3.25+ | Latest (0.3.29+ / 0.4.x when available) |

---

## 8. What Stays Cloud-Based (by necessity)

These components inherently require external connectivity — they are not LLM calls:

| Component | Why it needs network | Mitigation |
|-----------|---------------------|------------|
| **WhatsApp** (Baileys) | WhatsApp Web protocol connects to WhatsApp servers | Encrypted E2E by WhatsApp |
| **Email** (IMAP/SMTP) | Email is a networked protocol | TLS, DKIM, SPF |
| **Tailscale** (optional) | Remote access tunnel | Zero-trust, encrypted |
| **Edge TTS** (fallback) | Microsoft's TTS API | Optional; Piper is default |
| **Push notifications** | APNs/FCM | Minimal metadata, no content |

**All LLM inference, voice processing, and data storage remain 100% local.**

---

## 9. File Structure Overview (new/modified)

```
clawdbot/
├── src/
│   ├── providers/lmstudio/        # NEW — LM Studio provider
│   ├── voice/                     # NEW — Voice system (TTS+STT+realtime)
│   ├── email/                     # NEW — Email channel
│   ├── security/
│   │   ├── local-only-guard.ts    # NEW — Block cloud LLM calls
│   │   ├── vault.ts               # NEW — Encrypted credential storage
│   │   └── audit-log.ts           # NEW — Local audit trail
│   ├── wizard/                    # MODIFIED — Delegate to web UI
│   └── gateway/                   # MODIFIED — Serve onboarding UI
├── ui/
│   └── onboarding/                # NEW — React onboarding wizard
├── apps/
│   ├── ios/                       # MODIFIED — Voice, status, onboarding
│   └── android/                   # MODIFIED — Voice, status, onboarding
├── config/
│   └── lmstudio.service           # NEW — Systemd unit template
└── PLAN-secure-local-bot.md       # This document
```
