# Layer 1: Encoder

The encoder is a pure function that transforms structured data into a transaction containing an ordered set of claims.

## Contract

```
encode : (input: any, config?: EncoderConfig) => Transaction
```

The encoder has no dependencies on storage, indexing, or query layers. It is stateless, deterministic, and side-effect free.

## Transaction Structure

```typescript
interface Transaction {
  // Provenance metadata — shared by all claims in the transaction
  tx_id: string;        // UUID or ULID
  timestamp: number;    // Unix timestamp (milliseconds)
  host_id: string;      // System identifier (machine, node, etc.)
  author_id: string;    // Entity identifier (user, process, etc.)
  
  // Ordered payload
  claims: Claim[];
}

interface Claim {
  // Identity is positional within transaction
  // Full claim_id: <tx_id>.<index>
  // Within transaction, index is sufficient for reference
  
  // Structural content — the claim itself
  pointers: Pointer[];  // Ordered array of contextualized pointers
}

interface Pointer {
  // The semantic role of this pointer from the claim's perspective
  role: string;
  
  // The referenced entity or value
  target: Reference | Primitive;
}

interface Reference {
  // Globally unique identifier of the referenced entity
  id: string;
  
  // Optional organizational context
  context?: string;
}

type Primitive = string | number | boolean;
// Note: No null, undefined, or array primitives
// - Absence is represented by lack of claims
// - Arrays are represented by multiple pointers with the same role
```

## Claim Identity

A claim's canonical identity is `<tx_id>.<index>` where index is its position in the transaction's claims array.

- **Within transaction**: The index alone is sufficient. `claims[3]` is claim 3.
- **Outside transaction**: The full identifier is `tx_abc123.3`.
- **Extraction**: When a claim is extracted from its transaction context, it carries its full identity with it.

This design means:
- Claims never need explicit `claim_id` fields in storage
- References between claims within a transaction are compact (just an index)
- Cross-transaction references use the full identifier
- Identity is deterministic and derivable from provenance

## Why "Claim" Instead of "Delta"

rhizomedb called these "deltas" but this was misleading. A delta suggests a change — a difference between before and after states. The term carries baggage from event sourcing and CRDT literature that implies temporal causality.

A **claim** is an assertion that some relationship holds as of time `t`. It might reflect a real-time event, or it might be the ingestion of historical data. The claim itself is agnostic to this distinction. What matters is that it asserts structure within a system at a point in time.

Historical time and system time never collapse. A claim ingested in 2026 can assert relationships as of 1960. Both timestamps are preserved: the claim's `timestamp` field records when it was witnessed; the pointers themselves may contain historical timestamps or version information.

## Encoding Strategies

The encoder is not opinionated about input structure. Different strategies can be applied:

### Record Encoding

Input: `{ user: { id: "u1", name: "Alice", age: 30 } }`

Output claims:
```typescript
{
  pointers: [
    { role: "entity", target: { id: "u1" } },
    { role: "attribute", target: "name" },
    { role: "value", target: "Alice" }
  ]
}
```

### Graph Encoding

Input: Graph with nodes and edges

Output: One claim per edge, with pointers for source, target, and relationship type.

### Temporal Encoding

Input: Historical record with effective date

Output: Claims with pointers that include temporal context, enabling queries across historical time.

## Non-Goals

The encoder explicitly does NOT:

- **Validate schemas**: No structural constraints. You can encode anything.
- **Check uniqueness**: Multiple claims can assert the same relationship.
- **Resolve conflicts**: Contradictory claims are both valid.
- **Index data**: No lookups, no materialized views.
- **Store anything**: Pure function, no side effects.
- **Interpret meaning**: It produces claims. What they mean is a query-layer concern.
- **Diff against previous state**: The encoder takes stateful data and expresses it as claims. Whether those claims are novel or redundant is not its concern.

## Extensibility

Encoder configurations can provide:

- Custom pointer formats
- Different reference strategies
- Role conventions
- Value serialization rules

But the transaction envelope format is fixed. All encoders produce the same transaction structure, enabling the rest of the stack to work uniformly.

## Testing

Because the encoder is pure, it's fully testable without mocks:

```typescript
const input = { user: { id: "u1", name: "Alice" } };
const tx = encode(input);

assert(tx.claims.length === 1);
assert(tx.claims[0].pointers[0].role === "entity");
assert(tx.claims[0].pointers[0].target.id === "u1");
```
