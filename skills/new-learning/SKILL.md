---
name: new-learning
description: |
  Use when the user wants to find what to learn next, explore learning topics, get
  project ideas for a skill gap, or add a learning idea to their backlog. Supports
  repo-based ideas with --repo flag to generate milestones based on an existing project.
  Triggers on: "what should I learn", "suggest learning projects", "I want to learn X",
  "add to my learning backlog", "find me resources on X", or "I want to learn X in this repo".
license: MIT
version: "0.1.0"
metadata:
  author: david
  tags: learning, backlog, ideas, projects, resources, gaps, suggestions
---

## Overview

Generate learning ideas based on the learner's profile and add selected ones to
`~/.claude/learning/backlog.md`. Ideas can be hands-on projects (structured by
milestones) or reading/article recommendations.

## When to Use

- User wants to discover what to learn next
- User has a specific topic in mind and wants it structured
- Backlog is empty and user wants to start learning
- After completing a lesson, to find the next one

## Prerequisites

Profile must exist at `~/.claude/learning/profile.md`. If not, tell the user:
"No learning profile found. Run `/init-learning` first to set up your profile."

## Process

1. **Read context:**
   - `~/.claude/learning/profile.md` — gaps, goals, session_length
   - `~/.claude/learning/lessons/archive/` — scan "Still Fuzzy On" sections and abandoned lesson topics
   - `~/.claude/learning/backlog.md` — avoid duplicating existing items

2. **Determine source of ideas:**
   - If the user passed a topic and `--repo` flag (e.g., `/new-learning "LangGraph" --repo ./my-project`):
     focus on that topic AND analyze the repo
   - If the user passed a topic only (e.g., `/new-learning "WebSockets"`): focus on that topic
   - If no argument: use gaps from profile + "still fuzzy" themes from archives

   **When a `--repo` flag is provided:**
   1. Analyze the repo at the given path — dependencies, file structure, key source files
   2. Cross-reference with the requested topic — identify what's already implemented,
      what's partially done, and what's missing
   3. Generate milestones that build on top of existing code rather than starting from scratch

   Example: `/new-learning "LangGraph" --repo ./my-agent`
   Analysis: "LangGraph is installed, `src/agent.py` has a linear StateGraph with 2 nodes.
   No conditional edges, no subgraphs, no persistence layer."

3. **Search for current resources:**
   - Use web search to find up-to-date articles, tutorials, docs, and project ideas
   - Prioritize recent, well-regarded resources

4. **Propose 3-5 ideas** using this format for each:

   ```
   ### 1. [Title]
   **Gap:** [which gap from profile]  |  **Effort:** [N] sessions  |  **Style:** hands-on / reading

   [1-2 sentence description of what you'll build or learn]

   **Milestones** (if hands-on):
   1. [First milestone — simplest, get something working]
   2. [Build on it]
   3. [Polish or advanced feature]
   ```

   Example:
   ```
   ### 1. Build a REST API with Express + PostgreSQL
   **Gap:** backend  |  **Effort:** 3 sessions  |  **Style:** hands-on

   Build a task management API from scratch — routes, validation, database, and tests.

   **Milestones:**
   1. Scaffold Express app, define routes, return mock data
   2. Connect PostgreSQL, implement CRUD operations
   3. Add input validation, error handling, and integration tests
   ```

   **Repo-based proposal format** (when `--repo` was provided):

   Add a `Repo context` line showing what the repo currently has:

   ```
   ### 1. Add conditional routing to your LangGraph agent
   **Gap:** AI/orchestration  |  **Effort:** 2 sessions  |  **Style:** hands-on
   **Repo context:** `src/agent.py` has a linear StateGraph — no branching yet

   Build conditional edges that route to different tools based on LLM output.

   **Milestones:**
   1. Add a router node that classifies intent and branches
   2. Implement tool-specific subgraphs for each branch
   3. Add fallback handling and LangFuse tracing for the routing decisions
   ```

5. **User picks** — present as a numbered list. Ask: "Which ones should I add to your backlog? (e.g., 1, 3)"

6. **Write to backlog.md:**
   - Create the file if it doesn't exist (with `## Backlog` header)
   - Append selected items as checkbox lines
   - Format: `- [ ] <title> — **gap:** <gap> — **effort:** <N sessions> — **style:** <hands-on | reading> — **repo:** <path>`
   - The `— **repo:** <path>` segment is only included for repo-based items

## Important

- Do NOT re-suggest topics from abandoned lessons (check archive for `status: abandoned`)
- Calibrate milestone size to the user's `session_length` from profile
- Mix hands-on and reading suggestions — don't propose only one type
- If the user provides a specific topic, still search the web for the best current resources
- Postponed lessons are NOT re-suggested here — they show up in `/learn` as resumable
