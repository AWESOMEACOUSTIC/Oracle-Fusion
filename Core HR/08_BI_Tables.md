# BI Publisher for Oracle HCM: Phase 2, HCM Data Knowledge

> **Goal of this phase:** Know *which tables* hold HCM data, *how they connect*, and *how to pick the correct row* from date-tracked tables. Most BIP defects in HCM projects (duplicate rows, missing employees, wrong job or department) come from this phase, not from BIP itself.

**Topics in this phase**
7. Core HCM tables and how they join
8. Date-effectivity (`_F` vs `_M` tables) and picking the primary/latest assignment
9. Multi-language columns (`_TL` tables and `_VL` views)
10. Lookups (codes to meanings)
11. Flexfields (DFF, EFF, KFF)
12. Data security (secured list views)

---

## 7. Core HCM Tables

### How HCM data is organized
In Oracle Fusion HCM, a worker's data is spread across several layers:

```
PERSON  (who the person is: number, name, birth date, email, address)
   │
   └── WORK RELATIONSHIP / PERIOD OF SERVICE  (hire date, termination date, legal employer)
          │
          └── ASSIGNMENT  (the actual job: department, job, position, grade, location, manager, status)
```

- **Person** data stays the same no matter how many jobs the person has.
- **Work relationship** (period of service) records when they joined and left a legal employer.
- **Assignment** holds the job details. One person can have more than one assignment, so the **primary** one must be identified.

### Core tables to know

#### Person-level tables

| Table | What it holds | Key columns |
|---|---|---|
| **PER_ALL_PEOPLE_F** | Main person record (date-tracked) | `PERSON_ID`, `PERSON_NUMBER`, `EFFECTIVE_START_DATE`, `EFFECTIVE_END_DATE` |
| **PER_PERSONS** | Person data that never changes over time (not date-tracked) | `PERSON_ID`, `DATE_OF_BIRTH`, `COUNTRY_OF_BIRTH` |
| **PER_PERSON_NAMES_F** | Names (date-tracked) | `PERSON_ID`, `FIRST_NAME`, `LAST_NAME`, `FULL_NAME`, `DISPLAY_NAME`, `NAME_TYPE` |
| **PER_PEOPLE_LEGISLATIVE_F** | Country-specific personal data | `PERSON_ID`, `SEX`, `MARITAL_STATUS`, `LEGISLATION_CODE` |
| **PER_EMAIL_ADDRESSES** | Email addresses | `PERSON_ID`, `EMAIL_ADDRESS`, `EMAIL_TYPE`, `DATE_FROM`, `DATE_TO` |
| **PER_ADDRESSES_F** + **PER_PERSON_ADDR_USAGES_F** | Addresses, and which type each is (home, mailing) | `ADDRESS_ID`, `PERSON_ID`, `ADDRESS_TYPE` |
| **PER_PHONES** | Phone numbers | `PERSON_ID`, `PHONE_TYPE`, `PHONE_NUMBER` |
| **PER_NATIONAL_IDENTIFIERS** | National IDs (e.g., PAN, SSN) | `PERSON_ID`, `NATIONAL_IDENTIFIER_TYPE`, `NATIONAL_IDENTIFIER_NUMBER` |
| **PER_USERS** | Login user linked to a person | `USERNAME`, `PERSON_ID` |

**Remember about names:**
- `PER_PERSON_NAMES_F` can have more than one name row per person (global name and local name).
- Always filter **`NAME_TYPE = 'GLOBAL'`**, or you get duplicate rows.

**Remember about email:**
- `EMAIL_TYPE = 'W1'` is the **work email**.
- This table uses `DATE_FROM` / `DATE_TO`, not effective dates, and `DATE_TO` can be NULL.

#### Work relationship table

| Table | What it holds | Key columns |
|---|---|---|
| **PER_PERIODS_OF_SERVICE** | Work relationship: hire and termination (not date-tracked) | `PERIOD_OF_SERVICE_ID`, `PERSON_ID`, `DATE_START` (hire date), `ACTUAL_TERMINATION_DATE`, `LEGAL_ENTITY_ID`, `PRIMARY_FLAG` |

- **New hires report:** filter on `DATE_START`.
- **Terminations report:** filter on `ACTUAL_TERMINATION_DATE`.
- A rehired person has **more than one** period of service row.

#### Assignment-level tables

| Table | What it holds | Key columns |
|---|---|---|
| **PER_ALL_ASSIGNMENTS_M** | The assignment (job details) | `ASSIGNMENT_ID`, `ASSIGNMENT_NUMBER`, `PERSON_ID`, `PERIOD_OF_SERVICE_ID`, `ASSIGNMENT_TYPE`, `PRIMARY_FLAG`, `ASSIGNMENT_STATUS_TYPE`, `ORGANIZATION_ID` (department), `JOB_ID`, `POSITION_ID`, `GRADE_ID`, `LOCATION_ID`, `BUSINESS_UNIT_ID`, `LEGAL_ENTITY_ID`, `EFFECTIVE_SEQUENCE`, `EFFECTIVE_LATEST_CHANGE` |
| **PER_ASSIGNMENT_SUPERVISORS_F** | Manager of an assignment | `ASSIGNMENT_ID`, `MANAGER_ID` (manager's person ID), `MANAGER_ASSIGNMENT_ID`, `MANAGER_TYPE` (`'LINE_MANAGER'`) |
| **PER_PERSON_TYPE_USAGES_M** | Person types (employee, contingent worker) | `PERSON_ID`, `PERSON_TYPE_ID` |
| **CMP_SALARY** | Salary | `ASSIGNMENT_ID`, `SALARY_AMOUNT`, `DATE_FROM`, `DATE_TO` |

#### Work structure tables (the "names" behind the IDs on the assignment)

| Table | Holds | Name comes from |
|---|---|---|
| **HR_ALL_ORGANIZATION_UNITS_F** | Departments, business units, legal entities (all are "organizations") | **HR_ORGANIZATION_UNITS_F_TL** (`NAME`) |
| **PER_JOBS_F** | Jobs (`JOB_CODE`) | **PER_JOBS_F_TL** (`NAME`) |
| **HR_ALL_POSITIONS_F** | Positions | **HR_ALL_POSITIONS_F_TL** (`NAME`) |
| **PER_GRADES_F** | Grades | **PER_GRADES_F_TL** (`NAME`) |
| **HR_LOCATIONS_ALL_F_VL** | Locations | `LOCATION_NAME`, `LOCATION_CODE` |

#### Payroll and absence tables (awareness level)

| Area | Tables |
|---|---|
| Elements (earnings and deductions) | `PAY_ELEMENT_TYPES_F`, `PAY_INPUT_VALUES_F` |
| Element entries (what an employee gets) | `PAY_ELEMENT_ENTRIES_F`, `PAY_ELEMENT_ENTRY_VALUES_F` |
| Payroll run results | `PAY_PAYROLL_ACTIONS`, `PAY_PAYROLL_REL_ACTIONS`, `PAY_RUN_RESULTS`, `PAY_RUN_RESULT_VALUES` |
| Absences | `ANC_PER_ABS_ENTRIES` |

### Join map (how the main tables connect)

```
PER_ALL_PEOPLE_F (papf)
 ├── PER_PERSON_NAMES_F (ppnf)          ON ppnf.person_id = papf.person_id  AND ppnf.name_type = 'GLOBAL'
 ├── PER_PERSONS (pp)                   ON pp.person_id   = papf.person_id
 ├── PER_EMAIL_ADDRESSES (pea)          ON pea.person_id  = papf.person_id  AND pea.email_type = 'W1'
 ├── PER_PERIODS_OF_SERVICE (ppos)      ON ppos.person_id = papf.person_id
 └── PER_ALL_ASSIGNMENTS_M (paam)       ON paam.person_id = papf.person_id
        ├── PER_ASSIGNMENT_SUPERVISORS_F (pasf)  ON pasf.assignment_id   = paam.assignment_id
        ├── HR_ORGANIZATION_UNITS_F_TL (dept)    ON dept.organization_id = paam.organization_id
        ├── PER_JOBS_F_TL (job)                  ON job.job_id           = paam.job_id
        ├── HR_ALL_POSITIONS_F_TL (pos)          ON pos.position_id      = paam.position_id
        ├── PER_GRADES_F_TL (grd)                ON grd.grade_id         = paam.grade_id
        ├── HR_LOCATIONS_ALL_F_VL (loc)          ON loc.location_id      = paam.location_id
        └── CMP_SALARY (cs)                      ON cs.assignment_id     = paam.assignment_id
```

### Standard table aliases (used across projects; use them in interviews too)
`papf` = PER_ALL_PEOPLE_F · `ppnf` = PER_PERSON_NAMES_F · `paam` = PER_ALL_ASSIGNMENTS_M · `ppos` = PER_PERIODS_OF_SERVICE · `pasf` = PER_ASSIGNMENT_SUPERVISORS_F · `pea` = PER_EMAIL_ADDRESSES

### How to find the right table when you don't know it
1. **Oracle documentation:** "Tables and Views for HCM" lists every table and its columns for your release.
2. **OTBI physical SQL:** Build an OTBI analysis with the fields you need, run it, then use **Administration → Manage Sessions → View Log**. The log shows the SQL OTBI generated, including the tables it used.
3. **Seeded data models:** Open a seeded BIP data model on a similar topic and read its SQL. Don't change it.

> **Interview-ready line:**
> "Person data is in PER_ALL_PEOPLE_F and PER_PERSON_NAMES_F, the work relationship with hire and termination dates is in PER_PERIODS_OF_SERVICE, and job details are in PER_ALL_ASSIGNMENTS_M. The assignment holds IDs for department, job, position, grade and location, and their names come from the related _TL tables. The manager comes from PER_ASSIGNMENT_SUPERVISORS_F."

---

## 8. Date-Effectivity ⭐ (the most important topic in this phase)

### What is date-effectivity?
Fusion HCM **keeps history**. When an employee's job, department, or name changes, Oracle **doesn't overwrite the old row**. It **end-dates** the old row and **creates a new row** from the change date.

Each row has:
- **`EFFECTIVE_START_DATE`**: the date this version became valid
- **`EFFECTIVE_END_DATE`**: the date this version stopped being valid

The **current** row usually has `EFFECTIVE_END_DATE = 31-Dec-4712`, which Oracle uses to mean "no end date."

### Example: one employee's assignment history

| ASSIGNMENT_ID | JOB | EFFECTIVE_START_DATE | EFFECTIVE_END_DATE |
|---|---|---|---|
| 300100 | Software Engineer | 01-Aug-2024 | 31-Mar-2026 |
| 300100 | Senior Software Engineer | 01-Apr-2026 | 31-Dec-4712 |

- Query as of **15-Jan-2026** → returns *Software Engineer*.
- Query as of **today** → returns *Senior Software Engineer*.
- Query **without a date filter** → returns **both rows**, so the employee appears twice. This is a duplicate-row bug.

### Table suffixes and what they mean

| Suffix | Meaning | Example | How to filter |
|---|---|---|---|
| **`_F`** | Date-tracked. At most **one row per day** per record. | `PER_ALL_PEOPLE_F`, `PER_PERSON_NAMES_F`, `PER_JOBS_F` | `date BETWEEN effective_start_date AND effective_end_date` |
| **`_M`** | Date-tracked with **multiple changes on the same day** allowed. | `PER_ALL_ASSIGNMENTS_M`, `PER_PERSON_TYPE_USAGES_M` | Date filter **plus** `effective_latest_change = 'Y'` |
| **No suffix** | Not date-tracked. One row (or uses its own from/to dates). | `PER_PERSONS`, `PER_PERIODS_OF_SERVICE`, `PER_EMAIL_ADDRESSES` | Join directly, or use that table's own date columns |
| **`_TL`** | Translated names (see Topic 9) | `PER_JOBS_F_TL` | Language filter (+ date filter if it's `_F_TL`) |

### Why `_M` tables need extra care
On `_M` tables, if an assignment is changed **twice on the same day** (e.g., a transfer, then a correction), both rows have the **same effective start date**.
- **`EFFECTIVE_SEQUENCE`** numbers the changes on that day (1, 2, 3 …).
- **`EFFECTIVE_LATEST_CHANGE = 'Y'`** marks the **final** change of that day.

So the date filter alone can still return 2 rows. Always add:
```sql
AND paam.effective_latest_change = 'Y'
```

### The golden rule ⭐
> **Every date-tracked table (`_F`, `_M`, `_F_TL`) in your query must have its own date filter.**
> If even one table is missing the filter, you can get duplicate rows.

```sql
AND TRUNC(SYSDATE) BETWEEN papf.effective_start_date AND papf.effective_end_date
AND TRUNC(SYSDATE) BETWEEN ppnf.effective_start_date AND ppnf.effective_end_date
AND TRUNC(SYSDATE) BETWEEN paam.effective_start_date AND paam.effective_end_date
AND TRUNC(SYSDATE) BETWEEN dept.effective_start_date AND dept.effective_end_date
AND TRUNC(SYSDATE) BETWEEN job.effective_start_date  AND job.effective_end_date
```
- `TRUNC(SYSDATE)` removes the time part, so the comparison is on the date only.
- In real reports, `TRUNC(SYSDATE)` is usually replaced by a parameter such as `:P_EFFECTIVE_DATE`, so users can run the report "as of" any date. That is covered in Phase 3.

### Choosing the correct assignment row ⭐
A person can have several assignment records, so filter down to the one you need:

| Filter | Why |
|---|---|
| `paam.assignment_type IN ('E','C')` | `E` = Employee, `C` = Contingent Worker. Excludes terms records (`ET`, `CT`), pending workers (`P`), and nonworkers (`N`). |
| `paam.primary_flag = 'Y'` | Picks the **primary** assignment when a person has more than one. |
| `paam.assignment_status_type = 'ACTIVE'` | Only active assignments (other values include `INACTIVE` and `SUSPENDED`). |
| `paam.effective_latest_change = 'Y'` | Last change of the day (`_M` table rule). |
| Date filter on `paam` | The version valid on the report date. |

### Using the right date for the right report

| Report type | Which date to use |
|---|---|
| Active employees today | `TRUNC(SYSDATE)` (or the effective-date parameter) |
| Active employees as of a past date | That date (the parameter) |
| New hires between two dates | `ppos.date_start BETWEEN :from AND :to`; take the assignment as of `ppos.date_start` |
| Terminations between two dates | `ppos.actual_termination_date BETWEEN :from AND :to`; take the assignment as of `ppos.actual_termination_date` (their last working day) |

For hire and termination reports, using `SYSDATE` on every table is a common mistake. A terminated employee's details as of today may not match what they were on their last day, and an active-status filter would remove them completely. **Use the event date for the event.**

### Future-dated changes
- HR can enter a change **today** that becomes effective **next month** (e.g., a promotion from 01-Dec).
- Picking "the row with `EFFECTIVE_END_DATE = 4712-12-31`" would return **next month's** job, which is wrong for a "today" report.
- **Always use `date BETWEEN start AND end`**, not "latest row," unless the requirement says otherwise.

### Example: Active employee list (current date)

```sql
SELECT papf.person_number,
       ppnf.full_name,
       paam.assignment_number,
       dept.name               AS department_name,
       job.name                AS job_name,
       loc.location_name,
       pea.email_address       AS work_email,
       mgr.full_name           AS manager_name
FROM   per_all_people_f             papf
JOIN   per_person_names_f           ppnf
       ON  ppnf.person_id = papf.person_id
       AND ppnf.name_type = 'GLOBAL'
       AND TRUNC(SYSDATE) BETWEEN ppnf.effective_start_date AND ppnf.effective_end_date
JOIN   per_all_assignments_m        paam
       ON  paam.person_id = papf.person_id
       AND paam.assignment_type IN ('E','C')
       AND paam.primary_flag = 'Y'
       AND paam.assignment_status_type = 'ACTIVE'
       AND paam.effective_latest_change = 'Y'
       AND TRUNC(SYSDATE) BETWEEN paam.effective_start_date AND paam.effective_end_date
LEFT JOIN hr_organization_units_f_tl dept
       ON  dept.organization_id = paam.organization_id
       AND dept.language = USERENV('LANG')
       AND TRUNC(SYSDATE) BETWEEN dept.effective_start_date AND dept.effective_end_date
LEFT JOIN per_jobs_f_tl             job
       ON  job.job_id = paam.job_id
       AND job.language = USERENV('LANG')
       AND TRUNC(SYSDATE) BETWEEN job.effective_start_date AND job.effective_end_date
LEFT JOIN hr_locations_all_f_vl     loc
       ON  loc.location_id = paam.location_id
       AND TRUNC(SYSDATE) BETWEEN loc.effective_start_date AND loc.effective_end_date
LEFT JOIN per_email_addresses       pea
       ON  pea.person_id = papf.person_id
       AND pea.email_type = 'W1'
       AND TRUNC(SYSDATE) BETWEEN pea.date_from AND NVL(pea.date_to, DATE '4712-12-31')
LEFT JOIN per_assignment_supervisors_f pasf
       ON  pasf.assignment_id = paam.assignment_id
       AND pasf.manager_type  = 'LINE_MANAGER'
       AND TRUNC(SYSDATE) BETWEEN pasf.effective_start_date AND pasf.effective_end_date
LEFT JOIN per_person_names_f        mgr
       ON  mgr.person_id = pasf.manager_id
       AND mgr.name_type = 'GLOBAL'
       AND TRUNC(SYSDATE) BETWEEN mgr.effective_start_date AND mgr.effective_end_date
WHERE  TRUNC(SYSDATE) BETWEEN papf.effective_start_date AND papf.effective_end_date
ORDER BY papf.person_number
```

**Why some joins are `LEFT JOIN`:** not every employee has a job, a location, a work email, or a manager. An inner join would **drop** those employees from the report entirely. Use `LEFT JOIN` for optional data.

**Why `NVL(pea.date_to, DATE '4712-12-31')`:** `DATE_TO` can be NULL (no end date). `BETWEEN` with a NULL always fails, so NVL replaces NULL with a far-future date.

### Example: Terminations between two dates

```sql
SELECT papf.person_number,
       ppnf.full_name,
       ppos.date_start               AS hire_date,
       ppos.actual_termination_date  AS termination_date,
       dept.name                     AS department_name
FROM   per_periods_of_service    ppos
JOIN   per_all_people_f          papf
       ON  papf.person_id = ppos.person_id
       AND ppos.actual_termination_date BETWEEN papf.effective_start_date AND papf.effective_end_date
JOIN   per_person_names_f        ppnf
       ON  ppnf.person_id = ppos.person_id
       AND ppnf.name_type = 'GLOBAL'
       AND ppos.actual_termination_date BETWEEN ppnf.effective_start_date AND ppnf.effective_end_date
JOIN   per_all_assignments_m     paam
       ON  paam.period_of_service_id = ppos.period_of_service_id
       AND paam.assignment_type = 'E'
       AND paam.primary_flag = 'Y'
       AND paam.effective_latest_change = 'Y'
       AND ppos.actual_termination_date BETWEEN paam.effective_start_date AND paam.effective_end_date
LEFT JOIN hr_organization_units_f_tl dept
       ON  dept.organization_id = paam.organization_id
       AND dept.language = USERENV('LANG')
       AND ppos.actual_termination_date BETWEEN dept.effective_start_date AND dept.effective_end_date
WHERE  ppos.actual_termination_date BETWEEN :p_from_date AND :p_to_date
```

Note the differences from the active list:
- Every table is filtered **as of the termination date**, not `SYSDATE`.
- **No** `assignment_status_type = 'ACTIVE'` filter, because these people are leaving.
- The assignment is joined through **`period_of_service_id`**, so a rehired person's old and new jobs don't mix.

### Duplicate rows: checklist ⭐
When a report shows the same employee twice, check in this order:
1. Is a **date filter missing** on any `_F` / `_M` / `_F_TL` table?
2. Is **`effective_latest_change = 'Y'`** missing on an `_M` table?
3. Is **`name_type = 'GLOBAL'`** missing on names?
4. Is **`primary_flag = 'Y'`** missing on assignments?
5. Is the **language filter** missing on a `_TL` table?
6. Does the person have **multiple emails, phones, or addresses** with no type filter?
7. Is the person **rehired** (more than one period of service)?

> **Interview-ready line:**
> "HCM tables are date-tracked. Changes create new rows with effective start and end dates instead of overwriting old ones. In every query, each _F and _M table needs a filter like 'effective date BETWEEN effective_start_date AND effective_end_date'. For _M tables like PER_ALL_ASSIGNMENTS_M, multiple changes can happen on the same day, so we also filter effective_latest_change = 'Y'. To get the main job we add primary_flag = 'Y' and assignment_type IN ('E','C'). Missing any of these is the most common cause of duplicate rows."

---

## 9. Multi-Language Columns (`_TL` Tables and `_VL` Views)

### Why `_TL` tables exist
Fusion supports multiple languages. Names that users see (job names, department names, grade names) are stored in **translation tables**, with **one row per language**.

| Table type | Holds |
|---|---|
| **Base table** (e.g., `PER_JOBS_F`) | IDs, codes, dates, non-text data |
| **`_TL` table** (e.g., `PER_JOBS_F_TL`) | Translated text (`NAME`, `DESCRIPTION`), one row **per language** |
| **`_VL` view** (e.g., `HR_LOCATIONS_ALL_F_VL`) | Base table + `_TL` already joined for the **session language** |

### Key columns in a `_TL` table
- **`LANGUAGE`**: the language of this row (e.g., `US` for American English)
- **`SOURCE_LANG`**: the language the text was originally entered in

### The rule
If a client has 3 languages installed, a `_TL` table has **3 rows** for each job. Without a language filter, every employee appears **3 times**.

```sql
AND job.language = USERENV('LANG')
```
- `USERENV('LANG')` returns the language of the **current session** (the user running the report), so each user sees names in their own language.
- **`_F_TL`** tables (like `PER_JOBS_F_TL`) are **also date-tracked**, so they need **both** the language filter and the date filter.

### `_VL` views
- A `_VL` view already applies the session language, so you **don't** add a language filter.
- If the table is date-tracked, you **still** need the date filter.

> **Interview-ready line:**
> "_TL tables store translated text with one row per language, so we filter LANGUAGE = USERENV('LANG') to get one row in the user's session language. _VL views already combine the base and translated tables for the session language. If the table is also date-tracked, like PER_JOBS_F_TL, it needs the date filter too."

---

## 10. Lookups (Turning Codes into Meanings)

### What is a lookup?
Many columns store a **short code** instead of the full text. A **lookup** maps that code to its meaning.

| Column value (code) | Lookup type | Meaning shown to user |
|---|---|---|
| `M` | `SEX` | Male |
| `F` | `SEX` | Female |
| `M` | `MAR_STATUS` | Married |

- **`LOOKUP_TYPE`**: the group (e.g., `SEX`, `MAR_STATUS`)
- **`LOOKUP_CODE`**: the stored value (e.g., `M`)
- **`MEANING`**: the display text (e.g., Male)

### Lookup tables and views

| Object | Use |
|---|---|
| **`HR_LOOKUPS`** | View for HR lookups; returns the meaning in the session language. Simple to use. |
| **`FND_LOOKUP_VALUES`** | Base table for all lookups (all modules). Needs a `LANGUAGE` filter. |
| **`FND_LOOKUP_VALUES_VL`** | View over `FND_LOOKUP_VALUES` for the session language. |

### Example: show gender and marital status as text

```sql
SELECT papf.person_number,
       sex_lkp.meaning   AS gender,
       mar_lkp.meaning   AS marital_status
FROM   per_all_people_f          papf
JOIN   per_people_legislative_f  ppl
       ON  ppl.person_id = papf.person_id
       AND TRUNC(SYSDATE) BETWEEN ppl.effective_start_date AND ppl.effective_end_date
LEFT JOIN hr_lookups             sex_lkp
       ON  sex_lkp.lookup_type = 'SEX'
       AND sex_lkp.lookup_code = ppl.sex
LEFT JOIN hr_lookups             mar_lkp
       ON  mar_lkp.lookup_type = 'MAR_STATUS'
       AND mar_lkp.lookup_code = ppl.marital_status
WHERE  TRUNC(SYSDATE) BETWEEN papf.effective_start_date AND papf.effective_end_date
```

### Rules to remember
- **Always filter `LOOKUP_TYPE`** as well as the code. The same code (like `M`) exists in many lookup types; without the type you get wrong or duplicate rows.
- Use **`LEFT JOIN`** to lookups, because the code column may be NULL (e.g., marital status not entered).
- If you use `FND_LOOKUP_VALUES` directly, add `LANGUAGE = USERENV('LANG')`.
- To find the lookup type for a field: **Setup and Maintenance → Manage Common Lookups** (or Manage Lookups) and search by meaning.

> **Interview-ready line:**
> "Many HCM columns store codes, and lookups convert them to meanings. I join HR_LOOKUPS or FND_LOOKUP_VALUES on both LOOKUP_TYPE and LOOKUP_CODE, use a left join because the code can be null, and filter by language when using the base table."

---

## 11. Flexfields (DFF, EFF, KFF)

### What is a flexfield?
A **flexfield** lets a client **add their own fields** to Fusion without changing Oracle's table structure. Oracle provides **spare generic columns**, and the client's custom fields are **mapped** to those columns during setup.

As a BIP developer, you need to know **which generic column** holds **which custom field**.

### Three types

| Type | Full name | Purpose | Where data is stored |
|---|---|---|---|
| **DFF** | Descriptive Flexfield | Extra fields on an existing record (e.g., "Employee Badge Number" on a person) | Spare columns **on the same base table**: `ATTRIBUTE1…n`, `ATTRIBUTE_NUMBER1…n`, `ATTRIBUTE_DATE1…n`, and `ATTRIBUTE_CATEGORY` (context) |
| **EFF** | Extensible Flexfield | Groups of extra fields, can hold **multiple rows** per record (e.g., several certifications for one person) | **Separate extra-info tables**: e.g., `PER_PEOPLE_EXTRA_INFO_F` (`PEI_INFORMATION1…`), `PER_ASSIGNMENT_EXTRA_INFO_M` (`AEI_INFORMATION1…`), `HR_ORGANIZATION_INFORMATION_F` (`ORG_INFORMATION1…`) |
| **KFF** | Key Flexfield | A **structured code made of segments** (e.g., Company.Department.CostCenter) | Segment tables with `SEGMENT1…n` columns, e.g., People Group (`PAY_PEOPLE_GROUPS`), Cost Allocation (`PAY_COST_ALLOCATION_KEYFLEX`) |

### Key terms
- **Segment**: one custom field (e.g., "Badge Number").
- **Context**: a group of segments that applies only in certain cases (e.g., segments for India only). Stored in `ATTRIBUTE_CATEGORY` for DFFs and `INFORMATION_TYPE` for EFFs.
- **Value set**: the list of allowed values for a segment. The column may store an **ID or code**, not the display value.

### How to find which column holds a custom field
1. **Setup and Maintenance → Manage Descriptive Flexfields** (or **Manage Extensible Flexfields**).
2. Search for the flexfield (e.g., Person, Assignment).
3. Open the **context**, then the **segment**.
4. Note the **table column** (e.g., `ATTRIBUTE3`) and the **value set**.

### Example: read a DFF segment from the assignment
Suppose "Shift Type" is mapped to `ATTRIBUTE5` on assignments:

```sql
SELECT papf.person_number,
       paam.ass_attribute5   AS shift_type
FROM   per_all_assignments_m paam
JOIN   per_all_people_f      papf
       ON papf.person_id = paam.person_id
       AND TRUNC(SYSDATE) BETWEEN papf.effective_start_date AND papf.effective_end_date
WHERE  paam.primary_flag = 'Y'
AND    paam.effective_latest_change = 'Y'
AND    TRUNC(SYSDATE) BETWEEN paam.effective_start_date AND paam.effective_end_date
```
The assignment table's DFF columns are prefixed `ASS_ATTRIBUTE…`. Prefixes differ by table, so always check the column name in the flexfield setup or the table documentation.

### Example: read an EFF (Person Extra Information)

```sql
SELECT papf.person_number,
       pei.pei_information1  AS certification_name,
       pei.pei_information2  AS certification_date
FROM   per_people_extra_info_f pei
JOIN   per_all_people_f        papf
       ON papf.person_id = pei.person_id
       AND TRUNC(SYSDATE) BETWEEN papf.effective_start_date AND papf.effective_end_date
WHERE  pei.information_type = 'XX_CERTIFICATIONS'
AND    TRUNC(SYSDATE) BETWEEN pei.effective_start_date AND pei.effective_end_date
```
- **`INFORMATION_TYPE`** = the EFF **context** name. Always filter it, or you mix data from unrelated contexts.
- An EFF can return **multiple rows per person**, so expect that when designing the report (often a master–detail case, covered in Phase 3).

### Rules to remember
- **Always filter the context** (`ATTRIBUTE_CATEGORY` / `INFORMATION_TYPE`) when segments are context-specific.
- If a segment uses a **table-validated or independent value set**, the column may hold an ID or code. Join to the value set's source table (or `FND_FLEX_VALUES_VL` for independent value sets) to get the display value.
- Segment-to-column mappings **differ between clients**. Never assume `ATTRIBUTE1` means the same thing in two projects.

> **Interview-ready line:**
> "Flexfields let clients add custom fields. DFF values are stored in ATTRIBUTE columns on the base table, EFF values are in separate extra-information tables like PER_PEOPLE_EXTRA_INFO_F and can have multiple rows per record, and KFFs are multi-segment structured codes stored in SEGMENT columns. I find the column mapping in Manage Descriptive or Extensible Flexfields, filter by the context, and join the value set if the column stores an ID."

---

## 12. Data Security in BIP ⭐

### The problem
- In **OTBI**, security is applied **automatically**. A manager only sees their own team.
- In **BIP**, SQL on **base tables** like `PER_ALL_PEOPLE_F` returns **all employees in the company**, whoever runs the report. This can expose confidential data (salaries, personal details) to people who should not see it.

So **in BIP, the developer is responsible for applying data security inside the SQL.**

### How Fusion HCM security works (in short)
- Users get **data roles** with **security profiles** (e.g., "Person security profile: only my department").
- Oracle provides **secured list views** that return **only the records the logged-in user is allowed to see**, based on those security profiles.

### Secured list views

| View | Returns records the user can access for |
|---|---|
| **`PER_PERSON_SECURED_LIST_V`** | Persons |
| **`PER_ASSIGNMENT_SECURED_LIST_V`** | Assignments |
| **`PER_DEPARTMENT_SECURED_LIST_V`** | Departments |
| **`PER_POSITION_SECURED_LIST_V`** | Positions |
| **`PER_LEGAL_EMPL_SECURED_LIST_V`** | Legal employers |

### Example: secure an employee report

```sql
SELECT papf.person_number,
       ppnf.full_name
FROM   per_all_people_f    papf
JOIN   per_person_names_f  ppnf
       ON  ppnf.person_id = papf.person_id
       AND ppnf.name_type = 'GLOBAL'
       AND TRUNC(SYSDATE) BETWEEN ppnf.effective_start_date AND ppnf.effective_end_date
WHERE  TRUNC(SYSDATE) BETWEEN papf.effective_start_date AND papf.effective_end_date
AND    papf.person_id IN (SELECT psl.person_id
                          FROM   per_person_secured_list_v psl)
```
- The data model and catalog stay the same for everyone.
- **What each user sees changes** based on who is logged in.

### Restricting a report to "only my own data"
For self-service reports (e.g., an employee downloads their own letter), use the BIP system variable **`:xdo_user_name`**, which holds the **username of the person running the report**:

```sql
AND papf.person_id = (SELECT pu.person_id
                      FROM   per_users pu
                      WHERE  pu.username = :xdo_user_name)
```

### Points to remember
- **Secured views slow down queries**, especially for users with wide access. For admin or integration reports that are already restricted by role and catalog permissions, projects often use base tables. This must be a **conscious, documented decision**, usually agreed with the client.
- **Scheduled reports** run with the security of the **user who scheduled them**.
- **Catalog permissions** (Phase 1) control **who can run** a report. **Secured views** control **which data** they see. You often need both.
- Always ask during requirement gathering: **"Who will run this report, and what data should they see?"**

> **Interview-ready line:**
> "BIP SQL on base tables doesn't apply role-based security, so anyone who can run the report sees all data. To secure it, I join or filter with secured list views like PER_PERSON_SECURED_LIST_V or PER_ASSIGNMENT_SECURED_LIST_V, which return only the records allowed by the user's security profile. For self-service reports I use :xdo_user_name with PER_USERS to restrict to the logged-in person. Catalog permissions control who can run the report, and secured views control what data they see."

---

## Quick Revision Sheet

1. **Data layers:** Person → Work relationship (period of service) → Assignment.
2. **Person:** `PER_ALL_PEOPLE_F` (number), `PER_PERSON_NAMES_F` (names, `NAME_TYPE='GLOBAL'`), `PER_PERSONS` (birth date).
3. **Hire / termination dates:** `PER_PERIODS_OF_SERVICE` (`DATE_START`, `ACTUAL_TERMINATION_DATE`).
4. **Job details:** `PER_ALL_ASSIGNMENTS_M` holds IDs for department, job, position, grade, location.
5. **Manager:** `PER_ASSIGNMENT_SUPERVISORS_F` (`MANAGER_TYPE='LINE_MANAGER'`, `MANAGER_ID` = person ID).
6. **Work email:** `PER_EMAIL_ADDRESSES`, `EMAIL_TYPE='W1'`, uses `DATE_FROM` / `DATE_TO`.
7. **`_F`** = date-tracked, one change per day. **`_M`** = multiple changes per day; add `EFFECTIVE_LATEST_CHANGE='Y'`.
8. **Golden rule:** a date filter on **every** date-tracked table.
9. **Primary assignment:** `ASSIGNMENT_TYPE IN ('E','C')`, `PRIMARY_FLAG='Y'`, `ASSIGNMENT_STATUS_TYPE='ACTIVE'`.
10. **Event reports:** use the event date (hire date or termination date), not `SYSDATE`.
11. **Future-dated rows:** use `BETWEEN`, not "row ending 4712-12-31."
12. **`_TL`:** `LANGUAGE = USERENV('LANG')`. **`_VL`:** language already applied.
13. **Lookups:** join on `LOOKUP_TYPE` + `LOOKUP_CODE`, use a LEFT JOIN, read `MEANING`.
14. **DFF:** `ATTRIBUTE` columns on the base table. **EFF:** extra-info tables, multiple rows, filter `INFORMATION_TYPE`. **KFF:** `SEGMENT` columns.
15. **Security:** BIP base-table SQL is **not secured**. Use `PER_PERSON_SECURED_LIST_V` and related views. Use `:xdo_user_name` for "my data only."
16. **Optional data** (manager, email, location): use `LEFT JOIN` so employees aren't dropped.

---

## Practice Interview Questions with Answers

**Q1. What is the difference between `_F` and `_M` tables?**
Both are date-tracked with effective start and end dates. `_F` tables allow only one change per day. `_M` tables, like PER_ALL_ASSIGNMENTS_M, allow multiple changes on the same day, tracked by EFFECTIVE_SEQUENCE. To get the final change of that day, we filter EFFECTIVE_LATEST_CHANGE = 'Y'.

**Q2. Your report shows the same employee twice. What do you check?**
First, whether every date-tracked table has a date filter. Then `effective_latest_change = 'Y'` on `_M` tables, `name_type = 'GLOBAL'` on names, `primary_flag = 'Y'` on assignments, the language filter on `_TL` tables, multiple emails or addresses without a type filter, and whether the person was rehired with more than one period of service.

**Q3. How do you get an employee's current department name?**
Join PER_ALL_ASSIGNMENTS_M (primary, active, latest change, filtered as of the date) on ORGANIZATION_ID to HR_ORGANIZATION_UNITS_F_TL, with a language filter using USERENV('LANG') and a date filter on the TL table, then read NAME.

**Q4. Where do you find hire date and termination date?**
In PER_PERIODS_OF_SERVICE. DATE_START is the hire date and ACTUAL_TERMINATION_DATE is the termination date. A rehired person has more than one row.

**Q5. In a terminations report, why shouldn't you use SYSDATE for every table?**
The report needs the employee's details as they were on their last working day. Using SYSDATE might return a different version of the data, and an active-status filter would remove terminated employees altogether. Each table should be filtered as of ACTUAL_TERMINATION_DATE.

**Q6. What is the problem with future-dated records?**
HR can enter changes that become effective later. If a query picks the row with an end date of 31-Dec-4712, it may return a future job or department instead of the current one. Using "date BETWEEN effective_start_date AND effective_end_date" returns the correct version for the report date.

**Q7. Why do we filter `LANGUAGE = USERENV('LANG')` on `_TL` tables?**
_TL tables have one row per installed language. Without the filter, each record repeats once per language. USERENV('LANG') returns the language of the user's session.

**Q8. What is the difference between DFF and EFF?**
A DFF adds extra fields stored in ATTRIBUTE columns on the same base table, with one set of values per record. An EFF stores groups of fields in separate extra-information tables and can hold multiple rows per record. EFF data is filtered by INFORMATION_TYPE, which is the context.

**Q9. How do you know which ATTRIBUTE column holds a client's custom field?**
In Setup and Maintenance, through Manage Descriptive Flexfields (or Manage Extensible Flexfields): search the flexfield, open the context and segment, and check the mapped table column and its value set.

**Q10. How is data security handled in BIP for HCM?**
SQL on base tables isn't secured automatically. We use secured list views such as PER_PERSON_SECURED_LIST_V and PER_ASSIGNMENT_SECURED_LIST_V, which return only the records allowed by the user's security profile. For self-service reports, we restrict using :xdo_user_name joined to PER_USERS.

**Q11. How do you find which tables an OTBI field comes from?**
Build an OTBI analysis with that field, run it, then go to Administration → Manage Sessions and view the log, which shows the physical SQL with table names. You can also check Oracle's "Tables and Views for HCM" documentation.

**Q12. Why use LEFT JOIN for manager or email?**
Not every employee has a manager or a work email. An inner join would remove those employees from the report. A LEFT JOIN keeps them and shows the missing value as blankXML.