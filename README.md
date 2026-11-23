# Simple-Python-encryption-program

A simple, secure file and folder encryption utility built on **AES-256-GCM** with **PBKDF2-HMAC-SHA256** key derivation. Includes both a command-line interface (`crypt_tool.py`) and an interactive menu-driven launcher (`launcher.py`).

---

## Table of Contents

- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Usage — `crypt_tool.py`](#usage--crypt_toolpy)
  - [Commands](#commands)
  - [Examples](#examples)
- [Usage — `launcher.py`](#usage--launcherpy)
  - [Interactive Mode](#interactive-mode)
  - [Non-Interactive Mode](#non-interactive-mode)
- [File Format](#file-format)
- [Security Notes](#security-notes)
- [How It Works](#how-it-works)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)
- [License](#license)

---

## Features

- 🔐 **Strong encryption** — AES-256-GCM (authenticated encryption; detects tampering and wrong passwords).
- 🔑 **Secure key derivation** — PBKDF2-HMAC-SHA256 with 480,000 iterations and a random 16-byte salt per file.
- 📄 **Single-file encryption** — encrypt/decrypt individual files in place.
- 📁 **Recursive folder encryption** — encrypt or decrypt every file in a directory tree.
- 📦 **Archive mode** — pack an entire folder into a single encrypted `.tar.gz.enc` file.
- 🧭 **Interactive launcher** — friendly menu with hidden password input and confirmation prompts.
- 🗑️ **Optional source deletion** — remove originals after a successful operation (`--delete`).
- 🏷️ **Self-describing format** — encrypted files carry a magic header (`ENCF`) and version byte for future compatibility.

---

## Requirements

- **Python 3.10+** (the launcher uses `str | None` type hints)
- The [`cryptography`](https://pypi.org/project/cryptography/) package

---

## Installation

1. **Clone or download** the two scripts into the same directory:

   ```
   your-folder/
   ├── crypt_tool.py
   └── launcher.py
   ```

2. **Install the dependency:**

   ```bash
   pip install cryptography
   ```

   Or, if you prefer an isolated environment:

   ```bash
   python -m venv .venv
   source .venv/bin/activate      # Windows: .venv\Scripts\activate
   pip install cryptography
   ```

3. **(Optional) Make the scripts executable** on Linux/macOS:

   ```bash
   chmod +x crypt_tool.py launcher.py
   ```

---

## Quick Start

```bash
# Encrypt a file
python crypt_tool.py encrypt secret.pdf "MyStr0ngPass!"

# Decrypt it
python crypt_tool.py decrypt secret.pdf.enc "MyStr0ngPass!"

# Or use the friendly interactive menu
python launcher.py
```

---

## Usage — `crypt_tool.py`

### Commands

| Command | Description |
|---|---|
| `encrypt <path> <password>` | Encrypt a single file → `<name>.enc` |
| `decrypt <path> <password>` | Decrypt a single `.enc` file |
| `encrypt-folder <folder> <password>` | Recursively encrypt every file in a folder (in place) |
| `decrypt-folder <folder> <password>` | Recursively decrypt every `.enc` file in a folder |
| `encrypt-archive <folder> <password>` | Tar + gzip a folder, then encrypt into `<folder>.tar.gz.enc` |
| `decrypt-archive <archive> <password> [--out DIR]` | Decrypt and extract an archive |

All `encrypt`/`decrypt`/`*-folder` commands accept an optional `--delete` flag to remove the source file(s) after a successful operation.

### Examples

```bash
# Encrypt a single file
python crypt_tool.py encrypt secret.pdf "MyStr0ngPass!"

# Encrypt and delete the original
python crypt_tool.py encrypt secret.pdf "MyStr0ngPass!" --delete

# Decrypt it
python crypt_tool.py decrypt secret.pdf.enc "MyStr0ngPass!"

# Encrypt every file inside a folder (recursive, in place)
python crypt_tool.py encrypt-folder ./myfolder "password123" --delete

# Decrypt them all back
python crypt_tool.py decrypt-folder ./myfolder "password123"

# Pack an entire folder into ONE encrypted archive (recommended)
python crypt_tool.py encrypt-archive ./project "password123"
#  -> project.tar.gz.enc

# Decrypt and extract the archive
python crypt_tool.py decrypt-archive project.tar.gz.enc "password123" --out ./restored
```

> **Tip:** Always quote passwords containing special characters (`!`, `$`, `&`, spaces, etc.).

---

## Usage — `launcher.py`

The launcher is an interactive wrapper that prompts you for the operation, path, and password — no need to remember arguments. It must be placed in the same directory as `crypt_tool.py`, or you can point it at the tool with `--tool`.

### Interactive Mode

```bash
python launcher.py
```

You'll see a menu:

```
=== Encryption Tool ===
  1) Encrypt a single file
  2) Decrypt a single .enc file
  3) Encrypt every file in a folder (recursive)
  4) Decrypt every .enc file in a folder (recursive)
  q) Quit

Choose an option:
```

The launcher will then:

1. Ask for the file/folder path (and re-prompt if it doesn't exist).
2. Warn if you're decrypting a file without a `.enc` extension.
3. Ask whether to delete the source after success.
4. Prompt for the password **twice** (hidden input) and confirm they match.
5. Show a summary and ask you to confirm before running.

### Non-Interactive Mode

You can bypass the menu and supply arguments directly:

```bash
python launcher.py --command encrypt --path ./secret.pdf
python launcher.py --command encrypt-folder --path ./myfolder --delete
```

| Flag | Description |
|---|---|
| `--tool PATH` | Path to `crypt_tool.py` (default: look next to the launcher) |
| `--command CMD` | One of `encrypt`, `decrypt`, `encrypt-folder`, `decrypt-folder` |
| `--path PATH` | Target file or folder (required with `--command`) |
| `--delete` | Delete source files after success |

In this mode the password is still prompted interactively (hidden input).

---

## File Format

Each encrypted file is a self-contained binary blob with the following layout:

```
┌────────┬─────────┬───────────┬───────────┬──────────────────┐
│ MAGIC  │ VERSION │   SALT    │   NONCE   │    CIPHERTEXT    │
│ 4 B    │  1 B    │  16 B     │   12 B    │   variable       │
│ "ENCF" │  0x01   │  random   │  random   │  AES-256-GCM     │
└────────┴─────────┴───────────┴───────────┴──────────────────┘
```

| Field | Size | Purpose |
|---|---|---|
| `MAGIC` | 4 bytes | Identifies the file as this tool's format (`ENCF`) |
| `VERSION` | 1 byte | Format version (`0x01`) for future upgrades |
| `SALT` | 16 bytes | Random salt for PBKDF2 |
| `NONCE` | 12 bytes | Random GCM nonce (never reused per key) |
| `CIPHERTEXT` | variable | AES-256-GCM output + 16-byte auth tag |

A fresh **salt** and **nonce** are generated for every file, so encrypting the same file twice produces different output.

---

## Security Notes

- **Authenticated encryption (AEAD):** AES-GCM verifies integrity — if the file is tampered with or the password is wrong, decryption fails rather than returning garbage.
- **Strong KDF:** 480,000 PBKDF2 iterations make brute-force attacks on weak passwords expensive. Still, **use a strong, unique password** — the tool cannot protect you from a weak passphrase.
- **Random salt & nonce per file:** Prevents rainbow-table attacks and nonce reuse.
- **Password on the command line:** When using `crypt_tool.py` directly, the password is passed as an argument, which is visible to other users via `ps` or shell history. For sensitive use, prefer the launcher (which uses hidden `getpass` input) or clear your shell history afterward.
- **`--delete` is destructive:** The original file is removed with a normal `unlink()` — it is **not** securely wiped from disk. On SSDs and journaling filesystems, data may persist. For high-security needs, use full-disk encryption.
- **No key management:** There is no recovery mechanism. **If you lose the password, the data is unrecoverable.**

---

## How It Works

### Key derivation

For each file:

1. Generate a random 16-byte salt and 12-byte nonce.
2. Derive a 32-byte key: `PBKDF2-HMAC-SHA256(password, salt, iterations=480000, dklen=32)`.
3. Encrypt: `AESGCM(key).encrypt(nonce, plaintext, None)`.
4. Write `MAGIC | VERSION | SALT | NONCE | CIPHERTEXT`.

### Decryption

1. Read and validate the header (magic, version, minimum length).
2. Re-derive the key from the stored salt and the supplied password.
3. Decrypt and verify the GCM tag. A failure means either a wrong password or a corrupted file.
4. Write the plaintext to the original filename (`.enc` stripped).

### Folder mode

- `encrypt-folder` walks the tree with `Path.rglob("*")` and encrypts each file individually, **skipping** files already ending in `.enc`.
- `decrypt-folder` walks the tree looking for `*.enc` files and decrypts each.
- Errors on individual files are reported but don't stop the batch.

### Archive mode

- `encrypt-archive` builds a `.tar.gz` in memory (`io.BytesIO`), then encrypts the whole blob into a single `.tar.gz.enc` file — convenient for backups and moving folders around.
- `decrypt-archive` decrypts and extracts to the archive's directory (or `--out`).

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `ModuleNotFoundError: No module named 'cryptography'` | Run `pip install cryptography` (in the same Python environment you're using). |
| `Decryption failed (wrong password or corrupted file)` | Wrong password, or the file was modified/truncated. Verify the password and that the file wasn't altered in transit. |
| `Bad magic header (not our format)` | The file isn't encrypted with this tool, or the header was damaged. |
| `Unsupported version: b'...'` | The file was produced by a newer/incompatible format version. |
| `File too short / corrupt` | Truncated file — smaller than the header. |
| `Not an encrypted file (missing .enc)` | `decrypt` requires the `.enc` extension. Rename the file if needed. |
| `ERROR: could not find 'crypt_tool.py'` (launcher) | Put `launcher.py` next to `crypt_tool.py`, or pass `--tool /path/to/crypt_tool.py`. |
| Password works locally but not after transfer | Ensure the file was transferred in **binary** mode (e.g., `scp`, `rsync`, not plain FTP ASCII). |

---

## FAQ

**Q: Can I recover my files if I forget the password?**
No. There is no backdoor or key escrow. The password is the only key.

**Q: Does encrypting the same file twice produce the same output?**
No — a fresh random salt and nonce are generated each time, so the ciphertext differs.

**Q: Is it safe to encrypt a folder in place with `--delete`?**
The tool writes the `.enc` file first, then deletes the original. If a file fails mid-way, the original is left intact (the error is reported and the batch continues). Still, **test on a copy first** and keep backups.

**Q: What's the difference between `encrypt-folder` and `encrypt-archive`?**
`encrypt-folder` produces one `.enc` file per original file, preserving the folder structure. `encrypt-archive` produces a **single** `.tar.gz.enc` file containing the whole folder — better for backups or sharing.

**Q: Can I use this for large files?**
Yes, but note that both files and archives are read fully into memory. For multi-gigabyte files, consider a streaming tool.

**Q: Is the password stored anywhere?**
No. It exists only in memory during the operation and is passed as a command-line argument to `crypt_tool.py` (when using the launcher). See the [Security Notes](#security-notes).

---

## License

This project is provided as-is for personal and educational use. Add your preferred license here (e.g., MIT) if you plan to distribute it.
