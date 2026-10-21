---
name: html-playkit
description: Use when building or debugging a Forest game or app — a Forest Harness game (`@forest/runtime` frontend, hosted backend with `ctx.player`, `ctx.db`, `ctx.forest`) or a legacy HTML upload on the iframe postMessage bridge. Covers swaps, FOREST session quotes, swap sessions and their failure codes, game balance, burns, game actions (settlement), player identity, display-vs-base-unit amounts, and the trusted-backend boundary.
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
5. **Keep `requestId` stable until the outcome is final.** A `409` or any uncoded failure means
   retry the same `requestId` and body. Only `SWAP_SESSION_EXECUTOR_UNFUNDED` and
   `SWAP_SESSION_EXECUTION_EXPIRED` prove the request never ran; only then use a new ID.

## Which reference to load

| Task | File |
| --- | --- |
| Model, balances, amounts, Preview | `references/overview.md` |
| First game: frontend + hosted backend | `references/quickstart.md` |
| Backend methods, `ctx.player`, `ctx.db`, `ctx.forest` | `references/hosted-backend.md` |
| Quote / buy / sell, FOREST session quotes | `references/swaps.md` |
| User-approved auto-swaps and execution outcomes | `references/swap-sessions.md` |
| Read / deposit / withdraw Game Balance | `references/game-balance.md` |
| Token burns (in-game sinks) | `references/burns.md` |
| Game action authorization → settlement | `references/game-actions.md` |
| Wallet events and player identity | `references/identity.md` |
| Own database for a legacy HTML backend | `references/persistent-data.md` |
| RPC envelope and full method list | `references/rpc-reference.md` |
| Error codes and security model | `references/errors-security.md` |

`references/*.md` mirror the canonical docs at
[docs.forest.inc](https://docs.forest.inc/developers/forest-playkit); the docs win if they differ.

## Common mistakes

- Sending base units in a frontend SDK call (use display strings).
- Trusting a browser-supplied player ID, address, score, or payout.
- Crediting a player by funding the Game Vault.
- Settling credit or recording a burn from the frontend.
- Deriving swap-session limits from the display quote or a fixed price instead of a
  `currency: "FOREST"` session quote.
- Minting a new `requestId` after a timeout or `409` (double-trade risk).
- Caching a swap-session id, or treating an approval hash as completion.
- Treating Preview results marked `simulated: true` as live market data.
- Querying the database from the frontend; use backend methods.
