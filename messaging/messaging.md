# Pub/Sub Messaging

Schemas for everything passed over a **message bus / queue / stream**. There are two message families,
both framing the C1 engine protocol as bus messages rather than redefining it.

Machine-readable schema: [`pubsub-messages.schema.json`](./pubsub-messages.schema.json) (JSON Schema,
draft 2020-12). It references the C1 protocol schema
([`../engine/command-event-protocol.schema.json`](../engine/command-event-protocol.schema.json)) for
the command and event payloads.

## Message families

### (a) Command messages — queueable

A **command message** wraps a C1 command (`input`, `action`, `query_state`, `spawn_actor`,
`despawn_actor`, `move_actor`, `rotate_actor`, `snapshot`, `restore`, `graph_hash`, `quit`) and is
enqueued for a session's engine. Because a command is pure data, it is durably enqueueable and
replayable. The session's FIFO consumer applies commands in one total order.

### (b) Skein event messages — published

A **skein event message** wraps a C1 player-event envelope
(`{type:event, tick, event_type, event_name, data}`) and is published for subscribers. The ordered
sequence of event messages for a session **is the skein** — the durable, replayable record of play.

> Host-control reply records (e.g. `actor_spawned`, `state`, `graph_hash`) are point-to-point answers
> to a command, not part of the published event stream.

## Envelope

Every bus message carries:

| Field | Meaning |
|---|---|
| `message_type` | `command` or `event`. |
| `session` | The **ordering/partition key** — the engine session the message belongs to. |
| `seq` | Monotonic per-session sequence number establishing total order. Required on events; recommended on commands. |
| `topic` | Logical stream id (see conventions). |
| `produced_at` | Producer timestamp (informational; never used for ordering). |
| `trace_id` | Optional correlation id. |

Command messages additionally carry `idempotency_key` (at-most-once application of a redelivered
command) and the `command` payload. Event messages carry `tick` and the `event` envelope.

## Topic / stream conventions

- `tapestry.commands.<session>` — the inbound command stream for a session. Producers publish here.
- `tapestry.skein.<session>` — the outbound event stream (the skein) for a session. Consumers subscribe
  here.

## Ordering & determinism

**Per session, commands are consumed in a single total order.** The same seed plus the same ordered
command sequence reproduces a **byte-identical** event stream. Consequences for bus consumers and
producers:

- The command stream is the durable **replay input**; the event stream is its deterministic
  **projection**.
- Producers MUST order commands within a session by a monotonic `seq`; consumers MUST NOT reorder them.
- Cross-session ordering is neither defined nor required — partition by `session` and order only within
  a partition.
- A subscriber that has the seed and the ordered command stream can reconstruct the event stream
  exactly; this is what makes the bus safe for durable, exactly-ordered delivery.
