# Vision

Rhizomatic is a stack for declarative distributed state tracking. Not a database — a substrate. Assembly language for data.

## The Problem

Systems that track state over time tend to collapse several distinct concerns into a single layer: encoding, storage, indexing, querying, conflict resolution. This makes them powerful out of the box but hard to reason about, hard to decompose, and hard to adapt to novel use cases.

rhizomedb was an attempt at a rhizomatic database. It worked, but it tried to be everything at once. Rhizomatic decomposes the problem into layers, each independently useful, each composable.

## Core Concepts

### Claims, Not Deltas

The fundamental unit is the **claim** — a hyperedge asserting that some relationship holds between nodes within a system as of some point in time.

A claim is not a mutation. It doesn't say "X changed to Y." It says "I attest that this relationship exists." This distinction matters:

- An event stream encodes temporal causality: X happened, then Y happened
- A claim encodes witnessed structure: this relationship was observed/asserted at this time

Claims carry epistemic humility. They are put forward, not dictated. Multiple claims can coexist in contradiction. The system doesn't resolve truth — it accumulates perspective. Truth is assembled by the observer at query time, not imposed by the substrate.

This means historical time and system time never collapse. You can ingest a 1960 census record in 2026. The claims assert relationships as of 1960, witnessed by you in 2026 on your system. Both are preserved. Neither is privileged.

### Transactions

A **transaction** is an ordered set of claims sharing provenance:

- `tx_id` — unique identifier for the transaction
- `timestamp` — when the claims were recorded
- `host_id` — the system that recorded them
- `author_id` — the entity that authored them

Individual claims within a transaction are identified as `<tx_id>.<index>`. Within the transaction, the index is sufficient. When a claim travels outside its transaction context, the full identifier hydrates naturally.

### The Layer Stack

Rhizomatic is not one thing. It's a stack of composable layers:

1. **Encoder** — `structured data → transaction(claims)`. Pure function. No storage, no indexing, no schemas. Data in, claims out.
2. **Storage** — Persistence of transactions. Append-only. No interpretation.
3. **Indexing** — Materialized views over the claim graph. Multiple strategies, all derived.
4. **Query** — Composing claims into coherent views. This is where "truth" gets assembled.
5. **Schema** — Optional structural constraints. Layered on top, never baked in.

Each layer depends only on the one below it. Each is independently replaceable. The encoder doesn't know about storage. Storage doesn't know about queries. This is the decomposition rhizomedb lacked.

## Philosophy

> "A rhizome has no beginning or end; it is always in the middle, between things, interbeing, intermezzo." — Deleuze & Guattari

No single privileged view. No canonical root. Claims from different nodes, different times, different authors coexist. The substrate doesn't adjudicate — it witnesses. Meaning emerges from composition, not from authority.

## Prior Art

- **rhizomedb** — Immutable delta-CRDTs as hyperedges, state assembled at query-time. The direct ancestor. Collapsed too many layers into one.
- **Datomic** — Immutable facts, time-aware, query-time assembly. Closest in spirit but centralized and schema-first.
- **CRDTs** — Conflict-free replicated data types. Rhizomatic claims are CRDT-compatible but the conflict resolution strategy is not baked into the substrate.
- **Event Sourcing** — Append-only event logs. Similar append-only philosophy but events encode causality; claims encode structure.
