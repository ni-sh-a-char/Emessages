# Emessages  
**Image Steganography Tool for Encoding and Decoding a Picture with Encrypted Message**  

---  

## Table of Contents
1. [Overview](#overview)  
2. [Installation](#installation)  
3. [Quick Start (CLI)](#quick-start-cli)  
4. [Python API](#python-api)  
5. [Configuration & Advanced Options](#configuration--advanced-options)  
6. [Examples](#examples)  
7. [Testing](#testing)  
8. [Contributing](#contributing)  
9. [License](#license)  

---  

## Overview
**Emessages** is a lightweight, pure‑Python library and command‑line utility that lets you hide an encrypted text message inside an image (PNG, BMP, or lossless WebP). The tool uses:

* **AES‑256‑GCM** for symmetric encryption (authenticated encryption).  
* **Least Significant Bit (LSB)** steganography with optional **pixel‑shuffle** to reduce visual patterns.  
* Automatic capacity detection and error handling.  

The repository contains:

| Component | Description |
|-----------|-------------|
| `emessages/` | Core library (`encode`, `decode`, `encrypt`, `decrypt`, utilities). |
| `cli/` | Command‑line interface (`emessages-cli`). |
| `tests/` | PyTest suite covering edge cases and performance. |
| `docs/` | This documentation (generated with MkDocs). |
| `examples/` | Ready‑to‑run Jupyter notebooks and scripts. |

---  

## Installation  

### Prerequisites
| Requirement | Minimum version |
|-------------|-----------------|
| Python | 3.9+ |
| pip | 22.0+ |
| Pillow (PIL) | 9.0+ (installed automatically) |
| cryptography | 42.0+ (installed automatically) |

> **Note**: Emessages works on Windows, macOS, and Linux. No external C libraries are required.

### 1. Install from PyPI (recommended)

```bash
pip install emessages
```

### 2. Install from source (latest development)

```bash
# Clone the repo
git clone https://github.com/your-org/Emessages.git
cd Emessages

# Create a virtual environment (optional but recommended)
python -m venv .venv
source .venv/bin/activate   # .venv\Scripts\activate on Windows

# Install the package in editable mode
pip install -e .
```

### 3. Verify installation

```bash
python -c "import emessages; print(emessages.__version__)"
# Expected output: e.g. 1.2.3
```

---  

## Quick Start (CLI)

The command‑line tool is called **`emessages-cli`**. It provides two primary sub‑commands:

| Sub‑command | Description |
|-------------|-------------|
| `encode`    | Hide an encrypted message inside an image. |
| `decode`    | Extract and decrypt a hidden message from an image. |

### Encode a message

```bash
emessages-cli encode \
    --input-image  samples/input.png \
    --output-image samples/encoded.png \
    --message      "Top secret: launch at 0400 UTC." \
    --password     "S3cureP@ssw0rd!" \
    --shuffle      # optional: random pixel shuffle for extra obfuscation
```

**Options**

| Flag | Description |
|------|-------------|
| `--input-image` | Path to the cover image (PNG/BMP/WebP). |
| `--output-image`| Destination for the stego‑image. |
| `--message`     | Plain‑text message (or `--message-file` for a file). |
| `--password`    | Passphrase used to derive the AES‑256 key (PBKDF2‑SHA256). |
| `--iterations`  | PBKDF2 iterations (default: 200 000). |
| `--shuffle`     | Enable pixel‑shuffle (adds a random permutation). |
| `--verbose`     | Print detailed progress information. |

### Decode a message

```bash
emessages-cli decode \
    --input-image samples/encoded.png \
    --password    "S3cureP@ssw0rd!" \
    --output-file  decoded_message.txt
```

**Options**

| Flag | Description |
|------|-------------|
| `--input-image` | Path to the stego‑image. |
| `--password`    | Same passphrase used during encoding. |
| `--output-file` | Write the recovered plaintext to a file (omit to print to stdout). |
| `--verbose`     | Show detailed extraction steps. |

---  

## Python API  

The library can be used programmatically. All public functions are exported from `emessages.core`.

```python
from emessages import (
    encode_image,
    decode_image,
    encrypt_message,
    decrypt_message,
    generate_key,
    utils,
)
```

### Core Functions  

| Function | Signature | Description |
|----------|-----------|-------------|
| `encode_image` | `encode_image(cover_path: str, output_path: str, message: bytes, password: str, *, shuffle: bool = False, iterations: int = 200_000) -> None` | Encrypts `message` with a key derived from `password`, optionally shuffles pixels, and writes the stego‑image. |
| `decode_image` | `decode_image(stego_path: str, password: str, *, iterations: int = 200_000) -> bytes` | Reads the hidden payload, verifies the authentication tag, and returns the plaintext. |
| `encrypt_message` | `encrypt_message(message: bytes, password: str, *, iterations: int = 200_000) -> Tuple[bytes, bytes, bytes]` | Returns `(nonce, ciphertext, tag)`. |
| `decrypt_message` | `decrypt_message(nonce: bytes, ciphertext: bytes, tag: bytes, password: str, *, iterations: int = 200_000) -> bytes` | Decrypts and authenticates the payload. |
| `generate_key` | `generate_key(password: str, *, salt: bytes = None, iterations: int = 200_000) -> Tuple[bytes, bytes]` | Derives a 32‑byte AES‑256 key using PBKDF2‑SHA256. Returns `(key, salt)`. |
| `utils.calculate_capacity` | `utils.calculate_capacity(image: Image.Image) -> int` | Returns the maximum number of bytes that can be stored in the given image (LSB‑only). |
| `utils.shuffle_pixels` / `utils.unshuffle_pixels` | `shuffle_pixels(img: Image.Image, seed: int) -> Image.Image` | Deterministic pixel permutation based on the derived key. |

### Example – Using the API

```python
from pathlib import Path
from emessages import encode_image, decode_image

# ----------------------------------------------------------------------
# 1️⃣  Encode
# ----------------------------------------------------------------------
cover = Path("samples/input.png")
stego = Path("samples/encoded.png")
secret = b"Mission accomplished. Meet at 22:00."

encode_image(
    cover_path=str(cover),
    output_path=str(stego),
    message=secret,
    password="My$tr0ngP@ss!",
    shuffle=True,               # optional
    iterations=300_000,
)

print("✅ Message hidden in", stego)

# ----------------------------------------------------------------------
# 2️⃣  Decode
# ----------------------------------------------------------------------
recovered = decode_image(
    stego_path=str(stego),
    password="My$tr0ngP@ss!",
    iterations=300_000,
)

print("🔓 Recovered message:", recovered.decode())
```

### Error Handling  

All public functions raise `EmessagesError` (sub‑classed from `Exception`). Common subclasses:

| Exception | When raised |
|-----------|-------------|
| `CapacityError` | Message larger than image can hold. |
| `AuthenticationError` | Decryption tag verification failed (wrong password or tampered data). |
| `UnsupportedFormatError` | Input image format not supported. |
| `InvalidParameterError` | Bad arguments (e.g., negative iterations). |

```python
from emessages.exceptions import CapacityError, AuthenticationError

try:
    encode_image(...)
except CapacityError as e:
    print("❗️", e)
```

---  

## Configuration & Advanced Options  

| Option | Description | Default |
|--------|-------------|---------|
| `iterations` | PBKDF2 iteration count (higher = slower but more secure). | `200_000` |
| `shuffle` | Enable deterministic pixel shuffle (adds ~0.5 % overhead). | `False` |
| `seed` | Custom seed for shuffle (normally derived from the key). | `None` |
| `compression` | For PNG, set `compress_level` (0‑9). | `6` |
| `metadata` | Optional JSON metadata stored in the image’s `tEXt` chunk. | `None` |

You can