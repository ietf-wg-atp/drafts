---
title: "Authenticated Transfer: Repository and Synchronization"
abbrev: "AT Repo"
category: std

docname: draft-holmgren-at-repository-latest
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
  CBOR: RFC8949
  RFC7049: RFC7049
  RFC3986: RFC3986
  RFC4648: RFC4648
  RFC6455: RFC6455
  WEBASSEMBLY:
    title: WebAssembly Core Specification
    date: March 2026
    target: https://www.w3.org/TR/wasm-core-2
    author:
      -
        fullname: Andreas Rossberg
  SEC2:
    title: "SEC 2: Recommended Elliptic Curve Domain Parameters"
    date: January 2010
    target: https://www.secg.org/sec2-v2.pdf
    author:
      -
        organization: Standards for Efficient Cryptography Group

informative:
  MSTPAPER:
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
    date: March 2026
    target: https://datatracker.ietf.org/doc/draft-newbold-at-architecture/
    author:
      -
        fullname: Bryan Newbold
        organization: Bluesky Social
      -
        fullname: Daniel Holmgren
        organization: Bluesky Social
  DASL-CAR:
    title: "DASL: Content Addressable aRchives (CAR)"
    target: https://dasl.ing/car.html
  DRISL:
    title: "DRISL — Deterministic Representation for Interoperable Structures & Links"
    target: https://dasl.ing/drisl.html
...

--- abstract

This document specifies a repository data structure and synchronization mechanisms for public data as part of the Authenticated Transfer Protocol (ATP). It describes encoding formats for both individual data records and entire repositories. The repository data structure is content-addressable and cryptographically authenticated. For synchronization, it specifies both a low-latency streaming protocol over WebSocket, and a full-repository fetch mechanism over HTTP.

--- middle

# Introduction {#intro}

The Authenticated Transfer Protocol (ATP) enables the creation of decentralized networks for publication of self-certifying data. An introduction to the overall protocol architecture is given in {{AT-ARCH}}.

User accounts publish structured data records to the network by including them in their public repository. Records within a repository are identified by a unique path and current content version (hash). Records can be created, updated, and deleted at any time. Repositories contain the complete set of current records for the account, and do not include or reveal the existence of previous content.

The repository structure includes the account's persistent identifier, and the overall repository structure is cryptographically signed. The authenticity of the entire repository can be verified by resolving the account identifier to the current public key. Data records are not signed individually. Details of account identifier systems and their resolution process are out of scope for this document.

Large binary data such as images and media files are not stored directly within repositories. Instead, such data is stored externally and referenced in records by a hash link.

The protocol provides efficient synchronization mechanisms for propagating public repository state changes across the network, supporting both low-latency streaming updates and bulk synchronization scenarios. Synchronization can take place between any two parties, from an upstream publisher to a downstream consumer. Intermediate parties can redistribute ("relay") data, and consumers can cryptographically verify the integrity and authenticity of received repository data. This allows for flexibility in network topology to improve overall network resilience and efficiency.

Consumers can confirm the integrity over the entire repository to detect dropped or withheld updates. The protocol allows consumers to maintain partial replicas (eg, of only specific record types). It is also possible to request verifiable "inclusion proofs" for individual records on demand.

This document describes two synchronization mechanisms. Complete serialized repositories can be fetched over HTTP as a snapshot. Updates to one or more repositories can be distributed as a stream of messages. This document describes both a generic structure and semantics for streaming messages, and a specific WebSocket transport and message encoding scheme.

This document describes version `3` of the repository format.

# Repository Semantics {#repo-semantics}

Records within a repository are discrete units of structured data, identified by a unique path and versioned by content hash. Records conform to a generic data model, but the content, schema, and semantics of record is varied and application-specific. Records are grouped by type under "collections".

The current state of a repository is summarized in a signed "commit". Any change to the contents of the repository updates the current commit. Commits for an individual account's repository are serialized using a monotonically-increasing "revision" identifier.

Updates to repositories may include operations on multiple records in a batch mutation that results in a single signed commit. Implementations should apply practical limits on batch sizes to support efficient processing and distribution of repository changes.

## Account Identifiers {#account-ids}

Repository commit objects ({{commits}}) contain a persistent account identifier, which indicates the publisher of the repository. This account identifier can be resolved to obtain the current cryptographic public keys for the account, and those keys can be used to verify the authenticity of the repository and the records it contains.

The keys associated with an account may be rotated over time. The most recent commit must always be verifiable using the currently resolvable signing key. When rotating signing keys, a new repository commit must be created, even if the contents and structure of the repository remain unchanged.

This document does not include details or recommendations on account identifier systems.

## Revisions {#revs}

Repository commits include a revision field (`rev`) which acts as a logical clock for updates to the repository over time. The revision string is a Timestamp Identifier (TID) as described in {{tid}}.

Revisions may be used when comparing two versions of a repository to determine which is more recent. This is particularly relevant when synchronizing repositories indirectly, or from multiple sources over time.

If a commit TID value corresponds to a timestamp in the future (beyond a short period to accommodate clock drift) the commit SHOULD be ignored. This is to ensure that a newly published commit (with a TID corresponding to the current time) will reliably be accepted as current by the entire network.

# Repository Structure {#repo-structure}

Repositories are structured as a Merkle Search Tree ({{mst}}) with a cryptographically signed commit object referencing the tree root.

The MST structure provides several fundamental properties for repository operations. As a content-addressed structure, it enables efficient verification of data. The MST maintains lexicographic key ordering, enabling structural sharing of intermediate tree nodes for related records. It is probabilistically self-balancing, offering consistent performance characteristics. Additionally the MST exhibits unicity, meaning that any given set of keys and values will always produce the same tree structure and root hash regardless of insertion order.

Repository contents are encoded using deterministic CBOR serialization and organized as a directed acyclic graph where data objects reference each other through content hashes. These hash-identified data objects, referred to as "blocks," include three distinct types: commit objects, MST internal nodes, and data records.

## Record Paths {#repo-path}

Records within a repository are identified by a non-empty case-sensitive ASCII string called the "path". Records are stored sorted lexicographically by path, and the efficiency of some repository operations is impacted by sort order.

A path string is the combination of a collection type name and a record key, joined by a single forward slash character: `<collection>/<record-key>`. A path MUST consist of exactly two segments separated by `/`, with no leading or trailing slash.

Collection names use the Namespaced Identifier (NSID) syntax described in {{nsid}}. They have a prefix-ordered namespace structure, which means that records of the same collection are stored adjacently, and that collections under the same authority are grouped together.

Record keys uniquely identify records within a collection. Record keys are case-sensitive and MUST satisfy the following syntax:

- Allowed characters are ASCII alphanumerics (`A-Z`, `a-z`, `0-9`), period (`.`), hyphen (`-`), underscore (`_`), colon (`:`), and tilde (`~`)
- Length between 1 and 512 characters (inclusive)
- The literal values `.` and `..` are prohibited

The syntax of record keys may be constrained further on a per-collection basis at the application layer. A common choice is to use the Timestamp Identifier {{tid}} syntax, which results in lexicographic sorting by time within a collection. This means that "new" records are all grouped together within a given collection.

Note that both the NSID and record key string syntaxes are valid path components as defined in Section 3.3 of {{RFC3986}}. It is important to maintain this property.

## Commit Objects {#commits}

Commit objects serve as the authoritative root of each repository, establishing cryptographic ownership and providing a verifiable reference to the state of a repository at a particular point in time. Each commit is digitally signed by the repository account owner and contains metadata necessary for verification.

A commit object contains the following data fields:

- **`did`** (string, required): The resolvable account identifier associated with the repository as described in {{account-ids}}
- **`version`** (integer, required): Repository format version, fixed value of **`3`** for the current specification
- **`data`** (cid-link, required): Hash pointer to the root of the repository’s MST structure
- **`rev`** (string, required): Repository revision identifier that functions as a logical clock and must increase monotonically (see {{revs}}). Syntax MUST match {{tid}}.
- **`prev`** (cid-link, nullable): Optional pointer to the previous commit object in the repository's history chain. While included for backward compatibility with version 2 repositories, this field is typically `null` in version 3 implementations
- **`sig`** (byte array, required): Cryptographic signature over the commit contents.

Commit objects are signed by the key declared by the repository owner’s resolvable identifier. Neither the signature nor the signed commit object contains information about the curve type or specific public key used for signing. This information must be obtained by resolving the account identifier as described in {{account-ids}}.

The procedure for signing commit objects:

1. Encode the unsigned commit object (with `sig` field entirely absent) as CBOR
2. Sign the encoded bytes as described in {{crypto}}
3. Include the signature bytes in the `sig` field

To verify the signature, remove the `sig` field and encode the unsigned commit object as CBOR. Then verify the signature against those encoded bytes.

## Records {#records}

Records stored within a repository are always objects (or "maps") encoded as CBOR, following the data model and encoding rules described in {{data-model}}. Each record must include a top-level field named `$type` with a string value matching the collection type name (NSID) of the path that the record is stored at.

Invalid or corrupt data in individual records should not impact processing of the overall repository data structure, or the processing of other valid records in the same repository.

# Merkle Search Tree {#mst}

The Merkle Search Tree (MST) structure is deterministically reproducible from any given key-value mapping, where keys are non-empty byte strings (corresponding to a path) and values are hash link references to records. This deterministic construction ensures that identical input sets always produce the same root hash regardless of insertion order.

The tree's structural organization depends solely on the keys present, not on the record values they reference. When a record value changes, the new content hash propagates up through the tree nodes to the root, but the tree's shape and node organization remain unchanged.

The MST data structure was first published in {{MSTPAPER}}.

## Tree Structure {#mst-structure}

Each MST node contains a list of key-value entries and references to child subtrees. Entries and subtree links are maintained in lexicographic order, with all keys in a linked subtree falling within the range corresponding to that link's position. The ordering proceeds from left (lexicographically first) to right (lexicographically last).

Keys are assigned to tree levels based on a layer value computed from the key itself. Nodes at each level contain all keys with the corresponding layer value, while subtree links point to nodes containing keys that fall within specific lexicographic ranges but have lower layer values. Adjacent keys may appear within the same node, but adjacent subtrees must be separated by at least one key entry to prevent structural ambiguity.

## Layer Calculation {#mst-layer}

The layer for a given key is calculated using SHA-256 with a 2-bit grouping scheme that provides an average fanout of 4:

1. Compute the SHA-256 hash of the key (byte string) with binary output
2. Count the number of leading binary zeros in the hash
3. Divide by 2, rounding down to the nearest integer

Examples of layer calculation:

- `key1`: SHA-256 begins `100000010111...` → layer 0
- `key7`: SHA-256 begins `000111100011...` → layer 1
- `key515`: SHA-256 begins `000000000111...` → layer 4

When processing the MST structure, implementations must verify the layer assignment and ordering of keys. While this verification is most essential for untrusted inputs, implementations should perform these checks consistently regardless of data source. Additional validation of node size limits and other structural parameters is required to prevent resource exhaustion attacks, as detailed in Security Considerations ({{security}}).

## MST Construction Example {#mst-example}

The following is a Merkle Search Tree containing 9 records with keys A-I. Each key would include a pointer to some record hash, though that hash is irrelevant to the construction of the tree. Each asterisk (`*`) represents a hash pointer to the subtree under it.

For the sake of illustration assume the following layer calculations:

- `layer(D) = 2`
- `layer(A|E|I) = 1`
- `layer(B|C|F|G|H) = 0`

~~~aasvg
         *
         |
   -------------
  |      |      |
  *      D      *
  |             |
 ---          -----
|   |        |  |  |
A   *        E  *  I
    |           |
   ---        -----
  |   |      |  |  |
  B   C      F  G  H
~~~
{: #f-mst-example title="Example MST Structure"}

## Empty Nodes {#mst-empty-nodes}

An empty repository containing no records is represented as a single MST node with no entries. This is the only case where a node without entries is permitted.

Nodes that contain no key entries but do contain subtree links are allowed at intermediate positions, provided those subtrees eventually contain key entries. However, such nodes MUST NOT appear at the root position — the root MUST either contain key entries or be the special case of a completely empty repository. Similarly, nodes without key entries MUST NOT appear at leaf positions except for the empty repository case.

This structure ensures that nodes lacking key-value entries are pruned from the top and bottom of the tree while preserving intermediate nodes that maintain proper height relationships and prevent subtree links from skipping layers.

## MST Node Schema {#mst-nodes}

Given their prevalence through the repository structure, MST nodes require a compact binary representation for storage efficiency. Keys within each node use prefix compression, where each entry specifies the number of bytes it shares with the preceding key in the array. The first entry in each node contains the complete key with a prefix length of zero. This compression applies only within individual nodes and does not extend across node boundaries. The compression scheme is mandatory to ensure deterministic MST structure across all implementations.

MST nodes contain the following fields:

- `l` (hash link, nullable): Reference to a subtree node at a lower layer containing keys that sort lexicographically before all keys in the current node
- `e` (array, required): Ordered array of entry objects, each containing:
    - `p` (integer, required): Number of bytes shared with the previous entry in this node
    - `k` (byte string, required): Key suffix remaining after removing the shared prefix bytes
    - `v` (hash link, required): Reference to the record data for this entry
    - `t` (hash link, nullable): Reference to a subtree node at a lower layer containing keys that sort after this entry's key but before the next entry's key in the current node

Hash references appearing within an MST node — the `l` and `t` subtree links, and the `v` record link — MUST use the constrained content-hash format defined in {{cid-link}}.

## MST Node example {#mst-node-example}

The following example shows an MST node at layer 1 containing two subtree pointers and two key-value entries. The node contents in order are:

- Left subtree: hash link `0x01711220643b9326...`
- Entry: `key7` → record hash link `0x017112202d9aa87e...`
- Right subtree: hash link `0x0171122047e2886f...`
- Entry: `key10` → record hash link `0x0171122010b6da2c...`

This node would be encoded as follows:

~~~
{
  l: 0x01711220643b9326...
  e: [
    {
      p: 0,
      k: "key7",
      v: 0x017112202d9aa87e...
      t: 0x0171122047e2886f...
    },
    {
      p: 3,
      k: "10",
      v: 0x0171122010b6da2c...
      t: null
    }
  ]
}
~~~

# Repository Serialization Format {#serialization}

Repositories are serialized for transmission and storage as a concatenated sequence of block data, where blocks represent the CBOR-encoded records, MST nodes, and commit objects that comprise the repository structure. The serialization is prefixed with a header that identifies the root block, typically the repository's commit object.

Serialized repositories may contain partial repository state, such as when transmitting cryptographic proofs for specific records. In these situations, they may not include unrelated MST nodes or records outside the proof path.

The block-and-header layout described here is compatible with Content-Addressable archive (CAR) formats such as {{DASL-CAR}}.

## Header Format {#serialization-header}

The header is constructed by CBOR-encoding an object with the following fields:

- `version` (integer, required): Fixed value of `1`
- `roots` (array, required): Single-element array containing the hash link of the commit block

The CBOR-encoded header is prefixed with its byte length encoded as an unsigned LEB128 integer as described in Section 5.2.2 of {{WEBASSEMBLY}}.

## Block Format {#serialization-blocks}

Following the header, each repository block is serialized by concatenating:

1. The combined byte length of the following two components, encoded as an unsigned LEB128 integer
2. The block's content hash, prefixed with `0x01711220` as specified in {{cbor-encoding}}
3. The CBOR-encoded block data

~~~aasvg
|------- Header -------| |--------------------- Data --------------------|
 [ int | header block ]   [ int | hash | block ] [ int | hash | block ] …
~~~
{: #f-serialization title="Repo Serialization Layout"}

## Block Ordering {#serialization-ordering}

Producers SHOULD emit blocks in pre-order traversal of the included repository portion: header, commit object, root MST node, then a recursive depth-first interleaving of subtree nodes and the records they reference.

Preorder traversal enables streaming verification of repositories, allowing parsers to walk the MST structure and output key-to-record mappings while maintaining minimal MST state in memory. This approach supports efficient stream processing of large repositories without requiring complete buffering of the serialized data.

Parsers MUST tolerate other block orderings, duplicate occurrences of the same block, and additional unrelated blocks. Specifically:

- Duplicate blocks SHOULD be deduplicated rather than treated as an error.
- Dangling references — for example, hash links pointing to records or blobs that are not present in the serialized data — MAY be present and unresolvable; this is not an error in itself.
- Unrelated blocks not referenced by the repository structure SHOULD be ignored. Excessive quantities of such blocks MAY be treated as a form of resource abuse; see {{security}}.

# Account Hosting {#accounts}

Each node in the network which synchronizes, stores, and distributes repository data maintains hosting status for each account. The status indicates whether the account is overall "active", or has been temporarily or permanently removed from the network, in which case repository data should not be synchronized further. Hosting status can be set by the account holder themselves or their canonical hosting provider, and changes propagate to downstream consumers throughout the network. Each downstream service or node in the network may set a local inactive hosting status, declining to distribute that account's repository data.

Each account has a persistent identifier which can be resolved to both a public key and a canonical hosting location. Account identifier systems and their resolution mechanisms are out of scope for this document. Account hosting status is maintained and transmitted over the synchronization protocol, separate from the lifecycle of account identifiers.

The hosting status itself for accounts may always be redistributed, even for inactive accounts.

## Hosting Status {#account-status}

Account hosting status at any point in time can be summarized as the boolean state of being "active" or not. If the account hosting status is not active, repository data for that account MUST NOT be redistributed. Additional context may be provided using a "status" vocabulary, represented as a string. Account status is represented and transmitted as an `active` boolean and a `status` string together.

The defined status values and their meanings are:

- `deleted` (`active` is false): the user or host has deleted the account. Account data SHOULD be removed from the service's infrastructure within a reasonable time frame. Implied to be permanent, but MAY be reverted.
- `deactivated` (`active` is false): the user has temporarily paused the account. Account data MUST NOT be redistributed but does not need to be deleted from infrastructure. Implied time-limited.
- `takendown` (`active` is false): the host or service has taken down the account. Implied to be indefinite in duration, but MAY be reverted.
- `suspended` (`active` is false): the host or service has temporarily paused the account. Implied time-limited.

Two additional status values are relevant to the synchronization process itself, and do not imply that the overall hosting status is inactive:

- `desynchronized` (`active` MAY be true): the service has detected a problem synchronizing the account's repository and may be missing content.
- `throttled` (`active` MAY be true): the service has paused processing of new content for this account because a rate limit has been exceeded.

New status values may be defined in the future. Producers MAY emit `status` strings not listed above, and consumers MUST tolerate unrecognized values. Consumers MUST use the `active` boolean as the authoritative indicator of overall account visibility, treating the `status` string as clarification that may inform more specific behavior (for example, whether to delete cached data versus retain it pending reactivation).

Producers expose an HTTPS request-response operation that, given an account identifier, returns the producer's current hosting status for that account. This allows consumers to query the present state of an account without subscribing to the message stream — for example, when establishing initial state for an account they have not seen before, or when reconciling diverging upstream reports.

The details of the HTTPS request endpoint, the URL path, and the response media type are not specified by this document.

## Status Propagation {#account-status-propagation}

Account hosting status is not cryptographically authenticated. Status propagates hop-by-hop via streaming synchronization ({stream-sync}): each node emits an `#account` message ({{msg-account}}) to downstream consumers when the hosting status for an account changes. For intermediate synchronization nodes, this includes changes driven by an `#account` message received from an upstream.

Intermediaries MAY override their upstream's status. For example, a relaying node may take down an account that an upstream still reports as active. Such overrides are propagated downstream as `#account` messages from the intermediary.

When an upstream service is unreachable, downstream services SHOULD retain the previously reported status for some implementation-defined period rather than immediately changing the account to an inactive state. This preserves availability across short upstream outages, and increases resiliency of the network.

When account status reported by different upstreams diverges (for example, due to differing moderation policies, or a transient network partition between an upstream and its own upstream), services apply their own policies to reconcile. Querying the account's current authoritative hosting service directly is one way to resolve such ambiguity.

# Snapshot Sync {#snapshot-sync}

Consumers can retrieve a full serialized snapshot of an account's current repository at any point in time. This can be used to initialize synchronization state for the account during a bootstrap or backfill phase, or to re-synchronize ({{resync}}) and reconcile after any discontinuity in the streaming synchronization mechanism. It is also an option for applications and use-cases which do not require continuous updates or synchronization over time.

Producers expose an HTTP API endpoint which takes an account identifier as a parameter, and returns the serialized repository as the response body. If the account hosting status at the producer is not "active", this is indicated in an error response.

A producer MAY redirect the request to another producer that holds the requested data — for example, a relaying service redirecting to the account's canonical host, or to a mirroring service with the content cached. Consumers SHOULD follow such HTTP redirects when re-synchronizing per {{resync}}.


# Streaming Sync {#stream-sync}

This section describes a low-latency streaming synchronization mechanism which allows consumers to receive repository updates with minimal latency through a pull-based WebSocket {{RFC6455}} connection. Streams can contain updates from many distinct account repositories, even aggregating all updates in the network. In addition to repository updates, they include updates to account hosting status and network identities. Whether encompassing the full network or any subset of it, a stream is often referred to as a "firehose".

To ensure reliable delivery, each message on a given stream is given a monotonically-increasing sequence number. Consumers can track their progress on processing messages and reconnect to the stream using the last-processed sequence number as a cursor value as needed. Producers can maintain a "backfill window" of recently transmitted messages. All consumers receive the same messages in the same order with the same sequence numbers.

The stream synchronization mechanism allows consumers to maintain complete indices of authenticated repository contents without needing to store complete copies of the repository MST structure. This significantly reduces storage overhead.

## Repository Diffs {#diffs}

A repository diff carries the data that changed between two repository revisions: the new commit, any new MST nodes, and any created or updated record blocks. Applying a diff to a copy of the prior repository state results in the complete repository at the new revision. Repository diffs are used in the streaming synchronization mechanism.

The commits within diffs are signed. Receiving parties can always verify those signatures, and the integrity of the blocks in the diff itself. But unless the receiving party has a full copy of the repository just prior to the diff, it can not verify the overall integrity of the diff or the final state of the repository. In particular, if record deletion operations were included in the diff, the receiving party can not enumerate or verify which records were impacted just from the diff.

This section describes an "operation inversion" mechanism which allows receiving parties to verify the integrity of diffs when combined with metadata about the record-level operations encapsulated by the diff.

### Diff Serialization Format {#diff-format}

Diffs use the same serialization format as complete repositories (described in {{serialization}}), with the commit block serving as the root. A diff MUST include:

- The new commit block.
- All created and updated record blocks.
- All MST nodes in the current repository that did not exist in the prior revision.

Required blocks MUST be included in the diff regardless of their presence in earlier repository history. For example, if an MST node was previously present in the repository, then deleted, and subsequently reintroduced during the range that the diff represents, the diff MUST include that block.

Deleted records and prior versions of updated records are excluded from diffs.

With the exception of deleted record data, a diff MAY include additional blocks; receivers SHOULD ignore them.

### Operation Inversion {#operation-inversion}

A diff can be accompanied by an explicit operation list declaring the record-level creates, updates, and deletes it represents, along with a claimed previous repository tree root hash. Operation inversion verifies that this declared list is accurate and complete.

To invert a diff against its declared operations:

1. Extract the diff's MST nodes into a partial tree structure
2. For each entry in the operation list, apply the inverse operation to the partial MST: each "create" becomes a "delete", "delete" becomes "create", and "update" reverts the record value to the previous version
3. Compute the root hash of the resulting MST
4. Compare the computed root hash to the claimed previous root hash

If the hashes match, the operation list is accurate and exhaustive. If they differ, either the operation list is incomplete or the diff is internally inconsistent; in either case the diff MUST be rejected.

Producers of diffs intended to support operation inversion MUST include, in addition to the blocks required by {{diff-format}}, the MST nodes for keys directly adjacent (in lexicographic order) to mutated keys. Without these adjacent nodes, the inverse operation cannot be correctly applied to the partial MST. Diffs carried in streaming synchronization messages ({{msg-commit}}) MUST satisfy this requirement, since stateless consumers rely on operation inversion for verification.

Receivers can track repository root hash values for each account and verify the claimed root hashes provided with the diff. This fixed-size state is significantly smaller than maintaining full repository data.

## WebSocket Transport {#websocket}

A consumer (client) establishes a WebSocket connection {{RFC6455}} to the producer's (server) stream endpoint. Secure WebSockets (using TLS) MUST be used for any internet-facing deployment.

Once the connection is established, the producer sends a sequence of binary WebSocket frames to the consumer. Each frame carries a single message. Servers SHOULD ignore stream messages sent by the consumer: the protocol defined in this document is server-to-client only.

Servers and clients SHOULD implement a keepalive system using Ping and Pong WebSocket messages.

### Frame Format {#frame-format}

Each binary WebSocket frame contains two CBOR-encoded objects concatenated together: a header followed by a payload. Both objects MUST follow the deterministic CBOR encoding rules defined in {{cbor-encoding}}.

The header contains:

- `op` (integer, REQUIRED): the frame operation. The value `1` indicates a normal message; the value `-1` indicates an error.
- `t` (string, REQUIRED when `op = 1`): the message-type name, prefixed with `#`. For example, `#commit` for a commit message.

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

Synchronization streams include per-message sequence numbers to improve transmission reliability. Sequence numbers are positive integers that increase monotonically across the stream. Sequence semantics are flexible, and they may contain arbitrary gaps between consecutive messages.

Consumers track the last sequence number they successfully processed and can specify this as a cursor when reconnecting to receive any missed messages within the provider's backfill window. Consumers are responsible for managing and persisting cursor state themselves: producers do not maintain consumer-specific state across connections. The scope of a cursor is the (hostname, endpoint) pair: a cursor value is meaningful only when reconnecting to the same host and stream endpoint that issued it.

Sequence numbers MUST NOT be repeated by producers on the same (hostname, endpoint) pair. If a producer must reset sequence numbers for any reason, it MUST start with a number higher than any previously broadcast.

Sequence numbers are integers in the range `[1, 2^53)`. The upper bound is chosen so that cursors are exactly representable in 64-bit IEEE-754 floating point.

Stream behavior depends on the cursor value specified during connection:

- **No cursor specified**: The provider begins transmitting from the current stream position, providing only new messages generated after the connection is established.
- **Future cursor**: When the requested cursor exceeds the current stream sequence number, the provider sends an error message and closes the connection.
- **Cursor within backfill window**: The provider transmits all persisted messages with sequence numbers greater than or equal to the requested cursor, in order, then continues with the stream once caught up.
- **Cursor older than backfill window**: The provider sends an informational message indicating that the requested cursor is too old, then begins transmission at the oldest available message, sends the entire backfill window, and continues with the stream.
- **Cursor value of 0**: The provider treats this as a request for the complete available history, starting at the oldest available message, transmitting the entire backfill window, then continuing with the stream.

## Message Types {#msg-types}

The repository synchronization stream uses four message types: `#commit`, `#sync`, `#account`, and `#identity`. This section describes the schema and semantics of these messages. Some fields are common across all message types.

### Common Message Fields {#msg-common}

The following fields are common to all message payloads:

- `seq` (integer, REQUIRED): the sequence number (cursor; see {{cursors}}) of this message.
- `did` (string, REQUIRED): the account identifier of the repository this message concerns. For historical reasons the `#commit` message uses the field name `repo` rather than `did` for this purpose; the value has the same meaning.
- `time` (string, REQUIRED): an ISO 8601 datetime string indicating when the message was emitted. This timestamp is informational and is not authoritative for any verification purpose.

### `#commit` Message {#msg-commit}

A `#commit` message represents a repository update, as an atomic set of record operations. The message contains a repository diff combined with supporting metadata.

The payload contains:

- `seq` (integer, REQUIRED): see {{msg-common}}.
- `repo` (string, REQUIRED): the account identifier of the repository (see {{msg-common}}; this is the historical name of the `did` field). MUST match the `did` field in the commit object enclosed in `blocks`.
- `time` (string, REQUIRED): see {{msg-common}}.
- `rev` (string, REQUIRED): the new revision identifier of the repository after these modifications. MUST match the `rev` field in the commit object enclosed in `blocks`.
- `since` (string, REQUIRED, nullable): the revision identifier of the repository immediately prior to this commit. May be null only for the first commit of a repository.
- `commit` (hash link, REQUIRED): reference to the new commit object. MUST match the hash of the commit object enclosed in `blocks`.
- `blocks` (byte string, REQUIRED): the serialized diff (as defined in {{diffs}}) carrying all blocks required to invert and verify the operations in this message.
- `ops` (array, REQUIRED): the set of record operations encapsulated by this message. Multiple operations on the same record (path) are not allowed within a commit. Each entry is an object containing:
    - `action` (string, REQUIRED): one of `create`, `update`, or `delete`.
    - `path` (string, REQUIRED): the repository path of the record being mutated.
    - `cid` (hash link, REQUIRED, nullable): the hash link of the new record at this path, or `null` for `delete` actions.
    - `prev` (hash link, OPTIONAL): the hash link of the prior record at this path. Present for `update` and `delete` actions; absent for `create`.
- `prevData` (hash link, REQUIRED): the root hash of the repository's MST in the previous revision (the `data` field in the commit object). Used for operation-inversion validation as described in {{streaming-validation}}.
- `tooBig` (boolean, REQUIRED): retained for compatibility with earlier versions of this protocol. Producers MUST emit this field with the value `false`. Consumers MUST ignore the field's value.
- `blobs` (array, REQUIRED): retained for compatibility with earlier versions of this protocol. Producers MUST emit this field as an empty array. Consumers MUST ignore the field's contents.

A `#commit` message MUST contain no more than 200 entries in `ops`. The `blocks` field MUST NOT exceed 2 MB. Any single record block within `blocks` MUST NOT exceed 1 MB. Repository updates exceeding these limits MUST be communicated through `#sync` message instead.

A `#commit` message with an empty `ops` array (e.g., a commit issued solely to advance `rev` after a key rotation) is valid.

Note that the full message is *not* cryptographically authenticated end-to-end (from the origin account itself). Only the commit object contained within the `blocks` field is signed, with other blocks covered by proof chains. The `ops` array is *not* authenticated and must be verified via operation inversion.

### `#sync` Message {#msg-sync}

A `#sync` message declares the current state of a repository, regardless of the previous state. Sync messages are emitted when commit-message continuity cannot be maintained: large mutations exceeding the limits in {{msg-commit}}, recovery from data loss or corruption, or account migration between hosting providers.

The payload contains:

- `seq` (integer, REQUIRED): see {{msg-common}}.
- `did` (string, REQUIRED): see {{msg-common}}. MUST match the `did` field in the commit object encapsulated in `blocks`.
- `time` (string, REQUIRED): see {{msg-common}}.
- `rev` (string, REQUIRED): the current revision identifier of the repository. MUST match the `rev` field in the commit object encapsulated in `blocks`.
- `blocks` (byte string, REQUIRED): a serialized stream containing only the current commit object. Receivers reconstruct full repository state by fetching the complete repository as described in {{resync}}.

A `#sync` message provides a reset point that signals consumers to resynchronize against the current authoritative state without requiring knowledge of the intervening changes.

A `#sync` message with a `rev` which is lower or equal to the previously tracked revision for an account would constitute a "rollback" and should be ignored. The commit object signature (within the `blocks` field) should also be verified, and the message rejected if validation fails.

### `#account` Messages {#msg-account}

An `#account` message indicates a change in account hosting status for an indicated account. See {{account-status}} for details and semantics.

The payload contains:

- `seq` (integer, REQUIRED): see {{msg-common}}.
- `did` (string, REQUIRED): see {{msg-common}}.
- `time` (string, REQUIRED): see {{msg-common}}.
- `active` (boolean, REQUIRED): whether the account is currently active on the emitting service.
- `status` (string, OPTIONAL): a short status code describing the account state.

Note that account messages are *not* cryptographically authenticated end-to-end, and that they have hop-by-hop semantics.

### `#identity` Message {#msg-identity}

An `#identity` message indicates a possible change to the resolution result of an account identifier.

Note that identity messages are *not* cryptographically authenticated end-to-end. Consumers SHOULD invalidate any cached identity metadata for the named account on receipt of this message, and then re-resolve the identifier.

The payload contains:

- `seq` (integer, REQUIRED): see {{msg-common}}.
- `did` (string, REQUIRED): see {{msg-common}}.
- `time` (string, REQUIRED): see {{msg-common}}.
- `handle` (string, OPTIONAL): included for historical reasons. Non-authoritative.

`#identity` messages are best-effort: producers MAY emit them redundantly when no underlying change has occurred, and MAY fail to emit them when a change has occurred. Consumers SHOULD NOT rely on `#identity` messages as the sole signal of identity change.

## Commit Validation {#streaming-validation}

Validating a `#commit` message establishes both that the message is internally consistent (its declared operations match the diff it carries) and that it matches the prior state of the account's repository from the perspective of the consumer.

For each `#commit` message received, consumers MUST perform the following steps:

1. Verify wire-level fields: that the frame parses as deterministic CBOR, that the payload satisfies the schema in {{msg-commit}}, and that the size limits in {{msg-commit}} are not exceeded. Otherwise the message MUST be rejected.
2. Parse the repository diff from `blocks`, verifying the deterministic CBOR encoding, schema, and syntax of all fields of the commit and MST nodes. All created and updated record blocks must be present, but the content and internal structure of records are not validated at this stage. If the repository diff is invalid, the message MUST be rejected.
3. Apply operation inversion to the repository diff MST using the `ops` array, and match against the message's claimed `prevData`, as per {{operation-inversion}}. If inversion fails, the message MUST be rejected.
4. Verify the commit signature using the signing key resolved from the account identifier. If the signature is invalid, the message MUST be rejected.
5. Confirm that the message's `rev` is strictly greater than the previously observed `rev` for this account. Otherwise the message MUST be ignored.
6. Cross-check the message's `prevData` field against the consumer's previously seen `data` for this account. If they differ, the consumer has become desynchronized for this account and MUST initiate re-synchronization as defined in {{resync}}.

A signature failure at step 4 might indicate a recent key rotation rather than a malicious commit. Consumers SHOULD refresh the cached identity for the account and retry verification before rejecting the message.

## Re-synchronization {#resync}

When a consumer detects desynchronization, either through a disjunction in commit history or a `sync` message that does not match their local state, they must perform a complete re-synchronization process to restore consistency with the current repository state.

Re-synchronization requires fetching and processing the full repository structure, though the record contents themselves are optional depending on the consumer's needs. If the repository data is delivered in pre-order traversal, it can be validated incrementally as it is received.

Parsing the repository structure produces a mapping of keys (repository paths) to record versions (hashes) that represents the complete repository state. This key-to-hash mapping can be compared against existing local state to identify discrepancies and re-establish synchronization. Once validated, this mapping establishes the new repository state against which future commit messages can be applied.

Consumers SHOULD prefer requesting full repository data from their direct upstream rather than the resolved canonical host for the repository. Direct upstreams MAY coalesce and cache snapshot requests, or redirect consumers to alternative sources where appropriate. This reduces correlated load spikes on canonical hosts caused by re-synchronization message broadcast.

During the re-synchronization process, any incoming commit messages for the repository should be buffered rather than processed immediately. Once re-synchronization completes successfully, these buffered commits can be validated and applied in sequence to bring the consumer fully up to date with the current repository state.

# Media Blobs {#blob}

Larger binary media files, such as images or video, are not serialized inside repositories or synchronized over the stream mechanism. Instead, they are stored as "blobs" by the account's host, and referenced using a strong hash (CID). The blob file can be fetched out-of-band by any party, and the hash can be used to verify it's integrity.

Blob hosting and lifecycle is tied to a specific account. When an account first creates a new blob, the host places it in temporary storage and is not publicly available to the network. If the account then creates a record which includes a valid reference to the blob, then the blob becomes accessible to the network. Multiple records for the same account can reference the same blob. If all references to the blob are removed, the blob becomes inaccessible and may be deleted. A blob left lingering in temporary storage may expire and be deleted.

The hosting and access lifecycle of blobs matches that of the account's public repository data, as described in {{account-status}}. Blob data should not be served or redistributed for accounts with in-active hosting status.

The details of the account host HTTP upload and fetch endpoints, including the URL path and query parameters, are not specified by this document.

Applications SHOULD NOT rely on account hosts to distribute blobs directly to broad audiences. Applications are expected to bear the resource costs of mass distribution themselves, for example using a caching HTTP proxy or Content Distribution Network (CDN).

## Blob References {#blob-refs}

References to a blob are encoded as a special object within records. The object has a `$type` field with value `blob`, and a fixed set of fields. This pattern can be parsed and extracted from record data of any type. The reference itself does not include an account identifier: this is inferred from the repository containing the reference.

The reference object contains the following fields:

- **`$type`** (string, required): Has the fixed value `blob`
- **`ref`** (cid-link, required): Hash of the blob file. Uses the raw/arbitrary prefix as described in {{cid-link}}.
- **`mimeType`** (string, required): Content type of the blob. MUST NOT be an empty string. Use `application/octet-stream` if content type is not known.
- **`size`** (integer, required): Size of the blob in bytes. Must be non-zer and positive.

A blob object which is contains any additional fields MUST be rejected.

# Security Considerations {#security}

Repositories constitute untrusted input as account holders have complete control over repository contents and repository hosts control binary encoding. Implementations must handle potential denial of service vectors from both malicious actors and accidental conditions such as corrupted data or implementation bugs.

## CBOR Processing limits {#security-cbor}

Generic precautions must be followed when processing CBOR data, including enforcement of maximum serialized object size, maximum recursion depth for nested structures, and memory budget limits for deserialized data. While some CBOR implementations include these protections by default, implementations should verify and configure appropriate limits regardless of library defaults.

## MST Structure Attacks {#security-mst}

The efficiency of MST data structures depends on a uniform distribution of key hashes. Since account holders control record keys, they can perform key mining to generate sets of keys with specific layer assignments and sorting characteristics, resulting in inefficient tree structures. Such attacks can cause excessive storage overhead and network amplification during repository transmission.

To mitigate these attacks, implementations should:

- Limit the number of entries per MST node to a statistically reasonable maximum
- Impose limits on overall repository height
- Monitor and restrict other structural parameters that could be exploited through sophisticated key mining

## Repository Import Validation {#security-import}

When importing repositories, implementations should verify the completeness and integrity of the repository structure. Serialized repositories may contain additional unrelated blocks beyond those required for the repository structure. Care should be taken during storage to avoid resource waste on unreferenced blocks and to prevent potential storage exhaustion attacks.

## Resource Abuse {#security-resources}

A producer that routinely emits `#sync` messages could cause consumers to repeatedly fetch full repository snapshots, which is substantially more expensive than processing `#commit` messages. Similarly, a producer that rapidly updates the same key or issues many small commits can amplify message volume and bandwidth costs on downstream consumers. Consumers should apply rate limits and bandwidth budgets per repository, and may disconnect or deprioritize producers whose message patterns appear abusive.

## Repository Rewinds {#security-rewinds}

Intermediaries that relay firehose messages can present a consumer with an outdated view of a repository by replaying older commits or declining to forward newer ones. The `rev` field on each commit can help consumers detect this: a received commit whose `rev` is not greater than the most recently observed `rev` for that repository may indicate that the producer is serving a rewound view. In situations such as this, consumers can cross-check the latest observed `rev` against the canonical host for the repository.

## Server-Side Request Forgery {#security-ssrf}

Several aspects of synchronization involve following URLs or host endpoints derived from untrusted input: account-identifier resolution, retrieval of full repository data from a hosting service, and following redirects between hosts. Consumers MUST validate URLs derived from untrusted input before issuing requests, including any URLs reached via HTTP redirects. In particular, requests to internal-network addresses, loopback addresses, and link-local addresses MUST be rejected unless explicitly permitted by configuration.

## Validation Responsibility {#security-validation-responsibility}

Intermediaries that relay messages MAY apply some validation checks (for example, signature verification or size enforcement) before relaying. Consumers MUST NOT treat upstream relaying as evidence of validity: every consumer is ultimately responsible for performing the verification rules in {{streaming-validation}} on each message it processes.

## Blob Hosting {#security-blobs}

Serving arbitrary user-uploaded files (media blobs) from a web server raises several web content security issues, including cross-site scripting (XSS) of scripts or SVG content from the same web origin as authenticated web pages. Hosts SHOULD enable a strict Content Security Policy when serving blobs. Applications SHOULD serve media blobs from their own origin (proxy, CDN, etc) instead of directly linking to the canonical account host.

Processing untrusted binary media files is a common source of security exploits. Care should be taken when detecting content types or transforming media formats.


# IANA Considerations

This document has no IANA actions.

--- back

# Data Model {#data-model}

All components of the repository data structure conform to a limited data model and defined encoding rules. CBOR encoding (following the rules in {{cbor-encoding}}) is used for consistent hashing of data. A JSON encoding is also defined for record data, with lossless mapping between the CBOR and JSON encodings.

The data model includes the following types:

- **null values**: represented as 'null' in JSON, and the null special value (major 7) in CBOR
- **boolean values**: represented as 'true' / 'false' in JSON, and special values (major 7) in CBOR
- **integer values**: with signed 64-bit precision. Represented as numbers in JSON, and Integers (majors 0,1) in CBOR
- **string values**: represented as strings in JSON, and UTF-8 Strings (major 3) in CBOR
- **byte string values**: represented with a special object type in JSON (see {{json-encoding}}) and as a Byte String (major 2) in CBOR
- **content hash links**: as described in {{cid-link}}, represented as a special object type in JSON, and as a tag 42 byte string in CBOR
- **arrays**: represented as arrays in JSON, and Arrays (major 4) in CBOR
- **objects**: represented as objects in JSON, and Maps (major 5) in CBOR. Object keys must always be strings.

As a best practice to ensure compatibility with programming languages which represent all numbers in floating point by default, integer values should be limited to 53 bits of precision when possible.

## Content Identifier (CID) Hashes {#cid-link}

References to data objects by hash occur throughout the repository data structure. They also occur between records at the application layer. A consistent way of computing and encoding these hash links, named Content Identifier (CID), is described here. In addition to "CID Links" between objects, it is possible to represent CIDs as regular hash strings (without the "link" data model semantics). It is also possible to represent the hash of arbitrary binary data as a CID.

Data objects to be referenced are first encoded as CBOR. The encoded bytes are hashed using SHA-256, resulting in a 32-byte binary hash value. The hash bytes are prefixed with the 4-byte prefix value `0x01711220`, resulting in a 36-byte binary CID.

This fixed prefix value is used for historical reasons, and indicates that the referenced data is CBOR encoded. If using a CID to reference arbitrary binary data, use the fixed 4-byte value `0x01551220` instead.

When representing a CID link in CBOR, the binary CID value has an additional null byte (0x00) prepended, then the 37 bytes are stored as a byte string using the IANA-registered CBOR Tag 42.

When representing a CID value as a string, the 36-byte binary CID value is encoded using {{RFC4648}} lower-case base32, and then the ASCII character 'b' (lower-case B) is prefixed. This results in a 59 character lower-case ASCII string.

When referencing a CID link in JSON, first compute the string representation as described above. The link is then represented as a JSON object with a single key (`$link`) and the value being the string value. For example:

~~~json
{
  "$link": "bafyreidfayvfuwqa7qlnopdjiqrxzs6blmoeu4rujcjtnci5beludirz2a"
}
~~~

## CBOR Encoding {#cbor-encoding}

Repository content requires consistent binary representation across all implementations to ensure identical content hashes and verifiable integrity. All records, MST nodes, and commits must be encoded using Deterministically Encoded CBOR as specified in {{Section 4.2 of CBOR}}, with map key ordering following the original specification in {{Section 3.9 of RFC7049}} for historical compatibility.

The encoding rules that apply in this document are:

- Integers are encoded in their shortest form
- All arrays, maps, and strings are encoded with explicit lengths; CBOR's indefinite-length encoding is not used
- Floating-point values are not used; this includes NaN and infinity values
- Map keys are sorted using the legacy length-first ordering of {{Section 3.9 of RFC7049}}
- Maps MUST NOT contain duplicate keys

The encoding rules described here are compatible with similar deterministic-CBOR profiles such as {{DRISL}}.

## JSON Encoding {#json-encoding}

The JSON representation of records or other repository data objects does not need to have a deterministic binary encoding.

Byte strings are represented in JSON using a special object type. The binary data is first string encoded in base64, as described in {{RFC4648}} Section 4. This variant is not URL-safe, and `=` padding is optional. The special JSON object has a single string key `$bytes`, and the value is the base64 encoded data. For example:

~~~json
{
  "$bytes": "nFERjvLLiw9qm45JrqH9QTzyC2Lu1Xb4ne6+sBrCzI0"
}
~~~

Content hash links (CID links) are represented as special objects as described in {{cid-link}}.

# Cryptography {#crypto}

AT implementations must support both of the following elliptic curves and signature algorithms:

- NIST P-256 (also known as secp256r1 or p256) {{SEC2}}
- secp256k1 (also known as k256) {{SEC2}}

## Signature Malleability {#crypto-malleable}

ECDSA signatures exhibit malleability, allowing transformation into distinct but equally valid signatures without access to the private key or original data. While the security impact is limited, signature malleability could enable broadcast of multiple valid versions of the same repository commit with different hashes, potentially causing consumer confusion.

To prevent such scenarios, AT requires all ECDSA signatures to be canonicalized in low-S form. Specifically, the `s` component of the signature must satisfy `s ≤ n/2`, where `n` is the order of the curve's base point.

## Signature Generation {#crypto-sig}

To compute a signature over CBOR-encoded bytes in the context of AT:

1. Compute the SHA-256 hash of the encoded bytes. Do not encode the resulting hash bytes.
2. Sign the hash bytes using the current signing key associated with the account
3. Format the signature bytes as a concatenation of the 32-byte `r` and 32-byte `s` values

# Timestamp Identifier (TID) {#tid}

Timestamped Identifiers (TIDs) are compact string encodings of 64-bit integers, which can be used as logical clocks or locally-unique sorted identifiers. They are not expected to be globally unique.

They have the following structure:

- 64-bit integer with big-endian byte ordering
- Base32-sortable encoding using characters `234567abcdefghijklmnopqrstuvwxyz`
- Fixed 13-character length with no padding (integer zero encodes as `2222222222222`)

The layout of the 64-bit integer is:

- The top bit is always 0
- The next 53 bits represent microseconds since the UNIX epoch. 53 bits is chosen as the maximum safe integer precision in a 64-bit floating point number, as used by Javascript.
- The final 10 bits are an arbitrary "clock identifier."

When generating a sequence of TIDs in the same context (eg, for an individual account), care should be taken to ensure that the TID value always increments. If the system clock rolls backwards, or multiple TIDs are generated in the same microsecond, the microsecond component should be incremented past the previous generated value.

# Namespaced Identifier (NSID) Syntax {#nsid}

Collections are identified by a Namespaced Identifier (NSID): an ASCII string in reverse domain-name order followed by an additional name segment. The portion preceding the final segment is the **domain authority**; the final segment is the **name**.

NSIDs MUST conform to the following syntax:

- Overall:
    - MUST contain only ASCII characters
    - MUST separate the domain authority and the name by an ASCII period (`.`)
    - MUST contain at least three segments
    - MUST be at most 317 characters in total length
- Domain authority:
    - Composed of segments separated by ASCII periods (`.`)
    - At most 253 characters in total (including periods), and at least two segments
    - Each segment MUST contain at least 1 and at most 63 characters
    - The allowed characters are ASCII letters (`A-Z`, `a-z`), digits (`0-9`), and hyphens (`-`)
    - Segments MUST NOT start or end with a hyphen
    - The first segment (the top-level domain) MUST NOT start with a digit
    - The domain authority is not case-sensitive and SHOULD be normalized to lowercase
- Name:
    - MUST contain at least 1 and at most 63 characters
    - The allowed characters are ASCII letters and digits only (`A-Z`, `a-z`, `0-9`)
    - Hyphens are not allowed
    - MUST NOT start with a digit
    - Case-sensitive; implementations MUST NOT normalize case

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
