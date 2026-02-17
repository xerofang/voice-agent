# Codebase Structure

**Analysis Date:** 2026-02-17

## Directory Layout

```
voice-agent-simple/
├── main.py                              # Web server, UI, API (FastAPI)
├── agent_worker.py                      # Voice agent worker (LiveKit)
├── requirements.txt                     # Python dependencies
├── Dockerfile                           # Multi-service deployment image
├── Dockerfile.worker                    # Agent worker only (deprecated)
├── docker-compose.yml                   # Local development setup
├── start.sh                             # Docker entrypoint script
├── Caddyfile                            # Reverse proxy config (unused)
├── .env.example                         # Environment template
├── README.md                            # Deployment guide
├── n8n-workflow-agent-profiles.json     # N8N workflow: Profile routing
├── n8n-workflow-lead-capture.json       # N8N workflow: Lead capture/storage
└── .planning/                           # GSD planning artifacts
    └── codebase/                        # Analysis documents
        ├── ARCHITECTURE.md
        └── STRUCTURE.md
```

## Directory Purposes

**Root:**
- Purpose: Project root with configuration, scripts, and workflow definitions
- Contains: Entry point scripts, Docker configs, environment templates
- Key files: `main.py`, `agent_worker.py`, `docker-compose.yml`

**.planning/codebase:**
- Purpose: GSD-generated architecture and structure analysis documents
- Contains: ARCHITECTURE.md, STRUCTURE.md, CONVENTIONS.md, TESTING.md, CONCERNS.md
- Generated: Yes (via GSD commands)
- Committed: Yes (for team reference)

## Key File Locations

**Entry Points:**

**Web Server & UI:**
- `main.py`: FastAPI application server
  - Line 1-11: Module docstring explaining purpose
  - Line 522-525: Main execution block with uvicorn.run()
  - Serves on port 3000 (configurable via WEB_PORT env var)

**Voice Agent Worker:**
- `agent_worker.py`: LiveKit agent process
  - Line 1-10: Module docstring explaining purpose
  - Line 174-185: Main execution block with cli.run_app()
  - Requires command: `python agent_worker.py start`

**Configuration:**

**Environment:**
- `.env.example`: Template for all environment variables
  - Section 1: Sarvam AI credentials
  - Section 2: LiveKit credentials
  - Section 3: LLM provider selection (Groq or OpenAI)
  - Section 4: N8N optional integration
  - Section 5: Language/voice defaults

**Docker:**
- `Dockerfile`: Multi-service image (both web + agent)
- `Dockerfile.worker`: Agent worker only (deprecated, not in use)
- `docker-compose.yml`: Local dev orchestration
- `start.sh`: Bash script to start both services

**Reverse Proxy:**
- `Caddyfile`: Caddy reverse proxy config (present but not currently used)

**Core Logic:**

**Web Server Layer:**
- `main.py` (lines 36-46): AgentProfile Pydantic model
- `main.py` (lines 48-124): ConfigManager class (configuration, caching, N8N integration)
- `main.py` (lines 131-201): FastAPI routes (health, token generation, config API, languages)
- `main.py` (lines 236-511): Embedded HTML UI with LiveKit JavaScript client
- `main.py` (lines 513-516): Route handler for `/` UI endpoint

**Voice Agent Layer:**
- `agent_worker.py` (lines 39-70): get_agent_config() async function (fetches from web server API)
- `agent_worker.py` (lines 76-88): LeadNurtureAgent class (custom agent with greeting)
- `agent_worker.py` (lines 89-169): agent_entrypoint() async function (LiveKit job entry point)
  - Lines 91-109: Agent ID extraction from room metadata/name
  - Lines 112: Config fetch
  - Lines 114-115: Room connection with AutoSubscribe
  - Lines 117-136: Sarvam STT/TTS initialization
  - Lines 138-150: LLM provider selection
  - Lines 152-163: AgentSession creation and start

**Testing:**
- No test files present in repository

**External Integrations:**
- `n8n-workflow-agent-profiles.json`: N8N workflow for profile routing based on agent_id
- `n8n-workflow-lead-capture.json`: N8N workflow for capturing/storing leads

## Naming Conventions

**Files:**

**Python Files:**
- Pattern: `lowercase_with_underscores.py`
- Examples: `main.py`, `agent_worker.py`, `requirements.txt`

**Configuration Files:**
- Pattern: Dockerfile variants and dotenv files
- Examples: `Dockerfile`, `Dockerfile.worker`, `.env.example`, `Caddyfile`

**Workflow Files:**
- Pattern: `n8n-workflow-{purpose}.json`
- Examples: `n8n-workflow-agent-profiles.json`, `n8n-workflow-lead-capture.json`

**Directories:**
- Pattern: lowercase with hyphens for multi-word
- Examples: `.planning`, `codebase`

**Python Classes:**

- Pattern: PascalCase
- Examples:
  - `AgentProfile` (Pydantic model for configuration)
  - `ConfigManager` (Configuration manager with caching)
  - `LeadNurtureAgent` (Agent subclass)

**Python Functions:**

- Pattern: snake_case for regular functions, async functions also snake_case
- Examples:
  - `get_agent_config()` (async configuration fetch)
  - `get_profile()` (async profile retrieval with caching)
  - `health()` (health check endpoint)
  - `generate_token()` (token generation endpoint)
  - `startCall()` (JavaScript frontend function, camelCase)

**Python Variables & Methods:**

- Pattern: snake_case for module-level and instance variables
- Examples:
  - `self.n8n_base` (instance variable)
  - `self._cache` (private cache dict)
  - `self._default` (private default profile)
  - `app` (FastAPI instance)
  - `config_manager` (module-level singleton)

**Environment Variables:**

- Pattern: UPPERCASE_WITH_UNDERSCORES
- Examples:
  - `SARVAM_API_KEY`
  - `LIVEKIT_API_KEY`
  - `DEFAULT_LANGUAGE`
  - `N8N_BASE_URL`
  - `WEB_SERVER_URL`

**Room Names:**

- Pattern: `test-{agent_id}-{unix_timestamp}`
- Examples: `test-default-1692345600`, `test-real-estate-hindi-1692345601`

**Agent IDs:**

- Pattern: kebab-case for custom agents, "default" for fallback
- Examples: `default`, `real-estate-hindi`, `real-estate-tamil`, `real-estate-telugu`

## Where to Add New Code

**New Feature (Core Voice Logic):**
- Primary code: `agent_worker.py` - Add methods to LeadNurtureAgent class or agent_entrypoint()
- Configuration: `.env.example` and `docker-compose.yml` - Add new env vars if needed
- Integration: N8N workflow files if external coordination needed

**New API Endpoint (Web Server):**
- Implementation: `main.py` - Add @app.post() or @app.get() routes alongside existing endpoints (lines 141-230)
- Models: `main.py` - Add Pydantic models if accepting new request types (lines 36-46)
- Configuration: Update `.env.example` if new env vars needed

**New Agent Profile Type:**
- Definition: Extend AgentProfile Pydantic model in `main.py` (lines 36-46)
- N8N Routing: Add new case to `n8n-workflow-agent-profiles.json`
- Defaults: Update `_create_default()` method in ConfigManager (lines 63-97)

**New Language Support:**
- STT Config: Update Sarvam STT language in `agent_worker.py` (line 120)
- TTS Config: Update Sarvam TTS target_language_code in `agent_worker.py` (line 135)
- UI Options: Add language option in embedded HTML (main.py, lines 287-296)
- Defaults: Update `.env.example` DEFAULT_LANGUAGE

**New LLM Provider:**
- Implementation: Add conditional in `agent_worker.py` (lines 138-150) for new provider
- Configuration: Add env vars to `.env.example` for new provider credentials
- Session Setup: Update agent_entrypoint() with new LLM initialization

**Utilities & Helpers:**
- Shared helpers: Create in same file as usage (no separate utils/ directory currently)
- If shared across files: Could add `utils.py` module at root level

## Special Directories

**.planning:**
- Purpose: GSD-generated planning and codebase analysis documents
- Generated: Yes (via `/gsd:map-codebase`, `/gsd:plan-phase`, etc.)
- Committed: Yes (to support team planning and future Claude instances)

**.git:**
- Purpose: Git repository metadata
- Generated: Yes
- Committed: N/A (internal git data)

**.claude:**
- Purpose: Claude IDE/environment settings
- Generated: Yes (by IDE)
- Committed: Yes

## File Dependencies & Import Patterns

**main.py imports:**
- External: `os`, `json`, `datetime`, `httpx`, `dotenv`, `loguru`, `pydantic`, `fastapi`, `uvicorn`, `livekit.api`
- No internal imports (single-file monolith for web server)

**agent_worker.py imports:**
- External: `os`, `json`, `asyncio`, `datetime`, `httpx`, `dotenv`, `loguru`, `livekit.agents`, `livekit.plugins`
- No internal imports (single-file monolith for worker)

**Inter-Process Communication:**
- Web server to N8N: HTTP GET via httpx (ConfigManager)
- Agent worker to Web server: HTTP GET via httpx (get_agent_config())
- Web server to Agent worker: LiveKit WebRTC via LiveKit cloud

## Configuration File Structure

**requirements.txt:**
- Format: pip package specification (name[>=version])
- Sections: LiveKit Agents, HTTP clients, Environment/Config, Database (Redis), FastAPI, Static files, Utilities
- Line count: 32 (with comments and blank lines)

**docker-compose.yml:**
- Format: Docker Compose v3.8 syntax
- Services: `web-server` (FastAPI), `agent-worker` (LiveKit agent)
- Environment: Shared credentials for LiveKit, separate STT/TTS/LLM keys
- Health: Web server has curl-based health check

**Dockerfile:**
- Base: python:3.11-slim
- Dependencies: gcc, libffi-dev, curl (system packages)
- Build: pip install from requirements.txt
- Entrypoint: `./start.sh` script
- Port: 3000
- Health check: curl to /health endpoint

## Common Patterns

**Async/Await Throughout:**
```python
async def function_name():
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
```
- Pattern used in: ConfigManager.get_profile(), agent_entrypoint(), get_agent_config()

**Error Handling with Defaults:**
```python
try:
    result = await operation()
except Exception as e:
    logger.warning(f"Operation failed: {e}")
    return default_value
```
- Pattern used in: N8N config fetch, agent config fetch, room metadata parsing

**Environment Variable with Defaults:**
```python
value = os.getenv("ENV_VAR_NAME", "default_value")
```
- Pattern used in: All configuration loading throughout both files

**Pydantic Model with Defaults:**
```python
class Profile(BaseModel):
    field: str = "default_value"
```
- Pattern used in: AgentProfile model (lines 36-46)

---

*Structure analysis: 2026-02-17*
