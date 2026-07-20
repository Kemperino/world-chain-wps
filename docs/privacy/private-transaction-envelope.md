# Private Transaction and Encrypted Envelope

## Status and scope

This document defines the signed private transaction and public encrypted-envelope wire formats for
the first Arc-style privacy prototype on World Chain. It is the normative format specification for
M0.2 and is subordinate to the protocol invariants in
[`prototype-protocol.md`](prototype-protocol.md).

The format deliberately separates three identities:

- the private sender, recovered from the inner EIP-712 signature;
- the public submitter or relayer, which pays for the outer World Chain transaction; and
- the active privacy encryption key, identified by an epoch and key ID.

The public submitter need not be the private sender. Public World Chain execution validates only
the envelope metadata described below. Decryption, sender recovery, and private transaction
validation happen in the privacy service and do not affect public block validity.

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, and **MAY** describe normative
prototype behavior.

## Version 1 constants

| Constant | Value | Meaning |
| --- | ---: | --- |
| `PRIVATE_TX_TYPE_V1` | `0x01` | Inner private transaction format version |
| `ENVELOPE_TYPE_V1` | `0x01` | Public encrypted-envelope format version |
| `PRIVATE_TX_GAS_LIMIT_V1` | `100000` | Only accepted private gas limit in version 1 |
| `MAX_PRIVATE_CALLDATA_BYTES` | `4096` | Maximum plaintext transaction calldata |
| `MAX_PRIVATE_TX_BYTES` | `8192` | Maximum encoded signed private transaction |
| `MAX_ENCAPSULATED_KEY_BYTES` | `4096` | Maximum HPKE encapsulated-key encoding |
| `MAX_CIPHERTEXT_BYTES` | `8256` | Maximum HPKE ciphertext, including authentication tag |
| `MAX_ENVELOPE_BYTES` | `16384` | Maximum encoded envelope passed to the Privacy Inbox |

The concrete version 1 HPKE, KDF, and AEAD suite is assigned by M0.4. Every supported suite MUST
fit these format-level limits and MUST add no more than 64 bytes of ciphertext overhead to an
`MAX_PRIVATE_TX_BYTES` plaintext.

## Encoding rules

Both wire formats use an EIP-2718-style one-byte version prefix followed by one canonical RLP list:

```text
versioned_payload = version_prefix || rlp(list)
```

These prefixes version privacy payloads only. They are not new public Ethereum transaction types:
the outer submission remains an ordinary World Chain transaction that passes the encrypted envelope
as calldata to the Privacy Inbox.

All implementations MUST apply the following rules:

- integers use canonical RLP integer encoding: unsigned, big-endian, shortest form, with zero
  encoded as the empty byte string;
- fixed byte strings have the exact lengths specified below;
- lists contain exactly the specified number of fields in the specified order;
- nested lists, non-canonical encodings, integer overflow, and trailing bytes are invalid;
- byte limits apply to the complete versioned payload, including the prefix byte and RLP framing; and
- hashes use Keccak-256.

The prefix byte is the format version. It is not also included as an RLP field.

## Signed private transaction

### Logical fields

| Field | Type | Validation |
| --- | --- | --- |
| `version` | `uint8` | Implicitly `1` from `PRIVATE_TX_TYPE_V1` |
| `chain_id` | `uint256` | MUST equal the configured pEVM chain ID |
| `nonce` | `uint64` | MUST equal the recovered sender's current pEVM nonce |
| `gas_limit` | `uint64` | MUST equal `PRIVATE_TX_GAS_LIMIT_V1` |
| `to` | `address` | Exactly 20 bytes; contract creation is not encoded in version 1 |
| `value` | `uint256` | Private native-token value transferred to `to` |
| `data` | `bytes` | At most `MAX_PRIVATE_CALLDATA_BYTES` bytes |
| `signature` | `(y_parity, r, s)` | Recoverable secp256k1 ECDSA signature described below |

The transaction format carries calldata so later executors can support private calls without a wire
format change. The first prototype executor MAY support only empty calldata and value transfers;
unsupported private calls produce a private failed receipt rather than a public revert.

### EIP-712 signing payload

The sender signs this exact EIP-712 type:

```text
PrivateTransaction(uint8 version,uint256 chainId,uint64 nonce,uint64 gasLimit,address to,uint256 value,bytes data)
```

The EIP-712 domain is:

| Domain field | Value |
| --- | --- |
| `name` | `World Chain Private Transaction` |
| `version` | `1` |
| `chainId` | The transaction's `chain_id` |
| `verifyingContract` | The Privacy Inbox address on that chain |

The domain type is:

```text
EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)
```

The struct values use `version = 1` and the transaction fields above. EIP-712 encodes `data` as
`keccak256(data)`. The signing digest is the standard:

```text
signing_hash = keccak256(0x1901 || domain_separator || struct_hash)
```

Binding both `chain_id` and the Privacy Inbox address prevents a signature from being replayed on a
different chain or through a different privacy entrypoint. The inner signature does not bind the
public relayer, source block, envelope randomness, or key epoch.

### Signature format and sender recovery

The signature uses secp256k1 over `signing_hash` and is represented by:

| Component | Encoding |
| --- | --- |
| `y_parity` | RLP integer `0` or `1` |
| `r` | Canonical RLP integer in the secp256k1 scalar range |
| `s` | Canonical RLP integer in the lower half of the secp256k1 scalar range |

EIP-155 `v` values and EIP-2098 compact signatures are not valid version 1 encodings. The recovered
uncompressed public key's 64-byte `x || y` body is hashed with Keccak-256, and the low 20 bytes are
the private sender address. Recovery failure, zero scalars, out-of-range scalars, or high-`s`
signatures make the private transaction invalid.

### Wire encoding and transaction hash

The signed version 1 wire payload is:

```text
private_tx_v1 = 0x01 || rlp([
    chain_id,
    nonce,
    gas_limit,
    to,
    value,
    data,
    y_parity,
    r,
    s
])
```

Its stable transaction identifier is:

```text
private_tx_hash = keccak256(private_tx_v1)
```

The hash includes the sender signature but not encryption metadata. Re-encrypting the same signed
transaction under fresh HPKE randomness therefore changes the envelope bytes while preserving
`private_tx_hash`.

## Encrypted envelope

### Logical fields

The envelope is public calldata and is not itself confidential.

| Field | Type | Validation |
| --- | --- | --- |
| `version` | `uint8` | Implicitly `1` from `ENVELOPE_TYPE_V1` |
| `key_epoch` | `uint64` | MUST equal an epoch accepted by the Privacy Inbox |
| `key_id` | `bytes32` | MUST identify the advertised transaction-encryption public key |
| `suite_id` | `uint16` | MUST identify a supported M0.4 HPKE suite |
| `encapsulated_key` | `bytes` | Suite-defined HPKE `enc`, non-empty and at most 4096 bytes |
| `ciphertext` | `bytes` | HPKE ciphertext and authentication tag, non-empty and at most 8256 bytes |

The fixed-key prototype uses `key_epoch = 0`. Runtime epoch changes are out of scope, but the field
is present so rotating the key does not require a new envelope layout.

For an M0.4 suite with a canonical transaction-encryption public-key encoding `recipient_public_key`,
the key ID is:

```text
key_id = keccak256(
    ascii("WORLD_CHAIN_PRIVACY_KEY_V1") ||
    uint16_be(suite_id) ||
    recipient_public_key
)
```

The Privacy Inbox advertises the complete active tuple
`(key_epoch, key_id, suite_id, recipient_public_key)`. The prototype accepts only that tuple.

### Wire encoding

The version 1 envelope is:

```text
envelope_v1 = 0x01 || rlp([
    key_epoch,
    key_id,
    suite_id,
    encapsulated_key,
    ciphertext
])
```

The outer World Chain transaction passes `envelope_v1` as the `bytes` argument to the Privacy Inbox.
The outer transaction's sender and nonce belong to the relayer and are not part of the private
transaction identity.

### Authenticated encryption context

The HPKE plaintext is exactly `private_tx_v1`. Version 1 uses HPKE Base mode. M0.4 assigns the
algorithm tuple behind `suite_id`; this document fixes the protocol context supplied to that suite.

The HPKE `info` byte string is:

```text
ascii("WORLD_CHAIN_PRIVACY_ENVELOPE_V1")
```

The HPKE authenticated additional data is:

```text
aad_v1 = 0x01 || rlp([
    chain_id,
    privacy_inbox_address,
    key_epoch,
    key_id,
    suite_id,
    encapsulated_key
])
```

Here, `chain_id` and `privacy_inbox_address` are the target deployment's configured public values;
the remaining fields are copied from the envelope. The address is encoded as an exact 20-byte RLP
string. Including the encapsulated key authenticates every public encryption field. An envelope
copied to another chain or inbox, or with altered epoch, key ID, suite ID, or encapsulated key, fails
HPKE authentication.

The client encryption sequence is:

1. Read the active key tuple from the target Privacy Inbox.
2. Construct and sign `private_tx_v1` for that chain and inbox.
3. Initialize HPKE Base mode with the advertised public key and version 1 `info`, producing
   `encapsulated_key`.
4. Construct `aad_v1` from the target deployment and public encryption metadata.
5. Seal `private_tx_v1` once as HPKE sequence number zero.
6. Encode the resulting `encapsulated_key` and ciphertext in `envelope_v1`.

An implementation MUST create a fresh HPKE sender context for every envelope. Reusing an
encapsulation or sender context is forbidden.

## Replay protection

Replay protection has independent domain and state layers:

1. The EIP-712 domain and signed `chain_id` bind the transaction to one chain and Privacy Inbox.
2. The pEVM accepts only the recovered sender's exact current nonce. Future and stale nonces fail
   deterministically and do not mutate private state.
3. A transaction that passes pre-execution validation consumes that nonce exactly once, even if its
   subsequent EVM call reverts. A transaction rejected during pre-execution validation, including
   for insufficient balance, does not consume the nonce.
4. The canonical public block and log ordering determines which duplicate reaches the matching
   nonce first. Later duplicates produce private failed receipts.

The key epoch is intentionally not part of the inner signature. A client may re-encrypt an
unexecuted signed transaction for a newly advertised key without asking the sender to sign again.
Once the transaction passes pre-execution validation, its consumed sender nonce prevents the
re-encrypted copy from being applied again. Re-submitting a transaction that always fails
pre-execution validation has no state effect, even though it may create additional private failed
receipts.

## Deterministic rejection behavior

### Public Privacy Inbox validation

The Privacy Inbox validates only public envelope properties, in this order:

1. Reject an empty input or input larger than `MAX_ENVELOPE_BYTES`.
2. Read the envelope version prefix. Reject any value other than `ENVELOPE_TYPE_V1` without
   attempting to decode a version 1 body.
3. Decode one canonical RLP list and reject malformed fields, incorrect field counts, invalid integer
   encodings, incorrect fixed lengths, empty variable fields, or trailing bytes.
4. Enforce the encapsulated-key and ciphertext size limits.
5. Reject `suite_id` values that are not enabled by the deployed protocol profile.
6. Reject a `(key_epoch, key_id, suite_id)` tuple that is not active for transaction encryption.

These failures revert the outer inbox call with a stable category:

| Category | Meaning |
| --- | --- |
| `EnvelopeSize` | Empty or oversized complete envelope or bounded field |
| `UnsupportedEnvelopeVersion` | Unknown envelope version prefix |
| `MalformedEnvelope` | Non-canonical or structurally invalid version 1 body |
| `UnsupportedSuite` | Unknown or disabled crypto-suite ID |
| `UnknownEncryptionKey` | Unknown, inactive, or inconsistent epoch/key/suite tuple |

Exact contract error selectors are defined with the Privacy Inbox implementation. Implementations
MUST preserve the categories and validation order. A reverted inbox call does not create a privacy
input. An accepted call returns only the fixed acknowledgement and becomes a privacy input even if
later decryption or private validation fails.

### Private validation after acknowledgement

The privacy service processes an accepted envelope in this order:

1. Revalidate the canonical envelope and active key tuple from the source block.
2. Reconstruct `info` and `aad_v1`, then perform HPKE open.
3. Enforce `MAX_PRIVATE_TX_BYTES` before decoding plaintext.
4. Read the inner private transaction version prefix and reject an unknown version without version 1
   decoding.
5. Decode canonical version 1 RLP and validate field widths and limits.
6. Require the configured pEVM chain ID and `PRIVATE_TX_GAS_LIMIT_V1`.
7. Recompute `signing_hash`, recover the sender, and enforce canonical low-`s` signature rules.
8. Require the sender's exact current nonce before execution validation.

Every failure after public acknowledgement produces a deterministic private failed receipt and no
public revert, return value, or log. At minimum, receipt status distinguishes authenticated
decryption failure, unsupported private transaction version, malformed private transaction, wrong
chain ID, invalid gas limit, invalid signature, and nonce mismatch. The private receipt format and
execution-level failures are specified in M0.3 and M1.6.

## Security properties and limitations

- Authentication covers all public envelope metadata and the complete signed private transaction.
- The EIP-712 signature authenticates the private sender; the HPKE envelope authenticates ciphertext
  integrity but does not authenticate the public relayer.
- Randomized encryption hides equality between envelopes carrying the same signed transaction.
- Envelope length, key epoch, key ID, suite ID, encapsulated-key length, ciphertext length, public
  submitter, submission timing, and public gas payment remain visible.
- The fixed development key is not production-safe. Runtime key rotation, key expiry, dealer or DKG
  ceremonies, enclave custody, and historical-key recovery are deferred.
- Contract creation is not representable in version 1. Private contract execution remains outside
  the first transfer prototype even though the transaction format reserves calldata.

## Implementation boundary

This ticket specifies formats and validation behavior only. It does not implement signing,
encryption, decryption, or contract parsing.

- M0.4 selects the concrete HPKE, KDF, and AEAD implementation for each `suite_id` and publishes
  cryptographic test vectors.
- M1.2 implements the Privacy Inbox's public parser and metadata checks.
- M1.3 implements signing, sender recovery, encryption, decryption, and submission utilities.
- M1.6 implements private nonce, balance, intrinsic gas, destination, value, and execution checks.

## Dependencies

M0.1 is satisfied by [`prototype-protocol.md`](prototype-protocol.md). M0.3 and M1 implementation
work MUST consume this specification rather than defining competing transaction or envelope
encodings.

## Out of scope

- implementation of signing, sender recovery, encryption, or decryption utilities, which belongs to
  M1.3;
- selection and implementation of the concrete version 1 cryptographic suite, which belongs to
  M0.4;
- runtime key rotation, epoch activation rules, key expiry, historical-key recovery, and dealer or
  DKG-based key management;
- private contract deployment and execution semantics; and
- private receipt and transition commitment formats, which belong to M0.3.

## References

- [Prototype protocol](prototype-protocol.md)
- [Arc Privacy Sector paper](https://6778953.fs1.hubspotusercontent-na1.net/hubfs/6778953/PDFs/Whitepapers/Arc_Privacy_Sector%20(5).pdf)
- [EIP-712: Typed structured data hashing and signing](https://eips.ethereum.org/EIPS/eip-712)
- [EIP-2718: Typed transaction envelope](https://eips.ethereum.org/EIPS/eip-2718)
- [RFC 9180: Hybrid Public Key Encryption](https://www.rfc-editor.org/rfc/rfc9180)
