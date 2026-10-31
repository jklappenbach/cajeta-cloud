---
id: cloud-queue
applies-to: [dev/cajeta/cloud/queue]
title: dev.cajeta.cloud.queue — the queue port
description: Send messages and consume them with a visibility window, receipts, acknowledgement, release and redelivery against MessageQueue, with MemoryQueue as the reference driver on an injectable clock; QueueContract is the conformance suite for adapters, and QueueSelector picks the compiled-in provider.
---

# dev.cajeta.cloud.queue

The queue port of cajeta-cloud (cajeta-cloud-cloudflare spec §5.1): what SQS,
Service Bus, Pub/Sub pull and Cloudflare Queues share. Each message goes to one
consumer, who holds it by receipt for a visibility window and acknowledges or
releases it; an unacknowledged message is delivered again with its attempt
count raised. Delivery is at least once and unordered unless a capability says
more. A queue is not a stream: nothing is replayed, and there is no position.

```cajeta
import dev.cajeta.cloud.queue.MessageQueue;
import dev.cajeta.cloud.queue.MessageAttributes;
import dev.cajeta.cloud.queue.Received;

MessageAttributes a = heap MessageAttributes();
a.add("kind", "invoice");
String id #= q.send(body, a, 0L);                 // visible now; a delay in seconds otherwise

Received r #= q.receive(10, 30L);                 // up to 10, each held 30 s
for (int32 i = 0; i < r.count(); i++) {
    Message m = r.messageAt(i);                   // body(), attributes(), attempt(), sentAtSeconds()
    if (handle(m)) q.ack(m.receipt());            // gone for good
    else q.release(m.receipt(), 60L);             // back in a minute, attempt + 1
}
```

- **Receipts** are opaque and live for the window asked at `receive`. After
  it, `ack` and `release` throw `CloudException` `PRECONDITION_FAILED`: the
  message may already be with another consumer. Acknowledge inside the window
  or ask for a longer one.
- **Redelivery** happens when the window passes without an `ack`;
  `attempt()` counts deliveries from 1. Dead-letter queues are the provider's
  to configure; the port reports the attempt count (`Caps.QUEUE_DEAD_LETTER_REPORTING`
  where an adapter surfaces it).
- **Batches**: `sendBatch(ArrayList<Outgoing>)` and `ackAll(receipts)`; an
  adapter sends what you batched in one call where its API allows.
- **Capabilities** (`Caps.QUEUE_*`): `queue.delayed-send`, `queue.fifo`,
  `queue.exactly-once`, `queue.message-groups`, `queue.dead-letter-reporting`.
  Ask `capabilities().supports(...)` before relying on one; the memory driver
  declares delayed send only.

## The memory driver

`MemoryQueue(clock)` takes a `TimeSource`; a test hands it a
`ManualTimeSource` and steps time, so windows pass and redelivery happens in
no time. One lock serializes it: two consumers never hold one message.

## Conformance and selection

`QueueContract.verify(queue, waiter)` is the suite an adapter passes; the
`Waiter` is a `ManualWaiter` over the driver's clock or a `RealWaiter` that
sleeps against a real service. It checks delivery, holding, acknowledgement,
redelivery with the attempt raised, receipt expiry, release delay, batches,
attributes, and delayed send and FIFO where declared.

`QueueSelector` chooses the compiled-in `MessageQueue` by the identity port's
rule: `queue.provider` from a `QueueConfig` component, else `QUEUE_PROVIDER`;
the memory driver stands in unnamed only in a test or dev profile.
