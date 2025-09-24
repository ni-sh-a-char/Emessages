# Emessages  
**Image Steganography Tool for Encoding and Decoding a picture with encrypted message**  

---  

## Table of Contents  

| Section | Description |
|---------|-------------|
| **[Installation](#installation)** | How to get Emessages up and running on your machine. |
| **[Quick Start (CLI)](#quick-start-cli)** | One‑line commands to encode/decode images. |
| **[Usage (Python API)](#usage-python-api)** | Detailed description of the public API. |
| **[API Reference](#api-reference)** | Full reference for modules, classes, and functions. |
| **[Examples](#examples)** | Real‑world snippets – CLI and Python – showing typical workflows. |
| **[Configuration & Encryption](#configuration--encryption)** | How to choose encryption algorithms, key handling, and security tips. |
| **[Testing & Contributing](#testing--contributing)** | Run the test‑suite and help improve Emessages. |
| **[License](#license)** | Open‑source licensing information. |

---  

## Installation  

Emessages is pure‑Python and works on **Python 3.9+**. It has no external binary dependencies – only standard cryptography and image‑processing libraries.

### 1. Prerequisites  

| Dependency | Reason |
|------------|--------|
| `python >= 3.9` | Language runtime |
| `pip` | Package manager |
| `virtualenv` (optional) | Isolate the environment |
| `libjpeg` / `libpng` (system) | Required only if you plan to use the optional Pillow‑based image loaders on some Linux distros (most wheels bundle the needed binaries). |

### 2. Install via PyPI (recommended)

```bash
# Create an isolated environment (optional but recommended)
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# Install the latest stable release
pip install emessages
```

### 3. Install from source (latest development)

```bash
# Clone the repo
git clone https://github.com/your-org/Emessages.git
cd Emessages

# Install in editable mode (useful for development)
pip install -e .
```

### 4. Verify the installation  

```bash
$ emessages --version
Emessages 2.3.1
```

If you see the version string, you’re ready to go!

---  

## Quick Start (CLI)

Emessages ships with a convenient command‑line interface called `emessages`.  
All commands share a common `--key` argument for the symmetric encryption key (a 32‑byte base64 string).  

### 1. Generate a random encryption key  

```bash
# Generates a 256‑bit key and prints it in base64
emessages genkey
# Example output:
#   Your encryption key (keep it safe!):
#   bXlTZWNyZXRrZXlGb3JTdGFnZ2luZzEyMw==
```

> **⚠️** Store the key securely – losing it means you cannot recover hidden messages.

### 2. Encode (hide) a message  

```bash
emessages encode \
    --input  original.png \
    --output secret.png \
    --message "Meet me at 23:00 on the rooftop." \
    --key bXlTZWNyZXRrZXlGb3JTdGFnZ2luZzEyMw==
```

**Flags**

| Flag | Description |
|------|-------------|
| `--input` | Path to the cover image (PNG, JPEG, BMP, …). |
| `--output` | Destination file that will contain the hidden payload. |
| `--message` | Plain‑text message to embed. |
| `--key` | Base64‑encoded 256‑bit symmetric key (AES‑GCM). |
| `--bits-per-channel` *(optional)* | Number of LSBs to use per colour channel (default = 1). |
| `--compress` *(optional)* | Compress the message with zlib before encryption (default = True). |

### 3. Decode (extract) a message  

```bash
emessages decode \
    --input secret.png \
    --key bXlTZWNyZXRrZXlGb3JTdGFnZ2luZzEyMw==
```

The tool prints the recovered plaintext to `stdout`. Use `--out-file` to write it to a file.

### 4. Help & Full CLI reference  

```bash
emessages --help
```

---  

## Usage (Python API)

You can also embed/extract messages programmatically. The public API lives in the `emessages` package.

```python
from emessages import StegoEngine, CryptoHelper

# 1️⃣  Load or generate a symmetric key (bytes)
key = CryptoHelper.generate_key()          # 32‑byte key (AES‑256‑GCM)
# Or decode a stored base64 key:
# key = CryptoHelper.key_from_b64("bXlTZWNyZXRrZXlGb3JTdGFnZ2luZzEyMw==")

# 2️⃣  Create a StegoEngine instance (you can reuse it)
engine = StegoEngine(key=key, bits_per_channel=1)

# 3️⃣  Encode a message
engine.encode(
    cover_path="original.png",
    output_path="secret.png",
    message="The launch code is 0420."
)

# 4️⃣  Decode a message
recovered = engine.decode("secret.png")
print("Recovered:", recovered)
```

### Core Classes  

| Class | Purpose |
|-------|---------|
| `StegoEngine` | High‑level façade for encoding/decoding images. Handles LSB manipulation, optional compression, and encryption. |
| `CryptoHelper` | Thin wrapper around **cryptography** primitives (AES‑GCM, key generation, base64 helpers). |
| `ImageAdapter` (internal) | Abstracts Pillow / OpenCV image handling; you can plug your own loader if needed. |

### Important Parameters  

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `key` | `bytes` | **required** | 32‑byte secret used for AES‑GCM encryption. |
| `bits_per_channel` | `int` | `1` | How many least‑significant bits of each colour channel are used for payload. Higher values increase capacity but degrade visual quality. |
| `compress` | `bool` | `True` | Run `zlib.compress` on the plaintext before encryption (recommended). |
| `max_capacity` (property) | `int` | – | Maximum number of bytes that can be hidden in the current cover image with the chosen `bits_per_channel`. |

---  

## API Reference  

Below is the **public** API that you should rely on. All internal helpers are deliberately undocumented to allow future refactoring.

### `emessages.CryptoHelper`

| Method | Signature | Description |
|--------|-----------|-------------|
| `generate_key()` | `() -> bytes` | Returns a fresh 32‑byte random key (AES‑256‑GCM). |
| `key_to_b64(key: bytes) -> str` | `bytes` → `str` | Encode a raw key to a URL‑safe base64 string (for storage). |
| `key_from_b64(b64_key: str) -> bytes` | `str` → `bytes` | Decode a base64 key back to raw bytes. |
| `encrypt(plaintext: bytes, key: bytes) -> Tuple[bytes, bytes]` | `(bytes, bytes)` → `(ciphertext, nonce)` | AES‑GCM encryption; returns ciphertext and the 12‑byte nonce. |
| `decrypt(ciphertext: bytes, nonce: bytes, key: bytes) -> bytes` | `(bytes, bytes, bytes)` → `bytes` | Reverse of `encrypt`. Raises `InvalidTag` on tampering. |

### `emessages.StegoEngine`

| Method | Signature | Description |
|--------|-----------|-------------|
| `__init__(key: bytes, bits_per_channel: int = 1, compress: bool = True)` | – | Initialise the engine. |
| `encode(cover_path: str, output_path: str, message: str, **kwargs)` | – | Hide `message` inside `cover_path` and write to `output_path`. Raises `StegoError` on capacity overflow. |
| `decode(stego_path: str) -> str` | – | Extract and return the hidden plaintext from `stego_path`. |
| `max_capacity(cover_path