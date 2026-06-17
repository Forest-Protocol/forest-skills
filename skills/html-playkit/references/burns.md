# Burns

A burn is an **on-chain action**: the player sends Project Tokens from their own
wallet to a signed sink address — the dead address (`0x…dead`) by default, but the
backend may sign any destination. Forest records the verified on-chain transfer
with its `reason`/`metadata` for tracking. A transfer to `0x…dead` is a real,
provable supply burn; any other sink is just the app-chosen destination, not a
supply reduction.

Use it for in-game sinks such as activation fees. Burns are wallet-side and never touch [Game Balance](./game-balance.md).

> **WARN: Authorize before you transfer**

The burn settlement requires an *unused* action authorization, so authorize **first** —
otherwise the player can burn tokens on-chain and then fail Forest authorization/settlement,
stranding the burn. The transfer is irreversible; the authorization is not.

The **developer backend owns the burn intent** — it decides `actionId`, `amount`,
`burnAddress`, `reason`, and `metadata`. The client only executes that exact intent
on-chain and relays the resulting `txHash` back; it does not choose the destination
or purpose.

## The burn flow

### Step 1
### Intent and authorize

Your backend defines the intent (`actionId`, `amount`, `burnAddress`, `reason`,
`metadata`). The iframe calls
[`forest.game.action.authorize`](./game-actions.md#forestgameactionauthorize)
with that `actionId` and a base-unit debit limit — which is also the
player-authorized **max burn amount**. Do **not** use `0`.

### Step 2
### Burn (client)

```js
const { txHash, burnAddress } = await forest.game.burn({
	amount: "10", // display units
	burnAddress, // optional; defaults to 0x…dead
});
```

Forest performs the on-chain transfer and returns `{ txHash, burnAddress }`
**after receipt confirmation** — it does not call any API, and never surfaces a
pending tx hash. Relay `txHash` and `burnAddress` to your backend.

### Step 3
### Settle (backend)

Your backend HMAC-signs and submits `POST /campaigns/{projectId}/html/burns` with
the **same** `actionId`, the **base-unit** `amount`, `txHash`, `burnAddress`,
`reason`, `metadata`, and `timestamp`. Forest verifies on-chain that exactly
`amount` Project Tokens moved from a linked wallet to `burnAddress`, then records
the burn. Game Balance is never touched.

## Signed burn settlement

The signed amount is the **same economic amount** as the client burn — **display
units** in `forest.game.burn` (e.g. `"10"`), **base-unit integer** string in the
signed body (e.g. `"10000000000000000000"`).

```http
POST /campaigns/{projectId}/html/burns
Content-Type: application/json
X-Forest-Settlement-Signature: v1=<hex-hmac>
```

```js
await fetch(`${FOREST_API}/campaigns/${projectId}/html/burns`, {
	method: "POST",
	headers: {
		"Content-Type": "application/json",
		"X-Forest-Settlement-Signature": `v1=${hmac}`,
	},
	body: JSON.stringify({
		actionId: "same-action-id",
		amount: "10000000000000000000", // base-unit integer string
		txHash: "0x…",
		burnAddress: "0x000000000000000000000000000000000000dead",
		timestamp: Math.floor(Date.now() / 1000),
		reason: "activation_fee",
		metadata: { item: "gold_miner_rig", level: 3 },
	}),
});
```

Request fields:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `actionId` | string (UUID v4) | yes | Same UUID v4 used for forest.game.action.authorize. |
| `amount` | string (base-unit integer) | yes | Positive Project Token base-unit integer (the display amount times 10^decimals). Must equal the on-chain transferred value exactly and not exceed the authorized debit limit. |
| `txHash` | string (0x hash) | yes | Hash of the on-chain transfer from forest.game.burn. Verified against chain: sender is a linked wallet, recipient is burnAddress, token is the project token, value equals amount. |
| `burnAddress` | string (0x address) | yes | Destination the tokens were burned to (the value returned by forest.game.burn, e.g. the dead address). Signed as part of the body. |
| `timestamp` | number (Unix seconds) | yes | Current Unix timestamp in seconds. Generate it when sending the request. |
| `reason` | string (slug) | no | Optional stable lowercase slug (a-z, 0-9, _, e.g. activation_fee) that Forest groups and aggregates burns by. Signed, so it cannot be client-forged. |
| `metadata` | object | no | Optional free-form object for your own context. Signed, stored, and echoed back as-is (not Forest-interpreted). Capped at 4 KB serialized. |

Sign the exact request body with your Settlement Signing Secret, identical to the
[settlement flow](./game-actions.md#trusted-settlement). Successful responses return:

```js
{
  burnId: "burn-id",
  actionId: "same-action-id",
  amount: "10",
  reason: "activation_fee",
  metadata: { item: "gold_miner_rig", level: 3 },
  txHash: "0x…",
  burnAddress: "0x000000000000000000000000000000000000dead",
  replayed: false
}
```

> **INFO: Retrying a burn settlement**

Burns are idempotent by `actionId`. A safe retry MUST replay the **same
`actionId`, the exact same JSON body, and the same signature**, and arrive
**within the request-freshness window (~5 min)**. An exact replay returns
`replayed: true`. After the window, do **not** retry — confirm the burn settled by
looking up its `burnId` in `forest.game.burns`.

Idempotency is keyed on `actionId`, not `txHash`: a `txHash` backs exactly one
burn, and a `txHash` already recorded as a deposit is a conflict.

Burns do **not** change Game Balance (the tokens are burned straight from the
wallet) and are **not** part of `forest.game.transactions`.

Replay and validation failures return the shared settlement error codes —
`SETTLEMENT_TIMESTAMP_STALE`, `ACTION_ID_REUSED`, `BURN_TX_HASH_REUSED`,
`DEBIT_LIMIT_EXCEEDED`, `INVALID_SETTLEMENT_AMOUNT`, and signature errors. See
[Errors and Security](./errors-security.md#settlement-errors)
for each code's cause and resolution.

## Reading burns

`forest.game.burns` returns the current player's recorded burns (paginated,
newest first); `forest.game.burns.summary` returns the player's totals grouped by
`reason`. Both are player-scoped iframe reads (like `forest.game.balance`).

```js
const { data, meta } = await callForest("forest.game.burns", { page: 1, perPage: 20 });
// data[]: { id, amount, reason, metadata, txHash, burnAddress, createdAt }

const summary = await callForest("forest.game.burns.summary");
// { totalBurned, byReason: [{ reason, amount, count }] }
```

`totalBurned` here is "verified burn/sink settlements" — not necessarily supply
reduced (only dead/zero destinations reduce supply). Burns sent without a `reason`
are grouped under `reason: null`. Amounts are display token amount strings.

Token-level burn history is separate from Game Balance history. The token Burns
tab is populated from synced on-chain token transfers to the burn address or zero
address, alongside backend buyback burns.
