# Proposal Format Examples

Worked examples for the proposal formats used in `/new-learning` step 4.
The formal field structure lives inline in `SKILL.md`; this file holds the
concrete shapes the agent should mirror when drafting proposals.

## Standard proposal (no `--repo`)

```
### 1. Build a REST API with Express + PostgreSQL
**Gap:** backend  |  **Effort:** 3 sessions  |  **Style:** hands-on

Build a task management API from scratch — routes, validation, database, and tests.

**Milestones:**
1. Scaffold Express app, define routes, return mock data
2. Connect PostgreSQL, implement CRUD operations
3. Add input validation, error handling, and integration tests
```

## Repo-based proposal (when `--repo` was provided)

Adds a `**Repo context:**` line under the metadata so the user can see what the
proposal builds on. Milestones extend the existing code rather than starting from
scratch.

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
