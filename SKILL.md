---
name: meeting-minutes
description: "Generate meeting minutes from Granola recordings — fetch meeting data and transcript, fill the Meeting Minutes template, drop in INBOX."
user-invocable: true
argument-hint: "optional search term to filter meetings (e.g. 'OKR', 'Ventas', 'Feb 20')"
---

Generate structured meeting minutes from Granola meeting recordings, filling the vault's Meeting Minutes template and saving to INBOX.

## Philosophy

Meetings produce value only when their outputs are captured, connected, and actionable. This skill bridges the gap between Granola's raw recording and the vault's structured note system — extracting decisions, actions, and context into a format that integrates with the GTD pipeline.

## Vault Exception

This skill creates new files in `00 HUB/00 INBOX/`. It may also update existing project/program/area briefs in `01 PROJECTS/` and `02 AREAS/` with meeting outcomes (with user confirmation).

## Inputs

The user provides one of:
- **No argument** — list recent meetings from Granola and let the user pick one.
- **Search term** — filter the meeting list by title match (case-insensitive). If exactly one match, use it directly. If multiple matches, present the filtered list for the user to choose.
- **"batch"** — process multiple meetings at once (user selects which ones from the list).

## Template

The canonical meeting minutes template lives at:
`03 REFERENCE/CHECKLISTS & TEMPLATES/Meeting Minutes.md`

Read it fresh each time — do not hardcode the template contents. The user may update the template between sessions.

## Workflow

### Step 1 — List Meetings

Call `mcp__granola__list_meetings` with `time_range: "last_30_days"`.

If the user provided a search term, filter the results by case-insensitive title match. Present the matching meetings as a numbered list:

```
1. [Feb 25] Revision OKR Trimestrales y Compromisos... (15 attendees)
2. [Feb 24] Presentación Master Plan de Implementación de AI (6 attendees)
...
```

Ask the user to pick one (or multiple if batch mode). If exactly one match from a search term, confirm with the user before proceeding.

### Step 2 — Fetch Meeting Data

For each selected meeting, fetch data in parallel:
1. `mcp__granola__get_meetings` — for summary, attendees, notes, and metadata.
2. `mcp__granola__get_meeting_transcript` — for the full verbatim transcript.

### Step 3 — Read Template

Read the template file at `03 REFERENCE/CHECKLISTS & TEMPLATES/Meeting Minutes.md` to get the current structure.

### Step 4 — Fill Template

Map Granola data to the template fields:

**Frontmatter:**
- `title:` → `"Meeting Minutes — MEETING_TITLE — YYYY-MM-DD"`
- `AREA:` → infer from attendees/content (e.g., if all @bufalinda.com → "Bufalinda"; if Viniloversus members → "Viniloversus"). Leave blank if unclear.
- `SUB-AREA:` → infer if possible (e.g., "Ventas", "Operaciones", "IT"). Leave blank if unclear.
- `category:` → `meeting`
- `date:` → meeting date in YYYY-MM-DD format
- `attendees:` → YAML list of attendee names from Granola
- `location:` → "Virtual / Call" (default — Granola meetings are recorded calls)
- `tags:` → `[minutes]` plus contextual tags (e.g., `bufalinda` if applicable)

**Meeting Info table:**
- `Date` → formatted meeting date
- `Attendees` → comma-separated list of names (first name + last name, no emails)
- `Location / Call` → "Virtual / Call"
- `Called By` → the note creator from Granola (typically Alberto)

**Summary:**
- Extract from Granola's AI-generated summary. Write as one clear paragraph.

**Next Actions:**
- Extract action items from Granola notes. Format as `- [ ] Action item — @Owner (if identifiable)`
- If Granola provides no clear action items, extract them from the transcript/notes by identifying commitments, follow-ups, and to-dos mentioned.

**Waiting For:**
- Extract items where someone committed to deliver something to someone else. Populate the Owner / Action / Due table.
- If no due dates were mentioned, leave the Due column empty.

**Meeting Notes:**
- Organize Granola's meeting notes by topic. Use the topic headings from Granola if available.
- Under each topic, use bullet points with key discussion points.
- Include a `### Decisions` section with any decisions made during the meeting.

**Agenda linking:**

Search vault AGENDAS folders for an agenda file matching this meeting:
- Search locations: `02 AREAS/02 COMMUNITY/AGENDAS/`, `02 AREAS/03 BUSINESS/BUFALINDA/AGENDAS/`
- Do NOT search `04 ARCHIVES/AGENDAS/`
- Matching strategy (try in order, stop at first confident match):
  1. **Filename keyword overlap** — Normalize the meeting title and each agenda filename by removing prefixes ("Agenda", "Meeting Agenda", "Reunion") and common stop words. Compare normalized word sets. Score = shared words / total unique words. Threshold: 50%+ overlap.
  2. **Attendee-heading match** — For agendas with H2 headings that are person names, check if Granola attendee names appear as H2 headings. If 2+ attendee names match, this is a strong match.

If a matching agenda is found:
- Set `agenda:` frontmatter to `"[[Agenda Filename]]"` (without the `.md` extension)
- Fill the `## Related Agenda` section with `> [[Agenda Filename]] — living agenda document for this recurring meeting`
- Extract any open action items (`- [ ]`) from the agenda to cross-reference with the minutes' Next Actions — flag items that appear in the agenda but were not discussed in the meeting as a note at the end of the Next Actions section

If no matching agenda is found:
- Leave `agenda:` frontmatter empty
- Leave `## Related Agenda` with the default placeholder

**Transcript:**
- Include the full verbatim transcript from Granola in a collapsible callout block:
```
> [!note]- Full Transcript
> TRANSCRIPT_CONTENT
```
This keeps the note clean while preserving the full record.

### Step 5 — Save to INBOX

Save the filled template to:
`00 HUB/00 INBOX/Meeting Minutes — MEETING_TITLE — YYYY-MM-DD.md`

**Filename rules:**
- Replace characters not allowed in filenames: `/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|`
- Truncate the meeting title to 60 characters max if needed (to avoid overly long filenames)
- Use the meeting date from Granola, not today's date

### Step 6 — Report

Tell the user:
- Which file was created (with wikilink for easy navigation)
- Quick summary: attendee count, number of action items extracted, number of decisions captured
- If a matching agenda was found, report the cross-link (e.g., "Linked to agenda: [[Agenda Operaciones Oso]]"). If no matching agenda was found, suggest running `/meeting-agenda` to create one.
- Remind them the note is in INBOX for review and filing

For batch mode, report a summary table of all created files.

### Step 7 — Update Related Briefs

After creating the minutes file, search for related project/program/area briefs to update with meeting outcomes.

**Search strategy:**
1. Check the linked agenda's `## Key References` for project/program links
2. Search `01 PROJECTS/01 Active Projects/` for briefs whose title or content relates to the meeting topic
3. Check `02 AREAS/` MOC files if the meeting maps to an area but not a specific project

**What to update in matching briefs:**
- **Continuation Prompt** — overwrite with current state based on meeting outcomes
- **Next Actions** — mark items discussed/completed; add new action items from minutes
- **Waiting For** — add new waiting-for items extracted from the minutes
- **Log** — append a dated entry summarizing key outcomes with a wikilink to the minutes

**Rules:**
- Only update briefs the user confirms (present the list of candidate briefs and proposed changes before editing)
- Do not create new briefs — only update existing ones
- Preserve existing formatting and style of each brief
- If no matching brief is found, suggest the user create one or skip this step

## Batch Mode

When processing multiple meetings:
1. Present the full list, let the user select which ones to process (comma-separated numbers or "all").
2. Process each meeting sequentially — do NOT use parallel agents since each meeting needs Granola MCP calls which benefit from sequential processing.
3. After all are processed, present a summary table:

```
| Meeting | Date | File | Actions | Decisions |
|---------|------|------|---------|-----------|
| Title 1 | Feb 25 | [[Meeting Minutes — Title 1 — 2026-02-25]] | 3 | 2 |
| Title 2 | Feb 24 | [[Meeting Minutes — Title 2 — 2026-02-24]] | 5 | 1 |
```

## Rules

### Boundary Rules
- **Only create files in `00 HUB/00 INBOX/`** — never anywhere else in the vault.
- **Never modify existing files** — if a meeting minutes file already exists with the same name, warn the user and ask before overwriting.
- **Do not invent content** — only use data from Granola. If a field can't be filled from Granola data, leave it with the template placeholder or mark it as "Not captured."

### Agenda Cross-Linking Rules
- **Always search for a matching agenda** — never skip the agenda search step, even if the meeting title seems unfamiliar.
- **Best-match selection** — if multiple potential agendas match, pick the one with the highest keyword overlap score. Do not ask the user during minutes generation (keep it fast).
- **Link format** — use wikilinks without `.md` extension: `[[Agenda Operaciones Oso]]`, not `[[Agenda Operaciones Oso.md]]`.

### Operational Rules
- Vault working directory: `{{VAULT_ROOT}}`
- Read the template fresh each run — do not cache or hardcode.
- Always fetch both meeting details AND transcript — the transcript provides the richest source for extracting action items and decisions.
- For the transcript section, use a collapsed callout (`> [!note]-`) to keep the note scannable.
- Keep the Summary tight — one paragraph, no filler.
- Action items must be concrete and attributable. "Discuss further" is not an action item. "Schedule follow-up meeting on X topic — @Alberto" is.
- If Granola returns minimal data (e.g., a very short meeting), still create the note but flag thin sections with "(No content captured)" rather than inventing filler.
- Use Spanish for content that was originally in Spanish (most Bufalinda meetings). Do not translate — preserve the original language of the discussion.
