---
name: "MongoDB Atlas Streams"
description: "Build, operate, and debug Atlas Stream Processing pipelines using MCP tools"
triggers:
  - "stream processing"
  - "streaming pipeline"
  - "atlas streams"
  - "ASP"
  - "kafka pipeline"
  - "deploy processor"
  - "stream processor"
  - "manage streams"
  - "list processors"
  - "set up pipeline"
  - "set up a Kafka pipeline"
  - "why is my processor failing"
  - "stop my processor"
  - "delete my workspace"
---

# MongoDB Atlas Streams

Build, operate, and debug Atlas Stream Processing (ASP) pipelines using four MCP tools from the MongoDB MCP Server: `atlas-streams-discover`, `atlas-streams-build`, `atlas-streams-manage`, and `atlas-streams-teardown`.

## Tool Selection Matrix

### atlas-streams-discover — ALL read operations
| Action | Use when |
|--------|----------|
| `list-workspaces` | See all workspaces in a project |
| `inspect-workspace` | Review workspace config, state, region |
| `list-connections` | See all connections in a workspace |
| `inspect-connection` | Check connection state, config, health |
| `list-processors` | See all processors in a workspace |
| `inspect-processor` | Check processor state, pipeline, config |
| `diagnose-processor` | Full health report: state, stats, errors |
| `get-logs` | Operational logs (runtime errors) or audit logs (lifecycle) |
| `get-networking` | PrivateLink and VPC peering details |

### atlas-streams-build — ALL create operations
| Resource | Key parameters |
|----------|---------------|
| `workspace` | `cloudProvider`, `region`, `tier` (default SP10), `includeSampleData` |
| `connection` | `connectionName`, `connectionType` (Kafka/Cluster/S3/Https/Kinesis/Lambda/SchemaRegistry/Sample), `connectionConfig` |
| `processor` | `processorName`, `pipeline` (must start with `$source`, end with `$merge`/`$emit`), `dlq`, `autoStart` |
| `privatelink` | `privateLinkProvider`, `privateLinkConfig` |

### atlas-streams-manage — ALL update/state operations
| Action | Notes |
|--------|-------|
| `start-processor` | Begins billing. Optional `tier` override, `resumeFromCheckpoint` |
| `stop-processor` | Stops billing. Retains state 45 days |
| `modify-processor` | Processor must be stopped first. Change pipeline, DLQ, or name |
| `update-workspace` | Change tier or region |
| `update-connection` | Update config (networking is immutable — must delete and recreate) |
| `accept-peering` / `reject-peering` | VPC peering management |

### atlas-streams-teardown — ALL delete operations
| Resource | Safety behavior |
|----------|----------------|
| `processor` | Auto-stops before deleting |
| `connection` | Blocks if referenced by running processor |
| `workspace` | Cascading delete of all connections and processors |
| `privatelink` / `peering` | Remove networking resources |

## CRITICAL: Validate Before Creating Processors

**You MUST call `search-knowledge` before composing any processor pipeline.** This is not optional. Query with the sink/source type, e.g. "Atlas Stream Processing $emit S3 fields" or "Atlas Stream Processing Kafka $source configuration". This validates field names and catches errors like `prefix` vs `path` for S3 `$emit`.

Also consult the official ASP examples repo: **https://github.com/mongodb/ASP_example** (33+ processors, 6 quickstarts). Key references:
| Quickstart | Pattern |
|-----------|---------|
| `00_hello_world.json` | Inline `$source.documents` with `$match` (zero infra, ephemeral) |
| `01_changestream_basic.json` | Change stream → tumbling window → `$merge` to Atlas |
| `03_kafka_to_mongo.json` | Kafka source → tumbling window rollup → `$merge` to Atlas |

## Pipeline Rules & Warnings

**Invalid constructs** — these are NOT valid in streaming pipelines:
- **`$$NOW`**, **`$$ROOT`**, **`$$CURRENT`** — NOT available in stream processing. NEVER use these. Use `$_stream_meta.timestamp` for event time instead of `$$NOW`.
- **HTTPS connections as `$source`** — HTTPS is for `$https` enrichment only
- **Kafka `$source` without `topic`** — topic field is required
- **Pipelines without a sink** — `$merge`/`$emit` required for deployed processors (sinkless only works via `sp.process()`)
- **Lambda as `$emit` target** — Lambda uses `$externalFunction` (mid-pipeline enrichment), not `$emit`
- **`$validate` with `validationAction: "error"`** — crashes processor; use `"dlq"` instead

**Required fields by stage:**
- **`$source` (change stream)**: include `fullDocument: "updateLookup"` to get the full document content
- **`$source` (Kinesis)**: use `stream` (NOT `streamName` or `topic`) for the Kinesis stream name. Example: `{"$source": {"connectionName": "my-kinesis", "stream": "my-stream"}}`
- **`$emit` (Kinesis)**: MUST include `partitionKey`. Example: `{"$emit": {"connectionName": "my-kinesis", "stream": "my-stream", "partitionKey": "$fieldName"}}`
- **`$emit` (S3)**: use `path` (NOT `prefix`). Example: `{"$emit": {"connectionName": "my-s3", "bucket": "my-bucket", "path": "data/year={$year}", "config": {"outputFormat": {"name": "json"}}}}`
- **`$https`**: must include `connectionName`, `path`, `method` (GET/POST), `as`, and `onError: "dlq"`
- **`$externalFunction`**: must include `connectionName`, `functionName`, `execution` ("sync"/"async"), `as`, `onError: "dlq"`
- **`$validate`**: must include `validator` with `$jsonSchema` and `validationAction: "dlq"`
- **`$lookup`**: include `parallelism` setting (e.g., `parallelism: 2`) for concurrent I/O
- **AWS connections** (S3, Kinesis, Lambda): IAM role ARN must be **registered in the Atlas project via Cloud Provider Access** before creating the connection. This is a prerequisite — the connection creation will fail without it. **Always mention this prerequisite** in your response, even if the user says connections already exist. Confirm: "IAM role ARNs are registered via Atlas Cloud Provider Access" or "Ensure IAM role ARNs are registered via Atlas Cloud Provider Access before creating connections."

**SchemaRegistry connection — always show these parameters explicitly:**
When creating a SchemaRegistry connection, show the exact `connectionType` and `connectionConfig` fields:
```json
{
  "connectionType": "SchemaRegistry",
  "connectionConfig": {
    "schemaRegistryUrls": ["https://registry.example.com"],
    "schemaRegistryAuthentication": {
      "type": "USER_INFO",
      "username": "...",
      "password": "..."
    }
  }
}
```
- `connectionType` MUST be `"SchemaRegistry"` (not `"Kafka"` or `"Https"`)
- `schemaRegistryUrls` is an **array** (not a string). The tool auto-wraps a string into an array if needed.
- `schemaRegistryAuthentication.type` can be `"USER_INFO"` (explicit credentials) or `"SASL_INHERIT"` (inherit from Kafka connection)
- Tool elicitation will collect sensitive fields (password) — don't ask the user for these directly

## MCP Tool Behaviors

**Elicitation:** When creating connections, the build tool auto-collects missing sensitive fields (passwords, bootstrap servers) via MCP elicitation. Do NOT ask the user for these — let the tool collect them.

**Auto-normalization:**
- `bootstrapServers` array → auto-converted to comma-separated string
- `schemaRegistryUrls` string → auto-wrapped in array
- `dbRoleToExecute` → defaults to `{role: "readWriteAnyDatabase", type: "BUILT_IN"}` for Cluster connections

**Region naming:** AWS uses `VIRGINIA_USA` (us-east-1), `OREGON_USA` (us-west-2); GCP uses `US_CENTRAL1`; Azure uses `US_EAST_1`. If unsure, inspect an existing workspace.

## Tier Reference

| Tier | vCPU | RAM | Max Parallelism | Use case |
|------|------|-----|-----------------|----------|
| SP2 | 0.25 | 512MB | 1 | Minimal filtering, testing |
| SP5 | 0.5 | 1GB | 2 | Simple filtering and routing |
| SP10 | 1 | 2GB | 8 | Moderate workloads, joins, grouping |
| SP30 | 2 | 8GB | 16 | Windows, JavaScript UDFs (`$function`), production |
| SP50 | 8 | 32GB | 64 | High throughput, large window state |

**`$function` (JavaScript UDFs) requires SP30+.** Memory rule: user state must stay below 80% of tier RAM.

## Core Workflows

### Setup from scratch
1. `atlas-streams-discover` → `list-workspaces` (check existing)
2. `atlas-streams-build` → `resource: "workspace"` (region near data, SP10 for dev)
3. `atlas-streams-build` → `resource: "connection"` (for each source/sink/enrichment)
4. Consult `search-knowledge` and https://github.com/mongodb/ASP_example before building pipeline
5. `atlas-streams-build` → `resource: "processor"` (with DLQ configured)
6. `atlas-streams-manage` → `start-processor` (warn about billing)

### Modify a pipeline
1. `atlas-streams-manage` → `stop-processor`
2. `atlas-streams-manage` → `modify-processor` (new pipeline)
3. `atlas-streams-manage` → `start-processor`

### Debug a failing processor
1. `atlas-streams-discover` → `diagnose-processor` (state, stats, errors)
2. `atlas-streams-discover` → `get-logs` (operational errors)
3. Check DLQ: MongoDB `count` + `find` on DLQ collection
4. Check output: MongoDB `count` + `find` on output collection
5. Classify processor type before assuming low output is a problem (see `references/output-diagnostics.md`)

### Chained processors (multi-sink pattern)
**CRITICAL: A single pipeline can only have ONE terminal sink** (`$merge` or `$emit`). You CANNOT have both `$merge` and `$emit` as terminal stages. When a user requests multiple output destinations (e.g., "write to Atlas AND emit to Kafka" or "archive to S3 AND send to Lambda"), you MUST:
1. **Acknowledge** the single-sink constraint explicitly in your response
2. **Propose chained processors**: Processor A reads source → enriches (including `$lookup` with `parallelism`) → writes to Atlas via `$merge`. Processor B reads from that Atlas collection via change stream `$source` → emits to second destination.
3. **Show both processor pipelines** including any `$lookup` enrichment stages with `parallelism` settings.

Note: `$externalFunction` (Lambda) is a mid-pipeline stage, NOT a terminal sink. A pipeline can use `$externalFunction` AND still have a terminal `$merge`/`$emit` — this is a valid single-sink pattern, but explain WHY it works (Lambda is invoked mid-pipeline, not as a sink).

### Verify after deploy
1. `atlas-streams-discover` → `inspect-processor` (state = STARTED)
2. `atlas-streams-discover` → `diagnose-processor` (health report)
3. MongoDB `count` on DLQ collection (should be 0)
4. MongoDB `find` on output collection (documents arriving with correct shape)

## Pre-Deploy Checklist

Before creating any processor:
- [ ] **`search-knowledge` was called** to validate sink/source field names (REQUIRED, not optional)
- [ ] Pipeline starts with `$source` and ends with `$merge` or `$emit`
- [ ] No `$$NOW`, `$$ROOT`, or `$$CURRENT` (invalid in streaming)
- [ ] Kafka sources include `topic` field
- [ ] HTTPS connections used only in `$https` stages, never as `$source`
- [ ] Lambda invoked via `$externalFunction` (mid-pipeline), not `$emit`; `execution` explicitly set to `"sync"` or `"async"`
- [ ] Schema validation uses `$validate` with `validationAction: "dlq"` (not `"error"`)
- [ ] DLQ configured; `$https`/`$externalFunction` use `onError: "dlq"`
- [ ] Windowed Kafka/Kinesis sources include `partitionIdleTimeout`/`shardIdleTimeout`
- [ ] All `connectionName` references match actual connections in workspace
- [ ] API auth stored in connection settings, not hardcoded in pipeline
- [ ] Tier appropriate for complexity (`$function` requires SP30+; see `references/sizing-and-parallelism.md`)

## Troubleshooting

| Symptom | Likely cause | Action |
|---------|-------------|--------|
| Processor FAILED on start | Invalid pipeline syntax, missing connection, `$$NOW` used | `diagnose-processor` → read error → fix pipeline |
| DLQ filling up | Schema mismatch, `$https` failures, type errors | `find` on DLQ → fix pipeline or connection |
| Zero output (transformation) | Connection issue, wrong topic, filter too strict | Check source health → verify connections → check `$match` |
| Zero output (alert) | Probably normal — no anomalies detected | Verify with known test event |
| Windows not closing | Idle Kafka partitions | Add `partitionIdleTimeout` to `$source` |
| OOM / processor crash | Tier too small for window state | `diagnose-processor` → check `memoryUsageBytes` → upgrade tier |
| Slow throughput | Low parallelism on I/O stages | Increase `parallelism` on `$merge`/`$lookup`/`$https` |
| 402 error on start | No billing configured | Add payment method in Atlas. Use `sp.process()` in mongosh as free alternative for testing |
| Parallelism exceeded | Tier too small for requested parallelism | Start with higher tier (see `references/sizing-and-parallelism.md`) |
| Networking change needed | Networking is immutable after creation | Delete connection and recreate with new networking config |

## Billing & Cost

**Atlas Stream Processing has no free tier.** All deployed processors incur continuous charges while running.

- Charges are per-hour, calculated per-second, only while the processor is running
- `stop-processor` stops billing; stopped processors retain state for 45 days at no charge
- Always confirm billing setup before starting processors
- **For prototyping without billing:** Use `sp.process()` in mongosh — runs pipelines ephemerally without deploying a processor
- Stop processors when not actively needed
- See `references/sizing-and-parallelism.md` for tier pricing and cost optimization strategies

## Safety Rules

1. **Confirm before starting** — Always warn about billing implications before `start-processor`
2. **Stop before modify** — Processors must be stopped before modifying pipeline or DLQ
3. **Check references before delete** — Use `list-processors` or `inspect-connection` to verify no running processors reference a connection before deleting it
4. **DLQ is mandatory for production** — Never deploy without a dead letter queue
5. **Validate pipeline first** — Consult `search-knowledge` and ASP_example repo before creating processors
6. **Don't hardcode secrets** — Store auth in connection config, never in pipeline expressions
7. **Networking is immutable** — Cannot modify after connection creation; must delete and recreate

## Reference Files

| File | Read when... |
|------|-------------|
| [`references/pipeline-patterns.md`](references/pipeline-patterns.md) | Building or modifying processor pipelines |
| [`references/connection-configs.md`](references/connection-configs.md) | Creating connections (type-specific schemas) |
| [`references/development-workflow.md`](references/development-workflow.md) | Following lifecycle management or debugging decision trees |
| [`references/output-diagnostics.md`](references/output-diagnostics.md) | Processor output is unexpected (zero, low, or wrong) |
| [`references/sizing-and-parallelism.md`](references/sizing-and-parallelism.md) | Choosing tiers, tuning parallelism, or optimizing cost |
