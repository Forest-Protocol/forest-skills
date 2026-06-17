# Errors and Security

## Error Handling

Handle `error` responses as first-class UI states. The user may reject a wallet request, the quote may fail, or the token may not support swaps.

```js
if (data.status === "error") {
	showError(data.error.code, data.error.message);
}
```

This page is the reference for Forest HTML SDK error codes. Domain pages link here instead of repeating codes inline.

REST endpoints can also return normal HTTP validation errors without an app-specific `code` field, for example when a project or session is not found, auto-swaps are disabled, a route `sessionId` does not match the signed body, or a transaction fails chain validation. Treat those as terminal request failures and read the returned `message`.

## Wallet and RPC errors

Returned to the iframe over the RPC interface.

| Code                               | Cause                                                                                                  | Resolution                                                                                    |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| `WALLET_NOT_CONNECTED`             | The action needs a connected wallet and none is connected.                                             | Prompt the user to connect a wallet first.                                                    |
| `USER_REJECTED`                    | The user rejected a wallet request.                                                                    | Leave state unchanged and let the user retry explicitly.                                      |
| `TRANSACTION_FAILED`               | A submitted wallet transaction failed, reverted, or was replaced/underpriced.                          | Show the failure and let the user retry after checking wallet state.                          |
| `UNKNOWN_ERROR`                    | Forest could not map the failure to a more specific SDK code, or the RPC request envelope was invalid. | Log the message, verify the request shape, and show a generic retry state.                    |
| `INVALID_PARAMS`                   | The RPC params are missing or have the wrong shape.                                                    | Fix the params for the method (use `{}` when there are none).                                 |
| `GAME_BALANCE_FAILED`              | Forest could not load the player Game Balance.                                                         | Surface a transient error state and retry.                                                    |
| `GAME_TRANSACTIONS_FAILED`         | Forest could not load the player's Game Balance transaction feed.                                      | Surface a transient error state and retry.                                                    |
| `GAME_BURN_FAILED`                 | Forest could not complete the wallet-side burn transfer.                                               | Show the error, do not settle the burn, and let the user retry with a fresh intent if needed. |
| `GAME_WALLET_ACTION_IN_PROGRESS`   | A wallet-side game action is already in progress.                                                      | Wait for the in-flight action to finish before starting another.                              |
| `GAME_WALLET_UNAVAILABLE`          | Deposit or withdrawal is not configured for this game.                                                 | Hide wallet controls — the project has no wallet flow.                                        |
| `GAME_ACTION_AUTHORIZATION_FAILED` | Forest could not create an Action Authorization.                                                       | Retry with a fresh UUID v4 `actionId`.                                                        |

## Swap and session errors

| Code                    | Cause                                                                | Resolution                                                      |
| ----------------------- | -------------------------------------------------------------------- | --------------------------------------------------------------- |
| `INVALID_AMOUNT`        | The swap amount is missing, zero, negative, or not a finite decimal. | Ask the user for a positive display amount.                     |
| `QUOTE_FAILED`          | Forest could not quote the requested amount.                         | Retry, or adjust the amount.                                    |
| `SWAP_UNAVAILABLE`      | This token cannot be swapped through the current route.              | Disable swap UI for this token.                                 |
| `SWAP_IN_PROGRESS`      | A swap or session wallet flow is already in progress.                | Wait for the in-flight flow to finish before starting another.  |
| `CONFIG_TIMEOUT`        | Direction or slippage did not apply in time.                         | Re-apply the config and retry.                                  |
| `SWAP_SESSION_DISABLED` | Auto-swaps are disabled for this project.                            | Do not render session controls — `autoSwapsEnabled` is `false`. |
| `SWAP_SESSION_FAILED`   | Forest could not create or load the swap session.                    | Retry, or have the user recreate the session.                   |

## Settlement errors

Returned by the signed backend APIs for settlement and burns, and by the shared HMAC/timestamp checks used by identity redeem and swap-session execution. Swap-session execution can also return uncoded HTTP errors for session state or route validation.

| Code                                       | Cause                                                                                            | Resolution                                                                                    |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| `GAME_SETTLEMENT_SIGNING_SECRET_MISSING`   | The project has no active Settlement Signing Secret.                                             | Create a secret in the project editor's HTML Settlement card.                                 |
| `RAW_BODY_UNAVAILABLE`                     | Forest could not access the exact raw request body needed for HMAC verification.                 | Ensure the request is sent as a normal JSON body and contact Forest support if this persists. |
| `SETTLEMENT_SIGNATURE_MISSING`             | The `X-Forest-Settlement-Signature` header is missing.                                           | Send the signature header on every signed request.                                            |
| `SETTLEMENT_SIGNATURE_VERSION_UNSUPPORTED` | The header does not use the `v1=` format.                                                        | Prefix the hex HMAC with `v1=`.                                                               |
| `SETTLEMENT_SIGNATURE_MALFORMED`           | The signature is not a 64-character hex HMAC.                                                    | Hex-encode the full HMAC-SHA256 digest.                                                       |
| `INVALID_SETTLEMENT_SIGNATURE`             | The request body does not match the signature.                                                   | Sign the exact bytes you send — do not reserialize the body after signing.                    |
| `ACTION_AUTHORIZATION_UNAVAILABLE`         | The action was not authorized, was already used, or expired.                                     | Re-authorize with a fresh `actionId`, created close to the action.                            |
| `SETTLEMENT_TIMESTAMP_STALE`               | The signed `timestamp` is outside the request-freshness window (~5 min).                         | On retry, replay the **original** signed body — never re-sign with a new timestamp.           |
| `ACTION_ID_REUSED`                         | The same `actionId` was reused with a different request body.                                    | Resend the byte-identical body, or use a new `actionId` for a new action.                     |
| `INVALID_SETTLEMENT_AMOUNT`                | The amounts are invalid (e.g. negative, or both `0` for a settlement).                           | Send non-negative base-unit integer strings; at least one of debit/credit `> 0`.              |
| `INVALID_SETTLEMENT_METADATA`              | Burn `metadata` exceeds the serialized size limit.                                               | Keep signed metadata under 4 KB.                                                              |
| `DEBIT_LIMIT_EXCEEDED`                     | The amount exceeds the authorized debit limit.                                                   | Authorize a debit limit at least the amount (never `0` for a burn).                           |
| `BURN_TXHASH_REQUIRED`                     | A burn settlement omitted the required on-chain `txHash`.                                        | Submit burns only after `forest.game.burn` returns a confirmed transaction hash.              |
| `BURN_TX_HASH_REUSED`                      | The `txHash` is already recorded under a different `actionId`, or already recorded as a deposit. | Do not replay an old burn — each `txHash` backs exactly one burn.                             |

> **INFO: Two authorization codes, not one**

`GAME_ACTION_AUTHORIZATION_FAILED` (RPC) means the *authorize* step could not create an
authorization. `ACTION_AUTHORIZATION_UNAVAILABLE` (settlement) means the authorization was
missing, already used, or expired at *settle* time. They are distinct stages — do not treat them
as the same error.

## Identity errors

| Code                     | Cause                                                               | Resolution                                                                |
| ------------------------ | ------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| `IDENTITY_CODE_FAILED`   | Forest could not issue an identity code through the iframe RPC.     | Ensure the player is connected and retry with a fresh nonce.              |
| `IDENTITY_NO_WALLET`     | The current Forest session has no wallet bound when issuing a code. | Ask the player to reconnect their wallet, then restart the identity flow. |
| `IDENTITY_CODE_INVALID`  | The backend tried to redeem an unknown identity code.               | Restart the identity flow and redeem the fresh code server-to-server.     |
| `IDENTITY_CODE_CONSUMED` | The identity code was already redeemed (codes are single-use).      | Do not retry redeem — issue a fresh code by restarting at login.          |
| `IDENTITY_CODE_EXPIRED`  | The identity code expired before redemption.                        | Issue a fresh code and redeem it immediately.                             |

## Security Notes

> **WARN: Treat the iframe boundary as untrusted**

- Verify the `type` and shape of every incoming message before acting on it.
- Keep request ids unique.
- Do not assume your iframe can access parent-page storage, wallet providers, cookies, or settlement secrets.
- Forest-owned browser actions go through this message interface; credit-producing settlement must go through a trusted backend.

The `FOREST_WALLET_CONNECTED` address is display-only and forgeable by the player. Never use it for backend identity, anti-abuse, or accounting — resolve the player through the [Player Identity](./identity.md) flow instead. Credit always lands on the Forest player who authorized the action; your backend cannot redirect it to an arbitrary recipient.
