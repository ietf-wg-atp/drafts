---
title: "Authenticated Transfer Synchronization"
abbrev: "AT Sync"
category: std

docname: draft-holmgren-at-synchronization-latest
submissiontype: IETF
number:
updates:
date:
consensus:
v: 0
area: "Applications and Real-Time Area"
workgroup:
keyword:
venue:
  github: "bluesky-social/ietf-drafts"

author:
 -
    fullname: Daniel Holmgren
    organization: Bluesky Social
    email: daniel@blueskyweb.xyz
 -
    fullname: Bryan Newbold
    organization: Bluesky Social
    email: bryan@blueskyweb.xyz

normative:
  RFC6455:
  ATREPO:
    title: "Authenticated Transfer Repository"
    date: draft-holmgren-at-repository
    author:
      -
        fullname: Daniel Holmgren
        organization: Bluesky Social
      -
        fullname: Bryan Newbold
        organization: Bluesky Social

informative:
  MST:
    title: "Merkle Search Trees: Efficient State-Based CRDTs in Open Networks"
    date: October 2019
    target: https://inria.hal.science/hal-02303490/document
    author:
      -
        fullname: Alex Auvolat
      -
        fullname: François Taïani
  AT-ARCH:
    title: "Authenticated Transfer: Architecture Overview"
    date: draft-newbold-at-architecture
    author:
      -
        fullname: Bryan Newbold
        organization: Bluesky Social
      -
        fullname: Daniel Holmgren
        organization: Bluesky Social
...

--- abstract

This document specifies the synchronization protocol for Authenticated Transfer (AT), a protocol for cryptographically-verifiable storage and distribution of structured user-controlled data. It defines synchronization mechanisms that allow efficient distribution of repository changes to interested parties.

This document builds on the AT repository format specified in {{ATREPO}}. Overall network architecture is described further in {{AT-ARCH}}.

--- middle

# Introduction {#intro}

The Authenticated Transfer (AT) synchronization protocol addresses the challenges of building decentralized applications that require consistent data replication across distributed multi-party infrastructure. In the AT model, user data is stored in cryptographically signed repositories as specified in {{ATREPO}}. The synchronization system provides efficient mechanisms for propagating repository state changes across the network, supporting both real-time streaming updates and bulk synchronization scenarios. The protocol can detect dropped or withheld updates and provides cryptographic proofs for all operations, including record deletions, ensuring that consumers can maintain accurate and complete views of repository state.

The AT synchronization model operates on the principle that any participant can independently verify repository updates. This allows sync to occur between any client and server without requiring that the server is a canonical or trusted host.

AT supports multiple synchronization patterns: full repository synchronization for complete replicas, partial synchronization for specific record subsets, and proof-only synchronization for cryptographic verification without content retrieval.

The typical synchronization workflow establishes baseline state through full synchronization, then maintains currency through incremental updates. Full synchronization is performed by fetching a complete serialized repository over HTTPS, as specified in {{ATREPO}}.

This document combines two transport modes. Real-time updates are delivered through a long-lived WebSocket {{RFC6455}} connection over which the producer streams binary event frames to the consumer. Full-state transfer is performed via HTTPS retrieval of a serialized repository as defined in {{ATREPO}}.

The event-stream model defined here — a sequence of typed frames carrying repository diffs and metadata — is the substantive content of the synchronization protocol. The WebSocket framing in {{realtime}} is the currently-deployed binding for that model. The same event-stream semantics could be defined over other transports without altering the validation and re-synchronization rules in this document.

# Repository Diffs {#diffs}

Repository diffs enable efficient synchronization by containing only the data that changed between two repository revisions. A diff includes the commit object, MST nodes, and records that differ between an older baseline revision and the current revision. Applying a diff to the baseline repository reconstructs the complete current repository state.

Diffs use the same serialization format as complete repositories, with the commit block serving as the root. A diff must include:

- The new commit block
- All created and updated record blocks
- All MST nodes in the current repository that did not exist in the baseline revision
- MST nodes for keys directly adjacent (in lexicographic order) to mutated keys, where required to support operation inversion (see {{streaming-validation}})

Required blocks must be included in the diff regardless of their presence in earlier repository history. For example, if an MST node was previously present in the repository, then deleted, and subsequently reintroduced during the range that the diff represents, then the diff must include that block even though it appeared in prior revisions.

Deleted records and past versions of updated records are excluded from diffs.

With the exception of deleted record data, the diff may include additional blocks which receivers should ignore.

# Diff Verification Limitations {#diff-limits}

Repository diffs present verification challenges for consumers who do not maintain complete repository state. These consumers often wish to authenticate repository content and utilize records without persisting the entire repository structure, making diffs an attractive option for lightweight verification.

Diffs partially support this use case by providing a signed commit and the relevant portions of the Merkle tree, creating a verifiable proof chain for record creations, updates, and deletions. When a recipient possesses both a diff and a corresponding list of operations, they can use the diff contents to cryptographically verify that the operations are authentic.

However, observers without knowledge of the complete baseline repository state cannot reliably enumerate all operations by examining the diff contents alone. While comprehensive diffs  reveal created or updated records by traversing to the leaf nodes, they provide no information about deletion operations that occurred during the period that the diff represents.

This means that while diffs enable verification of a known operation list, they cannot be used to exhaustively reconstruct the complete operation list from diff contents alone. However, if a recipient has a complete repository structure from some prior revision and receives a diff representing changes since that revision, they can compute the complete set of operations that occurred between the two versions.

This asymmetry means diffs alone cannot substitute for complete state tracking when comprehensive operation enumeration is required. An efficient mechanism for cross-verification of a diff and enumerated operation list against the prior repository commit state is described in {{streaming-validation}}.

# Real-time synchronization {#realtime}

AT supports real-time synchronization, enabling applications to receive repository updates with minimal latency through a pull-based WebSocket {{RFC6455}} connection.

Real-time streams of repository updates are often referred to as the “firehose”. The firehose delivers events containing repository diffs along with supporting metadata necessary for verification and processing.

Each event includes a monotonic cursor that establishes a total ordering across all repository changes from a given host. This ordering enables reliable event replay and ensures that consumers can maintain consistent state even when reconnecting after network interruptions.

AT allows consumers to maintain fully-verified copies of repository records without storing the underlying Merkle tree structure, providing an efficient method for applications that need authenticated content access without the overhead of complete repository replication.

## Wire Protocol {#wire}

A consumer establishes a WebSocket connection {{RFC6455}} to the producer's stream endpoint. TLS MUST be used for any internet-facing deployment; cleartext WebSocket connections are intended only for local development.

Once the connection is established, the producer sends a sequence of binary WebSocket frames to the consumer. Each frame carries an event. Servers SHOULD ignore any frames sent by the consumer; the stream defined in this document is server-to-client only.

### Frame Format {#frame-format}

Each binary frame contains two CBOR-encoded objects concatenated together: a header followed by a payload. Both objects MUST follow the deterministic CBOR encoding rules defined in {{ATREPO}}.

The header contains:

- `op` (integer, REQUIRED): the frame operation. The value `1` indicates a normal message; the value `-1` indicates an error.
- `t` (string, REQUIRED when `op = 1`): the message-type name, prefixed with `#`. For example, `#commit` for a commit event.

The payload is a CBOR object whose schema is determined by the message type indicated in the header. Payloads are always CBOR objects, never arrays or scalars.

Producers MAY include additional fields beyond those defined for a given message type, and consumers MUST tolerate unknown fields. Strict schema validation is not performed on stream messages.

A frame MUST NOT exceed 5 MB in total size, inclusive of the header, payload, and CBOR encoding overhead.

### Error Frames {#error-frames}

When `op` is `-1`, the frame is an error frame. The payload contains:

- `error` (string, REQUIRED): a short machine-readable error name.
- `message` (string, OPTIONAL): a human-readable description of the error.

After sending an error frame, the producer MUST close the WebSocket connection.

### Pre-Upgrade HTTP Errors {#pre-upgrade-errors}

If the producer rejects the WebSocket upgrade request itself, it responds with a standard HTTP status code rather than an error frame. Response bodies SHOULD be JSON containing `error` and `message` fields matching the error-frame schema, but consumers MUST tolerate other body content.

## Cursors and Resumption {#cursors}

Real-time synchronization streams include per-message cursors to improve transmission reliability. Cursors are positive integers that increase monotonically across the stream. Cursor semantics are flexible, and they may contain arbitrary gaps between consecutive messages.

Consumers track the last cursor value they successfully processed and can specify this cursor when reconnecting to receive any missed messages within the provider's rollback window. Providers maintain no persistent consumer state across connections, relying entirely on the cursor values supplied by consumers during reconnection.

The scope of a cursor is the (host, endpoint) pair: a cursor value is meaningful only when reconnecting to the same host and stream endpoint that issued it. Cursor values MUST NOT be re-used: if a producer must reset its cursor state for any reason, it MUST resume issuing cursors at a value greater than any value it has previously issued on that endpoint.

Cursor values are integers in the range `[1, 2^53)`. The upper bound is chosen so that cursors are exactly representable in 64-bit IEEE-754 floating point.

Stream behavior depends on the cursor value specified during connection:

- **No cursor specified**: The provider begins transmitting from the current stream position, providing only new messages generated after the connection is established.
- **Future cursor**: When the requested cursor exceeds the current stream cursor, the provider sends an error message and closes the connection.
- **Cursor within rollback window**: The provider transmits all persisted messages with cursor numbers greater than or equal to the requested cursor, then continues with the real-time stream once caught up.
- **Cursor older than rollback window**: The provider sends an informational message indicating that the requested cursor is too old, then begins transmission at the oldest available event, sends the entire rollback window, and continues with the real-time stream.
- **Cursor value of 0**: The provider treats this as a request for the complete available history, starting at the oldest available event, transmitting the entire rollback window, then continuing with the real-time stream.

## Event Types {#event-types}

The real-time stream delivers four types of events: `#commit`, `#sync`, `#identity`, and `#account`.

### Common Fields {#event-common}

The following fields are common to all event payloads:

- `seq` (integer, REQUIRED): the sequence number (cursor; see {{cursors}}) of this event.
- `did` (string, REQUIRED): the account identifier of the repository this event concerns. For historical reasons the `#commit` event uses the field name `repo` rather than `did` for this purpose; the value has the same meaning.
- `time` (string, REQUIRED): an ISO 8601 datetime string indicating when the event was emitted. This timestamp is informational and is not authoritative for any verification purpose.

### `#commit` Events {#commit-events}

A `#commit` event represents an atomic set of repository modifications and consists of a repository diff combined with supporting metadata.

The payload contains:

- `seq` (integer, REQUIRED): see {{event-common}}.
- `repo` (string, REQUIRED): the account identifier of the repository (see {{event-common}}; this is the historical name of the `did` field).
- `time` (string, REQUIRED): see {{event-common}}.
- `rev` (string, REQUIRED): the new revision identifier of the repository after these modifications.
- `since` (string, REQUIRED, nullable): the revision identifier of the repository before these modifications. MAY be null only for the first commit of a repository.
- `commit` (hash reference, REQUIRED): hash reference to the new commit object.
- `blocks` (byte string, REQUIRED): the serialized diff (as defined in {{diffs}}) carrying all blocks required to verify the operations in this event.
- `ops` (array, REQUIRED): an ordered list of repository operations represented by this event. Each entry is an object containing:
    - `action` (string, REQUIRED): one of `create`, `update`, or `delete`.
    - `path` (string, REQUIRED): the repository path being mutated.
    - `cid` (hash reference, REQUIRED, nullable): the hash reference of the new record at this path, or `null` for `delete` actions.
    - `prev` (hash reference, OPTIONAL): the hash reference of the prior record at this path. Present for `update` and `delete` actions; absent for `create`.
- `prevData` (hash reference, REQUIRED): the root hash of the repository's MST in the previous revision. Used for operation-inversion validation as described in {{streaming-validation}}.
- `tooBig` (boolean, REQUIRED): retained for compatibility with earlier versions of this protocol. Producers MUST emit this field with the value `false`. Consumers MUST ignore the field's value.
- `blobs` (array, REQUIRED): retained for compatibility with earlier versions of this protocol. Producers MUST emit this field as an empty array. Consumers MUST ignore the field's contents.

A `#commit` event MUST contain no more than 200 entries in `ops`. The `blocks` field MUST NOT exceed 2 MB. Any single record block within `blocks` MUST NOT exceed 1 MB. Mutations exceeding these limits MUST be communicated through `#sync` events instead.

A `#commit` event with an empty `ops` array (e.g., a commit issued solely to advance `rev` after a key rotation) is valid.

### `#sync` Events {#sync-events}

A `#sync` event declares the current state of a repository, regardless of the previous state. Sync events are emitted when commit-event continuity cannot be maintained: large mutations exceeding the limits in {{commit-events}}, recovery from data loss or corruption, or account migration between hosting providers.

The payload contains:

- `seq` (integer, REQUIRED): see {{event-common}}.
- `did` (string, REQUIRED): see {{event-common}}.
- `time` (string, REQUIRED): see {{event-common}}.
- `rev` (string, REQUIRED): the current revision identifier of the repository.
- `blocks` (byte string, REQUIRED): a serialized stream containing the current commit object only. Receivers reconstruct full repository state by fetching the complete repository as described in {{resync}}.

A `#sync` event provides a reset point that signals consumers to resynchronize against the current authoritative state without requiring knowledge of the intervening changes.

### `#identity` Events {#identity-events}

An `#identity` event indicates a possible change to the resolution result of an account identifier. Consumers SHOULD invalidate any cached identity metadata for the named account on receipt of this event and re-resolve the identifier.

The payload contains:

- `seq` (integer, REQUIRED): see {{event-common}}.
- `did` (string, REQUIRED): see {{event-common}}.
- `time` (string, REQUIRED): see {{event-common}}.
- `handle` (string, OPTIONAL): the account's current handle, communicated non-authoritatively. 

`#identity` events are best-effort: producers MAY emit them redundantly when no underlying change has occurred, and MAY fail to emit them when a change has occurred. Consumers SHOULD NOT rely on `#identity` events as the sole signal of identity change.

### `#account` Events {#account-events}

An `#account` event indicates a change in account hosting status at the emitting service.

The payload contains:

- `seq` (integer, REQUIRED): see {{event-common}}.
- `did` (string, REQUIRED): see {{event-common}}.
- `time` (string, REQUIRED): see {{event-common}}.
- `active` (boolean, REQUIRED): whether the account is currently active on the emitting service.
- `status` (string, OPTIONAL): a short status code describing the account state. See {{account-status}} for known values and semantics.

## Commit Validation {#streaming-validation}

Commit validation occurs through a two-step process that ensures both the validity of the repository transition and the consumer's resulting synchronization state. First, the consumer validates that the commit represents a valid transition from a previous repository revision (`revA`) to the new revision (`revB`). Second, the consumer confirms that they last observed the repository at `revA`. Together, these steps establish that the repository is now definitively at `revB`.

### Operation Inversion {#operation-inversion}

The validation process inverts all operations against the partial MST provided in the diff. That is, each “create” operation will be inverted as a “delete” operation on the same key and applied to the tree. Each “delete” will become a “create” of the same record, and every “update” will be updated back to the previous value.

If the operation list is complete and accurate, applying the inverse operations will reconstruct the tree state as it existed before the commit. The hash of this reconstructed tree must match the previous root hash of the MST as specified in the commit event. If the hashes match then the provided list of operations is accurate and exhaustive.

Because the previous MST root hash is included in the commit event, commits can be validated for internal consistency independent of any local state. If the operation inversion process fails to produce a tree hash matching the declared previous root, the entire commit event should be treated as invalid.

If the commit is internally consistent but its declared previous root does not match the previous MST root stored locally, then the consumer has become desynchronized, indicating missed events or a disjunction in the producer’s commit history.

### Validation Algorithm {#validation-algorithm}

For each `#commit` event received, consumers MUST perform the following steps:

1. Verify wire-level fields: that the frame parses as deterministic CBOR, that the payload satisfies the schema in {{commit-events}}, and that the size limits in {{commit-events}} are not exceeded.
2. Parse the `blocks` byte string as a partial MST per {{ATREPO}}.
3. For each entry in `ops`, apply the inverse operation to the partial MST: `create` becomes `delete`, `delete` becomes `create`, `update` reverts the value to `prev`.
4. Compute the root hash of the resulting MST.
5. Compare the computed root hash to `prevData`. If they do not match, the event is internally inconsistent and MUST be rejected.
6. Verify the commit signature using the signing key resolved from the account identifier, as defined in {{ATREPO}}.
7. Confirm that the event's `rev` is strictly greater than the previously observed `rev` for this account.
8. Cross-check the event's `prevData` field against the consumer's locally tracked `data` for this account. If they differ, the consumer has become desynchronized for this account and MUST initiate re-synchronization as defined in {{resync}}.

A signature failure at step 6 MAY indicate a recent key rotation rather than a malicious commit. Consumers SHOULD refresh the cached identity for the account once and re-attempt verification before treating the failure as a hard rejection.

## Re-synchronization {#resync}

When a consumer detects desynchronization, either through a disjunction in commit history or a `sync` event that does not match their local state, they must perform a complete re-synchronization process to restore consistency with the current repository state.

Re-synchronization requires fetching and processing the full repository structure, though the record contents themselves are optional depending on the consumer's needs. If the repository data is delivered in pre-order traversal, it can be validated incrementally as it streams in, producing a mapping of keys to record hashes that represents the complete repository state.

This key-to-hash mapping can be compared against existing local state to identify discrepancies and verify the integrity of the re-synchronization. Once validated, this mapping establishes the new baseline state against which future commit events can be applied.

During the re-synchronization process, any incoming commit events for the repository should be buffered rather than processed immediately. Once re-synchronization completes successfully, these buffered commits can be validated and applied in sequence to bring the consumer fully up to date with the current repository state.

# Security Considerations {#security}

General security considerations for the repository format itself, including CBOR processing limits and MST structural validation, are covered in {{ATREPO}}.

## Resource Abuse {#security-resources}

A producer that routinely emits sync events forces consumers into full re-synchronization, which is substantially more expensive than processing commit events. Similarly, a producer that rapidly updates the same key or issues many small commits can amplify event volume and bandwidth costs on downstream consumers. Consumers should apply rate limits and bandwidth budgets per repository, and may disconnect or deprioritize producers whose event patterns appear abusive.

## Repository Rewinds {#security-rewinds}

Intermediaries that relay firehose events can present a consumer with an outdated view of a repository by replaying older commits or declining to forward newer ones. The `rev` field on each commit can help consumers detect this: a received commit whose `rev` is not greater than the most recently observed `rev` for that repository may indicate that the producer is serving a rewound view. In situations such as this, consumers can cross-check the latest observed `rev` against the canonical host for the repository.
