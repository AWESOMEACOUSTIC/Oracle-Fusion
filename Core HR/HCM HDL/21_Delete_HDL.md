# Oracle HDL — DELETE Deep Dive: Rules, Edge Cases & Alternatives

> Companion to the HDL study-guide series.
> Goal: know exactly what `DELETE` does, which dates and keys to send, what can and can't be deleted, and which **non-delete** technique to use when you really want to **end-date**, **cancel**, **remove future rows**, or **undo a load**.
> Rules are from the Oracle HCM Data Loader guide and the HCM Business Objects guide. Items marked *(field note)* come from practitioner/community experience: confirm them in your environment.

---

## Table of Contents
1. [Quick Answers to Your Questions](#1-quick-answers-to-your-questions)
2. [What DELETE Really Does: Purge vs End-Date](#2-what-delete-really-does-purge-vs-end-date)
3. [DELETE Syntax, Keys & Dates](#3-delete-syntax-keys--dates)
4. [What You Can and Can't Delete](#4-what-you-can-and-cant-delete)
5. [Worker Components: Special Delete Rules](#5-worker-components-special-delete-rules)
6. [Date-Effective Edge Cases](#6-date-effective-edge-cases)
7. [Parent–Child, Cascades & Translations](#7-parentchild-cascades--translations)
8. [File-Level Rules (MERGE + DELETE, Ordering)](#8-file-level-rules-merge--delete-ordering)
9. [Payroll & Compensation Objects](#9-payroll--compensation-objects)
10. [Decision Matrix: "I Want To…" → Use This](#10-decision-matrix-i-want-to--use-this)
11. [Safety Checklist Before Any Delete](#11-safety-checklist-before-any-delete)
12. [Common Delete Errors](#12-common-delete-errors)
13. [Interview Questions on DELETE](#13-interview-questions-on-delete)

---

## 1. Quick Answers to Your Questions

| Your question | Answer |
|---|---|
| **Is the record deleted permanently or end-dated?** | **Permanently purged.** `DELETE` removes the component from Oracle HCM. It does **not** end-date it, and **deleted records can't be recovered**. To keep history and just stop a record, use an end-date technique instead (Section 2). |
| **Should EffectiveStartDate and EffectiveEndDate be the same or different?** | Neither. On a DELETE line the dates **identify** the record; they don't define a range to delete. With **user keys**, you **must** supply **both** dates (the record's dates). With **source key / GUID / surrogate ID**, the dates are **optional**. |
| **Does DELETE remove one date-effective row?** | **No.** You **can't delete individual date-effective records**. DELETE removes the **whole logical record** (all its date splits). |
| **Do other columns change the behaviour?** | Yes, for some objects. E.g. deleting a **WorkRelationship** requires **`CancelWorkRelationshipFlag = Y`**; for primary email, phone or address you must delete the non-primary ones first. Details in Section 5. |

---

## 2. What DELETE Really Does: Purge vs End-Date

The **DELETE** instruction identifies a business-object component to **purge** from Oracle HCM. The component **doesn't have to have been loaded using HDL**.

> **Oracle's warning:** you can't recover a deleted object, so **try to correct your data instead of deleting and recreating it.**

### The "removal" toolbox: pick the right one
| Technique | What happens to the data | History kept? | Typical use |
|---|---|---|---|
| **`DELETE`** | Record (all date splits + children) **purged** | ❌ No | Wrong record created by mistake; data never valid |
| **Terminate work relationship** (`ActualTerminationDate` + action + `TerminateWorkRelationshipFlag=Y`) | Relationship and assignments **end** | ✅ Yes | Employee leaves |
| **Cancel work relationship** (DELETE `WorkRelationship` + `CancelWorkRelationshipFlag=Y`) | Relationship **removed** as if it never existed | ❌ No | Hire entered in error / no-show |
| **End an assignment** (assignment termination record, e.g. `TerminateAssignmentFlag=Y`) | Additional assignment **ends**, others continue | ✅ Yes | Stop one of several assignments |
| **`ReplaceLastEffectiveEndDate = Y`** + new `EffectiveEndDate` | Object's **last end date changed** (end-dated) | ✅ Yes | End-date a recurring element entry, close an object on a date (if supported) |
| **Status change** (e.g. `ActiveStatus = I`) | Object stays but becomes **inactive** | ✅ Yes | Retire a job, location, grade |
| **Replace mode** (`SET PURGE_FUTURE_CHANGES Y`) MERGE | **Future date splits** after your start date are **replaced/removed** | Partly | Remove wrong future-dated changes |
| **Roll Back** (Import and Load Data) | Reverses a **specific HDL load** (if the object supports rollback) | n/a | Undo a bad data set |
| **`#NULL`** on an attribute | Just that **field** is cleared | ✅ Yes | Remove a value, not a record |

**Memory hook:** *DELETE = shredder. End-date = archive.*

---

## 3. DELETE Syntax, Keys & Dates

### Format
```
METADATA|<Discriminator>|<key attributes>|[EffectiveStartDate]|[EffectiveEndDate]
DELETE|<Discriminator>|<key values>|[start]|[end]
```
- You only need the attributes that **identify** the record. Other attributes are ignored.
- The METADATA line for DELETE lines can be a **separate** METADATA line from the one used for MERGE lines in other files.

### Dates on a DELETE line: the key rule
| Key type used | EffectiveStartDate / EffectiveEndDate |
|---|---|
| **User key** | **Mandatory, both.** The dates help pinpoint the logical record. |
| **Source key, GUID, surrogate ID** | **Optional** (the key alone is unique) |

**Example: user key (dates required)**
```
METADATA|JobFamily|EffectiveStartDate|EffectiveEndDate|JobFamilyName
DELETE|JobFamily|2012/10/01|4712/12/31|Sales01
```

**Example: source key (dates optional)**
```
METADATA|PersonEmail|SourceSystemOwner|SourceSystemId
DELETE|PersonEmail|LEGACY|EMP1001_EMAIL_H1
```

### "Same or different dates?" in detail
- **Supply the record's actual dates** (first effective start date and last effective end date of the logical record, as they exist in Oracle) when using user keys. They are **not** "delete from X to Y".
- Even if you put a single split's dates, HDL still deletes the **whole logical record** (field note: users who tried to delete one Position split with user key + dates found **all rows** deleted).
- **Some objects identify the record by their dates.** Salary, for example, uses `DateFrom` as part of its user key (`AssignmentNumber` + `DateFrom`), so the date selects *which salary record* to delete (Section 9).

### Which key should you use for deletes?
**Source key** (or surrogate ID from a BIP extract) is safest: exact, no date ambiguity. User keys work but need correct dates, and some user-key values can change.

---

## 4. What You Can and Can't Delete

### Oracle's general rules
| ✅ You CAN delete | ❌ You CAN'T delete |
|---|---|
| Most **complete business objects**, except Worker | **Worker** objects (the whole person) |
| Most individual **business-object components** | **Individual date-effective records** (single splits) |
| **Some Worker child components**, e.g. **Person Email** | **Individual translation objects** |
| Records **not** loaded by HDL (UI-created, etc.) | Anything whose component isn't delete-enabled |

**Not all components support DELETE.** Check **View Business Objects → object → component details** for deletion support before you build the file.

### Examples that often surprise people
| Component | Delete via HDL? | Notes |
|---|---|---|
| **Worker** | ❌ | Never via HDL |
| **Assignment** | ❌ *(field note: error HRC-1035622 "…Assignment, which can't be deleted")* | End the assignment, or cancel the whole work relationship |
| **Work Relationship** | ✅ with `CancelWorkRelationshipFlag = Y` | This is a **cancel**, not a terminate |
| **Person Email / Phone / Address** | ✅ | Primary rules apply (Section 5) |
| **Person Legislative Data** | ✅ but not the **only** record | One must remain |
| **Salary** | ✅ | Only the **latest**, or all of them |
| **Salary Basis** | ✅ only if **not assigned** | |
| **Element Entry** | ✅ | Can't delete and recreate for the same element/assignment/date in one file |
| **Job, Grade, Location, Job Family…** | ✅ usually | Deleting a parent deletes its children. Likely to fail if workers or positions still reference it (then inactivate instead). |

---

## 5. Worker Components: Special Delete Rules

You can delete **some components** of Worker, but **not the entire worker**.

| Component | Deletable? | Rule / edge case |
|---|---|---|
| **Person Email** | ✅ | If it's the **only** email, you can delete it. If **several** exist, delete all **non-primary** emails first **or** make another email primary, then delete the old primary. |
| **Person Phone** | ✅ | Same primary rule as email |
| **Person Address** | ✅ | Subject to **country-specific regulations**. With multiple **mailing** addresses, delete non-primary ones first or set a new primary before deleting the current primary. |
| **Person Legislative Data** | ✅ (partly) | **One record must exist.** You can't delete the only one; you can delete extra ones. |
| **Work Relationship** | ✅ | Must include **`CancelWorkRelationshipFlag = Y`** |
| **Assignment Grade Steps** | ✅ | Setting the assignment's **`GradeId` to `#NULL`** deletes its grade steps **automatically**. If the same file also has a DELETE for Assignment Grade Steps, you get an **error**. |
| **User Information** | ❌ | Only available when **creating** workers |
| **Person User Manual Roles** | ❌ | Only on create. To remove roles, use the **User** object (or its **User Role** component for several roles). |

### Cancel vs Terminate (common interview trap)
| | **Terminate** | **Cancel** |
|---|---|---|
| Instruction | `MERGE` WorkRelationship | `DELETE` WorkRelationship |
| Key attribute | `TerminateWorkRelationshipFlag = Y` + `ActualTerminationDate` + action | `CancelWorkRelationshipFlag = Y` |
| Result | Relationship **ends**, history kept, worker becomes ex-employee | Relationship **removed** as if never hired |
| Use when | A real leaver | Hire entered by mistake, no-show |

```
-- Cancel a work relationship (illustrative; confirm attributes in View Business Objects)
METADATA|WorkRelationship|SourceSystemOwner|SourceSystemId|CancelWorkRelationshipFlag
DELETE|WorkRelationship|LEGACY|EMP1001_WR|Y
```
*(Field note: cancel can fail if the person is still someone's manager or has dependent records. Reassign direct reports and clean up dependents first.)*

### Deleting a primary email (two valid patterns)
```
-- Pattern A: delete non-primary first, then primary (two loads or two lines if allowed)
METADATA|PersonEmail|SourceSystemOwner|SourceSystemId
DELETE|PersonEmail|LEGACY|EMP1001_EMAIL_H1     ← non-primary (home)
DELETE|PersonEmail|LEGACY|EMP1001_EMAIL_W1     ← primary (work), now the only one

-- Pattern B: make another email primary first (MERGE), then delete the old one (separate file)
```
> Because line order isn't guaranteed, put dependent steps in **separate data sets** (load 1, then load 2) when the order matters.

---

## 6. Date-Effective Edge Cases

### 6.1 You can't DELETE a single date split
DELETE always removes the **whole logical record**. To fix a wrong split:

| Goal | Technique |
|---|---|
| Wrong **values** in a split | **Correction**: MERGE with the split's **same EffectiveStartDate** (+ its end date) |
| Remove **future** splits after date X | **Replace mode**: `SET PURGE_FUTURE_CHANGES Y` and MERGE effective X with a blank end date. Splits **after** X are removed and replaced by your record. |
| Remove a **middle** split | Usually re-supply the correct history in **Replace mode** for that date range. In the **UI**, a single physical record can be deleted (the previous record's end date extends to close the gap), but in most cases **not the first or only record**. |

**Replace-mode example (removing future splits).** In Oracle's tutorial, a Job had splits on 1-Jan, 1-Mar and 15-Mar. This file **deleted the 1-Mar and 15-Mar splits** because they were after the effective start date 1-Jan:
```
SET PURGE_FUTURE_CHANGES Y
METADATA|Job|JobCode|SetCode|EffectiveStartDate|StandardWorkingHours|FullPartTime
MERGE|Job|SALES|COMMON|2023/01/01|40|FULL_TIME
```
⚠️ Attributes you don't supply roll forward from the earlier split, so supply the full record.
⚠️ *(Field note)* For assignments, removing future splits can fail if child records (e.g. Assignment Supervisor) exist on those dates: "You cannot delete the Assignment record with effective start date … because a corresponding Assignment Supervisor record exists." Fix the children first.

### 6.2 End-dating instead of deleting: `ReplaceLastEffectiveEndDate`
- Set **`ReplaceLastEffectiveEndDate = Y`** and supply the **new end date** in `EffectiveEndDate`.
- **Hidden attribute:** it doesn't appear in templates or the Attributes tab. Add it **manually**. Check support under **Component Details → Date Effectivity** in View Business Objects.
- You must supply an **actual date**. **Don't** use `#RETAIN`, `#ALL` or `#NULL` with it.
- **Some objects must end on `4712/12/31`.** Changing their last end date gives an error.

```
METADATA|ElementEntry|ElementEntryId|EffectiveStartDate|EffectiveEndDate|ReplaceLastEffectiveEndDate
MERGE|ElementEntry|4634|2014/01/01|2014/04/05|Y
```

### 6.3 Moving the first start date: `ReplaceFirstEffectiveStartDate`
- `ReplaceFirstEffectiveStartDate = Y` + the revised `EffectiveStartDate` changes the object's **first** start date (only for supporting objects, e.g. Job).
- **Worker exception:** you can't change the Worker's first start date this way. Use **`NewStartDate`** instead.
- *(Field note)* Some objects only allow moving the date **earlier** with this attribute.

```
METADATA|Job|JobId|EffectiveStartDate|EffectiveEndDate|ReplaceFirstEffectiveStartDate
MERGE|Job|23452|1950/01/01|4712/12/31|Y
```

### 6.4 Multiple changes per day (`_M` objects)
- You can't DELETE one same-day change either. Use corrections, or Replace mode for the range.
- When re-supplying same-day history, follow the sequence rules: `EffectiveSequence` 1..n, `EffectiveLatestChange` = Y on the last row, and **end date = start date** on the `N` rows.

---

## 7. Parent–Child, Cascades & Translations

- **Deleting a parent deletes its children and translations.** E.g. a DELETE on `Grade` removes the grade **and** its child components.
- **To delete only a child**, use the **child's discriminator**. E.g. `GradeRateValue` to delete only a grade rate value.
- **Translations:** you **can't** include DELETE instructions in **translation data files**, and you can't delete **individual translation objects**. They go when the parent is deleted.
- **References from other objects:** deleting an object that's still **used** elsewhere (a job on assignments, a location on departments, a salary basis on salaries) is typically **blocked**. Salary basis is explicitly deletable **only if not assigned**. In such cases, **inactivate** instead.
- **Cascade side effects:** setting an attribute to `#NULL` can **auto-delete** dependent children (Assignment `GradeId` → Assignment Grade Steps). Don't also DELETE those children in the same file.

---

## 8. File-Level Rules (MERGE + DELETE, Ordering)

| Rule | Why |
|---|---|
| **Never put DELETE and MERGE for the same record in the same file** | HDL doesn't know which instruction to process first. Line order isn't guaranteed. |
| **Delete-then-recreate** for the same key/date → use **two separate data sets** (delete first, confirm, then load) | E.g. Oracle: don't delete an element entry and create one for the same element, assignment and date in the same file |
| **Dependent deletes** (non-primary then primary; child then parent) → separate loads when order matters | Order within a file is undefined |
| DELETE lines can live in the **same file** as MERGE lines for **different** records | Allowed, but keep them separate files/zips for clarity and auditability |
| SET instructions (e.g. `PURGE_FUTURE_CHANGES`) **don't** control DELETE | They apply to date-effective MERGE processing |
| Failed deletes show as **Failed Objects** like any load error | Monitor on Import and Load Data |

---

## 9. Payroll & Compensation Objects

### Element entries
- Deletable via `ElementEntry.dat`. **Be careful: deleted records can't be recovered.**
- **Don't** delete an entry and create one for the **same element, assignment and date** in the same file.
- Source-key delete example (Oracle):
```
METADATA|ElementEntry|SourceSystemOwner|SourceSystemId|EffectiveStartDate
DELETE|ElementEntry|VISION|E3141464_HSEALW_190401|2019/04/01
```
- **End-date instead of delete** (keeps payroll history, usually what payroll wants):
```
METADATA|ElementEntry|SourceSystemOwner|SourceSystemId|EffectiveStartDate|EffectiveEndDate|ReplaceLastEffectiveEndDate
MERGE|ElementEntry|VISION|12351_COM_ALW|2019/06/01|2019/06/30|Y
```
  ⚠️ Oracle: when end-dating this way, **you must supply a record for the element entry *and every element entry value*.**
- Entries **with costing** (`ElementEntryWithCosting`) follow the same rules: don't delete and recreate in the same file.
- *(Field note)* Deleting entries that were already **processed by payroll** may be blocked or have payroll consequences (retro, balances). Coordinate with payroll; end-dating is usually preferred.

### Salary
- You can delete **only the most recent** salary, **or all** of them. E.g. with salaries starting 1-Jan, 1-Mar and 1-Sep, you can delete just the 1-Sep one or all three, but **not** 1-Mar alone.
- Salary is identified by its **`DateFrom`** (with the assignment):
```
METADATA|Salary|AssignmentNumber|DateFrom
DELETE|Salary|1014|2016/01/01
```
- Deleting salary may also affect the **salary element entries** it created.

### Salary basis
- Deletable with HDL **only if it isn't assigned** to anyone.

---

## 10. Decision Matrix: "I Want To…" → Use This

| I want to… | Use |
|---|---|
| Remove a worker created by mistake (no-show) | **Cancel work relationship**: DELETE WorkRelationship + `CancelWorkRelationshipFlag=Y` (the Worker itself can't be deleted) |
| Record that an employee left | **Terminate**: MERGE WorkRelationship (`ActualTerminationDate`, action, `TerminateWorkRelationshipFlag=Y`) |
| Stop one of two assignments | **End the assignment** (assignment termination record), not DELETE |
| Remove a wrong email/phone/address | **DELETE** that component (respect the primary rules) |
| Clear one field | **`#NULL`** |
| Fix wrong values in a past split | **Correction** MERGE (same start date + its end date) |
| Remove wrong future-dated changes | **Replace mode** MERGE (`SET PURGE_FUTURE_CHANGES Y`) |
| Stop a recurring element on a date | **`ReplaceLastEffectiveEndDate=Y`** (+ every entry value) |
| Remove an element entry that should never have existed | **DELETE** ElementEntry (separate file from any re-create) |
| Retire a job/location/grade still referenced historically | **Inactivate** (status) instead of DELETE |
| Change an object's first start date | **`ReplaceFirstEffectiveStartDate=Y`** (Worker: **`NewStartDate`**) |
| Undo a whole bad HDL load | **Roll Back** the business object (if supported), else corrective files |
| Remove a salary change entered wrongly | **DELETE** Salary (latest only, or all) |

---

## 11. Safety Checklist Before Any Delete

1. **Is DELETE really needed?** Oracle says correct rather than delete-and-recreate. Prefer end-date or inactivate if history matters.
2. **Is the component delete-enabled?** Check **View Business Objects**.
3. **Extract a backup first.** BIP/HCM Extract of the records (and their source keys from `HRC_INTEGRATION_KEY_MAP`), because **there's no undo**.
4. **Use precise keys.** Prefer source keys or surrogate IDs. With user keys, supply **both** effective dates.
5. **Check dependencies.** Primary flags, manager relationships, payroll processing, references from other objects.
6. **Separate files** for delete and recreate, and for ordered steps.
7. **Test in a lower environment** with a few records, then verify in the UI and with SQL.
8. **Small batches + low load-error threshold** in production.
9. **Reconcile after**: counts before vs after; `Failed Objects` = 0.
10. *(Optional)* For PII removal, consider `SET PURGE_AUDIT_DATA Y` (can't be combined with `ENABLE_AUDIT_DATA Y` in the same file).

---

## 12. Common Delete Errors

| Error (paraphrased) | Cause | Fix |
|---|---|---|
| "You supplied a DELETE instruction for the object component X, which can't be deleted" | Component isn't delete-enabled (e.g. Assignment) | Use end-date, cancel, or correction instead |
| Record not found / key invalid | Wrong key, or user key without the correct start/end dates | Use source key/surrogate ID, or supply the record's actual dates |
| Can't delete primary email/phone/address | Other non-primary records exist | Delete non-primary first, or set a new primary |
| Can't delete the only Person Legislative Data | One must exist | Correct it instead |
| Error deleting Assignment Grade Steps | Same file sets `GradeId=#NULL` (auto-delete) **and** deletes the steps | Remove the explicit DELETE |
| Cancel work relationship fails | Person still has direct reports or dependent data *(field note)* | Reassign reports and remove dependents, then retry |
| Unpredictable result | MERGE and DELETE for the same record in one file | Split into two sequential loads |
| DELETE in a translation file rejected | Not allowed | Delete the parent object instead |
| Delete of salary basis fails | Basis is assigned | Unassign first, or keep it |
| Purging future assignment splits fails | Child (supervisor) records exist on those dates *(field note)* | Adjust/delete the child records first |

---

## 13. Interview Questions on DELETE

**Q1. Does HDL DELETE end-date a record?**
<details><summary>Answer</summary>No. It **purges** the component permanently. It can't be recovered. To keep history, end-date it (terminate, `ReplaceLastEffectiveEndDate`, inactivate).</details>

**Q2. Can you delete one date-effective split with DELETE?**
<details><summary>Answer</summary>No. You can't delete individual date-effective records. DELETE removes the whole logical record. Use a correction, or Replace mode (`SET PURGE_FUTURE_CHANGES Y`) to remove future splits.</details>

**Q3. When are effective dates mandatory on a DELETE line?**
<details><summary>Answer</summary>When the date-effective object is identified by **user keys**: supply **both** start and end dates. With source keys, GUIDs or surrogate IDs they're optional.</details>

**Q4. Can you delete a worker with HDL? What do you do instead for a hire entered in error?**
<details><summary>Answer</summary>No, Worker objects can't be deleted. **Cancel the work relationship**: DELETE WorkRelationship with `CancelWorkRelationshipFlag = Y`.</details>

**Q5. Cancel vs terminate?**
<details><summary>Answer</summary>Terminate = MERGE with termination date/action/`TerminateWorkRelationshipFlag=Y`; history is kept and the person is a leaver. Cancel = DELETE with `CancelWorkRelationshipFlag=Y`; the relationship is removed as if it never existed.</details>

**Q6. A worker has a work and a home email; the work one is primary. You DELETE the work email and it fails. Why?**
<details><summary>Answer</summary>With multiple emails, you must delete all non-primary emails first, or make another email primary, before deleting the current primary.</details>

**Q7. What happens to children when you delete a Grade?**
<details><summary>Answer</summary>Child components and translations are deleted too. To delete only a child, use the child's discriminator (e.g. `GradeRateValue`).</details>

**Q8. Why can't you put DELETE and MERGE for the same record in one file?**
<details><summary>Answer</summary>HDL doesn't guarantee line processing order, so it doesn't know which instruction to apply first. Use two sequential loads.</details>

**Q9. How do you end-date a recurring element entry without deleting it?**
<details><summary>Answer</summary>MERGE `ElementEntry` with `ReplaceLastEffectiveEndDate = Y` and the new `EffectiveEndDate`, supplying a record for the entry **and every element entry value**.</details>

**Q10. Salaries exist from 1-Jan, 1-Mar, 1-Sep. Can you delete only 1-Mar?**
<details><summary>Answer</summary>No. You can delete only the most recent (1-Sep) or all three.</details>

**Q11. You set `GradeId = #NULL` on an assignment and also DELETE its Assignment Grade Steps in the same file. Result?**
<details><summary>Answer</summary>Error. Nulling the GradeId auto-deletes the grade steps, so the explicit DELETE conflicts.</details>

**Q12. Can `ReplaceLastEffectiveEndDate` be used with `#RETAIN`?**
<details><summary>Answer</summary>No. You must supply an actual date. `#RETAIN`, `#ALL` and `#NULL` aren't allowed with it. Also, some objects must keep `4712/12/31` as the last end date.</details>

**Q13. How do you change a worker's hire (first start) date with HDL?**
<details><summary>Answer</summary>Not with `ReplaceFirstEffectiveStartDate` (not allowed for Worker). Use the **`NewStartDate`** attribute.</details>

**Q14. Must a record have been loaded by HDL to delete it with HDL?**
<details><summary>Answer</summary>No. The component doesn't have to have been loaded using HDL.</details>

**Q15. You need to remove a wrong future-dated promotion from an assignment. DELETE?**
<details><summary>Answer</summary>No, Assignment can't be deleted, and individual splits can't be deleted. Re-supply the assignment from the last correct date with `SET PURGE_FUTURE_CHANGES Y` (Replace mode) so the future split is replaced, supplying full values. Watch for child records (supervisor) on those dates.</details>

---