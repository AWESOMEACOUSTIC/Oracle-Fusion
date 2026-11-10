# Oracle HCM Data Loader (HDL) — Phase 1: Foundations & Fundamentals

> **Study guide · Phase 1 of 10** (merged: Core HR prerequisites + HDL fundamentals)
> Goal: understand the Core HR data HDL loads, and what HDL is and how it works, before writing any `.dat` file.

---

## Table of Contents

**Part A — Core HR Prerequisites**
1. [The Core HR Data Model: Person → Work Relationship → Work Terms → Assignment](#1-the-core-hr-data-model)
2. [Date-Effectivity (Effective Dates, Sequence, Latest Change)](#2-date-effectivity)
3. [Workforce Structures: Location, Department, Job, Grade, Position](#3-workforce-structures)
4. [Actions, Action Reasons, Lookups & Value Sets](#4-actions-action-reasons-lookups--value-sets)
5. [Roles & Security Needed for HDL](#5-roles--security-needed-for-hdl)

**Part B — HDL Fundamentals**
6. [What HDL Is: Initial (Conversion) vs Incremental Loads](#6-what-hdl-is)
7. [HDL Architecture: How HDL Works End to End](#7-hdl-architecture--how-hdl-works)
8. [HDL (Inbound) vs HCM Extracts (Outbound)](#8-hdl-inbound-vs-hcm-extracts-outbound)
9. [Supported Business Objects & the Data Exchange Work Area](#9-supported-business-objects--the-data-exchange-work-area)

**Wrap-up**
- [Phase 1 Cheat Sheet](#phase-1-cheat-sheet)
- [Practice Questions & Answers](#practice-questions--answers)

---

# PART A — CORE HR PREREQUISITES

## 1. The Core HR Data Model

### Concept
In Oracle HCM Cloud, a "worker" isn't one record. It's a **hierarchy** of records, and HDL loads that hierarchy through one business object called **Worker** (file `Worker.dat`).

```
PERSON  (who the human is — exists once, no matter how many jobs)
  │
  ├── Person Name, Email, Phone, Address, National Identifier,
  │   Legislative Info, Citizenship, etc.
  │
  └── WORK RELATIONSHIP  (the person's relationship with ONE Legal Employer)
        │   e.g. Employee of "Cognizant India Pvt Ltd"
        │
        └── WORK TERMS  (the contract-level terms — usually one per relationship)
              │
              └── ASSIGNMENT  (the actual job: dept, job, grade, position,
                              location, manager, working hours, etc.)
                    └── Assignment Supervisor (manager), Work Measures, etc.
```

### Each level explained

| Level | What it represents | Key facts |
|---|---|---|
| **Person** | The human being | One person record even if rehired many times. Identified by **Person Number**. |
| **Work Relationship** | Relationship with a **Legal Employer** | Has a **Worker Type**: Employee (E), Contingent Worker (C), Nonworker (N), Pending Worker (P). Has start date (hire date) and termination date. One relationship can be marked **primary**. |
| **Work Terms** | Terms & conditions of employment | Sits between the relationship and the assignment. In most setups there is one work terms record per relationship. |
| **Assignment** | The job the person does | Holds Business Unit, Department, Job, Grade, Position, Location, Manager, Assignment Status, FT/PT, etc. A worker can have **multiple assignments**. |

### Behind-the-scenes tables (interview favourite)

| Entity | Main table(s) |
|---|---|
| Person | `PER_ALL_PEOPLE_F`, `PER_PERSONS` |
| Person Name | `PER_PERSON_NAMES_F` |
| Email | `PER_EMAIL_ADDRESSES` |
| Phone | `PER_PHONES` |
| Address | `PER_ADDRESSES_F`, `PER_PERSON_ADDR_USAGES_F` |
| National Identifier | `PER_NATIONAL_IDENTIFIERS` |
| Work Relationship | `PER_PERIODS_OF_SERVICE` |
| Work Terms + Assignment | `PER_ALL_ASSIGNMENTS_M` (same table, told apart by `ASSIGNMENT_TYPE`) |
| Manager | `PER_ASSIGNMENT_SUPERVISORS_F` |

**`ASSIGNMENT_TYPE` values in `PER_ALL_ASSIGNMENTS_M`:**
- `E` = Employee assignment, `ET` = Employee work terms
- `C` = Contingent worker assignment, `CT` = Contingent work terms
- `N` = Nonworker assignment, `NT` = Nonworker terms
- `P` = Pending worker assignment, `PT` = Pending worker terms

### Why this matters for HDL
In `Worker.dat`, each of these levels is a separate **component** (discriminator) such as `Worker`, `PersonName`, `PersonEmail`, `WorkRelationship`, `WorkTerms`, `Assignment`. Each child line must point to its parent. If you don't know the hierarchy you can't build the file, and you can't work out why a record was rejected.

### Key points
- A person exists **once**. Rehires create a **new work relationship**, not a new person.
- A global transfer to another legal employer means a **new work relationship**.
- Work terms and assignments share a table but are different records.

---

## 2. Date-Effectivity

### Concept
Most HR data is **date-effective**: Oracle keeps the **history** of changes instead of overwriting. Every row has a validity window.

| Column | Meaning |
|---|---|
| `EFFECTIVE_START_DATE` | Date this version of the record becomes valid |
| `EFFECTIVE_END_DATE` | Date this version stops being valid. The latest row ends on **4712-12-31** ("end of time") |

### Example: a salary/department history for one assignment

| Effective Start | Effective End | Department | Reason |
|---|---|---|---|
| 2024-01-01 | 2025-03-31 | Sales | Hire |
| 2025-04-01 | 2026-06-30 | Marketing | Transfer |
| 2026-07-01 | 4712-12-31 | Marketing (promoted) | Promotion |

Each row is a "slice" of time. To find "where did they work on 2025-05-10?", you query:

```sql
WHERE DATE '2025-05-10' BETWEEN effective_start_date AND effective_end_date
```

### `_F` tables vs `_M` tables

| Suffix | Meaning | Example |
|---|---|---|
| `_F` | Date-tracked: **one change per day** max | `PER_ALL_PEOPLE_F`, `PER_PERSON_NAMES_F` |
| `_M` | Date-tracked with **multiple changes per day** | `PER_ALL_ASSIGNMENTS_M` |
| *(none)* | Not date-effective | `PER_EMAIL_ADDRESSES`, `PER_PHONES` |

### Multiple changes on the same day: `EFFECTIVE_SEQUENCE` and `EFFECTIVE_LATEST_CHANGE`
On `_M` tables (such as assignments), you can have **two or more changes on the same date**, for example a transfer and then a salary-grade change both on 2026-07-01.

| Column | Meaning |
|---|---|
| `EFFECTIVE_SEQUENCE` | Order of changes on the same date: 1, 2, 3… |
| `EFFECTIVE_LATEST_CHANGE` | `Y` = the final change for that date; `N` = an earlier change that day |

**The standard "current assignment" filter (memorise it):**

```sql
WHERE TRUNC(SYSDATE) BETWEEN paam.effective_start_date AND paam.effective_end_date
  AND paam.effective_latest_change = 'Y'
  AND paam.assignment_type = 'E'
  AND paam.primary_flag = 'Y'
```

### Update vs Correction (conceptual — Phase 7 covers it in HDL)
- **Update** = create a **new slice** from a date onward (history preserved).
- **Correction** = **change an existing slice** in place (history rewritten because it was wrong).

### Why this matters for HDL
HDL file lines carry `EffectiveStartDate`, `EffectiveEndDate`, `EffectiveSequence` and `EffectiveLatestChange`. Overlapping or gapped dates are one of the most common HDL load errors.

### Key points
- End-of-time = `4712-12-31`. In HDL files dates are written `YYYY/MM/DD`, e.g. `4712/12/31`.
- `_M` = multiple changes per day; use `EFFECTIVE_LATEST_CHANGE = 'Y'`.
- A date-effective object needs **continuous, non-overlapping** date ranges.

---

## 3. Workforce Structures

### Concept
Workforce structures are the **reference data** an assignment points to. They must exist **before** you load workers.

| Structure | What it is | Main table |
|---|---|---|
| **Location** | Physical place where people work (address) | `HR_LOCATIONS_ALL` |
| **Organization / Department** | Org units; a Department is an org with the *Department* classification | `HR_ALL_ORGANIZATION_UNITS_F`, `HR_ORG_UNIT_CLASSIFICATIONS_F` |
| **Legal Employer** | An org classified as Legal Employer: the entity that employs people | (org tables + classification) |
| **Business Unit** | Org that performs business functions | `HR_ALL_ORGANIZATION_UNITS_F` |
| **Job** | Generic role, e.g. "Software Engineer" | `PER_JOBS_F` |
| **Grade** | Pay/seniority level, e.g. "G5" | `PER_GRADES_F` |
| **Position** | A specific instance of a job in a dept, e.g. "Software Engineer #3 in Chennai Dev" | `HR_ALL_POSITIONS_F` |

### Dependency chain (this becomes your **HDL load order**)

```
Location ──► Organization/Department ──► Job ──► Grade ──► Position ──► Worker (Assignment)
```

- A **Department** may reference a **Location**.
- A **Position** references a **Job**, **Department**, **Location**, and (optionally) **Grades**.
- An **Assignment** references all of them.

So you load from the **left** first. If you load workers before their department exists, the load fails with an "invalid value" error.

### Job vs Position (a common interview question)
- **Job** = *what kind* of work ("Accountant"). Many people can hold the same job.
- **Position** = *a specific seat* ("Accountant – Finance Dept – Hyderabad – Seat 2"), with headcount/FTE.
- Organisations use **position-based** or **job-based** management. That decides whether assignments in your HDL file carry `PositionCode` or `JobCode`.

### Key points
- Workforce structures are mostly **date-effective** (`_F`).
- Load them in dependency order.
- Every one of them has a supported HDL business object (Location, Organization, Job, Grade, Position…).

---

## 4. Actions, Action Reasons, Lookups & Value Sets

### Actions & Action Reasons
Every change to a worker happens through an **Action**, which tells Oracle **what kind of event** it is.

| Example Action Code | Meaning |
|---|---|
| `HIRE` | New hire |
| `REHIRE` | Rehire |
| `ASG_CHANGE` | Assignment change |
| `TRANSFER` | Transfer |
| `PROMOTION` | Promotion |
| `TERMINATION` / `RESIGNATION` | Ending the work relationship |
| `GLB_TRANSFER` | Global transfer |

- An **Action Reason** gives more detail, e.g. Action = `TERMINATION`, Reason = `Better Opportunity`.
- Tables: `PER_ACTIONS_B` / `PER_ACTIONS_VL`, `PER_ACTION_REASONS_B`, `PER_ACTION_OCCURRENCES`.

> **HDL connection:** Worker, WorkRelationship and Assignment lines carry `ActionCode` (and optionally `ReasonCode`). Use the **code**, not the display name. Action codes can differ by environment because customers create their own, so always check the codes in your instance.

### Lookups
A **lookup** is a list of allowed codes with display meanings, e.g. `SEX`: `M` = Male, `F` = Female.
- Lookup *type* = the list (e.g. `MAR_STATUS`), lookup *code* = the stored value (e.g. `M`).
- Tables/views: `HCM_LOOKUPS`, `FND_LOOKUP_VALUES`.
- Types: **Standard** (can't add codes), **Extensible** (can add codes), **User** (fully editable).

### Value Sets
Validation lists used mainly by **flexfields** (DFF/EFF segments). They can be independent lists, table-validated, or format-only.

### Why this matters for HDL
HDL validates every coded attribute. A file that says `Gender = Male` instead of `M`, or an action code that doesn't exist, fails at **load** with an invalid-value error.

### Key points
- HDL files carry **codes**, not meanings.
- Always pull lookup and action codes from the target environment.

---

## 5. Roles & Security Needed for HDL

### Concept
To use HDL, a user needs access to (1) the HDL tasks in the **Data Exchange work area**, (2) the **WebCenter Content (UCM)** folder files are uploaded to, and (3) permission to load each business object.

### Main role
**Human Capital Management Integration Specialist** (predefined job role). It already inherits everything needed for HDL.

### What's inside (to give other roles HDL access)

| Role / Privilege | Grants access to |
|---|---|
| **HCM Data Load** duty — `ORA_HRC_HCM_DATA_LOAD_DUTY` | HDL tasks in the Data Exchange work area (View Business Objects, Recent File Loads, Import and Load Data, Delete Stage Table Data) |
| `HCM_DATALOADER_IMPORT_RWD` | Upload files to the `hcm/dataloader/import` folder (account) on **WebCenter Content (UCM)** |
| `HCM_DATALOADER_EXPORT_RWD` | Download files from the HCM export account |
| Manage Data Exchange Work Area — `HRC_MANAGE_DATA_EXCHANGE_PRIV` | Access to the work area itself |

### Role-based business object access (advanced but good to know)
- An admin can **restrict which business objects** a role may load. This is enabled in **Configure HCM Data Loader** through the **"Enable Configuration of Role-Based Business Object Access"** parameter.
- **Security-related objects** (Areas of Responsibility, security profiles, etc.) need **extra** privileges because they control who can see what data.
- Granting HDL roles requires the **IT Security Manager** role.

### Key points
- HDL users are usually integration specialists, not regular HR users.
- Uploading to UCM and loading in HDL are **separate permissions**.

---

# PART B — HDL FUNDAMENTALS

## 6. What HDL Is

### Definition
**HCM Data Loader (HDL)** is Oracle HCM Cloud's **file-based bulk data loading tool**. You prepare pipe-delimited `.dat` files (one per business object), zip them, and HDL **creates, updates, or deletes** records in HCM Cloud, running the **same validations as the UI**.

### Key characteristics
- **Bulk**: thousands to millions of records.
- **File-based**: `.dat` text files inside a `.zip`.
- **Business-object based**: you load *Worker*, *Job* or *Location*, not raw tables. You never insert into tables directly in Oracle Cloud.
- **Hierarchical**: one file carries a parent and all its children (Worker → Name → Email → Relationship → Assignment).
- **Validated**: the same business rules as the UI, so bad data is rejected and reported.
- **Supports `MERGE`** (create if new, update if it exists) and **`DELETE`**.
- **Can be automated** through web services or OIC (covered in Phase 9).

### Two main uses

#### a) Initial / Conversion Load (data migration)
Used **once** during implementation to move data from a legacy system (SAP, PeopleSoft, an old HRMS, Excel) into Oracle HCM Cloud.

**How it typically runs on a project:**
1. **Extract** data from the legacy system.
2. **Map** legacy fields to HDL attributes, and map legacy codes to Oracle codes (departments, jobs, action codes).
3. **Transform** the data into `.dat` files.
4. **Load in dependency order**: Locations → Orgs → Jobs → Grades → Positions → Workers → Salaries → Element Entries…
5. **Reconcile**: compare counts and values between legacy and Oracle using BIP/OTBI reports.
6. **Repeat in mock cycles**: Mock 1 → Mock 2 → SIT/UAT → **Cutover (Production)**. Each round fixes errors from the last one.

**Decisions on a conversion:**
- **How much history?** Current snapshot only, or full history (all transfers and promotions since hire)?
- **Key strategy:** usually **source keys** (legacy system name + legacy ID), so you can trace every record back to the source. Phase 3 covers this in detail.

#### b) Incremental / Ongoing Load (integration)
After go-live, HDL is used **regularly** to load **changes**:
- New hires from a recruiting system
- Daily or weekly updates from a third-party system
- Mass updates, e.g. changing the location for 5,000 assignments after an office move
- Periodic loads of element entries or other data

Here the file contains **only what changed**, and `MERGE` takes care of creating or updating.

| | Initial (Conversion) | Incremental (Ongoing) |
|---|---|---|
| When | During implementation / go-live | After go-live, repeatedly |
| Volume | Very large, all data | Smaller, changes only |
| Frequency | A few mock runs + cutover | Daily / weekly / on demand |
| Main challenge | Mapping, history, load order, reconciliation | Key matching, date-effective updates, automation |

---

## 7. HDL Architecture & How HDL Works

### The big picture

```
 ┌──────────────────┐
 │ Source / Legacy  │  (SAP, PeopleSoft, Excel, 3rd-party, OIC, HCM Extract…)
 └────────┬─────────┘
          │  1. Prepare .dat files (pipe-delimited) → zip them
          ▼
 ┌──────────────────────────────┐
 │ Oracle WebCenter Content     │  2. Upload zip to account hcm/dataloader/import
 │ (UCM)                        │     → file gets a unique Content ID
 └────────┬─────────────────────┘
          │  3. Submit HDL (UI: Import and Load Data, or web service)
          ▼
 ┌──────────────────────────────┐
 │ IMPORT PHASE                 │  • Unzip, read each .dat file
 │ (File → Stage tables)        │  • Validate FILE FORMAT vs METADATA line
 │                              │    (attribute names, data types, dates)
 │                              │  • Write rows to HDL STAGE tables
 │                              │  • Group physical lines into LOGICAL OBJECTS
 └────────┬─────────────────────┘    (e.g. 1 worker = all of its lines)
          ▼
 ┌──────────────────────────────┐
 │ LOAD PHASE                   │  • Validate BUSINESS RULES (same as UI):
 │ (Stage → Application tables) │    lookups, references, date overlaps…
 │                              │  • Call business-object logic to create/update
 │                              │  • Write to base tables (PER_ALL_PEOPLE_F…)
 │                              │  • Record keys in the key map table
 └────────┬─────────────────────┘
          ▼
 ┌──────────────────────────────┐
 │ RESULTS                      │  • Data set status, object status
 │ (Data Exchange work area)    │  • Errors/warnings per line → fix → resubmit
 └──────────────────────────────┘
```

### Step-by-step

**Step 1 — Prepare the file**
- One `.dat` file per business object, e.g. `Worker.dat`, `Location.dat`.
- Each file starts with a **`METADATA`** line (the column headers) followed by **`MERGE`** / **`DELETE`** data lines.
- Zip one or more `.dat` files together. The zip is called a **data set**.

```
METADATA|Location|SourceSystemOwner|SourceSystemId|SetCode|LocationCode|LocationName|EffectiveStartDate|ActiveStatus|AddressLine1|TownOrCity|Country|PostalCode
MERGE|Location|LEGACY|LOC_CHN_01|COMMON|CHN01|Chennai Office|1951/01/01|A|Siruseri IT Park|Chennai|IN|603103
```
*(Illustrative. Exact attributes come from the **View Business Objects** template; Phase 2 explains the anatomy.)*

**Step 2 — Upload to WebCenter Content (UCM)**
- The zip is stored in UCM under the **`hcm/dataloader/import`** account and gets a unique **Content ID**.
- When you upload through the *Import and Load Data* UI, this happens automatically. In automated integrations you upload by web service first.

**Step 3 — Import phase (file → stage)**
- HDL reads the zip and checks **file-level** problems: unknown attribute names, bad date formats, a missing METADATA line, wrong delimiters.
- Valid rows go to the **stage tables**.
- Rows are grouped into **logical objects**. For example, all lines belonging to employee 1001 (Worker, Name, Email, Relationship, Assignment) form **one logical Worker object**.
- Import errors mean the *file* is wrong.

**Step 4 — Load phase (stage → application)**
- Each logical object is validated against **business rules**: does the department exist, is the gender code valid, do dates overlap?
- Valid objects are written to the application (base) tables.
- **Atomicity is per logical object.** If one line of a worker fails (e.g. a bad email), the **whole worker** fails, but **other workers still load**. You don't get a half-created worker.
- Objects are processed **in parallel** (concurrent threads), which is why HDL is fast for bulk volumes.
- Load errors mean the *data* is wrong.

**Step 5 — Review results & fix**
- Check the data set and object status in the **Data Exchange** work area.
- Read the error messages per line, fix the file (or the setup data), and resubmit. Objects that already succeeded aren't reprocessed.

### Import vs Load (a common interview question)

| | Import | Load |
|---|---|---|
| Moves data | File → **Stage** tables | Stage → **Application** tables |
| Validates | **Format/structure** (METADATA, attribute names, data types, dates) | **Business rules** (lookups, references, dates, hierarchy) |
| Example error | "Attribute `PersonNumbr` not recognised" | "The value for Department is invalid" |
| Can run separately? | Yes. You can import only, review, then load. Or run *Import and Load* together. |

### Key points
- HDL never writes straight to base tables. It always goes File → UCM → Stage → Base.
- **Logical object = unit of success/failure.**
- Stage data stays until it's purged (the *Delete Stage Table Data* task), which helps with troubleshooting.

---

## 8. HDL (Inbound) vs HCM Extracts (Outbound)

### Concept
- **HDL** moves data **INTO** Oracle HCM (inbound).
- **HCM Extracts** moves data **OUT OF** Oracle HCM (outbound), to payroll vendors, benefits providers, banks, or downstream systems.

```
 External system ──[ HDL ]──►  ORACLE HCM CLOUD  ──[ HCM Extracts ]──► External system
       (inbound: create/update/delete)          (outbound: read & export)
```

### Comparison

| Aspect | HDL | HCM Extracts |
|---|---|---|
| Direction | **Inbound** (into HCM) | **Outbound** (out of HCM) |
| Purpose | Create / update / delete HCM data in bulk | Pull HCM data for interfaces & reports |
| Input / Output | Input: `.dat` files in a zip | Output: XML, formatted through BI Publisher templates into CSV, fixed-width, Excel, PDF, etc. |
| Built using | Business object METADATA + data lines | Extract definition: **blocks**, **database items (DBIs)**, **criteria/filters**, **user entities** |
| Changes data? | **Yes** | **No** (read-only) |
| Change detection | You send only what changed | Built-in **changes-only** modes (send only changed records since the last run) |
| Scheduling | Manual, web service, OIC | Scheduled through payroll flows / flow patterns |
| Delivery | File in from UCM | Output to UCM, SFTP, email, etc. (through BIP delivery) |
| Work area | Data Exchange | Data Exchange (Extract Definitions, Submit Extracts, View Extract Results) |

### How they work together
- **Loopback integration:** an HCM Extract **reads** data from HCM, the output is **formatted as an HDL file**, and HDL **loads it back** into HCM. This is used for mass updates driven by existing data, e.g. "set this flag for all workers in department X" (covered in Phase 9).
- **HCM Extracts / BIP for HDL support:** extracts and BIP queries are used to generate HDL **DELETE** files or to **reconcile** conversion loads.

### Key points
- HDL = write, Extracts = read.
- Both live in the **Data Exchange** work area.
- Don't mix up HCM Extracts with **BI Publisher reports** or **OTBI**. Extracts are built for **interfaces** (large volumes, changes-only, scheduled delivery).

---

## 9. Supported Business Objects & the Data Exchange Work Area

### Business objects
HDL loads data **by business object**: a logical unit with its own components (child records). Each business object has its own `.dat` file name.

**Common business objects by product area:**

| Product Area | Example Business Objects (file name) |
|---|---|
| **Global HR – Work Structures** | `Location.dat`, `Organization.dat`, `Job.dat`, `JobFamily.dat`, `Grade.dat`, `GradeRate.dat`, `GradeLadder.dat`, `Position.dat`, `ActionReason`-related objects |
| **Global HR – Workers** | `Worker.dat` (Person, Names, Emails, Phones, Addresses, National IDs, Work Relationship, Work Terms, Assignment, Supervisor…), `Contact.dat`, `PersonalPaymentMethod.dat` |
| **Global HR – Other** | `DocumentsOfRecord.dat`, `WorkerJourney`-related objects, `AreasOfResponsibility.dat` (security-related) |
| **Compensation** | `Salary.dat`, `SalaryBasis.dat`, `ElementEntry.dat` *(element entries are payroll-related)* |
| **Payroll** | `ElementEntry.dat`, `PayrollRelationship`-related objects, `AssignedPayroll` |
| **Talent / Profiles** | `TalentProfile.dat`, `ContentItem.dat`, `Goal.dat`, `PerformanceDocument`-related objects |
| **Absence** | `PersonAbsenceEntry.dat`, `PersonAccrualDetail.dat` |
| **Utility objects** | `SourceKey.dat` (change source keys), lookup objects (`CommonLookup.dat`) |

> This list isn't complete. **The only complete, current list is the *View Business Objects* task in your environment** (filter by Product Area). New objects are added each quarterly release.

**Hierarchy inside a business object (Worker example):**
```
Worker (top-level)
 ├── PersonName
 ├── PersonEmail
 ├── PersonPhone
 ├── PersonAddress
 ├── PersonNationalIdentifier
 ├── PersonLegislativeData
 └── WorkRelationship
       └── WorkTerms
             └── Assignment
                   ├── AssignmentSupervisor
                   └── AssignmentWorkMeasure
```
Each component is a **discriminator**: the second field on every `METADATA`/`MERGE` line (e.g. `MERGE|PersonName|…`).

### The Data Exchange Work Area
**Navigation:** Home → **My Client Groups** → **Data Exchange**

This is HDL's home. The main tasks:

| Task | What you use it for |
|---|---|
| **Import and Load Data** | Upload a zip (from your desktop or UCM), submit Import / Load / Import-and-Load, **monitor data sets**, view errors, resubmit, roll back loaded objects where supported |
| **View Business Objects** | Browse all supported objects; see components, attributes and supported keys; **generate templates** (METADATA lines); **Refresh Object** / **Refresh All Objects** (by product area) so templates include your latest flexfields |
| **Recent File Loads** | Quick view of your recently submitted files and their status |
| **Delete Stage Table Data** | Purge old HDL stage data (housekeeping) |
| **Spreadsheet loads (HSDL)** | Load data through spreadsheet templates (Phase 9) |
| **HCM Extracts tasks** | Extract Definitions, Submit Extracts, View Extract Results (outbound) |

Also in **Setup and Maintenance** → *HCM Data Loader* functional area → **Configure HCM Data Loader**: global defaults such as error thresholds, concurrency, file encryption and role-based object access (Phase 5).

### Using View Business Objects (first hands-on)
1. Data Exchange → **View Business Objects**.
2. Search e.g. **Job**. Use the **Product Area** column/filter to pick the right one (several objects start with "Job").
3. Select it and click **Refresh Object** until *Last Refreshed Date* = today. Refreshing regenerates the template file and the overview spreadsheet with your current flexfields.
4. Open the object to see its **components**, each attribute's **data type, mandatory flag, and supported keys**, and download the **template**.

> **Why refresh?** If your environment added a DFF segment yesterday, the template won't show it until the object is refreshed.

### Key points
- Business object = unit of loading; component = child record; discriminator = component name on each line.
- **View Business Objects** is where you check attributes. Don't guess attribute names.
- Navigation: **My Client Groups → Data Exchange**.

---

# PHASE 1 CHEAT SHEET

| Topic | Remember this |
|---|---|
| Worker hierarchy | Person → Work Relationship (Legal Employer) → Work Terms → Assignment |
| Key tables | `PER_ALL_PEOPLE_F`, `PER_PERSON_NAMES_F`, `PER_PERIODS_OF_SERVICE`, `PER_ALL_ASSIGNMENTS_M` |
| Assignment types | `E/ET`, `C/CT`, `N/NT`, `P/PT` (T = terms) |
| End of time | `4712-12-31` (HDL format `4712/12/31`) |
| `_F` vs `_M` | `_F` = one change/day; `_M` = many changes/day (`EFFECTIVE_SEQUENCE`, `EFFECTIVE_LATEST_CHANGE`) |
| Load order | Location → Org/Dept → Job → Grade → Position → Worker → Salary/Elements |
| Codes not meanings | HDL needs lookup codes and action codes |
| HDL role | Human Capital Management Integration Specialist |
| UCM account | `hcm/dataloader/import` |
| HDL flow | Prepare zip → UCM → **Import** (file→stage, format checks) → **Load** (stage→base, business rules) → results |
| Unit of failure | One **logical object** (e.g. one whole worker) |
| HDL vs Extracts | HDL = inbound write; Extracts = outbound read |
| Navigation | My Client Groups → Data Exchange |
| Source of truth for attributes | View Business Objects (refresh first!) |

---

# PRACTICE QUESTIONS & ANSWERS

### Section A — Concept checks

**Q1. An employee resigned in 2023 and is rehired in 2026. How many Person records and Work Relationship records will they have?**
<details><summary>Answer</summary>

**One Person** record (same person number) and **two Work Relationships**: the 2023 one (terminated) and a new 2026 one. A rehire creates a new relationship, not a new person.
</details>

**Q2. Work Terms and Assignments are stored in the same table. Which table, and how do you tell them apart?**
<details><summary>Answer</summary>

`PER_ALL_ASSIGNMENTS_M`. Use the `ASSIGNMENT_TYPE` column: `E` = employee assignment, `ET` = employee work terms (similarly `C/CT`, `N/NT`, `P/PT`).
</details>

**Q3. Why does `PER_ALL_ASSIGNMENTS_M` need `EFFECTIVE_LATEST_CHANGE` while `PER_ALL_PEOPLE_F` doesn't?**
<details><summary>Answer</summary>

`_M` tables allow **multiple changes on the same date**, so there can be several rows for the same effective date. `EFFECTIVE_SEQUENCE` orders them and `EFFECTIVE_LATEST_CHANGE = 'Y'` marks the final one. `_F` tables allow only one change per day, so this isn't needed.
</details>

**Q4. What is the difference between a Job and a Position?**
<details><summary>Answer</summary>

A **Job** is a generic role ("Accountant") that many people can hold. A **Position** is a specific instance of a job in a particular department/location, with headcount/FTE. Position-based organisations assign people to positions; job-based ones assign them to jobs directly.
</details>

**Q5. What is the difference between the Import phase and the Load phase?**
<details><summary>Answer</summary>

**Import** moves data from the file into the **stage tables** and validates **file format/structure** (attribute names, data types, date formats against the METADATA line). **Load** moves data from stage to the **application tables** and validates **business rules** (valid lookups, existing references, non-overlapping dates). Import errors mean the file is wrong; load errors mean the data is wrong.
</details>

**Q6. What is a logical object, and why does it matter?**
<details><summary>Answer</summary>

A logical object is the group of all file lines that make up one business-object instance, e.g. all lines (Worker, PersonName, Email, WorkRelationship, Assignment) for one employee. It is the **unit of success/failure**: if any line fails, that whole object fails and nothing is partially created, but other objects in the file still load.
</details>

**Q7. Name the four main tasks in the Data Exchange work area used for HDL.**
<details><summary>Answer</summary>

**Import and Load Data**, **View Business Objects**, **Recent File Loads**, **Delete Stage Table Data**. (Configure HCM Data Loader is in Setup and Maintenance.)
</details>

**Q8. HDL vs HCM Extracts in one line each.**
<details><summary>Answer</summary>

**HDL** = inbound, file-based bulk load that creates, updates or deletes data in HCM. **HCM Extracts** = outbound, read-only extraction of HCM data (often changes-only) delivered to external systems.
</details>

### Section B — Scenario-based (interview style)

**S1. Your Worker file loaded with errors: "The value for Department is invalid" for 200 workers. The department names look correct in the file. What could be wrong?**
<details><summary>Answer</summary>

Possible causes:
1. The departments **weren't loaded yet**. Load-order problem: Organization must load before Worker.
2. The department **exists but isn't effective** on the assignment's start date (e.g. department starts 2025/01/01 but the assignment starts 2024/06/01). This is a date-effectivity mismatch.
3. You passed the department **name** when the attribute expects a different reference (code / key), or the name doesn't match exactly (spaces, case).
4. The org exists but **lacks the Department classification**.
Fix the cause and resubmit. Successful workers aren't reprocessed.
</details>

**S2. A business user says: "Can't we just insert the data into `PER_ALL_PEOPLE_F` directly with SQL? It'll be faster." How do you respond?**
<details><summary>Answer</summary>

No. In Oracle Fusion Cloud you **can't write to base tables directly**: database access is read-only (through BIP/OTBI). Even if you could, direct inserts would skip all business validations, security, auditing and dependent records (relationship, terms, assignment, key maps). HDL is the supported bulk path: it runs the same validations as the UI and is still fast because it processes objects in parallel.
</details>

**S3. Your project is migrating 50,000 employees from SAP. The client asks: "Do we need all transfer and promotion history, or just current data?" What factors would you discuss?**
<details><summary>Answer</summary>

- **Reporting/legal needs**: do they need historical reports (headcount as of past dates, service calculations)?
- **Payroll/benefits dependencies**: seniority dates, retro calculations.
- **Effort and risk**: full history means much larger files, more date-overlap errors and longer mock cycles.
- **Data quality** of the legacy history.
- **Common compromise**: load the current snapshot plus key dates (original hire, seniority), and keep the legacy system or an archive for history lookups.
Whatever is chosen, load in dependency order and reconcile counts after each mock.
</details>

**S4. A new DFF segment was added to Assignment yesterday. Your downloaded Worker template doesn't show it. Why, and how do you fix it?**
<details><summary>Answer</summary>

The object "shape" (template) was generated **before** the flexfield change. In **View Business Objects**, select **Worker** → **Refresh Object** (or Refresh All Objects for the product area) and wait until *Last Refreshed Date* = today, then download the template again. The new segment will appear.
</details>

**S5. A user can open Import and Load Data but gets a security error when uploading the zip. What's likely missing?**
<details><summary>Answer</summary>

Access to the **UCM import account**. The user needs the `HCM_DATALOADER_IMPORT_RWD` role (upload to `hcm/dataloader/import`). Access to HDL tasks (`ORA_HRC_HCM_DATA_LOAD_DUTY`) and access to UCM are **separate**. Assigning the **Human Capital Management Integration Specialist** role covers both.
</details>

**S6. Write the SQL condition to get each employee's current primary assignment row.**
<details><summary>Answer</summary>

```sql
SELECT paam.*
FROM   per_all_assignments_m paam
WHERE  TRUNC(SYSDATE) BETWEEN paam.effective_start_date AND paam.effective_end_date
AND    paam.effective_latest_change = 'Y'
AND    paam.assignment_type = 'E'
AND    paam.primary_flag = 'Y';
```
</details>

---

**Next → Phase 2: HDL File Anatomy** (`.dat` structure, METADATA/MERGE/DELETE/SET/COMMENT lines, discriminators, formatting rules, flexfields, SET instructions)

*References: Oracle Help Center: HCM Data Loader guide (docs.oracle.com/en/cloud/saas/human-resources), "How You Enable Access to HCM Data Loader", "Import and Load Data", and Oracle tutorials "Creating and Loading Files with HDL" and "View Business Objects".*