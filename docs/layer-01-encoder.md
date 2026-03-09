# Encoder

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
  
  // Structural content — the actual claim
  nodes: NodeRef[];     // Ordered array of node references
  label?: string;       // Optional relationship label
  value?: any;          // Optional literal value
}

interface NodeRef {
  id: string;           // Globally unique node identifier
  type?: string;        // Optional type hint
}
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

## Encoding Strategies

The encoder is not opinionated about input structure. Different strategies can be applied:

### Record Encoding

Input: `{ user: { id: "u1", name: "Alice", age: 30 } }`

Output claims:
- `[{ id: "u1" }], "hasAttribute", "name", "Alice"`
- `[{ id: "u1" }], "hasAttribute", "age", 30`

### Graph Encoding

Input: Graph with nodes and edges

Output: One claim per edge, nodes in the nodes array

### Diff Encoding (Legacy)

Input: `{ before: {...}, after: {...} }`

Output: Claims only for changed fields. This is supported but not privileged — the claim format doesn't care whether it's derived from a diff or a snapshot.

## Non-Goals

The encoder explicitly does NOT:

- **Validate schemas**: No structural constraints. You can encode anything.
- **Check uniqueness**: Multiple claims can assert the same relationship.
- **Resolve conflicts**: Contradictory claims are both valid.
- **Index data**: No lookups, no materialized views.
- **Store anything**: Pure function, no side effects.
- **Interpret meaning**: It produces claims. What they mean is a query-layer concern.

## Extensibility

Encoder configurations can provide:

- Custom claim formats
- Different node reference strategies
- Label conventions
- Value serialization rules

But the transaction envelope format is fixed. All encoders produce the same transaction structure, enabling the rest of the stack to work uniformly.

## Testing

Because the encoder is pure, it's fully testable without mocks:

```typescript
const input = { user: { id: "u1", name: "Alice" } };
const tx = encode(input);

assert(tx.claims.length === 2);
assert(tx.claims[0].nodes[0].id === "u1");
assert(tx.claims[0].label === "hasAttribute");
```
