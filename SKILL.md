---
name: net-worth-tracker
description: Read and update the user's personal net-worth / accounting app over its HTTP API — log trades, record balance changes, and read net worth, holdings, and history. Use whenever the user asks to record a transaction or purchase, update an account balance, or check their portfolio / net worth in their accounting app.
---

# Net-worth tracker

This skill operates the user's single-tenant accounting app through its HTTP API.
Everything is authenticated with one bearer token and expressed in plain JSON.

## Setup (the user provides these once)

- **Base URL** — the app instance, e.g. `https://their-instance.example.com`.
- **Token** — minted by the user in the app under **Settings → API access**.
  It is shown only once. Treat it as a secret. Tokens are scoped: **read-only**
  (GET only — any write returns `403 this token is read-only`) or **read &
  edit** (full access). If a write is refused with that message, tell the user
  they gave you a read-only token — they can mint an edit one in the app.

Store them as environment variables and send the token on **every** request:

```
Authorization: Bearer $ACCOUNTING_TOKEN
```

If you can persist config, save `ACCOUNTING_URL` and `ACCOUNTING_TOKEN`. Never
echo the token back to the user or write it to logs.

### Desktop app (running on this machine)

The desktop app is the same server running locally, so every endpoint below is
identical. Two differences:

- **Base URL** is `http://127.0.0.1:8464` — that port is the app's default. If
  it was already taken the app picks another one for that session, so confirm
  with `curl -s http://127.0.0.1:8464/health` and ask the user for the address
  in their browser if it doesn't answer.
- **There is no token.** A local app has no sign-in, so requests from this
  machine are accepted as they are: **omit the `Authorization` header** from
  every example below. Nothing else changes.

The app must be running — it *is* the server. If requests are refused with
"connection refused", ask the user to open the app rather than retrying.

Confirm which one you're talking to with `curl -s $ACCOUNTING_URL/api/auth/me`:
`"local_mode": true` is the desktop app.

## First, learn the data model

Before writing anything, fetch the live spec and the valid reference values:

```bash
curl -s -H "Authorization: Bearer $ACCOUNTING_TOKEN" "$ACCOUNTING_URL/api/openapi.json"
curl -s -H "Authorization: Bearer $ACCOUNTING_TOKEN" "$ACCOUNTING_URL/api/reference-data"
```

`reference-data` returns the known **accounts, providers, tickers, asset classes,
and categories**. Reuse these exact strings when logging — do not invent new
ones unless the user clearly wants a new account/holding (then pass
`"allow_new_values": true`).

**`/api/openapi.json` is the source of truth** for what you can do — it lists
every agent-facing endpoint with its request fields. The examples below are the
common cases; for anything else — recurring buys, loans, interest rates,
amortization, price overrides, allocation targets, settings, forecast
assumptions, opening positions, merging split fills into one trade,
**capital-gains tax** (yearly report, per-portfolio config, basis
declarations, tax tags, per-holding quantity and cost-basis registers), the
**import review queue** (staged rows from connected exchanges/banks/wallets:
confirm, dismiss, merge, link), **connection sync** (`POST
/api/connections/sync-all` refreshes every linked balance; per-connection
list/sync/mode under `/api/connections`), and the holdings **news** feed —
read the spec. Owner-only actions —
managing tokens/credentials/billing, resetting data, changing encryption,
household membership — are intentionally absent and return **`403`** to a
token; ask the user to do those in the app.

## Actions

### Read net worth and holdings

```bash
curl -s -H "Authorization: Bearer $ACCOUNTING_TOKEN" "$ACCOUNTING_URL/api/dashboard"
```

Returns current net worth, total assets, liabilities, P/L, and the full balance
sheet (every holding with its value). Use `/api/net-worth-history` for the
historical curve.

### Log a trade (buy / sell / swap)

Every transaction is a move **from one asset to another**. Buying 1,000 of VWCE
is cash going out, VWCE coming in. A salary deposit is "income" → cash; spending
is cash → an expense category. `overall_eur_value` is the trade's total value in
**the ledger's base currency** (EUR, USD, GBP, or CHF — the `eur` in the field
name is historical). Cash tickers are currency codes: the example below uses
`EUR`; use whatever this ledger's base (or foreign cash) currency is.

```bash
curl -s -X POST "$ACCOUNTING_URL/api/transactions" \
  -H "Authorization: Bearer $ACCOUNTING_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "tx_date": "2026-06-17",
    "from_account": "Bank", "from_provider": "Revolut",
    "from_ticker": "EUR", "from_asset": "cash", "from_amount": 1000,
    "to_account": "Brokerage", "to_provider": "IBKR",
    "to_ticker": "VWCE", "to_asset": "stock", "to_amount": 8.5,
    "overall_eur_value": 1000
  }'
```

A `400` lists the offending fields. If it complains about "unknown values",
either correct the name to match `reference-data` or add `"allow_new_values": true`.

### Record a balance change (no counterparty)

For interest accrual, an external deposit, or a correction — adjust a holding's
quantity by a signed delta:

```bash
curl -s -X POST "$ACCOUNTING_URL/api/position-adjustments" \
  -H "Authorization: Bearer $ACCOUNTING_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "tx_date": "2026-06-17", "account": "Bank", "provider": "Revolut",
        "ticker": "EUR", "asset": "cash", "quantity_delta": 250 }'
```

### Update or delete

The API uses **PUT** to update and **DELETE** to remove — there is **no PATCH**
(PATCH returns `501`). Updates are a **full replace**: send the complete object,
not just the changed fields. To edit a transaction, read the current list, find
the row by `id`, change what you need, and PUT the whole object back:

```bash
curl -s "$ACCOUNTING_URL/api/transactions" -H "Authorization: Bearer $ACCOUNTING_TOKEN"   # find the id
curl -s -X PUT  "$ACCOUNTING_URL/api/transactions/<id>" -H "Authorization: Bearer $ACCOUNTING_TOKEN" \
  -H "Content-Type: application/json" -d '{ ...full transaction object... }'
curl -s -X DELETE "$ACCOUNTING_URL/api/transactions/<id>" -H "Authorization: Bearer $ACCOUNTING_TOKEN"
```

The same pattern applies to `/api/position-adjustments/{id}`. See
`/api/openapi.json` for every list / update / delete path.

## Rules

1. **Confirm before writing.** API writes commit immediately — there is no
   human "Confirm" step. Read `reference-data`, then echo back to the user
   exactly what you're about to log (amounts, assets, date) before POSTing.
2. **Reuse reference values.** Match existing accounts/providers/tickers so you
   don't create duplicates. Only create new ones on the user's explicit intent.
3. **Read `400` errors and fix the input** — don't retry the same body. The
   message names the bad field.
4. **Handle `423 locked`.** If the user enabled passphrase encryption and the
   ledger is locked, amount reads/writes return `423`. Tell the user to unlock
   it in the app — you cannot unlock it.
5. **Don't retry blindly on write timeouts.** There is no idempotency key, so a
   blind retry can double-log a trade. On an ambiguous failure, re-read
   `/api/dashboard` to check whether the write landed before retrying.
6. **Keep the token secret.** Never print it or commit it. (The desktop app
   has none — see Setup.)
