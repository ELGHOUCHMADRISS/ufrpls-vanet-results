# UFRPLS VANET Results Dataset

This repository contains the processed results dataset used in our paper on
physical-layer-security-aware geographic routing in VANETs.

**Paper:** Critical Re-Engineering and Experimental Validation of Physical Layer
Security Aware Geographic Routing in VANETs Using OMNeT++, INET, Veins, and SUMO

## Contents

- `plots_5pos/` — position-averaged main figures and CSV summaries
  - `avg_5pos.csv`, `raw_all.csv` — aggregated scalar data
  - `fig_main_compact_4in1.png` — paper Fig. 1 (4-in-1 summary)
  - `fig_chai_comparison_reproduced.png` — comparison with Chai et al.
  - Additional figures: alpha sensitivity, delay, PDR, speed groups
- `comp_plots/` — cross-protocol comparison dashboards and drop breakdown
  - `comprehensive_avg.csv`, `comprehensive_raw.csv`
  - Dashboard figures: CS mean, delay, hop count, PDR, drop breakdown, secure ratio
- `per_vehicle_plots/` — per-vehicle secrecy and forwarding behavior plots
  - `per_vehicle_summary.csv`
  - Per-vehicle PNG figures for GPSRCS and UFRPLS across all densities and eavesdropper positions
- `plots_5pos_by_rep/` — repetition-wise outputs (rep1–rep5) for stability analysis
- `plots_5pos_by_rep_pos/` — repetition-wise outputs grouped by eavesdropper position
- `run_manifest.csv` — run accounting (status, repetition, seed, config, scalar file)
- `run_summary.txt` — human-readable run summary
- `sca_analysis.xlsx` — consolidated scalar analysis workbook

## Protocols

- **GPSRCS** — GPSR baseline with post-hoc secrecy capacity measurement; security
  is observed but does not influence forwarding.
- **UFRPLS** (Utility Function Routing for Physical Layer Security) — utility-based
  forwarding that selects the next hop by maximizing
  `ε_ij = CS_ij^α × d_ij^β`, with a waiting/drop mechanism when no secure
  neighbor is available.

## Notes

This repository shares processed result artifacts (figures, CSV summaries, and
scalar analysis). It does not include the full simulation source code.
Folder and file names are aligned with the paper for reproducibility tracing.
All scalar metrics were extracted from `.sca` files produced by OMNeT++ 5.6.2
using a Python post-processing script.

## Citation

If you use this dataset, please cite our paper and this repository.

> El-Ghouchma, D., Benmenssour, M., Zytoune, O. (2026).
> *UFRPLS VANET Results Dataset*. GitHub repository.
> https://github.com/ELGHOUCHMADRISS/ufrpls-vanet-results

## License

Copyright (C) 2026 Driss El-Ghouchma, Mourad Benmenssour, Ouadoudi Zytoune

The processed dataset, figures, and CSV summaries in this repository are released
under Creative Commons Attribution 4.0 International (CC BY 4.0).
See `DATASET_LICENSE.txt` for details.
https://creativecommons.org/licenses/by/4.0/

Simulation source code (if published) is licensed under GNU General Public License v3.0.
See the [LICENSE](LICENSE) file for details.
https://www.gnu.org/licenses/gpl-3.0.html
