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
