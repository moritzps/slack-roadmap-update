---
name: slack-update
description: Creates a structured Slack roadmap update based on git changes since the last deployment. Interactive review per topic with status emojis. Works in any language.
user-invocable: true
argument-hint: "[commit-hash]"
---

# Slack Roadmap Update

Creates a professional Slack update based on code changes since the last deployment. Walks through each topic interactively so the user controls what gets posted and how.

## Ablauf

### Step 1: Determine base commit

1. **Argument:** If the user passed a commit hash as `$ARGUMENTS`, use it.
2. **Ask:** Otherwise, use `AskUserQuestion` to ask for the last deployed commit. Show the last 10 commits (from `git log --oneline -10`) as options so the user can pick instead of typing a hash.

### Step 2: Analyze changes

```bash
# Code changes only (exclude planning/docs artifacts)
git log <base>..HEAD --oneline --no-merges -- ':!.planning' ':!docs' ':!.claude' | grep -v '^.*docs('

# All commits for full context
git log <base>..HEAD --oneline --no-merges
```

Group changes into thematic blocks:
- Summarize features (don't list individual commits)
- Bundle bugfixes together
- Skip pure planning/docs commits unless stakeholder-relevant

### Step 3: Interactive review per topic

For each thematic block, use `AskUserQuestion`:

**Header:** Short topic name (max 12 chars)

**Question:** Show the proposed update text with a status emoji, then ask:

**Options:**
- **Post as-is** — Use the proposed text
- **Skip** — Don't mention this topic
- **Technical** — Mention briefly under "Technical updates"
- **Rewrite** — User provides a different wording

On "Rewrite": Follow up with concrete alternative suggestions or accept freeform input.

### Step 4: Status emojis

Tag each item with the appropriate emoji:
- ✅ Feature complete and live
- 🚀 Newly deployed / just released
- ⏸️ Parked / deferred
- 🔧 Bugfixes
- 🚧 Work in progress / partially complete
- 📋 Planning completed

### Step 5: Final preview

**ALWAYS** show the final message as formatted text before posting.

Use `AskUserQuestion` to confirm:
- **Post it** — Send to Slack
- **Edit again** — Go back to editing
- **Just copy** — User posts manually

### Step 6: Post

If confirmed, send via Slack MCP tool. Use `slack_search_channels` to find the target channel if not known.

## Message format

```
*Roadmap Update — {date}*

{emoji} *{Topic}* — {Description}

{emoji} *{Topic}* — {Description}

...
```

## Rules

- Match the language the user communicates in
- Max 2 sentences per topic
- No technical details (commit hashes, file names, internal phase numbers) — those are internal
- Stakeholder perspective: What does the change mean for end users?
- ALWAYS show the final version before posting
- ALWAYS use status emojis
