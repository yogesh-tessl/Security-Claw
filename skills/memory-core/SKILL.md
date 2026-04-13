---
name: memory-core
description: "Search and retrieve agent memory via `memory_search` and `memory_get` tools. Use when: (1) recalling facts, notes, or context saved in a previous session, (2) persisting important information for future sessions, (3) building agents with long-term memory across conversations. Requires memory-core plugin enabled in openclaw.json."
metadata: { "openclaw": { "emoji": "🧠" } }
---

# Memory Core — Long-Term Agent Memory

Search and retrieve facts stored across sessions using the memory plugin.

## Enable Memory

```json5
{
  plugins: {
    entries: {
      "memory-core": { enabled: true },
    },
  },
}
```

## Tools

| Tool            | Description                          |
| --------------- | ------------------------------------ |
| `memory_search` | Semantic search over stored memories |
| `memory_get`    | Retrieve a specific memory by ID     |

## Workflow

1. **Search** — query stored memories with `memory_search`
2. **Retrieve** — fetch full details of a relevant memory via `memory_get` using the returned ID
3. **Apply** — use the recalled context in the current task

## Common Patterns

### Search Memory

```json
{
  "tool": "memory_search",
  "query": "target scope for Acme engagement"
}
```

```json
{
  "tool": "memory_search",
  "query": "credentials or API keys found during pentest",
  "limit": 5
}
```

**Example response:**

```json
[
  { "id": "mem_abc123", "content": "Target scope: 192.168.10.0/24", "score": 0.92 },
  { "id": "mem_def456", "content": "Found SQLi on /api/v1/search", "score": 0.85 }
]
```

If no results are returned, broaden the query terms or check that memories were stored in a prior session.

### Get a Specific Memory

```json
{
  "tool": "memory_get",
  "id": "mem_abc123"
}
```

### What Gets Stored

Memory is typically written by the agent during `/new` or `/reset` (via the `session-memory` hook), or explicitly when you ask:

```
Remember that the target scope for Project Alpha is 192.168.10.0/24
Save the finding: SQL injection on /api/v1/search?q= parameter
```

## Config

```json5
{
  plugins: {
    entries: {
      "memory-core": {
        enabled: true,
        config: {
          maxMemories: 1000,
          embeddingModel: "text-embedding-3-small",
        },
      },
    },
  },
}
```

