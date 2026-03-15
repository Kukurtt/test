---
stepsCompleted: [1, 2, 3]
inputDocuments: ['Bonus Disputes.xlsx']
session_topic: 'Automating bonus dispute resolution process (Attendance, Performance, Engagement) via Freshdesk → n8n → AI skill pipeline'
session_goals: 'Reduce manual workload for 2-person team, automate ~90% repetitive disputes, build intelligent escalation flow, integrate MS Teams for human-in-the-loop decisions'
selected_approach: 'ai-recommended'
techniques_used: ['Role Playing', 'Morphological Analysis', 'Reverse Brainstorming']
ideas_generated: ['Workflow #1-18', 'Escalation #1-8', 'Response #1-3', 'Edge Case #1-9', 'Database #1-2', 'Safeguard #1-11', 'Rule #1-7', 'Design Philosophy #1', 'Design Decision #1-2', 'Gaming #1', 'Insight #1-5', 'Performance #1']
technique_execution_complete: true
context_file: ''
---

# Brainstorming Session Results

**Facilitator:** KukurtAbad
**Date:** 2026-03-15

## Session Overview

**Topic:** Automating bonus dispute resolution process (Attendance, Performance, Engagement) via Freshdesk → n8n → AI skill pipeline
**Goals:** Reduce manual workload for 2-person team, automate ~90% repetitive disputes, build intelligent escalation flow, integrate MS Teams for human-in-the-loop decisions

### Context Guidance

_No external context file provided — working from session discovery._

### Session Setup

- 3 bonus categories: Attendance, Performance, Engagement
- Freshdesk tickets with standardized subject lines trigger n8n workflows
- AI skills provide policy-based verdicts (Attendance skill exists, others in progress)
- Auto-reply and close for clear-cut cases
- Two-tier MS Teams escalation for complex/edge cases
- Re-dispute detection when associates challenge verdicts
- Only 2 team members currently handling all disputes manually
- Selected approach: AI-Recommended Techniques

## Technique Selection

**Approach:** AI-Recommended Techniques
**Analysis Context:** Automating bonus dispute resolution with focus on reducing workload, smart escalation, and MS Teams integration

**Recommended Techniques:**

- **Role Playing (Collaborative):** Examine the automation from all stakeholder perspectives — associate, AI agent, team reviewer, higher-ups, and bad actors — to uncover workflow gaps and ensure the system works for everyone.
- **Morphological Analysis (Deep):** Systematically map all parameter combinations (dispute category × trigger × confidence level × verdict × escalation path × re-dispute handling) to reveal hidden workflow configurations and edge cases.
- **Reverse Brainstorming (Creative):** Deliberately try to break the automation by asking "how could this fail?" — turning each failure scenario into a design requirement and safeguard.

**AI Rationale:** This sequence moves from stakeholder empathy → systematic mapping → resilience testing, ensuring the automation is comprehensive, covers all paths, and is robust against real-world edge cases. The progression builds layered understanding before stress-testing.

## Technique Execution Results

### Phase 1: Role Playing (Collaborative)

**Personas Explored:** Frustrated Associate, AI Skill Agent, Team Reviewer, Bad Actor Associate, Higher-Up Decision Maker

**Key Breakthroughs:**

**[Workflow #1]**: Auto-Verdict with Database Lookup — AI pulls attendance data directly from SQL Server, cross-references against policy rules, delivers verdict with full data transparency.

**[Workflow #2]**: Screenshot Bypass Logic — Screenshots irrelevant to verdict; policy + database is source of truth. Acknowledge screenshot but base decision on system records.

**[Workflow #3]**: Policy-First Verdict Engine — AI ignores ticket body for verdict purposes. Associate's email content only used for context/acknowledgment, never for decision-making.

**[Workflow #4]**: Binary Confidence Gate — If disqualification reason exists in policy/prepared list, AI handles autonomously. If NOT in list, immediately flagged for human review.

**[Workflow #5]**: Living Policy Source — Policy lives in Notion + attendance-dispute skill. New document uploaded when policies change. Separation of policy source from execution engine.

**[Workflow #6]**: Button-Driven Resolution in Teams — Escalation card includes Approve/Deny/Escalate to Higher-Ups buttons directly in Teams message. 30-second resolution vs 5-minute review-and-reply.

**[Escalation #1]**: Multi-Channel Re-Dispute Detection — Associates push back through tickets, Teams, AND managers. Automation needs to account for all three channels.

**[Escalation #2]**: No-Fault-Found Escalation — When AI finds no disqualification reason, does NOT auto-approve. Escalates to Team GC. AI can deny autonomously but cannot approve autonomously (asymmetric authority).

**[Escalation #3]**: Structured Escalation Card — Teams message contains: Bonus Month, Associate Name, Data Summary, AI Analysis/Reasoning, Freshdesk Ticket Link. Decision-ready at a glance.

**[Escalation #4]**: Two-Tier Escalation Rule — Team handles everything policy-related. Outside policy = higher-ups GC. Clean separation: team interprets policy, higher-ups create new policy.

**[Escalation #5]**: Authority-Gated Higher-Up Buttons — Only one designated person's button click is accepted in the higher-ups GC. Discussion is open, authority is singular.

**[Escalation #6]**: Executive-Level Concise Card — Higher-up card stripped down: associate name, bonus month, one-line summary, why it's outside policy, Approve/Deny buttons. 3-4 lines max.

**[Escalation #7]**: No-Note Fast Decision — Higher-up clicks Approve or Deny, no mandatory comment. Audit trail is the button click itself.

**[Escalation #8]**: Permanent Higher-Up Escalation Button — Every Team GC card always includes an "Escalate to Higher-Ups" button alongside Approve/Deny. No dead ends.

**[Response #1]**: Sympathetic-but-Concise Auto-Reply — 5 sentences max, sympathetic tone, specific data points. Feels personal despite being automated.

**[Response #2]**: Multi-Reason Transparency — When multiple disqualification reasons exist, list ALL of them. Reduces repeat disputes.

**[Response #3]**: Proactive Tardiness Education — Every tardiness-related denial explains the Sprout (10-min daily grace) vs. Policy (10-min monthly cumulative) gap upfront.

**[Database #1]**: Precedent Memory Database — Every approval outside standard policy gets logged. AI checks precedent DB before escalating. System gets smarter over time.

**[Database #2]**: Special Occasions Log — Dedicated table for system glitches, company-wide issues, policy exceptions, holiday edge cases. One-time decisions become reusable institutional knowledge.

**[Workflow #12]**: 4-Hour Reminder Ping — If no button click after 4 hours, bot sends follow-up reminder tagging the authorized person.

**[Workflow #13]**: 24-Hour Fallback to Team — If higher-up doesn't respond within 24 hours (after 4-hour reminder), case auto-falls back to Team GC.

**[Design Philosophy #1]**: Automation as Filter, Not Fortress — AI resolves easy cases fast. If an associate wants a human, they get a human. No adversarial gatekeeping.

### Phase 2: Morphological Analysis (Deep)

**Real Dispute Data Analyzed:** 97 disputes from Bonus Disputes.xlsx (Oct 2025 - Jan 2026)
- Attendance Bonus: 58 tickets (60%)
- Performance Bonus: 28 tickets (29%)
- Engagement Bonus: 11 tickets (11%)

**Edge Cases Discovered from Data:**

**[Edge Case #1]**: Undertime from Incorrect Filing — Auto-deny + explain correct filing procedure.

**[Edge Case #2]**: Holiday Undertime System Gap — System doesn't recognize holiday/working day mismatch. Escalate to team.

**[Edge Case #3]**: Sprout vs. Policy Tardiness Gap — Sprout shows 10-min daily grace, policy counts cumulative with 10-min monthly allowance. System-perception mismatch.

**[Edge Case #4]**: Undertime from Incorrect Filing — Most are user error, auto-deny with correction instructions.

**[Edge Case #5]**: Holiday Undertime System Gap — False undertime flags due to holiday. Precedent database candidate.

**[Edge Case #6]**: OB Without Clock-Out = Absent Tag — Cross-reference OB filings against attendance records. Multi-table SQL lookup.

**[Edge Case #7]**: Split Capacity / Not Offboarded — Associate in multiple system instances. Always escalate.

**[Edge Case #8]**: Overlapping Shift + Leave = False Absent — Morning clock-in, afternoon sick leave, evening clock-out. Requires time-range logic.

**[Edge Case #9]**: Duplicate Filing Without Cancellation — Auto-reply: "Cancel original via Level 2 approver, then resubmit."

**Policy Rules Captured:**

**[Rule #1]**: COA Limits — Only 1 COA per month. Exemptions: system downtime (checked against coa_exemption table), client-requested travel.

**[Rule #2]**: SA No Limit — Schedule Adjustments have no monthly limit.

**[Rule #3]**: Leave Disputes Always Escalate — No leave policy table exists yet. Always escalate.

**[Rule #4]**: Performance Timer Deletion = No Exceptions — Auto-disapprove regardless of manager approval.

**[Rule #5]**: COA Exemption Math — non_exempt_coas = total_coas - coas_matching_exemption_dates - coas_for_client_travel. If >1, auto-disapprove.

**[Rule #6]**: Holiday + Weekend Attendance — 0% on holiday/weekend is expected without SA. SA filed with is_restday = 1 → auto-reply to untick. SA filed + is_restday = 0 + approved + still 0% → data inconsistency → escalate.

**[Rule #7]**: SA Rest Day Logic — is_restday = 1 means system ignores the hours. Associate must untick to is_restday = 0 for the day to count.

**[Performance #1]**: Strict Timer/Timelog Policy — No manual time log additions (limited to 1), ANY deletion = auto-disapprove, forgot to stop timer uncorrected = auto-disapprove.

**Workflow Patterns from Data:**

**[Workflow #7]**: Real-Time System Glitch Flagging — Scan ticket body for "system downtime," "glitch," "error." Any match = no auto-verdict, escalate with Approve/Deny buttons.

**[Workflow #8]**: Batched Cutoff Dispute Approval — Gather cutoff disputes over 24-hour window, send ONE consolidated card. Approve All / Deny All / Review Individually + Exclude/Include specific disputes.

**[Workflow #9]**: Smart Batch Card with Selective Controls — Count + expandable list. Approve All, Deny All, Review Individually, Exclude specific, Include only specific.

**[Workflow #10]**: 24-Hour Batch Window — Timer starts at first cutoff dispute. 24 hours later, batch compiled and sent.

**[Workflow #11]**: Pre-Cutoff Only Filter — Only disputes where associate filed BEFORE cutoff get batched. Filed after = auto-deny.

**[Workflow #14]**: Manager-Filed Team Disputes — Parse names, run individual checks per associate, compile consolidated reply back to manager.

**[Workflow #15]**: Info-Request Auto-Detect — Classify ticket intent: DISPUTE vs INQUIRY. Inquiries get immediate data breakdown, no verdict needed.

**[Workflow #16]**: Unrelated Ticket Catch-All — No match to any known pattern = escalate with "Could not classify."

**[Workflow #17]**: SA Correction Auto-Reply — When issue is is_restday = 1, instruct to untick and resubmit. Third verdict type: not approved, not denied, but "fix this and come back."

**[Workflow #18]**: Rolling Batch Windows — First batch at 24hrs. Subsequent: 5+ disputes = new batch card. <5 = individual escalation.

**Data Insights:**

**[Insight #1]**: Team-Wide Disputes — Multiple tickets for same team issue. Detect and batch into ONE escalation. Could cut volume 20-30%.

**[Insight #2]**: "Not a Dispute" Tickets — Info requests disguised as disputes. Auto-reply with data, no verdict needed. 10-15% of volume.

**[Insight #3]**: Holiday SA is a Seasonal Bomb — December holidays cause predictable waves. Could pre-flag.

**[Insight #4]**: Performance Bonus Has Different Nature — More about system/data issues than policy. Needs data auditor architecture.

**[Insight #5]**: Emotional Escalations Exist — Some tickets emotionally charged. Tone detection could flag for human touch.

**[Decision #1]**: No Repeat Filer Tracking — Treat every dispute as independent. Report handles trends separately.

**[Design Decision #2]**: Language Doesn't Matter — Verdict based on SQL data and policy, not ticket body. Reply in English regardless.

### Phase 3: Reverse Brainstorming (Creative)

**Failure Scenarios Explored:** Wrong verdicts, gaming escalation paths, n8n workflow failures, data integrity issues, volume spikes, batching conflicts, API outages.

**Safeguards Designed:**

**[Safeguard #1]**: Data Completeness Check — Validate attendance records exist for every working day before making verdict. Missing data = escalate, not deny.

**[Safeguard #2]**: Policy Version Check — Reference policy last-updated date. Warn if stale.

**[Safeguard #3]**: Batch Anomaly Detection (Circuit Breaker) — 15+ denials in one hour → pause and alert Team GC. Requires button click to resume.

**[Safeguard #4]**: Daily Reconciliation Check — Compare Freshdesk tickets vs processing log. Surface any missed tickets.

**[Safeguard #5]**: Processing Status Tracker — SQL table with checkpoints: received → data_pulled → verdict_made → reply_sent → ticket_closed. Stuck >30 min = alert. (SELECTED AS CORE MONITORING SOLUTION)

**[Safeguard #6]**: Escalation Delivery Confirmation — Verify Teams message delivery. Retry once, fall back to email.

**[Safeguard #7]**: Button Response Timeout — Confirm downstream action completed after button click. Alert if reply failed to send.

**[Safeguard #8]**: Weekly Health Report — Friday summary: total tickets, auto-resolved, escalated, failures.

**[Safeguard #9]**: Data Integrity Cross-Validation — Cross-check for contradictions (0% attendance but biologs exist, timer >24hrs, 0% quality for certified associate). If data contradicts itself, never auto-deny — always escalate.

**[Safeguard #10]**: Service Item = Guaranteed Subject Match — Freshdesk form structure ensures consistent subject lines. No fuzzy matching needed.

**[Safeguard #11]**: AI Skill Outage Alert — API failure sends error message to Team GC. Tickets queue and retry when service recovers.

**[Gaming #1]**: Forced Escalation Attempts — Associates may keyword-stuff or reply-spam. Design philosophy: let them escalate. Automation is a filter, not a fortress.

### Creative Facilitation Narrative

_This session progressed from stakeholder empathy (Role Playing) through systematic mapping (Morphological Analysis) to resilience testing (Reverse Brainstorming). The real breakthrough came when analyzing 97 actual dispute records — transforming theoretical design into data-backed architecture. KukurtAbad demonstrated deep operational knowledge, especially around the Sprout/policy tardiness gap, COA exemption logic, and the asymmetric authority principle (AI can deny but not approve). The precedent database concept and batched approval workflow emerged as the most innovative ideas — transforming the automation from a static rule engine into a learning system._

### Session Highlights

**User Creative Strengths:** Deep operational expertise, pragmatic decision-making, clear authority boundaries
**AI Facilitation Approach:** Stakeholder-first exploration → data validation → stress testing
**Breakthrough Moments:** Asymmetric authority model, precedent database, batched cutoff approvals, SA correction as third verdict type
**Energy Flow:** Consistently engaged throughout, strongest energy during real-world edge case exploration
