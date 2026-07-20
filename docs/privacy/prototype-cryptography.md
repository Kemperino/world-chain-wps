# Prototype Cryptographic Suites

## Status and scope

This document selects the cryptographic suites, Rust implementations, and key derivation rules for
the first Arc-style privacy prototype on World Chain. It is the normative format specification for
M0.4 and depends on the protocol invariants in [`prototype-protocol.md`](prototype-protocol.md).

The prototype uses one fixed 32-byte development master secret key (`MSK`) as the only root secret.
All transaction encryption, state-root encryption, and persistence-encryption material is derived
from that `MSK` with explicit domain separation. This is development-only key management; runtime
key rotation, DKG, TEE attestation binding, and production key ceremonies are out of scope.

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, and **MAY** describe normative
prototype behavior.

## Version 1 suite selection

| ID | Name | Status | Use |
| ---: | --- | --- | --- |
| `0x0001` | `WC_PRIVACY_HPKE_X25519_HKDF_SHA256_CHACHA20POLY1305_V1` | Required | M0.2 transaction encryption and M0.3 state-root encryption |
| `0x8001` | `WC_PRIVACY_HPKE_XWING_HKDF_SHA256_CHACHA20POLY1305_DRAFT_V1` | Research only | Compatibility experiments with Arc-style X-Wing suites |

Suite `0x0001` is the only suite enabled for the first implementation milestones. It uses:

- HPKE mode: Base mode from RFC 9180;
- KEM: DHKEM(X25519, HKDF-SHA256);
- KDF: HKDF-SHA256;
- AEAD: ChaCha20-Poly1305;
- state-root AEAD nonce length: 12 bytes;
- state-root AEAD tag length: 16 bytes; and
- hash function for protocol commitments: Keccak-256, matching the M0.2 and M0.3 format specs.

Suite `0x8001` is reserved for X-Wing research and MUST NOT be accepted by the prototype Privacy
Inbox or private-transition verifier unless a later design explicitly enables it.

## Rust implementation choices

The recommended Rust dependency set for suite `0x0001` is:

| Purpose | Crate | Rationale |
| --- | --- | --- |
| HPKE | `hpke` | Implements RFC 9180-style HPKE and supports the required X25519, HKDF-SHA256, and ChaCha20-Poly1305 combination |
| HKDF | `hkdf` | RustCrypto HKDF implementation over `sha2::Sha256`; can be used directly for MSK derivations |
| AEAD | `chacha20poly1305` | RustCrypto AEAD implementation for deterministic state-root and persistence test vectors |
| Hashing | `alloy-primitives` / `sha3` | Use Alloy Keccak helpers where available in repo code; otherwise use a Keccak-256 implementation already accepted by the workspace |
| RLP | `alloy-rlp` | Matches the repo's existing RLP dependency and the M0.2/M0.3 canonical encoding rules |

The workspace already uses `alloy-primitives`, `alloy-rlp`, `sha2`, and Keccak-capable dependencies.
It does not currently include HPKE, HKDF, or AEAD crates as workspace dependencies; M1.1 should add
only the selected crates needed by the first implementation slice.

## X-Wing maturity and compatibility cost

Arc-style X-Wing matching is deferred because the dependency and standardization surface is still
less settled than the classical HPKE suite:

- X-Wing is a hybrid X25519 plus ML-KEM construction, so matching it adds post-quantum KEM code,
  larger public keys, larger encapsulated keys, larger test fixtures, and more dependency review.
- Rust support exists, but available crates expose draft-version caveats. The standalone `x-wing`
  crate documents draft-version and audit warnings, and `hpke-rs` exposes an `XWingDraft06`
  algorithm rather than a final stable identifier.
- The `hpke` crate has a recent line that includes X-Wing support behind feature flags, but adopting
  it for production compatibility would still require pinning the exact draft version, matching
  Arc's byte encodings, and producing cross-implementation vectors before enabling the suite.
- Suite `0x0001` keeps the first prototype focused on the private execution and commitment
  validity model. It leaves a clean suite-ID path for X-Wing once Arc compatibility requirements
  and Rust implementation maturity are pinned.

## Common HKDF rules

All prototype derivations use HKDF-SHA256 with the fixed 32-byte development `MSK` as input keying
material:

```text
okm = HKDF-SHA256(
    salt = kdf_salt_v1(chain_id, key_epoch),
    ikm = MSK,
    info = kdf_info_v1(label, context),
    length = output_length
)
```

The salt is:

```text
kdf_salt_v1 = ascii("WORLD_CHAIN_PRIVACY_HKDF_SALT_V1") ||
              uint256_be(chain_id) ||
              uint64_be(key_epoch)
```

The info string is:

```text
kdf_info_v1 = 0x01 || rlp([
    label,
    suite_id,
    context
])
```

`label` is an ASCII byte string. `suite_id` is a `uint16`. `context` is the canonical RLP byte
string of a domain-specific list defined below. Implementations MUST reject unknown labels,
malformed context fields, and output lengths that differ from this document.

All multi-byte fixed-width integer encodings in salts use big-endian. RLP contexts use the canonical
RLP integer rules from M0.2 and M0.3.

## Development MSK

The prototype accepts exactly one active development `MSK` per deployment. It MUST be exactly 32
bytes and MUST be treated as secret process configuration even though the first prototype is not
production-safe.

The canonical public test-vector `MSK` is:

```text
msk = 0x000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f
```

This value is for tests only and MUST NOT be used outside local fixtures.

## Transaction encryption keypair

The global transaction encryption keypair for an epoch is derived once from `MSK`.

```text
tx_key_context_v1 = rlp([chain_id, key_epoch, suite_id])

tx_key_seed_v1 = HKDF-SHA256(
    salt = kdf_salt_v1(chain_id, key_epoch),
    ikm = MSK,
    info = kdf_info_v1(
        "WORLD_CHAIN_TX_HPKE_KEYPAIR_V1",
        tx_key_context_v1
    ),
    length = 32
)
```

For suite `0x0001`, `tx_key_seed_v1` is the X25519 private scalar input. The selected HPKE
implementation MUST either accept this 32-byte scalar directly or expose deterministic keypair
construction that produces the same X25519 public key. The advertised key ID remains the M0.2
definition:

```text
key_id = keccak256(
    ascii("WORLD_CHAIN_PRIVACY_KEY_V1") ||
    uint16_be(suite_id) ||
    recipient_public_key
)
```

The HPKE sender side MUST use fresh encapsulation randomness for every encrypted private
transaction. Deterministic transaction-encryption vectors MAY fix the sender randomness only inside
test fixtures.

## State-root encryption key and nonce

The state-root key and nonce are derived per private transition. Because the prototype creates at
most one private transition for a source block, this is also the per-source-block state-root key.
The context is:

```text
state_root_context_v1 = rlp([
    chain_id,
    key_epoch,
    suite_id,
    private_stf_version,
    private_transition_index,
    source_l2_block_number,
    source_l2_block_hash,
    parent_transition_hash,
    input_root,
    receipts_root
])
```

The state-root AEAD key is:

```text
state_root_key_v1 = HKDF-SHA256(
    salt = kdf_salt_v1(chain_id, key_epoch),
    ikm = MSK,
    info = kdf_info_v1("WORLD_CHAIN_STATE_ROOT_KEY_V1", state_root_context_v1),
    length = 32
)
```

The state-root AEAD nonce is:

```text
state_root_nonce_v1 = HKDF-SHA256(
    salt = kdf_salt_v1(chain_id, key_epoch),
    ikm = MSK,
    info = kdf_info_v1("WORLD_CHAIN_STATE_ROOT_NONCE_V1", state_root_context_v1),
    length = 12
)
```

The plaintext is exactly the 32-byte private post-state root. The AAD is `state_root_aad_v1` from
[`private-transition-commitment.md`](private-transition-commitment.md). The output is split into:

```text
encrypted_state_root = ciphertext[0..32]
state_root_tag = tag[0..16]
```

The nonce is derived from transition identity and transmitted in the public commitment so verifiers
can reject mismatches. It MUST NOT be derived from the plaintext root or a state-change flag. Binding
only the source block height is invalid because a reorganization can reuse the same height on a
different branch.

## Persistence key and nonce

Persistence encryption is reserved in the first prototype but its derivation is fixed so TEE-backed
host persistence can reuse the same domain shape later. Each encrypted snapshot or write-ahead-log
emission has a fresh `emission_id`, exactly 32 bytes, generated by the private service and stored
with the encrypted artifact metadata.

```text
persistence_context_v1 = rlp([
    chain_id,
    key_epoch,
    suite_id,
    private_stf_version,
    artifact_kind,
    artifact_sequence,
    transition_hash,
    emission_id
])
```

`artifact_kind` is `1` for a snapshot and `2` for a write-ahead-log segment. `artifact_sequence` is
a monotonically increasing `uint64` within that artifact kind. `transition_hash` is the 32-byte
private transition hash the artifact is derived from, or 32 zero bytes for empty genesis artifacts.

```text
persistence_key_v1 = HKDF-SHA256(
    salt = kdf_salt_v1(chain_id, key_epoch),
    ikm = MSK,
    info = kdf_info_v1("WORLD_CHAIN_PERSISTENCE_KEY_V1", persistence_context_v1),
    length = 32
)

persistence_nonce_v1 = HKDF-SHA256(
    salt = kdf_salt_v1(chain_id, key_epoch),
    ikm = MSK,
    info = kdf_info_v1("WORLD_CHAIN_PERSISTENCE_NONCE_V1", persistence_context_v1),
    length = 12
)
```

Persistence AAD is:

```text
persistence_aad_v1 = 0x01 || rlp([
    chain_id,
    key_epoch,
    suite_id,
    private_stf_version,
    artifact_kind,
    artifact_sequence,
    transition_hash,
    emission_id
])
```

The prototype MAY leave persistent state plaintext, but any encrypted persistence implementation
MUST use this key, nonce, and AAD shape.

## Authenticated data summary

| Operation | AAD |
| --- | --- |
| Private transaction HPKE | `aad_v1` from M0.2 |
| State-root AEAD | `state_root_aad_v1` from M0.3 |
| Persistence AEAD | `persistence_aad_v1` from this document |

The AAD for each operation binds the public metadata needed to reject replay into another chain,
key epoch, source block, private transition, or artifact context. AAD fields are authenticated but
not encrypted.

## Test vector requirements

Every implementation MUST publish deterministic fixture vectors before dependent M1/M2 work merges.
The vector files SHOULD live under `docs/privacy/test-vectors/` and MUST include at least:

1. `crypto-v1-derivations`: derives `tx_key_seed_v1`, `state_root_key_v1`,
   `state_root_nonce_v1`, `persistence_key_v1`, and `persistence_nonce_v1` from the canonical test
   `MSK`.
2. `crypto-v1-state-root-aead`: encrypts a fixed 32-byte private state root with
   ChaCha20-Poly1305, fixed transition metadata, and `state_root_aad_v1`.
3. `crypto-v1-persistence-aead`: encrypts a fixed persistence plaintext with a fixed
   `emission_id` and `persistence_aad_v1`.
4. `crypto-v1-hpke-envelope`: encrypts a fixed M0.2 `private_tx_v1` with suite `0x0001`, fixed
   recipient key material, fixed sender encapsulation randomness, and M0.2 `aad_v1`.
5. `crypto-v1-branch-separation`: repeats state-root encryption at the same block height with a
   different source block hash and proves that the nonce, ciphertext, and tag differ.

Each vector MUST include decoded logical fields, canonical serialized bytes, derived keys or public
keys where safe, nonces, AAD, plaintexts, ciphertexts, tags, and final hashes. The canonical test
`MSK` is public and may appear in vectors; deployment `MSK` values MUST NOT.

## Implementation boundary

- M1.1 implements the fixed `MSK` loader and derivation functions.
- M1.3 implements transaction encryption utilities with suite `0x0001`.
- M2.2 implements state-root encryption with suite `0x0001`.
- M6-era work may revisit suite `0x8001` after X-Wing compatibility and key-custody decisions are
  pinned.

## Dependencies

M0.1 is satisfied by [`prototype-protocol.md`](prototype-protocol.md). M0.2 and M0.3 consume the
suite IDs, keys, nonces, and AAD defined here.

## Out of scope

- runtime key rotation and historical key recovery;
- DKG, dealer enclaves, or production key ceremony design;
- TEE attestation binding;
- production X-Wing enablement;
- ZK proof key derivation; and
- side-channel hardening beyond fixed public acknowledgement and root-ciphertext unlinkability.

## References

- [RFC 9180: Hybrid Public Key Encryption](https://www.rfc-editor.org/rfc/rfc9180)
- [`hpke` crate documentation](https://docs.rs/hpke/latest/hpke/)
- [`hkdf` crate documentation](https://docs.rs/hkdf/latest/hkdf/)
- [`chacha20poly1305` crate documentation](https://docs.rs/chacha20poly1305/latest/chacha20poly1305/)
- [X-Wing KEM draft](https://datatracker.ietf.org/doc/draft-connolly-cfrg-xwing-kem/)
- [`x-wing` crate documentation](https://docs.rs/x-wing/latest/x_wing/)
- [`hpke-rs` KEM algorithm documentation](https://docs.rs/hpke-rs/latest/hpke_rs/hpke_types/enum.KemAlgorithm.html)
