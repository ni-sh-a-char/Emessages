# Emessages  
**Image Steganography Tool for Encoding and Decoding a picture with an encrypted message**  

---  

## Table of Contents  

| Section | Description |
|---------|-------------|
| **[Installation](#installation)** | How to get the library up and running on your machine |
| **[Quick Start (CLI)](#quick-start-cli)** | One‑line commands to encode/decode images |
| **[Usage (Python API)](#usage-python-api)** | Detailed walkthrough of the public API |
| **[API Reference](#api-reference)** | Full description of classes, functions and exceptions |
| **[Examples](#examples)** | Real‑world snippets – CLI and Python code |
| **[Contributing](#contributing)** | How to help improve Emessages |
| **[License](#license)** | Open‑source license information |

---  

## Installation  

Emessages is pure Python (≥3.9) and has only a few lightweight dependencies. You can install it in three ways:

### 1️⃣ From PyPI (recommended)

```bash
pip install emessages
```

> **Tip** – Use a virtual environment (`venv` or `conda`) to keep your project dependencies isolated.

### 2️⃣ From Conda‑Forge  

```bash
conda install -c conda-forge emessages
```

### 3️⃣ From source (development mode)

```bash
# Clone the repo
git clone https://github.com/your‑org/Emessages.git
cd Emessages

# Install in editable mode
pip install -e .
```

#### System requirements  

| Requirement | Minimum version |
|-------------|-----------------|
| Python | 3.9 |
| Pillow (PIL) | 9.0 |
| cryptography | 42.0 |
| tqdm (optional – progress bars) | 4.66 |
| numpy (optional – for bulk operations) | 1.24 |

All required packages are listed in `requirements.txt` and will be installed automatically by the commands above.

---  

## Quick Start – Command‑Line Interface (CLI)  

Emessages ships with a convenient CLI (`emessages`) that can be used directly from the terminal.

### Encode a message into an image  

```bash
emessages encode \
    --input-image  path/to/cover.png \
    --output-image path/to/stego.png \
    --message "The secret is hidden!" \
    --password myStrongPass123
```

| Argument | Description |
|----------|-------------|
| `--input-image` | Path to the *cover* image (PNG, BMP, or JPEG). |
| `--output-image` | Destination path for the stego‑image. |
| `--message` | Plain‑text message to embed. |
| `--password` | Passphrase used to encrypt the message (AES‑256‑GCM). |
| `--bits-per-channel` | *(optional)* Number of LSBs to use per colour channel (default: `2`). |
| `--compress` | *(optional flag)* Compress the message with zlib before encryption. |
| `--quiet` | *(optional flag)* Suppress progress bars. |

### Decode a message from an image  

```bash
emessages decode \
    --input-image path/to/stego.png \
    --password myStrongPass123
```

The decoded (and decrypted) message is printed to `stdout`. Add `--output-file secret.txt` to write it to a file instead.

### Help  

```bash
emessages --help          # top‑level help
emessages encode --help   # encode‑specific help
emessages decode --help   # decode‑specific help
```

---  

## Usage – Python API  

If you prefer to embed steganography directly in your Python code, Emessages provides a clean, well‑typed API.

```python
from emessages import StegoEncoder, StegoDecoder, StegoError
```

### 1️⃣ Encode  

```python
encoder = StegoEncoder(
    bits_per_channel=2,          # how many LSBs per colour channel
    compress=True,              # compress before encryption (default: True)
    progress=True               # show tqdm progress bar (optional)
)

stego_image = encoder.encode(
    cover_image_path="cover.png",
    message="Top‑secret payload",
    password="myStrongPass123"
)

# Save the resulting image
stego_image.save("stego.png")
```

### 2️⃣ Decode  

```python
decoder = StegoDecoder(progress=True)

message = decoder.decode(
    stego_image_path="stego.png",
    password="myStrongPass123"
)

print("Recovered message:", message)
```

### 3️⃣ Low‑level helpers  

| Function | Description |
|----------|-------------|
| `emessages.utils.encrypt(message: bytes, password: str) -> bytes` | AES‑256‑GCM encryption (adds a random 12‑byte nonce). |
| `emessages.utils.decrypt(ciphertext: bytes, password: str) -> bytes` | Decrypts data encrypted with `encrypt`. |
| `emessages.utils.compress(data: bytes) -> bytes` | Zlib compression (level 9). |
| `emessages.utils.decompress(data: bytes) -> bytes` | Reverse of `compress`. |
| `emessages.utils.lsb_embed(image: Image, payload: bytes, bits: int) -> Image` | Core LSB embedding routine (private). |
| `emessages.utils.lsb_extract(image: Image, payload_len: int, bits: int) -> bytes` | Core LSB extraction routine (private). |

---  

## API Reference  

Below is the public API surface. All classes and functions are exported from `emessages/__init__.py`.

### `class StegoEncoder(bits_per_channel: int = 2, compress: bool = True, progress: bool = False)`

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `bits_per_channel` | `int` | `2` | Number of least‑significant bits used per colour channel (1‑4). |
| `compress` | `bool` | `True` | Whether to compress the message before encryption. |
| `progress` | `bool` | `False` | Show a `tqdm` progress bar while processing large images. |

#### Methods  

| Method | Signature | Returns | Raises |
|--------|-----------|---------|--------|
| `encode` | `encode(cover_image_path: str, message: str, password: str, output_path: Optional[str] = None) -> PIL.Image.Image` | Pillow `Image` object containing the stego data. If `output_path` is supplied the image is also saved to disk. | `StegoError`, `FileNotFoundError`, `ValueError` |
| `max_capacity` | `max_capacity(image_path: str) -> int` | Maximum number of **bytes** that can be stored in the given image with the current `bits_per_channel`. | `FileNotFoundError` |

### `class StegoDecoder(progress: bool = False)`

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `progress` | `bool` | `False` | Show a `tqdm` progress bar while extracting. |

#### Methods  

| Method | Signature | Returns | Raises |
|--------|-----------|---------|--------|
| `decode` | `decode(stego_image_path: str, password: str) -> str` | The recovered plaintext message. | `StegoError`, `FileNotFoundError`, `ValueError` |
| `detect_capacity` | `detect_capacity(stego_image_path: str) -> int` | Number of **bytes** that were embedded (extracted from the header). | `FileNotFoundError` |

### `class StegoError(Exception)`

Base exception for all runtime errors raised by Emessages (e.g., wrong password, corrupted payload, insufficient capacity).

### Exceptions hierarchy  

```
Exception
 └─ StegoError
     ├─ CapacityError   # not enough room in the cover image
     ├─ EncryptionError # decryption failed (wrong password or tampered data)
     └─ CorruptionError # payload header or checksum invalid
```

---  

## Examples  

### Example 1 – Encode & decode via CLI  

```bash
# Encode
emessages encode \
    --input-image  samples/landscape.jpg \
    --output-image samples/landscape_secret.png \
    --message "Meet at 02:00 UTC on the rooftop." \
    --password "S3cr3t!"

# Decode
emessages decode \
    --input-image samples/landscape_secret.png \
    --password "S3cr3t!"
```

Output:

```
Meet at 02:00 UTC on the rooftop.
```

### Example 2 – Encode & decode in a script  

```python
