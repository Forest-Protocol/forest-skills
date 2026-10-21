# Swaps

Swaps run through the connected wallet. Quote is a read; buy and sell open the wallet. Amounts are display decimal strings — see [Amounts](./overview.md#amounts-display-vs-base-units). For user-approved auto-swaps, see [Swap Sessions](./swap-sessions.md).

For Harness games, import `{ forest }` from `@forest/runtime` and await the calls below. Legacy HTML uploads pass the same params to `callForest("forest.swap.quote", params)` and the other named methods.

## Swap Methods

### `forest.swap.quote`

Gets a read-only quote and snapshot. This does not open the wallet or submit a transaction.

```js
await forest.swap.quote({
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
| `currency` | "FOREST" | no | Returns an executable swap session quote instead of the wallet quote. `amount` must then be an exact decimal string. |

### Session quotes

Pass `currency: "FOREST"` to price a [swap session](./swap-sessions.md) trade. Forest reads the market at one block and does not change the swap widget.

```js
const { sessionQuote, snapshot } = await forest.swap.quote({
	amount: "1",
	direction: "buy",
	currency: "FOREST",
	slippage: 1,
});
```

For a buy, `amount` is the most FOREST to spend, including slippage. For a sell, `amount` is the exact Project Token amount to sell. Its decimals cannot exceed the pay token's decimals. BNB amounts are not supported.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `projectAmount` | string (base-unit integer) | yes | Pass as `projectAmount`. Exact output for buys; exact input for sells. |
| `forestAmountLimit` | string (base-unit integer) | yes | Pass as `forestAmountLimit`. Maximum FOREST spent for buys; minimum FOREST received for sells. |
| `forestAmount` | string (base-unit integer) | yes | Expected FOREST spent or received, including fees. |
| `expiresAt` | number (Unix seconds) | yes | Quote expiry, 90 seconds after the quoted block. Pass as `deadline`. |
| `direction` | "buy" \| "sell" | yes | Direction the quote was computed for. |
| `chainId, blockNumber, quotedAt` | number, string, number | yes | Chain, block, and block timestamp the quote was read at. |
| `forestToken, projectToken` | string (address) | yes | Token addresses the quote applies to. |
| `forestDecimals, projectDecimals` | number | yes | Token decimals for converting the base-unit amounts. |

`snapshot` carries the same amounts as display strings for UI. Send `sessionQuote` to your backend, which passes its amounts and `expiresAt` to `executeSwapSession`. Fetch a new quote after `expiresAt`.

### `forest.swap.buy`

Submits a buy transaction through the connected wallet. `amount` is the amount of the pay token to spend.

```js
await forest.swap.buy({
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
await forest.swap.sell({
	amount: "100",
	slippage: 0.5,
});
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `amount` | string (display decimal) | yes | Positive display decimal string of launched token to sell. |
| `slippage` | number (percent) | no | Percent slippage. Forest uses its default when omitted. |

## Snapshot Reference

The SDK's returned object can include `snapshot`; legacy message responses expose it as `result.snapshot`. Store the latest snapshot and render your UI from it. The base snapshot always includes direction, slippage, pay token, and receive token. Quote and route fields are present only after Forest has enough data for them.

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
