---
name: cryptoflex
description: Use this skill whenever a task involves the cryptoflex Python package (local-first crypto-agility policy engine, hybrid X25519 + ML-KEM post-quantum encryption). Triggers include requests to encrypt/decrypt files or messages with quantum-safe or hybrid crypto, generate or manage cryptoflex keypairs/keystores, build a file-encryption or secure-messaging feature on top of cryptoflex, migrate ciphertext between security profiles, use the `cryptoflex` CLI, or write/review/debug code that imports the `cryptoflex` package. Also use it when asked to explain what cryptoflex is or does. Do not use for general/unrelated cryptography questions that don't involve this package.
license: MIT (matches the cryptoflex repository license)
---

# cryptoflex

## What this is

`cryptoflex` is a Python library, not a new cryptographic algorithm. It
orchestrates two already-audited primitives — classical X25519 (ECDH) and
post-quantum ML-KEM (via `liboqs`) — behind a small policy engine that
picks a key-exchange profile at runtime based only on local signals (what's
installed, a caller-given speed/security constraint, and a bundled offline
risk table). There is no network call, telemetry, or external service
anywhere in the library.

Use this skill to write correct integration code against `cryptoflex` —
not to invent your own use of its cryptographic primitives.

Package: `cryptoflex` on PyPI · repo: `keerthivasan-sankar/crypto_flex` ·
current version referenced by this skill: **0.5.3** · license: MIT.

## Installation

```bash
pip install cryptoflex          # classical-only (X25519), always works, no compile step
pip install cryptoflex[pqc]     # + post-quantum ML-KEM support via liboqs-python
```

The `[pqc]` extra compiles `liboqs` from source on first import unless a
prebuilt shared library is present — a multi-minute one-time build. Two
things matter for automated/CI contexts:

- To skip PQC and force `PolicyEngine` to gracefully fall back to
  `classical_only` (with `degraded=True` on the decision) instead of
  compiling anything, set `CRYPTOFLEX_DISABLE_PQC=1`.
- On Debian/Ubuntu, `sudo apt-get install -y liboqs-dev` before
  `pip install cryptoflex[pqc]` avoids the source build entirely.

Never suggest reimplementing X25519 or ML-KEM by hand, and never suggest a
different KEM/curve as a "simpler" substitute — the whole point of this
library is that it only orchestrates these two audited primitives.

## Core mental model

1. **Profiles** are fixed, named combinations of sources:
   | Profile ID | Sources | Quantum-safe |
   |---|---|---|
   | `classical_only` | X25519 | No |
   | `hybrid_standard` | X25519 + ML-KEM-768 | Yes |
   | `hybrid_high` | X25519 + ML-KEM-1024 | Yes |

   Hybrid profiles always keep the classical component even though it
   isn't quantum-safe on its own — it hedges against an undiscovered flaw
   in the newer PQC math (same design choice as Signal's PQXDH and
   Chrome's hybrid TLS).

2. **`PolicyEngine`** picks a profile for you, given a `Constraint`
   (`FAST`, `BALANCED`, `MAX_SECURITY`) and local availability. You almost
   never hand-pick a profile yourself — you go through the engine.

3. **One identity = one `establish_keys()` call.** This generates a fresh
   keypair per source in the chosen profile and returns a `KeySet`
   (`profile`, `public_bundle`, `private_handles`, `policy_decision`).
   `public_bundle` (a `PublicBundle`) is safe to share; `private_handles`
   must be kept secret (see Keystore below — never write raw private
   handles to disk).

4. **High-level AEAD API — use this by default:**
   `encrypt(bundle, plaintext) -> bytes` and
   `decrypt(private_handles, blob, *, min_profile=None) -> bytes`.
   Output is a self-contained blob: `header_bytes || AES-256-GCM(ciphertext
   || tag)`, with the full header (magic, version, profile ID, KEM
   ciphertexts, nonce) bound in as AEAD associated data. Any tampering with
   header or ciphertext fails the tag check.

   Only drop to the low-level `derive_root_key()` / `recover_root_key()`
   pair if you need to manage your own symmetric encryption — and if you
   do, you are responsible for binding the header as AEAD AAD and managing
   nonces yourself. Default to `encrypt`/`decrypt`.

5. **Uniform error boundary.** `decrypt()` and `recover_root_key()` catch
   every internal failure and raise a single `DecryptionError("decryption
   failed")` — callers never learn whether it was the classical source,
   the PQC source, or the AEAD tag that failed (this is deliberate,
   documented anti-oracle behavior). `DowngradeError` (a `DecryptionError`
   subclass) is the one distinguishable case, and it fires *before* any
   crypto if the ciphertext's profile is weaker than `min_profile`. Never
   write code that tries to inspect or recover more detail from a
   `DecryptionError` than "it failed" — that would defeat the point.

## Recommended workflows

### A. Encrypt/decrypt a file or blob between two parties (recommended path)

```python
from cryptoflex import PolicyEngine, Constraint, establish_keys, encrypt, decrypt

engine = PolicyEngine()

# Recipient generates an identity once, shares keyset.public_bundle
keyset = establish_keys(engine, constraint=Constraint.BALANCED)

# Sender encrypts to the recipient's public bundle
blob = encrypt(keyset.public_bundle, b"secret payload")

# Recipient decrypts with their own private handles
plaintext = decrypt(keyset.private_handles, blob)
assert plaintext == b"secret payload"
```

To refuse ever falling back to non-quantum-safe crypto, pass
`require_quantum_safe=True` to `establish_keys()` — this raises
`RuntimeError` instead of silently degrading if no PQC source is
available. Don't build your own "is PQC available" check; the engine
already reports `policy_decision.degraded` (bool) and `.reason` (str) —
log/surface these, don't swallow them.

### B. Enforce a minimum security profile on decrypt

```python
plaintext = decrypt(keyset.private_handles, blob, min_profile="hybrid_standard")
# raises DowngradeError if blob's header profile is weaker than hybrid_standard
```

Use this whenever ciphertext identity/provenance matters and a caller
should refuse to silently accept a downgraded (e.g. classical-only)
message — for example, verifying incoming files in a system that requires
PQC protection.

### C. Password-protected keystore (never persist raw private handles)

```python
from cryptoflex import export_keyset_bytes, import_keyset_bytes, serialize_public_bundle

# Save
open("id.keyset.cflk", "wb").write(export_keyset_bytes(keyset, "correct horse battery staple"))
open("id.bundle.json", "w").write(serialize_public_bundle(keyset.public_bundle))

# Load
keyset = import_keyset_bytes(open("id.keyset.cflk", "rb").read(), "correct horse battery staple")
```

`export_keyset_bytes` wraps the `KeySet` under AES-256-GCM with a password
key derived via Argon2id by default (`use_argon2=False` for Scrypt
legacy/compat). Never suggest storing `private_handles` as raw
pickled/JSON bytes outside this keystore format.

### D. Large files — streaming AEAD (don't load the whole file into memory)

```python
from cryptoflex import encrypt_stream, decrypt_stream

with open("big.iso", "rb") as fin, open("big.iso.cflx", "wb") as fout:
    encrypt_stream(keyset.public_bundle, fin, fout)

with open("big.iso.cflx", "rb") as fin, open("restored.iso", "wb") as fout:
    decrypt_stream(keyset.private_handles, fin, fout)
```

Use `encrypt_stream`/`decrypt_stream` for anything too large to hold in
memory as a single `bytes` object; use `encrypt`/`decrypt` otherwise.
Streaming uses per-chunk nonces derived from the base nonce plus a
sequence number, and a mandatory authenticated FINAL frame — don't
truncate a stream file, or decryption will correctly fail closed.

### E. Ephemeral / per-message keys for messaging (not file storage)

```python
from cryptoflex import ephemeral_encrypt, ephemeral_decrypt

msg = ephemeral_encrypt(keyset.public_bundle, b"hi")       # fresh ephemeral key per call
plaintext = ephemeral_decrypt(keyset.private_handles, msg)  # recipient's long-term handles
```

Use this for message-style workloads (fresh key per message) instead of
`encrypt`/`decrypt` when the caller talks about chat/messaging rather than
file storage. Note this gives ephemeral-key hygiene per message, not full
forward secrecy against a later compromise of the recipient's long-term
key — say so if asked.

### F. Crypto-agility migration (re-encrypt under a new profile/policy)

```python
from cryptoflex import migrate, migrate_file

new_blob = migrate(keyset.private_handles, old_blob, new_bundle)          # in-memory
migrate_file(keyset.private_handles, "old.cflx", "new.cflx", new_bundle,  # on disk
             stream=True)   # stream=True for large files
```

This is the mechanism for the library's headline feature: when the
bundled risk table or your policy changes (e.g. `classical_only` gets
deprecated), existing ciphertext doesn't need a bespoke rewrite — decrypt
under the old bundle, re-encrypt under a new one, offline.

### G. CLI, when the user wants a command-line tool rather than library code

```bash
cryptoflex keygen  --key id.cflk --bundle id.bundle.json [--kdf argon2id|scrypt] [--constraint fast|balanced|max_security]
cryptoflex encrypt --in file --out file.cflx --bundle recipient.bundle.json [--stream]
cryptoflex decrypt --in file.cflx --out file --key id.cflk [--min-profile hybrid_standard] [--stream]
cryptoflex migrate --in old.cflx --out new.cflx --key id.cflk --new-bundle new.bundle.json [--stream]
cryptoflex info    file.cflx    # inspect header metadata (profile, version) without decrypting
```

Passwords: `--password`, or the `CRYPTOFLEX_PASSWORD` env var, or omit
either for an interactive prompt.

## Things to always get right

- Default to `encrypt()`/`decrypt()` (or the streaming/ephemeral
  equivalents), not `derive_root_key()`/`recover_root_key()`, unless the
  user explicitly needs to manage their own symmetric layer.
- Never suggest branching logic on `DecryptionError`'s message or
  internals to figure out *why* decryption failed — that's the anti-oracle
  property the library is built around. Only `DowngradeError` is
  meaningfully distinguishable, and only because it's a pre-crypto policy
  check.
- Never suggest persisting `private_handles` outside `export_keyset_bytes`
  / the `.cflk` keystore format.
- Don't hardcode a profile ID when a `Constraint` + `PolicyEngine.decide()`
  would do — that defeats the crypto-agility purpose of the library.
  Hardcode a profile only when the user explicitly wants to pin one (e.g.
  interop with a fixed spec).
- `CRYPTOFLEX_DISABLE_PQC=1` is the correct way to keep CI/dev fast and
  deterministic — don't suggest mocking `liboqs` some other way.
- This is explicitly an **unaudited** library (see Status). Don't describe
  it to a user as "audited" or "production-hardened crypto"; it correctly
  describes itself as orchestration of audited *primitives*, with its own
  code still pre-audit.
- It is local-first by design: never add network calls, phone-home
  telemetry, or a remote risk-table fetch when extending it — that would
  contradict the project's stated non-goals.

## Status

As of package version 0.5.3: 140 automated tests passing, reproducibility
verification passing, and a published second-pass security hardening
review (chunked encryption, atomic writes, AAD-bound identity metadata,
malformed-input handling). 

## What you can build with this

Good fits — local/offline, file- or message-oriented, needing crypto
agility rather than a fixed algorithm forever:

- A file/folder encryption utility or backup tool with quantum-readiness
  built in from day one (encrypt now, migrate later without a rewrite).
- A desktop "vault" or password-protected secure-notes app using the
  keystore + `encrypt`/`decrypt` pair.
- USB-key or removable-media-bound encryption tools (pair `cryptoflex`
  with your own USB-presence check as an additional local factor).
- A peer-to-peer or LAN chat/messaging feature using `ephemeral_encrypt`/
  `ephemeral_decrypt` for per-message keys.
- Large-archive or media encryption using the streaming API, where
  loading the whole file into memory isn't an option.
- An internal CLI or automation script that encrypts artifacts (backups,
  exports, logs) with `min_profile` enforcement so nothing silently
  downgrades to classical-only in a pipeline.
- A migration/rotation job that walks a directory of `.cflx` files and
  re-encrypts them under a new profile after a policy or risk-table
  change, via `migrate_file(..., stream=True)`.
- Embedded/IoT tooling where there's no reliable network path to fetch a
  live threat feed, so a bundled offline risk table is the right shape.

Poor fits — don't reach for `cryptoflex` here:

- Network protocol design (TLS, VPNs) — use existing hybrid-KEM support
  in established protocol stacks instead; this library targets local/file
  workloads, not wire protocols.
- Anything needing a live/updatable threat-intelligence feed — the risk
  table is bundled and only updates with new package releases, by design.
- True forward secrecy for long-running conversations — `ephemeral_*`
  gives per-message key freshness, not a full ratcheting protocol like
  Signal's Double Ratchet.
- Regulated or high-stakes production data where an audited, certified
  library is a hard requirement — this is explicitly pre-audit.
