# forest-skills

The **`forest`** Claude Code plugin — Forest Protocol agent skillsets in the
[Agent Skills](https://agentskills.io/specification) standard. Each skill lives under `skills/`
with a `SKILL.md` entry point and is namespaced `forest:<skill>`.

## Skills

- **`html-playkit`** (`/forest:html-playkit`) — building/debugging HTML games & apps embedded in
  a Forest token page: the iframe bridge, swaps, swap sessions, game balance, burns, game actions,
  identity.

More Forest skills (creator playbook, builder) drop in as sibling folders under `skills/`.

## Structure

```
.claude-plugin/
  plugin.json     plugin manifest (name, version) → enables forest: namespace
skills/<skill>/
  SKILL.md        router: when-to-use, core model, gotchas, routing table
  references/     detailed per-topic spec, mirrored from the Forest developer docs
```

`SKILL.md` is the entry point; `references/*.md` holds the full spec. The canonical,
always-current docs live at [docs.forest.inc](https://docs.forest.inc/developers/forest-playkit).

## Install

Add as a Claude Code plugin from this repo:

```
/plugin marketplace add <owner>/forest-skills
/plugin install forest@forest-skills
```

Or symlink a single skill into your skills directory:

```bash
ln -s "$(pwd)/skills/html-playkit" ~/.claude/skills/html-playkit
```
