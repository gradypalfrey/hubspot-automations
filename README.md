# hubspot-automations

A scheduled Claude Code Routine that processes customer meeting summaries from Fyxer, logs them to HubSpot deals, creates action items as tasks, and works on each task in its own subagent.

Runs on Anthropic's cloud infrastructure (Claude Code Routines). No always-on hardware. Three connectors: Fyxer, Microsoft 365, HubSpot.

## High-level flow

```
Routine fires (every 15 min during business hours)
  ├─ Orchestrator: check Outlook for meetings ended in last 20 min
  └─ For each ended meeting:
      └─ Use fyxer-to-hubspot skill
          ├─ Pull Fyxer summary
          ├─ Match to HubSpot deal
          ├─ Idempotency check
          ├─ Add Note on the deal
          └─ For each action item:
              ├─ Create HubSpot task ([Auto] prefix) via task-creator
              └─ Spawn task-worker subagent with the task ID
                  ├─ Classify (auto_doable / prep_able / coordination / misattributed)
                  ├─ Identify principal (Grady / Adib)
                  ├─ Do the work (or prep, or flag)
                  ├─ Update task body with audit trail
                  └─ Return summary to orchestrator
```

## Repo layout

```
hubspot-automations/
├── .claude/
│   ├── agents/
│   │   ├── follow-up-drafter.md
│   │   ├── task-creator.md
│   │   └── task-worker.md
│   └── skills/
│       └── fyxer-to-hubspot/
│           └── SKILL.md
└── README.md
```

## Prerequisites

Before launching the routine:

1. **Connectors authorized in claude.ai**
   - Microsoft 365 (write scopes for Outlook calendar / mail)
   - HubSpot (write scopes; disconnect/reconnect if originally authorized before April 13, 2026)
   - Fyxer (custom MCP at `https://app.fyxer.com/mcp`)
2. **HubSpot configuration**
   - Sensitive Data must NOT be enabled (it blocks engagement objects from MCP access).
   - Operating user has owner-level permissions on the deals the system will touch.
3. **GitHub repo** — push this directory to a private repo under Grady's account before creating the routine.

## Routine configuration

Create the routine at `claude.ai/code/routines`:

| Setting | Value |
|---|---|
| Name | `fyxer-to-hubspot-sync` |
| Repository | `hubspot-automations` |
| Connectors | Microsoft 365, HubSpot, Fyxer (remove all others) |
| Trigger | Schedule: every 10 minutes, weekdays, 7:30 AM–2:00 PM Pacific + one sweep at 4:00 PM Pacific |
| Model | Sonnet |
| Allow unrestricted branch pushes | No |

## Top-level routine prompt

Paste this as the routine's top-level instruction:

```
Check Microsoft 365 calendar for meetings that ended between 4 hours ago and 5 minutes ago (today only, Pacific time). The wide lookback is intentional — the fyxer-to-hubspot skill is idempotent (it checks for an existing Fyxer-meeting-ID Note on the matched deal before doing anything), so re-seeing the same meeting on later polls is safe and cheap.

If none, output "No meetings to process" and stop. This is success.

For each ended meeting:
1. Skip if: solo event, focus time, blocked time, or no external attendees (no non-dialai.ca attendees).
2. Use the fyxer-to-hubspot skill to process it.

The skill handles: pulling the Fyxer summary, matching the deal, idempotency check, logging the note, creating tasks, and dispatching task-worker subagents. If Fyxer doesn't have the summary yet for a given meeting, the skill returns cleanly and the next poll will retry — do not loop or wait.

After processing all meetings, output one line:
"Processed N meetings: X notes added, Y tasks created, Z waiting on Fyxer, W unmatched."

Do not ask for confirmation at any step. If a meeting can't be matched to a deal, log it and continue — don't stop.
```

## Communication policy

- **Internal teammates** (Marcus Dunn, Sepehr Ahmadipourshirazi, Harper Grieve, Luke Duncan, Jaden Howard, Adib Vahedi): no auto-send. The agent drafts the proposed message into the HubSpot task body and Grady sends it manually. Teams send is parked until later.
- **Grady**: HubSpot task updates only. No pings.
- **External recipients** (customers, partners): Outlook drafts only via `follow-up-drafter`. Never auto-sent.

## Audit and review

- **Daily** — Grady reviews HubSpot tasks filtered to `[Auto]` prefix + due today.
- **Weekly (first 2 weeks)** — Grady + Adib spend 15 min reviewing classification accuracy. Adjust the `task-worker` prompt if needed.
- **Monthly** — review which tasks were misclassified, which got stuck, which were never reviewed. Tune.

## Rollout

**Week 1 — Shadow mode.** Add a temporary rule to `task-worker`: "Instead of marking any task complete, append `[WOULD COMPLETE]` to the body and leave it open." Review everything daily.

**Week 2 — Live for internal hygiene + drafts for external.** Remove the shadow rule. CRM hygiene goes live. Outlook drafts continue saving (never auto-sent). Grady + Adib do a 15-min Friday review.

**Week 3+ — Steady state.** Weekly checkpoints reduce to monthly. Consider adding a nightly retry/cleanup routine if failure patterns emerge.

## Known limits

- Standard HubSpot objects only — no custom objects.
- Sensitive Data setting blocks engagements. Verify it's off before launch.
- Routines is in research preview. API surface may shift. Monitor the Anthropic changelog.
- No retry on task-worker failure. A failed run leaves a `[Auto]` task in HubSpot with no agent activity. Add a nightly cleanup routine after observing real failure modes for 1–2 weeks.
- Teams send is not wired. Internal coordination ("ask Sepehr for an ETA") gets drafted into the task body for Grady to send.

## Kill switch

Disable the routine from `claude.ai/code/routines`. One click. In-flight task-workers complete; no new runs start.

## Open items to verify on first run

1. Does the Fyxer MCP return structured action items, or only prose summaries? If prose-only, the skill's "extract action items" step does the parsing — verify the extractions match what you'd expect.
2. Confirm HubSpot owner records exist for everyone in the team list (so `task-creator` can resolve owner assignments).
3. First-run sanity: pick one real meeting, run the routine manually, check that the Note + tasks land on the right deal before turning on the schedule.
