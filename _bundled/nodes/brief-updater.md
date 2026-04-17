---
name: brief-updater
description: "Surgical project brief section editing — insert tasks, toggle checkboxes, update waiting-for, append logs, write continuation prompts"
version: 1
---

Canonical edit operations for Obsidian project briefs. Every mode uses the Edit tool for surgical changes -- never rewrite the entire file. Always match existing formatting of the target brief.

## Core Logic

### Section Location
Sections are H2 headers: `## Next Actions`, `## Waiting For`, `## Log`. The `### Continuation Prompt` subsection lives at the top of `## Next Actions`. Locate by scanning for the exact header string.

### Canonical Insertion Point
After the last existing `- [ ]` line in the target section but BEFORE `### Continuation Prompt` or `## Log`, whichever comes first. If no existing tasks, insert after `## Next Actions` header (or after `### Continuation Prompt` if present).

### Edit Tool Usage
Always use Edit with `old_string`/`new_string`. To append after a line, use that line as `old_string` and same line + new line as `new_string`. Never rewrite sections wholesale.

### Priority Emoji Mapping
| Priority | Emoji |
|----------|-------|
| High / Apple 1-4 | 🔺 |
| Medium / Apple 5 | ⏫ |
| Low / Apple 6-9 | 🔼 |
| Normal / Apple 0 | *(omit)* |

---

## Modes

### task-insert
**Used by:** inbox-clear, reminders-sync, meeting-minutes

Add `- [ ]` line to `## Next Actions` at the canonical insertion point.

**Format:** `- [ ] Title [priority emoji] [📅 due date] [🔁 recurrence]`

Marker order: title -> priority -> due date -> recurrence. Omit any absent marker.
```
- [ ] Call supplier about delivery 🔺 📅 2026-03-20
- [ ] Monthly Love Letter to Niki ⏫ 📅 2026-03-19 🔁 every month
- [ ] Buy office supplies
```
Rules: priority emoji only if not Normal; due date with `📅` prefix; recurrence with `🔁` prefix (`🔁 every N units`); title max 200 chars. If `## Next Actions` missing, flag to user.

---

### wf-update
**Used by:** followup

Modify items in `## Waiting For`. Find target item by semantic match on description text.

- **Follow-up marker:** Append `📨 YYYY-MM-DD` after sending a message
- **RESCHEDULE:** Update or add `📅 YYYY-MM-DD` on the item line
- **REDIRECT/ASSIGN:** Replace `@Person` with `@NewName`, or add `@Name` prefix to unassigned items

Example: `- [ ] Waiting on @Carlos for quote 📅 2026-03-15` -> after RESCHEDULE:2026-04-01 + SEND -> `- [ ] Waiting on @Carlos for quote 📅 2026-04-01 📨 2026-04-10`

---

### checkbox-toggle
**Used by:** followup (COMPLETE), task-sync

Change `- [ ]` to `- [x]` in `## Next Actions` or `## Waiting For`. Find item by semantic match -- wording may differ between systems.

**With attribution (task-sync):** Also append a log entry (detect format first):
- Table: `| YYYY-MM-DD | Task | [Item text] (synced from [Trello/Reminders]) |`
- List: `- YYYY-MM-DD -- Task: [Item text] (synced from [Trello/Reminders])`

---

### log-entry
**Used by:** followup, session-close, close-day, meeting-minutes

Append row to `## Log`. Detect existing format first:
- **Table:** `| YYYY-MM-DD | Type | Description |` -- new row at end of table body
- **List:** `- YYYY-MM-DD -- Type: Description` -- after last list item

Type values: `Task`, `Milestone`, `Follow-up`, or as specified by calling skill. One item per row. Never duplicate same date + summary. Match existing format exactly.

Example: `| 2026-04-10 | Follow-up | Sent via Slack to @Carlos re: supplier quote. Channel: DM. |`

---

### continuation-prompt
**Used by:** session-close, close-day

Update or create `### Continuation Prompt` at top of `## Next Actions`. Only latest is kept -- overwritten each session.

```markdown
### Continuation Prompt
> **Date:** YYYY-MM-DD
> **Mode:** <Vault Management / Programming Projects>
> **Context:** <1-2 sentences>
>
> **Resume from:** <specific next step>
> **Key decisions this session:** <critical choices>
> **Blockers/open questions:** <unresolved>
> **Files touched:** <key files modified>
```

If exists: replace content (header through last `>` line). If missing: insert after `## Next Actions` before any `- [ ]` items.

---

### template-compliance
**Used by:** clean-projects

Add missing frontmatter fields and required sections (`## Outcome`, `## Next Actions`, `## Log`) per Brief Template. Do not modify existing content -- only add what is absent.
