# APEX v1.1 — FROZEN METHODOLOGY-DOCTRINE CONTRACTS (2026-05-30)

> Single source of truth for the **Methodology Doctrine** layer — the standing property that
> EVERY orchestra agent thinks and acts like an elite senior red-team operator, grounded in
> real, cited methodology (PTES / MITRE ATT&CK / kill-chain / threat-led RT / OPSEC doctrine /
> senior-operator decision craft). Realizes the user directive "always follow elite senior red
> teamer methodologies all across every agent." IMPLEMENTS settled design (§3/§4/§16/§17) — does
> NOT re-open it. Built from 3 cited deep-research streams (frameworks · per-domain ATT&CK-mapped
> playbooks · senior-operator decision doctrine) whose sources + `[UNVERIFIED]` register live in
> `apex/agents/doctrine/SOURCES.md`.

## 0. CORE PRINCIPLE
Methodology is a **standing, evolvable artifact** injected into every agent's prompt — NOT 8 static
prompt edits. One canon → all agents inherit it → a refinement loop (Meta-review + research) keeps
improving it. This mirrors the §3 broadcast-directive + §15 WORKING_MEMORY patterns.

**No-hallucination law (standing):** the doctrine CONTENT is research-grounded; every non-obvious
methodology claim traces to a real source in `SOURCES.md`; contested items are tagged `[UNVERIFIED]`
there. The doctrine REINFORCES (never weakens) the GroundingGate ("proven, not claimed") and the
code-enforced escalation rail (research independently confirms: escalation must be rail-enforced,
not left to model discretion — Unit42/Wiz/Microsoft 2026).

## 0.1 GROUND RULES
- Python 3.12, `from __future__ import annotations`, full hints, ruff+mypy clean, pure/offline.
- Doctrine canon = markdown DATA under `apex/agents/doctrine/` (packaged via build-backend includes,
  read via `importlib.resources` — never `__file__`, §10/§23). Token-lean: clean directives, no inline
  citation noise (citations live in the non-injected SOURCES.md).
- NEVER raises: a missing/garbled doctrine file degrades to "" (agents still work, just without the
  enrichment). Doctrine is advisory enrichment, never a hard dependency.

## 1. DOCTRINE CANON (markdown data — `apex/agents/doctrine/`)
- `CORE.md`        — cross-cutting senior-operator decision doctrine (hypothesis-driven · anti-sunk-cost
                      persist-vs-pivot · EV prioritization · grounding law · chaining · purple-team ·
                      blast-radius/scope · honest-failure/escalate · reporting). Injected into EVERY agent.
- `ROLES.md`       — per-role MINDSET + numbered DECISION RULES for all 8 roles. `for_role` injects the
                      matching block.
- `FRAMEWORKS.md`  — the methodology map (PTES 7 phases · ATT&CK 14 tactics in kill-chain order + tag-every-
                      action-with-a-technique-ID · CKC vs Unified Kill Chain · threat-led RT · adversary
                      emulation vs simulation · Diamond Model · OWASP WSTG v4.2). Shared "where are we" map.
- `PHASES.md`      — APEX Phase enum (INTAKE/RECON/FOOTHOLD/POST_EX/REPORT/CLEANUP) → PTES+UKC mapping +
                      the senior-operator focus for each. `for_phase` injects the current-phase block.
- `DOMAINS.md`     — per-domain ordered playbooks (AD · Web/API · Cloud[AWS/Azure/GCP] · network-pivot ·
                      privesc · lateral · cred-access) with real ATT&CK IDs. `for_domain` injects the block
                      matching the §8 knowledge fingerprint (ties doctrine ↔ knowledge system).
- `OPSEC.md`       — OPSEC doctrine (noise/detection budget · detection-risk modeling · hard-block→pivot ·
                      counter-detection response [burn vs quiet vs pivot-infra] · ROE-bounded never-destructive
                      cleanup · threat-emulation fidelity · purple-team detection mapping).
- `SOURCES.md`     — NOT injected. The citations + `[UNVERIFIED]` register (audit trail / no-hallucination
                      accountability).
- `WORKING_DOCTRINE.md` — the EVOLVING amendments block (starts ~empty). The refinement loop (Meta-review)
                      appends validated methodology deltas; `compose()` includes it so the orchestra improves
                      on a loop. Git-tracked → every methodology change is an auditable diff.

## 2. `apex/agents/doctrine.py` — `MethodologyDoctrine`
```python
class MethodologyDoctrine:
    def __init__(self, *, base_dir: "Traversable | Path | None" = None) -> None:
        """Loads canon from apex/agents/doctrine/ via importlib.resources (lazy, cached).
        base_dir override for tests. NEVER raises on a missing file → that section = ""."""
    def core(self) -> str: ...
    def for_role(self, role: str) -> str:
        """CORE + ROLES[role] + WORKING_DOCTRINE amendments — the block appended to a role's
        system prompt. Unknown role → CORE only."""
    def for_phase(self, phase: str) -> str:
        """PHASES[phase] block (RECON/FOOTHOLD/...). Unknown/empty phase → ""."""
    def for_domain(self, domain: str) -> str:
        """DOMAINS[domain] block; loose match (e.g. 'web'→web_application section, like the
        knowledge domain filter). Unknown → ""."""
    def frameworks(self) -> str: ...
    def compose(self, *, role: str, phase: str | None = None, domains: list[str] | None = None,
                max_chars: int | None = None) -> str:
        """The full doctrine block injected per turn: for_role(role) + for_phase(phase) +
        for_domain(d) for d in domains. Token-bounded by max_chars (truncate at a section
        boundary). NEVER raises."""
    def append_amendment(self, text: str) -> None:
        """Refinement loop: append a validated methodology delta to WORKING_DOCTRINE.md
        (dedup; bounded). Used by Meta-review. Best-effort; never raises."""
```
Default singleton accessor `default_doctrine() -> MethodologyDoctrine` (cached).

## 3. INTEGRATION SEAMS
- **AgentRegistry.system_prompt(role)** (`apex/agents/registry.py`): return `base_md + "\n\n" +
  doctrine.for_role(role)` when a doctrine is available (optional `doctrine` attr on the registry,
  default `default_doctrine()`; a `None`/disabled doctrine → base only → zero regression for tests that
  assert on the raw .md). Add a flag/param so existing `render_agent_files`/`system_prompt` tests can opt out.
- **Supervisor** (`apex/supervisor/core.py`): construct/hold `self.doctrine` (always-on, like grounding).
  In the ORIENT seam (where `[KNOWLEDGE]` + `[RECALL]` blocks are built) ALSO inject
  `doctrine.for_phase(self.phase.value)` + `doctrine.for_domain(d)` for the fingerprinted domains
  (reuse the knowledge `SituationTags.domains`) into the Strategist/Conductor context. Guarded/non-fatal.
  Construct in `build_supervisor`; pass `doctrine=` to the Supervisor (new optional param, default the
  singleton). The 8-role system prompts already carry `for_role` via the registry.
- **Refinement loop** (§4 below).

## 4. REFINEMENT LOOP (the "research deeply on a loop" runtime)
- **Meta-review → doctrine:** the LEARN step (Meta-review) already emits lessons. Add: when a lesson is a
  durable *methodology delta* (not engagement-specific trivia), the Supervisor routes it to
  `doctrine.append_amendment(...)` → `WORKING_DOCTRINE.md`. All agents pick it up next turn. Bounded +
  deduped; git-tracked = auditable.
- **Research loop hook:** a `MethodologyResearchLoop` scaffold (`apex/agents/doctrine_loop.py`) that, given
  the §8 `Knowledge` facade, can (a) detect doctrine gaps (e.g. the corpus gaps R3 flagged: no
  network_pivot/lateral/cred seed files; stale OWASP versioning) and (b) queue research to fill them. v1.1 =
  the loop INTERFACE + gap-detection + a manual `run_once()`; continuous/cron scheduling is a documented
  next step (NOT auto-run — costs tokens/network). Skip-safe offline.

## 5. TESTS
- `tests/agents/test_doctrine.py`: canon loads; for_role(each of 8) non-empty + contains role cues;
  for_phase/for_domain loose-match + unknown→""; compose token-bounds; append_amendment persists+dedups;
  never-raises on a missing dir (base_dir=tmp). registry.system_prompt(role) now CONTAINS the doctrine
  block; render_agent_files still works. Supervisor still builds + runs (acceptance unaffected — doctrine is
  additive prompt text the FakeEngine ignores). Offline/Windows-green.

## 6. FROZEN RULES
1. Doctrine = evolvable DATA injected into every agent; ONE canon, not 8 edits. 2. Research-grounded +
cited (SOURCES.md) + `[UNVERIFIED]` honesty — no fabricated methodology. 3. Reinforces grounding +
code-enforced escalation; never weakens a rail. 4. Token-lean (directives in canon, citations out-of-band),
never-raises, offline. 5. Refinement loop is opt-in/bounded; WORKING_DOCTRINE is git-tracked. 6. Zero
regression: doctrine is additive prompt text; FakeEngine-driven tests are unaffected.
