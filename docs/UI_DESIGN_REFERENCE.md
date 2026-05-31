# APEX UI Design Reference

> Captured from apex_demo.html — the approved design for implementation.

## 2026-05 update

The shell is now a **3-tab layout** (Main / Chats / Loot) rather than a single static panel grid. The Main tab holds the floating-panel workspace documented below; Chats and Loot are sibling tabs.

**Status-bar controls** (top, §13/§20): a **mode toggle** that switches between approve-gated ⇄ autonomous (hard rails stay ON in both); an **approval queue** badge that opens the pending-action popover; and a **KILL** switch. The **Orchestra** panel gains **per-role model-swap** dropdowns (§19) so each role's model can be hot-swapped live. All of this is driven by the **Supervisor event stream** over the WebSocket bridge (ref master-plan §13/§17/§20).

## Layout

7 floating panels in a grid, fully resizable with cooperative edge-dragging:

```
┌──────────────────────┬─────────────────┬──────────────┐
│                      │                 │              │
│    Conversation      │   Attack Tree   │  Loot Room   │
│       (46%)          │     (29%)       │    (25%)     │
│                      │                 │              │
├──────────────────────┤                 │              │
│  Input Bar (46%×6%)  │                 │              │
├──────────────────────┼─────────────────┼──────────────┤
│                      │                 │              │
│    Tool Log          │  Network Map    │  Orchestra   │
│       (46%)          │     (29%)       │    (25%)     │
│                      │                 │              │
└──────────────────────┴─────────────────┴──────────────┘
```

- Status bar: 36px fixed top
- Quick actions: 28px fixed bottom
- Main grid: fills remainder
- All panels resizable — dragging an edge pushes neighboring panels cooperatively

## Default Panel Positions (% of main-grid)

| Panel | Left | Top | Width | Height |
|---|---|---|---|---|
| Conversation | 0% | 0% | 46% | 50% |
| Input Bar | 0% | 50% | 46% | 6% |
| Attack Tree | 46% | 0% | 29% | 56% |
| Loot Room | 75% | 0% | 25% | 56% |
| Tool Log | 0% | 56% | 46% | 44% |
| Network Map | 46% | 56% | 29% | 44% |
| Orchestra | 75% | 56% | 25% | 44% |

## Color Palette (Design Tokens)

### Backgrounds
| Token | Value | Usage |
|---|---|---|
| `--bg-void` | `#000000` | Body, deepest background |
| `--bg-surface` | `#080808` | Status bar, panel headers, section headers |
| `--bg-elevated` | `#0e0e0e` | Panel bodies |
| `--bg-input` | `#0a0a0a` | Input fields, code blocks |

### Text
| Token | Value | Usage |
|---|---|---|
| `--text-primary` | `#c8c0b8` | Default body text |
| `--text-secondary` | `#7a706a` | Dimmed labels, metadata |
| `--text-bright` | `#f0e8e0` | Status values, emphasis |
| `--text-dim` | `#3e3632` | Lowest contrast, borders |

### Accent Colors
| Token | Value | Usage |
|---|---|---|
| `--color-primary` | `#e02020` | Red — brand, critical, exploit |
| `--color-secondary` | `#e86820` | Orange — terminal glow, conductor, warnings |

### Model Role Colors — §17 v1 orchestra (current)
| Role | Letter | Color | Hex |
|---|---|---|---|
| conductor | C | Orange | `#e86820` |
| strategist | S | Purple | `#9070c8` |
| operator | O | Green | `#44b85e` |
| analyst | A | Blue | `#5090c4` |
| reviewer | R | Gold | `#c8b84a` |
| ranker | K | Magenta | `#b860aa` |
| opsec | P | Red-Orange | `#e04420` |
| meta | M | Teal | `#50b0a0` |

> **SUPERSEDED** (pre-§17 / v2 Code Cell roles — kept for traceability, no longer in the roster):
> `reasoner` `#b860aa`, `executor` `#44b85e`, `exploit_spec` `#e02020`, `validator` `#c8b84a`, `counter_intel` `#e02020`, `recon` `#5090c4`, `swarm` `#e8c820`. Tokens `--color-exploit`/`--color-evasion` survive only as legacy aliases.

### DEFCON Levels
| Level | Color | Hex |
|---|---|---|
| DEFCON 5 | Green | `#44b85e` |
| DEFCON 4 | Blue | `#5090c4` |
| DEFCON 3 | Orange | `#e86820` |
| DEFCON 2 | Red-Orange | `#e04420` |
| DEFCON 1 | Red | `#e02020` |

### Severity Badges
| Severity | BG | Text | Border |
|---|---|---|---|
| Critical | `rgba(224,32,32,.15)` | `#e02020` | `rgba(224,32,32,.25)` |
| High | `rgba(232,104,32,.12)` | `#e86820` | `rgba(232,104,32,.2)` |
| Medium | `rgba(232,200,32,.1)` | `#e8c820` | `rgba(232,200,32,.18)` |
| Low | `rgba(80,144,196,.1)` | `#5090c4` | `rgba(80,144,196,.18)` |

## Typography

- **Font stack:** `'JetBrains Mono', 'Cascadia Code', 'Fira Code', 'Consolas', monospace`
- No external font imports — system monospace only

| Token | Size | Usage |
|---|---|---|
| `--text-xs` | `0.78rem` (~12px) | Badges, timestamps, tool args |
| `--text-sm` | `0.85rem` (~14px) | Messages, labels, tool names, model dots |
| `--text-base` | `0.95rem` (~15px) | Default body |
| `--text-md` | `0.925rem` (~15px) | Prompt character |
| `--text-lg` | `1.05rem` (~17px) | Panel titles, placeholder titles |

## Spacing

| Token | Value |
|---|---|
| `--space-1` | `4px` |
| `--space-2` | `8px` |
| `--space-3` | `12px` |
| `--space-4` | `16px` |
| `--space-5` | `20px` |
| `--space-6` | `24px` |
| `--space-8` | `32px` |

## Borders

| Token | Value |
|---|---|
| `--border-subtle` | `rgba(255,255,255,.04)` |
| `--border-default` | `rgba(255,255,255,.06)` |
| `--radius-sm` | `3px` |
| `--radius-md` | `6px` |

## Effects

- **Scanline overlay:** `repeating-linear-gradient(0deg, transparent 1px, rgba(0,0,0,.08) 2px)` at 40% opacity, full viewport, pointer-events:none, z-index:9999
- **Vignette:** `radial-gradient(ellipse at center, transparent 50%, rgba(0,0,0,.55) 100%)`, z-index:9998
- **Panel shadow:** `0 2px 16px rgba(0,0,0,.6), inset 0 1px 0 rgba(255,255,255,.02)`
- **Glow filter (SVG):** `feGaussianBlur stdDeviation=2` for network map nodes
- **Pulse animation:** `opacity 1 → 0.4 → 1` over 2s, used for active dots and swarm badge

## Panel Components

### Status Bar (36px)
- Left: APEX logo (red, bold, letter-spacing:2px) | Engagement ID | Target name (orange)
- Right: DEFCON badge | OPSEC bar (80px, 4px height) | Model dots (6px, green glow) | Cost (orange) | Elapsed time

### Panel Header (28px)
- Background: `--bg-surface`
- Font: `--text-xs`, uppercase, letter-spacing:1.5px, color:`--text-secondary`
- Right-aligned count/live indicator

### Conversation Messages
- Meta row: role (colored), model name (dim), timestamp (dim, right-aligned)
- Body: `--text-sm`, supports `<code>`, `.crit`, `.high`, `.good`, `.info` classes
- Output blocks: `--bg-void`, 2px left border (colored by type: vuln=red, loot=green, recon=blue)

### Orchestra Rows
- Grid: 24px letter circle | info column | metrics column
- Letter circle: colored background, black text, bold
- Info: role name (colored) + model name (dim), task description below
- Metrics: token count + load bar (48px wide, 3px height, colored by load)
- Blue Team Intel section below with label:value rows

### Attack Tree
- Recursive nodes with 8px status dots (complete=green, active=orange+pulse, pending=dim)
- Children indented 20px with left border
- Info text right-aligned, dim

### Loot Room
- Two sections: Findings (with severity badges) and Credentials
- Entries: badge | title | host/meta (right-aligned)
- Credential entries: green username | service description

### Tool Log
- Grid: tool name (orange, bold) | args (dim, ellipsis) | status badge | duration
- Status: success (green bg) | running (orange bg + pulse) | queued (dim)

### Network Map
- SVG with zoom (mouse wheel), pan (drag background), draggable nodes
- Nodes: 11px radius circles, OS-colored, letter overlay, hostname label below, port count badge
- Edges: dashed lines between same-subnet nodes, faint lines to gateway
- Attacker node: red ring with glow
- Glow filter on all nodes

### Input Bar
- Orange prompt character `❯` with text-shadow glow
- Transparent input, monospace, placeholder "talk to APEX..."

### Quick Actions (28px)
- Left: keyboard shortcuts (S/T/N/F/L//) with orange key highlights
- Right: finding counts (red critical, orange high), host count, credential count

## Network Map Interactions
- **Zoom:** Mouse wheel scales the viewport (0.3x to 5x)
- **Pan:** Click and drag on empty space moves the viewport
- **Drag nodes:** Click and drag individual nodes repositions them, edges follow
- **Tooltips:** Hover shows hostname, OS, open ports with services

## Panel Resize Behavior
- All panel edges are draggable (6px hit zone)
- Dragging a shared edge resizes both panels cooperatively
- Minimum panel size: 120px width, 48px height
- Cursor changes on edge hover (col-resize / row-resize)
- Panel positions persist to localStorage

## Login Screen
- Full-viewport canvas with 2500 fibonacci-sphere particles
- Particles: red→orange→white gradient based on depth, 0.4-1.6px size
- Mouse proximity (120px radius) scatters particles
- Click reveals auth form (fade in)
- Authenticate → canvas fades out (1.2s) → dashboard loads
- Subtitle: "operator authentication" (uppercase, letter-spaced, dim)
- Form: dark inputs with subtle borders, red-tinted focus state
- Button: transparent with red border, hover fills red bg

## Swarm Mode Indicator
- Positioned top-right of Orchestra panel
- Pulsing yellow dot + "SWARM MODE — N agents" text
- Color: `--color-swarm` (#e8c820)
