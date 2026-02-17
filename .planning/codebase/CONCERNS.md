# Codebase Concerns

**Analysis Date:** 2026-02-17

## Tech Debt

**Bare Exception Handling:**
- Issue: Agent metadata parsing uses bare `except:` which catches all exceptions including SystemExit and KeyboardInterrupt
- Files: `agent_worker.py:99-100`
- Impact: Silent failures in agent ID extraction; invalid agent_id="default" used without logging cause
- Fix approach: Change to specific exception types (`except (json.JSONDecodeError, KeyError, TypeError)`) and log the actual error

**HTML Template Embedded in Python:**
- Issue: 270+ lines of HTML embedded as string in main.py (lines 236-510)
- Files: `main.py:236-510`
- Impact: Difficult to maintain, test, and update UI; mixing frontend and backend concerns; hard to version control HTML separately
- Fix approach: Extract to separate HTML file or use template engine (Jinja2); serve with FastAPI StaticFiles

**Hardcoded Configuration Values:**
- Issue: Default system prompts, greetings, and qualification questions hardcoded in multiple places
- Files: `main.py:63-97`, `agent_worker.py:51-70`
- Impact: Configuration duplication creates maintenance burden; changes require code updates; profile values not easily versioned
- Fix approach: Load all defaults from config file or environment-provided JSON; create single source of truth

## Known Bugs

**Voice Speaker Mismatch:**
- Symptoms: Agent crashes with error "Speaker 'aditya' is not compatible with model 'bulbul:v2'"
- Files: `.env.example:47`, `docker-compose.yml:24`, `agent_worker.py:126-130`
- Trigger: DEFAULT_VOICE set to 'aditya' in .env.example and docker-compose.yml, but bulbul:v2 only supports [anushka, manisha, vidya, arya, abhilash, karun, hitesh]
- Workaround: Change DEFAULT_VOICE to 'arya' in environment configuration; agent_worker.py handles fallback but logs warning
- Evidence: Error visible in log file `ags084co8w08wc84gsgg88co-161750470207-all-logs-2026-02-17-16-20-41.txt:23`

**Room Name Parsing Fragility:**
- Symptoms: Agent ID extraction from room name may fail unpredictably
- Files: `agent_worker.py:103-107`
- Trigger: Room names not following "test-{agent_id}-{timestamp}" format; splitting on "-" assumes specific structure
- Workaround: Falls back to agent_id="default" silently
- Impact: Multi-tenant setups may use wrong agent profile

## Security Considerations

**CORS Configuration - Overly Permissive:**
- Risk: `allow_origins=["*"]` allows any domain to make requests; credentials allowed
- Files: `main.py:133-139`
- Current mitigation: None
- Recommendations:
  - Restrict `allow_origins` to specific domains (e.g., frontend domain, N8N instance)
  - Remove `allow_credentials=True` if not needed or restrict origins when enabled
  - Use environment variable for allowed origins list

**Sensitive Credentials in Environment:**
- Risk: API keys (Sarvam, LiveKit, Groq, OpenAI) must be passed via environment variables; if .env file is committed, secrets are exposed
- Files: `.env.example`, `docker-compose.yml`, `main.py:30`, `agent_worker.py:31`
- Current mitigation: .env.example provided as template (doesn't include actual keys)
- Recommendations:
  - Document .env in .gitignore enforcement
  - Use secret management for production (AWS Secrets Manager, HashiCorp Vault, Coolify secrets)
  - Validate required secrets on startup rather than failing silently

**No Request Validation on N8N Webhook Proxy:**
- Risk: `/api/invalidate-cache` endpoint exposed without authentication or rate limiting
- Files: `main.py:197-201`
- Current mitigation: None
- Recommendations:
  - Add API key validation for cache invalidation endpoint
  - Implement rate limiting
  - Require POST method (currently uses POST but could be made more explicit)

**No Input Sanitization:**
- Risk: User-provided `agentId`, `language`, `voice`, `userName` passed directly to token generation and room name creation
- Files: `main.py:151-156`
- Current mitigation: None (room names use timestamp so collision unlikely, but agent_id used in metadata)
- Recommendations:
  - Validate agentId format (alphanumeric, safe characters)
  - Validate language against allowed list
  - Sanitize userName for LiveKit (max 256 chars, allowed characters)

**Unauthenticated API Endpoints:**
- Risk: All endpoints (`/api/token`, `/api/config`, `/api/languages`, `/api/invalidate-cache`) are publicly accessible
- Files: `main.py:141-230`
- Current mitigation: LiveKit token already provides per-room access control
- Recommendations:
  - Add API key requirement for production deployments
  - Document that this is only suitable for internal/testing use or protected deployment
  - Add authentication middleware

## Performance Bottlenecks

**Synchronous N8N Calls in Request Path:**
- Problem: Token generation endpoint calls `/api/config/{agent_id}` which makes synchronous HTTP request to N8N during request handling
- Files: `main.py:103-117`, blocking on `httpx.AsyncClient.get()`
- Cause: 5.0 second timeout can cause slow API responses if N8N is slow
- Improvement path:
  - Implement profile caching with TTL (already has simple dict cache but no expiry)
  - Fetch profiles asynchronously in background
  - Return default config immediately if fetch exceeds timeout

**Embedded UI on Every Root Request:**
- Problem: 270+ line HTML string recreated/loaded on every `/` request
- Files: `main.py:236-516`
- Cause: String is in memory but serves same large HTML for every user
- Improvement path: Serve from static files, add caching headers (Cache-Control: public, max-age=3600)

**No Connection Pooling Configuration:**
- Problem: Each config fetch creates new `AsyncClient`
- Files: `main.py:105`, `agent_worker.py:42`
- Cause: Connection pools reset for each request
- Improvement path: Create persistent httpx.AsyncClient at module level with configured pool limits

## Fragile Areas

**Agent Profile Type Safety:**
- Files: `main.py:36-46`, `main.py:99-117`, `agent_worker.py:39-70`
- Why fragile: AgentProfile in main.py defined with defaults; N8N response may have missing/extra fields; if N8N schema changes, app silently uses defaults
- Safe modification:
  - Add strict validation with pydantic `model_validate` (not model construction with **)
  - Handle validation errors explicitly
  - Version N8N API responses
- Test coverage: No validation tests; no tests for N8N fetch failures

**LiveKit Token Generation:**
- Files: `main.py:146-189`
- Why fragile: Room name collision possible if timestamps collide (though unlikely); no validation of LiveKit credentials until token creation attempt
- Safe modification:
  - Validate credentials on startup (health check)
  - Use UUID instead of timestamp for room names
  - Add retry logic for transient failures
- Test coverage: No unit tests for token generation; no tests for missing credentials

**Agent Worker Entry Point:**
- Files: `agent_worker.py:89-168`
- Why fragile: Room name parsing assumes specific format (test-{agent_id}-{timestamp}); if LiveKit changes room naming, agent silently uses default profile
- Safe modification:
  - Pass agent_id explicitly via room metadata instead of parsing room name
  - Validate metadata before using
  - Log warnings for fallback behavior
- Test coverage: No tests; no tests for metadata parsing

**Configuration Merge Logic:**
- Files: `main.py:48-124`
- Why fragile: No deep merge of N8N config with defaults; missing fields in response silently become None or trigger validation errors
- Safe modification:
  - Implement explicit config merging with defaults
  - Validate all required fields present
  - Log which config source was used
- Test coverage: No tests for config fetching; no tests for partial N8N responses

## Scaling Limits

**Single Process Web Server:**
- Current capacity: ~50-100 concurrent users with single uvicorn worker
- Limit: CPU-bound during token generation; no load balancing
- Scaling path:
  - Use gunicorn/uvicorn with multiple workers (see Dockerfile: no worker count specified)
  - Add load balancer (Nginx, HAProxy)
  - Move config fetching to background task queue (Celery/Rq)

**In-Memory Profile Cache:**
- Current capacity: Stores profiles in `self._cache` dict with no size limit
- Limit: Memory grows unbounded with each unique agent_id requested
- Scaling path:
  - Add TTL to cache entries
  - Implement LRU eviction policy
  - Use Redis for distributed caching if multi-instance

**N8N Integration as Single Point of Failure:**
- Current capacity: 5 second timeout per fetch
- Limit: If N8N is slow/unavailable, all `/api/config` and token requests slow down
- Scaling path:
  - Implement cache warming (fetch profiles periodically)
  - Use Redis for profile caching across instances
  - Add circuit breaker pattern for N8N calls

## Dependencies at Risk

**livekit-agents >= 1.0.0 (unpinned major version):**
- Risk: Major version updates could break agent_entrypoint API, STT/TTS configuration, session handling
- Impact: Agent worker would fail to start; requires manual code updates
- Migration plan:
  - Pin to specific minor version (e.g., livekit-agents==1.x.y)
  - Test major version upgrades in staging before production
  - Monitor releases for breaking changes

**livekit-plugins-sarvam >= 1.0.0 (unpinned):**
- Risk: Voice speaker compatibility, STT language codes could change
- Impact: Agent crashes or degraded speech recognition
- Migration plan: Same as above; vendor test profile compatibility after updates

**Unpinned OpenAI/Groq plugins:**
- Risk: API compatibility, model deprecation
- Impact: LLM provider fails; agent has no inference
- Migration plan: Pin versions; monitor provider deprecation notices

**Python 3.11 Required:**
- Risk: Docker image uses python:3.11-slim; some dependencies might have compatibility issues with newer versions
- Impact: Cannot easily upgrade Python for security patches
- Migration plan: Test with Python 3.12+ periodically; update Dockerfile

## Missing Critical Features

**No Monitoring/Observability:**
- Problem: No structured logging of call metrics; no tracing of request flow through web server → agent worker
- Blocks: Cannot debug production issues; cannot identify performance problems
- Impact: High support burden for debugging failures
- Recommendation: Add OpenTelemetry instrumentation; log call metrics

**No Error Recovery:**
- Problem: If agent_worker crashes, it's not automatically restarted (relies on Docker restart policy)
- Blocks: Cannot guarantee service availability
- Recommendation: Use supervisor or systemd service file for SSH-deployed instances

**No Rate Limiting:**
- Problem: Endpoints can be flooded with requests
- Blocks: Cannot protect against DoS
- Recommendation: Add rate limiter middleware (slowapi); configure per-IP limits

**No Audit Logging:**
- Problem: No record of which agent_id, language, voice combinations are used
- Blocks: Cannot analyze usage patterns or comply with audit requirements
- Recommendation: Log all token requests with user identity

## Test Coverage Gaps

**No Tests for Web Server:**
- What's not tested:
  - Token generation endpoint with various inputs
  - N8N config fetching and caching
  - CORS handling
  - Health check endpoint
  - Language/voice list endpoint
  - Cache invalidation
- Files: `main.py` (entire file untested)
- Risk: Regressions in API behavior go undetected; N8N integration changes break silently
- Priority: **High** - Critical API endpoints untested

**No Tests for Agent Worker:**
- What's not tested:
  - Agent entry point and room metadata parsing
  - Config fetching from web server
  - Speaker/language validation
  - LLM provider fallback logic
  - Sarvam STT/TTS configuration
- Files: `agent_worker.py` (entire file untested)
- Risk: Agent worker failures only discovered in production
- Priority: **High** - Core agent logic untested

**No Integration Tests:**
- What's not tested: End-to-end call flow from browser → LiveKit → agent worker → TTS/LLM
- Risk: Component integration issues only found during manual testing
- Priority: **Medium** - Requires LiveKit test environment

**No Load/Stress Tests:**
- What's not tested: Behavior under concurrent connections; memory leaks; cache behavior at scale
- Risk: Performance issues appear in production
- Priority: **Medium** - Important for production readiness

---

*Concerns audit: 2026-02-17*
