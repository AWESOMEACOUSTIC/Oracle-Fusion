# Oracle Fusion Core HR — Person, Work Relationship & Assignment
### Study guide with worked examples and table-level data

> **How to read this guide:** one storyline runs all the way through. The same people (Rahul, Sarah, Vikram) appear in every section. At each step you see **what HR does in the UI**, then **what the rows in the tables look like afterwards**. Interviewers ask about the rows, so learn them.
>
> The IDs, numbers and action codes are illustrative. Column names are real Fusion columns. Exact action codes and lookup values can vary by instance, so check `PER_ACTIONS_B` in yours.

---

## 0. The cast

| Person | Person Number | PERSON_ID | Story |
| --- | --- | --- | --- |
| Rahul Mehta | 10045 | 300100 | Hired by Nike India, promoted, resigns, gets rehired later |
| Sarah Sharma (later Patel) | 10078 | 300200 | Works for Nike USA, has two assignments, changes surname, global transfer to Nike UK |
| Vikram Rao | 10091 | 300300 | Store Manager at Nike India, gets terminated (he has direct reports) |

Legal employers (LEGAL_ENTITY_ID): **Nike India Pvt Ltd = 1001**, **Nike USA Inc = 2001**, **Nike UK Ltd = 3001**. All three sit under one enterprise: **Nike**.

---

## 1. The three layers: Person → Work Relationship → Assignment

### The analogy

Think of a **SIM card in a phone**:

| Layer | Analogy | Question it answers | Lifetime | Main table |
| --- | --- | --- | --- | --- |
| **Person** | You, the human | *Who are you?* | Forever. Never deleted or recreated | `PER_ALL_PEOPLE_F`, `PER_PERSONS` |
| **Work Relationship** | Your contract with a telecom company | *Who legally employs you?* | Hire date → termination date | `PER_PERIODS_OF_SERVICE` |
| **Assignment** | The plan on that SIM (data, calls) | *What job are you doing?* | Starts with the WR, can end earlier | `PER_ALL_ASSIGNMENTS_M` |

```
PERSON  (Rahul, 10045)                       ← exists once, for life
  └── WORK RELATIONSHIP (Nike India, 2026–2027)   ← one per stint per legal employer
  │       └── ASSIGNMENT  (Store Associate → Senior Store Associate)
  └── WORK RELATIONSHIP (Nike India, 2027– )      ← rehire = NEW work relationship
          └── ASSIGNMENT  (Assistant Store Manager)
```

### Rahul's whole life in the tables (we build this up section by section)

**PER_ALL_PEOPLE_F**: one person, one person number, both stints

| PERSON_ID | PERSON_NUMBER | EFFECTIVE_START_DATE | EFFECTIVE_END_DATE |
| --- | --- | --- | --- |
| 300100 | 10045 | 05-Jan-2026 | 31-Dec-4712 |

**PER_PERIODS_OF_SERVICE**: one row per work relationship

| PERIOD_OF_SERVICE_ID | PERSON_ID | LEGAL_ENTITY_ID | PERIOD_TYPE | DATE_START | ACTUAL_TERMINATION_DATE | PRIMARY_FLAG |
| --- | --- | --- | --- | --- | --- | --- |
| 500001 | 300100 | 1001 (Nike India) | E | 05-Jan-2026 | 28-Feb-2027 | N |
| 500002 | 300100 | 1001 (Nike India) | E | 01-Aug-2027 | *(null = still active)* | Y |

**What to notice:**
- Same `PERSON_ID` on both rows. The person was **not** recreated.
- Two different `PERIOD_OF_SERVICE_ID`s. A rehire is a **new** work relationship.
- `ACTUAL_TERMINATION_DATE` is null on the active WR. That's how you find current employment in SQL.

---

## 2. Worker types

| Worker Type (system person type) | Example at Nike | Paid through | `PERIOD_TYPE` | `ASSIGNMENT_TYPE` (assignment / terms) |
| --- | --- | --- | --- | --- |
| **Employee** | Rahul, a Store Associate | Nike's own payroll | E | E / ET |
| **Contingent Worker** | A contractor from Infosys building Nike's app for 8 months | The supplier's invoice / purchase order, **not** Nike payroll | C | C / CT |
| **Nonworker** | A retiree still receiving pension benefits, or a board member | Not paid for work (may get benefits or pension) | N | N / NT |
| **Pending Worker** | Someone who signed the offer on 10-Dec and joins 05-Jan | Nobody yet | P | P / PT |

> **Correction to the notes:** a contingent worker isn't really a "temporary employee." The big difference is **who pays them**. An employee is on Nike's payroll. A contingent worker is paid by an agency or supplier (Nike pays the supplier's invoice). A temporary *employee*, like a 6-month fixed-term hire on Nike payroll, is still an **Employee** with assignment category = Temporary.

### ❓ Your question: an intern. Which worker type?

There is **no "Intern" system person type**. Fusion only has the four above. "Intern" gets set up as a **user person type** under one of them. The rule that decides which one is the same as before: **who pays them, and how**.

| Intern scenario | Worker type | Why |
| --- | --- | --- |
| Paid stipend through Nike's own payroll (taxes and payslip from Nike) | **Employee** (user person type "Intern", assignment category "Temporary") | On Nike's payroll |
| Placed and paid by a university or a staffing agency | **Contingent Worker** (user person type "Intern") | Nike doesn't pay them through payroll |
| Unpaid, but needs a badge, system access or a manager | **Contingent Worker** or **Nonworker**, depending on company policy | Needs to be tracked, but is not on payroll |
| Accepted an internship offer, starts next month | **Pending Worker**, converted on the start date | Hasn't joined yet |

**Interview answer:** *"It depends on how the intern is paid. If they're on our payroll, I'd hire them as an Employee with an 'Intern' user person type. If an agency or university pays them, they're a Contingent Worker. Fusion's system person type stays one of the four, and 'Intern' is a user person type layered on top."*

---

## 3. Employment models

Fusion supports three models. You choose one per enterprise or legal employer during implementation.

| Model | Layers stored | Multiple assignments per WR? | Typical use |
| --- | --- | --- | --- |
| **Two-tier, single assignment** | WR → 1 Assignment | No | Simple orgs, most common in practice |
| **Two-tier, multiple assignments** | WR → many Assignments | Yes | One person, several jobs, each with its own terms |
| **Three-tier** | WR → Employment Terms → Assignment(s) | Yes | Several assignments sharing **one** contract (hours, notice period, pay frequency) |

### How it looks in PER_ALL_ASSIGNMENTS_M

Rahul at Nike India with two jobs: **Java Developer** (IT) and **Technical Trainer** (Training).

**Two-tier, multiple assignments.** Each assignment carries its own terms:

| ASSIGNMENT_ID | ASSIGNMENT_TYPE | ASSIGNMENT_NUMBER | JOB | NORMAL_HOURS | WORK_TERMS_ASSIGNMENT_ID | PRIMARY_ASSIGNMENT_FLAG |
| --- | --- | --- | --- | --- | --- | --- |
| 700001 | E | E10045 | Java Developer | 30 | — | Y |
| 700002 | E | E10045-2 | Technical Trainer | 10 | — | N |

**Three-tier.** One terms row (`ET`) with both assignments hanging from it:

| ASSIGNMENT_ID | ASSIGNMENT_TYPE | ASSIGNMENT_NUMBER | JOB | NORMAL_HOURS | WORK_TERMS_ASSIGNMENT_ID | PRIMARY_ASSIGNMENT_FLAG |
| --- | --- | --- | --- | --- | --- | --- |
| 700000 | **ET** | ET10045 | — (Full time, 40 hrs, 90-day notice) | 40 | — | — |
| 700001 | E | E10045 | Java Developer | 30 | **700000** | Y |
| 700002 | E | E10045-2 | Technical Trainer | 10 | **700000** | N |

> **Interview trap:** in the three-tier model, employment terms are **also rows in `PER_ALL_ASSIGNMENTS_M`** (type ET/CT/NT/PT). If your report counts assignments without filtering `ASSIGNMENT_TYPE IN ('E','C','N','P')`, you'll double-count every worker.

---

## 4. Primary Flag vs Primary Assignment

These are two different flags at two different levels.

| Flag | Level | Question it answers | Column |
| --- | --- | --- | --- |
| **Primary work relationship** | Across legal employers | "Out of Sarah's employers, which one is her main one?" | `PER_PERIODS_OF_SERVICE.PRIMARY_FLAG` |
| **Primary assignment** | Inside one work relationship | "Within Nike USA, which of Sarah's jobs is her main one?" | `PER_ALL_ASSIGNMENTS_M.PRIMARY_ASSIGNMENT_FLAG` |
| **Overall primary assignment** | Across everything | "Which one assignment represents Sarah?" (used by absence, reporting, approvals) | `PER_ALL_ASSIGNMENTS_M.PRIMARY_FLAG` |

### Sarah's data

**PER_PERIODS_OF_SERVICE**

| PERIOD_OF_SERVICE_ID | LEGAL_ENTITY | PERIOD_TYPE | PRIMARY_FLAG |
| --- | --- | --- | --- |
| 510001 | Nike USA | E (Employee) | **Y** |
| 510002 | Nike UK | C (Contingent) | N |

**PER_ALL_ASSIGNMENTS_M** (current rows)

| ASSIGNMENT_NUMBER | WR | JOB | PRIMARY_ASSIGNMENT_FLAG (within WR) | PRIMARY_FLAG (overall) |
| --- | --- | --- | --- | --- |
| E10078 | Nike USA | Software Engineer | **Y** | **Y** |
| E10078-2 | Nike USA | Technical Trainer | N | N |
| C10078 | Nike UK | Consultant | **Y** (only one in that WR) | N |

**What to notice:** every work relationship has exactly one primary assignment. Only one assignment in the person's **whole** record has the overall `PRIMARY_FLAG = 'Y'`.

**Why it matters:** if Sarah's manager runs a "my team" report, or an approval routes "to the worker's manager," Fusion needs one answer. It uses the primary assignment.

---

## 5. Scenario cases (as a decision table)

| # | Situation | Allowed? | Correct action | What happens in the tables |
| --- | --- | --- | --- | --- |
| 1 | Move Sarah from Nike USA to Nike UK permanently | Can't just edit the legal employer on the assignment | **Global Transfer** | Old WR gets `ACTUAL_TERMINATION_DATE` = day before; new WR row is created; new assignment is created |
| 2 | Hire Rahul again as an *employee* into Nike India while his Nike India employee WR is active | ❌ No | — (add a 2nd **assignment** instead if he needs a second job) | Fusion blocks it. No new `PER_PERIODS_OF_SERVICE` row |
| 3 | Rahul is an employee of Nike India **and** Nike UK at the same time | ✅ Yes (same enterprise) | **Add Employment / concurrent work relationship** | 2 active `PER_PERIODS_OF_SERVICE` rows, one marked `PRIMARY_FLAG = Y` |
| 4 | Rahul drops his Technical Trainer job but stays a Java Developer | ✅ | **End Assignment** (assignment 2 only) | Assignment 2 gets an inactive row. WR stays open (`ACTUAL_TERMINATION_DATE` still null) |
| 5 | Rahul's last/only active assignment is ended | — | Use **Terminate**. Ending the only assignment ends the WR too | WR gets `ACTUAL_TERMINATION_DATE`; assignment goes INACTIVE |

---

## 6. Date-effective vs Non-date-effective data

**Date-effective** means Fusion keeps history. Every change becomes a new row with `EFFECTIVE_START_DATE` and `EFFECTIVE_END_DATE`.
**Non-date-effective** means there's one row, and a change overwrites it. No history in the table (only the audit trail, if audit is enabled).

### Example: Sarah marries on 15-Mar-2027

**PER_PERSON_NAMES_F** (date-effective, suffix `_F`)

| PERSON_ID | LAST_NAME | FIRST_NAME | EFFECTIVE_START_DATE | EFFECTIVE_END_DATE |
| --- | --- | --- | --- | --- |
| 300200 | Sharma | Sarah | 01-Jun-2024 | 14-Mar-2027 |
| 300200 | Patel | Sarah | 15-Mar-2027 | 31-Dec-4712 |

A payslip printed for February 2027 can still show "Sarah Sharma". Query the name "as of" a date and you get the correct one.

**PER_PERSONS** (non-date-effective). Date of birth has exactly one row:

| PERSON_ID | DATE_OF_BIRTH |
| --- | --- |
| 300200 | 12-Aug-1998 |

### Cheat table: which attribute lives where

| Attribute | Date-effective? | Table |
| --- | --- | --- |
| Name | ✅ Yes | `PER_PERSON_NAMES_F` |
| Address | ✅ Yes | `PER_ADDRESSES_F` (+ `PER_PERSON_ADDR_USAGES_F`) |
| Gender, marital status | ✅ Yes | `PER_PEOPLE_LEGISLATIVE_F` |
| Job, grade, department, location, hours | ✅ Yes (multiple changes/day) | `PER_ALL_ASSIGNMENTS_M` |
| Manager | ✅ Yes | `PER_ASSIGNMENT_SUPERVISORS_F` |
| Salary | ✅ Yes (uses `DATE_FROM` / `DATE_TO`) | `CMP_SALARY` |
| Date of birth | ❌ No | `PER_PERSONS` |
| National ID (Aadhaar, PAN, SSN) | ❌ No | `PER_NATIONAL_IDENTIFIERS` |
| Email | ❌ No | `PER_EMAIL_ADDRESSES` |
| Phone | ❌ No | `PER_PHONES` |

### The special dates

| Date | Meaning |
| --- | --- |
| **31-Dec-4712** | "End of time." This row is valid until something changes. The current row always ends here |
| **01-Jan-1951** | Default start date for **setup data** (jobs, departments, locations, grades) so they're valid for any historical hire |
| **Hire date** | The start date of a worker's **first** assignment row (not 1951) |

---

## 7. Update vs Correction

| | **Update** | **Correction** |
| --- | --- | --- |
| Meaning | "Something really changed from this date" | "This was wrong from the start" |
| New row? | ✅ Yes. The current row is end-dated and a new row starts | ❌ No. The existing row is overwritten |
| History preserved? | Yes, both old and new values are visible | No, the wrong value disappears from the table (audit only) |
| Example | Promotion, transfer, salary raise | Typo in hours, wrong grade keyed at hire |

### Worked example: Rahul (hired 05-Jan-2026, Store Associate, R1, Mumbai, 40 hrs)

**Step 1: the hire.** One row:

| Row | EFFECTIVE_START_DATE | EFFECTIVE_END_DATE | ACTION_CODE | JOB | GRADE | NORMAL_HOURS |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 05-Jan-2026 | 31-Dec-4712 | HIRE | Store Associate | R1 | 40 |

**Step 2: UPDATE.** Promotion effective 01-Jul-2026. Row 1 is end-dated and row 2 starts:

| Row | EFFECTIVE_START_DATE | EFFECTIVE_END_DATE | ACTION_CODE | JOB | GRADE | NORMAL_HOURS |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 05-Jan-2026 | **30-Jun-2026** | HIRE | Store Associate | R1 | 40 |
| 2 | **01-Jul-2026** | 31-Dec-4712 | PROMOTION | Senior Store Associate | R2 | 40 |

**Step 3: CORRECTION.** In August HR finds hours should have been 48 from day one. No new row. The value is overwritten in place:

| Row | EFFECTIVE_START_DATE | EFFECTIVE_END_DATE | ACTION_CODE | JOB | GRADE | NORMAL_HOURS |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 05-Jan-2026 | 30-Jun-2026 | HIRE | Store Associate | R1 | **48** ✏️ |
| 2 | 01-Jul-2026 | 31-Dec-4712 | PROMOTION | Senior Store Associate | R2 | **40 ⚠️ still wrong** |

> **The gotcha:** correcting row 1 does **not** automatically fix row 2. The promotion row copied "40" when it was created. HR must correct row 2 as well. Otherwise the history reads "48 hrs until June, then 40," which never happened.

**Final, correct history:**

| Row | EFFECTIVE_START_DATE | EFFECTIVE_END_DATE | ACTION_CODE | JOB | GRADE | NORMAL_HOURS |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 05-Jan-2026 | 30-Jun-2026 | HIRE | Store Associate | R1 | 48 |
| 2 | 01-Jul-2026 | 31-Dec-4712 | PROMOTION | Senior Store Associate | R2 | 48 |

**What if HR had used Update instead of Correction for the hours fix (effective 01-Aug)?**

| Row | Start | End | Hours | The history now claims… |
| --- | --- | --- | --- | --- |
| 1 | 05-Jan-2026 | 30-Jun-2026 | 40 | Rahul worked 40 hrs ❌ (false) |
| 2 | 01-Jul-2026 | 31-Jul-2026 | 40 | Rahul worked 40 hrs ❌ |
| 3 | 01-Aug-2026 | 31-Dec-4712 | 48 | Rahul's hours "increased" on 01-Aug ❌ (never happened) |

That's why the choice matters: **Update records an event. Correction fixes a mistake.**

---

### Multiple changes on the same day (_M tables)

On **01-Oct-2026** two separate transactions are approved for Rahul: (1) location Mumbai → Pune, (2) hours 48 → 45.

| Row | EFFECTIVE_START_DATE | EFFECTIVE_END_DATE | EFFECTIVE_SEQUENCE | EFFECTIVE_LATEST_CHANGE | ACTION_CODE | LOCATION | NORMAL_HOURS |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 2 | 01-Jul-2026 | 30-Sep-2026 | 1 | Y | PROMOTION | Mumbai | 48 |
| 3 | 01-Oct-2026 | 31-Dec-4712 | **1** | **N** | LOCATION_CHANGE | Pune | 48 |
| 4 | 01-Oct-2026 | 31-Dec-4712 | **2** | **Y** | ASG_CHANGE | Pune | **45** |

**What to notice:**
- Rows 3 and 4 have the **same dates**. Only `EFFECTIVE_SEQUENCE` tells them apart.
- Row 4 (sequence 2) carries **both** changes (Pune + 45 hrs). It is the true end-of-day state.
- If a report forgets `EFFECTIVE_LATEST_CHANGE = 'Y'`, Rahul appears **twice** for any date from 01-Oct onwards.

| Suffix | Rows per day per record | Examples |
| --- | --- | --- |
| `_F` | Exactly one | `PER_ALL_PEOPLE_F`, `PER_PERSON_NAMES_F`, `PER_ASSIGNMENT_SUPERVISORS_F` |
| `_M` | Many (needs sequence and latest-change flags) | `PER_ALL_ASSIGNMENTS_M`, `PER_PERSON_TYPE_USAGES_M` |

---

### ⭐ Back-dated (retroactive) changes (IMP: this fills the gap in your notes)

**Situation:** it's September 2026. HR learns Rahul actually moved to the **Pune** store on **01-Apr-2026**, but nobody entered it. Existing history:

| Row | Start | End | Action | Job | Location |
| --- | --- | --- | --- | --- | --- |
| 1 | 05-Jan-2026 | 30-Jun-2026 | HIRE | Store Associate | Mumbai |
| 2 | 01-Jul-2026 | 31-Dec-4712 | PROMOTION | Senior Store Associate | Mumbai |

HR does an **Update effective 01-Apr-2026** (a real move, so not a correction). Fusion **splits row 1**:

| Row | Start | End | Action | Job | Location |
| --- | --- | --- | --- | --- | --- |
| 1 | 05-Jan-2026 | **31-Mar-2026** | HIRE | Store Associate | Mumbai |
| 1a (new) | **01-Apr-2026** | 30-Jun-2026 | TRANSFER | Store Associate | **Pune** |
| 2 | 01-Jul-2026 | 31-Dec-4712 | PROMOTION | Senior Store Associate | **Mumbai ⚠️?** |

**The key question: what about row 2?** The promotion row was created back when Rahul was still "in Mumbai," so it holds Mumbai. Fusion warns you that **future-dated changes exist**. Depending on the change and the setup, the new value may or may not carry forward into the later row. **Always open the later row and check.** If it still says Mumbai, correct it to Pune.

**Correct final history:**

| Row | Start | End | Action | Job | Location |
| --- | --- | --- | --- | --- | --- |
| 1 | 05-Jan-2026 | 31-Mar-2026 | HIRE | Store Associate | Mumbai |
| 1a | 01-Apr-2026 | 30-Jun-2026 | TRANSFER | Store Associate | Pune |
| 2 | 01-Jul-2026 | 31-Dec-4712 | PROMOTION | Senior Store Associate | Pune |

**Knock-on effects of any back-dated change:**

| Area | What happens |
| --- | --- |
| Payroll | April–August payrolls already ran on Mumbai rules. If location drives pay (city allowance, tax), payroll calculates **retro pay** or recovery next run |
| Approvals / workflow | Notifications and approvals already sent with the old data aren't re-triggered |
| Reports | Any report saved for April–August now gives different answers when re-run |
| Absence / time | Location-based calendars and holidays for April–August may need recalculation |

**Interview line:** *"A back-dated update splits the row that covers the effective date. I always check the future-dated rows afterwards to make sure the new value flowed through, and I flag it to payroll because it triggers retro."*

---

## 8. Hire an Employee

**Rahul is hired on 05-Jan-2026 into Nike India.**

### What gets created (all in one transaction)

| # | Table | Row created | Key values |
| --- | --- | --- | --- |
| 1 | `PER_PERSONS` | 1 (non-DE) | PERSON_ID 300100, DOB |
| 2 | `PER_ALL_PEOPLE_F` | 1 | PERSON_NUMBER 10045, start 05-Jan-2026 |
| 3 | `PER_PERSON_NAMES_F` | 1 | Rahul Mehta |
| 4 | `PER_NATIONAL_IDENTIFIERS`, `PER_EMAIL_ADDRESSES`, `PER_PHONES`, `PER_ADDRESSES_F` | 1 each | PAN, email, phone, address |
| 5 | `PER_PERIODS_OF_SERVICE` | 1 | DATE_START 05-Jan-2026, LEGAL_ENTITY 1001, PERIOD_TYPE E |
| 6 | `PER_ALL_ASSIGNMENTS_M` | 1 (two-tier) or 2 (three-tier: ET + E) | ACTION_CODE HIRE, ASSIGNMENT_STATUS_TYPE ACTIVE |
| 7 | `PER_PERSON_TYPE_USAGES_M` | 1 | Employee |
| 8 | `PER_ASSIGNMENT_SUPERVISORS_F` | 1 | Manager = Vikram |
| 9 | `CMP_SALARY` | 1 | DATE_FROM 05-Jan-2026, salary amount |

### Hire edge cases: quick lookup

| Situation | What goes wrong | Right move |
| --- | --- | --- |
| Duplicate check finds an ex-employee Rahul with the same DOB and PAN | Creating a new person splits his history across 2 person numbers | Pick the existing person. It's a **Rehire** |
| Hire entered in December for 05-Jan | Rahul exists but isn't active. Many actions are unavailable | Fine. Or use **Pending Worker** and convert on the start date |
| Hire entered in February for 05-Jan (back-dated) | January payroll already ran | Payroll picks it up as **retro pay** |
| Rahul is already an active employee in Nike India | — | Fusion blocks a second employee WR. Add an assignment instead |
| Position's headcount / FTE is full | Warning or error (depends on config) | Increase headcount or choose another position |
| Hire date keyed wrong (should be 12-Jan) | — | **Change Start Date** on the WR (hard after payroll runs or later changes) |
| Rahul never turns up (no-show) | Termination would leave a fake employment history | **Cancel Work Relationship** (usually blocked once payroll has processed) |
| 300 seasonal store staff before a holiday sale | UI is too slow | **HCM Data Loader** with `Worker.dat` |

---

## 9. Rehire an Employee

**Rahul resigned (last day 28-Feb-2027) and is rehired on 01-Aug-2027 as Assistant Store Manager.**

| What | Before rehire | After rehire |
| --- | --- | --- |
| PERSON_ID / Person Number | 300100 / 10045 | **Same**: 300100 / 10045 |
| Personal data (name, DOB, PAN) | Exists | **Reused**, HR just reviews it and updates the address or phone |
| Work relationship | 500001 (ended 28-Feb-2027) | **New**: 500002 starting 01-Aug-2027 |
| Assignment | E10045 (inactive) | **New**: E10045-3 (old one stays in history) |
| Action code | — | **REHIRE** |

**PER_ALL_ASSIGNMENTS_M: Rahul's full assignment story**

| ASSIGNMENT_NUMBER | WR | Start | End | ACTION_CODE | ASSIGNMENT_STATUS_TYPE | JOB |
| --- | --- | --- | --- | --- | --- | --- |
| E10045 | 500001 | 05-Jan-2026 | 30-Jun-2026 | HIRE | ACTIVE | Store Associate |
| E10045 | 500001 | 01-Jul-2026 | 28-Feb-2027 | PROMOTION | ACTIVE | Senior Store Associate |
| E10045 | 500001 | 01-Mar-2027 | 31-Dec-4712 | RESIGNATION | **INACTIVE** | Senior Store Associate |
| E10045-3 | 500002 | 01-Aug-2027 | 31-Dec-4712 | **REHIRE** | ACTIVE | Assistant Store Manager |

### Rehire edge cases

| Situation | Outcome |
| --- | --- |
| Rehire Recommendation was set to **No** at exit | Fusion warns during rehire. Whether it blocks or only warns depends on setup |
| Try rehire on 15-Feb-2027 (old WR still active) | Not allowed. Earliest possible is **01-Mar-2027** (day after termination) |
| Resigned Monday, changed mind Wednesday, before the last day | **Reverse Termination**, not rehire |
| Rehired into **Nike USA** instead | Still a rehire (new WR with Nike USA). **Not** a global transfer, because there's no active WR to move from |
| Seniority | Old service counts only if the seniority rules are configured that way. Don't assume |
| HR ignored the duplicate check | Two person numbers, split history. Painful clean-up |
| Payroll relationship | May be reused or newly created depending on country rules |

---

## 10. Transfer (same legal employer)

**Rahul moves from the Mumbai store to the Pune store on 01-Oct-2026.** Same WR, same assignment, just a new row.

| Row | Start | End | ACTION_CODE | DEPARTMENT | LOCATION | MANAGER |
| --- | --- | --- | --- | --- | --- | --- |
| 2 | 01-Jul-2026 | 30-Sep-2026 | PROMOTION | Retail Mumbai | Mumbai | Vikram |
| 3 | 01-Oct-2026 | 31-Dec-4712 | TRANSFER | Retail Pune | Pune | Anita |

`PER_PERIODS_OF_SERVICE` doesn't change at all. The legal employer is the same.

### Transfer edge cases

| Situation | What to do |
| --- | --- |
| Transfer approved for 01-Oct. In September HR back-dates a salary change to 01-Sep | Fusion inserts the change before the transfer. **Check the 01-Oct row** so the new salary carried forward |
| Transfer still pending approval and another change comes in | May be blocked or warned. Approve or withdraw the first one |
| Position-based setup, target position full | Headcount / FTE warning or error |
| Transfer entered as a **Correction** | History now says Rahul was **always** in Pune. ❌ Always use **Update** with the real date |

---

## 11. Global Transfer (different legal employer)

**Sarah moves permanently from Nike USA to Nike UK on 01-Apr-2027.**

**PER_PERIODS_OF_SERVICE: before**

| PERIOD_OF_SERVICE_ID | LEGAL_ENTITY | DATE_START | ACTUAL_TERMINATION_DATE | PRIMARY_FLAG |
| --- | --- | --- | --- | --- |
| 510001 | Nike USA | 01-Jun-2024 | *(null)* | Y |

**PER_PERIODS_OF_SERVICE: after**

| PERIOD_OF_SERVICE_ID | LEGAL_ENTITY | DATE_START | ACTUAL_TERMINATION_DATE | PRIMARY_FLAG |
| --- | --- | --- | --- | --- |
| 510001 | Nike USA | 01-Jun-2024 | **31-Mar-2027** | N |
| 510003 | Nike UK | **01-Apr-2027** | *(null)* | **Y** |

**PER_ALL_ASSIGNMENTS_M: after**

| ASSIGNMENT_NUMBER | WR | Start | End | ACTION_CODE | STATUS |
| --- | --- | --- | --- | --- | --- |
| E10078 (Software Engineer) | 510001 | … | 31-Mar-2027 | … | ACTIVE |
| E10078 | 510001 | 01-Apr-2027 | 31-Dec-4712 | GLB_TRANSFER | INACTIVE |
| E10078-2 (Technical Trainer) | 510001 | 01-Apr-2027 | 31-Dec-4712 | GLB_TRANSFER | INACTIVE ← **both** USA assignments end |
| E10078-4 (Software Engineer UK) | 510003 | 01-Apr-2027 | 31-Dec-4712 | GLB_TRANSFER | ACTIVE |

### Transfer vs Global Transfer vs Rehire

| | Transfer | Global Transfer | Rehire |
| --- | --- | --- | --- |
| Legal employer changes? | No | Yes | Same or different |
| Active WR before? | Yes | Yes (it gets ended) | **No** (person already left) |
| New WR? | No | Yes | Yes |
| New assignment? | No (new row only) | Yes | Yes |
| Person number | Same | Same | Same |

### Global transfer edge cases

| Situation | Impact |
| --- | --- |
| UK needs a National Insurance number and UK address format | **Most common error.** Missing legislative data blocks submission |
| Absence balances, benefits, payroll element entries, seniority | **Don't move automatically.** Each needs rules or manual action |
| Sarah had 2 assignments in Nike USA | **All** of them end with the old WR |

---

## 12. Termination

**Vikram (Store Manager, Nike India, 4 direct reports including Rahul) is terminated with a termination date of 30-Nov-2026.**

**PER_PERIODS_OF_SERVICE**

| PERIOD_OF_SERVICE_ID | DATE_START | NOTIFIED_TERMINATION_DATE | ACTUAL_TERMINATION_DATE |
| --- | --- | --- | --- |
| 500300 | 10-Mar-2022 | 01-Nov-2026 | **30-Nov-2026** |

**PER_ALL_ASSIGNMENTS_M**

| Start | End | ACTION_CODE | ASSIGNMENT_STATUS_TYPE |
| --- | --- | --- | --- |
| 01-Apr-2025 | **30-Nov-2026** | PROMOTION | ACTIVE |
| **01-Dec-2026** | 31-Dec-4712 | TERMINATION | **INACTIVE** |

**What to notice:** Vikram is still active **on** 30-Nov (his last working day). The inactive row starts the **next day**. The person record (`PER_ALL_PEOPLE_F`) is untouched.

### The five related actions

| Action | What it does | Vikram example |
| --- | --- | --- |
| Terminate Employment | Ends the whole WR | Vikram leaves Nike India |
| End Assignment | Ends one assignment only | Vikram drops a side "Regional Trainer" assignment but stays Store Manager |
| Correct Termination | Fixes the date or reason | Date should have been 15-Dec, not 30-Nov |
| Reverse Termination | Undoes it fully, WR active again | Terminated the wrong Vikram by mistake |
| Cancel Work Relationship | Deletes the WR as if it never existed | A new hire who never showed up (not for Vikram, who actually worked) |

### Termination edge cases

| Situation | What happens / what to do |
| --- | --- |
| A future transfer exists dated 01-Jan-2027 | Blocks the termination. Remove the future change first |
| Vikram has direct reports | Fusion asks you to **reassign** them. Skip it and approvals route to an ex-employee |
| His primary WR is terminated but another WR is active | The other WR must become primary |
| Pending promotion for Vikram | Withdraw it first, or termination can fail or behave oddly |
| Back-dated termination after November payroll ran | He was overpaid for days after his termination date. Payroll recovers it |
| Reverse after he was rehired into the same LE | Not possible |
| Termination date earlier than hire date | Not allowed (shows up in HDL loads) |
| Death | Use the **Death** action. Date of death goes on the person. Benefits and final pay follow special rules |

---

## 13. Resignation

A resignation is a **voluntary** termination started by the employee. The rows look the same as a termination, but `ACTION_CODE` = RESIGNATION.

**Rahul resigns. He submits on 29-Jan-2027 with a last day of 28-Feb-2027.**

| Step | Who | Date | What happens |
| --- | --- | --- | --- |
| 1 | Rahul | 29-Jan-2027 | Me → Resign. Enters last day, reason "Better Opportunity", attaches letter |
| 2 | Store Manager → HR | 30-Jan to 02-Feb | Approval chain (as configured) |
| 3 | System | 02-Feb-2027 | Termination recorded: `NOTIFIED_TERMINATION_DATE` 29-Jan, `ACTUAL_TERMINATION_DATE` 28-Feb. Offboarding checklist starts (return uniform, exit interview, final settlement) |
| 4 | Rahul | Until 28-Feb-2027 | Still **active**. Works his notice period |
| 5 | System | 01-Mar-2027 | Inactive assignment row begins. Access revoked as configured |

### Withdrawing a resignation (completes your notes)

| When Rahul changes his mind | What HR does | Result |
| --- | --- | --- |
| **Before approval** | Rahul or HR **withdraws** the pending transaction | Nothing was ever recorded |
| **After approval, before 28-Feb** (still working) | **Reverse Termination** (Redwood may show this as withdrawing the resignation) | Termination is removed. WR active again, no gap in history |
| **After 28-Feb** (he has actually left) | Too late to reverse cleanly → **Rehire** | New WR, new assignment |
| Wants to stay longer, e.g. until 31-Mar | **Correct Termination** to move the date | Same WR, new termination date |

> Rehire isn't the right tool for an employee who never actually left. Reverse termination keeps the history clean.

---

## 14. One-page cheat sheet: "What action do I use?"

| The situation | Action |
| --- | --- |
| Brand-new person joins | Hire |
| Person exists (ex-employee) and joins again | Rehire |
| Accepted offer, not joined yet | Add Pending Worker → Convert |
| Different department / location / manager, **same** legal employer | Transfer (Update) |
| Permanently moves to a **different** legal employer | Global Transfer |
| Works for two legal employers at once | Add concurrent work relationship |
| Takes on a second job, same employer | Add Assignment |
| Drops one of several jobs | End Assignment |
| Leaves the company (company-initiated) | Terminate |
| Leaves the company (employee-initiated) | Resignation |
| Termination date or reason was wrong | Correct Termination |
| Terminated by mistake / withdrew resignation before leaving | Reverse Termination |
| Hired by mistake / never joined | Cancel Work Relationship |
| A real change happened from a date | Update |
| Data was wrong from the start | Correction |

---

## 15. Interview practice questions

**Q1. Rahul left Nike India in 2027 and is rejoining in 2029. Does he get a new person number?**
No. Same person, same person number. He gets a new work relationship (new `PERIOD_OF_SERVICE_ID`) and a new assignment.

**Q2. HR wants to change Sarah's legal employer from Nike USA to Nike UK by editing her assignment. Why won't that work?**
The legal employer belongs to the work relationship (`PER_PERIODS_OF_SERVICE.LEGAL_ENTITY_ID`), not the assignment. Use Global Transfer, which ends the USA WR on 31-Mar and creates a UK WR on 01-Apr.

**Q3. Your headcount report shows Rahul twice for 01-Oct-2026. What's the likely bug?**
Missing `EFFECTIVE_LATEST_CHANGE = 'Y'`. Two changes happened that day, so there are two rows with the same dates. (In three-tier, also check for a missing `ASSIGNMENT_TYPE` filter that lets ET rows through.)

**Q4. Grade was keyed as R1 at hire but should have been R2 from day one. Update or Correction? And what else must you check?**
Correction, because it was wrong from the start. Then check every **later** row: they copied R1 when they were created and may also need correcting.

**Q5. A new joiner never turns up. HR terminates him. What's wrong?**
Termination leaves a fake employment history. Use Cancel Work Relationship (if payroll hasn't processed him).

**Q6. On what date does a terminated employee's assignment become inactive?**
The day **after** `ACTUAL_TERMINATION_DATE`. The employee is still active on their last day.

**Q7. Sarah has 2 assignments in Nike USA and is globally transferred. What happens to the Technical Trainer assignment?**
It ends along with the USA work relationship. All assignments under the old WR end.

**Q8. Which of these keep history: name, DOB, address, email, salary?**
Name ✅ (`PER_PERSON_NAMES_F`), address ✅ (`PER_ADDRESSES_F`), salary ✅ (`CMP_SALARY`). DOB ❌ (`PER_PERSONS`), email ❌ (`PER_EMAIL_ADDRESSES`).

**Q9. A location change is back-dated to 01-Apr, but a promotion row exists from 01-Jul. What do you check?**
The 01-Jul row. It may still hold the old location. Verify it and correct it if needed, then tell payroll, because it triggers retro.

**Q10. An intern is placed by a university that pays the stipend. Worker type?**
Contingent Worker (user person type "Intern"), because Nike's payroll doesn't pay them.

---

## 16. Bonus: starter SQL for BI Publisher (ties it all together)

Current, active, primary employee assignment for everyone in Nike India, as of today:

```sql
SELECT  papf.person_number,
        ppnf.display_name,
        ppos.date_start            AS hire_date,
        paam.assignment_number,
        paam.action_code,
        paam.assignment_status_type
FROM    per_all_people_f        papf
JOIN    per_person_names_f      ppnf ON ppnf.person_id = papf.person_id
                                    AND ppnf.name_type = 'GLOBAL'
JOIN    per_periods_of_service  ppos ON ppos.person_id = papf.person_id
JOIN    per_all_assignments_m   paam ON paam.period_of_service_id = ppos.period_of_service_id
WHERE   TRUNC(SYSDATE) BETWEEN papf.effective_start_date AND papf.effective_end_date   -- _F: as-of date
AND     TRUNC(SYSDATE) BETWEEN ppnf.effective_start_date AND ppnf.effective_end_date   -- _F: as-of date
AND     TRUNC(SYSDATE) BETWEEN paam.effective_start_date AND paam.effective_end_date   -- _M: as-of date
AND     paam.effective_latest_change = 'Y'          -- _M: avoid same-day duplicates
AND     paam.assignment_type         = 'E'          -- skip ET/CT terms rows (three-tier)
AND     paam.primary_flag            = 'Y'          -- one row per person
AND     paam.assignment_status_type  = 'ACTIVE'     -- exclude terminated
AND     ppos.legal_entity_id         = :p_legal_entity_id
AND     ppos.actual_termination_date IS NULL;       -- open work relationship
```