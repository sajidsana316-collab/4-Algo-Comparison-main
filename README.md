# Empirical Benchmarking of Public-Key Cryptography Algorithms Across Three Real-World Datasets

## 1. Project Overview

This project is a large-scale **experimental benchmark** of four widely used public-key cryptography (PKC) algorithms — **RSA-2048**, **ECC (NIST P-256)**, **DSA-2048**, and **Diffie-Hellman (DH-2048)** — applied to real data drawn from three very different domains: network/IoT security, high-performance computing, and healthcare imaging metadata.

Each algorithm is evaluated in the specific role(s) it is actually capable of performing:

| Aspect | Algorithms evaluated | Purpose |
|---|---|---|
| **Encryption** | RSA (hybrid RSA-OAEP + AES-256-GCM), ECC (ECIES-style hybrid) | Protects confidentiality of bulk data |
| **Digital Signature** | RSA (RSA-PSS), ECC (ECDSA), DSA | Protects integrity/authenticity of data |
| **Key Exchange** | ECC (raw ECDH), DH (raw Diffie-Hellman) | Establishes a shared secret between two parties |

No algorithm is forced into a role it cannot perform — for example, DSA cannot encrypt, so it is only evaluated for digital signatures, and DH cannot sign or encrypt, so it is only evaluated for key exchange.

## 2. Purpose and Motivation

Public-key cryptography is everywhere — in TLS, VPNs, IoT device authentication, medical data protection, and cloud key management — and the *theoretical* security of RSA, ECC, DSA, and DH is already well established and not in question. What is far less commonly measured, and what motivates this project, is how these algorithms behave **in practice**, on **real data**, under **real workload conditions**: How much slower is RSA key generation than ECC's, in actual milliseconds? How much bigger is a signed record, actually? How does that change when the records being protected are large network flows versus tiny medical metadata rows? Does one algorithm's advantage hold up consistently across completely different kinds of data, or is it dataset-dependent?

This project answers those questions with controlled, repeated, real measurements rather than theoretical big-O comparisons.

## 3. Research Objective

The primary goal of this research is **not** to prove that one encryption algorithm is mathematically more secure than another — their theoretical security has already been established by decades of cryptographic research (and is reported here only as a standardized reference number, per NIST SP 800-57).

Instead, the objective is to answer practical, engineering-relevant questions:

- **Which algorithm is the fastest** — for key generation, and for actually encrypting/signing/exchanging data?
- **Which algorithm consumes the least memory?**
- **Which algorithm scales best** as data volume grows from kilobytes to hundreds of megabytes?
- **Which algorithm offers the best security-to-performance trade-off** — i.e., the most security gained per unit of time, memory, and energy spent?
- **Which algorithm is most suitable for IoT, cloud computing, healthcare, and scientific data** — domains where devices are often resource-constrained, data volumes are large, or both?

The emphasis throughout this project is on **experimental benchmarking, not theoretical cryptography**. Every number reported comes from real, repeated measurements on real data, not from asymptotic complexity analysis.

## 4. The Three Datasets

Three datasets were deliberately chosen to represent three very different real-world data profiles, so that findings could be checked for consistency rather than being an artifact of one dataset's particular shape:

| Dataset | Domain | Rows | Avg. record size | Folder |
|---|---|---|---|---|
| **RT-IoT2022** | IoT / network intrusion detection | 123,117 | ~2,344 bytes (large, uniform) | [`RT-IoT2022/`](./RT-IoT2022/README.md) |
| **MIT Supercloud Dataset** | HPC job scheduling (datacenter) | 287,173 | ~1,270 bytes (wide spread, GPU vs non-GPU) | [`MIT SuperCloud Dataset/`](./MIT%20SuperCloud%20Dataset/README.md) |
| **NIH ChestX-ray14 metadata** | Healthcare / medical imaging | 112,120 | ~289 bytes (small, tightly clustered) | [`NIH Chest X-rays/`](./NIH%20Chest%20X-rays/README.md) |

**Why three datasets, and why these three:** Each dataset represents a distinct, practically important use case for PKC — securing IoT telemetry, securing datacenter/cloud scheduling logs, and securing healthcare records — and, just as importantly, each has a **different typical record size**. This was essential to the research objective: a result that only appeared on one dataset would be a coincidence, but a result that holds across records ranging from ~289 bytes to ~2,344 bytes is a genuine, size-independent property of the algorithm itself. As shown below, that is exactly what was found.

**Dataset sources:**
- RT-IoT2022 — UCI Machine Learning Repository: https://archive.ics.uci.edu/dataset/942/rt-iot2022
- MIT Supercloud Dataset (Datacenter Challenge scheduler logs) — Kaggle: https://www.kaggle.com/datasets/skylarkphantom/mit-datacenter-challenge-data
- NIH ChestX-ray14 metadata — Kaggle (NIH Chest X-rays organization): https://www.kaggle.com/organizations/nih-chest-xrays/datasets

## 5. Methodology

The same measurement pipeline, algorithm implementations, key sizes, and repetition counts were applied identically across all three datasets, so results are directly comparable.

### 5.1 Algorithms and parameters
- **RSA:** 2048-bit modulus, hybrid encryption (RSA-OAEP wraps a fresh AES-256 key; AES-256-GCM encrypts the record) and RSA-PSS for signing
- **ECC:** NIST P-256 curve, ECIES-style hybrid encryption (ephemeral ECDH + HKDF-SHA256 derives an AES-256 key; AES-256-GCM encrypts the record) and ECDSA for signing
- **DSA:** 2048-bit domain parameters, standard DSA signing (signature-only — cannot encrypt or exchange keys)
- **DH:** 2048-bit MODP group, raw Diffie-Hellman key exchange (exchange-only — cannot encrypt or sign)

### 5.2 Metrics measured
For every algorithm and every applicable aspect, the following were computed, following formulas specified for this study:

- **KGT** — Key Generation Time = t_end − t_start, averaged over N runs
- **ET / DT** — Encryption/Signing Time and Decryption/Verification Time, averaged over N runs
- **CSO** — Ciphertext/Signature Size Overhead: O_c = S_c − S_p (bytes), and O_c(%) = (S_c − S_p)/S_p × 100
- **MC** — Peak Memory Consumption: M_c = M_peak − M_idle, averaged over N runs
- **CPU%** — CPU Utilization = (T_CPU / T_wall) × 100
- **TP** — Throughput = data size / encryption time (MB/s), measured across a 1 MB → 500 MB scalability sweep
- **EC** — Energy Consumption = P × T (an *estimate*, using CPU time × an assumed 15 W/core, since no physical power meter was available — clearly labeled as such everywhere it is reported)
- **SC** — Scalability = execution time / file size (single-size), or (T₂−T₁)/(S₂−S₁) between two sizes
- **Standard Deviation / CV% / 95% CI** — to quantify measurement variability across repeated runs
- **Speedup** — T_baseline / T_algorithm, comparing each algorithm to the slowest one in its aspect
- **Security Level** — standardized NIST SP 800-57 equivalent security bits (not calculated from scratch — the accepted standardized values are used: RSA-2048 ≈ 112 bits, ECC P-256 ≈ 128 bits, DSA-2048 ≈ 112 bits, DH-2048 ≈ 112 bits)
- **Composite Score (0–100)** — a weighted combination of the normalized metrics above (KGT 25%, ET 20%, CSO 15%, Memory 15%, Energy 10%, Security 15%), used to produce a single ranked score per algorithm per aspect

Each measurement was repeated **10 times** and averaged, with standard deviation, coefficient of variation, and 95% confidence intervals reported alongside the mean. Per-record timings (ET/DT/CSO) were measured on **200 real sampled records** from each dataset. Throughput and scalability were measured across data sizes from **1 MB up to 500 MB**, built from real serialized records (tiled/repeated once the sweep exceeded the amount of unique real data available, with this clearly flagged in the results). Every encryption, signature, and key-exchange pipeline was **correctness-checked** (round-trip verified) before any timing was trusted.

### 5.3 Experimental Workflow

1. **Load and serialize** each dataset's real records to JSON bytes (this becomes the plaintext `S_p` every algorithm protects).
2. **Generate keys** for each algorithm and measure Key Generation Time (KGT) over 10 repetitions.
3. **Run each aspect's pipeline** (Encryption, Digital Signature, Key Exchange) on 200 sampled real records, measuring ET, DT, CSO, memory, CPU%, and energy.
4. **Run the scalability sweep** (1 MB → 500 MB, Encryption and Digital Signature only) to measure throughput and overhead scaling.
5. **Verify correctness** of every pipeline before trusting any of the above.
6. **Compare algorithms within each aspect** — producing comparison tables, per-metric bar charts, a ranked composite score, and a tradeoff radar chart.
7. **Summarize across all three aspects** to identify the single best-performing algorithm per aspect, per dataset.

This entire workflow was run independently, start to finish, on each of the three datasets.

## 6. Results — Algorithm Comparison Across All Three Datasets

### 6.1 Composite Scores (0–100, higher is better)

| Aspect | Dataset | RSA | ECC | DSA | DH |
|---|---|---|---|---|---|
| Encryption | RT-IoT2022 | 20.0 | **80.0** | — | — |
| Encryption | MIT Supercloud | 20.0 | **80.0** | — | — |
| Encryption | NIH ChestX-rays | 20.0 | **80.0** | — | — |
| Digital Signature | RT-IoT2022 | 34.1 | **99.4** | 32.9 | — |
| Digital Signature | MIT Supercloud | 31.6 | **95.5** | 35.0 | — |
| Digital Signature | NIH ChestX-rays | 27.9 | **99.4** | 20.3 | — |
| Key Exchange | RT-IoT2022 | — | **65.0** | — | 35.0 |
| Key Exchange | MIT Supercloud | — | **65.0** | — | 35.0 |
| Key Exchange | NIH ChestX-rays | — | **65.0** | — | 35.0 |

**ECC won every single aspect on every single dataset — nine wins out of nine.**

### 6.2 Why ECC Won Everywhere

The reason is structural, not incidental, and it holds regardless of what kind of data is being protected:

1. **Key generation cost.** RSA-2048 and DSA-2048 both require constructing a large number-theoretic object — RSA needs two large probable primes; DSA needs a full domain-parameter set (an even heavier search). Both take over a second on average, and DSA's regularly took **6–13 seconds**, making it the single most expensive operation measured anywhere in this project. ECC's key generation is a single random scalar multiplication on a fixed, pre-agreed curve — it consistently completed in **under one millisecond**, 1,000–45,000× faster depending on the comparison.

2. **Equivalent security at a fraction of the key size.** A 256-bit elliptic curve key reaches ~128-bit security — *higher* than a 2,048-bit RSA/DSA/DH key's ~112-bit security, despite being roughly **8× smaller**. Smaller keys mean less arithmetic per operation, which is why ECC's advantage shows up simultaneously in key generation time, CPU utilization, and energy consumption.

3. **Lower overhead — and one that scales better with small records.** ECC's ephemeral public key (~65 bytes) is consistently smaller than RSA's wrapped-key overhead (~256 bytes) or DSA's signature. This gap becomes dramatically more important as record size shrinks: on RT-IoT2022's large ~2,344-byte records, RSA's overhead was a modest 13.24%; on the NIH dataset's tiny ~289-byte records, that same fixed-size overhead exploded to **97.75%** — effectively doubling every encrypted record. ECC's smaller fixed overhead kept its own percentage far lower (32.01%) on the exact same data. This is direct, reproducible evidence that **the smaller the records being protected, the larger ECC's practical advantage becomes**.

4. **Diffie-Hellman's exchange cost.** Classic DH's shared-secret derivation is modular exponentiation over a 2,048-bit prime — inherently expensive — measured consistently at **350–400 ms** per exchange across all three datasets, versus ECC's **4–7 ms**. DH's public value is also large relative to the 32-byte session key it produces, giving it a constant **700% transmission overhead** versus ECC's 103%.

### 6.3 Why RSA and DSA Could Not Match ECC

RSA is not a bad algorithm — its per-record encryption and verification are genuinely fast once a key already exists, and it remains the most universally deployed and interoperable PKC algorithm in existence. But its Achilles' heel, prime generation, is unavoidable: generating a fresh, cryptographically strong 2,048-bit RSA key means searching for large probable primes, a process that cannot be made instantaneous. In a workload that regenerates keys frequently — exactly the kind of repeated benchmarking done here — that one-time cost is paid over and over, and it dominates every composite score RSA received.

DSA suffers the same problem in a more extreme form: PyCryptodome's default DSA key generation regenerates the full domain parameter set (the prime group `p`, `q`, `g`) fresh each time, which is measurably heavier than even RSA's prime search. This is why DSA posted the single highest key-generation time recorded anywhere in this project (nearly 13 seconds, on RT-IoT2022) and, despite having a genuinely compact signature, could not overcome that cost in the composite ranking.

### 6.4 Limitations of the Weaker-Performing Algorithms

- **RSA-2048:** Best suited to systems where keys are generated rarely and reused for a long time (e.g., a server's long-lived TLS certificate) — poorly suited to systems that need to generate fresh keys frequently, or that handle many small records, where its fixed overhead becomes disproportionately expensive.
- **DSA-2048:** Signature-only (cannot encrypt or exchange keys), and its key-generation cost makes it the least practical of the four algorithms tested for any workload involving frequent key generation. Its only real advantage — signature compactness relative to RSA — was not enough to offset this in any of the three datasets.
- **DH-2048:** Its finite-field modular exponentiation is inherently more expensive than elliptic-curve arithmetic for equivalent security, and its public value is large relative to the secret it establishes — both of which are structural limitations of classical (non-elliptic-curve) Diffie-Hellman, not implementation issues.

## 7. Strengths and Practical Applications of ECC (the Best-Performing Algorithm)

Across all three datasets and all three PKC aspects, **ECC (P-256)** consistently delivered:
- The fastest key generation by a wide margin (sub-millisecond, versus seconds for RSA and DSA)
- The lowest energy consumption per operation
- Equal or better security (128-bit vs 112-bit) using an ~8× smaller key
- The lowest or most competitive overhead across every record size tested — from ~289-byte medical metadata to ~2,344-byte network-flow records
- The best composite score in **every single aspect on every single dataset** (9/9)

This makes ECC particularly well suited to exactly the domains named in the research objective:
- **IoT:** resource-constrained devices benefit enormously from ECC's tiny keys, low CPU/energy cost, and fast key generation — critical when devices generate or rotate keys frequently on limited hardware.
- **Cloud computing:** at scale, where millions of keys or sessions may be created, ECC's near-zero key-generation cost translates directly into lower compute cost and better throughput under load.
- **Healthcare:** as demonstrated directly by the NIH dataset results, medical records are often small, and ECC's overhead advantage grows precisely as record size shrinks — making it the more efficient choice for protecting large volumes of small patient/metadata records.
- **Scientific/HPC data:** the MIT Supercloud results show ECC holding its advantage even on a dataset with widely varying record sizes (GPU vs non-GPU job logs), confirming its performance is not dependent on uniform data.

## 8. Additional Observations

- **Bulk throughput converges at scale.** At large payload sizes (hundreds of MB), RSA's and ECC's *encryption* throughput converge to nearly identical values (e.g., 548.9 vs 541.7 MB/s on MIT Supercloud). This is expected: once a hybrid scheme's payload is large, the actual bulk encryption work is done by AES-256-GCM in both cases — the asymmetric algorithm's speed only matters for the one-time key setup, not for the bulk data itself. This makes ECC's advantage a *key-management and overhead* story more than a raw bulk-throughput story, and it is important to state that distinction plainly rather than overstate ECC's throughput advantage at scale.
- **Statistical confidence (MIT Supercloud dataset):** pairwise Welch's t-tests confirm every reported RSA-vs-ECC, RSA-vs-DSA, and ECC-vs-DSA difference in KGT and ET is statistically significant (p < 0.05, most by many orders of magnitude) — the performance gaps reported throughout this project are real effects, not measurement noise.
- **Record size is the single biggest driver of overhead differences between datasets.** The clean, monotonic relationship between average record size and RSA's ciphertext overhead percentage (13.24% → 40.75% → 97.75% as records shrink from ~2,344 → ~1,270 → ~289 bytes) is one of the strongest and most reproducible findings in this project, and directly supports the "security-to-performance trade-off" question in the research objective.

## 9. Project Structure

```
4-Algo-Comparison-main/
├── README.md                          # This file — full project documentation
├── RT-IoT2022/
│   ├── README.md                      # Dataset-specific documentation
│   ├── ALGO_EVALUATION_4algorithms.ipynb
│   ├── DataSet_CSV_processed_rt_iot2022.zip
│   └── PKC_RESULT/                    # Results, charts, comparison tables
├── MIT SuperCloud Dataset/
│   ├── README.md                      # Dataset-specific documentation
│   ├── MITSupercloud_Evaluation.ipynb
│   ├── scheduler_data.csv.zip
│   └── PKC_Result/                    # Results, charts, comparison tables, significance tests
└── NIH Chest X-rays/
    ├── README.md                      # Dataset-specific documentation
    ├── ChestXray_Evaluation.ipynb
    └── PKC_RESULT/                    # Results, charts, comparison tables
```

See each dataset folder's own `README.md` for that dataset's exact source, preprocessing steps, full result tables, and step-by-step instructions to reproduce that notebook's results.

## 10. How to Run This Project

Each dataset's notebook is fully self-contained and runs independently. General requirements (identical for all three notebooks):

### 10.1 Requirements
- **Python:** 3.9 or later
- **Environment:** Written for Google Colab (mounts Google Drive for data access), but runs in any standard Jupyter environment with minor changes to the data-loading path
- **Libraries:**
  - `pycryptodome` — RSA, DSA, ECC key generation, encryption, and signing
  - `cryptography` — Diffie-Hellman, AES-256-GCM, HKDF
  - `pandas`, `numpy` — data loading and preprocessing
  - `scipy` — confidence intervals and significance testing
  - `matplotlib` — all charts

### 10.2 Installation
```bash
pip install pycryptodome cryptography scipy pandas matplotlib
```

### 10.3 Running Each Dataset's Notebook
1. Obtain the dataset's source file (see the relevant dataset README for exact source and expected filename).
2. Open that dataset's `.ipynb` notebook in Jupyter or Google Colab.
3. Update the `DATA_PATH` variable in the **Config** cell to point to your local/Drive copy of the data.
4. Run all cells from top to bottom, in order.
5. Results (JSON) and charts (PNG) are saved automatically under that dataset's `PKC_Result(s)/<Aspect>/<Algorithm>/` and `.../Comparison/` folders, and every chart also displays inline in the notebook.

**Full, dataset-specific run instructions — including exact `DATA_PATH` examples, preprocessing notes, and any dataset-specific caveats — are provided in each dataset folder's own `README.md`:**
- [`RT-IoT2022/README.md`](./RT-IoT2022/README.md)
- [`MIT SuperCloud Dataset/README.md`](./MIT%20SuperCloud%20Dataset/README.md)
- [`NIH Chest X-rays/README.md`](./NIH%20Chest%20X-rays/README.md)

### 10.4 Reproducing the Exact Results Reported Here
All three notebooks use identical experimental settings for direct comparability:
- `REPS = 10` (every timing metric averaged over 10 runs)
- `RECORD_SAMPLE_SIZE = 200` (real records sampled for per-record ET/DT/CSO)
- `SCALABILITY_SIZES_MB = [1, 5, 10, 25, 50, 100, 250, 500]` (throughput/scalability sweep)
- `RSA_BITS = DSA_BITS = DH_BITS = 2048`, `ECC_CURVE = 'P-256'`
- `ASSUMED_CORE_WATTS = 15.0` (Energy Consumption estimate only — no physical power meter was used in this study)

Because random key generation and system-level timing noise are involved, exact millisecond values will vary slightly run-to-run and machine-to-machine; the *relative* ordering and the *magnitude* of the differences reported in this documentation should reproduce consistently.
