# APEX — AI Agent Guide

## Overview
APEX is an autonomous AI red-team operator. FastAPI + WebSocket UI, 8 LLM agents orchestrated through a deterministic Supervisor with safety rails. SQLite bi-temporal memory. FakeEngineClient for Windows dev.

## Project Layout
```
apex/                  # Main package
  agents/              # Agent registry + prompts + doctrine
  memory/              # SQLite store, vault, compaction, embeddings
  oracle/              # Claim verification
  rails/               # Deterministic safety gates
  supervisor/          # Core orchestration + fabric + engine
  tools/               # Tool catalog + parsers
  ui/                  # FastAPI + WebSocket frontend
  types.py             # Core dataclasses (FROZEN)
  capabilities.py      # Platform detection
config/                # .env, apex.toml
deploy/                # opencode config, mitmproxy scope gate
docs/                  # Frozen build contracts (source of truth)
tests/                 # Per-module test suites
```

## Key Conventions (frozen in V1_BUILD_CONTRACTS.md)
- Python 3.12, `from __future__ import annotations`, full type hints
- `pathlib.Path` only, no `os.path`
- Async: Supervisor + EngineClient + MemoryStore writes. Rails = pure sync.
- Packaged data via `importlib.resources` (never `__file__`)
- IDs = `uuid4().hex`. Timestamps = ISO-8601 UTC.
- SQLite-WAL, bi-temporal (invalidate-don't-delete), single-writer queue
- All blocking tests pass on Windows with no Linux tools / proxy / OpenCode

## Rails Evaluation Order (short-circuit)
1. KillSwitch.guard()
2. NoDestructiveGuard.check()
3. ScopeGate.decide(target)
4. CostBreaker.check()
5. ApprovalGate.evaluate(action, mode, tier)
AuditLog.record() runs on every authorize() regardless of outcome.

## Core Data Flow
User objective → Supervisor → Fabric (classify + route) → Agent prompt → Engine (LLM) → Action → Rails authorize → Execute → Oracle verify → Memory persist → Event bus → UI push

## Wave-2 Carry-Over (DONE)
- F3: EventBus persist-hook — all events persist with a monotonic seq. `EventBus.set_persist_hook` + `MemoryStore.persist_event`; component emits omit `seq` so the hook allocates it. (commit a56938e)
- F5: KillSwitch scoped cancel — `fire()` cancels only Supervisor-registered engagement tasks (never `all_tasks()`); the synchronous `guard()` at action sites raises `Killed` and is the real stop. (in `rails/kill.py`)
- F7: Loot reveal audit-logged — `SupervisorBridge.reveal_secret()` decrypts a single loot secret and records a `loot_reveal` audit event. (commit a56938e)

## Testing notes
- The opencode contract/live tests (`tests/engine/test_engine_contract.py`, `test_engine_opencode_live.py`) skip unless a server answers `/global/health`. A reachable-but-unusable server (unconfigured/unfunded model) can no longer hang the suite: live calls are wrapped in `_engine_contract.bounded()` (a hard `asyncio.wait_for` total bound) and `tests/engine/conftest.py` converts a stall into a SKIP. (commit 869a1c6)
- Two distinct env vars point the gates at a server: `OPENCODE_URL` (contract test) and `APEX_OPENCODE_BASE` (live test). Point both at a dead port to force-skip both for a fast local run.

## Windows TTS (Text-to-Speech)
- 3 voices: David (male, en-US), Zira (female, en-US), Sabina (female, es-MX)
- Reusable script at `install/tts.ps1`
- Usage:
  ```
  pwsh install/tts.ps1 -InputFile docs/my-file.md -OutputFile out.wav -Voice Zira
  pwsh install/tts.ps1 -ListVoices
  pwsh install/tts.ps1 -InputText "Hello world" -SpeakOnly
  ```
- Uses built-in `System.Speech` — no dependencies. WAV output only (convert with ffmpeg for MP3).

## Testing
- `pytest` with `asyncio_mode = auto`
- Blocking suites: `tests/rails/`, `tests/memory/`, `tests/fabric/`, `tests/engine/`, `tests/eval/`
- FakeEngineClient parametrized with OpenCodeEngineClient for contract tests
- No network/Linux/proxy in blocking suites
