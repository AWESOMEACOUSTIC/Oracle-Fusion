# Oracle HCM Data Loader (HDL) — Phase 2: File Anatomy

> **Study guide · Phase 2 of 10**
> Goal: read, write and debug any `.dat` file by hand: its structure, line types, components, formatting rules, flexfields and SET instructions.
> Prerequisite: Phase 1 (Worker hierarchy, date-effectivity, import vs load).

---

## Table of Contents
1. [.dat Files, File Naming & Zip Structure](#1-dat-files-file-naming--zip-structure)
2. [The Five Line Types: METADATA, MERGE, DELETE, SET, COMMENT](#2-the-five-line-types)
3. [Business Objects, Components & Discriminators](#3-business-objects-components--discriminators)
4. [Formatting Rules: Delimiter, Dates, Escape, #NULL](#4-formatting-rules)
5. [Loading Flexfields: DFF & EFF](#5-loading-flexfields-dff--eff)
6. [SET Instructions](#6-set-instructions)
7. [Full Annotated Example: Worker.dat](#7-full-annotated-example-workerdat)
8. [Common File-Level Errors](#8-common-file-level-errors)
- [Phase 2 Cheat Sheet](#phase-2-cheat-sheet)
- [Practice Questions & Answers](#practice-questions--answers)

---

## 1. .dat Files, File Naming & Zip Structure

### Concept
An HDL load is a **zip file** (the **data set**) containing one or more **`.dat` files**. Each `.dat` file holds data for **one business object**.

```
NewHires_Batch01.zip          ← zip name: anything meaningful (becomes the Data Set name)
 ├── Location.dat             ← one file per business object
 ├── Organization.dat
 ├── Job.dat
 └── Worker.dat
```

### Rules
| Rule | Detail |
|---|---|
| **File name = business object name** | `Worker.dat`, `Job.dat`, `Location.dat`. Use the exact name shown in **View Business Objects** (treat it as case-sensitive). `Employees.dat` won't be recognised. |
| **One object per file** | All components of Worker (names, emails, assignments…) go inside `Worker.dat`, not separate files. |
| **Extension** | `.dat` (plain text, not `.txt` or `.csv`) |
| **Encoding** | Save as **UTF-8**, especially for non-English characters (Hindi names, accents). |
| **Zip** | Put the `.dat` files at the **root** of the zip, not inside a sub-folder. |
| **Multiple objects per zip** | Allowed. HDL processes them in dependency order (e.g. Location before Worker) when they're in the same zip. |
| **Attachments** | Files such as documents or photos go in a `BlobFiles` folder inside the zip (covered later). |

### Key points
- Zip = data set; `.dat` = one business object.
- Wrong file name means HDL can't identify the object, and the **import fails**.

---

## 2. The Five Line Types

Every line in a `.dat` file starts with an **instruction**. There are five:

| Instruction | Purpose | Has a discriminator? |
|---|---|---|
| `METADATA` | Defines the **columns** (attribute names) for a component, like a header row | Yes |
| `MERGE` | **Create** the record if it doesn't exist, **update** it if it does | Yes |
| `DELETE` | **Delete** the record | Yes |
| `SET` | Changes **default processing** for the file | No |
| `COMMENT` | Ignored by HDL; notes for humans | No |

> There is **no INSERT or UPDATE** instruction in HDL. `MERGE` does both: HDL checks the **keys** and decides whether to create or update (Phase 3 covers keys).

### 2.1 METADATA
```
METADATA|Job|SetCode|JobCode|Name|EffectiveStartDate|EffectiveEndDate|ActiveStatus
```
- Format: `METADATA|<Discriminator>|<Attr1>|<Attr2>|...`
- It must appear **before** any data line for that discriminator.
- You only include the attributes you're loading. You don't need every attribute of the object.
- **Attribute order is up to you**, but every data line must follow the **same order** as its METADATA line.
- Every component you load (`PersonName`, `Assignment`…) needs **its own METADATA line**.

### 2.2 MERGE
```
MERGE|Job|COMMON|SE01|Software Engineer|2020/01/01|4712/12/31|A
```
- Values map **position-by-position** to the METADATA line:

| Position | METADATA | MERGE value |
|---|---|---|
| 1 | METADATA | MERGE |
| 2 | Job | Job |
| 3 | SetCode | COMMON |
| 4 | JobCode | SE01 |
| 5 | Name | Software Engineer |
| 6 | EffectiveStartDate | 2020/01/01 |
| 7 | EffectiveEndDate | 4712/12/31 |
| 8 | ActiveStatus | A |

### 2.3 DELETE
```
METADATA|PersonEmail|SourceSystemOwner|SourceSystemId
DELETE|PersonEmail|LEGACY|EMP1001_EMAIL_W1
```
- Usually needs only the **key attributes** that identify the record.
- **Not every object/component can be deleted with HDL.** View Business Objects shows whether delete is supported. Phase 7 covers delete rules (date-effective deletes, cascading to children).
- Avoid MERGE and DELETE of the **same record** in the same file.

### 2.4 SET
```
SET PURGE_FUTURE_CHANGES N
```
- No pipe-separated discriminator: just `SET <INSTRUCTION> <VALUE>`.
- **Must appear before the first METADATA line.** (Section 6 lists them.)

### 2.5 COMMENT
```
COMMENT Batch 01 – New hires for Chennai, prepared by Integration Team
```
- Ignored during processing. Useful for documenting files.

### Typical file layout
```
SET ...                     ← 1. optional SET instructions (top of file)
COMMENT ...                 ← 2. optional comments (anywhere)
METADATA|Component1|...     ← 3. header for component 1
MERGE|Component1|...        ←    data lines
MERGE|Component1|...
METADATA|Component2|...     ← 4. header for component 2
MERGE|Component2|...
```

---

## 3. Business Objects, Components & Discriminators

### Terminology
| Term | Meaning | Example |
|---|---|---|
| **Business object** | The top-level thing you're loading; decides the file name | Worker → `Worker.dat` |
| **Component** | A child record type inside the object's hierarchy | PersonName, PersonEmail, WorkRelationship, Assignment |
| **Discriminator** | The component name written in **position 2** of every METADATA/MERGE/DELETE line | `MERGE|PersonName|...` |

The **top-level component** usually has the same name as the object (`Worker` in `Worker.dat`, `Job` in `Job.dat`).

### Worker object hierarchy (main components)
```
Worker                        ← top-level (Person)
 ├── PersonName
 ├── PersonEmail
 ├── PersonPhone
 ├── PersonAddress
 ├── PersonNationalIdentifier
 ├── PersonLegislativeData    ← gender, marital status (legislation-specific)
 ├── PersonCitizenship
 └── WorkRelationship
       └── WorkTerms
             └── Assignment
                   ├── AssignmentSupervisor
                   └── AssignmentWorkMeasure   ← FTE / headcount
```

### How a child knows its parent
Every child line **references its parent's key**, using a special **key-hint** syntax in the METADATA line:

```
METADATA|PersonName|SourceSystemOwner|SourceSystemId|PersonId(SourceSystemId)|...
MERGE|PersonName|LEGACY|EMP1001_NAME|EMP1001|...
```
- `PersonId(SourceSystemId)` means: "the parent Person is identified by its **SourceSystemId** = `EMP1001`".
- Common parent-reference attributes:
  - PersonName / Email / Phone → `PersonId(...)`
  - WorkRelationship → `PersonId(...)`
  - WorkTerms → `PeriodOfServiceId(...)` (the work relationship)
  - Assignment → `WorkTermsAssignmentId(...)`
  - AssignmentSupervisor → `AssignmentId(...)`
- The parent can be **in the same file** or **already exist in Oracle**.

> This is a preview. **Phase 3 (Keys)** covers SourceSystemId, user keys, GUIDs, surrogate IDs and key precedence in detail.

### Logical object reminder (from Phase 1)
HDL uses these parent references to group lines into **one logical object per worker**. If any line of that worker fails, the **whole worker fails**.

---

## 4. Formatting Rules

### 4.1 Delimiter
- Default delimiter: **vertical bar `|`**.
- Change it per file with `SET FILE_DELIMITER ,` (or for the whole enterprise in *Configure HCM Data Loader*).
- Pipe is the default because commas often appear in data (addresses, names).

### 4.2 Dates
- Default format: **`YYYY/MM/DD`**, e.g. `2026/01/05`.
- End of time: **`4712/12/31`**.
- ❌ `05-01-2026`, `2026-01-05`, `05/01/2026` → import/format errors. Stick to `YYYY/MM/DD`.

### 4.3 Reserved characters & escaping
HDL has three reserved characters:

| Reserved | Default | To use it inside a value |
|---|---|---|
| Delimiter | `|` | Escape it: `\|` |
| Escape | `\` | Escape it: `\\` |
| New line | `n` (escaped as `\n`) | `\n` inserts a line break inside a value |

**Examples:**
```
MERGE|Job|COMMON|QA01|Quality \| Testing Lead|...      → Name = "Quality | Testing Lead"
MERGE|PersonAddress|...|Plot 12\nSiruseri IT Park|...  → AddressLine1 on two lines
```
- Change them with `SET FILE_ESCAPE <value>`, `SET FILE_DELIMITER <value>`, `SET FILE_NEW_LINE <value>` (values up to 10 characters).

### 4.4 Blank vs `#NULL` (very important)

| You supply | On **create** | On **update** (record already exists) |
|---|---|---|
| A value | Value is stored | Value **replaces** the old one |
| **Blank** (nothing between pipes `||`) | No value / default | **Existing value is kept** (attribute ignored) |
| **`#NULL`** | No value | **Existing value is cleared** |

```
METADATA|PersonPhone|SourceSystemOwner|SourceSystemId|PhoneType|PhoneNumber|Extension
MERGE|PersonPhone|LEGACY|EMP1001_PH_W1|W1|9840012345||        ← extension unchanged
MERGE|PersonPhone|LEGACY|EMP1001_PH_W1|W1|9840012345|#NULL    ← extension removed
```
> Interview trap: "I sent a blank Middle Name to remove it, but it's still there." Blank means *don't touch*. Use **`#NULL`** to clear.

### 4.5 Special date tokens (preview)
With `SET PURGE_FUTURE_CHANGES N`, `EffectiveEndDate` can take special values:
- `#RETAIN`: keep the existing end date and **don't overwrite future-dated values**
- `#ALL`: apply the change across all future-dated records
(Phase 7 covers these with date-effective scenarios.)

### 4.6 Other formatting rules
| Data type | Rule |
|---|---|
| Codes (lookups, actions) | Use **codes**, not display meanings (`M`, not `Male`; `HIRE`, not `Hire`) |
| Flags | `Y` / `N` |
| Numbers | No thousand separators (`50000`, not `50,000`); `.` as the decimal point |
| Spaces | Avoid stray leading/trailing spaces. They can cause "invalid value" errors |
| Line endings | One record per line; don't leave broken or partial lines |

---

## 5. Loading Flexfields: DFF & EFF

### Concept refresher
- **DFF (Descriptive Flexfield):** extra fields added to an existing record, e.g. "Laptop Asset Tag" on Assignment. One set of segments per record, decided by a **context**.
- **EFF (Extensible Flexfield):** extra **child records** with their own categories and contexts, which can be **multi-row**, e.g. multiple certifications per person. Often legislation-specific (e.g. `PER_JOBS_LEG_EFF`).
- **Global Data Elements** = the context for **global segments** that apply to every record regardless of context.

### Two pieces of syntax to memorise

| Purpose | Syntax on METADATA | Example |
|---|---|---|
| Declare the flexfield + carry its **context value** | `FLEX:<flexfield code>` | `FLEX:PER_JOBS_DFF` |
| Each **segment** attribute (with a hint) | `<segmentName>(<flexfield code>=<context code>)` | `clearanceLevel(PER_JOBS_DFF=US_JOB_REQ)` |

### DFF example (Job)
```
METADATA|Job|SetCode|JobCode|EffectiveStartDate|FLEX:PER_JOBS_DFF|clearanceLevel(PER_JOBS_DFF=US_JOB_REQ)
MERGE|Job|COMMON|SE01|2020/01/01|US_JOB_REQ|HIGH
```
- On MERGE, the **context value** (`US_JOB_REQ`) goes in the **same position** as `FLEX:PER_JOBS_DFF` on METADATA.
- `HIGH` goes to the `clearanceLevel` segment of the `US_JOB_REQ` context.

### Global segments + context segments together
```
METADATA|Grade|SetCode|GradeCode|EffectiveStartDate|FLEX:PER_GRADES_DF|gradeGlobalName(PER_GRADES_DF=Global Data Elements)|usName(PER_GRADES_DF=US)
MERGE|Grade|COMMON|G5|2020/01/01|US|Grade 5 Global|Grade 5 US
```
- A MERGE line can load **one context** plus the **Global Data Elements** segments.
- Because every segment carries a hint (flexfield + context), you can list attributes for **several flexfields and contexts** on one METADATA line.

> ⚠️ **Classic trap:** To update only a **global** segment on a record that already has context-specific values, pass the **existing context** (e.g. `US`), **not** `Global Data Elements`. If you switch the context to Global Data Elements, the context-specific segment values are **set to null** because the context changed.

### EFF example (structure)
```
METADATA|JobLegislative|EFF_CATEGORY_CODE|FLEX:PER_JOBS_LEG_EFF|JobId(SourceSystemId)|EffectiveStartDate|LegislationCode|SourceSystemOwner|SourceSystemId|_EEO1_CATEGORY(PER_JOBS_LEG_EFF=HRX_US_JOBS)
MERGE|JobLegislative|JOB_LEG|HRX_US_JOBS|SE01_SRC|2020/01/01|US|LEGACY|SE01_LEG_US|1
```
- EFF data is loaded as its **own component** (here `JobLegislative`).
- Needs `EFF_CATEGORY_CODE` (category) + `FLEX:<code>` (context) + hinted segments.
- **Each EFF MERGE line can have only one context.**

### Tip
The **generated template** from *View Business Objects* (after **Refresh Object**) already includes `FLEX:` codes and hinted segment names for your configured flexfields. **Copy them from there** instead of typing them by hand.

---

## 6. SET Instructions

**Rules:** optional; any number allowed; **must come before the first METADATA line**; they affect **only that `.dat` file**. Many have an enterprise-wide default in *Configure HCM Data Loader* that SET overrides.

| Instruction | Default | What it does |
|---|---|---|
| `SET PURGE_FUTURE_CHANGES Y\|N` | **Y** | **Y = Replace mode**: future-dated changes in the date range of your data are **purged/replaced**. **N = Retain mode**: existing future-dated changes are **kept**. Oracle best practice: use **N for updates**. |
| `SET FILE_DELIMITER <char>` | `\|` | Change the delimiter (up to 10 chars) |
| `SET FILE_ESCAPE <char>` | `\` | Change the escape character |
| `SET FILE_NEW_LINE <value>` | `n` | Change the newline representation |
| `SET ENABLE_AUDIT_DATA Y\|N` | N | Audit the changes made by this file. Oracle advises enabling it for **incremental loads, not data migration** (performance). |
| `SET PURGE_AUDIT_DATA Y\|N` | N | Purge audit data for objects in the file (e.g. removing PII). Can't be combined with `ENABLE_AUDIT_DATA Y` in the same file. |
| `SET INVOKE_POST_PROCESS Y\|N` | Y | Run (or skip) the post-load processes automatically |
| `SET DISABLE_POST_PROCESS_TASKS <process>` | — | Skip a **specific** post-process, e.g. `FlattenPositionHierarchy` |
| `SET CALCULATE_FTE Y\|N` | N | Auto-calculate FTE on assignments |
| `SET CREATE_DEFAULT_WORKING_HOUR_PATTERN Y\|N` | N | Create a default working-hour pattern |
| `SET ENABLE_INCREMENTAL_LOAD_EVENTS Y\|N` | — | Fire **events/atom feeds** for the changes (e.g. so payroll events or downstream subscribers see them) |

> The list grows each release (e.g. Oracle Search ingest, succession-plan history options). The **"The SET Instruction"** topic in the HDL guide has the current full list.

### Most important one: PURGE_FUTURE_CHANGES
Scenario: an assignment has a **future-dated** change on 2027/01/01 (promotion). Today you load a location change effective 2026/12/01.

| Mode | Result for the 2027/01/01 promotion |
|---|---|
| `SET PURGE_FUTURE_CHANGES Y` (default) | ❌ Can be **purged/replaced** within the date range of your data |
| `SET PURGE_FUTURE_CHANGES N` | ✅ **Kept** |

**Rule of thumb:** use **Y** when creating or reloading full history (conversion), and **N** when updating existing records (incremental).

---

## 7. Full Annotated Example: Worker.dat

A new hire, **Ravi Sharma**, joining the legal employer "Example India Pvt Ltd" on 2026/01/05.

```
SET PURGE_FUTURE_CHANGES N
COMMENT New hire – Ravi Sharma – Batch 01

METADATA|Worker|SourceSystemOwner|SourceSystemId|EffectiveStartDate|EffectiveEndDate|PersonNumber|StartDate|DateOfBirth|ActionCode
MERGE|Worker|LEGACY|EMP1001|2026/01/05|4712/12/31|1001|2026/01/05|1998/04/12|HIRE

METADATA|PersonName|SourceSystemOwner|SourceSystemId|PersonId(SourceSystemId)|EffectiveStartDate|EffectiveEndDate|LegislationCode|NameType|FirstName|LastName
MERGE|PersonName|LEGACY|EMP1001_NAME|EMP1001|2026/01/05|4712/12/31|IN|GLOBAL|Ravi|Sharma

METADATA|PersonEmail|SourceSystemOwner|SourceSystemId|PersonId(SourceSystemId)|DateFrom|EmailType|EmailAddress|PrimaryFlag
MERGE|PersonEmail|LEGACY|EMP1001_EMAIL_W1|EMP1001|2026/01/05|W1|ravi.sharma@example.com|Y

METADATA|PersonLegislativeData|SourceSystemOwner|SourceSystemId|PersonId(SourceSystemId)|EffectiveStartDate|EffectiveEndDate|LegislationCode|Sex
MERGE|PersonLegislativeData|LEGACY|EMP1001_LEG_IN|EMP1001|2026/01/05|4712/12/31|IN|M

METADATA|WorkRelationship|SourceSystemOwner|SourceSystemId|PersonId(SourceSystemId)|LegalEmployerName|DateStart|WorkerType|PrimaryFlag|ActionCode
MERGE|WorkRelationship|LEGACY|EMP1001_WR|EMP1001|Example India Pvt Ltd|2026/01/05|E|Y|HIRE

METADATA|WorkTerms|SourceSystemOwner|SourceSystemId|PeriodOfServiceId(SourceSystemId)|EffectiveStartDate|EffectiveEndDate|EffectiveSequence|EffectiveLatestChange|AssignmentNumber|ActionCode
MERGE|WorkTerms|LEGACY|EMP1001_WT|EMP1001_WR|2026/01/05|4712/12/31|1|Y|ET1001|HIRE

METADATA|Assignment|SourceSystemOwner|SourceSystemId|WorkTermsAssignmentId(SourceSystemId)|EffectiveStartDate|EffectiveEndDate|EffectiveSequence|EffectiveLatestChange|AssignmentNumber|ActionCode|BusinessUnitShortCode|DepartmentName|JobCode|GradeCode|LocationCode|AssignmentStatusTypeCode|PersonTypeCode
MERGE|Assignment|LEGACY|EMP1001_ASG|EMP1001_WT|2026/01/05|4712/12/31|1|Y|E1001|HIRE|IN_BU|Engineering|SE01|G5|CHN01|ACTIVE_PROCESS|Employee
```

### Reading it line by line
| Component | What to notice |
|---|---|
| `SET` | First line, before any METADATA. Retain mode is safe for future updates. |
| `Worker` | Top-level person. `ActionCode = HIRE`. Source key `LEGACY` + `EMP1001`. |
| `PersonName` | Points to its parent through `PersonId(SourceSystemId) = EMP1001`. `NameType = GLOBAL`. |
| `PersonEmail` | Not date-effective, so it uses `DateFrom` instead of effective dates. `W1` = work email code. |
| `PersonLegislativeData` | Gender is legislation-specific, so it lives here, not on Worker. Code `M`, not "Male". |
| `WorkRelationship` | Child of the person. Legal employer, start date, worker type `E`. |
| `WorkTerms` | Child of the relationship: `PeriodOfServiceId(SourceSystemId) = EMP1001_WR`. `_M` data, so it has `EffectiveSequence`/`EffectiveLatestChange`. |
| `Assignment` | Child of the work terms: `WorkTermsAssignmentId(SourceSystemId) = EMP1001_WT`. References workforce structures by code/name, so **they must already exist**. |

> **Illustrative file.** Attribute names and which ones are mandatory depend on your environment and release. Always generate the template from **View Business Objects → Worker** and build from that.

### The parent–child chain (follow the keys)
```
Worker            EMP1001
 ├─ PersonName        → PersonId(SourceSystemId)=EMP1001
 ├─ PersonEmail       → PersonId(SourceSystemId)=EMP1001
 ├─ PersonLegislative → PersonId(SourceSystemId)=EMP1001
 └─ WorkRelationship EMP1001_WR → PersonId=EMP1001
      └─ WorkTerms   EMP1001_WT → PeriodOfServiceId=EMP1001_WR
           └─ Assignment EMP1001_ASG → WorkTermsAssignmentId=EMP1001_WT
```

---

## 8. Common File-Level Errors

These are **import-phase** (format) errors. Load-phase (business rule) errors come in Phase 5.

| Error symptom | Likely cause | Fix |
|---|---|---|
| File not recognised / no business object found | Wrong `.dat` name (`Employee.dat`), or `.dat` inside a sub-folder of the zip | Rename to the exact object name; put it at the zip root |
| "Attribute … is unknown" | Typo in the attribute name, or an attribute that doesn't exist for that object version/component | Copy names from the generated template |
| "METADATA line can't be processed" | Wrong flexfield hint syntax / context code, or object not refreshed | Check `name(FLEX_CODE=CONTEXT)`; refresh the object; regenerate the template |
| Data line has the wrong number of values | Unescaped `|` in a value, or missing/extra pipes | Escape as `\|`; count the delimiters |
| Invalid date | Not `YYYY/MM/DD` | Reformat dates |
| SET instruction ignored / error | SET placed **after** a METADATA line | Move all SET lines to the top |
| Data line without a header | MERGE for a discriminator with no METADATA line above it | Add the METADATA line first |
| Garbled characters (é, हिंदी) | File not saved as UTF-8 | Re-save as UTF-8 |

---

# PHASE 2 CHEAT SHEET

| Topic | Remember this |
|---|---|
| Data set | One **zip** = one data set; `.dat` files at the zip root |
| File name | = business object name (`Worker.dat`) |
| Line types | `METADATA`, `MERGE`, `DELETE`, `SET`, `COMMENT` (no INSERT/UPDATE) |
| MERGE | Create if new, update if it exists (decided by keys) |
| Discriminator | Position 2 on METADATA/MERGE/DELETE = component name |
| Attribute order | Free on METADATA, but data lines must match it |
| Parent reference | `PersonId(SourceSystemId)`, `PeriodOfServiceId(...)`, `WorkTermsAssignmentId(...)` |
| Delimiter / escape / newline | `|` / `\` / `\n`; escape a pipe as `\|` |
| Date | `YYYY/MM/DD`; end of time `4712/12/31` |
| Blank vs #NULL | Blank = keep existing; `#NULL` = clear |
| DFF syntax | `FLEX:<CODE>` + `segment(CODE=CONTEXT)`; context value goes under `FLEX:` |
| Global segments | Context `Global Data Elements`; but when updating globals, pass the **existing** context |
| EFF | Own component + `EFF_CATEGORY_CODE` + `FLEX:` + one context per line |
| SET placement | Before the **first METADATA** line |
| PURGE_FUTURE_CHANGES | Default **Y** (replace); use **N** for updates (retain future changes) |
| Source of truth | Generated template from View Business Objects (refresh first) |

---

# PRACTICE QUESTIONS & ANSWERS

### Section A — Concept checks

**Q1. Name the five instruction types in an HDL file. Which two have no discriminator?**
<details><summary>Answer</summary>

`METADATA`, `MERGE`, `DELETE`, `SET`, `COMMENT`. **SET** and **COMMENT** have no discriminator.
</details>

**Q2. Why is there no UPDATE instruction in HDL?**
<details><summary>Answer</summary>

Because **MERGE** handles both cases: HDL uses the record's keys to check whether it already exists. If it does, the record is updated; if not, it's created.
</details>

**Q3. What is a discriminator? Give three examples from Worker.dat.**
<details><summary>Answer</summary>

The component name in the 2nd position of METADATA/MERGE/DELETE lines. It tells HDL which component the line belongs to. Examples: `PersonName`, `WorkRelationship`, `Assignment` (also `Worker`, `PersonEmail`, `WorkTerms`…).
</details>

**Q4. Does the attribute order on the METADATA line have to match Oracle's template order?**
<details><summary>Answer</summary>

No. You can choose any order and include only the attributes you need. But every MERGE/DELETE line for that discriminator must supply values in the **same order as its METADATA line**.
</details>

**Q5. A value contains a pipe: `R&D | Analytics`. How do you write it?**
<details><summary>Answer</summary>

Escape the pipe with a backslash: `R&D \| Analytics`. Alternatively, change the delimiter with `SET FILE_DELIMITER` at the top of the file.
</details>

**Q6. Where must SET instructions be placed, and what is their scope?**
<details><summary>Answer</summary>

Before the **first METADATA line** of the `.dat` file. They affect **only that file**, overriding enterprise defaults from Configure HCM Data Loader.
</details>

**Q7. What is the default value of PURGE_FUTURE_CHANGES and what does each value mean?**
<details><summary>Answer</summary>

Default **Y** = **Replace** mode: existing future-dated changes in the date range of your data are purged/replaced. **N** = **Retain** mode: existing future-dated changes are kept. Oracle recommends `N` when updating existing records.
</details>

**Q8. Write the METADATA syntax for a DFF segment `laptopTag` in context `IT_ASSETS` of flexfield `PER_ASG_DF`.**
<details><summary>Answer</summary>

```
...|FLEX:PER_ASG_DF|laptopTag(PER_ASG_DF=IT_ASSETS)
```
and on the MERGE line, put `IT_ASSETS` in the position of `FLEX:PER_ASG_DF` and the tag value in the position of `laptopTag(...)`.
</details>

### Section B — Scenario-based (interview style)

**S1. A user loaded a phone update with the Extension column left blank, hoping to remove the extension. It's still there. Why, and how do you fix it?**
<details><summary>Answer</summary>

On an update, a **blank** value means "don't change". HDL keeps the existing value. To clear it, send **`#NULL`** in that position: `MERGE|PersonPhone|...|9840012345|#NULL`.
</details>

**S2. Your zip `Jan_Hires.zip` contains a folder `files/` with `Worker.dat` inside. Import says no business objects found. What's wrong?**
<details><summary>Answer</summary>

The `.dat` file must be at the **root** of the zip, not inside a sub-folder. Re-zip with `Worker.dat` at the top level. Also confirm the file name exactly matches the business object name.
</details>

**S3. You updated a global DFF segment on 500 jobs using context `Global Data Elements`. Afterwards, their `US`-context segment values became empty. Explain and fix.**
<details><summary>Answer</summary>

Passing `Global Data Elements` **changed the record's context** from `US` to global, so the `US`-context segments were set to null. When updating a global segment on records that already have a context, pass the **existing context value** (`US`). To repair it, reload the jobs with context `US`, the global value, and the original US segment values (from a backup/extract).
</details>

**S4. An employee has a promotion already entered for 2027/01/01. You load a department change effective 2026/12/01 via HDL, and the promotion disappears. What happened, and how do you prevent it?**
<details><summary>Answer</summary>

The file ran in the default **Replace** mode (`PURGE_FUTURE_CHANGES Y`), which purges future-dated changes in the range of the update. Add **`SET PURGE_FUTURE_CHANGES N`** at the top of the file (before METADATA) so future-dated changes are retained. Then re-create the lost promotion.
</details>

**S5. Import fails with "The METADATA line can't be processed because the attribute territory(PER_ORGANIZATION_UNIT_DFF=Global Data … is unknown". List what you'd check.**
<details><summary>Answer</summary>

1. The **hint syntax** is complete and exact: `territory(PER_ORGANIZATION_UNIT_DFF=Global Data Elements)` (the error shows a truncated or mistyped context).
2. The **segment code** (`territory`) matches the configured segment's API name.
3. The flexfield is **deployed** and the object has been **refreshed** in View Business Objects.
4. The `FLEX:PER_ORGANIZATION_UNIT_DFF` attribute is also on the METADATA line.
Best fix: regenerate the template and copy the exact attribute names.
</details>

**S6. Spot the errors in this file:**
```
METADATA|Job|SetCode|JobCode|Name|EffectiveStartDate
SET PURGE_FUTURE_CHANGES N
MERGE|Job|COMMON|AN01|Analyst, Data|01-04-2026
MERGE|Job|COMMON|QA01|QA | Testing|2026/04/01
```
<details><summary>Answer</summary>

1. `SET` is **after** METADATA. Move it to the very top.
2. Date `01-04-2026` is wrong. Use `2026/04/01`.
3. `QA | Testing` has an **unescaped pipe**, which gives an extra column. Write `QA \| Testing`.
(The comma in `Analyst, Data` is fine because the delimiter is a pipe.)
</details>

**S7. Why might an HDL file author prefer the pipe delimiter over commas?**
<details><summary>Answer</summary>

Commas are common in real data (addresses, names like "Kumar, Aditya", job titles), so a comma delimiter would need lots of escaping and is error-prone. Pipes rarely appear in HR data.
</details>

