# Backlog Purge (Reusable)

Analyze ANY team's Jira backlog using **historical velocity data** to identify stale, blocked, and statistically unlikely-to-complete items for closure.

**Goal:** Reduce backlog to ≤2 quarters of work based on actual team velocity.

## What This Skill Does

1. **Analyzes team history** (last 6 months) to calculate:
   - Actual velocity (items/month)
   - Median and mean cycle time
   - Cycle time by issue type and priority
   - Completion probability patterns

2. **Scores every backlog item** based on data-driven criteria:
   - Age vs. historical completion patterns
   - Priority paradoxes (high priority items that never complete)
   - Active work protection (In Progress/QA/Review NEVER purged)
   - Type-specific age thresholds
   - Blocked duration analysis

3. **Creates deliverables:**
   - Scored CSV with all items (with Jira hyperlinks)
   - Summary report with velocity insights
   - Jira filters for "Close Now" and "Discuss" categories
   - Batch recommendations for purge execution

## Usage

When invoked, ask the user for:

1. **Team JQL query** - The query that defines the team's backlog
   - Example: `project = MYPROJ AND "Team Name" = "Platform Team" ORDER BY Rank ASC`
   - Or: `project in (PROJ1, PROJ2) AND labels = team-alpha`
   
2. **Jira base URL** (default: `https://redhat.atlassian.net`)
3. **Email** for authentication
4. **Personal Access Token (PAT)**
5. **Template ticket keys to exclude** (optional) - Tickets used for cloning that should never be closed

The skill will:
- Fetch team's completed work (last 6 months)
- Calculate actual velocity and cycle times
- Fetch current backlog
- Score all items
- Generate outputs and Jira filters

## Execution Workflow

### Step 1: Gather Inputs

Ask the user for:
- **Team JQL query** (required)
- **Jira base URL** (default: https://redhat.atlassian.net)
- **Email** (required for auth)
- **PAT** (required for auth)
- **Template tickets to exclude** (optional, comma-separated keys like "PROJ-123,PROJ-456")
- **Team size** (optional, for capacity calculations - if not provided, estimate from velocity)

### Step 2: Analyze Historical Velocity (REQUIRED FIRST)

Fetch completed items from last 6 months:
```bash
curl -s -u "${JIRA_EMAIL}:${JIRA_PAT}" \
  -X POST \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "jql": "<TEAM_JQL> AND status in (Done, Closed, Resolved) AND resolutiondate >= -180d ORDER BY resolutiondate DESC",
    "fields": ["created", "resolutiondate", "status", "issuetype", "priority"],
    "maxResults": 500
  }' \
  "${JIRA_BASE}/rest/api/3/search/jql"
```

Calculate:
- **Velocity:** Items completed per month
- **Cycle times:** Per item, by type, by priority
- **Median cycle time:** Realistic planning number
- **Distribution:** % of items in 0-30d, 31-90d, 91-180d, >180d buckets
- **Priority paradox:** Compare Critical/Major vs Normal avg cycle times

Output velocity summary before proceeding.

### Step 3: Calculate Target Backlog Size

Based on actual velocity:
- **2 quarters capacity** = velocity × 6 months
- **Current backlog** = total unresolved items
- **Target purge** = current backlog - (velocity × 4 to 6 months)

Example:
- Velocity = 50 items/month
- 2 quarters = 50 × 6 = 300 items max
- Current = 300 items
- Keep = 200-250 items (4-5 month buffer)
- Purge = 50-100 items

### Step 4: Fetch Current Backlog (All Pages)

Fetch ALL pages of unresolved items:
```bash
# Fetch page 1
curl ... -d '{
  "jql": "<TEAM_JQL> AND resolution = Unresolved ORDER BY Rank ASC",
  "fields": ["summary", "status", "created", "updated", "assignee", "priority", "issuetype", "duedate", "labels"],
  "maxResults": 100
}' ...

# If isLast = false, fetch next page using nextPageToken
# Repeat until all pages fetched
```

Combine all pages into single dataset.

### Step 5: Score Every Item for Closure

For each item, calculate closure score (0-100) based on velocity-adjusted thresholds.

---

## Closure Criteria & Scoring

Each item receives a **Closure Score (0-100)**. Higher = stronger candidate for closure.

## Closure Scoring Logic (Velocity-Adjusted)

### CRITICAL PROTECTION: Active Work (Score: 0-10)

**NEVER purge items in active statuses:**
- Status = "In Progress" → Score: 10 (Keep)
- Status = "Code Review" → Score: 10 (Keep)
- Status = "ON_QA" or "QA" → Score: 10 (Keep)
- Status = "ON_DEV" → Score: 10 (Keep)

**Exception:** Only flag for review (Score: 70) if:
- Age >365 days AND last update >180 days (likely abandoned/blocked)
- Category: "Review" not "Close Now" (requires manual inspection)

### CRITICAL - Immediate Action Required

**Score: 95-100**
- ❗ **Past Due Date** (due date < today, still not done)
  - These items missed their deadline
  - Top priority for closure unless there's a compelling reason
  - **Data insight:** Items that miss deadlines have near-zero completion probability

**Score: 90-94**
- 🔒 **Blocked >180 days** with no active resolution
  - Check `flagged` or `impediment` custom field
  - **Velocity insight:** Items >180 days old represent only ~11% of completed work
- 🚨 **Priority Paradox: Critical/Major items >365 days old, never started**
  - Score: 92
  - **Velocity insight:** High-priority items often take 4-5x longer than Normal priority
  - These are either permanently blocked or mis-prioritized → Close or re-scope
- 🚫 **CVE/Security items flagged for closure**
  - Category: "Discuss" not "Close Now"
  - NEVER auto-close CVEs - require team security review

### HIGH Priority for Closure

**Score: 70-89**
1. **Stale & Abandoned** (age + inactivity)
   - Created >180 days (6 months) AND no updates in last 90 days → Score: 85
     - **Data insight:** Only 11% of completed items had >180 day cycle time
   - Created >270 days (9 months) AND no updates in last 120 days → Score: 88
   - Created >365 days (1 year) AND no updates in last 180 days → Score: 90
   - Never assigned AND created >90 days → Score: 78
     - **Data insight:** Unassigned old items rarely get picked up

2. **Low Engagement**
   - Zero comments AND zero watchers AND >90 days old → Score: 80
   - No links to epics/initiatives/parent issues AND >180 days old → Score: 75
   - Status = "Backlog" or "New" for >180 days → Score: 82
     - **Data insight:** Items stuck in Backlog/New for >6 months rarely move to active

3. **Epic-Specific Criteria**
   - Epic >365 days old with ZERO child issues → Score: 88
     - **Data insight:** Epics avg 223 days to complete; 1+ year with no children = abandoned
   - Epic >365 days old with ALL children on closure list → Score: 85
   - Epic linked to CRCPlan but >365 days old, no children → Score: 87 + Flag "BU-DRIVEN - Why not started?"

4. **Vague/Incomplete**
   - Missing description or <50 characters → Score: 74
   - No acceptance criteria mentioned (check description) → Score: 71
   - Labels contain "needs-refinement" AND >120 days old → Score: 76

### MEDIUM Priority for Closure

**Score: 50-69**
1. **Low Strategic Value**
   - Priority = "Lowest" or "Low" AND >120 days old → Score: 65
   - No customer mention in description/comments AND >90 days old → Score: 60
   - Labels contain "nice-to-have" or "tech-debt" AND >90 days old → Score: 62

2. **Type-Specific Age Thresholds** (based on historical cycle time)
   - Epic >365 days with children but no activity in 180 days → Score: 67
     - **Data insight:** Epics avg 223 days; 1+ year suggests stalled
   - Story >180 days, not started → Score: 64
     - **Data insight:** Stories avg 56 days; 3x median suggests unlikely to start
   - Bug >180 days, not started → Score: 66
     - **Data insight:** Bugs avg 52 days; old bugs rarely get fixed
   - Task >90 days, not started → Score: 58
     - **Data insight:** Tasks avg 15 days; 6x median is extreme

3. **Potential Duplicates** (analyze summary/description similarity)
   - If summary similarity >80% with another ticket → Score: 68
   - If description mentions "duplicate of" or "similar to" → Score: 70
   - Flag for manual review: "POTENTIAL DUPLICATE: [ticket-key]"
   - **Team hygiene note:** Poor Jira hygiene means duplicates are common

### SPECIAL HANDLING - Not Scored for Closure

**Epics** (issuetype = Epic)
- ✅ **Do NOT add to closure list**
- ✅ **Evaluate hierarchy instead:**
  - Check if Epic has child issues
  - If Epic has 0 child issues AND >6 months old → Flag: "EMPTY EPIC - Review"
  - If Epic linked to **CRCPlan** ticket:
    - Extract CRCPlan link
    - Flag: "BU-DRIVEN - Why not completed?" 
    - Output separately: **Business Unit Features Requiring Review**

**Hierarchical Analysis:**
- If parent is on closure list, evaluate children
- If all children of an Epic are on closure list → Flag Epic for review
- If item is linked to CRCPlan AND on closure list → "BU-DRIVEN - Escalate"

### KEEP - Exempt from Closure

**Score: 0-49 or excluded**
- Updated in last 60 days → Score: 20
- Assigned AND status = "In Progress" → Score: 10
- Linked to active Epic (Epic updated in last 90 days) → Score: 15
- Labels contain "customer-commitment" or "sla" → Score: 0 (exclude)
- Security issue (issuetype = "Security" OR summary contains "CVE") → Score: 0 (exclude, flag for discussion)
- Compliance-related (labels contain "compliance", "sox", "gdpr") → Score: 0 (exclude)
- Due date in future (within next 60 days) → Score: 25

---

## Fields to Fetch from Jira

```json
{
  "fields": [
    "summary",
    "description",
    "status",
    "created",
    "updated",
    "assignee",
    "reporter",
    "issuetype",
    "issuelinks",
    "priority",
    "duedate",
    "labels",
    "resolution",
    "comment",
    "flagged",
    "customfield_*" 
  ]
}
```

**Custom field mapping to check:**
- `flagged` or `impediment` — blocked status
- Search for "Team" custom field if available
- Search for "CRCPlan" in issue links

---

## Workflow

### Step 0: Analyze Historical Velocity (REQUIRED FIRST)

Before analyzing the backlog, fetch completed items to understand actual team performance:

```bash
# Fetch completed items from last 6 months
curl -s -u "${JIRA_EMAIL}:${JIRA_PAT}" \
  -X POST \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "jql": "project in (RHCLOUD, IQE) AND \"Team[Team]\" = cc1c0d99-0567-45c8-bf77-8e6149d7ed83 AND status in (Done, Closed, Resolved) AND resolutiondate >= -180d ORDER BY resolutiondate DESC",
    "fields": ["created", "resolutiondate", "status", "issuetype", "priority", "labels"],
    "maxResults": 500
  }' \
  "${JIRA_BASE}/rest/api/3/search/jql"
```

Calculate:
- Items completed per month (velocity)
- Cycle time per item: `(resolutiondate - created) / 86400` days
- Median, mean, min, max cycle times
- Cycle time by issue type
- Cycle time by priority
- Distribution: 0-30d, 31-90d, 91-180d, >180d

**Key metrics to extract:**
- Actual velocity (items/month)
- Median cycle time (realistic planning number)
- % of items >180 days (outlier rate)
- Priority paradox: Compare Critical/Major vs Normal avg cycle times

Output velocity summary before proceeding to backlog analysis.

### Step 1: Fetch Current Backlog
```bash
JIRA_BASE="https://redhat.atlassian.net"
JIRA_EMAIL="<user-email>"
JIRA_PAT="<user-token>"

curl -s -u "${JIRA_EMAIL}:${JIRA_PAT}" \
  -X POST \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "jql": "project in (RHCLOUD, IQE) AND \"Team[Team]\" = cc1c0d99-0567-45c8-bf77-8e6149d7ed83 ORDER BY Rank ASC",
    "fields": ["summary", "description", "status", "created", "updated", "assignee", "reporter", "issuetype", "issuelinks", "priority", "duedate", "labels", "resolution", "comment", "flagged"],
    "maxResults": 500
  }' \
  "${JIRA_BASE}/rest/api/3/search/jql"
```

### Step 2: Calculate Closure Score for Each Item

For each ticket:
1. **Check if Epic** → Skip closure scoring, perform hierarchical analysis
2. **Check past due date** → Score: 95-100
3. **Check blocked duration** → Add to score
4. **Check CVE** → Flag for discussion if high score
5. **Evaluate age, activity, engagement** → Base score
6. **Check for duplicates** → Similarity analysis on summary/description
7. **Check CRCPlan links** → Flag business unit features

### Step 3: Categorize Items

**Category A: Close Now** (Score 70-100, excluding CVEs)
- Items safe to close immediately
- Provide closure rationale for each

**Category B: Discuss Before Closing** (Score 70-100, CVEs or BU-driven)
- CVEs on closure list
- Items linked to CRCPlan
- Flagged items with high scores

**Category C: Review for Closure** (Score 50-69)
- Borderline items needing team judgment
- Potential duplicates

**Category D: Keep** (Score 0-49)
- Active or strategically important work

**Category E: Empty Epics** (Special)
- Epics with no children or all children on closure list

**Category F: Business Unit Features** (Special)
- Items linked to CRCPlan on closure list → Why not completed?

### Step 4: Duplicate Detection

Use fuzzy string matching on summaries:
```python
# Pseudo-code for duplicate detection
for ticket_a in backlog:
    for ticket_b in backlog:
        if ticket_a.key != ticket_b.key:
            similarity = calculate_similarity(ticket_a.summary, ticket_b.summary)
            if similarity > 0.80:
                flag_potential_duplicate(ticket_a, ticket_b, similarity)
```

### Step 5: Output Results

Generate CSV files:

**1. closure_recommendations.csv**
```csv
Category,Ticket,Summary,Score,Age (days),Last Updated (days ago),Status,Assignee,Closure Rationale,Special Flags
Close Now,RHCLOUD-123,Fix login bug,85,245,120,Backlog,Unassigned,"Stale (245d old, no updates in 120d), no engagement",
Discuss,IQE-456,CVE-2025-1234 security fix,88,180,90,New,jdoe,"CVE - Requires discussion. Stale and unassigned.",CVE
Review,RHCLOUD-789,Improve dashboard performance,65,156,45,Backlog,jsmith,"Low priority, no customer ask, old",
Keep,IQE-234,Implement OAuth2,25,45,10,In Progress,ajones,"Active work, recently updated",
```

**2. business_unit_features_review.csv**
```csv
Ticket,Summary,Score,CRCPlan Link,Closure Rationale,Why Not Completed?
RHCLOUD-999,New billing integration,78,CRCPLAN-45,"Stale (300d), no updates in 180d","BU-driven feature not delivered - Escalate to PM"
```

**3. empty_epics_review.csv**
```csv
Epic,Summary,Child Count,Last Updated,Recommendation
RHCLOUD-100,Q3 Performance Improvements,0,365 days ago,"Empty epic - Close or link children"
RHCLOUD-200,Security Enhancements,3 (all on closure list),180 days ago,"All children recommended for closure - Review epic"
```

**4. potential_duplicates.csv**
```csv
Ticket A,Ticket B,Similarity %,Summary A,Summary B
RHCLOUD-111,RHCLOUD-222,92%,Fix auth timeout issue,Fix authentication timeout bug
IQE-333,IQE-444,85%,Add retry logic for API,Implement retry mechanism for API calls
```

### Step 6: Generate Outputs

Create the following files in the user's working directory:

**1. `backlog_purge_<YYYYMMDD>.csv`**
- All backlog items with scores, categories, and hyperlinks
- Columns: Category, Ticket (hyperlink), Summary, Type, Status, Priority, Score, Age, Last Updated, Assignee, Closure Rationale, Special Flags

**2. `backlog_purge_summary_<YYYYMMDD>.md`**
- Full analysis report with:
  - Team velocity metrics
  - Age distribution
  - Category breakdown
  - Top closure candidates
  - Next steps recommendations

**3. `cycle_time_insights_<YYYYMMDD>.md`**
- Historical velocity analysis
- Cycle time by type and priority
- Priority paradox findings

### Step 7: Create Jira Filters

Create two saved filters in Jira (owned by user, shared globally):

**Filter 1: "Backlog Purge - Close Now (N items)"**
```bash
curl -u "${JIRA_EMAIL}:${JIRA_PAT}" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Backlog Purge - Close Now (N items)",
    "description": "Items recommended for closure. Sorted oldest first. Generated YYYY-MM-DD.",
    "jql": "issueKey in (KEY-1,KEY-2,...) ORDER BY created ASC",
    "favourite": true,
    "sharePermissions": [{"type": "global"}]
  }' \
  "${JIRA_BASE}/rest/api/3/filter"
```

**Filter 2: "Backlog Purge - CVEs/Discuss (N items)"**
- Same structure, but for CVE/security items requiring discussion

Return filter URLs to user.

### Step 8: Summary Statistics

Output final summary:
```
# Backlog Purge Analysis - [Date]

## Current State
- Total items: [actual count]
- Team size: 7 people
- Historical velocity: 50 items/month (11.5 items/week)
- Median cycle time: 20 days
- Mean cycle time: 69 days

## Capacity Analysis
- 2-quarter capacity at current velocity: ~300 items
- Current backlog: [actual] items
- **Conclusion:** Backlog is [UNDER/AT/OVER] capacity
- **Strategy:** Purge for QUALITY (remove clutter) not QUANTITY

## Velocity Insights
- Normal priority items: 44 days avg (fast)
- Critical/Major items: 210-220 days avg (4-5x slower - PRIORITY PARADOX)
- Items >180 days old: Only 11% of completions
- Epic cycle time: 223 days avg

## Recommendations
- **Close Now:** [X] items ([%])
- **Discuss Before Closing:** [X] items ([%]) — [N] CVEs, [N] BU-driven, [N] Critical/Major paradox
- **Review for Closure:** [X] items ([%])
- **Keep:** [X] items ([%])

## Target Backlog Size
- **Recommended:** 170-200 items (4-6 month healthy pipeline)
- **Items to purge:** [X] items ([%]) to reach target

## Special Reviews Needed
- **Empty Epics:** [N] epics
- **Business Unit Features Not Completed:** [N] items (CRCPlan-linked)
- **Potential Duplicates:** [N] pairs
- **Priority Paradox Items:** [N] Critical/Major >1 year old

## Next Steps
1. Review "Close Now" list with team
2. Schedule discussion for CVEs and BU-driven items
3. Review Critical/Major items >1 year old (priority paradox)
4. Merge/close duplicate tickets
5. Review empty epics for closure or re-scoping
```

---

## Error Handling

- **401/403:** PAT expired or missing permissions → Report and stop
- **400:** Log JQL error and suggest correction
- **Empty results:** Report JQL used and ask user to verify
- **Rate limiting:** Implement exponential backoff for large backlogs

---

## Tone

Be direct and data-driven. This is a triage tool for backlog health, not a performance review. Frame recommendations as opportunities to focus on high-value work.

When suggesting closures, be specific about the rationale. When flagging CVEs or BU-driven work, emphasize the need for discussion rather than immediate closure.
