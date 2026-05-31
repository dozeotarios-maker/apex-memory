# APEX v1.1 — Deployment Runbook

Covers: Parrot OS VM (primary target) + Windows dev box (secondary).
Ties to §21 (deployment toolchain) and §23 (uv hermetic, systemd-app, Docker-optional).

---

## 0. One-click (fresh public-repo VM)

On a clean Linux/Parrot VM with internet, a single command installs everything
(git, Node 20, uv, Python 3.12, deps, opencode, embeddings, systemd units) and
scaffolds the secrets file:

```bash
curl -fsSL https://raw.githubusercontent.com/dozeotarios-maker/APEX/main/install/vm-oneclick.sh | bash
```

Then two manual steps it can't do for you: paste `DEEPSEEK_API_KEY` into
`~/.config/apex/apex.env` (the `APEX_VAULT_KEY` is auto-generated), and write an
RoE (§4). Override targets with env: `APEX_DIR=~/apex APEX_BRANCH=main`.
Idempotent — re-run any time to update (it fast-forward-pulls). The manual
clone+bootstrap path below (§2) is equivalent if you prefer step-by-step.

---

## 1. Prerequisites

### Parrot OS VM (target deploy)

| Requirement | Notes |
|---|---|
| Parrot OS 6+ (or Debian 12+) | Parrot's preloaded tools (nmap, metasploit, …) are used directly — do **not** containerise APEX away from them |
| Python 3.12 | Installed by bootstrap.sh via uv (python-build-standalone) |
| Node.js ≥ 20 | For opencode-ai; bootstrap.sh prints the apt install command if missing |
| uv | Installed by bootstrap.sh if absent |
| systemd --user | Standard on Parrot; needed for the three user units |
| Docker (optional) | Only needed for the optional mitmproxy container variant |

### Windows dev box

| Requirement | Notes |
|---|---|
| PowerShell 7+ (`pwsh`) | For `bootstrap.ps1` |
| uv | Installed by bootstrap.ps1 if absent |
| Node.js ≥ 20 | `winget install OpenJS.NodeJS.LTS` |
| Python 3.12 | Managed by uv |

---

## 2. Bootstrap

### Parrot VM

```bash
# Clone the repo (or copy it into the VM)
git clone https://github.com/your-org/apex.git ~/apex
cd ~/apex

# Run the idempotent bootstrap (safe to re-run)
bash install/bootstrap.sh
```

What it does (in order):
1. Installs `uv` if absent (`~/.local/bin/uv`)
2. `uv python install 3.12` (python-build-standalone, no system Python required)
3. `uv sync --frozen --extra linux-tools` (impacket + all deps, from `uv.lock`)
4. Checks Node / installs `opencode-ai@1.15.12` via `npm -g`
5. Checks Docker; if daemon is running, `docker compose -f deploy/compose.yaml up -d`
6. Pre-downloads `BAAI/bge-small-en-v1.5` so the embedding model is available offline
7. Installs systemd user units from `deploy/systemd/` to `~/.config/systemd/user/`
8. Enables linger (`loginctl enable-linger $USER`) so units survive logout

### Windows dev box

```powershell
pwsh -ExecutionPolicy Bypass -File install\bootstrap.ps1
```

Does steps 1–4 and 6 only. Systemd, mitmproxy, linux-tools, and opencode are
Linux-only and are skipped with a clear warning.

---

## 3. Secrets and environment file

Create `~/.config/apex/apex.env` **before** starting any service.
This file is loaded by all three systemd units via `EnvironmentFile=`.

```bash
mkdir -p ~/.config/apex
cat > ~/.config/apex/apex.env <<'EOF'
# AI provider key (required)
DEEPSEEK_API_KEY=sk-...

# Vault encryption key for loot at rest (required; any random 32+ char string)
APEX_VAULT_KEY=change-me-to-something-random-and-long

# Egress proxy (set after mitmproxy is running)
HTTP_PROXY=http://127.0.0.1:8080
HTTPS_PROXY=http://127.0.0.1:8080

# Offline embedding guard (prevents HuggingFace ping at startup)
HF_HUB_OFFLINE=1
TRANSFORMERS_OFFLINE=1
EOF
chmod 600 ~/.config/apex/apex.env
```

**Never commit this file.** It is in `.gitignore`.

---

## 4. RoE — Rules of Engagement

Before starting an engagement, create a RoE YAML under
`engagements/<id>/roe.yaml`.  The Supervisor loads this at startup.

Minimum required fields:

```yaml
engagement_id: parrot-lab-001
in_scope:
  - 10.10.0.0/24       # lab range
out_scope:
  - 10.10.0.1          # gateway — exclude the hypervisor
windows: []            # time windows (empty = always allowed)
forbidden_actions:
  - destructive
auto_actions:
  - recon
  - passive
require_human_actions:
  - exploit
  - lateral-move
  - exfil
data_handling:
  llm_providers:
    - deepseek
  disclosed: true       # client has been informed LLM processes data
control_plane_allow:
  - api.deepseek.com    # allowed outbound (not scope-gated)
```

The mitmproxy scope-gate (`deploy/proxy/scope_gate.py`) reads the compiled
allowlist from the Supervisor at startup. The proxy **blocks** any outbound
connection that is not in `control_plane_allow` OR the `in_scope` CIDR list.

---

## 5. Starting APEX

### Option A — Foreground (dev / debugging)

```bash
cd ~/apex

# Terminal 1: opencode engine
opencode serve --hostname 127.0.0.1 --port 4096

# Terminal 2: mitmproxy scope-gate
uv run mitmdump \
  --listen-host 127.0.0.1 --listen-port 8080 \
  -s deploy/proxy/scope_gate.py

# Terminal 3: APEX headless server
uv run apex-serve --host 127.0.0.1 --port 7860 --engine opencode
```

Open `http://127.0.0.1:7860` in the VM browser.

### Option B — systemd (headless / persistent)

```bash
# Enable and start all three units
systemctl --user enable --now opencode
systemctl --user enable --now mitmproxy
systemctl --user enable --now apex-serve

# Check status
systemctl --user status opencode mitmproxy apex-serve

# Tail logs
journalctl --user -u apex-serve -f
```

Units are defined in `deploy/systemd/`:

| Unit file | ExecStart | Port |
|---|---|---|
| `opencode.service` | `opencode serve --hostname 127.0.0.1 --port 4096` | 4096 |
| `mitmproxy.service` | `uv run mitmdump --listen-host 127.0.0.1 --listen-port 8080 -s …/scope_gate.py` | 8080 |
| `apex-serve.service` | `uv run apex-serve --host 127.0.0.1 --port 7860 --engine opencode` | 7860 |

All units use `EnvironmentFile=~/.config/apex/apex.env` and
`Restart=on-failure`.

---

## 6. Egress proxy wiring (mitmproxy scope-gate)

APEX routes all agent outbound traffic through the mitmproxy instance so the
scope-gate addon can enforce RoE egress rules.

The `HTTP_PROXY` / `HTTPS_PROXY` variables in `apex.env` are picked up
automatically by Python's `httpx` / `requests` clients inside the Supervisor.

Flow:

```
Supervisor (Python)
  └─ httpx (HTTP_PROXY=http://127.0.0.1:8080)
       └─ mitmproxy:8080  ← scope_gate.py evaluates every request
            ├─ ALLOW → forward to destination
            └─ BLOCK → 403 + audit log entry
```

The scope-gate reads the current engagement's compiled allowlist from
`deploy/proxy/scope_gate.py`. Out-of-scope destinations receive a `403
Forbidden` response; the block is written to the APEX audit log.

To verify the proxy is working:

```bash
# Should succeed (control-plane allow)
curl --proxy http://127.0.0.1:8080 https://api.deepseek.com/v1/models

# Should be blocked (not in scope / not in control_plane_allow)
curl --proxy http://127.0.0.1:8080 https://example.com
```

---

## 7. Optional Docker infra

`deploy/compose.yaml` provides an opt-in mitmproxy container.
Qdrant (vector store) is **commented out** — deferred to v2 (§23).

```bash
# Start optional infra
docker compose -f deploy/compose.yaml up -d

# Stop
docker compose -f deploy/compose.yaml down
```

APEX core (Supervisor + FastAPI + opencode) runs on the host, not in Docker.
The systemd mitmproxy unit is preferred over the Docker container on the
Parrot VM because it can access the uv venv directly.

---

## 8. Parrot VM — notes

- **Snapshot before any engagement.** Revert to snapshot if the VM state
  becomes inconsistent.
- opencode is installed as an npm global binary (`/usr/local/bin/opencode`).
  The systemd unit's `Environment=PATH=…` line must include this directory.
- The Parrot toolchain (nmap, metasploit, netcat, …) is accessed by opencode
  via native bash. **Do not containerise opencode** — it would lose access to
  these tools.
- SQLite engagement databases live under `engagements/<id>/engagement.db` on
  the host filesystem. Back them up before snapshots if you want to preserve
  engagement state.
- `loginctl enable-linger $USER` is required for systemd user services to
  start at boot without a login session. bootstrap.sh sets this automatically.

---

## 9. Capability-eval harness (§13/§19)

The eval scaffold lives in `apex/eval/harness.py` and `tests/eval/test_capability_eval.py`.

Running the test suite (no lab required — all eval tests SKIP gracefully):

```bash
uv run pytest tests/eval/ -q
```

Running against a live lab:

```bash
APEX_EVAL_LAB=1 APEX_EVAL_TARGET=10.10.0.5 \
    uv run pytest tests/eval/test_capability_eval.py -v
```

The `EvalCase` / `CapabilityCheck` / `EvalRunner` interface is defined in
`apex/eval/harness.py`. Wire `EvalRunner.run()` to a live Supervisor in v2
to drive real engagement loops and collect `EvalResult` verdicts.

---

## 10. Reference

- `docs/APEX-MASTER-PLAN.md` §21 — deployment architecture decisions
- `docs/APEX-MASTER-PLAN.md` §23 — uv hermetic, Qdrant deferred, bootstrap contract
- `deploy/systemd/` — all three user units
- `deploy/compose.yaml` — optional Docker infra (mitmproxy; Qdrant commented out)
- `deploy/proxy/scope_gate.py` — mitmproxy egress addon
- `install/bootstrap.sh` — idempotent Linux bootstrap
- `install/bootstrap.ps1` — idempotent Windows dev bootstrap
- `apex/eval/harness.py` — capability-eval scaffold
