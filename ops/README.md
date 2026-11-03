# ops

Deployment-side files that the checkout itself needs but that no script reads. Everything here is
optional to run dsh-superagent; it exists so a machine can be brought back after a reinstall.

## `watchdog-prompt.txt`

The prompt for the scheduled check described in
[006 Deployment](../docs/006-deployment.md#scheduled-check). It lives here because the schedule
itself lives in the harness's own storage (`~/.dsh/storages/schedule.json`), which is not part of
any checkout: reinstalling the harness or moving to another machine drops the schedule, and with it
the prompt, unless a copy is kept under version control.

The check has two steps. The second one is the reason this file exists:

1. `scripts/dsh-superagent --profile superagent status` — component health.
2. `scripts/dsh-superagent --profile superagent tickets` — ticket health.

Step 2 is easy to leave out, and leaving it out produces a monitor that looks healthy while work
is stuck. On 2026-11-03 that happened here: every one of the 13 parts reported `ready` and `status`
exited 0, but a Feishu request had become a Ticket that sat on `action: failed`
(`LlmError: Request timed out.`) for hours. A check that read only component health said nothing,
because a failed Turn is not a failed component.

## Restoring the check

The schedule must be created in the harness profile you normally use, not in `superagent`, so it
keeps running while the agent service is down.

1. Read `watchdog-prompt.txt` and replace `<checkout>` with this checkout's path.
2. Ask an agent in that profile to create a schedule with it. Once a day is enough when you also
   read the source chats yourself; the check is there to catch what you did not notice.
3. Confirm `scripts/dsh-superagent --profile superagent tickets` exits 0 on a quiet day and 4 when
   a Ticket needs a person, so the check has something to branch on.

The exit codes the check relies on:

| Command | `0` | `1` | `2` | `3` | `4` |
| --- | --- | --- | --- | --- | --- |
| `status` | healthy | not running, duplicated, or unreachable | a part reports an error | setup waits for input | — |
| `tickets` | nothing needs attention | the API did not answer | — | — | a Ticket needs a person |

`tickets` answers before the control lock is taken, so a check running while the service restarts
is not blocked by a lock it has no stake in.
