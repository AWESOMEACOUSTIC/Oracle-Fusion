# BI Publisher for Oracle HCM: Phase 3, Data Model

> **Goal of this phase:** Be able to build a complete data model (.xdm) on your own: write the SQL, add parameters and LOVs, link master–detail data sets, shape the XML structure, and save sample data for template design.

**Topics in this phase**
13. Creating a data model and writing the SQL query
14. Other data set types
15. Parameters and bind variables
16. List of Values (LOV)
17. Master–detail relationships
18. Data structure: XML elements, groups, aggregates, filters, sorting
19. Calculated columns and expressions
20. System (global) variables
21. Event triggers and data model properties
22. Viewing data and saving sample XML

---

## 13. Creating a Data Model and Writing the SQL Query

### What is a data model?
A **data model (.xdm)** is the BIP object that defines **what data to fetch** and **how that data is structured as XML**. It doesn't define how the output looks; that's the template's job.

### Steps to create a data model
1. **Navigator → Tools → Reports and Analytics → Browse Catalog** (or open `/xmlpserver`).
2. **New → Data Model**.
3. In the left pane, click **Data Sets → New Data Set (+) → SQL Query**.
4. Fill in the dialog:
   - **Name**: a meaningful name (e.g., `EMP_DS`), not the default.
   - **Data Source**: **`ApplicationDB_HCM`**
   - **Type of SQL**: **Standard SQL**
5. Write the query in the text box (or use **Query Builder**, a graphical tool; on projects most people type the SQL).
6. Click **OK**. BIP checks the SQL and creates the data set.
7. **Save** in `/Shared Folders/Custom/<your folder>/` with a proper name (e.g., `XX_ACTIVE_EMP_DM`).

### Data model editor layout (left pane)

| Component | Purpose |
|---|---|
| **Data Sets** | The queries that fetch data |
| **Event Triggers** | Logic that runs before or after data is fetched, or decides whether a scheduled job runs |
| **Flexfields** | Key flexfield handling (mainly used with E-Business Suite; rarely used in Fusion HCM) |
| **List of Values** | Value lists shown in parameter drop-downs |
| **Parameters** | Inputs the user enters when running the report |
| **Bursting** | Rules to split and deliver output (Phase 5) |
| **Properties** (top of tree) | Data model-level settings (Topic 21) |

### The three views of a data set

| Tab | What you do there |
|---|---|
| **Diagram** | See data sets visually, create links (master–detail), add aggregates, group by elements |
| **Structure** | Rename XML element names, set sorting, set null value handling |
| **Code** | See the generated XML structure as code (read-only overview) |

### SQL rules inside BIP ⭐
- **No semicolon (`;`)** at the end of the query. It causes an error.
- **Every column must have a unique name.** If two tables have `NAME`, give aliases (`dept.name AS department_name`).
- **Column aliases become XML element names.** BIP converts them to upper case by default (`department_name` → `DEPARTMENT_NAME`). Choose clean, meaningful aliases because the template uses these names.
- **Dates** come out in XML in a standard format (e.g., `2026-11-05T00:00:00.000+05:30`). Format them **in the template**, or use `TO_CHAR` in SQL if the output needs a fixed format (common for eText files).
- Avoid `SELECT *`. Select only the columns you need.

> **Interview-ready line:**
> "I create a data model from the catalog, add a SQL Query data set with ApplicationDB_HCM as the data source, and write standard SQL without a trailing semicolon. Column aliases become XML element names, so I keep them meaningful because the template refers to them."

---

## 14. Other Data Set Types

SQL Query is used most, but a data model can use other sources:

| Data set type | When it's used |
|---|---|
| **SQL Query** | Most common. Direct SQL on `ApplicationDB_HCM`. |
| **Oracle BI Analysis** | Uses an existing **OTBI analysis** as the data source. Useful to get OTBI's automatic security with BIP formatting. |
| **XML File** | Data from an uploaded XML file (e.g., for testing or static data). |
| **Microsoft Excel File / CSV File** | Data from an uploaded spreadsheet or CSV (e.g., a mapping table the client provides). |
| **Web Service / HTTP (XML Feed)** | Data from a web service or URL returning XML. |

**Points to remember about Oracle BI Analysis data sets:**
- The analysis must already exist in the catalog.
- Analysis filters can be connected to BIP parameters.
- They are usually slower and less flexible than SQL, so they're used only when OTBI security or an existing analysis is specifically needed.

**A data model can have multiple data sets**, even of different types. They can be independent (separate XML sections) or linked (master–detail).

> **Interview-ready line:**
> "Besides SQL, a data model can use an OTBI analysis, XML, Excel or CSV files, or a web service as data sets. In HCM projects SQL is the most common, and an OTBI analysis is used when we want OTBI's built-in security."

---

## 15. Parameters and Bind Variables ⭐

### What are they?
- A **parameter** is an **input the user gives** when running the report (e.g., Effective Date, Department).
- A **bind variable** is how that input is **used inside the SQL**, written with a colon: **`:P_EFFECTIVE_DATE`**.

At run time, BIP replaces the bind variable with the value the user entered.

### How to create them
1. Write the bind variable in the SQL, e.g., `WHERE ppos.date_start BETWEEN :P_FROM_DATE AND :P_TO_DATE`.
2. Click **OK** on the data set. BIP detects the bind variables and **offers to create parameters** for them automatically.
3. Open **Parameters** in the left pane and set each one's type and properties.

**The parameter name must exactly match the bind variable name** (without the colon). `:P_FROM_DATE` ↔ parameter `P_FROM_DATE`.

### Parameter types

| Type | What the user sees | Typical use |
|---|---|---|
| **Text** | A free text box | Person number, name search |
| **Menu** | A drop-down list (needs an **LOV**) | Department, business unit, legal employer |
| **Date** | A date picker | Effective date, from/to dates |

### Important parameter properties

| Property | Meaning |
|---|---|
| **Data Type** | String, Integer, Float, Boolean, Date |
| **Default Value** | Value pre-filled when the report opens |
| **Parameter Label** | The label the user sees (e.g., "Effective Date" instead of `P_EFFECTIVE_DATE`) |
| **Row Placement** | Which row the parameter appears on in the report viewer |
| **Mandatory** | Whether the user must enter a value |
| **Text field contains comma-separated values** *(Text)* | Lets a user type several values separated by commas |
| **Multiple Selection** *(Menu)* | User can pick more than one value |
| **Can Select All** *(Menu)* | Adds an "All" option; choose whether to pass **NULL** or **all values** |
| **Refresh other parameters on change** *(Menu)* | Reloads dependent parameters (cascading LOVs) |
| **Date Format** *(Date)* | e.g., `dd-MM-yyyy` |

### Date parameter defaults
BIP has built-in date expressions you can use as default values:

| Expression | Value |
|---|---|
| `{$SYSDATE()$}` | Today |
| `{$FIRST_DAY_OF_MONTH()$}` | First day of the current month |
| `{$LAST_DAY_OF_MONTH()$}` | Last day of the current month |
| `{$FIRST_DAY_OF_YEAR()$}` | First day of the current year |
| `{$LAST_DAY_OF_YEAR()$}` | Last day of the current year |

These are very useful for **scheduled reports**, e.g., a monthly new-hires report that always runs for "first day to last day of this month" without anyone entering dates.

### Using parameters in SQL: common patterns ⭐

**a) Effective date with default to today**
```sql
AND NVL(:P_EFFECTIVE_DATE, TRUNC(SYSDATE))
        BETWEEN paam.effective_start_date AND paam.effective_end_date
```
If the user leaves it blank, the report runs as of today.

**b) Optional single-value parameter**
```sql
AND (:P_DEPT_ID IS NULL OR paam.organization_id = :P_DEPT_ID)
```
- User leaves it blank → condition is true for all rows → all departments.
- User selects a department → only that department.

**c) Multi-select parameter**
```sql
AND paam.organization_id IN (:P_DEPT_ID)
```
BIP expands `:P_DEPT_ID` into the list of selected values: `IN (101, 102, 103)`.

**d) Multi-select + optional ("All" passes NULL)**
```sql
AND (COALESCE(NULL, :P_DEPT_ID) IS NULL OR paam.organization_id IN (:P_DEPT_ID))
```
**Why COALESCE and not `:P_DEPT_ID IS NULL`?**
When the user selects several values, BIP expands the bind into a list. `:P_DEPT_ID IS NULL` would become `101, 102 IS NULL`, which is invalid SQL and causes an error. `COALESCE` accepts many arguments, so `COALESCE(NULL, 101, 102)` is still valid SQL. When nothing (or "All" with NULL) is passed, it returns NULL and the condition is true for all rows.

**e) Text parameter with partial search**
```sql
AND UPPER(ppnf.full_name) LIKE '%' || UPPER(:P_NAME) || '%'
```

### "Can Select All": NULL vs All Values

| Option | What BIP passes | Performance |
|---|---|---|
| **NULL Value Passed** | NULL | **Better.** Use with pattern (d). |
| **All Values Passed** | Every value in the LOV | Slower with large lists, and can hit Oracle's 1000-item `IN` list limit |

> **Interview-ready line:**
> "Parameters are user inputs, and bind variables like :P_EFFECTIVE_DATE use them in SQL. The parameter name must match the bind variable. Parameters can be Text, Menu or Date. For optional parameters I use ':P IS NULL OR column = :P'. For multi-select I use 'IN (:P)', and for optional multi-select I use COALESCE(NULL, :P) IS NULL, because IS NULL fails when multiple values are passed. For 'Select All' I prefer passing NULL for better performance."

---

## 16. List of Values (LOV)

### What is an LOV?
An **LOV** is the **list of options** shown in a **Menu-type parameter's drop-down**. You create the LOV first, then attach it to the parameter.

### LOV types

| Type | How values are defined | Example |
|---|---|---|
| **SQL Query** | Values come from a query | List of all departments |
| **Fixed Data** | Values typed manually (label + value pairs) | Report Type: "Summary" = `S`, "Detail" = `D` |

### Writing an LOV query ⭐
An LOV query has **two columns**:
1. **First column**: the **label shown** to the user
2. **Second column**: the **value passed** to the bind variable

```sql
SELECT dept.name             AS display_value,
       dept.organization_id  AS return_value
FROM   hr_organization_units_f_tl     dept
JOIN   hr_org_unit_classifications_f  cls
       ON  cls.organization_id = dept.organization_id
       AND cls.classification_code = 'DEPARTMENT'
       AND TRUNC(SYSDATE) BETWEEN cls.effective_start_date AND cls.effective_end_date
WHERE  dept.language = USERENV('LANG')
AND    TRUNC(SYSDATE) BETWEEN dept.effective_start_date AND dept.effective_end_date
ORDER BY dept.name
```
- The user sees **department names**, but the SQL receives the **ORGANIZATION_ID**.
- Passing IDs is faster and safer than passing names, because names can repeat or change.
- `HR_ORG_UNIT_CLASSIFICATIONS_F` with `CLASSIFICATION_CODE = 'DEPARTMENT'` keeps only departments. Organizations also include business units, legal entities, and others.
- The LOV query follows the same Phase 2 rules: date filters, language filter, and secured views if users should only see their own departments.

### Cascading (dependent) LOVs
Example: the user picks a **Business Unit**, and the **Department** list shows only departments in that business unit.

1. Create the parent parameter `P_BU_ID` (Menu, with a BU LOV).
2. Write the department LOV with a bind to the parent:
   ```sql
   ... WHERE (:P_BU_ID IS NULL OR dept_bu_column = :P_BU_ID)
   ```
3. On the **parent** parameter (`P_BU_ID`), tick **"Refresh other parameters on change"**.
4. Make sure the parent parameter is listed **before** the child in the Parameters list.

> **Interview-ready line:**
> "An LOV provides drop-down values for Menu parameters. It can be SQL-based or fixed data. In a SQL LOV, the first column is the display value and the second is the value passed to the bind variable, so I show names but pass IDs. For cascading LOVs, the child LOV query references the parent parameter, and I enable 'Refresh other parameters on change' on the parent."

---

## 17. Master–Detail Relationships ⭐

### What is master–detail?
A **master–detail** (parent–child) relationship is used when **one master record has many detail records**:

| Master (one) | Detail (many) |
|---|---|
| Employee | Their assignment history |
| Employee | Their dependents |
| Department | Employees in that department |
| Employee | Their element entries (earnings/deductions) |

The goal is **nested XML**: each master record contains its own list of details. This nesting lets templates show one employee with a table of their records under it (and enables things like one page or one letter per employee).

### Why not just one big join?
A single join **repeats master data on every detail row**:

| PERSON_NUMBER | NAME | DEPENDENT |
|---|---|---|
| 1001 | Ravi Kumar | Anita |
| 1001 | Ravi Kumar | Arjun |

The XML is flat, so the template has to regroup it. Master–detail gives **hierarchical XML** directly.

### Method 1: Two data sets + element link (the standard way)

**Step 1: Master data set (`EMP_DS`)**
```sql
SELECT papf.person_id,
       papf.person_number,
       ppnf.full_name
FROM   per_all_people_f   papf
JOIN   per_person_names_f ppnf
       ON  ppnf.person_id = papf.person_id
       AND ppnf.name_type = 'GLOBAL'
       AND TRUNC(SYSDATE) BETWEEN ppnf.effective_start_date AND ppnf.effective_end_date
WHERE  TRUNC(SYSDATE) BETWEEN papf.effective_start_date AND papf.effective_end_date
```

**Step 2: Detail data set (`DEP_DS`)**
```sql
SELECT pcr.person_id       AS dep_emp_person_id,
       dpn.full_name       AS dependent_name,
       pcr.contact_type    AS relationship
FROM   per_contact_relships_f pcr
JOIN   per_person_names_f     dpn
       ON  dpn.person_id = pcr.contact_person_id
       AND dpn.name_type = 'GLOBAL'
       AND TRUNC(SYSDATE) BETWEEN dpn.effective_start_date AND dpn.effective_end_date
WHERE  TRUNC(SYSDATE) BETWEEN pcr.effective_start_date AND pcr.effective_end_date
```

**Step 3: Create the link**
- In the **Diagram** tab, on the master's `PERSON_ID` element, click the menu → **Create Link** (or drag from `PERSON_ID` to `DEP_EMP_PERSON_ID`).
- Choose the **parent element** (`PERSON_ID`) and the **child element** (`DEP_EMP_PERSON_ID`).
- This is an **element-level link**. It works like a join condition between the two data sets.

**Resulting XML**
```xml
<DATA_DS>
  <G_EMP>
    <PERSON_ID>300100</PERSON_ID>
    <PERSON_NUMBER>1001</PERSON_NUMBER>
    <FULL_NAME>Ravi Kumar</FULL_NAME>
    <G_DEP>
      <DEP_EMP_PERSON_ID>300100</DEP_EMP_PERSON_ID>
      <DEPENDENT_NAME>Anita Kumar</DEPENDENT_NAME>
      <RELATIONSHIP>S</RELATIONSHIP>
    </G_DEP>
    <G_DEP>
      <DEP_EMP_PERSON_ID>300100</DEP_EMP_PERSON_ID>
      <DEPENDENT_NAME>Arjun Kumar</DEPENDENT_NAME>
      <RELATIONSHIP>C</RELATIONSHIP>
    </G_DEP>
  </G_EMP>
</DATA_DS>
```
Each `G_EMP` (employee) contains **its own** `G_DEP` (dependents).

**Rules for element links**
- The parent and child linking columns **must have different names** (that's why the child uses `DEP_EMP_PERSON_ID`).
- Data types of both columns should match.
- An employee **with no details** still appears in the master group, with no child group. It behaves like an outer join.

### Method 2: Bind variable referencing the parent column
The detail query can refer directly to a **parent column** as a bind variable:
```sql
SELECT ...
FROM   per_contact_relships_f pcr
WHERE  pcr.person_id = :PERSON_ID
```
- `:PERSON_ID` here is **the master data set's column**, not a user parameter.
- BIP passes the value from each master row.
- This method can be slower for large data because the detail query runs per master row. **Element links are the standard approach.**

### Method 3: Single query + "Group By" in the diagram
- Write **one** joined query with both master and detail columns.
- In the Diagram tab, on the master column (e.g., `PERSON_NUMBER`), choose **Group By**.
- BIP creates a **parent group** with that element and moves the rest into a **child group**.
- This is useful when one query is simpler or faster, and you still want nested XML.

### Multiple levels
You can link several levels: **Department → Employee → Dependents**. Each link creates one more level of nesting.

> **Interview-ready line:**
> "Master–detail creates parent–child XML when one master record has many detail records, like an employee and their dependents. I create two data sets and link them in the diagram using an element-level link on the key column, with different column names in parent and child. This produces nested groups, so the template can loop through details within each master. Alternatives are a child query using the parent column as a bind variable, or a single query with Group By in the diagram."

---

## 18. Data Structure: XML Elements, Groups, Aggregates, Filters, Sorting

### Groups and elements
- A **group** = a repeating block in the XML (one per row). Default names are **`G_1`, `G_2`** …
- An **element** = a column inside a group (e.g., `PERSON_NUMBER`).
- The root tag is usually **`DATA_DS`**.

### Rename groups early ⭐
- Rename `G_1` → `G_EMP`, `G_2` → `G_DEP` **before** building templates.
- Templates loop on group names (`for-each:G_EMP`). **Renaming a group after the template is built breaks the template.**

### Structure tab: what you can change

| Setting | Use |
|---|---|
| **XML Tag Name** | The element name in XML (what the template uses) |
| **Display Name** | Friendly name shown in the BIP layout editor |
| **Sorting** | Sort order within the group (ascending or descending) |
| **Value if Null** | Default value when the column is NULL (e.g., `N/A` or `0`) |

### Aggregate functions (group-level totals)
Totals can be calculated **in the data model**, at the parent group level.

**How:** In the Diagram tab, drag an element from the child group onto the **aggregate area** of the parent group (or use the element menu) and choose the function.

| Function | Example |
|---|---|
| **Count** | Number of employees per department |
| **Summation** | Total salary per department |
| **Average** | Average salary |
| **Maximum / Minimum** | Highest or lowest salary |
| **First / Last** | First or last value in the group |

- The new element is created at the parent level, with default names like **`CS_1`**. Rename it (e.g., `DEPT_EMP_COUNT`).
- It also adds a **summary at the data model level**, so the template only needs to print it.

### Group filter
- A **filter condition on a group** (e.g., show only rows where `SALARY > 50000`).
- **Filtering in the SQL `WHERE` clause is usually better for performance.** Group filters are useful only when the data is already fetched and needs extra conditions at group level.

### Group sorting
- Sort within a group in the Structure tab.
- An `ORDER BY` in SQL also works, but it's **not guaranteed after linking**. For linked data sets, set the sort on the group.

> **Interview-ready line:**
> "In the data model, each data set becomes a group in the XML, and each column becomes an element. I rename default group names like G_1 to meaningful names before building the template, because templates loop on group names. In the Structure tab I can change XML tag names, sorting, and null values. At group level I can add aggregates like count and sum, and group filters, although I prefer filtering in SQL for performance."

---

## 19. Calculated Columns and Expressions

### Two ways to calculate values

| Where | How | When to use |
|---|---|---|
| **In SQL** (recommended) | `CASE`, `DECODE`, `NVL`, `TO_CHAR`, `||`, arithmetic, date functions | Almost always. Faster, easier to test, easy to read. |
| **In the data model** | Group menu → **Add Element by Expression** | When the value depends on other group-level elements (e.g., an aggregate), or the SQL can't be changed. Default names look like `CF_1`. |

### SQL calculation examples

```sql
-- Full name in a specific format
ppnf.last_name || ', ' || ppnf.first_name                AS formatted_name,

-- Years of service
TRUNC(MONTHS_BETWEEN(TRUNC(SYSDATE), ppos.date_start) / 12) AS years_of_service,

-- Status label
CASE WHEN ppos.actual_termination_date IS NULL
     THEN 'Active' ELSE 'Terminated' END                 AS emp_status,

-- Fixed-format date for a text file
TO_CHAR(ppos.date_start, 'YYYYMMDD')                     AS hire_date_yyyymmdd,

-- Default for missing data
NVL(pea.email_address, 'No Email')                       AS work_email
```

### Data model expression example
- Parent group has aggregates `TOTAL_SALARY` and `EMP_COUNT`.
- **Add Element by Expression** at the parent group: `TOTAL_SALARY / EMP_COUNT` → `AVG_SALARY_CALC`.

> **Interview-ready line:**
> "I do most calculations in SQL using CASE, NVL, TO_CHAR and date functions, because it's faster and easier to test. The data model also supports expression elements at group level, which I use when the value depends on group aggregates."

---

## 20. System (Global) Variables

BIP provides built-in variables you can use as **bind variables in SQL** without creating parameters:

| Variable | Holds |
|---|---|
| **`:xdo_user_name`** | Username of the person running the report |
| **`:xdo_user_roles`** | Roles of the user |
| **`:xdo_user_report_oracle_lang`** | User's report language (Oracle language code) |
| **`:xdo_user_report_locale`** | User's report locale (e.g., `en-US`) |
| **`:xdo_user_ui_oracle_lang`** | User's UI language |
| **`:xdo_user_ui_locale`** | User's UI locale |

### Common uses
**a) Self-service report: show only the logged-in person's data**
```sql
AND papf.person_id = (SELECT pu.person_id
                      FROM   per_users pu
                      WHERE  pu.username = :xdo_user_name)
```

**b) Manager report: show only the logged-in manager's direct reports**
```sql
AND pasf.manager_id = (SELECT pu.person_id
                       FROM   per_users pu
                       WHERE  pu.username = :xdo_user_name)
AND pasf.manager_type = 'LINE_MANAGER'
```

**c) Audit: print who ran the report**
```sql
SELECT :xdo_user_name AS run_by, ... FROM ...
```

- For a **scheduled** report, `:xdo_user_name` is the user who **scheduled** it.

> **Interview-ready line:**
> "BIP has system variables like :xdo_user_name, which gives the username of the person running the report. I use it with PER_USERS to restrict data to the logged-in employee, or to their direct reports through PER_ASSIGNMENT_SUPERVISORS_F."

---

## 21. Event Triggers and Data Model Properties

### Event triggers

| Trigger type | When it runs | Purpose |
|---|---|---|
| **Before Data** | Before the data sets execute | Run PL/SQL setup logic (e.g., populate a temporary table) |
| **After Data** | After the XML is generated | Run PL/SQL logic after extraction (e.g., update a flag, log the run) |
| **Schedule** | When a scheduled job is about to run | A SQL query that decides **whether the job should run**. If the query returns rows, the job runs; if not, it's skipped. |

**Fusion SaaS reality:**
- Before/After Data triggers call a PL/SQL function in a **database package** (set in the data model's **Oracle DB Default Package** property).
- In Fusion Cloud, you **can't create custom PL/SQL packages** in the database, so these triggers are rarely used in HCM projects.
- **Schedule triggers** are practical in Fusion. Example: run the "New Hires" report only if there were new hires yesterday:
  ```sql
  SELECT 1
  FROM   per_periods_of_service ppos
  WHERE  ppos.date_start = TRUNC(SYSDATE) - 1
  ```

### Data model properties (click the data model name at the top of the left pane)

| Property | Use |
|---|---|
| **Description** | Documentation of what the data model does |
| **Default Data Source** | Data source used by new data sets (`ApplicationDB_HCM`) |
| **Oracle DB Default Package** | PL/SQL package for event triggers (rarely used in Fusion) |
| **Database Fetch Size** | Rows fetched per round trip (performance tuning) |
| **Enable SQL Pruning** | Fetches only the columns the layout actually uses (performance; only with certain layout types) |
| **Include Parameter Tags** | Adds parameter values into the XML so the template can print them (e.g., "Report as of 05-Nov-2026") |
| **Include Empty Tags for Null Elements** | Adds an empty tag when a value is NULL instead of dropping the tag. **Important for eText files**, where a missing tag can shift fixed positions. |
| **XML Tag Display** | Upper case, lower case, or as written |

> **Interview-ready line:**
> "Data models support Before Data, After Data and Schedule triggers. In Fusion SaaS, we can't create custom PL/SQL packages, so Before and After Data triggers are rarely used, but Schedule triggers are useful: a SQL query decides whether a scheduled job runs, for example only when there are new hires. Data model properties include the default data source, including parameter tags in the XML, and including empty tags for nulls, which matters for fixed-format files."

---

## 22. Viewing Data and Saving Sample XML ⭐

### View Data
1. Open the **View Data** tab in the data model.
2. Enter parameter values and choose the **number of rows**.
3. Click **View**. You'll see the generated XML (as a tree or a table).

**Use it to check, before building any template:**
- Are the **row counts** correct? (no duplicates, no missing employees)
- Are the **group names and nesting** correct?
- Are **element names** clean and meaningful?
- Do **parameters** filter correctly?

### Save as Sample Data
- After viewing, click **Save As Sample Data**. The XML is **attached to the data model**.

**Why sample data is needed:**
- **BIP online layout editor** and **template viewers** use it to show previews.
- **Template Builder for Word** (RTF) needs the XML file to insert fields and preview output.
- You can **Export** the XML file and load it into Word or Excel while designing the template offline.

### Re-save sample data when the structure changes
If you add a column or rename a group, **save new sample data**. Otherwise the template preview uses the old structure and fields will be missing.

### Debugging with XML (Phase 1 recall)
- **Data in XML but not in output** → template problem (wrong tag name, wrong group in the loop, case mismatch; XML tags are **case-sensitive**).
- **Data missing from XML** → SQL or parameter problem.

> **Interview-ready line:**
> "Before building a template, I check the XML in the View Data tab: row counts, group nesting, and element names. Then I save it as sample data, which the layout editor and Template Builder for Word use for design and preview. Whenever the data structure changes, I re-save the sample data."

---

## Complete Example: Department-wise Employee Report Data Model

**Requirement:** Show employees grouped by department, as of a chosen date, with a department filter (multi-select, optional), and a count of employees per department.

**Parameters**
| Parameter | Type | Details |
|---|---|---|
| `P_EFFECTIVE_DATE` | Date | Default `{$SYSDATE()$}` |
| `P_DEPT_ID` | Menu | LOV `DEPT_LOV`, Multiple Selection, Can Select All → NULL Value Passed |

**Data set `EMP_DS`**
```sql
SELECT dept.name                AS department_name,
       papf.person_number,
       ppnf.full_name,
       job.name                 AS job_name,
       ppos.date_start          AS hire_date
FROM   per_all_assignments_m    paam
JOIN   per_all_people_f         papf
       ON  papf.person_id = paam.person_id
       AND :P_EFFECTIVE_DATE BETWEEN papf.effective_start_date AND papf.effective_end_date
JOIN   per_person_names_f       ppnf
       ON  ppnf.person_id = paam.person_id
       AND ppnf.name_type = 'GLOBAL'
       AND :P_EFFECTIVE_DATE BETWEEN ppnf.effective_start_date AND ppnf.effective_end_date
JOIN   per_periods_of_service   ppos
       ON  ppos.period_of_service_id = paam.period_of_service_id
JOIN   hr_organization_units_f_tl dept
       ON  dept.organization_id = paam.organization_id
       AND dept.language = USERENV('LANG')
       AND :P_EFFECTIVE_DATE BETWEEN dept.effective_start_date AND dept.effective_end_date
LEFT JOIN per_jobs_f_tl         job
       ON  job.job_id = paam.job_id
       AND job.language = USERENV('LANG')
       AND :P_EFFECTIVE_DATE BETWEEN job.effective_start_date AND job.effective_end_date
WHERE  paam.assignment_type IN ('E','C')
AND    paam.primary_flag = 'Y'
AND    paam.assignment_status_type = 'ACTIVE'
AND    paam.effective_latest_change = 'Y'
AND    :P_EFFECTIVE_DATE BETWEEN paam.effective_start_date AND paam.effective_end_date
AND    (COALESCE(NULL, :P_DEPT_ID) IS NULL OR paam.organization_id IN (:P_DEPT_ID))
```

**Structure steps**
1. In the Diagram, **Group By** `DEPARTMENT_NAME` → creates a parent group.
2. Rename groups: parent → `G_DEPT`, child → `G_EMP`.
3. Add aggregate **Count** of `PERSON_NUMBER` at `G_DEPT` → rename to `DEPT_EMP_COUNT`.
4. Sort `G_EMP` by `FULL_NAME`.
5. **View Data** → check → **Save As Sample Data**.

**Resulting XML (shape)**
```xml
<DATA_DS>
  <P_EFFECTIVE_DATE>2026-11-05</P_EFFECTIVE_DATE>
  <G_DEPT>
    <DEPARTMENT_NAME>Finance</DEPARTMENT_NAME>
    <DEPT_EMP_COUNT>2</DEPT_EMP_COUNT>
    <G_EMP>
      <PERSON_NUMBER>1001</PERSON_NUMBER>
      <FULL_NAME>Anita Rao</FULL_NAME>
      <JOB_NAME>Accountant</JOB_NAME>
      <HIRE_DATE>2023-06-01T00:00:00.000+05:30</HIRE_DATE>
    </G_EMP>
    <G_EMP>
      ...
    </G_EMP>
  </G_DEPT>
  <G_DEPT>
    <DEPARTMENT_NAME>IT</DEPARTMENT_NAME>
    ...
  </G_DEPT>
</DATA_DS>
```
(The parameter tag appears only if **Include Parameter Tags** is enabled.)

This XML is ready for an RTF template that prints each department as a section with its employee table and count. That's Phase 4.

---

## Quick Revision Sheet

1. **Data model = .xdm.** Defines data and XML structure, not layout.
2. **SQL data set:** data source `ApplicationDB_HCM`, Standard SQL, **no semicolon**, unique aliases.
3. **Aliases become XML tags** (upper case by default). Tags are **case-sensitive**.
4. **Other data sets:** OTBI analysis, XML, Excel, CSV, web service.
5. **Bind variable** `:P_NAME` must match **parameter** `P_NAME`. BIP offers to auto-create parameters.
6. **Parameter types:** Text, Menu (needs LOV), Date.
7. **Date defaults:** `{$SYSDATE()$}`, `{$FIRST_DAY_OF_MONTH()$}`, `{$LAST_DAY_OF_MONTH()$}`.
8. **Optional:** `(:P IS NULL OR col = :P)`. **Multi-select:** `col IN (:P)`. **Optional multi-select:** `(COALESCE(NULL, :P) IS NULL OR col IN (:P))`.
9. **Select All:** prefer **NULL Value Passed**.
10. **LOV SQL:** column 1 = label shown, column 2 = value passed. Show names, pass IDs.
11. **Cascading LOV:** child LOV uses parent bind + "Refresh other parameters on change" on the parent.
12. **Master–detail:** two data sets + **element link** (different column names) → nested XML. Alternatives: parent-column bind, or single query + Group By.
13. **Rename groups** (`G_1` → `G_EMP`) **before** building templates.
14. **Aggregates** at group level: Count, Sum, Avg, Max, Min, First, Last (`CS_` names).
15. **Calculations:** prefer SQL; data model expressions for group-level logic (`CF_` names).
16. **`:xdo_user_name`** = logged-in user. Use with `PER_USERS` for self-service and manager reports.
17. **Triggers:** Before Data, After Data (rare in SaaS), **Schedule** (decides whether a job runs).
18. **Properties:** Include Parameter Tags; **Include Empty Tags for Null Elements** (important for eText).
19. **View Data → Save As Sample Data.** Re-save when the structure changes.

---

## Practice Interview Questions with Answers

**Q1. What is the difference between a parameter and a bind variable?**
A parameter is the input defined in the data model that the user fills in when running the report. A bind variable is the reference to that input inside the SQL, written with a colon, like :P_DEPT_ID. The names must match, and BIP substitutes the parameter value at run time.

**Q2. How do you make a parameter optional?**
For single values: `(:P_DEPT_ID IS NULL OR column = :P_DEPT_ID)`. If the user leaves it blank, all rows are returned. For multi-select: `(COALESCE(NULL, :P_DEPT_ID) IS NULL OR column IN (:P_DEPT_ID))`.

**Q3. Why does `:P IS NULL` fail for a multi-select parameter?**
When multiple values are selected, BIP expands the bind into a comma-separated list, so `:P IS NULL` becomes something like `101, 102 IS NULL`, which is invalid SQL. COALESCE accepts multiple arguments, so `COALESCE(NULL, 101, 102)` stays valid.

**Q4. What's the difference between "NULL Value Passed" and "All Values Passed" in Can Select All?**
With NULL Value Passed, selecting All sends NULL and the SQL treats it as no filter. With All Values Passed, every LOV value is sent, which is slower for large lists and can exceed Oracle's 1000-item IN list limit. NULL Value Passed is preferred.

**Q5. How do you write an LOV query?**
With two columns: the first is the label the user sees and the second is the value passed to the parameter. For example, select the department name and the organization ID, so users choose by name but the SQL filters by ID.

**Q6. How do you create cascading parameters?**
The child LOV's query references the parent parameter as a bind variable. On the parent parameter, enable "Refresh other parameters on change", and place the parent before the child in the parameter list.

**Q7. How do you create a master–detail relationship in BIP?**
Create two data sets, then in the Diagram tab create an element-level link from the master's key column to the detail's key column. The linking columns must have different names. This produces nested XML with detail groups inside each master group. Alternatives are using the parent column as a bind variable in the child query, or a single query with Group By in the diagram.

**Q8. What happens to a master record that has no detail records?**
It still appears in the XML, without a child group. Element links behave like an outer join.

**Q9. Why should you rename groups before building templates?**
Templates loop on group names such as `for-each:G_EMP`. If a group is renamed after the template is built, the template no longer finds it and the output is blank.

**Q10. Where would you calculate a count of employees per department?**
Either in SQL with a group-by query, or in the data model by adding a Count aggregate at the department group level. The aggregate approach keeps the employee detail rows and adds the count to each department group.

**Q11. What is `:xdo_user_name` and how do you use it?**
It's a BIP system variable holding the username of the person running the report. Joined with PER_USERS, it gives the person ID, which can restrict data to the logged-in employee or to their direct reports.

**Q12. What is a schedule trigger?**
A SQL query attached to a scheduled job. When the job is due, BIP runs the query: if it returns rows, the job runs; if not, the job is skipped. For example, a new hires report that runs only when there are new hires.

**Q13. Why are Before/After Data triggers rarely used in Fusion HCM?**
They call PL/SQL functions in a database package, and in Fusion SaaS customers can't create custom database packages. So custom logic is usually done in SQL instead.

**Q14. Why do you save sample data?**
The layout editor and Template Builder for Word need sample XML to insert fields and preview output. It also confirms that the data structure is correct before template work starts. When the structure changes, the sample must be saved again.

**Q15. What does "Include Empty Tags for Null Elements" do and why does it matter?**
When enabled, a NULL value still produces an empty XML tag instead of omitting the tag. This matters for eText and fixed-format files, where a missing element can shift field positions or break the file layout.

---

**Next: Phase 4, Templates and Report Creation** (creating the report, RTF templates in depth, Excel templates, eText templates, output formats, sub-templates, and translations).