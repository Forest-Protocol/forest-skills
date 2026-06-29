# Persistent Data

Game Balance and Game Actions cover Forest's economic ledger — debits, credits, and settlement. Beyond that, most games need somewhere to keep score: match history, player stats, inventories, cosmetics, a leaderboard. For that you stand up your own live database, on the trusted backend — never in the iframe.

Three kinds of data:

- **Stats** — per-player counters that only go up: games played, wins, kills, items collected.
- **Metrics** — aggregate or time-series data you query later: session length, daily active players, average score per match.
- **Live game state** — a mutable "save file" per player that the game reads and overwrites as they play: current level, inventory, board position, in-progress run.

Put this on the same trusted backend that holds the Settlement Signing Secret (see `game-actions.md`).

> **WARN: Never query a database from the iframe**

The uploaded HTML is untrusted. Database credentials, connection strings, and queries belong only on the trusted backend. Scaffold a small API on that backend for the game to call instead.

## Schema shape

Default to three tables (or collections), keyed by the verified player identity from the Player Identity handshake (`identity.md`) — never the display-only wallet address:

```sql
-- append-only, one row per event
CREATE TABLE stats (
  player_id  TEXT NOT NULL,
  stat_key   TEXT NOT NULL,
  value      BIGINT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- one row per player, overwritten in place
CREATE TABLE game_state (
  player_id  TEXT PRIMARY KEY,
  state      JSONB NOT NULL,
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- aggregates, refreshed on a schedule or on write
CREATE TABLE metrics (
  metric_key TEXT NOT NULL,
  period     DATE NOT NULL,
  value      BIGINT NOT NULL,
  PRIMARY KEY (metric_key, period)
);
```

`stats` stays append-only so history isn't lost. `game_state` is the one table updated in place — the "living" file the game reads on load and writes on every meaningful change. `metrics` is for rollups a dashboard or leaderboard reads, kept separate so heavy aggregate queries don't lock the live tables.

## Where writes happen

Stat and state writes follow the same shape as a settlement, but they are **not** settlements — writing to `game_state` does not move Game Balance:

```txt
HTML game
  -> forest.game.action.authorize({ actionId, debitLimitAmount })
  -> your backend validates the gameplay result
  -> your backend signs and submits the settlement to Forest
  -> your backend writes to stats / game_state / metrics
  -> HTML game reads state back from your backend's API
```

Wire database writes into the same backend handler that signs the settlement, so a confirmed match always updates both the ledger and your stats in one place.

## Provider shortlist

If there's no database yet, scaffold against any of these:

| Provider | Good for |
| --- | --- |
| [Railway](https://railway.com) | One-click Postgres/MySQL/Redis/MongoDB next to your backend service; usage-based billing, fast to set up |
| [Render](https://render.com) | Managed Postgres + Redis with flat, predictable monthly pricing |
| [Supabase](https://supabase.com) | Hosted Postgres with a realtime layer and auto-generated REST API — a leaderboard that updates live without polling |

Default to Railway unless the user names another. Generate the connection setup, migrations, and the backend handler above.
