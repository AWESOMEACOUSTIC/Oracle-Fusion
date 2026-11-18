# Oracle HDL — Hands-On Practical Lab Guide

> Built from your practical-session notes: **Configure HCM Data Loader → Job Family → Action Reasons → Locations → (Job) → Positions → Deletes**, plus a **"break it on purpose"** error lab.
> Each lab gives: the goal → prerequisites → sample `.dat` → steps → how to verify → **errors you're likely to hit and how to fix them**.
> ⚠️ Sample files are **illustrative**. Always download the real template from **View Business Objects** (after **Refresh Object**) and match attribute names, codes and set codes to **your** environment.

---

## Table of Contents
- [0. Corrections & Clarifications to Your Notes](#0-corrections--clarifications-to-your-notes)
- [1. Lab Setup & Golden Rules](#1-lab-setup--golden-rules)
- [Lab 0 — Explore View Business Objects & Configure HCM Data Loader](#lab-0--explore-view-business-objects--configure-hcm-data-loader)
- [Lab 1 — Job Family](#lab-1--job-family)
- [Lab 2 — Action Reasons](#lab-2--action-reasons)
- [Lab 3 — Locations](#lab-3--locations)
- [Lab 4 — Job (bridge to Positions)](#lab-4--job-bridge-to-positions)
- [Lab 5 — Positions](#lab-5--positions)
- [Lab 6 — Deleting & Inactivating](#lab-6--deleting--inactivating)
- [Lab 7 — Break It On Purpose (Error Lab)](#lab-7--break-it-on-purpose-error-lab)
- [Verification SQL (BIP)](#verification-sql-bip)
- [Per-Load Checklist](#per-load-checklist)

---

## 0. Corrections & Clarifications to Your Notes

| Your note | Status | Clarification |
|---|---|---|
| Data migration tools: HDL and Spreadsheet Data Loader | ✅ | HSDL (spreadsheet loader) uses HDL underneath and has its own parameters tab in Configure HCM Data Loader |
| GUID, Source Key, **Surrogate Key**, User Key | ✅ (wording) | Oracle calls it **Surrogate ID** (the numeric internal ID) |
| Source key supports only integration-enabled objects | ✅ | Correct |
| User key = natural key, used when there's no source key | ✅ | Also required on **create** unless another key identifies the record |
| Setup: Workforce Deployment → Configure HCM Data Loader | ✅ | Setup and Maintenance → **HCM Data Loader** functional area → **Configure HCM Data Loader** |
| **"Delete child first, then parent"** | ⚠️ **Half true** | When you **delete a parent, its children are deleted automatically**, so you don't need to delete children first for that. Delete a child separately only when you want to **keep the parent**. "Child first" matters when the parent **can't** be deleted, or when deletion is **blocked by references**. |
| Worker can't be deleted | ✅ | Use **cancel work relationship** for hires entered in error |
| Parent deletion deletes children automatically | ✅ | Translations too |
| Locations: reason for creating them → action reasons | ✅ | **ActionReasonCode** on Location. Action reasons must **exist before** you load locations. |
| **"End date is always 4712/12/31 or null"** | ⚠️ **Mostly** | True for the **latest / open-ended** record. Not true when correcting a middle split, end-dating (`ReplaceLastEffectiveEndDate`), history rows, or `_M` rows with `EffectiveLatestChange=N`. Some objects use **StartDate/EndDate** or **DateFrom/DateTo** instead of EffectiveStart/EndDate (Action Reasons, Email). |
| **Locations / Jobs can be deleted** *(implied)* | ❌ | **You can't delete Location or Job objects with HDL. Make them inactive instead** (Lab 6). |

---

## 1. Lab Setup & Golden Rules

### Prerequisites
| Need | Why |
|---|---|
| Role **Human Capital Management Integration Specialist** (or equivalent) | HDL tasks + UCM upload |
| Access to **Setup and Maintenance** | Lookups, Configure HCM Data Loader |
| A **UTF-8** text editor (VS Code / Notepad++) | Encoding + visible pipes |
| A zip tool | `.dat` at the **zip root** |
| Your **initials** as a prefix in all codes, e.g. `AKP_` | Avoid clashing with your peers' data in a shared training instance |

### Golden rules (stick them on your screen)
1. **File name = object name, case-sensitive**: `JobFamily.dat`, `Location.dat`, `ActionReasons.dat`, `Position.dat`.
2. **One METADATA line per component** per file; values in the **same order** as METADATA.
3. Dates **`YYYY/MM/DD`**; open end = blank or `4712/12/31`.
4. Send **codes**, not meanings (`A` not `Active`).
5. **Source keys** for everything you create in labs. Register your owner in **`HRC_SOURCE_SYSTEM_OWNER`** first (e.g. `AKP_LAB`).
6. **Effective start date of referenced objects ≤ start date of the object that references them** (e.g. location starts before the position that uses it).
7. After every load, check **Failed Lines = 0 AND Failed Objects = 0**, not just Load Status.

### One-time step: register your source system owner
Setup and Maintenance → search task **Manage Common Lookups** → lookup type **`HRC_SOURCE_SYSTEM_OWNER`** → add code **`AKP_LAB`** (meaning "Aditya Lab") → Save.

---

## Lab 0 — Explore View Business Objects & Configure HCM Data Loader

### 0A. View Business Objects
**Path:** My Client Groups → **Data Exchange** → **View Business Objects**
1. Search **Job Family**. Select it → **Refresh Object** → click **Refresh** until *Last Refreshed Date* = today.
2. Open it and look at:
   - **Component Details**: discriminator, parent component, whether integration keys are supported, **deletion support**, date-effectivity (first start / last end changes)
   - **Attributes**: name, mandatory flag, data type, **key type** (user key / parent surrogate ID / foreign reference), **lookup type**
3. Download the **template `.dat`** and the overview spreadsheet.
4. Repeat for **Location**, **Action Reasons**, **Job**, **Position**. Note each one's user keys and mandatory attributes.

> 🎯 **Practice question to answer yourself:** for Position, which attributes are user keys, which are foreign references, and does Position support DELETE?

### 0B. Configure HCM Data Loader
**Path:** Setup and Maintenance → **HCM Data Loader** functional area → **Configure HCM Data Loader**
Look at (don't change in a shared instance without permission):
| Parameter | What to observe |
|---|---|
| File Action default | Import and load vs Import only |
| Maximum Percentage of Import/Load Errors | Thresholds before processing stops |
| Maximum Concurrent Threads (Import/Load) | Parallelism |
| **Date-Effective Update Mode** | Enterprise default for `PURGE_FUTURE_CHANGES` (Replace by default) |
| Enable Audit Data | Default No |
| File Delimiter / Escape | Enterprise defaults (`|`, `\`) |
| HCM Spreadsheet Data Loader tab | HSDL-specific settings |
| Enable Configuration of Role-Based Business Object Access | Restricts which roles can load which objects |

---

## Lab 1 — Job Family

**Goal:** create, update, translate, inactivate and delete job families. This is the simplest object, so it's perfect for learning the cycle.

### 1A. Create with USER keys (Oracle tutorial style)
`JobFamily.dat`
```
METADATA|JobFamily|JobFamilyCode|JobFamilyName|EffectiveStartDate|EffectiveEndDate
MERGE|JobFamily|AKP_CLERICAL|AKP Clerical|2000/01/01|
MERGE|JobFamily|AKP_MANAGERIAL|AKP Managerial|2000/01/01|
```
Steps: save as UTF-8 → zip as `AKP_L1A_JobFamily_UserKey.zip` → Data Exchange → **Import and Load Data** → **Import File** → Submit Now → Refresh until done.

### 1B. Create with SOURCE keys (Oracle example pattern)
```
METADATA|JobFamily|SourceSystemOwner|SourceSystemId|EffectiveStartDate|JobFamilyCode|JobFamilyName|ActiveStatus
MERGE|JobFamily|AKP_LAB|AKP_JF_ANALYST|2000/01/01|AKP_ANALYST|AKP Analyst|A
MERGE|JobFamily|AKP_LAB|AKP_JF_TECH|2000/01/01|AKP_TECHNICIAN|AKP Technician|A
```

### 1C. Update the name (date-effective update)
```
SET PURGE_FUTURE_CHANGES N
METADATA|JobFamily|SourceSystemOwner|SourceSystemId|EffectiveStartDate|JobFamilyName
MERGE|JobFamily|AKP_LAB|AKP_JF_ANALYST|2026/01/01|AKP Data Analyst
```
✔ Verify: the job family now has **two splits** (2000/01/01–2025/12/31 and 2026/01/01–end of time).

### 1D. Correct the first split (correction)
```
METADATA|JobFamily|SourceSystemOwner|SourceSystemId|EffectiveStartDate|EffectiveEndDate|JobFamilyName
MERGE|JobFamily|AKP_LAB|AKP_JF_ANALYST|2000/01/01|2025/12/31|AKP Business Analyst
```
✔ Verify: the first split's name changed and the 2026 split is untouched.

### 1E. Translate (optional)
```
METADATA|JobFamilyTranslation|SourceSystemOwner|SourceSystemId|EffectiveStartDate|Language|JobFamilyName
MERGE|JobFamilyTranslation|AKP_LAB|AKP_JF_TECH|2000/01/01|F|AKP Technicien
```
(Only if French is an installed language; the `Language` code depends on your instance.)

### 1F. Inactivate vs Delete
```
-- Inactivate (keeps history)
METADATA|JobFamily|SourceSystemOwner|SourceSystemId|EffectiveStartDate|ActiveStatus
MERGE|JobFamily|AKP_LAB|AKP_JF_TECH|2026/06/01|I
```
```
-- Delete (permanent) – only a job family NOT used by any job
METADATA|JobFamily|SourceSystemOwner|SourceSystemId
DELETE|JobFamily|AKP_LAB|AKP_JF_TECH
```

### Likely errors: Job Family
| Error | Cause | Fix |
|---|---|---|
| Job family name/code already exists | Peer or earlier load used the same code/name. **Names must be unique in the enterprise.** | Prefix with initials |
| Invalid SourceSystemOwner | `AKP_LAB` not in `HRC_SOURCE_SYSTEM_OWNER` | Add the lookup code |
| User-key delete fails | Using user keys without **both** effective dates | Add start and end dates, or use the source key |
| Delete fails, family in use | A job references it | Inactivate instead, or change the job first |
| "Attribute unknown" | Typo (`JobFamilyNme`) | Copy names from the template |

---

## Lab 2 — Action Reasons

**Goal:** create action reasons to use on Locations/Positions (your notes: "reason of creating locations").
Oracle: if you plan to use action reasons with work structures, **create them before loading the work structures**. Action reasons can be used on grades, grade rates, grade ladders, jobs, **locations**, organizations and **positions**.

### 2A. Create Action Reasons
`ActionReasons.dat`
```
METADATA|ActionReasons|SourceSystemOwner|SourceSystemId|ActionReasonCode|ActionReason|StartDate|EndDate
MERGE|ActionReasons|AKP_LAB|AKP_AR_NEWSITE|AKP_NEW_SITE|AKP New Office Opening|2000/01/01|
MERGE|ActionReasons|AKP_LAB|AKP_AR_RELOC|AKP_RELOCATION|AKP Office Relocation|2000/01/01|
```
> Note: this object uses **StartDate/EndDate**, not EffectiveStartDate. Always check the template.

### 2B. (Optional) Link a reason to an action: Action Reason Usage
`Actions.dat` (Oracle pattern; links an existing action to a reason)
```
METADATA|ActionReasonUsage|SourceSystemOwner|SourceSystemId|ActionCodeId(SourceSystemId)|ActionReasonId(SourceSystemId)|StartDate|EndDate
MERGE|ActionReasonUsage|AKP_LAB|AKP_USG_1|<action source id>|AKP_AR_NEWSITE|2000/01/01|
```
⚠️ The action must be referenceable by source key. For seeded actions use `FUSION` + ACTION_ID with the `(SourceSystemOwner)` hint, or skip this step in the lab.

### Likely errors: Action Reasons
| Error | Cause | Fix |
|---|---|---|
| Code already exists | Duplicate `ActionReasonCode` | Prefix with initials |
| Date errors | Used EffectiveStartDate instead of **StartDate** | Use the template's attribute names |
| Location later says "invalid action reason" | Reason not created yet, or the code is misspelled | Load action reasons **first**; copy the exact code |

Delete (allowed for Action Reasons):
```
METADATA|ActionReasons|SourceSystemOwner|SourceSystemId
DELETE|ActionReasons|AKP_LAB|AKP_AR_RELOC
```

---

## Lab 3 — Locations

**Goal:** create a location with an address and an action reason, update it, inactivate it.

### Prerequisites
- **Set code** exists (most labs use `COMMON`)
- Action reason from Lab 2 exists
- The country's **geography/address validation** setup (Manage Geographies). Your address values must match the configured address style for that country.
- Create locations with a start date **on or before** the objects that will reference them (organizations, positions).

### 3A. Create
`Location.dat`
```
METADATA|Location|SourceSystemOwner|SourceSystemId|SetCode|LocationCode|LocationName|EffectiveStartDate|EffectiveEndDate|ActiveStatus|ActionReasonCode|ShipToSiteFlag|ReceivingSiteFlag|BillToSiteFlag|OfficeSiteFlag|AddressLine1|TownOrCity|Region1|PostalCode|Country
MERGE|Location|AKP_LAB|AKP_LOC_CHN|COMMON|AKP_CHN01|AKP Chennai Office|2000/01/01||A|AKP_NEW_SITE|Y|Y|Y|Y|Siruseri IT Park|Chennai|Tamil Nadu|603103|IN
MERGE|Location|AKP_LAB|AKP_LOC_BLR|COMMON|AKP_BLR01|AKP Bangalore Office|2000/01/01||A|AKP_NEW_SITE|Y|Y|Y|Y|Whitefield Main Road|Bengaluru|Karnataka|560066|IN
```
*(Address attribute names like `Region1/Region2/TownOrCity` and which ones are mandatory depend on the country's address style. Check the template and the Location attribute list.)*

### 3B. Update (relocation with a reason, Retain mode)
```
SET PURGE_FUTURE_CHANGES N
METADATA|Location|SourceSystemOwner|SourceSystemId|EffectiveStartDate|ActionReasonCode|AddressLine1
MERGE|Location|AKP_LAB|AKP_LOC_CHN|2026/04/01|AKP_RELOCATION|DLF IT Park, Ramapuram
```
✔ Verify: the location history shows a 2026/04/01 split with the new address.

### 3C. Inactivate (you can't delete locations with HDL)
```
METADATA|Location|SourceSystemOwner|SourceSystemId|EffectiveStartDate|ActiveStatus
MERGE|Location|AKP_LAB|AKP_LOC_BLR|2026/12/01|I
```

### Likely errors: Locations
| Error | Cause | Fix |
|---|---|---|
| Invalid address / geography value | City/state/postal code don't match the geography setup or address style | Use values valid in Manage Geographies, e.g. the exact state name |
| SetCode required / invalid | Missing or wrong set | `COMMON` or a valid set code |
| Location code already exists in set | Duplicate code within the set | Prefix with initials |
| Invalid ActionReasonCode | Reason not loaded, or a typo | Run Lab 2 first |
| DELETE not allowed | Location doesn't support delete | Inactivate (`ActiveStatus = I`) |
| Organization/position later says location not valid on date | Location start date is **after** the referencing object's date | Start the location earlier (`2000/01/01`) |

---

## Lab 4 — Job (bridge to Positions)

A Position **requires a Job and a Department**, so create a job for your lab.

`Job.dat`
```
METADATA|Job|SourceSystemOwner|SourceSystemId|EffectiveStartDate|SetCode|JobCode|Name|ActiveStatus|FullPartTime|RegularTemporary|JobFamilyId(SourceSystemId)
MERGE|Job|AKP_LAB|AKP_JOB_DA|2000/01/01|COMMON|AKP_DA|AKP Data Analyst|A|FULL_TIME|R|AKP_JF_ANALYST
```
- `JobFamilyId(SourceSystemId)` points to the job family from Lab 1B. The owner is inherited from `AKP_LAB`.
- Jobs **can't be deleted** with HDL; inactivate them with `ActiveStatus = I`.

| Likely error | Fix |
|---|---|
| Job family not found | Lab 1B not loaded, or a different owner → add `JobFamilyId(SourceSystemOwner)` |
| Invalid FullPartTime / RegularTemporary | Use lookup codes (`FULL_TIME`/`PART_TIME`, `R`/`T`) |
| Job code exists in set | Prefix with initials |

---

## Lab 5 — Positions

**Goal:** create a single-incumbent and a shared position, update one, and learn the position-specific rules.

### Prerequisites (Oracle rules)
- **Job and Department are required** attributes of a position. Make sure the referenced job and department exist.
- Location and valid grades are optional, but if referenced they **must exist**.
- **A Position and its child components must have the same effective start date.**
- **Single** position: FTE **≤ 1** and headcount **= 1**.
- **Shared** position: supply an FTE and a headcount **≥ 1**.
- Position **code can be auto-generated**. Oracle recommends leaving it blank when automatic numbering is on.
- Use a **Business Unit** and **Department** that already exist in your instance (look them up in the UI). The BU name must be exactly right (see errors).

### 5A. Create a Single and a Shared position
`Position.dat`
```
METADATA|Position|SourceSystemOwner|SourceSystemId|EffectiveStartDate|EffectiveEndDate|BusinessUnitName|Name|PositionCode|DepartmentName|JobCode|JobSetCode|LocationCode|LocationSetCode|ActiveStatus|HiringStatus|PositionType|FTE|HeadCount|FullPartTime|RegularTemporary|ActionReasonCode
MERGE|Position|AKP_LAB|AKP_POS_DA1|2020/01/01||<Your BU Name>|AKP Data Analyst Chennai 1|AKP_DA_CHN_1|<Your Dept Name>|AKP_DA|COMMON|AKP_CHN01|COMMON|A|APPROVED|SINGLE|1|1|FULL_TIME|R|AKP_NEW_SITE
MERGE|Position|AKP_LAB|AKP_POS_DA_POOL|2020/01/01||<Your BU Name>|AKP Data Analyst Pool|AKP_DA_POOL|<Your Dept Name>|AKP_DA|COMMON|AKP_CHN01|COMMON|A|APPROVED|SHARED|5|5|FULL_TIME|R|AKP_NEW_SITE
```
⚠️ Verify against **your** template:
- whether the job is referenced by `JobCode`+`JobSetCode`, `JobId(SourceSystemId)` or `JobId`
- the exact lookup codes for **HiringStatus** (e.g. Approved/Proposed/Frozen) and **PositionType** (e.g. Single/Shared/None). Check the lookup type shown in View Business Objects.
- whether `PositionCode` should be blank (auto-numbering)

### 5B. Update FTE/headcount (date-effective)
```
SET PURGE_FUTURE_CHANGES N
METADATA|Position|SourceSystemOwner|SourceSystemId|EffectiveStartDate|FTE|HeadCount
MERGE|Position|AKP_LAB|AKP_POS_DA_POOL|2026/01/01|8|8
```

### 5C. Freeze hiring
```
SET PURGE_FUTURE_CHANGES N
METADATA|Position|SourceSystemOwner|SourceSystemId|EffectiveStartDate|HiringStatus
MERGE|Position|AKP_LAB|AKP_POS_DA1|2026/07/01|FROZEN
```

### Likely errors: Positions
| Error (paraphrased) | Cause | Fix |
|---|---|---|
| "The values {BU} aren't valid for the attribute BusinessUnitId" | BU name not exact, or the BU isn't usable by your user/setup | Copy the exact BU name; confirm the BU is valid for positions and your data access |
| Department invalid | Dept name wrong, not a *Department* classification, or not effective on the start date | Use an exact, active department; check its start date |
| Job not found | Lab 4 not loaded / wrong set code / wrong key hint | Load the job first; match the set code and key style |
| FTE/headcount validation | Single with FTE > 1 or headcount ≠ 1; shared with headcount < 1 | Follow the single/shared rules |
| Position code already exists / not allowed | Auto-numbering on, or a duplicate code | Leave the code blank (auto) or make it unique |
| Child components rejected | Child (e.g. valid grades) start date ≠ position start date | Same EffectiveStartDate for position and children |
| Location invalid on date | Location starts after the position | Back-date the location |
| Budget validation errors | Position budgeting enabled | Load budgets or stay within the allocated FTE/headcount |

---

## Lab 6 — Deleting & Inactivating

### What each lab object allows
| Object | DELETE via HDL? | Instead |
|---|---|---|
| Job Family | ✅ (if not referenced) | Inactivate |
| Action Reasons | ✅ | — |
| Location | ❌ | **Inactivate** (`ActiveStatus = I`) |
| Job | ❌ | **Inactivate** |
| Position | Check **View Business Objects** → deletion support | Inactivate / freeze hiring |
| Worker | ❌ | Cancel work relationship (hire in error) or terminate |

### Correct thinking about order
- **Deleting a parent** → children (and translations) go **automatically**. One DELETE on the parent discriminator is enough.
- **Deleting only a child** → use the **child discriminator**, and the parent stays.
- **Delete fails because something references it** (job family used by a job) → remove the reference or inactivate. This is a **cross-object** dependency, not parent/child.
- **Never** MERGE and DELETE the same record in one file. Use two sequential loads.

### Practice
1. Delete job family `AKP_JF_TECH` (not used by any job) → ✅ success.
2. Try to delete `AKP_JF_ANALYST` (used by job `AKP_DA`) → ❌ observe the error → inactivate instead.
3. Try `DELETE|Location|AKP_LAB|AKP_LOC_BLR` → ❌ observe "can't be deleted" → inactivate instead.

---

## Lab 7 — Break It On Purpose (Error Lab)

Make a copy of your working `JobFamily.dat` / `Location.dat` and introduce **one** error at a time. Predict **import or load error** first, then check.

| # | Break it like this | Expected stage | Expected symptom |
|---|---|---|---|
| 1 | Name the file `jobfamily.dat` | Import | Business object not recognised (names are case-sensitive) |
| 2 | Put the `.dat` inside a folder in the zip | Import | No business object found |
| 3 | Typo in a METADATA attribute (`JobFamilyNme`) | Import | Attribute unknown |
| 4 | Put a pipe inside a name: `R&D | Analytics` | Import | Wrong number of values |
| 5 | Date as `01-01-2000` | Import | Invalid date format |
| 6 | Put `SET PURGE_FUTURE_CHANGES N` **after** METADATA | Import | SET instruction error |
| 7 | Two METADATA lines for `JobFamily` | Import | Duplicate METADATA for component |
| 8 | SourceSystemOwner `XYZ` (not in lookup) | Load | Invalid source system owner |
| 9 | ActiveStatus `Active` instead of `A` | Load | Invalid value (lookup code) |
| 10 | Location with `ActionReasonCode = AKP_TYPO` | Load | Invalid action reason |
| 11 | Same JobFamilyName as an existing family | Load | Name must be unique |
| 12 | Position referencing job code `AKP_NOJOB` | Load | Job not found |
| 13 | Position SINGLE with FTE 2 | Load | FTE/headcount validation |
| 14 | Mixed file: 2 good job families + 1 with a broken line | Import (1) + Load (2 OK) | **Import error but Load = Success**: the "import error, load success" behaviour you asked about |
| 15 | MERGE and DELETE the same source key in one file | Unpredictable | Shows why that's forbidden |

**For each error, practise the full recovery:**
1. Find it: **Failed Lines** (import) or **Failed Objects** (load) → read the message → open **Object Hierarchy**.
2. Fix it:
   - import error → correct the file → **new zip**
   - load error → correct the source **or** fix in Attribute Details → **resubmit Load**
3. Re-verify the counts.

---

## Verification SQL (BIP)

Run in a BIP data model (read-only). *Table/column names are standard but verify them in your release.*

```sql
-- Job families you loaded
SELECT job_family_code, job_family_name, effective_start_date, effective_end_date, active_status
FROM   per_job_family_f_vl
WHERE  job_family_code LIKE 'AKP%'
ORDER  BY job_family_code, effective_start_date;

-- Locations (code, name, dates, status)
SELECT location_code, location_name, effective_start_date, effective_end_date, active_status
FROM   per_location_details_f_vl
WHERE  location_code LIKE 'AKP%'
ORDER  BY location_code, effective_start_date;

-- Jobs
SELECT job_code, name, effective_start_date, effective_end_date, active_status
FROM   per_jobs_f_vl
WHERE  job_code LIKE 'AKP%';

-- Positions
SELECT position_code, name, effective_start_date, effective_end_date,
       hiring_status, position_type, fte, max_persons
FROM   hr_all_positions_f_vl
WHERE  position_code LIKE 'AKP%'
ORDER  BY position_code, effective_start_date;

-- Your source keys (proves what HDL created and under which owner)
SELECT object_name, source_system_owner, source_system_id, surrogate_id
FROM   hrc_integration_key_map
WHERE  source_system_owner = 'AKP_LAB'
ORDER  BY object_name, source_system_id;
```

---

## Per-Load Checklist

**Before**
- [ ] Template downloaded after **Refresh Object**
- [ ] File name exact and case-sensitive; `.dat` at the zip root; UTF-8
- [ ] Owner registered; codes prefixed with initials
- [ ] Referenced objects exist and start **on/before** this object's start date
- [ ] SET lines at the top; `PURGE_FUTURE_CHANGES N` for updates
- [ ] No MERGE + DELETE of the same record

**After**
- [ ] Import Status and Load Status both checked
- [ ] **Failed Lines = 0** and **Failed Objects = 0**
- [ ] Loaded Objects = expected count
- [ ] Verified in the UI **and** with SQL (including `HRC_INTEGRATION_KEY_MAP`)
- [ ] Logged what failed, why, and the fix (your own error journal for interviews)

---
