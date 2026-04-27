# /debrief Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a `/debrief` skill in `skills/debrief/SKILL.md` that scans the in-context Claude Code session for project-specific friction and proposes CLAUDE.md edits, biased toward zero proposals.

**Architecture:** Single-file markdown skill. There is no executable code, so classic TDD does not apply — the skill prompt *is* the unit under test. Validation is by **manual smoke tests** against deliberately-constructed Claude Code sessions: a "smooth session" test (the trustworthiness gate), a "plant-and-find" test, and a "hint mode" test. Iterating on the prompt text is how we fix failures.

**Tech Stack:** Markdown only. skills.sh / Agent Skills frontmatter format (matches existing skills in this repo). Targets Claude Code as host; end users install via `npx skills add weorbitant/orbitant-poc-skills --skill debrief`; during development, the engineer symlinks the skill into `~/.claude/skills/debrief` for local invocation.

---

## File structure

| File | Action | Responsibility |
|---|---|---|
| `skills/debrief/SKILL.md` | Create | The skill prompt — frontmatter, when-to-use, detection logic, output format, application logic, edge cases. |
| `README.md` | Modify | Add a `### debrief` section under "Available Skills" so the skill is discoverable when users browse the repo. |

The skill is one file. Sections within it are added incrementally across tasks so each commit produces a coherent intermediate state.

**Note on testing:** The validation tasks (Tasks 6–8) are **manual** and run inside Claude Code, not from a test runner. Each one specifies an exact session to construct and an expected output to verify. If the output is wrong, the fix is to edit the skill prompt and re-run.

**Note on branching:** All work in Tasks 1–10 lands on the `feat/debrief-skill` branch created in Task 0. The plan does **not** push or open a PR — that is left to the engineer after Task 10 completes.

---

## Task 0: Create feature branch

**Files:** None modified. Git state change only — create and check out `feat/debrief-skill` from `main`.

- [ ] **Step 1: Verify the working tree is clean**

Run: `git status --short`
Expected: empty output. If there are uncommitted changes, **stop** and ask the engineer how to proceed — they may have in-progress work that should be committed or stashed first.

- [ ] **Step 2: Verify you're on `main`**

Run: `git branch --show-current`
Expected: `main`. If not, ask the engineer before switching — they may have intentionally been on another branch.

- [ ] **Step 3: Inspect remote state (optional pull)**

Run: `git fetch && git status -sb`
Expected: shows `## main...origin/main` and either `[up to date]` or an indication of how many commits behind. If behind, run `git pull --ff-only` to update before branching. If the repo has no remote configured (this is a local POC), the fetch will fail silently — proceed without pulling.

- [ ] **Step 4: Create and check out the feature branch**

Run: `git checkout -b feat/debrief-skill`
Expected: `Switched to a new branch 'feat/debrief-skill'`

- [ ] **Step 5: Verify**

Run: `git branch --show-current`
Expected: `feat/debrief-skill`

(No commit at this step — branching is a state change, not a content change. The first commit happens at the end of Task 1.)

---

## Task 1: Create skill scaffold (frontmatter + Overview + When to Use)

**Files:**
- Create: `/Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief/SKILL.md`

- [ ] **Step 1: Verify the parent directory does not yet exist**

Run: `ls /Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief 2>/dev/null && echo EXISTS || echo OK-MISSING`
Expected: `OK-MISSING`

- [ ] **Step 2: Create the skill directory**

Run: `mkdir -p /Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief`
Expected: no output, directory created.

- [ ] **Step 3: Write the scaffold to `skills/debrief/SKILL.md`**

Create the file with this exact content:

````markdown
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

(Sections added in subsequent tasks: detection, output, application, edge cases.)
````

- [ ] **Step 4: Commit**

```bash
git add /Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief/SKILL.md
git commit -m "$(cat <<'EOF'
feat(debrief): scaffold skill with frontmatter and overview

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: Detection logic — what Claude scans for

**Files:**
- Modify: `/Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief/SKILL.md` (replace the `## Process` placeholder with the seven detection sub-steps)

- [ ] **Step 1: Replace the `## Process` placeholder section**

In `skills/debrief/SKILL.md`, replace this block:

```markdown
## Process

(Sections added in subsequent tasks: detection, output, application, edge cases.)
```

with this block:

````markdown
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

(Format defined in next section.)

### 9. Apply selected proposals

(Application logic defined in next section.)
````

- [ ] **Step 2: Verify the file is well-formed by reading it back**

Run: `head -80 /Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief/SKILL.md`
Expected: shows frontmatter, Overview, When to Use, Prerequisites, and the start of the Process section through "### 7. Bias toward zero".

- [ ] **Step 3: Commit**

```bash
git add /Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief/SKILL.md
git commit -m "$(cat <<'EOF'
feat(debrief): add detection logic with single bar and exclusions

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Output formatting (both branches)

**Files:**
- Modify: `/Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief/SKILL.md` (replace the `### 8. Output the result` placeholder)

- [ ] **Step 1: Replace the `### 8. Output the result` placeholder**

In `skills/debrief/SKILL.md`, replace this block:

```markdown
### 8. Output the result

(Format defined in next section.)
```

with this block:

````markdown
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
````

- [ ] **Step 2: Commit**

```bash
git add /Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief/SKILL.md
git commit -m "$(cat <<'EOF'
feat(debrief): add output format for proposals and clean-session paths

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Application logic + Edge cases + Important section

**Files:**
- Modify: `/Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief/SKILL.md` (replace the `### 9. Apply selected proposals` placeholder and append the `## Edge cases` and `## Important` sections at the end)

- [ ] **Step 1: Replace the `### 9. Apply selected proposals` placeholder**

In `skills/debrief/SKILL.md`, replace this block:

```markdown
### 9. Apply selected proposals

(Application logic defined in next section.)
```

with this block:

````markdown
### 9. Apply selected proposals

Wait for the engineer's reply. Once they answer:

#### "none" or no selection

Reply: *"OK, nothing applied. CLAUDE.md unchanged."* and stop.

#### "all", "1 and 3", or specific indices

For each selected proposal, write it to **the project root `CLAUDE.md`**:

1. **If `CLAUDE.md` does not exist:** Create it with a minimal scaffold:

   ```markdown
   # <repo-name>

   ## <Section name from the proposal type>

   - <proposal markdown>
   ```

   Use the repo directory name as the H1. Pick the section heading based on the proposal type — e.g. `## Conventions` for project rules, `## Project knowledge` for project knowledge.

2. **If `CLAUDE.md` exists:**
   - If the proposal clearly fits an existing section heading (e.g., a new convention rule fits an existing `## Conventions` section), append within that section.
   - Otherwise, append a new section at the bottom of the file.
   - **Never modify or delete existing content.** Append-only.

3. **Do not target subdirectory `CLAUDE.md` files** (`api/CLAUDE.md`, etc.) — always edit project root in v1. The engineer can move content manually if needed.

4. **Do not stage and do not commit.** After writing, report:

   ```
   Applied [list of proposal titles] to CLAUDE.md.
   Review the diff and commit when ready.
   ```

#### "1 with edit: <change>"

Apply the engineer's edit verbatim. Do not second-guess. If the requested change is genuinely unclear, ask **one** clarifying question, then apply.

#### Engineer pushes back ("that's not actually a rule, I just changed my mind")

Drop or revise the proposal based on the engineer's reasoning. **Do not defend the proposal.** The engineer is the source of truth.
````

- [ ] **Step 2: Append the `## Edge cases` and `## Important` sections at the end of the file**

After the existing content, append:

````markdown

## Edge cases

- **Hint matches nothing.** If `/debrief <hint>` was used but no friction matches the hint, reply:

  > I scanned for "`<hint>`" but couldn't find friction matching it. Either the topic didn't surface in this session, or your existing CLAUDE.md already covers it.

  **Do not** fabricate a match.

- **Engineer rejects all proposals.** Reply *"OK, nothing applied. CLAUDE.md unchanged."* Do not re-pitch.

- **Multiple `/debrief` calls in one session.** This skill is stateless. Re-running scans the session afresh. Already-applied items will not re-surface (caught by the dedupe pass against CLAUDE.md). Previously-rejected items *may* re-surface — that's correct: the engineer can change their mind.

- **Brand-new fresh repo with no CLAUDE.md.** The application step handles this (creates the file with a minimal scaffold). No special detection logic needed.

- **Very short session.** Run normally. If the bar isn't met, return the "session looks clean" message.

- **Session crossed many topics.** Trust the bar to filter. The per-proposal evidence requirement keeps quality up regardless of session breadth. Do not summarize every topic — surface only what meets the bar.

## Important

- **Bias toward zero.** Your default outcome is "session looks clean". Finding something is the exception that needs justification, not the rule.
- **Append-only.** Never modify or delete existing CLAUDE.md content.
- **Engineer is source of truth.** If the engineer pushes back on a proposal, drop or revise — never defend.
- **No staging, no commits.** The engineer owns the git flow.
- **No proposals beyond CLAUDE.md edits.** This skill does not propose new skills, subagents, hooks, slash commands, or settings. If the engineer asks, point out that those are deliberate-design artifacts and out of scope for `/debrief`.
````

- [ ] **Step 3: Commit**

```bash
git add /Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief/SKILL.md
git commit -m "$(cat <<'EOF'
feat(debrief): add application logic, edge cases, and important reminders

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: Local install for testing

**Files:**
- No file changes in the repo.
- Create symlink: `~/.claude/skills/debrief` → `<repo>/skills/debrief` (so Claude Code can find the skill during local development without publishing).

- [ ] **Step 1: Verify `~/.claude/skills/` directory exists, create if missing**

Run: `mkdir -p ~/.claude/skills`
Expected: no output.

- [ ] **Step 2: Verify no existing `debrief` skill is installed (avoid overwriting)**

Run: `ls -la ~/.claude/skills/debrief 2>/dev/null && echo EXISTS || echo OK-MISSING`
Expected: `OK-MISSING`. If it shows `EXISTS`, stop and ask the engineer how to proceed (they may have an older version they don't want to overwrite).

- [ ] **Step 3: Create the symlink**

Run: `ln -s /Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief ~/.claude/skills/debrief`
Expected: no output. Verify with `ls -la ~/.claude/skills/debrief` — should show the symlink target.

- [ ] **Step 4: Verify the skill is loadable**

Open a fresh Claude Code session in any directory. Type `/help` or check the skill registry — `debrief` should appear. (No commit; this is environment setup, not a repo change.)

---

## Task 6: Smoke test — smooth session (the trustworthiness gate)

**Files:**
- No file changes unless the test fails. If it fails, edit `skills/debrief/SKILL.md` to fix the prompt and re-run.

This is **the most important test.** If the skill invents proposals on a clean session, it's broken regardless of what else works.

- [ ] **Step 1: Construct a deliberately clean Claude Code session**

In a separate terminal, open a fresh Claude Code session in this repo. Have a normal interaction with no corrections — for example: ask Claude to read `README.md` and summarize it. Do not correct or push back at any point.

- [ ] **Step 2: Run `/debrief`**

In that same session, type `/debrief`.

- [ ] **Step 3: Verify expected output**

**Expected:** Claude returns *exactly* the "Session looks clean" message defined in Task 3:

```
Session looks clean. No project rules or knowledge gaps surfaced — your CLAUDE.md held up.

If something nagged you that I missed, run `/debrief <hint>` (e.g. `/debrief the import ordering thing`) and I'll focus there.
```

**FAIL conditions** (any of these means the prompt is wrong):
- Claude lists one or more "proposals" despite no corrections in the session.
- Claude hedges with "but here's something just in case…" or similar.
- Claude apologizes for not finding anything.
- Claude proposes generic/industry-wide rules ("you should write tests", "use TypeScript strict mode").

- [ ] **Step 4: If FAIL, iterate on the prompt**

Edit `skills/debrief/SKILL.md`:
- Strengthen the language in `### 7. Bias toward zero` and the `## Important` section.
- Re-emphasize the evidence requirement (`### 6`) — no quote, no proposal.
- Tighten the exclusions list (`### 4`) if Claude is using a banned category.

After editing, commit, then **re-run from Step 1 in a new fresh session** (the prompt changes don't affect the session that already ran).

- [ ] **Step 5: Commit any prompt edits made during iteration**

```bash
git add /Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief/SKILL.md
git commit -m "$(cat <<'EOF'
fix(debrief): strengthen bias-toward-zero based on smooth-session test

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

If no edits were needed, skip this step.

---

## Task 7: Smoke test — plant-and-find

**Files:**
- No file changes unless the test fails.

- [ ] **Step 1: Construct a session with one planted correction**

In a fresh Claude Code session in this repo (or any test repo), have a normal interaction *plus* one deliberate correction. For example: ask Claude to write a small TypeScript snippet using `let`. After Claude responds, say:

> Actually, in this project we never use `let` — only `const`. Reactive variables aside, mutable bindings are a code smell here.

Continue the session naturally for a few more turns.

- [ ] **Step 2: Run `/debrief`**

Type `/debrief`.

- [ ] **Step 3: Verify expected output**

**Expected:** Claude returns a message starting with `Found 1 moment worth capturing.` (or `Found N moments…` if other friction surfaced incidentally), and the proposal includes:

- `**Type:** project rule`
- `**Evidence:** "...we never use \`let\` — only \`const\`..."` (or similar quote)
- A `**Proposed CLAUDE.md addition:**` blockquote with markdown text capturing the rule.

**FAIL conditions:**
- The planted rule is missing from the output.
- The evidence field doesn't quote the engineer's correction.
- The proposed addition is vague (e.g., "use const where possible") instead of crisp ("Use `const` exclusively in this project — no `let`.").
- The output lacks the `Tell me which to apply:` line.

- [ ] **Step 4: If FAIL, iterate**

Edit `skills/debrief/SKILL.md`:
- If the rule was missed: revisit `### 2. Scan the session for friction` and `### 3. Apply the bar` — Claude may not be recognizing explicit corrections.
- If evidence is weak: strengthen `### 6. Require evidence per surviving candidate` — make the quoting requirement more explicit.
- If formatting is off: re-check the output template in `### 8. Output the result`.

Commit and re-run from Step 1.

- [ ] **Step 5: Test the "1 with edit:" path**

After a successful proposal, reply: *"Apply 1 with edit: phrase it as 'Use `const` exclusively. `let` is never acceptable.'"*

Verify Claude applies the engineer's exact wording to CLAUDE.md without second-guessing.

- [ ] **Step 6: Commit any prompt fixes**

```bash
git add /Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief/SKILL.md
git commit -m "$(cat <<'EOF'
fix(debrief): tighten detection/output based on plant-and-find test

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

If no edits were needed, skip.

---

## Task 8: Smoke test — hint mode

**Files:**
- No file changes unless the test fails.

- [ ] **Step 1: Construct a session with multiple unrelated frictions**

In a fresh session, have an interaction containing **two distinct corrections** on different topics. For example:

- Correction A: *"snake_case for API field names — never camelCase here."*
- Correction B: *"we use Vitest, not Jest, for tests."*

- [ ] **Step 2: Run `/debrief` with a hint targeting only one**

Type: `/debrief the API naming thing`

- [ ] **Step 3: Verify expected output**

**Expected:** Claude returns proposals **only for the API naming correction** (Correction A). The Vitest correction (Correction B) is **not** surfaced because it's outside the hint scope.

**FAIL conditions:**
- Claude surfaces both corrections (hint scope ignored).
- Claude surfaces neither (hint mode failed even when there was a match).
- Claude surfaces a hallucinated match for the hint when no real friction matched.

- [ ] **Step 4: Run `/debrief <hint>` with a hint that matches nothing**

In the same session, type: `/debrief the build pipeline thing`

**Expected:**

```
I scanned for "the build pipeline thing" but couldn't find friction matching it. Either the topic didn't surface in this session, or your existing CLAUDE.md already covers it.
```

(Or wording very close to it. Do not fabricate a match.)

- [ ] **Step 5: If any FAIL, iterate**

Edit `skills/debrief/SKILL.md`:
- Strengthen the hint-mode handling. Add explicit guidance in `### 2. Scan the session for friction`: *"If a hint is provided, restrict candidates to those matching the hint topic. Drop everything else, even if it would otherwise pass the bar."*
- Strengthen the "hint matches nothing" edge case in the `## Edge cases` section.

Commit and re-run.

- [ ] **Step 6: Commit any fixes**

```bash
git add /Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief/SKILL.md
git commit -m "$(cat <<'EOF'
fix(debrief): tighten hint-mode scoping based on hint-mode test

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

If no edits were needed, skip.

---

## Task 9: Update README

**Files:**
- Modify: `/Users/kevinccbsg/orbitant/orbitant-poc-skills/README.md` (add a `### debrief` section under "Available Skills")

- [ ] **Step 1: Read the README to find the insertion point**

Run: `grep -n "Available Skills\|^### \|<!-- Add new skills" /Users/kevinccbsg/orbitant/orbitant-poc-skills/README.md`

Expected: shows the `## Available Skills` line, the existing `### learn` heading, and the `<!-- Add new skills below ... -->` HTML comment near the end.

- [ ] **Step 2: Insert the `### debrief` section directly above the HTML comment**

In `README.md`, find this block (near the end of "Available Skills"):

```markdown
<!-- Add new skills below as a new H3 section. Keep each one self-contained so
     it can be understood without reading the others. -->
```

Insert this block immediately **before** that comment:

````markdown
### debrief

A bottom-up friction-capture skill. Run at the end of a Claude Code session to
turn the corrections you made and the project knowledge you had to volunteer
into proposed `CLAUDE.md` edits — so the next session starts higher up the
curve. Designed to be high-signal: returning *"session looks clean"* with no
proposals is a first-class outcome, not a fallback.

**Use when:**
- "/debrief" / "/debrief the naming convention thing"
- "wrap up this session" / "what did we learn this session"
- "any rules to capture" / "should we update CLAUDE.md"

**Commands:**

| Command | Purpose |
|---|---|
| `/debrief` | Scan the full session for friction, propose `CLAUDE.md` edits. |
| `/debrief <hint>` | Same, but focused on the area named (e.g., `/debrief the API naming convention thing`). |

**Output:** Either one message listing N proposed edits with cited evidence per
proposal — which the engineer approves all/some/with-edits/none of — **or** a
celebratory *"session looks clean"* message if nothing meets the bar. Approved
edits are appended to the project root `CLAUDE.md`. The engineer commits via
their normal git flow.

**v1 scope:** CLAUDE.md edits only. No skills, subagents, hooks, slash commands,
or settings proposals. No subdirectory `CLAUDE.md` targeting. No SessionEnd
hook reminder. Stateless across runs.

````

- [ ] **Step 3: Verify the README parses cleanly**

Run: `head -120 /Users/kevinccbsg/orbitant/orbitant-poc-skills/README.md`
Expected: shows the existing `### learn` section followed by the new `### debrief` section, then the `<!-- Add new skills below ... -->` comment, then the "Contributing a skill" section.

- [ ] **Step 4: Commit**

```bash
git add /Users/kevinccbsg/orbitant/orbitant-poc-skills/README.md
git commit -m "$(cat <<'EOF'
docs: add debrief skill to README

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 10: Final consistency review

**Files:**
- Modify (only if needed): `/Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief/SKILL.md`

- [ ] **Step 1: Re-read the entire SKILL.md end-to-end**

Run: `cat /Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief/SKILL.md`

Check for:
- The `### N.` numbered process steps are sequential (1, 2, 3, …, 9) with no gaps or duplicates.
- The output template in `### 8` matches what's referenced in `### 9` (e.g., the `Tell me which to apply:` line).
- The `## Important` section's bullets are consistent with the body — no contradictions (e.g., "append-only" must not contradict any earlier instruction).
- The frontmatter description still accurately describes the skill (no drift after iteration).

- [ ] **Step 2: Cross-check against the spec**

Run: `diff -u <(grep -E "^##|^###" /Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief/SKILL.md) <(grep -E "^##|^###" /Users/kevinccbsg/orbitant/orbitant-poc-skills/docs/superpowers/specs/2026-04-27-debrief-skill-design.md)`

(This is just a structural check — section counts will differ; the goal is a sanity pass that the skill covers all the spec's design points. Eyeball the diff for surprises.)

For each spec section, confirm there's a corresponding behavior in the skill:
- Spec: "Detection bar (single principle)" → SKILL.md `### 3. Apply the bar` ✓
- Spec: "Skill-execution friction counts the same" → SKILL.md `### 5` ✓
- Spec: "Bias toward zero" → SKILL.md `### 7` and `## Important` ✓
- Spec: "Output: when proposals exist / when nothing surfaces" → SKILL.md `### 8` (both branches) ✓
- Spec: "Append-only, root only, no auto-commit" → SKILL.md `### 9` ✓
- Spec: "Edge cases (hint matches nothing, rejection, push-back, fresh repo, multiple debriefs, short session, broad session)" → SKILL.md `## Edge cases` ✓

If any item is missing, edit the SKILL.md to add it.

- [ ] **Step 3: Commit any consistency fixes**

```bash
git add /Users/kevinccbsg/orbitant/orbitant-poc-skills/skills/debrief/SKILL.md
git commit -m "$(cat <<'EOF'
chore(debrief): final consistency pass against spec

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

If no fixes, skip the commit.

- [ ] **Step 4: Report completion**

Tell the engineer:

> debrief skill is implemented, validated against three smoke tests, and added to the README. Symlink at `~/.claude/skills/debrief` for local use; end users will install via `npx skills add weorbitant/orbitant-poc-skills --skill debrief` once published.
>
> Suggested next: run `/debrief` after your next real Claude Code session and see what surfaces.

---

## Self-review (executed by plan author after writing the plan)

**1. Spec coverage:**
- Goals: CLAUDE.md edits only ✓ (Tasks 2–4 enforce this)
- Goals: bias toward zero ✓ (Task 2 step 7 + Task 6 smoke test gate)
- Goals: cited evidence per proposal ✓ (Task 2 step 6 + Task 3 output template)
- Goals: append-only ✓ (Task 4)
- Goals: engineer-driven approval ✓ (Task 4)
- Non-goals (other artifact types, subdirectory CLAUDE.md, SessionEnd hook, skill-cheatsheet graduation, statefulness, cross-project) ✓ (explicitly called out in `## Important` section + Task 9 README scope note)
- Detection bar single principle ✓ (Task 2 step 3)
- Two categories (rules + knowledge) ✓ (Task 2 step 2)
- Exclusions list ✓ (Task 2 step 4)
- Skill-execution friction counts same ✓ (Task 2 step 5)
- Output shapes (both branches) ✓ (Task 3)
- Application logic (no CLAUDE.md / existing CLAUDE.md / append-only / no auto-commit / engineer push-back) ✓ (Task 4)
- All edge cases ✓ (Task 4)
- Validation (smooth-session as gate, plant-and-find, hint mode) ✓ (Tasks 6, 7, 8)

**2. Placeholder scan:** No "TBD" / "implement later" / "similar to Task N" / "fill in details" found. Every code/markdown step contains the literal content to write.

**3. Type consistency:** No types or function signatures in this skill (markdown only). Section headings used in later tasks (`### 8. Output the result`, `### 9. Apply selected proposals`) match the placeholder names used in Task 2. The output template in Task 3 matches the application-flow handoff in Task 4 (`Tell me which to apply:` line is the contract).

**4. Granularity check:** Each step is one atomic action (write a file, run a command, observe output, commit). All steps fit in 2-5 minutes.

**5. Branch hygiene:** Task 0 establishes the `feat/debrief-skill` branch before any code changes; all subsequent commits land there. No push or PR is included — the engineer owns that final step.

---

## Execution handoff

**Plan complete and saved to `docs/superpowers/plans/2026-04-27-debrief-skill.md`.** Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration. Best for tasks where each one's output is self-contained.

**2. Inline Execution** — Execute tasks in this session using `superpowers:executing-plans`, batch execution with checkpoints. Best when continuity (prior reasoning, error recovery) matters more than isolation.

**Notes for the executor:**
- **Task 0** is the branch-creation step. Run it first; do not start Task 1 on `main`.
- **Tasks 6–8** (smoke tests) require **interactive Claude Code sessions** outside this plan's runtime. The executor can prepare and execute Tasks 0–5 and 9–10 autonomously, but Tasks 6–8 require the engineer to drive the test sessions and report back.
- **No push, no PR.** The plan ends with the engineer reviewing the branch locally. Push and PR are explicit next steps the engineer requests on their own.

Which approach?
