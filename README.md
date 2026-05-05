# Claude Skills

A collection of installable skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Each skill is a `.skill` file that extends Claude Code with new, task-specific capabilities — no MCP connectors or custom integrations required.

## Available Skills

| Skill | File | Description |
|-------|------|-------------|
| Outcome Rater | `outcome-rater.skill` | Fetch Jira tickets and rate them against four quality gates, producing actionable keep/close/clarify recommendations. |

See each skill's directory for detailed documentation.

---

## Getting Started

### 1. Download a skill

Download the `.skill` file directly from this repository.

### 2. Install it into Claude Code

```bash
claude skill install <skill-name>.skill
```

The skill is now available in every Claude Code session.

### 3. Use it

Navigate to your project and start a Claude Code session, then trigger the skill naturally using one of its supported phrases.

---

## Repository Structure

Each skill lives in its own directory alongside a pre-built `.skill` file (zip archive) for easy installation:

```
claude-skills/
├── outcome-rater/
│   ├── SKILL.md          # skill source — edit this
│   └── README.md         # skill-specific documentation
├── outcome-rater.skill    # installable zip — regenerate after edits
└── README.md
```

## Contributing

Skills are plain `.skill` files (zip archives containing a `SKILL.md`). To add a new skill:

1. Create a directory named after your skill, e.g. `my-skill/`.
2. Write `my-skill/SKILL.md` following the same front-matter + instructions format as existing skills.
3. Add a `my-skill/README.md` documenting how the skill works.
4. Build the installable: `zip my-skill.skill my-skill/SKILL.md`
5. Open a PR adding both the source directory and the `.skill` file, plus a row in the table above.

To update an existing skill, edit its `SKILL.md` then rebuild:

```bash
zip outcome-rater.skill outcome-rater/SKILL.md
```
