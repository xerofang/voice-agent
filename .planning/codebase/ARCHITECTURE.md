# Architecture

**Analysis Date:** 2026-02-17

## Pattern Overview

**Overall:** Distributed two-tier voice agent system with separated concerns between web/API layer and voice processing layer.

**Key Characteristics:**
- Web server and agent worker run as independent processes/containers
- Configuration-driven agent behavior from external N8N webhooks or defaults
- AsyncIO-based concurrency throughout (FastAPI + LiveKit agents framework)
- Language/voice parameterization for multi-regional support (Indian languages focus)
- Stateless design with minimal inter-component dependencies

## Layers

**Presentation Layer (Web UI & REST API):**
- Purpose: Browser-based testing interface and token management
- Location: `main.py` (lines 236-516)
- Contains: FastAPI application, embedded HTML UI, API endpoints
- Depends on: LiveKit SDK for token generation, N8N webhooks for configuration
- Used by: End users (browser) and agent workers (API calls)

**API/Orchestration Layer:**
- Purpose: Token generation, configuration management, cache invalidation
- Location: `main.py` (lines 141-201)
- Contains: REST endpoints for `/api/token`, `/api/config/{agent_id}`, `/api/invalidate-cache`, `/api/languages`
- Depends on: FastAPI, LiveKit API, ConfigManager
- Used by: Web UI frontend, external agent workers

**Configuration Management Layer:**
- Purpose: Fetch and cache agent profiles from N8N or use defaults
- Location: `main.py` (lines 48-124)
- Contains: ConfigManager class with caching, N8N webhook integration, default profile creation
- Depends on: httpx for async HTTP, pydantic for validation
- Used by: Main FastAPI app and agent worker via API

**Voice Processing Layer:**
- Purpose: Actual voice conversation handling (STT, LLM, TTS pipeline)
- Location: `agent_worker.py` (lines 89-169)
- Contains: Agent session setup, LiveKit connections, Sarvam AI integration, LLM selection
- Depends on: LiveKit agents SDK, Sarvam plugins (STT/TTS), OpenAI-compatible LLM (Groq or OpenAI)
- Used by: LiveKit rooms (called when participant joins)

**Voice Agent Logic Layer:**
- Purpose: Custom agent behavior and responses
- Location: `agent_worker.py` (lines 76-88)
- Contains: LeadNurtureAgent class with greeting logic
- Depends on: LiveKit Agent base class, configuration
- Used by: Voice processing layer via AgentSession

## Data Flow

**Token Generation Flow:**

1. Browser UI calls `/api/token` with agentId, language, voice, userName
2. FastAPI endpoint generates LiveKit VideoGrant
3. AccessToken created with identity and grants
4. JWT token returned to browser
5. Browser uses token to connect to LiveKit room

**Configuration Fetch Flow:**

1. Agent worker starts with agent_id from room name or metadata
2. Calls `get_agent_config()` which makes HTTP GET to `/api/config/{agent_id}`
3. Web server ConfigManager checks cache, then queries N8N if configured
4. If N8N unavailable or disabled, returns default AgentProfile
5. Profile cached in ConfigManager._cache for subsequent calls
6. Agent worker receives config and applies language, voice, system_prompt

**Voice Call Flow:**

1. Browser connects to LiveKit room with generated token
2. Agent worker's JobContext triggered via LiveKit event
3. Agent extracts agent_id from room name (format: `test-{agent_id}-{timestamp}`)
4. Fetches configuration for that agent_id
5. Initializes Sarvam STT (speech-to-text)
6. Initializes Sarvam TTS (text-to-speech)
7. Initializes LLM (Groq or OpenAI)
8. Creates AgentSession with all components
9. Agent says greeting from config
10. User speaks → STT transcribes → LLM generates response → TTS speaks
11. Session continues until room disconnects

**State Management:**
- Configuration caching in `ConfigManager._cache` (dict-based, in-memory)
- No persistent state between sessions
- Room metadata used to pass agent_id information
- Room name encoding pattern: `test-{agent_id}-{timestamp}`

## Key Abstractions

**AgentProfile:**
- Purpose: Type-safe configuration container
- Examples: `main.py` (lines 36-46)
- Pattern: Pydantic BaseModel with defaults, validates incoming config data

**ConfigManager:**
- Purpose: Centralized configuration resolution with caching and N8N integration
- Examples: `main.py` (lines 48-124)
- Pattern: Singleton-like manager with async fetch logic, fallback to defaults

**LeadNurtureAgent:**
- Purpose: Custom agent subclass for greeting and lead nurturing behavior
- Examples: `agent_worker.py` (lines 76-88)
- Pattern: Extends LiveKit Agent base with on_enter() lifecycle hook

**AgentSession:**
- Purpose: Orchestrates STT→LLM→TTS pipeline
- Examples: `agent_worker.py` (lines 156-163)
- Pattern: LiveKit framework component that wires conversational AI components

## Entry Points

**Web Server Entry Point:**
- Location: `main.py` (line 522-525)
- Triggers: Manual execution or Docker container start
- Responsibilities: Initialize FastAPI app, start Uvicorn on port 3000, serve UI and APIs

**Agent Worker Entry Point:**
- Location: `agent_worker.py` (lines 174-185)
- Triggers: Manual execution via `python agent_worker.py start` or Docker container start
- Responsibilities: Initialize LiveKit worker, register agent_entrypoint callback, listen for room events

**Web Server Entry Point (UI):**
- Location: `main.py` (lines 513-516)
- Triggers: HTTP GET to `/`
- Responsibilities: Serve embedded HTML with LiveKit client JavaScript

## Error Handling

**Strategy:** Graceful degradation with logging and fallbacks

**Patterns:**

- **Configuration Fetch Errors** (`main.py`, lines 103-117): Catch N8N HTTP exceptions, log warning, return default profile
- **Token Generation Errors** (`main.py`, lines 185-189): Catch all exceptions, log error, raise HTTPException with detail
- **Agent Config Fetch Errors** (`agent_worker.py`, lines 40-48): Catch HTTP errors, log warning, return hardcoded default
- **Invalid Speaker** (`agent_worker.py`, lines 126-130): Validate against allowed speakers list, log warning, fallback to "arya"
- **Room Metadata Parsing** (`agent_worker.py`, lines 95-100): Try/except around JSON parsing, gracefully ignore unparseable metadata

## Cross-Cutting Concerns

**Logging:**
- Tool: loguru (`from loguru import logger`)
- Usage: Structured logging with context (agent IDs, operation names, error details)
- Locations: Main.py (lines 59, 112, 115, 188), agent_worker.py (lines 91, 109, 129, 168, 175-182)

**Validation:**
- Tool: Pydantic (BaseModel)
- Pattern: Type hints with defaults on AgentProfile model
- Locations: `main.py` (lines 36-46)

**Authentication:**
- Pattern: LiveKit tokens with VideoGrants for room/publish/subscribe access
- Generation: OpenAI-compatible API key + secret
- Locations: `main.py` (lines 164-177)

**Multi-language Support:**
- Pattern: Language codes passed through configuration (e.g., "hi-IN", "ta-IN")
- Validation: Language code passed to Sarvam STT/TTS
- Auto-detection: "unknown" language code triggers auto-detection in Sarvam
- Locations: `main.py` (lines 40, 152, 206-219), `agent_worker.py` (lines 118-122, 135)

**Async/Concurrent Operations:**
- Pattern: AsyncIO throughout with async/await
- HTTP Clients: httpx.AsyncClient with timeouts
- API Framework: FastAPI (async-first)
- Agent Framework: LiveKit agents (async sessions)
- Locations: `main.py` (lines 99, 105-107), `agent_worker.py` (lines 39, 42-44)

---

*Architecture analysis: 2026-02-17*
