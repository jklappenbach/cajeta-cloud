# cajeta-cloud — provider-neutral cloud ports

**One platform-agnostic API for the services every cloud provides.**
Production code is written against a *port*; a provider adapter (a
separate artifact: `cajeta-cloud-aws`, `-azure`, `-gcp`) binds it to
S3, Service Bus, Pub/Sub. Choosing between them is configuration, not
a rewrite.

This library is **ports, in-memory drivers, and conformance testkits —
nothing else** (spec §14.11/§14.13): the manifest declares
`"capabilities": ["env"]` and `"dependencies": {}`, and `run-tests.sh`
enforces both. `env` is for one read, `IDENTITY_PROVIDER` (see
[Choosing the pool](#choosing-the-pool)). No filesystem, no network, no
provider SDK. The full
test suite — concurrent conditional writes included — runs with no
provider account (§14.12).

```cajeta
ObjectStore store = heap MemoryObjectStore();      // any adapter here
String etag #= store.put("reports/q3", bytes, "text/plain");
CondResult r #= store.putIfAbsent("leases/job-7", body, "t");
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
consumers build on. This release ships **object store** (§4–§8), **stream** (§12) and the
minimal cut of **identity** (the identity addendum, §3 and §7);
queue/topic/KV/SQL/cache ports come later, as their own packages.

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

## The identity port — `dev.cajeta.cloud.identity`

End-user identity is a family of its own (identity addendum §1.2): an
adapter's own credentials stay cross-cutting, the users an application
registers are a port. `UserPool` covers the minimal cut: `register` and
`registerWith` (user-exists on a taken name, policy-violation on a weak
password, and the message never carries the password), `confirm` with a
delivered code, keyed by username, `lookupBySubject` / `lookupByName`,
`delete`, whole-attribute
`setAttribute` / `removeAttribute`, and `addToGroup` / `removeFromGroup`
/ `groups`. Every input is copied and every return is an owned snapshot,
so a pooled request buffer is never retained (§2.3, §2.4). The id the
port hands back is the provider's **subject**, the value that later appears
in a token's `sub` claim. It is not the application's primary key: an
application mints its own user id and stores the subject against it, so a
provider change re-links subjects instead of rekeying the system. Every
operation is atomic under concurrent use: two registrations of one
username yield exactly one success (§2.5).

```cajeta
UserPool pool = heap MemoryUserPool();               // any adapter here
RegistrationResult r #= pool.register("ada", "correct horse battery");
pool.confirm("ada", codeFromEmail);
User ada #= pool.lookupByName("ada");
```

`MemoryUserPool` is the reference implementation and the default when
no adapter is compiled in. It stores a password as PBKDF2-SHA256 under a
format tag (`pbkdf2-sha256$600000$salt$hash`; a Unit 1
`sha256-salted$salt$hash` record still signs in and is rehashed when it
does), never the password, and it never fakes a delivery channel: every
code a test needs comes from `MemoryUserPool.hooks()` (§3.7, §6.5).
Failures are `IdentityException` with one of the §11.1 kinds.

### Signing in

The pool is the administrative plane. The other three are reached through
it, so choosing a pool chooses all four (§15.1):

- `pool.authenticator()` is the public plane: `signIn(username, password)`,
  `signInWith(username, Factor.EMAIL_OTP())` (or `SMS_OTP`, `TOTP`),
  `respond(session, kind, answer)`, `refresh`, `signOut`,
  `forgotPassword` / `confirmForgotPassword`, and `enrollTotpForChallenge`
  inside `MFA_SETUP`.
- `pool.account()` is the self-service plane, every call keyed by the
  caller's access token: `me`, `updateAttributes`, `changePassword`,
  `deleteMe`, `enrollTotp` / `activateTotp`, `signOutEverywhere`.
- `pool.verifier()` checks a token's signature, issuer, audience, expiry and
  use, and names the check that failed. `jwks()` publishes the keys.

```cajeta
Authenticator auth = pool.authenticator();
SignInResult r #= auth.signIn("ada", "correct horse battery");
while (r.isChallenge()) {                              // TOTP, EMAIL_OTP, ...
    String session #= r.challenge().session();
    r #= auth.respond(session, r.challenge().kind(),
        ChallengeAnswer.code(codeFromTheUser));
}
Claims c #= pool.verifier().verify(r.tokens().accessToken(), TokenUse.ACCESS());
```

Sign-in is password-free when the application wants it (§15.18):
`registerPasswordless(username, attributes)` makes an account with no
password, which signs in with an emailed or texted code, or with an
authenticator app alone under `TOTP_SIGN_IN`. TOTP is RFC 6238 at SHA-1, six
digits and thirty seconds, so Microsoft and Google Authenticator read the
`otpauth://` URI that enrollment returns. A wrong password and an unknown
username answer the same `INVALID_CREDENTIALS` and the same message.

The memory driver signs ES256 under a per-pool P-256 pair and publishes
its public half in the JWKS, so tokens verify the same way against it and
against a provider, and a test can check them without the driver
(§5.5, §10.4). Verifiers check RS256 and ES256 over the JWKS through
[dev.cajeta.crypto](https://github.com/jklappenbach/cajeta-crypto) and
refuse HS256, since a verifier holding public keys cannot check a shared
secret. Tokens last an hour, refresh tokens thirty days, codes three
minutes and reset codes an hour. Five wrong passwords or codes lock an
account for thirty seconds, doubling to fifteen minutes, and the error
carries `retryAfter()`. Signing out ends the session: its refresh token and
every Account call stop working, while an offline `verify` still accepts its
access token until it expires, as Cognito's does (§4.6). Its clock is a
`TimeSource`: `MemoryUserPool.withClock(clock)` over a `ManualTimeSource` lets a
test move time.

### Choosing the pool

Which implementations exist is decided at build; which one a program uses
is ordinary selection over that set (identity addendum §9). Each
implementation is a named `@Component`: `MemoryUserPool` is `memory`, and an
adapter declares its own name, such as `@Component(name = "cognito")`.
Consumers inject `IdentitySelector` and take `pool()`:

```cajeta
IdentitySelector s = IdentitySelector.__cajeta_inject();
UserPool users = s.pool();
```

`identity.provider` comes from the `IDENTITY_PROVIDER` environment
variable, and code overrides it, in the order the AWS SDKs use: an
`IdentityConfig` component the application declares wins when its
`provider()` returns a name, and null defers to the environment. A
`@Profile` on it names a provider per environment's build:

```cajeta
@Component
@Profile("staging")
public final class StagingIdentity implements IdentityConfig {
    public String provider() { return "cognito"; }
}
```

| Compiled in | `identity.provider` | Test or dev profile | Other profiles |
|---|---|---|---|
| memory only | unset | memory | refused: name it |
| anything | a present name | that one | that one |
| anything | an absent name | refused, listing what is present | same |
| several | unset | refused, listing them | same |

A refusal is an `IdentitySelectionException`, thrown when the selector is
built, so injecting it at startup refuses startup. The development
profiles are `test`, `dev` and `development`.

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
| `identity.confirmation` | yes (code via hooks) | — | Cognito adapter |
| `identity.custom-attributes` | yes | — | Cognito adapter |
| `identity.groups` | yes | — | Cognito adapter |
| `identity.totp` / `sms-otp` / `email-otp` | yes (codes via hooks) | — | Cognito adapter |
| `identity.refresh` / `revocation` / `admin` / `password-reset` | yes | — | Cognito adapter |
| `identity.jwks` | yes (ES256, public key published) | — | Cognito adapter (RS256, verifiable since 0.4.0) |
| `identity.passwordless` | yes | — | Cognito adapter (Essentials tier) |
| `identity.totp-sign-in` | yes | — | no: Cognito takes TOTP as a second factor only |
| `identity.passkeys` / `introspection` | no | — | Cognito adapter |

## The conformance testkit — `dev.cajeta.cloud.testkit`

The library's most valuable artifact (§10): what makes "swap the
provider" trustworthy rather than aspirational. Any adapter repo runs

```cajeta
ObjectStoreContract.verify(myAdapter, "conformance-scratch/");
```

`IdentityContract.verify(myPool, myHooks, "conformance-")` is the same
thing for the identity port, the full port included, and a pool whose
register skips the uniqueness check fails it
(`IdentityTest::skippedUniquenessIsCaught`). `verifyWithClock` takes the
pool's `ManualTimeSource` too, for the cases that move time (expiry,
lockout lifting, TOTP steps); without one they are skipped, as are the
cases that need a delivered code when there are no hooks. Each section of
`AuthenticationContract` fails a driver that always succeeds at what it
covers (`LiarTest`).

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
CAJETA=<path-to-cajeta> ./run-tests.sh   # 43 tests
CAJETA=<path-to-cajeta> ./run-tour.sh    # the self-checking tour
```

Since 0.4.0 the library depends on `dev.cajeta.crypto`; both scripts take
it from `~/.olla` at the `cajeta.json` pin, and `run-tests.sh` fetches it from
Olla when it is not there (`CRYPTO_CJA` overrides).

Toolchain: cajeta v0.29.0. v0.17.4, which this workflow pinned until
now, compiles the library and then the test binary segfaults walking
the drop chain before a single test reports, so that pin was green in
name only. v0.28.0 and v0.29.0 both run the suite at 35 passed, 0
failed, 1 skipped.

Two v0.17.4 defects are worked around in this codebase, and their
status now differs:

- String-valued `HashMap`s double-freed on drop, so `Capabilities`
  stores parallel `ArrayList`s. **Fixed.** A probe that builds a
  `HashMap<String, String>`, fills it, and lets it drop crashes on
  v0.17.4 and prints `survived the drop` on v0.29.0. The parallel
  lists stay because they work, cost nothing at a dozen capabilities,
  and are private to the class.
- Back-reference fields to interface-implementing classes crashed the
  compiler's drop synthesis, so writers and cursors hold the extracted
  `MemState`/`StreamState` guts instead of the store. **Unverified.** A
  minimal reproduction of that shape does not crash v0.17.4 either, so
  it is not a valid instrument and a clean run on v0.29.0 would prove
  nothing. The workaround stays until the real shape is recovered.
