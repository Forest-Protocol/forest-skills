# Game Actions

Game Actions are walletless in-game actions. A player first creates a short-lived Action Authorization, then your trusted backend validates the gameplay and submits the settlement to Forest. Harness-built games use `ctx.forest.settleAction` in their hosted backend. Legacy HTML uploads use an external backend with a Settlement Signing Secret.

Forest web uses the player's active Forest session behind the iframe boundary to authorize player-scoped game requests. Use the frontend SDK for authorization; it handles the Forest session for you. Authorization amounts are display strings, settlement amounts are base units — see [Amounts](./overview.md#amounts-display-vs-base-units).

### `forest.game.action.authorize`

Creates a one-use Action Authorization for a developer-provided `actionId`.

```js
import { forest } from "@forest/runtime";

const actionId = crypto.randomUUID();

await forest.game.action.authorize({
	actionId,
	debitLimitAmount: "1",
});
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `actionId` | string (UUID v4) | yes | UUID v4 for this one game action. Use crypto.randomUUID() if possible. |
| `debitLimitAmount` | string (display amount) | yes | Maximum player-authorized amount for this action — the cap for both a settlement's Game Balance debit and a burn's amount. For a burn, set this to at least the burn amount (do not use 0 — the burn will fail DEBIT_LIMIT_EXCEEDED). Use 0 only for actions that neither debit balance nor burn. |

Successful responses return:

```js
{
  id: "authorization-id",
  actionId: "same-action-id",
  debitLimitAmount: "1",
  expiresAt: "2026-06-01T12:05:00.000Z"
}
```

Send the same `actionId` and the player's gameplay input to your hosted backend with `forest.backend.call`. Legacy HTML uploads use `callForest("forest.game.action.authorize", params)` and their own backend endpoint. Do not ask the browser to choose the final payout.

Create the Action Authorization close to the actual game action. It is short-lived, and Forest returns `expiresAt` so your UI can reject stale attempts before calling your backend.

## Trusted Settlement

Forest handles settlement accounting and security checks. Your trusted backend handles game outcome validation.

### Hosted backend

This small example always charges one token and returns half a token. The backend fixes both amounts; the caller only supplies the action ID.

```ts title="backend/index.ts"
import type { BackendContext } from "@forest/runtime";

export const runtime = "forest/2";
export const methods = Object.freeze(
	Object.assign(Object.create(null), {
		async play(ctx: BackendContext) {
			const input = ctx.input;
			if (
				!input ||
				typeof input !== "object" ||
				!("actionId" in input) ||
				typeof input.actionId !== "string"
			) {
				throw new Error("An action ID is required.");
			}
			return ctx.forest.settleAction({
				actionId: input.actionId,
				debitAmount: "1000000000000000000",
				creditAmount: "500000000000000000",
			});
		},
	})
);
```

After the frontend authorization above succeeds:

```js
const result = await forest.backend.call("play", { actionId });
const balance = await forest.game.balance();
```

The player needs one token in Game Balance, and the game needs enough vault capacity for the credit. The settlement result contains display strings such as `debitAmount: "1.0"` and `creditAmount: "0.5"`.

Forest supplies authentication and the timestamp. Keep each action ID and its confirmed outcome in your game data. If the result is uncertain, stop automatic retries: a later backend invocation receives a new timestamp, so calling it again is not a byte-identical settlement replay.

### External backend for legacy HTML

The remaining signing instructions apply to legacy HTML uploads whose backend runs outside Forest. That backend validates the game outcome and signs its settlement request.

### Settlement Signing Secret

### Step 1: Obtain it once

Forest creates the first Settlement Signing Secret when an HTML project launch
completes and shows it **once** on the congratulations screen. Copy the backend
env block into the trusted backend that signs settlement requests. The same Forest
API base URL is also shown later in the project editor's HTML Settlement card.

```bash
FOREST_API_BASE_URL=__FOREST_API_BASE_URL__
SETTLEMENT_SECRET=fsc_live_...
```

### Step 2: Read its status later

After that, the project editor shows only secret status metadata: active state,
prefix, last 4 characters, and creation time. It cannot reveal the existing
plaintext secret again.

### Step 3: Create or rotate

Use **Create Secret** when no active secret exists. Use **Rotate Secret** to
replace the active secret. Rotation invalidates the previous backend secret and
returns the new plaintext value once.

> **INFO: The secret is server-only**
>
> Never put the Settlement Signing Secret in uploaded HTML, frontend JavaScript, app-template
> code, or any browser-accessible environment variable. Forest stores settlement secrets encrypted
> at rest and uses the active project secret only to verify settlement HMAC signatures — a
> database row holds encrypted secret material plus display metadata, not the plaintext value.

### Settlement request

`POST /playables/{projectId}/html/settlements`

```http
POST /playables/{projectId}/html/settlements
Content-Type: application/json
X-Forest-Settlement-Signature: v1=<hex-hmac>
```

```js
{
	actionId: "same-action-id",
	debitAmount: "1000000000000000000",
	creditAmount: "2000000000000000000",
	timestamp: Math.floor(Date.now() / 1000)
}
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `actionId` | string (UUID v4) | yes | Same UUID v4 used for forest.game.action.authorize. |
| `debitAmount` | string (base-unit integer) | yes | Non-negative Project Token base-unit integer string. Must not exceed authorized debit base units. |
| `creditAmount` | string (base-unit integer) | yes | Non-negative Project Token base-unit integer string. |
| `timestamp` | number (Unix seconds) | yes | Current Unix timestamp in seconds. Generate it when sending the request. |

At least one of `debitAmount` or `creditAmount` must be greater than `0`.

Node.js signing example:

```js
import crypto from "node:crypto";

const body = JSON.stringify({
	actionId,
	debitAmount: "1000000000000000000",
	creditAmount: "2000000000000000000",
	timestamp: Math.floor(Date.now() / 1000),
});

// Sign the exact string you send as the HTTP request body.
const signature = crypto
	.createHmac("sha256", process.env.SETTLEMENT_SECRET)
	.update(body)
	.digest("hex");

await fetch(`${FOREST_API}/playables/${projectId}/html/settlements`, {
	method: "POST",
	headers: {
		"Content-Type": "application/json",
		"X-Forest-Settlement-Signature": `v1=${signature}`,
	},
	body,
});
```

Successful settlement responses return:

```js
{
  id: "settlement-id",
  actionId: "same-action-id",
  debitAmount: "1",
  creditAmount: "2",
  replayed: false
}
```

Settlement responses return display decimal amounts. Settlement requests use base-unit integer strings because they are signed server-to-server API bodies.

> **INFO: Retrying a settlement**
>
> If a settlement request times out or the response is lost, retry by sending the **same**
> `actionId`, JSON body, and signature again. Do **not** create a new timestamp or change
> settlement amounts for the same `actionId`.

Settlement failures use the shared [settlement error taxonomy](./errors-security.md#settlement-errors) (signature, authorization, timestamp, amount, and reuse errors).

> **WARN: Only a trusted server can settle credit**
>
> Keep final debit and credit decisions in the backend. A Harness game uses its hosted backend; an
> external backend for legacy HTML keeps its Settlement Signing Secret server-side.

If your HTML game calls your trusted backend directly from the browser, normal browser origin rules still apply. No CORS setup is needed when the game and backend share the same origin. If they are on different origins, your backend must allow the game document's origin. Forest's iframe proxy relays SDK messages; it does not proxy arbitrary `fetch` calls to your backend.
