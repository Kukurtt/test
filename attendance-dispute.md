# Attendance Bonus Dispute Investigation

Investigate whether **$ARGUMENTS**'s attendance bonus disqualification is valid.

## Bonus Month
Always the **previous month** from today's date.

## Cutoff
3rd business day of the month following the bonus month.

## Checks (run all 3 in parallel)

### CHECK 1: Attendance & Tardiness

**Step 1 — Overview:** Query `analytics_powerbi.V_fact_attendance` (`system_name LIKE '%name%'`, bonus month). Summarize working/rest/leave days. Flag any `tardiness_hours > 0`, `undertime_hours > 0`, or `capacity_hours > 0` with `worked_hours` null/significantly less than capacity (absent/LWOP).

**Step 2 — Detail:** Query `staging_sprout.V_attendance_with_schedule` (`[First Name]`/`[Last Name]` LIKE, `[Is Rest Day]=0`, bonus month). Use `[SA Start Time]` if not null, else `[Original Shift Start Time]`. Check:
- **Late** = In Time > scheduled start. Even seconds count. Any late = **DISQUALIFIED**.
- **Absent/LWOP** = `[In Time]` or `[Out Time]` is null on a working day. Any = **DISQUALIFIED**.
- **Undertime** = `undertime_hours > 0` in overview, or Out Time < scheduled end. Any = **DISQUALIFIED**.

### CHECK 2: COA
Query `staging_integration.sprout_employee_COA_HTML` (`employee_name LIKE '%Last%First%'`). Found with `coa_total > 2` = **DISQUALIFIED**.

### CHECK 3: Schedule Adjustments
Query `staging_sprout.schedule_adjustments` (`firstName`/`lastName`, `elt_current_ind='Y'`). Parse `details` JSON for dates in bonus month. Two sub-checks:

- **Pending (statusId=1):** Any pending SA with bonus month dates = **DISQUALIFIED**.
- **Late approval (statusId=4):** For approved SAs with bonus month dates, query history (remove `elt_current_ind` filter, same `id`). Find `MIN(elt_created_datetime) WHERE statusId=4` — this is when approval was captured. If that date > cutoff = **DISQUALIFIED** (data wasn't reflected at cutoff).

Include supervisor info and `dateFiled` for any flagged SAs.

## Verdict
**DISQUALIFIED** if ANY check fails. **QUALIFIES** only if all 3 pass. No subjective overrides.

## Output

### Part 1: Investigation Report
```
============================================
  ATTENDANCE BONUS DISPUTE INVESTIGATION
============================================
Employee: [Full Name] | Bonus Month: [Month Year] ([range])
Investigation Date: [Today]

--- CHECK 1: ATTENDANCE & TARDINESS ---
Status: [PASSED/FAILED]
Overview: [X] working days, [X] rest, [X] leave, [X] absent/LWOP
Tardiness: [X] days | Undertime: [X] days | Absent/LWOP: [X] days
Detail: [table of flagged dates or "All on time, complete entries"]

--- CHECK 2: COA ---
Status: [PASSED/DISQUALIFIED]
[Details if found]

--- CHECK 3: SCHEDULE ADJUSTMENTS ---
Status: [PASSED/DISQUALIFIED]
[Pending/late-approved SA details + Supervisor *** FLAG ***]

=== VERDICT: [QUALIFIED/DISQUALIFIED] ===
[Which checks passed/failed]
============================================
```

### Part 2: Email-Ready Response
**Do NOT generate Part 2 automatically.** After presenting the Investigation Report (Part 1) and Verdict, ask the user if they agree with the verdict. Only generate the email-ready response after the user confirms.
When generating: Concise, warm, professional, copy-paste ready, under 150 words. No bullet lists, no technical IDs.

## Notes
- Use `mcp__azure-sql__read_data` for all queries.
- Use LIKE '%name%' for flexible matching.
- Present raw data evidence in Part 1.
