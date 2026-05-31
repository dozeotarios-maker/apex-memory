# APEX — Model Flexibility (choose / connect any model)

APEX is **model-agnostic by design** (master-plan §19). DeepSeek V4 Pro/Flash are
the *defaults*, not a lock-in. Because APEX drives **opencode** as its engine, it
can use **any provider opencode supports** (75+: Anthropic, OpenAI, Google,
OpenRouter, local Ollama/LM Studio, the OpenCode subscription, …) — per role,
hot-swappable mid-engagement.

There are two layers. APEX *selects* a model; opencode *connects* to it.

```
config/apex.toml [models]   ──selects──▶  ModelRegistry ("provider/model-id")
        │  (or live UI swap)                     │  passed verbatim to the engine
        ▼                                        ▼
   per-role model                        opencode serve  ──connects──▶  provider
                                          (deploy/opencode/opencode.json + auth)
```

---

## 1. Selecting a model (APEX side — already wired)

Per-role assignment in `config/apex.toml`, opencode `provider/model-id` notation:

```toml
[models.conductor]
model = "deepseek/deepseek-v4-pro"     # ← change to any configured provider/model
thinking = "adaptive"                   # on | off | adaptive | contested
```

- **Live swap:** the UI **Orchestra** panel shows each role's model with a swap
  dropdown → `POST /api/role-model` → `ModelRegistry.set_role_model(role, model)`.
  Takes effect on the role's next turn; no restart.
- The registry accepts **any** `provider/model-id` string — it does not validate
  against a fixed list, so a model you just configured in opencode is selectable
  immediately (type it into the swap control even if it's not in the suggestion
  list). The suggestion list lives in `apex/ui/static/js/app.js` (`MODEL_OPTIONS`).

---

## 2. Connecting a provider (opencode side)

The running `opencode serve` reads its config from `OPENCODE_CONFIG_DIR`, which the
systemd unit points at `deploy/opencode/opencode.json` (foreground runs use
`~/.config/opencode/` unless you set the same env var). Add a `provider` block;
put secrets in `~/.config/apex/apex.env` and reference them with `{env:VAR}`.

The DeepSeek block (already present) is the template:

```jsonc
"deepseek": {
  "name": "DeepSeek", "api": "openai",
  "baseURL": "https://api.deepseek.com/v1",
  "apiKey": "{env:DEEPSEEK_API_KEY}",
  "models": [ { "id": "deepseek-v4-pro", "contextWindow": 1048576,
    "supports": { "tool_calling": true, "structured_output": true, "thinking": true } } ]
}
```

Add more providers alongside it (set the matching key in `apex.env`):

```jsonc
"anthropic": {
  "name": "Anthropic", "api": "anthropic",
  "apiKey": "{env:ANTHROPIC_API_KEY}",
  "models": [ { "id": "claude-opus-4", "supports": { "tool_calling": true } } ]
},
"openai": {
  "name": "OpenAI", "api": "openai", "baseURL": "https://api.openai.com/v1",
  "apiKey": "{env:OPENAI_API_KEY}",
  "models": [ { "id": "gpt-5", "supports": { "tool_calling": true } } ]
},
"openrouter": {
  "name": "OpenRouter", "api": "openai", "baseURL": "https://openrouter.ai/api/v1",
  "apiKey": "{env:OPENROUTER_API_KEY}",
  "models": [ { "id": "anthropic/claude-3.7-sonnet", "supports": { "tool_calling": true } } ]
},
"ollama": {                                    // local, in-VM — max confidentiality, no egress
  "name": "Ollama (local)", "api": "openai", "baseURL": "http://127.0.0.1:11434/v1",
  "apiKey": "{env:OLLAMA_API_KEY}",            // any non-empty value; Ollama ignores it
  "models": [ { "id": "llama3.1:70b", "supports": { "tool_calling": true } } ]
}
```

Then point a role at it, e.g. `model = "anthropic/claude-opus-4"`.

> Note: `deploy/opencode/opencode.json` is strict JSON (no comments). The blocks
> above show `//` only for explanation — strip comments when pasting, or keep your
> live config in `~/.config/opencode/opencode.jsonc` (JSONC allows comments).

---

## 3. Connecting your OpenCode subscription (Zen)

The OpenCode subscription (Zen) is just a **key-based, OpenAI-compatible provider**
at `https://opencode.ai/zen/v1` — connect it with your OpenCode Zen **API key**,
exactly like any other provider. No interactive `opencode auth login` needed.

**Easiest — the UI:** Settings → OpenCode Subscription → paste your API key →
Connect. APEX stores it as `OPENCODE_API_KEY` (chmod 600) and writes the
`opencode` provider block for you.

**Equivalent by hand:** set `OPENCODE_API_KEY` in `apex.env` and add the provider:

```jsonc
"opencode": {
  "name": "OpenCode Zen", "api": "openai",
  "baseURL": "https://opencode.ai/zen/v1",
  "apiKey": "{env:OPENCODE_API_KEY}",
  "models": [ { "id": "grok-code", "supports": { "tool_calling": true } } ]
}
```

Either way the subscription's models are addressable as `opencode/<model-id>`
(e.g. `opencode/grok-code`, `opencode/gpt-5.5`, `opencode/claude-*`). Select one
per role like any other model:

```toml
[models.conductor]
model = "opencode/claude-sonnet-4-6"
```

or via the UI swap control. No `apex.env` key needed — auth is held by opencode.

---

## 4. Capability awareness (degradation)

`ModelSpec` carries capability flags (`caps`) and the opencode `supports{}` block
declares per-model features (`tool_calling`, `structured_output`, `thinking`).
Hard-requirement roles depend on these:

- **Operator** REQUIRES `tool_calling` + bash — don't assign a model without it.
- **thinking = on/adaptive** assumes a reasoning-capable model; on a model without
  it, opencode falls back to plain completion (the Fabric still runs, just no
  CoT budget). Full capability-matched fallback/refusal is a v1.1+ hardening (§19).

Pricing/limits differ per provider — keep the `[budget]` cost cap in `apex.toml`
in mind when swapping to a pricier model for every role.

---

## 5. Quick recipes

| Goal | Do |
|---|---|
| Test on DeepSeek (default) | nothing — it's the shipped config |
| Add Anthropic/OpenAI/etc. | add a `provider` block (§2) + key in `apex.env` → set role model |
| Use your OpenCode subscription | Settings → OpenCode Subscription → paste API key (§3) → set role model to `opencode/<id>` |
| Local-only / air-gapped brain | add the `ollama` provider (§2), run Ollama in the VM |
| Swap one role live | UI Orchestra panel → that role's dropdown |
| Swap defaults permanently | edit `config/apex.toml [models]` |

See also: `docs/APEX-MASTER-PLAN.md` §19 (model-flexibility decision),
`docs/DEPLOY.md` (env + services).
