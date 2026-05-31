# APEX — Persistent Memory (resume pointer)

The canonical, in-repo memory so context survives any session clear / machine move.
Read order on resume: **this file → `docs/SESSION_HANDOFF.md` (full detail) →
`docs/APEX-MASTER-PLAN.md` (design §18/§19) → `AGENTS.md` (architecture)**.

> Note: the local `claude-mem` DB holds **no** APEX entries (it only had unrelated
> projects — checkout/stadium HTML, Parrot SSH). All real project memory is the
> committed docs in this repo, listed below. Nothing of value lives outside git.

## Snapshot (2026-05-31)

- Repo: `github.com/dozeotarios-maker/APEX` — **PUBLIC**. Branch
  `feat/v1-walking-skeleton` == `origin/main`. Clean, fully pushed.
- State: code-complete for v1, **1249 passed / 19 skipped** on Windows (offline).
  Full in-UI config surface shipped (providers/models/secrets/subscription/budget/RoE,
  loopback-only + audited). Wave-2 (F3/F5/F7) done. Recon increments done.
- **OPEN BLOCKER (live, on the Parrot VM):** opencode `POST /session` → HTTP 500 in
  `apex/supervisor/engine_opencode.py:create_session` → no agent turn replies.
  Cause = opencode HTTP-API shape drift (APEX pinned 1.15.12, box runs newer). Fix:
  `curl` `/session` + `GET /doc` for the real schema → patch `create_session`
  (+ `prompt`/`prompt_async` if drifted) → restart `apex-serve` → verify a Main
  reply → commit+push. Details + exact curls in `docs/SESSION_HANDOFF.md` §0.

## Environment (Parrot VM)

- `~/apex` clone. Python 3.12 (uv), Node 20, opencode-ai global. **OpenCode Go** sub
  authed (`opencode auth login` → auth.json); `opencode models` lists `opencode-go/*`.
- `config/apex.toml [models.*]` = `opencode-go/deepseek-v4-pro` (flash for
  operator/analyst).
- systemd user units: `opencode`(:4096), `apex-serve`(:7860), `mitmproxy`(optional).
  UI `http://127.0.0.1:7860` (demo login any creds).
- `~/.config/apex/apex.env`: `APEX_VAULT_KEY`, `OPENCODE_API_KEY`, offline-embed
  flags, `NO_PROXY=127.0.0.1,localhost`. **HTTP_PROXY/HTTPS_PROXY OFF unless
  mitmproxy is running.**

## Test command (Windows, offline)

```bash
OPENCODE_URL=http://127.0.0.1:4099 APEX_OPENCODE_BASE=http://127.0.0.1:4099 \
  python -m pytest -p no:cacheprovider --timeout=60 -q -o addopts=""
```

## Security

No secrets/tokens are committed anywhere in this repo (verified). Public repo →
`clone`/`pull` need no auth. PUSH from a new box uses a locally-generated PAT
(fine-grained, Contents:Read+Write) stored via git credential helper — **never** put
the token in a committed file.

## All committed memory/docs

- `docs/SESSION_HANDOFF.md` — full cold-start context (every commit, env, blocker, skills).
- `docs/APEX-MASTER-PLAN.md` — master plan (§18/§19 authoritative).
- `docs/V1_BUILD_CONTRACTS.md` + `docs/V1_1_*_CONTRACTS.md` — frozen contracts.
- `docs/DEPLOY.md` — deploy runbook. `docs/MODELS.md` — model flexibility.
- `docs/APEX-TOOL-CATALOG.md`, `docs/UI_DESIGN_REFERENCE.md`, `docs/beyond-ooda.md`.
- `AGENTS.md` — architecture + conventions + Wave-2 status.

## Resume

Say **"resume apex"**, or on the box: `cd ~/apex && claude` →
*"read docs/MEMORY.md + docs/SESSION_HANDOFF.md, then fix the opencode /session 500
so a Main turn replies."*
