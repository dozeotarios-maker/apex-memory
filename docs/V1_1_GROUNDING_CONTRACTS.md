# APEX v1.1 — FROZEN GROUNDING / ANTI-HALLUCINATION CONTRACTS (2026-05-30)

> Single source of truth for the **grounding (anti-hallucination) layer** in the agent
> orchestra. IMPLEMENTS settled mandates — does NOT re-open design:
> **§5** "VERIFY: proven, not claimed — capture evidence" (currently a STUB at
> `core.py` ~L630), **§6** honest-failure ("faked or low-confidence work is worse than
> asking") + the no-destructive guardrail + human-fallback rail, **§3/§13** "rails are
> deterministic code; the LLM is never trusted to police itself on what matters."
> FROZEN before the worker fan-out. Code only against the signatures below + the
> already-written frozen types in `apex/types.py`.

---

## 0. THE PRINCIPLE (zero hallucination tolerance)
Every **material claim** an orchestra agent makes — a **finding**, a discovered **entity**
(host / ip / port / service), **loot** (cred), a **success/foothold**, an **objective
completion** — must be **grounded in the evidence the agent was shown** BEFORE it is
persisted as a FACT. Grounding is decided by **deterministic code** (`GroundingGate`),
never by an LLM (an LLM can't be trusted to catch its own confabulation).

"No hallucination threshold allowed" = **strict**: a claim whose concrete artifacts are
absent from (or contradicted by) the evidence is **NOT persisted as a confirmed fact** —
it is rejected, audited (`rail_event`), and surfaced for the human (honest-failure §6).
Recon leads are NOT lost: unconfirmed-but-not-contradicted claims persist as low-confidence
**hypotheses** (so the loop can chase them — §8 "never dead-ended"), but never as confirmed
facts. Only **CONFIRMED** claims become facts.

## 0.1 GROUND RULES
- Python 3.12, `from __future__ import annotations`, full hints, `ruff`+`mypy` clean.
- Pure sync, no I/O, no network, no LLM — Windows-unit-testable (like every other rail, §5/§23).
- NEVER raises on malformed claim/evidence — degrade to UNVERIFIED + log.
- Disjoint file ownership; each module ships its tests; green before "done".

## 1. FROZEN TYPES (already in `apex/types.py` — do NOT edit)
`GroundingStatus(CONFIRMED|UNVERIFIED|HALLUCINATED|CONTRADICTED)` ·
`GroundingVerdict(status, grounded, confidence, reason, claim_kind, evidence_ref, missing)`.
Also reuse existing `Claim`, `Verdict`, and the `Oracle` (`apex/oracle/verify.py`:
`verify(claim, evidence)->Verdict`, hard kinds port_open/host_up/shell_uid/schema_match).
RoE now carries `authorized_destructive: list[str]` (default `[]`).

---

## 2. WORKER G1 — `apex/rails/grounding.py` (the gate)
```python
class GroundingGate:
    """Deterministic anti-hallucination gate. Grounds agent claims against evidence
    BEFORE they become facts. Pure/sync; reuses the Oracle for testable claims and
    grounding-by-presence for the rest."""

    def __init__(self, oracle: "Oracle | None" = None, *, strict: bool = True) -> None: ...

    # -- oracle-backed (testable) claim ------------------------------------
    def assess_claim(self, claim: Claim, evidence: dict) -> GroundingVerdict:
        """If claim.kind is oracle-testable, oracle.verify(): ok→CONFIRMED(conf 1.0);
        not-ok→CONTRADICTED. If the oracle raises ValueError (unknown kind) or oracle is
        None → fall back to UNVERIFIED. evidence_ref carried through from Verdict."""

    # -- grounding-by-presence (the anti-hallucination core) ---------------
    def assess_finding(self, *, title: str, host: str | None, description: str,
                       evidence_text: str, claimed_values: list[str] | None = None,
                       evidence_ref: str | None = None) -> GroundingVerdict:
        """Ground a finding against the raw tool output (`evidence_text`) the Analyst
        parsed. The set of CONCRETE checkable tokens = claimed_values (if given) PLUS the
        host + any IPs/ports/CVE-ids/hostnames extractable from title+host+description.
        Rule:
          - evidence_text empty/blank        → UNVERIFIED (cannot confirm; not a fact).
          - every concrete token present in normalised evidence_text → CONFIRMED
            (confidence = present/total coverage, min 0.6).
          - one or more concrete tokens ABSENT from evidence_text → HALLUCINATED
            (missing = the absent tokens). In strict mode this is a REJECT.
        Matching is case-insensitive substring on a normalised copy of evidence_text.
        A finding with NO concrete tokens at all (pure prose) → UNVERIFIED (cannot ground)."""

    def assess_entity(self, *, kind: str, key: str, evidence_text: str,
                      attrs: dict | None = None, evidence_ref: str | None = None) -> GroundingVerdict:
        """The entity key (ip/host/etc.) MUST appear in evidence_text. Present→CONFIRMED;
        absent (and evidence non-empty)→HALLUCINATED(missing=[key]); evidence empty→UNVERIFIED.
        attrs ports/values, if present, are added to the concrete-token set."""

    def assess_loot(self, *, principal: str, source_host: str, evidence_text: str,
                    evidence_ref: str | None = None) -> GroundingVerdict:
        """principal + source_host must appear in evidence_text. Same CONFIRMED/HALLUCINATED/
        UNVERIFIED rules. (Secrets themselves are NOT required to appear verbatim — they may be
        masked in the operator output — so only principal+source_host are grounded.)"""
```
**Token extraction helpers** (module-level, tested): `extract_ips(text)->list[str]` (IPv4 regex),
`extract_ports(text)->list[str]`, `extract_cves(text)->list[str]` (CVE-\d{4}-\d+), plus a
`normalise(text)->str` (lower, collapse whitespace). Keep these conservative — only ground
HIGH-CONFIDENCE concrete artifacts; do NOT try to ground free prose (false-positive risk).

**`strict` flag:** `strict=True` (default) → HALLUCINATED/CONTRADICTED carry `grounded=False`
(caller rejects). `strict=False` is reserved for future tuning; behaviour identical in v1
except it MAY be used by the caller to downgrade rather than drop. The GATE only classifies;
the CALLER (Supervisor) decides persist/drop/escalate per §4.

**Tests (`tests/rails/test_grounding.py`):** oracle port_open ok→CONFIRMED, not-ok→CONTRADICTED,
unknown-kind→UNVERIFIED (no raise); finding whose IP is in the nmap text→CONFIRMED; finding
asserting an IP NOT in the text→HALLUCINATED(missing=[ip]); empty evidence→UNVERIFIED;
entity key present/absent; loot principal present/absent; extract_ips/ports/cves; never raises
on weird input.

## 3. WORKER G2 — reinforce `apex/rails/destructive.py` (NoDestructiveGuard)
"No destructive behaviour unless we ask it to." Add an RoE-grant path; keep the default DENY.
```python
class NoDestructiveGuard:
    def __init__(self, roe: "RoE | None" = None) -> None:
        # store roe.authorized_destructive (default []) — the explicit grant list.
    def check(self, action: Action) -> RailVerdict:
        # action.kind in (DESTRUCTIVE, DOS):
        #   - if action.kind.value in authorized_destructive → ALLOW
        #       (reason: "explicitly authorized in RoE; human approval still required").
        #       NOTE: destructive actions are reversible=False, so the downstream
        #       ApprovalGate stake-floor forces a human ASK on EVERY instance — even in
        #       autonomous mode. The grant authorizes the CLASS; the human authorizes each ACT.
        #   - else → DENY (unchanged v1 behaviour).
        # not destructive → ALLOW (unchanged).
```
**No change to the Rails facade eval order or the ApprovalGate truth table** — the grant
returns ALLOW, then scope/budget/approval run normally and approval's `not reversible → ASK`
stake-floor guarantees the human gate. Construction site (`build_supervisor`,
`core.py` ~L1241) becomes `NoDestructiveGuard(roe)`; the no-arg form must still work (default
no grants). The `DestructiveLike` protocol in `rails/__init__.py` is unchanged (`check` only).
**Tests (extend `tests/rails/test_destructive.py`):** ungranted DESTRUCTIVE→DENY; ungranted
DOS→DENY; granted (roe.authorized_destructive=["destructive"]) DESTRUCTIVE→ALLOW; granted but
DOS-not-listed→DENY; non-destructive→ALLOW; no-arg constructor still DENYs. (Also add ONE
end-to-end rails test: a granted destructive action through `Rails.authorize` in autonomous
mode → final decision ASK, not ALLOW — proving the human gate holds.)

## 4. CALLER WIRING (lead authors in `core.py`; workers do NOT touch core.py)
- **ORIENT persist-gate** (`run_ooda_step`, the entity/finding persistence ~L351-389): the raw
  operator output (`op_msg.content`, wrapped via `rails.injection.wrap_external`) is the
  evidence. Before `upsert_entity`/`add_finding`, call the GroundingGate. CONFIRMED → persist
  (entity confidence = verdict.confidence; finding evidence_ref set). HALLUCINATED/CONTRADICTED
  → DO NOT persist; emit `rail_event(rail="grounding", action="block", subject, reason)` +
  `AuditLog.record` + `Escalation.needs_human("hallucinated_claim", ctx)`. UNVERIFIED → persist
  entity at low confidence (≤0.4) as a hypothesis; finding severity downgraded to "unverified".
- **VERIFY phase** (`_execute_action`, the L629-632 stub): if the action produced a testable
  success claim (e.g. shell_uid post-exploit, port_open post-scan), build a `Claim` and run
  `GroundingGate.assess_claim` against the parsed tool output; record success/finding ONLY when
  CONFIRMED. Emit a `rail_event`/`memory_event` with the verdict. Never assert success on a
  non-CONFIRMED claim.
- The GroundingGate is constructed in `build_supervisor` (always on — it's a safety rail, not a
  feature flag) and stored on the Supervisor (`self.grounding`). It is NOT gated by
  `enable_knowledge`.

## 5. WORKER G3 — agent prompt hardening (`apex/agents/prompts/*.md`)
Add a shared **GROUNDING CONTRACT** block to each of: `analyst.md`, `operator.md`,
`strategist.md`, `reviewer.md`, `conductor.md`, `meta.md` (insert near each file's existing
boundaries/output-contract section; keep each file's existing structure + JSON keys intact —
ADD, don't rewrite). The block (adapt the wording to each role's voice, keep the substance):
```
## GROUNDING CONTRACT (anti-hallucination — non-negotiable)
- Report ONLY what is present in the evidence you were given. NEVER invent or assume hosts,
  IPs, ports, services, credentials, versions, CVEs, or findings.
- Every concrete artifact you assert (an IP, a port, a hostname, a cred principal) MUST appear
  in that evidence. A deterministic GroundingGate cross-checks every claim against the raw
  evidence and REJECTS + logs any claim whose concrete values are absent — fabrication is
  caught, never silently accepted.
- If you cannot verify a claim, say so explicitly ("unverified" / "hypothesis") or escalate.
  Honest failure beats confident fabrication: faked or low-confidence work is worse than asking
  (§6). When you infer rather than observe, label it an inference, not a fact.
- No destructive actions (data destruction / DoS) unless the RoE explicitly authorizes them AND
  a human approves each one.
```
Strategist/Ranker (the weakest today) especially need the "never propose capabilities the
target does not have; ground in observed entities or oracle-tested facts" line. No tests
(prose); the wiring/integration test in §4 covers the behaviour.

## 6. FROZEN RULES
1. Grounding is CODE, never an LLM self-check. 2. Strict: only CONFIRMED → fact; HALLUCINATED/
CONTRADICTED → reject+audit+escalate; UNVERIFIED → hypothesis (low-conf), never asserted.
3. Pure/sync/offline/never-raises. 4. Default no-destructive = DENY; grant = RoE
`authorized_destructive` + still human-ASK per instance. 5. Frozen `types.py`/`roe.py` surface
unchanged by workers. 6. Each module green via its own tests before done.
```
