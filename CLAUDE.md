# agent-skills-pub

Public Claude Code skills plugin. **No internal references** (company names,
personal names, internal URLs) — this repo is open-source and indexed by GitHub.
Use placeholders like `<your-company-code>`, `<成員 A>` in examples.

## Version sync points (bump together)

- `.claude-plugin/marketplace.json` → `plugins[0].version`
- `plugins/toolkit-pub/.claude-plugin/plugin.json` → `version`
- `plugins/toolkit-pub/skills/nueip/SKILL.md` → frontmatter `version:`
- `README.md` → `**Current version:**` line

## Companion repo

`nueip` skill calls `mcp__nueip__*` tools from [`nueip-mcp`](https://github.com/tankfinal/nueip-mcp)
(local: `~/CursorProjects/nueip-mcp`). Tool-shape changes must bump both repos.

## Release workflow

`gh release create vX.Y.Z -R tankfinal/agent-skills-pub --target main --latest --title "..." -F -`
(notes via stdin heredoc). Gotcha: `git push origin --delete <tag>` removes the
tag ref but **not** the GitHub Release page — also run `gh release delete <tag>`.

## Skill maintenance hook

Pure version bumps to `SKILL.md` / `plugin.json` don't require updating
`~/.claude/commands/skills-helper.md` — only name / triggers / 用途 changes do.
