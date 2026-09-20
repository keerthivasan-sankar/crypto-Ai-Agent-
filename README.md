# cryptoflex — Claude AI Skill

This repository packages **cryptoflex** as a ready-to-use **Claude Skill**,
so any Claude-powered coding agent (Claude Code, Claude Desktop, Cowork,
or any tool that supports the open Skills format) can encrypt data,
manage keys, and build crypto-agile / post-quantum-ready features on
request — correctly, without guessing at the API.

## What's in this repo

```
.claude/skills/cryptoflex/
├── SKILL.md      # the skill itself — Claude loads this
└── README.md     # condensed API reference the skill points to
cryptoflex/        # the underlying Python package
```

`SKILL.md` is the file that matters for AI usage: it tells Claude what
the `cryptoflex` package does, when to reach for it, exactly which
functions to call for which task, and what mistakes to avoid.

## How to install this skill

1. Copy the `.claude/skills/cryptoflex/` folder into any repo or into
   your personal skills directory:
   - Personal (all your projects): `~/.claude/skills/cryptoflex/`
   - Project-only: `<your-repo>/.claude/skills/cryptoflex/`
2. Make sure the `cryptoflex` Python package is installed in the
   environment Claude will run code in:
   ```bash
   pip install cryptoflex          # classical (X25519) only
   pip install cryptoflex[pqc]     # + post-quantum (ML-KEM)
   ```
3. That's it — no further setup. Claude discovers the skill
   automatically the next time it starts a session in that context.

## How it activates

Claude Skills activate two ways, and this one supports both:

- **Automatically** — Claude reads the skill's `description` and loads
  it whenever your request matches (e.g. "encrypt this file with
  post-quantum crypto," "add hybrid encryption to my app," "use
  cryptoflex to...").
- **Manually** — type `/cryptoflex` in a Claude session to force-load
  it regardless of phrasing.

## What the skill lets an AI agent do

Once active, Claude can correctly:

- Pick a security profile at runtime (classical, hybrid-standard, or
  hybrid-high post-quantum) instead of hardcoding one
- Generate keypairs and encrypt/decrypt files or in-memory data
- Stream-encrypt large files without loading them fully into memory
- Set up a password-protected keystore for private keys
- Build ephemeral-key messaging (fresh key per message)
- Migrate existing encrypted files to a new security profile
- Wire up and explain the `cryptoflex` CLI
- Avoid known misuse patterns (e.g. persisting raw private keys,
  branching on decryption-failure internals, silently falling back to
  weaker crypto)

## Example prompts that trigger this skill

- "Encrypt this config file so it's safe even against a quantum
  computer."
- "Add a password-protected vault feature to my Python app using
  cryptoflex."
- "Write a script that migrates all my `.cflx` files to the
  hybrid-high profile."
- "Build a simple two-party encrypted messaging demo with
  ephemeral keys."

## What you can build with it

- File/folder encryption tools with quantum-readiness from day one
- Password-protected desktop vaults or secure-notes apps
- USB-bound or removable-media encryption utilities
- Peer-to-peer / LAN messaging with per-message ephemeral keys
- Large-archive or media encryption via the streaming API
- Automation/CLI tooling that encrypts backups or exports and refuses
  to silently downgrade below a required security profile
- Batch migration jobs that re-encrypt a directory of files after a
  policy change

Not a fit for: network protocol design (TLS/VPN), systems needing a
live/updatable threat feed, full forward-secrecy messaging protocols,
or regulated production workloads requiring certified/audited
cryptography — `cryptoflex` itself is unaudited.

## Full reference

For the complete function signatures, code examples, error-handling
contract, environment variables, and file formats, see
[`.claude/skills/cryptoflex/SKILL.md`](.claude/skills/cryptoflex/SKILL.md)
and its companion
[`README.md`](.claude/skills/cryptoflex/README.md).
