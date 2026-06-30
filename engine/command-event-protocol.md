# Engine Command/Event Protocol — Contract C1

The wire contract for the Tapestry engine. The engine consumes **commands** (one JSON object per
line on its input stream) and produces **records** (one JSON object per line on its output stream).
This is the contract every host, gateway, and client builds against.

Machine-readable schema: [`command-event-protocol.schema.json`](./command-event-protocol.schema.json)
(JSON Schema, draft 2020-12).

The protocol carries **transport and dispatch semantics only — no ruleset (content) semantics.**
Content-specific payloads (turn requests, inventory, combat, etc.) ride the generic `SIGNAL` event
channel and are opaque to this contract.

---

## 1. Two record families on the output stream

Consumers MUST distinguish output records by the top-level `type`:

1. **Player events** — the replayable event stream, wrapped in the event envelope
   `{"type":"event", ...}` (§3). This is the durable, ordered record of play.
2. **Host-control replies** — bare `{"type":"<name>"}` records that answer a host-control command
   (§4). They are **not** part of the replayable player event stream.

Fatal errors are reported on a separate error stream; recoverable errors are `ERROR` events on the
output stream (§3).

---

## 2. Commands (input stream)

One JSON object per line, dispatched on the `type` key. A malformed line, or a missing/unknown
`type`, produces a recoverable `ERROR` event and the session continues.

| `type` | Purpose |
|---|---|
| `input` | Text-verb path: a raw command sentence the engine parses, resolving nouns to ids and running the verb workflow. |
| `action` | Structured, parser-bypassing command: nouns are already resolved entity ids. Runs the same verb workflow as `input`. |
| `query_state` | Read the live graph as JSON on demand. Pure read — never mutates, never advances the tick. |
| `spawn_actor` | Embody a character as an actor; optionally seeds from a portable blob (Contract C2). |
| `despawn_actor` | Remove an actor; returns its serialized portable blob (Contract C2). |
| `move_actor` | Host-control intra-container move to a cell; occupancy-validated. |
| `rotate_actor` | Set an actor's facing in degrees (0–359). |
| `snapshot` | Request a whole-graph checkpoint (opaque, deterministic binary blob). |
| `restore` | Rebuild the session graph from a checkpoint blob. |
| `graph_hash` | Request a deterministic hash of current graph state (determinism verification). |
| `quit` | Clean shutdown. End-of-input also exits. |

```jsonc
// Text-verb path
{"type":"input","raw":"attack throne","actor":42}

// Structured path — ids already resolved by the client
{"type":"action","actor":42,"verb":"attack","nouns":[51, 7],"params":{}}
```

**`input` vs `action`.** `input` parses a raw sentence through the verb grammar to resolve nouns to
entity ids, then runs the verb workflow. `action` enters at the already-resolved-id seam: a client
that already holds the ids it is acting on (e.g. point-and-click) emits `action` directly and skips
parsing. Both run the **same** verb workflow, so all rules, triggers, and dice fire identically.

**Determinism.** Given the same seed and the same ordered sequence of commands, the engine produces
a byte-identical event stream. Commands are data and can be durably enqueued and replayed.

---

## 3. Events (output stream)

Every player event shares one envelope:

```jsonc
{"type":"event","tick":<N>,"event_type":<int>,"event_name":"<NAME>","data":{ ... }}
```

| `event_type` | `event_name` | `data` payload |
|---|---|---|
| 0 | `INPUT` | `{raw, cleaned|null, actor}` |
| 1 | `OUTPUT` | `{text, verb|null, actor[, audience[], exits[]]}` — `audience` is the engine-computed observer set; visibility is filtered server-side. |
| 2 | `STATE_CHANGE` | `{entity, key, before, after[, x, y, z][, first_seen]}` — incremental graph mutation. |
| 3 | `DICE_ROLL` | `{expr, result, roller}` — deterministic, seeded. |
| 4 | `FIRED` | `{name, target, payload}` |
| 5 | `TICK_ADVANCED` | `{}` |
| 6 | `ERROR` | `{level, message, traceback|null}` — recoverable. |
| 7 | `SIGNAL` | `event_name` = caller-supplied content channel (e.g. `TURN_REQUEST`, `INVENTORY`); `data` = verbatim content payload. The ruleset-agnostic content channel. |

```jsonc
{"type":"event","tick":12,"event_type":1,"event_name":"OUTPUT",
 "data":{"text":"The skeleton clatters and strikes back.","verb":"attack","actor":42,"audience":[42,7]}}
{"type":"event","tick":12,"event_type":2,"event_name":"STATE_CHANGE",
 "data":{"entity":7,"key":"hp","before":9,"after":4}}
```

---

## 4. Host-control reply records (output stream)

Bare records (not the event envelope), not part of the replayable stream:

| `type` | Shape |
|---|---|
| `actor_spawned` | `{"type":"actor_spawned","actor":<id>,"room":<id>}` |
| `actor_despawned` | `{"type":"actor_despawned","actor":<id>,"state":<C2 blob \| {}>}` |
| `move_refused` | `{"type":"move_refused","actor":<id>,"x":,"y":,"z":,"reason":"blocked"}` |
| `rotate_refused` | `{"type":"rotate_refused","actor":<id>,"facing":,"reason":"out_of_range"}` |
| `state` | `{"type":"state","tick":<N>,"scope":...,"data"\|"node":...}` — reply to `query_state`; response shape owned by Contract C2. |
| `snapshot` | `{"type":"snapshot","blob":<opaque>}` — opaque, deterministic checkpoint blob for the host to store. |
| `graph_hash` | `{"type":"graph_hash","hash":<str>}` |

> **Note on `actor_despawned`.** An empty item set may serialize as `{}` rather than `[]`; consumers
> should coerce to a list.

---

## 5. Value encoding

Property values follow the engine variant system: scalars (string, boolean, integer, float), a
dice-expression as its canonical string (e.g. `"2d6+4"`), and lists as JSON arrays. The encoder is
deterministic: integers without a decimal point, floats at full precision, object keys sorted
lexicographically — so the byte stream is reproducible for replay.

---

## 6. Entity ids

Entity (node) ids are integers minted by the engine. They are monotonic and never reused **within a
single running session**, but are **not** stable across sessions or instances. Durable, cross-session
identity is carried by the portable character blob (Contract C2) and platform item ids
(control-plane), never by raw engine node ids.

---

## 7. Related contracts

- **C2** — portable character blob carried by `spawn_actor`/`despawn_actor`, and the `query_state`
  response shape. See [`../state/portable-state.schema.json`](../state/portable-state.schema.json).
- **C3** — the instance lifecycle/routing that hosts engines speaking this protocol. See
  [`../instance/instance-contract.openapi.yaml`](../instance/instance-contract.openapi.yaml).
- **C4** — authored content lowers to a runtime artifact the engine loads and then drives over this
  protocol. See [`../authoring/authoring-api.openapi.yaml`](../authoring/authoring-api.openapi.yaml).
