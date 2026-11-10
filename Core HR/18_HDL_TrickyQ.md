# Oracle HDL — Tricky Interview Questions & Answers

> Companion to the HDL study-guide series (Phases 1, 2, 3, 6).
> **Part A** answers your 6 questions and says whether your answer was right. **Part B** adds ~45 more tricky questions across all of HDL.
> Answers are checked against the Oracle HCM Data Loader guide, the HCM Business Objects guide, and Oracle's HDL tutorials.

---

## Table of Contents
- [Part A — Your 6 Questions](#part-a--your-6-questions)
- [Part B — More Tricky Questions](#part-b--more-tricky-questions)
  - [B1. Architecture & Process](#b1-architecture--process)
  - [B2. File Syntax Traps](#b2-file-syntax-traps)
  - [B3. Keys](#b3-keys)
  - [B4. Date-Effective Data (the hardest area)](#b4-date-effective-data-the-hardest-area)
  - [B5. Worker Scenarios](#b5-worker-scenarios)
  - [B6. Deleting Data](#b6-deleting-data)
  - [B7. Errors, Monitoring & Recovery](#b7-errors-monitoring--recovery)
  - [B8. Flexfields](#b8-flexfields)
- [One-Page Rapid Revision](#one-page-rapid-revision)

---

# PART A — YOUR 6 QUESTIONS

### A1. Explain the architecture of HDL.
**Answer (say it as a flow):**
1. **Source**: data is prepared as pipe-delimited `.dat` files (one per business object, e.g. `Worker.dat`) and **zipped**. The zip is a **data set**.
2. **WebCenter Content (UCM)**: the zip is uploaded to the **`hcm/dataloader/import`** account and gets a **Content ID**. This is done by the UI (*Import File*), File Import and Export, or automatically through the WebCenter Content APIs.
3. **Submit**: through the **Import and Load Data** page or the **HCM Data Loader web service**, with a File Action (*Import only / Import and load*) and parameters (error thresholds, threads, encryption).
4. **Import phase (file → stage tables)**: unzip, identify each object by **file name**, read SET and METADATA, check each **physical line** for **format errors**, write to the **stage tables**, and group lines into **logical objects** using keys.
5. **Load phase (stage → application tables)**: resolve keys (create vs update), resolve parent and foreign references, run the **same business validations as the UI**, and write to the base tables plus the **Integration Key Map**. Each **logical object succeeds or fails as a whole**. Processing runs in **parallel threads**.
6. **Post-processes** run (e.g. hierarchy refreshes).
7. **Monitoring and error correction**: Data Exchange → Import and Load Data (Failed Lines, Failed Objects, Object Hierarchy). Fix the data, then resubmit or reload.

> **Bonus point:** HDL never writes to base tables directly. It always goes **File → UCM → Stage → Application**.

---

### A2. Which discriminators are mandatory in Worker.dat?
**Answer: for a new hire, five components are mandatory:**

| # | Discriminator | Why |
|---|---|---|
| 1 | `Worker` | The person record (ActionCode `HIRE`) |
| 2 | `PersonName` | Every person needs a name |
| 3 | `WorkRelationship` | The relationship with the legal employer |
| 4 | `WorkTerms` | Links the relationship to the assignment |
| 5 | `Assignment` | The actual job assignment |

- **`PersonLegislativeData`** (gender, marital status) is **recommended** and usually supplied. Oracle's new-hire tutorial covers it alongside the mandatory set.
- **`PersonEmail`, `PersonPhone`, `PersonAddress`, `PersonNationalIdentifier`** are **optional**.

**Tricky follow-ups:**
- *"Employment terms aren't visible in the UI anymore. Do I still need WorkTerms?"* **Yes.** You still supply a **skeleton WorkTerms** record to link the WorkRelationship and the Assignment.
- *"Which component is needed to terminate?"* **Only `WorkRelationship`** (see B5).

---

### A3. When correcting an existing field, what is the relationship between EffectiveStartDate and EffectiveEndDate?
**Your answer:** "Keep the start date the same and leave the end date blank." ⚠️ **Partly right.** It's only safe for the **latest** record.

**Correct answer:**
- **EffectiveStartDate = the start date of the existing record (split) you're correcting.** A matching start date tells HDL to **correct** that split instead of creating a new one.
- **EffectiveEndDate = the existing record's end date.**
  - If it's the **latest** record, its end date is end of time, so **blank or `4712/12/31`** gives the same result. That's why your answer works there.
  - If **later (future) records exist**, a blank end date means "until end of time". In the default **Replace** mode that **replaces the future records too**. So supply the record's **actual end date**, or use `SET PURGE_FUTURE_CHANGES N` with `#RETAIN`.

```
-- Existing: 2024/01/01–2025/03/31 (Sales) and 2025/04/01–4712/12/31 (Marketing)
-- Correct the FIRST split (department was wrong):
SET PURGE_FUTURE_CHANGES N
METADATA|Assignment|SourceSystemOwner|SourceSystemId|EffectiveStartDate|EffectiveEndDate|...|DepartmentName
MERGE|Assignment|LEGACY|EMP1001_ASG|2024/01/01|2025/03/31|...|Pre-Sales
```
> Oracle's rule for correcting an assignment termination is the same: **pass the EffectiveStartDate and EffectiveEndDate of the existing record.**

---

### A4. When updating an existing field (a new change), what is the relationship between EffectiveStartDate and EffectiveEndDate?
**Answer:**
- **EffectiveStartDate = the date the change takes effect.** It's a **new** date that doesn't match any existing split, so HDL **creates a new split**.
- HDL **automatically end-dates the previous split** to the day before.
- **EffectiveEndDate:**
  - **Blank or `4712/12/31`** when the change should run to the end of time (no future records).
  - If **future-dated records exist**, use **Retain mode** (`SET PURGE_FUTURE_CHANGES N`) and choose:
    - `#RETAIN`: insert the change only up to the next future record and leave future values untouched
    - `#ALL`: carry the change into all future records
    - a specific date: carry it forward until that date
- For **Assignment**, WorkTerms and Assignment must be in the **same file** with the **same EffectiveStartDate and EffectiveEndDate**. If the assignment record is the latest, its end date must be **end of time**.

```
-- Transfer effective 2026/12/01 (no future records)
SET PURGE_FUTURE_CHANGES N
METADATA|Assignment|SourceSystemOwner|SourceSystemId|EffectiveStartDate|EffectiveEndDate|EffectiveSequence|EffectiveLatestChange|ActionCode|DepartmentName
MERGE|Assignment|LEGACY|EMP1001_ASG|2026/12/01|4712/12/31|1|Y|TRANSFER|Analytics
```
Result: the old split is end-dated **2026/11/30** and the new split starts **2026/12/01**.

**Correction vs update in one line:**
| | Start date | Result |
|---|---|---|
| **Correction** | **Matches** an existing split | That split is overwritten (history rewritten) |
| **Update** | **New** date | New split created; previous split end-dated |

---

### A5. How do you clear an existing field's value?
**Your answer:** `#NULL`. ✅ **Correct.**
- **Blank** → the existing value is **kept** (the attribute is ignored on update).
- **`#NULL`** → the value is **cleared**.
- **Extra tricky point:** when loading **date-effective history** in Replace mode, supply `#NULL` for every attribute that must be null in a later split. A blank value **rolls forward** from the previous split.

---

### A6. On an _M table with multiple changes on one day, what is the start/end date relationship for rows where EffectiveLatestChange = N?
**Your answer:** They're equal. ✅ **Correct.**

Full rule set for multiple changes per day (e.g. Assignment):
| Rule | Detail |
|---|---|
| `EffectiveEndDate = EffectiveStartDate` | For every row where `EffectiveLatestChange = N` |
| `EffectiveLatestChange = Y` | Only for the **last** change on that date. If only one change exists on a date, it **must be Y**. |
| `EffectiveSequence` | Starts at **1** for each new date and increases **sequentially** |
| Same key | The unique key is identical across all date-effective rows |
| No gaps | No break in dates across the history |
| Highest sequence wins | Changes are cumulative. The highest sequence is what transactions see for that date. |
| Blank sequence | Allowed for a single update only if the file has `SET PURGE_FUTURE_CHANGES N`. Otherwise the processing order for same-date rows is undefined. |

```
MERGE|Assignment|LEGACY|EMP1001_ASG|2026/07/01|2026/07/01|1|N|TRANSFER|...   ← N: end = start
MERGE|Assignment|LEGACY|EMP1001_ASG|2026/07/01|4712/12/31|2|Y|PROMOTION|...  ← Y: latest
```

---

# PART B — MORE TRICKY QUESTIONS

## B1. Architecture & Process

**1. What's the difference between an import error and a load error?**
Import errors are **file/format** problems (unknown attribute, bad date, unescaped pipe), found while moving file → stage. Load errors are **data/business-rule** problems (invalid department, missing parent, date overlap), found while moving stage → application tables.

**2. Can you reprocess the same zip after fixing an import error?**
**No.** Correct the source, put the file in a **new zip**, and import again. The existing file can't be processed again.

**3. Two ways to fix a load error?**
(1) Fix the source data and load a new zip. (2) Correct the value in **Object Hierarchy → Attribute Details** and **resubmit the load**.

**4. What's a logical object, and why does it matter?**
All file lines that make up one instance (e.g. one worker and its names, relationship, terms, assignment). It is the **unit of success/failure**: one bad line fails the whole worker, and other workers still load.

**5. You stopped an import and a load. Which can be restarted?**
Only the **load**. A stopped import can't be restarted, so you submit the file again.

**6. Is HDL line order guaranteed? Why does it matter?**
**No.** That's why you must **never put MERGE and DELETE for the same record in one file**, and why parents are linked by **keys**, not by line position.

**7. One zip with all objects, or one zip per object?**
Oracle's current best-practice guide says to supply all `.dat` files in one zip, and HDL processes them in dependency order. Older guides said one object per zip. Conversions often still load object by object to control errors and reconcile.

**8. Difference between "Maximum Percentage of Import Errors" and "Maximum Percentage of Load Errors"?**
Import % is measured on **file lines**. Load % is measured on **business-object instances**. Beyond the threshold, processing **stops** for that object.

**9. Which role do you need for HDL?**
**Human Capital Management Integration Specialist** (HDL tasks + UCM import access).

**10. Where does HDL pick up files from?**
**WebCenter Content (UCM)** account **`hcm/dataloader/import`**.

## B2. File Syntax Traps

**11. Are `.dat` file names case-sensitive?**
**Yes.** Business-object file names are case-sensitive: `Worker.dat`, not `worker.dat`.

**12. The zip has `Worker.dat` inside a folder. Result?**
Not recognised. `.dat` files must be at the **zip root**.

**13. Where must SET instructions go?**
**Before the first METADATA line.** They apply only to that file.

**14. Is there an UPDATE or INSERT instruction?**
**No.** `MERGE` creates or updates (decided by keys). The other instructions are `DELETE`, `SET`, `COMMENT`, `METADATA`.

**15. Must METADATA attributes follow Oracle's template order?**
**No.** Any order and any subset you need, but data lines must match their METADATA line.

**16. A value contains `|`. What do you do?**
Escape it as `\|`, or change the delimiter with `SET FILE_DELIMITER`.

**17. Default date format and end of time?**
`YYYY/MM/DD`; `4712/12/31`.

**18. Can you send "Male" for gender?**
**No.** Send **codes** (`M`), not meanings.

## B3. Keys

**19. Key resolution order?**
**GUID → Source Key → Surrogate ID → User Key.**

**20. You send a GUID for record A and a source key for record B. What happens?**
The **GUID wins** and record A is updated. The source key is ignored. **No error is raised**, because there's no cross-validation.

**21. Which keys can be used to create?**
Only **source keys** and **user keys**. GUIDs and surrogate IDs exist only after creation.

**22. Default source key for a record created in the UI?**
Owner **`FUSION`**, ID = **surrogate ID**.

**23. Load fails with invalid SourceSystemOwner. Fix?**
Add the owner to the **`HRC_SOURCE_SYSTEM_OWNER`** lookup (Manage Common Lookups).

**24. Can you use `JobId(SourceSystemId)` on a line that has no local source key?**
**No.** Source keys for foreign references are allowed only if the local record also uses a source key.

**25. How do you change a source key?**
Load **`SourceKey.dat`** (BusinessObject, Component, old owner/ID, new owner/ID). Don't send a MERGE with the new key, because that creates a duplicate.

**26. Where are source keys stored?**
**`HRC_INTEGRATION_KEY_MAP`**. They aren't on the object itself. There's no standard extract, so query the table with BIP.

**27. Why can't you update most Worker components using only a user key?**
Because the attribute you want to change may be the one used to **identify** the record (e.g. changing a name when the name identifies the row). Use a source key or surrogate ID.

**28. Date-effective object: how many source keys across its history?**
**One.** The key is the same across all date-effective records.

## B4. Date-Effective Data (the hardest area)

**29. Default maintenance mode, and what does it do?**
**Replace** (`SET PURGE_FUTURE_CHANGES Y`): the file's contents **replace** existing date-effective records in the date range, which can **purge future-dated changes**. Oracle recommends **Retain** (`N`) for incremental updates and Replace for full-history migration.

**30. `#RETAIN` vs `#ALL` in the EffectiveEndDate (Retain mode)?**
`#RETAIN`: future records keep their existing values; your change applies only up to the next future record (the old EBS "Update Change Insert"). `#ALL`: your changed values are **carried forward into all future records**, and all records are kept.

**31. In Retain mode, is a blank end date the same as `#ALL`?**
Leaving it blank or setting `4712/12/31` has the same effect as `#ALL`. But for objects with a **specific end date**, you must use `#ALL` to carry the change forward.

**32. Replace mode, you send only one attribute for a date with future records. Danger?**
Future records are replaced by one record, and attributes you didn't send **roll forward from the earlier split**. Later changes (e.g. FT/PT) can be **lost**. Oracle warns that Replace mode may have unexpected consequences if you upload less than the complete data for a record.

**33. Can you change ActionCode on an existing split with an update?**
**No.** Some attributes, such as **ActionCode**, are **protected from update**.

**34. Can there be gaps between date-effective records?**
**No.** History must be continuous ("Date-effective records can't have gaps…"). Records can appear in any order in the file, but the dates can't break.

**35. When loading history, why use `#NULL` instead of blank in later rows?**
Blank **rolls forward** the previous split's value. `#NULL` explicitly makes it null in that split.

**36. Do the "Update Change Insert / Override" modes apply only to `_F` tables?**
**No.** They apply to **all date-effective objects**, including `_M` (assignments). They don't apply to non-date-effective tables (emails, phones). The EBS names map to HDL's **Retain + #RETAIN** and **Replace**.

**37. Assignment change: must WorkTerms be included?**
**Yes.** Load **WorkTerms and Assignment together** in the same file with the **same EffectiveStartDate/EffectiveEndDate**. Otherwise you get **"WorkTerms records required."**

## B5. Worker Scenarios

**38. New hire rules HDL checks?**
- `ActionCode = HIRE`
- On Worker, **EffectiveStartDate must equal StartDate**
- **PersonName** (and **PersonLegislativeData**) EffectiveStartDate must **match the Worker's**
- If person numbers are **auto-generated**, you can't supply PersonNumber, so a **source key becomes mandatory**

**39. How do you terminate a worker?**
You terminate the **work relationship**, not the worker. Load **only `WorkRelationship`** with **`ActualTerminationDate`**, a termination **ActionCode** (e.g. `TERMINATION`/`RESIGNATION`, as configured) and **`TerminateWorkRelationshipFlag = Y`**. Child assignments are **terminated automatically**.

**40. Can you include Assignment/WorkTerms lines in the same termination file?**
**No.** Error: *"Work terms or assignment can't be specified when terminating a work relationship."*

**41. Should you also end-date PersonName when terminating?**
**No.** Oracle says not to terminate other child components such as Person Name.

**42. How do you reverse a termination?**
WorkRelationship with **`ReverseTerminationFlag = Y`** (commonly used; confirm the attribute in View Business Objects for your release). The **same** relationship is reactivated, not a new one.

**43. Rehire vs reverse termination?**
**Rehire** = a **new** WorkRelationship (+ WorkTerms + Assignment) with `ActionCode = REHIRE`; the person record is the same. **Reverse termination** = undo the termination on the **existing** relationship.

**44. Correcting an assignment termination date: prepone vs postpone?**
- **Correct without changing the date:** pass the existing termination record's EffectiveStartDate and EffectiveEndDate.
- **Prepone** (earlier date): allowed even if future updates exist after the termination record. **Those future records are deleted.**
- **Postpone** (later date): **not allowed** if future updates exist after the termination record.

**45. How is a global transfer done?**
Load a **new WorkRelationship** (new legal employer) **with all its child components** (WorkTerms, Assignment). There's also a `GlobalTransferFlag` approach, with `SourceAssignmentNumber`/`SourceAssignmentId` on the assignment.

**46. New hires and their managers (also new) are in the same file, and supervisor lines fail. Why?**
The managers didn't exist yet when their reports were processed (parallel processing, no guaranteed order). Load the workers first, then the `AssignmentSupervisor` lines in a second pass.

**47. Is PersonEmail mandatory?**
**No.** It's optional.

## B6. Deleting Data

**48. Can you delete a Worker with HDL?**
**No.** **Worker objects can't be deleted.** You **can** delete some Worker child components, e.g. **PersonEmail**.

**49. Can you delete a single date-effective split with DELETE?**
**No.** You can't delete **individual date-effective records** with DELETE (or individual translation objects). Fix history with corrections or Replace-mode loads instead.

**50. What happens to children when you delete a parent?**
Child components and translations are **deleted too**.

**51. Do you need to have loaded a record through HDL to delete it?**
**No.** The component doesn't have to have been loaded with HDL.

**52. Can deleted records be recovered?**
**No.** Oracle advises correcting data rather than deleting and recreating it.

**53. How do you know whether a component supports DELETE?**
Check the business-object details in **View Business Objects**. Unsupported deletes give errors like *"…Assignment, which can't be deleted."*

## B7. Errors, Monitoring & Recovery

**54. 1,000 workers, the load stopped at ~12% failures and the rest are "Not processed". Why?**
The **Maximum Percentage of Load Errors** threshold was exceeded. Find the common cause (Message Summary), fix it, and **resubmit the load**.

**55. How do you undo a completed bad load?**
**Roll Back** (File Action) **if the object supports it**. Otherwise use corrective MERGE files, or DELETE where allowed.

**56. What do you share with Oracle Support for an HDL issue?**
The **HCM Data Loader Error Analysis Report**, run with the **Content ID or Process ID** (+ business object, **Detailed Mode = Y**).

**57. You fixed errors in the UI during Mock 1, but they come back in Mock 2. Why?**
UI corrections change only the **stage** data. The **source/mapping** wasn't fixed.

**58. Load errors can't be corrected in the UI for one Worker file. Why?**
That file had **`SET ENABLE_AUDIT_DATA Y`**. Load errors from it can't be corrected in the UI.

**59. Stage tables are huge and loads are slow. What do you do?**
Run **Delete Stage Table Data** regularly (Oracle best practice for large volumes).

**60. Name the main HDL stage tables.**
`HRC_DL_DATA_SETS`, `HRC_DL_DATA_SET_BUS_OBJS`, `HRC_DL_PHYSICAL_LINES`, `HRC_DL_LOGICAL_LINES`, `HRC_DL_MESSAGE_LINES` (+ `HRC_INTEGRATION_KEY_MAP` for keys).

## B8. Flexfields

**61. Syntax to load a DFF segment?**
`FLEX:<FLEX_CODE>` holds the **context value**. Each segment is written `segmentName(<FLEX_CODE>=<CONTEXT>)`.

**62. You update a global segment using context "Global Data Elements" and context-specific values disappear. Why?**
You **changed the context**, so the old context's segments were nulled. When updating global segments, pass the **existing context** value.

**63. A new DFF segment doesn't appear in the template. Fix?**
**Refresh Object** in View Business Objects, then regenerate the template.

**64. EFF loading: how many contexts per MERGE line?**
**One** (plus `EFF_CATEGORY_CODE` and `FLEX:` code).

---

# ONE-PAGE RAPID REVISION

| Topic | Answer |
|---|---|
| Mandatory Worker components | Worker, PersonName, WorkRelationship, WorkTerms, Assignment (+ PersonLegislativeData recommended) |
| Terminate | WorkRelationship **only**, ActualTerminationDate + ActionCode + TerminateWorkRelationshipFlag=Y |
| Correction | Start date **matches** the existing split; end date = that split's end date |
| Update | **New** start date; blank/4712 end date; previous split auto end-dated |
| Clear a value | `#NULL` (blank = keep) |
| Multiple changes/day, N rows | End date = start date; sequence 1..n; last row Y |
| Default mode | Replace (`PURGE_FUTURE_CHANGES Y`); use `N` for updates |
| `#RETAIN` / `#ALL` | Keep future values / push change into all future records |
| Key order | GUID → Source → Surrogate → User (no cross-validation) |
| Create with | Source key or user key only |
| Delete | Not Worker, not individual date-effective splits; some children (PersonEmail) OK |
| Assignment change | WorkTerms + Assignment, same file, same dates |
| Import error fix | New zip (can't reprocess the same file) |
| Load error fix | New zip **or** Attribute Details + resubmit load |
| Stop | Load restartable; import not |
| File names | Case-sensitive, at the zip root |
| SET | Before the first METADATA |