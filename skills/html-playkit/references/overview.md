# Overview

Forest Playkit is your toolkit for building HTML games and apps that live natively inside Forest Protocol.

Your app runs inside an iframe on the token page and communicates with Forest through browser `postMessage` calls. With Playkit you can read the connected wallet, request swap quotes, submit swaps through the user's wallet, and tap into player-scoped game APIs. No package to install. No build step required.

New here? Start with the [Quickstart](./quickstart.md) — you'll have a working bridge and your first call running in minutes. This page covers the mental model behind it.

> **INFO: Code examples**

Throughout these docs:

- **Client snippets are JavaScript** and are valid TypeScript as-is.
- **Backend snippets are Node.js.**
- **HTTP endpoints** are shown as a raw request, plus a cURL example where useful.
- **Amounts** in client (iframe) calls are display decimals; signed backend bodies use base-unit integer strings.

> **INFO: The two surfaces**

Every integration is split across two trust boundaries. Keep them straight and the rest of the SDK follows.

- **The iframe (client).** Your uploaded HTML. It calls Forest RPC methods to read state and to ask Forest to open the user's wallet. It is untrusted — anything it reports can be forged by the player in their own browser.
- **Your trusted backend (server).** Holds the **Settlement Signing Secret** and is the only place allowed to produce credit (settlements), execute swap sessions, redeem identity, and record burns. The browser can _request_ these, but only your server can _authorize_ them.

Forest sits between the two: it mediates wallet actions for the iframe and verifies your backend's signed requests.

## Core balances

Three different pools of value — do not conflate them:

| Concept                   | What it is                                                                                    |
| ------------------------- | --------------------------------------------------------------------------------------------- |
| Wallet-held Project Token | Tokens in the player's own wallet. Buying gives them these; it does **not** add Game Balance. |
| Game Balance              | A per-player internal ledger Forest debits/credits through walletless Game Actions.           |
| Game Vault                | The project's payout capacity. Separate from any player; funds winnings. Creator-funded.      |

How value moves between them:

- A player deposit moves wallet tokens into their Game Balance.
- Funding the Game Vault does **not** credit any player.
- A player deposit does **not** add payout capacity.

## Amounts: display vs base units

This trips people up, so it is the single rule to remember:

- **You send display amounts.** In HTML SDK params, use normal token display strings like `"1"`, `"0.25"`, or `"1000.5"`. No wei/base-unit values, commas, token symbols, or unit suffixes.
- **Signed backend bodies and session/settlement responses use base units.** Forest converts display request amounts to base-unit integer strings inside the Forest-owned web bridge before it calls the API or opens the wallet. Auto-swap session response budgets and trusted backend settlement/execution/burn amount fields are base-unit integer strings.

> **WARN: Both sides must be the same economic value**

When a backend signs an amount that the iframe also passed, the two must represent the **same
economic value**: display units in the SDK call, base-unit integer in the signed body.

## Identity is not the wallet address

> **WARN: Never use the iframe wallet address for identity**

Forest sends the connected wallet address to the iframe for **display only**. Once it reaches your HTML it is a plain string the player can rewrite, so:

- **Never use it for backend identity or accounting.**
- When your backend needs to know which Forest player is in the session, run the [Player Identity](./identity.md) handshake instead.

## What you can use

- [Swaps](./swaps.md) — Quote, buy, and sell Project Token through the connected wallet.
- [Swap Sessions](./swap-sessions.md) — User-approved auto-swap sessions and trusted execution.
- [Game Balance](./game-balance.md) — Read, deposit to, and withdraw the player
- [Burns](./burns.md) — Burn wallet-held Project Token on-chain for in-game sinks.
- [Game Actions](./game-actions.md) — One-use authorizations for trusted backend settlement.
- [Player Identity](./identity.md) — Wallet events and the verified-player handshake.

For the request/response envelope and the full method list, see the [RPC Reference](./rpc-reference.md).
