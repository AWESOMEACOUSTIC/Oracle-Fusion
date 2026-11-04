# Oracle HCM Extracts — Core HR Study Guide

> **Goal of this guide:** understand, remember and explain HCM Extracts in interviews.
> Every topic has: **One-line memory hook → Explanation → Key points → Interview answer → Tricky questions**.
> Labels can differ slightly between the **classic "Define Extracts" page** and the **newer "Extract Definitions" page**, but the concepts are the same.

---

## The Big Picture (read this first)

```
 Parameters ──► Extract Definition
                   │
                   ├── Root Data Group  (User Entity + Threading DBI)
                   │      ├── Records ──► Attributes (DBI / Rule / String / Parameter / Calc)
                   │      └── Child Data Group (connected by DBIs)
                   │             └── Records ──► Attributes
                   │
                   ├── Filters / Criteria
                   └── Delivery Options (BI Publisher template + destination)

 Submit / Schedule (Flow Pattern)
      ► Extract engine reads data ► Generates XML ► (Changes Only comparison)
      ► BI Publisher applies RTF / eText template ► Delivers file (SFTP, UCM, Email...)
```

**Memory sentence:** *"Define → Fetch → Filter → Format → Deliver."*
- **Define** the structure (data groups, records, attributes)
- **Fetch** data using User Entities and DBIs
- **Filter** using criteria and rules
- **Format** using BI Publisher templates
- **Deliver** using delivery options and flow patterns

---

## 1. What HCM Extracts Are and Why They Are Used

### One-line hook
**HCM Extracts is Oracle's built-in outbound tool to pull HCM data out of Fusion in a structured format and send it to other systems.**

### Explanation
HCM Extracts is a feature of Oracle Fusion HCM used for **outbound integration**. It reads data from HCM (Core HR, Payroll, Benefits, Absence, etc.), builds an **XML output**, formats it using **BI Publisher templates**, and delivers it to a destination such as an SFTP server, WebCenter Content (UCM), or email.

It is configuration-driven, not code-driven. You do not write SQL directly — you select **User Entities** and **Database Items (DBIs)** that Oracle has already built and secured.

### Why it is used
| Reason | Simple explanation |
|---|---|
| Outbound integration | Send employee data to third parties: payroll vendors, benefits providers, banks, badge/access systems, IT directories |
| Large volume | Built to process thousands of employees using **multi-threading** |
| Changes Only | Can send only what changed since the last run (delta files) |
| Security | Uses secured User Entities, so data access follows HCM security |
| Formatting | Output can be XML, CSV, fixed-width text, Excel, PDF through BI Publisher |
| Scheduling | Runs through flow patterns and can be scheduled (daily, weekly, etc.) |
| Configurable | Business-specific logic through Fast Formulas, criteria and parameters |

### HCM Extracts vs BI Publisher report vs HDL
| Tool | Direction | Main use |
|---|---|---|
| **HCM Extracts** | Outbound | High-volume, scheduled integration files, changes-only feeds |
| **BI Publisher (SQL data model)** | Outbound / reporting | Reports and ad-hoc data with custom SQL |
| **HCM Data Loader (HDL)** | Inbound | Loading data **into** HCM |

### Key points
- Output is always generated as **XML first**, then formatted.
- Strong for **incremental (delta) files** — this is a major advantage over plain BIP reports.
- Uses the **flow framework** (same engine used by payroll flows).

### Interview answer
> "HCM Extracts is an outbound integration tool in Oracle Fusion HCM. We define what data to extract using user entities and database items, organize it into data groups, records and attributes, apply filters and fast formula rules, and the engine generates XML. That XML is formatted through a BI Publisher template — like eText for CSV or fixed-length files — and delivered to destinations like SFTP or UCM. It supports multi-threading for large volumes and a Changes Only mode for delta files, which makes it ideal for third-party interfaces like payroll or benefits vendors."

### Tricky questions
- **Why use HCM Extracts instead of a BIP report?** Changes Only support, multi-threading for high volume, secured predefined user entities, flow-based scheduling and delivery, and built-in archive of results.
- **Can HCM Extracts load data into Fusion?** No — it is outbound. Inbound is HDL / HSDL. (There is an "Inbound Interface" delivery type used to feed extract output to inbound processes, but the extract itself reads data.)

---

## 2. End-to-End Extract Processing Flow

### One-line hook
**Submit → Parameters → Threading → Data fetch → XML → Compare (Changes Only) → BI Publisher → Deliver.**

### Explanation (step by step)
1. **Submit** — The user submits the extract (Submit Extracts / Submit a Flow) or a schedule triggers it. This starts the **flow pattern** of the extract.
2. **Parameters are resolved** — Effective Date, Start/End Date, Changes Only mode, Legislative Data Group, etc.
3. **Root population is identified** — The engine runs the **root data group's User Entity** to get the main list of objects (for Core HR, usually persons or assignments).
4. **Threading (chunking)** — The population is split into chunks using the **Threading Database Item** (like Person ID). Multiple threads process chunks in parallel.
5. **Data is fetched for each object** — For every person, the engine evaluates records and attributes, and moves into **child data groups** using **connections**.
6. **Filters/criteria are applied** — Rows that do not meet the criteria are excluded.
7. **XML is generated and archived** — The extracted data is stored as the extract result and an XML output is produced.
8. **Changes Only comparison (if enabled)** — The current data is compared with the previous successful run, and only changed data is kept.
9. **Formatting** — The report/delivery task calls **BI Publisher**, which applies the RTF or eText template to the XML.
10. **Delivery** — The final file is sent to the configured destination(s).
11. **Monitoring** — Status is visible in Flow/Checklist pages, Scheduled Processes and View Extract Results.

### Simple flow
```
Submit ► Parameters ► Root UE population ► Threads (by Threading DBI)
      ► Records/Attributes + Child DGs ► Criteria ► XML + Archive
      ► Changes Only compare ► BIP Template ► Delivery
```

### Key points
- Two main stages: **Extraction (data → XML)** and **Reporting/Delivery (XML → file)**.
- If the extract stage succeeds but the file is wrong, the problem is usually in the **template or delivery**. If the XML is wrong, the problem is in the **definition, filters or rules**.

### Interview answer
> "When an extract is submitted, its flow pattern starts. The engine resolves the parameters, runs the root data group's user entity to identify the population, and splits it into chunks using the threading database item so multiple threads can run in parallel. For each object it evaluates records and attributes, traverses child data groups through connections, and applies criteria. The result is archived and XML is generated. If Changes Only is used, it compares against the last run. Then BI Publisher applies the template and the output is delivered to the configured destination."

### Tricky questions
- **Where would you check first if the output file is empty?** Check the extract XML/results. If XML is empty → check parameters (effective date), criteria and security. If XML has data but the file is empty → check the template's XML paths and delivery option.

---

## 3. HCM Extract Types

### One-line hook
**The extract type tells Oracle the purpose of the extract and drives how it behaves.**

### Common extract types
| Type | Purpose |
|---|---|
| **Full Profile** | Extracts the complete data set for the population as of the effective date. Most common for Core HR. |
| **Changes Only** | Extracts only data that changed since the previous run (delta). |
| **Payroll Interface** | Sends payroll-related data (pay elements, earnings, deductions) to a third-party payroll provider. |
| **EFT** | Electronic Funds Transfer / payment files. |
| **Archive Retrieval** | Reads data already archived by another process (for example payroll archive) instead of live tables. |

Other payroll/legislative types also exist depending on the modules and release.

### Important note about Changes Only
In practice, delta behaviour is mainly controlled by the **Changes Only parameter** (covered in Topic 15). A Full Profile extract with the Changes Only parameter can behave as full or delta depending on the value passed at runtime. This is why many implementations use one extract definition for both full and delta runs.

### Interview answer
> "The extract type defines the purpose of the extract. For Core HR we mostly use Full Profile, which extracts the complete data as of an effective date. Changes Only is for delta files. Payroll Interface is for third-party payroll providers, EFT is for payment files, and Archive Retrieval reads already archived data. In real projects, we often use Full Profile with the Changes Only parameter so the same extract can run in full or delta mode."

---

## 4. User Entities and Database Items

### One-line hook
**User Entity = the data source (a predefined SQL). Database Item (DBI) = one field from that source.**

### User Entity (UE)
- A **User Entity** is a predefined data source that Oracle built using a **route** (SQL behind the scenes).
- It decides **which rows** come back — for example, one row per person, or one row per assignment.
- Many UEs used in extracts have **"SEC"** in their name, meaning they apply **data security**.
- Examples commonly used in Core HR extracts:
  - `PER_EXT_SEC_PERSON_UE` — person-level data with security (very common root UE)
  - `PER_EXT_SEC_ASSIGNMENT_UE` — assignment-level data with security

### Database Item (DBI)
- A **DBI** is a named field inside a User Entity, like Person Number, First Name, Hire Date, Assignment Status.
- Each DBI maps to a column or expression in the route SQL.
- DBIs are used in three places:
  1. **Attributes** (to output a value)
  2. **Connections** (to join parent and child data groups)
  3. **Filters/criteria** and **Fast Formulas**

### Behind the scenes (for tricky questions)
| Object | Stored in (Fast Formula repository) |
|---|---|
| Routes (SQL) | `FF_ROUTES` |
| User Entities | `FF_USER_ENTITIES` |
| Database Items | `FF_DATABASE_ITEMS` |

So a User Entity sits on top of a Route, and DBIs sit on top of the User Entity.

```
FF_ROUTES (SQL)  ──►  FF_USER_ENTITIES  ──►  FF_DATABASE_ITEMS
   "how data is fetched"   "data source"        "individual fields"
```

### Key points
- You cannot write your own SQL inside an extract data group — you pick a UE. Custom logic goes into **Fast Formulas**.
- Choosing the right UE decides the **level** of data (person vs assignment). Choosing the wrong level causes duplicates.
- Secured UEs mean the person running the extract must have the right **data access**.

### Interview answer
> "A user entity is a predefined data source built on a route, which is SQL defined by Oracle. Database items are the individual fields exposed by that user entity. In an extract, each data group is based on a user entity, and attributes, connections and filters use database items. Behind the scenes these are stored in FF_ROUTES, FF_USER_ENTITIES and FF_DATABASE_ITEMS. For Core HR we commonly use secured user entities like PER_EXT_SEC_PERSON_UE as the root."

### Tricky questions
- **Why do I get duplicate person rows?** You may be using an assignment-level UE where you expected person-level, or a child data group returns multiple rows (multiple assignments, multiple addresses) without a filter like primary flag.
- **Why does the extract return fewer employees for one user than another?** Because secured ("SEC") UEs follow the submitting user's data security.

---

## 5. Extract Definition Structure

### One-line hook
**Extract → Data Groups → Records → Attributes. (Big box → smaller box → row → field.)**

### Explanation
| Level | What it is | Example |
|---|---|---|
| **Extract Definition** | The whole extract: name, type, LDG, parameters, delivery options | `XX_CORE_HR_EMP_EXTRACT` |
| **Data Group** | A block of data based on one User Entity | Person, Assignment, Address |
| **Record** | A row/section inside the data group that groups attributes | Person Details record |
| **Attribute** | A single field in the output | Person Number, Last Name |

### Other parts of the definition
- **Parameters** — runtime inputs (Effective Date, Changes Only)
- **Connections** — links between parent and child data groups
- **Criteria / Filters** — conditions on data groups
- **Delivery Options** — template, output format and destination
- **Extract Execution Tree** — a tree view showing the full hierarchy and **validation/compile status** of each component

### Tag names
Every extract, data group, record and attribute has a **Tag Name**. Tag names become the **XML element names** in the output, and the BI Publisher template refers to them. So tag names should be meaningful and stable.

### Key points
- One extract can have **many data groups**, but only **one root data group**.
- After changes, the extract must be **validated/compiled**. Oracle generates fast formulas internally for records, and if they are not compiled the extract will fail.

### Interview answer
> "An extract definition has a hierarchy: the extract itself, then data groups, each based on a user entity, then records inside the data groups, and attributes inside the records. Every component has a tag name, which becomes the XML element. The definition also holds parameters, data group connections, criteria and delivery options. The Extract Execution Tree shows the full structure and its validation status."

---

## 6. Parameters

### One-line hook
**Parameters are runtime inputs that control what data and which mode the extract runs in.**

### Explanation
Parameters are values passed when the extract is submitted. They can be used in filters, rules, and fast formulas.

### Common parameters
| Parameter | Tag (typical) | Purpose |
|---|---|---|
| Effective Date | `EFFECTIVE_DATE` | The "as of" date for data. Mandatory. |
| Start Date / End Date | `START_DATE` / `END_DATE` | Date range for changes or events |
| Changes Only | `CHANGES_ONLY` | Controls full vs delta mode |
| Legislative Data Group | — | Context for the extract |
| Custom parameters | e.g. `LEGAL_EMPLOYER`, `DEPARTMENT` | Business filters |

### Parameter properties
- **Name / Tag Name**
- **Data Type** — Text, Number, Date
- **Display** — whether the user sees it at submission (Yes / No / Hidden)
- **Mandatory** — required or optional
- **Default value**
- **Lookup / value set** for a list of values (where supported)

### Using parameters in filters
Parameter values are commonly read in filter expressions using:
```sql
pay_report_utils.get_parameter_value('EFFECTIVE_DATE')
```
Example idea: include assignments effective on the parameter date.

### Key points
- **Effective Date** drives date-effective data selection. Wrong effective date = wrong/no data.
- Parameter values can also be bound in the **flow pattern** (Topic 18) — for example, using system date for scheduled runs.

### Interview answer
> "Parameters are runtime inputs to the extract. Effective Date is mandatory and decides the as-of date for date-effective data. We also commonly use Start and End Date and the Changes Only parameter to control delta mode. Custom parameters like legal employer can be added and used in filters, typically read with pay_report_utils.get_parameter_value. For scheduled runs, we bind parameters in the flow pattern so the effective date moves automatically."

---

## 7. Root Data Group and Threading DBI

### One-line hook
**Root data group = the starting point / main population. Threading DBI = the ID used to split work into parallel chunks.**

### Root Data Group
- The **first and top-level** data group in the extract.
- Its User Entity defines **the main population** — for Core HR, usually one row per person (e.g. `PER_EXT_SEC_PERSON_UE`) or per assignment.
- Every other data group is a **child** directly or indirectly connected to the root.
- Only **one root data group** per extract.

### Threading Database Item
- Large extracts are split into **chunks**, and each chunk is processed by a **thread** in parallel.
- The **Threading DBI** tells Oracle *which ID* to chunk by — typically the **Person ID** DBI from the root UE.
- The **Threading Action Type** tells Oracle the type of object being threaded — for example **Person** or **Assignment**.
- The threading DBI must come from the **root UE** and should be **unique per object** at the threading level.

### Why it matters
| If correct | If wrong |
|---|---|
| Fast, parallel processing | Errors, missing data, duplicates or poor performance |
| Each person processed once | Same person may appear in multiple threads, or data mismatches |

Thread count and chunk size can be tuned through a **Process Configuration Group** (parameters like Threads and Chunk Size).

### Interview answer
> "The root data group is the top-level data group whose user entity defines the main population of the extract — for Core HR, usually persons using PER_EXT_SEC_PERSON_UE. There can be only one root. On the root we set the threading action type, like Person, and the threading database item, typically the Person ID from the root UE. The engine uses it to split the population into chunks and process them in parallel threads. Performance can be tuned using a process configuration group for threads and chunk size."

### Tricky questions
- **Can a child data group have a threading DBI?** No — threading is defined on the root data group.
- **Why is the extract slow?** Large population, no criteria on root, too many child data groups or fast formulas, low thread count, or heavy rules evaluated for every person.

---

## 8. Records and Attributes

### One-line hook
**Record = a group of fields in a data group. Attribute = one field and how its value is calculated.**

### Records
- Defined inside a data group.
- Each record has a **Name, Tag Name, Sequence** and a **record type** (for example detail, header or trailer type records depending on the page/version).
- A data group can have **multiple records** (e.g. one record for personal details, another for contact details).
- Oracle internally generates a fast formula per record — this is why records need **compiling**.

### Attributes
Each attribute has:
- **Name** and **Tag Name** (XML element)
- **Data Type** — Text, Number, Date
- **Attribute Type** — where the value comes from
- **Sequence** — order in output

### Attribute types
| Type | Value comes from | Example |
|---|---|---|
| **Database Item Group** | A DBI from the data group's UE | Person Number |
| **Rule** | An **Extract Rule Fast Formula** | Custom full name, derived status |
| **String** | A fixed constant value | `"H"` record identifier, `"CORE_HR"` |
| **Parameter** | A parameter value | Effective Date |
| **Record Calculation** | Calculation using other attributes of the same record | Concatenate first + last name |
| **Summary Calculation** | Aggregation across records (count, sum) | Total employee count for a trailer |

### Key points
- Use **DBI** attributes whenever possible — they are fastest.
- Use **Rule** (fast formula) only for logic that DBIs cannot provide.
- **Summary Calculation** is commonly used for **header/trailer** counts and totals.

### Interview answer
> "Records are containers inside a data group that group related attributes, and each has a tag name and sequence. Attributes are the individual output fields. Each attribute has a type: Database Item Group for direct DBI values, Rule for extract rule fast formulas, String for constants, Parameter for runtime values, Record Calculation for calculations within a record, and Summary Calculation for totals like a trailer count."

---

## 9. Child Data Groups and Connections

### One-line hook
**Child data groups add related data under the parent; connections are the "join" using matching DBIs.**

### Explanation
- A **child data group** is any data group below the root, based on its own UE.
- Example hierarchy for Core HR:
```
Person (Root - PER_EXT_SEC_PERSON_UE)
   └── Assignment (Child)
         └── Supervisor / Job / Department details (Child)
   └── Address (Child)
   └── Email / Phone (Child)
```

### Connections
A connection links the child to its parent using:
| Field | Meaning |
|---|---|
| **Parent Data Group** | Which data group is above |
| **Parent Data Group Database Item** | The ID from the parent (e.g. Person ID) |
| **Data Group Database Item** | The matching ID in the child (e.g. Person ID in assignment UE) |

It works like a SQL join condition: `child.person_id = parent.person_id`.

### One-to-many behaviour
- One parent can have **many child rows** — one person, many assignments or addresses.
- In XML, child rows appear **nested inside** the parent element.
- Use **filters** on the child (primary flag, address type, status) to control which rows come back.

### Key points
- Wrong connection DBIs → no child data, or wrong data attached to a person.
- Keep the hierarchy shallow where possible — every level adds processing cost.

### Interview answer
> "Child data groups bring in related data under the root, like assignments, addresses or emails. Each child is connected to its parent through a connection, where we map the parent data group's DBI to the child's DBI — similar to a join condition, for example person ID to person ID. Because it's one-to-many, a person can have multiple child rows, which appear nested in the XML. We use filters like primary flag to control them."

---

## 10. Build a Simple Full Extract

### One-line hook
**Create → Parameters → Root DG → Records/Attributes → Child DG → Connection → Validate → Delivery → Submit.**

### Example requirement
"Send a daily file of active employees with Person Number, First Name, Last Name, Hire Date, Assignment Number, Department."

### Steps
1. **Navigate** — My Client Groups → **Data Exchange** → Extract Definitions (or classic: HCM Extracts → Define Extracts).
2. **Create extract**
   - Name: `XX Core HR Employee Extract`
   - Type: **Full Profile**
   - Legislative Data Group: as required
   - Start Date: an early date (e.g. `01-01-1951`) so the definition is valid for all data
3. **Parameters** — confirm Effective Date; add Changes Only if delta will be needed later.
4. **Create Root Data Group**
   - Name: `Person`, Tag: `PERSON`
   - User Entity: `PER_EXT_SEC_PERSON_UE`
   - Root Data Group: **Yes**
   - Threading Action Type: **Person**
   - Threading DBI: Person ID DBI
5. **Create Record** in Person DG — `PERSON_DETAILS`
   - Attributes: Person Number, First Name, Last Name, Hire Date (type: Database Item Group)
6. **Create Child Data Group** — `Assignment`, UE: `PER_EXT_SEC_ASSIGNMENT_UE`
7. **Create Connection** — Parent: Person; Parent DBI: Person ID; Child DBI: Person ID.
8. **Create Record** in Assignment DG — Assignment Number, Department Name.
9. **Add Criteria** — active assignments, primary assignment (Topic 12).
10. **Validate / Compile** — check the **Extract Execution Tree**; all components should show valid/compiled.
11. **Delivery Options** — first use **XML output** to check data; then attach BIP template (Topic 16–17).
12. **Submit** — Submit Extracts, enter Effective Date, run, then check **View Extract Results**.

### Key points
- Always test with **XML output first** — it confirms the extract logic before formatting.
- Saving the extract creates a **flow pattern** with the same name.

### Interview answer
> "To build a simple full extract, I create the definition with type Full Profile, add the root data group on PER_EXT_SEC_PERSON_UE with threading action type Person and the person ID as threading DBI, add records and DBI attributes, then add a child data group like assignment and connect it on person ID. I add criteria for active primary assignments, validate the extract so the formulas compile, set a delivery option — first XML to verify — and then submit and check the results."

---

## 11. Understand the Generated XML

### One-line hook
**Tag names become XML tags; hierarchy becomes nesting.**

### Example structure
```xml
<XX_EMP_EXTRACT>
   <PERSON>                      <!-- Root data group -->
      <PERSON_DETAILS>           <!-- Record -->
         <PERSON_NUMBER>1001</PERSON_NUMBER>
         <FIRST_NAME>Ravi</FIRST_NAME>
         <LAST_NAME>Kumar</LAST_NAME>
         <HIRE_DATE>2020-06-15</HIRE_DATE>
      </PERSON_DETAILS>
      <ASSIGNMENT>               <!-- Child data group -->
         <ASSIGNMENT_DETAILS>    <!-- Record -->
            <ASSIGNMENT_NUMBER>E1001</ASSIGNMENT_NUMBER>
            <DEPARTMENT>Finance</DEPARTMENT>
         </ASSIGNMENT_DETAILS>
      </ASSIGNMENT>
   </PERSON>
   <PERSON> ... next person ... </PERSON>
</XX_EMP_EXTRACT>
```

### How BI Publisher sees it
When the extract output is passed to BI Publisher through the standard extract data model (**globalReportsDataModel**), the XML is typically wrapped as:
```
/DATA_DS/G_1/FILE_FRAGMENT/<extract tag>/<data group tag>/<record tag>/<attribute tag>
```
So template paths must follow this structure.

### Key points
- **Tag Name = XML element name** — changing a tag name breaks the template.
- Repeating elements (many persons, many assignments) are what the template loops on.
- With **Changes Only (ATTRIB_OLD mode)**, XML can also carry **old values** alongside new ones.

### Interview answer
> "The extract XML follows the definition hierarchy. The extract tag is the root element, each data group is a nested element, records are inside data groups, and attributes are leaf elements using their tag names. Child data groups are nested inside the parent. When passed to BI Publisher through globalReportsDataModel, the paths typically start with DATA_DS, G_1 and FILE_FRAGMENT, and the template loops on the repeating elements."

---

## 12. Add Filters and Criteria

### One-line hook
**Criteria decide which rows enter the extract.**

### Explanation
Filters can be added at the **data group level** (and in some cases at record level) to restrict rows.

### Common Core HR filters
- Only **active** employees (assignment status)
- Only **primary** assignment
- Only a **legal employer** / business unit
- Only **employees** (exclude contingent workers)
- Data effective as of the **Effective Date** parameter

### Ways to define criteria
1. **Simple criteria** — using DBIs, operators (`=`, `<>`, `IN`, `IS NULL`, `BETWEEN`) and values/parameters.
2. **Advanced filter / expression** — an SQL-style condition using DBIs and functions like:
   ```sql
   pay_report_utils.get_parameter_value('EFFECTIVE_DATE')
   ```
3. **Extract Criteria Fast Formula** — for complex logic that cannot be expressed in a simple condition.

### Key points
- **Filter at the root** whenever possible — it reduces the population early, which improves performance.
- Child filters control which child rows (addresses, assignments) are attached.
- Forgetting date logic is the #1 cause of **duplicate or historical rows**.

### Interview answer
> "Criteria restrict which rows are extracted. We define them at the data group level using DBIs, operators and parameters — for example active, primary assignments as of the effective date. For complex conditions we use an advanced filter expression, often reading parameters with pay_report_utils.get_parameter_value, or an extract criteria fast formula. Filtering at the root data group is best for performance because it reduces the population early."

---

## 13. Learn Extraction Rules

### One-line hook
**Extraction rules are the logic that decides each attribute's value and which rows qualify.**

### Explanation
"Extraction rules" covers all the ways the extract decides **what value to output** and **what to include**:

| Rule area | What it controls | How |
|---|---|---|
| **Value rules** | Value of an attribute | DBI, String, Parameter, Record Calculation, Summary Calculation, Rule (Fast Formula) |
| **Inclusion rules** | Whether a row is included | Data group criteria, advanced filters, criteria fast formula |
| **Change rules** | Whether a change is reported | Changes Only parameter + attribute change settings (Topic 15) |

### When to use which value rule
| Need | Best rule |
|---|---|
| Direct field from HCM | **Database Item** |
| Fixed value (record code "D", file type) | **String** |
| Value entered at runtime | **Parameter** |
| Combine/format fields of same record | **Record Calculation** |
| Count / total for header or trailer | **Summary Calculation** |
| Conditional / complex logic, data not in a DBI, lookups | **Rule (Extract Rule Fast Formula)** |

### Key points
- Order of preference: **DBI → String/Parameter → Calculation → Fast Formula**. Fast formulas are most flexible but cost the most performance.
- Data formatting (dates, padding) is often better done in the **BI Publisher template** rather than in rules.

### Interview answer
> "Extraction rules are the logic that controls attribute values and row inclusion. For values, we can use database items, strings, parameters, record calculations, summary calculations, or extract rule fast formulas for complex logic. For inclusion we use criteria and advanced filters. I prefer DBIs first for performance and use fast formulas only when the logic can't be achieved otherwise."

---

## 14. Learn Extract Rule Fast Formula

### One-line hook
**An Extract Rule Fast Formula returns one value for one attribute, and it must return `RULE_VALUE`.**

### Explanation
- Formula type: **Extract Rule**.
- Created in **Fast Formulas** (Setup and Maintenance / My Client Groups → Fast Formulas).
- Attached to an attribute whose type is **Rule**.
- It runs **for each row** (each person/assignment), using **contexts** like `PERSON_ID`, `EFFECTIVE_DATE`, `HR_ASSIGNMENT_ID` passed from the extract.
- It can read **DBIs**, use **IF/ELSE**, call functions, and must return **`RULE_VALUE`**.

### Basic structure
```
/***********************************************
 Formula Name : XX_EXT_EMP_CATEGORY
 Formula Type : Extract Rule
 Purpose      : Return 'FT' or 'PT' based on working hours
***********************************************/

DEFAULT FOR <WORKING_HOURS_DBI> IS 0     /* use the real DBI name from your pod */

l_hours = <WORKING_HOURS_DBI>

IF l_hours >= 40 THEN
   RULE_VALUE = 'FT'
ELSE
   RULE_VALUE = 'PT'

RETURN RULE_VALUE
```

### Parts to remember
| Part | Purpose |
|---|---|
| `DEFAULT FOR` | Gives a value when the DBI returns nothing — avoids runtime errors |
| `INPUTS ARE` | Optional inputs passed to the formula |
| Local variables | `l_` prefix by convention |
| `IF ... THEN ... ELSE` | Conditional logic |
| `RULE_VALUE` | The mandatory output variable for Extract Rule formulas |
| `RETURN` | Sends the value back to the attribute |

### Related formula types in extracts
- **Extract Rule** — returns a value for an attribute (most common)
- **Extract Criteria** — decides inclusion (filter logic)
- **Extract Record** — record-level processing

### Key points
- **Compile** the formula; then **re-validate** the extract.
- Always add **DEFAULT FOR** on DBIs — missing defaults are a very common runtime error.
- Formulas run per row, so heavy logic slows big extracts.

### Interview answer
> "An extract rule fast formula is used when an attribute needs logic that a DBI alone can't provide. It's created with formula type Extract Rule and attached to an attribute of type Rule. It runs for each row using contexts like person ID and effective date passed by the extract, can read DBIs with default values, apply conditions, and must return the output in RULE_VALUE. After creating it, we compile it and re-validate the extract."

### Tricky questions
- **Formula returns blank for some employees?** The DBI may have no data for that context — check DEFAULT FOR and whether the context (assignment, date) is available.
- **Formula works in one extract but not another?** Contexts differ by data group level — a formula needing an assignment context won't work in a person-level data group.

---

## 15. Learn Changes Only

### One-line hook
**Changes Only = compare current data with the last successful run and send only the difference.**

### Explanation
- The first run creates a **baseline** (full data).
- Every later run **compares** the current data with the previous run's **archived** data.
- Only **new, changed (and depending on setup, ended)** data is output.
- Controlled by the **Changes Only parameter** (`CHANGES_ONLY`) added to the extract.

### Common Changes Only parameter values
| Value | Meaning |
|---|---|
| **N** | No comparison — full extract (all data) |
| **ATTRIBUTE** | Changed attributes only (plus attributes marked to always show) |
| **ATTRIB_OLD** | Changed attributes **with their old values** |
| **BLOCK** | If anything in a block (data group/record) changed, send the whole block |

(Exact values and labels may vary by release — check the list of values on the Changes Only parameter in your environment.)

### Attribute-level controls
- **Always Show / key attributes** — attributes like Person Number that must appear even if unchanged, so the receiver can identify the record.
- **Change Insensitive** — changes to this attribute are **ignored** when deciding if a record changed (e.g. a run timestamp).

### Key points
- Changes Only depends on **archived results of previous runs** — results must be retained.
- Changing the extract structure can affect comparison with older runs.
- The Effective Date / date range should move forward each run (bind in the flow pattern).

### Interview answer
> "Changes Only means the extract compares current data with the archived output of the last successful run and sends only the differences. The first run is the baseline. We control it with the Changes Only parameter — N for full, ATTRIBUTE for changed attributes, ATTRIB_OLD for changed attributes with old values, and BLOCK to send the full block when anything in it changes. Key attributes like person number are set to always display, and attributes like timestamps can be marked change insensitive."

### Tricky questions
- **Delta run sends everyone again — why?** Baseline missing (previous run failed/rolled back), structure changed, or a frequently changing attribute (like a timestamp) is not marked change insensitive.
- **Delta sends nothing — why?** The effective date did not move, or there were no changes after the last run date.

---

## 16. Create BI Publisher RTF and eText Templates

### One-line hook
**RTF = formatted documents (PDF, Excel, HTML). eText = text files (CSV, pipe-delimited, fixed-width) for systems.**

### Setup in BI Publisher
1. Create a report in the BIP catalog (e.g. `/Shared Folders/Custom/HCM/XX_EMP_EXTRACT_RPT`).
2. Use the data model **`globalReportsDataModel`** (standard data model for HCM extract output, found under the Human Capital Management payroll data models folder).
3. Upload the **RTF or eText layout**.
4. Set the output format.
5. Reference this report path and template name in the extract's **Delivery Options**.

### RTF template
- Built in **MS Word** with the BI Publisher Template Builder.
- Uses form fields / tags:
```
<?for-each:/DATA_DS/G_1/FILE_FRAGMENT/XX_EMP_EXTRACT/PERSON?>
   <?PERSON_DETAILS/PERSON_NUMBER?>   <?PERSON_DETAILS/FIRST_NAME?>
<?end for-each?>
```
- Used for human-readable reports — letters, PDF lists, Excel output.

### eText template
- Also an RTF file, but written as **tables of commands**, not visual layout.
- Two types:
  - **FIXED_POSITION_BASED** — each field has a start position and length (bank/payroll vendor files)
  - **DELIMITER_BASED** — fields separated by a delimiter (CSV, pipe `|`)

**Format setup table** (example idea):
| Setting | Value |
|---|---|
| Template Type | DELIMITER_BASED |
| Output Character Set | UTF-8 |
| New Record Character | Carriage Return / Line Feed |

**Data record table** (delimited, example idea):
```
<DEFINE LEVEL> PERSON
<NEW RECORD> EMP_REC
  Data: PERSON_NUMBER      Delimiter: ,
  Data: FIRST_NAME         Delimiter: ,
  Data: LAST_NAME
<END LEVEL> PERSON
```
For fixed position, each row defines **Position, Length, Format, Pad** and **Data**.

eText also supports header/trailer records, counts (`COUNT()`), sequences, date formatting and padding.

### RTF vs eText
| | RTF | eText |
|---|---|---|
| Purpose | Human-readable | System-to-system files |
| Outputs | PDF, Excel, HTML, RTF | Text (CSV, delimited, fixed-width) |
| Design | Visual layout in Word | Table-based commands |
| Typical use | Reports, letters | Vendor interfaces, bank files |

### Interview answer
> "After the extract creates XML, we format it with a BI Publisher report built on the standard globalReportsDataModel. For human-readable outputs like PDF or Excel we use an RTF template with for-each loops on the XML paths. For interface files we use eText — either delimiter-based for CSV or pipe files, or fixed-position for files where each field has a fixed start and length. eText supports define level, new record, headers, trailers and counts. The report path and template are then linked in the extract delivery options."

---

## 17. Configure Delivery Options

### One-line hook
**Delivery options say: which template, which format, what file name, and where to send it.**

### Fields in a delivery option
| Field | Meaning |
|---|---|
| Start / End Date | Validity of the delivery option |
| Delivery Option Name | Identifier |
| Output Type | XML, Text, PDF, Excel, HTML, etc. |
| Report | BIP report path (e.g. `/Custom/HCM/XX_EMP_EXTRACT_RPT.xdo`) |
| Template Name | Layout name inside the report |
| Output Name | File name (can be built dynamically) |
| Delivery Type | Destination |
| Required | Whether failure of this delivery fails the process |
| Compress / Encrypt | Zip output, or PGP encryption (where supported) |

### Common delivery types
| Delivery type | Use |
|---|---|
| **FTP / SFTP** | Send to vendor servers (server must be set up in BIP Administration → Delivery) |
| **WebCenter Content (UCM)** | Store file in UCM; often picked up by OIC or downloaded |
| **Email** | Send file to recipients (email server set up in BIP) |
| **Printer / Fax** | Physical output |
| **Document of Record** | Store as employee document |
| **Inbound Interface** | Feed output into an inbound process |
| **None** | Only generate output (testing) |

### Key points
- **Multiple delivery options** can exist for one extract — e.g. CSV to SFTP and XML to UCM.
- SFTP/Email servers are configured in **BI Publisher Administration**, not in the extract.
- For integrations through **OIC**, a common pattern is: extract → UCM → OIC picks file → transforms → sends to target.

### Interview answer
> "Delivery options define how the extract output is formatted and where it goes. We set the output type, the BIP report path and template name, the output file name, and the delivery type such as SFTP, WebCenter Content, or Email. SFTP and email servers are configured in BIP administration. One extract can have multiple delivery options, and in OIC-based integrations we usually deliver to UCM so OIC can pick the file up."

---

## 18. Learn Flow Patterns and Scheduling

### One-line hook
**Every extract runs as a flow; the flow pattern controls tasks, parameters and scheduling.**

### Flow pattern
- When an extract is saved, Oracle creates a **flow pattern with the same name**.
- The flow typically includes the **extract task** (data → XML) and a **report/delivery task** (BIP formatting and delivery).
- In the classic UI, flows are edited through **Refine Extracts** (Refine HCM Extracts).
- Flows can include extra tasks, for example running two extracts in sequence.

### Parameter binding in flows
Each flow task parameter has a **Parameter Basis** that decides where its value comes from. Common ones:
| Basis | Meaning |
|---|---|
| **Bind to Flow** | Value comes from the flow submission parameter |
| **Constant Bind** | Fixed value |
| **Context Binding** | System context, e.g. **System Date** — useful for schedules |
| **Post SQL Bind** | Value derived via SQL after another bind |

Example: for a daily schedule, bind Effective Date to **System Date** so it moves forward automatically.

### Submitting and scheduling
- **Submit Extracts** (Data Exchange) or **Submit a Flow** (Payroll / Checklist area).
- Choose **As soon as possible** or **Using a schedule** — frequency (daily, weekly, monthly), start and end dates.
- Each run creates a **flow instance** with a unique name.

### Key points
- If a scheduled delta always uses the same date, check the **parameter basis**.
- Flow instance names must be unique for each submission.

### Interview answer
> "Every extract runs through a flow pattern that Oracle auto-creates with the same name as the extract. It contains the extract task and the report/delivery task. In the flow we can bind parameters — for example, binding effective date to system date through context binding so scheduled runs move forward automatically. We submit through Submit Extracts or Submit a Flow, either immediately or on a recurring schedule, and each run creates a flow instance."

---

## 19. Test, Troubleshoot and Monitor

### One-line hook
**Validate → run with XML → check results → check logs → fix layer by layer.**

### Where to monitor
| Place | What you see |
|---|---|
| **View Extract Results** | Output files, status, XML of each run |
| **Checklist / Flow monitoring** | Flow instance status, each task's status |
| **Scheduled Processes** | ESS job status and log files |
| **Extract Execution Tree** | Validation/compile status of components |

### Testing approach
1. **Validate** the extract — everything compiled.
2. Run with **XML output** (or delivery type None) for a **small population** (filter on a test person).
3. Compare the XML with HCM screens.
4. Attach the template and test the file format.
5. Test **delivery** (SFTP/UCM).
6. For delta: run baseline → make a change → run delta → verify only the change appears.

### Common issues and fixes
| Problem | Likely cause | Fix |
|---|---|---|
| Extract fails immediately | Formulas not compiled | Validate / compile the extract and the fast formulas |
| No data in XML | Wrong effective date, too strict criteria, security | Check parameters, filters, user's data access |
| Duplicate rows | Wrong UE level, multiple assignments/addresses, missing date filter | Use primary/effective filters, correct UE |
| Child data missing | Wrong connection DBIs | Fix parent/child DBI mapping |
| XML fine, file empty | Wrong XML path in template / tag changed | Fix template paths (`DATA_DS/G_1/FILE_FRAGMENT/...`) |
| File not delivered | SFTP/email server not configured or wrong credentials | Check BIP delivery configuration |
| Formula error at runtime | Missing `DEFAULT FOR`, missing context | Add defaults, check context availability |
| Slow performance | Large population, heavy formulas, few threads | Filter at root, reduce formulas, tune process configuration group |
| Delta sends full data | Baseline missing or structure changed | Re-establish baseline, check change-insensitive settings |

### Logging and tuning
- A **Process Configuration Group** can set **threads, chunk size and logging** for extract runs.
- Logs are available in the flow task / scheduled process output.

### Behind the scenes (for tricky questions)
Extract runs use the flow/payroll action framework — each run creates a **flow instance** (`PAY_FLOW_INSTANCES`) and a **payroll action** (`PAY_PAYROLL_ACTIONS`) with related ESS requests. Extract definitions are stored in the **`PER_EXT_*`** family of tables (verify exact table names in your environment).

### Interview answer
> "I first validate the extract so all records and formulas compile, then run it with XML output for a small test population and compare the data with the application. Then I test the template and delivery. I monitor through View Extract Results, flow and checklist status, and scheduled process logs. For common issues — no data, I check the effective date, criteria and security; duplicates, I check the UE level and filters; empty file with good XML, I check template paths; delivery failure, I check BIP delivery setup. For performance and logging I use a process configuration group."

---

## Scenario-Based Interview Questions

**1. Vendor wants a daily CSV of new hires and changes only.**
→ Full Profile extract on `PER_EXT_SEC_PERSON_UE`, threading by Person ID, child assignment DG, Changes Only parameter (`ATTRIBUTE` or `BLOCK`), eText delimiter-based template, SFTP delivery, flow scheduled daily with Effective Date bound to System Date. First run = baseline.

**2. Need employee's manager name, which is not a direct DBI in the UE used.**
→ Add a child data group for supervisor data connected on assignment, or use an **Extract Rule Fast Formula** to derive it.

**3. Output shows each employee three times.**
→ Multiple assignments/addresses or history rows. Add primary flag and effective date filters; check UE level.

**4. Extract runs fine for admin but returns fewer people for another user.**
→ Secured UEs follow data security; the submitting user's data role/security profile limits data.

**5. File needs a header with run date and a trailer with record count.**
→ Header record using String/Parameter attributes; trailer using **Summary Calculation**; or build header/trailer in the **eText template** with `COUNT()`.

**6. Third party wants a fixed-width file where Person Number is 10 characters left-padded with zeros.**
→ eText **FIXED_POSITION_BASED** template with Length 10, Pad with '0' on the left.

**7. OIC needs to pick up the extract output.**
→ Deliver to **WebCenter Content (UCM)**; OIC retrieves the file using the content ID / search, then transforms and sends it.

---

## Rapid-Fire Revision Q&A

| Q | A |
|---|---|
| Output format generated first? | XML |
| Data source of a data group? | User Entity |
| A field in a UE? | Database Item (DBI) |
| How many root data groups? | One |
| Where is threading defined? | Root data group (threading action type + threading DBI) |
| Common root UE for Core HR? | `PER_EXT_SEC_PERSON_UE` |
| What links parent and child DGs? | Connections (parent DBI ↔ child DBI) |
| What becomes the XML element name? | Tag Name |
| Mandatory parameter? | Effective Date |
| Function to read parameter in filters? | `pay_report_utils.get_parameter_value` |
| Output variable of Extract Rule FF? | `RULE_VALUE` |
| Attribute type for counts/totals? | Summary Calculation |
| Changes Only value for old + new values? | `ATTRIB_OLD` |
| Ignore an attribute's changes? | Change Insensitive |
| Standard BIP data model for extracts? | `globalReportsDataModel` |
| Template for CSV/fixed files? | eText |
| Where are SFTP servers configured? | BI Publisher Administration → Delivery |
| What gets auto-created on saving extract? | Flow pattern with same name |
| Make scheduled date move automatically? | Context Binding → System Date |
| Tune threads and logging? | Process Configuration Group |
| UE/DBI tables? | `FF_ROUTES`, `FF_USER_ENTITIES`, `FF_DATABASE_ITEMS` |

---

## Final Memory Map

```
WHAT     → Outbound integration tool, XML first
HOW      → Submit → Threads → Fetch → XML → Compare → BIP → Deliver
TYPES    → Full Profile | Changes Only | Payroll Interface | EFT | Archive Retrieval
SOURCE   → User Entity (route SQL) + DBIs
SHAPE    → Extract → Data Group → Record → Attribute (Tag Names = XML)
INPUTS   → Parameters (Effective Date, Changes Only)
START    → Root DG + Threading Action Type + Threading DBI
JOIN     → Child DG + Connections (parent DBI = child DBI)
FILTER   → Criteria / Advanced Filter / Criteria FF
LOGIC    → DBI → String → Parameter → Calc → Rule FF (RULE_VALUE)
DELTA    → CHANGES_ONLY: N | ATTRIBUTE | ATTRIB_OLD | BLOCK
FORMAT   → globalReportsDataModel + RTF (docs) / eText (files)
DELIVER  → SFTP | UCM | Email | Printer | DOR
RUN      → Flow Pattern + Parameter Basis + Schedule
FIX      → Validate → XML test → Results → Logs → Layer-by-layer
```