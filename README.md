# Atlassian Policy Tracking

Every day, reviews policy edits and compliance ticket transitions — what changed, who approved it, what evidence to update.

## Prerequisites
- An [Atlassian Cloud](https://www.atlassian.com/cloud) account with an API token and read/write access to the Confluence space and Jira project you want to watch
- A Slack workspace where you can install the agent's bot and invite it to one or more channels (typically a compliance / GRC channel)

<table>
  <tr>
    <td><strong>CHANNELS</strong></td>
    <td><code>slack</code> · <code>heartbeat</code> — daily</td>
  </tr>
  <tr>
    <td><strong>CONNECTORS</strong></td>
    <td><code>atlassian-mcp</code></td>
  </tr>
  <tr>
    <td colspan="2" align="center">
      <br />
      <a href="https://valet.dev/deploy?from=github.com/valet-agents/atlassian-policy-tracking">
        <img src="https://raw.githubusercontent.com/valet-agents/atlassian-policy-tracking/main/.github/deploy-button.svg" alt="Deploy Agent →" height="40" />
      </a>
      <br /><br />
    </td>
  </tr>
</table>
