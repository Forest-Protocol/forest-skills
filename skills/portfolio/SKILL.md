---
name: portfolio
description: Use when an AI agent needs to read a user's positions, wallet and staked balances, PnL, recent trades, or live prices and candles across Forest Protocol tokens, or to plan (not execute) a rebalance or take-profit — via the Forest REST API with a wallet the agent can sign with.
license: MIT
---

# Forest Portfolio (read & plan)

Read a user's positions, PnL, and market data across Forest Protocol tokens through the public
REST API, and turn them into rebalance / take-profit plans.

**Read-and-plan only.** There is no generic trade-execution API. Swaps happen in the Forest UI,
or inside a Forest game through a user-approved swap session (see the `html-playkit` skill).
Never invent a trading endpoint.

## API basics

- Base URL: `https://api.forest.inc`
- Interactive reference (Scalar): `https://api.forest.inc/docs`; raw OpenAPI: `/docs/json`
- Auth: SIWE (EIP-4361) → Bearer JWT. `/me/*` endpoints answer only for the authenticated
  user's linked wallets; there is no "portfolio of arbitrary wallet X" endpoint.

## Authenticate (SIWE → JWT)

1. `POST /auth/ethereum/challenge` → `201 { "nonce": "..." }` (single-use, expires after 5
   minutes).
2. Build a standard EIP-4361 message for the user's wallet address with that nonce, using
   `forest.inc` as the domain and `https://forest.inc` as the URI; sign it with the wallet.
3. `POST /auth/ethereum` with body `{ "message": "<siwe message>", "signature": "0x..." }`
   (optional `walletProvider` metadata object) → `201 { "accessToken": "..." }` plus the
   `forest-session` cookie.
4. Send `Authorization: Bearer <accessToken>`. The token lasts about 30 minutes; renew with
   `POST /auth/refresh`, which uses the session cookie.

## Endpoints

| Call | Auth | Purpose |
| --- | --- | --- |
| `GET /me/portfolio?currency=usd` | yes | Positions, balances, PnL, and summary. `currency`: `usd` (default) or `bnb`. |
| `GET /me/portfolio/activity?currency=usd` | yes | Last 10 buys/sells across the user's wallets. |
| `GET /tokens` | no | Paginated token list with market data. See query below. |
| `GET /tokens/:tokenId?currency=usd` | no | One token by UUID. |
| `GET /tokens/:tokenId/stats` | no | Derived token stats. |
| `GET /tokens/:tokenId/holders` | no | Paginated holders. |
| `GET /pairs/:pairAddress/candles` | no | OHLCV. `interval`: `1m 5m 15m 1h 4h 1d 1w` (default `1h`); `limit` 1–1000 (default 500); `to` = exclusive ISO cursor; `currency`. |
| `GET /pairs/:pairAddress/swaps` | no | Recent swaps for a pair. |

`GET /tokens` query: `page` (default 1), `perPage` (default 10, max 250), `orderBy` (`price`,
`marketCap`, `change24h`, `volume24h`, `liquidity`, `allTimeHigh`, `createdAt`, `name`, …),
`direction` (`asc`/`desc`), `search` (name or symbol), `graduated` (`true`/`false`),
`currency`, and `owner=me` (requires auth). Responses are `{ data, meta: { total, perPage,
currentPage, lastPage } }`.

### `GET /me/portfolio` response

```jsonc
{
	"currency": "usd",
	"pnlMethod": "average-buy-price-estimate",
	"balanceSnapshot": { "chainId": 56, "blockNumber": 0, "blockTimestamp": "ISO", "checkedAt": "ISO" }, // or null
	"summary": {
		"totalValue": "…", // requested currency
		"stakingValue": "…", // requested currency
		"tokensHeld": 0,
		"unrealizedPnl": "…", // requested currency, or null
		"realizedPnl": "…", // requested currency, or null
		"totalPnl": "…" // requested currency, or null
	},
	"positions": [
		{
			"id": "uuid", // token id for /tokens/:tokenId
			"name": "…",
			"symbol": "…",
			"chainId": 56,
			"decimals": 18,
			"address": "0x…", // ERC-20
			"pairAddress": "0x…", // for /pairs/:pairAddress/*
			"tier": "…",
			"walletBalance": "…", // base units, on-chain
			"stakedBalance": "…", // base units, on-chain
			"totalBalance": "…", // base units, wallet + staked
			"price": "…", // requested currency
			"change24h": 0, // percent
			"volume24h": "…", // requested currency
			"marketCap": "…", // requested currency
			"totalBought": "…", // token amount, WAD
			"totalSold": "…", // token amount, WAD
			"totalSpent": "…", // FOREST spent on buys, WAD
			"totalReceived": "…", // FOREST from sells, WAD
			"averageBuyPrice": "…", // FOREST per token, WAD
			"currentPrice": "…", // FOREST per token, WAD
			"unrealizedPnl": "…", // requested currency
			"unrealizedPnlPercent": 0,
			"realizedPnl": "…", // requested currency
			"tradeCount": 0,
			"firstTradeAt": "ISO | null",
			"lastTradeAt": "ISO | null"
		}
	]
}
```

Market and trade fields are nullable. `GET /me/portfolio/activity` returns
`[{ action: "Buy" | "Sell", tokenSymbol, amount /* base units */, price /* requested currency,
nullable */, timestamp, txHash, pairAddress }]`.

## Units

- **Base units** (`walletBalance`, `stakedBalance`, `totalBalance`, activity `amount`): divide
  by `10 ** decimals`.
- **WAD** (`totalBought`, `totalSold`, `totalSpent`, `totalReceived`, `averageBuyPrice`,
  `currentPrice`): fixed-point 1e18. Bought/sold are token amounts; the rest are FOREST.
- **Requested currency** (`price`, `volume24h`, `marketCap`, `*Pnl`, summary values): already
  converted server-side. Pass the same `currency` to every call you combine.
- `GET /tokens` without `currency` returns raw FOREST-denominated values; always pass it.
- Use `bigint` or a decimal library for base-unit and WAD math, never floating point.

## Position math

- Position size: `totalBalance` (on-chain wallet + staked), not `totalBought − totalSold`.
- Position value: `totalBalance / 10 ** decimals × price`; `summary.totalValue` already sums it.
- Rebalance plan: compute each position's share of `summary.totalValue`, diff against target
  weights, and emit a buy/sell list. Present the plan — do not execute it.

## Caveats (these cause wrong answers)

1. **PnL is an estimate from indexed trades** (`pnlMethod: "average-buy-price-estimate"`).
   Tokens received by transfer, airdrop, or game reward count in balances but not in cost
   basis, so PnL on those positions is not meaningful.
2. **Session-executed swaps may be missing from trade fields.** Auto-swap sessions trade
   through an executor contract, so they can be absent from `tradeCount`, cost basis, and
   activity even though balances reflect them.
3. **External tokens are excluded** from `/me/portfolio`, and their `/tokens` market fields
   are null.
4. **`503 Portfolio prices are temporarily unavailable`** means a price source is down; retry
   later instead of reporting zero value.
5. **Self-only.** Don't pass a wallet address — there is no such parameter.
