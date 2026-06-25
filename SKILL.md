---
name: wrap
description: End-of-session housekeeping. Discovers the project structure, updates the session log, primary context file, task tracker, and memory, then commits to git. Adapts to any project type without configuration.
argument-hint: "[optional note to include in the session summary]"
allowed-tools: Bash, Read, Edit, Write
user-invocable: true
author: eapreko
---

# /wrap — Session Wrap-up

Perform end-of-session housekeeping for the current project. **Be decisive — do not ask the user questions.** Read what you find, act on it, and report back.

---

## Phase 0: Discover the project

Run these in parallel before acting on anything.

**1. Primary context file** — find the first match, in this order:
- `CLAUDE.md`
- `.claude/overview.md`
- `README.md`

If the match is `CLAUDE.md` and its full content is already visible in this conversation's auto-injected system reminder, and nothing has edited it earlier this session, skip the full read — issue a `Read` with `limit: 5` instead, just to satisfy the "must Read before Edit" tool gate that Phase 2/3 will need. Otherwise (the match is `.claude/overview.md` or `README.md`, CLAUDE.md isn't in the injected context, or it may have changed already this session) read it in full.

This file tells you what the project is, what's outstanding, and how it is structured. If none exists, note that and skip steps that depend on it.

**2. Session log** — look for the first match:
- `Session Log.md` (project root)
- `SESSIONS.md`
- `CHANGELOG.md`
- Any `.md` at the project root with "session", "log", or "changelog" in the filename
- Any `.md` inside `.claude/` with "session", "log", or "changelog" in the filename

Note the path — Phase 1 appends to wherever the log was found. If none exists, create `Session Log.md` at the project root in Phase 1.

**3. Task tracker** — look for any file (case-insensitive) whose name contains: `tracker`, `action`, `todo`, `backlog`, or `tasks`. Note its path. If none found, skip Phase 4.

**4. Git state** — run `git status --short` and `git remote -v`. Note:
- Whether a remote exists
- Whether any staged/unstaged files suggest sensitive content (medical records, credentials, `.env` files, private keys, personal financial or legal records) that would make pushing unsafe

**5. Memory index** — `~/.claude/projects/<encoded-cwd>/memory/MEMORY.md` (cwd path components joined by `-`). If its content is already visible in this conversation's auto-injected system reminder, and nothing has edited it earlier this session, skip the full read — issue a `Read` with `limit: 5` instead, just to satisfy the "must Read before Edit" tool gate that Phase 5 will need. Otherwise read it in full. Either way, this primes Phase 5 with what memories already exist so you update rather than duplicate.

---

## Phase 1: Update the session log

If the log already has content, `Read` only its first ~20 lines (use the `limit` parameter) — enough to capture the existing top entry's heading and the line immediately after it (e.g. its `**Topics:**` line); do not read the whole file. Anchor `Edit`'s `old_string` on both lines together, not the heading alone — identical headings recur in practice (e.g. multiple same-day "(continued)" entries), and a heading-only anchor can match more than once file-wide and fail. Insert the new entry directly above that anchor, keeping all existing entries intact below it, untouched and unread. If no log exists, create `Session Log.md` with this as the first entry.

Use this structure — omit any section that has nothing to record:

    ## Session: DD Month YYYY
    **Topics:** <3–6 word subject tags, comma-separated — e.g. "auth refactor, CI pipeline, OT search">

    ### What was accomplished
    - <concrete bullet points, past tense>

    ### How key problems were solved
    - <problem> → <approach taken> — only include if non-obvious; skip if everything was routine>

    ### New critical findings
    1. <numbered — only genuinely new, significant discoveries>

    ### Files created
    - `path/to/file` — one-line description

    ### Files received / newly in repo
    - `path/to/file` — one-line description

    ### Immediate next actions
    - [ ] <action>
    - [ ] <action>

---

## Phase 2: Update the primary context file

Surgical edits only — do not restructure, reformat, or rewrite existing content.

- Change `- [ ]` to `- [x]` for actions completed this session
- Add new actions that emerged, in the appropriate section
- Update any status fields, contact details, or configuration that changed
- If the file has a "recent work", "what we've done", or equivalent section, append a line for significant new work
- Update cost figures, deadlines, or key facts if they changed

---

## Phase 3: Declutter agent files

Scan CLAUDE.md (and any `.claude/` instruction files found in Phase 0) for content that is safe to remove. **Be conservative** — only act on high-confidence cases. When in doubt, flag rather than delete.

**Auto-remove (no user confirmation needed):**
- Exact duplicate lines or paragraphs within the same file
- An instruction that is explicitly superseded by a newer one in the same file (e.g., an old model ID replaced on the next line by a newer one)

**Flag in the Phase 9 report, do not auto-remove:**
- References to tasks, branches, or projects that appear completed or closed (user may want the record)
- Instructions that look redundant with another but differ subtly in wording — list both so the user can decide
- Any instruction added in a prior session whose rationale is no longer visible

Make edits surgically with Edit, preserving all other content. If nothing qualifies for either category, skip this phase silently.

---

## Phase 4: Update the task tracker (if found in Phase 0)

- Tick off completed items
- Add new items with owner and deadline where known
- Update status of in-progress items
- Correct any figures or contact details that changed

---

## Phase 5: Update memory

Memory lives at `~/.claude/projects/<encoded-path>/memory/`.

**What to update:**
- `project_*.md` — if material facts, strategy, state, or key figures changed this session
- Write a new `feedback_*.md` — if the user corrected your approach or confirmed a non-obvious working pattern. **Ask yourself explicitly:** did you change your approach mid-session because of user feedback or an unexpected finding? If yes, write the memory now — do not rely on recalling it later.
- Write a new `reference_*.md` — if a significant external URL, credential, or resource was introduced
- Update `MEMORY.md` — add one line per new file; update the hook for changed files

**Staleness check (cheap pass):**
- Use the MEMORY.md index from Phase 0 step 5 (auto-injected content or fresh read, whichever applied) — do not open every memory file. Skim each one-line entry for anything this session's events directly contradict or supersede (e.g., a referenced branch was merged or deleted, a status the entry describes as "current" just changed, a fact the user corrected in passing).
- Open the specific memory file only when its index line looks contradicted by something concrete from this session.
- If confirmed stale with high confidence (the fact is now verifiably wrong) — update the memory file in place and note the change in the Phase 9 report.
- If merely ambiguous (looks old, but you can't confirm it's wrong) — leave it and flag it in the Phase 9 report instead, same conservative bar as Phase 3. Never delete a memory file on a hunch.
- Skip this step silently if nothing in the index looks contradicted.

**Memory file format:**

    ---
    name: <short-kebab-case-slug>
    description: <one-line summary>
    metadata:
      type: user | feedback | project | reference
    ---

    <the fact>

    **Why:** <why this matters>
    **How to apply:** <how to act on it next time>

    [[related-memory-name]]

Before creating a new memory, check whether an existing one already covers it — update rather than duplicate.

---

## Phase 6: Log technical debt (code projects only)

Skip this phase entirely if the project contains no source code (e.g. document-only, legal, personal records).

If the project is a software/code project, scan `git status --short` and your memory of the session for anything left unresolved:

- Unfinished refactors, hacks, or workarounds introduced this session
- Tests skipped or commented out
- TODOs left in code
- Known issues deferred to a future session

Add these to the task tracker (if one exists) or append a `### Technical debt` section to the session log entry. One line per item is enough — the goal is a retrievable record, not an essay.

---

## Phase 7: Surface automation opportunities

Review the current session context for repeated manual sequences — any multi-step task performed two or more times, or that the user explicitly described as recurring ("I always...", "every time I...", "we do this each session").

**Only act if a pattern meets the 2× threshold.** One-off tasks do not qualify — skip this phase entirely if none do.

**Classify the strongest candidate:**
- **Hook** — a trigger that fires automatically (e.g., before/after a tool call); the artifact is a config snippet for `settings.json`
- **Skill** — a reusable `/command` wrapping a multi-step flow; the artifact is a `SKILL.md` stub
- **Prompt File** — a reusable instruction block for a recurring task type; the artifact is a `.md` template

**Draft one artifact only** — highest-ROI candidate — to `_drafts/<slug>.md`. Open with a single comment line: what it automates and the estimated sessions-saved.

**Cross-session memory:**
- Check existing memory for `feedback_pattern_*.md` matching this pattern. If found, this is a confirmed recurrence — note it as high priority in the Phase 9 report.
- If this is the first sighting, write `~/.claude/projects/<encoded-cwd>/memory/feedback_pattern_<slug>.md` recording: the pattern, today's date, and which artifact type would address it. Next session confirms recurrence before a full draft is committed.

---

## Phase 8: Commit to git

Stage all changed tracked files and any new files created this session.

**Push decision — apply the first matching rule:**
1. No remote configured → commit only, do not push
2. Project contains sensitive personal data (medical, legal, financial, credentials) → commit only, do not push — note the reason in the report
3. Remote exists and no sensitive data concerns → commit and push to the default remote branch

**Commit message format:**

    Session wrap-up DD Mon YYYY: <one-line summary>

    - <key thing done or changed>
    - <key thing done or changed>

If the user passed an argument when invoking `/wrap`, incorporate it into the summary line.

---

## Phase 9: Report back

One short paragraph covering:
- What files were updated
- What was committed (hash) and whether it was pushed or kept local (and why)
- Any automation opportunity drafted (Phase 7) — what it is and where the draft lives
- Any items flagged for user review (Phase 3 declutter candidates, Phase 5 stale memory candidates) — list them so the user can decide
- The single most important next action for the next session
