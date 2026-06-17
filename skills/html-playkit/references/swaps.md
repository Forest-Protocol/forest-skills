# Swaps

Swaps run through the connected wallet. Quote is a read; buy and sell open the wallet. Amounts are display decimal strings — see [Amounts](./overview.md#amounts-display-vs-base-units). For user-approved auto-swaps, see [Swap Sessions](./swap-sessions.md).

## Swap Methods

### `forest.swap.quote`

Gets a read-only quote and snapshot. This does not open the wallet or submit a transaction.

```js
callForest("forest.swap.quote", {
	amount: "0.05",
	direction: "buy",
	slippage: 0.5,
});
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `amount` | string (display decimal) | yes | Positive display decimal string. |
| `direction` | "buy" \| "sell" | yes | Swap direction. |
| `slippage` | number (percent) | no | Percent slippage. Forest uses its default when omitted. |

### `forest.swap.buy`

Submits a buy transaction through the connected wallet. `amount` is the amount of the pay token to spend.

```js
callForest("forest.swap.buy", {
	amount: "0.05",
	slippage: 0.5,
});
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `amount` | string (display decimal) | yes | Positive display decimal string of pay token to spend. |
| `slippage` | number (percent) | no | Percent slippage. Forest uses its default when omitted. |

### `forest.swap.sell`

Submits a sell transaction through the connected wallet. `amount` is the amount of the launched token to sell.

```js
callForest("forest.swap.sell", {
	amount: "100",
	slippage: 0.5,
});
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `amount` | string (display decimal) | yes | Positive display decimal string of launched token to sell. |
| `slippage` | number (percent) | no | Percent slippage. Forest uses its default when omitted. |

## Snapshot Reference

Quote and swap responses can include `result.snapshot`. Store the latest snapshot and render your UI from it. The base snapshot always includes direction, slippage, pay token, and receive token. Quote and route fields are present only after Forest has enough data for them.

```js
{
  direction: "buy",
  slippage: 0.5,
  payToken: {
    symbol: "BNB",
    balance: 1.25,
    locked: 0
  },
  receiveToken: {
    symbol: "TOKEN"
  },
  quote: {
    payAmount: 0.05,
    receiveAmount: 1234.56,
    receivePerPay: 24691.2,
    payPerReceive: 0.0000405
  },
  priceImpact: 1.2,
  liquidityDepth: 8.7,
  swapBreakdown: {
    steps: [],
    totalEffectiveFee: 0.01,
    expectedOutput: 1234.56
  }
}
```

Snapshot values are display numbers, not base-unit integers.

### Top-level fields

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `direction` | "buy" \| "sell" | yes | Direction the quote was computed for. |
| `slippage` | number (percent) | yes | Percent slippage applied to the quote. |
| `payToken` | object | yes | Token being spent. Has symbol, balance, and locked sub-fields. |
| `receiveToken` | object | yes | Token being received. Has a symbol sub-field. |
| `quote` | object | no | Quoted amounts and rates. Present after a successful quote. Has payAmount, receiveAmount, receivePerPay, and payPerReceive sub-fields. |
| `priceImpact` | number | no | Price impact of the quote, when available. |
| `liquidityDepth` | number | no | Liquidity depth available for the quote, when available. |
| `swapBreakdown` | object | no | Route detail, when available. Has steps, totalEffectiveFee, and expectedOutput sub-fields. |

### `payToken` / `receiveToken`

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `symbol` | string | yes | Token symbol. |
| `balance` | number (display) | no | Wallet balance of the token in display units. Present on payToken. |
| `locked` | number (display) | no | Portion of the balance that is locked, in display units. Subtract from balance for spendable amount. Present on payToken. |

### `quote`

When `quote` is present, it has:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `payAmount` | number (display) | yes | Amount of the pay token spent, in display units. |
| `receiveAmount` | number (display) | yes | Amount of the receive token returned, in display units. |
| `receivePerPay` | number | yes | Receive token amount per unit of pay token. |
| `payPerReceive` | number | yes | Pay token amount per unit of receive token. |

### `swapBreakdown`

When `swapBreakdown` is present, it has:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `steps` | array | yes | Route steps for the swap. |
| `totalEffectiveFee` | number | yes | Total effective fee for the route. |
| `expectedOutput` | number (display) | yes | Expected output amount, in display units. |

Calculate spendable balance from the snapshot:

```js
const available = snapshot.payToken.balance - (snapshot.payToken.locked || 0);
```

## Error handling

Quote, buy, and sell failures use the shared swap error taxonomy. See [Errors and Security](./errors-security.md#swap-and-session-errors) for each code's cause and resolution.
