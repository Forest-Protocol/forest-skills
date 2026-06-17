# Player Identity

## Project and Wallet Messages

Forest sends project and wallet messages to your iframe. Your app can also request the latest wallet state.

| Message                      | Direction        | Purpose                                                           |
| ---------------------------- | ---------------- | ----------------------------------------------------------------- |
| `FOREST_PROJECT_CONTEXT`     | Forest to iframe | Provides the active Forest project id and auto-swap availability. |
| `FOREST_REQUEST_WALLET`      | iframe to Forest | Ask Forest to send the current wallet state.                      |
| `FOREST_WALLET_CONNECTED`    | Forest to iframe | Display-only connected wallet address. See the warning below.     |
| `FOREST_WALLET_DISCONNECTED` | Forest to iframe | Tells your app that no wallet is connected.                       |

Request the current wallet state:

```js
window.parent.postMessage({ type: "FOREST_REQUEST_WALLET" }, "*");
```

Handle wallet state:

```js
window.addEventListener("message", (event) => {
	if (event.source !== window.parent) return;

	const data = event.data || {};

	if (data.type === "FOREST_WALLET_CONNECTED") {
		wallet = data.walletAddress;
	}

	if (data.type === "FOREST_WALLET_DISCONNECTED") {
		wallet = null;
	}
});
```

Only the project id and public wallet address are exposed. Forest does not send auth tokens, private keys, cookies, or session data to the iframe.

> **WARN: FOREST_WALLET_CONNECTED is display-only**

The address is fine for rendering (greetings, avatars), but it is **not** an authenticated
credential. Once it reaches your uploaded HTML it is a plain string the player can rewrite in
their own browser and report to your backend. **Never use this address for backend identity or
accounting.** To learn the verified player identity, use the Player Identity flow below.

## Player Identity

When your trusted backend needs to know _which_ Forest player is in the session — to key accounting, anti-abuse, or per-player state — do not trust the wallet message. Run a one-time server-to-server handshake instead. The identity code carries no data and grants no spend authority; it is only a lookup key your backend redeems with the same Settlement Signing Secret HMAC it already uses for settlement.

### The identity handshake

### Step 1
### Mint a single-use nonce (your backend)

Your backend mints a single-use nonce for this login attempt and stores it. The nonce binds the identity to one login transaction and is verified on redeem.

### Step 2
### Request a code from the iframe

The iframe calls `forest.identity.code({ nonce })` with that nonce. Forest returns `{ code, expiresAt }`.

### Step 3
### Relay the code to your backend

The iframe relays the opaque `code` to your backend. The code carries no identity data — do not log it or put it in URLs.

### Step 4
### Redeem the code (your backend)

Your backend redeems the code against Forest with the HMAC-signed request, getting back `{ userId, walletAddress, nonce, issuedAt }`.

### Step 5
### Assert the nonce matches

Your backend asserts the returned `nonce` equals the unconsumed nonce you minted for this login, then burns it.

### Step 6
### Mint your session

Only after the nonce check passes does your backend mint its own session, keyed on the verified `userId`.

### `forest.identity.code`

Issues a short-lived (60s), single-use identity code for the authenticated player session.

```js
callForest("forest.identity.code", {
	// Required. A single-use nonce your backend minted for this login attempt and
	// will verify is echoed on redeem. Binds the identity to one login transaction.
	nonce,
});
// The { code, expiresAt } response arrives via your FOREST_RPC_RESPONSE listener.
```

The code is opaque and contains no identity data. Relay it to your backend; do not log it or put it in URLs.

### Redeem (trusted backend only)

```http
POST /campaigns/{projectId}/html/identity/redeem
X-Forest-Settlement-Signature: v1=<hmac-sha256 of the raw body>
```

Request fields:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `code` | string (opaque) | yes | The opaque code relayed from the iframe. |
| `timestamp` | number (Unix seconds) | yes | Unix seconds. Same freshness window as settlement. |

Sign and send exactly like a settlement (reuse your settlement signing helper):

```js
const body = JSON.stringify({ code, timestamp: Math.floor(Date.now() / 1000) });
const signature = `v1=${createHmac("sha256", SETTLEMENT_SIGNING_SECRET).update(body).digest("hex")}`;

const res = await fetch(`${FOREST_API}/campaigns/${projectId}/html/identity/redeem`, {
	method: "POST",
	headers: { "Content-Type": "application/json", "X-Forest-Settlement-Signature": signature },
	body,
});
// { userId, walletAddress, nonce, issuedAt }
```

Rules that matter:

- Derive `{projectId}` from your own server config, never from the frontend-relayed value.
- Assert the returned `nonce` equals the unconsumed nonce you minted for this login, then burn it.
- Key accounting on `userId` (immutable). Treat `walletAddress` as a verified display/payout reference.
- The code is single-use. A second redeem is rejected and the code must not be cached. See [Identity errors](./errors-security.md#identity-errors).
- If the redeem response is lost (network blip after Forest consumed the code), do not retry the redeem — it will fail as consumed. Recover by issuing a fresh code (restart at login).
