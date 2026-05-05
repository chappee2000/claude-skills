# Backlog Purge - Quick Start Checklist

## Prerequisites Checklist

- [ ] **Jira Personal Access Token (PAT)** - Create at Jira → Settings → Personal Access Tokens
- [ ] **Team JQL Query** - Test in Jira to confirm it returns your backlog
- [ ] **6+ months of team history** - At least 20-30 completed items for velocity calculation

---

## Step-by-Step Guide

### 1️⃣ **Invoke the Skill**

```
In Claude Code, type: /backlog-purge
```

### 2️⃣ **Provide Information When Prompted**

```
Team JQL query: project = MYPROJ AND team = "Alpha Team"
Jira base URL: https://your-instance.atlassian.net
Email: your-email@company.com
PAT: [paste your Personal Access Token]
Template tickets to exclude: PROJ-1000, PROJ-2000 (optional)
```

### 3️⃣ **Review Velocity Analysis**

Claude will output:
- Velocity: X items/month
- Median cycle time: Y days
- Backlog capacity: Z months of work
- Priority paradox findings (if any)

**Validate:** Does this match your team's intuition?

### 4️⃣ **Review Outputs**

Three files created:
- `backlog_purge_YYYYMMDD.csv` - Full scored list
- `backlog_purge_summary_YYYYMMDD.md` - Executive summary
- `cycle_time_insights_YYYYMMDD.md` - Team velocity metrics

**Check:** Are there 0 items in "In Progress" on the closure list?

### 5️⃣ **Schedule Purge Session (1-2 hours)**

**Invite:**
- Team lead
- 1-2 engineers
- Product manager (optional)

**Agenda:**
1. Review velocity metrics (5 min)
2. Evaluate CVEs/security items (15-30 min)
3. Review "Close Now" items oldest-first (45-60 min)
4. Flag "Needs Investigation" items (15 min)

### 6️⃣ **Execute Bulk Close**

Using Jira filters:
1. Open filter: "Backlog Purge - Close Now"
2. Select items to close
3. Bulk Change → Transition → Close
4. Resolution: "Won't Do" or "Obsolete"
5. Comment: "Closed during [DATE] backlog purge"

### 7️⃣ **Track "Needs Investigation" Items**

For items with due dates:
- [ ] Assign investigation tasks
- [ ] Set calendar reminders before due dates
- [ ] Review findings on due date
- [ ] Decide: Close or Keep

---

## Expected Timeline

| Phase | Time | Notes |
|-------|------|-------|
| **Setup** | 10 min | Get PAT, prepare JQL |
| **Skill execution** | 5-10 min | Claude analyzes backlog |
| **Review outputs** | 30 min | Validate results |
| **Purge session** | 1-2 hours | Team evaluation |
| **Bulk close** | 15-30 min | Execute in Jira |
| **Follow-up** | 15 min | Track investigations |

**Total first session:** ~2-3 hours

---

## Quick JQL Templates

### By Project
```jql
project = MYPROJECT AND resolution = Unresolved
```

### By Team Field (Red Hat)
```jql
project in (RHCLOUD, IQE) AND "Team[Team]" = team-id-12345
```

### By Label
```jql
project = MYPROJ AND labels = team-alpha AND resolution = Unresolved
```

### By Component
```jql
project = MYPROJ AND component = "Backend" AND resolution = Unresolved
```

### By Team Members
```jql
project = MYPROJ AND assignee in membersOf("platform-team")
```

**Tip:** Always test your JQL in Jira before running the skill!

---

## Expected Results (Based on Real Usage)

**For a typical 7-person team with 250 items:**

| Metric | Expected Range |
|--------|----------------|
| **Items recommended** | 150-200 (60-80%) |
| **CVE precision** | 100% |
| **Immediate closure rate** | 70-90% of evaluated items |
| **Needs investigation** | 10-20% of evaluated items |
| **False positives** | 0-5% |
| **Backlog reduction** | 20-30% |

**First session coverage:** ~30-40% of recommended items (time-limited)

---

## Common Pitfalls

### ❌ **JQL returns 0 items**
**Fix:** Test JQL in Jira, verify team identifier is correct

### ❌ **"Insufficient history"**
**Fix:** Team needs 6 months history. Extend to 12 months or accept lower accuracy.

### ❌ **Active work in closure list**
**Fix:** This should NEVER happen. Report to skill maintainer if it does.

### ❌ **All CVEs in "Close Now"**
**Fix:** CVEs should be in "Discuss" category. Verify skill is using latest version.

### ❌ **Team rejects 80%+ of recommendations**
**Fix:** Review status name mappings. Team may have custom workflows.

---

## Success Criteria

After your first purge session, you should see:

- ✅ **Backlog reduced by 20-30%**
- ✅ **0 false positives** (no valid work accidentally flagged)
- ✅ **Team understands velocity** (data-driven backlog targets)
- ✅ **CVEs properly handled** (in Discuss, not Close Now)
- ✅ **"Needs Investigation" items tracked** (not dismissed)

---

## Next Quarter

Set calendar reminder for 3 months:
- [ ] Re-run backlog-purge skill
- [ ] Compare velocity metrics (improving?)
- [ ] Track: Are "Needs Investigation" items resolved?
- [ ] Goal: Reduce purge size over time (better backlog hygiene)

---

## Need Help?

**Skill not working as expected?**
1. Check prerequisites (PAT permissions, JQL correctness)
2. Review compatibility notes in README.md
3. Report issues to skill maintainer

**Want to customize?**
- See README.md → Customization & Compatibility
- Edit `skill.md` for status/priority mappings
- Adjust scoring thresholds based on team culture

---

## Quick Reference Card

**Invoke:** `/backlog-purge`

**Inputs:**
1. Team JQL query
2. Jira base URL
3. Email
4. PAT
5. Template tickets (optional)

**Outputs:**
- CSV with scores
- Summary report
- Velocity insights
- Jira filters (optional)

**Session:** 1-2 hours, prioritize CVEs first

**Cadence:** Quarterly (every 3 months)

**Success:** 20-30% backlog reduction, 0 false positives
