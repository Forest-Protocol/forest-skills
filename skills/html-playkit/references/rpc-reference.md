# RPC Reference

## Harness frontend SDK

Import `{ forest }` from `@forest/runtime`. It sends the messages, validates the parent and nonce, waits for a session, and resolves each call with its final result.

| SDK method                                                     | Use                                             |
| -------------------------------------------------------------- | ----------------------------------------------- |
| `forest.backend.call(method, input)`                           | Call your hosted backend.                       |
| `forest.wallet.onChange(callback)` / `forest.wallet.request()` | Observe display wallet state or ask to connect. |
| `forest.game.balance()` / `.transactions(query)`               | Read the player's balance and activity.         |
| `forest.game.deposit({ amount })` / `.withdraw({ amount })`    | Wallet-approved deposits and withdrawals.       |
| `forest.game.action.authorize({ actionId, debitLimitAmount })` | Authorize one game action.                      |
| `forest.game.burn({ amount, burnAddress? })` / `.burns(query)` | Burn wallet tokens or read recorded burns.      |
| `forest.swap.quote(params)` / `.buy(params)` / `.sell(params)` | Quote or submit a wallet trade.                 |
| `forest.swap.session.create(params)` / `.current()`            | Enable or read an approved swap session.        |
| `forest.swap.session.get(sessionId)` / `.revoke(sessionId)`    | Read or revoke a particular session.            |

Game amounts and session budgets in frontend requests use display strings. Backend economic inputs use base units; see [amounts](./overview.md#amounts-display-vs-base-units).

Use `ctx.forest.game.getBurnSummary()` for a burn summary in a hosted backend. Legacy identity-code and `forest.runtime.invoke` RPCs are not exposed by the Harness SDK; use `ctx.player` and `forest.backend.call` instead.

## Legacy HTML transport

The remaining reference describes the legacy HTML SDK transport: the request and response envelopes, the `result.stage` progress values, the full list of exposed RPC methods, and the signed REST endpoints your backend calls. Topic-specific request and response shapes live on each topic page.

## RPC Request Format

Legacy HTML RPC calls use this request envelope. Harness games use the SDK above, which adds the required session nonce:

```js
{
  type: "FOREST_RPC_REQUEST",
  version: 1,
  id: "unique-request-id",
  method: "forest.swap.quote",
  params: {}
}
```

| Field     | Required | Description                                         |
| --------- | -------- | --------------------------------------------------- |
| `type`    | yes      | Always `FOREST_RPC_REQUEST`.                        |
| `version` | yes      | Use `1`.                                            |
| `id`      | yes      | Unique request id. Responses use the same id.       |
| `method`  | yes      | Forest method name.                                 |
| `params`  | yes      | Method-specific params. Use `{}` if there are none. |

## RPC Response Format

Forest replies with:

```js
{
  type: "FOREST_RPC_RESPONSE",
  version: 1,
  id: "same-request-id",
  status: "pending",
  result: {},
  error: undefined
}
```

| Status    | Meaning                                                                    |
| --------- | -------------------------------------------------------------------------- |
| `pending` | Forest accepted the request and is preparing, quoting, or submitting.      |
| `hash`    | A transaction hash is available, but the transaction is not confirmed yet. |
| `success` | The request completed. For swaps, the transaction receipt has confirmed.   |
| `error`   | The request failed. Read `error.code` and `error.message`.                 |

> **INFO: Error codes are documented centrally**
>
> On an `error` status, read `error.code` and `error.message`. Every code, what causes it, and how
> to recover is listed in the [Errors and
> Security](./errors-security.md) reference.

### Progress stages

Swap and session responses may include a `result.stage` value while Forest is preparing or submitting work. Treat these as progress updates — the request is done only on the final `success` (or `error`) response.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `accepted` | stage | no | Forest has accepted the request and is starting work. |
| `configured` | stage | no | Project config for the request has been resolved. |
| `warming` | stage | no | Forest is warming up the swap or session pipeline. |
| `quoting` | stage | no | A swap quote is being fetched. |
| `quoted` | stage | no | A swap quote is available. |
| `submitting` | stage | no | Forest is submitting the transaction. |
| `preparing` | stage | no | Forest is preparing the transaction for the user to sign. |
| `approving` | stage | no | A token approval is being requested before the main transaction. |
| `creating` | stage | no | The session create transaction is being prepared or sent. |
| `confirming` | stage | no | Forest is waiting for the transaction to confirm on-chain. |
| `loading_session` | stage | no | An existing auto-swap session is being loaded. |
| `revoking` | stage | no | A session revoke transaction is being prepared or sent. |
| `confirming_revoke` | stage | no | Forest is waiting for the revoke transaction to confirm on-chain. |

Session `hash` responses additionally carry one of these `*_submitted` stages so you can tell which transaction the hash belongs to:

| `result.stage`       | Transaction the hash belongs to            |
| -------------------- | ------------------------------------------ |
| `approval_submitted` | Token approval for the session.            |
| `create_submitted`   | The session (`createSession`) transaction. |
| `revoke_submitted`   | The session revoke transaction.            |

## Exposed RPC Methods

| Method                         | Wallet transaction | Purpose                                                                                                         |
| ------------------------------ | ------------------ | --------------------------------------------------------------------------------------------------------------- |
| `forest.swap.quote`            | no                 | Read a swap quote and token-page snapshot, or an executable session quote with `currency: "FOREST"`.            |
| `forest.swap.buy`              | yes                | Buy Project Token through the connected wallet.                                                                 |
| `forest.swap.sell`             | yes                | Sell Project Token through the connected wallet.                                                                |
| `forest.swap.session.create`   | yes                | Ask the user to enable an auto-swap session.                                                                    |
| `forest.swap.session.current`  | no                 | Read the latest active auto-swap session.                                                                       |
| `forest.swap.session.get`      | no                 | Read a confirmed auto-swap session.                                                                             |
| `forest.swap.session.revoke`   | yes                | Ask the user to revoke an active auto-swap session.                                                             |
| `forest.game.balance`          | no                 | Read the current player's Game Balance.                                                                         |
| `forest.game.deposit`          | yes                | Move wallet-held Project Token into Game Balance.                                                               |
| `forest.game.withdraw`         | yes                | Move Game Balance back to the connected wallet.                                                                 |
| `forest.game.burn`             | yes                | Transfer wallet-held Project Token on-chain to the signed sink (dead address by default); returns the `txHash`. |
| `forest.game.burns`            | no                 | List the current player's recorded burns (paginated).                                                           |
| `forest.game.burns.summary`    | no                 | Read the current player's burn totals, grouped by reason.                                                       |
| `forest.game.transactions`     | no                 | List the current player's Game Balance / Vault activity (deposits, claims, funding).                            |
| `forest.game.action.authorize` | no                 | Authorize one trusted-backend-settled game action.                                                              |
| `forest.identity.code`         | no                 | Issue a single-use code to resolve verified identity.                                                           |
| `forest.runtime.invoke`        | no                 | Invoke a declared backend capability on the Forest-hosted runtime (capability-gated projects only).             |

## REST endpoints

External backends for legacy HTML use these signed endpoints. Harness backends use `ctx.forest` instead. Signed/relayed endpoints under `/playables/{projectId}/html`. Full request and response detail lives on each topic page.

| Method | Path                                    | Caller               | Documented in                                                                                 |
| ------ | --------------------------------------- | -------------------- | --------------------------------------------------------------------------------------------- |
| POST   | `/action-authorizations`                | iframe (session)     | [Game Actions](./game-actions.md#forestgameactionauthorize)        |
| POST   | `/settlements`                          | backend (HMAC `v1=`) | [Game Actions](./game-actions.md#trusted-settlement)               |
| POST   | `/burns`                                | backend (HMAC `v1=`) | [Burns](./burns.md)                                                |
| POST   | `/identity/codes`                       | iframe (session)     | [Player Identity](./identity.md)                                   |
| POST   | `/identity/redeem`                      | backend (HMAC `v1=`) | [Player Identity](./identity.md#redeem-trusted-backend-only)       |
| POST   | `/swap-sessions/{sessionId}/executions` | backend (HMAC `v1=`) | [Swap Sessions](./swap-sessions.md#trusted-swap-session-execution) |
