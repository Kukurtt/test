# n8n Attendance Bonus Dispute — Asymmetric Authority Workflow

> **Core Principle:** AI can auto-deny. AI can NEVER auto-approve. Every approval requires a human button click in Teams.

## Architecture

```
Freshservice Ticket (Webhook)
       │
       ▼
  IF: Bonus Category?
       │
       ▼
  Enhanced Parse (bonus month, cutoff, issue_details → entries, mode detection)
       │
       ▼
  Lookup Requester (people.employee by email)
       │
       ▼
  Merge Data + Mode Detection
       │
       ▼
  Filing Mode Switch
       │
  ┌────┴──────────────────────────┐
  self                          manager
  │                               │
  │                         Split Entries (1 item per employee+category)
  │                               │
  │                         Lookup Employee by Name (SQL per item)
  │                               │
  │                         Validate Employee Match
  │                               │
  │                         Category Router
  │                          ┌────┴──────────┐
  │                    Attendance       Non-Attendance / Not Found
  │                          │               │
  ├── CHECK 1              CHECK 1         Escalation (not automated)
  ├── CHECK 2              CHECK 2           │
  └── CHECK 3              CHECK 3           │
       │                     │               │
  Merge All Checks     Mgr Verdict     ─────┤
       │                     │               │
  Verdict Engine        Merge (Append) ◄─────┘
       │                     │
  Action Router        Aggregate Manager Results
       │                     │
  ┌────┴──────────┐    Manager Action Router
  AUTO_DENY    ESCALATE   ┌──┴──────────┐
  │            _TEAM    AUTO_DENY    ESCALATE_TEAM
  │              │        │              │
  Build Deny   Send to  Mgr Deny     Send Mgr Report
  Reply        Team GC  Reply        to Team GC
  │            (Custom   │           (Custom Form:
  │            Form)     │           Approve All / Deny All /
  (TODO:         │       (TODO:      Review Individually /
  Freshservice Route     Freshservice Escalate to Higher-Ups)
  Reply+Close) Decision  Reply+Close)
             ┌─┼──────────────┐
           Approve Deny  Escalate
             │      │        │
           Approve Team   Send to Higher-Ups GC
           Reply   Deny   (Teams: Custom Form)
             │     Reply   [Approve / Deny]
             │      │        │
             (TODO: Freshservice Reply + Close)
```

## Node Inventory

| # | Node Name | Type | Purpose |
|---|-----------|------|---------|
| 1 | Ticket received via webhook | Webhook | POST trigger from Freshservice |
| 2 | If Subject Contains Bonus | IF | Filter: `body.issue_type` contains "Bonus" (accepts all categories) |
| 3 | Get all required parameters | Code | Parse bonus month, cutoff, email parts, issue_details entries, mode detection |
| 4 | Lookup Employee SQL | Microsoft SQL | Find employee by email in `people.employee` |
| 5 | Merge Data | Code | Combine employee + context, use parsedEntries for mode detection |
| 6 | Filing Mode | Switch | Route by `mode`: self or manager |
| 7 | CHECK1 Overview | Microsoft SQL | `analytics_powerbi.V_fact_attendance` by employee_id |
| 8 | CHECK1 Detail | Microsoft SQL | `staging_sprout.V_attendance_with_schedule` by name |
| 9 | Merge CHECK1 | Merge | Combine overview + detail |
| 10 | Evaluate CHECK1 | Code | Flag tardy/absent/undertime days |
| 11 | CHECK2 COA | Microsoft SQL | `staging_integration.sprout_employee_COA_HTML` by employee_id |
| 12 | Evaluate CHECK2 | Code | coa_total > 2 = FAILED |
| 13 | CHECK3 SA | Microsoft SQL | `staging_sprout.schedule_adjustments` by name |
| 14 | Evaluate CHECK3 | Code | Check pending SAs + identify late-approval candidates |
| 15 | Needs History Check? | IF | Routes based on CHECK3 status = NEEDS_HISTORY_CHECK |
| 16 | CHECK3 History | Microsoft SQL | Query approval dates for flagged SA ids |
| 17 | Final CHECK3 | Code | Compare approval dates vs cutoff |
| 18 | CHECK3 No History | Code | Pass-through when no history check needed |
| 19 | Merge All Checks | Merge | Combine CHECK1 + CHECK2 + CHECK3 results |
| 20 | Verdict Engine | Code | Determine verdict + action |
| 21 | Action Router | Switch | Route: AUTO_DENY or ESCALATE_TEAM |
| 22 | Build Deny Reply | Code | Generate denial email with specific reasons |
| 23 | Send to Team GC | Microsoft Teams | Send and Wait for Response (Custom Form) |
| 24 | Route Team Decision | Switch | Route: Approve / Deny / Escalate to Higher-Ups |
| 25 | Approve Reply | Code | Generate approval email |
| 26 | Team Deny Reply | Code | Generate team-confirmed denial email |
| 27 | Build Higher-Ups Card | Code | Generate escalation message |
| 28 | Send to Higher-Ups GC | Microsoft Teams | Send and Wait for Response (Custom Form) |

### Manager Path Nodes

| # | Node Name | Type | Purpose |
|---|-----------|------|---------|
| M1 | Split Manager Entries | Code | Split parsedEntries into 1 item per employee+category |
| M2 | Lookup Employee by Name | Microsoft SQL | `people.employee` by first/last name per item |
| M3 | Match & Validate Employees | Code | Group SQL results by entry, pick best match, flag not-found/ambiguous |
| M4 | Manager Category Router | Switch | Route: NOT_FOUND → M18, Attendance → checks, Other → M18 |
| M5-M17 | Manager Attendance Checks | Various | Copies of Nodes 7-20 wired into manager path |
| M18 | Non-Attendance / Not-Found Escalation | Code | Build escalation info for non-automated categories or unresolved names |
| M19 | Aggregate Manager Results | Code | Collect all per-employee verdicts into consolidated report |
| M20 | Manager Action Router | Switch | Route: AUTO_DENY or ESCALATE_TEAM |
| M21 | Build Manager Deny Reply | Code | Consolidated denial email for all associates |
| M22 | Send Manager Report to Team GC | Microsoft Teams | Send and Wait (Custom Form: Approve All/Deny All/Review Individually/Escalate) |
| M23 | Route Manager Team Decision | Switch | Route: Approve All / Deny All / Review Individually / Escalate to Higher-Ups |
| M24 | Build Manager Approve Reply | Code | Consolidated approval email for all associates |
| M25 | Build Manager Team-Deny Reply | Code | Consolidated team-confirmed denial email |
| M26 | Split for Individual Review | Code | Split aggregated results back into per-employee items |
| M27 | Send Individual Card to Team GC | Microsoft Teams | Send and Wait per employee (Approve/Deny/Escalate) |
| M28 | Collect Individual Decisions | Code | Aggregate individual decisions into final response |

## Key Configuration Details

### Freshservice Webhook Body
Configured in Freshservice Workflow Automator → Trigger Webhook action:
```json
{
  "account": "{{ticket.ri_513_cf_account}}",
  "team_lead": "{{ticket.ri_513_cf_team_lead}}",
  "bonus_month": "{{ticket.ri_513_cf_untitled}}",
  "issue_type": "{{ticket.ri_513_cf_issue_type}}",
  "bonus_month_detail": "{{ticket.ri_513_cf_please_specify_the_date_s}}",
  "issue_details": "{{ticket.ri_513_cf_issue_details}}",
  "requester_email": "REQUESTER_EMAIL_PLACEHOLDER",
  "ticket_id": "TICKET_ID_PLACEHOLDER"
}
```

### Employee Lookup Strategy
- Email → `people.employee` table (match by last_name + first_name from email prefix)
- Returns `employee_id` (unique integer) used for cross-table joins
- Handles name ambiguity (multiple employees with same last name) via email-based matching

### SQL Tables Used
| Table | Database | Used For |
|-------|----------|----------|
| `people.employee` | people | Employee lookup (id, name, email) |
| `analytics_powerbi.V_fact_attendance` | analytics_powerbi | CHECK1 overview (uses employee_id) |
| `staging_sprout.V_attendance_with_schedule` | staging_sprout | CHECK1 detail (uses First/Last Name) |
| `staging_integration.sprout_employee_COA_HTML` | staging_integration | CHECK2 (uses employee_id) |
| `staging_sprout.schedule_adjustments` | staging_sprout | CHECK3 (uses firstName/lastName) |

### Verdict Logic (Asymmetric Authority)
| Condition | Verdict | Action |
|-----------|---------|--------|
| Any check FAILED | DISQUALIFIED | AUTO_DENY (AI decides) |
| All checks PASSED | QUALIFIES | ESCALATE_TEAM (human decides) |
| Any check ERROR | INCOMPLETE_DATA | ESCALATE_TEAM (human decides) |
| Other | UNCERTAIN | ESCALATE_TEAM (human decides) |

### Teams Custom Form Configuration
Both "Send to Team GC" and "Send to Higher-Ups GC" use:
- **Operation:** Send and Wait for Response
- **Response Type:** Custom Form
- **Form Field:** "Decision" (Options type)
- **Response path:** `$json.data.Decision`

Team GC options: `Approve`, `Deny`, `Escalate to Higher-Ups`
Higher-Ups GC options: `Approve`, `Deny`

### Mode Detection (Self vs Manager)
Node 3 parses `issue_details` for employee names and bonus category mentions:
- **Self:** No parseable employee names found in issue_details → requester IS the employee
- **Manager:** One or more `{category, nameText}` entries parsed from issue_details → requester filed on behalf of others

**Parsing strategy:**
1. Scan for known category keywords: "attendance bonus", "performance bonus", "engagement bonus"
2. Split text into segments by category markers
3. Extract names from each segment (strip filler words, split by commas/and)
4. If no category keywords found → all names default to dropdown `issue_type`
5. SQL lookup validates each parsed name against `people.employee`

### Multi-Category Handling
- **Attendance Bonus:** Full 3-check pipeline (same as self mode)
- **Performance Bonus / Engagement Bonus:** Immediately escalated to Team GC as "not yet automated"
- **Employee not found:** Escalated with "manual lookup required" flag
- **Ambiguous match:** Best match selected; flagged if uncertain

## Policy Rules Implemented

### CHECK 1: Attendance & Tardiness
- **Late:** In Time > scheduled start (SA Start Time if exists, else Original Shift Start Time). Even seconds count. Any late = DISQUALIFIED.
- **Absent/LWOP:** In Time or Out Time null on a working day. Any = DISQUALIFIED.
- **Undertime:** Out Time < scheduled end, or undertime_hours > 0 in overview. Any = DISQUALIFIED.

### CHECK 2: COA
- `coa_total > 2` = DISQUALIFIED
- Table refreshes monthly; previous month view shows bonus month data
- Exemptions: system downtime (coa_exemption table), client travel (not yet automated)

### CHECK 3: Schedule Adjustments
- **Pending (statusId=1):** Any pending SA with bonus month dates = DISQUALIFIED
- **Late approval (statusId=4):** Query history for MIN(elt_created_datetime) WHERE statusId=4. If approval date > cutoff = DISQUALIFIED
- Cutoff = 3rd business day of month following bonus month

## Remaining Work

### High Priority
- [ ] Freshservice API integration (reply to ticket + resolve/close) for all branches
- [x] Manager-filed multi-employee loop (detect names → lookup each → run checks per employee) — **IMPLEMENTED & WIRED**
- [x] Multi-category handling (Attendance auto-checked, Performance/Engagement escalated) — **IMPLEMENTED & WIRED**
- [ ] Teams message formatting (line breaks not rendering in Adaptive Cards from Send and Wait)
- [x] Wire up manager attendance check nodes in n8n — **DONE** (SplitInBatches loop + combined Evaluate & Verdict node)
- [x] Review Individually flow (per-employee Teams cards) — **DONE**
- [x] Higher-Ups escalation from manager path — **DONE**
- [ ] Test manager mode end-to-end with real ticket data

### Medium Priority
- [ ] Processing status tracker (`dispute_processing_log` SQL table)
- [ ] Circuit breaker (15+ denials/hour → pause + alert)
- [ ] 4-hour reminder ping for unanswered escalations
- [ ] 24-hour fallback to Team GC if Higher-Ups don't respond
- [ ] Refactor attendance checks into sub-workflow (eliminate duplication between self/manager paths)

### Future
- [ ] Performance bonus category workflow (full automation, not just escalation)
- [ ] Engagement bonus category workflow (full automation, not just escalation)
- [ ] Precedent database for institutional memory
- [ ] Batched cutoff dispute approval
- [ ] Weekly health report
- [ ] COA exemption automation (system downtime + client travel)
