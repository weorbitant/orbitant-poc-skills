# /debrief Skill — Design

**Date:** 2026-04-27
**Status:** Approved (awaiting implementation plan)
**Authors:** Orbitant + Claude (via brainstorming)

## Summary

A Claude Code skill, invoked as `/debrief` at the end of a session, that reflects over the session already in context, identifies project-specific friction, and proposes one or more **CLAUDE.md edits** for the engineer to approve. Designed to be high-signal: returning "nothing to capture" is a first-class, celebrated outcome.

This is the v1 of the bottom-up half of the future Orbitant `orbitant-engineering` plugin. It ships as a standalone POC skill in this repo and is decoupled from `/ground-control` (top-down).

## Context & motivation

Orbitant is a consultancy with many client projects, each with its own conventions, stack quirks, and undocumented rules. Engineers using Claude Code repeatedly correct Claude on the same project-specific things ("we don't use moment.js", "snake_case for API fields") — and that learning is lost when the session ends.

`/debrief` captures that friction at zero extra token cost (the session is already in context) and turns it into a versioned `.claude/CLAUDE.md` artifact, so the next session — for the same engineer or a teammate — starts higher up the curve.

Over time, when the same proposed edit shows up across multiple Orbitant client projects, a tech lead can promote it to an Orbitant-wide standard in the future `orbitant-engineering` plugin's `references/`. That promotion path is **out of scope for this skill**, but the skill is designed to feed into it.

## Goals

- Capture project-specific friction from the session, with cited evidence per proposal.
- Produce **few, high-quality proposals** the engineer trusts. Bias toward zero.
- Restrict outputs to **CLAUDE.md edits only** — the simplest, lowest-risk artifact.
- Append-only edits to root `CLAUDE.md`. Never modify or delete existing content.
- Engineer-driven approval flow. The skill never writes without explicit approval.

## Non-goals (v1)

- **Other artifact types** — no proposals for skills, subagents, hooks, slash commands, or settings. Captured friction that *might* warrant those is reported as "out of scope; consider designing this deliberately" if the engineer asks; otherwise dropped.
- **Subdirectory CLAUDE.md.** Always target project root. Engineer can move content manually if needed.
- **SessionEnd hook reminder** ("run /debrief before closing"). Skipped pending the skill earning its place. Easy to add later via the existing `update-config` skill.
- **Skill-cheatsheet graduation** — the skill does *not* try to "graduate" content from invoked skills (e.g. `/twd`) into CLAUDE.md to avoid re-loading the skill. That is a skill-redesign concern, not a friction-capture one.
- **Stateful tracking across debriefs.** No persisted log of past proposals. The skill is stateless; re-running scans the session afresh. Dedupe is handled by reading the current CLAUDE.md before proposing.
- **Cross-project pattern detection.** Single session, single project. No correlation across repos.
- **Automated tests.** Validation is manual (see "Validation").

## Design principles

1. **Bias toward zero.** The default outcome is "session was smooth, nothing to capture". Finding something is the exception that needs justification, not the default.
2. **Evidence-required.** No proposal without a quoted user message or a concrete missing-knowledge moment from the session. This makes "no evidence → no proposal" mechanical, not a judgment call.
3. **One artifact type.** CLAUDE.md edits only. No drift into agents/skills/hooks.
4. **Append-only.** Never modify or delete existing CLAUDE.md content.
5. **Engineer is source of truth.** If the engineer pushes back on a proposal, drop or revise — never defend.

## Skill structure

### Location

`skills/debrief/SKILL.md` — single-file skill following the skills.sh / Agent Skills format used elsewhere in this repo.

### Frontmatter

```yaml
---
name: debrief
description: |
  Use at the end of a Claude Code session to capture project-specific friction
  as proposed CLAUDE.md edits. Triggered by /debrief (no args) or
  /debrief <hint> (focus on a specific area, e.g.,
  /debrief the API naming convention thing).
license: MIT
version: "0.1.0"
metadata:
  author: orbitant
  tags: claude-md, retrospective, friction-capture, session-end, orbitant-engineering
---
```

### Invocation

- `/debrief` — no argument. Claude scans the full session in context. The 80% case.
- `/debrief <free-text hint>` — Claude focuses on the area named. Implementation-wise, the hint is a single conditional branch in the skill prompt; no separate code path.

## Detection bar

The skill instructs Claude to scan the session for two narrow categories of friction:

1. **Project rules** — explicit corrections that signal a project-wide convention.
   Examples: *"we don't use moment.js"*, *"snake_case for API fields, not camelCase"*, *"don't add try/catch around X"*.
2. **Project knowledge** — facts about the codebase Claude lacked, that the engineer had to volunteer.
   Examples: *"we use Vitest not Jest"*, *"the API is at /api/v2/"*, *"auth tokens live in the `tokens` table"*.

### The principle (single bar)

> **Would adding this to CLAUDE.md have changed Claude's behavior in this session?**
> If no, drop it.

A single explicit correction passes the bar if it clearly signals a project rule. An ambient preference does not — one ambient instance won't pass the "would have changed behavior" test.

### Explicit exclusions in the skill prompt

To prevent drift, the skill prompt explicitly excludes:

- Engineer changing their mind mid-session — not a rule.
- One-off bugs Claude fixed after one prompt — not preventable by CLAUDE.md.
- Style nudges Claude got right after one correction — Claude was already responsive, so capture would be redundant.
- General coding wisdom — CLAUDE.md is project-specific, not industry-wide.
- Anything already covered in the existing CLAUDE.md (caught by the dedupe pass).

### Skill-execution friction

Corrections that surface *during* a skill invocation (e.g., the engineer corrects Claude while running `/twd`) count the same as any other correction. The skill prompt explicitly states this so Claude doesn't discount them as *"just clarifying the skill"*.

### Dedupe

Before proposing, the skill reads the project's CLAUDE.md (and any subdirectory CLAUDE.md files) and drops any candidate already covered there. This prevents re-proposing the same rule on every debrief.

## Output & application

### When proposals exist

Claude returns one message in this shape:

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

### When nothing surfaces (the celebrated path)

```
Session looks clean. No project rules or knowledge gaps surfaced — your CLAUDE.md held up.

If something nagged you that I missed, run `/debrief <hint>` (e.g. `/debrief the import ordering thing`) and I'll focus there.
```

### Application logic (after engineer picks)

1. **No CLAUDE.md exists?** Offer to create it with a minimal scaffold (`# <repo-name>` header) plus the approved entries.
2. **CLAUDE.md exists?**
   - If a proposal clearly fits an existing section heading (e.g., a new convention rule fits an existing `## Conventions` section), append within that section.
   - Otherwise, append a new section at the bottom.
   - **Never modify or delete existing content.**
3. **Subdirectory CLAUDE.md is not a target in v1.** Always edit project root.
4. After writing, Claude reports:
   *"Applied X to CLAUDE.md. Review the diff and commit when ready."*
5. **No staging, no auto-commit.** Engineer owns the git flow.

## Edge cases

| Case | Behavior |
|---|---|
| Hint matches nothing in session | Return *"I scanned for `<hint>` but couldn't find friction matching it. Either the topic didn't surface in this session, or your existing CLAUDE.md already covers it."* Do not fabricate a match. |
| Engineer rejects all proposals | Reply *"OK, nothing applied. CLAUDE.md unchanged."* Do not re-pitch. |
| Engineer pushes back on a specific proposal | Drop or revise based on engineer's reasoning. Do not defend. The engineer is the source of truth. |
| Fresh repo with no CLAUDE.md | Offer to create it (covered above). |
| Multiple `/debrief` calls in one session | Skill is stateless — re-running scans the full session afresh. Already-applied items won't re-surface (caught by dedupe). Previously-rejected items *may* re-surface; that's correct (engineer can change their mind). |
| Very short session with little content | Run normally. If the bar isn't met, return the "session looks clean" message. No special handling. |
| Session crossed many topics | Trust the bar to filter. Per-proposal evidence requirement keeps quality up regardless of session breadth. |

## Validation

There is no automated test for a skill that reasons over conversation context. Validation is **manual and observational**:

1. **Self-test on this session.** Run `/debrief` on the session that built the skill itself. Does it surface anything sensible? Does it correctly return "nothing" if nothing's there?
2. **Plant-and-find test.** Run a deliberate session where the engineer corrects Claude on a fake project rule (e.g., *"we don't use `let` in this project, only `const`"*). Run `/debrief`. The rule should appear as a proposal with the correction quoted as evidence.
3. **Smooth-session test (the trustworthiness gate).** Run a clean session with no corrections. `/debrief` should return the "session looks clean" message — not invent something. **This is the most important test.** If this fails, the skill is broken regardless of the other tests.
4. **Real Orbitant projects.** Once smoke tests pass, run `/debrief` after real client-project sessions. Over a few weeks, track: are proposals being applied? Are engineers ignoring the output? Are the same kinds of proposals showing up across projects (signal for promotion to a future Orbitant-wide standard)?

## Open questions / follow-ups

- **Promotion to Orbitant-wide standard.** Once `/debrief` is in real use, the same proposed edits will appear across multiple client repos. The future `orbitant-engineering` plugin's `references/` is the destination for promoted standards. The promotion mechanism (PR on `orbitant-os`) is out of scope for this skill but is the natural next step.
- **SessionEnd hook reminder.** Optional add-on once the skill earns its place. Defer until usage data exists.
- **Subdirectory CLAUDE.md targeting.** If real usage shows friction that's clearly scoped to a subtree (`api/`, `mobile/`), v2 could propose subdirectory CLAUDE.md edits. Skip for now.
- **Other artifact types.** If `/debrief` users repeatedly say *"this should be a hook / a skill / a slash command"*, that's signal to expand v2. Don't pre-build.
