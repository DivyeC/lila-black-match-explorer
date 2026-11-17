# LILA BLACK · Match Telemetry & Level Intelligence Explorer

An interactive, high-performance telemetry analysis suite built for **LILA Games** level designers and gameplay analysts. The tool visualizes player journeys, combat engagements, loot distribution, and spatial density across recorded matches from production telemetry datasets.

[![Live Deployment](https://img.shields.io/badge/Live%20Demo-GitHub%20Pages-5cf0e0?style=for-the-badge&logo=github)](https://divyec.github.io/lila-black-match-explorer/)
[![Repository](https://img.shields.io/badge/GitHub-DivyeC%2Flila--black--match--explorer-coral?style=for-the-badge&logo=github)](https://github.com/DivyeC/lila-black-match-explorer)

---

## 🌐 Working Deployment

- **Live Production URL:** **[https://divyec.github.io/lila-black-match-explorer/](https://divyec.github.io/lila-black-match-explorer/)**
- **Alternative Static Hosts:** Fully compatible with 1-click deployment on [Vercel](https://vercel.com/), [Netlify](https://www.netlify.com/), or any static web server (zero build step required).

---

## 🛠 Tech Stack

| Layer | Technology | Rationale & Purpose |
|---|---|---|
| **Core Frontend** | **Vanilla JavaScript (ES2022)** | Zero framework bloat (no React/Vue/Webpack overhead). Sub-millisecond UI reactivity, lightweight bundle size (<90 KB total JS/CSS), and instant boot time. |
| **Vector Map Layer** | **SVG 1.1 with Non-Scaling Strokes** | Crisp, resolution-independent player route traces and interactive event markers at arbitrary zoom and pan levels. |
| **Density Heatmaps** | **HTML5 Canvas 2D (`OffscreenCanvas` composition)** | Custom Gaussian kernel density estimation supporting multi-layer compositing (**Traffic**, **Kills**, **Deaths**) in real-time. |
| **UI & Visual Design** | **Modern CSS3 & Design System** | Cyberpunk tactical aesthetic aligned with the official LILA Games BLACK identity. Responsive across desktop, tablet, and mobile. |
| **AI Co-Pilot** | **Local Ollama (`llama3.2`) + Fallback Engine** | On-demand level design co-pilot providing automated level pacing reviews with real-time typewriter markdown streaming. |
| **Data Pipeline** | **Python 3.10+ (`PyArrow`, `Pillow`)** | Ingests raw Snappy-compressed Parquet files, performs coordinate transforms, globally deduplicates sessions, and exports chunked JSON. |
| **CI / CD Deployment** | **GitHub Actions + GitHub Pages** | Automated static deployment pipeline (`.github/workflows/deploy.yml`) on every push to `main`. |

---

## 🚀 Setup & Local Execution

### Prerequisites
- Modern web browser (Chrome, Edge, Firefox, Safari)
- Optional for local pipeline rebuilds: Python 3.10 or newer

### 1. Clone the Repository
```bash
git clone https://github.com/DivyeC/lila-black-match-explorer.git
cd lila-black-match-explorer
```

### 2. Run the Tool (Instant Local Server)
Because the pre-processed telemetry datasets (`data/`) and optimized WebP minimaps (`assets/minimaps/`) are bundled directly within the repository, no build step is needed to run the tool:

```bash
# Using Python
python -m http.server 8500 --bind 127.0.0.1

# Or using Node.js
npx serve -l 8500 .
```

Open your browser to: **`http://localhost:8500`**

*(Note: Do not open `index.html` via `file://` protocol due to browser CORS policies on relative JSON and WebP fetching).*

---

## ⚙️ Environment Variables & Configuration

- **Zero Required Environment Variables:** The application runs 100% client-side with static JSON and WebP assets. There are no API keys, secrets, or cloud database credentials required.
- **Optional Local AI Inference:** To use live local LLM inference for the **05 Level Intelligence Co-Pilot**, start Ollama locally:
  ```bash
  ollama run llama3.2
  ```
  *(If Ollama is not running, the application automatically switches to its built-in simulated telemetry engine, ensuring full offline functionality).*

---

## 🔄 Data Pipeline Rebuild (Optional)

If you have new or raw Parquet files from game telemetry (`February_10/` to `February_14/`):

1. Create and activate a Python virtual environment:
   ```bash
   python -m venv .venv
   # Windows
   .venv\Scripts\Activate.ps1
   # macOS / Linux
   source .venv/bin/activate
   ```
2. Install pipeline requirements:
   ```bash
   pip install -r requirements.txt
   ```
3. Run the data compiler:
   ```bash
   python scripts/build_data.py --input-root . --output-dir data --minimap-dir assets/minimaps
   ```
This updates `data/manifest.json`, date/match JSON splits, and aspect-preserving WebP minimap derivatives.

---

## 🎯 Key Capabilities

1. **Spatial Path Reconstruction:** Follow exact human and bot movement paths with color-coded vector lines, directional waypoints, and out-of-bounds boundary detection.
2. **Interactive Event Filtering:** Filter telemetry across the minimap, timeline scrubber, and signal log by:
   - ⚡ **Storm deaths** (`KilledByStorm`)
   - ⚔ **Combat records** (`Kill`, `BotKill`, `Killed`, `BotKilled`)
   - ◈ **Loot records** (`Loot`)
   - Interactive MAP KEY buttons for 1-click map isolation.
3. **Multi-Layer Density Heatmaps:** Concurrently toggle and overlay **◈ Traffic Density** (cyan), **⚔ Kill Locations** (coral), and **☠ Casualties** (violet) with Gaussian smoothing.
4. **Playback Scrubber & Speed Control:** Time-synced scrubbing from $0.25\times$ to $32\times$ with discrete event ticks along the timeline.
5. **Section 05 Level Design Intelligence:**
   - **Insights Dashboard:** Automated extraction pacing (First Blood timer, combat phase breakdown, bot-to-human threat ratio, storm lethality).
   - **Smart Analysis Co-Pilot:** AI tactical layout synthesis with selectable focus areas (Spawn Spacing, Choke Points, Storm Circle Pressure, Comprehensive Review).
   - **Direct Combat Feed & All Signals Log:** Clickable telemetry rows that jump the map and playback timeline directly to the incident.
6. **Multi-Match & Multi-Day Comparison:** Aggregate journeys across multiple matches or entire play sessions to detect recurring bottlenecks.

---

## 📁 Repository Structure

```
├── .github/workflows/deploy.yml  # GitHub Pages continuous deployment action
├── assets/                       # Minimap textures, brand artwork, fonts
│   ├── brand/                    # LILA Games official visual assets
│   ├── fonts/                    # Bundled typography (Barlow Condensed, Space Grotesk)
│   └── minimaps/                 # Optimized WebP minimap layers (Ambrose, Grand Rift, Lockdown)
├── data/                         # Preprocessed telemetry datasets & manifest index
│   ├── manifest.json             # Map, date, match, and file index
│   └── *.json                    # Map/date match trajectory slices
├── scripts/                      # Data engineering pipeline
│   └── build_data.py             # Parquet extraction, deduplication & coordinate mapper
├── index.html                    # Single-page application markup
├── styles.css                    # Tactical UI stylesheet & responsive grid
├── app.js                        # Viewer state, SVG/Canvas renderer, event scrubber
├── ARCHITECTURE.md               # 1-page technical architecture & coordinate math doc
├── INSIGHTS.md                   # 3 evidence-backed gameplay insights for level designers
├── requirements.txt              # Pipeline dependencies (pyarrow, pillow)
└── README.md                     # Documentation & setup guide
```

---

## 📑 Core Documentation Links

- **[ARCHITECTURE.md](ARCHITECTURE.md)**: Deep dive into the coordinate projection math, data pipeline, design assumptions, and architecture tradeoffs.
- **[INSIGHTS.md](INSIGHTS.md)**: Three concrete, evidence-backed gameplay insights with actionable level design modifications and impact metrics.
- **[DATA_FORMAT.md](DATA_FORMAT.md)**: Telemetry schema specification, coordinate ranges, and event taxonomy.
