# Worker onboarding prompt

Paste this **into the chat that should become a worker**. The task has to be created from inside that chat so it runs there and keeps the chat's context.

Replace the `{{PLACEHOLDERS}}` before pasting. `{{WORKER_MANDATE}}` is the standing job this worker does on every run. It can also point to an instruction document the worker already has.

---

```text
Connect this chat to the task dispatcher as a worker for "{{WORKER_NAME}}".
Do the setup now, by yourself, end to end.

How it works: a central dispatcher task wakes workers on its own schedule
({{ROUND_SCHEDULE}}, {{TIMEZONE}}). This chat gets exactly ONE permanent task.
The task stays paused between runs and is never replaced.

Settings:
- Registry: {{REGISTRY}}  (one row per worker)
- Dispatcher state file: {{STATE_FILE}}
- Gate block template: {{GATE_TEMPLATE_LOCATION}}

Steps:

1. Find your task-management tools (create / read / update tasks) and any
   tools needed to read and write the registry and the state file. Check
   read access to all of them BEFORE creating anything.

2. Check whether a permanent task already exists that clearly belongs to
   this chat and this worker. If it does, reuse it. If not, create one
   called "{{TASK_TITLE}}", attached to this chat:
   - initial prompt (safe placeholder): "Setup not finished. Do no work.
     Pause this task and end the run."
   - schedule: yearly, first run tomorrow at 23:59 {{TIMEZONE}} (work out
     the date from the actual current time). This is only a placeholder.
     The dispatcher controls real runs.

3. Right after creating it, pause it (disable it). Confirm the pause by
   reading the task back. Don't change any other task.

4. Take the real task ID and chat ID from the tool output. Never invent
   or guess IDs.

5. Build the full prompt: the gate block from the template with {{TASK_ID}},
   {{CHAT_ID}} and the other placeholders filled in, followed by MANDATE
   and this worker's standing job:
   {{WORKER_MANDATE}}
   Save it into the task, keeping the task paused. Read it back to verify.

6. In the registry, find the single row for "{{WORKER_NAME}}". Create one
   only if it truly doesn't exist, and never create a duplicate. Fill in:
   task ID, chat ID, chat link, schedule label. Leave the "enabled"
   checkbox as it is. Never turn it on yourself. Don't invent run IDs,
   run dates or round keys.

7. Re-read the registry row and the task. Check that the IDs are correct,
   that the binding is unique, that the gate block is complete, and that
   the task is actually paused.

8. Don't edit the dispatcher's state file or schedule, and don't start any
   other task. The dispatcher accepts new workers on its next round.

9. Report in this chat: the result, the task ID, the chat link, and the
   status "registered, waiting for dispatcher". Call it "connected" only
   after your task ID appears in the state file's enrolled_task_ids.
   If a tool stops you from finishing, name the exact blocker and what a
   human needs to do. Don't report the setup as complete.

Don't ask me to approve these steps again. Just do them.
```
