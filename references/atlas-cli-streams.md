# Atlas CLI — Streams Commands Reference

Use when MCP tools are unavailable (server not connected, credentials expired, or local debugging).

## Prerequisites

- Atlas CLI installed (`brew install mongodb-atlas-cli` or [download](https://www.mongodb.com/docs/atlas/cli/stable/install-atlas-cli/))
- Authenticated: `atlas auth login`
- Project context: pass `--projectId <id>` or set `MONGODB_ATLAS_PROJECT_ID`

## Workspace Commands

```bash
atlas streams instances list --projectId <id>
atlas streams instances create <name> --provider AWS --region VIRGINIA_USA --projectId <id>
atlas streams instances describe <name> --projectId <id>
atlas streams instances delete <name> --projectId <id> --force
```

## Connection Commands

```bash
atlas streams connections list --instance <workspace> --projectId <id>
atlas streams connections create --instance <workspace> --file connection.json --projectId <id>
atlas streams connections describe <name> --instance <workspace> --projectId <id>
atlas streams connections delete <name> --instance <workspace> --projectId <id> --force
```

Connection config JSON example (Kafka):
```json
{
  "name": "my-kafka",
  "type": "Kafka",
  "config": {
    "bootstrapServers": "broker1:9092,broker2:9092"
  },
  "authentication": {
    "mechanism": "SCRAM-256",
    "username": "user",
    "password": "pass"
  }
}
```

## Processor Commands

```bash
atlas streams processors list --instance <workspace> --projectId <id>
atlas streams processors create --instance <workspace> --file processor.json --projectId <id>
atlas streams processors describe <name> --instance <workspace> --projectId <id>
atlas streams processors start <name> --instance <workspace> --projectId <id>
atlas streams processors stop <name> --instance <workspace> --projectId <id>
atlas streams processors delete <name> --instance <workspace> --projectId <id> --force
```

## Log Download

```bash
atlas streams download-logs --instance <workspace> --projectId <id> \
  --start <ISO8601> --end <ISO8601> --output logs.gz
```

## MCP Tool to CLI Mapping

| MCP Tool + Action | CLI Equivalent |
|-------------------|----------------|
| `atlas-streams-discover` → `list-workspaces` | `atlas streams instances list` |
| `atlas-streams-discover` → `inspect-workspace` | `atlas streams instances describe <name>` |
| `atlas-streams-discover` → `list-connections` | `atlas streams connections list --instance <ws>` |
| `atlas-streams-discover` → `list-processors` | `atlas streams processors list --instance <ws>` |
| `atlas-streams-discover` → `inspect-processor` | `atlas streams processors describe <name> --instance <ws>` |
| `atlas-streams-build` → connection | `atlas streams connections create --instance <ws> --file <json>` |
| `atlas-streams-build` → processor | `atlas streams processors create --instance <ws> --file <json>` |
| `atlas-streams-manage` → `start-processor` | `atlas streams processors start <name> --instance <ws>` |
| `atlas-streams-manage` → `stop-processor` | `atlas streams processors stop <name> --instance <ws>` |
| `atlas-streams-teardown` → processor | `atlas streams processors delete <name> --instance <ws> --force` |
| `atlas-streams-teardown` → workspace | `atlas streams instances delete <name> --force` |

## Limitations vs MCP Tools

The CLI provides raw CRUD but lacks several MCP tool features:
- **No elicitation** — you must provide all connection config upfront (passwords, bootstrap servers)
- **No `diagnose-processor`** — must manually combine `describe` + logs to assess health
- **No safety checks** — no automatic inspection before workspace deletion, no connection dependency warnings
- **No `search-knowledge`** — cannot validate pipeline field names against documentation
- **No auto-normalization** — must format `bootstrapServers` as comma-separated string yourself
