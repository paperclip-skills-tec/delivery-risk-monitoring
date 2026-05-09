---
name: delivery-risk-monitoring
description: "Delivery Risk Monitor — standardized early-warning scan for project health signals. Use this skill whenever you are running a Delivery Risk Monitor routine, performing a daily project health scan, checking for scope creep or blocker accumulation, scanning for stakeholder disengagement, or producing a delivery risk report. Governs signal taxonomy, detection thresholds, severity levels (🔴 critical / 🟡 warning), output format, and escalation rules. Always invoke before interpreting issue data as risk signals — do not improvise thresholds or formats."
---

# Delivery Risk Monitoring Protocol

This skill defines **what to look for**, **when it's a problem**, and **how to report it**. Follow the exact signal taxonomy, thresholds, and output formats below. Do not invent new signals or adjust thresholds mid-run.

---

## Step 1 — Establish Scan Scope

Before pulling data, define what you are scanning. The scan always covers:

- **Active issues:** `status = todo, in_progress, in_review, blocked` across all projects
- **Timeframe:** last 7 days for trend signals; all-time for current-state signals
- **Excluded:** `done`, `cancelled`, `backlog` issues (unless they were recently closed and are context for a signal)

Pull active issues in one call:

```
GET /api/companies/{companyId}/issues?status=todo,in_progress,in_review,blocked
```

Also pull the dashboard for agent-level health context:

```
GET /api/companies/{companyId}/dashboard
```

Log your scan scope in the opening line of your output.

---

## Step 2 — Detect Signals

For each signal type, apply the exact detection criteria and threshold below. A signal fires only when the threshold is crossed — not when you're "worried". If the threshold is not crossed, the signal does not appear in the report.

### Signal A: Scope Creep

**Definition:** New high-priority (`critical` or `high`) issues added to an active project sprint within the last 7 days that were not present at the start of that window.

**Detection:** Filter issues with `priority = critical OR high` and `createdAt > now - 7d`. Group by project/goal. Count net-new issues added this week.

**Thresholds:**
- 🟡 Warning: 2–4 new critical/high issues in a single project this week
- 🔴 Critical: 5+ new critical/high issues in a single project this week, OR any new `critical` issue added after a project milestone was declared frozen

**Why it matters:** Uncontrolled scope addition mid-sprint increases WIP, delays existing commitments, and signals either poor backlog management or reactive firefighting.

---

### Signal B: Repeated Blockers

**Definition:** An issue that has been in `blocked` status for 2 or more consecutive days, or has been set to `blocked` more than once.

**Detection:** For each currently `blocked` issue, check `updatedAt` vs today. If `updatedAt < now - 2d` and `status = blocked`, the issue has been persistently blocked.

**Thresholds:**
- 🟡 Warning: blocked for 2–4 days with no comment activity in the last 24h
- 🔴 Critical: blocked for 5+ days, OR blocked with `priority = critical` for any duration, OR same issue has been blocked-then-unblocked-then-blocked again (re-blocking pattern)

**Re-blocking pattern check:** Scan the issue comment thread for prior `blocked` → `in_progress` transitions. If found, escalate severity to critical regardless of time.

---

### Signal C: Stakeholder Disengagement

**Definition:** An issue in `in_review` with `assigneeUserId` set (board-assigned for review) that has had no comment activity for N days.

**Detection:** Filter issues with `status = in_review` AND `assigneeUserId != null`. For each, check when the last comment was posted.

**Thresholds:**
- 🟡 Warning: in_review with no board comment for 3–5 days
- 🔴 Critical: in_review with no board comment for 6+ days, OR a `critical`-priority issue awaiting review for 2+ days

**Why it matters:** Stalled reviews block delivery pipelines and create uncertainty for agents waiting on decisions.

---

### Signal D: Resource Conflict

**Definition:** A single agent assigned more than N concurrent `critical` or `high` priority issues in `in_progress` status.

**Detection:** Group `in_progress` issues by `assigneeAgentId`. Count critical + high issues per agent.

**Thresholds:**
- 🟡 Warning: 2 critical/high in_progress issues assigned to the same agent
- 🔴 Critical: 3+ critical/high in_progress issues assigned to the same agent, OR any agent assigned both a `critical` issue and a routine execution issue with overlapping timing

**Why it matters:** Agents context-switching across multiple critical tasks deliver none of them well. This signal flags situations where an assignment triage is overdue.

---

## Step 3 — Score and Prioritize

After running all four detectors:

1. Collect all fired signals
2. Assign severity: any single 🔴 critical signal makes the overall scan status **ALERT**; 🟡 warning-only makes it **WATCH**; no signals makes it **CLEAR**
3. Sort signals by severity (critical first), then by age (oldest first within a tier)

If no signals fired, skip to the all-clear output in Step 5.

---

## Step 4 — Write the Alert Report

Use this exact template. Do not add prose paragraphs, do not omit sections, do not reorder columns.

```markdown
## Delivery Risk Monitor — {DATE}

**Scan scope:** All active issues · Projects: {N} · Issues scanned: {N}
**Overall status:** 🔴 ALERT  _(or 🟡 WATCH)_

---

### 🔴 Critical Signals

| Signal | Issue | Age | Recommended Action |
|--------|-------|-----|-------------------|
| Repeated Blocker | [TEC-XXXX](/TEC/issues/TEC-XXXX) — title | 7d | Escalate to board — unblock owner: @AgentName |
| Resource Conflict | [@AgentName](/TEC/agents/agentkey) | — | Reassign TEC-YYYY to free capacity |

### 🟡 Warning Signals

| Signal | Issue | Age | Recommended Action |
|--------|-------|-----|-------------------|
| Stakeholder Disengagement | [TEC-XXXX](/TEC/issues/TEC-XXXX) — title | 4d | Ping board: re-pin for review |
| Scope Creep | Project Ascend | +3 high issues | Review with board before sprint end |

---

_Next scan: tomorrow. Escalated items require acknowledgement before this signal clears._
```

**Field rules:**
- **Signal**: use the exact signal name from Step 2 (no paraphrasing)
- **Issue**: always link using `[TEC-XXXX](/TEC/issues/TEC-XXXX)` format; for resource-conflict signals, link the agent instead
- **Age**: time since the signal condition first became true (e.g., `3d` = 3 days)
- **Recommended Action**: one short sentence — who does what. Name the agent or role responsible.

If a signal type has no firings, omit its section entirely (do not include an empty table).

---

## Step 5 — All-Clear Format

When no signals fire, the entire output is a single line — no tables, no body, no recommendations:

```
✅ Delivery Risk Monitor — {DATE} — All clear. {N} issues scanned across {N} projects. No signals above threshold.
```

Post this as a comment on the routine execution issue and mark it `done`. Do not post a full report when no signals exist — it creates alert fatigue.

---

## Step 6 — Escalation Rules

Apply this decision tree for every critical signal before exiting the heartbeat:

| Condition | Action |
|-----------|--------|
| Any 🔴 signal involving a `critical`-priority issue | Create a new issue assigned to the board (`assigneeUserId: "local-board"`) with title `🔴 Delivery Risk: {signal type} on {issue/agent}` and link the source issue via `blockedByIssueIds` |
| 🔴 Repeated Blocker on any issue ≥ 5 days | Comment on the blocked issue with `[@Chief of Staff](agent://7de0f97a-1196-456c-8ef0-073eac20e67b)` and explain the staleness |
| 🔴 Resource Conflict with 3+ critical issues on one agent | Comment on the routine issue recommending a specific reassignment; do NOT reassign unilaterally |
| 🟡 Warning only | Comment on the relevant issue thread; do not create a new board issue |
| Signal resolves between scans | Note resolution in the next all-clear or next report under a `### Resolved Since Last Scan` section |

**Never escalate warnings directly to the board.** Warnings belong in the issue thread. Only 🔴 critical signals warrant a board-level issue.

---

## Step 7 — Post Output and Close

1. Post the full alert report (or all-clear line) as a comment on the routine execution issue.
2. For each critical signal requiring board escalation, create the escalation issue (Step 6) before marking the routine done.
3. Set the routine execution issue to `done` with the completion comment rule satisfied (heading `## Done` or `## Result` + no blocker language).
4. Do not leave any critical signal unacknowledged — if you cannot create an escalation issue due to a budget or permission error, set the routine issue to `blocked` and name the constraint.

---

## Reference: Signal Threshold Summary

| Signal | 🟡 Warning | 🔴 Critical |
|--------|-----------|------------|
| Scope Creep | 2–4 new critical/high issues this week | 5+ OR frozen milestone violated |
| Repeated Blocker | Blocked 2–4 days, no activity | Blocked 5+ days OR re-blocking pattern OR critical priority |
| Stakeholder Disengagement | In review 3–5 days, no board comment | In review 6+ days OR critical priority 2+ days |
| Resource Conflict | 2 critical/high in_progress on one agent | 3+ critical/high in_progress on one agent |

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*
