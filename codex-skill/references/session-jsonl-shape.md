# Session JSONL shape (observed, `claude` v2.1.201)

Path: `~/.claude/projects/<cwd with / replaced by ->/<full-session-uuid>.jsonl`

One JSON object per line. `type` values seen in a real background session transcript, in rough order of first appearance:

- `mode` — session-level metadata (`mode`, `sessionId`).
- `permission-mode` — recorded whenever the permission mode is set/changes.
- `file-history-snapshot` — snapshot bookkeeping for file edits (`isSnapshotUpdate`, `messageId`, `snapshot`).
- `user` — a real user/prompt turn. Has `cwd`, `gitBranch`, `message`, `parentUuid`, `timestamp`, `uuid`.
- `attachment` — attached context (system-reminders, etc.) associated with a `user` turn via `parentUuid`.
- `ai-title` — an auto-generated short title for the session (`aiTitle`).
- `assistant` — one assistant turn. **Split across one line per content block** (`thinking` / `text` / `tool_use`), all sharing the same `message.id` and `requestId`, all carrying the same full-turn `message.usage`. Fields: `message` (has `id`, `model`, `role`, `content`, `stop_reason`, `usage`), `requestId`, `uuid`, `parentUuid`, `timestamp`.
- `last-prompt` — bookkeeping for `/resume` picker (`lastPrompt`, `leafUuid`).

## `message.usage` fields (on `assistant` lines)

```json
{
  "input_tokens": 8,
  "cache_creation_input_tokens": 232,
  "cache_read_input_tokens": 25542,
  "output_tokens": 141,
  "server_tool_use": {"web_search_requests": 0, "web_fetch_requests": 0},
  "service_tier": "standard",
  "cache_creation": {"ephemeral_1h_input_tokens": 232, "ephemeral_5m_input_tokens": 0},
  "iterations": [...]
}
```

`message.model` (e.g. `"claude-haiku-4-5-20251001"`) tells you which model's pricing to apply — a single session can mix models if `--model`/`--agent` overrides happen mid-conversation.

## Dedup key

Use `message.id` (a `msg_...` string). Do **not** use the per-line `uuid` (unique per content block, not per turn) or `requestId` alone (fine in practice but `message.id` is the more direct match to "one billed API response").

## Why multiple lines per turn exist

Claude Code logs each content block of a single API response as its own JSONL line as it's produced (thinking block, then tool_use block, then eventually a text block for the final reply), rather than buffering the whole message. That's good for live-tailing (you see structure as it happens) but means naive `usage` summation triples counts on tool-heavy turns.
