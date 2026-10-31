# Employment Lifecycle in Oracle Fusion HCM – Study Guide

*Core HR: Person, Work Relationship, Assignment, Hire, Rehire, Transfer, Global Transfer, Termination, Resignation, Effective Dating*

---

## Meet Nike: our example company

Every concept in this guide is explained using one imaginary Nike setup in Oracle Fusion HCM. Nike's real configuration is not public, so treat the names below as a practice model, not facts about Nike.

| Fusion term | Nike example | What it means in simple words |
| --- | --- | --- |
| Enterprise | Nike Inc. | The whole company, top level |
| Legal Employer | Nike India Pvt Ltd, Nike USA Inc., Nike UK Ltd | The registered company that actually employs you and pays you |
| Business Unit | Retail India, Digital US | A unit that runs its own business operations |
| Department | Bengaluru Store Ops, Nike App Engineering | Where you sit in the org chart |
| Location | Nike store, Bengaluru; Beaverton campus, Oregon | The physical place you work |
| Job / Position | Store Associate, Software Engineer | What you do / the exact seat you fill |

**Our cast**

- **Rahul Verma** joins the Nike store in Bengaluru as a Store Associate. We follow him through hire, transfer, resignation and rehire.
- **Sarah Johnson** is a Software Engineer at Nike USA in Beaverton. We use her for the global transfer to Nike UK in London.

**How to use this guide:** read the concept sections once in order, then go straight to the Interview Q&A and the cheat sheet before every interview. Effective dating comes early on purpose: every other transaction depends on it.

---

## Person, Work Relationship and Assignment

Fusion stores every worker in three layers: **who you are** (Person), **who employs you** (Work Relationship), and **what job you do** (Assignment). If you get this one idea, every transaction later in this guide makes sense.

**Think of it like this for Rahul:**

- **Person** = Rahul himself. His name, date of birth, Aadhaar/PAN, address, phone. This exists once in Nike's system, for life. He gets one **person number** that never changes, even if he leaves and comes back years later.
- **Work Relationship** = Rahul's "contract" with one legal employer, Nike India Pvt Ltd. It has a start date (hire date) and, one day, an end date (termination date). The legal employer lives HERE, not on the assignment.
- **Assignment** = the actual job: Store Associate, Bengaluru store, reports to the Store Manager, full-time, grade, salary basis. This is the layer that changes most often (transfers, promotions, manager changes).

| Layer | What it holds | Nike example | How many? |
| --- | --- | --- | --- |
| Person | Name, DOB, national ID, address, contacts | Rahul Verma, person number 100245 | One per human, forever |
| Work Relationship | Legal employer, worker type, start date, termination date, primary flag | Rahul + Nike India Pvt Ltd, Employee, from 05-Jan-2026 | Many over a lifetime; many at the same time only across different legal employers |
| Employment Terms (optional) | Terms shared by assignments, like contract and pay | Nike India standard retail terms | Only in the three-tier model |
| Assignment | Job, position, department, location, manager, grade, hours, status | Store Associate, Bengaluru store | One or more per work relationship |

**Rahul keeps one person record; each stint gets its own work relationship**

```
                    Person: Rahul Verma
             (Person number 100245, same for life)
                             |
            +----------------+-----------------+
            |                                  |
  Work relationship 1 (ended)      Work relationship 2 (active, primary)
  Nike India Pvt Ltd, Employee     Nike India Pvt Ltd, Employee, Rehire
  05-Jan-2026 to 28-Feb-2027       01-Sep-2027 onwards
  (resigned)                                   |
            |                                  |
  Assignment E100245               Assignment E100245-2
  Store Associate, Bengaluru       Senior Store Associate
  at hire. Update rows:            New assignment, new number
  promotion, transfer to Mumbai
```

The person sits on top and never changes. Each time Rahul joins Nike India he gets a new work relationship with its own assignment; promotions and transfers only add rows to the assignment he already has.

### Worker types (work relationship types)

- **Employee** – on Nike's payroll, like Rahul.
- **Contingent Worker** – a contractor or agency worker, like a temporary staffer hired for the Diwali sale rush.
- **Nonworker** – tracked but not working, like a retiree who still gets benefits, or a board member.
- **Pending Worker** – someone who accepted the offer but hasn't joined yet. They get converted to an employee on the start date.

On top of these "system person types", companies create **user person types** like "Retail Employee" or "Intern" that map back to one system type.

### Employment models

This is set per enterprise or per legal employer during implementation, and it decides which layers exist:

- **Two-tier** = Work Relationship → Assignment. Simplest and most common. Can be single-assignment or multiple-assignment.
- **Three-tier** = Work Relationship → Employment Terms → Assignment. Used when several assignments share the same contract or pay terms.

### Primary flags

- If Sarah somehow has two work relationships at once (Nike USA employee + Nike UK contingent worker), exactly **one is primary**.
- Inside each work relationship, exactly **one assignment is primary**.
- Reports, payroll and approvals usually pick up the primary assignment by default, so this flag matters a lot in BI reports.

### Edge cases to remember

- **Changing legal employer is never an edit on the assignment.** Legal employer sits on the work relationship. To move Sarah from Nike USA to Nike UK, you do a Global Transfer, which ends one work relationship and starts another.
- **No two active employee work relationships in the same legal employer at the same time.** Rahul can't be hired twice into Nike India while the first one is still active.
- **One person, two legal employers at the same time is allowed.** That's concurrent employment, done with "Add Work Relationship".
- **Ending one of several assignments ≠ ending the work relationship.** If Rahul had two assignments and drops one, you use "End Assignment". If you end his last remaining assignment, the work relationship ends too.
- **Person number vs assignment number.** Person number stays for life. Each new assignment gets its own assignment number, so a rehire gets a new assignment number (often with a suffix like -2) but the same person number.
- **Some person data is not date-effective.** Date of birth sits in a non-date-effective table, so there's no history for it. Name and address are date-effective, so you can see old names after a marriage, for example.

---

## Effective dating: Update vs Correction

Fusion doesn't overwrite employee data like a whiteboard. It keeps a **diary**: every change is a new row with an **Effective Start Date** and an **Effective End Date**. The current row's end date is usually **31-Dec-4712**, which Oracle uses to mean "no end yet".

### The two choices you get when you edit

- **Update** = "something really changed from this date". Fusion end-dates the old row the day before and creates a new row. History is kept.
- **Correction** = "this was wrong from the start". Fusion fixes the existing row in place for its whole date range. No new row, no history of the wrong value on screen (audit can still capture it if enabled).

### Nike example

Rahul is hired on 05-Jan-2026 as Store Associate, grade R1.

1. On 01-Jul-2026 he is promoted to Senior Store Associate, grade R2. This is a real change, so you do an **Update** effective 01-Jul-2026.
2. In August, HR notices his working hours were entered as 40 when his offer letter said 48, right from day one. That was a data-entry mistake, so you do a **Correction**.

His assignment history now looks like this:

| Row | Effective Start | Effective End | Job | Grade | Why this row exists |
| --- | --- | --- | --- | --- | --- |
| 2 | 01-Jul-2026 | 31-Dec-4712 | Senior Store Associate | R2 | Update (Promotion) |
| 1 | 05-Jan-2026 | 30-Jun-2026 | Store Associate | R1 | Hire |

The hours fix changed values inside the rows. It did not add a third row.

| | Update | Correction |
| --- | --- | --- |
| Use it when | A real business change happens | Data was entered wrongly |
| New row created? | Yes | No |
| History kept? | Yes, old values stay visible | No, old value is replaced |
| Typical examples | Promotion, transfer, salary raise, manager change | Typo in name, wrong hours, wrong grade at hire |
| Payroll impact if back-dated | Retro pay from the new date | Retro pay for the whole corrected period |

### Multiple changes on the same day

What if Rahul gets a new manager AND a new location on the same day, in two separate transactions? Assignment data lives in a table ending in **`_M`** (`PER_ALL_ASSIGNMENTS_M`), which allows several rows for the same day. Two columns sort them out:

- **`EFFECTIVE_SEQUENCE`** – 1, 2, 3… the order of changes on that day.
- **`EFFECTIVE_LATEST_CHANGE`** – 'Y' on the last change of that day. Always filter on this in reports, or you get duplicate rows.

Tables ending in **`_F`** (like `PER_ALL_PEOPLE_F`) allow only one row per day.

### Edge cases to remember

- **Back-dated change into the middle of history.** If Rahul already has a future-dated row (say a transfer on 01-Oct-2026) and you add a salary-basis change effective 01-Aug-2026, Fusion inserts a new row between them. In general, Fusion carries the new value forward into later rows that still held the old value, but a later row that already has its own different value keeps it. Always open the later rows and check after a back-dated change, and test how your pod behaves.
- **Correcting a row that payroll already processed** triggers retro calculations. Payroll teams hate surprise corrections, so check with them first.
- **You can't update before the record starts.** No change can be dated before Rahul's hire date.
- **Wrong hire date is not a normal correction.** The start date is changed with a dedicated option on the work relationship (change or correct start date). It gets blocked or messy if payroll has run or if there are other changes between the old and new date.
- **Picking Correction for a real change destroys history.** If HR corrects the job to Senior Store Associate instead of updating, reports will show Rahul as Senior from day one. This is one of the most common mistakes in real projects.
- **Querying date-effective tables:** always add a date filter like `TRUNC(SYSDATE) BETWEEN effective_start_date AND effective_end_date`, or you'll get every historical row.

---

## Hire an Employee

A hire creates all three layers in one go: a new Person, a new Work Relationship with the legal employer, and the first Assignment. The hire date becomes the start date of the work relationship.

### Nike example

Nike India hires Rahul Verma as a Store Associate at the Bengaluru store, starting 05-Jan-2026.

### The flow (HR specialist)

Classic UI: My Client Groups → New Person → Hire an Employee. Redwood UI: Quick Actions → Hire an Employee. The steps are roughly:

1. **When and why** – hire date (05-Jan-2026), action (Hire), action reason (New Position / Replacement), legal employer (Nike India Pvt Ltd), person type (Employee).
2. **Personal details** – name, DOB, gender, national IDs (PAN, Aadhaar), address, phone, email. Fusion runs a **duplicate check** here (person matching on name, DOB, national ID).
3. **Employment details** – business unit, job, position, department, location, grade, manager, assignment category (full-time regular), working hours.
4. **Compensation and other info** – salary basis and amount, payroll, sometimes documents and checklists.
5. **Review and submit** – the transaction goes to approval if approval rules are configured. Once approved, the person number is generated (manual, automatic on submit, or automatic on final save, depending on setup).

### What gets created behind the scenes

- A person row (`PER_ALL_PEOPLE_F`, `PER_PERSONS`, `PER_PERSON_NAMES_F` and so on).
- A work relationship row (`PER_PERIODS_OF_SERVICE`) with `DATE_START` = 05-Jan-2026.
- Assignment rows (`PER_ALL_ASSIGNMENTS_M`) with `ACTION_CODE` = HIRE. In a three-tier model you also get a work-terms row.
- A manager row (`PER_ASSIGNMENT_SUPERVISORS_F`) and salary row (`CMP_SALARY`) if entered.
- A user account, usually through the user-provisioning process, so Rahul can log in to self-service.

### Edge cases to remember

- **Duplicate person found.** If the duplicate check finds an existing Rahul with the same DOB and PAN, stop. If he's an ex-employee, this is a **rehire**, not a new hire. Creating a second person record is a nasty clean-up job later.
- **Future-dated hire.** HR enters the hire in December with a 05-Jan start. Rahul exists in the system, but he's not active until that date and many actions aren't available before it. Some companies use **Pending Worker** instead and convert on the start date (there's a scheduled process that can do this automatically).
- **Back-dated hire.** Entered late, after payroll for that month already ran. Payroll picks it up as retro pay.
- **Already active in the same legal employer.** Fusion won't let you create a second active employee work relationship in Nike India.
- **Position full.** If hiring into a position whose headcount or FTE is used up, you get a warning or an error, depending on configuration.
- **Wrong hire date after saving.** Use the change-start-date option on the work relationship. It's blocked or complicated if payroll has run or other changes exist in between.
- **Hire made by mistake or the person never joins (no-show).** Use **Cancel Work Relationship**, not termination. Cancel removes it as if it never happened. Termination would leave a fake employment history. Cancel usually isn't possible once payroll has been processed for that relationship.
- **Bulk hiring** (for example, 300 seasonal store staff before the holiday sale) is done with **HCM Data Loader** using the Worker.dat file, not the UI.

---

## Rehire an Employee

A rehire reuses the **same Person** (same person number, same personal history) but creates a **brand-new Work Relationship and Assignment**. The old employment stays in history, untouched.

### Nike example

Rahul resigns from Nike India with a last day of 28-Feb-2027 to try a startup. It doesn't work out, and Nike India rehires him on 01-Sep-2027 as Senior Store Associate.

| | First stint | Rehire |
| --- | --- | --- |
| Person number | 100245 | 100245 (same) |
| Work relationship | 05-Jan-2026 to 28-Feb-2027 | 01-Sep-2027 onwards (new row) |
| Assignment number | E100245 | E100245-2 (new, format depends on setup) |
| Action on first assignment row | HIRE | REHIRE |

### The flow

1. HR starts **Hire an Employee** (or the Rehire / Add Work Relationship action from the person's record).
2. The duplicate check finds Rahul as an ex-employee. HR picks the existing person instead of creating a new one.
3. The action is **Rehire**. HR enters the new start date, legal employer and new job details. Personal details are already there; HR just reviews and updates them (new address, new phone).
4. Submit, approval, done. Rahul gets his user account access back through provisioning.

### Edge cases to remember

- **"Recommended for rehire" flag.** When Rahul left, HR set Rehire Recommendation = Yes or No (with a reason). If it was No, Fusion shows a warning during rehire. Whether it blocks or only warns depends on setup. Interviewers love this one.
- **Rehire date must be after the termination date.** Earliest possible is the day after (01-Mar-2027). You can't rehire into Nike India while the old work relationship is still active.
- **Rehire vs Reverse Termination.** If Rahul resigned on Monday and changed his mind on Wednesday, before his last day, you do **Reverse Termination**, not a rehire. Rehire is for someone who actually left and is coming back.
- **Rehire into a different legal employer** (say Nike USA) is also just a rehire with a new work relationship. It's not a Global Transfer, because there's no active work relationship to move from.
- **Seniority.** Whether his old service counts toward seniority (for leave or gratuity rules) depends on the seniority date rules configured. Don't assume it carries over.
- **Creating a new person by mistake.** If HR ignores the duplicate check, Rahul ends up with two person numbers and split history. Avoiding this is far easier than fixing it.
- **Payroll relationship.** Depending on country rules, the rehire may reuse his old payroll relationship or get a new one. Payroll consultants care about this.

---

## Transfer an Employee (same legal employer)

A transfer moves someone to a different department, location, business unit, manager or position **inside the same legal employer**. It's just an **Update** on the existing assignment: same work relationship, same assignment number, one new history row.

### Nike example

On 01-Oct-2026, Rahul moves from the Bengaluru store to the Mumbai store. His employer is still Nike India Pvt Ltd, so this is a normal transfer.

| Field | Before (till 30-Sep-2026) | After (from 01-Oct-2026) |
| --- | --- | --- |
| Legal employer | Nike India Pvt Ltd | Nike India Pvt Ltd (no change) |
| Location | Nike store, Bengaluru | Nike store, Mumbai |
| Department | Bengaluru Store Ops | Mumbai Store Ops |
| Manager | Bengaluru Store Manager | Mumbai Store Manager |
| Assignment number | E100245 | E100245 (same) |
| Action on new row | – | TRANSFER |

### The flow

Person's record → Actions → **Transfer** (Redwood: Quick Actions → Transfer). Enter the effective date, action reason (for example Employee Request, Business Need), then the changed fields: department, location, manager, position. Review, submit, approval.

Related actions that also just update the assignment: **Promotion** (job or grade goes up), **Change Location**, **Change Manager**, **Change Assignment** (general catch-all). Which one HR picks matters mostly for reporting: attrition and movement reports count by action.

### Edge cases to remember

- **Different legal employer = not a transfer.** Moving Rahul to Nike USA can't be done with Transfer, because legal employer sits on the work relationship. That's a Global Transfer.
- **Different business unit, same legal employer = still a transfer.** Moving from Retail India BU to a Digital India BU under Nike India Pvt Ltd is a normal transfer.
- **Transferring a manager.** If the Bengaluru Store Manager transfers, Fusion asks what to do with their **direct reports**. Forget this and 20 associates end up reporting to someone who isn't there anymore.
- **Future-dated transfer plus another change.** A transfer is approved for 01-Oct. In September, HR back-dates a salary change to 01-Sep. Fusion inserts a row before the transfer row; check the 01-Oct row afterwards so the new salary carried forward correctly.
- **Pending transaction lock.** If the transfer is still waiting for approval, another change on the same assignment may be blocked or warned. Approve or withdraw the first one.
- **Position-based setups.** The target position must be open, or you get a headcount/FTE warning or error.
- **Transfer entered as Correction.** Then the history says Rahul was always in Mumbai. Always use Update with the real transfer date.

---

## Global Transfer

A Global Transfer moves someone **permanently to a different legal employer**. Fusion ends the old work relationship the day before and starts a new work relationship (with a new assignment) on the transfer date. The person and person number stay the same.

### Nike example

Sarah Johnson, Software Engineer at Nike USA Inc. in Beaverton, moves permanently to Nike UK Ltd in London from 01-Apr-2027.

| | Old (Nike USA) | New (Nike UK) |
| --- | --- | --- |
| Person number | 200310 | 200310 (same) |
| Work relationship | Ends 31-Mar-2027, action Global Transfer | Starts 01-Apr-2027 (new row) |
| Assignment | Ends 31-Mar-2027 | New assignment, new assignment number |
| Legislation | US (SSN, US payroll) | UK (National Insurance number, UK payroll) |
| Primary work relationship | Was primary | Becomes primary |

### The flow

Person's record → Actions → **Global Transfer**. Enter the transfer date, action reason, the **destination legal employer** (Nike UK Ltd), then the new job, department, location, manager, salary and UK-specific info. Submit, approval.

### Global Transfer vs its cousins

This is the comparison interviewers ask most in this area:

| Action | Old work relationship | New work relationship | Nike example |
| --- | --- | --- | --- |
| Transfer | Same one, just updated | None | Rahul: Bengaluru → Mumbai, both Nike India |
| Global Transfer | Ended | Created, permanent | Sarah: Nike USA → Nike UK for good |
| Global Temporary Assignment | Stays, home assignment suspended | Created, temporary | Sarah goes to Nike UK for a 6-month project, then returns home |
| Add Work Relationship | Stays active | Created, both active together | Sarah keeps her US job and also works part-time for Nike UK |

For a **Global Temporary Assignment**, Fusion knows it's temporary: when it ends (End Global Temporary Assignment), the host assignment closes and the home assignment becomes active again.

### Edge cases to remember

- **Same legal employer?** Then it's not a global transfer. Use Transfer.
- **Country-specific mandatory data.** The UK may need a National Insurance number and UK address format. Missing legislative data is the most common reason the transaction errors out.
- **What doesn't move automatically.** Absence balances, benefits enrollment, payroll element entries, and sometimes seniority don't magically transfer. Each needs its own rules or manual action, and this is where real projects get messy.
- **Back-dated global transfer.** If entered after US payroll already paid Sarah for April, payroll needs retro and possibly recovery of overpayment.
- **Multiple assignments in the old legal employer.** All of them end with the old work relationship.
- **Direct reports.** If Sarah managed people in Beaverton, they need a new manager, same as any transfer.
- **Undoing a global transfer.** It touches two work relationships, so it's not a one-click undo. Typically the new relationship is cancelled and the old one's termination is reversed. Test this in your pod before you need it in production.
- **Person number** never changes. Assignment number always does, because it's a new assignment.

---

## Termination

Termination **ends the work relationship** on a given date. After that date, the person becomes an ex-employee for that legal employer, the assignment goes inactive, and system access is revoked as configured. The person record stays forever.

### Nike example

A store associate at the Mumbai store, Vikram, is terminated by Nike India for repeated policy violations. HR tells him on 10-Nov-2026, his last working day is 15-Nov-2026, and his termination date is 30-Nov-2026 (notice period paid in lieu).

### The dates people mix up

| Date | Meaning | Vikram's value |
| --- | --- | --- |
| Notification date | When the termination was communicated | 10-Nov-2026 |
| Last working date | Last day he physically worked | 15-Nov-2026 |
| Termination date | Official end of the work relationship. Assignment is inactive from the next day | 30-Nov-2026 |
| Final close date (payroll) | Payroll can still process final settlement until this date | Set by payroll rules |

### The flow

Person's record → Actions → **Terminate Employment** (Redwood: Quick Actions → Terminate Employment). Enter:

1. Termination date, action (Termination, Resignation, Retirement, Death and so on) and action reason.
2. Notification date and last working date.
3. **Recommended for rehire** – Yes or No, with a reason. For Vikram, likely No.
4. **Revoke user access** – immediately or after the termination date. For a misconduct case, often immediately.
5. Review, submit, approval.

Behind the scenes: `ACTUAL_TERMINATION_DATE` gets filled on `PER_PERIODS_OF_SERVICE`, and assignment rows from the next day carry an inactive status with `ACTION_CODE` = TERMINATION.

### Termination and its related actions

| Action | What it does | When to use it |
| --- | --- | --- |
| Terminate Employment | Ends the whole work relationship | Person is leaving the legal employer |
| End Assignment | Ends one assignment only | Person has several assignments and drops one |
| Correct Termination | Changes termination date or reason | Date or reason was entered wrongly |
| Reverse Termination | Undoes the termination fully, work relationship active again | Terminated by mistake, or resignation withdrawn |
| Cancel Work Relationship | Deletes the work relationship as if it never existed | Hire was a mistake or the person never joined |

### Edge cases to remember

- **Future-dated termination.** Vikram stays active until 30-Nov. Any change dated after 30-Nov (like a future transfer) blocks the termination; it must be removed first.
- **Terminating a manager.** Fusion asks you to reassign their direct reports. Skip it and approvals start routing to an ex-employee.
- **Primary work relationship terminated while another is active.** The other work relationship has to become primary.
- **Pending transactions.** A pending promotion or transfer for Vikram should be withdrawn before terminating, or it can fail or behave oddly.
- **Back-dated termination after payroll ran.** He was already paid for days after his termination date. Payroll has to recover the overpayment.
- **Reverse Termination limits.** You can't reverse it if the person was rehired into the same legal employer after the termination. Reversing may also be blocked or messy once final payroll is processed.
- **Termination date before hire date** is not allowed. Sounds obvious, but it shows up in HDL loads.
- **Death.** Use the Death action. The date of death is usually recorded on the person too, and benefits and final pay follow special rules.

---

## Resignation

A resignation is a **voluntary termination started by the employee**. Technically it ends up in the same place as a termination (work relationship ended, `ACTION_CODE` = RESIGNATION), but it usually starts in employee self-service and goes through manager and HR approval.

### Nike example

Rahul (now in the Mumbai store) decides to join a startup. On 25-Jan-2027 he submits his resignation in Fusion with a last day of 28-Feb-2027.

### The flow

1. **Rahul**: Me → Resignation (Redwood: "Resign" from his employment info). He enters the resignation date (last day), reason (Better Opportunity), comments, maybe the resignation letter as an attachment.
2. **Approval**: goes to his Store Manager, then HR, based on the approval rules.
3. **After approval**: Fusion records the termination with action Resignation, notification date = when he submitted, termination date = 28-Feb-2027. Offboarding checklist tasks (return uniform, exit interview, final settlement) can kick off.
4. On 01-Mar-2027 his assignment goes inactive and access is revoked as configured.

A manager or HR can also record a resignation on the employee's behalf: Terminate Employment with action = Resignation.

### Withdrawing a resignation

| When Rahul changes his mind | What happens |
| --- | --- |
| Before approval | He (or HR) withdraws the pending transaction. Nothing was ever saved |
| After approval, before or after the last day, no rehire yet | HR does **Reverse Termination**. Work relationship is active again as if he never left |
| After he's gone and someone needs to bring him back later | **Rehire**, a new work relationship |

### Edge cases to remember

- **Notice period.** Indian companies often have 30 to 90 days. Fusion can show or default the notice period based on setup, but HR may accept an earlier last day (buyout) or keep the agreed one.
- **Past dates.** Self-service usually doesn't allow a past resignation date. HR can back-date if needed, with payroll impact.
- **Resigning during a pending promotion or transfer.** Two pending transactions clash. Usually the promotion is withdrawn.
- **Resigning from one of two assignments.** If Rahul has two assignments and wants to drop one, that's End Assignment, not a resignation from the whole work relationship.
- **Counter offer accepted after the last day passed.** Reverse Termination can still work if he hasn't been rehired. Check payroll first: his final settlement may already be paid.
- **Voluntary vs involuntary matters.** Attrition reports, final settlement rules and rehire decisions often depend on whether the action was Resignation or Termination, so pick the right action, not just any termination.

---

## Behind the scenes: key tables

Interviewers on Fusion HCM projects often ask "where is this stored?" because you'll write BI Publisher reports and HDL loads on these tables. Suffix rule: **`_F`** = date-effective, one change per day; **`_M`** = date-effective, multiple changes per day; no suffix = usually not date-effective.

| Table | What it stores | Date-effective? | Columns worth knowing |
| --- | --- | --- | --- |
| `PER_PERSONS` | Core person data like date of birth | No | `PERSON_ID`, `DATE_OF_BIRTH` |
| `PER_ALL_PEOPLE_F` | Person record over time | Yes (`_F`) | `PERSON_ID`, `PERSON_NUMBER`, `EFFECTIVE_START_DATE`, `EFFECTIVE_END_DATE` |
| `PER_PERSON_NAMES_F` | Names | Yes (`_F`) | `PERSON_ID`, `NAME_TYPE` ('GLOBAL' vs local), `FIRST_NAME`, `LAST_NAME`, `DISPLAY_NAME` |
| `PER_PERIODS_OF_SERVICE` | Work relationships | No | `PERIOD_OF_SERVICE_ID`, `PERSON_ID`, `LEGAL_ENTITY_ID`, `PERIOD_TYPE`, `DATE_START`, `ACTUAL_TERMINATION_DATE`, `NOTIFIED_TERMINATION_DATE`, `PRIMARY_FLAG` |
| `PER_ALL_ASSIGNMENTS_M` | Assignments and work terms | Yes (`_M`) | `ASSIGNMENT_ID`, `ASSIGNMENT_NUMBER`, `ASSIGNMENT_TYPE`, `PRIMARY_FLAG`, `ACTION_CODE`, `REASON_CODE`, `ASSIGNMENT_STATUS_TYPE`, `EFFECTIVE_SEQUENCE`, `EFFECTIVE_LATEST_CHANGE`, `PERIOD_OF_SERVICE_ID`, `JOB_ID`, `POSITION_ID`, `ORGANIZATION_ID`, `LOCATION_ID`, `GRADE_ID` |
| `PER_ASSIGNMENT_SUPERVISORS_F` | Managers | Yes (`_F`) | `ASSIGNMENT_ID`, `MANAGER_ID`, `MANAGER_TYPE` |
| `PER_ACTIONS_B` / `PER_ACTIONS_VL` | Action definitions (Hire, Transfer…) | No | `ACTION_ID`, `ACTION_CODE` |
| `PER_ACTION_REASONS_B` / `_VL` | Action reasons | No | `ACTION_REASON_ID`, `ACTION_REASON_CODE` |
| `PER_ACTION_OCCURRENCES` | Each time an action actually happened | No | `ACTION_OCCURRENCE_ID`, `ACTION_ID`, `ACTION_REASON_ID` |
| `PER_ASSIGNMENT_STATUS_TYPES` | Assignment statuses | No | `ASSIGNMENT_STATUS_TYPE_ID`, `USER_STATUS` |
| `CMP_SALARY` | Salary | Date ranges | `ASSIGNMENT_ID`, `SALARY_AMOUNT`, `DATE_FROM`, `DATE_TO` |

**Codes to know**

- `PERIOD_TYPE` / worker type: **E** = Employee, **C** = Contingent Worker, **N** = Nonworker, **P** = Pending Worker.
- `ASSIGNMENT_TYPE`: **E, C, N, P** = assignments; **ET, CT, NT, PT** = employment terms rows (three-tier).
- Typical `ACTION_CODE` values: HIRE, REHIRE, TRANSFER, PROMOTION, GLB_TRANSFER, TERMINATION, RESIGNATION. Exact codes vary with seeded vs custom actions, so check `PER_ACTIONS_VL` in your instance.

**Where each transaction shows up**

| Transaction | Look here |
| --- | --- |
| Hire | New `PER_PERIODS_OF_SERVICE` row; first assignment row with `ACTION_CODE` = HIRE |
| Rehire | Same `PERSON_ID`, second `PER_PERIODS_OF_SERVICE` row; new assignment with `ACTION_CODE` = REHIRE |
| Transfer | New row in `PER_ALL_ASSIGNMENTS_M`, same `ASSIGNMENT_ID`, `ACTION_CODE` = TRANSFER |
| Global Transfer | Old period of service gets `ACTUAL_TERMINATION_DATE`; new period of service with a different `LEGAL_ENTITY_ID` |
| Termination / Resignation | `ACTUAL_TERMINATION_DATE` filled; inactive assignment rows from the next day |

**Sample query: current active employees with their primary assignment**

```sql
SELECT papf.person_number,
       ppnf.display_name,
       ppos.date_start           AS hire_date,
       paam.assignment_number,
       paam.action_code
FROM   per_all_people_f        papf
JOIN   per_person_names_f      ppnf ON ppnf.person_id = papf.person_id
                                   AND ppnf.name_type = 'GLOBAL'
                                   AND TRUNC(SYSDATE) BETWEEN ppnf.effective_start_date AND ppnf.effective_end_date
JOIN   per_periods_of_service  ppos ON ppos.person_id = papf.person_id
JOIN   per_all_assignments_m   paam ON paam.period_of_service_id = ppos.period_of_service_id
                                   AND paam.assignment_type = 'E'
                                   AND paam.primary_flag = 'Y'
                                   AND paam.effective_latest_change = 'Y'
                                   AND paam.assignment_status_type = 'ACTIVE'
                                   AND TRUNC(SYSDATE) BETWEEN paam.effective_start_date AND paam.effective_end_date
WHERE  TRUNC(SYSDATE) BETWEEN papf.effective_start_date AND papf.effective_end_date
AND    ppos.period_type = 'E';
```

The three filters people forget: the date filter on every `_F`/`_M` table, `EFFECTIVE_LATEST_CHANGE = 'Y'`, and `NAME_TYPE = 'GLOBAL'`. Miss any one and you get duplicate rows.

---

## Interview Q&A: basic to tricky

Answer in this order in the interview: one-line definition, then a Nike-style example, then one edge case. That pattern shows you understand it, not just memorised it.

### Level 1: Basics

**Q1. What are Person, Work Relationship and Assignment?**
Person is who you are (name, DOB, national ID), one per human. Work Relationship is your link with a legal employer, with a start and end date. Assignment is the job you do inside it: job, department, location, manager. Example: Rahul (person) works for Nike India Pvt Ltd (work relationship) as a Store Associate in Bengaluru (assignment).

**Q2. Where is the legal employer stored: work relationship or assignment?**
On the work relationship. That's exactly why changing legal employer needs a Global Transfer and can't be done by editing the assignment.

**Q3. What are the worker types?**
Employee, Contingent Worker, Nonworker, and Pending Worker. Companies can also create user person types (like Intern) mapped to these.

**Q4. What's the difference between two-tier and three-tier employment models?**
Two-tier: Work Relationship → Assignment. Three-tier adds Employment Terms in the middle, used when several assignments share the same contract or pay terms.

**Q5. What's a primary assignment / primary work relationship?**
If a person has several, one is marked primary. Reports, payroll defaults and approvals usually use the primary one. One primary work relationship per person; one primary assignment per work relationship.

**Q6. What is a Pending Worker?**
Someone who accepted an offer but hasn't joined. They get converted to an employee on their start date, manually or by a scheduled process.

### Level 2: Effective dating

**Q7. Update vs Correction?**
Update creates a new row from the effective date and keeps history: use it for real changes like Rahul's promotion. Correction fixes the existing row in place without history: use it for mistakes like wrong working hours at hire.

**Q8. What does 31-Dec-4712 mean?**
It's Oracle's "end of time". A row ending on that date is the current, open-ended row.

**Q9. What's the difference between `_F` and `_M` tables?**
`_F` allows one change per day. `_M` allows multiple changes on the same day, tracked with `EFFECTIVE_SEQUENCE`, and the last one has `EFFECTIVE_LATEST_CHANGE = 'Y'`. Assignments use `_M`.

**Q10. HR made a promotion using Correction instead of Update. What's the impact?**
History is lost. Reports will show Rahul as Senior Store Associate from his hire date, and if payroll is involved, retro pay will be calculated for the whole period. Fix it by correcting the row back, then doing a proper Update from the real promotion date.

**Q11. You make a back-dated change, and the person already has a future-dated change. What happens?**
Fusion inserts the new row between them. In general the new value carries forward into later rows that still had the old value, but a later row with its own different value keeps it. Always check the later rows after saving.

### Level 3: Transactions

**Q12. What gets created when you hire someone?**
A person record, a work relationship (period of service) starting on the hire date, and an assignment (plus employment terms in three-tier). Usually also a manager link, salary, and a user account through provisioning.

**Q13. Hire vs Rehire?**
Hire creates a new person. Rehire reuses the existing person and person number, and creates a new work relationship and assignment. Old history stays.

**Q14. Transfer vs Global Transfer?**
Transfer stays in the same legal employer and just updates the assignment (Rahul: Bengaluru to Mumbai, both Nike India). Global Transfer moves to a different legal employer: old work relationship ends, new one starts (Sarah: Nike USA to Nike UK).

**Q15. Global Transfer vs Global Temporary Assignment vs Add Work Relationship?**
Global Transfer: permanent, old ends. Global Temporary Assignment: temporary, home assignment suspended, returns home at the end. Add Work Relationship: both stay active at the same time (concurrent employment).

**Q16. Does the person number change in a global transfer?**
No. Person number never changes. The assignment number does, because it's a new assignment in a new work relationship.

**Q17. Termination vs Resignation?**
Both end the work relationship. Resignation is voluntary, usually started by the employee in self-service and approved by manager and HR. Termination is usually initiated by the employer. The action code matters for attrition reports and final settlement.

**Q18. Explain the termination dates.**
Notification date = when it was communicated. Last working date = last day physically worked. Termination date = official end; assignment is inactive from the next day. Final close date = last date payroll can still process final pay.

### Level 4: Tricky scenarios

**Q19. An employee was terminated by mistake. Rehire or Reverse Termination?**
Reverse Termination. It restores the same work relationship as if the termination never happened. Rehire would create a second work relationship and break the continuous service history.

**Q20. A new joiner never showed up on day one. Terminate them?**
No. Use Cancel Work Relationship, which removes it as if it never existed. Terminating would create fake employment history. Cancel usually isn't allowed once payroll has processed that relationship.

**Q21. Can you rehire someone on the same day as their termination?**
Not into the same legal employer. The earliest rehire date is the day after the termination date, because two employee work relationships in the same legal employer can't overlap.

**Q22. HR tries to terminate an employee but gets an error because of a future-dated transfer. Why?**
The transfer is dated after the termination date, so it would exist on an ended relationship. Remove or correct the future change first, then terminate.

**Q23. A store manager with 20 direct reports resigns. What should HR check?**
The direct reports step: reassign all 20 to a new manager, effective the day after the manager's termination date. Otherwise approvals route to an ex-employee and org charts break.

**Q24. Rahul resigned, it got approved, and two days later he accepted a counter offer. His last day hasn't come yet. What do you do?**
Reverse Termination. If his resignation was still pending approval, just withdraw the transaction.

**Q25. While rehiring, the system warns "not recommended for rehire". What does that mean?**
When he left, HR set the rehire recommendation to No. Depending on setup it's a warning or a block. HR should check the reason before going ahead.

**Q26. HR clicked Hire for a returning employee and created a new person with a new person number. What's the problem?**
The person now has two records and split history: two person numbers, separate absence and payroll history. The duplicate check should have been used to pick the existing person and do a Rehire. Prevention is key; clean-up is painful.

**Q27. Sarah is global-transferred from Nike USA to Nike UK. What doesn't move automatically?**
Absence balances, benefits, payroll element entries and sometimes seniority. Each needs configured rules or manual action. Also UK-specific mandatory data like the National Insurance number must be entered.

**Q28. Employee has two assignments in the same work relationship and wants to leave one. Terminate?**
No. Use End Assignment on that one. Terminate would end the whole work relationship, including the assignment they're keeping.

**Q29. Promotion and transfer on the same day in two separate transactions. How are they stored?**
Two rows in `PER_ALL_ASSIGNMENTS_M` with the same start date, `EFFECTIVE_SEQUENCE` 1 and 2. The second one has `EFFECTIVE_LATEST_CHANGE = 'Y'`.

**Q30. A termination was back-dated after payroll already paid the employee for days after the termination date. What happens?**
Payroll sees an overpayment and needs to recover it through retro or a manual adjustment. That's why back-dated terminations should go through payroll before they're saved.

### Level 5: Tables and reporting

**Q31. Which table stores work relationships, and which column holds the termination date?**
`PER_PERIODS_OF_SERVICE`. `ACTUAL_TERMINATION_DATE` holds the termination date; `DATE_START` holds the hire date.

**Q32. Your report on `PER_ALL_ASSIGNMENTS_M` shows duplicate rows for some employees. Why?**
Usually three reasons: no date filter, no `EFFECTIVE_LATEST_CHANGE = 'Y'` filter, or not filtering `ASSIGNMENT_TYPE` (so both E and ET rows come back). Also check `PRIMARY_FLAG` if they have several assignments.

**Q33. How do you find all employees rehired in 2027?**
Assignment rows with `ACTION_CODE = 'REHIRE'` and an effective start date in 2027, or people who have more than one `PER_PERIODS_OF_SERVICE` row with `DATE_START` in 2027 on the later one.

**Q34. How do you find who did a global transfer?**
Look for `ACTION_CODE` = GLB_TRANSFER (check the exact code in `PER_ACTIONS_VL`), or a person whose period of service ended on day X−1 and a new one started on day X with a different `LEGAL_ENTITY_ID`.

**Q35. How would you load 300 seasonal hires?**
HCM Data Loader with Worker.dat, including the Worker, PersonName, WorkRelationship, WorkTerms and Assignment components. Not the UI.

---

## Revision cheat sheet

Read this table the night before the interview.

| Action | Person | Work relationship | Assignment | Assignment number | Top edge case |
| --- | --- | --- | --- | --- | --- |
| Hire | New | New | New | New | Duplicate found → it's a rehire |
| Rehire | Same | New | New | New | Not recommended for rehire flag; date must be after termination |
| Transfer | Same | Same | Updated (new row) | Same | Different legal employer → Global Transfer instead |
| Global Transfer | Same | Old ends, new starts | New | New | Absence, benefits, seniority don't move automatically |
| Global Temporary Assignment | Same | Home stays, host added | Home suspended, host new | New for host | Home becomes active again at the end |
| Termination | Same | Ended | Inactive from next day | Same | Future-dated changes block it; reassign direct reports |
| Resignation | Same | Ended | Inactive from next day | Same | Withdraw before approval; Reverse Termination after |
| Reverse Termination | Same | Active again | Active again | Same | Not possible after a rehire in the same legal employer |
| Cancel Work Relationship | Same | Deleted | Deleted | – | For no-shows and mistaken hires, not for leavers |

**Five lines to remember**

1. Legal employer lives on the work relationship, never on the assignment.
2. Person number is for life; assignment number changes with every new assignment.
3. Update = real change with history; Correction = fixing a mistake, no history.
4. `_M` tables: always filter `EFFECTIVE_LATEST_CHANGE = 'Y'` plus the date range.
5. Mistaken termination → Reverse Termination. Never joined → Cancel Work Relationship. Came back later → Rehire.