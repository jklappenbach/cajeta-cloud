---
id: cloud-identity
applies-to: [dev/cajeta/cloud/identity]
title: dev.cajeta.cloud.identity — the end-user identity port, minimal cut
description: Register, confirm, look up and delete users, and manage their attributes and groups, against UserPool with MemoryUserPool as the reference driver; IdentityContract is the conformance suite for adapters.
---

# dev.cajeta.cloud.identity

The end-user identity port of cajeta-cloud (identity addendum §2, §3, §7,
§8). Code depends on `UserPool` and on nothing provider-specific. Selection
of a real adapter (`cajeta-cloud-aws` for Cognito) is configuration. This is
the **minimal cut**: registration, confirmation, lookup, attributes and
groups. Authentication, challenges and tokens are the full cut and are not
here yet.

## Task → entry point

| You want to… | Use |
| --- | --- |
| Create an account | `pool.register(username, password)` → `#RegistrationResult` (`userId()`, `confirmed()`) |
| Create with custom attributes | `pool.registerWith(username, password, attributes)`; needs `IdentityCapability.CUSTOM_ATTRIBUTES()` |
| Complete an unconfirmed account | `pool.confirm(userId, code)`; needs `CONFIRMATION()` |
| Read a user | `pool.lookupById(id)` / `pool.lookupByName(name)` → `#User` (`id()`, `username()`, `confirmed()`, `attributes()`, `inGroup()`, `groupCount()`, `groupAt()`) |
| Remove a user | `pool.delete(id)` → true when it existed |
| Change one attribute whole | `pool.setAttribute(id, name, value)` / `pool.removeAttribute(id, name)` |
| Group membership | `pool.addToGroup(id, g)` / `pool.removeFromGroup(id, g)` / `pool.groups(id)` → `#String[]`; needs `GROUPS()` |
| Ask before you rely on a feature | `pool.capabilities().supports(IdentityCapability.X())`, or `assertAll` at startup |
| Prove an adapter conforms | `IdentityContract.verify(pool, hooks, "prefix-")` from `dev.cajeta.cloud.testkit` |

## Rules that hold for every driver

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

## The memory driver

`MemoryUserPool` declares `CONFIRMATION`, `CUSTOM_ATTRIBUTES` and `GROUPS`.
A fresh user is unconfirmed; the code it would have sent is available only
through `pool.hooks().lastConfirmationCode(id)`. Passwords are stored as
`sha256-salted$<salt>$<hash>`; `pool.hooks().passwordRecord(id)` exposes the
record so a test can assert the tag. Attribute and group names are 1 to 64
bytes without whitespace, otherwise `POLICY_VIOLATION`; passwords are at
least 8 bytes. `MemoryUserPool.withCapabilities(false, false, false)` builds
a driver declaring nothing, for exercising the undeclared branches.

## Do not

- Do not catch `CloudException` expecting identity failures.
- Do not compare a password or a code yourself in a message or a log line.
- Do not build a filesystem-backed pool here: it belongs in
  `cajeta-cloud-local`, this library stays at `capabilities: []`.
