---
name: html-playkit
description: Use when building or debugging an HTML game or app embedded in a Forest token page — the iframe postMessage bridge, swaps, swap sessions, game balance, burns, game actions (settlement), or verified player identity. Covers display-vs-base-unit amounts, the trusted-backend boundary, and the Forest RPC method envelope.
---

# Forest Playkit (HTML SDK)

Toolkit for HTML games/apps that run **inside an iframe on a Forest token page** and talk to
Forest over browser `postMessage`. No package install, no build step. The iframe reads wallet
state and asks Forest to open the user's wallet; a trusted backend authorizes anything that
moves value.

## Core mental model (read before coding)

**Two trust surfaces — keep them straight and the rest follows:**

- **The iframe (client, untrusted).** Your uploaded HTML. Calls Forest RPC methods to read state
  and to ask Forest to open the user's wallet. Anything it reports can be forged by the player.
- **Your backend (trusted).** Holds the **Settlement Signing Secret**. The only place allowed to
  produce credit (settlements), execute swap sessions, redeem identity, and record burns. The
  browser can *request* these; only your server can *authorize* them.

Forest sits between the two: mediates wallet actions for the iframe, verifies the backend's
signed requests.

## Three rules that cause most bugs

1. **Amounts: display vs base units.** SDK (iframe) calls take **display** strings (`"1"`,
   `"0.25"`) — no wei, commas, symbols, or suffixes. **Signed backend bodies and
   session/settlement responses use base-unit integer strings.** When the backend signs an amount
   the iframe also passed, both must be the *same economic value* in their respective units.
2. **Identity is not the wallet address.** Forest sends the connected address to the iframe for
   **display only** — a player-rewritable string. Never use it for backend identity or
   accounting. Run the Player Identity handshake (`references/identity.md`).
3. **Three separate pools of value — never conflate.** Wallet-held Project Token (in the player's
   wallet) ≠ Game Balance (per-player internal ledger Forest debits/credits) ≠ Game Vault
   (project payout capacity, creator-funded). A deposit moves wallet→game balance; funding the
   vault credits no player.

## Which reference to load

| Task | File |
| --- | --- |
| First bridge + first call | `references/quickstart.md` |
| Mental model / balances | `references/overview.md` |
| Quote / buy / sell through wallet | `references/swaps.md` |
| User-approved auto-swaps, trusted execution | `references/swap-sessions.md` |
| Read / deposit / withdraw playable balance | `references/game-balance.md` |
| On-chain token burns (in-game sinks) | `references/burns.md` |
| One-use auth codes → backend settlement | `references/game-actions.md` |
| Wallet events + verified-player handshake | `references/identity.md` |
| Your own DB for stats / state / leaderboard | `references/persistent-data.md` |
| RPC envelope + full method list | `references/rpc-reference.md` |
| Error codes + security model | `references/errors-security.md` |

`references/*.md` is generated from the canonical fumadocs docs — treat as the detailed spec.

## Common mistakes

- Sending wei/base units in an iframe SDK call (use display strings).
- Trusting the iframe-reported wallet address for accounting.
- Crediting a player by funding the Game Vault (it funds payouts, not players).
- Settling credit or recording a burn from the iframe (server-only, needs the signing secret).
- Hardcoding `projectId` instead of storing the one Forest provides.
- Caching a swap-session id, or treating an approval hash as completion (it's progress).
- Querying your own game database (stats/state) from the iframe — keys and queries are server-only (`references/persistent-data.md`).
