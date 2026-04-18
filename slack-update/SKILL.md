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

Group into the following categories (only if applicable):
- **Development** – shipped features, improvements (new capability, not restoring broken behaviour)
- **Bugfixes** – fixes for broken or regressed behaviour, stabilisation work
- **Backlog** – if new items / todos have been added to the backlog
- **Roadmap** — phase completions, deferrals, new phases, upcoming work

Rule of thumb: if a change restores or repairs something that used to work, it is a Bugfix. If it adds new capability, it is Development. Pure infrastructure work (no user-visible effect, no fix) goes under Development with i.e. 🚧 or 🔧 depending on status.

### Step 6: Interactive review per topic

For each thematic block, use `AskUserQuestion`:

**Header:** Short topic name (max 12 chars)

**Question:** Show the proposed update text with a status emoji, then ask:

**Options:**
- **Post as-is** — Use the proposed text
- **Skip** — Don't mention this topic
- **Move to Bugfixes** — Re-categorise under the Bugfixes section
- **Rewrite** — User provides a different wording
- **Split into topics** — Topic bundles unrelated things. Break into multiple separate topics (each with its own emoji + header).
- **Itemize** — Topic is one coherent theme but has multiple concrete sub-points. Keep one header, list sub-points as bullets underneath.

On "Rewrite": Follow up with concrete alternative suggestions or accept freeform input.

#### Split into topics flow

When the user picks **Split into topics**:

1. Analyze the combined topic's underlying items (from git, STATE.md, context). Identify 2–4 distinct angles along which it could reasonably be split. Common axes:
   - By audience — user-facing vs. internal/technical
   - By feature area — separate product areas or modules
   - By status — shipped vs. in-progress vs. planned
   - By scope — headline change vs. supporting improvements
2. Present the split proposal via `AskUserQuestion`. Each proposed grouping is a selectable option (multi-select allowed). Also offer:
   - A "Custom split" option where the user describes the split in freeform
   - A "Cancel split" option to return to the original combined topic
3. After split is chosen, re-run Step 6 interactive review for each resulting topic individually. A sub-topic can itself be skipped, rewritten, split, or itemized.

#### Itemize flow

When the user picks **Itemize**:

1. Extract the concrete sub-points from the topic's source material (commits, STATE.md entries, etc.).
2. Propose a header line (short intro, ends with `:`) plus a bulleted list of sub-points. Example shape:
   ```
   🔧 *Review-Bugs behoben* (mit Regressionstests):
     • Dashboard-Error-Screen überarbeitet
     • Randkommentar-Edit repariert
     • Satz-Duplikate behoben
   ```
3. Sub-bullets use `•` (U+2022) prefixed by exactly **2 spaces** of indentation. No mixing with `-` or `*` prefixes.
4. Show the proposed itemization via `AskUserQuestion` with options: Accept / Edit bullets / Cancel.

Do not auto-split or auto-itemize without user confirmation — always show the proposal first.

### Step 7: Status emojis

Tag each item with the appropriate emoji, i.e.:
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

## Slack message formatting (CRITICAL)

Slack messages use Slack's own `mrkdwn` dialect — **not** standard Markdown. Refs:
- https://docs.slack.dev/messaging/formatting-message-text/
- https://www.markdownguide.org/tools/slack/

This is the single most common failure mode of this skill: writing `**Topic**` (standard Markdown) where Slack expects `*Topic*`. Slack treats double asterisks as literal characters around the inner text — which in practice produces broken styling that often reads as italic.

Exact syntax (apply as shown):

- Bold: `*text*` (single asterisks). Never `**text**`.
- Italic: `_text_` (single underscores). Never `*text*` (that is bold) and never `__text__` (Slack does not support that as italic).
- Strikethrough: `~text~` (single tildes). Never `~~text~~`.
- Inline code: `` `text` ``.
- Code block: triple backticks. Language-tag syntax highlighting is not supported in chat.
- Blockquote: line starts with `> `.
- Link: `<https://url|label>`. Never `[label](https://url)` — that is standard Markdown and renders as literal text in a Slack message.
- Bare URL: paste as-is, Slack auto-links it.
- Line break: actual newline character `\n`. Never `<br>`.
- User mention: `<@U123ABC>`. Channel: `<#C123ABC>`. Group: `<!subteam^SAZ…>`.
- Emoji: literal character (🚀) or shortcode (`:rocket:`).

Not supported in message text (will render as literal characters):

- Headings (`#`, `##`, `###`) — use a bold line (`*Section*`) instead.
- Horizontal rules (`---`).
- Tables.
- Images via Markdown (`![alt](url)`) — drag-and-drop only.

Lists: Slack has no native list syntax in message text. Fake lists by starting each line with `- ` or `• ` followed by a space. Pick ONE bullet style per message and stay consistent throughout. Do not mix `-`, `*`, and `•` in the same message.

Sub-bullets: indent with exactly 2 spaces before the bullet character.

Dashes: Use `–` (en dash, U+2013) as the separator between a topic name and its description. Never `—` (em dash). Never `-` (hyphen) for that purpose.

## Message format

```
*Status Update – {date}*

*Development*
{emoji} *{Topic}* – {Description}
{emoji} *{Topic}* – {Description}

*Bugfixes*
{emoji} *{Topic}* – {Description}

*Backlog*
{emoji} *{Topic}* – {Description}

*Roadmap*
{emoji} *{Topic}* – {Description}
```

If there are no items in a category, skip it. All section headers and topic names are bold with **single** asterisks (`*Section*`), not double.

## Rules

- Match the language the user communicates in
- Max 2 sentences per topic
- No technical details (commit hashes, file names, internal phase numbers) — those are internal
- Stakeholder perspective: What does the change mean for end users?
- ALWAYS show the final version before posting
- ALWAYS use status emojis
- Don't repeat topics that were in the previous update (check state file) unless there's meaningful progress
