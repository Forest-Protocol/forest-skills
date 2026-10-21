# forest-skills

[![skills.sh](https://skills.sh/b/Forest-Protocol/forest-skills)](https://skills.sh/Forest-Protocol/forest-skills)

The **`forest`** Claude Code plugin — Forest Protocol agent skillsets in the
[Agent Skills](https://agentskills.io/specification) standard. Each skill lives under `skills/`
with a `SKILL.md` entry point and is namespaced `forest:<skill>`.

## Skills

- **`html-playkit`** (`/forest:html-playkit`) — building/debugging Forest games: Harness games
  with `@forest/runtime` and a hosted backend, or legacy HTML uploads on the iframe bridge. Swaps,
  FOREST session quotes, swap sessions, game balance, burns, game actions, identity.
- **`portfolio`** (`/forest:portfolio`) — reading a user's balances, positions, PnL, recent
  trades, prices, and candles across Forest tokens via the REST API; rebalance/take-profit
  planning. Read-and-plan only — no trade execution.

More Forest skills (creator playbook, builder) drop in as sibling folders under `skills/`.

## Structure

```
.claude-plugin/
  plugin.json     plugin manifest (name, version) → enables forest: namespace
  marketplace.json  marketplace catalog for /plugin marketplace add
skills/<skill>/
  SKILL.md        router: when-to-use, core model, gotchas, routing table
  references/     detailed per-topic spec, mirrored from the Forest developer docs
```

`SKILL.md` is the entry point; `references/*.md` holds the full spec. The canonical,
always-current docs live at [docs.forest.inc](https://docs.forest.inc/developers/forest-playkit).

## Install

Any agent (Claude Code, Cursor, Codex, Copilot, Gemini, …) via the skills CLI:

```bash
npx skills add Forest-Protocol/forest-skills
```

Or as a Claude Code plugin marketplace:

```
/plugin marketplace add Forest-Protocol/forest-skills
/plugin install forest@forest-skills
```

Or symlink a single skill into your skills directory:

```bash
ln -s "$(pwd)/skills/html-playkit" ~/.claude/skills/html-playkit
```

## License

MIT — see [LICENSE](LICENSE).
