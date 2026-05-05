# Heartbeat — Watch Confluence Policies and Jira Compliance Tickets

The heartbeat fires every 10 minutes. There is no payload to
parse — your job is to detect real changes in the watched
Atlassian artifacts and post one Slack message per change set.

## What this heartbeat does

1. Read the de-dup state from `MEMORY.md` (see shape below).
2. Resolve the watch list (SOUL **Phase 1**):
   - **Confluence policy pages**: every page in the
     `POLICY_SPACE_KEY` space, plus any page workspace-wide
     labeled `policy`.
   - **Jira compliance tickets**: every issue in the
     `COMPLIANCE_JIRA_PROJECT` project updated within the last
     ~30 minutes.
3. **Confluence pass** (SOUL **Phase 2**): for each watched
   policy page, fetch metadata + body, compare `version.number`
   to the snapshot, diff the body line-by-line, apply the
   real-change threshold (skip pure cosmetic edits but still
   refresh the snapshot), capture the evidence pointer **only
   if it appears verbatim in the change**, and queue a Slack
   post per real edit.
4. **Jira pass** (SOUL **Phase 3**): for each watched compliance
   ticket, walk the changelog and new comments since the
   snapshot. Detect status transitions, assignee changes, and
   approval-style comments (`Approved`, `LGTM`, `Signed off`,
   `approved by:`). Capture the evidence pointer **only if it
   appears verbatim in a new comment**. Queue a Slack post per
   change set, with all events for that ticket in chronological
   order in one message.
5. **Post** (SOUL **Phase 4**): resolve target channels per SOUL
   **Where to post** — list every channel the bot is a member
   of and post once per change set per channel. If the bot is
   in zero channels, DM the workspace install user with the
   change post and a one-line invite hint.
6. **Update MEMORY.md** (SOUL **Phase 5**) for every page and
   ticket checked, posted or not. This is the contract that
   prevents duplicate posts.
7. Do not retry on failure — log the error in your session and
   continue with the remaining destinations. The next heartbeat
   is the recovery.

## MEMORY.md state shape

One block per watched Confluence page:

```
## confluence:<page_id>
- version: <int>
- last_edited_at: <iso>
- last_edited_by: <accountId>
- last_posted_at: <iso or "never">
- body:
  <line 1 of plain-text body>
  <line 2 …>
```

One block per watched Jira ticket:

```
## jira:<issue_key>
- status: <name>
- assignee: <accountId or "unassigned">
- last_seen_updated: <iso>
- last_posted_at: <iso or "never">
- last_comment_id: <id or "none">
```

The snapshot blocks are the de-dup keys. The Confluence body
lines drive the line-level diff; the Jira `last_seen_updated`
and `last_comment_id` drive the changelog and comment scans.

## Where to post

Per SOUL "Where to post":

- Post to every Slack channel the bot is a member of —
  typically the compliance / GRC channel the user invited it
  to.
- If the bot has been invited to zero channels, DM the
  workspace install user with the change post and a one-line
  nudge: *"I haven't been invited to a channel yet — invite me
  to your compliance / GRC channel and I'll post change pings
  there."*
- Never hard-code a channel name.

## Skip conditions

Stop silently (no post) if any of these are true:

- No watched Confluence pages have a version bump and no
  watched Jira tickets have any detected events. The heartbeat
  fires every 10 minutes — quiet runs are normal and expected.
- A watched Confluence page changed but the diff is below the
  real-change threshold (Phase 2). Refresh the snapshot anyway.
- Atlassian is unreachable or returns an error. Log and wait
  for the next heartbeat.

## Hard rules

- Never post the same edit or the same transition twice. The
  MEMORY.md snapshot update is the only gate; respect it.
- Never invent a "what evidence to update" pointer. If the
  change itself doesn't name the evidence, omit the section.
- Your turn ends after the posts and the MEMORY.md update. No
  follow-ups, no thread replies after the initial post.
