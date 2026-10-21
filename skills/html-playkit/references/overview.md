# Overview

Forest Playkit connects your game to Forest's wallet, trading, and game economy.

For games made in **Forest Harness**, Forest hosts the frontend, backend, and database. Import `forest` from `@forest/runtime` in the frontend. Put trusted game rules in the hosted backend, using `ctx.forest` for economic operations and `ctx.db` for persistent game data.

Start with the [Quickstart](./quickstart.md), then read [Hosted Backend](./hosted-backend.md) for the backend and database contract. Harness includes the authoring instructions for its AI assistant.

## Frontend and backend

| Where code runs | What it does                                                                                                              |
| --------------- | ------------------------------------------------------------------------------------------------------------------------- |
| Game frontend   | Draws the game, calls `forest.backend.call`, and asks Forest to handle wallet actions through the SDK.                    |
| Hosted backend  | Validates gameplay, identifies the player through `ctx.player`, stores data with `ctx.db`, and settles with `ctx.forest`. |
| Forest website  | Handles sign-in, wallet prompts, token launch, and transaction confirmation.                                              |

The browser is untrusted. Decide outcomes and rewards in the backend. Never accept a player ID, score, or payout from the browser without validation.

Hosted backends do not need a Settlement Signing Secret. Forest authenticates each invocation and supplies its verified player. Database and platform credentials stay outside your game code.

## Core balances

| Concept           | What it means                                                               |
| ----------------- | --------------------------------------------------------------------------- |
| Wallet-held token | Tokens in the player's wallet. Buying tokens increases this balance.        |
| Game Balance      | The player's deposited balance, which game actions can debit or credit.     |
| Game Vault        | The game's capacity to pay rewards, funded separately from player deposits. |

A deposit moves wallet tokens into Game Balance. Funding the Game Vault does not credit a player, and a player deposit does not add payout capacity. A burn spends wallet holdings and leaves Game Balance unchanged.

## Amounts: display vs base units

Use strings for exact token amounts:

| Operation                                                                      | Units                                                                              |
| ------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| Frontend game amounts, action debit limits, and swap-session budgets           | Display strings, such as `"1"` or `"0.25"`.                                        |
| Backend settlement, burn, and swap-execution inputs                            | Base-unit integer strings. With 18 decimals, one token is `"1000000000000000000"`. |
| Game Balance, settlement and burn results, transaction amounts, vault capacity | Display strings. A result may use `"1.0"` for one token.                           |
| Swap-session response budgets and backend swap-execution amounts               | Base-unit integer strings.                                                         |
| Swap quote snapshots                                                           | Display numbers for presentation.                                                  |
| `sessionQuote` amounts from `currency: "FOREST"` quotes                        | Base-unit integer strings.                                                         |

Do base-unit arithmetic with `bigint`, then serialize the result to a string. Do not pass base units through `Number` or `parseFloat`.

## Sign-in and player identity

Anonymous visitors can see the game immediately. A protected SDK call waits for sign-in. If the visitor dismisses sign-in, the call rejects with `SIGN_IN_REQUIRED`; handle that as a user choice.

`forest.wallet.onChange` supplies an address for display. The hosted backend gets the verified player through `ctx.player.id`; use that ID to scope private data. It does not need the legacy identity-code handshake.

## Persistent game data

Use `ctx.db` for inventory, progress, match history, and shared rankings. Filter private rows by `ctx.player.id`. Public rankings should return only the fields you intend to share. Each backend invocation starts with fresh globals, so durable data belongs in the database.

Game data and Forest's economic accounting are separate. A database write does not credit Game Balance, and a settlement does not save your application's progress automatically.

## Preview and launch

Harness Preview runs the backend with local persistent game data. The **Test wallet** and economic operations are simulated, with results marked `simulated: true`. Simulated balances reset when Preview restarts; local game data remains.

Use **Get ready to launch** in Harness, review the findings, then **Launch on forest.inc** to continue with your wallet. A published game is immutable, with one playable per token.

## Legacy HTML uploads with an external backend

Legacy HTML uploads continue to use the browser message bridge and an external backend. That backend holds the Settlement Signing Secret, verifies player identity, stores game data in its own database, and submits HMAC-signed requests to Forest.

The [legacy HTML quickstart](./quickstart.md#legacy-html-uploads), [identity handshake](./identity.md#player-identity), and signed sections of the economic guides remain available for these integrations.

## Guides

- [Hosted Backend](./hosted-backend.md) — Backend methods, verified players, and persistent SQL data.
- [Game Actions](./game-actions.md) — Player authorization and backend settlement.
- [Game Balance](./game-balance.md) — Balances, deposits, withdrawals, and vault funding.
- [Burns](./burns.md) — Wallet token sinks and verified burn records.
- [Swaps](./swaps.md) — Quotes and wallet-approved trading.
- [Swap Sessions](./swap-sessions.md) — User-approved automatic trading.
