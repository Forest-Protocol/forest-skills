# Swap Sessions

Swap sessions are user-approved auto-swap permissions. The iframe asks the user to enable a session; your trusted backend executes against it. They are available only when the project owner has enabled auto-swaps for the HTML project (`autoSwapsEnabled` in [project context](./quickstart.md#project-context)).

> **WARN: Gate session UI on availability**

If `autoSwapsEnabled` is `false`, do **not** render session controls — execution will be
rejected with `SWAP_SESSION_DISABLED`.

## How sessions work

### Step 1
### Enable (create)

The iframe requests a session with display-decimal budgets. Forest converts them
into contract base units, prepares the required token approvals for the session
executor contract, and prepares the session transaction. The user confirms the
wallet prompts; Forest then confirms the `createSession` transaction and stores
the active session.

### Step 2
### Execute (backend)

Session execution is a backend-to-Forest API flow. Forest uses the user's managed
executor wallet to send the on-chain execution transaction. HTML templates do not
configure executor wallet addresses — Forest owns the executor-wallet setup and
confirmation flow.

### Step 3
### Revoke

Users can revoke active sessions from the Forest UI or via
`forest.swap.session.revoke`.

> **WARN: Never cache the session id**

Before **every** backend execution, request `forest.swap.session.current` and use that returned
`session.id`. Do **not** execute against a session id restored from `localStorage` or another
stale client cache.

> **INFO: Approval hashes are progress, not completion**

The create flow may emit `hash` responses with `stage: "approval_submitted"` before the final
`stage: "create_submitted"` hash. Treat approval hashes as progress updates; the session is
active only after the final `success` response.

> **INFO: Executor wallet needs gas**

The managed executor wallet must hold native gas on the configured chain for executions to land.

## Session Methods

### `forest.swap.session.create`

```js
callForest("forest.swap.session.create", {
	forestBudget: "50",
	projectBudget: "1000.5",
	expiresAt: Math.floor(Date.now() / 1000) + 86400,
});
```

Unlimited budget:

```js
callForest("forest.swap.session.create", {
	forestBudget: null,
	projectBudget: null,
	expiresAt: Math.floor(Date.now() / 1000) + 86400,
});
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `forestBudget` | string \| null | yes | Max FOREST display amount the session may use for buys, or null for unlimited. |
| `projectBudget` | string \| null | yes | Max Project Token display amount the session may use for sells, or null for unlimited. |
| `expiresAt` | number (Unix seconds) | yes | When the session expires. |

Budgets are display decimal strings. Use `null` for a side that should be unlimited until `expiresAt` is reached or the user revokes the session. Successful responses return the confirmed session plus the transaction hash:

```js
{
  session: {
    id: "9db7f2d8-5e03-4c73-96f9-31b99747d0c6",
    onChainSessionId: "42",
    chainId: 97,
    status: "active",
    snapshot: {
      routerAddress: "0x...",
      forestBudget: "50000000000000000000",
      projectBudget: "1000500000000000000000",
      expiresAt: "2026-05-15T12:00:00.000Z",
      createTxHash: "0x...",
      createdAt: "2026-05-14T12:00:00.000Z"
    },
    createdAt: "2026-05-14T12:00:00.000Z",
    updatedAt: "2026-05-14T12:00:00.000Z"
  },
  limits: {
    forestBudget: "50000000000000000000",
    projectBudget: "1000500000000000000000",
    expiresAt: "2026-05-15T12:00:00.000Z"
  },
  txHash: "0x..."
}
```

### `forest.swap.session.current`

```js
callForest("forest.swap.session.current");
```

Returns the latest active session for the current project and authenticated wallet user, using the same response shape as `forest.swap.session.get`. Use this method immediately before passing a session id to your backend for execution.

### `forest.swap.session.get`

```js
callForest("forest.swap.session.get", {
	sessionId: "9db7f2d8-5e03-4c73-96f9-31b99747d0c6",
});
```

Returns the stored session so your app can inspect status, limits, and expiry. Forest resolves router and pair addresses internally; HTML apps never provide or persist those addresses.

> **WARN: Sessions do not re-route**

Sessions do not fall back to a new route after router or pair changes. If the on-chain session
route is no longer usable, the user must recreate the session.

```js
{
  limits: {
    forestBudget: "50000000000000000000",
    projectBudget: null,
    expiresAt: "2026-05-15T12:00:00.000Z"
  },
  session: {
    id: "9db7f2d8-5e03-4c73-96f9-31b99747d0c6",
    onChainSessionId: "42",
    chainId: 97,
    status: "active",
    snapshot: {
      routerAddress: "0x...",
      forestBudget: "50000000000000000000",
      projectBudget: null,
      expiresAt: "2026-05-15T12:00:00.000Z",
      createTxHash: "0x...",
      createdAt: "2026-05-14T12:00:00.000Z"
    },
    createdAt: "2026-05-14T12:00:00.000Z",
    updatedAt: "2026-05-14T12:00:00.000Z"
  }
}
```

HTML SDK session responses use base-unit integer strings for budgets. `limits` mirrors `session.snapshot` budget fields and exists only as a convenient place to read the configured session caps and expiry.

### `forest.swap.session.revoke`

```js
callForest("forest.swap.session.revoke", {
	sessionId: "9db7f2d8-5e03-4c73-96f9-31b99747d0c6",
});
```

Revokes an active session through the connected wallet, then confirms the revoke transaction with Forest. This opens a wallet prompt and returns the same response shape as `forest.swap.session.get`, plus `txHash` when the revoke transaction is confirmed.

## Trusted Swap Session Execution

Uploaded HTML can request that a user enables a session, but execution must come from your trusted backend. Use the same server-only Settlement Signing Secret and HMAC header as settlement:

```http
POST /campaigns/{projectId}/html/swap-sessions/{sessionId}/executions
Content-Type: application/json
X-Forest-Settlement-Signature: v1=<hex-hmac>
```

```js
{
	sessionId: "9db7f2d8-5e03-4c73-96f9-31b99747d0c6",
	requestId: "6ae1ea37-19a2-46db-a59f-742e58fb8c4d",
	direction: "buy",
	projectAmount: "1000000000000000000",
	forestAmountLimit: "1000000000000000000",
	timestamp: Math.floor(Date.now() / 1000)
}
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `sessionId` | string (UUID v4) | yes | Session ID being executed. Must match the route session ID. |
| `requestId` | string (UUID v4) | yes | UUID v4 generated by your backend for idempotency. Retry the same body for the same request. |
| `direction` | "buy" \| "sell" | yes | Swap direction. |
| `projectAmount` | string (base-unit integer) | yes | Project Token amount as a base-unit integer string. Buy treats it as exact output; sell as exact input. |
| `forestAmountLimit` | string (base-unit integer) | yes | FOREST amount as a base-unit integer string. Buy treats it as max input; sell as min output. |
| `timestamp` | number (Unix seconds) | yes | Current Unix timestamp in seconds. |
| `deadline` | number (Unix seconds) | no | Optional router deadline. Forest defaults it when omitted. |

Sign the exact JSON string you send. Store the signed body by `sessionId + requestId` and replay that same body if the response is lost. Reusing a request id with a different direction, amount, or deadline is rejected.

A successful response contains the confirmed execution transaction:

```js
{
  id: "execution-id",
  sessionId: "9db7f2d8-5e03-4c73-96f9-31b99747d0c6",
  requestId: "same-request-id",
  direction: "buy",
  txHash: "0x...",
  projectAmount: "1000000000000000000",
  forestAmount: "100000000000000000",
  forestBudgetRemaining: "49000000000000000000",
  projectBudgetRemaining: "100000000000000000000",
  replayed: false
}
```

Forest records confirmed executions only after the contract emits the matching `SwapSessionExecuted` event. If your backend loses the response, retry the same `requestId` and exact signed body. Signature and timestamp failures use the shared [settlement error taxonomy](./errors-security.md#settlement-errors); session state, route mismatches, and chain-validation failures can return normal HTTP errors without an app-specific `code`.

Embedded HTML apps can revoke sessions with `forest.swap.session.revoke`; users can also revoke active sessions from Forest UI.
