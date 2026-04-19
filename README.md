# Prompt: Build a unified OpenAI-compatible gateway

Copy everything below the `---` line into your AI coding agent (Claude Code, Cursor, etc.)
to scaffold a local OpenAI-compatible proxy that fans out to multiple LLM and speech providers.

---

## Task

Build a single self-hosted HTTP server that exposes the **OpenAI REST API** (chat completions,
audio speech, audio transcriptions) and transparently routes each request to the right upstream
provider based on the `model` field in the request body.

Clients (my Chrome extension, any OpenAI SDK, `curl`) should only ever talk to this one server.
I should be able to swap providers by changing the `model` string in the request — no client-side
code changes.

## Tech stack

- **Python 3.12 + FastAPI + Uvicorn**. Async throughout. Use `httpx.AsyncClient` for upstream calls.
- Single `pyproject.toml` with `uv`-compatible deps.
- Config via a single `config.yaml` loaded at startup. Hot-reload is not required.
- Dockerfile with a non-root user, runs `uvicorn` with `--proxy-headers`.
- One `docker-compose.yml` that includes the gateway + any local TTS servers (Qwen3-TTS, Piper).

## Endpoints (OpenAI-compatible shape)

1. `POST /v1/chat/completions` — streaming and non-streaming. Must accept the full OpenAI request
   body including `tools`, `tool_choice`, `response_format`, `temperature`, `max_tokens`, etc.
   Translate to each provider's native SDK, then translate the response back to OpenAI shape.
2. `POST /v1/audio/speech` — text-to-speech. Accepts `{ model, voice, input, response_format, speed }`.
   Return raw audio bytes with the correct `Content-Type`.
3. `POST /v1/audio/transcriptions` — speech-to-text. `multipart/form-data` with the audio file
   plus `model`, `language`, `prompt`, `response_format`.
4. `GET /v1/models` — returns the flat list of every model configured in `config.yaml`, in OpenAI
   shape (`{ id, object: "model", owned_by }`).
5. `GET /health` — returns `{ status: "ok", providers: { <name>: "up" | "down" } }`.

## Providers to support

### LLM (chat completions)
- **OpenAI** (`gpt-*`, `o1-*`) — native passthrough, just forward with auth.
- **Anthropic Claude** (`claude-*`) — use the `anthropic` SDK. Translate OpenAI tool-calling format
  to Anthropic's `tool_use` blocks and back. Handle the system prompt correctly (OpenAI puts it in
  `messages[0]`, Anthropic expects a top-level `system` field).
- **Google Gemini** (`gemini-*`) — use the `google-genai` SDK. Translate `tools` to Gemini's
  `function_declarations`. Translate tool responses (`role: "tool"`) into Gemini's `functionResponse`
  parts. Multimodal input (images, audio inline-data) must pass through.
- **xAI Grok** (`grok-*`) — OpenAI-compatible upstream, passthrough with different base URL.
- **Groq** (`llama-*`, `mixtral-*` prefixed with `groq/`) — OpenAI-compatible passthrough.
- **Ollama** (`ollama/*`) — forward to `http://ollama:11434/v1/chat/completions` (already OpenAI-compatible).

Streaming: every provider that supports SSE must stream token-by-token. If the upstream doesn't
support streaming for a given model, the gateway must synthesize SSE by chunking the final response
word-by-word so clients see a consistent streaming interface.

### TTS (text-to-speech)
- **OpenAI TTS** (`tts-1`, `tts-1-hd`) — passthrough.
- **ElevenLabs** (`elevenlabs/<voice-id>` as model, or a shortname map) — translate to
  `/v1/text-to-speech/{voice_id}`.
- **Qwen3-TTS local** (`qwen3-tts`) — forward to local `Qwen3-TTS-Openai-Fastapi` (default
  `http://qwen-tts:8880/v1/audio/speech`). Already OpenAI-compatible, so essentially a rewrite
  of the URL.
- **Piper local** (`piper/<voice>`) — call the Piper HTTP server, return WAV bytes.
- **Kokoro local** (`kokoro`) — same pattern.

Response format: always honor the client's `response_format` (`mp3`, `opus`, `wav`, `pcm`).
If the upstream returns a different codec, transcode with `ffmpeg` (subprocess, streamed).

### STT (speech-to-text)
- **OpenAI Whisper** (`whisper-1`) — passthrough.
- **Groq Whisper** (`whisper-large-v3`) — passthrough to Groq.
- **Gemini STT** (`gemini-2.5-flash` as STT) — wrap Gemini's multimodal API: upload the audio
  as inline-data, prompt "Transcribe this audio exactly. Output only the transcription.", return
  the text in OpenAI's shape (`{ text: "..." }` for non-verbose, full JSON for verbose).
- **Local `faster-whisper`** (`local-whisper`) — run in-process or via a sidecar container.

## Config format (`config.yaml`)

```yaml
# Bind address
host: 0.0.0.0
port: 4000

# Shared secret clients must send as Authorization: Bearer <token>
api_key: ${GATEWAY_API_KEY}  # env var expansion

# Upstream provider credentials
providers:
  openai:
    api_key: ${OPENAI_API_KEY}
    base_url: https://api.openai.com/v1
  anthropic:
    api_key: ${ANTHROPIC_API_KEY}
  google:
    api_key: ${GOOGLE_API_KEY}
  xai:
    api_key: ${XAI_API_KEY}
    base_url: https://api.x.ai/v1
  groq:
    api_key: ${GROQ_API_KEY}
    base_url: https://api.groq.com/openai/v1
  elevenlabs:
    api_key: ${ELEVENLABS_API_KEY}
  ollama:
    base_url: http://ollama:11434
  qwen3_tts:
    base_url: http://qwen-tts:8880
  piper:
    base_url: http://piper:5000

# Model → provider routing. Wildcards supported via glob.
models:
  # LLM
  - id: gpt-4o
    provider: openai
  - id: gpt-4o-mini
    provider: openai
  - id: o1-*
    provider: openai
  - id: claude-*
    provider: anthropic
  - id: gemini-*
    provider: google
  - id: grok-*
    provider: xai
  - id: groq/*
    provider: groq
    strip_prefix: true   # forward as "llama-..." to Groq
  - id: ollama/*
    provider: ollama
    strip_prefix: true

  # TTS
  - id: tts-1
    provider: openai
  - id: tts-1-hd
    provider: openai
  - id: elevenlabs/*
    provider: elevenlabs
    strip_prefix: true   # treat the rest as voice_id
  - id: qwen3-tts
    provider: qwen3_tts
  - id: piper/*
    provider: piper
    strip_prefix: true

  # STT
  - id: whisper-1
    provider: openai
  - id: whisper-large-v3
    provider: groq
  - id: gemini-stt
    provider: google
    kind: stt           # forces STT handler even if name looks like chat
```

## Error handling

- If the upstream returns an error, propagate it using OpenAI's error shape:
  `{ "error": { "message", "type", "param", "code" } }`.
- On timeout or network failure, return `502` with a clear `message` identifying the upstream.
- On unknown model, return `404` with `error.type: "model_not_found"`.
- Log every request with: model, upstream, latency_ms, status, token usage (if available).
  Use `structlog` JSON logs.

## Auth

- Single shared bearer token (`Authorization: Bearer <token>`) checked against `config.api_key`.
- If `api_key` is empty in config, auth is disabled (useful for fully local dev).
- Upstream API keys are never exposed to clients — always injected server-side.

## Performance requirements

- First token latency (streaming LLM) should add ≤ 50ms of overhead vs. direct upstream call.
- Support at least 100 concurrent streams on a single container (use `httpx.AsyncClient` with
  a shared connection pool, `httpx.Limits(max_connections=500)`).
- Zero-copy streaming for audio: read the upstream response chunk-by-chunk and forward to the
  client without buffering the full audio in memory.

## Tests

- `pytest` with `pytest-asyncio`.
- **Unit tests** for each provider's request/response translator, using recorded fixture JSON.
- **Integration tests** that spin up the gateway against a `respx`-mocked httpx transport for
  every provider. Cover: non-streaming chat, streaming chat, tool-calling round-trip, TTS with
  each response_format, STT with each response_format, error propagation, unknown model.
- **Contract test**: use the official `openai` Python SDK pointed at `http://localhost:4000`
  and verify a full chat → tool-call → tool-response → final-answer flow works against each
  provider (at least OpenAI, Claude, Gemini).

## Deliverables

1. A working FastAPI app in `src/gateway/`.
2. `Dockerfile` and `docker-compose.yml` that brings up the gateway, an Ollama instance, and a
   Qwen3-TTS instance on a shared network.
3. `config.example.yaml` with all providers wired up and clear `${ENV_VAR}` placeholders.
4. `README.md` with: one-command start (`docker compose up`), how to add a new provider, how to
   add a new model, how to point the OpenAI SDK at it.
5. `tests/` with ≥80% coverage on the translator modules.
6. A Postman/Bruno collection or a `smoke.sh` script that exercises each endpoint against a
   running instance.

## What to skip

- No web UI. It's a headless proxy.
- No user management, no per-key rate limiting, no billing. Single shared key is enough.
- No caching of responses (adds complexity, can be added later).
- No fine-tuning / embeddings / image generation endpoints (can be added in a follow-up).

## Coding style

- Type hints everywhere (`mypy --strict` passes).
- One file per provider under `src/gateway/providers/`, each exporting `ChatProvider`,
  `TTSProvider`, or `STTProvider` classes that inherit from a base `Provider` interface.
- Router logic lives in `src/gateway/router.py` — keep it dumb: given a model id, return the
  provider instance. No business logic there.
- Avoid clever metaprogramming. Boring, explicit code.

## Definition of done

I should be able to:

1. `docker compose up` and see the gateway healthy at `http://localhost:4000/health`.
2. Run `curl http://localhost:4000/v1/chat/completions -H "Authorization: Bearer $KEY" -d '{"model":"claude-sonnet-4-6","messages":[{"role":"user","content":"hi"}]}'` and get a Claude response in OpenAI shape.
3. Swap `"model"` to `"gemini-2.5-flash"`, `"gpt-4o"`, or `"ollama/llama3.1"` and get a valid response with no other changes.
4. Call `/v1/audio/speech` with `model: "qwen3-tts"` and get MP3 bytes back from my local server.
5. Point the `openai` Python SDK's `base_url` at the gateway and have it Just Work for every endpoint.
