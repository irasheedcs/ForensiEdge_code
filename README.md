# ForensiEdge

**Privacy-Preserving Federated Split Learning for Network Forensics in Edge-Intelligent 6G-IoT**

Reference implementation of *ForensiEdge*: a federated split-learning framework that
performs on-device network-forensics evidence encoding under Rényi differential
privacy (RDP), reconstructs multi-stage attack timelines with a Causal Forensic
Graph Network (CFGN), and schedules device participation with a Lyapunov
drift-plus-penalty controller.

> This repository is self-contained: it ships a synthetic multi-stage attack
> generator so every script runs out of the box, plus drop-in loaders for
> CIC-IoT-2023 and UNSW-NB15. No `torch-geometric` dependency — the graph
> attention network is implemented natively.

## Installation

```bash
git clone https://github.com/<your-org>/ForensiEdge.git
cd ForensiEdge
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

Python ≥ 3.9 and PyTorch ≥ 2.0 (CPU is fine for the synthetic demo).

## Quick start

```bash
# Train + evaluate on the built-in synthetic benchmark
python scripts/train.py --rounds 200 --num-devices 100 --seed 0

# Sweep the privacy level
python scripts/train.py --target-epsilon 0.5

# Ablations over 5 seeds with 95% confidence intervals (Table I)
python scripts/run_ablations.py --seeds 5 --rounds 200

# Regenerate the three figures
python scripts/make_figures.py

# Run the privacy-accounting unit tests
python tests/test_rdp.py
```

## Method ↔ code map

| Paper component | Location |
|---|---|
| Forensic Evidence Encoder (FEE) | `forensiedge/models/fee.py` |
| Causal Forensic Graph Network (CFGN, native GAT) | `forensiedge/models/cfgn.py` |
| RDP accounting: per-round rate, adaptive allocation, Theorem 1 conversion | `forensiedge/privacy/rdp.py` |
| Gaussian mechanism on released smashed data (Eq. 7, 10) | `forensiedge/federated/fsap.py` |
| Lyapunov participation / frequency control (Eqs. 14–17) | `forensiedge/optim/lyapunov.py` |
| Forensic Split Aggregation Protocol (Algorithm 1) | `forensiedge/federated/fsap.py` |
| Non-IID Dirichlet partitioning | `forensiedge/data/partition.py` |
| Detection + forensic metrics (F1, evidence recall, edge recall, SHD) | `forensiedge/metrics/forensic.py` |

## Privacy accounting

The RDP accountant enforces a **budget-consistent** allocation: with per-round
weights `β_t` summing to 1 and `σ_t² = Δ_s² / (2 ρ_max β_t)`, the per-round budgets
compose to `ρ_max` **exactly** (`sum_t ρ_t = ρ_max`). Conversion to `(ε, δ)`-DP uses
the closed form

```
ε* = ρ_max + 2·sqrt(ρ_max · ln(1/δ)),   α* = 1 + sqrt(ln(1/δ)/ρ_max).
```

`tests/test_rdp.py` verifies budget composition, the closed-form optimum, and that
adaptive allocation minimises the privacy penalty (Cauchy–Schwarz).

## Using the real datasets

```bash
# 1. Download CIC-IoT-2023 (or UNSW-NB15) CSVs.
# 2. Convert to the .npz format the loader expects (edit the column / attack-stage
#    maps at the top of the script for your local copy):
python scripts/prepare_data.py --csv path/to/CICIoT2023.csv \
    --name cic-iot-2023 --out ./data_files --seq-len 32
# 3. Train:
python scripts/train.py --dataset cic-iot-2023 --data-root ./data_files
```

The `.npz` must contain `x (M,1,L)`, `y (M,)`, `stage (M,)`, `tau (M,)`, and
optionally `incident (M,)` (kill-chain instance id used to build sparse causal
ground-truth graphs).

## Notes on reproducibility

* The synthetic generator is a **smoke test**: it exercises every code path and
  reproduces the qualitative story (near-centralized detection, exact privacy
  accounting, forensic metrics that degrade as ε shrinks). Absolute numbers on
  the synthetic data are **not** the paper's headline figures — run on
  CIC-IoT-2023 / UNSW-NB15 for those.
* Detection uses on-device classification heads (no data leaves the device), so
  detection F1 stays close to centralized across privacy levels; the DP noise is
  applied to the *released* smashed embeddings and its cost therefore lands on
  forensic reconstruction, matching the paper's design.
* All hyperparameters default to the values in the paper (see
  `forensiedge/config.py`); override any of them from the CLI.

## Citation

```bibtex
@article{rasheed2026forensiedge,
  title   = {ForensiEdge: Privacy-Preserving Federated Split Learning for
             Network Forensics in Edge-Intelligent 6G-IoT},
  author  = {Rasheed, Iftikhar and Mostafa, Hala and Alahmari, Saad and
             AlTamimi, Saad Nasser},
  journal = {IEEE Communications Letters},
  year    = {2026},
  note    = {Under review}
}
```

## License

MIT — see [LICENSE](LICENSE).
