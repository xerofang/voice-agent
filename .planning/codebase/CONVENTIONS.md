# Coding Conventions

**Analysis Date:** 2026-02-17

## Naming Patterns

**Files:**
- Snake case: `main.py`, `agent_worker.py`
- Descriptive names indicating primary responsibility
- Worker files use `_worker.py` suffix pattern

**Functions:**
- Snake case: `get_agent_config()`, `generate_token()`, `agent_entrypoint()`
- Async functions use `async def` and are named with verbs: `get_`, `fetch_`, `invalidate_`
- Private methods use underscore prefix: `_create_default()`, `_cache`

**Variables:**
- Snake case: `agent_id`, `llm_provider`, `sarvam_api_key`
- Descriptive names: `config_manager`, `start_time`, `durationInterval` (in JavaScript context)
- Configuration variables use full capitalization in module scope: `WEB_SERVER_URL`, `WEB_UI_HTML`
- Cache and internal state prefixed with underscore: `_cache`, `_default`

**Types:**
- PascalCase classes: `AgentProfile`, `ConfigManager`, `LeadNurtureAgent`
- Generic types from external libraries: `BaseModel`, `HTTPException`, `Agent`, `JobContext`

## Code Style

**Formatting:**
- No explicit linter configuration detected
- 4-space indentation (Python standard)
- Multi-line imports wrapped with parentheses: `from livekit.agents import (Agent, AgentSession, ...)`
- Async context managers used: `async with httpx.AsyncClient(timeout=5.0) as client:`

**Linting:**
- Not detected - no .pylintrc, setup.cfg, or flake8 configuration
- No formatting tool (Black, autopep8) configuration found
- Codebase follows PEP 8 conventions implicitly

## Import Organization

**Order:**
1. Standard library imports (`os`, `json`, `asyncio`, `datetime`)
2. Third-party framework imports (`httpx`, `dotenv`, `loguru`, `pydantic`, `fastapi`, `uvicorn`)
3. Third-party provider SDKs (`livekit`, `livekit.agents`, `livekit.plugins`)

**Pattern Example from `main.py`:**
```python
import os
import json
from datetime import datetime
from typing import Optional

import httpx
from dotenv import load_dotenv
from loguru import logger
from pydantic import BaseModel
from fastapi import FastAPI, HTTPException, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import HTMLResponse
import uvicorn

from livekit import api as livekit_api
```

**Path Aliases:**
- Using full import paths: `from livekit.agents import Agent, AgentSession, JobContext`
- Aliasing long imports: `from livekit.plugins import sarvam, openai as lk_openai` in `agent_worker.py`

## Error Handling

**Patterns:**
- Broad exception catching with logging: `except Exception as e: logger.warning(f"...")`
- Specific exception raising for HTTP responses: `raise HTTPException(status_code=500, detail="...")`
- Re-raising specific exceptions: `except HTTPException: raise` (line 185-186 in main.py)
- Bare except used in limited cases where any error should be silently handled: `except: pass` (line 99 in agent_worker.py)

**Fallback Pattern:**
- Default values returned when external calls fail: See `get_agent_config()` in agent_worker.py (lines 39-70)
- Validation with fallbacks: Invalid speakers fallback to default: `if speaker not in valid_speakers: speaker = "arya"` (line 128-130)

**HTTP Exception Pattern:**
```python
try:
    # operation
    raise HTTPException(status_code=500, detail="message")
except HTTPException:
    raise
except Exception as e:
    logger.error(f"Error: {e}")
    raise HTTPException(status_code=500, detail=str(e))
```

## Logging

**Framework:** `loguru`

**Log Levels Used:**
- `logger.info()`: Configuration loaded, service started, agent lifecycle events
- `logger.warning()`: Configuration fetch failures, invalid inputs with fallback
- `logger.error()`: Critical failures in token generation

**Patterns:**
- Informational logging on startup: Lines 175-183 in agent_worker.py
- Warning logs with context: `logger.warning(f"N8N fetch failed: {e}, using default")`
- Error logs in exception handlers with context: `logger.error(f"Token generation failed: {e}")`
- Formatted strings with f-strings: `f"Agent started: {config.get('name')}"`

**When to Log:**
- Service lifecycle events (startup, shutdown, connections)
- Configuration changes and cache operations
- External service failures with fallback behavior
- Operational state changes (e.g., agent profile loaded)

## Comments

**When to Comment:**
- File-level docstrings explain module purpose: Lines 1-11 in main.py and agent_worker.py
- Section comments with ASCII boxes for major code sections: `# ==========================================`
- Class and function docstrings for public APIs
- Inline comments rare - code is self-documenting through naming

**JSDoc/TSDoc:**
- Not used in Python code
- JavaScript in embedded HTML uses minimal comments
- Section headers in JavaScript: `// Check for secure context`, `// Request microphone permission early`

## Function Design

**Size:**
- Small, focused functions: Most functions 10-40 lines
- Longer functions reserved for initialization: `agent_entrypoint()` is 77 lines (complex initialization)
- Web UI HTML embedded as single string for simplicity

**Parameters:**
- Default values used: `async def get_agent_config(agent_id: str = "default")`
- Optional parameters marked with `Optional`: `async def invalidate_cache(agent_id: str = None)`
- Request objects used for complex input: `async def generate_token(request: Request)`

**Return Values:**
- Typed returns: `async def get_agent_config(agent_id: str = "default") -> dict:`
- Model returns for structured data: Returns `AgentProfile` instances
- Dictionary returns for API responses: `return {...}`
- Coroutine returns for async functions

## Module Design

**Exports:**
- Classes instantiated at module level: `config_manager = ConfigManager()` (line 125 in main.py)
- FastAPI app instantiated at module level: `app = FastAPI(title="Voice Agent API")`
- Entry point functions are exported: `agent_entrypoint` in agent_worker.py

**Barrel Files:**
- Not used - single entry file per module (main.py, agent_worker.py)
- All imports are explicit and direct

## Configuration Pattern

**Environment-driven:**
- Configuration loaded via `load_dotenv()` at module start
- All external services configured through environment variables
- `os.getenv()` used with sensible defaults: `os.getenv("LIVEKIT_URL")`, `os.getenv("DEFAULT_VOICE", "arya")`

**Configuration Classes:**
- Pydantic `BaseModel` used for validated config: `class AgentProfile(BaseModel):`
- Manager classes handle config lifecycle: `ConfigManager` with caching and invalidation

## Async Patterns

**Async Context:**
- Used for HTTP clients: `async with httpx.AsyncClient(timeout=5.0) as client:`
- FastAPI async endpoints: All route handlers are `async def`
- Async configuration fetching: `async def get_agent_config()`

**Pattern Example:**
```python
async def generate_token(request: Request):
    try:
        data = await request.json()
        # operations
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

---

*Convention analysis: 2026-02-17*
