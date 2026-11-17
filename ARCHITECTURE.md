# System Architecture & Technical Specifications

This document outlines the architectural design, coordinate projection mathematics, telemetry data pipeline, assumptions, and engineering tradeoffs of the **LILA BLACK Match Telemetry Explorer**.

---

## 1. Technology Selection & Rationale

We selected a **Static Client-Side Architecture** powered by **Vanilla JavaScript (ES2022)**, **SVG 1.1 Vector Graphics**, **HTML5 Canvas 2D**, and an offline **Python (`PyArrow`, `Pillow`)** compiler:

- **Zero-Backend Footprint:** Production telemetry is compiled once into chunked, compressed JSON files. The viewer requires no Node/Python backend server or database at runtime, eliminating hosting costs, server maintenance, and cold-start latency.
- **SVG for Trajectories & Events:** Player routes and event markers are rendered as resolution-independent vector paths (`vector-effect="non-scaling-stroke"`). This ensures crisp lines, smooth panning/zooming, and per-element DOM interactivity (hover tooltips, click-to-seek) without canvas redraw overhead.
- **HTML5 Canvas for Multi-Layer Heatmaps:** Density overlays utilize an offscreen 2D canvas buffer with custom Gaussian kernel smoothing. Canvas handles thousands of position points across multiple aggregated matches at 60 FPS, supporting concurrent multi-layer composition (Traffic, Kills, Deaths).
- **Python / PyArrow for Data Engineering:** PyArrow processes columnar Snappy Parquet files orders of magnitude faster than pandas while consuming minimal RAM. Pillow performs aspect-ratio-preserving WebP minimap compression.

---

## 2. End-to-End Data Flow: Parquet to Screen

```
┌────────────────────────┐      ┌──────────────────────────┐      ┌─────────────────────────┐
│ Raw Parquet Telemetry  │ ───► │  scripts/build_data.py   │ ───► │   Static Output Files   │
│ (February_10-14 files) │      │  - Snappy decompression  │      │  - data/manifest.json   │
│ - ts, x, y, z, event   │      │  - Coordinate projection │      │  - data/*.json chunks   │
│ - user_id, match_id    │      │  - Global deduplication  │      │  - assets/minimaps/*.webp│
└────────────────────────┘      │  - SHA-256 pseudonymize  │      └─────────────────────────┘
                                └──────────────────────────┘                   │
                                                                               ▼
┌────────────────────────┐      ┌──────────────────────────┐      ┌─────────────────────────┐
│   Screen Renderers     │ ◄─── │  app.js (Client Engine)  │ ◄─── │   On-Demand HTTP Fetch  │
│ - SVG pathLayer (walk) │      │  - Playback cursor state │      │  - manifest.json (boot) │
│ - SVG eventLayer (mark)│      │  - Timeline scrubber     │      │  - Selected map/date    │
│ - Canvas 2D (heatmaps) │      │  - Event type filtering  │      │    match slice          │
│ - Telemetry log feeds  │      │  - AI Pacing heuristics  │      └─────────────────────────┘
└────────────────────────┘      └──────────────────────────┘
```

1. **Parquet Ingestion:** `build_data.py` decodes columnar records (`ts`, `x`, `y`, `z`, `event_type`, `user_id`, `match_id`) across all date folders.
2. **Global Deduplication:** Removes byte-identical repeated files and consolidates fragmented player journeys belonging to the same match. Matches appearing across date boundaries (e.g., spanning midnight UTC) are reconstructed once globally and indexed under both dates in `manifest.json`.
3. **Coordinate Transformation:** Real-world game engine units $(x, z)$ are converted into normalized $[0, 1]$ UV space per map configuration.
4. **Pseudonymization:** User IDs are converted to irreversible, deterministic `HUMAN-` or `BOT-` tokens via SHA-256 hashes.
5. **On-Demand Fetching:** The browser fetches `manifest.json` on initialization (~12 KB), then fetches only the active map/date dataset slice when selected, keeping initial load under 300 ms.
6. **Time-Synchronized Rendering:** The playback engine steps through records by relative match time $t = \text{ts} - \text{ts}_{\text{start}}$, dynamically rendering SVG paths up to cursor $t$, applying Gaussian heat blits, and updating discrete signal feeds.

---

## 3. Minimap Coordinate Mapping (World Space to Screen)

Mapping 3D Unreal/Unity game coordinates $(x, y, z)$ to a 2D minimap image requires resolving differing coordinate systems and vertical axis conventions.

### Coordinate Convention
- **Game Engine World Space:** $X$ is East/West, $Z$ is North/South, and $Y$ represents vertical elevation. North is $+Z$ (pointing upwards).
- **Screen / SVG / Image Space:** Origin $(0, 0)$ is at the **top-left corner**. $U$ runs left-to-right $[0, 1]$, while $V$ runs **top-to-bottom** $[0, 1]$. Downwards is $+V$.

### Mathematical Transformation
To align North with the top of the minimap image, the vertical axis must be inverted ($1.0 - \dots$):

$$u = \frac{x - \text{origin}_x}{\text{scale}}$$

$$v = 1.0 - \frac{z - \text{origin}_z}{\text{scale}}$$

To convert normalized $(u, v)$ to image pixel space $(P_x, P_y)$ for rendering:

$$P_x = u \times W_{\text{image}}, \quad P_y = v \times H_{\text{image}}$$

### Map Calibration Constants
| Map Name | World Origin $(X_0, Z_0)$ | World Scale | Source Minimap Resolution |
|---|---|---|---|
| **Ambrose Valley** | $(-370.0, -473.0)$ | $900.0$ | $4320 \times 4320$ px |
| **Grand Rift** | $(-290.0, -290.0)$ | $581.0$ | $2160 \times 2158$ px |
| **Lockdown** | $(-500.0, -500.0)$ | $1000.0$ | $9000 \times 9000$ px |

### Concrete Worked Example
Consider a player located at world coordinates $(x = -301.45, z = -355.55)$ on **Ambrose Valley**:
1. $u = \frac{-301.45 - (-370.0)}{900.0} = \frac{68.55}{900.0} \approx 0.076167$
2. $v = 1.0 - \frac{-355.55 - (-473.0)}{900.0} = 1.0 - \frac{117.45}{900.0} = 1.0 - 0.1305 = 0.869500$
3. On a $1024 \times 1024$ SVG viewport:
   - $P_x = 0.076167 \times 1024 \approx \mathbf{78.0\text{ px}}$
   - $P_y = 0.869500 \times 1024 \approx \mathbf{890.4\text{ px}}$

*Out-of-Bounds Handling:* If coordinates fall outside $[0, 1]$, points are intentionally retained rather than clamped, and the UI displays an out-of-bounds notification badge (`"N samples clipped"`) to preserve data integrity.

---

## 4. Ambiguities & Data Assumptions

1. **Timestamp Unit Anomaly:**
   - *Problem:* The Parquet schema defines `ts` as `timestamp[ms]`, but values (e.g. `1770754537`) represent Unix **seconds** (corresponding to February 10, 2026, 20:15:37 UTC). Interpreting them as milliseconds produces year 1970 dates.
   - *Resolution:* Treated raw values as seconds. All playback calculations use elapsed relative match time: $t_{\text{rel}} = \text{ts} - \text{ts}_{\text{first\_match\_event}}$.
2. **Actor vs. Target Attribution in Bot Events:**
   - *Problem:* The source specification stated `BotKill` = player kills bot, and `BotKilled` = bot kills player. However, telemetry reveals `BotKill` records appearing on bot journeys and `BotKilled` on human journeys.
   - *Resolution:* Avoided speculative inference. Displayed verbatim telemetry labels in the UI with clear tooltips and legend keys.
3. **Cross-Date Match Splitting:**
   - *Problem:* Match `c20bbffb...` spanned midnight UTC, causing journey files to appear in both `February_10` and `February_11` folders.
   - *Resolution:* Reconstructed globally by match UUID, then indexed under both dates in `manifest.json`.
4. **Reused Numeric Bot IDs:**
   - *Problem:* 81 of 94 numeric bot IDs appear across multiple matches.
   - *Resolution:* Categorized numeric IDs as bots, but avoided assuming persistent cross-match bot profiles.

---

## 5. Architectural Tradeoffs

| Considered Approach | Chosen Implementation | Tradeoff & Rationale |
|---|---|---|
| **Full-Stack Node/Python Server** | **Static Client-Side SPA** | Eliminates server hosting costs, cold starts, and backend failure points; shifts light JSON parsing to client. |
| **Heavy UI Framework (React/Vue)** | **Vanilla ES2022 + SVG DOM** | Avoids ~200 KB build bundles; enables direct, high-performance SVG DOM manipulation at 60 FPS. |
| **Canvas Raster Lines** | **SVG Path Traces** | SVG allows discrete click-to-seek, hover inspection, and perfect vector scaling during zoom; uses slightly more DOM memory. |
| **Pre-computed Server Heatmaps** | **Client-Side Canvas Compositing** | Allows real-time interactive toggling and multi-layer blending (Traffic + Kills + Deaths) with zero server lag. |
| **Mandatory Cloud LLM API** | **Local Ollama with Rule Fallback** | Zero data privacy risk (telemetry never leaves the machine); works 100% offline via built-in deterministic heuristic fallback. |
| **Absolute Wall-Clock Playback** | **Relative Elapsed Playback** | Relative time ($00:00 \to 15:00$) allows multi-match journey overlay and direct pacing comparisons regardless of when matches started. |
