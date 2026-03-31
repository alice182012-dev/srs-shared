# FloodNav Hanoi — Benchmark Architecture & Test Plan

**Three Workflow Options · On-Demand 24h Forecast · Detailed Test Cases**

Version 0.2 — March 2026 | Classification: Internal Technical Review

---

## 1. Shared Foundation (Common to All Workflows)

### 1.1 System Assumptions

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Engine | 3Di (Nelen & Schuurmans) | Full 0D-1D-2D coupled, sub-grid native, cloud + on-prem HPC |
| Compute | Cray supercomputer (on-premises) | Unlimited budget; parallel 3Di instances |
| Domain | Hanoi urban core ~300 km² (12 inner districts) | Primary flood-affected zone |
| DTM | 1m LiDAR | Acquired, processed, buildings burned in |
| Drainage network | Full combined sewer, rivers, canals | HUDC data integrated |
| Trigger | Independent forecast model threshold | System idles when dry |
| Real-time forecast horizon | 2 hours | 9 output timesteps at 15-min intervals |
| Real-time SLA | 30 minutes wall-clock | From trigger to CDN tile update |
| Extended forecast (24h) | On-demand only | Triggered by operator or synoptic event classification |
| Scenario library | Pre-computed per tile at 4m sub-grid | Instant Layer 1 response + extended forecast fallback |

### 1.2 On-Demand 24h Forecast (Replaces Scheduled background_cycle)

The old architecture ran a 24h simulation every 3 hours on a fixed schedule. This is replaced by:

**Watch 🟡 alert** — generated directly from the independent rainfall forecast model crossing a threshold. No hydrodynamic simulation needed. Alert text: *"Mưa lớn dự báo có thể gây ngập khu vực của bạn trong 24h tới."*

**Institution histograms (hours 3–24)** — served from the scenario library's closest-match pre-computed results. Visual treatment: hatched/faded bars with explicit "dự báo kịch bản" (scenario-based projection) label, distinct from solid "mô phỏng thời gian thực" (real-time simulation) bars.

**Dashboard 24h forecast panel** — scrubs through scenario library time evolution. Clear visual distinction between real-time data (first 2h, high confidence) and scenario projection (hours 3–24, degrading confidence).

**On-demand 24h trigger conditions:**
- Operator manual trigger from city council dashboard
- Automatic trigger when NCHMF issues typhoon warning or prolonged heavy rain advisory (synoptic-scale event with reliable 24h rainfall forecast)
- Automatic trigger when 3 consecutive constant_cycles show increasing peak depth trend

**On-demand 24h simulation spec:**
- Duration: 86,400s (24h), output timestep: 3,600s (60 min), 25 GeoTIFFs
- Uses latest NCHMF 24h forecast + current gauge readings
- Runs on separate Cray compute pool (does not compete with constant_cycle resources)
- Budget: 60–90 min (no SLA pressure — results improve scenario library data)
- Output overwrites scenario library bars 3–24 in TimescaleDB with fresher data
- Applies to all 3 workflows identically

### 1.3 Tile Design (Common to WF1 & WF2)

| Rule | Specification |
|------|---------------|
| Tile boundaries | Aligned with topographic ridges + major 1D network junctions |
| Tile size | 5–15 km² (~30 tiles for urban core) |
| Overlap buffer | 100–200m per side (simulated by both adjacent tiles) |
| 1D network at boundaries | Trunk sewers/channels included in BOTH tiles within overlap zone. BC applied at nominal edge. |
| Pre-built schematisations | One 3Di gridadmin.h5 per tile, versioned |
| Scenario library | Per-tile, pre-computed at 4m sub-grid. ~360 scenarios per tile (5 intensities × 4 durations × 6 spatial patterns × 3 initial sewer states) |

### 1.4 Scenario Library Boundary Seeding

For tiled workflows (WF1, WF2), the scenario library provides **initial boundary conditions** at T+0. When the forecast trigger fires:

1. Forecast rainfall pattern is matched against the scenario library
2. The closest-match scenario's cross-tile boundary flows (Q and h at each tile edge) are extracted
3. These serve as the initial boundary conditions for the first real-time simulation cycle
4. This avoids the "cold start" problem where tiles have no information about neighbours

### 1.5 Postprocessing Pipeline (Common to All Workflows)

All workflows produce a GeoTIFF series that feeds the same postprocessing pipeline:

**Tier 0 — Flood Metric Extractor** (sequential, must complete first):
- Per grid cell and road segment: peak depth time, recession start time, return-to-passable time per vehicle type
- Pre-closure flags: first timestep where T+1h forecast depth ≥ hard block threshold
- **Wall-clock re-anchoring**: all times reported relative to pipeline_completion_time, not simulation T+0
- Forecast depths at T+15m, T+30m, T+1h, T+1h30, T+2h per segment → PostGIS

**Tier 1 — Parallel Workers** (fan-out from Redis):
- Tile generator (T+0 GeoTIFF → colour ramp → XYZ PNG → CDN)
- Segment extractor (sample road network → per-segment depth → PostGIS → re-weight Valhalla)
- Institution histogram builder (bars 1–4 from real-time data, solid style)
- User alert engine (Danger 🔴 / Warning 🟠 push notifications)

**Tier 2 — Dependent Workers** (blocked until segment extractor completes):
- Segment alert engine (pre-closure warnings per segment)
- District aggregator (impassable segment counts, institution access, district summary)

---

## 2. Workflow 1 — Coarse Mother → Fine Children

### 2.1 Architecture Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    FORECAST TRIGGER                         │
└──────────────────────────┬──────────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │  Scenario Library Match  │ ← Instant Layer 1 → CDN
              │  (per tile, 4m quality)  │
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │    MOTHER MODEL         │ Single 3Di simulation
              │    20–40m comp grid     │ Full city, 1m DTM sub-grid
              │    ~200k–750k cells     │ Full 1D-2D coupled
              │    2h sim, 1-min output │ Runtime: 2–4 min
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │  BOUNDARY EXTRACTION    │ Extract h(t), Q(t) at
              │  (Cray CPU, <1 min)     │ every tile edge from mother
              └────────────┬────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
  ┌──────────┐      ┌──────────┐       ┌──────────┐
  │ CHILD 1  │      │ CHILD 2  │  ...  │ CHILD 30 │  30× parallel
  │ 4m sub   │      │ 4m sub   │       │ 4m sub   │  3Di instances
  │ 300k–900k│      │ 300k–900k│       │ 300k–900k│  One-way BCs
  │ cells    │      │ cells    │       │ cells    │  from mother
  │ 8–14 min │      │ 8–14 min │       │ 8–14 min │
  └────┬─────┘      └────┬─────┘       └────┬─────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          ▼
              ┌───────────────────────┐
              │  STITCH + POSTPROCESS │ Merge tiles, discard
              │  (5–8 min)            │ overlap zones, run
              │                       │ standard Tier 0/1/2
              └───────────────────────┘
```

### 2.2 Time Budget

| Stage | Duration | Cumulative | Resource |
|-------|----------|------------|----------|
| Trigger + ingestion | 1 min | T+1 | Cray CPU |
| Scenario library → CDN (Layer 1) | 0.5 min | T+1.5 | App servers |
| Mother model (20–40m sub-grid) | 2–4 min | T+5 | 1× 3Di on Cray |
| Boundary extraction | 0.5 min | T+5.5 | Cray CPU |
| Children ×30 parallel (4m sub-grid) | 8–14 min | T+19 | 30× 3Di on Cray |
| Stitch + postprocessing | 5–8 min | T+27 | Cray CPU cluster |
| CDN + API update | 1–2 min | T+29 | App servers |
| **Total** | **~25–29 min** | | **Within 30-min SLA** |

### 2.3 Cross-Tile Coupling

- **Method**: One-way (mother → children). No feedback from children to mother.
- **2D boundary**: Water level time series from mother at each tile edge (1-min temporal resolution).
- **1D boundary**: Discharge time series from mother at each pipe/channel crossing point.
- **Weakness**: Downstream backwater cannot propagate upstream. If a child tile floods heavily and backs up, the upstream child doesn't know.
- **Mitigation**: The mother model captures the coarse backwater at 20–40m resolution. The children refine locally but don't need to re-propagate it.

### 2.4 3Di API Call Pattern

```
1× simulations_from_template (mother)       → constant_cycle template variant (20m)
1× upload initial_waterlevels.csv            → gauge data
1× upload rain.nc                            → NCHMF forecast
1× start + poll (mother)                     → ~3 min
1× download results_3di.nc (mother)          → boundary extraction source
---
30× simulations_from_template (children)     → per-tile templates (4m sub-grid)
30× upload initial_waterlevels.csv           → gauge data + mother water levels
30× upload boundary_conditions.csv           → extracted from mother
30× upload rain.nc                           → same NCHMF forecast
30× start + poll (children)                  → ~12 min parallel
30× download results_3di.nc (children)       → postprocessing input
---
Total API calls: ~124 per cycle
```

### 2.5 On-Demand 24h Integration

When 24h forecast is triggered:
- Run a **separate mother model** with 24h duration, 60-min output, 20–40m grid → ~15–20 min
- Extract 24h boundary conditions from mother
- Run 30 children with 24h duration, 60-min output, 4m sub-grid → ~60–90 min parallel
- Results overwrite scenario library bars 3–24 in TimescaleDB
- Does NOT interfere with constant_cycle — runs on separate Cray compute pool

---

## 3. Workflow 2 — Iterative Boundary Exchange (Schwarz)

### 3.1 Architecture Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    FORECAST TRIGGER                         │
└──────────────────────────┬──────────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │  Scenario Library Match  │ ← Instant Layer 1 → CDN
              │  + Boundary Seeding     │ ← Initial BCs for chunk 1
              └────────────┬────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         │         CHUNK LOOP (×8)           │
         │  Each chunk = 15 min sim time     │
         │                                   │
         │  ┌─────────┐ ┌─────────┐          │
         │  │ Tile 1  │ │ Tile 2  │ ... ×30  │  All parallel
         │  │ 4m sub  │ │ 4m sub  │          │  15 min sim
         │  │ 900k    │ │ 900k    │          │  per chunk
         │  │ ~2 min  │ │ ~2 min  │          │
         │  └────┬────┘ └────┬────┘          │
         │       │           │               │
         │       ▼           ▼               │
         │  ┌──────────────────────┐         │
         │  │ BOUNDARY EXCHANGE    │         │  Extract h,Q at
         │  │ COORDINATOR          │         │  all tile edges,
         │  │ (~0.5 min)           │         │  distribute to
         │  └──────────┬───────────┘         │  neighbours
         │             │                     │
         │  ┌──────────▼───────────┐         │
         │  │ RESTART FROM SAVED   │         │  3Di saved state
         │  │ STATE + NEW BCs      │         │  at chunk boundary
         │  └──────────────────────┘         │
         │             │                     │
         │         [next chunk]              │
         └─────────────┼─────────────────────┘
                       ▼
              ┌───────────────────────┐
              │  STITCH + POSTPROCESS │  After all 8 chunks
              │  (3–4 min)            │  Standard Tier 0/1/2
              └───────────────────────┘
```

### 3.2 Time Budget

| Stage | Duration | Cumulative | Notes |
|-------|----------|------------|-------|
| Trigger + ingestion | 1 min | T+1 | |
| Scenario library → CDN (Layer 1) | 0.5 min | T+1.5 | + boundary seeding |
| Chunk 1 (30 tiles × 15 min sim) | 3 min | T+4.5 | BCs from scenario library |
| Chunk 2 | 3 min | T+7.5 | BCs from chunk 1 exchange |
| Chunk 3 | 3 min | T+10.5 | |
| Chunk 4 | 3 min | T+13.5 | **→ Serve T+1h intermediate results** |
| Chunks 5–8 | 12 min | T+25.5 | |
| Final stitch + postprocessing | 3–4 min | T+29 | Full 2h horizon |
| **Total** | **~27–29 min** | | **Within 30-min SLA (tight)** |

### 3.3 Cross-Tile Coupling

- **Method**: Bidirectional exchange every 15 minutes of simulated time.
- **Mechanism**: At each chunk boundary, every tile exports its edge water levels and discharges. The coordinator redistributes these as boundary conditions for the next chunk. Each tile restarts from its 3Di saved state with updated BCs.
- **2D boundary**: Weighted-average water level from both tiles at shared edge cells.
- **1D boundary**: Downstream tile receives upstream tile's pipe discharge as inflow BC.
- **Strength**: Captures backwater propagation with 1-chunk (15-min sim time) delay.
- **Weakness**: Fast pressure waves in fully pressurised sewers travel faster than 15-min exchange interval. Some surcharge events may lag by one chunk.

### 3.4 3Di API Call Pattern (Per Cycle)

```
Per chunk (×8 chunks):
  30× simulations_from_template OR simulations_from_saved_state
  30× upload boundary_conditions.csv
  30× upload rain.nc (or inherit from template)
  30× start + poll
  30× save_state at chunk end
  30× download results for boundary extraction
  1×  boundary exchange coordinator run
---
Total: 30 tiles × 8 chunks × ~5 API calls = ~1,200 API calls per cycle

CRITICAL RISK: API rate limiting, network latency, saved state reliability.
Each failed API call blocks the entire chunk → cascading delay.
```

### 3.5 Intermediate Result Serving

A unique advantage: after chunk 4 completes (T+13.5 wall-clock), the first hour of forecast data is available. The pipeline can serve **intermediate results** covering T+0 to T+1h while chunks 5–8 continue running. Citizens get partial data 15 minutes earlier than WF1 or WF3.

### 3.6 On-Demand 24h Integration

When 24h forecast is triggered:
- Run 30 tiles with 24h duration using chunk loop (chunk size: 60 min sim time, 24 chunks)
- Exchange boundaries every chunk
- Budget: ~90–120 min (24 chunks × ~4 min each)
- Most complex 24h variant due to chunk orchestration
- Separate Cray compute pool

---

## 4. Workflow 3 — Single Mega-Model (8m + 4m Refinement)

### 4.1 Architecture Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    FORECAST TRIGGER                         │
└──────────────────────────┬──────────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │  Scenario Library Match  │ ← Instant Layer 1 → CDN
              │  (city-wide, 4m library) │
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │    PRIMARY MODEL        │ Single 3Di simulation
              │    8m computational     │ 1m DTM sub-grid
              │    ~4.7M cells          │ Full 0D-1D-2D coupled
              │    Complete sewer       │ Zero tile boundaries
              │    network intact       │ Zero stitching
              │    2h sim, 15-min out   │ Runtime: 8–15 min
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │  POSTPROCESSING         │ Direct to Tier 0/1/2
              │  No stitching needed    │ 5–7 min
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │  4m REFINEMENT LAYER    │ Parallel, non-blocking
              │  (priority districts)   │ Runs AFTER 8m results
              │  Only threat-zone tiles │ served; overwrites
              │  5–10 tiles × 4m sub   │ specific areas when ready
              │  Runtime: 10–15 min     │
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │  REFINEMENT MERGE       │ Overwrite 8m depth in
              │  (2–3 min)              │ refined areas with 4m data
              │                         │ Re-run segment extractor
              │                         │ for affected segments only
              └───────────────────────────┘
```

### 4.2 The 8m + 4m Refinement Strategy

The primary model runs at **8m computational grid with 1m DTM sub-grid** and covers the entire domain in one shot. This is the backbone — it provides correct 1D sewer coupling, correct river hydraulics, and correct mass conservation everywhere.

The **4m refinement layer** runs afterward for **threat-zone tiles only** — the districts where the primary model detected the worst flooding. This is NOT a separate workflow; it's a quality enhancement that improves street-level depth discrimination for routing in the most affected areas.

| Aspect | Primary (8m) | Refinement (4m) |
|--------|-------------|-----------------|
| Domain | Entire urban core (300 km²) | 5–10 worst-affected tiles only |
| Computational grid | 8m | 4m |
| Cell count | ~4.7M | ~600k–1.2M per tile |
| 1D network | Complete, intact | Per-tile subset |
| Boundary conditions | Self-contained (no BCs needed) | Extracted from primary model results |
| Purpose | Threshold detection, pre-closure, routing backbone | Improved route discrimination between parallel streets |
| SLA | Must complete within 30 min | Non-blocking. Results arrive ~T+40–45 min |
| Output treatment | Served immediately as definitive data | Overwrites primary data in refined areas |

### 4.3 Refinement Tile Selection Logic

After the primary model's metric extractor completes, the refinement selector:

1. Identifies all road segments where current or T+1h depth is within **±10cm of any vehicle threshold** (the "ambiguous zone" where 8m sub-grid averaging might misclassify)
2. Groups these segments by tile
3. Ranks tiles by count of ambiguous segments × segment traffic importance
4. Selects top 5–10 tiles for refinement
5. Launches 4m sub-grid 3Di simulations for these tiles with BCs from primary model

### 4.4 Time Budget

| Stage | Duration | Cumulative | Blocking? |
|-------|----------|------------|-----------|
| Trigger + ingestion | 1 min | T+1 | Yes |
| Scenario library → CDN (Layer 1) | 0.5 min | T+1.5 | Yes |
| Primary model (8m sub-grid, single) | 8–15 min | T+16 | Yes |
| Depth calculation (9 timesteps) | 2–3 min | T+19 | Yes |
| Postprocessing (Tier 0/1/2) | 5–7 min | T+26 | Yes |
| CDN + API update | 1–2 min | T+28 | Yes |
| **Primary complete** | | **T+28** | **SLA MET** |
| Refinement tile selection | 0.5 min | T+28.5 | No |
| 4m refinement ×5–10 tiles (parallel) | 10–15 min | T+43 | No |
| Refinement merge + segment re-extract | 2–3 min | T+46 | No |
| Refined CDN update | 1 min | T+47 | No |

Citizens see 8m-quality data at **T+28 min** (within SLA), then silently upgraded to 4m quality in worst-hit areas at **T+47 min**. The app does not interrupt the user — the map tiles simply improve.

### 4.5 3Di API Call Pattern

```
PRIMARY (blocking):
  1× simulations_from_template (8m mega-model)
  1× upload initial_waterlevels.csv
  1× upload rain.nc
  1× start + poll → 8–15 min
  1× download results_3di.nc
  Total: ~5 API calls

REFINEMENT (non-blocking, after primary):
  5–10× simulations_from_template (4m per-tile models)
  5–10× upload boundary_conditions.csv (from primary)
  5–10× upload rain.nc
  5–10× start + poll → ~12 min parallel
  5–10× download results
  Total: ~25–50 API calls

Grand total: ~30–55 API calls per cycle
```

### 4.6 Cross-Tile Accuracy

| Aspect | Rating | Notes |
|--------|--------|-------|
| 1D sewer coupling | ★★★★★ | Perfect. Single matrix. The #1 accuracy driver for threshold detection. |
| River backwater | ★★★★★ | Perfect. All rivers continuous. |
| Mass conservation | ★★★★★ | Perfect. Single model. |
| Threshold detection (8m primary) | ★★★★☆ | Sub-grid tables preserve threshold crossings well. Occasional ambiguity in narrow streets. |
| Threshold detection (4m refined) | ★★★★★ | Near-explicit resolution. Parallel streets distinguishable. |
| Route discrimination | ★★★★★ | 8m for backbone + 4m for worst areas = best practical combination. |
| Boundary artefacts | ★★★★★ | None in primary. Refinement tiles use primary BCs (one-way, same as WF1 children). |

### 4.7 On-Demand 24h Integration

When 24h forecast is triggered:
- Run the **same mega-model** with 24h duration, 60-min output, 8m sub-grid → ~45–60 min
- Simplest 24h variant: one API call, one simulation, one download
- No refinement layer for 24h (unnecessary at hourly output resolution)
- Results overwrite scenario library bars 3–24 in TimescaleDB
- Separate Cray compute pool

---

## 5. Head-to-Head Comparison

| Criterion | WF1: Coarse→Fine | WF2: Iterative | WF3: 8m+4m Refinement |
|-----------|-------------------|----------------|----------------------|
| **1D sewer coupling fidelity** | ★★★☆☆ (cut at tiles) | ★★★★☆ (exchanged every 15 min) | ★★★★★ (single matrix) |
| **Threshold detection accuracy** | ★★★★☆ | ★★★★☆ | ★★★★★ (primary) + ★★★★★ (refined) |
| **Pre-closure timing accuracy** | ★★★☆☆ (one-way lag) | ★★★★☆ (1-chunk delay) | ★★★★★ (no lag) |
| **Mass conservation** | ★★★☆☆ | ★★★★☆ | ★★★★★ |
| **Effective resolution** | 4m everywhere | 4m everywhere | 8m base + 4m in worst areas |
| **SLA risk** | Low (5 min margin) | High (1–3 min margin) | Low (2–4 min margin) |
| **Intermediate results** | No | Yes (every chunk) | No (but faster primary) |
| **Orchestration complexity** | Moderate | Very high | Low (primary) + Moderate (refinement) |
| **API calls per cycle** | ~124 | ~1,200 | ~30–55 |
| **Failure blast radius** | 1 child fails → 1 tile stale | 1 chunk fails → all tiles delayed | 1 sim fails → total stale (high) |
| **Model build effort** | 31 schematisations | 30 schematisations + exchange logic | 1 mega + 30 tile models |
| **Debug/validate ease** | Moderate | Hard | Easy (primary) + Moderate (refinement) |
| **24h on-demand complexity** | Moderate (mother + 30 children) | High (24 chunks × 30 tiles) | Low (single simulation) |
| **Recommended role** | Pilot / fallback | Research / validation | **Primary production system** |

---

## 6. Benchmark Test Plan

### 6.1 Reference Event Selection

All tests use the **same historical flood event** applied to the **same geographic domain**.

**Primary test event**: Select a well-documented Hanoi flood event with:
- Available NCHMF rainfall data (or radar-derived QPE)
- Available gauge water level records (≥5 stations within domain)
- Available crowd-sourced flood reports (social media photos with timestamps and locations)
- Flood duration 2–6 hours (tests both the 2h real-time and 24h on-demand horizons)

**Candidate events** (requires data verification):
- July 2018 Hanoi flood (widespread urban flooding from intense rainfall)
- October 2008 (prolonged monsoon, Red River involvement)
- May 2022 (localised convective event)

If historical data is insufficient, use a **synthetic design storm**: 100mm/2h, spatially uniform over the urban core, with NCHMF-typical spatial uncertainty applied.

### 6.2 Reference Simulation (Ground Truth)

Before benchmarking the 3 workflows, establish a **reference simulation** that serves as "truth":

| Parameter | Reference Specification |
|-----------|------------------------|
| Model | Single 3Di schematisation, full urban core |
| Computational grid | **2m** (finest feasible for city-wide on Cray) |
| DTM | 1m LiDAR, sub-grid enabled |
| 1D network | Complete, no cuts |
| Duration | 6 hours (captures full event including recession) |
| Output timestep | 5 min (high temporal resolution for threshold timing analysis) |
| Runtime budget | Unlimited (offline — expected 12–24 hours on Cray) |
| Output | Full results_3di.nc + 72× depth GeoTIFFs |

This reference simulation provides the "true" depth at every road segment at every timestep. All workflow outputs are compared against this reference.

### 6.3 Test Metrics

#### 6.3.1 Threshold Classification Metrics (Primary — Navigation-Critical)

For each road segment S, vehicle type V, and timestep T:

- **Ground truth label**: `flooded(S, V, T) = 1 if reference_depth(S, T) ≥ threshold(V), else 0`
- **Workflow prediction**: `predicted(S, V, T) = 1 if workflow_depth(S, T) ≥ threshold(V), else 0`

| Metric | Formula | Target | Why It Matters |
|--------|---------|--------|----------------|
| **True Positive Rate (Recall)** | TP / (TP + FN) | ≥ 95% | Citizens must be warned about flooded roads. Missed floods are dangerous. |
| **False Positive Rate** | FP / (FP + TN) | ≤ 15% | Too many false closures erode trust and cause unnecessary detours. |
| **F1 Score** | 2×(Precision×Recall)/(Precision+Recall) | ≥ 0.90 | Balanced measure of detection quality. |
| **Segment-level accuracy** | (TP + TN) / total | ≥ 90% | Overall correctness rate. |

Computed per vehicle type (xe máy is most important — lowest threshold, most users).

#### 6.3.2 Threshold Timing Metrics (Critical for Pre-Closure)

For each segment that is correctly classified as flooded by both reference and workflow:

| Metric | Definition | Target | Why It Matters |
|--------|------------|--------|----------------|
| **Crossing time error** | `workflow_crossing_time(S,V) − reference_crossing_time(S,V)` | Mean < 5 min, P95 < 15 min | Pre-closure alert accuracy depends on correct timing. |
| **Early bias** | % of segments where workflow predicts crossing earlier than reference | ≥ 60% | Better to warn early than late. Slight over-prediction is desirable. |
| **Peak depth error** | `workflow_peak_depth(S) − reference_peak_depth(S)` | RMSE < 5cm | Peak depth determines severity classification. |
| **Recession time error** | `workflow_recession_time(S,V) − reference_recession_time(S,V)` | Mean < 10 min | Affects "when can I use this road again" prediction. |

#### 6.3.3 Hydraulic Fidelity Metrics (Secondary — Validation)

| Metric | Definition | Target |
|--------|------------|--------|
| Depth RMSE (all wet cells) | √(mean(workflow_depth − reference_depth)²) | < 8cm |
| Depth RMSE (road segments only) | Same, filtered to road network | < 5cm |
| Mass balance error | (total_volume_workflow − total_volume_reference) / total_volume_reference | < 5% |
| Gauge comparison | Workflow vs. observed gauge water level | RMSE < 10cm |
| Spatial flood extent overlap (IoU) | Intersection / Union of wet areas | ≥ 85% |

#### 6.3.4 Operational Metrics

| Metric | Definition | Target |
|--------|------------|--------|
| Wall-clock time (total pipeline) | Trigger → CDN update | ≤ 30 min |
| Simulation time (3Di only) | Simulation start → results downloaded | ≤ 18 min |
| API call count | Total 3Di API calls per cycle | Logged |
| API failure rate | Failed calls / total calls | < 1% |
| Postprocessing time | Tier 0 + Tier 1 + Tier 2 | ≤ 10 min |
| Cray GPU utilisation | Peak and average during cycle | Logged |
| Scenario library match quality | RMSE of library match vs. real-time result | Tracked |

### 6.4 Test Cases

#### TEST CASE 1 — Single-Tile Sub-Grid Accuracy

**Purpose**: Validate that 8m and 4m sub-grid produce acceptable threshold classification vs. explicit 2m reference.

**Setup**:
- Select one district (~10 km²) with dense road network and known flood-prone segments
- Three models: (a) 2m reference (no sub-grid), (b) 4m sub-grid, (c) 8m sub-grid
- Same rainfall, same 1D network, same initial conditions
- Single 3Di simulation each, 2h duration, 5-min output

**Procedure**:
1. Run all three models with identical inputs
2. Extract depth at every road segment centroid at every output timestep
3. Classify each segment as flooded/not-flooded for each vehicle type at each timestep
4. Compute threshold classification metrics (6.3.1) for 4m and 8m vs. 2m reference
5. Compute crossing time error (6.3.2) for correctly classified segments
6. Identify all segments where 8m and 4m disagree — these are the "ambiguous zone" that the refinement layer targets

**Success criteria**:
- 4m sub-grid: Recall ≥ 97%, FPR ≤ 10%, crossing time RMSE < 3 min
- 8m sub-grid: Recall ≥ 93%, FPR ≤ 15%, crossing time RMSE < 8 min
- Ambiguous zone segments < 10% of total road network

**Deliverable**: Confusion matrices per vehicle type, crossing-time histograms, map of ambiguous segments.

---

#### TEST CASE 2 — Two-Tile Boundary Coupling

**Purpose**: Validate cross-tile boundary handling for all three workflows against a non-split reference.

**Setup**:
- Select two adjacent districts (~20 km² total) with a trunk sewer crossing the boundary and the Tô Lịch river crossing the boundary
- Build: (a) non-split 2m reference model, (b) WF1 split (mother at 20m + 2 children at 4m), (c) WF2 split (2 tiles at 4m, 4 chunks), (d) WF3 single model at 8m
- Overlap buffer: 200m
- Same rainfall, 2h duration

**Procedure**:
1. Run all four configurations with identical inputs
2. Focus analysis on a **50m corridor along the tile boundary** — this is where coupling errors manifest
3. Extract depth time series at 20 road segments within the corridor
4. Compute threshold classification metrics within the corridor vs. reference
5. Compute mass balance: compare total water volume in each tile vs. reference
6. Visually inspect depth maps at the boundary for seam artefacts

**Key measurements**:
- For each segment in the corridor: does the workflow correctly predict threshold crossing? At the right time?
- Does a downstream surcharge in tile B propagate to tile A? How much delay?
- Does river backwater from tile B affect tile A?

**Success criteria**:
- All workflows: boundary corridor recall ≥ 90%, crossing time error < 10 min
- WF2 and WF3: mass balance error < 3%
- WF1: mass balance error < 8% (expected weaker due to one-way coupling)
- No visible seam in output depth maps (manual visual inspection)

**Deliverable**: Depth time series plots at boundary segments for all 4 configurations overlaid, mass balance table, seam inspection images.

---

#### TEST CASE 3 — Sewer Surcharge Propagation

**Purpose**: Specifically test the 1D sewer coupling accuracy that we identified as the #1 accuracy driver for threshold detection.

**Setup**:
- Select a sub-domain (~15 km²) containing a **known surcharge-prone trunk sewer** that spans at least 2 tile boundaries
- In the reference model, induce surcharge by applying heavy rainfall + reduced pipe capacity (simulate partial blockage)
- Record: which manholes surcharge, at what time, and how much surface flooding results

**Procedure**:
1. Run reference model (2m, non-split, intact sewer network)
2. Record surcharge sequence: manhole A surcharges at T+22 min → manhole B at T+28 min → manhole C at T+35 min
3. Run WF1: does the mother model capture the surcharge sequence? Do the children correctly reflect surcharge at manholes near tile boundaries?
4. Run WF2: does the chunk-based exchange propagate the surcharge pressure wave across tile boundaries? How much delay?
5. Run WF3 (8m): does the single mega-model capture all three surcharge events with correct timing?

**Key measurements**:
- Surcharge timing error per manhole (workflow vs. reference)
- Surface flood depth at road segments above surcharging manholes
- Threshold crossing timing at these critical segments

**Success criteria**:
- WF3: surcharge timing error < 2 min for all manholes (single matrix = should be near-perfect)
- WF2: surcharge timing error < 5 min (one chunk delay expected)
- WF1: surcharge timing error < 10 min for manholes near tile boundary; < 3 min for interior manholes

**Deliverable**: Surcharge event timeline comparison chart (reference vs. each workflow), surface depth time series at critical segments.

---

#### TEST CASE 4 — City-Scale Runtime Benchmark

**Purpose**: Validate that each workflow completes within 30-min SLA on Cray hardware.

**Setup**:
- Full 30-tile / full-domain configuration for all three workflows
- Realistic NCHMF rainfall input (replayed from historical event)
- Realistic gauge initial conditions
- Full postprocessing pipeline enabled (Tier 0, 1, 2)

**Procedure**:
1. Run each workflow 10 consecutive times (simulate 10 consecutive constant_cycles)
2. Log detailed timing for every stage: ingestion, 3Di API calls (upload, poll, download), simulation, postprocessing stages
3. Identify the critical path bottleneck in each workflow
4. Measure Cray resource utilisation: GPU occupancy, memory, network bandwidth

**Measurements**:

| Metric | WF1 | WF2 | WF3 |
|--------|-----|-----|-----|
| P50 total wall-clock | | | |
| P95 total wall-clock | | | |
| Max wall-clock | | | |
| P50 simulation time | | | |
| API call success rate | | | |
| Postprocessing time | | | |
| Peak GPU utilisation | | | |
| Peak memory | | | |
| Network data transfer (GB/cycle) | | | |

**Success criteria**:
- All workflows: P95 wall-clock ≤ 30 min
- WF3 primary: P95 wall-clock ≤ 28 min (2 min margin)
- No workflow exceeds 35 min in any single run (hard ceiling)
- API failure rate < 1% across all 10 runs

**Deliverable**: Runtime distribution plots per workflow, stage-by-stage waterfall charts, resource utilisation heat maps.

---

#### TEST CASE 5 — City-Scale Threshold Detection (Full Benchmark)

**Purpose**: The definitive test. Compare all three workflows against the city-wide 2m reference for threshold classification accuracy across the entire road network.

**Setup**:
- Full urban core domain (~300 km², ~15,000 road segments)
- 2m reference simulation (offline, unlimited time)
- WF1, WF2, WF3 (8m primary + 4m refinement) run in real-time mode

**Procedure**:
1. Run 2m reference simulation for the historical event (6h, 5-min output)
2. Run all three workflows in real-time mode for the same event
3. For WF3: run primary (8m), then run refinement (4m) for the top 10 tiles
4. Extract threshold classification for every segment, every vehicle type, every 15-min window
5. Compute all metrics from 6.3.1 and 6.3.2

**Analysis**:
- Aggregate metrics (recall, FPR, F1) per workflow, per vehicle type
- Spatial analysis: map the false negatives (missed floods) and false positives (phantom closures) for each workflow. Where do errors concentrate?
- Temporal analysis: are errors worse in the first hour (rising limb) vs. second hour (peak/recession)?
- For WF3: compare 8m-only metrics vs. 8m+4m-refined metrics. Quantify the refinement layer's improvement.
- Cross-tile analysis (WF1, WF2 only): are errors concentrated at tile boundaries?

**Success criteria**:
- WF3 (8m+4m): Recall ≥ 95% (xe máy), F1 ≥ 0.92
- WF1: Recall ≥ 92% (xe máy), F1 ≥ 0.88
- WF2: Recall ≥ 94% (xe máy), F1 ≥ 0.90
- All workflows: crossing time RMSE < 10 min (xe máy segments)
- WF3 refinement improvement: ≥ 2% F1 gain in refined areas vs. 8m-only

**Deliverable**: Full confusion matrix per workflow per vehicle type, spatial error maps, temporal error curves, tile-boundary error analysis, refinement layer marginal improvement quantification.

---

#### TEST CASE 6 — Scenario Library Match Quality

**Purpose**: Validate that the scenario library provides acceptable instant Layer 1 response quality.

**Setup**:
- Use the historical event as the "live" forecast
- Query the scenario library for the closest match
- Compare library match output against the 2m reference and against each workflow's real-time output

**Procedure**:
1. Extract rainfall pattern from historical event
2. Run scenario library matching algorithm
3. Retrieve matched scenario's depth time series for all road segments
4. Compute threshold classification metrics (library match vs. reference)
5. Compute the same metrics for the real-time workflows
6. Calculate: how much does real-time simulation improve over the scenario library?

**Key question**: Is the scenario library good enough that a 30-second response with 85% recall is preferable to waiting 28 minutes for 95% recall?

**Success criteria**:
- Scenario library match: Recall ≥ 85% (xe máy), F1 ≥ 0.80
- Real-time improvement over library: ≥ 8% F1 gain
- Library crossing time RMSE < 20 min (acceptable for instant response)

**Deliverable**: Side-by-side comparison of library vs. real-time accuracy, marginal improvement quantification, analysis of which event types the library handles well vs. poorly.

---

#### TEST CASE 7 — On-Demand 24h Forecast Validation

**Purpose**: Validate the on-demand 24h simulation against extended event observations.

**Setup**:
- Select a prolonged flood event (≥12 hours) with available gauge data covering the full duration
- Run on-demand 24h simulation for each workflow
- Compare against gauge observations and any available late-event depth reports

**Procedure**:
1. Trigger on-demand 24h for each workflow using the event's T+0 NCHMF forecast
2. Record runtime and resource consumption
3. Compare predicted peak timing, recession timing, and return-to-passable timing against observed data
4. Compare institution histogram predictions against actual institutional accessibility reports (if available)

**Success criteria**:
- All workflows complete 24h simulation within 90 min
- Peak timing error < 2 hours (limited by rainfall forecast skill, not hydraulics)
- Recession timing error < 3 hours
- WF3 24h is simplest to operate (1 API call vs. 124+ for WF1 vs. 1200+ for WF2)

**Deliverable**: 24h depth hydrograph comparison (predicted vs. observed) at gauge locations, institution accessibility timeline comparison.

---

## 7. Benchmark Execution Schedule

| Phase | Duration | Activity | Deliverable |
|-------|----------|----------|-------------|
| A | Month 1–3 | Test Case 1 (single-tile sub-grid accuracy) | Sub-grid accuracy report; ambiguous zone map |
| B | Month 3–5 | Test Case 2 + 3 (boundary coupling + sewer propagation) | Cross-tile error report; sewer timing analysis |
| C | Month 5–7 | Test Case 4 (city-scale runtime) | Runtime benchmark report; bottleneck identification |
| D | Month 7–9 | Test Case 5 (city-scale threshold detection) | **Primary benchmark report — workflow recommendation** |
| E | Month 9–10 | Test Case 6 + 7 (scenario library + 24h) | Scenario library quality report; 24h validation |
| F | Month 10–11 | Synthesis and architecture decision | Final architecture selection document |

### 7.1 Go/No-Go Decision Points

**After Phase A**: If 8m sub-grid recall < 90% for xe máy, escalate. Consider 6m sub-grid as compromise, or accept that WF3 primary must run at 4m (pushing SLA to ~45 min).

**After Phase B**: If cross-tile coupling errors at tile boundaries cause recall < 85%, WF1 and WF2 may be unsuitable as standalone options. WF3 becomes the only viable primary.

**After Phase D**: Select primary production workflow based on F1 score, operational complexity, and SLA compliance. Expected outcome: WF3 as primary, WF1 as pilot/fallback, WF2 as research tool.

---

## 8. Recommended Architecture Decision

Based on the analysis in this document and the preceding discussions, the expected benchmark outcome is:

**WF3 (8m + 4m refinement) as the production system**, because:
1. Perfect 1D sewer coupling = highest threshold detection accuracy (the #1 metric for navigation)
2. Simplest pipeline = lowest operational risk during a crisis
3. Fewest API calls = lowest failure probability
4. No cross-tile problems = no boundary artefacts, no mass conservation errors
5. 4m refinement layer adds street-level detail where it matters most without risking SLA
6. Simplest 24h on-demand integration (1 API call)

**WF1 retained** as fallback if WF3's mega-model schematisation proves technically infeasible (3Di model size limits) or if 8m sub-grid accuracy proves insufficient even with refinement.

**WF2 retained** as validation and research tool for post-event analysis where maximum cross-tile fidelity is desired and runtime is unconstrained.

**Scenario library** serves as the instant-response foundation for all workflows and replaces the scheduled 24h background_cycle for extended forecast.

---

*End of Document — FloodNav Hanoi Benchmark Architecture v0.2*
