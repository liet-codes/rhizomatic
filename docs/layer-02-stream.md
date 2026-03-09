# Layer 2: Stream

The stream is the backbone. Encoders produce transactions; the stream transports and orders them. Storage, indexing, federation, and query layers are all consumers of the stream.

## The Protocol

```typescript
interface Stream {
  // Append a transaction to the stream
  append(tx: Transaction): Promise<void>;
  
  // Subscribe to new transactions
  subscribe(handler: (tx: Transaction) => void): Subscription;
  
  // Query the transaction log
  // (Implementation detail varies by storage backend)
  query(options: QueryOptions): Promise<Transaction[]>;
}
```

The stream is **append-only** and **ordered**. Once a transaction is appended, it has a permanent position in the log. The stream maintains the source of truth — everything else is derived from it.

## Why Streaming?

Traditional databases invert the dependency:

```
Encoder → Data → Storage ← Query Engine
                           ↑
                       Indexing
```

Storage is "where data lives." Indexing is a separate concern that must be kept in sync. If indexing falls behind or gets corrupted, you have inconsistency.

With streaming:

```
Encoder → Stream → [Storage, Indexing, Federation, UI, ...]
          ↑ source of truth
          ↑ append-only log
```

The stream is the source of truth. Every consumer subscribes to it and materializes its own view. If a consumer falls behind, it can replay from a checkpoint. If a consumer crashes, it just resumes where it left off.

## Architecture

### Sources

Who produces transactions for the stream?

- **Encoders**: Domain-specific encoders (code, LLM, human) produce transactions
- **Federation**: Peer systems send replicated transactions
- **Import pipelines**: Bulk ingestion from external sources
- **Repair/correction**: Negation claims or repair transactions

### Consumers

Who subscribes to the stream?

- **Storage**: Persists transactions to disk/database
- **Indexing**: Builds materialized views (indexes, search indices, etc.)
- **Federation**: Replicates to peer systems
- **Real-time UI**: Updates dashboards as claims arrive
- **Compute**: Derives insights or triggers workflows
- **Archival**: Long-term cold storage

Each consumer has its own subscription. They don't wait for each other. A slow indexer doesn't block storage. A down UI doesn't block federation.

### Transport Implementations

The stream protocol doesn't prescribe transport. Different implementations suit different use cases:

**In-process (simplest)**
```typescript
class InMemoryStream implements Stream {
  private transactions: Transaction[] = [];
  private subscribers: Set<Subscriber> = new Set();
  
  append(tx: Transaction) {
    this.transactions.push(tx);
    this.subscribers.forEach(s => s.handle(tx));
  }
  
  subscribe(handler) {
    const subscriber = { handle: handler };
    this.subscribers.add(subscriber);
    return { unsubscribe: () => this.subscribers.delete(subscriber) };
  }
}
```

**Message queue (Kafka, NATS, RabbitMQ)**
Transactions published to a distributed log. Consumers track their position independently. Enables federation and cross-network replication.

**Blockchain**
An append-only ledger with cryptographic verification. Suitable for high-trust scenarios where verifiability matters. More overhead, but transparent and auditable.

**HTTP feed (RSS/Atom-like)**
A simple REST API that serves transactions as a feed. Lowest-friction federation. Consumers poll or use webhooks.

**Custom**
Run your own transport. The protocol is the only contract.

## Ordering & Consistency

The stream guarantees:

- **Total order**: Every transaction has a position. Position is immutable.
- **Append-only**: Transactions never change or are deleted. To undo, you negate.
- **No gaps**: All positions from 0 to N are filled (no skipped indices).

These properties enable:
- Deterministic replay (two consumers seeing the same sequence see the same state)
- Easy federation (peers can sync by position)
- Auditability (full history is preserved)

Within these constraints, the implementation is flexible. You could use a file, a database, a blockchain, or a message broker.

## Sources & Sinks Terminology

- **Source**: A producer of transactions (encoder, peer, importer)
- **Sink**: A consumer that persists or re-emits (storage, federation gateway)
- **Servers**: Node-level components that run source/sink logic

A single system might host multiple sources and sinks. A research lab might run:
- An encoder that takes CSV → transactions (source)
- An indexing sink that materializes query indices
- A federation sink that replicates to collaborators
- A storage sink that archives to S3

Each is independently scalable.

## Schema in the Stream?

The stream might carry metadata about which vocabulary (roles + contexts) it uses. This helps consumers validate on intake:

```typescript
interface StreamMetadata {
  vocabulary: {
    roles: string[];
    contexts: string[];
  };
  source?: string; // Where this stream originated
  version?: string;
}
```

Consumers can:
- Reject claims with unknown roles/contexts
- Route to appropriate handlers (orgchem stream → chemistry tools)
- Validate conformance before persistence

This is an open question — where exactly does schema validation live? With the stream metadata? With individual encoders? With consumers?

## Non-Goals

The stream does NOT:

- **Validate semantics**: It enforces append-only and order, not meaning
- **Resolve conflicts**: Multiple contradictory claims are both valid
- **Make queries fast**: It's for replication and ordering, not query optimization
- **Handle storage**: It moves data; persistence is a consumer concern
- **Index for search**: Indexing is a consumer

## Key Property: Replay-able

Any consumer can catch up by replaying from a checkpoint. This means:

- **New indexing strategy?** Spin up a new consumer, replay the entire stream, build the new index, then go live.
- **Bug fix in a sink?** Fix the code, reset the checkpoint, replay.
- **Federation/replication?** New peer joins, replays from the start, materializes its own copy.

The stream is the single source of truth. Everything else is expendable.
