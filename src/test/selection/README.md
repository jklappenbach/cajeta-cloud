# Selection fixtures (cajeta-cloud-identity Unit 2)

`run-tests.sh` copies `app`, plus `stub` and `config` where a case needs
them, into one source root, builds it against the library under a profile,
and checks the line the program prints. The DI graph is decided at build,
so each case is its own build.

| Case | Sources | Profile | Expected |
|---|---|---|---|
| 2.1.1, 2.3.1 | app | `test` | `selected memory` |
| 2.1.2, 2.3.1 | app | default (prod) | refused, naming `identity.provider` |
| 2.1.2 | app, config | `pick-memory` | `selected memory` |
| 2.1.3 | app, stub, config | `pick-stub` | `selected stub` |
| 2.1.3 | app, stub, config | `pick-memory` | `selected memory` |
| 2.1.3 | app, stub, config | `pick-nope` | refused, listing `memory` and `stub` |
| env | app | default (prod), `IDENTITY_PROVIDER=memory` | `selected memory` |
| env | app, stub | default (prod), `IDENTITY_PROVIDER=stub` | `selected stub` |
| code wins | app, stub, config | `pick-stub`, `IDENTITY_PROVIDER=memory` | `selected stub` |

Every case without an `IDENTITY_PROVIDER` column runs with it unset.
