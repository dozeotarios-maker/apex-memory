# APEX — Session Handoff (full context)

Last updated: 2026-05-31. Read this to pick up cold. Paired with `AGENTS.md`
(architecture), `docs/V1_BUILD_CONTRACTS.md` (frozen contracts), `docs/MODELS.md`
(model flexibility), `docs/DEPLOY.md` (deploy runbook), `docs/APEX-MASTER-PLAN.md`
(§18/§19 design).

---

## 0. TL;DR — the one open blocker

**`POST /session` to opencode returns HTTP 500**, so no agent turn ever completes
(UI shows your message but no reply). Traceback:
`apex/supervisor/engine_opencode.py:~204 create_session → resp.raise_for_status()
→ httpx.HTTPStatusError: 500 for http://127.0.0.1:4096/session`.

This is almost certainly an **opencode HTTP-API shape drift**: APEX was built/pinned
against opencode **1.15.12**, the box runs a newer build, and the `/session`
request body (`{"agent":..., "model":{"id":..., "providerID":...}}`) is no longer
accepted as-is. It fails the SAME way for `deepseek/*` and `opencode-go/*`, so it's
NOT a model/auth problem — it's the request schema.

**Next step (do this on the box):**
```bash
# 1. See opencode's real error body + the correct schema
curl -s -X POST http://127.0.0.1:4096/session -H 'Content-Type: application/json' \
  -d '{"agent":"conductor","model":{"id":"deepseek-v4-pro","providerID":"opencode-go"}}'; echo
curl -s -X POST http://127.0.0.1:4096/session -H 'Content-Type: application/json' -d '{}'; echo
curl -s http://127.0.0.1:4096/doc | python3 -m json.tool | less   # OpenAPI: find /session schema
```
Then patch `engine_opencode.create_session` (and `prompt` / `prompt_async` if they
drifted too) to match the running opencode's API. Restart `apex-serve`, send a Main
message, confirm a reply, commit+push. The pinned version note lives in
`deploy/opencode/opencode.json` / `install/bootstrap.sh` (`OPENCODE_VERSION=1.15.12`).

---

## 1. What APEX is

Autonomous AI red-team operator for **authorized** pentests. FastAPI + WebSocket UI →
deterministic **Supervisor** (OODA state machine) → **Fabric** (tier/model routing) →
8 LLM agents via **opencode** (headless engine, loopback :4096) → safety **rails**
(kill/scope/approval/cost/destructive/profile) → bi-temporal SQLite memory + git vault.
Windows dev uses `FakeEngineClient`; Linux/Parrot uses `OpenCodeEngineClient`.

## 2. Repo / branch / remote

- GitHub: `https://github.com/dozeotarios-maker/APEX.git` — **PUBLIC**.
- Working branch `feat/v1-walking-skeleton` == `origin/main` (I push to BOTH; they're
  identical). HEAD at handoff time: **`ec6b9c8`**.
- Clone/pull need NO auth (public). For PUSH from Parrot: create your own
  fine-grained PAT (Contents: Read+Write, scoped to this repo) and let git store it
  via `git config --global credential.helper store` (then push once, enter token as
  password). **Do not paste the token into any committed file.** (No token is stored
  in this repo or this handoff by design.)

## 3. Environment on the Parrot VM

- `~/apex` = the clone. Python 3.12 via uv (`~/.local/bin/uv`). Node 20. opencode-ai
  installed globally (`/usr/local/bin/opencode`, symlinked).
- **OpenCode Go** subscription is authenticated (`opencode auth login` → OpenCode Go).
  `opencode models` lists `opencode-go/*` (deepseek-v4-pro, deepseek-v4-flash, glm-5,
  glm-5.1, kimi-k2.5/2.6, mimo-v2.5(-pro), minimax-m2.5/2.7, qwen3.6-plus, qwen3.7-max)
  plus free `opencode/*` (big-pickle, deepseek-v4-flash-free, mimo-v2.5-free,
  nemotron-3-super-free). Go auth lives in `~/.local/share/opencode/auth.json`.
- Services = systemd USER units in `~/.config/systemd/user/`:
  - `opencode.service` → `opencode serve --hostname 127.0.0.1 --port 4096`
    (`OPENCODE_CONFIG_DIR=~/apex/deploy/opencode`, `EnvironmentFile=~/.config/apex/apex.env`).
  - `apex-serve.service` → `uv run apex-serve --host 127.0.0.1 --port 7860 --engine opencode`.
  - `mitmproxy.service` (optional egress scope-gate; NOT needed for a smoke test).
  - Start: `systemctl --user enable --now opencode apex-serve`. Logs:
    `journalctl --user -u apex-serve -n 50 --no-pager`.
- UI: `http://127.0.0.1:7860` (demo login accepts any user/pass). Parrot Firefox may
  proxy localhost — set "No proxy" or add 127.0.0.1 to no-proxy if "Unable to connect".
- `~/.config/apex/apex.env` (chmod 600, loaded by both units): `APEX_VAULT_KEY` (auto-
  generated), `OPENCODE_API_KEY` (set when the subscription was connected),
  `HF_HUB_OFFLINE=1`, `TRANSFORMERS_OFFLINE=1`, `NO_PROXY=127.0.0.1,localhost`.
  **HTTP_PROXY/HTTPS_PROXY must stay OFF/empty unless mitmproxy is running** — else
  every connection (incl. loopback to opencode) fails "All connection attempts failed".
- `config/apex.toml [models.*]` currently set to `opencode-go/deepseek-v4-pro` (flash
  for operator/analyst). (Was briefly corrupted to `deepseek/opencode-go/...` by a bad
  sed; fixed with `sed -i 's|deepseek/opencode-go/|opencode-go/|g' config/apex.toml`.)
- fastembed `bge-small` prebake failed (no/limited net at install) → episodic recall
  returns zero-vectors (v1.1, non-fatal). Ignore unless doing semantic recall.

## 4. Everything built this session (all on `main`)

Wave-2 + hardening + the entire in-UI config surface + recon increments:
- `a56938e` Wave-2 F3 (event-spine seq via persist-hook) + F7 (audit-logged loot
  reveal) + UI server lifespan/importlib.resources modernization.
- `869a1c6` opencode live-tests bounded → reachable-but-stalled serve SKIPS (no hang).
- `e5a650f` recon: passive-capture (tshark) per-port services feed ProfileGate.
- `47e3d6b` loot hashes export enabled; role-model swap test.
- `324b39c` `install/vm-oneclick.sh` (curl|bash one-click VM installer).
- `aa87d78` `docs/MODELS.md` (connect any provider / the OpenCode subscription).
- Settings UI (in-browser config of everything): `1a0c544` read foundation + live
  discovery, `e38245f` write layer (providers/models/budget/secrets/RoE, atomic +
  audited + loopback-only), `405fb74` Settings tab frontend, `bfce134` real
  subscription connect (key-based provider).
- `60afa7a` Playwright e2e smoke test (opt-in: `APEX_E2E=1`).
- `e143cd4` capability-aware role validation + graceful thinking degradation (§19).
- `02887a6` live model discovery from engine (bounded). `6a6e331` Chats per-agent
  threads (derived from the event spine, no schema change). `083964a` auto-summon
  Tool Researcher on a tool-missing gap. `cb2a19f` wired `effective_thinking` to a
  per-call consumer. `67f07a4` WinDump tool config (Windows capture fallback).
  `19a6ef0` port-level audit chain (recon→action traceability).
- Fresh-VM bring-up fixes: `ce3e825` (opencode npm -g needs sudo), `20a702d` (unit
  ExecStart via `/usr/bin/env opencode`), `e2b3e68` (UI boots without a model +
  proxy off by default), `77f509f` (headless honors `--port 7860`).
- In-UI config polish: `0669951` delete/clear secrets, `d429f61` auto-restart
  opencode on key save (no manual restart; `APEX_NO_AUTO_RELOAD=1` to disable),
  `42c5ad2` live discovery via async path, `f72ede4` per-role model = editable
  combobox (type ANY id, e.g. `opencode-go/deepseek-v4-pro`), `ec6b9c8` Main grid no
  longer blank on first load (window-manager 0-width-at-init fix).

Tests: full suite **1249 passed / 19 skipped** on Windows (offline). Run with both
opencode env vars at a dead port so the live tests skip:
```bash
OPENCODE_URL=http://127.0.0.1:4099 APEX_OPENCODE_BASE=http://127.0.0.1:4099 \
  python -m pytest -p no:cacheprovider --timeout=60 -q -o addopts=""
```

## 5. In-UI config (Settings tab) — what works

Providers (add/edit/delete) · per-role model (**editable combobox** — pick or type
any id) + thinking mode · Secrets (masked, set + **Clear**) · OpenCode Subscription
connect (key) · Budget · RoE editor. All writes are atomic, audit-logged, and
**loopback-only** (`_require_local` → 403 off-127.0.0.1). Saving a provider key /
connecting the subscription auto-restarts opencode so it applies without a manual
restart. Model dropdowns: opencode hides subscription models from `/config/providers`,
so auto-discovery may miss `opencode-go/*` — the combobox lets you type them directly.

## 6. v1.0 acceptance (the milestone after the /session fix)

Once a turn works: against ONE authorized lab box (Metasploitable/DVWA/GOAD on an
isolated net) write `~/apex/engagements/<id>/roe.yaml` (scope), then:
```bash
APEX_EVAL_LAB=1 APEX_EVAL_TARGET=<lab-ip> uv run pytest tests/eval/test_v1_acceptance.py -v
```
Rails gate irreversible actions (approval in approve-gated mode) — by design. Only
authorized lab/dev targets.

## 7. Backlog (not blocking)

- Loot ZIP/report export (deliberate stubs). Recon: deeper port-level audit query
  surfacing; more passive-capture parsing. fastembed prebake (semantic recall).
- Auto-discovery of subscription models (opencode doesn't expose them via
  `/config/providers`; `opencode models` does — could wire APEX to that source).

## 8. Tooling / skills available to the agent (OMC)

This dev env runs **oh-my-claudecode** (multi-agent orchestration) + **caveman mode**
(terse output). Relevant skills if you're driving with Claude Code:
- Orchestration: `executor` (route code here; `model=opus` for complex), `architect`,
  `planner`/`omc-plan`, `explore`, `verifier`, `code-reviewer`, `security-reviewer`,
  `debugger`, `test-engineer`, `document-specialist`, `designer`.
- Tier-0 workflows: `autopilot`, `ultrawork`, `ralph`, `team`, `ralplan`.
- Caveman: `/caveman` (lite|full|ultra), `caveman-commit`, `caveman-review`.
- Frontend/UI: `frontend-design`, `ui-ux-pro-max`, `playwright`.
- Misc: `deepinit`, `graphify`, `vibesec`, `wiki`, claude-mem suite.
Conventions: delegate multi-file work to `executor`; keep authoring vs review in
separate passes; verify with tests before claiming done; commit messages end with the
Co-Authored-By trailer; push only when asked.

## 9. Frozen conventions (don't break)

Python 3.12, `from __future__ import annotations`, full type hints, `pathlib` only,
`importlib.resources` for packaged data (never `__file__`), IDs `uuid4().hex`, ISO-8601
UTC, SQLite-WAL bi-temporal (invalidate-don't-delete, single-writer). Rails are pure
sync; kill `guard()` **raises `Killed`** (frozen §5 — do not change to a verdict).
Blocking suites must pass on Windows with no Linux tools / network / opencode.
