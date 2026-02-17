# Voice Agent for Lead Nurturing

## What This Is

A real-time voice AI agent that handles inbound/outbound calls for lead nurturing in real estate. The agent speaks Indian languages (Hindi, Tamil, Telugu, etc.) naturally using Sarvam AI, qualifies leads by understanding their property requirements, and schedules follow-ups.

## Core Value

**The ONE thing that must work:** A user can call in, speak in Hindi/Hinglish, and have a natural conversation with an AI agent that understands their property requirements and captures lead information.

## Context

- **Domain:** Real estate lead qualification via voice
- **Users:** Property buyers calling RAA Estate (or similar real estate firms)
- **Scale:** Single agent deployment initially, scalable to multiple agent profiles
- **Constraints:**
  - Must support Indian languages (Hindi primary, regional languages secondary)
  - Runs on Coolify (Docker-based deployment)
  - Uses Sarvam AI for Indian language speech processing
  - Uses Groq (free tier) or OpenAI for LLM inference

## Technical Stack (Existing)

Based on codebase analysis:
- **Runtime:** Python 3.11
- **Web Framework:** FastAPI + Uvicorn
- **Voice Platform:** LiveKit Agents Framework
- **STT/TTS:** Sarvam AI (saaras:v3 for STT, bulbul:v2 for TTS)
- **LLM:** Groq (llama-3.3-70b-versatile) or OpenAI (gpt-4o-mini)
- **Configuration:** N8N webhooks for agent profiles
- **Deployment:** Docker on Coolify, domain voice.enabble.com

## Requirements

### Validated

- ✓ Voice connection via LiveKit WebRTC — existing
- ✓ Speech-to-text in Hindi using Sarvam AI — existing
- ✓ Text-to-speech in Hindi using Sarvam AI — existing
- ✓ LLM inference via Groq/OpenAI — existing
- ✓ Agent profile configuration via N8N — existing
- ✓ Web-based testing UI — existing
- ✓ Docker deployment — existing
- ✓ Speaker validation with fallback — existing

### Active

- [ ] Production deployment to Coolify (voice.enabble.com)
- [ ] End-to-end voice call flow working without errors
- [ ] Lead data capture and storage via N8N
- [ ] Multi-language support (Hindi, Tamil, Telugu, Bengali)
- [ ] Agent profile management for different use cases

### Out of Scope

- Outbound calling — focus on inbound first
- CRM integration — use N8N webhooks as intermediary
- Multi-tenant SaaS — single deployment per customer
- Phone number integration (Twilio/Plivo) — use LiveKit directly

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Sarvam AI for Indian languages | Best STT/TTS for Hindi, supports 10+ Indian languages | ✓ Working |
| LiveKit for WebRTC | Modern, well-documented, agent framework available | ✓ Working |
| Groq as primary LLM | Free tier, fast inference, llama-3.3-70b quality | ✓ Working |
| N8N for configuration | User already has N8N, enables no-code profile editing | ✓ Working |
| Single Docker container | Simplifies Coolify deployment | — Pending |

## Success Criteria

1. User can call voice.enabble.com and have a conversation in Hindi
2. Agent correctly identifies property requirements (budget, location, timeline)
3. Lead data is captured and sent to N8N webhook
4. Call quality is acceptable (no major latency, clear audio)
5. Agent handles conversation naturally (Hinglish, polite, professional)

---

*Last updated: 2026-02-17 after codebase mapping and project initialization*
