# HCM Extracts — Study Guide

*As of Nov 2, 2026*

## 1. What HCM Extracts are

HCM Extracts is Oracle Fusion HCM's built-in tool for pulling HCM data **out** of Fusion in bulk, shaping it, and delivering it to another system or person. Think of it as the outbound twin of HCM Data Loader (HDL): HDL brings data in, HCM Extracts sends data out.

**Typical real-world uses**

- Sending employee data to a third-party payroll vendor (ADP, a local payroll provider)
- Sending eligible employees to a benefits carrier or insurance company
- Feeding identity/access systems (Active Directory, Okta) with new hires and terminations
- Sending a nightly "changes only" file to a downstream HR data warehouse
- Bulk data for audits, government reporting, migrations

**Where you find it:** My Client Groups → Data Exchange → HCM Extracts → Extract Definitions (older releases: Navigator → Data Exchange → Manage Extract Definitions).

### How it compares to the other Fusion reporting tools

| Tool | Direction | Best for | Data access | Volume |
| --- | --- | --- | --- | --- |
| HCM Extracts | Outbound | Scheduled integrations to vendors, incremental (changes-only) files | User Entities + Database Items, Fast Formula logic | Very high, multi-threaded |
| BI Publisher (BIP) | Outbound / reporting | Pixel-perfect reports, custom SQL data models | Direct SQL on Fusion tables | Medium to high |
| OTBI | Reporting | Ad-hoc self-service analysis by business users | Subject areas (no SQL) | Low to medium |
| HDL / HSDL | Inbound | Loading and bulk-updating HCM objects | .dat files / spreadsheets | High |

**The one line to say in an interview:** use HCM Extracts when you need a *scheduled, secured, high-volume, effective-dated outbound feed* that can detect changes since the last run; use BIP when you need custom SQL and a formatted report.

**Key vocabulary you will hear in every discussion**

- **Extract Definition** — the whole design (like a "report definition")
- **Data Group** (older name: **Block**) — one level of the hierarchy, e.g., Worker, Assignment
- **User Entity (UE)** — Oracle-delivered SQL that a data group reads from
- **Database Item (DBI)** — a single field exposed by a UE, e.g., `PER_EXT_PER_PERSON_NUMBER`
- **Record** — a set of fields (output row) under a data group
- **Data Element** — one field/column of the output
- **Fast Formula** — Oracle's mini-language for custom logic/filters
- **Delivery Option** — output format + where the file goes (BI Publisher template, SFTP, UCM, email)

## 2. Architecture: the building blocks

An extract definition is a tree: parameters at the top, one **root data group**, child data groups connected to it, records inside each group, attributes inside each record, and delivery options at the end. Learn this tree and every other question becomes easy.

```
Extract Definition  (XX_PAYROLL_VENDOR_EXTRACT)
├── Parameters            Effective Date, Start Date, Changes Only, custom
├── Root data group: Worker           [PER_EXT_SEC_PERSON_UE, threading DBI: Person ID]
│   ├── Record: Header                String: company code, Parameter: effective date
│   └── Child data group: Assignment  [PER_EXT_SEC_ASSIGNMENT_UE]
│       │   Link: Person ID = Assignment Person ID
│       │   Criteria: Active, Primary = Y
│       └── Record: Employee Details
│             Person Number  · Database Item
│             Vendor Status  · Fast Formula
│             Company Code   · String
└── Delivery options      BIP template (RTF / eText) → SFTP · UCM · Email
```

Everything hangs off the root data group: child groups join it through connections, records sit inside groups, and attributes inside records become the output columns.

### 2.1 Extract Definition

The complete setup: data groups, criteria, records, attributes, advanced conditions and delivery options ([Oracle: Extract Components](https://docs.oracle.com/en/cloud/saas/human-resources/25b/fahex/extract-components.html)).

- **Name** — also becomes the name of the flow pattern used to run it
- **Type** — e.g., *HR Archive*, *Full Profile*, *Payroll Interface*. Type decides which parameters come seeded. For absence extracts Oracle tells you to pick *HR Archive* and display the *Start Date* parameter ([Oracle: HCM Extracts for Absence Data](https://docs.oracle.com/en/cloud/saas/human-resources/faiam/hcm-extracts-for-absence-data.html))
- **Parameters** — *Effective Date* is always there; you add *Start Date*, *End Date*, *Changes Only*, or your own (e.g., Department). Mark *Display = Yes* to let the user enter it at run time

### 2.2 Data Groups (old name: Blocks)

A data group represents one business entity — person, assignment, department, absence, benefits enrollment.

- Each data group sits on exactly one **User Entity (UE)**. The UE is Oracle's prebuilt SQL; you never write that SQL yourself
- Exactly **one root data group** per extract — the starting point of execution
- The root group has a **Threading Database Item** (e.g., Person ID, Assignment ID). The engine splits work into threads by this key, and Changes Only compares data at this level
- Child groups link to their parent through a **Data Group Connection**: a parent DBI = a child DBI. Oracle's department example connects *Extract Organization ID* (parent) to *Extract Assignment Department ID* (child) ([Oracle: Create Data Groups](https://docs.oracle.com/en/cloud/saas/human-resources/fahex/create-data-groups-and-extract-data-group-connections.html))

**User Entities you should recognise**

| User Entity | What it returns | Typical use |
| --- | --- | --- |
| PER_EXT_SEC_PERSON_UE | Person / worker | Root of most Core HR extracts |
| PER_EXT_SEC_ASSIGNMENT_UE | Assignments with job, grade, department, location | Child of Person |
| PER_EXT_SEC_ASSIGNMENT_BASIC_UE | Lighter assignment UE | Employees under a department |
| PER_EXT_SEC_ORGANIZATION_UE | Departments / organizations | Department-rooted extract |
| ABS_EXT_ABSENCE_ENTRY_UE | Absence entries | Absence feeds |
| BEN_EXT_ENRT_RSLT_UE | Benefit enrollment results | Benefit carrier feeds |

`SEC` in the name means **secured**: the UE applies the data security of the user running the extract.

### 2.3 Data Group Criteria (filters)

Criteria restrict what each group returns. Two ways to write them ([Oracle: Create Data Groups](https://docs.oracle.com/en/cloud/saas/human-resources/fahex/create-data-groups-and-extract-data-group-connections.html)):

1. **Expression / advanced condition** — e.g., `Assignment Status = 'ACTIVE'`. The engine appends it to the UE's SQL, so it filters in the database. **Preferred** for performance
2. **Fast Formula** (type *Extract Criteria*) returning Y or N per row — use only when the logic can't be an expression

If you give both, **both** are applied.

### 2.4 Records

A record is a group of related fields inside a data group — e.g., an Employee group with records *Basic Details*, *Pay Details*, *Location Details*, *Primary Contact*. Each has a **sequence** that sets processing order ([Oracle: Extract Components](https://docs.oracle.com/en/cloud/saas/human-resources/25b/fahex/extract-components.html)).

### 2.5 Attributes (Data Elements)

One attribute = one field in the output, with an XML tag name. Attribute types:

| Type | What it is | Example |
| --- | --- | --- |
| Database Item Group | A DBI from the group's UE | Person Number, Hire Date |
| Fast Formula (Rule) | Value calculated by an *Extract Rule* formula | Status code 'A'/'I' for the vendor |
| String | A fixed constant | `'ACME'` company code |
| Parameter | Value of a run-time parameter | Effective Date printed on the header |
| Record Calculation / Expression | Concatenation, decode, arithmetic over other attributes | First Name + ' ' + Last Name |
| Group function | SUM, COUNT, MIN, MAX over another record | Total employees in a trailer row |

### 2.6 Delivery Options

Where and how output leaves: BI Publisher report + template, output format (CSV, XML, Excel, HTML, RTF, PDF, eText), delivery type (SFTP, email, WebCenter Content/UCM, Document of Record…), output file name. Covered in section 6.

## 3. Building and running an extract, step by step

Worked example: *send every active employee's Person Number, Name, Department and Hire Date to a payroll vendor as a CSV over SFTP, every night.*

### 3.1 Design time

1. **Create the definition** — Extract Definitions → **+** → Name `XX_PAYROLL_VENDOR_EXTRACT`, Type *HR Archive* (or *Payroll Interface*), Start Date `01-01-1951` (an early date so it's valid for any effective date), Legislative Data Group if needed
2. **Parameters** — keep *Effective Date*; add *Changes Only* if the vendor wants deltas (section 4)
3. **Root data group** — Name `Worker`, UE `PER_EXT_SEC_PERSON_UE`, Root = Yes, Threading DBI = Person ID
4. **Child data group** — Name `Assignment`, UE `PER_EXT_SEC_ASSIGNMENT_UE`; connect parent *Person ID* = child *Assignment Person ID*
5. **Criteria** — on Assignment: assignment status = Active, Primary flag = Y, Assignment type = Employee
6. **Records and attributes** — record `Employee Details` under Assignment with Person Number, Full Name, Department Name, Hire Date (DBI type); add a `Header` record at root with a String/Parameter and a `Trailer` with a COUNT if the vendor wants control totals
7. **Delivery option** — point to the BI Publisher report/template, Output = CSV (or eText), Delivery = SFTP, Output Name e.g. `ACME_EMP_` + date
8. **Validate and compile** — open the **Extract Execution Tree** and run **Compile All Formula**; the definition must show as *Valid*. Skip this and the run errors out

### 3.2 Run time

HCM Extracts run as **payroll flows**. When you save a definition Fusion creates a flow pattern with the same name.

1. **Submit Extracts** (Data Exchange) → pick the extract → give a **Flow Instance Name** (unique per run, e.g. `PAYVENDOR_02NOV2026`) → enter parameters → Submit
2. The flow runs its tasks: **Extract** (engine pulls and archives data) → **Report/Deliver** (BI Publisher formats it, delivery sends it)
3. **View Extract Results** → search the instance → *Extract Delivery Options* tab to download the file ([Oracle: View the Results](https://docs.oracle.com/en/cloud/saas/human-resources/fahex/view-the-results-of-an-hcm-data-loader-extract.html))
4. **Schedule** it (daily, weekly) from the same submit page
5. **Refine Extracts** lets you edit the flow pattern itself — e.g., add a task after the extract such as *Initiate HCM Data Loader* (an extract that produces an HDL file and loads it)

Other ways to trigger it: the **ExtractActions** web service (submit + fetch archived XML; Oracle recommends it only for small extracts, [Oracle: Extract Actions](https://docs.oracle.com/en/cloud/saas/human-resources/oeswh/extractactions-d19859e993.html)), the FlowActionsService, or an **OIC** integration that submits the flow and picks up the file from UCM. OIC + UCM is the most common pattern on real projects.

### 3.3 What the engine does internally

1. Reads parameters, resolves the root UE's SQL with your criteria appended
2. Splits root objects (e.g., Person IDs) into chunks; **multiple threads** process chunks in parallel
3. For each root row, walks the child data groups through the connections
4. Evaluates attributes (DBIs, formulas) and **archives** the results
5. Builds one XML document from the archive (tags = your XML tag names)
6. Hands the XML to BI Publisher, which applies the template and delivers

## 4. Changes Only (incremental) extracts

Changes Only makes the extract send just what changed since the last successful run. It is the single most-asked HCM Extracts topic in interviews.

### 4.1 How it works

Every run is **archived**. In Changes Only mode, the engine builds the current snapshot, compares it with the **baseline** (the archived data of previous runs), and outputs only new or modified data ([Oracle: Guidelines for Using Different Modes](https://docs.oracle.com/en/cloud/saas/human-resources/fahex/guidelines-for-using-different-modes-for-hcm-extracts.html)). It is a **snapshot comparison**, not a read of an audit table.

**Setup checklist**

1. Add a parameter whose base name is **`CHANGES_ONLY`** (don't rename it — the engine looks for that exact parameter)
2. Link it to the lookup **`PER_EXT_CHANGES_ONLY`** or **`ORA_HRY_CHANGES_ONLY`** so the user picks a mode at submit time
3. Set the **threading database item** on the data groups — comparison happens at the threading level
4. Mark unique attributes (Person ID, Assignment ID, Person Number) as **Key** so the engine can match rows between runs
5. Optionally mark attributes as **always include** (e.g., Person Number must appear even if unchanged) or **exclude from comparison** (e.g., a run timestamp that changes every time)

### 4.2 The modes

| Mode (lookup code) | Meaning | What you get |
| --- | --- | --- |
| N | All attributes | Full extract; also archived as the baseline |
| Y | Changed attributes | Only data that changed against the baseline |
| ATTRIBUTE | Changed and marked attributes | Changed elements + elements marked as required |
| ATTRIB_OLD | Changed and marked attributes with previous values | As above + the **old value** next to each changed one |
| BLOCK | Changed and marked attributes under threading group | Changed + required data of the threading group + parent group values |
| BLOCK_OLD | Same as BLOCK + previous values | Changed, required, previous, parent values |

Source: [Oracle: Guidelines for Using Different Modes](https://docs.oracle.com/en/cloud/saas/human-resources/fahex/guidelines-for-using-different-modes-for-hcm-extracts.html). Practical pick: **ATTRIB_OLD** when the vendor needs "old → new" (e.g., salary change from 50,000 to 55,000); **BLOCK** when the vendor needs the whole assignment row whenever anything in it changes.

### 4.3 Rules that catch people out

- **First run = full.** No baseline exists yet, so everything is "new"
- **Editing the definition resets the baseline.** Oracle states any change to the extract definition needs a new baseline, so the next run extracts everything ([Oracle](https://docs.oracle.com/en/cloud/saas/human-resources/fahex/guidelines-for-using-different-modes-for-hcm-extracts.html)). Warn the vendor before you move a change to production
- **Rolling back a run** removes it from the baseline — the standard way to resend a delta that the vendor lost
- **A change that doesn't alter the snapshot isn't sent.** A back-dated correction that only changes history (not the value as of the run's effective date) produces no difference in the snapshot
- **Records that drop out of your criteria** (e.g., a terminated person filtered out) don't announce themselves; design an explicit termination record or date-range criteria
- **Debugging:** run *Extracts Process Diagnostics Report* with scope *Changes Only Diagnostics*; mode *Previous Extract Run* is for `CHANGES_ONLY = Y`, *Object Extracted But Not Expected* for the advanced modes ([Oracle 24B What's New](https://docs.oracle.com/en/cloud/saas/readiness/hcm/24b/hcom-24b/24B-hcm-common-wn-f31820.htm))

## 5. Fast Formulas in extracts

Fast Formula is how you add logic the UE doesn't give you. You can't write SQL inside an extract, so formulas (plus value sets) are your escape hatch.

| Formula type | Attached at | Returns | Use it for |
| --- | --- | --- | --- |
| Extract Criteria | Data group criteria | Y / N | Row filters too complex for an expression |
| Extract Rule | Attribute of type Fast Formula | A value | Derived fields: codes, formatted values, look-ups |
| Extract Record | Record level | Values for the record | Record-level calculations |

**Example — Extract Rule: vendor status code**

```
/* Formula type: Extract Rule */
DEFAULT FOR PER_ASG_ASSIGNMENT_STATUS_TYPE IS 'X'

l_status = PER_ASG_ASSIGNMENT_STATUS_TYPE

IF l_status = 'ACTIVE_PROCESS' THEN
  rule_value = 'A'
ELSE
  rule_value = 'I'

RETURN rule_value
```

**Example — calling SQL through a value set**

```
l_person_id = GET_CONTEXT(PERSON_ID, 0)
l_union = GET_VALUE_SET('XX_PERSON_UNION_VS',
            '|=P_PERSON_ID=''' || TO_CHAR(l_person_id) || '''')
```

`GET_VALUE_SET` runs the SQL of a table-validated value set and returns one value. This is the standard answer to "how do you get data the UE doesn't expose?"

**Formula habits interviewers check**

- Always give a **DEFAULT** for every DBI — a DBI with no value and no default fails the formula, which fails that person's row
- Use **contexts** (`PERSON_ID`, `HR_ASSIGNMENT_ID`, `EFFECTIVE_DATE`) through `GET_CONTEXT`; the engine sets them for the current row
- Prefer an **expression criterion** over a criteria formula: expressions are pushed into the SQL, formulas run row by row
- After editing a formula used by an extract, **recompile** (Extract Execution Tree → Compile All Formula)
- Status names like `ACTIVE_PROCESS` are system statuses; check the values in your pod before hard-coding them

## 6. Delivery and BI Publisher templates

The extract engine only produces **XML**. Everything the vendor sees — CSV, fixed-width text, PDF, Excel — is BI Publisher formatting that XML.

### 6.1 How the BIP side is built

1. Run the extract once with no delivery (or XML output) and download the XML
2. In BI Publisher create a **report** under e.g. `/Custom/HCM/Extracts/`, using that XML as the sample data
3. Add a **template**: RTF (Word layout for PDF/Excel), **eText** (fixed-width or delimited text files — what most payroll and bank vendors want), or Excel
4. In the extract's Delivery Options, set the **report path** and **template name** to that report

### 6.2 Delivery option fields

| Field | What to set |
| --- | --- |
| Output Type | CSV, XML, Excel, HTML, RTF, PDF, eText |
| Report / Template | BIP report path + template name |
| Delivery Type | None, Email, FTP/SFTP, WebCenter Content (UCM), Printer, Document of Record |
| Output Name | File name, often with a date expression |
| Bursting Node | XML node to split on — one file/email per employee (e.g., payslips) |
| Encryption / compression | PGP and ZIP, typically for SFTP to vendors |

**Two extracts, one file?** Not supported directly; the usual answer is one extract with multiple data groups/records, or OIC merging files.

**Multiple delivery options on one extract** are allowed — e.g., CSV to the vendor by SFTP and an Excel copy to HR by email from the same run.

**WebCenter Content (UCM)** is the most common target when OIC picks up the file: the extract drops it in a UCM account, OIC downloads and forwards it.

## 7. Behind the scenes: tables and security

The definition lives in `PER_EXT_*` tables; data groups and records reuse the payroll reporting tables (`PAY_REPORT_*`); every run is a payroll flow in `PAY_FLOW_*`. That mix — PER for design, PAY for run — is itself a good interview answer.

### 7.1 Design-time tables (the definition)

| Table | Holds | Key columns |
| --- | --- | --- |
| [PER_EXT_DEFINITIONS_B](https://docs.oracle.com/en/cloud/saas/human-resources/25a/oedmh/perextdefinitionsb-26594.html) / _TL / _VL | Extract definitions | EXT_DEFINITION_ID |
| [PER_EXT_PARAMETERS_B](https://docs.oracle.com/en/cloud/saas/human-resources/25a/oedmh/perextparametersb-30967.html) | Parameters of a definition | EXT_PARAM_ID, EXT_DEFINITION_ID, XML_TAG_NAME |
| [PAY_REPORT_DEFINITIONS](https://docs.oracle.com/en/cloud/saas/human-resources/25a/oedmh/payreportdefinitions-20820.html) | Links a definition to its root block | EXT_DEFINITION_ID, REPORT_BLOCK_ID |
| PAY_REPORT_BLOCKS | Data groups (blocks) and their user entity | REPORT_BLOCK_ID |
| [PER_EXT_BLOCK_LINKS](https://docs.oracle.com/en/cloud/saas/human-resources/24b/oedmh/perextblocklinks-18841.html) | Parent–child data group connections | REPORT_BLOCK_ID, PARENT_BLOCK_ID |
| [PAY_REPORT_RECORDS_F](https://docs.oracle.com/en/cloud/saas/human-resources/25a/oedmh/payreportrecordsf-25005.html) | Records (date-effective) | REPORT_RECORD_ID, FORMULA_ID |
| [PER_EXT_DATA_ELEMENTS_B](https://docs.oracle.com/en/cloud/saas/human-resources/25a/oedmh/perextdataelementsb-7592.html) / _TL / [_VL](https://docs.oracle.com/en/cloud/saas/human-resources/oedmh/perextdataelementsvl-3699.html) | Attributes | EXT_DATA_ELEMENT_ID, REPORT_RECORD_ID, TYPE_CODE, DBI_GROUP_ID, RULE_ID, EXT_PARAM_ID, STRING_VALUE |
| [PER_EXT_DELIVERY_OPTIONS_B](https://docs.oracle.com/en/cloud/saas/human-resources/25b/oedmh/perextdeliveryoptionsb-3182.html) | Delivery options | EXT_DELIVERY_OPTION_ID |
| [PER_EXT_ACTIONS](https://docs.oracle.com/en/cloud/saas/human-resources/25a/oedmh/perextactions-23942.html) | Actions on records / data elements | REPORT_RECORD_ID, EXT_DATA_ELEMENT_ID, ACTION_CODE |
| [PER_EXT_PROPERTIES](https://docs.oracle.com/en/cloud/saas/human-resources/oedmh/perextproperties-8044.html) | Extra properties of a definition | EXT_DEFINITION_ID, PROPERTY_CODE, PROPERTY_VALUE |

Read the data-element row like this: `TYPE_CODE` says what kind of attribute it is, and exactly one of `DBI_GROUP_ID` (DBI), `RULE_ID` (fast formula), `EXT_PARAM_ID` (parameter) or `STRING_VALUE` (constant) is filled. Group attributes use `GROUP_FUNCTION_CODE` (SUM, COUNT, MIN…) and expressions use `FUNCTION_EXPRESSION`.

### 7.2 Run-time tables (every execution)

| Table | Holds |
| --- | --- |
| PAY_FLOW_INSTANCES | One row per submitted run (INSTANCE_NAME, FLOW_INSTANCE_ID, BASE_FLOW_ID) |
| PAY_REQUESTS | The request behind a flow instance (PAY_REQUEST_ID, FLOW_INSTANCE_ID) |
| PAY_PAYROLL_ACTIONS | The process action (PAYROLL_ACTION_ID, PAY_REQUEST_ID) |
| PAY_FLOW_PARAMETERS / PAY_FLOW_PARAM_VALUES | Parameter definitions and the values entered for a run |

**Query — the parameter values a given extract run used** (adapted from [Doyensys](https://doyensys.com/blogs/query-to-get-hcm-extract-input-values-in-bi-report/); verify column names in your pod):

```sql
SELECT flow.instance_name,
       par.base_flow_parameter_name,
       val.flow_param_value
FROM   pay_flow_instances   flow,
       pay_flow_parameters  par,
       pay_flow_param_values val
WHERE  flow.instance_name     = :p_instance_name
AND    par.base_flow_id       = flow.base_flow_id
AND    val.flow_instance_id   = flow.flow_instance_id
AND    val.base_flow_parameter_id = par.base_flow_parameter_id;
```

The chain `PAY_PAYROLL_ACTIONS.PAY_REQUEST_ID → PAY_REQUESTS.FLOW_INSTANCE_ID → PAY_FLOW_INSTANCES` is how a BIP template finds which run (and which parameters) produced its XML.

### 7.3 Security

- **Secured UEs** (`_SEC_` in the name) apply the data security — person and organization security profiles — of the **user who submits** the extract. Same extract, two users, different row counts: that's why
- Scheduled integration extracts usually run under a dedicated integration user with view-all security
- Running extracts is a payroll flow action: the ExtractActions service maps to the privilege *Submit Payroll Flow* (`PAY_SUBMIT_PAYROLL_FLOW_PRIV`) ([Oracle: Extract Actions](https://docs.oracle.com/en/cloud/saas/human-resources/oeswh/extractactions-d19859e993.html))
- Defining extracts needs the extract-definition privileges, typically through the *Human Capital Management Integration Specialist* job role

## 8. Edge cases and gotchas

Most extract bugs come from three things: the **effective date**, **multiple rows per person**, and **security**. Check those first.

| # | Scenario | What goes wrong | How to handle it |
| --- | --- | --- | --- |
| 1 | Future-dated hire (hire date after the run's effective date) | Person missing from file | Data is read *as of* Effective Date; add Start/End Date range criteria or run with a later date |
| 2 | Terminations in the period | Terminated people silently vanish (filtered by active status) | Criteria like "active OR terminated between Start Date and End Date"; a separate termination record |
| 3 | Person with multiple assignments | Duplicate rows for the vendor | Filter Primary flag = Y, or design one row per assignment on purpose |
| 4 | Multiple phones / emails / addresses | Several rows from one DBI group | Filter by type (Work email, Home address) and primary flag |
| 5 | Rehire or global transfer | New work relationship and assignment; vendor sees a "new" employee | Key the vendor file on Person Number, not Assignment Number |
| 6 | Back-dated correction | Changes Only doesn't send it | Snapshot as of the run date didn't change; plan a full run or date-range logic |
| 7 | Definition edited in production | Next Changes Only run sends the full population | Baseline reset — tell the vendor, or migrate the change with a planned full file |
| 8 | Two users run the same extract | Different counts | Secured UEs apply the submitter's security profile |
| 9 | DBI is null | Formula error, row or run fails | DEFAULT FOR every DBI in formulas; default values on attributes |
| 10 | Leading zeros, dates, commas in names | Vendor file rejected | Format in the BIP/eText template; quote CSV fields; fixed date masks |
| 11 | Very large population | Slow runs, timeouts | Expression criteria (pushed to SQL), correct threading DBI, avoid row-by-row formulas |
| 12 | Extract pulled through the ExtractActions SOAP service | Large XML fails | Oracle recommends the service only for small extracts; use flow + UCM + OIC for volume |
| 13 | Formula changed but output didn't | Old compiled formula still used | Compile All Formula in the Extract Execution Tree |
| 14 | Flow instance name reused | Submit rejected | Instance name must be unique — add a date/time to it |
| 15 | Vendor lost yesterday's delta | Need to resend without a full file | Roll back yesterday's run, then rerun it (baseline goes back one step) |

## 9. Tricky interview questions with answers

Answer in this order every time: **what it is → how it works → a real example → a gotcha**. That shows you've built one, not just read about it.

### Fundamentals

1. **HCM Extract vs BI Publisher report — when would you use which?**
   Extract: scheduled, high-volume, effective-dated outbound integration, multi-threaded, with Changes Only and built-in delivery. BIP: custom SQL, formatted reports, joins the UEs don't offer. In practice they work together — the extract produces XML, BIP formats it.
2. **Can you write SQL in an HCM Extract?**
   Not directly — data comes from User Entities. You reach extra data through Fast Formula with `GET_VALUE_SET` on a table-validated value set (SQL inside the value set), or by choosing another UE.
3. **What is a User Entity? Can you create your own?**
   Oracle-delivered SQL behind a data group that exposes Database Items. You can't create custom UEs; you pick the right one, add criteria, or extend with formulas/value sets.
4. **How many root data groups can an extract have?**
   Exactly one. All other groups hang under it through connections.
5. **What is the threading database item and why does it matter?**
   The key (e.g., Person ID) used to split work into parallel threads and to compare data in Changes Only. Wrong choice = slow runs or wrong delta detection.
6. **Where is an extract's output stored?**
   The engine archives the data and builds XML; delivery sends formatted files to SFTP/UCM/email. You can always re-download from *View Extract Results*.

### Changes Only

7. **How does Changes Only actually find changes? Does it read audit tables?**
   No. It compares the current snapshot with the archived baseline of previous runs, at the threading level, using the Key attributes.
8. **First run in Changes Only mode — what comes out?**
   Everything, because there's no baseline yet.
9. **You changed one attribute's format in production and the vendor got 40,000 rows instead of 50. Why?**
   Changing the definition resets the baseline; the next run is effectively full.
10. **Vendor needs old and new salary. Which mode?**
    `ATTRIB_OLD` — changed and marked attributes with previous values.
11. **Vendor needs the whole assignment row whenever any field changes. Which mode?**
    `BLOCK` (or `BLOCK_OLD` with old values).
12. **Vendor lost Tuesday's delta. How do you resend it without a full file?**
    Roll back Tuesday's flow and rerun; rollback removes that run from the baseline.
13. **A salary was corrected back-dated to last month. Changes Only didn't pick it up. Why?**
    The value as of the run's effective date didn't differ from the baseline, so the snapshot comparison saw nothing. Handle with date-range logic or a full run.

### Scenarios and debugging

14. **Extract shows 900 employees for you and 1,200 for the integration user. Why?**
    Secured UEs (`_SEC_`) apply the submitter's data security profile.
15. **New hire joining next Monday isn't in today's file. Why and how to fix?**
    The extract reads as of Effective Date; future-dated people aren't effective yet. Add Start/End Date parameters and date-range criteria, or agree with the vendor to send on/after the hire date.
16. **Employees with two assignments appear twice. Fix?**
    Filter Primary Assignment = Y in criteria, or deliberately output one row per assignment.
17. **Expression criteria vs criteria Fast Formula — which is faster and why?**
    Expression: it's appended to the UE SQL and filters in the database. The formula runs per row after data is fetched.
18. **How do you add a header and trailer with a record count?**
    Root-level records: header with String/Parameter attributes; trailer with a COUNT group attribute; positions controlled by record sequence and the eText template.
19. **How do you run an extract from OIC?**
    OIC submits the flow (FlowActionsService), waits for completion, then downloads the file the extract delivered to UCM and sends it on.
20. **The formula was fixed but the output is still wrong.**
    Recompile: Extract Execution Tree → Compile All Formula.
21. **Which tables store an extract's design?**
    `PER_EXT_DEFINITIONS_B`, `PER_EXT_PARAMETERS_B`, `PAY_REPORT_DEFINITIONS`, `PAY_REPORT_BLOCKS` + `PER_EXT_BLOCK_LINKS` (groups and connections), `PAY_REPORT_RECORDS_F`, `PER_EXT_DATA_ELEMENTS_B`, `PER_EXT_DELIVERY_OPTIONS_B`. Runs: `PAY_FLOW_INSTANCES`, `PAY_REQUESTS`, `PAY_PAYROLL_ACTIONS`.
22. **Why do extracts use PAY tables when it's Core HR?**
    Extracts run on the payroll flow and reporting engine (flows, archive, report blocks), shared across HCM.
23. **How do you debug a wrong or missing change?**
    Run *Extracts Process Diagnostics Report* — design-time, runtime, data, or Changes Only diagnostics scope.

## 10. Quick revision cheat sheet

- **Direction:** HDL in, HCM Extracts out
- **Tree:** Definition → Parameters → 1 Root Data Group (UE + threading DBI) → child groups via connections → Records → Attributes → Delivery
- **UE** = Oracle's SQL; **DBI** = one field of a UE; `_SEC_` = secured by submitter's security
- **Attribute types:** DBI, Fast Formula (Rule), String, Parameter, Expression, Group function
- **Criteria:** expression (pushed to SQL, fast) or Extract Criteria formula (Y/N); both → both applied
- **Formula types:** Extract Criteria, Extract Rule, Extract Record; always DEFAULT FOR; `GET_VALUE_SET` for SQL
- **Run:** extracts are payroll flows → Submit Extracts, unique flow instance name, View Extract Results, Refine Extracts to add tasks
- **Compile:** Extract Execution Tree → Compile All Formula
- **Output:** engine makes XML → BIP template (RTF / eText / Excel) → SFTP, UCM, email, DOR
- **Changes Only:** parameter `CHANGES_ONLY` + lookup `PER_EXT_CHANGES_ONLY` / `ORA_HRY_CHANGES_ONLY`; modes N, Y, ATTRIBUTE, ATTRIB_OLD, BLOCK, BLOCK_OLD; first run full; definition edit resets baseline; rollback to resend
- **Tables:** `PER_EXT_DEFINITIONS_B`, `PAY_REPORT_BLOCKS`, `PER_EXT_BLOCK_LINKS`, `PAY_REPORT_RECORDS_F`, `PER_EXT_DATA_ELEMENTS_B`, `PER_EXT_DELIVERY_OPTIONS_B`; runs in `PAY_FLOW_INSTANCES` → `PAY_REQUESTS` → `PAY_PAYROLL_ACTIONS`
- **Debug:** Extracts Process Diagnostics Report

**Practice task for your pod:** build `XX_ACTIVE_EMP_EXTRACT` (Worker → Assignment, 4 attributes, CSV via BIP), run it in mode N, change one employee's department, rerun in ATTRIB_OLD, and compare the two XML outputs. Then query `PER_EXT_DATA_ELEMENTS_VL` for your definition and match each row to an attribute on screen.

## Sources

- [Extract Components — Oracle](https://docs.oracle.com/en/cloud/saas/human-resources/25b/fahex/extract-components.html)
- [Create Data Groups and Extract Data Group Connections — Oracle](https://docs.oracle.com/en/cloud/saas/human-resources/fahex/create-data-groups-and-extract-data-group-connections.html)
- [Guidelines for Using Different Modes for HCM Extracts — Oracle](https://docs.oracle.com/en/cloud/saas/human-resources/fahex/guidelines-for-using-different-modes-for-hcm-extracts.html)
- [Extracts Process Diagnostics Report — Oracle](https://docs.oracle.com/en/cloud/saas/human-resources/fahex/extracts-process-diagnostics-report.html)
- [Extract Actions web service — Oracle](https://docs.oracle.com/en/cloud/saas/human-resources/oeswh/extractactions-d19859e993.html)
- [HCM Extracts for Absence Data — Oracle](https://docs.oracle.com/en/cloud/saas/human-resources/faiam/hcm-extracts-for-absence-data.html)
- [Tables and Views for HCM — Oracle](https://docs.oracle.com/en/cloud/saas/human-resources/25a/oedmh/perextdataelementsb-7592.html)
- [Query for HCM Extract input values — Doyensys](https://doyensys.com/blogs/query-to-get-hcm-extract-input-values-in-bi-report/)