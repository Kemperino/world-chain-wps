# Private Transition and Commitment Format

## Status and scope

This document defines the private transition header and public `N+1` commitment payload for the
first Arc-style privacy prototype on World Chain. It is the normative format specification for M0.3
and depends on the protocol invariants in [`prototype-protocol.md`](prototype-protocol.md) and the
private transaction format in [`private-transaction-envelope.md`](private-transaction-envelope.md).

The format binds the private transition to one canonical World Chain source block, one parent
private transition, one ordered input batch, one ordered private receipt batch, and one encrypted
private post-state root. Public World Chain nodes still treat the payload as ordinary public
calldata written by a system transaction; privacy verifiers enforce the private lineage rules.

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, and **MAY** describe normative
prototype behavior.

## Version 1 constants

| Constant | Value | Meaning |
| --- | ---: | --- |
| `PRIVATE_TRANSITION_HEADER_V1` | `0x01` | Private transition header format version |
| `PRIVATE_COMMITMENT_PAYLOAD_V1` | `0x01` | Public `N+1` commitment payload format version |
| `PRIVATE_INPUT_ROOT_V1` | `0x01` | Ordered private input root version |
| `PRIVATE_RECEIPT_ROOT_V1` | `0x01` | Ordered private receipt root version |
| `PRIVATE_GENESIS_PARENT` | `0x0000...0000` | Parent hash for the first private transition |
| `PRIVATE_STF_VERSION_V1` | `0x00000001` | First prototype private state-transition function |

`PRIVATE_GENESIS_PARENT` is exactly 32 zero bytes. Any non-genesis transition with a zero parent
transition hash is invalid.

## Encoding rules

All version 1 structures use a one-byte version prefix followed by one canonical RLP list:

```text
versioned_payload = version_prefix || rlp(list)
```

All implementations MUST apply the following rules:

- integers use canonical RLP integer encoding: unsigned, big-endian, shortest form, with zero
  encoded as the empty byte string;
- fixed byte strings have the exact lengths specified below;
- lists contain exactly the specified number of fields in the specified order;
- nested lists are allowed only where this document explicitly defines them;
- non-canonical encodings, integer overflow, unknown versions, and trailing bytes are invalid;
- hashes use Keccak-256; and
- domain strings are ASCII bytes exactly as written.

The prefix byte is the format version. It is not also included as an RLP field.

## Private inputs

Only successful public privacy-entrypoint operations from a canonical source block become private
inputs. Version 1 has two input kinds:

| Kind | Name | Meaning |
| ---: | --- | --- |
| `1` | `EncryptedTransaction` | A successful Privacy Inbox submission containing an M0.2 encrypted envelope |
| `2` | `ShieldDeposit` | A successful shield deposit into private state |

An encrypted transaction input leaf is:

```text
encrypted_tx_input_leaf_v1 = keccak256(
    ascii("WORLD_CHAIN_PRIVATE_INPUT_LEAF_V1") ||
    rlp([
        kind,                    // 1
        source_tx_index,
        source_log_index,
        inbox_address,
        outer_tx_hash,
        envelope_hash
    ])
)
```

`inbox_address` is exactly 20 bytes. `outer_tx_hash` and `envelope_hash` are exactly 32 bytes.
`envelope_hash = keccak256(envelope_v1)` from M0.2.

A shield deposit input leaf is:

```text
shield_deposit_input_leaf_v1 = keccak256(
    ascii("WORLD_CHAIN_PRIVATE_INPUT_LEAF_V1") ||
    rlp([
        kind,                    // 2
        source_tx_index,
        source_log_index,
        shield_contract_address,
        deposit_id,
        token_address,
        recipient_commitment,
        amount
    ])
)
```

`shield_contract_address` and `token_address` are exactly 20 bytes. `deposit_id` and
`recipient_commitment` are exactly 32 bytes. `amount` is a `uint256`.

Inputs MUST be sorted by public transaction index and then log index from the canonical source
block. A source block with no private inputs creates no private transition and therefore has no
input root.

The ordered input root is:

```text
input_root_v1 = keccak256(
    PRIVATE_INPUT_ROOT_V1 ||
    rlp([input_leaf_0, input_leaf_1, ...])
)
```

Each input leaf is encoded as an exact 32-byte RLP string. The list MUST contain at least one leaf.
This root is an ordered list commitment, not an inclusion-proof tree.

## Private receipts

Every accepted private input creates exactly one private receipt. The receipt is private service
data; only the receipt root is public in the commitment. Failed private transactions MUST still
produce receipts, even when they do not mutate private state.

Version 1 receipt status codes are:

| Status | Name | State mutation |
| ---: | --- | --- |
| `0` | `Success` | May mutate state |
| `1` | `EncryptedTransactionDecryptFailed` | No |
| `2` | `UnsupportedPrivateTransactionVersion` | No |
| `3` | `MalformedPrivateTransaction` | No |
| `4` | `WrongPrivateChainId` | No |
| `5` | `InvalidPrivateGasLimit` | No |
| `6` | `InvalidPrivateSignature` | No |
| `7` | `PrivateNonceMismatch` | No |
| `8` | `InsufficientPrivateBalance` | No |
| `9` | `PrivateExecutionReverted` | Nonce consumed if pre-execution validation passed |
| `10` | `UnsupportedPrivateCall` | Nonce consumed if pre-execution validation passed |
| `255` | `InternalPrivateExecutorError` | No; transition MUST NOT be sealed |

`InternalPrivateExecutorError` is reserved for local execution or storage failures. A transition
containing such a receipt MUST NOT be committed. All other receipt statuses are deterministic
private execution outcomes and MAY appear in a committed transition.

The private receipt leaf is:

```text
private_receipt_leaf_v1 = keccak256(
    ascii("WORLD_CHAIN_PRIVATE_RECEIPT_LEAF_V1") ||
    rlp([
        input_index,
        input_leaf,
        status,
        private_tx_hash_or_zero,
        recovered_sender_or_zero,
        sender_nonce_or_zero,
        private_gas_used,
        state_changed,
        output_commitment
    ])
)
```

`input_leaf`, `private_tx_hash_or_zero`, and `output_commitment` are exactly 32 bytes.
`recovered_sender_or_zero` is exactly 20 bytes. The zero forms are all-zero byte strings of the
same length. `state_changed` is encoded as integer `0` or `1`. `output_commitment` is zero for the
first transfer prototype unless a later receipt format defines a private output commitment.

The ordered receipt root is:

```text
receipts_root_v1 = keccak256(
    PRIVATE_RECEIPT_ROOT_V1 ||
    rlp([receipt_leaf_0, receipt_leaf_1, ...])
)
```

The receipt list length MUST equal the input list length. Receipt `i` MUST refer to input leaf `i`.
This commits failed private transactions through both `input_root_v1` and `receipts_root_v1`; a
block containing only failed private transactions still creates a private transition whose private
state root may equal its parent.

## Private transition header

The version 1 private transition header is:

```text
private_transition_header_v1 = 0x01 || rlp([
    chain_id,
    private_transition_index,
    source_l2_block_number,
    source_l2_block_hash,
    parent_transition_hash,
    input_root,
    receipts_root,
    encrypted_state_root,
    state_root_nonce,
    state_root_tag,
    key_epoch,
    private_stf_version,
    crypto_suite_id
])
```

| Field | Type | Validation |
| --- | --- | --- |
| `chain_id` | `uint256` | MUST equal the configured World Chain chain ID |
| `private_transition_index` | `uint64` | Contiguous; increments only for source blocks with privacy inputs |
| `source_l2_block_number` | `uint64` | Public block `N` that supplied the private inputs |
| `source_l2_block_hash` | `bytes32` | Canonical hash of source block `N` |
| `parent_transition_hash` | `bytes32` | Previous verified private transition hash, or `PRIVATE_GENESIS_PARENT` for index `0` |
| `input_root` | `bytes32` | Ordered input root for source block `N` |
| `receipts_root` | `bytes32` | Ordered private receipt root for source block `N` |
| `encrypted_state_root` | `bytes` | AEAD ciphertext of the 32-byte private post-state root |
| `state_root_nonce` | `bytes` | AEAD nonce from M0.4, 12 bytes in suite `0x0001` |
| `state_root_tag` | `bytes` | AEAD authentication tag, 16 bytes in suite `0x0001` |
| `key_epoch` | `uint64` | Key epoch used for state-root encryption |
| `private_stf_version` | `uint32` | MUST equal an enabled private STF version |
| `crypto_suite_id` | `uint16` | MUST equal an enabled M0.4 state-root encryption suite |

`encrypted_state_root` length is suite-defined. For M0.4 suite `0x0001`, it is exactly 32 bytes
because the AEAD tag is carried separately in `state_root_tag`.

The private transition hash is:

```text
private_transition_hash_v1 = keccak256(
    ascii("WORLD_CHAIN_PRIVATE_TRANSITION_HASH_V1") ||
    private_transition_header_v1
)
```

The hash is the parent pointer for the next private transition and the stable identifier used by
privacy verifiers and private-state RPCs. It commits to the encrypted root material, not the
plaintext private state root.

## State-root encryption binding

The plaintext encrypted by the state-root AEAD is exactly the 32-byte private post-state root.
M0.4 defines the key, nonce, AEAD, and versioned AAD. Version 1 state-root AAD is:

```text
state_root_aad_v1 = 0x01 || rlp([
    chain_id,
    private_transition_index,
    source_l2_block_number,
    source_l2_block_hash,
    parent_transition_hash,
    input_root,
    receipts_root,
    key_epoch,
    private_stf_version,
    crypto_suite_id
])
```

The AAD intentionally excludes `encrypted_state_root`, `state_root_nonce`, and `state_root_tag`.
Privacy verifiers derive the expected nonce from the same transition identity and reject a
commitment whose transmitted nonce differs before comparing ciphertext and tag.

Equal plaintext private roots in different transitions MUST NOT produce linkable public ciphertexts.
The nonce derivation in M0.4 therefore binds branch-specific transition identity, not the plaintext
root and not only the block height.

## Public N+1 commitment payload

The builder injects one public system transaction in block `N+1` for each source block `N` that
contains privacy inputs. The public calldata payload to the private-transition registry is:

```text
private_commitment_payload_v1 = 0x01 || rlp([
    private_transition_hash,
    private_transition_header
])
```

`private_transition_hash` is exactly 32 bytes. `private_transition_header` is the complete
`private_transition_header_v1` byte string, encoded as an RLP byte string. The registry
implementation MAY decode and store individual fields, but this byte string is the canonical
payload for hashing and test vectors.

A privacy verifier accepts a public commitment for source block `N` only if all of the following
hold:

1. The payload appears in canonical public block `N+1`.
2. The payload version is `PRIVATE_COMMITMENT_PAYLOAD_V1`.
3. The embedded header version is `PRIVATE_TRANSITION_HEADER_V1`.
4. `keccak256(ascii("WORLD_CHAIN_PRIVATE_TRANSITION_HASH_V1") || private_transition_header)` equals
   the transmitted `private_transition_hash`.
5. The header's source block number and hash match canonical block `N`.
6. The header's parent transition hash matches the latest verified private transition before `N`.
7. The header's input and receipt roots match independent extraction, decryption, and execution.
8. The header's key epoch, private STF version, and crypto suite ID are enabled.
9. The state-root ciphertext, nonce, tag, and AAD verify under M0.4.

The public registry contract in M2 MAY enforce a subset of these checks, such as payload shape,
source block number, and duplicate prevention. Full private transition validity remains a privacy
verifier responsibility in the prototype.

## Replay resistance

A commitment cannot be replayed for another source block or branch because the transition hash,
state-root AAD, and nonce derivation bind:

- `chain_id`;
- `private_transition_index`;
- `source_l2_block_number`;
- `source_l2_block_hash`;
- `parent_transition_hash`;
- `input_root`;
- `receipts_root`;
- `key_epoch`;
- `private_stf_version`; and
- `crypto_suite_id`.

Copying a valid payload to a later block fails the exact `N -> N+1` schedule. Copying it onto
another branch at the same height fails because the source block hash, parent transition hash,
input root, and receipt root differ. Copying only the encrypted root material into another header
fails AEAD verification because the AAD and derived nonce differ.

## Test vector requirements

Every implementation MUST include stable test vectors for the following cases:

1. One source block with one successful encrypted transaction input and one successful private
   receipt.
2. One source block with one accepted envelope that fails decryption.
3. One source block with one decrypted private transaction that fails for insufficient private
   balance, leaving the plaintext private state root unchanged.
4. Two branches with the same source block number and different source block hashes, proving that a
   commitment from one branch fails on the other.
5. Two transitions with equal plaintext private state roots, proving that encrypted state-root
   ciphertexts are unlinkable.

Each vector MUST publish:

- decoded logical fields;
- canonical input leaves and `input_root`;
- canonical receipt leaves and `receipts_root`;
- `state_root_aad_v1`;
- `private_transition_header_v1`;
- `private_transition_hash_v1`; and
- `private_commitment_payload_v1`.

Vectors MUST use fixed fixture keys from M0.4 and MUST NOT use production key material.

## Dependencies

M0.1 is satisfied by [`prototype-protocol.md`](prototype-protocol.md). M0.2 is satisfied by
[`private-transaction-envelope.md`](private-transaction-envelope.md). M0.4 defines the concrete
state-root encryption suite consumed by this specification.

## Out of scope

- implementation of private transition sealing or registry contracts, which belongs to M2;
- ZK proof, TEE attestation, or dispute payload formats;
- private contract execution semantics beyond receipt status reservation;
- production key rotation, DKG, historical key recovery, or key revocation; and
- inclusion-proof Merkle trees for private inputs or receipts.

## References

- [Prototype protocol](prototype-protocol.md)
- [Private transaction and encrypted envelope](private-transaction-envelope.md)
- [Prototype cryptographic suites](prototype-cryptography.md)
- [Ethereum RLP](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/)
