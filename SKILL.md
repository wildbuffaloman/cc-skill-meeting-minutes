---
name: meeting-minutes
version: "0.0.4"
description: "Generate meeting minutes from Granola recordings — fetch meeting data and transcript, fill the Meeting Minutes template, drop in INBOX. v0.0.4: CC dedup-ledger append is now universal (Step 5.5, all modes) + pre-flight self-heals a missing ledger row, so non-cron callers like /aiac-close no longer leave /close-day re-flagging already-processed meetings."
user-invocable: true
argument-hint: "optional: search term, 'batch', or 'cron' (today-only, ledger-dedup, 7-day hard cutoff)"
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
- **"cron"** — automated mode for scheduled triggers. No user interaction. Processes only meetings with `created_at` matching today's date, after applying a 7-day hard cutoff and checking the CC ledger for duplicates. See "Cron Mode" section below.

## Cron Mode

When invoked with the `cron` argument, the skill runs non-interactively for scheduled execution. The caller (a scheduled trigger) does not sit at a terminal, so there is no user to pick from a list.

### Hard Rules (cron mode only)

1. **7-day hard cutoff.** Ignore any meeting whose `created_at` is older than `today - 7 days`. This filter runs BEFORE the ledger check and cannot be overridden in cron mode.
2. **Today-only default.** Process only meetings with `created_at` matching today's date (local). Meetings from yesterday / earlier days within the 7-day window are NOT processed in cron mode — they are handled by manual invocation if the user wants them.
3. **Ledger dedup.** Before processing a candidate meeting, check the CC ledger (path below). If the meeting's Granola document ID appears in the `Processed` table, skip it silently.
4. **Ledger append after success.** After each meeting's minutes file is written to INBOX, append a new row to the `Processed` table in the ledger with: date, title, Granola ID, vault note wikilink. This is a durable write — do not skip on failure (if ledger append fails, surface it as an error so we don't silently re-process). **As of 0.0.4 this is no longer cron-exclusive** — the canonical idempotent append lives at Step 5.5 and runs in ALL modes (interactive, batch, cron). Cron mode inherits it; this rule remains for explicitness.
5. **No user prompts.** If zero meetings qualify today, write a single-line report and exit. Do not ask the user anything.
6. **No brief updates in cron mode.** Step 7 (Update Related Briefs) is handled by the downstream `action-extraction` skill — skip it entirely in cron mode to avoid double-writes.

### CC Ledger Path

`~/.claude/state/processed-meetings.md` — resolved at runtime via `Path.home()` / `os.path.expanduser("~")`. Works on both MacBook Pro (`/Users/albertoduhau/.claude/state/`) and Mac Mini (`/Users/leoatreidis/.claude/state/`) without machine-specific hardcoding.

Before reading or appending to the ledger, ensure the parent directory exists:

```python
from pathlib import Path
ledger = Path.home() / ".claude" / "state" / "processed-meetings.md"
ledger.parent.mkdir(parents=True, exist_ok=True)
if not ledger.exists():
    ledger.write_text("# Processed Meetings\n\n## Processed\n\n| Date | Meeting Title | Granola ID | Vault Note |\n|------|--------------|------------|------------|\n")
```

Format (markdown table under the `## Processed` heading):

```
| Date | Meeting Title | Granola ID | Vault Note |
|------|--------------|------------|------------|
| YYYY-MM-DD | Title | granola-doc-id | [[Meeting Minutes — Title — YYYY-MM-DD]] |
```

Dedup key: Granola document ID (column 3). Exact string match.

### Cron-Mode Workflow

1. List today's meetings via Method A or B (as determined by Step 0).
2. Apply 7-day hard cutoff filter.
3. Apply today-only filter.
4. Read the CC ledger; drop any meeting whose Granola ID is already present.
5. For each surviving meeting: run Steps 2–6 as normal (fetch, fill template, save to INBOX, report).
6. After each save, append a ledger row.
7. Skip Step 7 entirely.
8. Final report: terminal-only summary listing meetings processed (or "no new meetings today").

### Synthesis-Status Tracking (rate-limit resilience)

Meeting Minutes generation has two LLM-cost tiers:
- **Cheap step** — fetch transcript from Granola, write the file scaffold (frontmatter + Transcript collapsible block).
- **Expensive step** — synthesize Action Items, Waiting For, Decisions, Summary from the transcript.

The expensive step can fail mid-run (rate limit, model timeout, agent context exhaustion) AFTER the cheap step has already written a partially-populated file to INBOX. Without explicit status tracking, downstream consumers (`/action-extraction`) cannot tell a "synthesis incomplete" file from an "intentionally empty" file (e.g., a meeting genuinely had no actions).

**Required behavior:**

1. **At the start of synthesis** (before extraction begins), write `synthesis_status: pending` to the Meeting Minutes file's frontmatter, alongside any other captured metadata (`title`, `date`, `attendees`, `granola_id`).

2. **Before each major synthesis sub-step** (Action Items extraction, Waiting For extraction, Decisions extraction, Summary synthesis), write a placeholder line into the corresponding section like:
   ```markdown
   ## Action Items
   _<!-- Synthesis pending — re-run /meeting-minutes ${granola_id} -->_
   ```

3. **On successful synthesis completion** (all four sections populated with real content, even if "None"), update frontmatter to `synthesis_status: complete`.

4. **If the agent is rate-limited or otherwise aborts** during synthesis, the file remains on disk with `synthesis_status: pending` and the placeholder lines visible. The CC ledger row should NOT be appended in this case — re-running the skill must be allowed to overwrite the stub.

5. **Re-run behavior:** when invoked with a meeting_id that already has a Meeting Minutes file:
   - If `synthesis_status: complete` → skip (idempotent dedup).
   - If `synthesis_status: pending` or absent → overwrite the synthesis sections (preserve frontmatter id/date), set `synthesis_status: complete` on success.

**Downstream contract:** `/action-extraction` reads `synthesis_status` before processing. Files with `synthesis_status: pending` are surfaced as a clear error ("synthesis incomplete — re-run `/meeting-minutes ${granola_id}` first") rather than treated as input. See `~/.claude/skills/action-extraction/SKILL.md` § "Pre-flight: Synthesis-Status Check".

**Evidence:** 2026-04-29 `/close-day` Phase 1A.7 — `/meeting-minutes cron` was rate-limited mid-run after capturing the G&P transcript (188KB) but before extracting actions. The resulting file had `## Next Actions: - None` and the raw timestamped transcript in `## Summary`. `/action-extraction` correctly refused to operate but couldn't tell whether the absence of actions was real or an extraction failure. A `synthesis_status: pending` flag would have made the failure explicit and unambiguous on first read.

### User-Pasted Transcript Path

If the user pastes a raw transcript directly in the conversation (or provides meeting metadata + transcript text), **use that as the primary source** instead of fetching from Granola. This path is valuable when:
- Granola transcript is behind a paywall (paid tier)
- The user has a transcript from another source (manual recording, Zoom, etc.)
- The user wants to enrich existing minutes with a transcript they obtained separately

When a user-pasted transcript is available:
1. Skip Granola transcript fetch (Steps 1-2 transcript portions). Still use Granola for meeting metadata/summary if available.
2. Use the pasted transcript as the verbatim source for the `## Transcript` collapsible block.
3. Extract action items, decisions, waiting-for, and meeting notes directly from the transcript text — this often yields significantly richer output than Granola's AI summary alone (typically 50%+ more actionable items).

## Template

The canonical meeting minutes template lives at:
`03 REFERENCE/CHECKLISTS & TEMPLATES/Meeting Minutes.md`

Read it fresh each time — do not hardcode the template contents. The user may update the template between sessions.

## Granola Data Access

This skill supports two methods for accessing Granola data. **Always try Method A first.** Fall back to Method B if MCP tools are unavailable.

### Method A — Granola MCP (preferred)

Use when `mcp__claude_ai_Granola__*` tools are available in the session (check via ToolSearch for "granola").

- **List meetings:** `mcp__claude_ai_Granola__list_meetings` with `time_range: "last_30_days"`
- **Fetch meeting:** `mcp__claude_ai_Granola__get_meetings`
- **Fetch transcript:** `mcp__claude_ai_Granola__get_meeting_transcript`

**Reliability notes:**
- The MCP namespace is `mcp__claude_ai_Granola__*`, NOT `mcp__claude_ai_Granola__*`. Older docs may reference the latter — that prefix does not exist in the current MCP catalog.
- `time_range: "this_week"` returns empty results even when meetings exist in-range (verified 2026-04-28). Always use `"last_30_days"` and filter client-side: parse each meeting's `created_at` ISO timestamp into local TZ before date-comparing. Naive string-prefix matching on `created_at` can fail across TZ boundaries.

### Method B — Granola Local Cache + REST API (fallback)

Use when the Granola MCP server shows "Connected" in `claude mcp list` but exposes zero tools. This is a known issue with Granola's MCP OAuth.

**Step B1 — List meetings from local cache:**
```python
import json
cache_path = "~/Library/Application Support/Granola/cache-v6.json"
# data["cache"]["state"]["documents"] → dict of doc_id → meeting data
# Each document has: title, created_at, people, google_calendar_event, type, notes_markdown
```

**Step B2 — Fetch transcript via REST API:**
```python
import json, urllib.request, gzip
# Auth token from: ~/Library/Application Support/Granola/supabase.json
#   → json.loads(data["workos_tokens"])["access_token"]
# POST https://api.granola.ai/v1/get-document-transcript
#   Body: {"document_id": "<id>"}
#   Headers: Authorization header with access token, Accept-Encoding: gzip
# Response is gzip-encoded list of {document_id, start_timestamp, text, source, id}
```

**Step B3 — Fetch AI summary panels via REST API:**
```python
# POST https://api.granola.ai/v1/get-document-panels
#   Body: {"document_id": "<id>"}
#   Same auth + gzip handling as transcript
# Returns list of panels — look for template_slug: "meeting-summary-consolidated"
#   → panel["original_content"] contains HTML summary
```

**Step B4 — Fetch full document via REST API (if local cache is stale):**
```python
# POST https://api.granola.ai/v1/get-documents-batch
#   Body: {"document_ids": ["<id>"]}
#   Returns {"docs": [...]} with full document objects
```

## Workflow

### Step 0 — Determine Access Method

1. Search for Granola MCP tools via ToolSearch ("granola").
2. If tools found → use Method A for all subsequent steps.
3. If no tools found → check `claude mcp list` to confirm server status, then use Method B.
4. Do NOT ask the user to reconnect or re-authorize — use the fallback silently.

### Pre-flight — Cross-folder granola_id check

Before any other work, scan the vault for an existing Meeting Minutes file that already covers a target Granola meeting. For each candidate meeting ID:

1. Run `grep -rl "granola_id: <id>" "03 REFERENCE/" "02 AREAS/" "00 HUB/00 INBOX/" 2>/dev/null` (excluding `04 ARCHIVES/`).
2. If a match is found AND that file has `synthesis_status: complete` in frontmatter → treat as already-routed; **skip the INBOX write entirely** and log as a dedup hit. **Also ensure the CC ledger has a row for this `granola_id` — if absent, append it now (idempotent backfill per Step 5.5).** This self-heals the exact drift where minutes were generated by a non-cron path (e.g. `/aiac-close` Step 3.5 invoking `/meeting-minutes` interactively) that wrote the file but not the ledger, leaving `/close-day` to re-flag the meeting as new. Evidence: 2026-05-29 — AIAC JV minutes existed in REFERENCE but the ledger (last written 5-28) had no row; close-day re-proposed it.
3. If a match is found but `synthesis_status: pending` (or absent) → treat as a re-run case per the existing Synthesis-Status Tracking spec.
4. If no match → proceed to Step 1.

The CC ledger at `~/.claude/state/processed-meetings.md` is the cron-mode dedup signal; cross-folder file presence is the truer source. The ledger can miss entries (e.g., when a prior session crashed before the append) while the routed file already exists in REFERENCE. Evidence: 2026-05-13 — Repaso Semanal Ventas 2026-05-11 ledger entry was missing but yesterday's REF copy existed in `03 REFERENCE/BUSINESS/01 BUFALINDA/MINUTES/`; re-running `/meeting-minutes` produced a 62 KB INBOX duplicate alongside the 36 KB REF that needed manual dedupe + archive during `/action-extraction`.

### Step 1 — List Meetings

**Method A:** Call `mcp__claude_ai_Granola__list_meetings` with `time_range: "last_30_days"`.

**Method B:** Read the local cache at `~/Library/Application Support/Granola/cache-v6.json`. Extract `state.documents` and combine with `state.sharedDocuments`. Build a meeting list from document titles, `created_at` dates, and people/attendee counts.

If the user provided a search term, filter the results by case-insensitive title match. Present the matching meetings as a numbered list:

```
1. [Feb 25] Revision OKR Trimestrales y Compromisos... (15 attendees)
2. [Feb 24] Presentación Master Plan de Implementación de AI (6 attendees)
...
```

Ask the user to pick one (or multiple if batch mode). If exactly one match from a search term, confirm with the user before proceeding.

### Step 2 — Fetch Meeting Data

**Method A:** For each selected meeting, fetch data in parallel:
1. `mcp__claude_ai_Granola__get_meetings` — for summary, attendees, notes, and metadata.
2. `mcp__claude_ai_Granola__get_meeting_transcript` — for the full verbatim transcript.

**Method B:** For each selected meeting, fetch via REST API (use Python with `urllib.request`):
1. `get-document-transcript` — verbatim transcript segments, combine into full text.
2. `get-document-panels` — AI-generated summary (use `original_content` from the summary panel).
3. Meeting metadata (title, people, calendar event) from the local cache document object.

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
- **Ownership split rule** (per [[Brief Template Compliance]] convention, added 2026-04-21): route items where I (@Alberto) am **at least one of the owners** — including mixed ownership like `@Alberto / @Alondra` — to Next Actions. Items assigned **fully to others** (e.g., `@Alondra`, `@JC Zerpa / @Philippe`) go to Waiting For with the person populated as Owner. **Never** place @others-only items in Next Actions.

**Waiting For:**
- Extract items where someone committed to deliver something to someone else (i.e., @others-only items). Populate the Owner / Action / Due table.
- If no due dates were mentioned, leave the Due column empty.

**Meeting Notes:**
- Organize Granola's meeting notes by topic. Use the topic headings from Granola if available.
- Under each topic, use bullet points with key discussion points.
- Include a `### Decisions` section with any decisions made during the meeting.

**Agenda linking:**

Search the vault for an agenda file matching this meeting:
- **Search scope:** vault-wide via frontmatter — `grep -rl "category: meeting-agenda" "02 AREAS/" "03 REFERENCE/" "01 PROJECTS/" 2>/dev/null`
- Do NOT search `04 ARCHIVES/`
- **Why frontmatter scan, not hardcoded folders:** agendas live deeper than the conventional `AGENDAS/` subfolders. The canonical `[[Meeting Agenda Convention]]` requires `category: meeting-agenda` frontmatter on every agenda file, but the file can be located in any topical home (e.g., `02 AREAS/03 BUSINESS/BUFALINDA/MERCADEO/Distribución y Ventas/Agenda Repaso Semanal Ventas.md`). Hardcoded folder scans miss these. Evidence: 2026-05-13 — Repaso Semanal Ventas falsely reported "no agenda exists" based on a 2-folder scan; yesterday's REF copy had correctly linked the agenda via vault-wide search.
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

### Step 4.5 — Organization Extraction (ambient enrichment)

Scan the transcript for organization names mentioned (companies, vendors, regulators, partners, distributors). For each org not already present at `03 REFERENCE/COMMUNITY/ORGANIZATIONS/`, call `/organization-card quick {Name}` (Silent Quick Add). Cap at **3 per run, shared with contact extraction** per [[Contact & Organization Awareness]]. Domain-based candidates derived from any `@domain.com` mentioned in the transcript also count toward the shared cap.

### Step 5 — Save to INBOX

Save the filled template to:
`00 HUB/00 INBOX/Meeting Minutes — MEETING_TITLE — YYYY-MM-DD.md`

**Filename rules:**
- Replace characters not allowed in filenames: `/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|`
- Truncate the meeting title to 60 characters max if needed (to avoid overly long filenames)
- Use the meeting date from Granola, not today's date

**Frontmatter:** the new file's YAML must comply with [[Frontmatter Schema]] — emit every Required field for `category: meeting-minutes` per the Per-Category Required-Field Matrix, use canonical key names only, wikilink-typed fields as resolving `"[[wikilinks]]"`, unquoted ISO dates, and omit empty optional keys.

### Step 5.5 — Append to CC Ledger (ALL modes — idempotent)

After every successful INBOX write, append a row to the CC dedup ledger so that **any** caller of this skill keeps the ledger current — not just cron mode. This is what lets `/close-day`'s Granola discovery scan recognize already-processed meetings regardless of which path generated them (interactive `/meeting-minutes`, `batch`, `cron`, or `/aiac-close` Step 3.5).

> **Why this is universal, not cron-only (2026-05-29 fix):** Previously only cron mode (Hard Rule #4) appended the ledger. Interactive invocations — including `/aiac-close` Step 3.5 — wrote the minutes file but never the ledger, so `/close-day` re-flagged the meeting every morning. The ledger has a single canonical writer assumption; honoring it in all modes is the root fix. See [[feedback_aiac_session_minutes_ledger_gap]].

**Ledger path:** `~/.claude/state/processed-meetings.md` — resolve via `Path.home()` (works on both machines; never hardcode `/Users/<name>/...`). Ensure parent dir + header exist (see the snippet in the Cron Mode → CC Ledger Path section).

**Idempotent append (run after each saved minutes file):**

```python
from pathlib import Path
ledger = Path.home() / ".claude" / "state" / "processed-meetings.md"
ledger.parent.mkdir(parents=True, exist_ok=True)
if not ledger.exists():
    ledger.write_text("# Processed Meetings\n\n## Processed\n\n| Date | Meeting Title | Granola ID | Vault Note |\n|------|--------------|------------|------------|\n")
text = ledger.read_text()
if granola_id not in text:                      # dedup key = Granola doc ID, exact match
    row = f"| {meeting_date} | {title} | {granola_id} | [[Meeting Minutes — {title} — {meeting_date}]] |"
    if not text.endswith("\n"): text += "\n"
    ledger.write_text(text + row + "\n")
```

**Rules:**
- Dedup key is the Granola document ID (exact string match) — never append a duplicate row.
- Append ONLY when `synthesis_status: complete`. If synthesis aborted (`synthesis_status: pending`), do NOT append — re-running must be allowed to overwrite the stub (consistent with Synthesis-Status Tracking rule #4).
- If the ledger append fails, surface it as an error — do not silently skip (a missing row causes downstream re-processing).
- This step makes cron Hard Rule #4 a special case of the universal behavior; both write the same row in the same format.

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

## Dependencies

Shared capabilities from `_shared/nodes/` (resolve via `{{VAULT_ROOT}}/05 AI/CLAUDE CODE/skills/_shared/nodes/{name}.md`):
- [[brief-updater]] — mode: log-entry, continuation-prompt, task-insert, wf-update

## Related Skills
- [[meeting-agenda]] — link minutes to agenda
- [[followup]] — extract WF items from meeting decisions
