# Layer 1: Encoder

The encoder transforms input into transactions containing ordered sets of claims.

This is a **protocol**, not a library. The interface is abstract — implementations can be programmatic code, LLM agents, or even humans manually encoding observations. Rhizomatic is agent-first: the most interesting encoders will likely be LLMs given a vocabulary and asked to extract claims from unstructured input.

## The Protocol

```typescript
interface Encoder<T> {
  encode(input: T, metadata: EncoderMetadata): Transaction;
}

interface EncoderMetadata {
  author_id: string;
  host_id: string;
  source?: string;  // Where input came from (URL, DOI, file path, etc.)
}
```

The protocol is implementation-agnostic. What matters is the contract: input goes in, valid transactions come out.

## Vocabulary

Each encoder operates with a **vocabulary** — a set of meaningful roles and contexts for its domain. The vocabulary is the domain-specific configuration that tells the encoder what kinds of relationships to express.

Example vocabulary for academic research:
```
roles: author, finding, method, cites, contradicts, supports, dataset
contexts: bibliography, methodology, results, discussion
```

Example vocabulary for e-commerce:
```
roles: purchaser, vendor, item, price, currency, date
contexts: purchase, sales, receipt, inventory
```

The vocabulary constrains what the encoder produces. Validation at this layer is vocabulary conformance: "Did you produce claims using known roles and contexts?"

## Implementations

### Programmatic Encoder
A function that transforms structured data into claims. Deterministic, testable, fast.

```typescript
const userEncoder: Encoder<UserRecord> = {
  encode(input, meta) {
    return {
      tx_id: generateId(),
      timestamp: Date.now(),
      host_id: meta.host_id,
      author_id: meta.author_id,
      claims: [
        { pointers: [
          { role: "entity", target: { id: input.id, context: "name" } },
          { role: "name", target: input.name }
        ]},
        { pointers: [
          { role: "entity", target: { id: input.id, context: "age" } },
          { role: "age", target: input.age }
        ]}
      ]
    };
  }
};
```

### LLM Encoder
An agent given vocabulary + unstructured input, asked to extract claims.

```
System: You are a claim encoder for an academic research database.
Your vocabulary:
  Roles: author, finding, method, cites, contradicts, supports
  Contexts: bibliography, methodology, results

Given the following paper, extract claims as JSON transactions
using only the roles and contexts above.

User: [full text of research paper]

Output: Transaction { claims: [...] }
```

The LLM is the encoder. Same protocol, same output format, radically different implementation. This is what "agent-first" means — the most powerful encoders aren't hand-coded parsers, they're agents that understand context.

### Human Encoder
A person manually encoding observations into claims via a UI or form. Same output format.

## Data Structures

### Transaction

```typescript
interface Transaction {
  tx_id: string;        // UUID or ULID
  timestamp: number;    // Unix timestamp (milliseconds)
  host_id: string;      // System that recorded
  author_id: string;    // Entity that authored
  source?: string;      // Where input came from
  claims: Claim[];      // Ordered payload
}
```

### Claim

```typescript
interface Claim {
  pointers: Pointer[];
}

interface Pointer {
  role: string;
  target: Reference | Primitive;
}

interface Reference {
  id: string;           // Entity, claim (tx_id.index), or transaction ID
  context?: string;     // Under which property of this node the claim is embedded
}

type Primitive = string | number | boolean;
```

## Understanding Context

Context answers: **"From this node's point of view, what kind of relationship is this?"**

For every Reference in a claim, the context specifies the property of the targeted node under which the entire claim is embedded.

```typescript
// A purchase claim:
{ pointers: [
  { role: "purchaser", target: { id: "buyer_123", context: "purchase" } },
  // ^ From buyer_123's perspective, this claim is a "purchase"
  { role: "vendor", target: { id: "vendor_456", context: "sales" } },
  // ^ From vendor_456's perspective, this claim is a "sale"
  { role: "receipt", target: { id: "purchase_789", context: "receipt" } },
  // ^ From the purchase record's perspective, this is its "receipt"
  { role: "item", target: { id: "widget_42" } },
  // ^ No context — or context could be "sold" if we want to track from item's view
  { role: "price", target: 15.99 },
  // ^ Primitive — no context. 15.99 is not a node.
  { role: "currency", target: "USD" }
  // ^ Primitive — just a value.
]}
```

**Primitives don't get context.** Context is for entities (nodes with identity). Primitives are terminal values — they don't have properties, perspectives, or embedded relationships.

## Claim Identity & Discovery

A claim's identity is `<tx_id>.<index>`.

- **Within transaction**: Index suffices.
- **Outside transaction**: Full compound ID.
- **Claims are nodes**: Any claim can be referenced by other claims via its compound ID in a Reference.
- **Transactions are nodes**: A transaction's `tx_id` can be targeted by References.

**Associative discovery**: Claims sharing a transaction are semantically related by provenance. Encountering one claim lets you discover its siblings — a high-dimensional associative space where co-occurrence signals relevance.

## Negation

Claims can negate other claims or void entire transactions:

```typescript
// Negate a specific claim
{ pointers: [
  { role: "negates", target: { id: "tx_abc123.5" } }
]}

// Void an entire transaction
{ pointers: [
  { role: "voids", target: { id: "tx_def456" } }
]}
```

Negation is produced by the encoder but interpreted by the query layer.

## Validation

Layer 1 validation is lightweight:
- Does the transaction have valid provenance metadata?
- Does each claim have at least one pointer?
- Are all roles and contexts from the declared vocabulary?

Structural/semantic validation (does this claim *make sense*?) is not Layer 1's concern. The encoder produces claims. Whether they're meaningful is between the encoder implementation and its domain.

## Non-Goals

The encoder does NOT:
- Validate schemas or structural constraints
- Check uniqueness or resolve conflicts
- Index, store, or query anything
- Enforce encoding strategy or granularity
- Prescribe implementation (code, LLM, human)

## Interface Boundaries

**Left (input)**: Anything. Structured data, unstructured text, images, events. Domain-specific.

**Right (output)**: `Transaction` — the universal interchange format. Whatever consumes transactions (Layer 2: Storage) accepts this structure and only this structure.

**Configuration**: Vocabulary (roles + contexts) provided by the domain. Validation checks conformance.

## Ecosystem: Claim Builder Libraries

The encoder protocol defines shapes. On top of it, domain communities can build and share **claim builder libraries** — packages of functions that produce claims with standardized roles and contexts for a specific field.

### Example: Organic Chemistry

```typescript
// @rhizomatic/orgchem — shared claim builders for computational chemistry

function reaction(opts: {
  reactants: string[],
  products: string[],
  catalyst?: string,
  conditions?: string
}): Claim {
  return {
    pointers: [
      ...opts.reactants.map(r => ({ role: "reactant", target: { id: r, context: "reactions" } })),
      ...opts.products.map(p => ({ role: "product", target: { id: p, context: "synthesis" } })),
      ...(opts.catalyst ? [{ role: "catalyst", target: { id: opts.catalyst, context: "catalysis" } }] : []),
      ...(opts.conditions ? [{ role: "conditions", target: opts.conditions }] : [])
    ]
  };
}

function synthesis(opts: {
  steps: string[],
  yield_pct: number,
  purity_pct: number
}): Claim { /* ... */ }

function spectralAnalysis(opts: {
  compound: string,
  method: string, // NMR, IR, MS, etc.
  peaks: number[]
}): Claim { /* ... */ }
```

### The Interoperability Play

If you use `@rhizomatic/orgchem` builders:
- Your claims interoperate with any tool that understands the orgchem vocabulary
- A growing ecosystem of computational chemistry tools can consume your data
- Claims can be used to fine-tune domain-specific models
- Different encoders (LLM, programmatic, instrument pipelines) produce compatible output

The builders are just convenience. They don't add new semantics — they encode domain conventions into reusable functions that guarantee vocabulary conformance. The claims they produce are standard claims. Nothing special about them structurally.

### Hybrid Encoder Patterns

An LLM encoder might:
1. **Receive builders as tools** — call `reaction()`, `synthesis()` as function calls
2. **Write its own builders** — generate helper functions for consistent output structure, then use them
3. **Hand-roll claims** — construct each claim from scratch using the raw vocabulary
4. **Mix approaches** — use builders for common patterns, hand-roll for unusual observations

A programmatic encoder might:
1. **Import builders directly** — call them from instrument data pipelines
2. **Compose builders** — chain multiple builders to represent complex multi-step processes

A human encoder might:
1. **Use a UI powered by builders** — form fields map to builder arguments
2. **Override with raw claims** — when the builders don't capture what they need to express

The protocol doesn't care. Builders are a convenience layer, not a requirement. The only requirement is: valid transactions come out the other end.
