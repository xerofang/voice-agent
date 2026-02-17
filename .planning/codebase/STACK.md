# Technology Stack

**Analysis Date:** 2026-02-17

## Languages

**Primary:**
- Python 3.11 - Voice agent backend, web server, worker process

## Runtime

**Environment:**
- Python 3.11-slim (Docker base image)

**Package Manager:**
- pip
- Lockfile: `requirements.txt` (present)

## Frameworks

**Core:**
- FastAPI 0.109.0+ - REST API and web server (`main.py`)
- Uvicorn 0.27.0+ - ASGI application server

**Voice Processing:**
- LiveKit Agents 1.0.0+ - Agent framework for voice call handling (`agent_worker.py`)
- Sarvam AI Plugin 1.0.0+ - STT (Speech-to-Text) and TTS (Text-to-Speech) for Indian languages
- OpenAI Plugin 1.0.0+ - LLM integration (via LiveKit's plugin)

**Testing/Utilities:**
- Python-dotenv 1.0.0+ - Environment configuration
- Pydantic 2.0.0+ - Data validation and settings management
- Pydantic-settings 2.0.0+ - Environment variable configuration

## Key Dependencies

**Critical:**
- livekit-agents>=1.0.0 - Core agent framework for voice processing
- livekit-plugins-sarvam>=1.0.0 - STT/TTS for Indian languages (Hindi, Tamil, Telugu, Bengali, Marathi, Gujarati, Kannada, Malayalam, Punjabi, Odia)
- livekit-plugins-openai>=1.0.0 - LLM provider integration (supports Groq and OpenAI)
- livekit-api>=0.6.0 - Server SDK for token generation and room management
- fastapi>=0.109.0 - Web framework for UI and APIs
- uvicorn>=0.27.0 - Application server

**HTTP/Networking:**
- httpx>=0.25.0 - Async HTTP client for external API calls (N8N webhooks)
- aiohttp>=3.9.0 - Async HTTP support

**Infrastructure:**
- redis>=5.0.0 - State management (declared in requirements but not actively used in current implementation)
- aiofiles>=23.0.0 - Async file operations for static files

**Utilities:**
- loguru>=0.7.0 - Structured logging
- tenacity>=8.2.0 - Retry logic for API calls

## Configuration

**Environment:**
- Environment variables via `.env` file (python-dotenv)
- Required variables:
  - LiveKit: `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`, `LIVEKIT_URL`
  - LLM: `GROQ_API_KEY` or `OPENAI_API_KEY`, `LLM_PROVIDER`, `LLM_MODEL`
  - Voice: `SARVAM_API_KEY`
  - Defaults: `DEFAULT_LANGUAGE`, `DEFAULT_VOICE`
  - N8N: `N8N_BASE_URL`, `N8N_WEBHOOK_AGENT_CONFIG`, `N8N_WEBHOOK_LEAD_CAPTURE`
  - Server: `WEB_SERVER_URL` (for agent worker communicating with web server)

**Build:**
- `Dockerfile` - Web server container (main.py)
- `Dockerfile.worker` - Agent worker container (agent_worker.py)
- `docker-compose.yml` - Multi-container orchestration for both components

## Platform Requirements

**Development:**
- Python 3.11+
- Docker (optional, for containerized deployment)
- System libraries: gcc, libffi-dev for Python package compilation

**Production:**
- Deployment target: Coolify (recommended), Docker, or standard VPS
- Web server: Uvicorn on port 3000
- Agent worker: Runs as separate Python process (can use screen/tmux)
- External cloud services required: LiveKit Cloud (WebRTC relay)

## Architecture Components

**Web Server (`main.py`):**
- FastAPI application running on port 3000
- Serves embedded HTML UI for browser-based testing
- Provides REST APIs for token generation, configuration, and N8N webhooks
- Implements configuration caching for agent profiles

**Agent Worker (`agent_worker.py`):**
- LiveKit agent worker that processes incoming voice calls
- Uses Sarvam AI for STT/TTS in Indian languages
- Integrates with Groq or OpenAI LLM
- Communicates with web server for agent configuration
- Runs as separate process for scalability

**Frontend:**
- Embedded HTML/JavaScript UI in `main.py` (no separate frontend build)
- Uses TailwindCSS CDN and LiveKit JavaScript client library
- Browser-based microphone access via WebRTC

---

*Stack analysis: 2026-02-17*
