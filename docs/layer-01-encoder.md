# Layer 1: Encoder

The encoder transforms structured input into transactions containing ordered sets of claims.

This is not a single function. It is a **protocol** — an interface that users implement for their domains. The core library provides the data structures and validation; you provide the domain-specific encoding logic.

## The Protocol

```typescript
interface Encoder<T> {
  encode(input: T, metadata: EncoderMetadata): Transaction;
}

interface EncoderMetadata {
  author_id: string;
  host_id: string;
  // Optional source reference — where this input came from
  source?: string;  // URL, file path, citation, DOI, etc.
}
```

Encoders are **pure functions** (stateless, deterministic, side-effect free) but they are **domain-specific**. There is no universal encoder. A user record, a purchase transaction, and a research paper each need their own encoder that understands their semantics.

## Why Domain-Specific?

Consider two inputs:

### User Record: `{ id: "u1", name: "Alice", age: 30 }`

Name and age can vary independently. Encode as **separate claims**:

```typescript
// Claim 1: name
{ pointers: [
  { role: "entity", target: { id: "u1", context: "name" } },
  { role: "name", target: "Alice" }
]}

// Claim 2: age  
{ pointers: [
  { role: "entity", target: { id: "u1", context: "age" } },
  { role: "age", target: 30 }
]}
```

### Purchase Record: `{ purchase_id, item_id, vendor_name, purchase_date, buyer_id, price_number, price_currency }`

These fields are **tightly coupled**. Changing any one describes a different purchase. Encode as **one complex claim**:

```typescript
{ pointers: [
  { role: "purchase", target: { id: purchase_id } },
  { role: "item", target: { id: item_id } },
  { role: "vendor", target: vendor_name },
  { role: "date", target: purchase_date },
  { role: "buyer", target: { id: buyer_id } },
  { role: "price", target: price_number },
  { role: "currency", target: price_currency }
]}
```

### Research Paper: Full PDF + metadata

The encoder might extract **multiple claims** representing key insights, findings, or citations. Each claim carries different pointers to the paper's entities, methods, results.

The encoder decides: one input → one claim, or one input → many claims, or one input → claims plus nested transactions. The library doesn't prescribe; the domain decides.

## Data Structures

### Transaction

```typescript
interface Transaction {
  // Provenance metadata — shared by all claims
  tx_id: string;        // UUID or ULID
  timestamp: number;    // Unix timestamp (milliseconds)
  host_id: string;      // System that recorded
  author_id: string;    // Entity that authored
  
  // Optional source reference — where input came from
  // URL, file path, DOI, citation, database record ID, etc.
  source?: string;
  
  // Ordered payload
  claims: Claim[];
}
```

The `source` field enables provenance tracking: "This transaction represents claims extracted from [source]."

### Claim

```typescript
interface Claim {
  pointers: Pointer[];
}

interface Pointer {
  role: string;
  target: Reference | Primitive | ClaimRef;
}

interface Reference {
  id: string;
  context?: string;
}

interface ClaimRef {
  claim_id: string;  // <tx_id>.<index> format
}

type Primitive = string | number | boolean;
```

Claims can reference other claims via `ClaimRef`, enabling:
- Negation (claim with role "negates" targets another claim)
- Commentary (claim with role "elaborates" targets another claim)
- Composition (claims building on claims)

## Claim Identity & Discovery

A claim's identity is `<tx_id>.<index>` where `index` is its position in the transaction's `claims` array.

**Within transaction**: Index alone suffices (`claims[3]`).

**Outside transaction**: Full ID (`tx_abc123.3`).

**Key property**: Claims are never orphaned. If you encounter a claim in isolation, you can resolve its full identity and retrieve its siblings from the same transaction. This creates a **high-dimensional associative space** — claims physically colocated in a transaction are semantically related by shared provenance, even if they don't directly reference each other.

Transactions can also be modeled as nodes. A claim can reference another transaction by its `tx_id`, enabling claims about claims, meta-commentary, or bulk negation.

## Negation

Claims can negate other claims or entire transactions:

```typescript
// Negate a specific claim
{ pointers: [
  { role: "negates", target: { claim_id: "tx_abc123.5" } }
]}

// Void an entire transaction
{ pointers: [
  { role: "voids", target: { id: "tx_def456" } }
]}
```

The query layer (Layer 4) interprets negation. The encoder just produces the claims.

## Non-Goals

The encoder layer explicitly does NOT:

- **Validate schemas**: No structural constraints on input
- **Check uniqueness**: Multiple claims can assert the same relationship
- **Resolve conflicts**: Contradictory claims are both valid
- **Index data**: No lookups, no materialized views
- **Store anything**: Pure function, no side effects
- **Interpret meaning**: Produces claims; meaning is query-layer concern
- **Enforce encoding strategy**: Domain decides granularity

## Testing

Encoders are pure and testable:

```typescript
const encoder: Encoder<UserRecord> = {
  encode(input, meta) {
    return {
      tx_id: generateUUID(),
      timestamp: Date.now(),
      host_id: meta.host_id,
      author_id: meta.author_id,
      claims: [
        { pointers: [
          { role: "entity", target: { id: input.id, context: "name" } },
          { role: "name", target: input.name }
        ]}
        // ... more claims for other fields
      ]
    };
  }
};

// Test
test("user encoder produces claim for name", () => {
  const tx = encoder.encode({ id: "u1", name: "Alice" }, meta);
  assert(tx.claims[0].pointers[0].target.context === "name");
});
```

## Layer 1 Summary

- **Input**: Domain-specific structured data (records, events, documents)
- **Process**: Domain-specific encoder implements `encode(input, metadata) → Transaction`
- **Output**: Transaction containing ordered claims with pointers
- **Key properties**: Pure, domain-specific, no storage/query/indexing, provenance-tracked
