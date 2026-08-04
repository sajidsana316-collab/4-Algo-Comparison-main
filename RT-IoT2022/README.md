# PKC Benchmarking on RT-IoT2022

## 1. Overview

This folder benchmarks four public-key cryptography (PKC) algorithms — **RSA-2048**, **ECC (NIST P-256)**, **DSA-2048**, and **Diffie-Hellman (DH-2048)** — on real network-flow records from the **RT-IoT2022** dataset. The goal is not to argue that one algorithm is "more secure" than another (their theoretical security is already well established); it is to measure, empirically, how each algorithm actually behaves when applied to real data: how fast it is, how much memory and energy it consumes, how much overhead it adds, and how well it scales.

Every algorithm is evaluated across the three practical roles that PKC is actually used for:

| Aspect | Algorithms tested | Why these algorithms |
|---|---|---|
| **Encryption** | RSA (RSA-OAEP hybrid), ECC (ECIES-style hybrid) | These are the only two of the four that can protect confidentiality of bulk data. Neither RSA nor ECC encrypts large payloads directly — both wrap a fresh AES-256 key and let AES-256-GCM do the bulk encryption. This is the standard, honest "hybrid encryption" model used in real-world systems (e.g., TLS, PGP). |
| **Digital Signature** | RSA (RSA-PSS), ECC (ECDSA), DSA | All three natively support signing. DSA exists *only* to sign — it cannot encrypt — so it only appears in this aspect. |
| **Key Exchange** | ECC (raw ECDH), DH (raw Diffie-Hellman) | Only these two establish a shared secret between two parties without either one encrypting anything. |

## 2. Dataset

**Name:** RT-IoT2022
**What it is:** A real-time IoT/IIoT network-traffic dataset containing labeled network flow records (normal traffic and multiple attack types) captured from a real IoT test-bed.
**Source:** RT-IoT2022 is a publicly released dataset available on the **UCI Machine Learning Repository**: https://archive.ics.uci.edu/dataset/942/rt-iot2022
**File used:** `processed_rt_iot2022.csv` (provided in this folder as `DataSet_CSV_processed_rt_iot2022.zip`)
**Size after loading:** 123,117 rows × 85 columns
**Serialization:** Each row is converted to JSON and encoded to bytes before being fed to the algorithms — this JSON blob is the "plaintext" (`S_p`) that every algorithm protects.
**Record size (real data):** mean ≈ 2,344 bytes, median ≈ 2,366 bytes, min ≈ 2,017 bytes, max ≈ 2,499 bytes — i.e., fairly large, uniform-sized records.

## 3. What Was Implemented

The notebook (`ALGO_EVALUATION_4algorithms.ipynb`) implements, for each algorithm/aspect combination, a full measurement pipeline that computes:

- **KGT** — Key Generation Time (mean over 10 repetitions, plus std/CV%/95% CI)
- **ET / DT** — Encryption/Signing Time and Decryption/Verification Time, measured on 200 real sampled records
- **CSO** — Ciphertext/Signature Size Overhead, in bytes and as a percentage of plaintext size
- **MC** — Peak Memory Consumption (via `tracemalloc`, idle-corrected)
- **CPU%** — CPU utilization (CPU time ÷ wall time)
- **EC** — Estimated Energy Consumption (CPU-time × an assumed 15 W/core — clearly marked as an *estimate*, since no physical power meter was available)
- **TP** — Throughput (MB/s), measured across a scalability sweep of 1 MB → 500 MB of real (tiled where necessary) data
- **SC** — Scalability behavior across that same sweep
- **Speedup** — Relative speed of each algorithm against the slowest one in its aspect
- **Security Level** — Standardized NIST SP 800-57 equivalent security bits (RSA-2048 ≈ 112 bits, ECC P-256 ≈ 128 bits, DSA-2048 ≈ 112 bits, DH-2048 ≈ 112 bits)
- **Composite Score (0–100)** — A weighted combination of the normalized metrics above (KGT 25%, ET 20%, CSO 15%, Memory 15%, Energy 10%, Security 15%), used to rank algorithms within each aspect

Correctness is verified before any timing is trusted: every encrypt→decrypt, sign→verify, and key-exchange round-trip is checked to actually produce the correct result.

## 4. Results — What Was Achieved

### 4.1 Encryption (RSA vs ECC)

| Metric | RSA-2048 | ECC (P-256) |
|---|---|---|
| Key Generation Time | 1272.05 ms | 0.28 ms |
| Encryption Time | 4.04 ms | 7.61 ms |
| Decryption Time | 4.86 ms | 5.12 ms |
| Ciphertext Overhead | 13.24% | 4.33% |
| Peak Memory (KeyGen) | 1.75 KB | 1.12 KB |
| Energy (KeyGen) | 18,898.6 mJ | 4.4 mJ |
| Throughput @ 500 MB | 272.76 MB/s | 563.03 MB/s |
| Security Level | 112 bits | 128 bits |
| **Composite Score** | **20.0** | **80.0** |

**Winner: ECC.** RSA key generation requires finding two large probabilistic primes (a genuinely expensive search), taking ~1.27 seconds — roughly **4,550× slower** than ECC's key generation, which is just one random scalar multiplication on a fixed elliptic curve and takes a fraction of a millisecond. RSA does win narrowly on raw per-record encryption time (4.04 ms vs 7.61 ms for ECC), since ECC's hybrid encryption performs an extra EC point multiplication to derive its session key. But RSA loses everywhere else that matters in practice: it uses ~1,600× more energy per key generation, needs a larger 2,048-bit key to reach only 112-bit security (versus ECC's 256-bit key reaching 128-bit security), and adds ~3× more ciphertext overhead per record (its wrapped AES key alone is 256 bytes, versus ECC's ~65-byte ephemeral public key).

### 4.2 Digital Signature (RSA vs ECC vs DSA)

| Metric | RSA (RSA-PSS) | ECC (ECDSA) | DSA |
|---|---|---|---|
| Key Generation Time | 988.99 ms | 0.28 ms | **12,829.74 ms** |
| Signing Time | 8.86 ms | 4.14 ms | 4.63 ms |
| Verification Time | 5.47 ms | 7.22 ms | 4.21 ms |
| Signature Overhead | 11.93% | 2.98% | 2.61% |
| Energy (KeyGen) | 14,564.9 mJ | 4.5 mJ | 190,895.4 mJ |
| Security Level | 112 bits | 128 bits | 112 bits |
| **Composite Score** | **34.1** | **99.4** | **32.9** |

**Winner: ECC (ECDSA).** DSA is the clearest loser here: generating a fresh 2048-bit DSA keypair means generating a full domain parameter set (a large prime group), which took **~12.8 seconds** on average — over **45,000× slower** than ECC and even ~13× slower than RSA. This single cost dominates DSA's overall score despite DSA having a small, efficient signature. ECC wins because ECDSA signatures are compact (~2.98% overhead vs RSA's ~11.93%) *and* its key generation is essentially free, giving it the best balance across every weighted category.

### 4.3 Key Exchange (ECC/ECDH vs DH)

| Metric | ECC (ECDH) | DH |
|---|---|---|
| Key Generation Time | 0.73 ms | 2.67 ms |
| Initiator Derivation Time | 7.10 ms | 3.14 ms |
| Responder Derivation Time | 5.00 ms | **367.66 ms** |
| Public-Value Overhead | 103.1% | 700.0% |
| Security Level | 128 bits | 112 bits |
| **Composite Score** | **65.0** | **35.0** |

**Winner: ECC (ECDH).** This is the starkest result in the whole benchmark. Classic Diffie-Hellman derives its shared secret using modular exponentiation over a 2048-bit prime group — an inherently heavy operation — which is why the responder's derivation step took **~368 ms**, roughly **70× slower** than ECC's equivalent step (~5 ms), which only needs a point multiplication on a 256-bit curve. On top of that, DH's public value (its 2048-bit `y`) is far larger than the 32-byte session key it ultimately produces, giving it a **700% transmission overhead** versus ECC's 103%. ECC reaches a higher security level (128 vs 112 bits) with a smaller key and a dramatically cheaper exchange.

## 5. Overall Conclusion for This Dataset

**ECC won all three aspects on RT-IoT2022** (Encryption: 80.0, Digital Signature: 99.4, Key Exchange: 65.0). The consistent, underlying reason is structural, not incidental: elliptic-curve cryptography reaches the same (or better) security level as RSA/DSA/DH using a key roughly **8× smaller** (256-bit curve vs 2,048-bit modulus). A smaller key means less arithmetic per operation — which shows up directly as faster key generation, lower CPU time, lower energy use, and smaller transmitted overhead, everywhere key generation or modular-exponentiation-style math dominates. RSA remains competitive on raw per-record encryption/decryption speed (a place where finite-field modular arithmetic on already-generated keys is genuinely fast), and DSA's compact signature is respectable — but neither can offset the enormous one-time cost of generating their large keys, especially DSA, whose fresh domain-parameter generation makes it the single most expensive operation measured in this entire study.

## 6. Strengths & Weaknesses Summary

| Algorithm | Strengths | Weaknesses |
|---|---|---|
| **RSA-2048** | Fast per-record encrypt/verify once a key exists; mature, universally supported | Very slow, energy-hungry key generation; larger ciphertext/signature overhead for the same security |
| **ECC (P-256)** | Near-instant key generation; smallest keys for the security level; lowest energy use; smallest overhead in most cases | Marginally slower raw encryption than RSA on tiny records; requires careful curve/implementation choices |
| **DSA-2048** | Compact signatures; fast sign/verify once keys exist | By far the most expensive key generation of any algorithm tested (fresh domain parameters); signature-only, cannot encrypt or exchange keys |
| **DH-2048** | Simple, well-understood key exchange | Very slow shared-secret derivation (large modular exponentiation); huge public-value transmission overhead relative to the secret it produces |

## 7. Files in This Folder

```
RT-IoT2022/
├── ALGO_EVALUATION_4algorithms.ipynb      # Full benchmarking notebook (this dataset)
├── DataSet_CSV_processed_rt_iot2022.zip   # Source data
└── PKC_RESULT/
    ├── Encryption/{RSA,ECC}/...           # Per-algorithm JSON results + charts
    ├── Encryption/Comparison/...          # RSA vs ECC comparison table + charts
    ├── DigitalSignature/{RSA,ECC,DSA}/... # Per-algorithm JSON results + charts
    ├── DigitalSignature/Comparison/...    # RSA vs ECC vs DSA comparison table + charts
    ├── KeyExchange/{ECC,DH}/...           # Per-algorithm JSON results + charts
    ├── KeyExchange/Comparison/...         # ECC vs DH comparison table + charts
    └── OverallSummary/overall_summary.png # Best algorithm per aspect, side by side
```

Each `Comparison/` folder contains:
- `<Aspect>_comparison_table.csv` / `.json` — the raw numeric results behind the tables above
- `1_kgt.png` … `9_security.png` — individual per-metric bar charts
- `10_speedup.png`, `11_ranking.png`, `12_radar_tradeoff.png` — speedup, ranked composite score, and a tradeoff radar chart
- `13_scalability_all.png`, `14_cso_scaling_all.png` — throughput and overhead scaling from 1 MB → 500 MB (Encryption and Digital Signature only; Key Exchange has no bulk payload to scale)

## 8. How to Run This Notebook

### 8.1 Requirements
- **Python:** 3.9 or later
- **Environment:** Written for Google Colab (uses `google.colab.drive` to mount Drive), but runs in any standard Jupyter environment if the Drive-mount cell is skipped/replaced with a local path
- **Libraries:**
  - `pycryptodome` (RSA, DSA, ECC, signature schemes)
  - `cryptography` (Diffie-Hellman, AES-GCM, HKDF)
  - `pandas`, `numpy`
  - `scipy` (confidence intervals)
  - `matplotlib` (all charts)

### 8.2 Installation
```bash
pip install pycryptodome cryptography scipy pandas matplotlib
```

### 8.3 Data Setup
1. Unzip `DataSet_CSV_processed_rt_iot2022.zip` to obtain `processed_rt_iot2022.csv`.
2. If running in Google Colab: upload the CSV to your Google Drive and update the `DATA_PATH` variable in the **Config** cell to point to it, e.g.:
   ```python
   DATA_PATH = "/content/drive/MyDrive/rt-iot2022-extracted/processed_rt_iot2022.csv"
   ```
3. If running locally / outside Colab: remove or skip the "Connect Drive" cell, and simply set `DATA_PATH` to the local file path.

### 8.4 Running the Notebook
1. Open `ALGO_EVALUATION_4algorithms.ipynb` in Jupyter or Google Colab.
2. Run all cells from top to bottom, in order:
   - **Connect Drive** → **Install Packages** → **Imports** → **Config** → **Load Data & Serialize Rows** → **Measurement Core** → the three pipeline definition sections (Encryption, Digital Signature, Key Exchange) → **Correctness Checks** → **Benchmark Runners** → each **Run —** cell for every algorithm → the **Compare** cells → **Overall Summary**.
3. Results (JSON) and charts (PNG) are saved automatically to `PKC_RESULT/<Aspect>/<Algorithm>/` and `PKC_RESULT/<Aspect>/Comparison/`, and every chart is also displayed inline in the notebook as it runs.

### 8.5 Notes / Prerequisites
- `REPS = 10` — every timing measurement is repeated 10 times and averaged; increase for tighter confidence intervals, at the cost of longer runtime.
- `RECORD_SAMPLE_SIZE = 200` — the number of real rows sampled for per-record ET/DT/CSO measurement.
- `SCALABILITY_SIZES_MB = [1, 5, 10, 25, 50, 100, 250, 500]` — the data sizes used for the throughput/scalability sweep. Real serialized data is tiled (repeated) once the sweep exceeds the amount of unique real data available, and this is flagged in the output.
- `ASSUMED_CORE_WATTS = 15.0` — used only for the Energy Consumption estimate; this is **not** a physical measurement (no power meter was used), and is clearly labeled as an estimate in all outputs.
- The **Correctness Checks** cell must pass before trusting any timing results — it confirms every algorithm actually encrypts/decrypts, signs/verifies, or exchanges keys correctly.
