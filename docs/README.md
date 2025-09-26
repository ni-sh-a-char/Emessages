# Emessages  
**Image Steganography Tool for Encoding and Decoding a picture with encrypted message.**  

---  

## Table of Contents  

| Section | Description |
|---------|-------------|
| **[Installation](#installation)** | How to get Emessages up and running on your machine. |
| **[Quick Start](#quick-start)** | One‑line commands to encode/decode a message. |
| **[Usage](#usage)** | Detailed CLI options, configuration, and runtime flags. |
| **[API Documentation](#api-documentation)** | Python functions, classes, and type hints for developers. |
| **[Examples](#examples)** | Real‑world scenarios – from the command line and from Python code. |
| **[Troubleshooting & FAQ](#troubleshooting--faq)** | Common pitfalls and their solutions. |
| **[Contributing](#contributing)** | How to help improve Emessages. |
| **[License](#license)** | Open‑source license information. |

---  

## Installation  

Emessages is a pure‑Python package that works on Windows, macOS, and Linux. It requires **Python 3.9+**.

### 1️⃣ From PyPI (recommended)

```bash
# Create a virtual environment (optional but recommended)
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# Install the latest stable release
pip install emessages
```

### 2️⃣ From source (latest development version)

```bash
# Clone the repository
git clone https://github.com/your-org/Emessages.git
cd Emessages

# Install in editable mode with all optional dependencies
pip install -e ".[dev]"
```

> **Note** – The optional `dev` extras install testing tools (`pytest`, `tox`) and documentation helpers (`mkdocs`).  

### 3️⃣ System dependencies  

| Dependency | Why it’s needed | Install (apt) | Install (brew) |
|------------|----------------|---------------|----------------|
| `libjpeg` / `libpng` | Pillow needs the underlying image libraries. | `sudo apt-get install libjpeg-dev libpng-dev` | `brew install libjpeg libpng` |
| `ffmpeg` (optional) | Allows embedding messages into video frames. | `sudo apt-get install ffmpeg` | `brew install ffmpeg` |

If you only work with PNG/BMP images, Pillow will work out‑of‑the‑box on most platforms.

---  

## Quick Start  

```bash
# Encode a secret message into an image (output will be saved as out.png)
emessages encode -i secret.png -o out.png -m "The password is 42!"

# Decode the hidden message
emessages decode -i out.png
```

Both commands will automatically encrypt the message using AES‑256‑GCM with a password you provide (`--password`). If you omit `--password`, Emessages will prompt you securely.

---  

## Usage  

Emessages ships a single entry‑point script: **`emessages`**.  
Run `emessages --help` for a full list of sub‑commands and options.

### 1️⃣ Encode  

```bash
emessages encode \
    -i <input_image> \
    -o <output_image> \
    -m "<message>" \
    [--password <pwd>] \
    [--algorithm aes256gcm|chacha20] \
    [--bits-per-channel <1|2|4|8>] \
    [--compress] \
    [--metadata "<key=value>" ...]
```

| Option | Description | Default |
|--------|-------------|---------|
| `-i, --input` | Path to the cover image (PNG, BMP, JPEG). | – |
| `-o, --output` | Path where the stego‑image will be written. | – |
| `-m, --message` | Plain‑text message to hide. Use `@file.txt` to read from a file. | – |
| `--password` | Password for symmetric encryption. If omitted, you’ll be prompted. | Prompt |
| `--algorithm` | Encryption algorithm (`aes256gcm` is the default). | `aes256gcm` |
| `--bits-per-channel` | Number of LSBs used per colour channel (1‑8). Higher values increase capacity but reduce visual quality. | `2` |
| `--compress` | Run zlib compression on the plaintext before encryption (recommended for long messages). | Off |
| `--metadata` | Optional key‑value pairs stored as PNG text chunks (e.g., `author=John`). | – |
| `--dry-run` | Validate capacity and parameters without writing a file. | Off |
| `-v, --verbose` | Show detailed progress information. | Off |

#### Capacity Check  

```bash
emessages capacity -i secret.png --bits-per-channel 2
```

The command prints the maximum number of bytes that can be stored with the chosen LSB depth.

### 2️⃣ Decode  

```bash
emessages decode \
    -i <stego_image> \
    [--password <pwd>] \
    [--algorithm aes256gcm|chacha20] \
    [--output <output_file>] \
    [--verbose]
```

| Option | Description |
|--------|-------------|
| `-i, --input` | Path to the image that contains a hidden message. |
| `--password` | Password used during encoding. If omitted, you’ll be prompted. |
| `--algorithm` | Must match the algorithm used for encoding. |
| `--output` | Write the recovered plaintext to a file instead of stdout. |
| `-v, --verbose` | Show intermediate steps (e.g., extracted ciphertext size). |

### 3️⃣ Miscellaneous Sub‑commands  

| Sub‑command | Purpose |
|-------------|---------|
| `capacity` | Compute how many bytes can be hidden in a given image. |
| `info` | Print image metadata, colour mode, and steganographic parameters (if present). |
| `version` | Show the installed Emessages version. |

---  

## API Documentation  

> **Tip** – The public API lives in the `emessages` package. Import it in your Python code:

```python
from emessages import StegoEngine, EncryptionScheme, StegoError
```

### Core Classes  

| Class | Description |
|-------|-------------|
| **`StegoEngine`** | High‑level façade that bundles encoding/decoding, encryption, and image handling. |
| **`EncryptionScheme`** | Abstract base class for encryption algorithms (`AES256GCM`, `ChaCha20Poly1305`). |
| **`StegoError`** | Custom exception hierarchy (`CapacityError`, `DecryptionError`, `InvalidImageError`). |

#### `StegoEngine`  

```python
class StegoEngine:
    def __init__(
        self,
        bits_per_channel: int = 2,
        algorithm: EncryptionScheme = AES256GCM(),
        compress: bool = False,
        metadata: dict[str, str] | None = None,
    ) -> None:
        ...
```

| Method | Signature | Description |
|--------|-----------|-------------|
| **`encode`** | `encode(cover: Path | Image.Image, message: bytes, password: str) -> Image.Image` | Returns a new Pillow `Image` with the encrypted payload hidden. |
| **`decode`** | `decode(stego: Path | Image.Image, password: str) -> bytes` | Extracts, decrypts, and returns the original plaintext. |
| **`capacity`** | `capacity(image: Path | Image.Image) -> int` | Maximum number of **bytes** that can be stored with the current `bits_per_channel`. |
| **`set_metadata`** | `set_metadata(image: Image.Image, meta: dict[str, str]) -> Image.Image` | Stores arbitrary key‑value pairs in PNG text chunks (or EXIF for JPEG). |
| **`get_metadata`** | `get_metadata(image: Path | Image.Image) -> dict[str, str]` | Retrieves stored metadata. |

#### Encryption Schemes  

```python
class AES256GCM(EncryptionScheme):
    def encrypt(self, plaintext: bytes, password: str) -> bytes: ...
    def decrypt(self, ciphertext: bytes, password: str) -> bytes: ...

class ChaCha20Poly1305(EncryptionScheme):
    ...
```

Both schemes use **PBKDF2‑HMAC‑SHA256** (default 200 000 iterations) to derive a 256‑bit key from the password, and they prepend a 12‑byte nonce to the ciphertext.

### Helper Functions  

| Function | Signature