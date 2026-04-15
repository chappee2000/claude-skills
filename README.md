# Claude Skills

A collection of installable skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Each skill is a `.skill` file that extends Claude Code with new, task-specific capabilities — no MCP connectors or custom integrations required.

## Skills

| Skill | File | Description |
|-------|------|-------------|
| Outcome Rater | `outcome-rater.skill` | Fetch Jira tickets and rate them against four quality gates, producing actionable keep/close/clarify recommendations. |

---

## Getting Started

### 1. Download a skill

Download the `.skill` file directly from this repository.

### 2. Install it into Claude Code

```bash
claude skill install outcome-rater.skill
```

The skill is now available in every Claude Code session.

### 3. Use it

Navigate to your project and start a session:

```bash
cd ~/your-project
claude
```

Then ask naturally — Claude Code knows what to do:

```
rate the outcomes
```

---

## Skill Details

### `outcome-rater.skill`

Evaluates Jira Outcome tickets against four quality gates and produces a structured recommendation for each one.

**Trigger phrases:** "rate the outcomes", "rate these tickets", "should we close this", "triage these Jira tickets", "pull tickets from Jira and rate them", etc.

#### How tickets can be provided

| Method | How |
|--------|-----|
| **Jira PAT fetch** | Claude Code fetches tickets directly from Jira via the REST API — no MCP required |
| **Pasted text** | Copy/paste ticket content into the conversation |
| **PDF upload** | Upload exported Jira PDFs |

#### What happens when you say "rate the outcomes"

1. Claude Code prompts for your **Jira base URL** and **Personal Access Token (PAT)**.
2. It fetches all open `Outcome` issues from the `HCMSTRAT` project (default JQL; you can override).
3. It rates each ticket against the four closing gates (see below).
4. It writes results to **`outcome-ratings-YYYYMMDD.md`** in your working directory.

#### The four closing gates

| # | Gate | Close if… |
|---|------|-----------|
| 1 | **Press-release worthy?** | The work couldn't be described compellingly in an internal announcement. |
| 2 | **On the leadership priority list?** | This wouldn't make a ranked top-10 list for the CEO. |
| 3 | **Is this a bucket?** | The ticket is a container for other work, not a discrete outcome. |
| 4 | **Must be addressed in CY26/27?** | There's no reason this needs to happen in the next 1–2 years. |

A single gate failure → **MARK FOR CLOSING**. Each ticket also gets a **Suggested next step** (close, rewrite, clarify, or split).

#### Output

A markdown file with per-ticket ratings plus a summary table:

```
## HCMSTRAT-44 / <title>

Gate 1 – Press-release worthy: Yes ✅
Gate 2 – Leadership priority:  No  ❌
Gate 3 – Bucket check:         Not a bucket ✅
Gate 4 – CY26/27 relevance:    Yes ✅

Recommendation: MARK FOR CLOSING
Reason: Gate 2 — would not make the leadership priority list.
Suggested next step: Close ticket; underlying need may resurface in roadmap planning.
```

---

## Contributing

Skills are plain `.skill` files (zip archives containing a `SKILL.md`). To add a new skill:

1. Create a directory named after your skill, e.g. `my-skill/`.
2. Write `my-skill/SKILL.md` following the same front-matter + instructions format as `outcome-rater`.
3. Zip it: `zip my-skill.skill my-skill/SKILL.md`
4. Open a PR adding the `.skill` file and a row in the table above.
