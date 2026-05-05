This folder contains the source for a Skilled Agent originally built for the Valet runtime. Changes should follow the Skilled Agent open standard.

## Setup

### Connectors

- **atlassian-mcp**: The Atlassian MCP server. The agent uses it to read watched Confluence policy pages, diff page bodies against MEMORY.md, list Jira issues in the configured compliance project, walk each issue's changelog for status and assignee transitions, and read approval comments. Writes (edit a page, transition a ticket, add a comment) only happen when a Slack user explicitly asks. Authenticated via an `ATLASSIAN_API_KEY` secret. Add it from the catalog at the org level so other Atlassian-powered agents can share the same credential.

### Channels

- **slack** (slack): The agent's per-agent Slack bot. Listens for @mentions and replies in-thread, and posts the per-change pings to whichever channels the bot has been invited to. Slack writes use the auto-injected outbound Slack connector.
- **heartbeat** (heartbeat): Fires every 5 minutes. Declared inline in `valet.yaml`, so it's created automatically by the dashboard setup flow. The heartbeat job diffs watched Confluence policy pages against the MEMORY.md snapshot, walks the Jira changelog for the configured compliance project, and posts one message per real change set (de-dup via MEMORY.md).

### Secrets

- **ATLASSIAN_API_KEY**: Base64-encoded `email:token` for your Atlassian Cloud account. Generate the API token at [id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens), then base64-encode `your-email@example.com:your-token` (e.g. `printf 'you@co.com:abc123' | base64`). Scope: read access on the watched Confluence space and Jira project (and write access on those two only if you want @mention edits to work). Stored as a secret on the `atlassian-mcp` connector and injected into the MCP server connection at runtime.

### External Setup

1. Generate an Atlassian API token at [id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens). Base64-encode `your-email@example.com:your-token` and paste the result as `ATLASSIAN_API_KEY` during setup. The Atlassian account that owns the token must have read access on the Confluence space and Jira project you'll watch.
2. Set the agent env vars that scope the watch list:
   - `POLICY_SPACE_KEY` — the Confluence space key whose pages are watched as policies (e.g. `POL`). Pages anywhere in the workspace labeled `policy` are also included.
   - `POLICY_PAGE_IDS` — alternative or addition: comma-separated Confluence page IDs to watch explicitly (e.g. `123456,789012`). Use this when your policies live across spaces.
   - `JIRA_PROJECT_KEY` — the Jira project key whose issues are watched as compliance tickets (e.g. `COMP`).
   - At least one of `POLICY_SPACE_KEY`, `POLICY_PAGE_IDS`, or `JIRA_PROJECT_KEY` must be set. If multiple are set, all are watched.
3. After deploy, invite the agent's Slack bot to your compliance / GRC channel (or wherever you want change cards to land). The agent posts to every channel it's a member of — invite it to one focused channel, or several. If the bot has not been invited anywhere, change cards are sent as a DM to the workspace install user with a one-line nudge to invite it somewhere.
4. Invite the bot to any additional channels where teammates should be able to @mention it for ad-hoc compliance questions (e.g. *"any pending compliance tickets?"*, *"who approved the data retention policy update?"*).
5. The first heartbeat fires within 5 minutes of deploy. To smoke-test sooner, edit a watched policy page in Confluence (or transition a watched compliance ticket) — the next heartbeat will pick it up. You can also @mention the bot in Slack for an immediate read-only path.

## Customizing

- **Change the heartbeat interval**: edit `every` on the inline `heartbeat` channel in `valet.yaml` (e.g. `5m`, `30m`, `1h`), then redeploy. The default `5m` keeps cards close to real-time; widen it to lower Atlassian API volume.
- **Change what counts as a meaningful change**: edit SOUL.md → "Phase 2" (Confluence real-change threshold) and "Phase 3" (Jira event detection). The Confluence threshold filters cosmetic edits (under 20 chars, no heading or bullet structure changes). The Jira detector recognizes status transitions, assignee changes, and approval-style comments — broaden or tighten the approval phrase list there.
- **Change the watched scope**: edit the `POLICY_SPACE_KEY`, `POLICY_PAGE_IDS`, and `JIRA_PROJECT_KEY` env vars on the agent. The Confluence watch list also picks up any page workspace-wide labeled `policy`, so labelling a one-off page is enough — no redeploy needed.
- **Evidence-hint mapping**: SOUL Phase 2.8 / Phase 3.7 reads from this table to compose the *"Evidence to update"* line on each card. The default mapping (edit in SOUL.md if you want to extend):

  | When the page title or labels match… | Typical-evidence hint surfaced on the card |
  |---|---|
  | `access control`, `iam`, `identity` | "Update SOC 2 CC6 control language; refresh the screenshot in the audit folder." |
  | `data retention`, `data deletion` | "Refresh the data-retention matrix; re-attest the evidence in your latest SOC 2 / ISO folder." |
  | `incident response`, `ir plan` | "Update the IR runbook attachment; confirm the on-call rotation list is current." |
  | `vendor`, `third party`, `subprocessor` | "Refresh the vendor inventory and the latest signed DPA for the named vendor." |
  | `change management`, `sdlc` | "Confirm the change-approval ticket is linked and the deployment evidence is attached." |
  | (no match) | "Update any linked control language and refresh the related evidence file." |

  And the Jira ticket-field hint by transition:

  | Transition | Typical ticket-field hint surfaced on the card |
  |---|---|
  | `* → Approved` | "Set Approved-By, Approval Date, and attach the signed approval. Close any linked evidence sub-tasks." |
  | `* → Rejected` | "Capture the rejection reason in the comment, link the follow-up ticket, and notify the requesting control owner." |
  | `Approved → Closed` / `* → Done` | "Confirm the linked control owner attestation and the evidence file timestamp." |
  | (any other) | "Confirm any linked evidence sub-tasks and update the related control owner." |

- **Control where change cards post**: invite or remove the bot from channels in Slack — that's the only signal the agent uses. There is no channel name in the configuration.
