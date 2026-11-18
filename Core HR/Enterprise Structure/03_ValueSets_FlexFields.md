# Value Sets & Flexfields — Oracle Fusion HCM Study Guide

*As of Oct 31, 2026*

## 1. The big picture — why any of this exists

Oracle Fusion is SaaS: one codebase, thousands of clients. Oracle cannot change its screens or tables for each client. Flexfields and value sets are how a client gets *custom fields with custom rules* without touching Oracle's code.

**A simple analogy.** Think of a college admission form printed by the university. It has fixed boxes (Name, DOB, Address). But every department wants something extra: Sports wants "Preferred sport", Hostel wants "Room preference". So the university prints a few **blank, unlabelled boxes** at the bottom of every form.

- The **blank boxes** = spare columns Oracle already put in its tables (`ATTRIBUTE1`, `ATTRIBUTE2`, … `ATTRIBUTE_NUMBER1`, `ATTRIBUTE_DATE1` …).
- A **flexfield** = the thing that lets you switch on a blank box, give it a label, make it mandatory or optional, and show it on a page.
- One switched-on box = a **segment**.
- A **value set** = the rule for *what is allowed* inside that box: "only numbers 1–100", "pick from this list", "pick a city that belongs to the country you chose", "pick a person from the HR table".

**How the pieces connect (memorise this chain):**

`Value set (the rule)` → attached to a `Segment (the field)` → which lives inside a `Flexfield (DFF / KFF / EFF)` → which shows on a `Page (e.g. Manage Locations)` → and stores its data in a `Table column (e.g. ATTRIBUTE1)`.

**Where you will actually use this in a Core HR project:**

- Client wants extra fields on Person, Assignment, Location, Department, Job — e.g. "Blood group", "Badge number", "Union name", "Site code".
- Client wants a dropdown with their own values — e.g. list of company canteens, list of shift types.
- Client wants one field to filter another — e.g. pick State, then only that State's cities show.
- Client wants fields that change depending on something — e.g. India locations show *GST Number*, US locations show *County*. That is the **context** idea (section 9).

Value sets are reusable. Build one "Country" value set once and attach it to the Location DFF, the Person EFF and a report parameter. Change the values in one place and all three get the change.

## 2. Value sets — the validation types, in plain English

A value set answers one question: *"What is the user allowed to put in this field?"* The **validation type** is how strict that answer is. Oracle's own docs list five types you create yourself: [Format Only, Independent, Dependent, Subset, Table](https://docs.oracle.com/en/cloud/saas/human-resources/faucf/considerations-for-planning-value-sets.html).

**About the 6th one in your notes ("View Object"):** in Fusion SaaS you won't see "View Object" as a choice when you click *Create Value Set*. Oracle uses value sets built on internal view objects for its own delivered fields, so you may *see* them while browsing, but you build your own using the five types below. If your trainer counts it as a 6th type, that's what they mean — say "five that implementers create, plus Oracle-delivered VO-based ones" in an interview.

| Type | What the user experiences | Where the allowed values live | Typical HCM example |
| --- | --- | --- | --- |
| Format Only | Types freely, but must follow a format (length, number range, uppercase) | Nowhere — only rules | Badge number: 6 digits only; Notice period: 0–180 days |
| Independent | Picks from a fixed list you typed in | Value set values tables (`FND_VS_VALUES_B` / `_TL`) | Blood group, T-shirt size, Shift type |
| Dependent | Picks from a list that changes based on another field | Same tables, each value tied to a parent value | Country → City, State → District |
| Subset | Picks from a *trimmed* version of an existing independent list | Points to the independent set; you choose which values to keep | Full list = 7 weekdays; subset = Sat, Sun only |
| Table | Picks from live data in an Oracle table/view | The actual application table (e.g. `PER_PERSON_NAMES_F`) | Pick an employee, a department, a job |

**How to choose — ask yourself in this order:**

1. Does the data already exist in Fusion (people, departments, jobs, locations)? → **Table**. Never re-type data that already lives in a table; it goes stale the day someone adds a new department.
2. Is it a small, fixed list the client owns (shift types, canteens, blood groups)? → **Independent**.
3. Does the list depend on what was picked in an earlier field? → **Dependent**.
4. Do you need only *part* of a list that already exists as an independent value set? → **Subset** (reuse, don't duplicate).
5. Is it free text/number with only shape rules (length, range, digits)? → **Format Only**.

One more thing to know: when you use a value set on a DFF **context** field, only **Independent** or **Table** types are allowed, and the data type must be Character ([Oracle docs](https://docs.oracle.com/en/cloud/saas/sales/oawic/considerations-for-planning-value-sets.html)).

## 3. Format Only & Independent — your doubts answered

Your navigation is right: **Setup and Maintenance → your implementation project → + (Select and Add) → Task Lists and Tasks → search "Flexfields" → Define Flexfields → Manage Value Sets**. In many pods you can also just search the task **Manage Value Sets** directly from Setup and Maintenance.

### The header fields (same for every value set)

| Field | What it really means | Tip |
| --- | --- | --- |
| Value Set Code | The unique technical name. Flexfields and reports refer to this. | Cannot be renamed after saving. Use a convention like `XX_HCL_NOTICE_DAYS_VS` (client prefix, uppercase, underscores). Avoid spaces. |
| Description | Human note for other consultants. | Write *what it's for*, not just the name. |
| Module | Which product "owns" it. Used for grouping, search and for migrating setup between environments. | Pick the module you're working in, e.g. Global Human Resources. |
| Validation Type | Format Only / Independent / Dependent / Subset / Table. | Cannot be changed after save. |
| Value Data Type | Character, Number, Date, Date Time. | See below — the most common beginner mistake. |

### Doubt 1: Character or Number — how do I choose?

Ask: *will anyone ever do maths on this, or compare it as a number?*

- **Number** → yes: notice period in days, years of experience, percentage, allowance amount.
- **Character** → no, even if it *looks* numeric: PIN code, phone number, employee badge, Aadhaar-style IDs. Reason: numbers lose leading zeros (`007` becomes `7`) and you never add two phone numbers.
- **Date / Date Time** → real dates: certification expiry, visa end date.

Why it matters later: a DFF segment's data type must match the column it's stored in (`ATTRIBUTE1` = character, `ATTRIBUTE_NUMBER1` = number, `ATTRIBUTE_DATE1` = date). Wrong choice = you can't attach the value set to the segment you want.

### Doubt 2: Value Subtype — Text vs Translated text vs Numeric digits only vs Time

Only appears for **Character** data type ([Oracle docs](https://docs.oracle.com/en/cloud/saas/human-resources/faucf/considerations-for-planning-value-sets.html)).

| Subtype | Pick it when | Example |
| --- | --- | --- |
| Text | Normal codes/words, one language | Shift codes: DAY, NIGHT |
| Translated text | Client has users in more than one language and the *displayed* value must translate | A global client with French and English users |
| Numeric digits only | Character field that must contain only 0–9 | 6-digit PIN code, keeps leading zeros |
| Time (20:08) / Time (20:08:08) | A time of day | Shift start time |

**Trap:** Oracle doesn't let you switch from Text to Translated text after creation. If a client *might* go multi-language, choose Translated text up front — you don't have to enter translations immediately.

### Doubt 3: Precision, Scale, Min, Max — are they mandatory?

- **Precision** = maximum total digits (e.g. 5 → up to 99999). **Scale** = digits after the decimal point (e.g. 2 → 123.45). They apply to the Number type.
- **Minimum / Maximum value** = an allowed range. **Optional.** Leave blank if any value of the right format is fine.
- If the screen shows an asterisk (\*) on a field in your release, it's required there; otherwise treat these as optional but good practice. For a Number value set, always set Precision and Scale deliberately — it's what stops someone entering 3.14159 days of notice.
- **Trap with Character + Min/Max:** text is compared alphabetically, not numerically. With min `1` and max `100` on a Character set, `9` is *rejected* because "9" sorts after "100". Use Number type when you need a numeric range.

### Doubt 4: "Uppercase only" and "Zero fill" — what do these checkboxes do?

- **Uppercase only:** whatever the user types is converted to capitals. `ind` → `IND`. Use it for codes so you never get `Ind`, `IND` and `ind` stored as three different things — reports and comparisons stay clean.
- **Zero fill:** pads purely numeric values with leading zeros up to the maximum length. Max length 6, user types `4512` → stored as `004512`. Use it when an external system (badge reader, payroll vendor) expects fixed-length codes.

### Doubt 5: Manage Values — what each column is for (Independent)

| Column | Meaning | Why you care |
| --- | --- | --- |
| Value | The code that is **stored** in the table | Keep short and stable: `DAY`, `NIGHT`, `ROT` |
| Description | The friendly text users can see next to it | "Day shift (9am–6pm)" |
| Enabled | Tick = selectable; untick = hidden from the list | Use instead of deleting a value already used on records |
| Start Date / End Date | Value is valid only inside this window | Plan ahead: a new canteen from 1 Jan, an old one ends 31 Dec |
| Sort Order | Order in the dropdown | Put the most-used value first, or a logical order (Small, Medium, Large) |

**Enabled vs End Date — the interview difference:** unticking Enabled hides the value *now and forever* until you tick it again. End Date retires it *from a date* — great for planned changes. In both cases, **old records keep the stored value**; it just can't be newly selected.

### Quick practice to cement it

1. Format Only, Number: `XX_NOTICE_DAYS_VS`, Precision 3, Scale 0, Min 0, Max 180.
2. Format Only, Character, Numeric digits only: `XX_PIN_CODE_VS`, Max length 6, Zero fill ticked.
3. Independent, Character, Text, Uppercase only: `XX_SHIFT_TYPE_VS` with values DAY, NIGHT, ROT.

## 4. Dependent value set — Country → City

A dependent value set is a list that **filters itself based on a parent value**. Oracle describes it as one sub-list per value of the independent set. Your steps were right; here's the full flow with the *why* at each step.

**Step A — Parent (Independent) value set**

1. Create `XX_COUNTRY_VS`: Validation Type *Independent*, Data Type *Character*, Subtype *Text*, Max length 30, Uppercase only ticked.
2. Manage Values → add `IN` (India), `US` (United States), `AE` (UAE). Short codes are stored; descriptions are shown.

**Step B — Child (Dependent) value set**

1. Create `XX_CITY_VS`: Validation Type *Dependent*, Data Type *Character*, Subtype *Text*, Max length 30.
2. **Independent Value Set** field → enter `XX_COUNTRY_VS`. *This is the link.* It tells Oracle "every city must belong to one country from that list." You can only pick an *Independent* set here — not another dependent one.
3. Save, then Manage Values. Oracle now asks for an **Independent Value** first. Pick `IN`, add `CHENNAI`, `BENGALURU`, `MUMBAI`. Switch to `US`, add `NEW YORK`, `AUSTIN`. Switch to `AE`, add `DUBAI`.

**What happens behind the scenes:** each city row stores a pointer to its country value. That's why the same city name can exist under two countries without clashing — `HYDERABAD` under India and under Pakistan are two separate rows.

**Using it (the part people forget):** a dependent value set does nothing on its own. When you attach these to flexfield segments, the **Country segment must come before the City segment** (lower sequence number) in the same flexfield. At runtime, the user picks Country → Fusion filters City to that country's list. If the user changes Country, the old City is no longer valid and must be re-picked.

**Order of creation is forced:** independent first, then dependent, then values for the independent, then values for the dependent. If the parent value doesn't exist, you can't add children under it.

## 5. Subset value set — hands-on exercise

A subset value set is **a filtered window onto an existing independent list**. You don't type new values; you pick which of the parent's values to show. Oracle's own example: a weekdays list, and a weekend subset of just Saturday and Sunday ([Oracle docs](https://docs.oracle.com/en/cloud/saas/human-resources/23d/faucf/considerations-for-planning-value-sets.html)).

### The scenario

The client has one master list of worker categories. HR wants a new field **"Bonus eligible category"** where only Full Time and Part Time can be picked. Contractors, interns and consultants must never appear there. They don't want a second list to maintain.

### Step 1 — Parent (Independent) value set

Create `XX_WORKER_CATEGORY_VS`: Independent, Character, Text, Max length 30, Uppercase only.

Manage Values:

| Value | Description | Sort order |
| --- | --- | --- |
| FULL_TIME | Full-time employee | 1 |
| PART_TIME | Part-time employee | 2 |
| CONTRACT | Contract worker | 3 |
| INTERN | Intern | 4 |
| CONSULTANT | External consultant | 5 |

### Step 2 — The Subset value set

Manage Value Sets → Create.

| Field | Enter | Why |
| --- | --- | --- |
| Value Set Code | `XX_BONUS_ELIG_CATEGORY_VS` | Name says what the *filtered* list is for |
| Description | Worker categories eligible for annual bonus | Future consultants will thank you |
| Module | Global Human Resources | Groups it with your HR setup |
| Validation Type | **Subset** | This is the whole point |
| Value Data Type | Character | Must match the parent; a subset can't change the type of its parent's values |
| Independent Value Set | `XX_WORKER_CATEGORY_VS` | The parent list this window looks into. It must already exist and be Independent |

Save and close.

### Step 3 — Pick the values

Open `XX_BONUS_ELIG_CATEGORY_VS` → Manage Values → Create. Instead of typing a free value, you choose from the parent's list. Add `FULL_TIME` and `PART_TIME`. You can give each its own enabled flag, dates and sort order inside the subset.

### Step 4 — Test it

Attach `XX_BONUS_ELIG_CATEGORY_VS` to a test DFF segment (section 8 shows how), deploy, and open the page. The dropdown shows only Full Time and Part Time.

### What each choice is doing

- **One source of truth.** Rename "Full-time employee" in the parent and the subset shows the new description too — nothing to sync.
- **You can't sneak in a new value.** If HR wants `APPRENTICE` in the bonus list, it must first be added to the parent. This is by design: the subset is always a true subset.
- **Subset ≠ Dependent.** A dependent list changes based on *another field the user picked* (Country → City). A subset is a fixed slice, decided by you at setup time; the user doesn't pick anything first.
- **Parent changes ripple.** If you disable or end-date a value in the parent, test the subset straight away — the subset only points at the parent's value, so treat the parent as the master switch.

**Interview one-liner:** "Subset value sets let me reuse an independent list but expose only part of it, so there's one list to maintain instead of two."

## 6. Table value set — show full name from PER_PERSON_NAMES_F

A table value set **reads its list live from a Fusion table or view**, using a small SQL you write in pieces. Nothing is typed into Manage Values — new hires appear automatically.

### First, understand the table (this is where the traps are)

`PER_PERSON_NAMES_F` holds people's names. Two facts make it tricky:

- **`_F` = date-effective.** One person can have several rows over time (a name change after marriage = a new row with new `EFFECTIVE_START_DATE` / `EFFECTIVE_END_DATE`).
- **`NAME_TYPE`.** A person can have a `GLOBAL` name row and also local/legislative name rows. Without a filter, one person appears twice.

So the same `PERSON_ID` can appear on many rows. Your WHERE clause must narrow it to **one row per person**.

### Two ways to read the requirement

- **Literal ask:** user enters the Person ID, sees the full name beside it → *Value = PERSON_ID, Description = FULL_NAME*.
- **What clients usually want:** user picks a *name* from a searchable list, Fusion quietly stores the ID → *Value = FULL_NAME, ID = PERSON_ID*. Storing the ID is safer: names change, IDs don't.

Below is the second (recommended) design, with the literal one noted.

### The fields

Manage Value Sets → Create → Validation Type **Table**, Value Data Type **Character** (because the displayed value is a name).

| Field | Enter | What it's doing |
| --- | --- | --- |
| Value Set Code | `XX_PERSON_FULL_NAME_VS` | — |
| FROM Clause | `PER_PERSON_NAMES_F ppnf` | The table (the SQL `FROM`), with a short alias |
| Value Attributes Table Alias | `ppnf` | Tells Oracle which alias the value/ID/description columns belong to |
| Value Column Name | `FULL_NAME` | What the user **sees and searches** in the list |
| Value Column Type / Length | VARCHAR2 / 240 | Must match the real column |
| Description Column Name | `DISPLAY_NAME` (optional) | Extra text shown next to the value |
| ID Column Name | `PERSON_ID` | What actually gets **stored** in the flexfield column |
| ID Column Type | NUMBER | Matches `PERSON_ID` |
| WHERE Clause | `ppnf.NAME_TYPE = 'GLOBAL' AND TRUNC(SYSDATE) BETWEEN ppnf.EFFECTIVE_START_DATE AND ppnf.EFFECTIVE_END_DATE` | Leaves exactly one current row per person |
| ORDER BY Clause | `ppnf.FULL_NAME` | Alphabetical dropdown instead of random order |

Literal version: Value Column = `PERSON_ID` (Number data type), Description Column = `FULL_NAME`, no ID column.

**Mental model** — Oracle stitches your pieces into roughly this query:

```sql
SELECT ppnf.FULL_NAME   -- value (shown)
     , ppnf.PERSON_ID   -- id (stored)
FROM   PER_PERSON_NAMES_F ppnf
WHERE  ppnf.NAME_TYPE = 'GLOBAL'
AND    TRUNC(SYSDATE) BETWEEN ppnf.EFFECTIVE_START_DATE AND ppnf.EFFECTIVE_END_DATE
ORDER  BY ppnf.FULL_NAME
```

### Things that will bite you

- **Two employees called "Rahul Sharma".** The value must identify one row. Using an ID column solves storage, but users still can't tell them apart in the list — add person number or email via the Description column or a join in the FROM clause.
- **Terminated workers still show.** The name table doesn't know about employment. If the field should list only active workers, join to assignment/period-of-service tables and filter on status.
- **No data security.** A table value set runs its SQL directly; it does **not** apply the user's HCM data roles. A manager could see every name in the company. Keep sensitive tables out, and filter in the WHERE clause.
- **Performance.** Huge tables + no filter = slow list. Always filter.
- **Referencing other fields.** The WHERE clause can use bind variables such as `:{SEGMENT.<segment_code>}` to filter by another segment in the same flexfield (e.g. only people in the department chosen above). Check the exact bind syntax in your release before using it in production.

## 7. Flexfields — DFF vs KFF vs EFF

All three let you add fields without code. The difference is **what kind of extra information** you're capturing and **how many rows** of it you need.

**DFF (Descriptive Flexfield)** — *"a few extra fields on this record."* One record, one set of extra values, stored in the spare `ATTRIBUTE` columns of the **same table**. Example: on a Location, add Site Code and Canteen Available (Y/N).

**EFF (Extensible Flexfield)** — *"a whole extra section, possibly with many rows."* Stored in a separate **extra-information table**, so it isn't limited by the base table's spare columns. Contexts can be **single-row** (one value set per record) or **multi-row** (a table on the page: e.g. a person's list of professional certifications, each with name, number and expiry). Contexts are grouped into categories and placed on pages.

**KFF (Key Flexfield)** — *"an intelligent code made of parts."* Not extra info — it's the **identity** of something. A Job code like `IT.DEVELOPER.L3` is made of segments (Function, Family, Level). Each unique combination is stored **once** in a combinations table and gets an ID; records point to that ID. In HCM you'll meet Job, Position, Grade, People Group and Costing KFFs.

| | DFF | EFF | KFF |
| --- | --- | --- | --- |
| Purpose | Add descriptive fields | Add sections / repeating rows | Build a multi-part identifying code |
| Rows per record | One | One or many | One combination, referenced by ID |
| Where data lives | Same table, `ATTRIBUTE*` columns | Separate extra-info table | Combinations table + ID on the record |
| Limit | Number of spare columns | Practically very high | Number of segments in the structure |
| Key words | Global segment, context, context-sensitive segment | Category, context, page, single/multi-row | Structure, structure instance, segment, combination |
| HCM example | Location: Site Code | Person: Certifications (multi-row) | Job: Function.Level |

**How to decide in a project:** a handful of single-value fields → DFF. Many fields, a logical section of its own, or repeating rows → EFF. The client is designing the *code* of jobs/grades/costing → KFF.

Common to all three: you configure them and then **deploy** before anyone sees them. Section 10 walks through an EFF hands-on, including single-row vs multi-row behaviour.

## 8. DFF hands-on — Country & City on Location

### Why you can't "create" a DFF in SaaS

Creating (registering) a brand-new DFF means three things: adding spare columns to a database table, teaching the application's data layer about them, and placing a flexfield region on a page. All three are **code and database changes**.

In SaaS you have no database access, no ability to change Oracle's code, and every client runs the same code. So Oracle **pre-registers** DFFs on most business objects (Person, Assignment, Location, Department, Job…) and ships the spare `ATTRIBUTE` columns already in the tables. Your job as an implementer is to **configure** them: add segments, contexts and value sets, then deploy. In an interview: *"Registration is Oracle's job; configuration is ours."*

### Before you start

- Value sets `XX_COUNTRY_VS` (Independent) and `XX_CITY_VS` (Dependent on it) from section 4 must exist.
- The Location DFF code is `PER_LOCATIONS_DF` ("Location Attributes"). Its data is stored on `PER_LOCATION_DETAILS_F` in columns `ATTRIBUTE1`, `ATTRIBUTE2`… plus `ATTRIBUTE_CATEGORY` for the context ([Oracle table reference](https://docs.oracle.com/en/cloud/saas/human-resources/25a/oedmh/perlocationdetailsf-19082.html)).

### Steps

1. **Setup and Maintenance → Manage Descriptive Flexfields** (or via your implementation project → Define Flexfields).
2. Search Flexfield Code `PER_LOCATIONS_DF` → select it → **Edit**.
3. Look first. Note which `ATTRIBUTE` columns existing global segments already use, so you don't collide.
4. In **Global Segments → Create**, fill the Country segment:

| Field | Enter | Why |
| --- | --- | --- |
| Name / Code | Country / `XX_COUNTRY` | Code is permanent; used by HDL, reports, bind variables |
| API Name | Auto-filled (e.g. `xxCountry`) | Used by REST APIs and integrations |
| Sequence | 10 | Display order. **Must be lower than City** for the dependency to work |
| Enabled | Ticked | Unticked = hidden but kept |
| Data Type | Character | Matches the value set |
| Table Column | A free `ATTRIBUTE` column, e.g. `ATTRIBUTE1` | Where the value is physically stored |
| Value Set | `XX_COUNTRY_VS` | The rule for what can be entered |
| Prompt | Country | The label users see |
| Display Type | List of Values (or Drop-down List) | How the field looks |
| Required | Your call | If ticked, every location save needs it — including old locations |
| BI Enabled | Ticked | Makes it available in OTBI reports later |

5. Save and Close. Create the **City** segment the same way: Code `XX_CITY`, Sequence 20, Table Column `ATTRIBUTE2`, Value Set `XX_CITY_VS`, Prompt City.
6. Save and Close → back on Manage Descriptive Flexfields, select the row → **Deploy Flexfield**. Wait for the deployment to finish successfully.
7. **Sign out and sign back in.** The page caches the old definition; you won't see the fields until you log in again.
8. Test: **My Client Groups → Workforce Structures → Locations → Create** (or edit one). Pick Country = India → City shows only Chennai, Bengaluru, Mumbai. Change Country to US → City clears, shows only US cities.

**Why both are Global segments here:** global segments appear for *every* location, always. That's right if every location needs Country and City. If India locations need extra fields that US locations don't, that's where **context** comes in — next section.

**About "global segments can be deleted":** true, but careful. Deleting a segment doesn't wipe what's already stored in `ATTRIBUTE1`. If you later map a *new* segment to the same column, the old values suddenly reappear under the new label. Clean the data first, or pick a different column.

## 9. The Context field — what, why, how

The context field is **a switch that decides which extra fields appear**. Pick "Factory" and the page shows factory fields; pick "Office" and it shows office fields. Global segments ignore the switch and always show.

```
              ┌──────────────────────────────────┐
              │ Global segments (every location) │
              │      Country · ATTRIBUTE1        │
              │      City    · ATTRIBUTE2        │
              └────────────────┬─────────────────┘
                               │
                      ◇ Location Type? ◇     ← context segment;
                               │               value saved in ATTRIBUTE_CATEGORY
          ┌────────────────────┼──────────────────────┐
          ▼                    ▼                      ▼
 ┌──────────────────┐ ┌────────────────────┐ ┌──────────────────┐
 │ OFFICE context   │ │ FACTORY context    │ │ WAREHOUSE context│
 │ Seating capacity │ │ Safety officer     │ │ Storage area     │
 │   · ATTRIBUTE3   │ │   · ATTRIBUTE3     │ │   · ATTRIBUTE3   │
 │ Cafeteria (Y/N)  │ │ Shifts/day         │ │ Cold storage     │
 │   · ATTRIBUTE4   │ │ · ATTRIBUTE_NUMBER1│ │   · ATTRIBUTE4   │
 └──────────────────┘ └────────────────────┘ └──────────────────┘

 ATTRIBUTE3 means something different in each context;
 a row holds only one context, so they never clash.
```

Pick FACTORY and only the factory box's fields join Country and City on the page; the other contexts' fields stay hidden.

### The vocabulary (interviewers love these four terms)

| Term | Plain meaning | Location example |
| --- | --- | --- |
| Global segment | Shows on every record, always | Country, City |
| Context segment | The switch field itself (one per DFF) | "Location Type" |
| Context (context value) | One position of the switch, created in *Manage Contexts* | OFFICE, FACTORY, WAREHOUSE |
| Context-sensitive segment | A field that belongs to one context and shows only when that context is chosen | FACTORY → Safety Officer, Shifts per Day |

### Why do we need it?

1. **Different records need different fields.** A factory needs a safety officer; an office doesn't. Without contexts you'd show every field to everyone and users would leave half of them blank or fill them wrongly.
2. **It saves columns — the clever bit.** A record can only be in *one* context at a time. So a context-sensitive segment in OFFICE and another in FACTORY can **share the same `ATTRIBUTE` column**. OFFICE's "Seating Capacity" and FACTORY's "Safety Officer" can both use `ATTRIBUTE3`. Global segments can't share — they're always on.
3. **Cleaner UI and validation.** Required fields can be required only for the context that needs them.

### Where it's stored

The chosen context code (e.g. `FACTORY`) is stored in the **`ATTRIBUTE_CATEGORY`** column of the same row. When the page loads, Fusion reads `ATTRIBUTE_CATEGORY`, then knows how to interpret `ATTRIBUTE3` — as Seating Capacity or as Safety Officer.

### How the context value gets chosen

- **User picks it** — context segment displayed as a list (most common for learning).
- **Defaulted or derived** — Fusion fills it from another value (e.g. the record's legislation or business unit) and you can hide it. Users then just see the right fields appear. In HCM, many delivered DFFs use a legislation-based context for exactly this reason.

If you attach a value set to the context segment, it must be **Independent or Table** and **Character**, and the values must not be longer than the `ATTRIBUTE_CATEGORY` column (30 characters on the Location table).

### Practical: add a context to the Location DFF

1. Manage Descriptive Flexfields → `PER_LOCATIONS_DF` → Edit.
2. **Context Segment** section: Prompt *Location Type*, Display Type *List of Values*, Required as needed. Leave Value Set blank (the list then comes from the contexts you create) or use an Independent value set whose values match your context codes.
3. Click **Manage Contexts → Create**: Display Name *Factory*, Code `FACTORY`, API name auto, Enabled.
4. Inside FACTORY → **Context Sensitive Segments → Create**: *Safety Officer* (`ATTRIBUTE3`, use your `XX_PERSON_FULL_NAME_VS` table value set!) and *Shifts per Day* (number column, Format Only value set 1–3).
5. Create context `OFFICE` → segment *Seating Capacity* on `ATTRIBUTE3` again — same column, different context. That's legal and normal.
6. Save and Close → **Deploy Flexfield** → sign out/in → open a location, choose Factory, then Office, and watch the fields swap.

## 10. EFF hands-on — Extra Information & Legislative Information on Locations

The **Extra Information** and **Legislative Information** regions you see on a location are two EFFs. Their data does **not** go into `PER_LOCATION_DETAILS_F`. It goes into separate tables that point back to the location through `LOCATION_ID` — that's the "saved in another table with reference" in your notes.

### The two Location EFFs

| Region on the Location page | EFF code | Category | Stored in | Key columns |
| --- | --- | --- | --- | --- |
| Extra Information | `PER_LOCATION_INFORMATION_EFF` | `HCM_LOC` | `PER_LOCATION_EXTRA_INFO_F` | `LOCATION_ID` (link), `INFORMATION_TYPE` / `LEI_INFORMATION_CATEGORY` (which context), `LEI_INFORMATION1…` (segments) |
| Legislative Information | `PER_LOCATION_LEG_EFF` | `HCM_LOC_LEG` | `PER_LOCATION_LEG_F` | `LOCATION_ID` (link), `LLE_INFORMATION_CATEGORY` (which context), `SEQUENCE_NUMBER` (tells multi-row rows apart), `LLE_INFORMATION1…` (segments) |

Sources: Oracle's [Location HDL guide](https://docs.oracle.com/en/cloud/saas/human-resources/fahbo/guidelines-for-loading-locations.html), [PER_LOCATION_EXTRA_INFO_F](https://docs.oracle.com/en/cloud/saas/human-resources/24b/oedmh/perlocationextrainfof-26433.html) and [PER_LOCATION_LEG_F](https://docs.oracle.com/en/cloud/saas/human-resources/25a/oedmh/perlocationlegf-12077.html) table references.

**Which one do I use?**

- **Extra Information** → company-specific facts any location might need (facility details, safety certificates, site contacts).
- **Legislative Information** → country-specific or statutory facts. Its contexts are tied to a legislation (country), so a context built for India appears only on locations where that legislation applies. Oracle itself delivers contexts here (e.g. Canadian workers' compensation overrides).

Both tables end in `_F`, so EFF rows are **date-effective** just like the location itself.

### EFF vocabulary (different from DFF!)

```
EFF  (PER_LOCATION_INFORMATION_EFF)
 └─ Category            HCM_LOC          → "which contexts are allowed for locations"
     └─ Page            Site Details     → a tab/section on the UI that groups contexts
         └─ Context     Fire Safety Certificates   (single row OR multiple rows)
             └─ Segment Certificate Type, Certificate No., Expiry Date
```

- **Context** = a named group of fields (Oracle also calls it an *attribute group*). In an EFF, **every segment belongs to a context** — there are no global segments.
- **Category** = decides which contexts are valid for the business object. A context that isn't associated with the category never shows.
- **Page** = how contexts are arranged on screen.
- **Usage** = which screen/flow the context or page is used on.

### Single row vs multiple rows — the behaviour

When you create an EFF context, you pick its **Behavior**: *Single Row* or *Multiple Rows*. Oracle's rule: single-row contexts show as a form; multi-row contexts show as a table where users can enter a list ([Oracle docs](https://docs.oracle.com/cd/E29505_01/fusionapps.1111/e15524/flex_ext.htm)).

| | Single Row | Multiple Rows |
| --- | --- | --- |
| Question it answers | "What is *the* value for this location?" | "What is the *list* of things for this location?" |
| How the user sees it | A form: label + field, like a DFF | A table: each segment is a column; Add/Delete row buttons |
| Rows stored per location per context (per date) | Exactly one | Zero, one or many |
| Location example | Site Facility Details: Building Owner, Number of Floors, Lease End Date | Fire Safety Certificates: Type, Number, Issue Date, Expiry Date — one row per certificate |
| How rows are told apart | Not needed — only one | A sequence number per row (`SEQUENCE_NUMBER` on the legislative table; HDL asks you for one per row) |
| In a BIP report | One row per location — easy joins | Many rows per location — joining it directly multiplies your location rows |

**Rule of thumb:** if the honest answer to "can there be more than one?" is *yes, ever*, make it Multiple Rows. Deciding later is painful, because data entered as one shape doesn't convert to the other — treat the choice as permanent once data exists.

### Practical: build both kinds on Location Extra Information

**Before you start:** create any value sets you need, e.g. `XX_FIRE_CERT_TYPE_VS` (Independent: `NOC`, `AUDIT`, `DRILL`).

1. **Setup and Maintenance → Manage Extensible Flexfields.**
2. Search Flexfield Code `PER_LOCATION_INFORMATION_EFF` → select → **Edit**.
3. **Manage Contexts → Create** the single-row context:
    - Display Name *Site Facility Details*, Code `XX_SITE_FACILITY`, API name auto-filled.
    - **Behavior: Single Row.** Enabled.
    - **Context Usages:** add the location usage so it can appear on location pages. Leave view/edit privileges blank unless only certain roles should see or edit it.
    - **Context Sensitive Segments → Create:** *Building Owner* (Character, Format Only value set), *Number of Floors* (Number), *Lease End Date* (Date). Pick a table column of the matching data type for each.
    - Save and Close.
4. **Create** the multi-row context the same way:
    - Display Name *Fire Safety Certificates*, Code `XX_FIRE_CERT`.
    - **Behavior: Multiple Rows.**
    - Segments: *Certificate Type* (`XX_FIRE_CERT_TYPE_VS`), *Certificate Number* (Character), *Issue Date* (Date), *Expiry Date* (Date).
    - Save and Close.
5. Back on the Edit Extensible Flexfield page, select **category `HCM_LOC`**. In its **Associated Contexts**, add both contexts. *Skipping this is the #1 reason an EFF "doesn't show up".*
6. Under the same category, **Pages → Create** a page, e.g. *Site Details* (code `XX_SITE_DETAILS`), and add both contexts to it. Set the page's usage to the location page.
7. Save and Close → **Deploy Flexfield** → wait for success → **sign out and sign in**.
8. Test: **My Client Groups → Workforce Structures → Locations** → edit a location → **Extra Information**.
    - *Site Facility Details* shows as a form. Fill it once.
    - *Fire Safety Certificates* shows as a table. Click Add three times for three certificates; delete one; save. Each row is its own record.

**Legislative Information** works the same way on `PER_LOCATION_LEG_EFF`, category `HCM_LOC_LEG`. The difference: tie the context to the legislation it belongs to. Test on a location whose country matches — on other countries' locations it won't appear.

### Behind the scenes

```sql
-- Multi-row EFF: expect several rows per location
SELECT lei.location_id
     , lei.information_type      AS context_code   -- e.g. XX_FIRE_CERT
     , lei.lei_information1      AS cert_type      -- column depends on what you mapped
     , lei.effective_start_date
FROM   per_location_extra_info_f lei
WHERE  lei.information_type = 'XX_FIRE_CERT'
AND    TRUNC(SYSDATE) BETWEEN lei.effective_start_date AND lei.effective_end_date
```

- Each context row is one row in the EFF table, with `LOCATION_ID` pointing back to the location. Single row = one row per location per context; multi row = many.
- Because segment columns are reused across contexts (just like DFF context-sensitive segments), always filter on the context column before reading `LEI_INFORMATION1`.
- **HDL:** these load as child records of the Location object — `LocationExtraInfo` (Extra Information, `EFF_CATEGORY_CODE` = `HCM_LOC`) and `LocationLegislative` (Legislative Information, `HCM_LOC_LEG`), with one line per row for multi-row contexts ([Oracle HDL guide](https://docs.oracle.com/en/cloud/saas/human-resources/fahbo/guidelines-for-loading-locations.html)).

### EFF vs DFF context — don't mix them up

| | DFF context | EFF context |
| --- | --- | --- |
| How many active per record | One (picked by the context segment) | Many at once — every associated context shows |
| Global segments | Yes | No — all segments are in a context |
| Multiple rows | No | Yes, if the context is Multiple Rows |
| Where data lives | Same table (`ATTRIBUTE*`) | Separate table linked by ID |
| Extra setup | Context segment | Category association + page |

## 11. Behind the scenes — what happens to the tables

Two kinds of data are involved, and mixing them up is the #1 internals mistake: **setup metadata** (what fields exist and their rules) and **transaction data** (what users actually typed).

### Setup metadata — where your configuration lives

| What | Table(s) | Holds |
| --- | --- | --- |
| Value set header | `FND_VS_VALUE_SETS` | Code, validation type, data type, length, uppercase/zero-fill flags |
| Values (Independent/Dependent/Subset) | `FND_VS_VALUES_B` | Value code, enabled flag, start/end dates, sort order, link to parent value for dependent sets |
| Value descriptions / translations | `FND_VS_VALUES_TL` | One row per language ([Oracle table reference](https://docs.oracle.com/en/cloud/saas/applications-common/25d/oedma/fndvsvaluestl-5214.html)) |
| DFF definitions | `FND_DF_FLEXFIELDS_B`, `FND_DF_CONTEXTS_B`, `FND_DF_SEGMENTS_B` (+ `_TL`) | Which flexfield, which contexts, which segment maps to which column with which value set |
| KFF definitions | `FND_KF_*` tables | Structures, segments, structure instances |

You'll also see `ENTERPRISE_ID` and `SANDBOX_ID` on these tables — Fusion keeps sandbox versions of setup separate from the live one.

### Transaction data — where the user's input lands

- **DFF:** in the business object's own table. Location DFF → `PER_LOCATION_DETAILS_F.ATTRIBUTE1`, `ATTRIBUTE2`… and the context code in `ATTRIBUTE_CATEGORY`.
- **EFF:** in a separate extra-info table, one row per context row (that's how multi-row works).
- **KFF:** combination stored once in a combinations table; the record stores only the combination's ID.

**What gets stored — code, not description.** For Independent/Dependent sets, the column holds the **value code** (`IN`, `CHENNAI`). For a Table value set with an ID column, it holds the **ID** (`PERSON_ID`). Descriptions are looked up when the page or report shows them. That's why renaming a description changes it everywhere instantly, while changing a stored code would orphan old records.

**Date-effective tables carry DFFs with them.** `PER_LOCATION_DETAILS_F` is date-tracked, so a DFF value changed with *Update* creates a new dated row; with *Correct* it overwrites the current row. Interviewers love asking how to see a location's DFF value "as of last year" — filter on the effective dates.

### What "Deploy Flexfield" actually does

Saving only stores your definition (the metadata above). **Deploy** turns it into something the application can use: Fusion regenerates the flexfield's business-layer objects and UI definitions so pages, REST APIs and integrations know about the new attributes. Status moves from *Edited* to *Deployed* (or *Error*, with a log). You re-login because your session cached the old definition.

After deploying, a few other things don't update by themselves:

- **OTBI reports:** BI-enabled segments reach the subject areas only after the ESS job *Import Oracle Fusion Data Extensions for Transactional Business Intelligence* runs.
- **HDL loads:** refresh the business object in *Data Exchange → View Business Objects* so the new DFF attributes appear in the HDL template.
- **Other environments:** the change exists only in this pod. Move it to TEST/PROD with a configuration package (Functional Setup Manager export/import) or redo it — then **deploy there too**.

### Reading DFF data in BI Publisher

```sql
SELECT pld.location_id
     , pld.attribute_category  AS location_type   -- context code
     , pld.attribute1          AS country_code    -- global segment
     , pld.attribute2          AS city_code
     , pld.attribute3          AS attr3_raw       -- meaning depends on attribute_category!
FROM   per_location_details_f pld
WHERE  TRUNC(SYSDATE) BETWEEN pld.effective_start_date AND pld.effective_end_date
```

Note `attr3_raw`: in a report you must `CASE` on `attribute_category` before labelling `ATTRIBUTE3`, because it means Seating Capacity for OFFICE and Safety Officer for FACTORY. To show descriptions instead of codes, join to the value set values (many BIP developers use the `FND_FLEX_VALUES_VL` / `FND_FLEX_VALUE_SETS` views — confirm the names in your pod).

## 12. Edge cases you'll actually hit on a project

Most flexfield problems come from **changing something after data already exists** or **forgetting a step after deploy**. Read this table twice; nearly every row is also an interview question.

| Situation | What goes wrong | How to handle it |
| --- | --- | --- |
| Fields don't show after setup | You saved but didn't deploy, or deployed but didn't re-login | Deploy, check status = Deployed, sign out and in |
| Deployment fails | Usually a broken table value set (bad SQL, wrong column type) or a column clash | Open the deployment log, fix the value set, redeploy |
| Made a segment Required after go-live | Every old record now fails validation the next time someone edits it | Backfill old records via HDL first, or default a value, then make it required |
| Value no longer valid (city closed) | Deleting it breaks old records that store it | Disable it or set an End Date; old records keep the code |
| Description wrong | — | Just edit the description; it's looked up live, no data change needed |
| Need to change a stored value code | Old records still hold the old code and stop matching | Avoid. If unavoidable: add new value, migrate data via HDL, then disable old |
| Picked Number but codes have leading zeros | `007` saved as `7` | Use Character + Numeric digits only (+ Zero fill) |
| Min/Max range on a Character set behaves oddly | Text compares alphabetically (`9` > `100`) | Use Number data type for numeric ranges |
| Need translations later, chose Text | Can't switch to Translated text after creation | Choose Translated text up front for global clients |
| City empty after selecting Country | Country segment sequence is after City, or dependent set points to the wrong independent set | Country sequence lower; check the Independent Value Set field |
| Table value set shows a person twice | Date-effective rows or multiple name types | Filter `NAME_TYPE` and effective dates in WHERE |
| Table value set shows everyone in the company | Table value sets don't apply HCM data security | Filter in WHERE; avoid sensitive tables |
| Deleted a segment, mapped a new one to the same column | Old data reappears under the new label | Clear data first, or use an unused column |
| Ran out of `ATTRIBUTE` columns | DFF has a fixed number per data type | Reuse columns across contexts, or move to an EFF |
| Field missing in OTBI | BI Enabled not ticked, or ESS import job not run | Tick BI Enabled, redeploy, run the TBI extensions import job |
| Field missing in HDL template | Business object not refreshed | Data Exchange → View Business Objects → refresh |
| Works in DEV, missing in TEST | Setup is per pod | Migrate via configuration package and deploy in the target pod |
| Context value too long | Context codes must fit the context column (30 on Location) | Keep context codes short |
| EFF deployed but nothing shows | Context not associated with the category, not added to a page, or no usage | Category → Associated Contexts; add to a page; check usages; redeploy |
| Built a context as Single Row, client now needs a list | Shape can't be converted once data exists | Create a new Multiple Rows context, migrate data via HDL, retire the old one |
| Legislative EFF context missing on a location | Location's country/legislation doesn't match the context | Test on a location of the right country |
| BIP report shows each location several times | Joined a multi-row EFF table directly | Aggregate, pick one row (e.g. latest), or report the EFF as its own detail block |
| EFF section visible but read-only for some users | Context usage privileges restrict view/edit | Check view/edit privileges on the context usage |

## 13. Interview questions — with answers

Answer in three beats: **what it is → why it exists → one real example.** Cover each answer before reading it.

### Concepts

**Q1. What is a value set and why not just use a free-text field?**
A rule set that controls what can be entered in a field. Free text lets users type `Chennai`, `chennai`, `Chenai` — three spellings, broken reports. A value set keeps data clean and reportable.

**Q2. Independent vs Dependent vs Subset?**
Independent = a fixed list. Dependent = a list filtered by what was chosen in another field (Country → City). Subset = a fixed slice of an existing independent list, no prior selection needed (weekdays → weekend).

**Q3. When would you choose a Table value set over an Independent one?**
When the data already lives in Fusion (people, departments, jobs). An independent copy goes stale the day a new department is created; a table value set is always current.

**Q4. DFF vs EFF vs KFF in one line each?**
DFF: a few extra fields stored on the same row. EFF: extra sections, single or multi-row, stored in a separate table. KFF: a multi-part identifying code (Job, Grade) stored once as a combination.

**Q5. What is a context segment?**
The switch field of a DFF. Its value decides which context-sensitive segments show. Its value is stored in `ATTRIBUTE_CATEGORY`.

**Q6. Global vs context-sensitive segment?**
Global shows on every record. Context-sensitive shows only when its context is selected.

### Tricky ones

**Q7. Why can't you create a new DFF in Fusion SaaS?**
Creating one means adding table columns and changing application code — impossible in a shared SaaS codebase. Oracle pre-registers DFFs with spare `ATTRIBUTE` columns; implementers only configure segments and contexts and deploy.

**Q8. Can two segments use the same ATTRIBUTE column?**
Yes, if they're in **different contexts** — a row only has one context, so no clash. No for two global segments, or a global and a context-sensitive one.

**Q9. What can be used as a value set on a context segment?**
Only Independent or Table, with Character data type, and values must fit the context column's length.

**Q10. Can a Dependent value set's parent be another Dependent value set?**
No. The parent must be an Independent value set.

**Q11. You disabled a value. What happens to employees who already have it?**
Nothing — their record keeps the stored code and still shows it. They just can't pick it on new entries. If someone edits that record, validation may ask for a valid value.

**Q12. Enabled flag vs End Date?**
Enabled is an on/off switch now. End Date retires a value from a specific date, so you can plan changes in advance.

**Q13. Number or Character for a 6-digit PIN code?**
Character with Numeric digits only. Number drops leading zeros and you never do maths on it.

**Q14. Why do some people say Table value sets are a security risk?**
They run SQL directly and don't apply the user's HCM data security, so a list can expose every person in the company. Filter in the WHERE clause.

### Internals

**Q15. Where is a Location DFF value physically stored?**
In `PER_LOCATION_DETAILS_F`, columns `ATTRIBUTE1…n` (plus number/date variants); context in `ATTRIBUTE_CATEGORY`.

**Q16. Where are value set values stored?**
Header in `FND_VS_VALUE_SETS`, values in `FND_VS_VALUES_B`, descriptions per language in `FND_VS_VALUES_TL`.

**Q17. Does the DFF column store the value or its description?**
The value code (or the ID for a table value set with an ID column). Descriptions are looked up when shown.

**Q18. What does Deploy do?**
Turns the saved definition into runtime objects so pages, APIs and integrations know the new attributes. Without deploy, the setup exists but nobody sees it.

**Q19. A DFF segment isn't available in OTBI. Why?**
BI Enabled wasn't ticked, or the *Import Oracle Fusion Data Extensions for Transactional Business Intelligence* job hasn't run since deploy.

### Scenarios

**Q20. Client wants India locations to capture GST number and US locations to capture County. Design it.**
Location DFF `PER_LOCATIONS_DF`. Context segment based on country (user-selected or defaulted). Context `IN` with segment GST Number; context `US` with segment County. Both can reuse the same `ATTRIBUTE` column. Deploy, re-login, test.

**Q21. Client wants a "Buddy" field on the person record listing current employees by name.**
Person DFF segment with a Table value set on the names table: Value = full name, ID = person ID, WHERE filters `NAME_TYPE = 'GLOBAL'` and current dates; join to assignment data to exclude terminated workers. Store the ID, not the name.

**Q22. After go-live, HR wants an existing optional segment made mandatory. What do you check?**
How many existing records are blank. Backfill them (HDL or a default) before making it required, otherwise every edit of an old record will fail.

**Q23. A user says the City list is empty for India.**
Check: the dependent value set's Independent Value Set points to the right country set; the India value exists, is enabled and in date; cities exist under India and are enabled; Country segment sequence is before City; flexfield redeployed after changes.

**Q24. Client wants to track multiple certifications per employee, each with name, number and expiry.**
Not a DFF — a DFF holds one value per field per record. Use a Person EFF with a multi-row context.

**Q25. Setup works in DEV but not in TEST. Why?**
Configuration is per environment. Migrate it (configuration package via Functional Setup Manager) and deploy the flexfield in TEST.

### EFF-specific

**Q26. What's the difference between a single-row and a multi-row EFF context?**
Single row stores one set of values per record and shows as a form. Multi row stores a list and shows as a table with Add/Delete — e.g. one row per fire safety certificate.

**Q27. Where is Location Extra Information stored, and how is it linked to the location?**
In `PER_LOCATION_EXTRA_INFO_F`, one row per context row, linked by `LOCATION_ID`; the context is in `INFORMATION_TYPE`. Legislative Information goes to `PER_LOCATION_LEG_F`.

**Q28. You created and deployed an EFF context but users can't see it. What did you miss?**
Usually the category association (`HCM_LOC` → Associated Contexts) or adding it to a page. Then check usages/privileges, redeploy and re-login.

**Q29. Extra Information vs Legislative Information on a location?**
Extra Information holds company-specific data for any location. Legislative Information holds country-specific/statutory data; its contexts appear only for the matching legislation.

**Q30. Can an EFF have global segments like a DFF?**
No. In an EFF every segment belongs to a context. Many contexts can show at once, unlike a DFF where one context is active per record.

## 14. One-page revision cheat sheet

- **Value set** = the rule for a field. **Flexfield** = the field(s). **Segment** = one field. **Deploy** = make it live. **Re-login** = see it.
- **5 types you create:** Format Only (shape rules), Independent (fixed list), Dependent (list filtered by a parent field), Subset (slice of an independent list), Table (live from a Fusion table).
- **Create order:** independent → dependent/subset → values → segments → deploy.
- **Character vs Number:** maths or numeric range → Number; codes, IDs, PINs → Character.
- **Uppercase only** = forces capitals. **Zero fill** = pads digits with leading zeros to max length.
- **Translated text** must be chosen at creation; can't switch later.
- **Disable / end-date, don't delete** values that are in use.
- **Dependent in a DFF:** parent segment sequence lower than child.
- **Table value set:** filter date-effective rows and name type; store the ID; no data security applied.
- **DFF** = same table, `ATTRIBUTE*` columns. **EFF** = separate table, multi-row possible. **KFF** = combination code, stored once.
- **Context** = switch; stored in `ATTRIBUTE_CATEGORY`; context-sensitive segments in different contexts can **share** a column.
- **Context value set:** only Independent or Table, Character.
- **Can't create DFFs in SaaS** — Oracle registers, we configure.
- **Location DFF:** `PER_LOCATIONS_DF` on `PER_LOCATION_DETAILS_F`.
- **Location EFFs:** Extra Information `PER_LOCATION_INFORMATION_EFF` (category `HCM_LOC`, table `PER_LOCATION_EXTRA_INFO_F`); Legislative Information `PER_LOCATION_LEG_EFF` (category `HCM_LOC_LEG`, table `PER_LOCATION_LEG_F`). Linked by `LOCATION_ID`.
- **EFF single row** = form, one row. **Multiple rows** = table, a list. Choose up front; can't convert once data exists.
- **EFF checklist:** context → segments → associate with category → add to page → usage → deploy → re-login.
- **Value set tables:** `FND_VS_VALUE_SETS`, `FND_VS_VALUES_B`, `FND_VS_VALUES_TL`.
- **After deploy:** OTBI needs BI Enabled + the TBI extensions import job; HDL needs the business object refreshed; other pods need migration + deploy.