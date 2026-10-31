---
id: cloud-key-value
applies-to: [dev/cajeta/cloud/kv]
title: dev.cajeta.cloud.kv — the key-value port
description: Put, get, delete and list keyed values with metadata and expiry against KeyValueStore, with conditional writes and the consistency model declared as capabilities; MemoryKeyValue is the reference driver, strong by default or eventual with a lag; KeyValueContract is the conformance suite, KeyValueSelector picks the provider.
---

# dev.cajeta.cloud.kv

The key-value port of cajeta-cloud (cajeta-cloud-cloudflare spec §6.1): bytes
under string keys with metadata and expiry, prefix listing behind a cursor,
and conditional writes where the provider offers them. Workers KV has no
conditional write and is eventually consistent, so both are declared, never
assumed.

```cajeta
import dev.cajeta.cloud.kv.KeyValueStore;
import dev.cajeta.cloud.kv.KvEntry;
import dev.cajeta.cloud.kv.PutOptions;

PutOptions o = heap PutOptions();
o.metadata("{\"role\":\"admin\"}");
o.timeToLive(3600L);                                // or o.expiresAt(epochSeconds)
kv.put("user:1", bytes, o);

KvEntry e #= kv.get("user:1");                      // null when absent or expired
if (e != null) { use(e.value(), e.metadata(), e.version()); }

KvListPage p #= kv.list("user:", "", 100L);        // keys under the prefix; p.cursor() for the next page, "" at the end
```

- **Consistency** is declared: `Caps.KV_STRONG_CONSISTENCY` (and
  `KV_READ_YOUR_WRITES`) when a read sees a write at once; otherwise
  `propagationBoundSeconds()` is the documented bound within which it will.
  Code that reads back what it wrote must allow for the bound on an eventual
  store.
- **Conditional writes** are capabilities: `putIfAbsent` needs
  `Caps.KV_CONDITIONAL_CREATE`, `compareAndSet(key, expectedVersion, ...)`
  needs `Caps.KV_COMPARE_AND_SET`. Without the capability the call throws
  `CloudException` naming it (parent spec §3.2). A loser is told the current
  version in `KvCondResult.version`.
- **Expiry** (`Caps.KV_EXPIRY`) is absolute or a time to live; an expired key
  reads as absent. Metadata (`Caps.KV_METADATA`) is a string the provider
  stores beside the value.
- **Listing** is complete under the prefix across pages; order is not
  promised.

## The memory driver

`MemoryKeyValue(clock)` is strongly consistent with every capability.
`MemoryKeyValue.eventual(clock, lagSeconds)` makes each write and delete
readable only `lagSeconds` later, so code written against Workers KV can be
tested for stale reads; conditional writes still decide on the latest state.
Versions are the driver's sequence; keys list in byte order.

## Conformance and selection

`KeyValueContract.verify(store, waiter, prefix)` is the suite an adapter
passes: round-trip with metadata, absence, delete, expiry, complete listing
behind the cursor, conditional writes that lose when they should (or refuse
cleanly when undeclared), and reads that follow the declared model, at once
or within the bound. `KeyValueSelector` chooses the compiled-in store:
`kv.provider` from a `KeyValueConfig` component, else `KV_PROVIDER`.
