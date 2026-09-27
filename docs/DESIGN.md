# PathwayPilot — Design

The visual design system for the PathwayPilot web UI: design direction, typography, colors, layout and components.

---

## 1. Design context

**Who's looking:** a scientist checking an answer, and a hiring manager judging craft in under a minute.

**What the page must do:** ask a question → read a grounded answer → **verify it** through citations, the tool trace and the graph.

**Direction: "lab notebook meets instrument panel."**
- Calm, precise and readable, like a well-kept lab notebook
- Data shown the way instruments show it: monospace IDs, exact timings, clear states
- Color **carries meaning** (node types, status) and is not used for decoration
- No gradients, glows, emoji or stock AI look (no purple-to-blue hero, no sparkle icons)

**Principles**
1. **Evidence beside every claim.** A citation is never more than one click from its node in the graph.
2. **Show the work.** The tool trace is first-class content, not something hidden in a debug drawer.
3. **Meaning is never color alone.** Node types differ in color **and** shape **and** label.
4. **Quiet by default.** One accent color; everything else is neutral or semantic.

---

## 2. Typography

| Role | Font | Fallback stack | Used for |
|---|---|---|---|
| UI & body | **IBM Plex Sans** (400, 500, 600) | `system-ui, -apple-system, "Segoe UI", sans-serif` | Headings, answer text, labels, buttons |
| Data | **IBM Plex Mono** (400, 500) | `ui-monospace, Menlo, Consolas, monospace` | Node IDs, tool names, arguments, timings, metrics |

Load from Google Fonts:
```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:wght@400;500&family=IBM+Plex+Sans:wght@400;500;600&display=swap">
```
*Why Plex:* it was designed for technical and scientific interfaces. The sans and mono are cut to match, so node IDs sit naturally inside sentences.

### Type scale (rem at 16px root)
| Token | Size / line height | Weight | Use |
|---|---|---|---|
| `--fs-display` | 1.75rem / 1.2 (28px) | 600 | App title only |
| `--fs-h2` | 1.25rem / 1.3 (20px) | 600 | Panel titles |
| `--fs-body-lg` | 1.0625rem / 1.65 (17px) | 400 | **Answer text** (the most important text on the page) |
| `--fs-body` | 0.9375rem / 1.55 (15px) | 400 | Default text |
| `--fs-small` | 0.8125rem / 1.45 (13px) | 400/500 | Trace details, captions |
| `--fs-label` | 0.75rem / 1.3 (12px) | 500, uppercase, `letter-spacing: 0.06em` | Section labels, metric labels |
| `--fs-mono` | 0.8125rem / 1.45 (13px) | 400/500 | IDs, tool names, numbers |

Rules:
- Answer text is capped at **68ch** wide
- Headings use `text-wrap: balance`
- All numbers use `font-variant-numeric: tabular-nums`
- Tool names and node IDs are always in mono, even inside sentences

---

## 3. Color

### 3.1 Base tokens
| Token | Light | Dark | Use |
|---|---|---|---|
| `--bg` | `#F5F7F8` | `#0E1419` | Page background (cool neutral with a slight blue-green tint) |
| `--surface` | `#FFFFFF` | `#151E25` | Panels |
| `--surface-2` | `#EEF2F4` | `#1C2730` | Trace rows, input backgrounds, hover |
| `--ink` | `#14202B` | `#E4EAEF` | Primary text |
| `--muted` | `#5A6875` | `#93A1AD` | Secondary text, captions |
| `--line` | `#D8DFE5` | `#26333D` | Borders, dividers, graph edges |
| `--accent` | `#0F6E7A` | `#4FB3BF` | Primary button, links, focus-adjacent emphasis (deep teal) |
| `--accent-soft` | `#E0F0F1` | `#16343A` | Selected states, active example chip |
| `--focus` | `#2F7FD0` | `#6FB0F0` | Focus ring (2px outline, 2px offset) |

### 3.2 Node-type colors (semantic, the core of the palette)
| Node type | Light | Dark | Graph shape | Chip marker |
|---|---|---|---|---|
| Metabolite | `#2F6FB5` | `#6FA8E8` | Ellipse | ● circle |
| Enzyme | `#2E8B57` | `#5DBB86` | Round rectangle | ▬ rounded bar |
| Reaction | `#7A8793` | `#9AA6B1` | Small diamond | ◆ diamond |
| Pathway | `#B7791F` | `#E0A948` | Hexagon (outline only) | ⬡ hexagon |
| Inhibitor | `#C2413B` | `#EE7A72` | Octagon | ⯃ octagon |

Rules:
- These colors are used for **fills, borders and markers**, never for body text. Text on or beside them stays `--ink`, which keeps contrast AA in both themes.
- `INHIBITS` edges are the only red edges: dashed, with a flat "tee" arrowhead.
- Edges on the found path use `--accent` at 2.5px; other edges use `--line` at 1px.

### 3.3 Status colors
| Token | Light | Dark | Use |
|---|---|---|---|
| `--ok` | `#2E7D4F` | `#5DBB86` | Tool step succeeded, eval passed |
| `--warn` | `#A86A12` | `#E0A948` | Truncated answer, slow step (> 2 s) |
| `--error` | `#B3362F` | `#EE7A72` | Tool error, request failed, refusal notice |

### 3.4 Theme implementation
```css
:root { /* full light palette */ }
@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) { color-scheme: dark; /* dark tokens */ }
}
:root[data-theme="dark"] { color-scheme: dark; /* dark tokens */ }
body { background: var(--bg); color: var(--ink); }
```
Cytoscape styles must read these tokens at render time (`getComputedStyle`), and redraw when the theme changes.

---

## 4. Spacing, shape, elevation
| Token | Value |
|---|---|
| Spacing scale | `4, 8, 12, 16, 24, 32, 48` px (`--s-1` … `--s-7`) |
| `--radius-sm` | 6px (chips, inputs, trace rows) |
| `--radius-md` | 10px (panels) |
| Borders | 1px `--line` |
| Shadow | One only, for the sticky ask bar when scrolled: `0 1px 0 var(--line), 0 6px 16px rgb(0 0 0 / 0.06)` |

Use `gap` in flex and grid layouts rather than margins between siblings.

---

## 5. Layout

### Desktop (≥ 1024px)
```
┌──────────────────────────────────────────────────────────────┐
│ PathwayPilot                     3 pathways · 74 nodes  ◐    │  header
├──────────────────────────────────────────────────────────────┤
│ [ Ask about glycolysis, TCA or shikimate…            ] [Ask] │  ask bar (sticky)
│ Try: (glucose → pyruvate) (what does glyphosate block?) …    │  example chips
├───────────────────────────────┬──────────────────────────────┤
│ ANSWER                        │ GRAPH                        │
│ Answer text with inline       │ Cytoscape view of the        │
│ citation chips…               │ subgraph (sticky, 60vh)      │
│ ─────────────────────────     │                              │
│ 3.1 s · 4 steps · 2,042 tok   │ Legend: ● ▬ ◆ ⬡ ⯃            │
│                               │                              │
│ TOOL TRACE                    │                              │
│ 1 search_entities  42 ms  ✓   │                              │
│ 2 get_inhibitors   8 ms   ✓   │                              │
│ 3 get_downstream   11 ms  ✓   │                              │
└───────────────────────────────┴──────────────────────────────┘
```
- Content max width 1200px; side gutter 24px (16px on mobile)
- Grid: `minmax(0, 1.1fr) minmax(0, 0.9fr)`, gap 24px

### Tablet and mobile (< 1024px)
Stack: ask bar → answer → graph (height 360px) → trace. The ask bar stays sticky at the top (`top: env(safe-area-inset-top, 0px)`).

---

## 6. Components

### 6.1 Header
- Left: "PathwayPilot" in `--fs-display`
- Right: dataset badge in mono (`3 pathways · 74 nodes`, taken from `/api/health`), plus the provider label (`ollama` / `openai` / `fake`) as a small neutral chip
- Theme toggle button (sun/moon icon + `aria-label`)

### 6.2 Ask bar
- Text input: full width, 48px tall, `--surface-2` background, `--radius-sm`, placeholder *"Ask about glycolysis, the TCA cycle or the shikimate pathway"*
- **Ask** button: `--accent` background, white text, 600 weight. While running it's disabled and reads *"Thinking…"*.
- Enter submits; Shift+Enter is ignored (single line)
- Character limit 500 with a counter that appears after 400

### 6.3 Example chips
- 4 chips, one per category (lookup, path, cross-pathway, impact)
- Neutral outline style; hover → `--surface-2`; clicking fills the input and submits

### 6.4 Answer panel
- Label "ANSWER" (`--fs-label`, `--muted`)
- Answer text in `--fs-body-lg`
- **Citation chip** (inline): replaces `[node_id]` in the text.
  - Shape marker in the node-type color + node name in `--ink` + ID in mono `--muted` on hover or focus (tooltip)
  - Background `--surface-2`, radius 6px, padding `2px 8px`
  - Click or Enter → highlight that node in the graph (pulse ring for 600ms, then a steady 3px `--accent` ring) and scroll the graph into view on mobile
- **Refusal state:** when the agent says the graph can't answer, show a left-bordered note in `--error` with the text *"Not in the PathwayPilot graph. It covers glycolysis, the TCA cycle and the shikimate pathway."*
- **Truncated state:** an `--warn` note: *"Stopped after 6 tool steps. The answer may be incomplete."*

### 6.5 Metrics strip
- One row in mono `--fs-small`: `3.1 s · 4 steps · 2,042 tokens`
- Tabular numbers; latency turns `--warn` above 8 s

### 6.6 Tool trace timeline
- Label "TOOL TRACE"
- One row per step, as a grid: `step #` | `tool name (mono, 500)` | `short args` | `duration (right-aligned)` | status icon
- Status: ✓ in `--ok`, ✕ in `--error` (with the error message shown under the row)
- Rows expand on click (`<details>`) to show full args and the result as formatted JSON (mono, `--surface-2`, max height 240px, scrollable)
- A 2px vertical line in `--line` connects the step numbers, like a timeline

### 6.7 Graph view (Cytoscape.js)
- Panel label "GRAPH" plus a legend row with all 5 node types (marker + name)
- Layout: `breadthfirst` (directed, top to bottom) for path answers; `cose` for everything else. Animate layout 300ms (0ms with `prefers-reduced-motion`).
- Node style: fill = type color at 18% opacity, 2px border in the type color, label in `--ink` 12px Plex Sans below the node
- Cited nodes: full-opacity fill, and a label weight of 600
- Controls: fit-to-screen button, zoom via scroll or pinch; clicking a node shows its name, type, ID and description in a small popover
- Empty state: *"The graph for your answer appears here."*

### 6.8 States
| State | Behavior |
|---|---|
| **Empty (first load)** | Short intro: *"Ask a question. PathwayPilot answers from a curated graph of 3 pathways and shows every step it took."* plus the example chips |
| **Loading** | Ask button disabled; answer area shows 3 skeleton lines; trace shows *"Calling tools…"* in mono `--muted`. No spinners over the page. |
| **Error** | `--error` note: what failed + what to do, e.g. *"The model provider didn't respond (ollama, timeout after 30 s). Check that Ollama is running, or try again with LLM_PROVIDER=fake."* |
| **Success** | Answer, metrics, trace and graph all appear together |

### 6.9 Buttons
| Type | Style |
|---|---|
| Primary | `--accent` fill, white text, radius 6px, 40–48px tall |
| Secondary | 1px `--line` border, `--ink` text, transparent background |
| Icon | 36px square, `aria-label` required |
All buttons have a visible focus ring (`--focus`, 2px outline, 2px offset) and 44px minimum touch targets on mobile.

---

## 7. Motion
- Only three animations: skeleton shimmer (1.2s), citation highlight pulse (600ms), graph layout (300ms)
- All are disabled under `@media (prefers-reduced-motion: reduce)`

---

## 8. Accessibility checklist
- [ ] Text contrast ≥ 4.5:1 in both themes (node colors are never used for text)
- [ ] Node types distinguishable in grayscale (shape + label)
- [ ] Answer region has `aria-live="polite"`
- [ ] Every control reachable and usable by keyboard; citation chips are `<button>`s
- [ ] Graph has a text equivalent: the trace plus the citation list
- [ ] Layout works at 400px wide with no horizontal page scroll

---

## 9. README visuals (the portfolio side)
- **Demo GIF** (15 to 20 s): ask the glyphosate question → the answer appears → click a citation → the node highlights in the graph → open one trace step
- **Architecture diagram:** the Mermaid chart from ARCHITECTURE §1 (renders on GitHub)
- **Eval table:** baseline vs best run, with the metric names exactly as defined in PRD §9
- Screenshot in both light and dark themes
