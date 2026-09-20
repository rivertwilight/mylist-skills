# mylist-skills

Skills that teach an AI agent to work with [MyList](https://mylist.world) —
the lists of things people love — through the `mylist` command line.

| Skill | What it teaches |
| --- | --- |
| [`skills/mylist-cli`](skills/mylist-cli/SKILL.md) | Signing in, reading and searching lists and saved things, creating and editing them, tags, entries, publishing, safe deletes, and the `mylist api` escape hatch. |

The skill assumes the CLI is installed and signed in:

```bash
npm install -g @mylist-world/cli
mylist auth login        # opens mylist.world to authorize this machine
```

## Agents without a shell

A hosted agent — ChatGPT, Claude on claude.ai or Cowork, anything that cannot
run a command — connects to MyList's MCP server instead of the CLI:

```
https://mylist.world/api/mcp
```

Add it as a custom connector or remote MCP server. The first connection
opens MyList to sign in and approve the app; nothing to paste. The tools
mirror the CLI (`list_lists`, `get_list`, `create_item`, `add_to_list`,
`search`, `fetch`, …) and every connected app shows up under Settings › Apps
& command line, where it can be disconnected. This skill is not needed there:
the server describes its own tools.

## Install

**Claude Code** — as a plugin from this repo's marketplace:

```
/plugin marketplace add rivertwilight/mylist-skills
/plugin install mylist@mylist-skills
```

or, from a shell:

```bash
claude plugin marketplace add rivertwilight/mylist-skills
claude plugin install mylist@mylist-skills
```

**Claude Code, without the plugin system** — copy the skill folder to
`~/.claude/skills/mylist-cli/` (every project) or `.claude/skills/mylist-cli/`
(one project).

**Any other agent** that reads `SKILL.md` folders (the
[Agent Skills](https://agentskills.io) layout) — copy `skills/mylist-cli/`
into wherever that agent loads skills from. The skill needs nothing but a
shell with `mylist` on the path.

## Layout

```
skills/mylist-cli/
├── SKILL.md                  what the agent reads when the skill triggers
└── references/
    ├── commands.md           every command, flag, exit code and JSON shape
    └── recipes.md            multi-step jobs: build a list from saved things, bulk tag, export, dedupe
.claude-plugin/
├── marketplace.json          lets Claude Code add this repo as a marketplace
└── plugin.json               the plugin the marketplace publishes (the repo root)
```

## Keeping it current

The CLI lives in the [MyList monorepo](https://github.com/rivertwilight/mylist)
under `cli/`. When a command or flag changes there, `references/commands.md`
here is the file to update — it is what agents plan against.
