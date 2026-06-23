---
name: program-health
description: >
  Use this skill whenever someone wants to check, generate, or report on the health
  of a program or initiative in any Jira project at any Atlassian cloud instance.
  Triggers include: "program health", "health check", "initiative status", "sprint progress",
  "what's the status of [EPIC-KEY]", "generate a status report", "TPM dashboard",
  "what epics are at risk this quarter", "program health for [team/project/program]",
  or any mention of checking initiative or epic progress across a team or tribe.
  Always use this skill even if the user only provides a Jira link, epic key, program key,
  or team name — Claude will ask for the rest interactively.
  This skill is tribe-agnostic and project-agnostic. It works with any Jira cloud instance
  and any project key. It outputs an exec-ready Confluence page with narrative, risk flags,
  stuck ticket detection, deadlines, work completion %, and recommended actions.
---

# Program Health Dashboard Skill

Generates an executive program health summary for any team or tribe at any Jira Cloud
instance, outputting both a chat summary and a Confluence page readable by anyone including
stakeholders without Jira access.

A program is any parent issue (Epic, Theme, Initiative, MOKR Program, or equivalent) that
groups child work items across one or more teams. This skill works regardless of how the
organization structures its hierarchy.

---

## Workflow Overview

1. Collect scope from user
2. Detect Jira instance and resolve cloud ID
3. Fetch the program or epic to get linked child epics
4. Fetch each epic: status, dates, assignee, comments
5. Fetch child tasks per epic: compute work completion %
6. Deep-dive comments on epics and child tasks for hidden context
7. Pull Bitbucket data for linked repositories (PR counts, review age, commit activity)
8. Derive health signals: status labels, overdue flags, deadline risk, stuck tickets
9. Ask user for manual context not in Jira
10. Show chat summary
11. Create Confluence page

---

## Step 1: Collect Scope

If the user has not provided enough to begin, ask:

> "To generate the program health report, I need:
> 1. The program, epic, or theme key or URL (e.g. PROJ-123 or a Jira link)
> 2. The Jira instance URL (e.g. yourcompany.atlassian.net) — skip if already known
> 3. The Confluence space to publish to — or skip if you only want a chat summary"

If the user provides a squad or team name instead of a key, ask for the Jira project key
associated with that team, or run a search:
```
project = [PROJECT] AND issuetype = Epic AND status != Done
```

---

## Step 2: Detect Jira Instance and Cloud ID

If cloudId is not already known:
- Call `getAccessibleAtlassianResources` to list all accessible Atlassian sites
- Match the user's instance URL to the correct cloudId
- Store cloudId for all subsequent calls in this session

---

## Step 2b: Discover Custom Fields (optional but recommended)

Different organizations configure Jira differently. Before fetching epics, check what custom
fields are available on the project's epic issue type:

Call `getJiraIssueTypeMetaWithFields`:
- projectKey: the project key (e.g. PROJ, TEAM)
- issueType: Epic (or the equivalent issue type the org uses for initiatives)

Scan the returned fields for names matching these concepts:

| Concept | Common field names to look for |
|---|---|
| Planned start date | "Start Date", "Est Start", "Planned Start", "Target Start" |
| Planned delivery date | "Due Date", "Est Delivery", "Target Date", "End Date" |
| Actual start date | "Actual Start", "Start Actual" |
| Actual delivery date | "Actual Delivery", "Delivery Date", "Completed Date" |
| Health or status signal | "Project Status", "Health", "RAG Status", "Status", "Risk Level" |
| Priority or criticality | "Priority", "Criticality", "Severity" |
| Team or squad | "Team", "Squad", "Tribe", "Department" |
| Quarter or period | "Quarter", "Period", "Cycle", "Fiscal Quarter" |

Store the discovered field IDs and append them to all subsequent `getJiraIssue` calls.

If `getJiraIssueTypeMetaWithFields` is not available or returns an error: skip this step
and rely on `duedate`, `status`, `assignee`, `labels`, and `priority` as standard fallbacks.

---

## Step 3: Fetch the Program or Top-Level Epic

Call `getJiraIssue`:
- issueIdOrKey: the program or epic key
- fields: `["summary", "status", "assignee", "issuelinks", "duedate", "labels", "comment", "description", "priority"]`
- Append any custom field IDs discovered in Step 2b for dates or health signals

Extract:
- Program or epic name from `fields.summary`
- Quarter label: parse from summary (e.g. "26Q2", "Q3 2025") or ask the user
- Linked child epics: all `issuelinks` where `type.inward == "is satisfied by"` or `type.name == "Satisfy"` → extract `inwardIssue.key`
- If no issue links found, treat the key itself as a single epic and skip to Step 4

---

## Step 4: Fetch Each Epic

For each epic key, call `getJiraIssue`:
- fields: `["summary", "status", "assignee", "duedate", "labels", "comment", "description", "priority"]`
- Append any custom field IDs discovered in Step 2b

Capture per epic:

| Field | Source |
|---|---|
| Key | key |
| Name | fields.summary |
| Team | Derived from project key prefix, label, component, or user input |
| Jira Status | fields.status.name + statusCategory.key |
| Health Signal | Custom health/status field if discovered — use as primary signal. Fall back to computed health if not available. |
| Assignee | fields.assignee.displayName or "Unassigned" |
| Est Start | Custom date field if discovered, else null |
| Est Delivery | Custom date field if discovered, else fields.duedate |
| Actual Start | Custom date field if discovered, else null |
| Actual Delivery | Custom date field if discovered, else null |
| Days Left | Est Delivery minus today (negative = overdue) |
| Labels | fields.labels |
| Latest Comment | fields.comment.comments — last item |
| Criticality | Custom priority/criticality field if discovered, else fields.priority |

### Health Signal Priority
1. If a custom health or status field is present and populated: use it as the primary health label
2. If absent or null: fall back to computing health from due date + progress (see Step 8)
3. Never show "No Signal" silently — flag as "Health field empty, using computed health"

### Team Detection (if not in fields)
1. Check project key prefix and ask the user to confirm the mapping if unknown
2. Check labels for team or squad identifiers
3. Check components field
4. Ask user if still ambiguous

---

## Step 5: Fetch Child Tasks per Epic (Work Completion %)

For each epic, call `searchJiraIssuesUsingJql`:

Primary query: `"Epic Link" = EPIC-KEY OR parent = EPIC-KEY`
Fallback query: `issueFunction in subtasksOf("key = EPIC-KEY")`

- fields: `["summary", "status", "assignee", "comment", "priority", "created", "updated"]`
- maxResults: 100

Compute per epic:
- `total_items` = count of issues returned
- `done_items` = count where `statusCategory.key == "done"`
- `in_progress_items` = count where `statusCategory.key == "indeterminate"`
- `todo_items` = count where `statusCategory.key == "new"`
- `work_pct` = round(done_items / total_items * 100) if total_items > 0 else 0

Display as: `62% (15/24 work items done)`

Program-wide totals:
- `program_total` = sum of all epics' total_items
- `program_done` = sum of all epics' done_items
- `program_pct` = round(program_done / program_total * 100)

If total_items == 0 after both queries:
- Set work_pct = 0
- Set status label = "No Tasks Linked"
- Flag in epic table AND in a dedicated "Needs Attention" section
- Add recommended action: "Ask squad lead to break down scope into child tasks"
- Do NOT show "—" silently

If total_items > 100:
- Note: "% is approximate — capped at 100 items sampled"

---

## Step 6: Deep-Dive Comments for Hidden Context

For each epic AND each child task where:
- Epic status is NOT Done, AND
- work_pct < 80% OR the epic has a blocker label OR days_left < 14

Scan all comments (epic-level and child task-level) for:

**Blocker keywords**: blocked, waiting on, dependency, external, pending, stuck, no movement,
not deployed, not in staging, unassigned, no owner, escalate, at risk, overdue, delayed,
pushed, deferred, not started, no response, waiting for approval

**Decision keywords**: need to decide, pending decision, waiting for confirmation, needs sign-off,
descope, defer, scope cut, reprioritize, no ETA

**Positive keywords**: deployed, merged, shipped, done, released, resolved, fixed, closed

For each epic, extract:
- Latest blocker mention (date + author + excerpt, max 80 chars)
- Latest decision pending (date + author + excerpt)
- Any positive signals contradicting a "stuck" status

Surface in the chat summary and Confluence page under "Context from Comments."
This catches cases where status says "In Progress" but a comment from 3 days ago says
"waiting on external team, no ETA."

---

## Step 7: Pull Bitbucket Data (if available)

If the Jira project has linked Bitbucket repositories (check via `getJiraIssueRemoteIssueLinks`
or ask the user):

For each linked repository, check:
- Open PRs per initiative: count PRs whose title or branch name references the epic key
- PR review age: flag any PR open for more than 5 days without a merge or close
- Commit activity: check if any commits reference the epic key in the last 7 days

Cross-reference against Jira status:
- If Jira shows "In Progress" but 0 commits in the last 7 days: flag as "No Recent Code Activity"
- If a PR has been in review for more than 5 days: flag as "PR Review Stale"
- If Jira shows "Done" but PR is still open: flag as "Jira/Code Status Mismatch"

If Bitbucket is not accessible or not linked: skip this step silently and note
"Code activity data not available — Bitbucket not linked" in the report footer.

---

## Step 8: Derive Health Signals

For each epic, assign one of four health labels:

| Label | Condition |
|---|---|
| ON TRACK | work_pct >= expected_pct AND days_left > 0 AND no blockers |
| AT RISK | work_pct < expected_pct by >10% OR blocker detected OR days_left < 14 and work_pct < 70% |
| OVERDUE | days_left < 0 AND status != Done |
| WRAP UP NEEDED | status effectively done but Jira not formally closed (work_pct >= 95% OR actual delivery set) |
| NO TASKS LINKED | total_items == 0 |
| BACKLOG | status is Backlog AND no delivery date set |

### Expected Progress Calculation
```
quarter_start = first day of the current quarter (ask user if unknown)
today = current date
days_elapsed = (today - quarter_start).days
quarter_length = 91  # approximate
expected_pct = round(days_elapsed / quarter_length * 100)
```

### Delivery Feasibility (for AT RISK and OVERDUE epics)
```
velocity_per_day = done_items / days_elapsed if days_elapsed > 0 else 0
remaining_items = in_progress_items + todo_items
projected_days_to_finish = remaining_items / velocity_per_day if velocity_per_day > 0 else 999
projected_finish_date = today + projected_days_to_finish days
gap_days = (projected_finish_date - delivery_date).days
```

If gap_days <= 0: On Track
If 0 < gap_days <= 14: Slight Risk — show Options A/B/C
If gap_days > 14: High Risk — recommend scope reduction

### Stuck Ticket Detection
Flag any child task as stuck if:
- Status has not changed in the last 7 days, AND
- Status is not Done or Won't Fix, AND
- Task is assigned (unassigned tasks are flagged separately)

Threshold is 7 days by default. If the user's team works in 2-week sprints, use 10 days.
Ask the user if unsure.

### Unassigned Task Detection
Flag any child task as unassigned if:
- assignee is null or "Unassigned", AND
- status is To Do or In Progress (not Done)

Group by epic and surface as: "N unassigned tasks on [EPIC-KEY]"

---

## Step 9: Ask for Manual Context

Before generating the report, ask the user:

> "A few things I can't get from Jira — answer any that are relevant:
> 1. Any off-Jira blockers or risks I should include? (e.g. vendor dependency, team leave, budget hold)
> 2. Any epics already decided to descope or defer that Jira hasn't been updated to reflect?
> 3. Any delivery decisions made this week that should appear as context?
> 4. Confluence space and parent page to publish to — or skip if chat summary only?"

If the user wants to skip, proceed with Jira data only and note "Manual context: none provided."

---

## Step 10: Chat Summary

Output before creating the Confluence page:

```
## Program Health: [Program Name]
[Quarter] | As of [today's date] | Delivery Risk: [Low / Medium / High]

### Quick Stats
- Total epics: N
- On track: N
- At risk: N
- Overdue: N
- No tasks linked: N
- Program completion: N% (N/N work items done)

### Flags Requiring Action This Week
[List each flag: epic key, issue, named owner, deadline]

### Epics Summary
[One line per epic: key | name | health label | work% | days left]
```

Ask: "Ready to publish to Confluence? If yes, confirm the space and parent page."

---

## Step 11: Create Confluence Page

Call `createConfluencePage` (or `updateConfluencePage` if a page already exists for this report):

### Page Title Format
`Program Health: [Program Name] — [Quarter] — [Date]`

Example: `Program Health: Advanced SCM Use-cases — 26Q2 — 7 May 2026`

### Page Structure

```
[Program key] | [Quarter] | As of [date] | Team: [team name] | Delivery Risk: [label]

## Executive Summary
- [4–5 action-oriented bullets]
- Cover: overall %, key win, key risk, decision needed, what's on track]

## Program Scope
| Metric | Value |
| Total work items | N |
| Completed | N (N%) |
| Total epics | N |
| Delivered | N |
| In progress | N |

## Epic Status Overview
| Epic | Team | Health | Progress | Est Start / Actual | Est Delivery | Actual Delivery | Days Left |
[one row per epic]

## Timeline Analysis
[One paragraph per epic with notable timing, velocity, or delay context]

## Risk and Deadline Flags
### Confirmed Blockers
[Named blocker per epic — what is blocked, by what, who needs to act, by when]

### Needs Attention
[Stuck tickets, stale PRs, unassigned tasks, Jira/code mismatches]

### Clear
[Epics with no active flags]

## Recommended Actions
| # | Action | Owner | By When | Priority |
[Numbered list, High before Medium]

---
Generated from Jira via MCP | Source: [program key] | [date]
```

---

## Health Label Reference

| Label | Display | Meaning |
|---|---|---|
| ON TRACK | 🟢 ON TRACK | Progress matches or exceeds expected pace, no blockers |
| AT RISK | 🟡 AT RISK | Behind expected pace or blocker detected |
| OVERDUE | 🔴 OVERDUE | Past delivery date, not done |
| WRAP UP NEEDED | WRAP UP NEEDED | Functionally done but Jira not closed |
| NO TASKS LINKED | NO TASKS LINKED | Epic has no child work items |
| BACKLOG | BACKLOG | No active work, no delivery date |

---

## Writing Style

- No em dashes. Use commas, colons, or short separate sentences instead.
- No AI-sounding filler: avoid "it is worth noting", "as previously mentioned", "it is important to highlight."
- Bullet points in latest update columns — not run-on sentences.
- Name specific people and tickets in flags and actions.
- State things plainly: "Blocked on SSO team" not "Currently pending resolution of the SSO dependency."
- Active voice, short sentences.
- Emoji used only as functional status markers (🔴🟡🟢✅) in tables, not in body text.

---

## Edge Cases

| Situation | Handling |
|---|---|
| No linked epics on program | Tell user, offer to check a different key |
| Epic fetch fails | Show "Data unavailable" in that row, do not abort |
| No due dates on any epic | Flag all as BACKLOG, recommend setting dates as first action |
| JQL returns 0 stories | Flag "No Tasks Linked", do not show "—" silently |
| More than 100 stories per epic | Note "% approximate, capped at 100 items" |
| Bitbucket not linked | Skip Step 7, note in report footer |
| Cloud ID unknown | Run getAccessibleAtlassianResources first |
| Confluence space not found | Run getConfluenceSpaces to list options |
| User wants chat summary only | Skip Confluence page creation |
| Epic assignee is null | Do not flag as a gap — epics are often intentionally unassigned. Surface most active child task assignee as de-facto owner instead |
| Zero assigned child tasks | Flag as genuine ownership gap, add recommended action |

---

## Custom Field Discovery Guide

This skill does not hardcode any custom field IDs because every Jira instance configures
them differently. Use Step 2b to discover fields at runtime.

### Standard fields that exist in every Jira instance (always safe to fetch)

| Field | Key |
|---|---|
| Summary | summary |
| Status | status |
| Assignee | assignee |
| Due date | duedate |
| Labels | labels |
| Comments | comment |
| Description | description |
| Priority | priority |
| Issue links | issuelinks |
| Created | created |
| Updated | updated |

### Fields to discover via `getJiraIssueTypeMetaWithFields`

Look for fields matching these concepts — the actual IDs vary per instance:

- Planned start date
- Planned delivery / target date
- Actual start date
- Actual delivery date
- Health or RAG status
- Team or squad
- Criticality or severity
- Quarter or planning period

### If field discovery is not possible

Fall back to standard fields only:
- Use `duedate` as the delivery date
- Use `status` and computed progress as the health signal
- Use `priority` as criticality
- Ask the user to provide any key dates not in Jira
