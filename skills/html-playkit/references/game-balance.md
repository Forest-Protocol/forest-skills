# Game Balance

Buying Project Token gives the player wallet-held tokens. It does not increase Game Balance. Game Balance is the deposited amount that can be debited or credited by walletless [Game Actions](./game-actions.md). For the relationship between wallet, Game Balance, and Game Vault, see [Core balances](./overview.md#core-balances). Amounts are display decimal strings.

> **INFO: Burns are wallet-side, not Game Balance**

Burns are wallet-side on-chain transfers and never appear in Game Balance — they live on the
[Burns](./burns.md) page.

### `forest.game.balance`

Reads the current player's Game Balance for this HTML project.

```js
callForest("forest.game.balance");
```

Successful responses return:

```js
{
  balance: "10",
  totalDeposited: "10",
  totalWithdrawn: "0",
  totalWagered: "2.5",
  totalWon: "1.25"
}
```

`forest.game.balance` is the player's **Game Balance** (internal ledger) state only. Burns do **not** appear here — read them from `forest.game.burns` / `forest.game.burns.summary`.

### `forest.game.deposit`

Asks Forest to deposit wallet-held Project Token into the player's Game Balance. Forest opens the connected wallet, submits the on-chain deposit transaction, verifies the transaction through the API, then returns the updated Game Balance.

```js
callForest("forest.game.deposit", {
	amount: "5",
});
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `amount` | string (display amount) | yes | Positive Project Token decimal string, up to 18 decimals. |

Successful responses return:

```js
{
  txHash: "0x...",
  balance: {
    balance: "15",
    totalDeposited: "15",
    totalWithdrawn: "0",
    totalWagered: "2.5",
    totalWon: "1.25"
  }
}
```

### `forest.game.withdraw`

Asks Forest to withdraw player Game Balance back to the connected wallet. Forest requests the withdrawal signature from the API, opens the connected wallet, submits the on-chain withdrawal, confirms it through the API, then returns the updated Game Balance.

```js
callForest("forest.game.withdraw", {
	amount: "2.5",
});
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `amount` | string (display amount) | yes | Positive Project Token decimal string, up to 18 decimals. |

Successful responses return the same shape as `forest.game.deposit`.

> **WARN: Deposits and withdrawals are Forest-owned wallet flows**

Do **not** self-credit deposits or sign withdrawals from your game server. After either call
succeeds, render `result.balance` or request `forest.game.balance` again.

#### Claim lifecycle

A withdrawal is a **claim** that is meant to reach `completed`. Each claim is `pending` (reserved, awaiting the wallet), `completed` (paid on-chain), or `failed` (reverted, balance restored). Your game owns surfacing claim state; Forest owns reverting it:

- **Track** a claim with `forest.game.transactions` — each `claim` row carries `id`, `status`, `requestId`, and `txHash`.
- **Retry** by calling `forest.game.withdraw` again with the same amount — Forest reuses the existing pending claim and re-prompts the wallet.

> **WARN: No cancel — claims are finalize-only**

If a player abandons a pending claim, Forest releases the reservation automatically once it is
confirmed unspent on-chain, so the player's balance returns with no client action. Do **not**
build a "cancel claim" control.

### `forest.game.transactions`

Reads the current player's paginated **Game Balance / Vault activity** for this HTML project — deposits, funding, and withdrawal claims (each `claim` row is a withdrawal request).

```js
callForest("forest.game.transactions", {
	page: 1,
	perPage: 20,
});
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `page` | number | no | Page number. Defaults to 1. |
| `perPage` | number | no | Number of rows per page. Defaults to 20. |

Successful responses return:

```js
{
  data: [
    {
      id: "transaction-id",
      type: "deposit",
      amount: "5",
      status: "confirmed",
      txHash: "0x…",
      createdAt: "2026-06-01T12:00:00.000Z"
    }
  ],
  meta: {
    total: 1,
    perPage: 20,
    currentPage: 1,
    lastPage: 1
  }
}
```

Each `data` entry has the following shape:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `id` | string | yes | Transaction identifier. |
| `type` | deposit \| claim \| fund | yes | Entry kind. A claim row is a withdrawal request and additionally carries requestId. |
| `amount` | string (display amount) | yes | Display token amount string. |
| `status` | string | yes | Entry status as returned by the feed (e.g. confirmed). |
| `requestId` | string | no | Present on claim rows only — the withdrawal request identifier. |
| `txHash` | string (0x hash) | no | On-chain transaction hash. |
| `createdAt` | string (ISO 8601) | yes | Timestamp the entry was created. |

The `meta` object carries `total`, `perPage`, `currentPage`, and `lastPage`.

> **WARN: Player-scoped read — do not proxy through your backend**

This is a Forest-authenticated, player-scoped read through the iframe proxy: it returns only the
signed-in player's rows, never all users' withdrawal requests. It is **not** a creator/admin
listing API. Do **not** proxy it through your game backend.

> **INFO: Burns are a separate domain**

Burns are **not** in this feed — use `forest.game.burns` for the player's recorded burns. See
[Burns](./burns.md).

## Game Vault Funding

HTML projects use a Game Vault.

The Game Vault is the project's payout capacity. It is separate from a player's Game Balance:

- Game Vault funding lets the project pay player winnings.
- Player Game Balance is the deposited, playable balance for one player.
- Funding the Game Vault does not credit any player.
- A player deposit does not add project payout capacity.

If the Game Vault runs out, credit-producing settlements can fail even when the player's action was authorized. The creator must top up the Game Vault with Project Tokens, then Forest must record that funding transaction.

> **WARN: Funding is not an iframe RPC method**

Do **not** ask uploaded HTML to transfer funds or record vault funding. Uploaded HTML cannot
read or fund Game Vault capacity through iframe RPC. Keep creator funding controls in Forest
project tools or another trusted creator/admin surface. Call `forest.game.balance` only when you
need the current player's playable balance.

A creator-side flow should:

### Step 1
### Transfer Project Tokens

Transfer Project Tokens to the project's Game Vault address.

### Step 2
### Wait for the transaction hash

Wait for the on-chain transfer to be mined and capture its transaction hash.

### Step 3
### Record the funding transaction

Record the funding transaction through the authenticated Forest API:

```http
POST /campaigns/{projectId}/html/fund
Content-Type: application/json
```

```json
{
	"txHash": "0x..."
}
```

Forest verifies the transaction on-chain before increasing recorded Game Vault capacity. The transfer token must be the project's token, and the recipient must be the project's Game Vault.

Creators can fund the Game Vault and inspect recent vault funding records from the Forest project edit page. That owner-only history is separate from player transaction history.
