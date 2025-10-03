# Emessages  
**Image Steganography Tool for Encoding and Decoding a picture with encrypted message.**  

---  

## Table of Contents  

| # | Section |
|---|---------|
| 1 | [Overview](#overview) |
| 2 | [Features](#features) |
| 3 | [Prerequisites](#prerequisites) |
| 4 | [Installation](#installation) |
| 5 | [Quick‑Start (CLI)](#quick-start-cli) |
| 6 | [Usage (Command‑Line Interface)](#usage-cli) |
| 7 | [Python API](#python-api) |
| 8 | [API Reference](#api-reference) |
| 9 | [Examples](#examples) |
|10 | [Testing](#testing) |
|11 | [Contributing](#contributing) |
|12 | [License](#license) |
|13 | [Acknowledgements](#acknowledgements) |

---  

## 1. Overview  

**Emessages** is a lightweight, pure‑Python library + CLI that lets you hide an arbitrary text message inside an image using **least‑significant‑bit (LSB) steganography**.  
The message is first encrypted with **AES‑256‑GCM** (derived from a user‑supplied password) and then embedded pixel‑by‑pixel.  

* Encode → `input.png` + secret → `output.png` (no visible artefacts)  
* Decode → `output.png` + same password → original secret  

The tool works with PNG, BMP, and any lossless format supported by Pillow.  

---  

## 2. Features  

| ✅ | Feature |
|---|---------|
| ✅ | **AES‑256‑GCM** encryption (authenticated, no padding) |
| ✅ | Automatic key‑derivation with **PBKDF2‑HMAC‑SHA256** (default 200 k iterations) |
| ✅ | Supports PNG, BMP, TIFF (any Pillow‑compatible lossless format) |
| ✅ | CLI (`emessages encode|decode`) and Python API |
| ✅ | Optional **compression** (zlib) before encryption to reduce payload size |
| ✅ | Payload size validation (max ≈ (Width × Height × 3 / 8) bytes) |
| ✅ | Customizable LSB depth (1‑4 bits per channel) |
| ✅ | Unit‑tested (pytest) and type‑annotated (PEP‑484) |
| ✅ | Cross‑platform (Linux, macOS, Windows) |
| ✅ | Zero‑runtime external binaries – pure Python + Pillow + cryptography |

---  

## 3. Prerequisites  

| Item | Minimum version |
|------|-----------------|
| Python | **3.8** (3.9+ recommended) |
| pip | 21.0+ |
| OS | Any OS with a working Python interpreter |

---  

## 4. Installation  

### 4.1 From PyPI (recommended)

```bash
# Create a virtual environment (optional but recommended)
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# Install the package
pip install emessages
```

### 4.2 From source (latest development)

```bash
# Clone the repository
git clone https://github.com/your‑org/Emessages.git
cd Emessages

# Install in editable mode (useful for development)
pip install -e .
```

### 4.3 Optional dependencies  

| Feature | Extra name | Packages installed |
|---------|------------|--------------------|
| Compression (zlib) | `compression` | `zstandard` (optional, faster) |
| Image format support beyond Pillow defaults | `formats` | `pillow[webp]`, `pillow[tiff]` |
| Testing | `dev` | `pytest`, `pytest‑cov`, `black`, `ruff` |

Install extras with:

```bash
pip install emessages[compression,formats,dev]
```

---  

## 5. Quick‑Start (CLI)

```bash
# Encode a secret message
emessages encode \
    -i samples/landscape.png \
    -o samples/landscape_secret.png \
    -m "The treasure is buried under the oak tree." \
    -p "S3cr3tP@ssw0rd!"

# Decode the secret message
emessages decode \
    -i samples/landscape_secret.png \
    -p "S3cr3tP@ssw0rd!"
```

Both commands will print a short status line. The decode command prints the recovered plaintext to **stdout** (or `--out-file` to write to a file).

---  

## 6. Usage (Command‑Line Interface)

### 6.1 General options  

| Flag | Description |
|------|-------------|
| `-h`, `--help` | Show help and exit |
| `-v`, `--version` | Print version number |
| `-q`, `--quiet` | Suppress non‑essential output |
| `--log-level LEVEL` | Set logging level (`DEBUG`, `INFO`, `WARNING`, `ERROR`) |

### 6.2 `encode` sub‑command  

```bash
emessages encode \
    -i INPUT_IMAGE \
    -o OUTPUT_IMAGE \
    -m MESSAGE \
    [options]
```

| Option | Required? | Description |
|--------|-----------|-------------|
| `-i`, `--input` | ✅ | Path to the cover image (PNG/BMP/TIFF) |
| `-o`, `--output` | ✅ | Path where the stego‑image will be saved |
| `-m`, `--message` | ✅ | Plain‑text message to hide (use `--msg-file` for large payloads) |
| `--msg-file` | ❌ | Path to a text file containing the message (mutually exclusive with `--message`) |
| `-p`, `--password` | ✅ | Password used for encryption (will be prompted if omitted) |
| `-c`, `--compress` | ❌ | Enable zlib compression before encryption (`store_true`) |
| `-b`, `--bits-per-channel` | ❌ | Number of LSBs to use (1‑4, default = 1) |
| `--salt` | ❌ | Hex‑encoded salt (if you want deterministic KDF; otherwise generated randomly) |
| `--iterations` | ❌ | PBKDF2 iteration count (default = 200 000) |
| `--metadata` | ❌ | Optional JSON string that will be stored alongside the payload (e.g., `{ "author":"Alice" }`) |

**Example – using a file for the message and 2‑bit depth**

```bash
emessages encode \
    -i secret.png \
    -o secret_embedded.png \
    --msg-file my_secret.txt \
    -p "MyStrongPassword" \
    -b 2 \
    --compress
```

### 6.3 `decode` sub‑command  

```bash
emessages decode \
    -i STEGO_IMAGE \
    [options]
```

| Option | Required? | Description |
|--------|-----------|-------------|
| `-i`, `--input` | ✅ | Path to the image that contains the hidden message |
| `-p`, `--password` | ✅ | Password used for decryption (prompted if omitted) |
| `-o`, `--out-file` | ❌ | Write the recovered message to this file (otherwise printed to stdout) |
| `--bits-per-channel` | ❌ | Number of LSBs that were used during encoding (default = 1) |
| `--metadata` | ❌ | If present, prints the stored JSON metadata to stdout |
| `--no-verify` | ❌ | Skip authentication tag verification (dangerous – only for forensic analysis) |

**Example – decode and write to a file**

```bash
emessages decode \
    -i secret_embedded.png \
    -p "MyStrongPassword" \
    -o recovered.txt
```

---  

## 7. Python API  

You can embed Emessages directly in your own Python code. The public API lives in the `emessages` package.

```python
from emessages import