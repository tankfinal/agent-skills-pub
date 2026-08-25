# agent-skills-pub

Public Claude Code skills plugin. **No internal references** (company names,
personal names, internal URLs) — this repo is open-source and indexed by GitHub.
Use placeholders like `<your-company-code>`, `<成員 A>` in examples.

## Version sync points (bump together)

- `.claude-plugin/marketplace.json` → `plugins[0].version`
- `plugins/toolkit-pub/.claude-plugin/plugin.json` → `version`
- `plugins/toolkit-pub/skills/nueip/SKILL.md` → frontmatter `version:`
- `README.md` → `**Current version:**` line

## Upstream dependency

`nueip` skill calls `mcp__claude_ai_NUEIP__*` tools from NUEiP's **official remote
MCP** (`https://mcp.nueip.com/mcp`), connected via OAuth. No local server, no
credential handling in this repo. The tool surface is controlled by NUEiP — when
it changes, update `SKILL.md` and bump; there is no companion repo to sync.

Notable gaps vs. a local wrapper (documented in SKILL.md, don't "fix" by guessing):
- no `whoami` — derive own hashid from an unfiltered `get_attendance_records`
- no leave-balance aggregation — returns one row per grant period, skill aggregates
- no team-leave range query — `list_subordinate_leaves_today` is single-date only
- no approval `badges` / status filter on team leaves — count from `items` instead

## Release workflow

`gh release create vX.Y.Z -R tankfinal/agent-skills-pub --target main --latest --title "..." -F -`
(notes via stdin heredoc). Gotcha: `git push origin --delete <tag>` removes the
tag ref but **not** the GitHub Release page — also run `gh release delete <tag>`.

## Skill maintenance hook

Pure version bumps to `SKILL.md` / `plugin.json` don't require updating
`~/.claude/commands/skills-helper.md` — only name / triggers / 用途 changes do.
