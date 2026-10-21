# Hosted Backend

Forest Harness games keep trusted rules in a Forest-hosted backend. The frontend calls a named method through `forest.backend.call(method, input)`. Forest verifies the signed-in player before running it.

## Backend methods

The backend bundle exports only `runtime` and `methods`. The methods object must be frozen, have a null prototype, and contain its own functions. The Harness starter already uses this shape.

```ts title="backend/index.ts"
import type { BackendContext } from "@forest/runtime";

export const runtime = "forest/2";
export const methods = Object.freeze(
	Object.assign(Object.create(null), {
		async inventory(ctx: BackendContext) {
			const result = await ctx.db.execute(
				"SELECT id FROM inventory WHERE player_id = ? ORDER BY id",
				[ctx.player.id]
			);
			return result.rows;
		},
		async add(ctx: BackendContext) {
			await ctx.db.execute("INSERT INTO inventory (player_id) VALUES (?)", [ctx.player.id]);
			return { added: true };
		},
	})
);
```

Create the table before using those methods:

```sql title="migrations/0001_init.sql"
CREATE TABLE inventory (id INTEGER PRIMARY KEY, player_id TEXT NOT NULL);
CREATE INDEX inventory_player ON inventory (player_id);
```

Call the backend from the frontend:

```js
import { forest } from "@forest/runtime";

await forest.backend.call("add", {});
const items = await forest.backend.call("inventory", {});
```

This example lets each player add an item. A real game should validate the action and its cost before granting valuable items.

## Available context

| Value                       | Purpose                                                                  |
| --------------------------- | ------------------------------------------------------------------------ |
| `ctx.input`                 | JSON supplied by the caller. Validate it before use.                     |
| `ctx.player.id`             | Verified Forest player ID. Use it for private data and ownership checks. |
| `ctx.player.walletAddress`  | Verified wallet address, or `null`.                                      |
| `ctx.token`                 | The game's token: `id`, `symbol`, `decimals`, and `address`.             |
| `ctx.db.execute(sql, args)` | Run one parameterized SQL statement. Returns `rows` and `rowsAffected`.  |
| `ctx.db.batch(statements)`  | Run an atomic group of `{ sql, args }` statements.                       |
| `ctx.forest`                | Settlements, burns, approved swap execution, and economic reads.         |
| `ctx.log(message, data?)`   | Record bounded game diagnostics. Keep private data out of logs.          |

`ctx.db.execute({ sql, args })` is also supported. Results and method returns must be JSON; serialize large integers as strings.

Each invocation has fresh globals. Bundle dependencies into the backend: it cannot load runtime imports, use Node globals, read files or environment variables, or make arbitrary network requests.

## Schema changes and persistence

Set `database: true` and `backend: true` in `forest.json`. Put ordered schema files in `migrations/`. Forest applies them before the game can launch; Preview applies them to its local database.

- Supported schema operations are `CREATE TABLE`, `CREATE INDEX`, and `ALTER TABLE ADD COLUMN`.
- Applied files must stay unchanged. Add a new file for a schema change.
- Runtime queries can select, insert, update, and delete data, but cannot create or alter schema.
- Use parameters for values and database constraints for uniqueness. Explicit transaction statements are unavailable; use `ctx.db.batch` for atomic writes.
- Forest's migration history and names beginning with `_forest_` or `sqlite_` are reserved.
- Keep ordinary Preview data when editing the game. An incompatible edit to an applied schema file requires an explicit local data reset.

Private and shared data can use the same database. Every private read or write must filter by `ctx.player.id`. Shared tables need their own rules for who may change each row.

## Economic operations

The hosted backend exposes:

```ts
ctx.forest.settleAction({ actionId, debitAmount, creditAmount });
ctx.forest.settleBurn({ actionId, amount, txHash, burnAddress, reason, metadata });
ctx.forest.executeSwapSession(sessionId, payload);
ctx.forest.game.getBalance();
ctx.forest.game.getTransactions({ page: 1, perPage: 20 });
ctx.forest.game.getBurns({ page: 1, perPage: 20 });
ctx.forest.game.getBurnSummary();
ctx.forest.game.getVaultCapacity();
```

Forest supplies authentication and timestamps. Settlement and burn **inputs** use base-unit strings; their **results**, balance fields, and vault capacity use display strings. Swap-execution inputs and results use base units. See [Game Actions](./game-actions.md) for the frontend authorization step.

Wallet transactions remain frontend SDK operations. Hosted backends do not expose legacy player-state, playable-state, store, leaderboard, or identity-redeem helpers; use SQL for application data and `ctx.player` for identity.

A SQL write and a Forest settlement are separate operations. Preserve action IDs and outcomes, and reconcile an uncertain result before trying another charge. Do not assume retrying an entire backend method is harmless.

## Declared game economy

Maintain `forest.economy.json` alongside the code:

```json
{
	"levers": [],
	"usesRewardPool": false,
	"usesSwaps": false
}
```

Each lever has an `id`, `label`, `value`, and `source: { file, line }` pointing to its implementation. Harness checks those locations, checks that rewards have a positive reward-pool allocation, and checks that swap usage has `autoSwaps: true` in `forest.json`.

These checks help explain the economy to the creator. Enforce game rules in the backend.
