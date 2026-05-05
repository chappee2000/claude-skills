---
name: outcome-rater
description: Rate and triage Jira tickets (Outcomes) against four closing criteria and provide actionable recommendations. Use this skill whenever the user shares one or more Jira tickets, Outcomes, or work items and asks for ratings, triage, prioritization review, or recommendations — whether pasted directly, uploaded as PDFs, or fetched from Jira via PAT. Trigger on phrases like "rate these tickets", "should we close this", "review these outcomes", "triage these Jira tickets", "prioritize these", "pull tickets from Jira and rate them", or any time tickets/outcomes need evaluation. In Claude Code, this skill can fetch tickets directly from Jira using a Personal Access Token — no MCP connector required.
---

# Outcome Rater

Evaluate one or more Outcome tickets against four quality gates and produce a structured recommendation for each.

Tickets can be provided three ways:
1. **Pasted text** — copy/paste ticket content directly into the conversation
2. **PDF upload** — exported Jira PDFs uploaded to the conversation
3. **Jira PAT fetch** — Claude Code fetches tickets directly via the Jira REST API (see below)

---

## Fetching Tickets from Jira (Claude Code / PAT)

Use this path when running in Claude Code and the user wants to pull tickets directly rather than paste or upload them.

### Step 0 — Gather inputs

**Defaults (do not ask for these unless overridden):**
- Project: `HCMSTRAT`
- Issue type: `Outcome`

Only ask for what isn't already provided:

| Input | Default | Example |
|---|---|---|
| Jira base URL | _(ask)_ | `https://redhat.atlassian.net` |
| PAT | _(ask)_ | Personal Access Token — store in shell var, never log |
| How to select tickets | All open Outcomes in HCMSTRAT | Label, JQL, specific IDs, or parent Epic |

If the user just says "rate the outcomes" or "pull the tickets" with no further filter, default to:
```
project = HCMSTRAT AND issuetype = Outcome AND resolution = Unresolved ORDER BY updated DESC
```

### Step 1 — Fetch ticket list

Store credentials in shell variables first, then query:

```bash
JIRA_BASE="https://redhat.atlassian.net"
JIRA_PAT="<user-provided-token>"

# Default: all open Outcomes in HCMSTRAT
curl -s -G \
  -H "Authorization: Bearer $JIRA_PAT" \
  -H "Content-Type: application/json" \
  --data-urlencode 'jql=project = HCMSTRAT AND issuetype = Outcome AND resolution = Unresolved ORDER BY updated DESC' \
  --data-urlencode 'fields=summary,description,status,priority,labels,parent' \
  --data-urlencode 'maxResults=50' \
  "$JIRA_BASE/rest/api/3/search"
```

Common JQL patterns:
```
# All open Outcomes in HCMSTRAT (default)
project = HCMSTRAT AND issuetype = Outcome AND resolution = Unresolved

# By label
project = HCMSTRAT AND issuetype = Outcome AND labels = "HPSTRAT-audit"

# By specific IDs
issueKey in (HCMSTRAT-44, HCMSTRAT-45, HCMSTRAT-46)

# By parent Epic
project = HCMSTRAT AND issuetype = Outcome AND "Epic Link" = HATSTRAT-254
```

### Step 2 — Extract fields

From each issue in the response, extract:
- `key` — ticket ID
- `fields.summary` — title
- `fields.description` — ADF content; extract plain text from `content[].content[].text` nodes
- `fields.status.name` — current status
- `fields.priority.name` — priority

Then proceed to rating each ticket using the four gates below.

### Step 3 — Output results to a file

When running in Claude Code, write results to a markdown file the user can open:

```bash
cat > outcome-ratings-$(date +%Y%m%d).md << 'EOF'
# Outcome Ratings — <date>

<ratings content>
EOF
```

### Error handling
- **401/403**: PAT expired or missing permissions — report and stop
- **400**: Log the JQL error message and suggest a corrected query
- **Empty results**: Report the JQL used and ask user to verify label/project key

---

## What Is an Outcome?

> A change in human behavior that drives business results — ideally, both the human behavior and the business result will be observable and/or measurable.

An Outcome is **not**:
- A project, epic, or bucket of work
- A task list or initiative container
- Something purely aspirational with no observable end state
- Work that falls outside CY26/27 planning horizon

---

## The Four Closing Gates

Apply each gate in order. A single **No** (or **Yes** for gate 3) means the Outcome should be **marked for closing**. Gates are not weighted — one failure is sufficient.

| # | Gate | Close if... |
|---|------|-------------|
| 1 | **Press-release worthy?** | The work couldn't be described compellingly in an internal press release — even if purely internal, it should feel like a meaningful win worth announcing. |
| 2 | **On the leadership priority list?** | If leadership needed a single ranked view of HP (high-priority) outcomes, this one wouldn't make the cut. |
| 3 | **Is this a bucket?** | The ticket is a container or umbrella for other work rather than a discrete, outcome-oriented item. |
| 4 | **Must be addressed in CY26/27?** | The work doesn't need to happen within the current two-year window. |

---

## Output Format

For each ticket provided, produce this structure:

```
## [Ticket ID / Title]

**Gate 1 – Press-release worthy:** [Yes / No / Unclear]
> [1–2 sentence rationale]

**Gate 2 – Leadership priority:** [Yes / No / Unclear]
> [1–2 sentence rationale]

**Gate 3 – Bucket check:** [Not a bucket / IS a bucket]
> [1–2 sentence rationale]

**Gate 4 – CY26/27 relevance:** [Yes / No / Unclear]
> [1–2 sentence rationale]

**Recommendation:** [KEEP OPEN / MARK FOR CLOSING / NEEDS CLARIFICATION]
**Reason:** [Which gate(s) failed, or what info is missing]
**Suggested next step:** [e.g., "Close ticket", "Rewrite to focus on behavior change X", "Clarify scope with PM", "Split into discrete outcomes"]
```

If multiple tickets are provided, process each one in order, then add a **Summary Table** at the end:

| Ticket | Gate 1 | Gate 2 | Gate 3 | Gate 4 | Recommendation |
|--------|--------|--------|--------|--------|----------------|
| ...    | ✅/❌/❓ | ✅/❌/❓ | ✅/❌/❓ | ✅/❌/❓ | KEEP / CLOSE / CLARIFY |

---

## Rating Guidance

### Gate 1 — Press-release worthy
Ask: *Could someone write a 2-sentence internal announcement about this that would make people go "that's great"?*

- **Yes signals**: Clear user-facing or operational impact, measurable improvement, addresses a known pain point
- **No signals**: Vague, housekeeping work, purely technical debt with no visible outcome, catch-all
- **Unclear signals**: Missing context about what "done" looks like

### Gate 2 — Leadership priority
Ask: *If the CEO asked for a top-10 list of the most important outcomes this year, would this be on it?*

- **Yes signals**: Directly tied to revenue, clinical outcomes, retention, compliance, or core product differentiation
- **No signals**: Nice-to-have, internally motivated, long-tail edge case
- **Unclear signals**: Could be high priority depending on context not in the ticket

### Gate 3 — Bucket check
Ask: *Is this a container for other work, or is it a thing that can actually be "done"?*

- **Bucket signals**: Title includes "Initiative", "Program", "Workstream", "Various", "Misc"; acceptance criteria reference other tickets; no single clear end state
- **Not a bucket**: Has a specific, observable change described; can be closed without sub-tickets

### Gate 4 — CY26/27 relevance
Ask: *Is there a reason this needs to happen in the next 1–2 years specifically?*

- **Yes signals**: Tied to roadmap, compliance deadline, customer commitment, or competitive pressure
- **No signals**: "Someday/maybe", exploratory, no urgency signal
- **Unclear signals**: No timeline info in ticket

---

## Handling Ambiguous Tickets

If a ticket is missing enough information to evaluate a gate confidently:
- Mark that gate as **Unclear (❓)**
- Note what specific information is needed
- Set recommendation to **NEEDS CLARIFICATION** if 2+ gates are unclear
- Still evaluate the gates you *can* assess

---

## Tone

Be direct and useful. This is a triage tool, not a performance review. Recommendations should be actionable. When suggesting a ticket be closed, briefly note if the underlying need could be better captured in a rewritten, more focused outcome.
