# PKC Benchmarking on the MIT Supercloud Dataset

## 1. Overview

This folder benchmarks four public-key cryptography (PKC) algorithms — **RSA-2048**, **ECC (NIST P-256)**, **DSA-2048**, and **Diffie-Hellman (DH-2048)** — on real HPC job-scheduler records from the **MIT Supercloud Dataset**. As with the other two datasets in this project, the objective is experimental benchmarking, not theoretical cryptanalysis: the security of these algorithms is already well established, so the question this notebook answers is which algorithm actually performs best on this kind of real-world data, and why.

| Aspect | Algorithms tested | Why these algorithms |
|---|---|---|
| **Encryption** | RSA (RSA-OAEP hybrid), ECC (ECIES-style hybrid) | Only these two can encrypt bulk data — each wraps a fresh AES-256 key and lets AES-256-GCM handle the actual record, the standard "hybrid encryption" model used in real systems. |
| **Digital Signature** | RSA (RSA-PSS), ECC (ECDSA), DSA | All three support signing; DSA can *only* sign. |
| **Key Exchange** | ECC (raw ECDH), DH (raw Diffie-Hellman) | Only these two establish a shared secret between two parties. |

## 2. Dataset

**Name:** MIT Supercloud Dataset (`scheduler_data.csv`)
**What it is:** Real Slurm job-scheduler logs from the MIT Supercloud high-performance computing / datacenter cluster — one row represents one submitted compute job, with fields describing job resource requests/allocations, timing, and GPU usage.
**Source:** The MIT Supercloud Dataset (Datacenter Challenge scheduler logs) is publicly available on Kaggle: https://www.kaggle.com/datasets/skylarkphantom/mit-datacenter-challenge-data
**File used:** `scheduler_data.csv` (provided in this folder as `scheduler_data.csv.zip`)
**Preprocessing applied:**
- Raw file has 31 columns; the `gres_used` column is 100% empty and was dropped.
- `gres_req`, `gres_alloc`, and `tres_alloc` (GPU-resource fields) are populated only for GPU jobs — the ~24.55% of rows without GPU requests had these fields filled with `'none'`.
- After preprocessing: **287,173 rows × 30 columns**, with **zero missing values**.

**Dataset-specific insight (GPU vs. non-GPU jobs):** GPU jobs make up 216,676 rows (75.5%) and non-GPU jobs make up 70,497 rows (24.5%). GPU jobs carry populated resource-allocation strings and are, on average, **~2.5× larger when serialized** (≈1,616 bytes) than non-GPU jobs (≈651 bytes). This directly affects per-record encryption/signing cost, since larger records cost more to protect — it is reported here as a real, measurable finding alongside the standard PKC metrics (see `PKC_Result/DatasetInsights/gpu_vs_nongpu_breakdown.png`).

**Record size (real data, all jobs):** mean ≈ 1,270 bytes, median ≈ 725 bytes, min ≈ 652 bytes, max ≈ 9,343 bytes — a much wider spread than the other two datasets, driven by the GPU/non-GPU split above.

## 3. What Was Implemented

The notebook (`MITSupercloud_Evaluation.ipynb`) implements the same measurement pipeline used across all three datasets in this project, computing per algorithm/aspect: **KGT, ET, DT, CSO (bytes and %), MC, CPU%, EC (estimate), TP, SC, Speedup, standardized Security Level, and a weighted Composite Score (0–100)** used to rank algorithms within each aspect. Correctness (encrypt→decrypt, sign→verify, matching shared secrets) is verified before any timing is trusted.

This notebook additionally includes a **General Evaluation Metrics** layer not present in the other two dataset notebooks: full distribution statistics (mean/median/min/max/std/CV%/95% CI) for KGT and ET, plus **pairwise Welch's t-tests** between every pair of algorithms in each aspect, confirming that the performance differences reported below are statistically significant and not just measurement noise (see `*_general_eval_metrics.csv` and `*_significance_tests.csv` in each `Comparison/` folder).

## 4. Results — What Was Achieved

### 4.1 Encryption (RSA vs ECC)

| Metric | RSA-2048 | ECC (P-256) |
|---|---|---|
| Key Generation Time | 1165.55 ms | 0.44 ms |
| Encryption Time | 4.07 ms | 6.48 ms |
| Decryption Time | 4.96 ms | 4.33 ms |
| Ciphertext Overhead | 40.75% | 13.34% |
| Energy (KeyGen) | 17,269.6 mJ | 6.8 mJ |
| Throughput @ 500 MB | 548.92 MB/s | 541.68 MB/s |
| Security Level | 112 bits | 128 bits |
| **Composite Score** | **20.0** | **80.0** |

**Winner: ECC.** The pattern is the same as on the other two datasets: RSA's key generation (~1.17 s) is roughly **2,650× slower** than ECC's (~0.44 ms), and this dominates the outcome. One interesting dataset-specific detail: at the largest data size (500 MB), RSA and ECC's throughput **converge to nearly the same value** (548.9 vs 541.7 MB/s). This is expected — once the payload is large, the actual bulk encryption work is done almost entirely by AES-256-GCM in *both* hybrid schemes, so the asymmetric algorithm's own speed stops mattering at scale; the real, lasting difference between RSA and ECC on this dataset is in key generation and per-record overhead, not bulk throughput. Statistical testing confirms both the KGT and ET differences between RSA and ECC are significant (p < 0.002 and p < 1×10⁻⁴⁵ respectively — see §4.4).

Also notable: RSA's ciphertext overhead here (40.75%) is much higher than on RT-IoT2022 (13.24%). This is a direct consequence of this dataset's smaller average record size (1,270 bytes vs 2,344 bytes) — RSA's wrapped-key overhead is a roughly fixed ~256 bytes regardless of message size, so it eats up a larger *percentage* of a smaller record. See §5 for the full explanation of this effect across all three datasets.

### 4.2 Digital Signature (RSA vs ECC vs DSA)

| Metric | RSA (RSA-PSS) | ECC (ECDSA) | DSA |
|---|---|---|---|
| Key Generation Time | 1355.64 ms | 0.43 ms | **8791.73 ms** |
| Signing Time | 9.96 ms | 4.40 ms | 3.07 ms |
| Verification Time | 6.13 ms | 7.61 ms | 2.79 ms |
| Signature Overhead | 36.73% | 9.18% | 8.04% |
| Energy (KeyGen) | 20,172.1 mJ | 6.7 mJ | 130,753.4 mJ |
| Security Level | 112 bits | 128 bits | 112 bits |
| **Composite Score** | **31.6** | **95.5** | **35.0** |

**Winner: ECC (ECDSA).** DSA's fresh domain-parameter generation again makes its key generation the single most expensive operation in the whole benchmark (~8.79 seconds — over 20,000× slower than ECC). ECC wins the aspect overall because it combines near-zero key generation cost with the smallest signature overhead of the three (9.18%, versus RSA's 36.73%). Pairwise significance testing confirms every difference here — RSA vs ECC, RSA vs DSA, and ECC vs DSA — is statistically significant for both KGT and ET (all p < 0.05; see §4.4).

### 4.3 Key Exchange (ECC/ECDH vs DH)

| Metric | ECC (ECDH) | DH |
|---|---|---|
| Key Generation Time | 0.36 ms | 2.61 ms |
| Initiator Derivation Time | 6.24 ms | 3.22 ms |
| Responder Derivation Time | 4.37 ms | **380.31 ms** |
| Public-Value Overhead | 103.1% | 700.0% |
| Security Level | 128 bits | 112 bits |
| **Composite Score** | **65.0** | **35.0** |

**Winner: ECC (ECDH).** Consistent with the other two datasets: DH's shared-secret derivation relies on modular exponentiation over a 2048-bit prime, making it roughly **87× slower** than ECC's equivalent elliptic-curve point multiplication (380.31 ms vs 4.37 ms), and its transmitted public value carries a **700% overhead** relative to the 32-byte session key it ultimately produces.

### 4.4 Statistical Significance (this dataset only)

Because this notebook includes pairwise Welch's t-tests, we can say with confidence — not just observation — that these results are real effects, not noise:

| Aspect | Metric | Comparison | p-value | Significant? |
|---|---|---|---|---|
| Encryption | KGT | RSA vs ECC | 0.0018 | Yes |
| Encryption | ET | RSA vs ECC | 1.9 × 10⁻⁴⁶ | Yes |
| Digital Signature | KGT | RSA vs ECC | 1.0 × 10⁻⁴ | Yes |
| Digital Signature | KGT | RSA vs DSA | 0.041 | Yes |
| Digital Signature | KGT | ECC vs DSA | 0.020 | Yes |
| Digital Signature | ET | RSA vs ECC | 1.1 × 10⁻¹¹⁷ | Yes |
| Digital Signature | ET | RSA vs DSA | 6.2 × 10⁻¹²⁴ | Yes |
| Digital Signature | ET | ECC vs DSA | 5.8 × 10⁻³⁵ | Yes |

Every comparison tested is significant at α = 0.05, most by an enormous margin — the performance gaps reported above are not measurement artifacts.

## 5. Overall Conclusion for This Dataset

**ECC won all three aspects on the MIT Supercloud dataset** (Encryption: 80.0, Digital Signature: 95.5, Key Exchange: 65.0), for the same structural reason seen throughout this project: a 256-bit elliptic curve reaches the same or better security as a 2,048-bit RSA/DSA/DH key while requiring dramatically less arithmetic, which translates directly into faster key generation, lower energy use, and lower CPU load. This dataset in particular highlights **how record size shapes overhead**: because MIT Supercloud's average job record (1,270 bytes) is smaller than RT-IoT2022's (2,344 bytes), RSA's fixed ~256-byte key-wrap overhead consumes a noticeably larger *percentage* of each record here (40.75% vs 13.24%) — a pattern that becomes even more extreme on the NIH Chest X-rays dataset, where records are smaller still (see that dataset's README, §5). ECC's much smaller ephemeral public key (~65 bytes) is far less sensitive to this effect, which is part of why it holds its overhead advantage across all three datasets regardless of how large or small the underlying records are.

## 6. Strengths & Weaknesses Summary

| Algorithm | Strengths | Weaknesses |
|---|---|---|
| **RSA-2048** | Fast per-record encrypt/verify once a key exists; mature, widely supported | Slow, energy-hungry key generation; overhead becomes very large (>40%) on datasets with small records |
| **ECC (P-256)** | Near-instant key generation; smallest keys for the security level; lowest energy use; overhead stays low regardless of record size | Slightly slower raw encryption than RSA per record |
| **DSA-2048** | Compact signatures; fast sign/verify once keys exist | The most expensive key generation measured in this project (fresh domain parameters, ~8.8 s here); signature-only |
| **DH-2048** | Simple, well-understood key exchange | Very slow shared-secret derivation; enormous public-value overhead relative to the session key produced |

## 7. Files in This Folder

```
MIT SuperCloud Dataset/
├── MITSupercloud_Evaluation.ipynb   # Full benchmarking notebook (this dataset)
├── scheduler_data.csv.zip           # Source data
└── PKC_Result/
    ├── DatasetInsights/
    │   └── gpu_vs_nongpu_breakdown.png     # GPU vs non-GPU job cost comparison
    ├── Encryption/{RSA,ECC}/...            # Per-algorithm JSON results + charts
    ├── Encryption/Comparison/...           # Comparison table, charts, general eval metrics, significance tests
    ├── DigitalSignature/{RSA,ECC,DSA}/...  # Per-algorithm JSON results + charts
    ├── DigitalSignature/Comparison/...     # Comparison table, charts, general eval metrics, significance tests
    ├── KeyExchange/{ECC,DH}/...            # Per-algorithm JSON results + charts
    ├── KeyExchange/Comparison/...          # Comparison table + charts
    └── OverallSummary/overall_summary.png  # Best algorithm per aspect, side by side
```

## 8. How to Run This Notebook

### 8.1 Requirements
- **Python:** 3.9 or later
- **Environment:** Written for Google Colab (uses `google.colab.drive` to mount Drive), but runs in any standard Jupyter environment if the Drive-mount cell is skipped/replaced with a local path
- **Libraries:** `pycryptodome`, `cryptography`, `pandas`, `numpy`, `scipy`, `matplotlib`

### 8.2 Installation
```bash
pip install pycryptodome cryptography scipy pandas matplotlib
```

### 8.3 Data Setup
1. Unzip `scheduler_data.csv.zip` to obtain `scheduler_data.csv`.
2. If running in Google Colab: upload the CSV to your Google Drive and update `DATA_PATH` in the **Config** cell, e.g.:
   ```python
   DATA_PATH = "/content/drive/MyDrive/MIT SuperCloud Dataset/scheduler_data.csv"
   ```
3. If running locally: skip the "Connect Drive" cell and point `DATA_PATH` to the local file.

### 8.4 Running the Notebook
Run all cells top to bottom: **Connect Drive → Install Packages → Imports → Config → Load Data & Preprocess → Dataset-Specific Insight (GPU vs non-GPU) → Measurement Core →** the three pipeline definition sections **→ Correctness Checks → Benchmark Runners →** each **Run —** cell **→** the **Compare** cells **→ Overall Summary → General Evaluation Metrics**.

Results (JSON) and charts (PNG) save automatically to `PKC_Result/<Aspect>/<Algorithm>/` and `PKC_Result/<Aspect>/Comparison/`, and every chart also displays inline as it runs.

### 8.5 Notes / Prerequisites
- `REPS = 10`, `RECORD_SAMPLE_SIZE = 200`, `SCALABILITY_SIZES_MB = [1, 5, 10, 25, 50, 100, 250, 500]` — same experimental settings as the other two datasets in this project, for direct comparability.
- `ASSUMED_CORE_WATTS = 15.0` — Energy Consumption is an **estimate only** (CPU-time × assumed wattage), not a physical power measurement.
- The preprocessing step (dropping `gres_used`, filling `gres_req`/`gres_alloc`/`tres_alloc` with `'none'`) runs automatically in the **Load Data & Preprocess** cell — no manual cleaning is required.
- The **Correctness Checks** cell must pass before trusting any timing results.
