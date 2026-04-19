# CLAUDE.md

This repository contains a **build specification**, not an implementation. The spec is in `README.md`.

## What to do when working in this repo

When a user opens this repo in Claude Code and asks you to implement it:

1. **Read `README.md` first** — the spec below the `---` line is the source of truth.
2. **Scaffold the project** per the "Deliverables" section: `src/gateway/`, `Dockerfile`, `docker-compose.yml`, `config.example.yaml`, `tests/`, and a smoke script.
3. **Respect the coding style**: Python 3.12, async-only, `mypy --strict`, one file per provider under `src/gateway/providers/`, boring explicit code — no metaprogramming.
4. **Translator-first**: for each new provider, write the request/response translator with unit tests against fixture JSON before wiring it into the router.
5. **Do not deviate from the contract**: every endpoint must match the OpenAI REST shape exactly. If a provider can't support a feature (e.g. streaming), synthesize it in the gateway so clients see a uniform API.

## Non-negotiables

- **Zero client-side changes** when switching providers. Model-id is the only routing signal.
- **Never leak upstream API keys** to clients. They live server-side only.
- **Streaming** must work for every chat provider, real or synthesized.
- **Error shape** is always OpenAI's `{ "error": { "message", "type", "param", "code" } }`.

## What to skip

No web UI, no auth beyond a shared bearer token, no rate-limiting, no response caching, no embeddings/image-gen endpoints. Keep the scope tight.

## Testing checklist before declaring done

- `docker compose up` brings up the gateway healthy at `http://localhost:4000/health`.
- `pytest` passes with ≥ 80% coverage on translator modules.
- `mypy --strict src/` is clean.
- The `smoke.sh` script hits every endpoint against a running instance and exits `0`.
- The `openai` Python SDK pointed at `http://localhost:4000` works for chat, TTS, STT with at least one provider each.
