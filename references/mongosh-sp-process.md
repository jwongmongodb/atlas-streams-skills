# mongosh sp.process() Reference

Use `sp.process()` for ephemeral pipeline execution — prototyping, debugging, and monitoring without deploying a processor.

## Connecting to a Streams Workspace

```javascript
// Connection string format (from Atlas UI → Streams → Connect)
mongosh "mongodb://atlas-stream-<workspace-id>.virginia-usa.a]...:27017/?authSource=%24external&authMechanism=MONGODB-X509&retryWrites=true&w=majority&tls=true&tlsCertificateKeyFile=<cert-path>"
```

Or from the Atlas UI: Streams workspace → Connect → mongosh tab → copy connection string.

## sp.process() — Ephemeral Pipeline Execution

Runs a pipeline in real-time without deploying a processor. Results stream to your terminal.

```javascript
// Basic usage
sp.process([
  { $source: { connectionName: "my-kafka", topic: "events" } },
  { $match: { status: "error" } },
  { $project: { message: 1, timestamp: 1 } }
])
```

Key properties:
- **Ephemeral** — no processor is created, no state is persisted
- **No billing** — does not incur Atlas Stream Processing charges
- **Sinkless OK** — terminal sink (`$merge`/`$emit`) is NOT required; output streams to your terminal
- **Real-time** — results appear as documents flow through the pipeline
- **Ctrl+C to stop** — execution ends immediately

## When to Use sp.process() vs Deployed Processors

| Aspect | sp.process() | Deployed Processor |
|--------|-------------|-------------------|
| **Billing** | Free | Per-second while running |
| **Persistence** | None — ephemeral | Checkpointed, survives restarts |
| **Sink required** | No — output to terminal | Yes — must end with `$merge`/`$emit` |
| **Use case** | Prototyping, debugging, monitoring | Production workloads |
| **State (windows)** | Lost on Ctrl+C | Preserved across stop/start |
| **Parallelism** | Single-threaded | Configurable per-stage |
| **Creation** | Instant | Provisioning delay |

## Inspection Commands

```javascript
// List all connections in the workspace
sp.listConnections()

// List all deployed processors
sp.listStreamProcessors()

// Get processor stats
sp.stats()
```

## Common Patterns

### Tail a Kafka topic (like `tail -f`)
```javascript
sp.process([
  { $source: { connectionName: "my-kafka", topic: "events" } }
])
```

### Filter and inspect
```javascript
sp.process([
  { $source: { connectionName: "my-kafka", topic: "orders" } },
  { $match: { "order.total": { $gt: 1000 } } },
  { $project: { orderId: 1, total: "$order.total", customer: "$order.customerId" } }
])
```

### Test a windowed aggregation
```javascript
sp.process([
  { $source: { connectionName: "my-kafka", topic: "clicks" } },
  { $tumblingWindow: {
      interval: { size: 30, unit: "second" },
      pipeline: [
        { $group: { _id: "$page", count: { $sum: 1 } } }
      ]
  }},
  { $project: { page: "$_id", clickCount: "$count" } }
])
```

### Validate a full pipeline before deploying
```javascript
// Test the exact pipeline you plan to deploy (minus the sink)
sp.process([
  { $source: { connectionName: "my-kafka", topic: "events" } },
  { $match: { type: "purchase" } },
  { $addFields: { processed_at: { $toDate: "$_stream_meta.arrivalTime" } } },
  { $project: { userId: 1, amount: 1, processed_at: 1 } }
  // Omit $merge/$emit — output goes to terminal
])
```

## Limitations

- No checkpointing — if you Ctrl+C during a window, accumulated state is lost
- Single-threaded — cannot test parallelism behavior
- No DLQ — errors appear inline in the terminal output
- Not suitable for load testing — use a deployed processor for throughput benchmarks
