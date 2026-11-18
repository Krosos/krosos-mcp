# Krosos MCP server

Connect your AI assistant to your own net-worth ledger. [Krosos](https://krosos.com)
is a personal finance app for European investors: your entire financial
picture, the tax math and the forecast, in a private database of your own.

Krosos exposes a **remote MCP server** (Model Context Protocol), so Claude,
ChatGPT and most AI assistants can work with your ledger through typed tools:
ask about your net worth, or hand over your statements and let the assistant
prepare everything you own and owe for you to review.

> **This is not the app's source code.** Krosos is a commercial product; this
> repository holds the connector artifacts: the MCP manifest, the agent skill,
> and how to connect. On why the app itself isn't open source: a
> source-available edition is genuinely under consideration, and the hosted
> trust story today is your own database and keys, read-only connections, and
> export-and-leave. For zero operator trust there is a
> [desktop app](https://krosos.com/download) that runs entirely on your machine.

## Connect

The server is at `https://app.krosos.com/api/mcp`. You connect by **signing in**:
there is no token to copy.

**Claude** (web, desktop app and mobile): in Krosos, open **Settings → AI
assistant** and press **Connect Claude**. Claude opens with Krosos filled in;
confirm, press Connect, and approve the sign-in. Or use the link directly:

```
https://claude.ai/customize/connectors?modal=add-custom-connector&connectorName=Krosos&connectorUrl=https%3A%2F%2Fapp.krosos.com%2Fapi%2Fmcp
```

**ChatGPT** (a plan that allows custom connectors): in ChatGPT's settings, add a
custom MCP server named Krosos with the address above, leave sign-in on OAuth,
and approve the sign-in. Step by step: **Settings → AI assistant → ChatGPT →
Set up** in Krosos.

When you approve, you pick **Read only** (the default) or **Read & edit**.

**Claude Code, Cursor and other agents** connect with a token from **Settings →
API access**. In Claude Code:

```bash
claude mcp add krosos --transport http https://app.krosos.com/api/mcp --header "Authorization: Bearer YOUR_TOKEN"
```

**Desktop app:** your ledger stays on your computer, so only an assistant
running on that computer can reach it. There is no sign-in and no token:

```bash
claude mcp add krosos --transport http http://localhost:8464/api/mcp
```

(8464 is the default port; **Settings → AI assistant** shows the exact command.)
Claude's and ChatGPT's own apps connect from their servers and cannot reach the
desktop app.

## Tools

| Tool | What it does |
|---|---|
| `get_dashboard` | Net worth, balance sheet and current holdings |
| `get_net_worth_history` | Historical net-worth snapshots |
| `list_transactions` | Transaction history, newest first |
| `get_reference_data` | Valid accounts, providers, tickers and the base currency: read before writing |
| `get_tax_report` | Capital-gains / income tax estimate for a year, exemption carry-forward included |
| `get_projection_config` | The FIRE plan's assumptions |
| `get_prices` | Current price per ticker, with its source |
| `list_connections` | Linked banks, brokers, exchanges and wallets, and when each last synced |
| `propose_holdings_import` | Prepare an import of what you own and owe, one row per holding. Saves a **draft** you review in Krosos; works on a read-only connection |
| `refresh_prices` | Fetch fresh market prices *(read & edit)* |
| `sync_connections` | Sync every linked connection *(read & edit)* |
| `log_trade` | Log a buy / sell / swap *(read & edit)* |
| `record_balance_adjustment` | Record a quantity change on one holding *(read & edit)* |

Every tool is annotated as read-only or not, so your assistant asks before one
that changes something.

### Importing from statements

Attach your statements and ask the assistant to prepare an import of everything
you own and owe. It calls `propose_holdings_import`, and nothing is added: Krosos
shows **Your assistant prepared N holdings to import**, you open **Review**, see
every row exactly as it will be stored, and press **Import** or **Discard**.

## Boundaries, by design

- A **read-only connection can never change a number**. The one thing it can
  store is an import draft, which only you can apply, signed in to the app.
- **Nobody can move money**: Krosos holds read-only connections to banks,
  brokers and exchanges. There is no trade or transfer to call.
- Disconnecting in **Settings → AI assistant** (or revoking a token in **API
  access**) cuts access instantly.
- Only the protocol handshake answers without a sign-in; every data call
  requires yours.

Listed on the official MCP Registry as
[`com.krosos/krosos`](https://registry.modelcontextprotocol.io/v0/servers?search=krosos).
`server.json` in this repo is the published manifest.

## Docs

The full how-to: [krosos.com/docs/ai-assistant](https://krosos.com/docs/ai-assistant),
and [krosos.com/docs/api-access](https://krosos.com/docs/api-access) for tokens.
Tax figures are estimates, not advice.

For agents that cannot use MCP, [`SKILL.md`](SKILL.md) is a drop-in skill that
documents the same API over plain HTTP: add it to the agent's skills folder and
give it your app's address and a token.
