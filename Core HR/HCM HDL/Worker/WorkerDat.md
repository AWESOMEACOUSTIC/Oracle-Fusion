# Worker.dat Explained: Your Sheet, Filled In, With Edge Cases

> Companion to **Worker_HDL_Filled.xlsx** and the ready-to-zip **Worker.dat**.
> Two sample new hires (**Ravi Sharma**, **Priya Iyer**) are used throughout so you can see how every component connects.
> Based on Oracle's tutorial *Create New Hires with HDL*, Oracle by Example *Creating and Loading Files with HDL*, and the *HCM Business Objects* guide. Always confirm attribute names in **View Business Objects → Worker** for your release.

---

## Table of Contents
1. [What Was Wrong in Your Sheet](#1-what-was-wrong-in-your-sheet)
2. [The Chain at a Glance](#2-the-chain-at-a-glance)
3. [Component-by-Component Walkthrough](#3-component-by-component-walkthrough)
4. [The Date-Matching Rule](#4-the-date-matching-rule)
5. [Before You Load](#5-before-you-load)
6. [Edge Cases per Component](#6-edge-cases-per-component)
7. [Next Practice Scenarios (with sample lines)](#7-next-practice-scenarios-with-sample-lines)
8. [Error → Cause → Fix Table](#8-error--cause--fix-table)

---

## 1. What Was Wrong in Your Sheet

| # | Component | Problem | Severity | Fixed to |
|---|---|---|---|---|
| 1 | **WorkRelationship** | **No `PersonId(SourceSystemId)`**: the relationship had no link to its person | 🔴 Critical | Added the parent reference |
| 2 | **WorkTerms** | **Two METADATA lines** for WorkTerms. Only **one METADATA line per component** is allowed in a file. | 🔴 Critical | Merged into one |
| 3 | **WorkTerms** (1st line) | Parent given as `PersonId`. WorkTerms' parent is the **WorkRelationship** (`PeriodOfServiceId`), not the person. | 🔴 Critical | Removed |
| 4 | **Assignment** | No **EffectiveStartDate**, ActionCode, AssignmentNumber, status, person type, BU or primary flag | 🔴 Critical | All added |
| 5 | **Worker** | No **StartDate** (hire date), **ActionCode** (`HIRE`) or PersonNumber | 🟠 High | Added |
| 6 | **WorkRelationship** | Used `EffectiveStartDate` + `StartDate`. The relationship's start attribute is **`DateStart`**. | 🟠 High | Replaced with `DateStart` |
| 7 | **PersonName** | Hint written `SourceSystemID` (capital D); **LegislationCode** missing | 🟠 High | `PersonId(SourceSystemId)` + `LegislationCode` |
| 8 | **WorkTerms** | `BusinessUnitId` given a **name**. Attributes ending in `Id` expect the **numeric surrogate ID**. | 🟡 Medium | `BusinessUnitShortCode` |
| 9 | **PersonLegislativeData** | Missing (recommended). If omitted, Oracle **auto-generates** one with an auto source key. | 🟡 Medium | Added |
| 10 | **WorkTerms** | No AssignmentNumber / AssignmentName / PrimaryWorkTermsFlag | 🟡 Medium | Added |

> **Rule from #8, worth remembering:** an attribute ending in **`Id`** (e.g. `BusinessUnitId`, `JobId`, `PersonId`) expects the **surrogate ID**, unless you add a hint like `(SourceSystemId)`. To use a **name or code**, use the user-key alternative attribute (`BusinessUnitShortCode`, `JobCode`, `DepartmentName`…).

---

## 2. The Chain at a Glance

```
Worker                     AKP_LAB | AKP_E1001            StartDate 2026/01/05, ActionCode HIRE
 ├─ PersonName             AKP_E1001_NAME   ─ PersonId(SourceSystemId) = AKP_E1001
 ├─ PersonLegislativeData  AKP_E1001_LEG    ─ PersonId(SourceSystemId) = AKP_E1001
 ├─ PersonEmail            AKP_E1001_EML_W1 ─ PersonId(SourceSystemId) = AKP_E1001
 └─ WorkRelationship       AKP_E1001_WR     ─ PersonId(SourceSystemId) = AKP_E1001
      └─ WorkTerms         AKP_E1001_WT     ─ PeriodOfServiceId(SourceSystemId) = AKP_E1001_WR
           └─ Assignment   AKP_E1001_ASG    ─ WorkTermsAssignmentId(SourceSystemId) = AKP_E1001_WT
                 ├─ JobId(SourceSystemId)      = AKP_JOB_DA   (foreign: Job from Lab 4)
                 ├─ LocationId(SourceSystemId) = AKP_LOC_CHN  (foreign: Location from Lab 3)
                 └─ DepartmentName             = <Your Department>  (foreign, by user key)
```
- **Every line has its own key** (AKP_LAB + a unique SourceSystemId).
- **Every child carries its parent's key** in the parent-reference attribute.
- The **owner** (`AKP_LAB`) is inherited by all `(SourceSystemId)` references because they all use the same owner.

---

## 3. Component-by-Component Walkthrough

### 3.1 Worker: the person (top level)
```
METADATA|Worker|SourceSystemOwner|SourceSystemId|EffectiveStartDate|EffectiveEndDate|PersonNumber|StartDate|DateOfBirth|ActionCode
MERGE|Worker|AKP_LAB|AKP_E1001|2026/01/05||AKP1001|2026/01/05|1998/04/12|HIRE
```
| Attribute | Value | Why |
|---|---|---|
| SourceSystemOwner + SourceSystemId | AKP_LAB, AKP_E1001 | The person's identity. Every child points to `AKP_E1001`. |
| EffectiveStartDate | 2026/01/05 | Must **equal StartDate** for a new hire |
| EffectiveEndDate | *(blank)* | Blank = end of time |
| PersonNumber | AKP1001 | The **user key**. Leave blank if your environment **auto-generates** person numbers; then the source key is **mandatory**. |
| StartDate | 2026/01/05 | Hire date |
| DateOfBirth | 1998/04/12 | Optional but normally supplied |
| ActionCode | HIRE | **HIRE is needed for new hires** |

### 3.2 PersonName
```
METADATA|PersonName|SourceSystemOwner|SourceSystemId|PersonId(SourceSystemId)|EffectiveStartDate|EffectiveEndDate|LegislationCode|NameType|FirstName|LastName
MERGE|PersonName|AKP_LAB|AKP_E1001_NAME|AKP_E1001|2026/01/05||IN|GLOBAL|Ravi|Sharma
```
| Attribute | Value | Why |
|---|---|---|
| PersonId(SourceSystemId) | AKP_E1001 | **Parent link** to the Worker |
| EffectiveStartDate | 2026/01/05 | Must **match the Worker's** EffectiveStartDate for new hires |
| LegislationCode | IN | Decides the name format/rules for the country |
| NameType | GLOBAL | The global name every person has |

### 3.3 PersonLegislativeData (recommended)
```
METADATA|PersonLegislativeData|SourceSystemOwner|SourceSystemId|PersonId(SourceSystemId)|EffectiveStartDate|EffectiveEndDate|LegislationCode|Sex|MaritalStatus
MERGE|PersonLegislativeData|AKP_LAB|AKP_E1001_LEG|AKP_E1001|2026/01/05||IN|M|S
```
- **LegislationCode decides which Sex and MaritalStatus values are valid.**
- Recommended. If you don't supply it, Oracle **auto-generates** a legislative record with an **auto-generated source key**, which is harder to update later.

### 3.4 PersonEmail (optional, added for practice)
```
METADATA|PersonEmail|SourceSystemOwner|SourceSystemId|PersonId(SourceSystemId)|DateFrom|EmailType|EmailAddress|PrimaryFlag
MERGE|PersonEmail|AKP_LAB|AKP_E1001_EML_W1|AKP_E1001|2026/01/05|W1|ravi.sharma@example.com|Y
```
- **Not date-effective**: it uses `DateFrom`/`DateTo`, not EffectiveStartDate.
- `EmailType` is a **lookup code** (`W1` = work, `H1` = home in standard setups; check your lookup).

### 3.5 WorkRelationship
```
METADATA|WorkRelationship|SourceSystemOwner|SourceSystemId|PersonId(SourceSystemId)|LegalEmployerName|DateStart|WorkerType|PrimaryFlag|ActionCode
MERGE|WorkRelationship|AKP_LAB|AKP_E1001_WR|AKP_E1001|<Your Legal Employer>|2026/01/05|E|Y|HIRE
```
| Attribute | Value | Why |
|---|---|---|
| PersonId(SourceSystemId) | AKP_E1001 | **Parent link** (this was missing in your sheet) |
| LegalEmployerName | exact name from your instance | Who employs the person |
| DateStart | 2026/01/05 | Must **match the Worker's StartDate** |
| WorkerType | E | E = Employee, C = Contingent, N = Nonworker, P = Pending |
| PrimaryFlag | Y | Primary relationship |
| ActionCode | HIRE | Hire action |

### 3.6 WorkTerms (the "skeleton" link)
```
METADATA|WorkTerms|SourceSystemOwner|SourceSystemId|PeriodOfServiceId(SourceSystemId)|ActionCode|EffectiveStartDate|EffectiveEndDate|EffectiveSequence|EffectiveLatestChange|AssignmentName|AssignmentNumber|PrimaryWorkTermsFlag|BusinessUnitShortCode
MERGE|WorkTerms|AKP_LAB|AKP_E1001_WT|AKP_E1001_WR|HIRE|2026/01/05||1|Y|ET-AKP1001|ET-AKP1001|Y|<Your BU>
```
- **Parent = WorkRelationship**, referenced through **`PeriodOfServiceId`** (the work relationship's ID).
- Employment terms are **no longer visible in the UI**, but you still supply a **skeleton WorkTerms** record to link the relationship to the assignment.
- `_M` data: needs **EffectiveSequence = 1** and **EffectiveLatestChange = Y** (only one change on that date).
- EffectiveStartDate **must match the WorkRelationship** start.

### 3.7 Assignment
```
METADATA|Assignment|SourceSystemOwner|SourceSystemId|WorkTermsAssignmentId(SourceSystemId)|ActionCode|EffectiveStartDate|EffectiveEndDate|EffectiveSequence|EffectiveLatestChange|AssignmentName|AssignmentNumber|AssignmentStatusTypeCode|PersonTypeCode|BusinessUnitShortCode|PrimaryAssignmentFlag|JobId(SourceSystemId)|LocationId(SourceSystemId)|DepartmentName
MERGE|Assignment|AKP_LAB|AKP_E1001_ASG|AKP_E1001_WT|HIRE|2026/01/05||1|Y|E-AKP1001|E-AKP1001|ACTIVE_PROCESS|Employee|<Your BU>|Y|AKP_JOB_DA|AKP_LOC_CHN|<Your Department>
```
| Attribute | Value | Why |
|---|---|---|
| WorkTermsAssignmentId(SourceSystemId) | AKP_E1001_WT | **Parent link** to WorkTerms |
| EffectiveStartDate | 2026/01/05 | Must **match the WorkTerms** start |
| AssignmentStatusTypeCode | ACTIVE_PROCESS | Active, payroll-eligible (as in Oracle's example) |
| PersonTypeCode | Employee | User person type (as in Oracle's example) |
| BusinessUnitShortCode | your BU | By name/short code, not by `Id` |
| PrimaryAssignmentFlag | Y | Primary assignment |
| JobId(SourceSystemId) | AKP_JOB_DA | Foreign reference to the job you loaded in Lab 4 (same owner, so no owner hint needed). Oracle's example also uses `JobId(SourceSystemId)` + `JobId(SourceSystemOwner)` when the owner differs. |
| LocationId(SourceSystemId) | AKP_LOC_CHN | Foreign reference to the Lab 3 location |
| DepartmentName | your department | Foreign reference by user key |

> 💡 For FTE: either add an `AssignmentWorkMeasure` component, or put **`SET CALCULATE_FTE Y`** at the top of the file to calculate it automatically. If you use the SET, don't also send AssignmentWorkMeasure.

---

## 4. The Date-Matching Rule

For a **new hire**, these dates must all be the **same day**:

```
Worker.EffectiveStartDate  =  Worker.StartDate
          =  PersonName.EffectiveStartDate
          =  PersonLegislativeData.EffectiveStartDate
          =  WorkRelationship.DateStart
          =  WorkTerms.EffectiveStartDate
          =  Assignment.EffectiveStartDate
```
Oracle's tutorial states each link: Worker EffectiveStartDate must match StartDate; PersonName's EffectiveStartDate must match the Worker's; WorkRelationship DateStart must match the Worker's StartDate; WorkTerms must match the WorkRelationship; Assignment must match WorkTerms.

**Also:** referenced objects (job, location, department, BU, legal employer) must be **effective on or before** that hire date.

---

## 5. Before You Load

1. Add **`AKP_LAB`** to lookup **`HRC_SOURCE_SYSTEM_OWNER`**.
2. Load **Lab 3 (Location)** and **Lab 4 (Job)** first, or change the job/location references to existing ones.
3. Replace `<Your Legal Employer>`, `<Your BU>`, `<Your Department>` with **exact** names from your instance.
4. If person numbers are **auto-generated**, blank out `PersonNumber` on the Worker lines (keep the source keys).
5. Check whether **assignment numbers** are auto-generated in your setup. If so, follow your template's guidance for `AssignmentNumber`.
6. Save as **UTF-8**, name it **`Worker.dat`** (case-sensitive), zip it with the file at the **root**, then **Import and Load Data**.
7. Check **Failed Lines = 0** and **Failed Objects = 0**, then open each worker in the UI.

---

## 6. Edge Cases per Component

### Worker
| Edge case | What happens / what to do |
|---|---|
| Auto-generated person numbers | You **can't** supply PersonNumber, so the **source key is mandatory** |
| EffectiveStartDate ≠ StartDate on a hire | Load error. Keep them equal. |
| Same SourceSystemId reused for a different person | HDL treats it as the **same** person (an update, or conflicting history) |
| Re-running the same file | With source keys it's a **safe update** (no duplicate person) |
| Duplicate person check | Oracle can flag likely duplicates (same name/DOB/national ID); dedupe your source data |
| Updating Worker components with only PersonNumber (user key) | Mostly **not possible**: the attribute being changed is part of what identifies the record |
| Deleting a Worker | **Not possible** in HDL. Cancel the work relationship (hire in error) or terminate. |

### PersonName / PersonLegislativeData
| Edge case | What happens / what to do |
|---|---|
| Name history (name changes over time) | **Source keys required** for date-effective name history. Otherwise the user key is PersonNumber + EffectiveStartDate + NameType. |
| Several names/emails for one person in one load | Process them together and **include the parent Worker line**. If the worker exists, the Worker line needs only its **key attributes**. |
| Sex / MaritalStatus invalid | The codes are **legislation-specific**. Check them against the LegislationCode. |
| Omitting PersonLegislativeData | An auto record with an auto key is created, so later updates are harder |
| Deleting legislative data | You can't delete the **only** record |

### PersonEmail
| Edge case | What happens / what to do |
|---|---|
| Deleting the primary email when others exist | Delete the non-primary ones first, **or** make another email primary |
| Two emails of the same type | Usually rejected or overwritten, depending on key and type rules. Use distinct keys and types. |
| EffectiveStartDate on an email | Wrong attribute. Emails use `DateFrom`. |

### WorkRelationship
| Edge case | What happens / what to do |
|---|---|
| Rehire | **New** WorkRelationship (new key, e.g. `AKP_E1001_WR2`) + WorkTerms + Assignment, ActionCode `REHIRE`. Same person. |
| Terminate | Load **only WorkRelationship** with ActualTerminationDate + termination action + `TerminateWorkRelationshipFlag = Y`. Assignments end automatically. **Don't** include WorkTerms/Assignment lines. |
| Hire entered in error | **Cancel**: DELETE WorkRelationship with `CancelWorkRelationshipFlag = Y` |
| Global transfer | **New** relationship in the new legal employer, with all its children and new keys/assignment number |
| LegalEmployerName typo | Load error: invalid value |

### WorkTerms & Assignment
| Edge case | What happens / what to do |
|---|---|
| Assignment change (transfer, promotion) | Send **WorkTerms and Assignment together**, same file, **same EffectiveStartDate/EndDate** |
| Two changes on the same day | EffectiveSequence 1, 2…; the last one has `EffectiveLatestChange = Y`; on the `N` rows, **EffectiveEndDate = EffectiveStartDate** |
| Update with future-dated changes present | Use `SET PURGE_FUTURE_CHANGES N` (Retain) and `#RETAIN` / `#ALL` as needed |
| ActionCode on an existing split | **Protected from update**: you can't change it by MERGE |
| `BusinessUnitId` with a name | Error: `Id` attributes expect surrogate IDs → use `BusinessUnitShortCode` |
| Job/Location/Department not effective on the hire date | "Invalid value" load error → back-date the reference object |
| Job loaded under another owner | Add `JobId(SourceSystemOwner)` |
| Manager is also a new hire in the same file | Supervisor reference may fail → load `AssignmentSupervisor` in a second pass |
| Deleting an assignment | Not supported → end the assignment or cancel the relationship |

---

## 7. Next Practice Scenarios (with sample lines)

> Use **one scenario per zip** and verify after each. These are illustrative: check attributes in your template.

**A. Add a home email to Ravi (child only, parent already in Oracle)**
```
METADATA|PersonEmail|SourceSystemOwner|SourceSystemId|PersonId(SourceSystemId)|DateFrom|EmailType|EmailAddress|PrimaryFlag
MERGE|PersonEmail|AKP_LAB|AKP_E1001_EML_H1|AKP_E1001|2026/03/01|H1|ravi.home@example.com|N
```

**B. Transfer Ravi to another department on 2026/06/01**
```
SET PURGE_FUTURE_CHANGES N
METADATA|WorkTerms|SourceSystemOwner|SourceSystemId|PeriodOfServiceId(SourceSystemId)|ActionCode|EffectiveStartDate|EffectiveEndDate|EffectiveSequence|EffectiveLatestChange
MERGE|WorkTerms|AKP_LAB|AKP_E1001_WT|AKP_E1001_WR|ASG_CHANGE|2026/06/01||1|Y
METADATA|Assignment|SourceSystemOwner|SourceSystemId|WorkTermsAssignmentId(SourceSystemId)|ActionCode|EffectiveStartDate|EffectiveEndDate|EffectiveSequence|EffectiveLatestChange|DepartmentName
MERGE|Assignment|AKP_LAB|AKP_E1001_ASG|AKP_E1001_WT|ASG_CHANGE|2026/06/01||1|Y|<New Department>
```
(Use an action code that exists in your instance, e.g. `ASG_CHANGE` or `TRANSFER`.)

**C. Correct Priya's last name (PersonName correction on the same start date)**
```
METADATA|PersonName|SourceSystemOwner|SourceSystemId|PersonId(SourceSystemId)|EffectiveStartDate|LegislationCode|NameType|LastName
MERGE|PersonName|AKP_LAB|AKP_E1002_NAME|AKP_E1002|2026/02/02|IN|GLOBAL|Iyengar
```

**D. Terminate Priya on 2026/12/31**
```
METADATA|WorkRelationship|SourceSystemOwner|SourceSystemId|PersonId(SourceSystemId)|LegalEmployerName|DateStart|WorkerType|ActualTerminationDate|ActionCode|TerminateWorkRelationshipFlag
MERGE|WorkRelationship|AKP_LAB|AKP_E1002_WR|AKP_E1002|<Your Legal Employer>|2026/02/02|E|2026/12/31|<Your termination action>|Y
```

**E. Break it on purpose (watch the error)**
- Remove `PersonId(SourceSystemId)` from WorkRelationship → parent not found.
- Change the Assignment EffectiveStartDate to 2026/01/06 → date mismatch with WorkTerms.
- Put two WorkTerms METADATA lines back → import error.
- Use `BusinessUnitId` with a name → invalid value.

---

## 8. Error → Cause → Fix Table

| Symptom (paraphrased) | Cause | Fix |
|---|---|---|
| Parent record can't be found | Missing or wrong parent reference / key typo / hint misspelled | Check `PersonId(SourceSystemId)`, `PeriodOfServiceId(...)`, `WorkTermsAssignmentId(...)` |
| Duplicate METADATA for component | Two METADATA lines for one component | Keep one |
| Invalid SourceSystemOwner | Owner not in `HRC_SOURCE_SYSTEM_OWNER` | Add it |
| Invalid legal employer / BU / department | Name not exact or not effective on the hire date | Copy exact names; check dates |
| Value isn't valid for attribute …Id | Name supplied to an `Id` attribute | Use the code/name attribute instead |
| Start dates don't match | Component dates differ for a new hire | Apply the date-matching rule (Section 4) |
| Invalid Sex/MaritalStatus | Code not valid for the legislation | Use legislation-valid codes |
| EffectiveSequence/LatestChange errors | Missing sequence or the wrong Y/N | 1 / Y for a single change per day |
| Job not found | Job not loaded, different owner, or not effective | Load the job first; add `JobId(SourceSystemOwner)`; back-date |
| PersonNumber not allowed | Auto-numbering is on | Blank PersonNumber; rely on the source key |

---