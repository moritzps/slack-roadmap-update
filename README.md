# slack-update

A Claude Code skill that creates structured Slack status updates from your git history, project state, and roadmap.

## What it does

After a deployment, run `/slack-update` to:

1. Analyze commits since the last deployment
2. Pull context from project state/tracking files (e.g. `STATE.md`, `ROADMAP.md`)
3. Group changes into **Development** (shipped features, bugfixes) and **Roadmap** (phase progress, strategic changes)
4. Walk you through each topic interactively — post, skip, rewrite, or mark as technical
5. Add status emojis (✅ 🚀 ⏸️ 🔧 🚧 📋 🎯 ⚠️)
6. Show a final preview before posting
7. Post to your Slack channel
8. Remember what was posted (so next time it picks up where you left off)

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

On subsequent runs, the skill remembers the last update and suggests the right base commit automatically.

## Requirements

- Claude Code with Slack MCP integration enabled
- A git repository

## State file

The skill saves `.slack-update-state.json` in your project root to track previous updates. Add it to `.gitignore`.
