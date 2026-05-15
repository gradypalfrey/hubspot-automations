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
│   ├── memory/
│   │   └── proposed-learnings.md   # agent-appended, human-reviewed weekly
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
| Trigger | Schedule: hourly at :00, weekdays, 8 AM–6 PM Pacific (Routines is capped at hourly firings as of May 2026) |
| Model | Sonnet |
| Allow unrestricted branch pushes | No |

## Top-level routine prompt

Paste this as the routine's top-level instruction:

```
Check Microsoft 365 calendar for meetings that ended between 4 hours ago and 5 minutes ago (today only, Pacific time). The wide lookback is intentional — the fyxer-to-hubspot skill is idempotent (it checks for an existing Fyxer-meeting-ID Note on the matched deal before doing anything), so re-seeing the same meeting on later polls is safe and cheap.

If none, output "No meetings to process" and stop. This is success — do not send an email, do not write to the repo.

For each ended meeting:
1. Skip if: solo event, focus time, blocked time, or no external attendees (no non-dialai.ca attendees).
2. Use the fyxer-to-hubspot skill to process it.

The skill handles: pulling the Fyxer summary, matching the deal, idempotency check, logging the note, creating tasks, and dispatching task-worker subagents. If Fyxer doesn't have the summary yet for a given meeting, the skill returns cleanly and the next poll will retry — do not loop or wait.

Collect every task-worker's structured return value (task ID, classification, principal, summary, HubSpot URL, and optional proposed learning).

## End-of-run actions (only if at least one task was actually processed)

1. **Send a summary email** via Microsoft 365 from grady@dialai.ca to grady@dialai.ca. Subject: "[grace] <N> tasks processed across <M> meeting(s)". Body format:

   ```
   Meeting: <title> (<deal name>)
     - <task summary> [classification, principal] — <HubSpot URL>
     - <task summary> [classification, principal] — <HubSpot URL>

   Meeting: <title> (<deal name>)
     - ...

   To redirect any task, edit its HubSpot body and add one of:
     STOP                     (agent leaves it alone)
     RETRY: <new context>     (agent re-runs with your context)
     REDIRECT: <classification>  (agent overrides its classification)
   ```

   Skip the email entirely if zero tasks were processed this run.

2. **Append any proposed learnings** to `.claude/memory/proposed-learnings.md` (one line each, ISO timestamp prefix). If any were appended, commit and push to the repo:

   ```
   git add .claude/memory/proposed-learnings.md
   git commit -m "learning: <one-line summary or count if multiple>"
   git push origin main
   ```

   If no learnings were proposed, do not touch the repo.

3. **Output the run summary line** to the routine log:
   "Processed N meetings: X notes added, Y tasks created, Z waiting on Fyxer, W unmatched, L learnings proposed."

Do not ask for confirmation at any step. If a meeting can't be matched to a deal, log it and continue — don't stop.
```

## Feedback loop — how Grady redirects the agent

The "click to continue this subagent's thread" UX isn't directly available in Routines (subagent runs aren't exposed as resumable conversations). The practical equivalent runs through HubSpot task bodies and the end-of-run email:

1. Each productive run sends one email to grady@dialai.ca with every task worked on, each linked to its HubSpot URL.
2. To redirect any task, open it in HubSpot and add one of these markers to the body:
   - `STOP` (on its own line) — agent never touches the task again
   - `RETRY: <new context>` — agent re-runs the task with your additional context folded in
   - `REDIRECT: <classification>` — agent overrides its own classification with the one you specify (e.g. `REDIRECT: prep_able`)
3. The next routine firing (≤ 1 hour later) picks up the marker and acts on it.

If you just want the agent to back off without specific instructions, any free-form edit to the task body that doesn't include a marker triggers the "human touched it, stand down" rule — same effect as `STOP` but you also leave a note for yourself.

## Learning loop — how the agent suggests prompt updates

`.claude/memory/proposed-learnings.md` is an advisory file the `task-worker` reads at the start of every invocation. The agent appends proposed learnings to it only when it spots a genuinely novel pattern (the prompt rules describe what qualifies). The orchestrator commits and pushes any new entries at the end of the run.

Weekly review:
1. Read the file.
2. For any entry that captures a real rule, edit `.claude/agents/task-worker.md` to canonize it.
3. Delete every entry from `proposed-learnings.md` once reviewed (good ones promoted, bad ones discarded).
4. Commit and push.

Expected volume: 0–2 entries per week once steady state is reached. If you see more than ~5 entries in a week, the prompt is rotting and needs a structural rewrite, not more rules.

## Communication policy

- **Internal teammates** (Marcus Dunn, Sepehr Ahmadipourshirazi, Harper Grieve, Luke Duncan, Jaden Howard, Adib Vahedi): no auto-send. The agent drafts the proposed message into the HubSpot task body and Grady sends it manually. Teams send is parked until later.
- **Grady**: HubSpot task updates only. No pings.
- **External recipients** (customers, partners): Outlook drafts only via `follow-up-drafter`. Never auto-sent.

## Audit and review

- **Daily** — Grady reviews HubSpot tasks filtered to `[Auto]` prefix + due today.
- **Weekly (first 2 weeks)** — Grady + Adib spend 15 min reviewing classification accuracy. Adjust the `task-worker` prompt if needed.
- **Monthly** — review which tasks were misclassified, which got stuck, which were never reviewed. Tune.

## Rollout

**Latency expectation.** With hourly polling, a follow-up draft lands in Outlook on average ~30 min after the call ends, worst case ~60 min. Fyxer needs ~10–20 min to produce a summary, so a call ending at 9:05 typically gets caught by the 10:00 poll. The 4-hour lookback means a missed poll has 3 more chances to catch up.

**Week 1 — Shadow mode.** Add a temporary rule to `task-worker`: "Instead of marking any task complete, append `[WOULD COMPLETE]` to the body and leave it open." Review everything daily.

**Week 2 — Live for internal hygiene + drafts for external.** Remove the shadow rule. CRM hygiene goes live. Outlook drafts continue saving (never auto-sent). Grady + Adib do a 15-min Friday review.

**Week 3+ — Steady state.** Weekly checkpoints reduce to monthly. Consider adding a nightly retry/cleanup routine if failure patterns emerge.

## Known limits

- Standard HubSpot objects only — no custom objects.
- Sensitive Data setting blocks engagements. Verify it's off before launch.
- Routines is in research preview. API surface may shift. Monitor the Anthropic changelog.
- No retry on task-worker failure. A failed run leaves a `[Auto]` task in HubSpot with no agent activity. Add a nightly cleanup routine after observing real failure modes for 1–2 weeks.
- Teams send is not wired. Internal coordination ("ask Sepehr for an ETA") gets drafted into the task body for Grady to send.

## Testing

Run these in order before turning on the schedule.

**1. Connector smoke test.** Open a regular claude.ai chat with HubSpot, Microsoft 365, and Fyxer connectors all enabled. Try:

- *Fyxer:* "List my last 3 meetings and return their summaries. If the API supports structured action items, return those separately." — Note whether action items come back structured or only as prose. This determines whether the skill's "extract action items" step needs to do parsing.
- *HubSpot:* "Find the open deal for [recent customer name]. Then show me the existing notes on that deal." — Confirms read access and deal-matching basics.
- *HubSpot write:* "Create a task on deal [ID] titled '[Auto] connector test' due tomorrow, then delete it." — Confirms write scopes are live (this is the step that fails if Sensitive Data is enabled).
- *Microsoft 365:* "Show me my calendar events from yesterday with their attendees." — Confirms calendar read.

If any of these fail, fix the connector before going further.

**2. Manual end-to-end on one real meeting.** Pick a customer meeting from earlier today. In claude.ai chat (with all three connectors), paste:

```
Use the fyxer-to-hubspot skill to process the meeting titled "<meeting title>" that ended at <time>.
```

The chat will execute the skill against the real services. Check:
- Did the right deal get matched?
- Did a Note land on it with the correct format (Fyxer meeting ID on line 1)?
- Did tasks get created with `[Auto]` prefix?
- Did each task body get classified and updated by `task-worker`?

If anything looks off, you're tuning the prompts in the repo, not the routine itself.

**3. Idempotency check.** Run the same prompt from step 2 a second time. The skill should detect the existing Note and skip. If it creates a duplicate Note, the idempotency check needs work.

**4. Routine first run.** In `claude.ai/code/routines`, click "Run now" on the routine (or wait for the next scheduled hour). Watch the run log — you should see the orchestrator output land on the summary line. Check HubSpot for new Notes/tasks against meetings from today.

**5. Shadow mode for week 1.** See the rollout section above. Don't let the agent close tasks until you've watched a week of behavior.

## Kill switch

Disable the routine from `claude.ai/code/routines`. One click. In-flight task-workers complete; no new runs start.

## Open items to verify on first run

1. Does the Fyxer MCP return structured action items, or only prose summaries? If prose-only, the skill's "extract action items" step does the parsing — verify the extractions match what you'd expect.
2. Confirm HubSpot owner records exist for everyone in the team list (so `task-creator` can resolve owner assignments).
3. First-run sanity: pick one real meeting, run the routine manually, check that the Note + tasks land on the right deal before turning on the schedule.
