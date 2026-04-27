---
name: debrief
description: |
  Use at the end of a Claude Code session to capture project-specific friction
  as proposed CLAUDE.md edits. Triggered by /debrief (no args) to scan the full
  session, or /debrief <hint> to focus on a specific area
  (e.g., /debrief the API naming convention thing). Returning "session looks
  clean" with no proposals is a first-class outcome, not a fallback. Triggers
  on: "/debrief", "debrief the session", "wrap up this session", "what did we
  learn", "any rules to capture", "should we update CLAUDE.md".
license: MIT
version: "0.1.0"
metadata:
  author: orbitant
  tags: claude-md, retrospective, friction-capture, session-end, orbitant-engineering
---

## Overview

A bottom-up friction-capture skill. At the end of a Claude Code session, scan the conversation already in context, identify project-specific friction (project rules and project knowledge), and propose CLAUDE.md edits the engineer can approve. Bias toward zero proposals — returning "session looks clean" is a first-class outcome, not a fallback.

This skill is part of the future Orbitant `orbitant-engineering` plugin's bottom-up half. It is decoupled from `/ground-control` (the top-down half, not yet built).

## When to Use

- The engineer types `/debrief` (no args) at session end.
- The engineer types `/debrief <hint>` to focus on a specific area (e.g. `/debrief the API naming convention thing`).
- Skip with the message *"No active session content to debrief — run me at the end of a session that had real engineering work."* if the session has no engineering work in it (e.g. a brand-new chat with only this `/debrief` command).

## Prerequisites

- The skill assumes Claude has the full Claude Code session already in context. It does not re-read the session from disk.
- The project may or may not have a `CLAUDE.md`. The skill handles both cases (see the application step).

## Process

### 1. Read the existing CLAUDE.md

Before proposing anything, read the project's `CLAUDE.md` if it exists. Hold its contents in mind so you can dedupe — never propose something already covered.

If the project has subdirectory `CLAUDE.md` files (e.g. `api/CLAUDE.md`, `mobile/CLAUDE.md`), read those too. They count for dedupe.

If no `CLAUDE.md` exists at all, that's fine — proceed. The application step (later) will offer to create one.

### 2. Scan the session for friction

Look for two narrow categories. **Nothing else qualifies.**

**Category A — Project rules.** Explicit corrections that signal a project-wide convention.

Examples:
- *"we don't use moment.js"*
- *"snake_case for API fields, not camelCase"*
- *"don't add try/catch around X"*
- *"we always write tests in Vitest, not Jest"*

**Category B — Project knowledge.** Facts about the codebase Claude lacked, that the engineer had to volunteer.

Examples:
- *"the API is at /api/v2/"*
- *"auth tokens live in the `tokens` table"*
- *"run `npm run gen` after editing schemas"*
- *"the dev server requires DATABASE_URL in .env.local"*

### 3. Apply the bar

For each candidate, ask: **Would adding this to CLAUDE.md have changed Claude's behavior in this session?** If no, drop it. The bar is single, mechanical, evidence-driven. There is no "must see N times" threshold — the principle handles it.

### 4. Apply the exclusions

Drop any candidate that fits any of these:

- **Engineer changed their mind mid-session.** *("Actually, let's not do X."* — that's a reversal, not a rule.)
- **One-off bugs Claude fixed after one prompt.** (Wouldn't have been prevented by CLAUDE.md.)
- **Style nudges Claude got right after one correction.** (Already responsive — no future risk.)
- **General coding wisdom.** (*"Don't use `var`"* is true everywhere; CLAUDE.md is for project-specifics.)
- **Anything already covered in existing CLAUDE.md** (deduped in Step 1).

### 5. Skill-execution friction counts the same

If a correction surfaced *during* a skill invocation (e.g., the engineer corrected you while running `/twd` or another skill), it counts the same as any other correction. **Do not** discount it as "just clarifying the skill's instructions". The bar is still: would it have changed your behavior if it had been in CLAUDE.md? If yes, it qualifies.

### 6. Require evidence per surviving candidate

Every surviving candidate must have at least one quoted user message or a concrete missing-knowledge moment from the session. **No evidence → no proposal.** This makes the empty-result outcome mechanical, not a judgment call.

### 7. Bias toward zero

If you find nothing that meets the bar, **that is the correct outcome**. Do not invent proposals to feel productive. A clean session is a successful one. Returning zero proposals is celebrated, not apologized for.

### 8. Output the result

#### When proposals exist

Return ONE message in this exact shape — and nothing else:

```
Found N moments worth capturing.

## 1. <short title>
**Type:** project rule | project knowledge
**Evidence:** "<quoted user message or specific session moment>"
**Proposed CLAUDE.md addition:**

> <exact markdown that would land in CLAUDE.md>

## 2. ...

---
Tell me which to apply: "all", "1 and 3", "1 with edit: <change>", or "none".
```

Each proposal **must** include the four labelled fields:

- `**Type:**` — exactly `project rule` or `project knowledge`.
- `**Evidence:**` — at least one quoted user message OR a concrete description of the missing-knowledge moment. The engineer must be able to recognize the moment without scrolling.
- `**Proposed CLAUDE.md addition:**` followed by a blockquoted markdown block — the **literal text** that would land in CLAUDE.md, not a description of it. The engineer reads this verbatim, so it must be ready-to-paste.
- The `---` separator and the `Tell me which to apply:` line at the very end.

#### When nothing surfaces (the celebrated path)

Return this message — exactly this, no embellishment, no additional commentary:

```
Session looks clean. No project rules or knowledge gaps surfaced — your CLAUDE.md held up.

If something nagged you that I missed, run `/debrief <hint>` (e.g. `/debrief the import ordering thing`) and I'll focus there.
```

This is the **default** outcome the skill is biased toward. Do not append "but here's something just in case…" or hedge — a clean session is the good outcome.

### 9. Apply selected proposals

(Application logic defined in next section.)
