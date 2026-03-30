# Analytics - Bonus Disputes: Complete Workflow Reference

> **Workflow ID:** `5Ol9zvnHASarHAGx`
> **Name:** Analytics - Bonus Disputes
> **Active:** true
> **Archived:** false
> **Created:** 2026-03-15T11:32:21.801Z
> **Updated:** 2026-03-16T00:45:32.412Z
> **Version ID:** 79f3aa1b-6243-4ecc-8da3-5c65ee5f8c13
> **Active Version ID:** 1fe25b9f-ea3a-4b49-90c5-b0fc5a7bdc98
> **Version Counter:** 365
> **Trigger Count:** 1
> **Owner Project:** nbfxOSGxgb3fUlj1 (Biena Villagonzalo, personal)

---

## Table of Contents

1. [All Nodes (30 total)](#all-nodes)
2. [Connections (raw JSON)](#connections)
3. [Credentials](#credentials)
4. [Settings](#settings)

---

## All Nodes

### Node 1: Ticket received via webhook

| Field | Value |
|-------|-------|
| **id** | `25d2c38c-0814-4d11-85d4-5c958d8bfa30` |
| **name** | Ticket received via webhook |
| **type** | `n8n-nodes-base.webhook` |
| **typeVersion** | 2.1 |
| **position** | [0, 0] |
| **webhookId** | `b408717a-4924-41eb-90d6-dcd2fd151f47` |

**Parameters:**
- `httpMethod`: `"POST"`
- `path`: `"b408717a-4924-41eb-90d6-dcd2fd151f47"`
- `options`: `{}`

**Description:** Receives incoming POST webhook requests. This is the workflow trigger -- external systems (e.g., a ticketing system) send bonus dispute tickets to this endpoint.

---

### Node 2: If Subject Contains {Category} Bonus

| Field | Value |
|-------|-------|
| **id** | `fc06ce2e-b8af-4848-a4bb-a346f1085bfa` |
| **name** | If Subject Contains {Category} Bonus |
| **type** | `n8n-nodes-base.if` |
| **typeVersion** | 2.3 |
| **position** | [208, 0] |

**Parameters (conditions):**
- **Combinator:** `and`
- **Condition 1:**
  - id: `bb7974d7-8b95-4ff2-811b-dc3822b0bfb1`
  - leftValue: `={{ $json.body.issue_type }}`
  - rightValue: `"Attendance Bonus"`
  - operator: `string` / `contains`
- **Options:** caseSensitive=true, typeValidation=strict, version=3

**Description:** Filters tickets: only those whose `issue_type` contains "Attendance Bonus" continue to the true branch. Others go to the false branch (currently unconnected).

---

### Node 3: Get all required parameters

| Field | Value |
|-------|-------|
| **id** | `91a117ca-cd45-47a9-adbd-55b2b9c17e72` |
| **name** | Get all required parameters |
| **type** | `n8n-nodes-base.code` |
| **typeVersion** | 2 |
| **position** | [416, -96] |

**Full jsCode:**
```javascript
  // Extract employee info from the webhook
  const requesterEmail = $input.first().json.body.requester_email;
  const ticketId = $input.first().json.body.ticket_id;
  const issueType = $input.first().json.body.issue_type;
  const issueDetails = $input.first().json.body.issue_details;
  const account = $input.first().json.body.account;
  const teamLead = $input.first().json.body.team_lead;

  // Parse the bonus month from the ticket
  // bonus_month_detail = "February", bonus_month = "Feb"
  const monthName =  $input.first().json.body.bonus_month_detail|| $input.first().json.body.bonus_month;

  // Convert month name to date range
  const monthMap = {
    'january': 0, 'february': 1, 'march': 2, 'april': 3,
    'may': 4, 'june': 5, 'july': 6, 'august': 7,
    'september': 8, 'october': 9, 'november': 10, 'december': 11,
    'jan': 0, 'feb': 1, 'mar': 2, 'apr': 3,
    'jun': 5, 'jul': 6, 'aug': 7, 'sep': 8,
    'oct': 9, 'nov': 10, 'dec': 11
  };

  const monthIndex = monthMap[monthName.toLowerCase()];
  const now = new Date();

  // Determine the year (if month is ahead of current month, it's previous year)
  let year = now.getFullYear();
  if (monthIndex > now.getMonth()) {
    year = year - 1;
  }

  const bonusMonthStart = new Date(year, monthIndex, 1);
  const bonusMonthEnd = new Date(year, monthIndex + 1, 0); // last day of month

  // Format as YYYY-MM-DD for SQL
  const fmt = (d) => d.toISOString().slice(0, 10);

  // Calculate cutoff: 3rd business day of the month AFTER the bonus month
  const cutoffMonthStart = new Date(year, monthIndex + 1, 1);
  let businessDays = 0;
  let cutoffDate = new Date(cutoffMonthStart);
  cutoffDate.setDate(0); // last day of bonus month
  while (businessDays < 3) {
    cutoffDate.setDate(cutoffDate.getDate() + 1);
    const day = cutoffDate.getDay();
    if (day !== 0 && day !== 6) businessDays++;
  }

    // Extract last name from email for SQL lookup
  // email format: first.last@domain.com
  const emailPrefix = requesterEmail.split('@')[0]; // "el.husain"
  const emailParts = emailPrefix.split('.');         // ["el", "husain"]
  const emailLastName = emailParts.length > 1 ? emailParts[emailParts.length - 1] : emailParts[0];
  const emailFirstPart = emailParts[0] || '';

  return [{
    json: {
      ticketId,
      requesterEmail,
      issueType,
      issueDetails,
      account,
      teamLead,
      emailLastName,    // "husain" — for SQL lookup
      emailFirstPart,   // "el" — partial first name hint
      bonusMonth: {
        start: fmt(bonusMonthStart),
        end: fmt(bonusMonthEnd),
        label: bonusMonthStart.toLocaleDateString('en-US', { month: 'long', year: 'numeric' })
      },
      cutoffDate: fmt(cutoffDate)
    }
  }];
```

**Description:** Extracts all required fields from the webhook body (requesterEmail, ticketId, issueType, issueDetails, account, teamLead). Parses the bonus month name into start/end date range. Calculates the cutoff date (3rd business day of the following month). Extracts first/last name parts from the requester email for SQL employee lookup.

---

### Node 4: Lookup Employee SQL

| Field | Value |
|-------|-------|
| **id** | `a879779d-8a01-4243-83a4-4ca3155620d0` |
| **name** | Lookup Employee SQL |
| **type** | `n8n-nodes-base.microsoftSql` |
| **typeVersion** | 1.1 |
| **position** | [624, -96] |
| **alwaysOutputData** | true |

**Credentials:** `microsoftSql` -- id: `GbaRqtLuVtlqs0TT`, name: `Microsoft SQL account`

**Full SQL Query:**
```sql
  SELECT TOP 1 id, first_name, last_name, email
  FROM people.employee
  WHERE last_name LIKE '%{{ $json.emailLastName }}%'
    AND first_name LIKE '{{ $json.emailFirstPart }}%'
    AND deleted_at IS NULL
```

**Description:** Looks up the employee record by matching the last name and first name parts extracted from the requester's email. Returns at most 1 row from `people.employee`.

---

### Node 5: Code (Employee Found Check)

| Field | Value |
|-------|-------|
| **id** | `2d2b8bf9-fa49-4237-b156-a23987f5e507` |
| **name** | Code |
| **type** | `n8n-nodes-base.if` |
| **typeVersion** | 2.3 |
| **position** | [832, -96] |

**Parameters (conditions):**
- **Combinator:** `and`
- **Condition 1:**
  - id: `afa589e4-cead-4974-8c9c-4e692a7f4c34`
  - leftValue: `={{ $json.first_name }}`
  - rightValue: `""` (empty)
  - operator: `string` / `notEmpty` (singleValue=true)
- **Options:** caseSensitive=true, typeValidation=strict, version=3

**Description:** Checks if the employee lookup returned a result (first_name is not empty). True branch continues to Merge Data; false branch is unconnected.

---

### Node 6: Merge Data

| Field | Value |
|-------|-------|
| **id** | `f79ed667-ff20-452d-8195-9c39607591ca` |
| **name** | Merge Data |
| **type** | `n8n-nodes-base.code` |
| **typeVersion** | 2 |
| **position** | [1040, -192] |

**Full jsCode:**
```javascript
const employee = $input.first()?.json || {};
const context = $('Get all required parameters').first().json;

// Detect: is the requester the employee or a TL/manager?
const requesterEmail = context.requesterEmail?.toLowerCase();
const teamLead = context.teamLead;
const issueDetails = context.issueDetails || '';

function extractNames(text) {
  if (!text) return [];

  const triggerPattern =
    /(?:affected|check|following|employees?|associates?|names?\s*(?:are|:)|listed|below|team\s*members?)\s*[:\-]?\s*(.+)/i;

  const match = text.match(triggerPattern);
  if (!match) return [];

  const nameSection = match[1];

  const nonNameWords =
    /records|absence|late|minute|sprout|bonus|attendance|system|time|shift|schedule|please|would|could|should|file|check|advise|appropriate/i;

  const parts = nameSection
    .split(/,|\band\b/i)
    .map(s => s.trim())
    .map(s => s.replace(/[.?!]+$/, '').trim())
    .filter(s => {
      if (s.length < 3) return false;
      if (!/[a-zA-Z]{2,}/.test(s)) return false;

      const words = s.split(/\s+/).filter(w => w.length > 1);
      if (words.length < 2 || words.length > 4) return false;

      if (nonNameWords.test(s)) return false;

      return true;
    });

  return parts;
}

const extractedNames = extractNames(issueDetails);

// Check if requester email matches employee email
const requesterFound =
  employee?.email &&
  employee.email.toLowerCase() === requesterEmail;

const hasExtractedNames = extractedNames.length > 0;

let employees = [];
let mode = 'self';

if (requesterFound && !hasExtractedNames) {
  mode = 'self';

  employees = [
    {
      id: employee.id,
      firstName: employee.first_name,
      lastName: employee.last_name,
      fullName: `${employee.first_name} ${employee.last_name}`,
      source: 'requester'
    }
  ];
}
else if (hasExtractedNames) {
  mode = 'manager';
  // Names will be looked up later
}

return [
  {
    json: {
      ticketId: context.ticketId,
      requesterEmail: context.requesterEmail,
      issueType: context.issueType,
      issueDetails: context.issueDetails,
      account: context.account,
      teamLead: context.teamLead,
      bonusMonth: context.bonusMonth,
      cutoffDate: context.cutoffDate,
      mode,
      employees,
      extractedNames,
      requesterEmployee: requesterFound
        ? {
            id: employee.id,
            firstName: employee.first_name,
            lastName: employee.last_name
          }
        : null
    }
  }
];
```

**Description:** Merges employee SQL result with the extracted parameters. Determines filing mode: `"self"` (requester is the employee) or `"manager"` (names extracted from issue_details indicate a TL/manager filing on behalf of others). Builds the unified context object with employees array and extracted names.

---

### Node 7: Filing Mode

| Field | Value |
|-------|-------|
| **id** | `ac3ca058-faa2-4f33-9423-29353d9ff615` |
| **name** | Filing Mode |
| **type** | `n8n-nodes-base.switch` |
| **typeVersion** | 3.4 |
| **position** | [1184, -192] |

**Parameters (rules):**

**Rule 1 (Output: `self`):**
- leftValue: `={{ $json.mode }}`
- rightValue: `"self"`
- operator: `string` / `equals`
- id: `731314d6-411d-404e-b2c0-bd2fec734408`

**Rule 2 (Output: `manager`):**
- leftValue: `={{ $json.mode }}`
- rightValue: `"manager"`
- operator: `string` / `equals`
- id: `23f29e0d-a438-4999-98b4-17a13dcf0cf1`

**Description:** Routes to different branches based on filing mode. Output 0 (`self`) goes to CHECK queries. Output 1 (`manager`) currently goes to an empty connection (future manager-mode handling).

---

### Node 8: CHECK1 Overview

| Field | Value |
|-------|-------|
| **id** | `a77250e1-4a2f-4d6a-81ae-991b9690cdb4` |
| **name** | CHECK1 Overview |
| **type** | `n8n-nodes-base.microsoftSql` |
| **typeVersion** | 1.1 |
| **position** | [1696, -304] |
| **alwaysOutputData** | true |

**Credentials:** `microsoftSql` -- id: `GbaRqtLuVtlqs0TT`, name: `Microsoft SQL account`

**Full SQL Query:**
```sql
  SELECT *
  FROM analytics_powerbi.V_fact_attendance
  WHERE employee_id = {{ $json.employees[0].id }}
    AND date >= '{{ $json.bonusMonth.start }}'
    AND date <= '{{ $json.bonusMonth.end }}'
  ORDER BY date
```

**Description:** Retrieves the attendance overview/fact data for the employee during the bonus month from the Power BI analytics view. Used to cross-check undertime from the overview data.

---

### Node 9: CHECK1 Detail

| Field | Value |
|-------|-------|
| **id** | `85b7b1e7-6197-4585-a3a3-66f88cf082a8` |
| **name** | CHECK1 Detail |
| **type** | `n8n-nodes-base.microsoftSql` |
| **typeVersion** | 1.1 |
| **position** | [1696, -144] |
| **alwaysOutputData** | true |

**Credentials:** `microsoftSql` -- id: `GbaRqtLuVtlqs0TT`, name: `Microsoft SQL account`

**Full SQL Query:**
```sql
  SELECT * FROM staging_sprout.V_attendance_with_schedule                                          WHERE [First Name] LIKE '%{{ $json.employees[0].firstName }}%'
      AND [Last Name] LIKE '%{{ $json.employees[0].lastName }}%'
      AND [Is Rest Day] = 0
      AND [Date] >= '{{ $json.bonusMonth.start }}'
      AND [Date] <= '{{ $json.bonusMonth.end }}'
  ORDER BY [Date]
```

**Description:** Retrieves detailed Sprout attendance-with-schedule records for the employee during the bonus month. Filters out rest days. This is the primary data source for CHECK1 evaluation (tardiness, absence, undertime).

---

### Node 10: Merge CHECK1

| Field | Value |
|-------|-------|
| **id** | `c6a53003-6826-423b-9d1e-e95ccff0a0ea` |
| **name** | Merge CHECK1 |
| **type** | `n8n-nodes-base.merge` |
| **typeVersion** | 3.2 |
| **position** | [1904, -224] |

**Parameters:** `{}` (default merge -- appends inputs)

**Description:** Merges the outputs of CHECK1 Overview (input 0) and CHECK1 Detail (input 1) so the Evaluate CHECK1 node can access both datasets.

---

### Node 11: Evaluate CHECK1

| Field | Value |
|-------|-------|
| **id** | `1d346488-1f98-43ed-a513-5f539ebb8ce2` |
| **name** | Evaluate CHECK1 |
| **type** | `n8n-nodes-base.code` |
| **typeVersion** | 2 |
| **position** | [2112, -224] |

**Full jsCode:**
```javascript
// Reference both SQL nodes by name
  const overview = $('CHECK1 Overview').all();
  const detail = $('CHECK1 Detail').all();

  // Helper: parse "6:00:00 AM" to seconds since midnight
  function parseTime(timeStr) {
    if (!timeStr) return null;
    const match = timeStr.match(/(\d+):(\d+):(\d+)\s*(AM|PM)/i);
    if (!match) return null;
    let hours = parseInt(match[1]);
    const minutes = parseInt(match[2]);
    const seconds = parseInt(match[3]);
    const period = match[4].toUpperCase();
    if (period === 'PM' && hours !== 12) hours += 12;
    if (period === 'AM' && hours === 12) hours = 0;
    return hours * 3600 + minutes * 60 + seconds;
  }

  let tardyDays = [];
  let absentDays = [];
  let undertimeDays = [];

  for (const row of detail) {
    const d = row.json;
    const date = d['Date'];

    // Scheduled start: SA Start Time if exists, else Original Shift Start Time
    const scheduledStart = d['SA Start Time'] || d['Original Shift Start Time'];
    const scheduledEnd = d['SA End Time'] || d['Original Shift End Time'];

    const inTime = d['In Time'];
    const outTime = d['Out Time'];

    // CHECK: Absent/LWOP — null In Time or Out Time on working day
    if (!inTime || !outTime) {
      absentDays.push({ date, inTime: inTime || 'MISSING', outTime: outTime || 'MISSING' });
      continue; // no point checking late/undertime if absent
    }

    // CHECK: Late — In Time > Scheduled Start (even seconds count)
    const inSeconds = parseTime(inTime);
    const startSeconds = parseTime(scheduledStart);
    if (inSeconds !== null && startSeconds !== null && inSeconds > startSeconds) {
      tardyDays.push({ date, inTime, scheduledStart });
    }

    // CHECK: Undertime — Out Time < Scheduled End
    const outSeconds = parseTime(outTime);
    const endSeconds = parseTime(scheduledEnd);
    if (outSeconds !== null && endSeconds !== null && outSeconds < endSeconds) {
      undertimeDays.push({ date, outTime, scheduledEnd });
    }
  }

  // Also flag undertime from overview (catches cases detail might miss)
  for (const row of overview) {
    const d = row.json;
    if (d.undertime_hours && parseFloat(d.undertime_hours) > 0) {
      const alreadyFlagged = undertimeDays.some(u => u.date === d.date);
      if (!alreadyFlagged) {
        undertimeDays.push({ date: d.date, source: 'overview', undertimeHours: d.undertime_hours });
      }
    }
  }

  const failed = tardyDays.length > 0 || absentDays.length > 0 || undertimeDays.length > 0;

  return [{
    json: {
      check: 'ATTENDANCE_TARDINESS',
      status: failed ? 'FAILED' : 'PASSED',
      summary: {
        totalWorkingDays: detail.length,
        tardyDays: tardyDays.length,
        absentDays: absentDays.length,
        undertimeDays: undertimeDays.length
      },
      flaggedDates: { tardyDays, absentDays, undertimeDays }
    }
  }];
```

**Description:** Evaluates CHECK1 (Attendance/Tardiness). Parses time strings, compares In Time vs Scheduled Start (late), Out Time vs Scheduled End (undertime), and flags absent days (missing In/Out Time). Also cross-references the overview for undertime. Returns PASSED or FAILED with detailed summary and flagged dates.

---

### Node 12: CHECK2 COA

| Field | Value |
|-------|-------|
| **id** | `f8cdf4fc-df44-40ad-87fa-2c393bc374a4` |
| **name** | CHECK2 COA |
| **type** | `n8n-nodes-base.microsoftSql` |
| **typeVersion** | 1.1 |
| **position** | [1696, 16] |
| **alwaysOutputData** | true |

**Credentials:** `microsoftSql` -- id: `GbaRqtLuVtlqs0TT`, name: `Microsoft SQL account`

**Full SQL Query:**
```sql
  SELECT *
  FROM staging_integration.sprout_employee_COA_HTML
  WHERE employee_id = {{ $json.employees[0].id }}
```

**Description:** Retrieves all Change of Attendance (COA) records for the employee from the Sprout integration staging table.

---

### Node 13: Evaluate CHECK2

| Field | Value |
|-------|-------|
| **id** | `f3ac3777-0665-4fe0-8512-875431527435` |
| **name** | Evaluate CHECK2 |
| **type** | `n8n-nodes-base.code` |
| **typeVersion** | 2 |
| **position** | [1904, 16] |

**Full jsCode:**
```javascript
  const results = $('CHECK2 COA').all();

  // No results = 0 COAs = PASSED
  // Has results = check coa_total
  let coaTotal = 0;
  if (results.length > 0 && results[0].json.coa_total !== undefined) {
    coaTotal = parseInt(results[0].json.coa_total) || 0;
  }

  return [{
    json: {
      check: 'COA',
      status: coaTotal > 2 ? 'FAILED' : 'PASSED',
      coaTotal,
      details: results.map(r => r.json)
    }
  }];
```

**Description:** Evaluates CHECK2 (COA). Counts total COA filings. If coa_total exceeds 2, the check FAILS. Otherwise PASSES.

---

### Node 14: CHECK3 SA

| Field | Value |
|-------|-------|
| **id** | `50a95c05-7e83-4746-9170-dd4e20f096bc` |
| **name** | CHECK3 SA |
| **type** | `n8n-nodes-base.microsoftSql` |
| **typeVersion** | 1.1 |
| **position** | [1696, 160] |
| **alwaysOutputData** | true |

**Credentials:** `microsoftSql` -- id: `GbaRqtLuVtlqs0TT`, name: `Microsoft SQL account`

**Full SQL Query:**
```sql
SELECT *
  FROM staging_sprout.schedule_adjustments
  WHERE firstName LIKE '%{{ $json.employees[0].firstName }}%'
    AND lastName LIKE '%{{ $json.employees[0].lastName }}%'
    AND elt_current_ind = 'Y'
```

**Description:** Retrieves all current schedule adjustment records for the employee from Sprout staging.

---

### Node 15: Evaluate CHECK3

| Field | Value |
|-------|-------|
| **id** | `ff547d08-98e3-4681-942a-b7a60f1554c5` |
| **name** | Evaluate CHECK3 |
| **type** | `n8n-nodes-base.code` |
| **typeVersion** | 2 |
| **position** | [1904, 160] |

**Full jsCode:**
```javascript
  const results = $input.all();
  const bonusStart = $('Filing Mode').first().json.bonusMonth.start;
  const bonusEnd = $('Filing Mode').first().json.bonusMonth.end;
  const cutoffDate = $('Filing Mode').first().json.cutoffDate;

  let pendingSAs = [];
  let needsHistoryCheck = []; // statusId=4 SAs in bonus month

  for (const row of results) {
    const sa = row.json;

    // Skip empty items (from "Always Output Data")
    if (!sa.id) continue;

    // Parse details JSON to find dates in bonus month
    let details = [];
    try {
      details = typeof sa.details === 'string' ? JSON.parse(sa.details) : sa.details;
      if (!Array.isArray(details)) details = [details];
    } catch(e) {
      details = [];
    }

    // Check if any date in details falls within bonus month
    const bonusMonthDates = details.filter(d => {
      const date = (d.date || '').slice(0, 10);
      return date >= bonusStart && date <= bonusEnd;
    });

    if (bonusMonthDates.length === 0) continue; // SA not in bonus month

    // Sub-check 1: Pending (statusId = 1)
    if (sa.statusId === 1 || sa.statusId === '1') {
      pendingSAs.push({
        id: sa.id,
        dates: bonusMonthDates.map(d => d.date),
        reason: sa.reason,
        dateFiled: sa.dateFiled,
        supervisor: sa.supervisor_firstName + ' ' + sa.supervisor_lastName
      });
    }

    // Sub-check 2: Approved (statusId = 4) — needs history check
    if (sa.statusId === 4 || sa.statusId === '4') {
      needsHistoryCheck.push({
        id: sa.id,
        dates: bonusMonthDates.map(d => d.date),
        reason: sa.reason,
        dateFiled: sa.dateFiled,
        supervisor: sa.supervisor_firstName + ' ' + sa.supervisor_lastName
      });
    }
  }

  // If there are pending SAs, already FAILED
  const failedFromPending = pendingSAs.length > 0;

  return [{
    json: {
      check: 'SCHEDULE_ADJUSTMENTS',
      pendingSAs,
      needsHistoryCheck,
      failedFromPending,
      cutoffDate,
      // If no history check needed and no pending, we can determine status now
      status: failedFromPending ? 'FAILED' :
              (needsHistoryCheck.length > 0 ? 'NEEDS_HISTORY_CHECK' : 'PASSED')
    }
  }];
```

**Description:** Evaluates CHECK3 (Schedule Adjustments). Parses each SA's details JSON to find dates within the bonus month. Flags pending SAs (statusId=1) as immediate FAIL. Approved SAs (statusId=4) require a history check to see if they were approved after the cutoff date. Returns FAILED, NEEDS_HISTORY_CHECK, or PASSED.

---

### Node 16: Needs History Check?

| Field | Value |
|-------|-------|
| **id** | `e6d6925e-17af-4b98-bf05-b276b9d77b4e` |
| **name** | Needs History Check? |
| **type** | `n8n-nodes-base.if` |
| **typeVersion** | 2.3 |
| **position** | [2112, 160] |

**Parameters (conditions):**
- **Combinator:** `and`
- **Condition 1:**
  - id: `f5b5a970-11a3-4ec4-a9ec-21379ead940f`
  - leftValue: `={{ $json.status }}`
  - rightValue: `"NEEDS_HISTORY_CHECK"`
  - operator: `string` / `equals`
- **Options:** caseSensitive=true, typeValidation=strict, version=3

**Description:** If Evaluate CHECK3 returned status "NEEDS_HISTORY_CHECK", routes to the true branch (CHECK3 History SQL). Otherwise routes to CHECK3 No History.

---

### Node 17: CHECK3 History

| Field | Value |
|-------|-------|
| **id** | `8b4229f5-e1b7-425f-9559-9ed32042b737` |
| **name** | CHECK3 History |
| **type** | `n8n-nodes-base.microsoftSql` |
| **typeVersion** | 1.1 |
| **position** | [2320, 64] |
| **alwaysOutputData** | true |

**Credentials:** `microsoftSql` -- id: `GbaRqtLuVtlqs0TT`, name: `Microsoft SQL account`

**Full SQL Query:**
```sql
  SELECT
    id,
    MIN(elt_created_datetime) AS approval_date
  FROM staging_sprout.schedule_adjustments
  WHERE id IN ({{ $json.needsHistoryCheck.map(sa => sa.id).join(',') }})
    AND statusId = 4
  GROUP BY id
```

**Description:** For SAs flagged as needing a history check, retrieves the earliest created datetime (approval_date) to determine whether approval came before or after the cutoff date.

---

### Node 18: Final CHECK3

| Field | Value |
|-------|-------|
| **id** | `6d3d2cae-1ef5-4915-8ced-08fc7c07f51d` |
| **name** | Final CHECK3 |
| **type** | `n8n-nodes-base.code` |
| **typeVersion** | 2 |
| **position** | [2528, 64] |

**Full jsCode:**
```javascript
  const historyResults = $input.all();
  const check3 = $('Evaluate CHECK3').first().json;
  const cutoff = check3.cutoffDate;

  let lateApprovedSAs = [];

  for (const row of historyResults) {
    if (!row.json.id) continue;

    const approvalDate = (row.json.approval_date || '').slice(0, 10);

    if (approvalDate > cutoff) {
      // Find the original SA details from the earlier evaluation
      const original = check3.needsHistoryCheck.find(sa => sa.id === row.json.id);
      lateApprovedSAs.push({
        id: row.json.id,
        approvalDate: row.json.approval_date,
        cutoffDate: cutoff,
        ...original
      });
    }
  }

  const failed = check3.failedFromPending || lateApprovedSAs.length > 0;

  return [{
    json: {
      check: 'SCHEDULE_ADJUSTMENTS',
      status: failed ? 'FAILED' : 'PASSED',
      pendingSAs: check3.pendingSAs,
      lateApprovedSAs,
      summary: {
        pendingCount: check3.pendingSAs.length,
        lateApprovedCount: lateApprovedSAs.length
      }
    }
  }];
```

**Description:** Compares each SA's approval_date against the cutoff. If approved after cutoff, it's flagged as late-approved. Combined with pending SAs, determines the final CHECK3 status (FAILED or PASSED).

---

### Node 19: CHECK3 No History

| Field | Value |
|-------|-------|
| **id** | `6e09f46c-6fc4-4fee-8102-f282b7abef82` |
| **name** | CHECK3 No History |
| **type** | `n8n-nodes-base.code` |
| **typeVersion** | 2 |
| **position** | [2320, 256] |
| **alwaysOutputData** | true |

**Full jsCode:**
```javascript
  const check3 = $input.first().json;
  return [{
    json: {
      check: 'SCHEDULE_ADJUSTMENTS',
      status: check3.status,
      pendingSAs: check3.pendingSAs,
      lateApprovedSAs: [],
      summary: {
        pendingCount: check3.pendingSAs.length,
        lateApprovedCount: 0
      }
    }
  }];
```

**Description:** Passthrough for when no history check is needed. Forwards the CHECK3 status as-is with lateApprovedSAs=[] and lateApprovedCount=0.

---

### Node 20: Merge all CHECKS

| Field | Value |
|-------|-------|
| **id** | `f28eb398-2ecd-413e-aad8-098aa0401885` |
| **name** | Merge all CHECKS |
| **type** | `n8n-nodes-base.merge` |
| **typeVersion** | 3.2 |
| **position** | [2768, -16] |

**Parameters:**
- `numberInputs`: `4`

**Description:** 4-input merge node that combines results from Evaluate CHECK1 (input 0), Evaluate CHECK2 (input 1), Final CHECK3 (input 2), and CHECK3 No History (input 3).

---

### Node 21: Verdict Engine

| Field | Value |
|-------|-------|
| **id** | `4521c6bd-e9c4-47b0-b0b2-9e7d257c91fd` |
| **name** | Verdict Engine |
| **type** | `n8n-nodes-base.code` |
| **typeVersion** | 2 |
| **position** | [2912, 16] |

**Full jsCode:**
```javascript
  const check1 = $('Evaluate CHECK1').first().json;
  const check2 = $('Evaluate CHECK2').first().json;

  // CHECK3 could come from either Final CHECK3 or CHECK3 No History
  let check3;
  try { check3 = $('Final CHECK3').first().json; } catch(e) {
    try { check3 = $('CHECK3 No History').first().json; } catch(e2) {
      check3 = { status: 'ERROR' };
    }
  }

  // Get ticket context from Filing Mode
  const context = $('Filing Mode').first().json;

  const anyError = [check1, check2, check3].some(c => c.status === 'ERROR');
  const anyFailed = [check1, check2, check3].some(c => c.status === 'FAILED');
  const allPassed = [check1, check2, check3].every(c => c.status === 'PASSED');

  let verdict, action;

  if (anyError) {
    verdict = 'INCOMPLETE_DATA';
    action = 'ESCALATE_TEAM';       // Safeguard #1: never auto-deny on missing data
  } else if (anyFailed) {
    verdict = 'DISQUALIFIED';
    action = 'AUTO_DENY';            // AI CAN deny
  } else if (allPassed) {
    verdict = 'QUALIFIES';
    action = 'ESCALATE_TEAM';        // AI CANNOT approve
  } else {
    verdict = 'UNCERTAIN';
    action = 'ESCALATE_TEAM';
  }

  return [{
    json: {
      verdict,
      action,
      checks: {
        check1: { status: check1.status, summary: check1.summary, flaggedDates: check1.flaggedDates },
        check2: { status: check2.status, coaTotal: check2.coaTotal },
        check3: { status: check3.status, summary: check3.summary, pendingSAs: check3.pendingSAs, lateApprovedSAs:
  check3.lateApprovedSAs }
      },
      employee: context.employees[0],
      bonusMonth: context.bonusMonth,
      ticketId: context.ticketId,
      cutoffDate: context.cutoffDate,
      requesterEmail: context.requesterEmail,
      account: context.account,
      teamLead: context.teamLead
    }
  }];
```

**Description:** The core decision engine. Aggregates all 3 checks. Applies asymmetric authority rules:
- **Any ERROR** -> INCOMPLETE_DATA -> ESCALATE_TEAM (never auto-deny on missing data)
- **Any FAILED** -> DISQUALIFIED -> AUTO_DENY (AI can deny)
- **All PASSED** -> QUALIFIES -> ESCALATE_TEAM (AI cannot approve)
- **Otherwise** -> UNCERTAIN -> ESCALATE_TEAM

---

### Node 22: Action Router

| Field | Value |
|-------|-------|
| **id** | `5dec86a2-1957-408b-8a33-efbd5c3ba913` |
| **name** | Action Router |
| **type** | `n8n-nodes-base.switch` |
| **typeVersion** | 3.4 |
| **position** | [3120, 16] |

**Parameters (rules):**

**Rule 1 (Output: `auto_deny`):**
- leftValue: `={{ $json.action }}`
- rightValue: `"AUTO_DENY"`
- operator: `string` / `equals`
- id: `55e7b941-3851-443f-87b1-638a4d6a017d`

**Rule 2 (Output: `escalate`):**
- leftValue: `={{ $json.action }}`
- rightValue: `"ESCALATE_TEAM"`
- operator: `string` / `equals`
- id: `4a6d4d79-b444-4b71-a628-6e27a2893563`

**Description:** Routes based on the Verdict Engine's action decision. Output 0 (`auto_deny`) -> Auto Deny Reply. Output 1 (`escalate`) -> Build Teams Card.

---

### Node 23: Auto Deny Reply

| Field | Value |
|-------|-------|
| **id** | `316cec66-2576-447c-8c80-9508dbd7f366` |
| **name** | Auto Deny Reply |
| **type** | `n8n-nodes-base.code` |
| **typeVersion** | 2 |
| **position** | [3328, -48] |
| **alwaysOutputData** | true |

**Full jsCode:**
```javascript
  const d = $input.first().json;
  const emp = d.employee;
  const checks = d.checks;

  // Build reason list from failed checks
  const reasons = [];

  if (checks.check1.status === 'FAILED') {
    const s = checks.check1.summary;
    const parts = [];
    if (s.tardyDays > 0) parts.push(`${s.tardyDays} late arrival(s)`);
    if (s.absentDays > 0) parts.push(`${s.absentDays} absent/LWOP day(s)`);
    if (s.undertimeDays > 0) parts.push(`${s.undertimeDays} undertime day(s)`);
    reasons.push(`Attendance: ${parts.join(', ')} recorded during ${d.bonusMonth.label}`);
  }

  if (checks.check2.status === 'FAILED') {
    reasons.push(`Change of Attendance: ${checks.check2.coaTotal} COA filing(s) found, exceeding the allowed limit`);
  }

  if (checks.check3.status === 'FAILED') {
    const s = checks.check3.summary || {};
    const parts = [];
    if (s.pendingCount > 0) parts.push(`${s.pendingCount} pending schedule adjustment(s)`);
    if (s.lateApprovedCount > 0) parts.push(`${s.lateApprovedCount} schedule adjustment(s) approved after cutoff`);
    reasons.push(`Schedule Adjustments: ${parts.join(', ')}`);
  }

  const reasonText = reasons.map((r, i) => `${i + 1}. ${r}`).join('\n');

  const reply = `Hi ${emp.firstName},

  Thank you for reaching out regarding your ${d.bonusMonth.label} attendance bonus. We have reviewed your records and unfortunately,
  the disqualification stands based on the following:

  ${reasonText}

  Our records are based on system data as of the cutoff date (${d.cutoffDate}). If you believe there is a discrepancy, please ensure
  all filings are corrected in Sprout and submit a new dispute for the next review cycle.

  We understand this may be disappointing, and we appreciate your understanding.

  Best regards,
  IM - Analytics`;

  return [{
    json: {
      ...d,
      replyBody: reply
    }
  }];
```

**Description:** Generates an automatic denial reply with specific reasons derived from the failed checks. Includes numbered reason list (attendance issues, COA count, schedule adjustments). The reply is personalized with the employee's first name and bonus month label. Currently the output connection is empty (no further node sends this reply).

---

### Node 24: Build Teams Card

| Field | Value |
|-------|-------|
| **id** | `1c2c94c1-e869-4347-a3bc-72ef48ec5530` |
| **name** | Build Teams Card |
| **type** | `n8n-nodes-base.code` |
| **typeVersion** | 2 |
| **position** | [3328, 112] |

**Full jsCode:**
```javascript
const d = $input.first().json;

const checks = d.checks || {};
const check1 = checks.check1 || {};
const check2 = checks.check2 || {};
const check3 = checks.check3 || {};

const summary1 = check1.summary || {};
const summary3 = check3.summary || {};

let aiAnalysis;

switch (d.verdict) {
  case 'QUALIFIES':
    aiAnalysis =
      'All 3 checks PASSED. AI found no disqualification reason. ' +
      'Escalated because AI cannot auto-approve (asymmetric authority).';
    break;

  case 'INCOMPLETE_DATA':
    aiAnalysis =
      'Data is incomplete or contradictory. ' +
      'AI could not make a confident verdict. Human review required.';
    break;

  default:
    aiAnalysis = 'AI could not determine a reliable verdict. Human review required.';
}

const checkSummary = [
  `CHECK 1 (Attendance): ${check1.status || '?'} — ` +
    `${summary1.totalWorkingDays ?? '?'} days, ` +
    `${summary1.tardyDays ?? 0} tardy, ` +
    `${summary1.absentDays ?? 0} absent, ` +
    `${summary1.undertimeDays ?? 0} undertime`,

  `CHECK 2 (COA): ${check2.status || '?'} — ${check2.coaTotal ?? 0} COAs`,

  `CHECK 3 (Schedule Adj): ${check3.status || '?'} — ` +
    `${summary3.pendingCount ?? 0} pending, ` +
    `${summary3.lateApprovedCount ?? 0} late-approved`

].join('\n');

const card = {
  type: "AdaptiveCard",
  $schema: "http://adaptivecards.io/schemas/adaptive-card.json",
  version: "1.4",

  body: [
    {
      type: "TextBlock",
      text: "Attendance Bonus Dispute — Decision Required",
      weight: "Bolder",
      size: "Medium"
    },

    {
      type: "FactSet",
      facts: [
        { title: "Employee", value: d.employee?.fullName || "Unknown" },
        { title: "Bonus Month", value: d.bonusMonth?.label || "Unknown" },
        { title: "Account", value: d.account || "Unknown" },
        { title: "AI Verdict", value: d.verdict || "Unknown" },
        { title: "Ticket #", value: String(d.ticketId || "") }
      ]
    },

    {
      type: "TextBlock",
      text: aiAnalysis,
      wrap: true
    },

    {
      type: "TextBlock",
      text: checkSummary,
      wrap: true
    }
  ],

  actions: [
    {
      type: "Action.Submit",
      title: "Approve",
      data: {
        action: "APPROVE",
        ticketId: d.ticketId,
        employee: d.employee?.fullName,
        bonusMonth: d.bonusMonth?.label,
        requesterEmail: d.requesterEmail
      }
    },

    {
      type: "Action.Submit",
      title: "Deny",
      data: {
        action: "DENY",
        ticketId: d.ticketId,
        employee: d.employee?.fullName,
        bonusMonth: d.bonusMonth?.label,
        requesterEmail: d.requesterEmail
      }
    },

    {
      type: "Action.Submit",
      title: "Escalate to Higher-Ups",
      data: {
        action: "ESCALATE_HIGHER",
        ticketId: d.ticketId,
        employee: d.employee?.fullName,
        bonusMonth: d.bonusMonth?.label,
        requesterEmail: d.requesterEmail,
        checks: JSON.stringify(checks)
      }
    }
  ]
};

return [
  {
    json: {
      ...d,
      adaptiveCard: card
    }
  }
];
```

**Description:** Builds a Microsoft Teams Adaptive Card for human decision-making. Includes employee info, AI verdict, check summaries, and three action buttons (Approve, Deny, Escalate to Higher-Ups). Used for ESCALATE_TEAM cases where AI cannot auto-approve.

---

### Node 25: Send to Team GC

| Field | Value |
|-------|-------|
| **id** | `a8783856-4414-4aea-9b4e-b169608d2286` |
| **name** | Send to Team GC |
| **type** | `n8n-nodes-base.microsoftTeams` |
| **typeVersion** | 2 |
| **position** | [3536, 112] |
| **webhookId** | `819652e5-678e-460f-99ed-f28e5f4b122c` |

**Credentials:** `microsoftTeamsOAuth2Api` -- id: `Hy7nmGVPixldBadk`, name: `Service@smartsourcing`

**Parameters:**
- `resource`: `"chatMessage"`
- `operation`: `"sendAndWait"`
- `chatId`:
  - value: `"19:13edc4883f2f468c81325b61573fb9ce@thread.v2"`
  - mode: `"list"`
  - cachedResultName: `"IM Analytics - n8n Workflow Error (group)"`
  - cachedResultUrl: `"https://teams.microsoft.com/l/chat/19%3A13edc4883f2f468c81325b61573fb9ce%40thread.v2/0?tenantId=f425cdcb-8a88-4e36-a0b7-1a8a9f91e2ee"`
- `responseType`: `"customForm"`
- `formFields.values`:
  - Field 1: label=`"Decision"`, type=`"dropdown"`, required=true, options: `["Approve", "Deny", "Escalate to PX-IM"]`
- `message` (template):
```
  Attendance Bonus Dispute — Decision Required

  Employee: {{ $json.employee.fullName }}
  Bonus Month: {{ $json.bonusMonth.label }}
  Account: {{ $json.account }}
  Ticket #: {{ $json.ticketId }}

  AI Verdict: {{ $json.verdict }}

  CHECK 1 (Attendance): {{ $json.checks.check1.status }}
    Working Days: {{ $json.checks.check1.summary.totalWorkingDays }} | Tardy: {{ $json.checks.check1.summary.tardyDays }} | Absent: {{ $json.checks.check1.summary.absentDays }} | Undertime: {{ $json.checks.check1.summary.undertimeDays }}

  CHECK 2 (COA): {{ $json.checks.check2.status }} — {{ $json.checks.check2.coaTotal }} COAs

  CHECK 3 (Schedule Adj): {{ $json.checks.check3.status }}
    Pending: {{ $json.checks.check3.summary.pendingCount }} | Late Approved: {{ $json.checks.check3.summary.lateApprovedCount }}

  AI cannot auto-approve. Human decision required.
```

**Description:** Sends a message to the Teams group chat ("IM Analytics - n8n Workflow Error") with a custom form containing a Decision dropdown (Approve/Deny/Escalate to PX-IM). Uses sendAndWait to pause workflow until a human responds.

---

### Node 26: Route Team Decision

| Field | Value |
|-------|-------|
| **id** | `30ab8290-83d0-4543-8407-1093b7df15f1` |
| **name** | Route Team Decision |
| **type** | `n8n-nodes-base.switch` |
| **typeVersion** | 3.4 |
| **position** | [3744, 112] |
| **alwaysOutputData** | false |
| **retryOnFail** | false |

**Parameters (rules):**

**Rule 1 (Output: `approve`):**
- leftValue: `={{ $json.data.Decision }}`
- rightValue: `"Approve"`
- operator: `string` / `equals`
- id: `ada8d8d2-9da9-4071-80d4-d8521e481951`

**Rule 2 (Output: `deny`):**
- leftValue: `={{ $json.data.Decision }}`
- rightValue: `"Deny"`
- operator: `string` / `equals`
- id: `0acb4fa5-988a-4c7e-8c5f-e6d9169ff4ca`

**Rule 3 (Output: `escalate_higher`):**
- leftValue: `={{ $json.data.Decision }}`
- rightValue: `"Escalate to PX-IM"`
- operator: `string` / `equals`
- id: `efe27512-70d8-4721-bef5-6bf05990fbe4`

**Description:** Routes based on the team's decision from the Teams form. Output 0 -> Approve Reply, Output 1 -> Team Deny Reply, Output 2 -> PX-IM Escalate.

---

### Node 27: Approve Reply

| Field | Value |
|-------|-------|
| **id** | `f7f10d39-bfe1-4cb9-bc54-77ab1d89b372` |
| **name** | Approve Reply |
| **type** | `n8n-nodes-base.code` |
| **typeVersion** | 2 |
| **position** | [3984, -64] |

**Full jsCode:**
```javascript
const d = $('Verdict Engine').first().json || {};
const emp = d.employee || {};

const firstName = emp.firstName || 'there';
const bonusMonth = d.bonusMonth?.label || 'this bonus period';

const reply = `Hi ${firstName},

Thank you for reaching out regarding your ${bonusMonth} attendance bonus.

After a thorough review of your records, we are happy to confirm that your dispute has been approved. The adjustment will be reflected in your next payout cycle.

If you have any further questions, feel free to reach out.

Best regards,
IM - Analytics`;

return [
  {
    json: {
      ...d,
      replyBody: reply,
      decision: 'Approved'
    }
  }
];
```

**Description:** Generates an approval reply email body for the employee. Personalized with first name and bonus month. Currently a terminal node (no outgoing connections).

---

### Node 28: Team Deny Reply

| Field | Value |
|-------|-------|
| **id** | `124aa930-81ed-4c44-872a-21966ff222da` |
| **name** | Team Deny Reply |
| **type** | `n8n-nodes-base.code` |
| **typeVersion** | 2 |
| **position** | [3984, 80] |

**Full jsCode:**
```javascript
const d = $('Verdict Engine').first().json || {};
const emp = d.employee || {};

const firstName = emp.firstName || 'there';
const bonusMonth = d.bonusMonth?.label || 'this bonus period';

const reply = `Hi ${firstName},

Thank you for reaching out regarding your ${bonusMonth} attendance bonus.

After a thorough review by our team, the disqualification has been upheld. If you have additional supporting information, you may submit a new dispute for further review.

We appreciate your understanding.

Best regards,
IM - Analytics`;

return [
  {
    json: {
      ...d,
      replyBody: reply,
      decision: 'Denied'
    }
  }
];
```

**Description:** Generates a team-initiated denial reply. Less detailed than the auto-deny (no specific reasons listed). Currently a terminal node (no outgoing connections).

---

### Node 29: PX-IM Escalate

| Field | Value |
|-------|-------|
| **id** | `aed3eaf1-6eeb-452a-9132-a75f13f8eafc` |
| **name** | PX-IM Escalate |
| **type** | `n8n-nodes-base.code` |
| **typeVersion** | 2 |
| **position** | [3984, 224] |

**Full jsCode:**
```javascript
  const d = $('Verdict Engine').first().json;
  const emp = d.employee;

  let summary = '';
  if (d.verdict === 'QUALIFIES') {
    summary = 'All checks passed. Team chose to escalate rather than approve directly.';
  } else if (d.verdict === 'INCOMPLETE_DATA') {
    summary = 'Data incomplete or contradictory. Team unable to make confident decision.';
  } else {
    summary = 'Team requests higher-up decision on this case.';
  }

  const message = "**Attendance Bonus Dispute — Higher-Up Decision Required**\\n\\n" +
    "**Employee:** " + emp.fullName + "\\n" +
    "**Bonus Month:** " + d.bonusMonth.label + "\\n" +
    "**Account:** " + d.account + "\\n" +
    "**Ticket #:** " + d.ticketId + "\\n\\n" +
    "**Reason for Escalation:** " + summary;

  return [{
    json: {
      ...d,
      higherUpMessage: message
    }
  }];
```

**Description:** Prepares an escalation message for PX-IM (higher-ups). Summarizes the reason for escalation based on the verdict. Passes data to the next node for Teams delivery.

---

### Node 30: Send message and wait for response

| Field | Value |
|-------|-------|
| **id** | `bc3c4850-0662-4453-a232-3d299e11cbf7` |
| **name** | Send message and wait for response |
| **type** | `n8n-nodes-base.microsoftTeams` |
| **typeVersion** | 2 |
| **position** | [4160, 240] |
| **webhookId** | `f8bb07bb-3c8c-464b-8317-6e5cf97f6c7e` |

**Credentials:** `microsoftTeamsOAuth2Api` -- id: `Hy7nmGVPixldBadk`, name: `Service@smartsourcing`

**Parameters:**
- `resource`: `"chatMessage"`
- `operation`: `"sendAndWait"`
- `chatId`:
  - value: `"19:13edc4883f2f468c81325b61573fb9ce@thread.v2"`
  - mode: `"list"`
  - cachedResultName: `"IM Analytics - n8n Workflow Error (group)"`
  - cachedResultUrl: `"https://teams.microsoft.com/l/chat/19%3A13edc4883f2f468c81325b61573fb9ce%40thread.v2/0?tenantId=f425cdcb-8a88-4e36-a0b7-1a8a9f91e2ee"`
- `approvalOptions`:
  - approvalType: `"double"` (Approve/Disapprove)
  - disapproveLabel: `"Deny"` (with cross mark)
- `message` (template):
```
  Attendance Bonus Dispute — Higher-Up Decision Required

  Employee: {{ $('Verdict Engine').first().json.employee.fullName }}

  Bonus Month: {{ $('Verdict Engine').first().json.bonusMonth.label }}

  Account: {{ $('Verdict Engine').first().json.account }}

  Ticket #: {{ $('Verdict Engine').first().json.ticketId }}

  Escalated by Team GC for higher-up decision. AI found no disqualification reason but cannot auto-approve.
```

**Description:** Sends a second-level escalation message to the same Teams group with Approve/Deny buttons (double approval type). This is for cases escalated beyond the initial team (PX-IM higher-up review). Currently a terminal node.

---

## Connections

The complete raw JSON for the connections object:

```json
{
  "Ticket received via webhook": {
    "main": [
      [
        {
          "node": "If Subject Contains {Category} Bonus",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "If Subject Contains {Category} Bonus": {
    "main": [
      [
        {
          "node": "Get all required parameters",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "Get all required parameters": {
    "main": [
      [
        {
          "node": "Lookup Employee SQL",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "Merge Data": {
    "main": [
      [
        {
          "node": "Filing Mode",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "Lookup Employee SQL": {
    "main": [
      [
        {
          "node": "Code",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "CHECK1 Detail": {
    "main": [
      [
        {
          "node": "Merge CHECK1",
          "type": "main",
          "index": 1
        }
      ]
    ]
  },
  "CHECK1 Overview": {
    "main": [
      [
        {
          "node": "Merge CHECK1",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "Merge CHECK1": {
    "main": [
      [
        {
          "node": "Evaluate CHECK1",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "CHECK2 COA": {
    "main": [
      [
        {
          "node": "Evaluate CHECK2",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "Code": {
    "main": [
      [
        {
          "node": "Merge Data",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "CHECK3 SA": {
    "main": [
      [
        {
          "node": "Evaluate CHECK3",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "Filing Mode": {
    "main": [
      [
        {
          "node": "CHECK1 Detail",
          "type": "main",
          "index": 0
        },
        {
          "node": "CHECK2 COA",
          "type": "main",
          "index": 0
        },
        {
          "node": "CHECK3 SA",
          "type": "main",
          "index": 0
        },
        {
          "node": "CHECK1 Overview",
          "type": "main",
          "index": 0
        }
      ],
      []
    ]
  },
  "Evaluate CHECK3": {
    "main": [
      [
        {
          "node": "Needs History Check?",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "Needs History Check?": {
    "main": [
      [
        {
          "node": "CHECK3 History",
          "type": "main",
          "index": 0
        }
      ],
      [
        {
          "node": "CHECK3 No History",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "CHECK3 History": {
    "main": [
      [
        {
          "node": "Final CHECK3",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "Evaluate CHECK1": {
    "main": [
      [
        {
          "node": "Merge all CHECKS",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "Evaluate CHECK2": {
    "main": [
      [
        {
          "node": "Merge all CHECKS",
          "type": "main",
          "index": 1
        }
      ]
    ]
  },
  "Final CHECK3": {
    "main": [
      [
        {
          "node": "Merge all CHECKS",
          "type": "main",
          "index": 2
        }
      ]
    ]
  },
  "CHECK3 No History": {
    "main": [
      [
        {
          "node": "Merge all CHECKS",
          "type": "main",
          "index": 3
        }
      ]
    ]
  },
  "Merge all CHECKS": {
    "main": [
      [
        {
          "node": "Verdict Engine",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "Verdict Engine": {
    "main": [
      [
        {
          "node": "Action Router",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "Action Router": {
    "main": [
      [
        {
          "node": "Auto Deny Reply",
          "type": "main",
          "index": 0
        }
      ],
      [
        {
          "node": "Build Teams Card",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "Build Teams Card": {
    "main": [
      [
        {
          "node": "Send to Team GC",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "Send to Team GC": {
    "main": [
      [
        {
          "node": "Route Team Decision",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "Route Team Decision": {
    "main": [
      [
        {
          "node": "Approve Reply",
          "type": "main",
          "index": 0
        }
      ],
      [
        {
          "node": "Team Deny Reply",
          "type": "main",
          "index": 0
        }
      ],
      [
        {
          "node": "PX-IM Escalate",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "Auto Deny Reply": {
    "main": [
      []
    ]
  },
  "PX-IM Escalate": {
    "main": [
      [
        {
          "node": "Send message and wait for response",
          "type": "main",
          "index": 0
        }
      ]
    ]
  }
}
```

### Connection Flow Summary

```
Ticket received via webhook
  -> If Subject Contains {Category} Bonus
      -> [true] Get all required parameters
          -> Lookup Employee SQL
              -> Code (employee found check)
                  -> [true] Merge Data
                      -> Filing Mode
                          -> [self / output 0] CHECK1 Detail, CHECK2 COA, CHECK3 SA, CHECK1 Overview (parallel)
                          -> [manager / output 1] (empty -- not connected)

CHECK1 Overview -> Merge CHECK1 (input 0)
CHECK1 Detail   -> Merge CHECK1 (input 1)
Merge CHECK1    -> Evaluate CHECK1 -> Merge all CHECKS (input 0)

CHECK2 COA      -> Evaluate CHECK2 -> Merge all CHECKS (input 1)

CHECK3 SA       -> Evaluate CHECK3 -> Needs History Check?
  -> [true]  CHECK3 History -> Final CHECK3     -> Merge all CHECKS (input 2)
  -> [false] CHECK3 No History                  -> Merge all CHECKS (input 3)

Merge all CHECKS -> Verdict Engine -> Action Router
  -> [auto_deny / output 0]  Auto Deny Reply -> (terminal, empty connection)
  -> [escalate / output 1]   Build Teams Card -> Send to Team GC -> Route Team Decision
      -> [approve / output 0]         Approve Reply (terminal)
      -> [deny / output 1]            Team Deny Reply (terminal)
      -> [escalate_higher / output 2] PX-IM Escalate -> Send message and wait for response (terminal)
```

---

## Credentials

```json
{
  "microsoftSql": {
    "id": "GbaRqtLuVtlqs0TT",
    "name": "Microsoft SQL account"
  },
  "microsoftTeamsOAuth2Api": {
    "id": "Hy7nmGVPixldBadk",
    "name": "Service@smartsourcing"
  }
}
```

**Usage by node:**

| Credential | Type | Nodes Using It |
|------------|------|----------------|
| Microsoft SQL account (`GbaRqtLuVtlqs0TT`) | `microsoftSql` | Lookup Employee SQL, CHECK1 Overview, CHECK1 Detail, CHECK2 COA, CHECK3 SA, CHECK3 History |
| Service@smartsourcing (`Hy7nmGVPixldBadk`) | `microsoftTeamsOAuth2Api` | Send to Team GC, Send message and wait for response |

---

## Settings

```json
{
  "executionOrder": "v1",
  "binaryMode": "separate",
  "availableInMCP": false
}
```

---

## Metadata

| Field | Value |
|-------|-------|
| **staticData** | `null` |
| **meta.templateCredsSetupCompleted** | `true` |
| **pinData** | `{}` |
| **versionId** | `79f3aa1b-6243-4ecc-8da3-5c65ee5f8c13` |
| **activeVersionId** | `1fe25b9f-ea3a-4b49-90c5-b0fc5a7bdc98` |
| **versionCounter** | 365 |
| **triggerCount** | 1 |
| **tags** | `[]` |
| **shared.role** | `workflow:owner` |
| **shared.workflowId** | `5Ol9zvnHASarHAGx` |
| **shared.projectId** | `nbfxOSGxgb3fUlj1` |
| **shared.project.name** | `Biena Villagonzalo <biena.villagonzalo@smartsourcing.co>` |
| **shared.project.type** | `personal` |

---

## Node ID Quick Reference

| # | Node Name | Node ID | Type |
|---|-----------|---------|------|
| 1 | Ticket received via webhook | `25d2c38c-0814-4d11-85d4-5c958d8bfa30` | webhook |
| 2 | If Subject Contains {Category} Bonus | `fc06ce2e-b8af-4848-a4bb-a346f1085bfa` | if |
| 3 | Get all required parameters | `91a117ca-cd45-47a9-adbd-55b2b9c17e72` | code |
| 4 | Lookup Employee SQL | `a879779d-8a01-4243-83a4-4ca3155620d0` | microsoftSql |
| 5 | Code (Employee Found Check) | `2d2b8bf9-fa49-4237-b156-a23987f5e507` | if |
| 6 | Merge Data | `f79ed667-ff20-452d-8195-9c39607591ca` | code |
| 7 | Filing Mode | `ac3ca058-faa2-4f33-9423-29353d9ff615` | switch |
| 8 | CHECK1 Overview | `a77250e1-4a2f-4d6a-81ae-991b9690cdb4` | microsoftSql |
| 9 | CHECK1 Detail | `85b7b1e7-6197-4585-a3a3-66f88cf082a8` | microsoftSql |
| 10 | Merge CHECK1 | `c6a53003-6826-423b-9d1e-e95ccff0a0ea` | merge |
| 11 | Evaluate CHECK1 | `1d346488-1f98-43ed-a513-5f539ebb8ce2` | code |
| 12 | CHECK2 COA | `f8cdf4fc-df44-40ad-87fa-2c393bc374a4` | microsoftSql |
| 13 | Evaluate CHECK2 | `f3ac3777-0665-4fe0-8512-875431527435` | code |
| 14 | CHECK3 SA | `50a95c05-7e83-4746-9170-dd4e20f096bc` | microsoftSql |
| 15 | Evaluate CHECK3 | `ff547d08-98e3-4681-942a-b7a60f1554c5` | code |
| 16 | Needs History Check? | `e6d6925e-17af-4b98-bf05-b276b9d77b4e` | if |
| 17 | CHECK3 History | `8b4229f5-e1b7-425f-9559-9ed32042b737` | microsoftSql |
| 18 | Final CHECK3 | `6d3d2cae-1ef5-4915-8ced-08fc7c07f51d` | code |
| 19 | CHECK3 No History | `6e09f46c-6fc4-4fee-8102-f282b7abef82` | code |
| 20 | Merge all CHECKS | `f28eb398-2ecd-413e-aad8-098aa0401885` | merge |
| 21 | Verdict Engine | `4521c6bd-e9c4-47b0-b0b2-9e7d257c91fd` | code |
| 22 | Action Router | `5dec86a2-1957-408b-8a33-efbd5c3ba913` | switch |
| 23 | Auto Deny Reply | `316cec66-2576-447c-8c80-9508dbd7f366` | code |
| 24 | Build Teams Card | `1c2c94c1-e869-4347-a3bc-72ef48ec5530` | code |
| 25 | Send to Team GC | `a8783856-4414-4aea-9b4e-b169608d2286` | microsoftTeams |
| 26 | Route Team Decision | `30ab8290-83d0-4543-8407-1093b7df15f1` | switch |
| 27 | Approve Reply | `f7f10d39-bfe1-4cb9-bc54-77ab1d89b372` | code |
| 28 | Team Deny Reply | `124aa930-81ed-4c44-872a-21966ff222da` | code |
| 29 | PX-IM Escalate | `aed3eaf1-6eeb-452a-9132-a75f13f8eafc` | code |
| 30 | Send message and wait for response | `bc3c4850-0662-4453-a232-3d299e11cbf7` | microsoftTeams |
