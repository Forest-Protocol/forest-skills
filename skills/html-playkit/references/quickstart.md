# Quickstart

This gets a Forest HTML template talking to the parent page: request the wallet, handle Forest's messages, and fire your first quote. For the concepts behind it, see the [Overview](./overview.md).

## Add the bridge

Drop this into your HTML app. It sends RPC requests with `callForest(...)` and routes Forest's messages back to your code.

```js
let wallet = null;
let projectId = null;
let autoSwapsEnabled = false;
let latestSnapshot = null;

function requestId() {
	if (crypto.randomUUID) return crypto.randomUUID();
	return `${Date.now()}-${Math.random().toString(16).slice(2)}`;
}

function callForest(method, params = {}) {
	const id = requestId();

	window.parent.postMessage(
		{
			type: "FOREST_RPC_REQUEST",
			version: 1,
			id,
			method,
			params,
		},
		"*"
	);

	return id;
}

window.addEventListener("message", (event) => {
	if (event.source !== window.parent) return;

	const data = event.data || {};

	if (data.type === "FOREST_WALLET_CONNECTED") {
		// Display-only. Never send this to your backend as identity — see Player Identity.
		wallet = data.walletAddress;
		callForest("forest.game.balance");
		callForest("forest.swap.quote", { amount: "0.05", direction: "buy" });
		return;
	}

	if (data.type === "FOREST_WALLET_DISCONNECTED") {
		wallet = null;
		latestSnapshot = null;
		return;
	}

	if (data.type === "FOREST_PROJECT_CONTEXT" && typeof data.projectId === "string") {
		projectId = data.projectId;
		autoSwapsEnabled = data.autoSwapsEnabled === true;
		return;
	}

	if (data.type !== "FOREST_RPC_RESPONSE") return;

	if (data.result && data.result.snapshot) {
		latestSnapshot = data.result.snapshot;
	}

	if (data.result && data.result.balance) {
		console.log("Game balance:", data.result.balance);
	}

	if (data.status === "hash") {
		console.log("Transaction submitted:", data.result && data.result.txHash);
	}

	if (data.status === "success") {
		console.log("Forest request complete:", data.result && data.result.txHash);
	}

	if (data.status === "error") {
		console.error(data.error && data.error.code, data.error && data.error.message);
	}
});

window.parent.postMessage({ type: "FOREST_REQUEST_WALLET" }, "*");
```

> **INFO: The wallet address is display-only**

`FOREST_WALLET_CONNECTED` carries a `walletAddress` for display only. Never send it to your
backend as player identity — use the [Player Identity](./identity.md) flow
instead. Error responses (`status: "error"`) carry an `error.code`; see the [error
reference](./errors-security.md#wallet-and-rpc-errors) for the codes.

## Project context

On load, Forest sends the current project context to your iframe:

```js
{
  type: "FOREST_PROJECT_CONTEXT",
  projectId: "project-id",
  gameEntity: "html",
  autoSwapsEnabled: false
}
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `projectId` | string | yes | Active project id. Your trusted backend uses it in Forest API paths such as /campaigns/{projectId}/html/settlements. |
| `gameEntity` | string | yes | Entity type for this project. 'html' for HTML templates. |
| `autoSwapsEnabled` | boolean | yes | Whether the project owner enabled user-approved auto-swap sessions for this HTML project. |

> **INFO: Store projectId, never hardcode it**

`projectId` is **not** a secret, but do not hardcode it. The same HTML file can be uploaded to
different projects, and Forest provides the active id at runtime — store the id Forest sends.

> **INFO: Gate session UI on autoSwapsEnabled**

`autoSwapsEnabled` tells the iframe whether the owner enabled user-approved auto-swap sessions.
If it is `false`, do not render session controls.

## The core loop

Most templates follow the same swap loop. Game, session, burn, and identity flows layer on top — each is documented on its own page.

### Step 1
### Request wallet state on load

Send `FOREST_REQUEST_WALLET` and store the incoming `FOREST_PROJECT_CONTEXT`.

### Step 2
### Quote when the wallet connects

Call `forest.swap.quote` once the wallet connects, and re-quote whenever amount, direction, or slippage changes.

### Step 3
### Submit on confirm

Call `forest.swap.buy` or `forest.swap.sell` when the user confirms.

### Step 4
### Refresh after a swap

On `success`, request another quote so wallet balances update:

```js
if (response.status === "success") {
	setTimeout(() => {
		callForest("forest.swap.quote", {
			amount: currentAmount,
			direction: currentDirection,
		});
	}, 750);
}
```

From here:

- [Swaps](./swaps.md) — Buy, sell, and quote details.
- [Swap Sessions](./swap-sessions.md) — User-approved auto-swaps.
- [Game Balance](./game-balance.md) — Deposits, withdrawals, transactions.
- [Game Actions](./game-actions.md) — Trusted backend settlement.
