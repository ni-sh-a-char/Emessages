# Emessages  
**Image Steganography Tool for Encoding and Decoding a picture with encrypted message.**  

---  

## Table of Contents
1. [Overview](#overview)  
2. [Prerequisites](#prerequisites)  
3. [Installation](#installation)  
   - [From PyPI (recommended)](#install-from-pypi)  
   - [From source (development)](#install-from-source)  
4. [Quick Start (CLI)](#quick-start-cli)  
5. [Python API](#python-api)  
   - [Core Classes & Functions](#core-classes-functions)  
   - [Error handling](#error-handling)  
6. [Examples](#examples)  
   - [CLI examples](#cli-examples)  
   - [Python examples](#python-examples)  
7. [Configuration & Advanced Options](#configuration)  
8. [Testing](#testing)  
9. [Contributing](#contributing)  
10. [License](#license)  
11. [Changelog](#changelog)  

---  

## Overview <a name="overview"></a>

`Emessages` is a lightweight, pure‑Python library and command‑line utility that lets you **hide an encrypted text message inside an image** (PNG, BMP, or lossless WebP) and later retrieve it.  

Key features:

| Feature | Description |
|---------|-------------|
| **AES‑256 encryption** of the payload (optional) |
| **Least‑Significant‑Bit (LSB) steganography** with optional random pixel distribution |
| **Support for PNG, BMP, and lossless WebP** (any format that stores raw pixel data) |
| **CLI** (`emessages`) for quick one‑liners |
| **Python API** for integration into other projects |
| **Automatic integrity check** (SHA‑256 hash) to detect tampering |
| **Cross‑platform** (Windows, macOS, Linux) |
| **Zero external binary dependencies** – only pure‑Python packages |

---  

## Prerequisites <a name="prerequisites"></a>

| Requirement | Minimum version |
|-------------|-----------------|
| Python | 3.9+ |
| pip | 22.0+ |
| Pillow (PIL fork) | 10.0+ (installed automatically) |
| cryptography | 42.0+ (installed automatically) |

> **Note**: On Windows, you may need the Visual C++ Build Tools for the `cryptography` wheel if a pre‑compiled wheel isn’t available for your platform.  

---  

## Installation <a name="installation"></a>

### 1. Install from PyPI (recommended) <a name="install-from-pypi"></a>

```bash
# Create a virtual environment (optional but recommended)
python -m venv .venv
source .venv/bin/activate   # .venv\Scripts\activate on Windows

# Install the package
pip install emessages
```

The command installs:

* `emessages` – the CLI entry point  
* `emessages.core` – the Python API  
* `Pillow` and `cryptography` as dependencies  

### 2. Install from source (development) <a name="install-from-source"></a>

```bash
# Clone the repository
git clone https://github.com/your-org/Emessages.git
cd Emessages

# Optional: install development extras (testing, linting)
python -m venv .venv
source .venv/bin/activate
pip install -e .[dev]

# Verify the installation
emessages --version
```

> **Tip** – The `-e` flag installs the package in “editable” mode, so changes you make to the source are reflected immediately without reinstalling.

---  

## Quick Start (CLI) <a name="quick-start-cli"></a>

The CLI provides two primary commands: `encode` and `decode`.

```bash
# Encode a message
emessages encode \
    --input  original.png \
    --output secret.png \
    --message "Meet me at 23:00 GMT" \
    --password "S3cureP@ssw0rd!" \
    --bits-per-channel 2 \
    --random-seed 12345

# Decode a message
emessages decode \
    --input secret.png \
    --password "S3cureP@ssw0rd!" \
    --bits-per-channel 2
```

### Common flags

| Flag | Description |
|------|-------------|
| `--input` | Path to the source image (encode) or stego‑image (decode). |
| `--output` | Path where the stego‑image will be written (encode only). |
| `--message` | Plain‑text message to embed. Use `--message-file` to read from a file. |
| `--password` | Passphrase used to derive the AES‑256 key (optional – if omitted the payload is stored **plain**). |
| `--bits-per-channel` | Number of LSBs per colour channel (1‑4). Higher values increase capacity but reduce visual quality. |
| `--random-seed` | Integer seed for pseudo‑random pixel distribution (default: `None` → sequential). |
| `--verbose` | Print detailed progress information. |
| `--dry-run` | Validate capacity and parameters without writing an output file. |

---  

## Python API <a name="python-api"></a>

The library is organized around two high‑level classes: `StegoEncoder` and `StegoDecoder`. Both accept a `StegoConfig` object that centralises all options.

```python
from emessages import StegoEncoder, StegoDecoder, StegoConfig
```

### Core Classes & Functions <a name="core-classes-functions"></a>

| Class / Function | Purpose | Typical usage |
|------------------|---------|---------------|
| **`StegoConfig`** | Holds configuration (bits per channel, password, random seed, etc.) | `cfg = StegoConfig(bits_per_channel=2, password="my‑key")` |
| **`StegoEncoder.encode(image_path, message, output_path, config=None)`** | Embed a message into `image_path` and write to `output_path`. Returns a `StegoResult`. | `result = StegoEncoder.encode("cat.png", "Hello", "cat_secret.png", cfg)` |
| **`StegoDecoder.decode(image_path, config=None)`** | Extract a hidden message from `image_path`. Returns a `StegoResult`. | `result = StegoDecoder.decode("cat_secret.png", cfg)` |
| **`StegoResult`** | Simple data container (`payload`, `success`, `error`, `metadata`). | `if result.success: print(result.payload)` |
| **`encrypt(payload: bytes, password: str) -> bytes`** | Internal helper – AES‑256‑GCM encryption. | `ciphertext = encrypt(b"msg", "pwd")` |
| **`decrypt(ciphertext: bytes, password: str) -> bytes`** | Internal helper – AES‑256‑GCM decryption. | `plaintext = decrypt(ciphertext, "pwd")` |
| **`capacity(image_path, bits_per_channel=1) -> int`** | Compute maximum number of bytes that can be stored in the given image. | `max_bytes = capacity("cat.png", 2)` |

#### Example: Using the API

```python
from emessages import StegoEncoder, StegoDecoder, StegoConfig

# 1️⃣ Create a configuration
cfg = StegoConfig(
    bits_per_channel=2,
    password="S3cureP@ssw0rd!",
    random_seed=42,
    verbose=True,
)

# 2️⃣ Encode
encoder = StegoEncoder()
result = encoder.encode(
    image_path="samples/landscape.png",
    message="The launch code is 0077.",
    output_path="samples/landscape_stego.png",
    config=cfg,
)

if result.success:
    print("✅ Message encoded successfully!")
else:
    print("❌ Encoding failed:", result.error)

# 3️⃣ Decode
decoder = StegoDecoder()
decoded = decoder.decode(
    image_path="samples/landscape_stego.png",
    config=cfg,
)

if decoded.success:
    print("🔓 Decoded message:", decoded.payload.decode())
else:
    print("❌ Decoding failed:", decoded.error)
```

### Error handling <a name="error-handling"></a>

All public methods raise **`StegoError`** (sub‑class of `Exception`) for unrecoverable problems (e.g., unsupported format, insufficient capacity). The `StegoResult` object also contains an `error` attribute for non‑exception failures (e.g., wrong password).

```python
from emessages import StegoError

try:
    encoder.encode(...)
except StegoError as exc:
    print(f"Fatal error: {exc}")
```

---  

## Examples <a name="examples"></a>

### CLI Examples <a