# Tapestry API Contracts

The **public, source-free** API contracts for the Tapestry platform — a text-adventure /
dungeon-crawler engine and the services around it. This repository is the **single source of truth**
that internal Tapestry programs and clients, and external integrations (such as the Draper MCP
server), build against.

It contains **specifications only — no source, and no internal implementation detail.** Each contract
describes *what* an API is (its commands, events, endpoints, and request/response shapes), never *how*
it is implemented. The specs are machine-readable so you can generate clients and validators directly
from them: JSON Schema (draft 2020-12) for message/data contracts, and OpenAPI 3.1 (YAML) for
HTTP/WebSocket services.

## Contract index

| Contract | What it is | Spec |
|---|---|---|
| **Object model — the BOM** | The canonical, definitive object model of the game world: entities, property-bag value types, kind/prototype inheritance, typed relations (role/cardinality/exclusion/payload), and overlays. The schema C2 and C4 reference for entity shapes. | [`model/object-model.schema.json`](model/object-model.schema.json) · [`model/object-model.md`](model/object-model.md) |
| **C1 — Engine command/event protocol** | The line-delimited JSON wire the engine reads (commands) and writes (events + host-control replies). | [`engine/command-event-protocol.schema.json`](engine/command-event-protocol.schema.json) · [`engine/command-event-protocol.md`](engine/command-event-protocol.md) |
| **Messaging — pub/sub** | Bus/queue message schemas: queueable C1 command messages and published skein event messages, with topic conventions and the ordering/determinism rules. References C1. | [`messaging/pubsub-messages.schema.json`](messaging/pubsub-messages.schema.json) · [`messaging/messaging.md`](messaging/messaging.md) |
| **C2 — Portable state** | The durable portable character blob `{inventory, stats, props}`, the engine-side spawn/despawn blob, and the on-demand JSON state export. | [`state/portable-state.schema.json`](state/portable-state.schema.json) |
| **C3 — Instance contract** | The HTTP lifecycle/routing seam: the endpoints a managed instance exposes, and the gateway endpoints clients call to be routed/connected. | [`instance/instance-contract.openapi.yaml`](instance/instance-contract.openapi.yaml) |
| **C4 — Authoring API** | Transactional CRUD over an authoring graph (entities, relations, property bags, versioned triggers, geometry) that lowers to a runtime artifact. The primary contract for authoring clients and the MCP server. | [`authoring/authoring-api.openapi.yaml`](authoring/authoring-api.openapi.yaml) |
| **Control plane — Character Store** | CRUD on durable characters + load/save of the portable state blob + per-owner roster. | [`control-plane/character-store.openapi.yaml`](control-plane/character-store.openapi.yaml) |
| **Control plane — Party / Membership** | Parties and membership: party CRUD, the caller's own parties, leader-driven member management, leadership transfer, and the service-scoped membership check that authorizes dungeon routing. | [`control-plane/party.openapi.yaml`](control-plane/party.openapi.yaml) |
| **Control plane — Catalog** *(planned)* | Durable prototype catalog that scripted items and authored kinds resolve against. | [`control-plane/catalog.openapi.yaml`](control-plane/catalog.openapi.yaml) |
| **Control plane — Leaderboards** *(planned)* | Per-floor best-run leaderboards with commit-reveal seed integrity. | [`control-plane/leaderboards.openapi.yaml`](control-plane/leaderboards.openapi.yaml) |
| **Control plane — Economy** *(planned)* | Transactional item ledger enforcing the item-singleton invariant. | [`control-plane/economy.openapi.yaml`](control-plane/economy.openapi.yaml) |

How the contracts relate: content authored over **C4** lowers to a runtime artifact the engine loads
and then drives over **C1**; durable character state crosses every instance boundary as the **C2**
portable blob; **C3** governs how instances (each speaking C1) are spawned and routed to; the control
plane is the durable home of characters, catalog prototypes, leaderboards, and the item ledger.

## How to consume

- **Data/message contracts (C1, C2)** are JSON Schema (draft 2020-12). Validate payloads or generate
  types with any draft-2020-12 tooling.
- **Service contracts (C3, C4, control plane)** are OpenAPI 3.1 (YAML). Generate clients/servers or
  mock servers with standard OpenAPI tooling.
- Every spec is concrete: each major message and endpoint carries at least one worked example.

```bash
# Validate every JSON Schema parses
python -c "import json,glob; [json.load(open(f)) for f in glob.glob('**/*.json',recursive=True)]"
# Validate every OpenAPI YAML parses
python -c "import yaml,glob; [yaml.safe_load(open(f)) for f in glob.glob('**/*.yaml',recursive=True)]"
```

## Specs only, no source

This repository is **public** and deliberately contains **no source code and no internal
implementation detail** — no file paths, no internal symbol/function names, no private repository
structure, no design rationale. It is the consumer-facing API surface and nothing else. The
authoritative internal design documents live in private repositories and are not published here.

## Status & versioning

- Contracts marked **C1–C4** and **Character Store** reflect implemented or frozen-contract surfaces.
- Contracts marked **(planned)** (Catalog, Leaderboards, Economy) are **drafts derived from documented
  design intent** for services that are not yet built; their shapes are **subject to change**. They
  carry a `0.0.0-draft` version and are published to show the planned surface, not as frozen contracts.
- Each spec declares its own version in its `info.version` (OpenAPI) or `$id`/title (JSON Schema).
  Treat a non-`draft` version as the stable contract; breaking changes will bump the major version.
