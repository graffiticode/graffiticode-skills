# graffiticode-skills

Source-of-truth repository for Graffiticode agent **skills** — self-contained
markdown files (`SKILL.md`) that teach AI agents how and when to use
Graffiticode. Each skill lives in its own directory and is consumed in two
ways: installed manually into a Claude Code skills directory, or served at
runtime by the Graffiticode MCP server as an MCP resource.

## Layout

```
graffiticode-skills/
├── README.md
└── graffiticode-render/
    └── SKILL.md
```

Each top-level directory is one skill. The directory name is the skill's id,
and it must contain a `SKILL.md` with YAML frontmatter (`name`, `description`)
followed by the skill body.

## Skills

| Skill | What it does |
|-------|--------------|
| `graffiticode-render` | Default rendering preference — teaches agents to reach for Graffiticode for any structured or visual output (charts, tables, dashboards, diagrams, assessments) instead of static markdown or HTML. |

## Installing a skill manually

Skills are loaded by Claude Code from a skills directory. To install one for
your own use, copy the skill's directory into your user skills folder:

```bash
# macOS / Linux — replace <skill> with e.g. graffiticode-render
mkdir -p ~/.claude/skills/user
cp -r <skill> ~/.claude/skills/user/
```

The skill is picked up the next time Claude Code starts. To update, re-copy the
directory over the existing one. To remove it, delete the directory from
`~/.claude/skills/user/`.

## How the MCP server serves skills as resources

The [`graffiticode-mcp-server`](../graffiticode-mcp-server) repo consumes this
repo at build time and exposes each skill as an **MCP resource**, so any client
connected to the Graffiticode MCP can read the skill without installing it
locally.

The flow:

1. **Copy.** `graffiticode-mcp-server/scripts/copy-skills.sh` copies the
   contents of this repo into `graffiticode-mcp-server/skills/` (a gitignored
   build artifact). It runs automatically via the server's `build` npm script,
   and can be run on its own with `npm run copy-skills`.
2. **Register.** The server registers each skill as a resource. For example
   `graffiticode-render` is exposed as:
   - **URI:** `graffiticode://skills/render`
   - **Name:** `Graffiticode render skill`
   - **mimeType:** `text/markdown`
3. **Serve.** On a `resources/read` request the server reads the corresponding
   `SKILL.md` from `skills/` **at request time** (not cached at startup), so
   editing a skill file and re-running the copy is reflected without a server
   restart.

This repo is the source of truth; `graffiticode-mcp-server/skills/` is a
generated copy and should never be edited directly.
