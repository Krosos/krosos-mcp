# Krosos MCP server & agent skill

Connect your AI assistant to your own accounting ledger. [Krosos](https://krosos.com)
is a personal finance app for European investors: your entire financial
picture, the tax math and the forecast, in a private database of your own.

Every Krosos instance — hosted and the desktop app — exposes a **remote MCP
server** (Model Context Protocol) so Claude, Cursor and most AI assistants can
work with your ledger through typed tools, plus a **skill file** for agents
that don't speak MCP.

> **This is not the app's source code.** Krosos is a commercial product; this
> repository holds the connector artifacts: the MCP manifest, the agent skill,
> and how to connect. On why the app itself isn't open source: a
> source-available edition is genuinely under consideration, and the hosted
> trust story today is your own database and keys, read-only connections, and
> export-and-leave. For zero operator trust there is a
> [desktop app](https://krosos.com/download) that runs entirely on your machine.

## MCP (recommended)

The server lives at `https://app.krosos.com/api/mcp` (hosted) or
`http://localhost:<port>/api/mcp` (desktop, port shown in the app). Mint a
token in **Settings → API access** — **read-only** is the sensible default for
an assistant — and connect. Claude Code:

```bash
claude mcp add krosos --transport http https://app.krosos.com/api/mcp --header "Authorization: Bearer YOUR_TOKEN"
```

Cursor and Claude Desktop take the same URL and header in their MCP settings.
On the desktop app no token exists (no sign-in exists), so it is just:

```bash
claude mcp add krosos --transport http http://localhost:8000/api/mcp
```

### Tools

| Tool | What it does |
|---|---|
| `get_dashboard` | Net worth, balance sheet and current holdings |
| `get_net_worth_history` | Historical net-worth snapshots |
| `list_transactions` | Transaction history, newest first |
| `get_reference_data` | Valid accounts, providers, tickers — read before writing |
| `get_tax_report` | Capital-gains / income tax estimate for a year, exemption carry-forward included |
| `get_projection_config` | The FIRE plan's assumptions |
| `log_trade` | Log a buy / sell / swap *(requires a read & edit token)* |
| `record_balance_adjustment` | Record a quantity change on one holding *(requires a read & edit token)* |

### Boundaries, by design

- A **read-only token can never change a number**; write tools refuse it.
- **Nobody can move money**: Krosos holds read-only connections to banks,
  brokers and exchanges — there is no trade or transfer to call.
- Revoking the token in Settings cuts access instantly.
- Only the protocol handshake answers without a token (the same information
  the public [OpenAPI spec](https://krosos.com/docs/api-access) lists); every
  data call requires yours.

Listed on the official MCP Registry as
[`com.krosos/krosos`](https://registry.modelcontextprotocol.io/v0/servers?search=krosos) —
`server.json` in this repo is the published manifest.

## Agent skill (for agents without MCP)

[`SKILL.md`](SKILL.md) is the drop-in skill: add it to your agent's skills
folder and give it your app's address and a token. It documents the same API
the MCP tools ride.

## Docs

The full how-to lives at [krosos.com/docs/api-access](https://krosos.com/docs/api-access).
Tax figures are estimates, not advice.
