# Oracle HDL — Parent–Child Relationships & How Key Matching Really Works

> Companion to the HDL study-guide series (builds on Phase 3: Keys).
> Goal: understand **exactly** how HDL knows (1) **which parent** a child line belongs to, (2) whether a MERGE line is a **new** record or an **existing** one, and (3) **which date split** to change. Plus the edge cases that break loads.
> Rules are from the Oracle HCM Data Loader guide, the HCM Business Objects guide, and Oracle's HDL tutorials. Items marked *(field note)* are practitioner experience: verify them in your environment.

---

## Table of Contents
1. [Three Kinds of Relationships in HDL](#1-three-kinds-of-relationships-in-hdl)
2. [Anatomy of a Child Line](#2-anatomy-of-a-child-line)
3. [The Parent-Reference Attribute](#3-the-parent-reference-attribute)
4. [Attribute Hints: (SourceSystemId) and (SourceSystemOwner)](#4-attribute-hints-sourcesystemid-and-sourcesystemowner)
5. [How HDL Matches a Line to an Existing Record](#5-how-hdl-matches-a-line-to-an-existing-record)
6. [How Each Key Type Is Looked Up](#6-how-each-key-type-is-looked-up)
7. [How Lines Are Grouped into Logical Objects](#7-how-lines-are-grouped-into-logical-objects)
8. [Worked Examples](#8-worked-examples)
9. [Edge Cases & Traps](#9-edge-cases--traps)
10. [Where to Look It Up: View Business Objects](#10-where-to-look-it-up-view-business-objects)
11. [Cheat Sheet](#cheat-sheet)
12. [Interview Questions](#interview-questions)

---

## 1. Three Kinds of Relationships in HDL

```
                 ┌───────────────────── ONE BUSINESS OBJECT (Worker.dat) ─────────────────────┐
                 │                                                                            │
                 │  Worker (top-level / parent component)                                     │
                 │   ├── PersonName          ◄── (A) PARENT–CHILD inside the hierarchy        │
                 │   ├── PersonEmail                                                          │
                 │   └── WorkRelationship                                                     │
                 │         └── WorkTerms     ◄── child of WorkRelationship                    │
                 │               └── Assignment  ── JobId ──────────┐                         │
                 └──────────────────────────────────────────────────┼─────────────────────────┘
                                                                    │ (B) FOREIGN-OBJECT REFERENCE
                                                                    ▼     (points to ANOTHER object)
                                                         Job (Job.dat), Location, Department, Grade…

 (C) LOGICAL OBJECT = all lines that belong to ONE top-level record (e.g. one worker) → loaded as one unit
```

| Relationship | Meaning | Example |
|---|---|---|
| **(A) Parent–child (component hierarchy)** | A component belongs to a parent component of the **same business object** | PersonName → Worker; Assignment → WorkTerms |
| **(B) Foreign-object reference** | A record points to a **different** business object | Assignment → Job / Location / Department |
| **(C) Logical object** | All lines of one top-level instance, processed together | One worker = Worker + names + emails + relationship + terms + assignment |

Each Oracle HCM business object is a hierarchy of components. The **parent component** is at the top, with **child and grandchild** components beneath it. Example: the Rating Model object has Rating Model as the parent of Rating Level and Rating Category.

**The key insight: parents and foreign objects are identified the same way, with keys.** In HDL you use unique keys to identify (1) the record you're creating or updating, (2) its **parent**, which can be **in the same file or already in Oracle**, and (3) any **objects it references**.

---

## 2. Anatomy of a Child Line

Every child line has **three parts**:

```
METADATA|PersonEmail|SourceSystemOwner|SourceSystemId|PersonId(SourceSystemId)|DateFrom|EmailType|EmailAddress
MERGE   |PersonEmail|LEGACY           |EMP1001_EML_W1 |EMP1001                 |2026/01/05|W1     |ravi@x.com
          └──────────┘ └────────── ① OWN KEY ─────────┘ └─── ② PARENT REFERENCE ─┘ └──── ③ DATA ─────────┘
          discriminator
```

| Part | Purpose | Answers the question |
|---|---|---|
| **① Own key** (local key) | Identifies **this** email record | "Is this email new or one that already exists?" |
| **② Parent reference** | Identifies **which person** owns it | "Whose email is this?" |
| **③ Data attributes** | The values to create/update | "What should it contain?" |

> **Direct answer to your question:** the column that makes a line a child of a specific parent is the **parent-reference attribute**, usually the **parent's surrogate-ID attribute name** (e.g. `PersonId`, `PeriodOfServiceId`, `WorkTermsAssignmentId`, `JobId`). You fill it with a key value that identifies the parent: a source key (with the `(SourceSystemId)` hint), a surrogate ID, or the parent's user-key attributes.

---

## 3. The Parent-Reference Attribute

### 3.1 Naming convention
The parent reference uses the **parent's surrogate-ID attribute name**. View Business Objects marks such attributes with key type **"parent surrogate ID"** (parent) or **"foreign-object reference"** (other objects).

### 3.2 Worker hierarchy: who points to whom
| Child component | Parent component | Parent-reference attribute | Typical value with source keys |
|---|---|---|---|
| PersonName | Worker | `PersonId(SourceSystemId)` | `EMP1001` |
| PersonEmail / PersonPhone / PersonAddress | Worker | `PersonId(SourceSystemId)` | `EMP1001` |
| PersonLegislativeData | Worker | `PersonId(SourceSystemId)` | `EMP1001` |
| WorkRelationship | Worker | `PersonId(SourceSystemId)` | `EMP1001` |
| WorkTerms | WorkRelationship | `PeriodOfServiceId(SourceSystemId)` | `EMP1001_WR` |
| Assignment | WorkTerms | `WorkTermsAssignmentId(SourceSystemId)` | `EMP1001_WT` |
| AssignmentSupervisor | Assignment | `AssignmentId(SourceSystemId)` | `EMP1001_ASG` |
| JobGrade (valid grades) | Job | `JobId(SourceSystemId)` | `JOB_SE01` |

Oracle's new-hire tutorial describes exactly this chain:
- PersonName identifies its Worker by supplying the worker's source system ID to the parent attribute **`PersonId` with the `(SourceSystemId)` hint**.
- **WorkTerms'** parent is the WorkRelationship, identified by **`PeriodOfServiceId`**.
- **Assignment's** parent is the WorkTerms, identified by **`WorkTermsAssignmentId`**.

### 3.3 The ways to fill a parent/foreign reference
| Method | METADATA attribute | Value | When |
|---|---|---|---|
| **Source key** | `PersonId(SourceSystemId)` | Parent's SourceSystemId, e.g. `EMP1001` | **Recommended**: parent created with a source key (or has a default `FUSION` key) |
| **Surrogate ID** | `PersonId` (no hint) | Parent's numeric ID, e.g. `300000012345678` | You have IDs from a BIP extract (they differ per environment) |
| **User key (alternative attributes)** | e.g. `PersonNumber` (instead of `PersonId`) | `1001` | Parent known by business key; View Business Objects lists which user-key attributes can replace the surrogate ID |
| **GUID** | GUID-hinted reference | Hex GUID | System integrations (rare by hand) |

```
-- Same PersonEmail child, parent referenced 3 ways
METADATA|PersonEmail|SourceSystemOwner|SourceSystemId|PersonId(SourceSystemId)|...   ← source key
METADATA|PersonEmail|SourceSystemOwner|SourceSystemId|PersonId|...                   ← surrogate ID
METADATA|PersonEmail|...|PersonNumber|...                                            ← parent's user key
```

### 3.4 Rule: be consistent with the parent's key type
Oracle's guidance for many objects: **reference the parent using the same key type used to identify the parent record**. With source keys, supply the parent's `SourceSystemId`; otherwise supply the **user-key attributes with the same values as the parent**.

### 3.5 Rule: source-key references need a local source key
You **can't** use source keys for parent or foreign references unless the **local record also uses a source key**. Source keys work for foreign objects only if those objects are **integration-enabled**.

---

## 4. Attribute Hints: (SourceSystemId) and (SourceSystemOwner)

**Anything in brackets after an attribute name is an attribute hint.** The hint tells HDL **what kind of value** you're putting in that column. For example, `JobId(SourceSystemId)` means "this column holds the job's **source system ID**, not its numeric ID".

| Hint | Meaning |
|---|---|
| `JobId` (no hint) | Value is the **surrogate ID** |
| `JobId(SourceSystemId)` | Value is the job's **SourceSystemId** |
| `JobId(SourceSystemOwner)` | Value is the job's **SourceSystemOwner** (only when it differs from the local record's owner) |

### Owner inheritance (very important)
- By default, the parent/foreign **SourceSystemOwner is inherited** from the line's own `SourceSystemOwner`. When the local PersonName and its parent Worker use the same owner, the parent's owner is taken from the local `SourceSystemOwner` attribute.
- Oracle's guide states the general rule: the `SourceSystemOwner` is common to all source keys in a record, so foreign objects identified by source key **must have the same owner as the record being merged**.
- **When the owners differ**, supply the foreign object's owner with the **`(SourceSystemOwner)` hint**:

```
METADATA|Assignment|SourceSystemId|SourceSystemOwner|JobId(SourceSystemId)|JobId(SourceSystemOwner)|EffectiveStartDate|EffectiveEndDate
MERGE|Assignment|234234|EBS-UK|12349|VISION|2013/01/01|4712/12/31
```
→ The assignment's own owner is `EBS-UK`; the job it points to was loaded under owner `VISION`.

> **Classic trap:** a Job created in the **UI** has the default key `FUSION` + `<JobId>`. A worker file with owner `LEGACY` that says `JobId(SourceSystemId)=300000…` will look for `LEGACY` + `300000…` and **not find it**. Fix: add `JobId(SourceSystemOwner)=FUSION`, or reference the job by **user key** (JobCode/SetCode) or by plain **surrogate ID** (`JobId`).

---

## 5. How HDL Matches a Line to an Existing Record

When HDL processes a MERGE line, it works through these steps:

```
 MERGE line
   │
   ▼
 STEP 1 ─ Which keys did the line supply for ITSELF?
          GUID? Source key? Surrogate ID? User key?
          → choose ONE by precedence:  GUID → Source key → Surrogate ID → User key
          (others ignored, no cross-check)
   │
   ▼
 STEP 2 ─ Resolve the PARENT reference (if a child)
          → look for the parent in the SAME FILE / data set, else in ORACLE
          → not found anywhere ⇒ error ("parent … could not be found")
   │
   ▼
 STEP 3 ─ Look up the LOCAL key
          ┌──────────────┬───────────────────────────────────────────────┐
          │ Found        │ → this is an UPDATE of that logical record     │
          │ Not found    │ → this is a CREATE (if the key type allows it) │
          └──────────────┴───────────────────────────────────────────────┘
          (GUID / surrogate not found ⇒ error: they can't create)
   │
   ▼
 STEP 4 ─ (Date-effective only) Which SPLIT?
          EffectiveStartDate = an existing split's start → CORRECTION of that split
          EffectiveStartDate = new date                  → UPDATE (new split; previous split end-dated)
          + maintenance mode (Replace / Retain + #RETAIN/#ALL) decides what happens to future splits
   │
   ▼
 STEP 5 ─ (_M objects, multiple changes per day) Which same-day change?
          EffectiveSequence identifies the change; EffectiveLatestChange marks the final one
   │
   ▼
 STEP 6 ─ Apply attribute values
          value → set it · blank → keep existing · #NULL → clear it
```

### Precedence reminder
When more than one key is supplied, HDL resolves them in the order **GUID → source key → surrogate ID → user key**. **No cross-validation** happens: if you give a GUID and a source key, the **GUID is used** and the source key is ignored, even if they point to **different** records.

The tutorial says the same for the common case: **when you supply both source keys and user keys, the source key is used** to identify each record.

---

## 6. How Each Key Type Is Looked Up

| Key type | Where HDL looks | Found → | Not found → |
|---|---|---|---|
| **Source key** (Owner + Id) | **`HRC_INTEGRATION_KEY_MAP`** (maps it to the surrogate ID) | Update that record | **Create** a new record and store the key in the key map |
| **User key** (e.g. JobCode + SetCode) | The **object's own table** (natural business columns) | Update | **Create** (all user-key attributes must be supplied) |
| **Surrogate ID** | The object's table (primary key) | Update | **Error** (IDs exist only after creation) |
| **GUID** | Integration Key Map | Update | **Error** |

### Details that matter
- **Source keys aren't stored on the record.** They live only in the key map, which is why a source key that was never registered (typo, different owner) is treated as **new** and HDL tries to create.
- **User keys are required on create**, and also on **update unless another key type** identifies the record. A user key can be several attributes; **supply them all** if you don't use another key.
- **Default source keys:** a record created without a source key gets owner **`FUSION`** and Id = **surrogate ID**. You can use that `FUSION` key to reference UI-created records.
- **Auto-generated person numbers:** if Oracle generates PersonNumber, you **can't** supply it, so a **source key becomes mandatory** for the Worker.
- **Changed user keys:** user-key values can change (and can be translated), while a **source key never changes** for the life of the record. That's why Oracle recommends source keys.

---

## 7. How Lines Are Grouped into Logical Objects

- During **import**, HDL groups file lines into **logical objects**: one occurrence of the business object, e.g. one Worker. The lines belonging to a logical object are identified by the **unique key of each component**.
- Children attach to their parent through the **parent reference**, so a worker's PersonName, emails and relationship join the worker's logical object.
- **Line order doesn't matter**; keys do. The same key on several lines with different `EffectiveStartDate`s = **date-effective history of one record**, not several records.
- **Only one METADATA line per component in a file.** There can be only one definition of which attributes are included for each component.
- The logical object **succeeds or fails as a whole**.
- **Foreign references across objects in one zip:** HDL loads objects in dependency order, so a foreign reference fails **only if the referenced object itself failed to load**.

---

## 8. Worked Examples

### Example 1: add a new email to a worker who was loaded by HDL
The worker exists with source key `LEGACY`/`EMP1001`. Just send the child line. The parent is found **in Oracle** through the key map.
```
METADATA|PersonEmail|SourceSystemOwner|SourceSystemId|PersonId(SourceSystemId)|DateFrom|EmailType|EmailAddress|PrimaryFlag
MERGE|PersonEmail|LEGACY|EMP1001_EML_H1|EMP1001|2026/11/01|H1|ravi.home@x.com|N
```
- Own key `LEGACY/EMP1001_EML_H1` → **not in the key map** → **CREATE**
- Parent `LEGACY/EMP1001` → found → attached to person 1001

### Example 2: update that same email later
```
MERGE|PersonEmail|LEGACY|EMP1001_EML_H1|EMP1001|2026/11/01|H1|ravi.sharma.home@x.com|N
```
- Own key **found** in the key map → **UPDATE** (address changed)

### Example 3: same intent, but a typo in the key
```
MERGE|PersonEmail|LEGACY|EMP1001_EML_HI|EMP1001|...      ← "HI" instead of "H1"
```
- Own key **not found** → HDL tries to **CREATE a second home email** → duplicate, or an error if only one email of that type is allowed. **Keys are identity; a typo creates a new identity.**

### Example 4: worker created in the UI (no source key known)
Option A: use the **default FUSION key** (owner `FUSION`, id = PERSON_ID):
```
METADATA|PersonEmail|SourceSystemOwner|SourceSystemId|PersonId(SourceSystemId)|PersonId(SourceSystemOwner)|DateFrom|EmailType|EmailAddress
MERGE|PersonEmail|LEGACY|EMP1001_EML_H1|300000012345678|FUSION|2026/11/01|H1|ravi.home@x.com
```
Option B: reference the parent by **surrogate ID**: `PersonId` = `300000012345678` (no hint).
Option C: replace the default keys once with **`SourceKey.dat`**, then use your own keys everywhere.

### Example 5: a date-effective child: correction vs update
The assignment `LEGACY/EMP1001_ASG` has splits starting 2024/01/01 and 2025/04/01.
```
SET PURGE_FUTURE_CHANGES N
METADATA|Assignment|SourceSystemOwner|SourceSystemId|EffectiveStartDate|EffectiveEndDate|...
MERGE|Assignment|LEGACY|EMP1001_ASG|2025/04/01|4712/12/31|...   ← matches existing split → CORRECTION
MERGE|Assignment|LEGACY|EMP1001_ASG|2026/12/01|4712/12/31|...   ← new date → UPDATE (new split)
```
(For assignment changes also send WorkTerms with the same dates.)

### Example 6: a foreign reference with a different owner
```
METADATA|Assignment|SourceSystemOwner|SourceSystemId|WorkTermsAssignmentId(SourceSystemId)|JobId(SourceSystemId)|JobId(SourceSystemOwner)|...
MERGE|Assignment|LEGACY|EMP1001_ASG|EMP1001_WT|JOB_SE01|SETUP|...
```
Parent WorkTerms: owner `LEGACY` (inherited). Job: owner `SETUP` (from the hint).

---

## 9. Edge Cases & Traps

| # | Situation | What happens / rule |
|---|---|---|
| 1 | **Parent not in the file and not in Oracle** | Child fails ("parent … could not be found, either in the file or in Oracle") |
| 2 | **Parent is in the file but fails** | The whole logical object fails (child included) |
| 3 | **Foreign object in the same zip fails** | Records referencing it fail. Otherwise HDL loads dependencies first. |
| 4 | **Local record has no source key, but the parent reference uses `(SourceSystemId)`** | Not allowed. Use a local source key, or reference the parent by surrogate ID / user key. |
| 5 | **Parent owner ≠ local owner** | Add the `(SourceSystemOwner)` hint; otherwise the lookup uses the local owner and fails |
| 6 | **Referencing UI-created records by source key** | Their key is `FUSION` + surrogate ID |
| 7 | **Typo or changed SourceSystemId** | Treated as a **new** record → duplicate or error. To change keys, use **SourceKey.dat**. |
| 8 | **Same SourceSystemId reused for two different real records** | HDL thinks they're one record → merged history, overlaps/gaps, wrong data |
| 9 | **New source key but the user key already exists** (e.g. a job created in the UI, now sent with a new source key and the same JobCode) *(field note)* | HDL tries to create → uniqueness/duplicate error. Use the record's existing key, or convert its key with SourceKey.dat. |
| 10 | **GUID + source key pointing to different records** | GUID wins silently. No error. |
| 11 | **User-key value renamed** (department name changed) | Old value no longer matches → "invalid value"/not found. User keys can change; source keys don't. |
| 12 | **Updating Worker components with only a user key** | Mostly **not possible**: the attribute you'd change is the one that identifies the record |
| 13 | **Person Name history** | Date-effective history needs **source keys**. Otherwise the user key is `PersonNumber` + `EffectiveStartDate` + `NameType`. |
| 14 | **User key changed during history** | When loading history where the user key changed, you must also supply a **source key** |
| 15 | **Auto-generated PersonNumber** | Can't supply PersonNumber, so a **source key is mandatory** |
| 16 | **Multiple occurrences of a component processed together** (e.g. several names/emails) | Process them together and **include the parent Worker** line. If the worker exists, the Worker line needs only its **key attributes**. |
| 17 | **Two METADATA lines for the same component in one file** | Not allowed: one METADATA line per component per file |
| 18 | **Assignment change without WorkTerms** | Error "WorkTerms records required". Send both with the same dates. |
| 19 | **Moving a child to a different parent** *(field note)* | Generally not supported by changing the parent reference. Delete and recreate (where delete is allowed), or use the proper business action (e.g. global transfer). |
| 20 | **Source keys and line order** | Line order is irrelevant; grouping uses keys. Never MERGE + DELETE the same key in one file. |
| 21 | **Foreign object not integration-enabled** | Can't be referenced by source key. Use its user key. |
| 22 | **Case/whitespace in keys** *(field note)* | Treat keys as exact strings: keep case consistent and trim spaces |

---

## 10. Where to Look It Up: View Business Objects

Data Exchange → **View Business Objects** → select an object → **component details**. For each component you'll see:
- its **parent component**, **file discriminator**, `.dat` file and translation object
- whether it **supports integration keys** (source key / GUID)
- for each attribute: whether it's a **key**, and the **key type**: *surrogate ID*, **parent surrogate ID** (the parent reference), or *foreign-object reference*
- for parent/foreign references: whether **integration keys** are supported (so `(SourceSystemId)` can be used)
- the **user-key attributes you can use instead of surrogate IDs**
- lookup types for lookup-validated attributes
- for multirow flexfield contexts, the flexfield attributes that act as **user keys**

> **Habit:** before writing a new child file, open the component in View Business Objects and note (1) its parent component, (2) the parent-reference attribute, (3) its own user-key attributes, (4) whether it supports source keys.

---

# CHEAT SHEET

| Concept | Remember |
|---|---|
| What links a child to its parent | The **parent-reference attribute** (parent's surrogate-ID name: `PersonId`, `PeriodOfServiceId`, `WorkTermsAssignmentId`, `AssignmentId`, `JobId`…) |
| How to fill it | `(SourceSystemId)` hint + parent's source ID · plain surrogate ID · parent's user-key attributes |
| Hint | Text in brackets after an attribute: tells HDL the value type |
| Owner | Inherited from the local `SourceSystemOwner`; use `Attr(SourceSystemOwner)` when different |
| Parent location | Same file **or** already in Oracle |
| New vs existing | Local key lookup: found = **update**, not found = **create** (source/user key only) |
| Lookup tables | Source key/GUID → `HRC_INTEGRATION_KEY_MAP`; user key/surrogate → object table |
| Which split | Same start date = **correction**; new start date = **update** |
| Same-day changes | `EffectiveSequence` + `EffectiveLatestChange` |
| Key precedence | GUID → Source → Surrogate → User; no cross-validation |
| Logical object | Lines grouped by component keys; all succeed or fail together |
| UI-created records | Default key `FUSION` + surrogate ID |
| One METADATA per component per file | Yes |

---

# INTERVIEW QUESTIONS

**Q1. In a PersonEmail line, which column tells HDL which person the email belongs to?**
<details><summary>Answer</summary>The parent-reference attribute <code>PersonId</code>, usually written <code>PersonId(SourceSystemId)</code> with the worker's SourceSystemId as the value. It could also be a plain surrogate PERSON_ID, or the parent's user key (PersonNumber).</details>

**Q2. What's the parent of Assignment, and which attribute references it?**
<details><summary>Answer</summary>WorkTerms, referenced by <code>WorkTermsAssignmentId</code> (with the <code>(SourceSystemId)</code> hint when using source keys). WorkTerms' parent is WorkRelationship via <code>PeriodOfServiceId</code>.</details>

**Q3. What is an attribute hint?**
<details><summary>Answer</summary>Text in brackets after an attribute name, e.g. <code>JobId(SourceSystemId)</code>. It tells HDL what kind of value is supplied: here the job's source system ID instead of its surrogate ID.</details>

**Q4. The worker has owner LEGACY; the job was loaded with owner VISION. How do you reference the job from the assignment?**
<details><summary>Answer</summary>Supply both <code>JobId(SourceSystemId)</code> and <code>JobId(SourceSystemOwner)</code> = VISION. Without the owner hint, the local owner (LEGACY) is assumed and the lookup fails.</details>

**Q5. How does HDL decide whether a MERGE creates or updates?**
<details><summary>Answer</summary>It picks the line's local key by precedence (GUID → source → surrogate → user) and looks it up: in the Integration Key Map for source keys/GUIDs, in the object's table for user keys/surrogate IDs. Found = update; not found = create (only possible with source or user keys).</details>

**Q6. Same source key on three lines with different EffectiveStartDates. Three records or one?**
<details><summary>Answer</summary>One logical record with three date-effective splits. The key must stay the same across history.</details>

**Q7. You need to add a phone to a worker created in the UI. You don't know any source key. Options?**
<details><summary>Answer</summary>Reference the parent by the default source key (owner FUSION + PERSON_ID, using the <code>(SourceSystemOwner)</code> hint if your line's owner differs), by surrogate ID <code>PersonId</code>, or by user key (PersonNumber). Or convert the default key once with SourceKey.dat.</details>

**Q8. Your Assignment line uses AssignmentNumber (no source key) and references the job with JobId(SourceSystemId). Why does it fail?**
<details><summary>Answer</summary>Source-key references are allowed only when the local record is also identified by a source key. Add a local source key, or reference the job by JobCode/SetCode or surrogate JobId.</details>

**Q9. Why can't you update a person's name using only the user key?**
<details><summary>Answer</summary>Most Worker components can't be updated with only a user key, because the attribute being changed is part of what identifies the record. Use the source key (or surrogate ID).</details>

**Q10. In one zip, Job.dat and Worker.dat are both present. A job fails to load. What happens to workers pointing to it?**
<details><summary>Answer</summary>HDL loads dependencies first. The references to that job fail (because the referenced object failed), so those workers fail. Workers pointing to other jobs load normally.</details>

**Q11. Can you supply two METADATA lines for PersonEmail in one Worker.dat?**
<details><summary>Answer</summary>No. Only one METADATA line per component of the hierarchy is allowed in a file.</details>

**Q12. A developer changed the SourceSystemId format mid-project and reloaded. Now workers have duplicate emails. Why, and what's the fix?**
<details><summary>Answer</summary>The new IDs weren't in the key map, so HDL treated them as new records and created duplicates. Delete the duplicates (PersonEmail can be deleted), then use SourceKey.dat to rename the old keys to the new convention before reloading.</details>

---