---
name: slack-update
description: Creates structured Slack status updates (development + roadmap) from git history, project state, and roadmap. Interactive per-topic review with status emojis. Remembers previous updates.
user-invocable: true
argument-hint: "[commit-hash]"
---

# Slack Status Update

Creates professional Slack updates combining development progress and roadmap status. Walks through each topic interactively so the user controls what gets posted and how.

The update has two sections:
1. **Development Update** — What changed since the last deployment (from git + project state)
2. **Roadmap Update** — Current project status, upcoming work, strategic changes (from roadmap/planning files)

## Process

### Step 1: Load previous update context

Check if `.slack-update-state.json` exists in the project root. If it does, read it to know:
- The base commit from the last update
- The date of the last update
- Which topics were included last time

This avoids repeating the same items and helps the user pick up where they left off.

### Step 2: Determine base commit

1. **State file:** If `.slack-update-state.json` has a `lastCommit` and `lastDate`, suggest it as the default.
2. **Argument:** If the user passed a commit hash as `$ARGUMENTS`, use it instead.
3. **Ask:** Otherwise, use `AskUserQuestion` to ask for the last deployed commit. Show the last 10 commits (from `git log --oneline -10`) as options so the user can pick instead of typing a hash.

### Step 3: Gather context from multiple sources

#### Source A: Git history (development changes)
```bash
# Code changes only (exclude planning/docs artifacts)
git log <base>..HEAD --oneline --no-merges -- ':!.planning' ':!docs' ':!.claude' | grep -v '^.*docs('

# All commits for full context
git log <base>..HEAD --oneline --no-merges
```

#### Source B: Project state file (better descriptions)
Look for state/tracking files that contain task summaries:
- `.planning/STATE.md` — "Quick Tasks Completed" table, "Decisions", "Roadmap Evolution"
- Any similar project tracking file

These often have cleaner descriptions than commit messages, especially when multiple features were bundled into a single commit by automation tools.

#### Source C: Roadmap (strategic context)
If the project uses a roadmap file (e.g. `.planning/ROADMAP.md`):
- Check for recently completed phases
- Check for newly added or deferred phases
- Check for milestone progress

This feeds the "Roadmap Update" section of the message.

### Step 4: Ask about current project status

Use `AskUserQuestion` to ask the user about things that aren't in the code:
- Any blockers or risks to mention?
- Anything upcoming that stakeholders should know about?
- Any strategic changes to the roadmap?

Options: provide a few detected topics from the roadmap, plus "Nothing to add".

### Step 5: Group into thematic blocks

Combine all sources into thematic blocks. Prefer STATE.md descriptions over raw commit messages where available.

Group into two categories:
- **Development** — shipped features, bugfixes, improvements
- **Roadmap** — phase completions, deferrals, new phases, upcoming work

### Step 6: Interactive review per topic

For each thematic block, use `AskUserQuestion`:

**Header:** Short topic name (max 12 chars)

**Question:** Show the proposed update text with a status emoji, then ask:

**Options:**
- **Post as-is** — Use the proposed text
- **Skip** — Don't mention this topic
- **Technical** — Mention briefly under "Technical updates"
- **Rewrite** — User provides a different wording

On "Rewrite": Follow up with concrete alternative suggestions or accept freeform input.

### Step 7: Status emojis

Tag each item with the appropriate emoji:
- ✅ Feature complete and live
- 🚀 Newly deployed / just released
- ⏸️ Parked / deferred
- 🔧 Bugfixes
- 🚧 Work in progress / partially complete
- 📋 Planning completed
- 🎯 Milestone reached
- ⚠️ Blocker / risk

### Step 8: Final preview

**ALWAYS** show the final message as formatted text before posting.

Use `AskUserQuestion` to confirm:
- **Post it** — Send to Slack
- **Edit again** — Go back to editing
- **Just copy** — User posts manually

### Step 9: Post and remember

If confirmed:
1. Send via Slack MCP tool. Use `slack_search_channels` to find the target channel if not known.
2. Save state to `.slack-update-state.json`:

```json
{
  "lastCommit": "<HEAD commit hash>",
  "lastDate": "<ISO date>",
  "lastTopics": ["topic1", "topic2"],
  "channel": "<channel-id>"
}
```

This file should be gitignored (remind the user to add it if not).

## Message format

```
*Status Update — {date}*

*Development*
{emoji} *{Topic}* — {Description}
{emoji} *{Topic}* — {Description}

*Roadmap*
{emoji} *{Topic}* — {Description}

*Technical*
- {brief item}
```

If there are no roadmap items, skip that section. Same for technical items.

## Rules

- Match the language the user communicates in
- Max 2 sentences per topic
- No technical details (commit hashes, file names, internal phase numbers) — those are internal
- Stakeholder perspective: What does the change mean for end users?
- ALWAYS show the final version before posting
- ALWAYS use status emojis
- Don't repeat topics that were in the previous update (check state file) unless there's meaningful progress
