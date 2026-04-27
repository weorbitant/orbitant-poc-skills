# Orbitant POC Skills

A workbench where the Orbitant team prototypes and shares skills for AI coding
agents. People drop ideas in, we try them out together, and the ones that prove
useful graduate into something we all use.

Skills follow the [Agent Skills](https://skills.sh) format and work with Claude
Code, Cursor, Copilot, Codex, and [others](https://skills.sh).

## Installation

```bash
npx skills add weorbitant/orbitant-poc-skills
```

### Other install options

```bash
# Install only one skill from this repo
npx skills add weorbitant/orbitant-poc-skills --skill learn

# Install globally (available across all your projects)
npx skills add weorbitant/orbitant-poc-skills -g

# Target a specific agent
npx skills add weorbitant/orbitant-poc-skills -a claude-code

# List available skills without installing
npx skills add weorbitant/orbitant-poc-skills --list
```

See the [`skills` CLI docs](https://github.com/vercel-labs/skills) for the full
list of flags.

## Available Skills

### learn

A personal learning coach. A suite of six interconnected skills that help you
figure out what to learn, sit down and learn it, and look back at what you've
learned. All six share a simple markdown state store at `~/.claude/learning/`
(see [`skills/learning-data-spec.md`](skills/learning-data-spec.md)).

**Use when:**
- "Set up my learning profile" / "I want to start learning"
- "What should I learn next?" / "Add X to my backlog"
- "Let's learn" / "Continue my lesson" / "Pick up where I left off"
- "Where am I with my learning?" / "Learning status"
- "What have I learned this month?" / "Generate a learning report"
- "I'm done for today" / "Wrap up this lesson"

**Commands:**

| Command | Purpose |
|---|---|
| `/init-learning` | Create or update your profile — role, gaps, goals, preferred learning style. |
| `/new-learning` | Add a learning idea to your backlog or get suggestions for what to learn. |
| `/learn` | Start or continue an active lesson, milestone by milestone. |
| `/learning-status` | Fast check of your current lesson, backlog, and streak. |
| `/learning-report` | Generate a progress summary over a period of time. |
| `/end-learn` | Close out a session or lesson, log what you learned, archive it. |

**Typical flow:** `/init-learning` once → `/new-learning` to queue topics →
`/learn` to work through them → `/end-learn` to wrap up → `/learning-report`
to reflect.

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

**v1 scope:** `CLAUDE.md` edits only. No skills, subagents, hooks, slash
commands, or settings proposals. No subdirectory `CLAUDE.md` targeting. No
SessionEnd hook reminder. Stateless across runs.

<!-- Add new skills below as a new H3 section. Keep each one self-contained so
     it can be understood without reading the others. -->

## Contributing a skill

1. Create a folder under `skills/<your-skill-name>/` with a `SKILL.md`.
2. Follow the frontmatter convention used by existing skills
   (`name`, `description`, `license`, `version`, `metadata.author`,
   `metadata.tags`).
3. Add a section to this README with a short description, a **Use when:**
   bullet list of trigger phrases, and any commands the skill exposes.
4. Open a PR.

Browse [skills.sh](https://skills.sh) for inspiration — what makes a skill
discoverable is a clear description and a concrete "use when" list.
