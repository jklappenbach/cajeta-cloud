---
id: cloud-identity
applies-to: [dev/cajeta/cloud/identity]
title: dev.cajeta.cloud.identity — the end-user identity port
description: Register, confirm, sign in (password, emailed or texted codes, authenticator apps), verify tokens and serve the signed-in user against UserPool and its Authenticator, Account and TokenVerifier planes, with MemoryUserPool as the reference driver; IdentityContract is the conformance suite for adapters.
---

# dev.cajeta.cloud.identity

The end-user identity port of cajeta-cloud (identity addendum §2 to §8,
§15). Code depends on `UserPool` and on nothing provider-specific. Selection
of a real adapter (`cajeta-cloud-aws` for Cognito) is configuration. The pool
is the administrative plane; `pool.authenticator()` (sign-in, challenges,
refresh, sign-out, reset), `pool.account()` (the caller's own user, keyed by
an access token) and `pool.verifier()` (token checks, JWKS) are the other
three, so selecting a pool selects all four.

## Task → entry point

| You want to… | Use |
| --- | --- |
| Create an account | `pool.register(username, password)` → `#RegistrationResult` (`subject()`, `confirmed()`) |
| Create with custom attributes | `pool.registerWith(username, password, attributes)`; needs `IdentityCapability.CUSTOM_ATTRIBUTES()` |
| Complete an unconfirmed account | `pool.confirm(username, code)`, keyed by username like every public-plane call; needs `CONFIRMATION()` |
| Read a user | `pool.lookupBySubject(subject)` / `pool.lookupByName(name)` → `#User` (`subject()`, `username()`, `confirmed()`, `attributes()`, `inGroup()`, `groupCount()`, `groupAt()`) |
| Remove a user | `pool.delete(subject)` → true when it existed |
| Change one attribute whole | `pool.setAttribute(subject, name, value)` / `pool.removeAttribute(subject, name)` |
| Group membership | `pool.addToGroup(subject, g)` / `pool.removeFromGroup(subject, g)` / `pool.groups(subject)` → `#String[]`; needs `GROUPS()` |
| Create an account with no password | `pool.registerPasswordless(username, attributes)`; needs `PASSWORDLESS()` |
| Sign in with a password | `pool.authenticator().signIn(username, password)` → `#SignInResult`: `isTokens()` / `tokens()`, or `isChallenge()` / `challenge()` |
| Sign in with a code or an authenticator | `signInWith(username, Factor.EMAIL_OTP())` (`SMS_OTP()`, or `TOTP()` under `TOTP_SIGN_IN()`) → a challenge |
| Answer a challenge | `respond(challenge.session(), challenge.kind(), ChallengeAnswer.code(c))` → `#SignInResult`; loop while it is a challenge |
| Keep a session going | `refresh(refreshToken)` → `#Tokens` (`refreshToken()` is empty when the old one stays good); `signOut(refreshToken)` ends it |
| Reset a forgotten password | `forgotPassword(username)`, then `confirmForgotPassword(username, code, newPassword)` |
| Check a token | `pool.verifier().verify(token, TokenUse.ACCESS())` → `#Claims` (`subject()`, `username()`, `inGroup()`, `expiresAt()`, `attributes()`, `claim(name)`) |
| The signed-in user's own account | `pool.account().me(accessToken)`, `updateAttributes`, `changePassword`, `deleteMe`, `signOutEverywhere` |
| Add an authenticator app | `account().enrollTotp(accessToken)` → `secret()` (Base32) and `uri()` (`otpauth://`, render as a QR code), then `activateTotp(accessToken, firstCode)` |
| Ask before you rely on a feature | `pool.capabilities().supports(IdentityCapability.X())`, or `assertAll` at startup |
| Prove an adapter conforms | `IdentityContract.verify(pool, hooks, "prefix-")` from `dev.cajeta.cloud.testkit` |

## Rules that hold for every driver

- **The subject is the provider's key, not yours.** `subject()` is what the
  provider knows the user by and what a token's `sub` carries. An application
  mints its own user id and stores the subject against it.
- **Inputs are copied, returns are owned.** A `String` handed to any
  operation is never retained; a `User` is a snapshot. Bind results with
  `#=`.
- **Atomic under concurrency.** N fibers registering one username get
  exactly one success and N-1 `USER_EXISTS`.
- **Failures are `IdentityException`.** Read `kind()` against the static
  constants (`USER_NOT_FOUND`, `USER_EXISTS`, `INVALID_CODE`,
  `POLICY_VIOLATION`, `UNSUPPORTED_CAPABILITY`, …). The message names the
  provider and the operation and never a password or a code. It is not a
  `CloudException`: catch it by its own type.
- **Undeclared capability is a clean failure**, kind
  `UNSUPPORTED_CAPABILITY`, never a silent no-op.
- **Sign-in is never retried for you.** A retry is an attempt against the
  lockout: `LOCKED` carries `retryAfter()` in seconds.
- **Unknown user and wrong password look the same**, `INVALID_CREDENTIALS`
  with one message. Do not add a lookup that tells them apart.
- **Signing out does not kill an issued access token** for an offline
  `verify`; it does for every `Account` call and for its refresh token.
- **Tokens are secrets.** Log `tokens.redacted()` or `Jwt.redact(token)`
  (subject and expiry), never the token.

## The memory driver

`MemoryUserPool` declares every capability but `PASSKEYS` and
`INTROSPECTION`. A fresh user is unconfirmed; every code it would have sent
is available only through the hooks: `lastConfirmationCode(subject)`,
`lastSignInCode(subject)`, `lastPasswordResetCode(subject)`. Passwords are
stored as `pbkdf2-sha256$600000$<salt>$<hash>`;
`pool.hooks().passwordRecord(subject)` exposes the record so a test can
assert the tag. Tokens are HS256 under a per-pool key the JWKS publishes,
so it is for tests and development only. Five wrong passwords or codes lock
the account with backoff from thirty seconds to fifteen minutes. TOTP seeds
live in process memory. Build it with `MemoryUserPool.withClock(clock)` and a
`ManualTimeSource` to move time in a test; `requireTotp(true)` makes every
password sign-in without an authenticator answer `MFA_SETUP`. Attribute and
group names are 1 to 64 bytes without whitespace, otherwise
`POLICY_VIOLATION`; passwords are at least 8 bytes.
`MemoryUserPool.withCapabilities(false, false, false)` builds a driver
declaring nothing, for exercising the undeclared branches.

## Choosing the pool

Inject `IdentitySelector` and call `pool()`; never `heap MemoryUserPool()` in
application code. Every implementation is a named `@Component`
(`MemoryUserPool` is `memory`; an adapter must declare its own name). `identity.provider`
is the `IDENTITY_PROVIDER` environment variable unless code overrides it: an
`IdentityConfig` component whose `provider()` returns a name wins, and null
defers to the environment. Declare at most one; a `@Profile` on it picks per
environment. Unset selects the only implementation, and the memory
driver unnamed is accepted only in the `test`, `dev` or `development`
profile. A bad choice is an `IdentitySelectionException` when the selector
is built, naming `identity.provider` and the implementations present.

## Do not

- Do not catch `CloudException` expecting identity failures.
- Do not put a password, a code, a TOTP secret or a token in a message or a
  log line.
- Do not offer `TOTP` as the only factor on Cognito: it has no
  `TOTP_SIGN_IN`. Assert the capabilities your configuration needs at
  startup.
- Do not read any environment variable here but `IDENTITY_PROVIDER`; the
  `env` capability is declared for that one read.
- Do not build a filesystem-backed pool here: it belongs in
  `cajeta-cloud-local`; this library declares only `env`.
