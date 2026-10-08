# ChatGPT Tasks Dispatcher

**Run more scheduled ChatGPT agents than your plan's active-task limit allows: one active dispatcher task wakes many paused worker tasks.**

ChatGPT caps how many scheduled tasks can be *active* at once (the cap depends on your plan; on some plans it is 10). If you want one long-running agent per project, client, repo, inbox or topic, each with its own chat and its own context, you hit that cap quickly.

This repo describes a pattern that works around it, with copy-paste prompt templates:

- **One dispatcher task** is the only task that stays active and on a real schedule.
- **Every worker task stays paused** between runs, so it does not occupy an active slot.
- **A shared registry** (a spreadsheet or a text file) lists the workers and whether each one is allowed to run.
- On each round, the dispatcher **wakes the enabled workers**. Each worker checks that the wake-up is legitimate, **pauses itself again**, does its work in its own chat, and records the result.

The result: the number of workers is limited by your round length and usage limits, not by the active-task cap.

> **Status.** This is a field-tested pattern, not an official feature. It relies on how ChatGPT Tasks behave today, which can change at any time. Read [Caveats](#caveats) before relying on it for anything important.

---

## Why it works

Three observations make the pattern possible:

1. **Paused tasks don't take an active slot.** OpenAI's help article tells you to pause or delete a task to make room when you're at the limit.
2. **A task can be created and paused right away.** A new worker creates its own task in its own chat and then pauses it within the same run. Its slot is freed straight away, so onboarding new workers never blocks.
3. **A task keeps its chat.** A worker task always runs in the chat where it was created, so it keeps the full context of that chat (history, files, decisions). The dispatcher does not need to know *how* to do the work, only *when*.

---

## Architecture

```
                 ┌──────────────────────────┐
                 │  Dispatcher task (ACTIVE)│   real schedule, e.g. weekdays 06:00 and 13:00
                 └────────────┬─────────────┘
                              │ 1. read registry + state
                              ▼
   ┌───────────────────────────────────────────────────────┐
   │ Registry (sheet / file)                                │
   │ enabled? │ name │ task_id │ chat_id │ status │ last run │
   └───────────────────────────────────────────────────────┘
                              │ 2. wake enabled workers (in batches that fit free slots)
           ┌──────────────────┼──────────────────┐
           ▼                  ▼                  ▼
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │ Worker A     │   │ Worker B     │   │ Worker N     │   each in its OWN chat,
   │ (PAUSED)     │   │ (PAUSED)     │   │ (PAUSED)     │   paused between runs
   └──────────────┘   └──────────────┘   └──────────────┘
     3. gate check → pause self → do the work → write result → short summary in its chat
```

### Components

| Component | What it is | Active? |
|---|---|---|
| **Dispatcher task** | One task with a real recurring schedule. It reads the registry and wakes workers. It never does the workers' jobs. | Always |
| **Worker task** | One permanent task per workstream, created **inside that workstream's chat**. Its formal schedule is a placeholder. | Only briefly, during its run |
| **Registry** | A sheet or file with one row per worker: an on/off checkbox, task ID, chat ID, chat link, status and last-run info. Humans control the checkbox. | — |
| **Dispatcher state** | A small file holding the list of accepted (`enrolled`) task IDs and the current round key. | — |
| **Worker gate block** | A fixed block at the top of every worker prompt. It decides whether the run is legitimate before any real work happens. | — |

### Slot budget

With an active-task cap of `L`:

- 1 slot: the dispatcher.
- `R` reserved slots (we use **2**): kept free so new workers can be onboarded at any time (create → pause).
- `L − 1 − R` slots: the batch size the dispatcher can wake at once.

With `L = 10`, `R = 2`, the dispatcher wakes up to 7 workers at a time and works through the rest in further batches during the same round.

---

## The lifecycle

### 1. Onboarding a worker (once per workstream)

Run this **inside the worker's own chat** (template: [`templates/worker-onboarding-prompt.md`](templates/worker-onboarding-prompt.md)):

1. Check read access to the registry and to any tools the worker needs.
2. If this chat already has a permanent task, reuse it. Otherwise create one.
3. Create it with a **safe initial prompt** ("setup not finished, do no work, pause this task and stop") and a **placeholder schedule**, for example yearly, first fire tomorrow at 23:59. Real runs come from the dispatcher.
4. **Immediately pause it**, and confirm the pause by re-reading the task.
5. Take the real task ID and chat ID from the tool output. Never type or guess them.
6. Write the full prompt into the task: the **gate block** (with its IDs filled in) first, then the worker's actual mandate. Keep it paused.
7. Find (or create) the worker's single row in the registry and fill in the task ID, chat ID and chat link. Don't touch the human-controlled checkbox.
8. Re-read the task and the registry to verify everything.
9. Report back as *"registered, waiting for dispatcher"*. Only call it **connected** after the dispatcher has accepted the task ID into its state file on its next round.

### 2. A dispatch round

The dispatcher prompt ([`templates/dispatcher-prompt.md`](templates/dispatcher-prompt.md)):

1. Generate a **round key**, e.g. `2026-10-07-AM`. Store it in the state file.
2. Read the registry. Accept new valid registrations into `enrolled_task_ids`.
3. For each enrolled worker whose checkbox is on, and that hasn't already completed this round:
   - mark the row as *requested* for this round key;
   - **wake the worker task**: re-enable it with a one-off fire time a few minutes ahead (the exact call depends on the task tool your account exposes).
4. Respect the batch size: don't have more workers enabled at once than the free slots allow. Wait for workers to pause themselves before waking the next batch.
5. At the end of the round, record which workers confirmed start and completion, and flag any that didn't.

### 3. A worker run

The gate block ([`templates/worker-gate-block.md`](templates/worker-gate-block.md)) runs first, every time:

1. **Is this my run?** Re-read the registry row: the checkbox is on, the task ID and chat ID match, and there's an open request for the current round key. If anything doesn't match, pause and stop without doing any work.
2. **Not a duplicate?** If this round key is already marked *started* or *done*, pause and stop.
3. **Record the start** (`started` + round key + timestamp).
4. **Pause itself**, freeing the slot. The run continues even after the task is paused.
5. **Do the actual work**, following the mandate below the gate block.
6. **Save the results and the next step** where the mandate says to. **Never create a new task** and never set its own schedule.
7. **Record the completion** (`done`, or `blocked` with the reason).
8. **Post a 3–5 line summary in its own chat**: which round, what was checked, what was done, what comes next. Post it even when nothing happened, so a human can see at a glance that the automation is running.

---

## Repo contents

| File | Purpose |
|---|---|
| [`templates/worker-gate-block.md`](templates/worker-gate-block.md) | The control block placed at the top of every worker task prompt. |
| [`templates/worker-onboarding-prompt.md`](templates/worker-onboarding-prompt.md) | Paste into a worker's chat to create, pause, wire up and register its task. |
| [`templates/dispatcher-prompt.md`](templates/dispatcher-prompt.md) | Prompt for the single active dispatcher task. |
| [`templates/registry-columns.md`](templates/registry-columns.md) | Suggested registry layout (sheet columns) and state-file format. |

All templates use `{{PLACEHOLDERS}}` for the values you fill in.

---

## Design rules that matter

- **One task per chat, forever.** Workers reuse their permanent task. Creating a fresh task per run is how you hit the cap again.
- **Pause first, then work.** A worker frees its slot as soon as it has confirmed the run, not when the work is finished.
- **Humans own the on/off switch.** Agents never turn their own checkbox on.
- **IDs come from tool output only.** A made-up task or chat ID will silently break routing.
- **Idempotency by round key.** Every run is tied to one round key, and a repeated or late wake-up is a no-op.
- **Honest reporting.** "Registered" is not "connected", "started" is not "done", and a draft is not a sent message.
- **The instruction lives with the worker.** Put the gate block in the task prompt itself. A separate shared instruction file only helps if every worker is told to read it.

---

## Caveats

- **Unofficial and fragile.** This depends on current ChatGPT Tasks behavior, on tool names that aren't a public API, and on what your account exposes. Any of it can change without notice.
- **Limits differ by plan.** The active-task cap, exact-time schedules, minimum intervals and usage limits all vary by plan. Check yours first.
- **It doesn't add compute.** Every worker run still counts against your normal usage limits. Many workers × two rounds a day adds up.
- **Rounds take time.** Batching means the last worker in a large round can start well after the round begins.
- **Approvals pause tasks.** If a worker hits an action that needs your approval, the task stops until you review it.
- **Test end to end before trusting it.** Onboard one test worker and watch it through a full round, from creation to accepted to woken to done, before rolling the pattern out.
- **Data.** The registry and state files will contain task and chat IDs. Keep them private, and never put secrets in prompts.
- **Respect the Terms of Use.** This pattern only changes *when* your own tasks run. It doesn't bypass usage limits or access controls, so keep it that way.

## Desktop vs. cloud

If you run agents from a local desktop setup with your own scheduler (cron, Task Scheduler, launchd), you don't need any of this. The pattern is for the **hosted ChatGPT Tasks** environment, where the cap is enforced server-side.

## License

MIT. See [LICENSE](LICENSE).
