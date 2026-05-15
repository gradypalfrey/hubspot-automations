---
name: fyxer-to-hubspot
description: Pulls the Fyxer summary for a specific recently-ended meeting and logs it to the matching HubSpot deal, then creates tasks for each action item and dispatches each task to the task-worker subagent.
---

You were triggered to process one specific meeting. The orchestrator passes you the meeting title, end time, and attendee list.

## Step 1: Pull the Fyxer summary

Use the Fyxer MCP to retrieve the summary for the meeting that just ended. Identify it by title and end time.

If Fyxer doesn't have the summary yet, log `"Fyxer summary not ready: <meeting title>"` and return cleanly. Do not retry inside this run — the next routine poll will pick the meeting up again (its idempotency check will find no Note yet and try again).

## Step 2: Match the meeting to a HubSpot deal

- Look up each external attendee (non-`dialai.ca` email) in HubSpot contacts.
- For each matched contact, find their primary open deal.
- If multiple open deals match across attendees, pick the most recently updated.
- If no match, log `"unmatched: <meeting title>"` and stop.

## Step 3: Idempotency check

Read the deal's existing notes. If any note's first line contains `Fyxer meeting ID: <id>` matching this meeting, stop — this run already processed it.

## Step 4: Add the summary as a Note on the deal

Format:

```
Fyxer meeting ID: <id>
Date: <meeting end time, ISO>

<full Fyxer summary>
```

## Step 5: Extract action items

If the Fyxer MCP returns structured action items, use them directly. If it returns only prose, extract action items from the summary text — look for explicit commitments ("X to do Y", "send Z by Friday", etc.). Skip vague statements without a clear deliverable.

## Step 6: Create and dispatch each task

For each action item:

1. Invoke the `task-creator` subagent with the action item text, the deal ID, and the meeting metadata. It returns a HubSpot task ID.
2. Invoke the `task-worker` subagent with the task ID. Run task-workers in parallel — do not wait for one to finish before dispatching the next.

## Step 7: Return summary

Return a single line:

```
Deal <name>: note added, <N> tasks created, <N> action items dispatched.
```
