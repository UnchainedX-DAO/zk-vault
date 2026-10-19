# zk-vault

**Quantum-secure, distributed, and persistent encrypted backup.**

*Encrypt it once. Keep it for a thousand years.*

> Store data that must never be exposed on the internet, making data management easy, while achieving quantum-secure distributed persistent data with no single point of failure.

## What is zk-vault?

zk-vault encrypts your important data on your device with post-quantum cryptography and stores it across a distributed network with no single point of failure. No server, validator, or storage provider ever sees your plaintext data. It is built for data that must not disappear for millennia — ideally forever.

**"zk" carries two meanings, and zk-vault pursues both:** (1) **zero-knowledge architecture** — no server, validator, or storage provider ever sees your plaintext; (2) **zero-knowledge proofs** — cryptographic verification of facts about your data without revealing it.

## Architecture

zk-vault works in two modes: **Personal Mode** for standalone use, and **Chain Mode** for full web3 capabilities. Start personal, go networked when you want stronger guarantees.

### Personal Mode (Mode A)

```
+---------------------------------------------------------------+
|                         CLIENT                                 |
|  Data Sources --> PQ Encryption --> Round-trip Verification    |
|                                                                |
|  Layer 0: Local Encrypted Backup (USB / External HDD / NAS)   |
+-------------------------------+-------------------------------+
                                |
                                v
                  +----------------------------+
                  |   Storage Provider         |
                  |   S3 / GCS / Azure Blob /  |
                  |   Backblaze B2 / MinIO     |
                  +----------------------------+
```

No chain, no token, no validators. Encrypt locally, push to any S3-compatible storage provider.

### Chain Mode (Mode B / Mode C)

```
+---------------------------------------------------------------+
|                         CLIENT                                 |
|  Data Sources --> PQ Encryption --> Round-trip Verification    |
|                                                                |
|  Layer 0: Local Encrypted Backup (USB / External HDD / NAS)   |
+-------------------------------+-------------------------------+
                                |
                                v
+---------------------------------------------------------------+
|                      zk-vault Chain                            |
|                  BFT Consensus (PoA --> DPoS)                  |
|                                                                |
|  On-chain state:                                               |
|    File Registry / Manifests / Merkle Roots                    |
|                                                                |
|  Storage (user selects):                                       |
|    Mode B: Native --> validators store data directly           |
|    Mode C: Filecoin --> delegated to Filecoin SPs              |
|    (can use both simultaneously)                               |
+----------+--------------------------+-------------------------+
           |                          |
           v                          v
+-------------------+      +--------------------+
|     Filecoin      |      |    BTC / ETH       |
|   (Mode C only)   |      |    Anchoring       |
|                   |      |                    |
|  Encrypted data   |      |  Merkle root       |
|  PoSt verified    |      |  tamper-proof      |
|  Multi-SP/region  |      |  timestamp         |
+-------------------+      +--------------------+
```

### Why Chain Mode

Personal Mode is fully functional on its own. Chain Mode adds guarantees that are impossible to achieve alone:

| | Personal (A) | Chain (B/C) |
|---|---|---|
| PQ encryption | Yes | Yes |
| Local backup | Yes | Yes |
| Storage verification | Trust the provider | BFT / Filecoin PoSt |
| Data integrity proof | Local hash chain | BTC/ETH anchoring |
| Single point of failure | Provider + local | None |
| Key recovery | Mnemonic (24-word) | Mnemonic + Guardian network |
| Permanent storage | Provider-dependent | Endowment pool (Mode B) + Layer 0 safety net |
| Multi-device sync | Manual | Chain state |

Data encrypted in Personal Mode is fully compatible with Chain Mode — migration requires no re-encryption.

| Component | Role |
|---|---|
| Local (Layer 0) | Safety net — independent of all crypto-economics |
| Storage Provider (Mode A) | Simple cloud storage for encrypted data |
| zk-vault Chain | State management + storage + coordination |
| Filecoin | External verified storage (user's choice) |
| BTC / ETH | Immutable proof of data integrity |

## Encryption

Post-quantum hybrid encryption — secure against both classical and quantum attacks.

| Component | Algorithm |
|---|---|
| Key encapsulation (PQ) | ML-KEM-768 |
| Key encapsulation (classical) | X25519 |
| Symmetric encryption | XChaCha20-Poly1305 |
| Signatures (PQ) | ML-DSA-65 |
| Signatures (classical) | Ed25519 |
| Hashing | BLAKE3 |
| KDF | Argon2id (t=3, m=256MB, p=4) |
| Memory safety | `zeroize` on all key material |

**Why hybrid:** If either the post-quantum or classical algorithm is broken, the other still protects data.

### Key Hierarchy

```
User Passphrase + [optional keyfile] + [optional hardware key]
    |
    v  Argon2id (t=3, m=256MB, p=4) + BLAKE3 factor mixing
Passphrase-Derived Key (PDK)
    |
    v  PDK encrypts only MK
Master Key (MK) -- 256-bit random, client-side
    |
    +-- ML-KEM-768 secret key       -- encrypted by MK
    +-- X25519 secret key            -- encrypted by MK
    +-- ML-DSA-65 secret key         -- encrypted by MK
    +-- Ed25519 secret key           -- encrypted by MK
    +-- Per-backup DEKs via HKDF     -- derived from MK

Recovery:
    +-- 24-word BIP-39 mnemonic (encodes MK, generated at init)
    +-- Guardian Shamir SSS (MK split into shares, 3-of-5 default)
```

### Per-File Encryption

1. Generate random 256-bit symmetric key
2. Encrypt: `XChaCha20-Poly1305(sym_key, nonce, plaintext, aad)`
3. Hybrid KEM key wrap: `ML-KEM-768 + X25519 → wrapping_key → wrap sym_key`
4. Bundle: `[version | kem_ct(1088B) | eph_pk(32B) | nonce(24B) | wrapped_key(48B) | ciphertext]`
5. Zeroize all key material from memory

Files larger than 64 KiB use streaming encryption with per-chunk nonces (`base_nonce XOR chunk_index`) and truncation-resistant AAD.

## Storage

Users select their storage mode:

| | Mode A (Personal) | Mode B (Native) | Mode C (Filecoin) |
|---|---|---|---|
| Requires chain | No | Yes | Yes |
| Data stored by | Cloud provider | Chain validators | Filecoin Storage Providers |
| Verification | Local hash chain | BFT consensus | PoSt (zk-SNARK) |
| Speed | Provider-dependent | Fast (direct access) | Slower (IPFS gateway) |
| Best for | Personal use, quick start | Smaller data, fast access | Large data, maximum distribution |

Mode B and Mode C can be used simultaneously for critical data.

Layer 0 (local encrypted backup) is always independent of all modes.

## Anchoring

A 32-byte Merkle root (BLAKE3) of all backed-up data is written to Bitcoin (OP_RETURN) and Ethereum (calldata). This proves data existed at a specific time and has not been tampered with since.

All users' roots are batched into a single Super Merkle Tree, so one transaction per chain covers all users.

| Users | Per-user cost (BTC) |
|---|---|
| 1 | ~$0.50 |
| 1,000 | ~$0.0005 |
| 100,000 | ~$0.000005 |

## Development

### Workspace Structure

```
zk-vault/
├── crates/
│   ├── core/    # Crypto library (encryption, keys, Merkle trees, manifests)
│   ├── cli/     # CLI binary (`zk-vault`)
│   └── chain/   # Chain node (consensus, state, mempool, RPC)
```

### Build & Test

```bash
cargo build --workspace          # build all crates
cargo test --workspace           # all tests (162 total)
cargo clippy --workspace -- -D warnings
cargo fmt
```

### Quick Start

```bash
zk-vault init                                    # generate keys
zk-vault backup --local ./backups ./my-data      # encrypt + backup
zk-vault backup --local ./bk --chain http://localhost:3030 ./data  # + chain
zk-vault verify <backup-id>                      # check integrity
zk-vault restore <backup-id> -o ./restored       # decrypt + restore
zk-vault status                                  # show vault info
```

See [docs/CLI.md](docs/CLI.md) for full command reference (all flags, chain RPC, config, dev commands, E2E workflows).

## Documentation

- **[docs/PRODUCT.md](docs/PRODUCT.md)** — Full product vision, architecture details, and design exploration
- **[docs/CLI.md](docs/CLI.md)** — CLI reference (all commands, flags, config, examples)
- **[crates/chain/README.md](crates/chain/README.md)** — Chain node architecture, modules, and RPC reference
- **[config.example.toml](config.example.toml)** — Storage and anchoring configuration reference

## License

AGPL-3.0-or-later. See [LICENSE](LICENSE).
