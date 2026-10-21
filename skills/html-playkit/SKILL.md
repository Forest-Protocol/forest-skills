---
name: html-playkit
description: Use when building or debugging a Forest game or app — a Forest Harness game (`@forest/runtime` frontend, hosted backend with `ctx.player`, `ctx.db`, `ctx.forest`) or a legacy HTML upload on the iframe postMessage bridge. Covers swaps, FOREST session quotes, swap sessions and their failure codes, game balance, burns, game actions (settlement), player identity, display-vs-base-unit amounts, and the trusted-backend boundary.
license: MIT
---

# Forest Playkit

Connects a game to Forest's wallet, trading, and game economy. Two shapes:

- **Forest Harness game (current).** Forest hosts the frontend, backend, and database. The
  frontend imports `forest` from `@forest/runtime`; trusted rules live in the hosted backend,
  which gets the verified player as `ctx.player`, stores data with `ctx.db`, and moves value
  with `ctx.forest`. No signing secret.
- **Legacy HTML upload.** Uploaded HTML in an iframe calls `callForest(method, params)` over
  `postMessage`; your own backend holds the Settlement Signing Secret and signs REST calls.

## Core mental model (read before coding)

**The browser is untrusted; the backend decides.** The frontend draws the game and asks Forest
to open the wallet. Anything it reports — scores, player IDs, payouts — can be forged. Only the
backend settles credit, executes swap sessions, and records burns.

## Rules that cause most bugs

1. **Amounts: display vs base units.** Frontend SDK calls take **display** strings (`"1"`,
   `"0.25"`). Backend settlement, burn, and swap-execution inputs, session budgets, and
   `sessionQuote` amounts are **base-unit integer strings**. Do the math with `bigint`; never
   pass base units through `Number`.
2. **Identity is not the wallet address.** The address the frontend sees is for display only.
   Hosted backends use `ctx.player.id`; legacy backends run the identity handshake.
3. **Three separate pools of value.** Wallet-held Project Token ≠ Game Balance (per-player
   ledger) ≠ Game Vault (creator-funded payout capacity). Funding the vault credits no player.
4. **Swap-session amounts come from a session quote.** Call `forest.swap.quote` with
   `currency: "FOREST"` and pass its `sessionQuote` `projectAmount`, `forestAmountLimit`, and
   `expiresAt` (as `deadline`) to `executeSwapSession`. The plain quote is display-only.
5. **Keep `requestId` stable until the outcome is final.** In a hosted backend every failure
   except the two codes below arrives as `RUNTIME_FAILED`; retry the same `requestId` and
   payload. Only `SWAP_SESSION_EXECUTOR_UNFUNDED` and `SWAP_SESSION_EXECUTION_EXPIRED` prove the
   request never ran; only then use a new ID.
6. **Settlements replay by `actionId`.** Calling `settleAction`/`settleBurn` again with the same
   `actionId` and amounts returns `replayed: true` instead of charging twice. Your own SQL
   writes are not replayed — make them idempotent.
7. **Stay inside the backend budget.** 32 `ctx.db` calls and 32 `ctx.forest` calls per
   invocation; batch reads. Migrations reject SQL functions such as `now()`. Limits, jobs, and
   `forest.json` are in `references/hosted-backend.md`.

## Which reference to load

| Task | File |
| --- | --- |
| Model, balances, amounts, Preview | `references/overview.md` |
| First game: frontend + hosted backend | `references/quickstart.md` |
| Backend methods, `ctx`, limits, `forest.json`, scheduled jobs | `references/hosted-backend.md` |
| Quote / buy / sell, FOREST session quotes | `references/swaps.md` |
| User-approved auto-swaps and execution outcomes | `references/swap-sessions.md` |
| Read / deposit / withdraw Game Balance | `references/game-balance.md` |
| Token burns (in-game sinks) | `references/burns.md` |
| Game action authorization → settlement | `references/game-actions.md` |
| Wallet events and player identity | `references/identity.md` |
| Legacy HTML only: your own external database | `references/persistent-data.md` |
| RPC envelope and full method list | `references/rpc-reference.md` |
| Error codes and security model | `references/errors-security.md` |

`references/*.md` mirror the docs at
[docs.forest.inc](https://docs.forest.inc/developers/forest-playkit). Harness games: skip sections
marked legacy (`callForest`, HMAC signing, identity codes).

## Preview differs from live

Harness Preview simulates the economy (`simulated: true`): swaps run at a fixed 1:1, session
quotes return no `sessionQuote`, `swap.session.current` resolves `{ session: null }` instead of
failing, the two swap failure codes never occur, and jobs do not run on a timer. Cover those
paths with test fixtures, and never present Preview numbers as market data.

## Common mistakes

- Sending base units in a frontend SDK call (use display strings).
- Trusting a browser-supplied player ID, address, score, or payout.
- Crediting a player by funding the Game Vault.
- Settling credit or recording a burn from the frontend.
- Deriving swap-session limits from the display quote or a fixed price instead of a
  `currency: "FOREST"` session quote.
- Minting a new `requestId` after a timeout or `409` (double-trade risk).
- Caching a swap-session id, or treating an approval hash as completion.
- Calling `forest.backend.call` and expecting an error `code` — it throws a plain error;
  return `{ ok, data | error }` envelopes from methods instead.
- Using `defineBackend(...)` without `Object.freeze` — the backend fails to load.
- Querying the database from the frontend; use backend methods.
