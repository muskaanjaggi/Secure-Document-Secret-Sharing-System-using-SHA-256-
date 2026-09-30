# 🔐 Secure Document Secret Sharing (DSS) System using SHA-256 Share Authentication

A Python implementation of a **Document Secret Sharing (DSS)** system built on **Shamir's (k, n) Secret Sharing Scheme**, with **per-share SHA-256 authentication** and a **Tkinter GUI**. Any document (PDF, DOCX, PPTX, image, etc.) is split into `n` independent shares such that any `k` of them reconstruct the original file bit-for-bit, while fewer than `k` shares reveal no information about it.

---

## 📑 Table of Contents

1. [Overview](#-overview)
2. [Key Features](#-key-features)
3. [How It Works](#-how-it-works)
4. [System Architecture](#-system-architecture)
5. [Tech Stack](#-tech-stack)
6. [Project Structure](#-project-structure)
7. [Installation](#-installation)
8. [Usage](#-usage)
9. [Data Formats](#-data-formats)
10. [Experimental Results](#-experimental-results)
11. [Security Model & Limitations](#-security-model--limitations)
12. [Future Work](#-future-work)
13. [License](#-license)


---

## 🧭 Overview

Sensitive documents are vulnerable when stored on devices or cloud services, or when transmitted over a network. Traditional encryption depends on protecting a single key or file. **Secret sharing** removes that single point of failure: the document is split into multiple shares, each meaningless on its own, and only a predefined threshold of shares can recover it.

Inspired by **Visual Secret Sharing (VSS)** and Visual Cryptography, this project applies secret sharing at the **document level** and adds **hash-based share authentication**, so tampered or corrupted shares are detected and rejected *before* reconstruction.

**Problem addressed:**
> *How can documents be securely split, shared, and reconstructed with integrity verification using secret sharing schemes?*

**Motivation:** Ransomware and data-theft incidents (e.g., the 2016 infection of three Indian banks' computer systems) show the danger of keeping documents in one place. DSS ensures that compromising a single share — or even `k-1` shares — exposes nothing.

---

## ✨ Key Features

- **(k, n) threshold secret sharing** — any `k` of `n` shares reconstruct the document (e.g., (2,3), (3,5), (4,6), (5,8)).
- **Information-theoretic confidentiality** — fewer than `k` shares reveal no information about the secret (Shamir's scheme).
- **Format-agnostic** — works at the binary level, so it supports PDF, DOCX, PPTX, images, and any other file.
- **SHA-256 share authentication** — every share is hashed at generation time; hashes are verified before reconstruction.
- **Tamper detection** — modified or invalid shares are automatically rejected.
- **Insufficient-share detection** — warns the user when fewer than `k` valid shares are supplied.
- **Bit-exact reconstruction** — 100% reconstruction accuracy verified by SHA-256 comparison on all tested file types.
- **Desktop GUI** — Tkinter interface for share generation and document reconstruction.
- **Low overhead** — roughly 4–5.5% share size increase and fast processing (see [results](#-experimental-results)).

---

## ⚙️ How It Works

### 1. Share Generation

1. **Read** the input file in binary mode.
2. **Chunk** it into fixed-size **132-byte** blocks; pad the last block if needed.
3. **Select a large prime** `p` to use as the modulus for all finite-field arithmetic.
4. For each chunk:
   - Convert the chunk to an integer `S` (the secret).
   - Build a random polynomial of degree `k − 1`:

     ```
     f(x) = S + a₁x + a₂x² + … + a_{k-1}x^{k-1}   (mod p)
     ```
     where `S` is the constant term and `a₁…a_{k-1}` are random coefficients.
   - Evaluate `f(x)` at `n` distinct non-zero points `x = 1 … n` to produce the share points `(x, y)`.
5. **Write** each participant's share to a text file containing `(x, y)` pairs.
6. **Compute** the SHA-256 hash of each share file and store it in `hashes.json`.
7. **Save metadata** (prime, chunk size, hashes) needed for reconstruction.

### 2. Share Reconstruction

1. **Load** the shares from the selected folder.
2. **Verify** each share's SHA-256 hash against `hashes.json`. Shares that fail are rejected.
3. Confirm that at least **`k` valid shares** remain; otherwise warn and abort.
4. For each chunk, apply **Lagrange interpolation** at `x = 0` over `GF(p)` to recover the constant term (the original chunk integer):

   ```
   S = f(0) = Σᵢ yᵢ · Π_{j≠i} ( xⱼ / (xⱼ − xᵢ) )   (mod p)
   ```
5. **Convert** each integer back to bytes, strip padding, and concatenate.
6. **Save** the reconstructed document to the user-chosen output path.
7. *(Optional verification)* Compare the SHA-256 of the reconstructed file with the original.

---

## 🏗️ System Architecture

```
┌────────────────┐   ┌──────────────────┐   ┌─────────────────────┐   ┌────────────────────┐
│ Input Document │ → │ Document Chunking│ → │ Secret Sharing      │ → │ Share Hashing      │
│                │   │ (132-byte blocks)│   │ (Shamir's k,n)      │   │ (SHA-256)          │
└────────────────┘   └──────────────────┘   └─────────────────────┘   └─────────┬──────────┘
                                                                                │
┌────────────────┐   ┌──────────────────┐   ┌─────────────────────┐   ┌─────────▼──────────┐
│ Final Document │ ← │ Hash Checking    │ ← │ GUI-based           │ ← │ Share Transmission │
│                │   │                  │   │ Reconstruction      │   │ / Storage          │
└────────────────┘   └──────────────────┘   └─────────────────────┘   └────────────────────┘
```

The implementation has three components:

| Component | Responsibility |
|---|---|
| **Share Generator** | Chunking, prime selection, polynomial generation/evaluation, share file output, SHA-256 hashing, metadata storage. |
| **Share Reconstructor** | Loading shares, hash verification, Lagrange interpolation, byte reassembly, writing the output file. |
| **GUI Controller** | Tkinter front end: choose input file, `n`, `k`, output/share directories; run generation or reconstruction. |

### GUI Workflow

**Generate shares**
1. Browse and select the input document.
2. Specify the number of shares (`n`) and the threshold (`k`).
3. Choose an output directory for shares and metadata.
4. Click **Generate** — the GUI calls the share-generator script.

**Reconstruct document**
1. Select the directory containing the shares and metadata.
2. Enter the same `n` and `k` used during generation.
3. Choose where to save the reconstructed document.
4. Click **Reconstruct** — the GUI calls the reconstructor script.

---

## 🧰 Tech Stack

- **Language:** Python 3.8+
- **GUI:** Tkinter (bundled with most Python distributions)
- **Hashing:** `hashlib` (SHA-256)
- **Randomness:** `secrets` (recommended for cryptographic coefficient generation)
- **Big-integer math / primes:** Python's native arbitrary-precision integers (optionally `sympy` / `Crypto.Util.number` for prime generation)
- **Metadata:** `json`

---

## 📁 Project Structure

> The exact file names may differ in your codebase — adjust to match your repository.

```
document-secret-sharing/
├── share_generator.py        # Splits a document into n shares + hashes.json
├── share_reconstructor.py    # Verifies shares and rebuilds the document
├── gui.py                    # Tkinter GUI controller
├── utils/
│   ├── shamir.py             # Polynomial creation, evaluation, Lagrange interpolation
│   └── hashing.py            # SHA-256 helpers
├── shares/                   # Generated output (example)
│   ├── share_1.txt
│   ├── share_2.txt
│   ├── ...
│   └── hashes.json           # SHA-256 digest per share + metadata
├── docs/
│   └── paper.pdf             # Research paper
├── requirements.txt
└── README.md
```

---

## 🚀 Installation

```bash
# 1. Clone the repository
git clone https://github.com/<your-username>/document-secret-sharing.git
cd document-secret-sharing

# 2. (Recommended) Create a virtual environment
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# 3. Install dependencies (if any)
pip install -r requirements.txt
```

**Tkinter note:** on some Linux distributions Tkinter must be installed separately:

```bash
sudo apt-get install python3-tk
```

---

## 💻 Usage

### Option A — GUI

```bash
python gui.py
```

Then use **Generate Shares** and **Reconstruct Document** as described in [GUI Workflow](#gui-workflow).

### Option B — Command line (if you expose CLI entry points)

```bash
# Generate 5 shares, any 3 of which can reconstruct the document
python share_generator.py --input report.pdf --n 5 --k 3 --out ./shares

# Reconstruct from any 3+ valid shares
python share_reconstructor.py --shares ./shares --k 3 --out ./report_restored.pdf
```

### Option C — Programmatic sketch

```python
from share_generator import generate_shares
from share_reconstructor import reconstruct_document

# Split
generate_shares("report.pdf", n=5, k=3, output_dir="shares/")

# Distribute share_1.txt ... share_5.txt to different holders / locations.
# Later, collect any >= 3 of them into a folder and reconstruct:
reconstruct_document("collected_shares/", k=3, output_path="report_restored.pdf")
```

### Recommended Parameter Choices

| Scheme (k, n) | Good for |
|---|---|
| (2, 3) | Fastest; minimal compute, basic redundancy. |
| (3, 5) | Best balance of speed, fault tolerance, and security. |
| (4, 6), (5, 8) | Higher security/fault tolerance at greater computational cost. |

---

## 🗂️ Data Formats

**Share file** (text): a list of `(x, y)` pairs — one pair per document chunk — where `x` is the participant's evaluation point and `y` is the polynomial value (mod `p`).

```
(1, 83749238478...)
(1, 19283746573...)
...
```

**`hashes.json`** (metadata): stores the prime modulus, chunk size, and the SHA-256 digest of each share file.

```json
{
  "prime": "<large prime modulus>",
  "chunk_size": 132,
  "n": 5,
  "k": 3,
  "hashes": {
    "share_1.txt": "e3b0c44298fc1c149afbf4c8996fb924...",
    "share_2.txt": "9f86d081884c7d659a2feaa0c55ad015..."
  }
}
```

> The schema above is illustrative; align it with your actual implementation.

---

## 📊 Experimental Results

Tests were run on DOCX, PPTX, PDF, and image files across multiple (k, n) configurations.

### Reconstruction Accuracy

| Document Type | Identical to Original? | SHA-256 Match? |
|---|---|---|
| PDF | ✅ Yes | ✅ Matched |
| Image | ✅ Yes | ✅ Matched |
| Word Document | ✅ Yes | ✅ Matched |
| PowerPoint | ✅ Yes | ✅ Matched |

### Share Size Overhead

| File | Original | Avg. Share Size | Overhead |
|---|---|---|---|
| DOCX | 2.00 MB | 2.11 MB | 5.5% |
| PPTX | 3.00 MB | 3.13 MB | 4.3% |
| PDF | 1.00 MB | 1.04 MB | 4.0% |

### Execution Time (seconds)

| File Size | (2,3) Gen | (2,3) Recon | (3,5) Gen | (3,5) Recon | (4,6) Gen | (4,6) Recon | (5,8) Gen | (5,8) Recon |
|---|---|---|---|---|---|---|---|---|
| 0.5 MB | 0.19 | 0.32 | 1.24 | 0.15 | 0.29 | 0.21 | 0.85 | 0.21 |
| 1 MB | 0.50 | 0.21 | 0.39 | 0.24 | 0.43 | 0.36 | 0.62 | 0.35 |
| 2 MB | 0.32 | 0.32 | 0.86 | 0.38 | 0.60 | 0.70 | 0.81 | 0.66 |
| 3 MB | 0.51 | 0.46 | 0.62 | 0.57 | 0.81 | 1.03 | 0.99 | 0.91 |
| 5 MB | 0.62 | 0.70 | 1.18 | 0.85 | 1.24 | 1.59 | 1.59 | 1.48 |
| 7 MB | 0.93 | 0.92 | 1.49 | 1.16 | 1.48 | 2.22 | 2.51 | 2.00 |
| 10 MB | 1.37 | 1.30 | 1.80 | 1.61 | 2.43 | 3.04 | 3.52 | 2.91 |

### Key Takeaways

- **100% reconstruction accuracy** across all tested configurations.
- **Near-linear scaling** of generation and reconstruction time with file size.
- **(2,3)** is the fastest scheme; **(5,8)** is the slowest, since the threshold `k` influences performance more than `n`.
- Processing completes in **under ~4 seconds for files up to 10 MB**.
- All **tampered shares were rejected** via SHA-256 mismatch detection in testing.
- Fixed-cost effects (e.g., the (3,5) generation time at 0.5 MB) appear at small file sizes.

### Security Validation Test Cases

- ✅ Detection and warning when an insufficient number of shares is provided.
- ✅ Automatic rejection of tampered/invalid shares via hash mismatch.

---

## 🛡️ Security Model & Limitations

**What the system provides**

- **Confidentiality:** With fewer than `k` shares, an attacker learns nothing about the document (Shamir's scheme is information-theoretically secure when the coefficients are uniformly random over `GF(p)`).
- **Integrity / tamper detection:** Modified shares are detected through SHA-256 verification.
- **Availability / fault tolerance:** Up to `n − k` shares can be lost without losing the document.

**Important considerations for real-world use**

- **Protect the hash metadata.** A plain SHA-256 digest stored in an unauthenticated `hashes.json` detects accidental corruption and naive tampering, but an attacker who can modify both a share *and* `hashes.json` can defeat it. For stronger guarantees, use an HMAC or digital signature over the metadata, or store hashes through a separate trusted channel.
- **Use a cryptographically secure RNG** (`secrets` module) for polynomial coefficients — never `random`.
- **Choose the prime carefully.** The prime modulus must be larger than the largest possible chunk integer (for 132-byte chunks, greater than 2¹⁰⁵⁶); otherwise chunk values wrap around and cannot be recovered.
- **Shares are not compressed or encrypted.** Each share is roughly the size of the original document (plus encoding overhead), so total storage is about `n ×` the document size.
- **Shamir is not verifiable on its own.** SHA-256 authenticates shares *after* generation but does not protect against a malicious dealer who distributes inconsistent shares (this would require a full Verifiable Secret Sharing scheme, e.g., Feldman/Pedersen VSS).
- **Keep `k`, `n`, and the prime consistent** between generation and reconstruction.
- **Distribute shares over separate channels/locations;** storing all shares together defeats the purpose.

---

## 🔭 Future Work

- Authenticated metadata (HMAC / digital signatures) and dealer-verifiable schemes (Feldman/Pedersen VSS).
- Dynamic adaptation to document size, format, and bandwidth constraints.
- Streaming / chunk-parallel processing (multiprocessing) for large files.
- Compact binary share encoding to reduce overhead.
- Integration with cloud storage and blockchain-based integrity registries.
- Hybrid approaches combining encryption + secret sharing for lower storage cost.
- Multi-party collaboration and access-control features.

---

## 📜 License

Add your preferred license here (e.g., MIT, Apache-2.0). The accompanying paper is © Grenze Scientific Society, 2026.

---

⭐ If you find this project useful, consider starring the repository!
