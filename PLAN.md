# Plan: Manager Mode + Multi-Category Handling

## Problem
1. When a manager (Biena) files a ticket, the workflow runs checks against Biena instead of the associates listed in `issue_details`
2. `issue_details` is free text — managers can mention different bonus categories per employee (e.g., "Attendance Bonus Ei Husain Performance Bonus John Smith")
3. Non-attendance categories (Performance, Engagement) have no workflow yet and should escalate to Team GC immediately

## Strategy: Keep Self Path Untouched, Build Manager Path

The existing self-mode flow (Nodes 7-28) works. We won't touch it. We'll fix mode detection and build the manager branch.

## Changes

### Step 1: Enhance Node 3 (Get all required parameters)

Add parsing logic to extract entries from `issue_details`:

**Category detection:** Scan for known keywords — "attendance bonus", "performance bonus", "engagement bonus"
**Name extraction:** Text between category markers (or after the last marker to end) = potential employee names
**Fallback:** If no category keywords found in text but names exist → all names inherit the dropdown `issue_type`
**Self detection:** If no parseable names found → self mode (existing behavior)

Output adds:
```json
{
  "parsedEntries": [
    { "category": "Attendance Bonus", "nameText": "Ei Husain" },
    { "category": "Performance Bonus", "nameText": "John Smith" }
  ],
  "detectedMode": "manager"  // or "self"
}
```

Name extraction approach for free text:
1. Replace known category keywords with a delimiter marker (e.g., `|||ATTENDANCE BONUS|||`)
2. Split by delimiter
3. For each section: clean out filler words (for, please, check, the, and, etc.), trim
4. Remaining text = employee name(s). Split by commas/newlines if multiple
5. For each name, split into first/last parts

### Step 2: Modify Node 5 (Merge Data) — Mode Detection

Replace current keyword-based detection:
- If `parsedEntries.length > 0` → `mode = "manager"`
- If `parsedEntries.length === 0` → `mode = "self"` (existing behavior)

Store requester info as `filedBy` (not as the investigation subject).

### Step 3: Build Manager Path (New Nodes from Node 6 "manager" output)

#### Node M1: "Split Manager Entries" (Code)
- Takes `parsedEntries` array
- Outputs N n8n items, one per entry
- Each item carries: `{category, nameText, parsedFirst, parsedLast, bonusMonth, cutoffDate, ticketId, filedByEmail, filedByName, account, teamLead}`

#### Node M2: "Lookup Employee by Name" (Microsoft SQL)
- Query per item: `SELECT TOP 5 * FROM people.employee WHERE last_name LIKE '%{{ $json.parsedLast }}%' AND first_name LIKE '%{{ $json.parsedFirst }}%'`
- n8n auto-processes each item

#### Node M3: "Validate Employee Found" (IF)
- If result is empty → mark as `EMPLOYEE_NOT_FOUND` → include in final report
- If multiple matches → flag ambiguity → escalate

#### Node M4: "Manager Category Router" (Switch)
- Routes by `category`:
  - `"Attendance Bonus"` → attendance check pipeline
  - `"Performance Bonus"` / `"Engagement Bonus"` / other → non-attendance escalation

#### Attendance Path (from M4):
Duplicate the existing check structure for manager items:
- M5: CHECK1 Overview SQL
- M6: CHECK1 Detail SQL
- M7: Merge CHECK1
- M8: Evaluate CHECK1
- M9: CHECK2 COA SQL
- M10: Evaluate CHECK2
- M11: CHECK3 SA SQL
- M12: Evaluate CHECK3
- M13: Needs History Check? (IF)
- M14: CHECK3 History SQL / No History passthrough
- M15: Final CHECK3
- M16: Merge All Checks
- M17: Verdict Engine

These are copies of Nodes 7-20 but wired into the manager path. Same SQL queries, same evaluation logic — just different source of employee data.

> **Future refactor:** Extract checks into a sub-workflow callable from both paths to eliminate duplication.

#### Non-Attendance Path (from M4):
- M18: "Build Non-Attendance Escalation" (Code) — message: "This [category] dispute for [employee] requires manual review. Category not yet automated."
- Routes to existing "Send to Team GC" Teams node (Node 23) or a new dedicated one

#### Node M19: "Aggregate Manager Results" (Code)
- Collects all per-employee verdicts (attendance checks + non-attendance escalations)
- Builds consolidated report:
  ```
  MANAGER-FILED DISPUTE — [N] Associates
  Filed by: [Manager Name] | Ticket #[ID]

  [Employee 1] — Attendance Bonus — DISQUALIFIED (Late: 2 days)
  [Employee 2] — Performance Bonus — ESCALATED (Category not automated)
  [Employee 3] — Attendance Bonus — QUALIFIES (All checks passed)
  ```

#### Node M20: "Manager Action Router" (Switch)
- If ALL attendance entries are DISQUALIFIED and no non-attendance entries → AUTO_DENY with consolidated reply
- If ANY entry needs human decision → ESCALATE_TEAM with consolidated report
- Non-attendance entries always force escalation

### Step 4: Update Architecture Diagram

```
Freshservice Ticket (Webhook)
       │
       ▼
  IF: Bonus Category?
       │
       ▼
  Enhanced Parse (bonus month, cutoff, entries, mode)
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
  ┌────┴──────────────────────┐
  self                      manager
  │                           │
  [existing Nodes 7-28]     Split Entries (1 item per employee)
                              │
                           Lookup by Name
                              │
                           Employee Found?
                              │
                           Category Router
                              │
                     ┌────────┴────────┐
               Attendance         Non-Attendance
                     │                  │
              [CHECK 1,2,3]      Build Escalation
                     │                  │
              Verdict Engine     → Team GC (not automated)
                     │
                     ├──── all done ────┤
                     │
              Aggregate Results
                     │
              Manager Action Router
                     │
              ┌──────┴──────┐
          AUTO_DENY    ESCALATE_TEAM
              │              │
        Consolidated    Consolidated
        Deny Reply     Teams Card
```

## Implementation Order

1. **Node 3** — Enhanced parsing code (testable standalone)
2. **Node 5** — Mode detection fix
3. **Nodes M1-M3** — Entry splitting + employee lookup + validation
4. **Node M4** — Category router
5. **Node M18** — Non-attendance escalation (quick win)
6. **Nodes M5-M17** — Attendance checks for manager path (duplicate of existing)
7. **Nodes M19-M20** — Result aggregation + action routing

## What I'll Provide
- Full updated code for Node 3 (enhanced parsing)
- Full updated code for Node 5 (mode detection)
- Code for each new manager-path Code node (M1, M8, M10, M12, M15, M17, M18, M19)
- SQL queries for new manager-path SQL nodes (M2, M5, M6, M9, M11, M14)
- Switch/IF configurations for M3, M4, M13, M20
- Updated architecture guide
