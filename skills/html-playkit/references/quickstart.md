# Quickstart

For a new game, use **New game** in Forest Harness. Its starter includes the frontend SDK and hosted backend setup. You can ask your assistant to make the following changes.

## Read Game Balance from a hosted backend

The backend uses Forest's verified player automatically:

```ts title="backend/index.ts"
import type { BackendContext } from "@forest/runtime";

export const runtime = "forest/2";
export const methods = Object.freeze(
	Object.assign(Object.create(null), {
		async balance(ctx: BackendContext) {
			return ctx.forest.game.getBalance();
		},
	})
);
```

Call that method from the frontend:

```ts title="frontend/index.ts"
import { forest } from "@forest/runtime";

const output = document.createElement("p");
document.body.append(output);
output.textContent = "Sign in to see your game balance.";

try {
	const result = await forest.backend.call<{ balance: string }>("balance", {});
	output.textContent = `Game balance: ${result.balance}`;
} catch (error) {
	output.textContent = error instanceof Error ? error.message : "Could not read your balance.";
}
```

The SDK waits for sign-in when needed. Dismissing sign-in rejects the call with `SIGN_IN_REQUIRED`; do not retry automatically.

Ask your assistant to prepare the game and start **Preview**. With **Test wallet** enabled, economic calls return simulated results. **Get ready to launch** runs the game checks; **Launch on forest.inc** opens the website for the wallet steps.

Next, add [persistent data](./hosted-backend.md) or a [game action](./game-actions.md). The hosted backend needs no signing secret or identity-code handshake.

## Legacy HTML uploads

The rest of this page is for an HTML file uploaded through the website. It uses the existing message bridge and `callForest` helper. Harness-built games should use `@forest/runtime`, which also handles their session and message nonce.

### Add the bridge

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
>
> `FOREST_WALLET_CONNECTED` carries a `walletAddress` for display only. Never send it to your
> backend as player identity — use the [Player Identity](./identity.md)
> flow instead. Error responses (`status: "error"`) carry an `error.code`; see the [error
> reference](./errors-security.md#wallet-and-rpc-errors) for the codes.

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
| `projectId` | string | yes | Active project id. Your trusted backend uses it in Forest API paths such as /playables/{projectId}/html/settlements. |
| `gameEntity` | string | yes | Entity type for this project. 'html' for HTML templates. |
| `autoSwapsEnabled` | boolean | yes | Whether the project owner enabled user-approved auto-swap sessions for this HTML project. |

> **INFO: Store projectId, never hardcode it**
>
> `projectId` is **not** a secret, but do not hardcode it. The same HTML file can be uploaded to
> different projects, and Forest provides the active id at runtime — store the id Forest sends.

> **INFO: Gate session UI on autoSwapsEnabled**
>
> `autoSwapsEnabled` tells the iframe whether the owner enabled user-approved auto-swap sessions.
> If it is `false`, do not render session controls.

## The core loop

Most templates follow the same swap loop. Game, session, burn, and identity flows layer on top — each is documented on its own page.

### Step 1: Request wallet state on load

Send `FOREST_REQUEST_WALLET` and store the incoming `FOREST_PROJECT_CONTEXT`.

### Step 2: Quote when the wallet connects

Call `forest.swap.quote` once the wallet connects, and re-quote whenever amount, direction, or slippage changes.

### Step 3: Submit on confirm

Call `forest.swap.buy` or `forest.swap.sell` when the user confirms.

### Step 4: Refresh after a swap

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
