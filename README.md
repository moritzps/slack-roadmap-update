# slack-update

A Claude Code skill that creates structured Slack roadmap updates from your git history.

## What it does

After a deployment, run `/slack-update` to:

1. Analyze all commits since the last deployment
2. Group them into thematic blocks (features, bugfixes, etc.)
3. Walk you through each topic interactively — post, skip, rewrite, or mark as technical
4. Add status emojis (✅ 🚀 ⏸️ 🔧 🚧 📋)
5. Show a final preview before posting
6. Post to your Slack channel via Claude's Slack integration

## Install

```bash
npx skills add moritzps/slack-roadmap-update@slack-update -g
```

Or manually copy `slack-update/SKILL.md` to `~/.claude/skills/slack-update/SKILL.md`.

## Usage

```
/slack-update              # asks for the base commit interactively
/slack-update abc1234      # uses abc1234 as the base commit
```

## Requirements

- Claude Code with Slack MCP integration enabled
- A git repository
