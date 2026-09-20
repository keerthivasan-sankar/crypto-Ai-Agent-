# cryptoflex — AI/developer reference

This README is a condensed, task-oriented reference for using the
`cryptoflex` Python package correctly. It's meant to pair with
`SKILL.md` in this folder. It intentionally skips the project's design
rationale, academic framing, and research history — see the repository's
own `README.md`, `docs/THREAT_MODEL.md`, and the figshare paper for that.

- Repo: `keerthivasan-sankar/crypto_flex`
- Package: `cryptoflex` (PyPI)
- Version this reference matches: **0.5.3**
- License: MIT
- Language: Python 3.10+

## Install

```bash
pip install cryptoflex          # X25519 only, no compile step
pip install cryptoflex[pqc]     # + ML-KEM via liboqs-python (compiles liboqs on first import)
```

Fast/deterministic dev & CI path (skips the liboqs build entirely):

```bash
export CRYPTOFLEX_DISABLE_PQC=1
```

## What it does, in one paragraph

`cryptoflex` picks, at runtime, which key-exchange algorithm(s) an
application should use — classical X25519, or X25519 hybridized with
post-quantum ML-KEM — based only on local signals (what's installed, a
caller-given speed/security preference, and a bundled offline risk
table). It then gives you a high-level `encrypt`/`decrypt` pair that
does AES-256-GCM under the derived key, with the full header
authenticated as AEAD associated data. No network calls anywhere.

## Public API surface (import from `cryptoflex`)

| Name | Use for |
|---|---|
| `PolicyEngine`, `Constraint` | choosing a security profile at runtime |
| `establish_keys()` | generate one identity's keypair(s) |
| `encrypt(bundle, plaintext)` / `decrypt(private_handles, blob, *, min_profile=None)` | **recommended** file/blob encryption |
| `encrypt_stream()` / `decrypt_stream()` | chunked AEAD for large files |
| `migrate()` / `migrate_file()` / `migrate_stream()` | re-encrypt existing ciphertext under a new profile |
| `ephemeral_encrypt()` / `ephemeral_decrypt()` / `WireMessage` | per-message ephemeral keys for messaging |
| `export_keyset_bytes()` / `import_keyset_bytes()` / `serialize_public_bundle()` / `deserialize_public_bundle()` | password-protected keystore, public bundle (de)serialization |
| `derive_root_key()` / `recover_root_key()` | low-level key derivation (advanced callers only) |
| `KeySet`, `PublicBundle`, `DerivedRoot`, `PolicyDecision` | data classes returned by the above |
| `CryptoflexHeader`, `HeaderParseError` | on-disk/wire header format |
| `DecryptionError`, `DowngradeError` | the only exceptions that cross the API boundary |
| `PROFILES`, `SecurityProfile`, `get_profile()` | inspect the fixed named profiles |
| `zeroize()` | best-effort in-memory key wipe helper |

CLI entry point: `cryptoflex {keygen,encrypt,decrypt,migrate,info}` — see
`SKILL.md` §G for exact flags.

## Security profiles

| Profile ID | Sources | Quantum-safe | strength_level |
|---|---|---|---|
| `classical_only` | X25519 | No | 0 |
| `hybrid_standard` | X25519 + ML-KEM-768 | Yes | 1 |
| `hybrid_high` | X25519 + ML-KEM-1024 | Yes | 2 |

`PolicyEngine.decide(constraint)` returns a `PolicyDecision(profile,
reason, degraded, min_accepted_profile)`. Always check/log `.degraded`
and `.reason` rather than assuming the ideal profile was used.

## Minimal end-to-end example

```python
from cryptoflex import PolicyEngine, Constraint, establish_keys, encrypt, decrypt

engine = PolicyEngine()
keyset = establish_keys(engine, constraint=Constraint.BALANCED)

blob = encrypt(keyset.public_bundle, b"hello")
assert decrypt(keyset.private_handles, blob) == b"hello"
```

For the password-protected keystore, streaming, ephemeral messaging, and
migration examples, see `SKILL.md`.

## Error handling contract

- `decrypt()`, `recover_root_key()`, `ephemeral_decrypt()` raise exactly
  one exception type on any crypto failure: `DecryptionError("decryption
  failed")`. There is no way — by design — to distinguish "wrong key"
  from "tampered ciphertext" from "corrupt header" from this exception.
  Do not write code that tries to.
- `DowngradeError` (subclass of `DecryptionError`) is raised *before* any
  cryptographic operation, only when `min_profile` is set and the
  ciphertext's profile is weaker. This one is meaningfully distinguishable
  and safe to branch on.
- `establish_keys(..., require_quantum_safe=True)` raises `RuntimeError`
  instead of silently falling back to `classical_only`.

## Environment variables

| Var | Effect |
|---|---|
| `CRYPTOFLEX_DISABLE_PQC=1` | `PQCSource` reports unavailable immediately; `PolicyEngine` falls back to `classical_only` with `degraded=True` instead of compiling `liboqs` |
| `CRYPTOFLEX_PASSWORD` | password source for CLI `keygen`/`decrypt`/`migrate`, instead of `--password` or an interactive prompt |

## File formats produced

| Extension | Format |
|---|---|
| `.cflx` | encrypted blob/file: `CFLX` header + AES-256-GCM ciphertext (or streamed DATA/FINAL frames) |
| `.cflk` | password-encrypted keystore: `CFLA` (Argon2id) or `CFLK` (Scrypt) magic + salt + nonce + AES-256-GCM ciphertext |
| `.bundle.json` | plaintext `PublicBundle` — profile ID + public keys, safe to share |



## What can be built on this

See "What you can build with this" in `SKILL.md` for a fuller list.
Short version: local/offline file encryption tools, password-protected
vaults, USB-bound encryption utilities, P2P/LAN messaging with ephemeral
keys, large-archive streaming encryption, and crypto-agility migration
jobs that re-encrypt a corpus of files when the recommended profile
changes. Not a fit for network protocol design, live threat-feed-driven
policy, full forward-secrecy messaging (e.g. a Double-Ratchet-style
protocol), or regulated production data requiring certified/audited
cryptography.
