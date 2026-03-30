# Manager Mode + Multi-Category Implementation

> Copy-paste each code block into the corresponding n8n node.
> SQL blocks go into Microsoft SQL nodes. Code blocks go into Code nodes.

---

## NODE 3: Get all required parameters (REPLACE existing code)

**Type:** Code node

```javascript
// ===== EXTRACT WEBHOOK DATA =====
const data = $input.first().json.body;
const requesterEmail = data.requester_email;
const ticketId = data.ticket_id;
const issueType = data.issue_type;       // dropdown: "Attendance Bonus"
const issueDetails = data.issue_details || '';
const account = data.account;
const teamLead = data.team_lead;

// ===== PARSE BONUS MONTH =====
const rawMonth = data.bonus_month_detail || data.bonus_month;
const monthName = rawMonth.replace(/[\d\s]+$/g, '').trim(); // "Feb 16" → "Feb"

const monthMap = {
  'january': 0, 'february': 1, 'march': 2, 'april': 3,
  'may': 4, 'june': 5, 'july': 6, 'august': 7,
  'september': 8, 'october': 9, 'november': 10, 'december': 11,
  'jan': 0, 'feb': 1, 'mar': 2, 'apr': 3,
  'jun': 5, 'jul': 6, 'aug': 7, 'sep': 8,
  'oct': 9, 'nov': 10, 'dec': 11
};

const monthIndex = monthMap[monthName.toLowerCase()];
if (monthIndex === undefined) {
  throw new Error(`Could not parse bonus month from: "${rawMonth}"`);
}

const now = new Date();
let year = now.getFullYear();
if (monthIndex > now.getMonth()) {
  year = year - 1;
}

const bonusMonthStart = new Date(year, monthIndex, 1);
const bonusMonthEnd = new Date(year, monthIndex + 1, 0);
const fmt = (d) => d.toISOString().slice(0, 10);

// Cutoff: 3rd business day of the month AFTER the bonus month
let businessDays = 0;
let cutoffDate = new Date(year, monthIndex + 1, 0); // last day of bonus month
while (businessDays < 3) {
  cutoffDate.setDate(cutoffDate.getDate() + 1);
  const day = cutoffDate.getDay();
  if (day !== 0 && day !== 6) businessDays++;
}

// ===== REQUESTER EMAIL PARTS (for self-mode lookup) =====
const emailPrefix = requesterEmail.split('@')[0];
const emailParts = emailPrefix.split('.');
const emailLastName = emailParts.length > 1 ? emailParts[emailParts.length - 1] : emailParts[0];
const emailFirstPart = emailParts[0] || '';

// ===== PARSE ISSUE DETAILS FOR MANAGER-FILED ENTRIES =====
const KNOWN_CATEGORIES = [
  { key: 'attendance bonus', label: 'Attendance Bonus' },
  { key: 'performance bonus', label: 'Performance Bonus' },
  { key: 'engagement bonus', label: 'Engagement Bonus' }
];

// Filler words to strip when extracting names
const FILLER = new Set([
  'for', 'please', 'check', 'the', 'and', 'of', 'in', 'on', 'is', 'was',
  'has', 'had', 'have', 'been', 'not', 'no', 'but', 'or', 'if', 'then',
  'so', 'because', 'he', 'she', 'they', 'his', 'her', 'their', 'my',
  'our', 'your', 'a', 'an', 'it', 'its', 'also', 'too', 'as', 'well',
  'with', 'from', 'to', 'by', 'about', 'regarding', 're', 'per',
  'filed', 'filing', 'dispute', 'request', 'issue', 'concern',
  'bonus', 'month', 'need', 'needs', 'wanted', 'want', 'be',
  'late', 'absent', 'tardy', 'undertime', 'overtime', 'leave',
  '-', '–', '—', ':', ';', 'hi', 'hello', 'good', 'day', 'morning',
  'afternoon', 'evening', 'sir', 'maam', 'ma\'am', 'thank', 'thanks',
  'kindly', 'pls', 'po'
]);

/**
 * Extract name candidates from a text segment.
 * Strategy: remove filler words, split by commas/and, keep what's left.
 */
function extractNames(text) {
  return text
    .replace(/\band\b/gi, ',')       // "Ei Husain and John Smith" → split
    .replace(/[\n\r]+/g, ',')        // newlines → split
    .split(',')
    .map(segment => {
      return segment
        .trim()
        .split(/\s+/)
        .filter(w => !FILLER.has(w.toLowerCase()) && w.length > 0)
        .join(' ')
        .trim();
    })
    .filter(name => {
      // Must have at least 2 characters and look like a name (not pure numbers/symbols)
      return name.length >= 2 && /[a-zA-Z]/.test(name);
    });
}

/**
 * Split a name string into first/last parts for SQL lookup.
 */
function parseNameParts(nameText) {
  const parts = nameText.trim().split(/\s+/);
  if (parts.length === 0) return null;
  if (parts.length === 1) {
    // Single word — use it for both first and last (SQL will LIKE match)
    return { parsedFirst: parts[0], parsedLast: parts[0] };
  }
  // First word = first name, last word = last name
  // Middle words preserved in nameText for display
  return {
    parsedFirst: parts[0],
    parsedLast: parts[parts.length - 1]
  };
}

// --- Parse issue_details ---
const lowerDetails = issueDetails.toLowerCase();
const parsedEntries = [];

// Find all category mentions and their positions
const categoryHits = [];
for (const cat of KNOWN_CATEGORIES) {
  let pos = lowerDetails.indexOf(cat.key);
  while (pos !== -1) {
    categoryHits.push({
      label: cat.label,
      position: pos,
      length: cat.key.length
    });
    pos = lowerDetails.indexOf(cat.key, pos + cat.key.length);
  }
}
categoryHits.sort((a, b) => a.position - b.position);

if (categoryHits.length > 0) {
  // Split issue_details by category markers, extract names from each segment
  for (let i = 0; i < categoryHits.length; i++) {
    const hit = categoryHits[i];
    const segStart = hit.position + hit.length;
    const segEnd = (i + 1 < categoryHits.length)
      ? categoryHits[i + 1].position
      : issueDetails.length;

    const segment = issueDetails.slice(segStart, segEnd);
    const names = extractNames(segment);

    for (const name of names) {
      const parts = parseNameParts(name);
      if (parts) {
        parsedEntries.push({ category: hit.label, nameText: name, ...parts });
      }
    }
  }

  // Check text BEFORE first category mention (might have names)
  if (categoryHits[0].position > 0) {
    const before = issueDetails.slice(0, categoryHits[0].position);
    const names = extractNames(before);
    for (const name of names) {
      const parts = parseNameParts(name);
      if (parts) {
        // Names before any category default to the first mentioned category
        parsedEntries.unshift({ category: categoryHits[0].label, nameText: name, ...parts });
      }
    }
  }
} else if (issueDetails.trim().length > 0) {
  // No category keywords found — extract names, default ALL to dropdown issue_type
  const names = extractNames(issueDetails);
  for (const name of names) {
    const parts = parseNameParts(name);
    if (parts) {
      parsedEntries.push({ category: issueType, nameText: name, ...parts });
    }
  }
}

// ===== DETERMINE MODE =====
const detectedMode = parsedEntries.length > 0 ? 'manager' : 'self';

// ===== OUTPUT =====
return [{
  json: {
    ticketId,
    requesterEmail,
    issueType,
    issueDetails,
    account,
    teamLead,
    emailLastName,
    emailFirstPart,
    detectedMode,
    parsedEntries,
    bonusMonth: {
      start: fmt(bonusMonthStart),
      end: fmt(bonusMonthEnd),
      label: bonusMonthStart.toLocaleDateString('en-US', { month: 'long', year: 'numeric' })
    },
    cutoffDate: fmt(cutoffDate)
  }
}];
```

---

## NODE 5: Merge Data (REPLACE existing code)

**Type:** Code node
**Inputs:** Node 3 output + Node 4 (Lookup Employee SQL) output

```javascript
// Combine employee lookup result with parsed context
const context = $('Get all required parameters').first().json;
const employeeRows = $('Lookup Employee SQL').all().map(i => i.json);

const mode = context.detectedMode; // "self" or "manager"

if (mode === 'self') {
  // --- SELF MODE: requester IS the employee ---
  const employee = employeeRows[0] || null;

  if (!employee) {
    // Employee not found by email — escalate
    return [{
      json: {
        ...context,
        mode: 'self',
        employee: null,
        employeeName: `${context.emailFirstPart} ${context.emailLastName}`,
        employeeId: null,
        error: 'EMPLOYEE_NOT_FOUND',
        errorDetail: `No employee found matching email: ${context.requesterEmail}`
      }
    }];
  }

  return [{
    json: {
      ...context,
      mode: 'self',
      employee,
      employeeName: `${employee.first_name || ''} ${employee.last_name || ''}`.trim(),
      employeeId: employee.employee_id || employee.id,
      firstName: employee.first_name,
      lastName: employee.last_name,
      category: context.issueType // single category from dropdown
    }
  }];

} else {
  // --- MANAGER MODE: requester filed on behalf of others ---
  const requesterEmployee = employeeRows[0] || null;

  return [{
    json: {
      ...context,
      mode: 'manager',
      filedBy: {
        email: context.requesterEmail,
        name: requesterEmployee
          ? `${requesterEmployee.first_name || ''} ${requesterEmployee.last_name || ''}`.trim()
          : `${context.emailFirstPart} ${context.emailLastName}`,
        employeeId: requesterEmployee?.employee_id || requesterEmployee?.id || null
      },
      // parsedEntries already in context — downstream nodes will split and process
    }
  }];
}
```

---

## NODE 6: Filing Mode Switch (UPDATE configuration)

**Type:** Switch node
**Route by:** `{{ $json.mode }}`
**Rules:**
| Rule | Value | Output |
|------|-------|--------|
| 1 | `self` | → existing self path (Node 7+) |
| 2 | `manager` | → NEW Node M1 (Split Manager Entries) |

---

## NODE M1: Split Manager Entries

**Type:** Code node
**Input:** Node 6 "manager" output

```javascript
// Split parsedEntries into individual n8n items — one per employee+category
const data = $input.first().json;
const entries = data.parsedEntries || [];

if (entries.length === 0) {
  // No entries parsed — escalate whole ticket
  return [{
    json: {
      ...data,
      category: data.issueType,
      nameText: '(none parsed)',
      parsedFirst: '',
      parsedLast: '',
      lookupStatus: 'NO_ENTRIES_PARSED'
    }
  }];
}

// One item per entry, carrying all common fields
return entries.map((entry, index) => ({
  json: {
    ticketId: data.ticketId,
    requesterEmail: data.requesterEmail,
    issueType: data.issueType,
    issueDetails: data.issueDetails,
    account: data.account,
    teamLead: data.teamLead,
    bonusMonth: data.bonusMonth,
    cutoffDate: data.cutoffDate,
    filedBy: data.filedBy,
    mode: 'manager',
    entryIndex: index,
    totalEntries: entries.length,
    // Per-employee fields
    category: entry.category,
    nameText: entry.nameText,
    parsedFirst: entry.parsedFirst,
    parsedLast: entry.parsedLast
  }
}));
```

---

## NODE M2: Lookup Employee by Name

**Type:** Microsoft SQL node
**Execute Once:** OFF (processes each item)

```sql
SELECT TOP 5
  *,
  '{{ $json.nameText }}' AS searched_name,
  '{{ $json.category }}' AS searched_category,
  {{ $json.entryIndex }} AS entry_index
FROM people.employee
WHERE last_name LIKE '%{{ $json.parsedLast }}%'
  AND first_name LIKE '%{{ $json.parsedFirst }}%'
```

> **Note:** The `searched_name`, `searched_category`, and `entry_index` columns are passthrough markers so we can match results back to entries in the next node.

---

## NODE M3: Match & Validate Employees

**Type:** Code node
**Input:** Node M2 output (all SQL results)

```javascript
// SQL results come as flat items — group by entry_index and pick best match
const results = $input.all().map(i => i.json);

// Also grab the original split entries for entries that had NO SQL results
const splitItems = $('Split Manager Entries').all().map(i => i.json);

// Group SQL results by entry_index
const grouped = {};
for (const row of results) {
  const idx = row.entry_index;
  if (!grouped[idx]) grouped[idx] = [];
  grouped[idx].push(row);
}

// Build one output item per original entry
const output = [];

for (const entry of splitItems) {
  const idx = entry.entryIndex;
  const matches = grouped[idx] || [];

  if (matches.length === 0) {
    // No employee found
    output.push({
      json: {
        ...entry,
        employee: null,
        employeeName: entry.nameText,
        employeeId: null,
        firstName: entry.parsedFirst,
        lastName: entry.parsedLast,
        lookupStatus: 'NOT_FOUND'
      }
    });
  } else if (matches.length === 1) {
    // Exact match
    const emp = matches[0];
    output.push({
      json: {
        ...entry,
        employee: emp,
        employeeName: `${emp.first_name || ''} ${emp.last_name || ''}`.trim(),
        employeeId: emp.employee_id || emp.id,
        firstName: emp.first_name,
        lastName: emp.last_name,
        lookupStatus: 'FOUND'
      }
    });
  } else {
    // Multiple matches — ALWAYS escalate for human to pick the right employee
    output.push({
      json: {
        ...entry,
        employee: null,
        employeeName: entry.nameText,
        employeeId: null,
        firstName: entry.parsedFirst,
        lastName: entry.parsedLast,
        lookupStatus: 'AMBIGUOUS',
        matchCount: matches.length,
        ambiguousMatches: matches.map(m => ({
          id: m.employee_id || m.id,
          name: `${m.first_name || ''} ${m.last_name || ''}`.trim(),
          email: m.email || ''
        }))
      }
    });
  }
}

return output;
```

---

## NODE M4: Manager Category Router

**Type:** Switch node
**Route by:** Expression

**Rules:**

| Rule | Condition | Output |
|------|-----------|--------|
| 1 | `{{ $json.lookupStatus }}` equals `NOT_FOUND` | → Node M18 (Escalate — employee not found) |
| 2 | `{{ $json.lookupStatus }}` equals `AMBIGUOUS` | → Node M18 (Escalate — multiple matches, human picks) |
| 3 | `{{ $json.category }}` equals `Attendance Bonus` | → Attendance Check Pipeline (M5+) |
| 4 | Default | → Node M18 (Non-Attendance Escalation) |

> Rules 1-2 catch unresolvable/ambiguous names before any checks run.
> Rule 4 catches Performance Bonus, Engagement Bonus, and anything else.

---

## ATTENDANCE CHECK PIPELINE (Manager Path)

These are **copies** of existing Nodes 7-20, wired into the manager path. Same SQL queries, same evaluation logic — just sourced from the manager-split items instead of the self-mode employee.

### NODE M5: CHECK1 Overview (Microsoft SQL)

Same query as Node 7, but using manager item fields:

```sql
SELECT *
FROM analytics_powerbi.V_fact_attendance
WHERE employee_id = {{ $json.employeeId }}
  AND date >= '{{ $json.bonusMonth.start }}'
  AND date <= '{{ $json.bonusMonth.end }}'
```

### NODE M6: CHECK1 Detail (Microsoft SQL)

Same query as Node 8:

```sql
SELECT *
FROM staging_sprout.V_attendance_with_schedule
WHERE [First Name] LIKE '%{{ $json.firstName }}%'
  AND [Last Name] LIKE '%{{ $json.lastName }}%'
  AND [Date] >= '{{ $json.bonusMonth.start }}'
  AND [Date] <= '{{ $json.bonusMonth.end }}'
  AND [Is Rest Day] = 0
ORDER BY [Date]
```

### NODE M7: Merge CHECK1 (Merge node)

**Type:** Merge
**Mode:** Combine — Merge By Position
**Inputs:** M5 + M6

### NODE M8: Evaluate CHECK1 (Code)

**Same code as existing Node 10 (Evaluate CHECK1).** Copy it exactly.

### NODE M9: CHECK2 COA (Microsoft SQL)

Same query as Node 11:

```sql
SELECT *
FROM staging_integration.sprout_employee_COA_HTML
WHERE employee_id = {{ $json.employeeId }}
```

### NODE M10: Evaluate CHECK2 (Code)

**Same code as existing Node 12 (Evaluate CHECK2).** Copy it exactly.

### NODE M11: CHECK3 SA (Microsoft SQL)

Same query as Node 13:

```sql
SELECT *
FROM staging_sprout.schedule_adjustments
WHERE firstName LIKE '%{{ $json.firstName }}%'
  AND lastName LIKE '%{{ $json.lastName }}%'
  AND elt_current_ind = 'Y'
```

### NODE M12: Evaluate CHECK3 (Code)

**Same code as existing Node 14 (Evaluate CHECK3).** Copy it exactly.

### NODE M13: Needs History Check? (IF)

**Same condition as existing Node 15.** Copy it exactly.

### NODE M14: CHECK3 History (Microsoft SQL)

**Same query as existing Node 16.** Copy it exactly.

### NODE M15: Final CHECK3 / No History (Code)

**Same code as existing Nodes 17/18.** Copy them exactly.

### NODE M16: Merge All Manager Checks (Merge)

**Type:** Merge
**Mode:** Combine — Merge By Position
**Inputs:** M8 (CHECK1) + M10 (CHECK2) + M15 (CHECK3)

### NODE M17: Manager Verdict Engine (Code)

**Same code as existing Node 20 (Verdict Engine).** Copy it exactly.
The verdict engine doesn't need to change — it still produces `{verdict, action, reasons}` per item.

---

## NODE M18: Non-Attendance / Not-Found Escalation

**Type:** Code node
**Input:** From M4 (non-attendance route OR not-found route)

```javascript
// Build escalation info for entries that can't be auto-processed
const items = $input.all();

return items.map(item => {
  const d = item.json;
  let reason, escalationType;

  if (d.lookupStatus === 'NOT_FOUND') {
    reason = `Employee "${d.nameText}" not found in system. Manual lookup required.`;
    escalationType = 'EMPLOYEE_NOT_FOUND';
  } else if (d.lookupStatus === 'AMBIGUOUS') {
    const matchList = (d.ambiguousMatches || [])
      .map(m => `${m.name} (${m.email || 'ID: ' + m.id})`)
      .join(', ');
    reason = `Multiple employees match "${d.nameText}" (${d.matchCount} results): ${matchList}. Please select the correct employee.`;
    escalationType = 'AMBIGUOUS_MATCH';
  } else if (d.lookupStatus === 'NO_ENTRIES_PARSED') {
    reason = `Could not parse any employee names from issue details. Manual review required.`;
    escalationType = 'PARSE_FAILED';
  } else {
    reason = `${d.category} disputes are not yet automated. Manual review required.`;
    escalationType = 'CATEGORY_NOT_AUTOMATED';
  }

  return {
    json: {
      ...d,
      verdict: 'ESCALATED',
      action: 'ESCALATE_TEAM',
      escalationType,
      reason,
      check1: { status: 'SKIPPED', detail: reason },
      check2: { status: 'SKIPPED', detail: reason },
      check3: { status: 'SKIPPED', detail: reason }
    }
  };
});
```

---

## NODE M19: Aggregate Manager Results

**Type:** Code node
**Inputs:** Must receive items from ALL paths — M17 (attendance verdicts) + M18 (escalations)

> **n8n wiring:** Use a Merge node (Mode: Append) to combine outputs from M17 and M18 before this node.

```javascript
// Collect all per-employee results and build consolidated report
const items = $input.all().map(i => i.json);

// Sort by entry index to maintain original order
items.sort((a, b) => (a.entryIndex || 0) - (b.entryIndex || 0));

// Build per-employee summary lines
const lines = items.map(r => {
  const name = r.employeeName || r.nameText;
  const cat = r.category;
  const verdict = r.verdict; // DISQUALIFIED, QUALIFIES, ESCALATED

  let statusLabel, detail;
  if (verdict === 'DISQUALIFIED') {
    const reasons = r.reasons || [];
    statusLabel = 'DISQUALIFIED';
    detail = reasons.join(', ') || r.reason || '';
  } else if (verdict === 'QUALIFIES') {
    statusLabel = 'QUALIFIES';
    detail = 'All checks passed';
  } else {
    statusLabel = 'NEEDS REVIEW';
    detail = r.reason || r.escalationType || '';
  }

  return `• ${name} — ${cat} — ${statusLabel}${detail ? ' (' + detail + ')' : ''}`;
});

// Determine overall action
const allDenied = items.length > 0 && items.every(r => r.verdict === 'DISQUALIFIED');
const hasQualified = items.some(r => r.verdict === 'QUALIFIES');
const hasEscalated = items.some(r => r.verdict === 'ESCALATED');

// If ALL are auto-deny, we could auto-deny the whole ticket.
// If ANY need human decision, escalate everything.
const overallAction = allDenied ? 'AUTO_DENY' : 'ESCALATE_TEAM';

const common = items[0] || {};

const report = [
  `MANAGER-FILED DISPUTE — ${items.length} Associate(s)`,
  `Filed by: ${common.filedBy?.name || common.requesterEmail}`,
  `Account: ${common.account}`,
  `Ticket #: ${common.ticketId}`,
  `Bonus Month: ${common.bonusMonth?.label}`,
  ``,
  `--- RESULTS ---`,
  ...lines,
  ``,
  `Overall: ${allDenied ? 'ALL DISQUALIFIED' : 'HUMAN DECISION REQUIRED'}`,
  allDenied ? '' : `(${hasQualified ? 'Some qualify, ' : ''}${hasEscalated ? 'some need manual review' : ''})`.replace(', )', ')')
].filter(l => l !== undefined).join('\n');

return [{
  json: {
    report,
    overallAction,
    items,
    ticketId: common.ticketId,
    requesterEmail: common.requesterEmail,
    filedBy: common.filedBy,
    account: common.account,
    teamLead: common.teamLead,
    bonusMonth: common.bonusMonth,
    totalEmployees: items.length,
    deniedCount: items.filter(r => r.verdict === 'DISQUALIFIED').length,
    qualifiedCount: items.filter(r => r.verdict === 'QUALIFIES').length,
    escalatedCount: items.filter(r => r.verdict === 'ESCALATED').length
  }
}];
```

---

## NODE M20: Manager Action Router

**Type:** Switch node
**Route by:** `{{ $json.overallAction }}`

| Rule | Value | Output |
|------|-------|--------|
| 1 | `AUTO_DENY` | → Build Manager Deny Reply (M21) |
| 2 | `ESCALATE_TEAM` | → Send Manager Report to Team GC (M22) |

---

## NODE M21: Build Manager Deny Reply

**Type:** Code node

```javascript
// All employees disqualified — build consolidated denial reply
const data = $input.first().json;

const employeeLines = data.items.map(r => {
  const name = r.employeeName || r.nameText;
  const reasons = r.reasons || [r.reason] || ['Policy violation'];
  return `- ${name} (${r.category}): ${reasons.join('; ')}`;
}).join('\n');

const reply = `Hi ${data.filedBy?.name || 'there'},

Thank you for reaching out regarding the bonus dispute for your team members.

After reviewing the records for ${data.bonusMonth?.label}, the following associates did not meet the qualification criteria:

${employeeLines}

The disqualification is based on verified system records and company policy. If you believe there is a discrepancy in the data, please feel free to reply to this ticket with specific details, and we will review further.

Best regards,
Bonus Administration`;

return [{
  json: {
    ...data,
    emailReply: reply
  }
}];
```

---

## NODE M22: Send Manager Report to Team GC

**Type:** Microsoft Teams — Send and Wait for Response
**Operation:** Send and Wait for Response
**Response Type:** Custom Form

**Message:** `{{ $json.report }}`

**Form Field:** Decision (Options)
**Options:** `Approve All`, `Deny All`, `Review Individually`, `Escalate to Higher-Ups`

> After this node, add a Switch to route by `$json.data.Decision` — same pattern as existing Node 24 (Route Team Decision), but with the extra "Review Individually" and "Approve All" options.

---

## NODE 2: IF Subject Contains Bonus (UPDATE condition)

**Change the IF condition from:**
```
{{ $json.body.issue_type }} contains "Attendance Bonus"
```

**To:**
```
{{ $json.body.issue_type }} contains "Bonus"
```

This lets Performance Bonus, Engagement Bonus, and Attendance Bonus tickets all enter the workflow. Non-attendance categories are handled by the manager category router (M4) which escalates them to Team GC.

---

## NODE M23: Route Manager Team Decision

**Type:** Switch node
**Route by:** `{{ $json.data.Decision }}`
**Input:** Node M22 (Send Manager Report to Team GC) response

| Rule | Value | Output |
|------|-------|--------|
| 1 | `Approve All` | → Node M24 (Build Manager Approve Reply) |
| 2 | `Deny All` | → Node M25 (Build Manager Team-Deny Reply) |
| 3 | `Review Individually` | → Node M26 (Split for Individual Review) |
| 4 | `Escalate to Higher-Ups` | → Node M29 (Build Higher-Ups Manager Card) |

---

## NODE M24: Build Manager Approve Reply

**Type:** Code node

```javascript
const data = $input.first().json;

const employeeLines = data.items
  .map(r => `- ${r.employeeName || r.nameText} (${r.category})`)
  .join('\n');

const reply = `Hi ${data.filedBy?.name || 'there'},

Thank you for reaching out. After reviewing the records for ${data.bonusMonth?.label}, the following associates have been approved for their bonus:

${employeeLines}

If you have further questions, feel free to reply to this ticket.

Best regards,
Bonus Administration`;

return [{
  json: {
    ...data,
    emailReply: reply
  }
}];
```

---

## NODE M25: Build Manager Team-Deny Reply

**Type:** Code node

```javascript
// Team confirmed denial for all
const data = $input.first().json;

const employeeLines = data.items.map(r => {
  const name = r.employeeName || r.nameText;
  const reasons = r.reasons || [r.reason] || ['Policy violation'];
  return `- ${name} (${r.category}): ${reasons.join('; ')}`;
}).join('\n');

const reply = `Hi ${data.filedBy?.name || 'there'},

Thank you for reaching out regarding the bonus dispute for your team members.

After a thorough review of the records for ${data.bonusMonth?.label}, the following associates did not meet the qualification criteria:

${employeeLines}

This decision has been reviewed and confirmed by our team. If you believe there is a discrepancy, please reply with specific details and we will look into it.

Best regards,
Bonus Administration`;

return [{
  json: {
    ...data,
    emailReply: reply
  }
}];
```

---

## NODE M26: Split for Individual Review

**Type:** Code node
**Input:** M23 "Review Individually" output

```javascript
// Split the aggregated results back into individual items for per-employee Teams cards
const data = $input.first().json;
const items = data.items || [];

return items.map((item, index) => ({
  json: {
    ...item,
    // Carry forward common fields for final aggregation
    ticketId: data.ticketId,
    filedBy: data.filedBy,
    account: data.account,
    teamLead: data.teamLead,
    bonusMonth: data.bonusMonth,
    totalInBatch: items.length,
    individualIndex: index,
    // Build per-employee message
    individualMessage: [
      `INDIVIDUAL REVIEW (${index + 1}/${items.length})`,
      `Employee: ${item.employeeName || item.nameText}`,
      `Category: ${item.category}`,
      `Account: ${data.account}`,
      `Bonus Month: ${data.bonusMonth?.label}`,
      `Ticket #: ${data.ticketId}`,
      `Filed by: ${data.filedBy?.name || data.requesterEmail}`,
      ``,
      `AI Verdict: ${item.verdict}`,
      item.verdict === 'DISQUALIFIED'
        ? `Reasons: ${(item.reasons || [item.reason]).join(', ')}`
        : item.verdict === 'QUALIFIES'
          ? `All checks passed. AI cannot auto-approve.`
          : `Reason: ${item.reason || 'Needs manual review'}`,
      ``,
      item.check1 ? `CHECK 1 (Attendance): ${item.check1.status}` : '',
      item.check2 ? `CHECK 2 (COA): ${item.check2.status}` : '',
      item.check3 ? `CHECK 3 (Schedule Adj): ${item.check3.status}` : ''
    ].filter(l => l !== '').join('\n')
  }
}));
```

---

## NODE M27: Send Individual Card to Team GC

**Type:** Microsoft Teams — Send and Wait for Response
**Operation:** Send and Wait for Response
**Response Type:** Custom Form
**Execute Once:** OFF (sends one card per item)

**Message:** `{{ $json.individualMessage }}`

**Form Field:** Decision (Options)
**Options:** `Approve`, `Deny`, `Escalate to Higher-Ups`

> Each employee gets their own card. The team can Approve one and Deny another independently.

---

## NODE M28: Collect Individual Decisions

**Type:** Code node
**Input:** All responses from M27

```javascript
// Collect all individual decisions and build final response
const items = $input.all().map(i => i.json);

const approved = items.filter(i => i.data?.Decision === 'Approve');
const denied = items.filter(i => i.data?.Decision === 'Deny');
const escalated = items.filter(i => i.data?.Decision === 'Escalate to Higher-Ups');

const common = items[0] || {};

// Build lines for the email
const resultLines = items.map(i => {
  const name = i.employeeName || i.nameText;
  const decision = i.data?.Decision || 'Unknown';
  return `- ${name} (${i.category}): ${decision.toUpperCase()}`;
}).join('\n');

const reply = `Hi ${common.filedBy?.name || 'there'},

Thank you for reaching out regarding the bonus dispute for your team members.

After reviewing the records for ${common.bonusMonth?.label}, here are the results:

${resultLines}

If you have further questions, feel free to reply to this ticket.

Best regards,
Bonus Administration`;

return [{
  json: {
    ticketId: common.ticketId,
    filedBy: common.filedBy,
    account: common.account,
    bonusMonth: common.bonusMonth,
    emailReply: reply,
    approvedCount: approved.length,
    deniedCount: denied.length,
    escalatedCount: escalated.length,
    escalatedItems: escalated,
    allDecisions: items.map(i => ({
      name: i.employeeName || i.nameText,
      category: i.category,
      decision: i.data?.Decision
    }))
  }
}];
```

> **Note:** If any items were "Escalate to Higher-Ups", add an IF node after M28 to check `escalatedCount > 0` and route those to the Higher-Ups GC flow.

---

## N8N WIRING SUMMARY

```
Webhook → IF (contains "Bonus") → Node 3 (Enhanced Parse) → Node 4 (Lookup by Email)
  → Node 5 (Mode Detect) → Node 6 (Switch)
     │
     ├── self → [existing Nodes 7-28, unchanged]
     │
     └── manager → M1 (Split Entries)
           → M2 (Lookup by Name SQL)
           → M3 (Validate Match)
           → M4 (Category Router)
              │
              ├── NOT_FOUND ──────→ M18 (Escalation) ─┐
              ├── AMBIGUOUS ──────→ M18 (Escalation) ─┤
              ├── Attendance ─→ M5-M17 (Checks) ─────→│ Merge (Append)
              └── Non-Attendance → M18 (Escalation) ──┘     │
                                                        M19 (Aggregate Results)
                                                             │
                                                        M20 (Action Router)
                                                        ┌────┴────────────┐
                                                   AUTO_DENY         ESCALATE_TEAM
                                                        │                 │
                                                   M21 (Deny Reply) M22 (Summary to Team GC)
                                                                         │
                                                                    M23 (Route Decision)
                                                     ┌──────┬───────────┼───────────┐
                                                Approve All │    Review Individually │
                                                     │   Deny All        │      Escalate to
                                                M24 (Reply) │      M26 (Split)  Higher-Ups
                                                        M25 (Reply)      │          │
                                                                    M27 (Individual M29 (Higher-
                                                                     Cards per      Ups Card)
                                                                     Employee)
                                                                         │
                                                                    M28 (Collect
                                                                     Decisions
                                                                     + Reply)
```

---

## TESTING CHECKLIST

1. **Self mode (no names in issue_details):** Requester = employee. Should work exactly as before.
2. **Self mode + Performance Bonus dropdown:** Now enters workflow (Node 2 changed). Since self mode has no manager entries, routes to self path. Existing self checks run (attendance checks against a Performance ticket — may need future handling).
3. **Manager + single attendance employee:** One name in issue_details with "Attendance Bonus". Should run 3 checks against that employee, NOT the requester.
4. **Manager + multiple employees, same category:** Multiple names, all Attendance. Each gets checked independently.
5. **Manager + mixed categories:** "Attendance Bonus Ei Husain Performance Bonus John Smith". Ei Husain → full checks. John Smith → escalated as "not automated".
6. **Manager + no parseable names:** Garbage text → falls through to escalation.
7. **Employee not found:** Parsed name doesn't match anyone in `people.employee` → escalated with "not found" reason.
8. **Ambiguous match:** Name matches multiple employees → escalated with list of matches for team to pick.
9. **Summary card → Approve All:** All employees approved, consolidated reply sent.
10. **Summary card → Deny All:** All employees denied, consolidated reply sent.
11. **Summary card → Review Individually:** Individual cards sent per employee. Team approves some, denies others. Consolidated reply reflects mixed decisions.
12. **Summary card → Escalate to Higher-Ups:** Forwarded to higher-ups GC for decision.
