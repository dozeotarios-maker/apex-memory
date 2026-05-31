# APEX v1.1 — FROZEN KNOWLEDGE-SYSTEM CONTRACTS (2026-05-30)

> Single source of truth for the v1.1 **knowledge system** (`apex/knowledge/`).
> Realizes master-plan **§8** (knowledge / "where to look") + **§9** (web-research
> never-block ladder). FROZEN before the worker fan-out — every parallel worker
> imports `apex/knowledge/types.py` (already written) and codes ONLY against the
> signatures below. Do **not** change a frozen signature/field without a re-broadcast.
> Do **not** re-open settled v1 design (§1–§24). This extends v1; it does not touch
> the frozen v1 contracts in `docs/V1_BUILD_CONTRACTS.md`.

---

## 0. GROUND RULES (apply everywhere)
- Python **3.12**, `from __future__ import annotations`, full type hints, `ruff` + `mypy` clean.
- `pathlib.Path` only. LF endings. No machine-bound anything.
- **Offline-first / Windows-green:** the whole package + its BLOCKING tests MUST pass on the
  Windows dev box with **no network, no LLM, no Linux tools, and the bge embedding model ABSENT**
  (it is not on disk here — `Embeddings.embed()` returns zero-vectors). The deterministic
  **SQLite-FTS5 keyword + trust** floor is the PRIMARY retrieval path; embeddings are an
  OPTIONAL rerank booster that activates only when the model is present. **No Qdrant** (§23 defers
  it to v2; the `KnowledgeRAG` interface is the documented swap-in point).
- **Parse-before-feed (§10):** what the LLM sees is COMPACT (summary + a couple of commands), never
  the full 1k-word corpus blob. The blob is indexed, not surfaced.
- Every module ships `tests/knowledge/test_<module>.py`. A module is "done" only when green.
- Inject `clock`/`id_gen` where time/ids matter (testability); default to real ones.

## 1. PACKAGE LAYOUT & OWNERSHIP (disjoint files — no two workers touch one file)
```
apex/knowledge/types.py        # DONE (frozen, written) — shared dataclasses/enums
apex/knowledge/__init__.py     # DONE (minimal) — Phase-3 facade added by lead, do NOT edit
apex/knowledge/trust.py        # WORKER A
apex/knowledge/corpus.py       # WORKER B   (imports trust)
apex/knowledge/rag.py          # WORKER C
apex/knowledge/fingerprint.py  # WORKER D
apex/knowledge/search.py       # WORKER E
apex/knowledge/playbooks.py    # WORKER F
tests/knowledge/__init__.py    # (any worker may create; keep it empty)
tests/knowledge/test_trust.py        # A
tests/knowledge/test_corpus.py       # B
tests/knowledge/test_rag.py          # C
tests/knowledge/test_fingerprint.py  # D
tests/knowledge/test_search.py       # E
tests/knowledge/test_playbooks.py    # F
config/apex.toml               # DONE — [knowledge] + [search] sections added
```
Workers code against FROZEN interfaces in this doc + `types.py`, NEVER against another worker's
function body. Cross-module calls use only the signatures below.

## 2. FROZEN TYPES (in `apex/knowledge/types.py` — read it; do not change)
`TrustTier(RESEARCHED|VETTED|SEED)` · `KnowledgeItem` · `Retrieval` · `SituationTags` ·
`PlaybookStep` · `Playbook` · `KnowledgeContext(.to_prompt_block())` · `SearchResult` · `FetchResult`.
Field lists are authoritative in the source file.

---

## 3. WORKER A — `apex/knowledge/trust.py`
Pure logic. Provenance/confidence/freshness tiers + ranking weights (§8).
```python
TRUST_WEIGHTS: dict[TrustTier, float]   # {RESEARCHED:1.0, VETTED:0.8, SEED:0.6} (the config defaults)

def infer_trust(rec: dict, *, source_file: str = "") -> tuple[TrustTier, float]:
    """From a RAW seed-corpus record dict → (tier, confidence 0..1).
    v1.1 rule: the shipped corpus is SEED. Confidence derives from rec['quality_score']
    (0..1, or 0..100 normalized) if present, else 0.5. A record carrying an explicit
    'provenance'/'trust' == 'researched'/'vetted' (future re-research output) maps accordingly."""

def trust_weight(tier: TrustTier, confidence: float, *, weights: dict[TrustTier, float] | None = None) -> float:
    """Ranking multiplier: weights[tier] * (0.5 + 0.5*confidence), clamped to [0,1].
    Pass weights from config; default to TRUST_WEIGHTS."""
```
**Tests:** seed default; quality_score→confidence (both 0..1 and 0..100 scales); weight monotonic in
tier and confidence; unknown/empty record → (SEED, 0.5).

---

## 4. WORKER B — `apex/knowledge/corpus.py`
Load + normalize the seed JSON into `KnowledgeItem[]`. Imports `trust.infer_trust`.
```python
def normalize_record(rec: dict, *, source_file: str = "") -> KnowledgeItem | None:
    """One raw record → KnowledgeItem, or None if it lacks id/technique_name.
    - summary  = rec['description'] truncated to ~280 chars at a word boundary.
    - text     = join(description, real_world_cases, operational_tips, chain_examples) — the FTS blob.
    - tools    = [tools.primary.name] + [a.name for a in tools.alternatives] (dedup, drop falsy).
    - commands = [c.get('command') for c in commands_reference if it has one] (cap 8).
    - opsec    = flatten opsec_notes (artifacts_created + cleanup_commands + network_artifacts), cap 8.
    - mitre_ids/references pulled if present (lists of str).
    - trust/confidence from trust.infer_trust(rec, source_file=source_file).
    Be DEFENSIVE: every nested field is optional and shapes vary across the 142 files
    (some records are flat, some methodology files add mitre_ids/quality_score/etc.).
    Never raise on a malformed record — return None and let the loader skip it."""

def load_corpus(path: Path) -> list[KnowledgeItem]:
    """Read every *.json under `path` (each file is a LIST of records, OR a dict with a
    list under 'techniques'/'entries' — handle both). Skip non-.json (e.g. SEED_PROGRESS.md)
    and unparseable files (log + continue). Dedup by id (first wins). Returns the flat list.
    Uses utf-8. Path is a directory."""
```
**Tests:** real `apex/data/seed/` loads ≥ ~600 items, none raise; a hand-built record with the full
nested schema normalizes correctly (summary truncated, tools/commands/opsec flattened); malformed
record → skipped; a dict-wrapped file shape handled; missing id/technique_name → None.
(Use `importlib.resources`/repo-relative path to find the seed dir; a `seed_dir()` helper is fine.)

---

## 5. WORKER C — `apex/knowledge/rag.py`
Hybrid retrieval: SQLite-FTS5 keyword floor + trust weighting + optional embedding rerank.
```python
class KnowledgeRAG:
    def __init__(self, items: list[KnowledgeItem], *, db_path: str | Path = ":memory:",
                 embeddings: "Embeddings | None" = None,
                 weights: dict | None = None) -> None:
        """weights keys: 'keyword','trust','semantic','trust_weights'(tier→float).
        Defaults match config/apex.toml [knowledge]. Does NOT build the index in __init__."""
    def build_index(self) -> None:
        """Idempotent. Create an FTS5 virtual table over (technique_name, summary, text, tools,
        domain, subdomain) + a side table of item metadata. If `embeddings` present AND the model
        loads (embed() returns a non-zero vector), precompute + store item vectors for rerank;
        otherwise skip the vector path silently."""
    def search(self, query: str, *, domains: list[str] | None = None, k: int = 5) -> list[Retrieval]:
        """build_index() if not built. FTS5 MATCH (sanitize the query → safe MATCH terms; a query with
        no alnum tokens returns []). Optional `domains` filters to those corpus domains (post-filter ok).
        Blend: score = w_keyword*norm_bm25 + w_trust*trust_weight(item) [+ w_semantic*cosine(query,item)
        when vectors exist]. Return top-k Retrieval sorted desc, `why` = dominant signal.
        MUST work with embeddings=None / model-absent (keyword+trust only)."""
    def close(self) -> None: ...
```
Reuse `from apex.memory.embeddings import Embeddings`. Use stdlib `sqlite3` (FTS5 is available — the v1
schema already uses it). bm25 via `bm25(fts_table)` ordering; normalize to 0..1 across the candidate set.
Cosine in numpy (already a transitive dep) — guard import; if numpy/vectors absent, semantic term = 0.
**Tests (model-absent path is BLOCKING):** build over a small KnowledgeItem list; a query matching a
technique returns it ranked #1; `domains` filter respected; higher-trust item outranks a lower-trust
item on an equal keyword match; empty/garbage query → []; `:memory:` and a tmp-file db both work;
search before explicit build_index() still works (auto-build). Do NOT require the embedding model.

---

## 6. WORKER D — `apex/knowledge/fingerprint.py`
Parse whatweb/wafw00f/httpx OUTPUT (canned strings) → `SituationTags`. Pure; no tool execution.
```python
def fingerprint(*, whatweb: str | None = None, wafw00f: str | None = None,
                httpx: str | None = None, observations: dict | None = None) -> SituationTags:
    """Detect cdn/waf/stack/cloud/os from tool output text + an optional pre-parsed
    `observations` dict (e.g. {'tech':[...], 'waf':'cloudflare', 'os':'linux'}).
    - whatweb: comma/bracket tech list (e.g. 'nginx, PHP, WordPress, Cloudflare').
    - wafw00f: 'is behind <WAF>' / 'No WAF detected'.
    - httpx: JSON or text with tech/webserver/cdn fields.
    Lowercase-normalize; map known names (cloudflare/akamai/fastly→cdn; awswaf/imperva/cloudflare→waf;
    aws/azure/gcp→cloud). Populate `domains` via tags_to_domains. `raw` keeps the parsed signals."""

def tags_to_domains(tags: SituationTags) -> list[str]:
    """Map fingerprint → knowledge-corpus domain filters. Examples: any web stack/waf/cdn → 'web';
    cloud=='aws' → 'cloud_aws'/'cloud'; os=='windows' → 'active_directory','windows'; etc.
    Always include a sensible default (['web']) if nothing else matched but there is web evidence.
    Return a DEDUPED list; [] if truly nothing detected."""
```
Domain strings should match the corpus `domain` values seen in `apex/data/seed/` (e.g. `web`,
`active_directory`, `cloud_aws`, `cloud`...). Loose/substring matching at the RAG filter is fine —
keep this best-effort, never raising.
**Tests:** Cloudflare-behind whatweb+wafw00f → cdn/waf='cloudflare', stack has nginx/php/wordpress,
domains includes 'web'; AWS httpx → cloud='aws'; empty inputs → empty SituationTags (no raise);
observations dict merges with parsed text.

---

## 7. WORKER E — `apex/knowledge/search.py`
The §9 never-block ladder behind ONE interface, with multi-provider failover + a SHA256/TTL/LRU cache.
Real providers are pluggable + key-gated (skipped offline). Cache + failover are pure-testable.
```python
class SearchProvider(Protocol):
    name: str
    def available(self) -> bool: ...                       # False if its API key is unset
    async def search(self, query: str, *, k: int) -> list[SearchResult]: ...

class ResearchCache:
    def __init__(self, *, ttl_seconds: int = 86400, max_entries: int = 1024,
                 clock: Callable[[], float] | None = None) -> None: ...
    def get(self, key: str) -> list[SearchResult] | None: ...   # None if absent/expired
    def put(self, key: str, results: list[SearchResult]) -> None: ...   # LRU evict at max_entries
    @staticmethod
    def key(query: str) -> str: ...                          # sha256 of normalized query

class WebResearch:
    def __init__(self, providers: list[SearchProvider], *, cache: ResearchCache | None = None,
                 order: list[str] | None = None) -> None: ...
    async def search(self, query: str, *, k: int = 5) -> list[SearchResult]:
        """Never-block ladder: cache hit → return; else try providers in `order`
        (skip unavailable / those that raise / those that return []), cache + return the first
        non-empty result set; if ALL fail → return [] (never raises — 'never dead-ended' is a
        higher layer's job; this returns empty so the caller falls back to corpus/creative)."""
    async def fetch(self, url: str) -> FetchResult: ...        # thin httpx GET; errors → FetchResult(status=0)

# Concrete providers (key-gated; available()==False when key missing → skipped offline):
class TavilyProvider(SearchProvider): ...     # TAVILY_API_KEY
class BraveProvider(SearchProvider): ...      # BRAVE_API_KEY
class SerperProvider(SearchProvider): ...     # SERPER_API_KEY
class SearxngProvider(SearchProvider): ...    # [search].searxng_url (free floor; available if url set)

def build_default_providers(config: dict | None = None) -> list[SearchProvider]:
    """Construct the provider list from env keys + config; all are constructible even with no keys
    (available() just returns False)."""
```
Use `httpx.AsyncClient`. **§9 note (document in a module docstring):** research traffic is host-side /
control-plane — NOT scope-gated; research hosts belong in `roe.control_plane_allow`. Do not wire the
ScopeGate here.
**Tests (no network — inject fake providers):** cache hit short-circuits providers; failover skips an
unavailable provider and one that raises, returns the next non-empty; all-fail → []; cache TTL expiry
(inject clock); LRU eviction at max_entries; `available()` False when key env unset (monkeypatch env).

---

## 8. WORKER F — `apex/knowledge/playbooks.py`
Situation → ordered techniques + pivot branches (§8). Imports `types` (+ may call a `KnowledgeRAG`).
```python
CURATED: dict[str, Playbook]
    """A small set of hand-authored playbooks keyed by situation label. MUST include at least
    'behind_cloudflare' realizing the §8 example: CT logs → passive DNS → Shodan favicon/cert →
    SPF/MX → non-proxied subdomains → direct-IP + Host header. Each step has a pivot_if_blocked."""

class Playbooks:
    def __init__(self, rag: "KnowledgeRAG | None" = None, *, curated: dict | None = None) -> None: ...
    def for_situation(self, tags: SituationTags, *, objective: str = "", k: int = 5) -> Playbook | None:
        """1) If a CURATED playbook matches the tags (e.g. waf/cdn=='cloudflare' → 'behind_cloudflare'),
        return it. 2) Else, if `rag` present, assemble a Playbook from the top techniques in
        tags.domains (rag.search(objective or domain-name, domains=tags.domains, k)) → one PlaybookStep
        per hit (technique=item.technique_name, tools=item.tools, knowledge_id=item.id). 3) Else None.
        Never raises."""
```
**Tests:** cloudflare tags → curated 'behind_cloudflare' with the documented ordered steps + pivots;
a tags-with-domains + a fake/real RAG → corpus-assembled playbook referencing knowledge_ids; no
match + no rag → None.

---

## 9. PHASE-3 FACADE (lead authors `apex/knowledge/__init__.py` — workers DON'T)
```python
class Knowledge:
    def __init__(self, *, corpus: list[KnowledgeItem], rag: KnowledgeRAG,
                 playbooks: Playbooks, web: WebResearch | None = None,
                 top_k: int = 3) -> None: ...
    @classmethod
    def from_config(cls, config: dict, *, seed_dir: Path, db_path=":memory:") -> "Knowledge": ...
    def retrieve(self, *, objective: str = "", analyst_finding: str = "",
                 already_tried: list | None = None, observations: dict | None = None,
                 whatweb: str | None = None, wafw00f: str | None = None,
                 httpx: str | None = None) -> KnowledgeContext:
        """SYNC, pure-local (no network/LLM): fingerprint(...) → tags → rag.search(query, domains)
        → playbooks.for_situation(tags) → KnowledgeContext. Query = analyst_finding or objective."""
    async def research(self, query: str) -> list[SearchResult]: ...   # delegates to web ladder
```
Workers only need to know `retrieve()` is SYNC and returns `KnowledgeContext`.

## 10. INTEGRATION SEAM (lead wires — zero-regression)
`apex/supervisor/core.py` ORIENT (≈line 425): the Supervisor gains an OPTIONAL `self.knowledge:
Knowledge | None` (default None). Before building `strategist_prompt`, if `self.knowledge` is set,
call `kctx = self.knowledge.retrieve(objective=obj_id or "", analyst_finding=analyst_msg.content,
already_tried=self._already_tried)` (wrapped in try/except — advisory, never breaks the loop) and
append `kctx.to_prompt_block()` to the prompt. The existing 572 tests construct the Supervisor with
`knowledge=None` → behaviour unchanged. UI boot constructs `Knowledge.from_config(...)`.

## 11. FROZEN RULES (every worker confirms)
1. Offline/Windows-green with the embedding model ABSENT — keyword+trust floor is primary; semantic is
   optional. No Qdrant. 2. Frozen `types.py` — never edited. 3. Disjoint file ownership. 4. Parse-before-
   feed: compact LLM-facing fields only. 5. Never raise on malformed corpus/tool-output/provider error —
   degrade + log. 6. Research traffic is control-plane (not scope-gated). 7. Every module green via its own
   `tests/knowledge/test_*.py` before "done".
```
