# Registry and state file

## Registry (one row per worker)

A Google Sheet works well because humans get a real checkbox and agents can read and write narrow ranges. A CSV or Markdown table in a shared drive works too.

| Column | Owner | Example | Notes |
|---|---|---|---|
| `enabled` | **Human** | ☑ | Checkbox. Agents never turn it on. |
| `name` | Human / worker | `Repo: acme/api` | Unique, human-readable. |
| `chat_link` | Worker | `https://chatgpt.com/c/<chat_id>` | Lets a human jump straight to the worker's chat. |
| `schedule_label` | Worker | `Weekdays: AM + PM` | Informational. |
| `connection_status` | Formula | `Enabled` / `Missing IDs` | Optional formula, e.g. "Enabled" when the box is ticked and both IDs are present. |
| `notes` | Human | | Free text. Agents don't edit it. |
| `worker_key` | Worker | `acme-api` | Optional stable key. |
| `task_id` | Worker | | Copied from tool output only. |
| `chat_id` | Worker | | Copied from tool output only. |
| `run_status` | Dispatcher / worker | `idle` → `requested` → `started` → `done` / `blocked` / `not started` | |
| `round_key` | Dispatcher / worker | `2026-10-07-AM` | The round the status refers to. |
| `last_started_at` | Worker | ISO timestamp | |
| `last_finished_at` | Worker | ISO timestamp | |
| `last_note` | Worker | `blocked: API token expired` | One line. |

## Dispatcher state file

A small JSON text file the dispatcher owns:

```json
{
  "version": 2,
  "current_round_key": "2026-10-07-AM",
  "round_status": "complete",
  "enrolled_task_ids": ["<task_id_1>", "<task_id_2>"],
  "last_round": {
    "key": "2026-10-07-AM",
    "started": 11,
    "done": 10,
    "blocked": 1,
    "not_started": ["<task_id_7>"],
    "closed_at": "2026-10-07T06:41:00-07:00"
  }
}
```

Only the dispatcher writes this file. Workers only read `current_round_key` and `enrolled_task_ids`.
