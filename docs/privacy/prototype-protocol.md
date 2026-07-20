# Arc Privacy Network Prototype Protocol

## Status and scope

This document is the protocol source of truth for the first Arc-style privacy prototype on World
Chain. It defines the behavior that the prototype builder, validator, private executor, system
contracts, and end-to-end tests must implement.

The prototype deliberately differs from the long-term architecture. In particular, it uses a
deterministic one-block pipeline: privacy inputs ordered in public block `N` produce a private
transition commitment in public block `N+1`. This rule supersedes same-block private-execution and
commitment language in earlier research documents **for the prototype only**. It does not change the
recommended long-term design, where private execution may return to the block-sealing path after its
latency, availability, and attestation requirements are understood.

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, and **MAY** describe normative
prototype behavior.

## Goals

The prototype must demonstrate that World Chain can:

1. Order encrypted private transactions through the normal public chain.
2. Derive a separate, persistent pEVM state from privacy-relevant public inputs.
3. Commit an encrypted private state root on a deterministic schedule.
4. Re-execute and validate the same private transition through another prototype privacy verifier.
5. Shield public funds into private state and later support a controlled unshield path.
6. Preserve public block production and public validation independently of private-service
   availability or private-transition validity.

The prototype is not intended to establish a production confidentiality or validity model.

## Protocol invariants

The following invariants apply to every prototype implementation:

1. **One canonical log.** World Chain is the only source of canonical ordering and data
   availability. The pEVM does not order transactions independently.
2. **One additional state machine, not another chain.** The pEVM has its own persistent database,
   account state, storage, bytecode, receipts, and state root. It has no private mempool, P2P
   network, sequencer, or consensus protocol.
3. **Sparse private transitions.** Only a public block containing at least one privacy-relevant
   input creates a private transition.
4. **Initial private inputs.** Privacy-relevant inputs initially consist of successful encrypted
   transaction submissions to the Privacy Inbox and successful shield deposits.
5. **Exact next-block commitment for private validity.** If public block `N` contains a
   privacy-relevant input, a valid private lineage MUST contain exactly one commitment to the
   transition derived from `N` in public block `N+1`.
6. **No empty transition.** If public block `N` contains no privacy-relevant inputs, it creates no
   private transition, and a valid private lineage MUST NOT contain a transition commitment for `N`
   in `N+1`.
7. **Acknowledgement-only inbox.** Public execution observes only whether the public inbox call
   succeeded and its fixed acknowledgement. It cannot observe private return values, logs, revert
   reasons, or private execution gas usage.
8. **Prototype key material.** The prototype uses a fixed 32-byte master secret key (`MSK`) and
   fixed, low private gas parameters. Both are development configuration and MUST NOT be presented
   as production-safe.
9. **Domain-separated keys.** Transaction encryption, state-root encryption, and persistence
   encryption use separate keys derived from `MSK`.
10. **Encrypted public commitment.** The public system contract stores an encrypted private state
    root and its public transition metadata. It MUST NOT receive the plaintext private state root.
11. **No TEE requirement.** Prototype private execution and persistence run outside a TEE. TEE
    execution, attestation, dealer or DKG ceremonies, and ZK adjudication are deferred.
12. **Private validation before private progress.** After observing privacy inputs in `N`, a privacy
    verifier MUST execute and validate that transition before it promotes the commitment from
    `N+1`, advances its verified private head, or serves state derived from that transition as
    verified. This requirement does not gate public block acceptance.

Invariants 5, 6, and 12 are enforced by privacy verifiers. They constrain the verified private
lineage but do not add private execution or commitment validity to public World Chain consensus.

## Components and trust boundaries

### Public World Chain execution

The public EVM executes the Privacy Inbox, shield escrow, and private-transition registry. Public
nodes continue to execute and validate ordinary World Chain behavior normally. The encrypted
payload and encrypted private root are opaque to nodes that do not run the prototype privacy
service.

Private-transition validity is not a public World Chain block-validity condition in the prototype.
A non-enclave node validates public EVM execution, including the registry call's public rules, but
does not independently validate the private state transition represented by its ciphertext. It may
rely on privacy-verifier RPCs operated by enclave-backed nodes for verified private-state answers.
The non-TEE prototype service implements the same RPC boundary as a stand-in for those future
enclaves.

### Privacy Inbox

The Privacy Inbox is a public system contract or precompile at a protocol-defined address. A user or
relayer submits an encrypted signed private transaction through an ordinary L2 transaction.

The inbox MAY validate only public information, including:

- envelope version;
- key or epoch identifier;
- ciphertext size;
- public fee payment;
- structural encoding.

Semantic validation happens only after decryption. An accepted inbox call returns a fixed
acknowledgement of public admission, not a promise that private execution will succeed or that the
resulting private transition will be verified. The privacy-entrypoint execution charge is fixed and
independent of private execution; ordinary L2 calldata gas still applies to the outer transaction.

A reverted or otherwise unsuccessful inbox call is not a private input.

### Private executor

The private executor consumes privacy-relevant inputs from canonical public blocks. It decrypts
private transactions, authenticates their senders, and executes them with revm against the pEVM
database.

The executor processes relevant inputs in public transaction order and, where multiple relevant
logs occur in one transaction, log order. It does not inspect the public transaction pool or accept
an independent private transaction stream.

The executor uses the source public block number and timestamp as the private EVM block context.
Consequently, pEVM `BLOCKNUMBER` and `TIMESTAMP` may jump across public blocks that contain no
privacy inputs. The private transition index remains contiguous and advances only for blocks with
privacy inputs.

### Private database

The pEVM database is independent from the public World Chain database. It starts from an empty
private genesis and persists at least:

- accounts, nonces, balances, bytecode, and storage;
- private transaction receipts;
- private transition metadata;
- the last scanned canonical public block;
- the latest sealed private transition;
- any transition computed from `N` and awaiting commitment in `N+1`;
- sufficient revert or checkpoint data to unwind a public-chain reorganization.

Private execution for one source block occurs against an overlay. After executing `N`, the service
may atomically persist that overlay and its receipts as a sealed pending transition so it can recover
before `N+1`. It MUST NOT promote the pending transition to the canonical private head until the
matching commitment is verified from `N+1`. A process or storage failure MUST NOT leave a partially
applied or partially promoted private transition.

The prototype stores this database in plaintext on local disk. The storage boundary must still
allow a future TEE backend to persist authenticated encrypted snapshots and write-ahead logs through
an untrusted host.

### Privacy verifiers and RPCs

Prototype privacy verifiers use the same fixed development `MSK`, private STF version, and genesis.
They independently extract and execute block `N` inputs, retain the expected transition, and
validate its commitment after observing `N+1`. They expose private-state and verification results
through a dedicated RPC boundary. Long term, this verifier and RPC service runs inside an enclave;
the first prototype runs the same role outside a TEE.

This demonstrates deterministic replay, not a production trust model. A node without the prototype
key and pEVM state can validate public World Chain execution and the structural presence of registry
calls, but cannot independently validate the encrypted private transition. Such a node treats the
private commitment as opaque public data and relies on a privacy verifier when it needs an assertion
about private state. A verifier's rejection affects the private lineage and private RPC results, not
the canonical status of the public block.

## Public and private cursors

The implementation maintains two distinct cursors:

| Cursor | Advances when | Purpose |
| --- | --- | --- |
| `last_scanned_l2_block` | Every canonical public block | Ensures inputs are detected exactly once |
| `private_transition_index` | Only when a public block has privacy inputs | Versions actual pEVM transitions |

Scanning a public block with no privacy inputs MUST NOT open a private execution overlay, advance the
private transition index, encrypt a root, or produce a commitment.

A block containing only invalid private transactions still creates a private transition. Its state
root may equal its parent state root, but its input and receipt commitments differ and prove that the
inputs were processed rather than ignored.

## Transition lifecycle

### 1. Public admission and ordering

The public sequencer orders structurally valid Privacy Inbox and shield transactions through normal
public block production. Their inclusion does not depend on the private executor or privacy
verifier being available at that moment. Public execution returns the fixed acknowledgement and
continues independently of the eventual private result.

Private services scan the canonical public log and must not silently abandon an observed privacy
input. A service outage can make the corresponding private transition unavailable or invalid under
the exact `N -> N+1` rule, but it does not stop public block production or public validation.

### 2. Source block `N`

Block `N` executes public transactions normally. Successful inbox submissions and shield events
become the ordered private input batch for `N`. The public result of an inbox submission is only the
fixed acknowledgement.

The private executor and privacy verifiers record the final canonical input batch from `N`. They do
not derive private state from provisional Flashblocks or discarded payload candidates.

### 3. Private execution between `N` and `N+1`

The private executor applies the complete ordered batch to the pEVM state inherited from the latest
sealed private transition. It produces a transition that conceptually binds:

- private transition index;
- source L2 block number and hash;
- parent private transition commitment;
- commitment to the ordered private inputs;
- commitment to private receipts;
- plaintext private post-state root;
- key epoch and private STF version.

The exact serialization and hashing rules are defined by the private-transition format specification.

Invalid semantic private transactions, such as an unfunded transfer, produce failed private receipts
and no state mutation. They do not invalidate block `N` or the private batch as a whole.

The resulting overlay is sealed as the pending transition for `N`. It is available for commitment
construction and restart recovery, but it is not yet the canonical private head.

### 4. Root encryption

The executor derives the state-root encryption key from `MSK` using a domain-separated derivation
bound to source block `N`, not commitment block `N+1`. It encrypts the plaintext private state root
with a fresh nonce and authenticated context that includes the algorithm version, chain, key epoch,
private STF version, and source block context.

The public commitment contains the resulting ciphertext, authentication tag, nonce, and public
transition metadata. The plaintext private state root remains inside the private service boundary.

### 5. Commitment block `N+1`

To satisfy the private protocol schedule, the builder obtains the transition from the privacy
executor and injects one system transaction into `N+1` that writes the commitment derived from `N`
to the private-transition registry. Its position relative to other system transactions must be
deterministic. Failure to inject a valid commitment breaks the verified private lineage but does not
make the public block invalid.

`N+1` may also contain new privacy inputs. Those inputs form a separate transition whose commitment
is required in `N+2`; they are not included in the commitment for `N`.

After observing canonical block `N+1`, a privacy verifier:

1. Determines whether `N` contained privacy inputs.
2. Requires zero commitments for `N` if it contained none, otherwise exactly one.
3. Compares the source block, parent transition, input commitment, receipt commitment, key epoch,
   and STF version with its independently computed transition.
4. Derives the same per-source-block root key and re-encrypts its computed plaintext root using the
   transmitted nonce and authenticated context.
5. Marks the private commitment verified only if the ciphertext and authentication tag match.
6. Promotes the pending private transition only after all checks pass.

These checks do not participate in public block validation. A non-enclave node accepts `N+1` based
on public World Chain rules and may query a privacy-verifier RPC for the private transition's
verification status.

## Invalid commitment behavior

The prototype has no arbitrary asynchronous or catch-up commitment path. The following conditions
make the private transition sourced from `N` invalid for a prototype privacy verifier; they do not
make public block `N+1` invalid:

- `N` contained privacy inputs but `N+1` has no commitment for `N`;
- `N+1` contains more than one commitment for `N`;
- `N` contained no privacy inputs but `N+1` contains a commitment claiming `N` as its source;
- the commitment identifies the wrong source block number or hash;
- the parent private transition does not match the latest verified private transition;
- the input or receipt commitment differs from independent replay;
- the key epoch or private STF version is incorrect;
- state-root ciphertext, nonce, authentication tag, or authenticated context fails verification;
- transition indices are duplicated, skipped, or reordered.

There is no canonical `abortPrivateTransition` operation in the first prototype. If the required
commitment is missing or invalid in `N+1`, public World Chain continues and `N+1` remains canonical.
Privacy verifiers MUST keep the verified private head at its last valid transition, MUST NOT promote
the pending transition, and MUST NOT serve state derived from the failed transition as verified.
Later privacy inputs may still be publicly included and acknowledged, but no descendant transition
can become part of the verified private lineage until a protocol-defined recovery occurs.

The missed `N+1` slot cannot be repaired by publishing the same commitment at an arbitrary later
height. A future protocol may define deterministic recovery, abort, or timeout behavior, but doing
so affects censorship, settlement, and validity semantics and requires a separate design decision.

## Shield and unshield boundaries

A successful shield locks public funds and emits a uniquely identified deposit. The deposit is a
privacy-relevant input and mints the corresponding private balance in the transition sourced from
that public block. Deposit identity must prevent replay across restart and reorganization.

An unshield starts from a private-state fact: a private balance is burned or locked and a withdrawal
output is created. The initial prototype may use a trusted publisher to release public escrow after
the source transition commitment is verified. TEE attestations or ZK proofs do not secure this path
in the first prototype, and the trusted assumption must be visible in the implementation and tests.

## Key hierarchy

The prototype loads one fixed 32-byte development `MSK`. All derived keys use versioned,
domain-separated derivations:

| Derived material | Scope | Prototype use |
| --- | --- | --- |
| Transaction encryption keypair | Fixed prototype key epoch | Encrypt and decrypt signed private transactions |
| State-root encryption key | One source block with privacy inputs | Encrypt the private post-state root committed in `N+1` |
| Persistence encryption key | One snapshot or WAL emission | Reserved and tested now; used when persistence crosses a TEE boundary |

The transaction encryption public key is available through the public protocol entrypoint or its
registry. The secret key and all symmetric derived keys are private-service configuration.

Although the prototype database is plaintext, the persistence derivation must remain distinct from
transaction and state-root encryption. Reusing one derived key across these domains is forbidden.

## Reorganizations

Private transitions follow the canonical World Chain branch. Transition identity binds the source
block hash, not only its height.

If a reorganization removes a source block with privacy inputs, the private service unwinds to the
latest transition whose source block remains canonical. It discards any pending commitment derived
from the removed branch, scans the replacement branch, and executes replacement inputs in canonical
order. A commitment computed for an orphaned source hash cannot validate on the replacement branch.

The exact `N -> N+1` rule applies independently on the replacement branch.

## Gas and observable behavior

The prototype uses fixed, low private gas parameters defined as protocol constants shared by the
builder and all privacy verifiers. Their exact values are frozen with the private transaction format.
Private execution gas usage is recorded only in private receipts and cannot change the public inbox
result or public entrypoint execution charge.

The outer public transaction still pays normal World Chain transaction, calldata, and DA costs. The
fixed acknowledgement rule prevents decrypted control flow, success, revert data, or private gas
usage from becoming a public execution side channel.

## Required end-to-end behavior

Every implementation milestone must add automated coverage against the native local World Chain
devnet. At minimum, the complete prototype test suite must demonstrate:

1. Encrypted submission, public acknowledgement, private decryption, and a failed unfunded transfer.
2. No private transition or root commitment for a public block without privacy inputs.
3. Exactly one valid commitment in `N+1` after privacy inputs in `N`.
4. Privacy-verifier rejection of missing, duplicate, unexpected, or invalid commitments without
   rejecting the corresponding public block.
5. Shielding, private minting, and a successful private value transfer.
6. Restart without duplicate input processing.
7. Branch-specific unwind and replay after a public reorganization.
8. Continued public block production and public validation while the private executor or verifier
   is unavailable, or after a missing or invalid private commitment.

## Deferred production work

The following features are explicitly outside the first prototype:

- TEE execution and hardware attestation;
- encrypted host persistence enforced by an enclave;
- dealer enclave setup, threshold share custody, and DKG ceremonies;
- key rotation and historical key recovery beyond the fixed development key;
- private contract deployment and contract execution;
- challenger admission and censorship-resistant challenge submission;
- confidential ZK fault-proof generation and adjudication;
- production unshield security and challenge windows;
- private-lineage recovery after a missing or invalid `N+1` commitment;
- production fee markets, capacity limits, and side-channel hardening.

These features must extend the deterministic transition model rather than silently changing its
ordering or commitment rules.

## References

- [Arc Privacy Sector paper](https://6778953.fs1.hubspotusercontent-na1.net/hubfs/6778953/PDFs/Whitepapers/Arc_Privacy_Sector%20(5).pdf)
- [Research Spike v2: Arc-Style Privacy Sector on OP Stack](https://app.notion.com/p/3908614bdf8c8075a4a6e09c244e1b2a)
