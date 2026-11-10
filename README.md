# common-skills

Reusable skills for AI coding agents, kept in one place and installed into any project with
standard tools. Skills follow the [Agent Skills](https://agentskills.io) format
(`skills/<name>/SKILL.md`), so they work with any agent that reads it: Claude Code, Codex,
Cursor, Gemini CLI, GitHub Copilot, OpenCode and others.

## Skills

| Skill | What it does |
| --- | --- |
| [space-init](skills/space-init/SKILL.md) | Session bootstrap. Loads standing rules: terse communication, do only what was asked, dedicated tools over shell commands, mcp-document-service as memory (optional), comment style. Run it at the start of a session or after a context compaction. |
| [service-map](skills/service-map/SKILL.md) | Maps one microservice into mcp-document-service: a short service doc, an index doc per set of related services, and relations between them. Lets agents understand a multi-service system without reading every repo. |

Both use [mcp-document-service](https://github.com/Ekrosrb/mcp-document-service) as shared
memory. `service-map` needs it connected as an MCP server; `space-init` works without it
and warns that memory tools are unavailable. Install both. `service-map` reads its session rules from `space-init`, and `space-init` refers
to the docs that `service-map` builds.

## Install

### npx skills (recommended)

[skills](https://github.com/vercel-labs/skills) CLI. Needs Node.js 22.20+.

```bash
npx skills add Ekrosrb/common-skills
```

Installs into the current project. The main copy goes to `.agents/skills/`, which Codex,
Cursor, Gemini CLI, GitHub Copilot, OpenCode, Amp, Cline and others read. Agents with their
own folder get a symlink, for example `.claude/skills/` for Claude Code.

Options:

- `--list`: show the skills without installing.
- `--skill space-init`: install one skill.
- `-a claude-code -a codex`: only these agents.
- `-g`: install for the user (`~/.agents/skills/`, `~/.claude/skills/`, ...) instead of the project.
- `--copy`: copy instead of symlink.
- `-y`: no prompts.

```bash
npx skills update
```

```bash
npx skills remove space-init service-map
```

### openskills

[openskills](https://github.com/numman-ali/openskills) CLI. Needs Node.js 20.6+.

```bash
npx openskills install Ekrosrb/common-skills --universal
```

```bash
npx openskills sync
```

- `--universal` installs into `.agent/skills/` (singular `.agent`). Without it the skills go
  to `.claude/skills/`. Use `--global` to install into `~/.claude/skills/`.
- `sync` lists the installed skills in `AGENTS.md`, so agents without native skill support
  can find them.

### Claude Code plugin marketplace

The repo is also a Claude Code marketplace (`.claude-plugin/marketplace.json`). Each skill is
a plugin.

```bash
claude plugin marketplace add Ekrosrb/common-skills
```

```bash
claude plugin install space-init@common-skills
```

```bash
claude plugin install service-map@common-skills
```

Inside a session: `/plugin marketplace add Ekrosrb/common-skills`, then `/plugin`.

### Manual

Clone the repo and copy or symlink the skill folders into the agent's skills folder:

- `.agents/skills/` (project) or `~/.agents/skills/` (user): Codex, and most agents through
  npx skills.
- `.claude/skills/` (project) or `~/.claude/skills/` (user): Claude Code.

```bash
git clone https://github.com/Ekrosrb/common-skills.git
```

```bash
mkdir -p .agents/skills && cp -r common-skills/skills/* .agents/skills/
```

## Layout

```text
skills/
  space-init/
    SKILL.md
    references/      rule files, one per area
  service-map/
    SKILL.md
    references/      doc templates, code discovery steps
.claude-plugin/
  marketplace.json   Claude Code marketplace
```

To add a skill, create `skills/<name>/SKILL.md`. `name` in its frontmatter must match the
folder name. For the Claude Code marketplace, also add an entry to
`.claude-plugin/marketplace.json`.

## License

[Apache 2.0](LICENSE)
