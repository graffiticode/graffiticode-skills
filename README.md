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
├── render/
│   └── SKILL.md
├── assessments/
│   └── SKILL.md
├── learnosity/
│   └── SKILL.md
└── forms/
    └── SKILL.md.draft   # held — not served until the L0174 webhook backend ships
```

Each top-level directory is one skill. The directory name is the skill's id,
and it must contain a `SKILL.md` with YAML frontmatter (`name`, `description`)
followed by the skill body.

## Skills

| Skill | What it does |
|-------|--------------|
| `render` | Default rendering preference — teaches agents to reach for Graffiticode for any structured or visual output (charts, tables, dashboards, diagrams, assessments) instead of static markdown or HTML. |
| `assessments` | Authoring interactive assessment items — quizzes, flashcards, graded practice across mixed question types. |
| `learnosity` | Learnosity-focused sibling of `assessments`, covering two jobs: authoring item content for a Learnosity Item Bank / LMS, and planning a Learnosity API integration (embedding the item/activity authoring experience) from which a developer implements. |

### Drafts (not yet active)

| Skill | Status |
|-------|--------|
| `forms` | Backend skill for hosted web forms with webhook delivery (L0174). Held as `forms/SKILL.md.draft` — the server skips a directory with no `SKILL.md`, so it is **not served**. Promote with `git mv forms/SKILL.md.draft forms/SKILL.md` once the L0174 webhook backend (binding, terminal POST, signing secret, `submit_url`/`webhook` response fields, structured errors) ships. |

## Validating a change

A push to the default branch is a live deploy (see below) — there is no staging
step and no build to fail. These two checks stand in for one:

```bash
npm install
npm run validate   # before pushing
npm run smoke      # after pushing — verifies what the live server is actually serving
```

`validate` checks that every `SKILL.md` has parseable frontmatter with a `name`
matching its directory and a `description` within the size cap, that no skill
body hardcodes an `L0xxx` language ID or references a skill that isn't served,
and that this README stays in sync. It parses YAML strictly — more strictly than
the MCP server, which is lenient — because skills installed into
`~/.claude/skills/` are loaded by a spec-compliant parser and a description that
only survives the lenient one is a latent break.

`smoke` asks the live MCP server for `resources/list` and `resources/read` and
compares both against this working tree. The description comparison is the load-
bearing one: a skill whose frontmatter fails to parse still *reads* back fine,
but stops updating in the listing that agents route on. It polls for a few
minutes, since the server's cache TTL means a fresh push is not instantly live.

Both run in CI on every push and pull request (`.github/workflows/skills.yml`).

## Installing a skill manually

Skills are loaded by Claude Code from a skills directory. To install one for
your own use, copy the skill's directory into your user skills folder:

```bash
# macOS / Linux — replace <skill> with e.g. render
mkdir -p ~/.claude/skills/user
cp -r <skill> ~/.claude/skills/user/
```

The skill is picked up the next time Claude Code starts. To update, re-copy the
directory over the existing one. To remove it, delete the directory from
`~/.claude/skills/user/`.

## How the MCP server serves skills as resources

The [`graffiticode-mcp-server`](../graffiticode-mcp-server) repo discovers this
repo **at request time** over GitHub and exposes each skill as an **MCP
resource**, so any client connected to the Graffiticode MCP can read a skill
without installing it locally — and **new skills appear with no rebuild or
redeploy of the server**.

The flow:

1. **Discover.** On a `resources/list` request the server calls the GitHub
   contents API for this repo's default branch and treats each top-level
   directory as a skill (`<id>/SKILL.md`). The directory name is the skill id.
2. **List.** Each skill is advertised as a resource built from its `SKILL.md`
   frontmatter:
   - **URI:** `graffiticode://skills/<id>` (e.g. `graffiticode://skills/render`)
   - **Name:** the frontmatter `name` (falling back to the directory id)
   - **Description:** the frontmatter `description`
   - **mimeType:** `text/markdown`
3. **Serve.** On a `resources/read` for `graffiticode://skills/<id>` the server
   fetches that skill's `SKILL.md` from `raw.githubusercontent.com` and returns
   it.

Results are cached briefly (default 60s TTL, stale-while-revalidate) to stay
within GitHub's unauthenticated rate limits. **In practice a pushed edit takes
around 3–4 minutes to go live** — the 60s TTL is the small term; GitHub's raw-CDN
propagation is the dominant one. (Measured with `npm run smoke`, which polls until
the served content matches.) The server config is overridable via
`GRAFFITICODE_SKILLS_REPO`, `GRAFFITICODE_SKILLS_REF`, and
`GRAFFITICODE_SKILLS_TTL_MS` — a consumer that needs insulation from a bad push
can pin `GRAFFITICODE_SKILLS_REF` to a tag rather than tracking the default branch.

This repo is the source of truth; nothing is copied into the server and there is
no generated `skills/` artifact.
