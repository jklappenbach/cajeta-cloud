# cajeta-cloud — provider-neutral cloud ports

**One platform-agnostic API for the services every cloud provides.**
Production code is written against a *port*; a provider adapter (a
separate artifact: `cajeta-cloud-aws`, `-azure`, `-gcp`) binds it to
S3, Service Bus, Pub/Sub. Choosing between them is configuration, not
a rewrite.

This library is **ports, in-memory drivers, and conformance testkits —
nothing else** (spec §14.11/§14.13): the manifest declares
`"capabilities": []` and `"dependencies": {}`, and `run-tests.sh`
enforces both. No filesystem, no network, no provider SDK. The full
test suite — concurrent conditional writes included — runs with no
provider account (§14.12).

```cajeta
ObjectStore store = heap MemoryObjectStore();      // any adapter here
String etag = store.put("reports/q3", bytes, "text/plain");
CondResult r = store.putIfAbsent("leases/job-7", body, "t");
if (!r.won) { /* r.etag is the CURRENT token — re-read and retry */ }
```

Run `./run-tour.sh` for the self-checking tour of every claim below.

## The family taxonomy (§1.2)

Each service *family* gets its own port, in its own package, separable
at the package level — using the object store links none of the others
(§2.5). In particular, **queue, topic, and stream are three families,
not one**: a queue deletes on acknowledge and redelivers on timeout; a
topic fans out to independent subscriptions; a stream is an ordered,
replayable log consumed by position. Collapsing them would force each
to lie about ordering, replay, or fan-out — precisely the properties
consumers build on. This release ships **object store** (§4–§8) and
**stream** (§12); queue/topic/KV/SQL/cache ports come later, as their
own packages.

Local/on-prem backends (filesystem object store, an embedded queue)
are a **separate, uncommitted release** (§1.5) — they would drag
filesystem capability into every consumer of a pure interface, so they
cannot live here (§13.10).

## The object-store port — `dev.cajeta.cloud.object`

`ObjectStore` covers: atomic `put` (a reader sees the whole old object
or the whole new one, never a tear), explicit absence (`get` throws
`NOT_FOUND`; `stat` answers `exists=false` — never null, never empty
bytes), streaming reads/writes (`openRead`/`openWrite` — nothing
visible until `complete()`), multipart uploads (incomplete uploads are
*never* readable, §8.2), prefix listing with continuation-token
pagination and delimiter grouping, range reads that transfer **only**
the requested bytes (measured by the stats counters — `cajeta-dqe`'s
column pruning rests on this), and **conditional writes**:

- `putIfAbsent(key, bytes, ct)` — wins only if the key does not exist.
- `putIfMatch(key, bytes, ct, etag)` — wins only against the current
  version token.
- Losing is a **result** (`CondResult.won == false`), not an
  exception; the loser receives the *current* token — exactly what it
  needs to re-read and retry. This is the consensus-replacement
  primitive `cajeta-dqe`'s no-consensus commit stands on (§7).

Listing order is a documented guarantee carried **in-band**
(`ListPage.ordered`), declared only where the provider actually
promises lexicographic order (§5.4).

Position semantics worth knowing: `put`/`putIfAbsent`/`putIfMatch`
return the new version token; `stat` and `list` carry the current one.

## The stream port — `dev.cajeta.cloud.stream`

`EventStream` covers what Kafka, Kinesis, Pub/Sub, and Event Hubs
genuinely share — the shape was verified against the Kinesis API
(§12.1):

- **Position is an opaque `String` token, never an integer** (§12.2):
  Kafka offsets are `int64`, but Kinesis sequence numbers are strings
  up to 129 digits. Store the token a `Batch` gives you; hand it back
  as `StartAt.position(token)`. **The port never stores your
  position** (§12.5.2) — where a checkpoint belongs differs per
  deployment.
- **The sub-stream set is dynamic** (§12.3): `subStreamIds()` is the
  currently-open set. A closed sub-stream (Kinesis shards split and
  merge) drains to its end (`Batch.closed` turns true only once every
  record is delivered — nothing skipped), then `describe(id)` hands
  you its successors. Kafka will simply never exercise this.
- Starting positions: `oldest()`, `newest()`, `position(token)`,
  `timestamp(ms)` — mapping one-for-one to TRIM_HORIZON / LATEST /
  AT_SEQUENCE_NUMBER / AT_TIMESTAMP and Kafka's earliest / latest /
  seek / offsetsForTimes (§12.4).
- A publish-returned token is *inclusive* (a cursor there sees that
  record); a batch's `nextPosition()` is *exclusive* (resumes after
  what you consumed) — matching AT_SEQUENCE_NUMBER vs checkpoint
  semantics.
- Rate limits surface as **transient** `CloudException`s with retry
  guidance (§12.5.4). Cursor lifecycle is invisible (§12.6.1): Kinesis
  shard iterators expire after five minutes; the adapter rolls them —
  you only ever see positions.
- **External coordination differs by adapter and the port does not
  pretend otherwise** (§12.6.2): Kafka coordinates consumer groups
  broker-side; Kinesis needs KCL plus a DynamoDB lease table you
  provision. Each adapter documents its own requirement.

`MemoryStream(n)` is the in-memory driver; its `split(subId)` reshard
hook closes a sub-stream and registers two successors so §12.3 is
testable without a Kinesis account.

## Capability negotiation — `Capabilities` / `Caps`

A definitive answer **before** any operation (§3). Partial support is
granular names, so no boolean is forced to lie (§3.4): an adapter with
conditional create but not conditional overwrite declares
`COND_PUT_CREATE` and omits `COND_PUT_OVERWRITE`. Calling an
undeclared capability fails immediately, naming the capability *and*
the adapter (§3.2). Deployments pin their needs at startup:

```cajeta
store.capabilities().assertAll(needed);   // fails on LAUNCH, not mid-write
```

`cajeta-dqe` asserts conditional put this way — an adapter silently
lacking it would surface as data corruption, not a startup error.

### Who supports what

| Capability | memory | memory-stream | S3 (planned) |
|---|---|---|---|
| `object.conditional-put-create` | yes | — | yes (If-None-Match) |
| `object.conditional-put-overwrite` | yes | — | yes (If-Match) |
| `object.server-side-copy` | yes | — | yes |
| `object.list-lexicographic` | yes | — | yes (documented by AWS) |
| `object.read-after-write` | yes (strong) | — | yes (since 2020, with caveats) |
| `object.multipart` | yes | — | yes |
| `object.range-read` | yes | — | yes |
| `stream.resharding` | — | yes (`split()`) | Kinesis adapter |
| `stream.start-at-timestamp` | — | yes (logical clock) | Kinesis adapter |
| `stream.log-compaction` | — | no | Kafka only (§12.7) |
| `stream.transactional-produce` | — | no | Kafka only (§12.7) |
| `stream.consumer-groups` | — | no | Kafka only (§12.7) |

## The conformance testkit — `dev.cajeta.cloud.testkit`

The library's most valuable artifact (§10): what makes "swap the
provider" trustworthy rather than aspirational. Any adapter repo runs

```cajeta
ObjectStoreContract.verify(myAdapter, "conformance-scratch/");
```

One suite, whole port, pass or fail. Selection is
capability-conditional (declared → tested; undeclared → asserted to
fail cleanly per §3.2), documented guarantees are tested rather than
incidental behaviour (§10.5), and the concurrent-writer check runs
**real fibers** and requires exactly one winner (§10.3). The suite is
not vacuous: a deliberately always-winning adapter fails it (see
`TestkitTest::alwaysWinningAdapterIsCaught`).

## Errors, retry, observability

`CloudException` carries one of five kinds — `NOT_FOUND`,
`ACCESS_DENIED`, `PRECONDITION_FAILED`, `TRANSIENT`, `PROVIDER_ERROR`
(§11.1) — and every message names provider, operation, and key
(§11.3). `RetryPolicy` retries transient failures only, with capped
exponential backoff and seeded (testable) jitter (§11.2). `CloudStats`
exposes request counts, failure counts, retries, bytes in/out, and
latency total/max (§11.4). `Credentials` renders redacted
(`AKIA…[redacted]`) from `toString()`/`redacted()` — secrets never
reach a log (§9.5).

## Building

```sh
CAJETA=<path-to-cajeta> ./run-tests.sh   # 35 tests
CAJETA=<path-to-cajeta> ./run-tour.sh    # the self-checking tour
```

Toolchain: cajeta v0.17.4. Two v0.17.4 defects are worked around in
this codebase (both registered upstream): String-valued `HashMap`s
double-free on drop (`Capabilities` uses parallel ArrayLists), and
back-reference fields to interface-implementing classes crash the
compiler's drop synthesis (writers/cursors hold the extracted
`MemState`/`StreamState` guts instead of the store).
