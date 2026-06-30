# Canonical Object Model — the BOM

The definitive, consumer-facing **object model** of the Tapestry game world — its bill of materials.
This is the published definition of every object type the platform's other contracts build on. C2
(portable state) and C4 (authoring) reference these shapes for entities, properties, relations, and
overlays.

Machine-readable schema: [`object-model.schema.json`](./object-model.schema.json) (JSON Schema, draft
2020-12).

## The world is a graph

The world is a **labeled property graph**: **entities** (nodes) connected by **typed relations**
(edges). Both nodes and edges carry **schemaless property bags**. There is no privileged containment
hierarchy — containment is one relation type among many.

The object types below are the complete bill of materials.

---

## 1. Entity (node)

The fundamental object. An entity carries:

- **Identity** — a runtime **id** (a monotonic, never-reused integer; O(1) access; stable within a
  running session but **not** across sessions/instances) and/or an author-facing **alias** (a stable
  string; the durable handle). Authors address by alias; the engine maps alias → id at load.
- **Property bag** — the entity's own state (§2).
- **Kind / prototype** — an optional prototype the entity delegates to (§3).
- **Lifecycle** — `active` | `inactive` | `dead` | `destroyed`. Entities are **never hard-deleted**;
  they transition lifecycle state and stay referenceable forever. Perception gates on this state (a
  destroyed item is still *known-about* but not *perceived*; a corpse is still *seen*).

---

## 2. Property bag & value types

A property bag is an unbounded, schemaless map of `key → value`. Both entities and relation payloads
carry one. The **variant value types** are:

| Value type | Encoding | Example |
|---|---|---|
| **scalar** | string, boolean, integer, or float | `13`, `true`, `"undead"` |
| **identifier** | a symbolic name (string-encoded) | `"north"` |
| **prose** | literate text (string-encoded) | `"A low basilica of bone."` |
| **code** | an authored script body (string-encoded) | `"engine.set_prop(ctx.actor,'rattled',true)"` |
| **link** | a typed reference to another entity, by id or alias | `{ "ref": "altar" }` |
| **list** | an ordered list of values | `[101, 104, 117]` |
| **dice-expression** | a rollable expression, as its canonical string | `"2d6+3"` |
| **roll-result** | the structured outcome of a roll | `{ "expr": "2d6+3", "total": 11, "faces": [4,3], "modifier": 4 }` |

> JSON cannot distinguish identifier / prose / code from a plain string at the type level, so those
> three travel as strings and are disambiguated by the property's role in content. Dice-expressions are
> strings recognized by form; roll-results are the structured record.

A roll-result records per-die faces, kept/dropped dice, modifier, total, and flags (e.g. `nat20`,
`exploded`, `advantage`) — so a roll is verifiable and presentable, not just a bare number.

---

## 3. Kind / prototype inheritance

An entity may delegate to a **prototype** via `kind` (sugar) or `parent` (explicit). The prototype is
itself an entity; the delegating entity inherits its properties and behaviour. Resolution order is
**explicit parent > kind > default > root**.

The **effective** view of an entity folds its own property bag over its prototype chain (own properties
layer on inherited ones) and then folds active overlays (§5). Perception and rules always read the
effective view.

---

## 4. Relations (typed edges)

A **relation type** is a first-class, declarable edge schema:

| Field | Meaning |
|---|---|
| `name` | The relation type name. |
| `directionality` | `directed` or `undirected`. |
| `cardinality` | `(source) × (target)`, each `one` or `various`. The `one` side is a single slot; the `various` side is an ordered set. |
| `symmetry` | `symmetric` auto-maintains the reverse edge; `asymmetric` does not. |
| `role` | The semantic role (below) that determines how spatial/perception machinery treats it. |
| `exclusion_group` | Optional mutual-exclusion set on a node. |
| `invariants` | Structural invariants enforced at write (e.g. `acyclic`). |
| `payload_schema` | Optional shape for the edge payload. |

**Roles**: `none` (a plain content relation the spatial walk ignores) · `containment` (in-ness) ·
`support` (on-ness) · `incorporation` (part-of; structural, **outside** the location exclusion group) ·
`map` (room adjacency) · `portal` (a passage/door between containers) · `sensory` (a perception
channel: `see` | `touch` | `hear` | `smell` | `know`).

A **relation** (edge) is an instance of a relation type between a `from` entity and a `to` entity, with
an optional **payload** property bag.

**The location group.** `{in, on, carried, worn}` is a single exclusion group and is various-to-one
across the group: an entity is in exactly one of them. Moving an entity is setting its location-group
edge (which evicts the prior one). The location-group edge payload carries the entity's integer **cell
position** `{x, y, z}` within its container — optional, and absent for entities that don't care about
tactical position. **Cells are integers only**, never floats.

**Integrity, enforced at edge-write**: cardinality (a second edge into a `one` slot is rejected),
exclusion-group exclusivity, and acyclicity for containment/support/incorporation. Illegal edges are
rejected, never silently corrupting the graph.

---

## 5. Overlays (the `affects` edge)

An **overlay** is an effect expressed as an `affects(source → target)` edge carrying
`{ property, op, magnitude, precedence, duration? }`:

- `op` ∈ `set` | `add` | `mul` | `clamp` | `tag` — a closed, declarative op-algebra the engine folds
  without knowing content meaning.
- `precedence` orders overlays folded onto the same property.
- `duration` (optional) is remaining ticks; the engine decrements it and emits an expiry event when it
  elapses.

Conditions, materials (`made-of`), equipment bonuses, auras, and spell effects **all** express as
overlays. The **effective property** of an entity is its base bag folded with all inbound `affects`
edges by precedence.

> The line: if a modification needs arbitrary computation, it is **not** an overlay — it is a trigger
> that writes a property. Overlays are declarative deltas only.

---

## 6. How the other contracts reference this model

- **C2 (portable state)** — a portable character is an **entity** (its property bag = the durable
  stats/props), carrying **items** (entities) and **conditions** (overlay snapshots). The live state
  export is the effective entity view.
- **C4 (authoring)** — authoring is CRUD over this exact model: create **entities**, set **property
  bags**, declare **relation types**, create **relations**, attach behaviour, and set overlays
  (e.g. light) — then lower to a runtime artifact.
- **C1 (engine protocol)** — at runtime, state changes are deltas to this model (property writes,
  relation moves) carried as events.
