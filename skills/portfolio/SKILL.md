---
name: portfolio
description: Use when an AI agent needs to read a user's positions, holdings, PnL, cost basis, or live prices across Forest Protocol tokens, or to plan (not execute) a rebalance or take-profit — via the Forest REST API with a wallet the agent can sign with.
---

# Forest Portfolio (read & plan)

Read a user's positions, PnL, and live prices across all Forest Protocol tokens through the
public REST API, and turn them into rebalance / take-profit plans.

**Read-and-plan only.** There is no generic trade-execution API surface. Executing swaps happens
in the Forest UI, or — inside an HTML game/app context — through per-project swap sessions (see
the `html-playkit` skill). Never invent a trading endpoint.

## API basics

- Base URL: `https://api.forest.inc`
- Interactive reference (Scalar): `https://api.forest.inc/docs`
- Auth: SIWE (EIP-4361) → Bearer JWT. Portfolio endpoints are self-only — they cover the
  authenticated user's linked wallets; there is no "positions of arbitrary wallet X" endpoint.

## Authenticate (SIWE → JWT)

1. `POST /auth/ethereum/challenge` → `201 { "nonce": "..." }` (single-use).
2. Build a standard EIP-4361 message for the user's wallet address with that nonce; sign it
   with the wallet.
3. `POST /auth/ethereum` with body `{ "message": "<siwe message>", "signature": "0x..." }`
   (optional `walletProvider` metadata object) → `201 { "accessToken": "..." }` plus a session
   cookie.
4. Send `Authorization: Bearer <accessToken>` on subsequent requests. When it expires,
   `POST /auth/refresh` (uses the session cookie).

## Endpoints

| Call | Purpose |
| --- | --- |
| `GET /me/portfolio/pnl?currency=usd` | Positions + PnL for every token the user's wallets have traded. `currency`: `usd` (default) or `bnb`. Auth required. |
| `GET /tokens?currency=usd` | Batch market data for all published tokens: `price`, `marketCap`, `change24h`, `volume24h`, `liquidity`. Paginated; supports `search`, `graduated`, `owner=me`. |
| `GET /tokens/:tokenId?currency=usd` | Single-token detail. |

### `GET /me/portfolio/pnl` response

```jsonc
{
	"tokens": [
		{
			"tokenId": "uuid",
			"tokenName": "…",
			"tokenSymbol": "…",
			"tokenAddress": "0x…", // ERC-20 on BNB Chain
			"pairAddress": "0x…",
			"totalBought": "…", // WAD string (1e18)
			"totalSold": "…", // WAD string
			"totalSpent": "…", // FOREST spent on buys, WAD
			"totalReceived": "…", // FOREST from sells, WAD
			"averageBuyPrice": "…", // FOREST per token, WAD
			"currentPrice": "…", // FOREST per token, WAD
			"unrealizedPnl": "…", // in requested currency
			"unrealizedPnlPercent": 0,
			"realizedPnl": "…", // in requested currency
			"tradeCount": 0,
			"firstTradeAt": "ISO date | null",
			"lastTradeAt": "ISO date | null"
		}
	],
	"totalUnrealizedPnl": "…",
	"totalRealizedPnl": "…",
	"totalPnl": "…",
	"currency": "usd"
}
```

## Amounts: WAD vs currency

- `totalBought/totalSold/totalSpent/totalReceived/averageBuyPrice/currentPrice` are **WAD
  strings** — fixed-point 1e18. Divide by `1e18` before showing to a human.
- `*Pnl` totals are already in the requested `currency` (usd/bnb).
- Token prices are FOREST-quoted internally; the `currency` param converts server-side. Always
  pass the same `currency` to `/tokens` and `/me/portfolio/pnl` when combining them.

## Position math

- Position size: `netHolding = totalBought − totalSold` (WAD).
- Position value: `netHolding × currentPrice` (then convert/display via `currency` fields).
- Rebalance plan: compute each position's share of total value, diff against target weights,
  emit a buy/sell list. Present the plan — do not execute it.

## Caveats (these cause wrong answers)

1. **Holdings are swap-derived, not wallet balances.** `netHolding` comes from indexed DEX
   swaps only. Tokens received via transfers, airdrops, or game rewards are invisible to it.
   Treat it as the *trading position*; for the exact spendable balance, read the ERC-20
   `balanceOf(tokenAddress)` on BNB Chain.
2. **Session-executed swaps may be missing.** Swaps executed through a Forest-managed executor
   wallet (auto-swap sessions) attribute to the executor on-chain and may not appear in the
   user's PnL.
3. **Self-only.** The PnL endpoint always answers for the authenticated user. Don't try to pass
   a wallet address — there is no such parameter.
