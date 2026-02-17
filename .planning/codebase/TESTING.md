# Testing Patterns

**Analysis Date:** 2026-02-17

## Test Framework

**Runner:**
- Not detected - no test runner configuration found
- No pytest.ini, conftest.py, or tox.ini in codebase

**Assertion Library:**
- Not detected - no test files present

**Run Commands:**
```bash
# No testing infrastructure detected
# No test command configured in requirements.txt
```

## Test File Organization

**Location:**
- No test files found in codebase
- No standard test directory structure (tests/, test/, spec/)
- No co-located test files (*_test.py or *spec.py)

**Naming:**
- Not applicable - no tests present

**Structure:**
- Not applicable - no tests present

## Current Testing Status

**Test Coverage:**
- No test files detected: 0% coverage
- No test infrastructure configured
- No CI/CD test pipeline detected

**Manual Testing Approach:**
- Browser-based testing via embedded web UI (main.py lines 236-510)
- Manual LiveKit room connection testing through `/` route
- Configuration can be tested via `/api/config/{agent_id}` endpoint
- Token generation can be tested via `/api/token` POST endpoint

## Testing Opportunities

**High Priority (Missing Coverage):**

**API Endpoints:**
- `POST /api/token` - Token generation with LiveKit
  - Should test: Valid inputs, missing credentials, timeout handling
  - Location to test: `main.py` lines 147-189
  - Requires: Mock `httpx.AsyncClient`, `livekit_api.AccessToken`

- `GET /api/config/{agent_id}` - Agent configuration retrieval
  - Should test: Cache hits, cache misses, N8N fetch failures
  - Location to test: `main.py` lines 192-195
  - Requires: Mock `ConfigManager.get_profile()`

- `GET /health` - Health check
  - Should test: Response structure and status code
  - Location to test: `main.py` lines 142-144
  - Requires: Simple HTTP assertion

**ConfigManager Class:**
- `__init__()` - Configuration initialization
  - Test: Default profile creation with environment variables
  - Location: `main.py` lines 48-61

- `get_profile()` - Profile retrieval with caching
  - Test: Cache behavior, N8N fetch, fallback to default
  - Location: `main.py` lines 99-117

- `_create_default()` - Default profile generation
  - Test: All fields populated, environment variable fallbacks
  - Location: `main.py` lines 63-97

- `invalidate()` - Cache invalidation
  - Test: Single agent invalidation, full cache clear
  - Location: `main.py` lines 119-123

**AgentProfile Validation:**
- Pydantic model validation
  - Location: `main.py` lines 36-46
  - Test: Valid profiles, missing required fields, type validation

**Agent Worker Configuration:**
- `agent_entrypoint()` - LiveKit agent initialization
  - Location: `agent_worker.py` lines 89-168
  - Should test: Agent ID extraction, configuration loading, session creation
  - Requires: Mock `JobContext`, `AsyncSubscribe`, `Sarvam` plugins

- `get_agent_config()` - Configuration fetching with fallback
  - Location: `agent_worker.py` lines 39-70
  - Test: Successful fetch, timeout, fallback to defaults
  - Requires: Mock `httpx.AsyncClient`

**LeadNurtureAgent Class:**
- `on_enter()` - Greeting delivery
  - Location: `agent_worker.py` lines 83-87
  - Test: Greeting message sent when configured

## Recommended Testing Setup

**Framework Recommendation:**
- Use `pytest` for Python testing (commonly used for FastAPI + async code)
- Use `pytest-asyncio` for async test support
- Use `pytest-mock` for mocking external dependencies

**Recommended Dependencies (to add to requirements.txt):**
```
pytest>=7.0.0
pytest-asyncio>=0.21.0
pytest-mock>=3.10.0
httpx[testing]>=0.25.0
```

**Configuration File (conftest.py):**
```python
# At project root: conftest.py
import pytest
from fastapi.testclient import TestClient
from main import app

@pytest.fixture
def client():
    return TestClient(app)

@pytest.fixture
def async_client():
    return httpx.AsyncClient(app=app, base_url="http://test")
```

## Testing Patterns (To Implement)

**API Endpoint Testing Pattern:**
```python
@pytest.mark.asyncio
async def test_generate_token_success():
    """Test successful token generation"""
    mock_request = AsyncMock()
    mock_request.json.return_value = {
        "agentId": "default",
        "language": "hi-IN",
        "voice": "arya",
        "userName": "TestUser"
    }

    response = await generate_token(mock_request)

    assert "token" in response
    assert response["roomName"].startswith("test-default-")
    assert response["livekitUrl"] is not None
```

**ConfigManager Testing Pattern:**
```python
@pytest.mark.asyncio
async def test_config_manager_caching():
    """Test that ConfigManager caches profiles"""
    manager = ConfigManager()

    # First call fetches profile
    profile1 = await manager.get_profile("default")

    # Second call returns cached version
    profile2 = await manager.get_profile("default")

    assert profile1 is profile2  # Same object reference
```

**Async HTTP Mocking Pattern:**
```python
@pytest.mark.asyncio
async def test_n8n_fetch_with_timeout(monkeypatch):
    """Test handling of N8N timeout"""
    async def mock_get(*args, **kwargs):
        raise httpx.ConnectTimeout("Connection timeout")

    monkeypatch.setattr(httpx.AsyncClient, "get", mock_get)
    manager = ConfigManager()

    # Should return default when N8N fails
    profile = await manager.get_profile("custom")
    assert profile.id == "default"
```

**Error Handling Testing Pattern:**
```python
@pytest.mark.asyncio
async def test_token_generation_missing_credentials(monkeypatch):
    """Test token generation fails gracefully without LiveKit credentials"""
    monkeypatch.setenv("LIVEKIT_API_KEY", "")
    monkeypatch.setenv("LIVEKIT_API_SECRET", "")

    mock_request = AsyncMock()
    mock_request.json.return_value = {"agentId": "default"}

    with pytest.raises(HTTPException) as exc:
        await generate_token(mock_request)

    assert exc.value.status_code == 500
    assert "credentials not configured" in exc.value.detail
```

## Current Gaps and Risks

**No Test Coverage For:**
- LiveKit token generation (critical path)
- External service failures (N8N, Sarvam API timeout handling)
- Concurrent configuration requests and cache behavior
- Input validation on Pydantic models
- HTML UI functionality (no JavaScript test framework)
- Agent worker lifecycle (no test setup for LiveKit SDK)

**Untested Error Paths:**
- Missing environment variables fallback behavior
- Invalid speaker selection in agent_worker.py (line 128-130)
- Bare exception catch in metadata parsing (line 99)

**High-Risk Untested Code:**
- `generate_token()` function - Creates LiveKit tokens, external SDK call
- `ConfigManager.get_profile()` - Network request, caching logic, fallback logic
- `agent_entrypoint()` - Complex LiveKit agent initialization

---

*Testing analysis: 2026-02-17*
