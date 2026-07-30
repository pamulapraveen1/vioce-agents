# VoxAgent 🎙️

**Open-source AI voice agent platform for phone calls and customer support** — with a
real-time voice pipeline, agentic tool calling, built-in RAG knowledge bases,
speech-to-text APIs, analytics, and a landing page + dashboard.

**Runs 100% free out of the box** on local models — no API keys required:

| Layer | Free default (no key) | Paid upgrade (optional) |
|---|---|---|
| Speech-to-text | **faster-whisper** (local) | Deepgram Flux / Nova-3, OpenAI Whisper API |
| Brain (LLM) | **Ollama** (Llama 3.2, local) | Anthropic Claude |
| Text-to-speech | **Microsoft Edge TTS** (free, 400+ voices) | ElevenLabs Flash, OpenAI TTS |
| Telephony | Web calls (built-in WebSocket) | Twilio phone numbers |

Providers are hot-swappable **per agent** — start free, flip a field to premium.

## Features

- ⚡ **Real-time pipeline** — streaming STT → token-streamed LLM → streaming TTS.
  Audio starts on the first phrase; latency is measured on every turn.
- 🗣️ **Barge-in** — interrupt the agent mid-sentence; playback flushes instantly.
- 🛠️ **Tool calling** — order lookup, refunds, tickets, callbacks, human transfer,
  with spoken filler phrases so the line never goes silent. Add your own in
  `backend/app/voice/tools.py`.
- 📚 **Knowledge base (RAG)** — upload docs per agent; answers are grounded in them.
- 📞 **Phone + web** — Twilio Media Streams for PSTN, WebSocket API for in-app voice.
- 📝 **Speech-to-text API** — batch transcription with word timestamps & diarization.
- 📊 **Post-call intelligence** — automatic summary, sentiment, resolution status,
  entity extraction, and an analytics dashboard.

## Quick start (free stack)

Prereqs: Python 3.11+, [ffmpeg](https://ffmpeg.org), [Ollama](https://ollama.com).

```bash
# 1. Get a free local LLM
ollama pull llama3.2

# 2. Install & run the backend
cd backend
pip install -r requirements.txt
cp ../.env.example .env
python seed.py                      # creates the "Ava" demo support agent
uvicorn app.main:app --reload

# 3. Talk to it
open http://localhost:8000/demo     # allow mic, click Start Call
```

- Landing page: `http://localhost:8000/`
- Live voice demo: `http://localhost:8000/demo`
- Dashboard (agents, calls, analytics): `http://localhost:8000/dashboard`
- API docs (OpenAPI): `http://localhost:8000/docs`

### Docker (API + Postgres + Ollama)

```bash
docker compose up -d
docker compose exec ollama ollama pull llama3.2
```

## Phone calls (Twilio)

1. Set `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_PHONE_NUMBER`, and a public
   `API_BASE_URL` in `.env`.
2. Point your Twilio number's voice webhook at `POST {API_BASE_URL}/telephony/inbound`.
3. Outbound: `POST /api/calls/outbound {"agent_id": "...", "to_number": "+1..."}`.

## API overview

| Endpoint | Purpose |
|---|---|
| `POST /api/agents` | Create an agent (prompt, voice, models, tools) |
| `GET /api/agents/tools` | List available call-time tools |
| `POST /api/agents/{id}/knowledge` | Add a knowledge-base document |
| `WS /ws/voice/{agent_id}` | Live web voice session (PCM16 in/out) |
| `POST /telephony/inbound` | Twilio voice webhook |
| `POST /api/calls/outbound` | Place an outbound phone call |
| `GET /api/calls` / `/api/calls/{id}` | Call history with transcripts & analysis |
| `POST /api/transcriptions` | Batch speech-to-text (upload a file) |
| `GET /api/analytics` | Dashboard metrics |

## Architecture

```
Caller ──phone──> Twilio Media Stream ─┐
Browser ──mic──> /ws/voice/{agent} ────┤
                                       ▼
                        ┌──────── VoicePipeline ────────┐
                        │ STT (whisper/deepgram)        │
                        │  └─ end-of-turn + barge-in    │
                        │ RAG retrieve → LLM (ollama/   │
                        │  claude) → tools (orders,     │
                        │  refunds, tickets, transfer)  │
                        │ TTS (edge/elevenlabs) stream  │
                        └───────────┬───────────────────┘
                                    ▼
                 Post-call analysis → DB (calls, messages,
                 summaries, sentiment) → /dashboard analytics
```

See [`docs/MARKET_RESEARCH.md`](docs/MARKET_RESEARCH.md) for the July 2026 competitive
landscape (Retell, Bland, Vapi, Deepgram, ElevenLabs) and how VoxAgent is positioned
to beat them on latency, cost, and openness.
