# Orbitant POC Skills

A workbench where the Orbitant team prototypes and shares **Claude Code skills**.
Think of it as a sandbox: people drop their ideas in, we try them out together, and
the ones that prove useful graduate into something we all use.

Each skill lives under `skills/<skill-name>/` as a self-contained `SKILL.md`.

## Installation

The easiest way to install these skills is via [skills.sh](https://skills.sh):

```bash
npx skillsadd weorbitant/orbitant-poc-skills
```

That command fetches this repo and wires the skills into your agent (Claude Code,
Cursor, Copilot, etc.). Once installed, the slash commands below become available.

## Skills

### Learn — personal learning coach

A suite of six interconnected skills that turn Claude into a learning coach. They
share a simple markdown-based state store at `~/.claude/learning/` (see
[`skills/learning-data-spec.md`](skills/learning-data-spec.md) for the format).

| Skill | What it does |
|---|---|
| `/init-learning` | Set up or update your learner profile (role, gaps, goals, style). |
| `/new-learning` | Add a learning idea to your backlog or get suggestions for what to learn next. |
| `/learn` | Start or continue an active lesson — milestone by milestone. |
| `/learning-status` | Quick check of where you are: current lesson, backlog, streak. |
| `/learning-report` | Generate a progress report over a period of time. |
| `/end-learn` | Close out a session or lesson, log what you learned, archive it. |

Typical flow: `/init-learning` once → `/new-learning` to queue topics →
`/learn` to work on them → `/end-learn` to wrap up → `/learning-report` when you
want to see how far you've come.

<!-- Add new skills below as new sections. Keep each one self-contained so it
     can be understood without reading the others. -->

## Contributing a skill

1. Create a folder under `skills/<your-skill-name>/` with a `SKILL.md`.
2. Follow the frontmatter convention used by existing skills (`name`,
   `description`, `license`, `version`, `metadata.author`, `metadata.tags`).
3. Add a section to this README describing what the skill does and when to use it.
4. Open a PR.
