# APEX v1 — FROZEN BUILD CONTRACTS (ralplan-approved, 2026-05-29)

> **This file is the single source of truth for the v1 walking-skeleton build.** It is FROZEN before the Wave-1 fan-out. Every parallel worker imports the stubs/types defined here and MUST NOT change a signature, event field, DDL column, or truth-table without a re-broadcast. Design rationale lives in `docs/APEX-MASTER-PLAN.md` §1–§23 (do not re-open). This doc realizes §22 (structure) + §23 (portability) + the 13 Architect/Critic contract fixes.
>
> Build strategy = **contract-first → parallel fan-out → vertical integration**. A `FakeEngineClient` decouples the whole slice from OpenCode/DeepSeek/Linux-tools so the entire v1 acceptance test runs + tests on the **Windows dev box**. Done-gate = the §18 v1.0 acceptance test (see §11 below).

---

## 0. CONVENTIONS (apply everywhere)
- Python **3.12**, `from __future__ import annotations`, full type hints, `ruff` + `mypy` clean.
- `pathlib.Path` only (never os.path string-joins). LF endings. No machine-bound secrets (12-factor env).
- Async: the Supervisor + EngineClient + MemoryStore writes are `async`. Rails decision logic is **pure sync** (testable without an event loop).
- Packaged data accessed via `importlib.resources` (NEVER `__file__`) — see §10.
- Every module ships `tests/<area>/test_*.py`. Rails + memory + fabric unit tests MUST pass on Windows with no Linux tools and no proxy.
- IDs = `uuid4().hex`. Timestamps = ISO-8601 UTC strings (`datetime.now(UTC).isoformat()` — but pass a clock in, see §12 testability).

---

## 1. PACKAGE LAYOUT & FILE OWNERSHIP (the DAG)
Root: `C:\Users\stvxy\Desktop\Desktop\NIGLETSTUFF\APEX` (git `main`). Each Wave-1 unit owns DISJOINT files (no two workers touch the same file).

```
pyproject.toml · uv.lock · .python-version           # U0.1 (Wave 0)
install/bootstrap.sh · install/bootstrap.ps1         # U0.1
config/apex.toml · config/.env.example               # U0.3
apex/__init__.py · apex/__main__.py                  # U0.1 stub → U2.2 real
apex/capabilities.py                                 # U0.1
apex/supervisor/events.py                            # U0.2 (REAL in Wave 0)
apex/supervisor/engine.py        (Protocol+types)    # U0.4 (frozen)
apex/supervisor/engine_fake.py   (FakeEngineClient)  # U0.4 (REAL in Wave 0)
apex/supervisor/engine_opencode.py                   # U1.7 (Linux-gated)
apex/supervisor/fabric.py                            # U1.5
apex/supervisor/budget.py        (CostBreaker)       # U1.2
apex/supervisor/core.py · checkpoint.py              # U2.1
apex/rails/__init__.py (Rails facade) · roe.py       # roe=U0.3, facade=U1.2
apex/rails/scope.py · scope_transport.py             # U1.1
apex/rails/kill.py·approval.py·audit.py·injection.py·destructive.py·escalation.py   # U1.2
apex/memory/schema.sql                               # U0.3 (REAL in Wave 0)
apex/memory/state.py (MemoryStore)                   # U1.3
apex/memory/vault.py·compaction.py·embeddings.py     # U1.4
apex/oracle/verify.py                                # U1.6
apex/agents/registry.py · agents/prompts/*.md        # U1.8
deploy/opencode/opencode.json · deploy/opencode/agents/ (rendered)   # U1.8
deploy/proxy/scope_gate.py        (mitmproxy addon)  # U1.1
apex/tools/catalog.py · apex/tools/parsers/*.py      # U1.9
apex/ui/bridge.py (rewrite) · ui/server.py (allowlist+) · ui/static/js/app.js (cases) · netmap   # U3.1/U3.2
tests/{rails,memory,fabric,engine,eval}/             # each worker + U4.1
```

Shared frozen modules every worker imports: `apex/supervisor/events.py`, `apex/types.py` (core dataclasses, §5), `apex/supervisor/engine.py` (protocol+types), `apex/rails/roe.py`. These are Wave-0, frozen.

---

## 2. CORE DATACLASSES — `apex/types.py` (Wave 0, FROZEN)
All shared value types live here so no worker re-invents them.

```python
from __future__ import annotations
from dataclasses import dataclass, field
from enum import Enum
from typing import Any

# ---- actions & destinations -------------------------------------------------
@dataclass(frozen=True)
class Destination:
    """Normalized network destination. BOTH ScopeGate enforcement points
    (mitmproxy addon + httpx transport) MUST build this identically (Fix #6)."""
    host: str                 # hostname OR ip literal, lowercased, no brackets
    port: int | None = None
    scheme: str = "https"     # http/https/tcp
    ip: str | None = None     # resolved/literal ip if known (used for CIDR/metadata checks)

    @staticmethod
    def from_url(url: str) -> "Destination": ...   # canonical parser (impl in scope.py, tested)
    @staticmethod
    def from_host_port(host: str, port: int | None, scheme: str = "https") -> "Destination": ...

class ActionKind(str, Enum):
    RECON = "recon"; PASSIVE = "passive"; PORT_SCAN = "port-scan"; ENUM = "enum"
    EXPLOIT = "exploit"; LATERAL = "lateral-move"; EXFIL = "exfil"
    CRED_ACCESS = "cred-access"; PRIVESC = "privesc"; DESTRUCTIVE = "destructive"; OTHER = "other"

@dataclass(frozen=True)
class Action:
    """A proposed action the Supervisor routes through the rails+fabric."""
    kind: ActionKind
    tool: str | None              # e.g. "nmap" (None for pure-reasoning)
    args: list[str]               # argv (already scope-safe-ish; rails re-check)
    target: Destination | None    # primary destination, if network-facing
    reversible: bool              # CALLER-SET, DETERMINISTIC (Supervisor tags from kind) — Fix #4
    objective_id: str | None = None
    rationale: str = ""

# ---- fabric decision types (Fix #4: freeze the PAYLOADS not just methods) ----
class Tier(str, Enum):
    GATE0 = "gate0"; T0 = "t0"; T1 = "t1"; T2 = "t2"; T3 = "t3"

@dataclass(frozen=True)
class DecisionRequest:
    kind: str                     # "next_action" | "parse" | "verify" | "go_no_go" | ...
    role: str                     # producing role (conductor/operator/...)
    objective_id: str | None
    candidates: list[dict] = field(default_factory=list)   # for ranking; may be empty
    evidence_ref: str | None = None
    reversible: bool = True       # CALLER-SET (Supervisor), drives stake-floor; Fabric never infers irreversibility
    stakes: str = "low"           # "low"|"medium"|"high"|"irreversible"  (caller-set)
    testable: bool = False        # is there an oracle/probe? → Gate-0
    probe: dict | None = None     # how to test it (tool+expectation) if testable

@dataclass(frozen=True)
class DecisionResult:
    tier: Tier
    chosen: dict | None           # the selected candidate / action dict
    alternatives: list[dict]      # runners-up kept as fallbacks
    why: str                      # human-readable rationale (→ `decision` event)
    confidence: float             # 0..1 (advisory only; gates use disagreement not this)
    needs_human: bool = False     # T3 go/no-go routed to ApprovalGate
    approval_id: str | None = None

# ---- oracle ----------------------------------------------------------------
@dataclass(frozen=True)
class Claim:
    kind: str                     # "port_open"|"host_up"|"shell_uid"|"schema_match"|...
    subject: str                  # what the claim is about (host, parsed-blob id)
    expectation: dict             # e.g. {"port": 445, "state": "open"}

@dataclass(frozen=True)
class Verdict:
    ok: bool
    confidence: float             # 1.0 for hard oracle, <1 for soft
    reason: str
    claim_kind: str = ""          # which claim this answers (Fix: Verdict says WHICH claim)
    evidence_ref: str | None = None

# ---- engine ----------------------------------------------------------------
@dataclass(frozen=True)
class TokenUsage:
    input: int; output: int; total: int
    cost_usd: float = 0.0
    cached_input: int = 0

@dataclass(frozen=True)
class Message:
    role: str                     # "assistant"
    content: str
    usage: TokenUsage             # ALWAYS present (Fix #8); for no_reply seeds usage=0s, content=""
    session_id: str
    permission_id: str | None = None   # set if the turn yielded a permission ask

# ---- rails verdict ---------------------------------------------------------
class RailDecision(str, Enum):
    ALLOW = "allow"; ASK = "ask"; DENY = "deny"

@dataclass(frozen=True)
class RailVerdict:
    decision: RailDecision
    rail: str                     # which rail produced it
    reason: str
    approval_id: str | None = None
```

---

## 3. EVENT STREAM — `apex/supervisor/events.py` (Wave 0, REAL)
The spine: one schema → WS-to-UI + `timeline.jsonl` + SQLite `events`/`audit` + replay.

```python
class EventType(str, Enum):
    AGENT_MESSAGE="agent_message"; DECISION="decision"; RAIL_EVENT="rail_event"
    APPROVAL_REQUEST="approval_request"; APPROVAL_RESOLVED="approval_resolved"
    ESCALATION_REQUEST="escalation_request"          # Fix #10 (distinct event; NOT folded into approval)
    TOOL_EVENT="tool_event"; MEMORY_EVENT="memory_event"
    ENTITY_GRAPH_DELTA="entity_graph_delta"; OBJECTIVE_TREE_DELTA="objective_tree_delta"
    COST_UPDATED="cost_updated"; OPSEC_CHANGED="opsec_changed"
    MODE_CHANGED="mode_changed"; KILLED="killed"

@dataclass(frozen=True)
class Event:
    id: str; type: EventType; ts: str; engagement_id: str; seq: int
    data: dict
    objective_id: str | None = None
    agent: str | None = None
    to: str | None = None         # agent | "all" | "you" | None  (threading target)

class EventBus:
    def emit(self, type, data, *, engagement_id, seq, objective_id=None, agent=None, to=None) -> Event
    def subscribe(self, pattern: str, cb) -> str       # "agent_message","rail_event","*"
    def unsubscribe(self, sub_id: str) -> None
    def add_sink(self, sink) -> None                   # TimelineSink(jsonl), SqliteSink
```

**`seq` ownership (Fix #10):** `seq` is allocated **solely by `MemoryStore.append_event()`** (monotonic per engagement, from the DB). The EventBus carries whatever the writer assigned. Flow: Supervisor builds the event payload → `MemoryStore.append_event()` assigns `seq` + persists → returns the `Event` → bus re-broadcasts it. Never allocate `seq` in two places.

**WS ENVELOPE MAPPING RULE (Fix #1) — FROZEN.** The UI bridge re-emits each internal `Event` to the websocket as:
```
{ "event": <Event.type value>, "data": { **Event.data, "agent": Event.agent, "to": Event.to,
                                          "seq": Event.seq, "id": Event.id, "ts": Event.ts } }
```
Rationale: `app.js` reads `const {event,data}=msg` and `data.agent` — top-level Event fields are invisible to it, so they MUST be folded into `data`.

### EventType `data` payloads (FROZEN)
| type | `data` fields |
|---|---|
| `agent_message` | `role, content, model` (+ `agent`,`to` from envelope) |
| `decision` | `tier, why, chosen, alternatives` |
| `rail_event` | `rail, action(allow/ask/deny/block/fire), subject, reason` |
| `approval_request` | `id, action, agent, tier, risk, detection_risk, created` |
| `approval_resolved` | `id, decision` |
| `escalation_request` | `id, reason, context, resume_token` |
| `tool_event` | `tool, args, status(running/success/error), duration, target` |
| `memory_event` | `kind(compaction/rehydrate/checkpoint/write), summary, tokens_before, tokens_after` |
| `entity_graph_delta` | `added:[Host], updated:[Host], removed:[ip], edges:[{src,dst,type}]` (see §9 Host shape) |
| `objective_tree_delta` | `node:{id,name,status,parent,depth}` |
| `cost_updated` | `total, per_model:{role:usd}` |
| `opsec_changed` | `score, posture` |
| `mode_changed` | `mode(approve-gated/autonomous)` |
| `killed` | `at, reason` |

**Wave-3 obligation (Fix #2):** `apex/ui/server.py::_wire_bridge_events` allowlist MUST gain `decision, rail_event, entity_graph_delta, objective_tree_delta, memory_event, escalation_request`. `app.js::handleEvent` MUST gain `case`s for them + a `default` that logs (so future events never vanish silently). "Keep server.py unchanged" was FALSE.

---

## 4. SQLITE DDL — `apex/memory/schema.sql` (Wave 0, REAL). SQLite-WAL, Supervisor = single writer.
Bi-temporal everywhere; **invalidate-don't-delete** enforced in `state.py` (UPDATE sets `valid_to`/`invalidated_at`, never DELETE).

```sql
PRAGMA journal_mode=WAL; PRAGMA foreign_keys=ON;

CREATE TABLE entities(
  id TEXT PRIMARY KEY, engagement_id TEXT NOT NULL, kind TEXT NOT NULL, key TEXT NOT NULL,
  label TEXT, attrs TEXT,                                   -- attrs = JSON
  valid_from TEXT NOT NULL, valid_to TEXT, observed_at TEXT NOT NULL, invalidated_at TEXT,
  content_hash TEXT, source TEXT, confidence REAL DEFAULT 0.5);
CREATE INDEX ix_entities_active ON entities(engagement_id, kind, key, valid_to);

CREATE TABLE edges(
  id TEXT PRIMARY KEY, engagement_id TEXT NOT NULL, src_id TEXT NOT NULL, dst_id TEXT NOT NULL,
  type TEXT NOT NULL, attrs TEXT,
  valid_from TEXT NOT NULL, valid_to TEXT, observed_at TEXT NOT NULL, invalidated_at TEXT, content_hash TEXT);
CREATE INDEX ix_edges_active ON edges(engagement_id, valid_to);

CREATE TABLE findings(
  id TEXT PRIMARY KEY, engagement_id TEXT NOT NULL, title TEXT, severity TEXT, host TEXT,
  description TEXT, tool TEXT, evidence_ref TEXT,
  valid_from TEXT NOT NULL, valid_to TEXT, observed_at TEXT NOT NULL, invalidated_at TEXT);

CREATE TABLE objectives(
  id TEXT PRIMARY KEY, engagement_id TEXT NOT NULL, parent_id TEXT, name TEXT NOT NULL,
  status TEXT NOT NULL, depth INTEGER NOT NULL, created_at TEXT NOT NULL, updated_at TEXT NOT NULL);

CREATE TABLE loot(
  id TEXT PRIMARY KEY, engagement_id TEXT NOT NULL, type TEXT, principal TEXT,
  secret_enc TEXT,                                          -- encrypted at rest (APEX_VAULT_KEY)
  source_host TEXT, tool TEXT, cracked INTEGER DEFAULT 0,
  valid_on TEXT,                                            -- JSON list of hosts (Fix #9)
  observed_at TEXT NOT NULL);

CREATE TABLE audit(                                         -- append-only (no UPDATE/DELETE)
  id TEXT PRIMARY KEY, engagement_id TEXT NOT NULL, ts TEXT NOT NULL,
  actor TEXT, action TEXT, subject TEXT, detail TEXT);      -- detail = JSON

CREATE TABLE events(                                        -- the timeline / replay source
  id TEXT PRIMARY KEY, engagement_id TEXT NOT NULL, seq INTEGER NOT NULL,
  ts TEXT NOT NULL, type TEXT NOT NULL, data TEXT NOT NULL, objective_id TEXT, agent TEXT, "to" TEXT,
  UNIQUE(engagement_id, seq));
CREATE INDEX ix_events_seq ON events(engagement_id, seq);

CREATE TABLE sessions(                                      -- live handles MUST be durable (§15)
  id TEXT PRIMARY KEY, engagement_id TEXT NOT NULL, agent TEXT,
  opencode_session_id TEXT, kind TEXT, live_handle TEXT, status TEXT,
  started_at TEXT, ended_at TEXT);
CREATE INDEX ix_sessions_ocid ON sessions(engagement_id, opencode_session_id);   -- Fix #9 (re-bind SSE on resume)

CREATE TABLE checkpoints(                                   -- deterministic crash-resume (Fix #9)
  id TEXT PRIMARY KEY, engagement_id TEXT NOT NULL, seq INTEGER NOT NULL,
  phase TEXT NOT NULL, objective_id TEXT, brief_json TEXT NOT NULL,
  broadcast_directive TEXT, pending_approvals_json TEXT, ts TEXT NOT NULL);

CREATE VIRTUAL TABLE findings_fts USING fts5(title, description, content='findings', content_rowid='rowid');
```

---

## 5. RAILS — deterministic, pure, Windows-unit-testable (§6, §23). Owner: U1.1 (scope), U1.2 (rest).

### 5.1 ScopeGate — `apex/rails/scope.py` (the §23 keystone; Fix #6)
```python
class ScopeGate:
    def __init__(self, roe: RoE): ...
    def decide(self, dest: Destination) -> RailVerdict:
        # ALWAYS-ALLOW control plane (roe.control_plane_allow: api.deepseek.com, search providers)
        # DENY 169.254.0.0/16 (cloud metadata) ALWAYS (even if a private range is in-scope)
        # DENY RFC1918 (10/8,172.16/12,192.168/16) + loopback + link-local UNLESS explicitly in roe.in_scope
        # DENY anything not matched by roe.in_scope (allow/exclude lists; exclude wins)
        # ALLOW in-scope (CIDR/wildcard-domain/exact)
    def rescope(self, new_hosts: list[str]) -> None: ...   # lateral-pivot re-run; only adds in-scope
```
Both enforcement points construct `Destination` via `Destination.from_url`/`from_host_port` then call `decide`. `deploy/proxy/scope_gate.py` (mitmproxy addon) → `flow.kill()` on DENY/ASK-without-approval. `apex/rails/scope_transport.py` (httpx `BaseTransport`) → raise `ScopeViolation` before I/O. **Test the matrix against `decide` directly (no proxy).**

### 5.2 Rails facade — `apex/rails/__init__.py` (Fix #5: eval order FROZEN)
```python
class Rails:
    scope; kill; approval; audit; injection; destructive; escalation; budget
    def authorize(self, action: Action, *, mode: str, tier: Tier) -> RailVerdict:
        # EVALUATION ORDER (short-circuit, ANY deny → DENY, no overrides):
        # 1. kill.guard()            -> if killed: raise/DENY immediately (a rail, not a prompt)
        # 2. destructive.check()     -> DENY if DoS/data-destruction outside authorized impact
        # 3. scope.decide(target)    -> DENY if out of scope
        # 4. budget.check()          -> DENY/ASK if cost breaker tripped
        # 5. approval.evaluate(action,mode,tier) -> AUTO|ASK|DENY  (the gate)
        # audit.record(...) every authorize() call regardless of outcome.
```

### 5.3 ApprovalGate truth table — `apex/rails/approval.py` (Fix #5: FROZEN)
`evaluate(action, mode, tier) -> RailDecision`. RoE injected at construction.

| condition | result |
|---|---|
| `action.kind` in `roe.forbidden_actions` | **DENY** (always, both modes) |
| `tier == T3` OR `not action.reversible` OR `action.kind` in `roe.require_human_actions` | **ASK** (always — stake-floor; even in autonomous mode) |
| `mode == "autonomous"` (and none of the above) | **ALLOW** |
| `mode == "approve-gated"` and `action.kind` in `roe.auto_actions` | **ALLOW** |
| `mode == "approve-gated"` (anything else) | **ASK** |

**Hard rails ON in BOTH modes:** kill/scope/destructive/budget never depend on `mode`. `mode` only relaxes the *approval* step, and never below the stake-floor.

### 5.4 Other rails (U1.2)
- `KillSwitch.fire(reason)` sets a hard flag + cancels the Supervisor's asyncio task group + emits `killed`. `guard()` is called at every action site & raises `Killed` if fired. `is_killed: bool`.
- `AuditLog.record(actor, action, subject, detail)` → append-only INSERT into `audit` + emit `rail_event` where relevant.
- `InjectionDefense.wrap_external(content, source) -> str` wraps tool/web/banner output in a DATA fence + neutralizes instruction-like spans (marks, does not execute). The corpus test asserts a wrapped poisoned banner cannot flip any rail.
- `NoDestructiveGuard.check(action) -> RailVerdict` (DENY `DESTRUCTIVE`/dos unless explicitly authorized in RoE — v1: never authorized).
- `Escalation.needs_human(condition, context) -> str` returns a `resume_token`, emits `escalation_request`, pauses the objective thread. (§6 human-fallback rail; v1 surfaces it, full handling thin.)
- `CostBreaker` (in `apex/supervisor/budget.py`): `add(role, usage)`, `check()->OK|WARN|TRIP`, `breakdown()->{total,per_model}`. WARN at `soft_usd` (alert), TRIP at `hard_usd` (pause+stop+emit rail_event).

### 5.5 RoE — `apex/rails/roe.py` (Wave 0)
```python
@dataclass
class RoE:
    engagement_id: str
    in_scope: list[str]; out_scope: list[str]           # CIDR/wildcard-domain/exact; exclude wins
    windows: list[tuple[str,str]]
    forbidden_actions: list[str]; auto_actions: list[str]; require_human_actions: list[str]
    data_handling: dict                                  # {llm_providers:[...], disclosed:bool}
    control_plane_allow: list[str]                       # ScopeGate always-allow
def load_roe(path: Path) -> RoE
def compile_allowlist(roe: RoE) -> list[str]             # → egress allowlist
```
`roe.yaml` schema: see §13 in master plan (in_scope/out_scope/windows/forbidden_actions/auto_actions/require_human_actions/data_handling/control_plane_allow).

---

## 6. MEMORY — `apex/memory/` (§15). Owner: U1.3 (state), U1.4 (vault/compaction).

### 6.1 MemoryStore — `apex/memory/state.py` (Fix #7: owns the single-writer queue)
```python
class MemoryStore:
    def __init__(self, db_path: Path, bus: EventBus): ...   # applies schema.sql; WAL
    # ALL writes serialize through ONE internal asyncio queue (single-writer; async-safe).
    async def upsert_entity(self, engagement_id, kind, key, *, label=None, attrs=None,
                            source=None, confidence=0.5) -> str    # content-hash dedup; bi-temporal
    async def add_edge(self, engagement_id, src_id, dst_id, type, attrs=None) -> str
    async def add_finding(self, engagement_id, *, title, severity, host, description, tool, evidence_ref=None) -> str
    async def add_loot(self, engagement_id, *, type, principal, secret, source_host, tool, valid_on:list[str]) -> str
    async def invalidate(self, table, id, reason) -> None       # sets valid_to/invalidated_at (NEVER delete)
    async def add_objective(self, engagement_id, name, *, parent_id=None) -> str
    async def set_objective_status(self, id, status) -> None
    async def append_event(self, engagement_id, type, data, **env) -> Event   # ALLOCATES seq; persists; returns Event
    async def checkpoint(self, engagement_id, *, phase, objective_id, brief, broadcast_directive, pending_approvals) -> None
    # READ (sync ok; compact/distilled; fetch-don't-hold):
    def query_state(self, engagement_id, kind, *, filters=None, as_of=None, active_only=True) -> list[dict]  # Fix #9: as_of time-travel
    def memory_map(self, engagement_id) -> dict        # {hosts:N, creds:M, findings:K, objectives:{...}, pointers}
    def get_network_map(self, engagement_id) -> dict   # {hosts:[Host], edges:[...]}  (see §9)
    def list_engagements(self) -> list[str]
    def load_latest_checkpoint(self, engagement_id) -> dict | None
```
**Single-writer test (§19 #5, BLOCKING):** spawn N concurrent `upsert_entity`/`add_finding` coroutines → assert no corruption, no "database is locked", monotonic `seq`. Runs on Windows.

### 6.2 Vault + Compaction — `apex/memory/{vault,compaction}.py`
```python
class Vault:                       # git-backed Obsidian view; render-only (SQLite is truth)
    def render_entity(self, e) -> None; def render_index(self, engagement_id) -> None
    def commit(self, msg) -> None
class Compaction:                  # Supervisor-owned pager (deterministic)
    def should_compact(self, tokens, model_ctx, *, at_boundary) -> bool   # absolute sharp-zone target (config)
    def build_brief(self, engagement_id, objective_id) -> dict
        # deterministic: objective+parent chain · active target top-N entities · open leads ·
        # recent decisions · OPSEC posture · ACTIVE BROADCAST DIRECTIVE (must survive reset) ·
        # pending approvals · already-tried-here (anti-repetition) · memory_map manifest
    async def compact(self, engine, session_id, engagement_id, objective_id) -> str
        # persist-first → build_brief → engine.reset_session(seed=brief) → return new sid; emit memory_event
```
`embeddings.py` = fastembed bge-small wrapper (`HF_HUB_OFFLINE=1`); thin in v1 (episodic recall = v1.1).

---

## 7. FABRIC — `apex/supervisor/fabric.py` (§16). Owner: U1.5.
```python
class ModelRegistry:               # §19 per-role registry; DeepSeek defaults, hot-swappable
    def for_role(self, role) -> ModelSpec      # {model:"provider/id", thinking:"on|off|adaptive|contested", caps}
    def set_role_model(self, role, model) -> None
class Fabric:
    def __init__(self, engine, oracle, registry, rails, bus): ...
    def classify(self, req: DecisionRequest) -> Tier:
        # Gate-0 if req.testable and req.probe  (oracle, NEVER debate)
        # T3 if req.stakes=="irreversible" or not req.reversible  (STAKE-FLOOR; Supervisor-tagged, Fabric never infers)
        # else T0 default; T1 if flagged sample; T2 if medium-stake
    async def route(self, req: DecisionRequest) -> DecisionResult:
        # GATE0 -> oracle.verify; T0 -> 1 flash pass; T1 -> sample k≈3 disagreement-gated;
        # T2 -> 1 pro critic (anonymized, different role); T3 -> generate N(3-5)→reflect→rank(listwise)→evolve(≤2)→ApprovalGate
        # emits a `decision` event (tier+why) every call.
```
**Cheap-by-default cost-regression test (§19 #3):** a routine engagement stays mostly Gate0/T0.

---

## 8. ENGINE — `apex/supervisor/engine.py` (Protocol, Wave 0) + `engine_fake.py` (REAL, Wave 0) + `engine_opencode.py` (U1.7).
```python
class EngineClient(Protocol):
    async def create_session(self, agent: str, system: str, model: str) -> str
    async def prompt(self, session_id: str, content: str, *, no_reply: bool=False) -> Message
        # no_reply=True (seed): returns Message(content="", usage=TokenUsage(0,0,0), ...) — Fix #8
    def prompt_async(self, session_id: str, content: str) -> "AsyncIterator[StreamEvent]"
        # yields StreamEvent(kind="token"|"permission_asked"|"usage"|"done", ...); usage carried in stream — Fix #8
    def token_usage(self, session_id: str) -> TokenUsage
    async def reply_permission(self, perm_id: str, decision: str) -> None     # decision in {"once","always","reject"}
    async def reset_session(self, session_id: str, system: str) -> str        # §15 epoch rollover (new sid)
    async def close(self) -> None
```
**FakeEngineClient (Fix #8) — a first-class CONTRACT, REAL in Wave 0.** Scripted, deterministic. Script format supports steps:
- `say{content, usage}` → assistant turn.
- `yield_permission{id, action, tier}` → emits a `permission_asked` StreamEvent carrying `id`, then **suspends** until `reply_permission(id,...)` is called; resumes per the decision. (This is what makes "approval-before-exploit" expressible.)
- a **drivable rising token counter** so `Compaction.should_compact` can be forced. `token_usage()` and per-turn `Message.usage` read the SAME counter.

**`tests/engine/test_engine_contract.py` (Fix #8):** parametrized over `[FakeEngineClient (always), OpenCodeEngineClient (skipif not LINUX or not opencode)]` — same assertions for both, so the Fake is held to the real client's semantics. `engine_opencode.py` adds the **deny-rail smoke-test** (bugs #6396/#15386: a `deny`d tool is genuinely blocked via the SDK).

---

## 9. UI CONTRACT (§20). Owner: U3.1 (bridge/server), U3.2 (frontend). Re-wire, do NOT rebuild.
- **`snapshot()` return (Fix #10):** `Supervisor.snapshot()` returns a dict covering the **v1 subset** of `protocols.py`'s 27 read methods; v2-only reads (`get_implants`/`get_techniques`/`get_defender_profile`) keep the existing safe-empty defaults. Keys: `model_health, opsec_score, engagement_state, cost_breakdown, approval_queue, network_map, attack_tree, orchestra, loot, mode, killed`.
- **Bridge re-point:** collapse the 8 backend params → ONE `supervisor`. Subscribe to `EventBus`; re-emit via the §3 WS-envelope rule. Reads via `supervisor.snapshot()` / `store.query_state()`.
- **Netmap (Fix #1+#2+#3, Critic items 1&2):** `entity_graph_delta.data` carries `Host` rows. **`Host` shape = `{"ip": str|None, "hostname": str|None, "os": str|None, "ports": [int]}`.** The bridge ACCUMULATES deltas into an in-memory graph; `get_network_map()` returns `{"hosts":[Host...], "edges":[...]}` (the snapshot `netmap.js:91` consumes). **v1 netmap renders `hosts` and derives edges client-side** (netmap.js:104 recomputes edges); bridge `edges` are carried in the contract but **NOT rendered in v1** (real entity-graph edges = a known v1.1 netmap upgrade — documented, not silent loss). `invalidated_at` host → emit in delta `removed:[ip]` → bridge drops it from the accumulated snapshot.
- **Main feed driver (Critic item 3):** the orchestration feed is **push-driven by `agent_message` + `decision`**. The legacy `task_event`/`message_sent` full-reload paths are **deprecated/removed** in Wave 3 (no double-render). One push path.
- **Frontend (v1 subset only):** add `app.js handleEvent` cases for `decision, entity_graph_delta, objective_tree_delta, rail_event, escalation_request, memory_event` + a `default`. Render the orchestration feed + netmap + objective tree + status-bar mode/kill/approval (already present). DEFER to v1.1: Chats per-agent threads, full Loot export, model-swap UI. Keep `python -m apex.ui.demo` working (refresh its sim only if roster/events drift).

---

## 10. PORTABILITY & PACKAGING (§23). Owner: U0.1.
- `pyproject.toml`: PEP 621 `[project]`, `requires-python=">=3.12"`, `[project.scripts] apex="apex.cli:main"` + `apex-serve="apex.cli:serve"`, `[tool.uv] package=true`, hatchling build backend, `[project.optional-dependencies] linux-tools=[...; sys_platform=='linux']`, `[dependency-groups] dev=[pytest,pytest-asyncio,ruff,mypy,...]`. Build-backend data includes: `apex/ui/static/**`, `apex/memory/schema.sql`, `apex/data/seed/**`, `apex/tools/configs/**`, `apex/agents/prompts/**`.
- `.python-version` = `3.12`. Bootstrap: `uv python install 3.12 && uv sync`. Run: `uv run python -m apex`.
- `apex/capabilities.py`: `LINUX = sys.platform=="linux"`; `HAS_NMAP = shutil.which("nmap") is not None`; `have(tool)->bool`. Core logic + Supervisor route on availability; `pytest.mark.skipif(not LINUX)` gates tool-integration tests.
- Runtime data via `importlib.resources.files("apex")` (never `__file__`); extract to `~/.local/share/apex` (XDG) / `%APPDATA%\apex` on first run if needed.
- `install/bootstrap.sh` + `install/bootstrap.ps1`: ensure uv → `uv sync` → ensure Node + `opencode-ai@1.15.12` → optional docker → optional service unit. Skeletons in Wave 0.
- Proxy = `mitmdump` (pip, explicit mode) — NOT a container in v1. Process supervision = `psutil.Popen` + asyncio in Supervisor. Qdrant deferred to v2.

---

## 11. ACCEPTANCE TEST (§18 v1.0 first-slice = the DONE-GATE). Owner: U4.1.
`tests/eval/test_v1_acceptance.py` (Windows/FakeEngine) drives the full flow; each clause → an assertion:
1. **Load RoE** → `load_roe()`+`compile_allowlist()` build the ScopeGate; an out-of-scope `Destination` is DENIED.
2. **Conductor plans** → `objective_tree_delta` events + `objectives` rows form a tree.
3. **Operator+Analyst recon** → `tool_event`(nmap, via FakeEngine scripted output) → Analyst parse → `findings` row; Analyst T0 schema-oracle passes.
4. **Entity graph in SQLite** → `entities`/`edges` rows; `entity_graph_delta` events fired; `get_network_map()` returns hosts.
5. **Memory survives forced compaction+rehydrate** → drive the Fake token ramp → `Compaction.compact()` → reseed → `query_state` STILL returns the creds/objective/tried-set (lossless).
6. **Approval-before-exploit; deny-rail verified** → an `EXPLOIT` action (require_human) → `ApprovalGate` → ASK → `approval_request` emitted; a DENY decision → the tool does NOT run (Fake `yield_permission` rejected). (Linux: live deny-rail smoke-test in `test_engine_contract.py`.)
7. **Captures proof** → `Oracle.verify(shell_uid)` ok → `findings` + evidence; loot encrypted-at-rest, reveal audit-logged.
8. **Audit + event stream intact** → `audit` append-only + `events`(seq-ordered) + `timeline.jsonl` replay reconstructs the run; WS broadcast delivered each event.
Plus BLOCKING suites green: rails (scope matrix · injection corpus · kill-immediate · approval truth-table · no-destructive · cost trip · hard-rails-both-modes) · memory (compaction-lossless · crash-resume · single-writer-concurrency · bi-temporal as_of) · fabric (oracle-first · stake-floor→T3 · cheap-by-default). And `python -m apex` boots the UI on Windows (FakeEngine); `python -m apex.ui.demo` still works.

---

## 12. TESTABILITY RULES
- Inject a `clock: Callable[[], datetime]` and an `id_gen: Callable[[], str]` into MemoryStore/Supervisor/EventBus so tests are deterministic (no wall-clock/uuid in assertions). Default to real ones.
- No network/Linux-tool/proxy/OpenCode dependency in the BLOCKING suites — FakeEngine + `capabilities.have()`-False cover them.
- Each Wave-1 worker delivers code AND tests; a module is "done" only when its tests are green.

---

## 13. THE 13 FROZEN FIXES (checklist — every worker confirms compliance)
1. WS envelope mapping rule (§3). 2. 5+escalation new events wired in server.py allowlist + app.js cases + default (§3/§9). 3. Netmap delta→accumulated-snapshot in bridge; v1 renders hosts (§9). 4. DecisionRequest/Result/Verdict frozen; reversibility/stakes CALLER-set, Fabric never infers (§2/§7). 5. ApprovalGate truth table + Rails eval order frozen (§5.2/§5.3). 6. ScopeGate.decide(Destination), identical construction both points (§2/§5.1). 7. Single-writer queue in MemoryStore, async-safe (§6.1). 8. EngineClient usage-in-Message+stream, FakeEngine yield_permission, prompt(no_reply) return, parametrized engine contract test (§8). 9. query_state as_of; structured checkpoints row; sessions ocid index; loot.valid_on JSON (§4/§6.1). 10. escalation_request event; seq allocated only by MemoryStore; snapshot() covers v1 read subset + safe-empty rest (§3/§9). 11. Netmap edges carried-not-rendered in v1 (documented, Critic#1). 12. entity_graph_delta Host field mapping frozen (Critic#2/§9). 13. Main feed push-driven by agent_message+decision; task_event/message_sent deprecated (Critic#3/§9).

---

## 14. WAVE-2 CARRY-OVER (from the opus Wave-1 review, 2026-05-29)
Wave-1 modules built + reviewed; module-local findings (F1 facade typing, F2 kill-path audit, F4 embeddings typing, F6 DoS taxonomy) are FIXED. These three are integration/wiring concerns to implement during Wave 2 (the Supervisor seam):
- **F3 — Event-spine persistence (REQUIRED for the §11 acceptance test):** Today `MemoryStore.append_event` correctly persists+allocates seq, but `ApprovalGate.enqueue/resolve`, `KillSwitch.fire`, `CostBreaker._emit_rail_event`, `Escalation.needs_human`, and `Compaction.compact` call `bus.emit(..., seq=0)` directly → not persisted, bogus seq. FIX (persist-hook): make `EventBus.emit` take `seq=None` (optional) + `set_persist_hook(fn)`; when seq is None and a hook is set, the hook (`MemoryStore.persist_event`, sync, allocates MAX(seq)+1 + INSERT + returns seq) persists + returns the real seq. `MemoryStore.__init__` wires `bus.set_persist_hook(self.persist_event)`; `append_event` reuses `persist_event` then emits with explicit seq (no double-persist). Drop `seq=0` from the component emits. Net: EVERY event persists with a monotonic seq → replay/audit intact.
- **F5 — Scoped kill cancel:** `KillSwitch.fire` currently cancels `asyncio.all_tasks()` (would tear down the UI/event loop that must REPORT the kill). FIX: the Supervisor registers its engagement task-group/cancel-scope with the KillSwitch; `fire()` cancels only that scope (default: cancel nothing, rely on the synchronous `guard()` at action sites).
- **F7 — Loot reveal audit:** `MemoryStore.reveal_secret` isn't audit-logged. FIX: the Supervisor/bridge routes reveals through `AuditLog.record` (§11 clause 7).
