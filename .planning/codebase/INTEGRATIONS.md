# External Integrations

**Analysis Date:** 2026-02-17

## APIs & External Services

**Voice Processing:**
- Sarvam AI - STT and TTS for Indian languages
  - SDK/Client: `livekit-plugins-sarvam`
  - Auth: `SARVAM_API_KEY` environment variable
  - Usage: `agent_worker.py` (lines 118-136)
  - Models: STT uses `saaras:v3`, TTS uses `bulbul:v2`
  - Supported languages: Hindi, Tamil, Telugu, Bengali, Marathi, Gujarati, Kannada, Malayalam, Punjabi, Odia
  - Supported voices: arya, abhilash, karun, hitesh (male); anushka, manisha, vidya (female)

**LLM Services (Choose ONE):**
- Groq - Fast, free-tier LLM
  - SDK/Client: `livekit-plugins-openai` (Groq uses OpenAI-compatible API)
  - Auth: `GROQ_API_KEY` environment variable
  - Endpoint: `https://api.groq.com/openai/v1`
  - Model: `llama-3.3-70b-versatile` (default)
  - Usage: `agent_worker.py` (lines 140-144)

- OpenAI - Alternative LLM provider
  - SDK/Client: `livekit-plugins-openai`
  - Auth: `OPENAI_API_KEY` environment variable
  - Model: `gpt-4o-mini` (default)
  - Usage: `agent_worker.py` (lines 147-149)
  - Selection: Set `LLM_PROVIDER=openai` environment variable

**WebRTC Relay:**
- LiveKit Cloud - Real-time communication infrastructure
  - SDK/Client: `livekit-agents`, `livekit-api`
  - Auth: `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET` environment variables
  - Endpoint: `LIVEKIT_URL` (format: `wss://your-project.livekit.cloud`)
  - Usage:
    - Token generation: `main.py` (lines 146-189)
    - Agent connection: `agent_worker.py` (lines 114-115)
  - Pricing: 1,000 free minutes/month (free tier available at https://cloud.livekit.io/)

## Data Storage

**Databases:**
- None actively used in current implementation
- Note: `redis>=5.0.0` is in `requirements.txt` but not currently utilized; available for future state management

**File Storage:**
- Local filesystem only - Static assets served via FastAPI/aiofiles
- Embedded HTML UI served from memory (not file-based)

**Caching:**
- In-memory Python dictionary - Agent profile cache in `main.py` (line 55: `self._cache: dict[str, AgentProfile]`)
- Invalidation endpoint: `POST /api/invalidate-cache` (lines 197-201)

## Authentication & Identity

**Auth Provider:**
- Custom token generation using LiveKit's JWT system
- Auth method: LiveKit access tokens (JWT-based)
- Implementation: `main.py` (lines 146-189)
  - Generates VideoGrants with room-specific permissions
  - Token includes identity, name, and video grants
  - Used for browser-based WebRTC connection validation

## Workflow Automation

**N8N Integration (Optional):**
- Base URL: `N8N_BASE_URL` environment variable
- Agent Profile Webhook: `N8N_WEBHOOK_AGENT_CONFIG` (default: `/webhook/agent-config`)
  - Method: GET
  - Usage: `main.py` (lines 103-117) - ConfigManager fetches agent profiles
  - Fallback: Uses in-memory default profile if N8N unavailable
  - Workflow: `n8n-workflow-agent-profiles.json`

- Lead Capture Webhook: `N8N_WEBHOOK_LEAD_CAPTURE` (default: `/webhook/lead-capture`)
  - Method: POST
  - Workflow: `n8n-workflow-lead-capture.json`
  - Purpose: Receive lead data from voice calls (not implemented in current codebase but configured for future use)
  - Expected format: Call metadata, transcript, call duration, lead info

## Monitoring & Observability

**Error Tracking:**
- None detected - No Sentry, Rollbar, or similar integration

**Logs:**
- Framework: `loguru>=0.7.0`
- Output: Console/stdout (can be captured by Docker/Coolify)
- Usage throughout: `main.py` and `agent_worker.py`
- Key log points:
  - Configuration loading
  - N8N fetch failures with fallback
  - Token generation
  - Agent session lifecycle

**Health Check:**
- Endpoint: `GET /health` (`main.py`, line 141-144)
- Returns: `{"status": "healthy", "timestamp": "ISO8601"}`
- Used by: Docker HEALTHCHECK, load balancers, monitoring systems

## CI/CD & Deployment

**Hosting:**
- Coolify (recommended) - Docker-based PaaS
- Docker - Container deployment
- Standard VPS - Direct Python execution

**CI Pipeline:**
- None detected in codebase

**Deployment Components:**
- Web Server: Deployed via Dockerfile as single container
- Agent Worker: Run separately via SSH/screen/tmux or Docker container
- Docker Compose: `docker-compose.yml` orchestrates both components together

## Environment Configuration

**Required env vars:**

**For Web Server (`main.py`):**
```
LIVEKIT_API_KEY=...
LIVEKIT_API_SECRET=...
LIVEKIT_URL=wss://...
DEFAULT_LANGUAGE=hi-IN
DEFAULT_VOICE=arya
WEB_PORT=3000 (optional, defaults to 3000)
N8N_BASE_URL=... (optional)
N8N_WEBHOOK_AGENT_CONFIG=... (optional)
N8N_WEBHOOK_LEAD_CAPTURE=... (optional)
```

**For Agent Worker (`agent_worker.py`):**
```
SARVAM_API_KEY=...
LIVEKIT_API_KEY=...
LIVEKIT_API_SECRET=...
LIVEKIT_URL=wss://...
GROQ_API_KEY=... (if using Groq)
OR
OPENAI_API_KEY=... (if using OpenAI)
LLM_PROVIDER=groq|openai
LLM_MODEL=llama-3.3-70b-versatile|gpt-4o-mini
DEFAULT_LANGUAGE=hi-IN
DEFAULT_VOICE=arya
WEB_SERVER_URL=http://localhost:3000
```

**Secrets location:**
- `.env` file (local development, git-ignored)
- Environment variables (Docker, Coolify, VPS)
- See `.env.example` for template

## API Communication Patterns

**Agent Configuration Fetch:**
- Agent Worker → Web Server: `GET /api/config/{agent_id}` (httpx async client)
- Web Server → N8N: `GET {N8N_BASE_URL}{N8N_WEBHOOK_AGENT_CONFIG}?agent_id={id}` (httpx async)
- Timeout: 5 seconds with fallback to defaults

**Token Generation:**
- Browser → Web Server: `POST /api/token` with agentId, language, voice, userName
- Returns: LiveKit token, room name, LiveKit URL

**Language & Voice Enumeration:**
- Browser → Web Server: `GET /api/languages`
- Returns: Supported languages and voices list

## Browser Integration

**Frontend Libraries (CDN):**
- TailwindCSS - UI styling
- LiveKit JavaScript Client (v2.0.0) - WebRTC connection management

**Browser APIs Used:**
- Microphone access (getUserMedia)
- WebRTC (via LiveKit client)
- Secure context required (HTTPS)

## Webhooks & Callbacks

**Incoming:**
- N8N agent config webhook (GET) - Fetched on-demand
- N8N lead capture webhook (POST) - Configured but not called from current code

**Outgoing:**
- None currently implemented

## Connection Flow Diagram

```
User Browser
    ↓
    └─→ POST /api/token → FastAPI (main.py)
        ↓
        └─→ Generate JWT → LiveKit SDK
        ↓
        ↑ Return token, room name, URL
    ↓
    └─→ WebRTC Connection → LiveKit Cloud
        ↓
        ↑ Room subscription (audio)
    ↓
Agent Worker (agent_worker.py)
    ↓
    ├─→ Listen to LiveKit room
    ├─→ Speech-to-Text → Sarvam AI
    ├─→ LLM Inference → Groq/OpenAI
    ├─→ Text-to-Speech → Sarvam AI
    └─→ Send audio → LiveKit Cloud → User Browser
```

---

*Integration audit: 2026-02-17*
