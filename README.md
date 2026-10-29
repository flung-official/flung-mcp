<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://assets.flung.dev/brand/logo/flung-lockup-on-dark.svg">
    <img alt="Flung" src="https://assets.flung.dev/brand/logo/flung-lockup-on-light.svg" width="260">
  </picture>
</p>

# Flung MCP server

Let an AI agent post to social media for you. Through Flung's MCP (Model Context Protocol) server an agent sees the accounts in your workspace, starts a connect when one needs a person, schedules and retries posts, reads comments and replies to them, and follows RSS feeds. It is hosted: there is nothing to install or run.

## TL;DR

| Question | Answer |
|---|---|
| Where is it? | `https://mcp.flung.dev/mcp-oauth` (sign in with your browser) or `https://mcp.flung.dev/mcp` (an API key). |
| Do I need an API key? | No, if your client supports MCP sign-in: you approve the agent in your browser and pick the workspace. Otherwise, a key that starts `flung_sk_`. |
| Can an agent connect accounts by itself? | No. It starts the connect and gives you the platform's own consent link; you approve on the platform. |
| Can it delete things by accident? | Disconnecting an account needs `confirm: true` in the call, and the tools say which ones change something. |
| Which platforms? | X, Instagram, Facebook, Threads, LinkedIn, TikTok, YouTube, Pinterest, Bluesky, Mastodon, Discord and Farcaster. |
| Is there a preview? | `https://mcp-preview.flung.dev`, with the same two paths, for trying changes before production. |

## Install

### Claude Code

```sh
claude mcp add --transport http flung https://mcp.flung.dev/mcp-oauth
```

Run `/mcp`, pick `flung`, and sign in when the browser opens. With an API key instead:

```sh
claude mcp add --transport http flung https://mcp.flung.dev/mcp --header "X-API-Key: $FLUNG_API_KEY"
```

### Claude Desktop and claude.ai

**Settings → Connectors → Add custom connector**, with the address `https://mcp.flung.dev/mcp-oauth`. Sign in when asked.

### Cursor

In `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "flung": { "url": "https://mcp.flung.dev/mcp-oauth" }
  }
}
```

### VS Code

In `.vscode/mcp.json`:

```json
{
  "servers": {
    "flung": { "type": "http", "url": "https://mcp.flung.dev/mcp-oauth" }
  }
}
```

### Any other client

Point it at `https://mcp.flung.dev/mcp-oauth` over Streamable HTTP. A client that cannot sign in sends a key to `https://mcp.flung.dev/mcp` as `X-API-Key: flung_sk_...` or `Authorization: Bearer flung_sk_...`. Keep the key in an environment variable, never in a file you commit.

## Try it

Once it is connected, ask your agent:

- "Which of my social accounts need reconnecting?"
- "Connect my LinkedIn." (it gives you a link to approve)
- "Post 'We're live' to X and Bluesky tomorrow at 9am my time."
- "When do people on Instagram usually read posts? Schedule this for the next good slot."
- "Did yesterday's post fail anywhere? Retry it."
- "Any new comments? Reply thanks to the nice ones."
- "Post every new item from our blog feed to LinkedIn."

## Workflows

Each does the work of several tools in one call, for the jobs agents are asked for most. An agent picks these first.

| Workflow | What it does |
|---|---|
| `review_workspace` | One summary: accounts that need you, posts due soon, posts that failed, and unread comments, with the next step for each. |
| `publish_to_platforms` | Posts by platform name: finds your accounts, checks each platform's limits, imports images or video from links, and posts now, later, or at the best time. Calling it again does not post twice. |
| `connect_platforms` | Starts connecting several platforms at once and returns a link for each. |
| `reconnect_accounts_needing_attention` | A reconnect link for every account that has stopped or is about to. |
| `retry_failed_posts` | Publishes failed posts again, skipping any whose account still needs reconnecting. |
| `report_post_performance` | Views, likes, comments and reposts for your latest posts, best first, with totals. |

If a step is refused, the workflow stops there and says which step and why, as the tool would.

## Tools

| Tool | What it does |
|---|---|
| **Accounts** | |
| `find_accounts_needing_attention` | Accounts that need reconnecting now or soon. A good first call. |
| `list_accounts`, `get_account` | Accounts with their status, how they are renewed, and when they must be reconnected by. |
| `connect_account`, `reconnect_account` | Starts a connect and returns the platform's consent link for you to open. |
| `get_connect_session` | Whether you finished: pending, connected, failed with the reason, or expired. |
| `refresh_account` | Renews an account's access now and checks it still works. |
| `disconnect_account` | Deletes the stored access; the account stops posting. Needs `confirm: true`. |
| **Posts** | |
| `schedule_post` | Posts now or later to one or more accounts, with images or a video. |
| `update_post`, `cancel_post` | Changes or stops a post that has not started publishing. |
| `retry_post` | Publishes again where a post failed; accounts that already published are left alone. |
| `import_media` | Fetches an image or video from a public link, ready to attach. |
| `get_post`, `list_posts` | A post's result on each account, with the link on the platform or the reason it failed. |
| `get_post_metrics` | Views, likes, comments and reposts per account, read from the platforms now. |
| **Planning** | |
| `list_platforms` | Each platform's text and media limits, and whether it can be connected. |
| `suggest_posting_times` | The next times a platform's audience is usually most active. |
| `get_posting_options` | TikTok's audience and interaction choices, which TikTok asks apps to offer before each post. |
| **Inbox** | |
| `list_inbox` | Comments on your posts and mentions of your accounts, newest first. |
| `reply_to_inbox_item` | Replies publicly as the account. |
| **Automation** | |
| `list_templates`, `deploy_template` | Reusable posts, scheduled with their `{{variables}}` filled in. |
| `list_variables`, `set_variable` | The values put into `{{name}}` placeholders. |
| `list_feeds`, `follow_feed` | RSS and Atom feeds that post their new items. |
| **Progress** | |
| `get_experience` | The workspace's level, posting streak and achievements. |

Tool names are a contract: new tools are added, existing ones are never renamed.

## Security

| Question | Answer |
|---|---|
| What can the agent reach? | Only the workspace you approved, or the one the API key belongs to. |
| What does a refusal look like? | A normal tool result with `isError`, a sentence saying what to do next, and `_meta["flung/refusal"]` for clients that branch on it. |
| What is logged? | One line per tool call: the tool, whether it worked, the API's status, how long it took, the size of the arguments and a fingerprint of the key. Never the key, the token or what the arguments said. |
| How do I cut an agent off? | Revoke its API key (`DELETE /v1/api-keys/{id}`); it stops on its next call. An agent signed in with OAuth stops at once when the person who approved it is removed from the workspace. |

Found a security problem? See [SECURITY.md](./SECURITY.md).

## Docs

The full tool reference, with every argument and an example call for each workflow and tool, is at [docs.flung.dev/mcp/tools](https://docs.flung.dev/mcp/tools). An agent can read it all at [docs.flung.dev/llms.txt](https://docs.flung.dev/llms.txt).

## Related

| Repository | What it is |
|---|---|
| [flung-sdk](https://github.com/flung-official/flung-sdk) | `@flung.dev/connect`, the connect panel for your own pages, and `@flung.dev/sdk`, the typed API client. |

## License

MIT for everything in this repository. The server itself is run by Flung.
