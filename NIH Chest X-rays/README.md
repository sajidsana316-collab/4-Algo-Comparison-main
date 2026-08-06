# PKC Benchmarking on NIH Chest X-rays (ChestX-ray14) Metadata

## 1. Overview

This folder benchmarks four public-key cryptography (PKC) algorithms — **RSA-2048**, **ECC (NIST P-256)**, **DSA-2048**, and **Diffie-Hellman (DH-2048)** — on real medical-imaging metadata from the **NIH ChestX-ray14** dataset. As with the other two datasets in this project, the aim is purely experimental: to measure how each algorithm behaves on real healthcare data, not to re-litigate theoretical security, which is already established for all four algorithms.

| Aspect | Algorithms tested | Why these algorithms |
|---|---|---|
| **Encryption** | RSA (RSA-OAEP hybrid), ECC (ECIES-style hybrid) | Only these two can encrypt bulk data via a hybrid scheme (a fresh AES-256 key is wrapped by the asymmetric algorithm, and AES-256-GCM does the actual encryption). |
| **Digital Signature** | RSA (RSA-PSS), ECC (ECDSA), DSA | All three support signing; DSA does *only* this. |
| **Key Exchange** | ECC (raw ECDH), DH (raw Diffie-Hellman) | Only these two establish a shared secret between two parties. |

## 2. Dataset

**Name:** NIH ChestX-ray14 metadata (`Data_Entry_2017.csv`)
**What it is:** The metadata table accompanying the NIH Clinical Center's ChestX-ray14 dataset — one of the largest publicly available chest X-ray datasets. Each row describes one X-ray image: image filename, diagnostic finding labels, follow-up number, patient ID, patient age, patient gender, view position, and image dimensions/pixel spacing.
**Source:** ChestX-ray14 was released publicly by the **NIH Clinical Center** and is available on Kaggle under the NIH Chest X-rays organization: https://www.kaggle.com/organizations/nih-chest-xrays/datasets
**File used:** `Data_Entry_2017.csv`
**Size after loading:** 112,120 rows × 12 columns
**Record size (real data):** mean ≈ 289 bytes, median ≈ 287 bytes, min ≈ 279 bytes, max ≈ 343 bytes — by far the **smallest and most tightly clustered** records of the three datasets in this project, since only metadata (not image pixels) is being protected.

## 3. What Was Implemented

The notebook (`ChestXray_Evaluation.ipynb`) implements the same measurement pipeline used throughout this project, computing per algorithm/aspect: **KGT, ET, DT, CSO (bytes and %), MC, CPU%, EC (estimate), TP, SC, Speedup, standardized Security Level, and a weighted Composite Score (0–100)** used to rank algorithms within each aspect. Correctness (encrypt→decrypt, sign→verify, matching shared secrets) is verified before any timing is trusted.

## 4. Results — What Was Achieved

### 4.1 Encryption (RSA vs ECC)

| Metric | RSA-2048 | ECC (P-256) |
|---|---|---|
| Key Generation Time | 1649.79 ms | 0.53 ms |
| Encryption Time | 4.52 ms | 5.90 ms |
| Decryption Time | 5.51 ms | 4.00 ms |
| Ciphertext Overhead | **97.75%** | 32.01% |
| Energy (KeyGen) | 24,448.9 mJ | 5.8 mJ |
| Throughput @ 500 MB | 563.92 MB/s | 545.78 MB/s |
| Security Level | 112 bits | 128 bits |
| **Composite Score** | **20.0** | **80.0** |

**Winner: ECC.** The headline number here is RSA's ciphertext overhead: **97.75%**, meaning the encrypted output is almost *double* the size of the original metadata record. This is not a flaw specific to this run — it is a direct, predictable consequence of this dataset's very small record size (~289 bytes). RSA's hybrid encryption always attaches a ~256-byte RSA-OAEP-wrapped AES key plus a 12-byte nonce and a 16-byte GCM tag, a roughly fixed cost of ~284 bytes *no matter how small the plaintext is*. Against a 289-byte record, that fixed cost is almost as large as the record itself. ECC's overhead is far lower (32.0%) for the same reason in reverse: its ephemeral public key (~65 bytes over SEC1) is a much smaller fixed cost, so it stays proportionally cheaper even on tiny records. See §5 for the full three-dataset comparison of this effect.

### 4.2 Digital Signature (RSA vs ECC vs DSA)

| Metric | RSA (RSA-PSS) | ECC (ECDSA) | DSA |
|---|---|---|---|
| Key Generation Time | 1584.33 ms | 0.29 ms | **6159.49 ms** |
| Signing Time | 6.39 ms | 3.48 ms | 5.62 ms |
| Verification Time | 3.99 ms | 5.93 ms | 5.19 ms |
| Signature Overhead | **88.11%** | 22.03% | 19.27% |
| Energy (KeyGen) | 22,059.2 mJ | 4.1 mJ | 91,281.7 mJ |
| Security Level | 112 bits | 128 bits | 112 bits |
| **Composite Score** | **27.9** | **99.4** | **20.3** |

**Winner: ECC (ECDSA)**, by a wide margin (99.4 vs the next-best score of 27.9). RSA's fixed ~256-byte signature again dominates a tiny ~289-byte record, producing an 88.11% overhead. DSA's signature is smaller and cheaper proportionally (19.27%), but its key-generation cost (~6.16 seconds, driven by generating fresh DSA domain parameters) is so large that it drags DSA to the *lowest* composite score of any algorithm on any dataset in this project. ECC again combines the best of both worlds: a compact ECDSA signature and near-instant key generation.

### 4.3 Key Exchange (ECC/ECDH vs DH)

| Metric | ECC (ECDH) | DH |
|---|---|---|
| Key Generation Time | 0.30 ms | 3.12 ms |
| Initiator Derivation Time | 5.72 ms | 3.51 ms |
| Responder Derivation Time | 4.00 ms | **395.88 ms** |
| Public-Value Overhead | 103.1% | 700.0% |
| Security Level | 128 bits | 112 bits |
| **Composite Score** | **65.0** | **35.0** |

**Winner: ECC (ECDH).** Key Exchange is unaffected by record size (it doesn't process the dataset's records at all), so this result matches the other two datasets almost exactly: DH's shared-secret derivation via 2048-bit modular exponentiation is roughly **99× slower** than ECC's point multiplication (395.88 ms vs 4.00 ms), and its public value is proportionally far larger than the session key it produces (700% vs 103% overhead).

## 5. Overall Conclusion for This Dataset

**ECC won all three aspects on the NIH Chest X-rays dataset** (Encryption: 80.0, Digital Signature: 99.4, Key Exchange: 65.0). This dataset provides the clearest demonstration in the whole project of *why record size matters* when choosing an asymmetric algorithm for encryption or signing. RSA (and, for signatures, DSA) attach a roughly fixed number of overhead bytes to every operation, regardless of message size — a cost that is nearly invisible on large payloads but becomes crippling on small ones:

| Dataset | Avg. record size | RSA encryption overhead |
|---|---|---|
| RT-IoT2022 | ~2,344 bytes | 13.24% |
| MIT Supercloud | ~1,270 bytes | 40.75% |
| **NIH Chest X-rays** | **~289 bytes** | **97.75%** |

This is a genuine, dataset-driven finding, not a coincidence: as average record size shrinks, RSA's fixed-size overhead consumes a proportionally larger share of the ciphertext. ECC's much smaller fixed overhead (its ephemeral public key is roughly a quarter the size of RSA's wrapped key) means it scales far more gracefully to small records, which is precisely the situation medical metadata, IoT telemetry, and other small-record systems are in. Combined with ECC's consistently faster key generation and lower energy use (seen across all three datasets), this is strong practical evidence that **ECC is the better-suited PKC choice for protecting large volumes of small healthcare records** — exactly the kind of workload this dataset represents.

## 6. Strengths & Weaknesses Summary

| Algorithm | Strengths | Weaknesses |
|---|---|---|
| **RSA-2048** | Fast per-record encrypt/verify once a key exists; mature, widely supported | Overhead becomes extreme (nearly 100%) on small records like this dataset's metadata; slow, energy-hungry key generation |
| **ECC (P-256)** | Near-instant key generation; lowest energy use; overhead stays manageable even on very small records | Slightly slower raw encryption/signing than RSA per record |
| **DSA-2048** | Proportionally smaller signature overhead than RSA on small records | The single most expensive key generation observed anywhere in this project relative to its own record-level gains; lowest overall score of any algorithm tested; signature-only |
| **DH-2048** | Simple, well-understood key exchange | Very slow shared-secret derivation; enormous public-value overhead relative to the session key produced |

## 7. Files in This Folder

```
NIH Chest X-rays/
├── ChestXray_Evaluation.ipynb          # Full benchmarking notebook (this dataset)
└── PKC_RESULT/
    ├── Encryption/{RSA,ECC}/...            # Per-algorithm JSON results + charts
    ├── Encryption/Comparison/...           # Comparison table + charts
    ├── DigitalSignature/{RSA,ECC,DSA}/...  # Per-algorithm JSON results + charts
    ├── DigitalSignature/Comparison/...     # Comparison table + charts
    ├── KeyExchange/{ECC,DH}/...            # Per-algorithm JSON results + charts
    ├── KeyExchange/Comparison/...          # Comparison table + charts
    └── OverallSummary/overall_summary.png  # Best algorithm per aspect, side by side
```

*Note: the source metadata CSV (`Data_Entry_2017.csv`) is not bundled in this folder — see §8.3 below for how to obtain it.*

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
1. Download the ChestX-ray14 metadata file (commonly named `Data_Entry_2017.csv`) from the dataset's public source (see §2 above).
2. If running in Google Colab: upload the CSV to your Google Drive and update `DATA_PATH` in the **Config** cell, e.g.:
   ```python
   DATA_PATH = "/content/drive/MyDrive/X-RAY-Dataset/Data_Entry_2017.csv"
   ```
3. If running locally: skip the "Connect Drive" cell and point `DATA_PATH` to the local file. Only the metadata CSV is required — the actual X-ray image files are **not** used or needed by this notebook, since it evaluates cryptography on the metadata records, not the images.

### 8.4 Running the Notebook
Run all cells top to bottom: **Connect Drive → Install Packages → Imports → Config → Load Data & Serialize Rows → Measurement Core →** the three pipeline definition sections (Encryption, Digital Signature, Key Exchange) **→ Correctness Checks → Benchmark Runners →** each **Run —** cell for every algorithm **→** the **Compare** cells **→ Overall Summary**.

Results (JSON) and charts (PNG) save automatically to `PKC_RESULT/<Aspect>/<Algorithm>/` and `PKC_RESULT/<Aspect>/Comparison/`, and every chart also displays inline as it runs.

### 8.5 Notes / Prerequisites
- `REPS = 10`, `RECORD_SAMPLE_SIZE = 200`, `SCALABILITY_SIZES_MB = [1, 5, 10, 25, 50, 100, 250, 500]` — identical experimental settings to the other two datasets, for direct comparability.
- `ASSUMED_CORE_WATTS = 15.0` — Energy Consumption is an **estimate only**, not a physical power measurement.
- The **Correctness Checks** cell must pass before trusting any timing results.
