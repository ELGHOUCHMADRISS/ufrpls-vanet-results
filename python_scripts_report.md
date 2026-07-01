# Python Scripts Report
## Directory: `simulations/` and `simulations/_maps/pls/`
### VANET PLS Simulation — Complete Python Script Reference

---

## Table of Contents

1. [Overview & Script Inventory](#1-overview--script-inventory)
2. [Simulation Runner](#2-simulation-runner-run_all_simulations_9py)
3. [Primary Analysis Engine](#3-primary-analysis-engine-analyze_results_5pospy)
4. [Per-Repetition Analysis](#4-per-repetition-analysis-analyze_results_5pos_per_reppy)
5. [Per-Rep Per-Position Analysis](#5-per-rep-per-position-analysis-analyze_results_5pos_per_rep_per_pospy)
6. [Per-Vehicle CS Diagnostic](#6-per-vehicle-cs-diagnostic-plot_per_vehiclepy)
7. [Excel Export](#7-excel-export-analyze_to_excelpy)
8. [Comprehensive Metric Extractor](#8-comprehensive-metric-extractor-comprehensive_analysispy)
9. [Chai Comparison Figure](#9-chai-comparison-figure-gen_chai_comparison_figpy)
10. [Route File Inspector](#10-route-file-inspector-analyze_allpy-maps)
11. [80-Vehicle Fast Route Generator](#11-80-vehicle-fast-route-generator-gen_80veh_fast_fixpy)
12. [Alternate Route Generator](#12-alternate-route-generator-gen_80veh_reduced_gappy)
13. [Script Dependency Map](#13-script-dependency-map)

---

## 1. Overview & Script Inventory

The project contains **11 Python scripts** spread across two directories. They form a complete post-simulation analysis pipeline plus map utility scripts.

| Script | Location | Lines | Category |
|--------|----------|-------|----------|
| `run_all_simulations_9.py` | `simulations/` | 644 | Simulation runner |
| `analyze_results_5pos.py` | `simulations/` | 1004 | Primary analysis |
| `analyze_results_5pos_per_rep.py` | `simulations/` | 159 | Granularity variant |
| `analyze_results_5pos_per_rep_per_pos.py` | `simulations/` | 184 | Granularity variant |
| `plot_per_vehicle.py` | `simulations/` | 369 | Diagnostic |
| `analyze_to_excel.py` | `simulations/` | 276 | Export |
| `comprehensive_analysis.py` | `simulations/` | 643 | Comprehensive metrics |
| `gen_chai_comparison_fig.py` | `simulations/` | 67 | Paper figure |
| `analyze_all.py` | `_maps/pls/` | 64 | Map utility |
| `gen_80veh_fast_fix.py` | `_maps/pls/` | 147 | Route generator |
| `gen_80veh_reduced_gap.py` | `_maps/pls/` | 93 | Route generator |

### Analysis Pipeline Overview

```
OMNeT++ simulation runs
        │
        ▼
results/*.sca files
        │
        ├──→ run_all_simulations_9.py     (orchestrates runs)
        │
        ├──→ analyze_results_5pos.py       (primary figures & CSVs)
        │         └──→ plots_5pos/
        │
        ├──→ analyze_results_5pos_per_rep.py
        │         └──→ plots_5pos_by_rep/
        │
        ├──→ analyze_results_5pos_per_rep_per_pos.py
        │         └──→ plots_5pos_by_rep_pos/
        │
        ├──→ plot_per_vehicle.py
        │         └──→ per_vehicle_plots/
        │
        ├──→ analyze_to_excel.py
        │         └──→ sca_analysis.xlsx
        │
        ├──→ comprehensive_analysis.py
        │         └──→ comp_plots/
        │
        └──→ gen_chai_comparison_fig.py    (standalone, no .sca input)
                  └──→ plots_5pos/fig_chai_comparison_reproduced.png
```

---

## 2. Simulation Runner — `run_all_simulations_9.py`

**Location:** `simulations/run_all_simulations_9.py`  
**Lines:** 644

### Purpose

Batch orchestrator that drives the complete OMNeT++ experiment matrix. Iterates over all combinations of (protocol × vehicle density × alpha × speed × eavesdropper position × repetition), manages SUMO route file injection, and launches OMNeT++.

### Input Files

| File | Role |
|------|------|
| `simulations/vanet/ufrpls/omnetpp_5pos_full.ini` | OMNeT++ configuration with all named configs |
| `_maps/pls/traffic_{N}veh_{speed}_rou.xml` | Route files — modified per run to inject eavesdropper position |
| `_maps/pls/launchd_{N}veh_{speed}.xml` | Referenced by INI; used by SUMO bridge |
| OMNeT++ executable (`opp_run` or similar) | The simulator binary |

### Produced Output Files

- `results/UFRPLS-{N}veh-{speed}-a{alpha}-{pos}-rep{r}.sca` — scalar results per run
- `results/GPSRCS-{N}veh-{speed}-{pos}-rep{r}.sca` — GPSR baseline results
- Log files per run in `results/logs/`

### Execution Logic

```
For protocol in [UFRPLS, GPSRCS]:
  For N in [20, 40, 60, 80, 100]:
    For speed in [slow, fast]:
      For alpha in [0.1..0.9] (UFRPLS only; slow: all, fast: 0.3/0.5/0.7):
        For rep in [1..5]:
          For pos in [e100, e300, e500, e700, e900]:
            1. Patch route XML: replace vehicle00.departPos with pos value
            2. Run OMNeT++: opp_run -c {ConfigName} --seed-set={rep}
            3. Wait for completion; collect .sca output
```

**Key implementation details:**
- Eavesdropper position injection: regex substitutes `departPos="..."` in vehicle00's XML element
- Repetitions rep1–rep5 correspond to `seed-set=1..5` for reproducibility
- DEST_VEH mapping: `{20:19, 40:39, 60:59, 80:79, 100:99}` — last vehicle is always sink
- Alpha sweep: slow scenarios test all `α ∈ {0.1, 0.2, …, 0.9}`; fast tests subset `{0.3, 0.5, 0.7}` only
- sumo-launchd.py is started before each run on port 9999 (TraCI bridge)

### Project Utility

This script is the **entry point** for the entire experiment. Without it, the 1250+ individual simulation runs (5 densities × 2 speeds × 5 positions × 5 reps × 9 alpha + GPSRCS baseline) would need to be launched manually. It enforces reproducibility through fixed seeds and systematic file naming.

---

## 3. Primary Analysis Engine — `analyze_results_5pos.py`

**Location:** `simulations/analyze_results_5pos.py`  
**Lines:** 1004

### Purpose

The central analysis and figure-generation script. Parses all `.sca` result files, extracts PLS metrics, averages across 5 eavesdropper positions, and reproduces all Chai et al. figures as well as new diagnostic figures.

### Input Files

| File Pattern | Content |
|-------------|---------|
| `results/*.sca` | OMNeT++ scalar result files (Format A and B — see below) |

### .sca Format Handling

The script handles two formats produced by different OMNeT++ modules:

**Format A — Single-line scalar:**
```
scalar UFRPLSSim.vehicle[3].routing secrecyCapacity 2.45
```

**Format B — Statistic block:**
```
statistic UFRPLSSim.vehicle[3].routing secrecyCapacity:stats
field count   1000
field mean    2.45
field stddev  0.88
field min     0.00
field max     5.21
```

The `get_scalar()` and `get_stat_field()` functions use compiled regex patterns to locate either format for a given `(vehicle_id, metric_name)` pair.

**Position extraction from filename:** The eavesdropper position tag (e.g., `e500`) is extracted via regex `-(e\d+)[-_]` applied to the `.sca` filename. It is NOT present in the `configname` attribute inside the file.

### Averaging Logic

`average_over_positions()` groups loaded data by `(protocol, N, alpha, speed)` and for each group:
1. Collects one data point per eavesdropper position (e100…e900)
2. Computes the mean across the 5 positions
3. Records per-position extremes (min/max) for error bars

### Produced Output Files

All outputs go to `results/plots_5pos/`:

| File | Description |
|------|-------------|
| `fig3_alpha05.png` | CS vs. vehicle density at α=0.5 — primary Chai Fig.3 replica |
| `fig4_alpha_lt05.png` | CS vs. density for α ∈ {0.1, 0.3} |
| `fig5_alpha_gt05.png` | CS vs. density for α ∈ {0.7, 0.9} |
| `fig6_delay_all_alpha.png` | E2E delay vs. density for all alpha values |
| `fig7_comparison.png` | UFRPLS vs. GPSRCS CS head-to-head |
| `fig8_delay_comparison.png` | UFRPLS vs. GPSRCS delay comparison |
| `fig_speed_a0.3.png` | CS vs. density: slow vs. fast, α=0.3 |
| `fig_speed_a0.5.png` | CS vs. density: slow vs. fast, α=0.5 |
| `fig_speed_a0.7.png` | CS vs. density: slow vs. fast, α=0.7 |
| `fig13_gpsrcs_speed.png` | GPSRCS CS vs. density: slow vs. fast |
| `fig_pdr.png` | PDR vs. density for selected alpha |
| `fig_pdr_all_alpha.png` | PDR vs. density for all alpha values |
| `fig_paper_compact_4in1.png` | 2×2 summary panel figure |
| `fig_new_mincs_stats.png` | Minimum CS statistics |
| `fig_new_secrecy_stats.png` | Secrecy capacity distribution (mean±std) |
| `fig_new_sample_counts.png` | Sample count per configuration |
| `avg_5pos.csv` | Averaged metrics per (protocol, N, alpha, speed) |
| `raw_all.csv` | All raw scalar rows from all .sca files |

### Project Utility

This is the **single most important script** for paper-quality figure generation. It implements the full Chai et al. averaging methodology and produces all figures needed for result comparison. The `avg_5pos.csv` serves as ground truth for validation.

---

## 4. Per-Repetition Analysis — `analyze_results_5pos_per_rep.py`

**Location:** `simulations/analyze_results_5pos_per_rep.py`  
**Lines:** 159

### Purpose

Wrapper around `analyze_results_5pos.py`'s core functions. Separates data by **repetition number** before averaging, generating one complete figure set per repetition. Used to detect run-to-run variability and verify that individual seeds produce consistent results.

### Input Files

| File | Role |
|------|------|
| `results/*.sca` | Same as primary script; filtered by repetition number (rep1–rep5) |

### Produced Output Files

For each repetition `N` ∈ {1, 2, 3, 4, 5}:

```
results/plots_5pos_by_rep/rep{N}/
    fig3_alpha05.png
    fig4_alpha_lt05.png
    fig5_alpha_gt05.png
    fig6_delay_all_alpha.png
    fig7_comparison.png
    fig8_delay_comparison.png
    fig_speed_a0.3.png / a0.5.png / a0.7.png
    fig13_gpsrcs_speed.png
    fig_pdr.png
    fig_pdr_all_alpha.png
    fig_paper_compact_4in1.png
    fig_new_mincs_stats.png
    fig_new_secrecy_stats.png
    fig_new_sample_counts.png
    avg_5pos_rep.csv
    raw_rep.csv
```

Total: ~15 PNG files + 2 CSVs × 5 reps = **75 PNG files + 10 CSVs**.

### Project Utility

Validates that no single repetition is an outlier. If one rep produces dramatically different figures, it signals a simulation anomaly (e.g., SUMO crash mid-run producing a partial .sca file). Comparing rep2–rep5 figures also verifies that the random eavesdropper positions in each rep produce similar ensemble statistics.

---

## 5. Per-Rep Per-Position Analysis — `analyze_results_5pos_per_rep_per_pos.py`

**Location:** `simulations/analyze_results_5pos_per_rep_per_pos.py`  
**Lines:** 184

### Purpose

The finest-grained analysis tool. Produces a complete figure set for each **(repetition × eavesdropper position)** combination — 25 slices in total (5 reps × 5 positions). No averaging across positions or repetitions. Used to examine individual simulation instance behaviour.

### Input Files

Same pattern as primary script: `results/*.sca`, filtered by `(rep, position)` pair extracted from the filename.

### Key Difference from Primary Script

Disables the `_add_pos_extremes` monkey-patch since each data slice contains only one eavesdropper position — there is no min/max envelope to compute across positions.

### Produced Output Files

For each `(repN, posM)` combination:

```
results/plots_5pos_by_rep_pos/rep{N}/{pos}/
    fig3_alpha05_rep{N}_{pos}.png
    ... (same 14 figure files, with _rep{N}_{pos} suffix)
    avg_rep{N}_{pos}.csv
    raw_rep{N}_{pos}.csv
```

Total: ~14 PNG files + 2 CSVs × 25 slices = **350 PNG files + 50 CSVs**.

### Project Utility

Primary use: debugging anomalies in specific (rep, position) pairs. If `avg_5pos.csv` shows an unexpected dip at e700, this script lets you inspect which exact rep and position is the outlier. Also used to verify that the eavesdropper position scanning produces a monotonic or expected CS profile along the highway.

---

## 6. Per-Vehicle CS Diagnostic — `plot_per_vehicle.py`

**Location:** `simulations/plot_per_vehicle.py`  
**Lines:** 369

### Purpose

Generates a bar chart of **Secrecy Capacity per vehicle** for each `.sca` file, showing mean ± standard deviation with min (▼) and max (▲) markers. Identifies which vehicle IDs produce the highest/lowest CS values and detects anomalous vehicles.

### Input Files

| File | Role |
|------|------|
| `results/*.sca` | Processes each file independently |

### Vehicle Role Color Coding

| Vehicle ID | Role | Bar Color |
|------------|------|-----------|
| vehicle[0] | Eavesdropper | Red |
| vehicle[1] | Source | Green |
| vehicle[2..N-2] | Relay | Blue |
| vehicle[N-1] | Destination | Purple |

### Produced Output Files

```
results/per_vehicle_plots/
    {stem}_per_vehicle.png     (one bar chart per .sca file)
    per_vehicle_summary.csv    (mean/std/min/max per vehicle per file)
```

Where `{stem}` is the `.sca` filename without extension.

### What the Plot Shows

Each bar represents one vehicle's CS distribution across all hops it participated in during the simulation. Bars include:
- Bar height: mean CS (bits/s/Hz)
- Error bar: ±1 standard deviation
- ▲ marker: maximum CS observed for that vehicle
- ▼ marker: minimum CS (≥0 by definition)

This enables immediate visual detection of the eavesdropper's geometric impact: vehicles near the eavesdropper's position will show lower CS bars, while vehicles far away show higher CS.

### Project Utility

Diagnostic tool for understanding **spatial CS distribution** along the highway. Particularly useful when the averaged results show unexpectedly low overall CS — this plot reveals whether the problem is concentrated in a few relay positions or uniformly distributed. The `per_vehicle_summary.csv` enables quantitative follow-up.

---

## 7. Excel Export — `analyze_to_excel.py`

**Location:** `simulations/analyze_to_excel.py`  
**Lines:** 276

### Purpose

Exports all scalar data from `.sca` files to a formatted multi-sheet Excel workbook using `openpyxl`. Designed for manual inspection and sharing with collaborators who do not have Python analysis tools.

### Input Files

| File | Role |
|------|------|
| `results/*.sca` | All scalar result files |

### Produced Output Files

`results/sca_analysis.xlsx` with four worksheets:

| Sheet | Content | Format |
|-------|---------|--------|
| `summary` | Key per-configuration metrics: CS mean, PDR, E2E delay, hop count | One row per (config, N, alpha, speed, pos, rep) |
| `all_vehicles` | Pivoted per-vehicle scalars | Rows: configs; columns: vehicle[0]..vehicle[N-1] CS values |
| `avg_by_position` | Chai et al. metrics averaged across reps per (config, N, alpha, speed, pos) | Matches averaging in analyze_results_5pos.py |
| `raw_scalars` | Every individual scalar line extracted from all .sca files | Full flat table |

**Formatting:** Blue header row, auto-filter enabled on all sheets, freeze panes at row 2. Column widths auto-sized to content.

### Project Utility

Primary use for **quick validation** and **data sharing**. The Excel format makes it easy to apply custom filters (e.g., "all UFRPLS α=0.5 runs with N=60"), create pivot tables, or share specific metric subsets with non-Python collaborators. The `raw_scalars` sheet serves as a human-readable audit trail of the raw simulator output.

---

## 8. Comprehensive Metric Extractor — `comprehensive_analysis.py`

**Location:** `simulations/comprehensive_analysis.py`  
**Lines:** 643

### Purpose

The most complete parser in the project. Extracts **24+ metrics** from `.sca` files in a single-pass index (`_build_index()`), covering not just PLS metrics but also MAC-layer events, IP routing decisions, and per-hop counts. Generates a multi-panel dashboard.

### Input Files

| File | Role |
|------|------|
| `results/*.sca` | All scalar result files — single-pass indexed |

### Metric Extraction

`_build_index()` parses each `.sca` file once, building a dictionary keyed by `(module_path, metric_name)`. This avoids the N² regex scanning of the simpler scripts. Extracted metrics include:

**PLS Metrics:**
- `secrecyCapacity` (mean, stddev, min, max, count)
- `minSecrecyCapacity` (worst-case CS per packet)
- `networkSecureRatio` — fraction of hops with CS > 0
- `csSignalCount` — number of CS computations emitted

**Routing Metrics:**
- `packetDeliveryRatio` (PDR)
- `endToEndDelay` (ms, mean)
- `hopCount` (average hops per delivered packet)
- `noRouteDrops` — packets dropped due to no forwarding candidate
- `hopLimitDrops` — IP TTL-exceeded drops

**MAC Layer Metrics:**
- `mac:linkBroken` — frames dropped after max retry exceeded
- `mac:retryLimit` — retransmission attempts at MAC
- `mac:droppedPackets` — total MAC drops

**Protocol-Specific:**
- UFRPLS: `waitTimeExceeded`, `secureHopCount`
- GPSRCS: `perimeterRouteActivations`, `voidNodeCount`

### Produced Output Files

All outputs go to `results/comp_plots/`:

| File | Content |
|------|---------|
| `comprehensive_raw.csv` | All 24+ metrics, one row per (file, config) |
| `comprehensive_avg.csv` | Metrics averaged by (protocol, N, alpha, speed) |
| `comp_pdr.png` | PDR vs. vehicle density |
| `comp_delay.png` | E2E delay vs. vehicle density |
| `comp_hop_count.png` | Average hop count vs. density |
| `comp_secure_ratio.png` | Network secure ratio vs. density |
| `comp_cs_mean.png` | Mean secrecy capacity comparison |
| `comp_mcs_mean.png` | Mean minimum CS comparison |
| `comp_mac_drops.png` | MAC-layer drop rate vs. density |
| `comp_drop_breakdown.png` | Stacked drop sources (MAC/IP/routing) |
| `comp_dashboard_slow.png` | 8-panel dashboard for slow speed |
| `comp_dashboard_fast.png` | 8-panel dashboard for fast speed |

### Project Utility

Used for **root cause analysis** when aggregate PLS metrics look unexpected. For example, if PDR is low despite positive CS, the `comp_mac_drops.png` and `comp_hop_count.png` plots reveal whether the bottleneck is MAC-layer congestion (too many retransmissions) or UFRPLS wait timeouts (too few secure hops found). The dashboard PNGs are suitable for inclusion in technical reports.

---

## 9. Chai Comparison Figure — `gen_chai_comparison_fig.py`

**Location:** `simulations/gen_chai_comparison_fig.py`  
**Lines:** 67

### Purpose

Generates a side-by-side comparison figure showing Chai et al.'s published results (Monte Carlo, MATLAB) alongside our OMNeT++/SUMO replication results. Uses **hardcoded data arrays** — no `.sca` file parsing.

### Input Files

None — all data is hardcoded:

```python
# Chai et al. (Electronics 2023, Monte Carlo/MATLAB)
n_values = [20, 40, 60, 80, 100]        # vehicle density
chai_gpsr = [2.25, 2.15, 2.10, 2.05, 2.00]
chai_a03  = [3.00, 3.48, 3.65, 3.85, 3.78]
chai_a05  = [3.70, 4.50, 4.70, 4.95, 5.05]
chai_a07  = [4.20, 5.85, 6.85, 7.50, 7.92]

# This work (OMNeT++/SUMO, read from avg_5pos.csv)
ours_gpsrcs = [1.21, 0.86, 0.68, 0.87, 0.80]
ours_a03    = [1.37, 1.25, 1.27, 1.24, 1.27]
ours_a05    = [2.58, 2.53, 2.21, 2.57, 2.57]
ours_a07    = [6.84, 5.01, 4.72, 5.50, 5.93]
```

### Produced Output Files

`results/plots_5pos/fig_chai_comparison_reproduced.png`

A 1×2 side-by-side figure:
- **Left panel:** Chai et al. original Monte Carlo results from the paper
- **Right panel:** This work's OMNeT++ replication results

Both panels plot Secrecy Capacity (bits/s/Hz) vs. Vehicle Density (N) for GPSRCS baseline and UFRPLS at α={0.3, 0.5, 0.7}.

### Project Utility

Key figure for the **paper/thesis**: directly demonstrates agreement (or divergence) between the Monte Carlo theoretical model and the OMNeT++ discrete-event simulation. Hardcoded values match the manually extracted data from `avg_5pos.csv` at a specific analysis snapshot. **Note:** if new simulation runs change the averaged results, the `ours_*` arrays must be updated manually.

---

## 10. Route File Inspector — `analyze_all.py` (_maps/)

**Location:** `simulations/_maps/pls/analyze_all.py`  
**Lines:** 64

### Purpose

Utility script run by the developer after creating or modifying any route file. Validates that all route files meet the simulation requirements.

### Input Files

| File | Role |
|------|------|
| `_maps/pls/traffic_*_rou.xml` | All route XML files in the same directory (glob) |

### Checks Performed

1. **Vehicle count** — reports total vehicles and sigma/maxSpeed per vType
2. **Eavesdropper gap** — finds vehicle00's position; locates nearest eastbound and westbound relay before and after it; reports gap boundaries
3. **Relay spacing** — computes average and maximum inter-vehicle distance among relays
4. **Braking safety check:**
   ```python
   braking_dist = speed**2 / (2 * 4.5)   # v²/2a, decel=4.5 m/s²
   if depart_pos + braking_dist > 1000:
       print(f"UNSAFE: {vehicle_id} at pos={pos}, speed={speed}")
   ```
   If a vehicle cannot stop before the x=1000 junction at its departure speed, SUMO will generate collision errors.

### Produced Output Files

Console text only. No files written. Example output:
```
=== traffic_80veh_80-120kmh_rou.xml ===
Vehicles: 80, maxSpeed=33.33 m/s, sigma=0.5
Eavesdropper: vehicle00 at x=500.0m, lane=1
  Nearest relay before EVE: vehicle32 at x=470.0m (gap: 30.0m)
  Nearest relay after  EVE: vehicle33 at x=510.0m (gap: 10.0m)
  Average relay spacing: 12.4m
  Max relay spacing: 40.0m
BRAKING SAFETY: OK (all vehicles within safe bounds)
```

### Project Utility

**Pre-simulation validation tool.** Running this after any manual route edit or after re-running a generator script catches two common errors: eavesdropper gap too small (SUMO collision) and boundary speed too high (SUMO junction error). Should be run before any batch simulation launch.

---

## 11. 80-Vehicle Fast Route Generator — `gen_80veh_fast_fix.py`

**Location:** `simulations/_maps/pls/gen_80veh_fast_fix.py`  
**Lines:** 147

### Purpose

Programmatically generates `traffic_80veh_80-120kmh_rou.xml` with a precisely designed eavesdropper gap and boundary speed safety caps. Replaces manual XML editing for the most complex route file.

### Input Files

None. All vehicle positions are hardcoded in the script.

### Generation Logic

```python
OUTFILE = "traffic_80veh_80-120kmh_rou.xml"
EVE_POS = 500.0    # eavesdropper midpoint
GAP_START = 470.0  # relay exclusion zone start
GAP_END   = 510.0  # relay exclusion zone end

# Eastbound relays: 17 before gap, 21 after
east_before = np.linspace(113.0, GAP_START, 17)   # 113m to 470m
east_after  = np.linspace(GAP_END, 935.0, 21)      # 510m to 935m

# Speed caps at boundary
if pos >= 915.0: speed = 22.22   # cannot stop before x=1000 at higher speed
if pos >= 880.0: speed = min(speed, 25.0)
if pos >= 855.0: speed = min(speed, 27.78)
```

Lane assignment: `lane = i % 4` cycles through lanes 0, 1, 2, 3 for successive vehicles.

### Produced Output Files

`simulations/_maps/pls/traffic_80veh_80-120kmh_rou.xml`

### Project Utility

Fills the gap in the automatically-generable route files. The 80-vehicle fast scenario requires a specially tuned eavesdropper gap (~30m instead of the default ~60m) because higher vehicle density with fast speeds creates more collision risk. The script encodes this domain knowledge explicitly and is the authoritative source for that route file.

---

## 12. Alternate Route Generator — `gen_80veh_reduced_gap.py`

**Location:** `simulations/_maps/pls/gen_80veh_reduced_gap.py`  
**Lines:** 93

### Purpose

Alternative version of `gen_80veh_fast_fix.py` with a different speed pool generation strategy. Both scripts produce the same output filename and same gap boundaries. This script represents an earlier iteration on the route design.

### Input Files

None. All positions hardcoded.

### Key Differences from gen_80veh_fast_fix.py

| Aspect | gen_80veh_fast_fix.py | gen_80veh_reduced_gap.py |
|--------|----------------------|--------------------------|
| Speed pool | Deterministic boundary caps | Random pool from [22.22, 27.78, 33.33] |
| Code structure | Cleaner, explicit | More compact |
| Eastbound count | 17+21=38 | Slightly different distribution |
| Gap | 470–510m (30m) | 470–510m (30m) — same |

### Produced Output Files

`simulations/_maps/pls/traffic_80veh_80-120kmh_rou.xml` (same as gen_80veh_fast_fix.py)

**Warning:** Running both scripts in sequence means the second run overwrites the first. `gen_80veh_fast_fix.py` should be considered the canonical version.

### Project Utility

Preserved as a development artifact. If the canonical generator produces unexpected SUMO errors, this alternate version can be tested to isolate whether the speed pool logic is the cause.

---

## 13. Script Dependency Map

```
                         ┌─────────────────────────────────────────┐
                         │        Route Preparation Layer           │
                         │  gen_80veh_fast_fix.py                  │
                         │  gen_80veh_reduced_gap.py               │
                         │       ↓ writes                          │
                         │  traffic_80veh_80-120kmh_rou.xml        │
                         │       ↓ validated by                    │
                         │  analyze_all.py                         │
                         └──────────────────┬──────────────────────┘
                                            │
                                            ▼
                         ┌─────────────────────────────────────────┐
                         │       Simulation Execution Layer         │
                         │   run_all_simulations_9.py              │
                         │   (injects eavesdropper positions,      │
                         │    launches OMNeT++, collects .sca)     │
                         └──────────────────┬──────────────────────┘
                                            │
                                 results/*.sca files
                                            │
               ┌────────────────────────────┼─────────────────────────────┐
               │                            │                             │
               ▼                            ▼                             ▼
  ┌────────────────────────┐  ┌─────────────────────────┐  ┌─────────────────────┐
  │  Averaging Analysis    │  │   Diagnostic Analysis   │  │   Export Tools      │
  │                        │  │                         │  │                     │
  │ analyze_results_5pos   │  │ plot_per_vehicle.py     │  │ analyze_to_excel.py │
  │ _per_rep.py            │  │ comprehensive_analysis  │  │                     │
  │ _per_rep_per_pos.py    │  │ .py                     │  │                     │
  │                        │  │                         │  │                     │
  │ → plots_5pos*/         │  │ → per_vehicle_plots/    │  │ → sca_analysis.xlsx │
  │ → avg_5pos*.csv        │  │ → comp_plots/           │  │                     │
  └────────────────────────┘  └─────────────────────────┘  └─────────────────────┘
               │
               ▼
  ┌─────────────────────┐
  │  Paper Figures      │
  │                     │
  │ gen_chai_comparison │
  │ _fig.py             │
  │ (hardcoded data)    │
  │                     │
  │ → fig_chai_*.png    │
  └─────────────────────┘
```

### Common Library Dependencies

| Library | Used By | Purpose |
|---------|---------|---------|
| `re` | all analysis scripts | .sca regex parsing |
| `matplotlib` | all analysis + generators | figure output |
| `numpy` | all analysis + generators | numerical computation |
| `pandas` | analyze_results_5pos, comprehensive | CSV I/O |
| `openpyxl` | analyze_to_excel | Excel workbook |
| `glob` | all analysis, analyze_all | file discovery |
| `xml.etree.ElementTree` | gen_* scripts | route XML generation |
| `subprocess` | run_all_simulations_9 | OMNeT++ execution |

---

*Report generated from 11 Python scripts: 8 in `simulations/` and 3 in `simulations/_maps/pls/`.*
