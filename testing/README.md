# streams-mcp-tools — Eval Suite

Measures how well the `streams-mcp-tools` skill improves Claude's ability to build, operate, and debug Atlas Stream Processing (ASP) pipelines.

## Overview

- **22 eval prompts**, each with **4 graded assertions**
- Run in two configs: **with_skill** (SKILL.md loaded) and **without_skill** (general knowledge only)
- Recommended: **4 samples per eval per config** for statistical robustness
- Total: **352 graded assertions**

## Latest Results

| Model | with_skill | without_skill | Skill lift |
|-------|-----------|---------------|-----------|
| Claude Opus 4.6 | 352/352 (100%) | 185/352 (52.6%) | +47.4pp |
| Claude Sonnet 4.6 | 337/352 (95.7%) | 145/352 (41.2%) | +54.5pp |

Sonnet 4.6 has lower baseline ASP knowledge, making the skill lift larger (+54.5pp vs +47.4pp).

## Files

| Path | Purpose |
|------|---------|
| `evals/evals.json` | 22 eval prompts + assertions (self-contained) |
| `evals/metadata/*.json` | Rich detail files for top evals (common mistakes, expected pipelines) |

## The 22 Evals

| # | Name | What it tests | Sonnet Delta |
|---|------|---------------|-------------|
| 1 | $$NOW trap | $$NOW is invalid; must use `_stream_meta.arrivalTime` | +75pp |
| 2 | Teardown safety | Inspect before deleting; show counts; ask confirmation | +75pp |
| 3 | Sink requirement | Deployed processors need a sink; `sp.process()` for monitoring | +69pp |
| 4 | $emit for Kafka | Use `$emit` not `$merge` for Kafka sinks | +56pp |
| 5 | Sizing/parallelism | SP30+ for high throughput; parallelism on `$lookup`/`$merge` | +50pp |
| 6 | Connection deps | Connection deletion blocked by running processors | +56pp |
| 7 | Parallelism calc | `sum(parallelism-1)` formula to determine minimum tier | +88pp |
| 8 | Elicitation | Don't ask for passwords; let MCP tool elicitation collect them | +88pp |
| 9 | Window debugging | Idle Kafka partitions block windows; `partitionIdleTimeout` fix | +69pp |
| 10 | S3 sink | Use `path` (not `prefix`); `outputFormat`; IAM prereq | +81pp |
| 11 | HTTPS source trap | HTTPS cannot be `$source`; use Kafka/Cluster + HTTPS enrichment | +63pp |
| 12 | HTTPS enrichment | `$https` needs `connectionName`, `path`, `method`, `as`, `onError: "dlq"` | +50pp |
| 13 | Kinesis | Use `stream` field (not `topic`); `$emit` needs `partitionKey` | +69pp |
| 14 | Lambda mid-pipeline | Use `$externalFunction` not `$emit`; include `onError: "dlq"` | +81pp |
| 15 | Lambda sink | `$externalFunction` as sink must use `execution: "async"` | +63pp |
| 16 | SchemaRegistry | `connectionType: "SchemaRegistry"` with provider + URLs + auth | +44pp |
| 17 | Chained processors | Single-sink constraint; two chained processors for dual output | +75pp |
| 18 | $validate DLQ | `validationAction: "error"` crashes processor; use `"dlq"` | +31pp |
| 19 | mongosh inspection | `sp.listConnections()`, `sp.listStreamProcessors()`, `sp.stats()` | +87pp |
| 20 | No-tools pipeline test | Pivot to `sp.process()` when atlas-streams tools unavailable | +63pp |
| 21 | No-tools config fix | Diagnose missing `previewFeatures: ["streams"]`; offer CLI | +62pp |
| 22 | CLI list processors | `atlas streams processors list --instance` fallback syntax | +56pp |

## Assertion Types

- **`contains`** — Response must include specific knowledge
- **`absence`** — Response must NOT contain an anti-pattern
- **`structural`** — Pipeline JSON must have correct structure
- **`process`** — Correct tool call sequence

## Running the Evals

For each eval in `evals/evals.json`:

1. Set up two Claude sessions:
   - **with_skill:** Load `SKILL.md` into context, then give the eval prompt
   - **without_skill:** Give the eval prompt directly (no skill)

2. Grade each response against the 4 assertions

3. Record pass/fail. Calculate:
   - `with_skill pass rate = passed / total`
   - `without_skill pass rate = passed / total`
   - `delta = with_skill - without_skill`

Recommended: run each eval 4 times per config. Single runs can be misleading due to LLM variance.

## Eval History

Evals removed (2026-03-11): Change stream (+6pp, general knowledge covers it), AWS IAM prereqs (0pp on Opus, tool description covers it), Billing friction prod/seq (hypothesis validated over 5 iterations).

Evals added (2026-03-11): mongosh inspection (+87pp), no-tools pipeline (+63pp), no-tools config fix (+62pp), CLI list processors (+56pp).

## What This Does NOT Cover

- **Live MCP tool execution** — Tests knowledge/behavior, not API connectivity
- **Multi-turn conversations** — Each eval is a single prompt
- **Edge cases in tool responses** — e.g., Atlas API error handling
