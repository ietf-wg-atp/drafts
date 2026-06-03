---
title: "Authenticated Transfer: Synchronization"
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
workgroup: "Authenticated Transfer"
keyword:
venue:
  group: "Authenticated Transfer"
  type: "Working Group"
  mail: "atp@ietf.org"
  github: "ietf-wg-atp/drafts"

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

# Relays {#relays}

The synchronization protocol is designed so that consumers do not need to obtain repository data directly from each repository's canonical host. Repository content is cryptographically signed by the account that owns it, allowing any third party to redistribute that content without compromising authenticity. This enables a class of intermediary services, called **relays**, that subscribe to upstream event streams and re-emit events to their own downstream consumers.

A relay typically:

- Subscribes to one or more upstream event streams from canonical hosts and/or other relays.
- Optionally validates and filters incoming events according to its own policies.
- Re-emits events to downstream consumers using the same protocol described in this document.
- Serves or redirects requests for full-repository data ({{ATREPO}}) on behalf of accounts whose content it redistributes.

Relays exist for a number of reasons, including: aggregating events from many canonical hosts into a single stream that downstream consumers can subscribe to once; offloading bandwidth and connection load from canonical hosts; and applying policy transformations such as filtering or moderation.

Together, these capabilities allow a relay to fulfill the full synchronization contract — real-time event subscription and re-synchronization — for its downstream consumers, without those consumers needing to locate or contact the canonical host directly.

A relay is just another producer from a downstream consumer's perspective. The consumer does not need to know whether its direct upstream is a canonical host or a relay; the validation rules in {{streaming-validation}} apply identically in either case.

# Accounts {#accounts}

Each service that participates in the synchronization protocol maintains an independent **hosting status** for every account whose content it redistributes.

Account identifiers themselves, and the resolution of an identifier to its current signing key, are defined in {{ATREPO}}; this section is concerned with the hosting status that producers maintain about accounts they redistribute, not with identity resolution.

## Hosting Status {#account-status}

An account is, at any point in time, either active or not — represented by an `active` boolean.

When `active` is false, an optional `status` string clarifies the reason. The known status values are:

- `deleted`: the user or host has deleted the account. Content SHOULD be removed from the service's infrastructure. Implied to be permanent, but MAY be reverted.
- `deactivated`: the user has temporarily paused the account. Content MUST NOT be redistributed but does not need to be deleted from infrastructure. Implied time-limited.
- `takendown`: the host or service has taken down the account. Implied to be permanent or long-term, but MAY be reverted.
- `suspended`: the host or service has temporarily paused the account. Implied time-limited.
- `desynchronized` (`active` MAY be true): the service has detected a problem synchronizing the account's repository and may be missing content.
- `throttled` (`active` MAY be true): the service has paused processing of new content for this account because a rate limit has been exceeded.

New status values MAY be defined in the future. Producers MAY emit `status` strings not listed above, and consumers MUST tolerate unrecognized values. Consumers SHOULD use the `active` boolean as the authoritative indicator of overall account visibility, treating the `status` string as clarification that may inform more specific behavior (for example, whether to delete cached data versus retain it pending reactivation).

Producers expose an HTTPS request-response operation that, given an account identifier, returns the producer's current hosting status for that account. This allows consumers to query the present state of an account without subscribing to the event stream — for example, when establishing initial state for an account they have not seen before, or when reconciling diverging upstream reports.

The wire-level details of the request, the URL path, and the response media type are not specified by this document. The currently-deployed binding is described separately.

## Account Status Propagation {#account-status-propagation}

Account hosting status is not cryptographically authenticated. Status propagates hop-by-hop on the real-time event stream: each redistributing service emits an `#account` event ({{account-events}}) to its own downstream consumers when its local hosting status for an account changes.

Intermediaries MAY override their upstream's status. For example, a relay may take down an account that an upstream still reports as active. Such overrides are propagated downstream as `#account` events from the intermediary.

When an upstream service is unreachable, downstream services SHOULD retain the previously reported status for some implementation-defined period rather than immediately changing the account to an inactive state. This preserves availability across short upstream outages.

When account status reported by different upstreams diverges (for example, due to differing moderation policies, or a transient network partition between an upstream and its own upstream), services MUST apply their own policies to reconcile. Querying the account's current authoritative hosting service directly is one way to resolve such ambiguity.

# Full Repository Retrieval {#full-sync}

A consumer establishes full synchronization by retrieving the complete state of a repository at a single point in time. This is the foundation for both initial synchronization and consumer re-synchronization ({{resync}}), and is also used by consumers that do not maintain a continuous subscription.

The retrieval is performed via HTTPS request to a producer's full-repository endpoint. Sync producers expose an operation that, given an account identifier, returns the current state of the named repository as a serialized stream conforming to {{ATREPO}}. The serialized stream's header points at the current commit; the stream contains the full repository graph reachable from that commit.

A producer MAY support additional request parameters that scope the response, including:

- A baseline revision, in which case the response is a diff representing the changes from that baseline to the current revision and follows the rules of {{diffs}}.
- A subset of repository paths, in which case the response contains only the blocks required to verify those paths.

A producer MAY redirect the request to another producer that holds the requested data — for example, a relay redirecting to the account's canonical host, or to a peer relay with the content cached. Consumers SHOULD follow such redirects when re-synchronizing per {{resync}}.

# Repository Diffs {#diffs}

A repository diff carries the data that changed between two repository revisions: the new commit, any new MST nodes, and any created or updated record blocks. Applying a diff to a known baseline reconstructs the complete repository state at the new revision. Diffs are used in two places in this protocol: as the response to a baselined full-repository retrieval ({{full-sync}}), and as the `blocks` payload of `#commit` events on the real-time stream ({{commit-events}}).

## Diff Format {#diff-format}

Diffs use the same serialization format as complete repositories, with the commit block serving as the root. A diff MUST include:

- The new commit block.
- All created and updated record blocks.
- All MST nodes in the current repository that did not exist in the baseline revision.

Required blocks MUST be included in the diff regardless of their presence in earlier repository history. For example, if an MST node was previously present in the repository, then deleted, and subsequently reintroduced during the range that the diff represents, the diff MUST include that block even though it appeared in prior revisions.

Deleted records and past versions of updated records are excluded from diffs.

With the exception of deleted record data, a diff MAY include additional blocks; receivers SHOULD ignore them.

## Stateful Diff Verification {#diff-stateful-verify}

Consumers that maintain a complete current copy of a repository can verify a diff directly: each created or updated record's hash chain can be checked from the new commit through the MST to the leaf, and the consumer's own state provides any context needed to confirm that the operation list is exhaustive (for example, by enumerating which keys disappeared between the consumer's prior state and the new state).

Stateless consumers — those that do not maintain a copy of the full repository — cannot enumerate deletions from the diff alone, since the diff does not contain the values of deleted records. For these consumers, the protocol provides operation inversion as a stateless verification mechanism.

## Operation Inversion {#operation-inversion}

In some cases, a diff is accompanied by an explicit operation list (`ops`) declaring the creates, updates, and deletes it represents. Operation inversion verifies that this declared list is accurate and exhaustive without requiring to any prior local state.

To invert a diff against its declared operations:

1. Parse the diff's `blocks` as a partial MST per {{ATREPO}}.
2. For each entry in `ops`, apply the inverse operation to the partial MST: `create` becomes `delete`, `delete` becomes `create`, and `update` reverts the value to the operation's `prev` field.
3. Compute the root hash of the resulting MST.
4. Compare the computed root hash to the previous root hash declared by the diff (the `prevData` field of a `#commit` event, or the equivalent baseline-revision root for diffs from {{full-sync}}).

If the hashes match, the operation list is accurate and exhaustive. If they differ, either the operation list is incomplete or the diff is internally inconsistent; in either case the diff MUST be rejected.

Producers of diffs intended to support operation inversion MUST include, in addition to the blocks required by {{diff-format}}, the MST nodes for keys directly adjacent (in lexicographic order) to mutated keys. Without these adjacent nodes, the inverse operation cannot be correctly applied to the partial MST. Diffs carried in `#commit` events ({{commit-events}}) MUST satisfy this requirement, since stateless consumers rely on operation inversion for verification.

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

Validating a `#commit` event establishes both that the event is internally consistent (its declared operations match the diff it carries) and that the consumer can apply it to its existing state without missing intervening events.

For each `#commit` event received, consumers MUST perform the following steps:

1. Verify wire-level fields: that the frame parses as deterministic CBOR, that the payload satisfies the schema in {{commit-events}}, and that the size limits in {{commit-events}} are not exceeded.
2. Apply operation inversion to the event's `blocks` and `ops`, using the event's `prevData` as the expected previous root, per {{operation-inversion}}. If inversion fails, the event MUST be rejected.
3. Verify the commit signature using the signing key resolved from the account identifier, as defined in {{ATREPO}}.
4. Confirm that the event's `rev` is strictly greater than the previously observed `rev` for this account.
5. Cross-check the event's `prevData` field against the consumer's locally tracked `data` for this account. If they differ, the consumer has become desynchronized for this account and MUST initiate re-synchronization as defined in {{resync}}.

A signature failure at step 3 MAY indicate a recent key rotation rather than a malicious commit. Consumers SHOULD refresh the cached identity for the account once and re-attempt verification before treating the failure as a hard rejection.

## Re-synchronization {#resync}

When a consumer detects desynchronization, either through a disjunction in commit history or a `sync` event that does not match their local state, they must perform a complete re-synchronization process to restore consistency with the current repository state.

Re-synchronization requires fetching and processing the full repository structure, though the record contents themselves are optional depending on the consumer's needs. If the repository data is delivered in pre-order traversal, it can be validated incrementally as it streams in, producing a mapping of keys to record hashes that represents the complete repository state.

This key-to-hash mapping can be compared against existing local state to identify discrepancies and verify the integrity of the re-synchronization. Once validated, this mapping establishes the new baseline state against which future commit events can be applied.

Consumers SHOULD prefer requesting full repository data from their direct upstream rather than the canonical host for the repository. Direct upstreams can coalesce or cache concurrent requests and redirect consumers to other sources where appropriate, reducing load on canonical hosts during correlated re-synchronization events.

During the re-synchronization process, any incoming commit events for the repository should be buffered rather than processed immediately. Once re-synchronization completes successfully, these buffered commits can be validated and applied in sequence to bring the consumer fully up to date with the current repository state.

# Security Considerations {#security}

General security considerations for the repository format itself, including CBOR processing limits and MST structural validation, are covered in {{ATREPO}}.

## Resource Abuse {#security-resources}

A producer that routinely emits sync events forces consumers into full re-synchronization, which is substantially more expensive than processing commit events. Similarly, a producer that rapidly updates the same key or issues many small commits can amplify event volume and bandwidth costs on downstream consumers. Consumers should apply rate limits and bandwidth budgets per repository, and may disconnect or deprioritize producers whose event patterns appear abusive.

## Repository Rewinds {#security-rewinds}

Intermediaries that relay firehose events can present a consumer with an outdated view of a repository by replaying older commits or declining to forward newer ones. The `rev` field on each commit can help consumers detect this: a received commit whose `rev` is not greater than the most recently observed `rev` for that repository may indicate that the producer is serving a rewound view. In situations such as this, consumers can cross-check the latest observed `rev` against the canonical host for the repository.

## Server-Side Request Forgery {#security-ssrf}

Several aspects of synchronization involve following URLs or host endpoints derived from untrusted input: account-identifier resolution, retrieval of full repository data from a hosting service, and following redirects between hosts. Consumers MUST validate URLs derived from untrusted input before issuing requests, including any URLs reached via HTTP redirects. In particular, requests to internal-network addresses, loopback addresses, and link-local addresses MUST be rejected unless explicitly permitted by configuration.

## Validation Responsibility {#security-validation-responsibility}

Intermediaries that relay events MAY apply some validation checks (for example, signature verification or size enforcement) before relaying. Consumers MUST NOT treat upstream relaying as evidence of validity: every consumer is ultimately responsible for performing the verification rules in {{streaming-validation}} on each event it processes.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO: acknowledge
