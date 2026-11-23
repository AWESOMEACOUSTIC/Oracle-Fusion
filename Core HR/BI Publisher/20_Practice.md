# 🧾 Mission Kaveri: The BI Publisher Interview Casebook
### One fictional project, every BIP topic (data source → data model → templates → bursting → scheduling → ESS), with the questions a 20-year veteran will ask at each step

> **How to use this casebook**
> 1. Read the **story** for each chapter. It tells you *why* the feature exists.
> 2. Read the **concepts** and **code**.
> 3. Cover the answers and attempt the **🎤 Interviewer questions** out loud. Each question is tagged with the *angle* it comes from, so you learn the different ways the same topic gets attacked.
> 4. Open the `Model answer` blocks only after you've tried.

---

## 🎭 Meet your interviewer

**Mr. Venkat Raman**: 20+ years in Oracle Applications. He started on EBS 11i with Oracle Reports, then XML Publisher, and now Fusion BIP. He has seen many production failures.

**How he asks:**

| His habit | What it sounds like | What he's testing | How to handle it |
|---|---|---|---|
| **Starts easy, then drills** | "What's a data model?" → "And?" → "And then?" | Whether you have depth or only a definition | Give the definition, then **one example**, then **one gotcha** |
| **"Why not…?"** | "Why use eText? Why not just CSV?" | Design judgment | Compare options and name the **trade-off** |
| **Production failure stories** | "Report ran fine yesterday. Today the email didn't go. Where do you look?" | Troubleshooting order | Answer **in sequence**, layer by layer |
| **Table/column grilling** | "Which table has the work email? Which column filters primary assignment?" | Hands-on experience | Know the key tables cold (see the cheat sheet at the end) |
| **EBS comparison** | "In EBS we had concurrent programs. What's the Fusion equivalent?" | Whether you understand concepts, not just screens | Map the concepts: concurrent program → ESS job |
| **"Have you done it, or read it?"** | "Tell me what *you* did when it failed." | Honesty | As a fresher: "In my training instance I…" is fine. **Never bluff.** |
| **Silence** | He just waits after your answer | Confidence | Don't ramble. Add **one** gotcha, then stop. |

> 💡 **Golden rule:** If you don't know something, say *"I haven't faced that, but here's how I'd approach it: …"* and reason from fundamentals. Veterans respect reasoning far more than memorized answers.

---

## 🏢 The scenario: Kaveri Consumer Goods Pvt Ltd *(fictional)*

Kaveri is going live on **Oracle Fusion HCM + Financials**. You're the **BI Publisher developer** on the project. Five stakeholders hand you requirements:

| # | Requester | Requirement | BIP features it forces you to learn |
|---|---|---|---|
| **R1** | HR Director | **Monthly Workforce Report**: active employees by department, with one **Excel sheet per department** | Data source, data model, HCM date-effective SQL, parameters, LOVs, **Excel template** |
| **R2** | Payroll Head | **Compensation Statement**: one PDF per employee, **emailed automatically** to each employee, in their language | **RTF template**, sub-templates, translations, **bursting**, **scheduling** |
| **R3** | Treasury | **Positive Pay file**: fixed-width text file of issued payments, sent **daily to the bank's SFTP** | **eText template**, FTP/SFTP delivery, schedule triggers |
| **R4** | AP Manager | **Open Invoices report** that AP users can filter and sort on screen, secured by BU | **XPT (Layout Editor)**, BIP security for Financials |
| **R5** | AP Team | Run R4 from **Scheduled Processes** like any other Fusion job, right after seeded **Create Accounting** | **ESS job definition**, **seeded jobs**, job sets |
| **R6** | Project Manager | Move everything **DEV → TEST → PROD** safely | Catalog, archive/unarchive, ACL |

---

# CHAPTER 1: BIP architecture and the catalog

### 📖 Story
Day 1. You log in to Fusion → **Reports and Analytics** → **Browse Catalog** (or go straight to `/xmlpserver`). Venkat's first question is about the building blocks.

### 🧠 Concepts

**BIP separates *data* from *layout*:**

```
DATA SOURCE ──► DATA MODEL ──► XML DATA ──► REPORT (+ LAYOUT TEMPLATES) ──► OUTPUT ──► DELIVERY
(where data    (SQL, params,   (the raw     (RTF / Excel / eText / XPT)     (PDF,     (online, email,
 lives)         LOVs, bursting) data tree)                                    XLSX,     SFTP, printer,
                                                                              TXT…)     content server)
```

| Catalog object | File extension | What it holds |
|---|---|---|
| **Data model** | `.xdm` | Data sets (SQL), parameters, LOVs, event triggers, bursting definitions |
| **Report** | `.xdo` | Link to one data model + one or more **layouts** + report properties |
| **Sub template** | `.xsb` | Reusable layout pieces (header, logo, address block) |
| **Style template** | `.xss` | Reusable styles (fonts, table formats) for RTF |

**Catalog folders:**
- `/Shared Folders/...` holds **Oracle's seeded content**. Don't change it, because updates can overwrite it.
- `/Shared Folders/Custom/...` holds **your content**. Always build here.
- `My Folders` is personal and **not** visible to others.

### 🎤 Interviewer questions

**Q1.1** *(Basics)* "What is BI Publisher, in one line?"
<details><summary>Model answer</summary>

A reporting tool that **separates data from layout**: the data model produces XML, and templates format that XML into PDF, Excel, text files and so on, which can be viewed online, scheduled, or delivered (email, SFTP, printer). One data model can feed many layouts.
</details>

**Q1.2** *(Why-not)* "Why separate data model and report? Why not one object?"
<details><summary>Model answer</summary>

- **Reuse**: one data model can feed several reports or layouts (e.g. PDF for managers, Excel for analysts, eText for the bank).
- **Different skills**: SQL developers own the data model; functional or business users can own layouts.
- **Maintenance**: change the layout without touching the SQL, or fix the SQL once for all layouts.
</details>

**Q1.3** *(Experience check)* "Where do you save your custom reports and why?"
<details><summary>Model answer</summary>

Under `/Shared Folders/Custom/<module>/...`. Seeded folders can be overwritten in quarterly updates, and `My Folders` isn't shared. Custom folders also make ACLs and migration cleaner.
</details>

**Q1.4** *(EBS comparison)* "How is this different from XML Publisher in EBS?"
<details><summary>Model answer</summary>

In EBS, the data came from a **data template or Oracle Reports (RDF)** run by a **concurrent program**, and templates were registered in the **XML Publisher Administrator** responsibility. In Fusion, the **data model is built inside BIP** (SQL data sets, parameters, bursting all in one place), templates are uploaded directly into the **catalog**, and the concurrent program's role is played by **ESS jobs** or the **BIP scheduler**. Same core idea: XML data + template.
</details>

**Q1.5** *(Curveball)* "Can a report have more than one data model?"
<details><summary>Model answer</summary>

No. A report points to **one** data model. But a data model can have **many data sets** (even from different data sources), and many reports can use the same data model.
</details>

---

# CHAPTER 2: Data source

### 📖 Story
You create a new data model for R1. The first thing BIP asks for is a **data source** for your SQL data set.

### 🧠 Concepts

**Data source** = *where* the data comes from. In Fusion SaaS you mostly use the predefined **JDBC connections**:

| Data source | Use for |
|---|---|
| `ApplicationDB_HCM` | HCM tables (PER_, PAY_, CMP_, HR_...) |
| `ApplicationDB_FSCM` | Financials, Procurement, SCM (AP_, AR_, GL_, PO_, POZ_, IBY_...) |
| `ApplicationDB_CRM` | Sales/CX tables |

You can't create your own database connections to the Fusion database. You use the ones Oracle provides. The **default data source** is set per data model.

**Other data set types:** SQL Query (90% of the time), **Oracle BI Analysis** (an OTBI analysis as the data source), Web Service, HTTP (XML feed), XML / Excel / CSV file, LDAP.

### 🎤 Interviewer questions

**Q2.1** *(Basics)* "What data source would you pick for an HCM report? For an AP report?"
<details><summary>Model answer</summary>

`ApplicationDB_HCM` for HCM and `ApplicationDB_FSCM` for AP. The tables are physically in one database, but the connection matters for **access**: some objects are only reachable, or behave correctly, through the matching connection.
</details>

**Q2.2** *(Scenario)* "Your report needs HR data and AP data together, e.g. employee expense reimbursements with the employee's department. How?"
<details><summary>Model answer</summary>

Use **two data sets in the same data model**, each with its own data source (HCM and FSCM), and **link them** on a common column (e.g. PERSON_ID). Alternatively, if one connection can see both sets of tables, a single SQL. I'd test which works and is faster.
</details>

**Q2.3** *(Why-not)* "Why would you ever use an OTBI analysis as a data set instead of SQL?"
<details><summary>Model answer</summary>

OTBI applies **Fusion security automatically** and reuses subject-area logic (no need to know the tables). It's good when a business user already has an analysis and wants a **pixel-perfect or bursted** version of it. The downsides are less control, usually slower, and limited to what the subject area exposes.
</details>

**Q2.4** *(Security link)* "Does the SQL run as the logged-in user?"
<details><summary>Model answer</summary>

No. It runs through the **data source connection**, so Fusion **data security isn't applied automatically**. For HCM I add **secured list views** (`PER_PERSON_SECURED_LIST_V`, `PER_ASSIGNMENT_SECURED_LIST_V`). For Financials I filter by the user's BU access through `FUN_USER_ROLE_DATA_ASGNMNTS` + `PER_USERS` + `:xdo_user_name`.
</details>

---

# CHAPTER 3: Data model (the heart of BIP)

### 📖 Story
R1 needs: **all active employees** of a selected **legal employer**, optionally filtered by **one or more departments**, **as of a date**, with department, job, hire date, work email and salary.

### 🧠 Concepts

**Data model components:**

| Component | Purpose |
|---|---|
| **Data sets** | The queries (SQL etc.). Each becomes a **group** in the XML |
| **Parameters** | User inputs: Text, Menu (from an LOV), Date |
| **List of values (LOV)** | The values behind Menu parameters (SQL or fixed list) |
| **Event triggers** | Before Data / After Data logic (PL/SQL in on-premise; limited in SaaS) |
| **Flexfields** | Lexical support for key flexfields (e.g. GL account segments) |
| **Bursting** | Split + delivery definitions (Chapter 11) |
| **Schedule triggers** | A query that decides whether a scheduled job should run (Chapter 12) |
| **Properties** | XML tag case, include empty tags, include parameter tags, etc. |
| **Structure** | Rename XML tags and display names; **group-level functions** (sum, count) |

**Parameter tricks:**
- **Bind variable** `:P_LE`: value passed safely. Use this almost always.
- **Lexical** `&P_WHERE`: text pasted into the SQL. Powerful but risky. Mainly used for flexfields and dynamic clauses.
- **Multi-select** menu with **"Can select all"**: choose whether "All" passes **NULL** or **all values**. With NULL, write:
  `AND (:P_DEPT IS NULL OR paam.organization_id IN (:P_DEPT))`. Some developers prefer `COALESCE(:P_DEPT, ...)`-style patterns. Test both single and multiple selections.
- **Cascading parameters**: the department LOV filters by the selected legal employer (`WHERE ... = :P_LE` inside the LOV SQL). Tick **"Refresh other parameters on change."**

### 💻 The R1 data model SQL

```sql
-- Data set: DS_EMP   (data source: ApplicationDB_HCM)
SELECT papf.person_id,
       papf.person_number,
       ppnf.display_name                         AS employee_name,
       paam.assignment_number,
       hou.name                                  AS department_name,
       pjf.name                                  AS job_name,
       ppos.date_start                           AS hire_date,
       pea.email_address                         AS work_email,
       cs.salary_amount
FROM   per_all_people_f           papf,
       per_person_names_f         ppnf,
       per_all_assignments_m      paam,
       per_periods_of_service     ppos,
       hr_organization_units_f_tl hou,
       per_jobs_f_vl              pjf,
       per_email_addresses        pea,
       cmp_salary                 cs
WHERE  papf.person_id            = ppnf.person_id
AND    ppnf.name_type            = 'GLOBAL'
AND    papf.person_id            = paam.person_id
AND    paam.assignment_type      = 'E'            -- employee assignments
AND    paam.primary_flag         = 'Y'            -- primary assignment only
AND    paam.assignment_status_type = 'ACTIVE'
AND    paam.effective_latest_change = 'Y'         -- _M table: last change on the day
AND    paam.period_of_service_id = ppos.period_of_service_id
AND    paam.organization_id      = hou.organization_id (+)
AND    hou.language (+)          = USERENV('LANG')
AND    paam.job_id               = pjf.job_id (+)
AND    papf.person_id            = pea.person_id (+)
AND    pea.email_type (+)        = 'W1'           -- work email
AND    paam.assignment_id        = cs.assignment_id (+)
-- date-effective filters (_F and _M tables)
AND    :P_AS_OF_DATE BETWEEN papf.effective_start_date AND papf.effective_end_date
AND    :P_AS_OF_DATE BETWEEN ppnf.effective_start_date AND ppnf.effective_end_date
AND    :P_AS_OF_DATE BETWEEN paam.effective_start_date AND paam.effective_end_date
AND    :P_AS_OF_DATE BETWEEN hou.effective_start_date (+) AND hou.effective_end_date (+)
AND    :P_AS_OF_DATE BETWEEN pjf.effective_start_date (+) AND pjf.effective_end_date (+)
AND    :P_AS_OF_DATE BETWEEN cs.date_from (+) AND cs.date_to (+)
-- parameters
AND    paam.legal_entity_id      = :P_LE
AND    (:P_DEPT IS NULL OR paam.organization_id IN (:P_DEPT))
-- security
AND    EXISTS (SELECT 1 FROM per_person_secured_list_v sec
               WHERE sec.person_id = papf.person_id)
ORDER  BY hou.name, ppnf.display_name
```

```sql
-- LOV: LOV_LE (legal employers)
SELECT hou.name, hou.organization_id
FROM   hr_organization_units_f_tl     hou,
       hr_org_unit_classifications_f  houc
WHERE  hou.organization_id   = houc.organization_id
AND    houc.classification_code = 'HCM_LEMP'
AND    hou.language          = USERENV('LANG')
AND    TRUNC(SYSDATE) BETWEEN hou.effective_start_date  AND hou.effective_end_date
AND    TRUNC(SYSDATE) BETWEEN houc.effective_start_date AND houc.effective_end_date
ORDER  BY hou.name
```

> ✅ Always verify column names in your own instance. The *patterns* above (date-effective filters, `_M` + `effective_latest_change`, `primary_flag`, `name_type = 'GLOBAL'`, `email_type = 'W1'`) are what interviewers check.

**Resulting XML (simplified):**
```xml
<DATA_DS>
  <P_LE>300000001234</P_LE>
  <G_EMP>
    <PERSON_NUMBER>10021</PERSON_NUMBER>
    <EMPLOYEE_NAME>Anita Rao</EMPLOYEE_NAME>
    <DEPARTMENT_NAME>Finance</DEPARTMENT_NAME>
    ...
  </G_EMP>
</DATA_DS>
```

### 🎤 Interviewer questions

**Q3.1** *(Basics)* "What are the components of a data model?"
<details><summary>Model answer</summary>

Data sets, parameters, LOVs, event triggers, flexfields, bursting definitions, schedule triggers, and properties. The structure tab lets you rename XML elements and add group-level aggregates.
</details>

**Q3.2** *(Drill: HCM tables)* "Why do you have `effective_latest_change = 'Y'` on assignments?"
<details><summary>Model answer</summary>

`PER_ALL_ASSIGNMENTS_M` is **multiple changes per day** enabled. Several rows can exist for the same date. `effective_latest_change = 'Y'` picks the **last change of that day**, so I don't get duplicates. `_F` tables allow one row per date range, so the date-between filter is enough there.
</details>

**Q3.3** *(Drill)* "Your employee count is double what HR expects. What went wrong?"
<details><summary>Model answer</summary>

Check in this order:
1. A **missing date-effective filter** on one of the `_F`/`_M` tables (multiple history rows).
2. **effective_latest_change** missing.
3. **primary_flag** missing (employees with secondary assignments).
4. **name_type** missing (GLOBAL + LOCAL names).
5. A **one-to-many join** (e.g. multiple emails, or several salary rows).
6. **assignment_type** not filtered (E, C, P, N... including pending workers).
</details>

**Q3.4** *(Bind vs lexical)* "Difference between `:P_DEPT` and `&P_DEPT`?"
<details><summary>Model answer</summary>

`:` is a **bind variable**: the value is passed safely, the SQL text stays the same, and the database can reuse the execution plan. `&` is **lexical**: the text is substituted into the SQL before it runs, so you can inject a whole clause. It's used for **flexfields** and dynamic WHERE/ORDER BY. It's risky (SQL injection) and prevents plan reuse, so I use lexicals only when necessary.
</details>

**Q3.5** *(Scenario)* "User selects 'All' in the department parameter and gets **no rows**. Why?"
<details><summary>Model answer</summary>

The multi-select parameter is set so that **"All" passes NULL**, but the SQL is `organization_id IN (:P_DEPT)`, and `IN (NULL)` matches nothing. Fix it with `(:P_DEPT IS NULL OR organization_id IN (:P_DEPT))`, or change the parameter to pass all values (slower for long lists).
</details>

**Q3.6** *(Scenario)* "HR wants the department list to show only departments of the legal employer they picked."
<details><summary>Model answer</summary>

**Cascading LOV**: the department LOV query filters on `:P_LE`. On the P_LE parameter, tick **Refresh other parameters on change**. Order the parameters so that P_LE comes first.
</details>

**Q3.7** *(Master–detail)* "You need each department with its total headcount at the top and the employees below. Do you write one query or two?"
<details><summary>Model answer</summary>

Either works:
- **Two data sets** (DS_DEPT and DS_EMP) **linked** on ORGANIZATION_ID → nested XML (G_DEPT containing G_EMP). Add a **group-level COUNT** on G_DEPT.
- Or **one query** and **regroup in the template** (`<?for-each-group:G_EMP;./DEPARTMENT_NAME?>`).

Linking gives cleaner XML for bursting and Excel sheet splits. A single query is often faster. I'd pick based on volume and template needs.
</details>

**Q3.8** *(Testing)* "How do you test a data model before building a template?"
<details><summary>Model answer</summary>

Use **View Data**, enter parameters, check the rows, then **Save as Sample Data** (needed by Template Builder and the layout editor). **Export** the XML to build the RTF offline. I also run the SQL separately and compare counts with HR's expectations.
</details>

**Q3.9** *(Curveball)* "Can you write PL/SQL or create a database function for your report in Fusion SaaS?"
<details><summary>Model answer</summary>

No. You can't create database objects in SaaS. Logic must go in the SQL itself (CASE, analytic functions, WITH clauses, inline views), in the template (XSL functions), or in data model features. That's why strong SQL skills matter so much in Fusion BIP.
</details>

**Q3.10** *(Curveball)* "Where are parameter values visible in the XML? Why would you want them there?"
<details><summary>Model answer</summary>

With **Include Parameter Tags** on, they appear at the top of the XML (e.g. `<P_LE>`). This is useful for printing "Report run for: Legal Employer X, As of 31-Oct-2026" in the header, and for bursting or file names.
</details>

---

# CHAPTER 4: The report object, layouts and output formats

### 📖 Story
Data is ready. Now you create the **report** that points to the data model, then add layouts.

### 🧠 Concepts

| Layout type | Built with | Best for | Output formats |
|---|---|---|---|
| **RTF** | MS Word + **BI Publisher Template Builder** add-in | Formatted documents: letters, statements, invoices, payslips | PDF, RTF, HTML, Excel, PPT… |
| **Excel template** | Excel (`.xls`) + Template Builder for Excel | Analyst reports, multi-sheet workbooks, Excel formulas | Excel |
| **eText** | RTF file containing **tables of rules** | **Flat files**: bank files, positive pay, government and benefit vendor files | Text |
| **XPT** (Layout Editor) | **Browser-based** drag-and-drop in BIP | Quick reports and **interactive** viewer output | Interactive, PDF, Excel, HTML… |
| PDF / XSL-FO / XSL | Acrobat forms / XSL code | Prefilled government forms / special cases | PDF / XML… |

**Important report properties:**
- **Output formats** allowed and **default** layout/format.
- **Run report online**: untick it for **heavy reports** so users must schedule them.
- **Enable bursting** + choose which bursting definition.
- **Translation and locale** handling.

### 🎤 Interviewer questions

**Q4.1** *(Basics)* "Which template types have you worked with, and when would you choose each?"
<details><summary>Model answer</summary>

- **RTF** for formatted documents like statements or letters.
- **Excel** when users need a workbook, multiple sheets or formulas.
- **eText** for strict fixed-width or delimited files for banks and vendors.
- **XPT** for quick interactive reports where users filter and sort on screen.

Then give the R1–R4 examples.
</details>

**Q4.2** *(Why-not)* "Why not just use RTF with Excel output instead of an Excel template?"
<details><summary>Model answer</summary>

RTF → Excel output is really an HTML-based conversion. The cells may not be true numbers or dates, formatting is fragile, and you can't easily create **multiple sheets** or keep **Excel formulas**. An Excel template gives real cells, named ranges, **sheet splitting per group** and native formulas.
</details>

**Q4.3** *(Scenario)* "One report, three audiences: managers want PDF, analysts want Excel, the bank wants a text file. How many reports?"
<details><summary>Model answer</summary>

Often **one report with three layouts** on the same data model: RTF→PDF, an Excel template, and eText. You choose the layout when running or scheduling, or per row in bursting through the TEMPLATE column. If security or audiences are very different, I'd split into separate reports for cleaner ACLs.
</details>

**Q4.4** *(Production)* "Users run a 2-lakh-row report online and it times out. What do you do?"
<details><summary>Model answer</summary>

Tune the SQL first. Then **untick "Run report online"** so it must be **scheduled**, prefer Excel or CSV output over PDF for big volumes, and deliver the output by email or SFTP instead of the browser. Also check whether users really need all rows (add mandatory parameters).
</details>

---

# CHAPTER 5: RTF templates (R2: Compensation Statement)

### 📖 Story
Payroll wants a branded **one-page Compensation Statement** per employee: company logo, employee details, salary components in a table, a total, a "Confidential" watermark, and page numbers.

### 🧠 Concepts: tags you must know

| Need | RTF tag |
|---|---|
| Print a field | `<?EMPLOYEE_NAME?>` |
| Loop over a group | `<?for-each:G_EMP?> … <?end for-each?>` |
| New page per employee | `<?for-each@section:G_EMP?>` (also resets page numbers per section) or `<?split-by-page-break:?>` |
| Condition | `<?if:SALARY_AMOUNT>100000?> … <?end if?>` |
| If / else-if / else | `<?choose:?><?when:STATUS='A'?>…<?end when?><?otherwise:?>…<?end otherwise?><?end choose?>` |
| Sort | `<?sort:EMPLOYEE_NAME?>` inside the for-each |
| Regroup flat data | `<?for-each-group:G_EMP;./DEPARTMENT_NAME?> … <?current-group()?>` |
| Totals | `<?sum(current-group()/SALARY_AMOUNT)?>`, `<?count(G_EMP)?>` |
| Format date | `<?format-date:HIRE_DATE;'DD-MON-YYYY'?>` |
| Format number | `<?format-number:SALARY_AMOUNT;'999G999D99'?>` |
| Variables | `<?xdoxslt:set_variable($_XDOCTX,'TOT',0)?>` / `<?xdoxslt:get_variable($_XDOCTX,'TOT')?>` |
| Header/footer | Word header/footer, or `<?start:body?> … <?end body?>` |
| Page X of Y | Word page fields (Template Builder supports them) |
| Call a sub-template | `<?import:xdoxsl:///Custom/Common/Kaveri_Header.xsb?>` then `<?call-template:KAVERI_HEADER?>` |

**Tags are case-sensitive** and must match XML element names exactly.

### 💻 R2 template skeleton (what you'd type into Word form fields)

```
<?import:xdoxsl:///Custom/Common/Kaveri_Header.xsb?>
<?for-each@section:G_EMP?>
   <?call-template:KAVERI_HEADER?>
   Employee: <?EMPLOYEE_NAME?>    Number: <?PERSON_NUMBER?>
   Department: <?DEPARTMENT_NAME?>  Hire date: <?format-date:HIRE_DATE;'DD-MON-YYYY'?>

   | Component            | Amount                                   |
   |<?for-each:G_COMP?>   |                                          |
   | <?COMPONENT_NAME?>   | <?format-number:AMOUNT;'999G999D99'?>    |
   |<?end for-each?>      |                                          |
   | Total                | <?format-number:sum(G_COMP/AMOUNT);'999G999D99'?> |

   <?if:count(G_COMP)=0?>No compensation components found for this period.<?end if?>
<?end for-each?>
```

### 🎤 Interviewer questions

**Q5.1** *(Basics)* "How do you build an RTF template? Walk me through it."
<details><summary>Model answer</summary>

Export **sample XML** from the data model → open Word with the **BI Publisher Template Builder** add-in → **load the XML** → insert fields and tables (wizard or manual form fields) → add loops, conditions and formatting → **preview** with sample data → upload the RTF to the report as a layout → set the default format → test from the report viewer.
</details>

**Q5.2** *(Drill)* "Difference between `for-each` and `for-each-group`?"
<details><summary>Model answer</summary>

`for-each` loops over **existing** XML groups. `for-each-group` **regroups flat data** in the template by a key (e.g. department) when the data model didn't nest it. Inside, `current-group()` gives that group's rows for nested loops and totals.
</details>

**Q5.3** *(Scenario)* "Page numbers should restart at 1 for each employee in a bursted batch PDF."
<details><summary>Model answer</summary>

Use `<?for-each@section:G_EMP?>`. The `@section` context makes each iteration a new section, which **resets page numbering** and repeats headers and footers per employee.
</details>

**Q5.4** *(Troubleshooting)* "Template preview works in Word, but the report shows blank fields on the server."
<details><summary>Model answer</summary>

1. **Tag name or case mismatch**: the server XML differs from the sample (renamed elements, the structure tab changed names, or upper/lower case).
2. **Wrong context**: a field placed outside its for-each loop, so the path doesn't resolve.
3. **Outdated sample data**: the data model changed after you exported the sample.
4. The real data actually is **empty** for those parameters.

Use **Get XML / View Data** on the server and compare it with the template paths.
</details>

**Q5.5** *(Scenario)* "Show salary in red if it's below the band minimum."
<details><summary>Model answer</summary>

**Conditional formatting**: wrap the field with `<?if:SALARY_AMOUNT < BAND_MIN?><xsl:attribute name="color">red</xsl:attribute><?end if?>` (Template Builder's conditional format wizard generates this), or use a choose block with two differently formatted copies of the field.
</details>

**Q5.6** *(Why-not)* "Why calculate totals in the template rather than in SQL, or the other way around?"
<details><summary>Model answer</summary>

- **SQL** for heavy aggregation on large data. It's faster and reusable across layouts, and needed when the same total feeds bursting or eText trailers.
- **Template** for presentation-specific totals (subtotals per group, running totals). It keeps the data model generic.

The rule of thumb: business logic in SQL, display logic in the template.
</details>

**Q5.7** *(Curveball)* "Payroll wants the statement in **Hindi** for some employees and English for others."
<details><summary>Model answer</summary>

Use template **translations**. Export the **XLIFF** translation file from the template, translate it, upload it with the Hindi locale, or upload a separate locale-specific RTF. With bursting, the **LOCALE** column per employee (e.g. `hi-IN` vs `en-US`) picks the right translation automatically. You also need fonts that support Devanagari in PDF output.
</details>

---

# CHAPTER 6: Sub-templates and reuse

### 📖 Story
Payroll, HR and AP each want the **same Kaveri header** (logo, address, CIN number) on all documents. Marketing changes the logo next quarter.

### 🧠 Concepts
- A **sub-template** is a catalog object (`.xsb`). An **RTF sub-template** holds named `<?template:KAVERI_HEADER?> … <?end template?>` blocks.
- Main templates **import** it with `<?import:xdoxsl:///Custom/Common/Kaveri_Header.xsb?>` and **call** a block with `<?call-template:KAVERI_HEADER?>`.
- **XSL sub-templates** hold reusable *logic* (functions or calculations).
- **Style templates** (`.xss`) standardize fonts and table styles.

### 🎤 Interviewer questions

**Q6.1** *(Design)* "Logo changes in 40 reports. How would you have designed this to make it a 5-minute job?"
<details><summary>Model answer</summary>

Put the header and logo in **one sub-template** that every report imports. Change the sub-template once, and all reports pick it up the next time they run. No need to edit 40 RTFs.
</details>

**Q6.2** *(Troubleshooting)* "After migration to PROD, reports fail with a sub-template error."
<details><summary>Model answer</summary>

The **sub-template wasn't migrated**, or it lives at a **different path**. The import path is absolute (`xdoxsl:///Custom/...`). The user may also lack **ACL access** to the sub-template folder. Migrate the sub-template along with the reports, and keep the folder path identical across environments.
</details>

---

# CHAPTER 7: Excel templates (R1: Workforce Report)

### 📖 Story
The HR Director wants an Excel workbook with **one sheet per department**, each sheet named after the department, with a headcount formula at the bottom.

### 🧠 Concepts
- Designed in **`.xls`** with the **BI Publisher Template Builder for Excel**.
- **Named cells/ranges** link to data:
  - `XDO_?EMPLOYEE_NAME?`: a cell bound to the field EMPLOYEE_NAME.
  - `XDO_GROUP_?G_EMP?`: a range that **repeats** for each G_EMP row.
- A hidden sheet called **`XDO_METADATA`** holds the header section and **Data Constraints**:

| Column A | Column B |
|---|---|
| Version | 1.0 |
| … | … |
| **Data Constraints:** | |
| `XDO_SHEET_?` | `<?.//G_DEPT?>` *(new sheet per department group)* |
| `XDO_SHEET_NAME_?` | `<?.//DEPARTMENT_NAME?>` *(sheet name)* |

- Native **Excel formulas** (e.g. `=COUNTA(...)`) survive, because the output is real Excel.
- Sheet splitting needs a **group to split on**, so DS_DEPT → DS_EMP linked (Chapter 3, Q3.7) makes this easy.

### 🎤 Interviewer questions

**Q7.1** *(Basics)* "How does an Excel template know which cell gets which field?"
<details><summary>Model answer</summary>

Through **defined names**: `XDO_?FIELD?` for single fields and `XDO_GROUP_?GROUP?` for repeating ranges, plus the `XDO_METADATA` sheet for advanced settings like sheet splitting.
</details>

**Q7.2** *(Scenario)* "One sheet per department, named after the department. How?"
<details><summary>Model answer</summary>

In `XDO_METADATA` → Data Constraints: `XDO_SHEET_?` = `<?.//G_DEPT?>` and `XDO_SHEET_NAME_?` = `<?.//DEPARTMENT_NAME?>`. The data must be grouped by department, either with linked data sets or with SQL ordered and grouped in the data model.
</details>

**Q7.3** *(Troubleshooting)* "Sheet generation fails for some departments."
<details><summary>Model answer</summary>

Excel sheet names have rules: **31 characters max**, no `/ \ ? * [ ] :`, and they must be **unique**. Department names like "R&D / Labs" or long names break it. Clean the name in SQL (SUBSTR, REPLACE) and make it unique, e.g. by appending the org ID.
</details>

**Q7.4** *(Why-not)* "Why not just CSV for HR?"
<details><summary>Model answer</summary>

CSV has no formatting, no multiple sheets, no formulas, and opens badly with leading zeros, dates and Unicode names. The director asked for a ready-to-use workbook. CSV is better for **system feeds** or huge data dumps.
</details>

---

# CHAPTER 8: eText templates (R3: Positive Pay file)

### 📖 Story
Treasury must send the bank a **Positive Pay** file every day: a list of payments issued, so the bank can reject tampered cheques. The bank's spec says:
- **Fixed width**, ASCII, one line per record.
- **Header**: record type `H`, company account number, file date.
- **Detail**: record type `D`, cheque number (10, zero-padded left), amount (12, implied decimals, zero-padded), issue date `YYYYMMDD`, payee name (40, space-padded right).
- **Trailer**: record type `T`, record count, total amount.

### 🧠 Concepts
An eText template is an **RTF file made of tables**:

**1. Format Setup table**

| Setting | Value |
|---|---|
| Template Type | `FIXED_POSITION_BASED` *(or `DELIMITER_BASED` for CSV/pipe files)* |
| Output Character Set | `iso-8859-1` / `utf-8` |
| New Record Character | `Carriage Return` / `Line Feed` |
| Case Conversion | `UPPER` |

**2. Data tables** (one per record type), each with columns: **Position, Length, Format, Pad, Data, Comments**

**Header record**

| Position | Length | Format | Pad | Data |
|---|---|---|---|---|
| `<LEVEL>` | | | | `DATA_DS` |
| `<NEW RECORD>` | | | | `HEADER_REC` |
| 1 | 1 | Alpha | | `'H'` |
| 2 | 15 | Alpha | R, ' ' | `BANK_ACCOUNT_NUM` |
| 17 | 8 | Date, YYYYMMDD | | `SYSDATE` |

**Detail record**

| Position | Length | Format | Pad | Data |
|---|---|---|---|---|
| `<LEVEL>` | | | | `G_PAY` |
| `<NEW RECORD>` | | | | `DETAIL_REC` |
| 1 | 1 | Alpha | | `'D'` |
| 2 | 10 | Number | L, '0' | `CHECK_NUMBER` |
| 12 | 12 | Number | L, '0' | `AMOUNT * 100` |
| 24 | 8 | Date, YYYYMMDD | | `CHECK_DATE` |
| 32 | 40 | Alpha | R, ' ' | `PAYEE_NAME` |
| `<END LEVEL>` | | | | `G_PAY` |

**Trailer record**

| Position | Length | Format | Pad | Data |
|---|---|---|---|---|
| `<NEW RECORD>` | | | | `TRAILER_REC` |
| 1 | 1 | Alpha | | `'T'` |
| 2 | 6 | Number | L, '0' | `COUNT(DETAIL_REC)` |
| 8 | 14 | Number | L, '0' | `SUM(G_PAY/AMOUNT) * 100` |
| `<END LEVEL>` | | | | `DATA_DS` |

Useful commands and functions: `<LEVEL>`, `<NEW RECORD>`, `<END LEVEL>`, `<DISPLAY CONDITION>`, `<DEFINE SEQUENCE>`, `<DEFINE CONCATENATION>`, `COUNT()`, `SUM()`, `SEQUENCE_NUMBER()`, `REPLACE()`, `SUBSTR()`, `NVL()`.

### 💻 R3 data model SQL (data source: ApplicationDB_FSCM)

```sql
SELECT aca.check_number,
       aca.amount,
       aca.check_date,
       aca.vendor_name     AS payee_name,
       aca.org_id
FROM   ap_checks_all aca
WHERE  aca.check_date     = :P_PAYMENT_DATE
AND    aca.void_date      IS NULL
AND    aca.status_lookup_code NOT IN ('VOIDED')
ORDER  BY aca.check_number
```

> 🧠 **Senior insight:** In a real Fusion Financials project, bank payment and positive pay files are usually produced through **Oracle Payments** (IBY): a custom eText template built on Oracle's seeded payment file data models, registered as a **Format**, and used in a **Payment Process Profile**. Our custom data model shows the mechanics. Mentioning the Payments route shows the interviewer you know how it's done in practice.

### 🎤 Interviewer questions

**Q8.1** *(Basics)* "What's eText and when do you use it?"
<details><summary>Model answer</summary>

An RTF-based template made of **rule tables** that produces **flat text files**, either **fixed-position** or **delimiter-based**. It's used when a bank, government body or vendor gives a strict file specification: positions, lengths, padding, header/detail/trailer records, counts and totals.
</details>

**Q8.2** *(Why-not)* "Why eText? Why not an RTF with text output, or a CSV?"
<details><summary>Model answer</summary>

RTF text output can't guarantee exact **positions, padding and line endings**. CSV can't do fixed width, can't easily mix record types, and can't produce trailer counts and totals reliably. eText is designed for **byte-exact** file specs.
</details>

**Q8.3** *(Drill)* "Explain LEVEL and NEW RECORD."
<details><summary>Model answer</summary>

**LEVEL** ties a table to an **XML group** and repeats once per instance of that group (e.g. `G_PAY` repeats per payment). **NEW RECORD** starts a new **record type** (a new line definition) inside that level. Header and trailer sit at the outer level (`DATA_DS`), and details at `G_PAY`.
</details>

**Q8.4** *(Scenario)* "Bank rejects the file: 'Amount field invalid'. What do you check?"
<details><summary>Model answer</summary>

1. **Implied decimals**: did I multiply by 100 and drop the decimal point?
2. **Padding direction and character**: left-padded zeros for numbers.
3. **Length overflow**: an amount longer than 12 digits.
4. **Negative amounts or signs**.
5. **Position shift** from an earlier field: one wrong length pushes every later field.
6. **Character set and line endings** (CRLF vs LF) as the bank expects.

Compare the file byte by byte against the spec, using a text editor with column numbers.
</details>

**Q8.5** *(Scenario)* "Bank wants the file **only if** there are payments. Empty files cause false alerts."
<details><summary>Model answer</summary>

Use a **schedule trigger** on the data model: a SQL query that returns rows only when payments exist for the date. The scheduled job then **skips** the run when there's no data (Chapter 12). Some teams also use a bursting query that returns no rows, so nothing is delivered.
</details>

**Q8.6** *(Curveball)* "Payee names have special characters (é, &, ñ). The bank only accepts uppercase ASCII."
<details><summary>Model answer</summary>

Set **Case Conversion = UPPER** and an ASCII-compatible **output character set** in Format Setup. Strip or replace unsupported characters in SQL (`TRANSLATE`, `REGEXP_REPLACE`) or with eText functions. Agree the replacement rules with the bank.
</details>

---

# CHAPTER 9: XPT layouts (R4: Open Invoices, interactive)

### 📖 Story
The AP Manager wants AP users to open a report, **filter by supplier, sort by due date, and see totals**, all on screen, without Excel.

### 🧠 Concepts
- **Layout Editor** is BIP's **browser-based** designer. It saves layouts as **`.xpt`**.
- Drag and drop **tables, charts, pivot tables, gauges** and text.
- **Interactive output**: users can **filter, sort and click charts to filter tables** in the viewer.
- Needs **sample data** saved in the data model.
- Limits: less control than RTF for complex document logic, and not suited for flat files.

### 💻 R4 SQL with Financials BU security

```sql
SELECT aia.invoice_num,
       hp.party_name          AS supplier_name,
       aia.invoice_date,
       aps.due_date,
       aps.amount_remaining,
       aia.invoice_currency_code,
       aia.org_id
FROM   ap_invoices_all          aia,
       ap_payment_schedules_all aps,
       poz_suppliers            ps,
       hz_parties               hp
WHERE  aia.invoice_id     = aps.invoice_id
AND    aia.vendor_id      = ps.vendor_id
AND    ps.party_id        = hp.party_id
AND    aps.amount_remaining <> 0
AND    aia.cancelled_date IS NULL
AND    aia.org_id IN (SELECT furda.org_id
                      FROM   fun_user_role_data_asgnmnts furda,
                             per_users                   pu
                      WHERE  furda.user_guid   = pu.user_guid
                      AND    furda.active_flag = 'Y'
                      AND    pu.username       = :xdo_user_name)
```

### 🎤 Interviewer questions

**Q9.1** *(Basics)* "What's XPT and who's it for?"
<details><summary>Model answer</summary>

A layout built in BIP's browser-based **Layout Editor**. It's fast to build, needs no Word add-in, and supports **interactive** output (filter, sort, linked charts). It suits business users and quick operational reports.
</details>

**Q9.2** *(Why-not)* "Why not just give AP users an OTBI analysis?"
<details><summary>Model answer</summary>

OTBI is often the better choice for pure self-service, with **automatic security**. I'd pick BIP XPT when the data isn't available in a subject area, needs **custom SQL logic**, or the same data model must also feed bursted or scheduled outputs. It's a trade-off I'd discuss with the client.
</details>

**Q9.3** *(Security)* "An AP user in the India BU sees US invoices. Why?"
<details><summary>Model answer</summary>

The SQL hits `AP_INVOICES_ALL` directly. BIP doesn't apply Fusion data security. Add the BU filter using `FUN_USER_ROLE_DATA_ASGNMNTS` + `PER_USERS` + `:xdo_user_name`. Also confirm the user's data access in **Manage Data Access for Users** is correct.
</details>

---

# CHAPTER 10: BIP security (quick recap)

| Layer | Controls | Where |
|---|---|---|
| Catalog **ACL** | Who can see, open, run or schedule the report, data model and sub-template | Permissions on folders/objects (Open, Modify, Full Control, No Access, Run/Schedule Publisher Report, Traverse) |
| **Data security in SQL** | Which rows return | HCM secured list views; ERP `FUN_USER_ROLE_DATA_ASGNMNTS` + `:xdo_user_name` |
| **Data source access** | Who can use the JDBC connection | BI Administration → data sources |
| **Scheduled/bursted runs** | Run with the **submitter's** security | Be careful when the scheduler is an admin |

### 🎤 Interviewer questions

**Q10.1** *(Scenario)* "An HR admin schedules the bursted Compensation Statement. Each employee gets only their own PDF. Is data security a concern here?"
<details><summary>Model answer</summary>

Yes, in a different way. The job runs with the **admin's** security, so the data set can see everyone, which is intended. **Correctness comes from bursting**: split by person and deliver to *that person's* email. The risks are a wrong email (personal vs work), duplicate emails, or a bad join that mixes rows. I'd validate the KEY/split logic, use **work email (`W1`)**, and test on a small population first.
</details>

**Q10.2** *(Production)* "Someone could see salary details in the Custom folder."
<details><summary>Model answer</summary>

Tighten the **ACL**: remove broad roles (e.g. BI Consumer / all authenticated users) from that folder, grant only the payroll role, and check that **Open** isn't inherited from the parent folder. Data model folders need protection too, because anyone who can open the data model can **View Data**.
</details>

---

# CHAPTER 11: Bursting (R2 + R3 delivery)

### 📖 Story
The Payroll Head: *"5,000 employees. Each must get **only their own** statement, by **email**, as a **PDF**, in their **language**. And save a copy."* Treasury: *"Push the positive pay file to the bank's **SFTP**."*

### 🧠 Concepts

**Bursting = run once, split the output into pieces, deliver each piece to its own destination.**

A bursting definition lives in the **data model** and has:
- **Split By**: the XML group that defines one "piece", e.g. `/DATA_DS/G_EMP`.
- **Deliver By**: the element whose value identifies the piece, e.g. `/DATA_DS/G_EMP/PERSON_ID`.
- **Bursting query**: a SQL that returns **one row per piece** with fixed column names:

| Column | Meaning |
|---|---|
| `KEY` | Must **match the Deliver By value** (e.g. PERSON_ID) |
| `TEMPLATE` | Layout name in the report |
| `LOCALE` | e.g. `en-US`, `hi-IN` |
| `OUTPUT_FORMAT` | `PDF`, `RTF`, `EXCEL`, `HTML`, `ETEXT`… |
| `DEL_CHANNEL` | `EMAIL`, `FTP`, `FILE`, `PRINT`, `FAX`, `WEBDAV`, content server |
| `PARAMETER1..10` | Channel-specific values (below) |
| `OUTPUT_NAME` | File name of the output |
| `SAVE_OUTPUT` | `true`/`false`: keep a copy in job history |
| `TIMEZONE`, `CALENDAR` | Optional |

**EMAIL channel parameters:** P1 = To, P2 = Cc, P3 = From, P4 = Subject, P5 = Body, P6 = Attachment (`true`), P7 = Reply-To, P8 = Bcc.

**FTP channel parameters:** P1 = server name (as registered in BI Administration), P2 = user, P3 = password, P4 = remote directory, P5 = remote file name, P6 = `true` for **SFTP**.

**Prerequisites:**
- Email and FTP servers configured in **BI Administration → Delivery**.
- **Enable bursting** in report properties and select the definition.
- Bursting runs only when **scheduled** (with the "use bursting definition" option), not when viewing online.

### 💻 R2 bursting query

```sql
SELECT papf.person_id                                   AS "KEY",
       'Compensation_Statement'                         AS template,
       CASE WHEN pul.language = 'HI' THEN 'hi-IN'
            ELSE 'en-US' END                            AS locale,
       'PDF'                                            AS output_format,
       'EMAIL'                                          AS del_channel,
       pea.email_address                                AS parameter1,   -- To
       NULL                                             AS parameter2,   -- Cc
       'payroll.noreply@kaveri.example'                 AS parameter3,   -- From
       'Your Compensation Statement - ' ||
         TO_CHAR(:P_AS_OF_DATE,'Mon YYYY')              AS parameter4,   -- Subject
       'Dear Employee, please find your statement attached.' AS parameter5, -- Body
       'true'                                           AS parameter6,   -- Attach
       'Compensation_' || papf.person_number            AS output_name,
       'true'                                           AS save_output
FROM   per_all_people_f    papf,
       per_email_addresses pea,
       per_user_language_v pul        -- illustrative: wherever you store preferred language
WHERE  papf.person_id = pea.person_id
AND    pea.email_type = 'W1'
AND    papf.person_id = pul.person_id (+)
AND    :P_AS_OF_DATE BETWEEN papf.effective_start_date AND papf.effective_end_date
```
*Split By:* `/DATA_DS/G_EMP` · *Deliver By:* `/DATA_DS/G_EMP/PERSON_ID`

> The language source above is illustrative. Use whatever your client stores for preferred language. The bursting **pattern** is what matters.

### 💻 R3 bursting query (SFTP)

```sql
SELECT 'POSPAY'                                           AS "KEY",
       'Positive_Pay_eText'                               AS template,
       'en-US'                                            AS locale,
       'ETEXT'                                            AS output_format,
       'FTP'                                              AS del_channel,
       'KAVERI_BANK_SFTP'                                 AS parameter1,  -- server (BI Admin)
       :P_SFTP_USER                                       AS parameter2,
       :P_SFTP_PWD                                        AS parameter3,
       '/inbound/pospay'                                  AS parameter4,
       'POSPAY_' || TO_CHAR(:P_PAYMENT_DATE,'YYYYMMDD') || '.txt' AS parameter5,
       'true'                                             AS parameter6,  -- SFTP
       'POSPAY_' || TO_CHAR(:P_PAYMENT_DATE,'YYYYMMDD')   AS output_name,
       'true'                                             AS save_output
FROM   dual
```
> ⚠️ Hard-coding credentials in SQL is a security risk. In interviews, mention that teams often deliver through a **scheduled job with a preconfigured FTP destination** in BI Administration, or hand the file to **OIC**, instead of putting passwords in the bursting query.

### 🎤 Interviewer questions

**Q11.1** *(Basics)* "What is bursting? Give a real example."
<details><summary>Model answer</summary>

Running a report **once** and **splitting** its output by a key, so each piece is formatted and **delivered separately**. Example: 5,000 compensation statements, one PDF per employee, emailed to each employee's work email in their own language, with a copy saved.
</details>

**Q11.2** *(Drill)* "Explain Split By and Deliver By. Can they be different?"
<details><summary>Model answer</summary>

**Split By** decides how the XML is cut into documents (one per group instance). **Deliver By** decides which delivery row (KEY) each document goes to. They can be at different levels. Example: **split by employee** but **deliver by department**, so each department manager gets **one email containing all their employees' statements**. KEY would then be the department ID.
</details>

**Q11.3** *(Production)* "Bursting job succeeded, but **nobody** received emails."
<details><summary>Model answer</summary>

1. **KEY doesn't match the Deliver By value** (data type, case, a number vs string, or a different column), so no document maps to any delivery row.
2. **Bursting not enabled** in report properties, or the schedule didn't tick **use bursting definition**.
3. **Email server** not configured or blocked (BI Administration → Delivery).
4. **PARAMETER1 empty** (no work email).
5. Emails landing in **spam**, or external domains blocked.

Check **Report Job History**: it shows **delivery status per destination**.
</details>

**Q11.4** *(Production)* "Some employees got **someone else's** statement."
<details><summary>Model answer</summary>

This is the most dangerous bursting bug. Usual causes: the data set and bursting query use **different keys or filters** (e.g. date-effective rows differ), a **one-to-many join** in the bursting query (multiple emails per person), or Split By is at the wrong level so one document contains several people. Fix the keys, make the bursting query return **exactly one row per KEY**, and test with a small population and internal emails before go-live.
</details>

**Q11.5** *(Scenario)* "Some employees have no work email. HR wants those PDFs to go to the HR mailbox instead."
<details><summary>Model answer</summary>

In the bursting query: `NVL(pea.email_address, 'hr.ops@kaveri.example') AS parameter1`. Optionally put the person number in the **subject** so HR knows who it's for. Or use a **CASE** to switch DEL_CHANNEL to FILE or FTP for those rows.
</details>

**Q11.6** *(Curveball)* "Can one bursting run send PDF to some people and Excel to others?"
<details><summary>Model answer</summary>

Yes. **TEMPLATE**, **OUTPUT_FORMAT**, **LOCALE** and **DEL_CHANNEL** are columns, so each row can differ (CASE on a person attribute). That's the power of bursting being query-driven.
</details>

**Q11.7** *(Why-not)* "Why burst? Why not schedule 5,000 separate jobs?"
<details><summary>Model answer</summary>

One bursting job **queries once**, scales far better, is easier to monitor in a single job history, and keeps everything consistent. 5,000 jobs would hit the database 5,000 times and flood the scheduler.
</details>

---

# CHAPTER 12: Scheduling (BIP scheduler)

### 📖 Story
- R2 runs **monthly** on the last working day.
- R3 runs **daily at 7 PM**, only if payments exist.
- R1 runs on the **1st of each month** for the previous month-end, emailed to the HR Director.

### 🧠 Concepts
From the catalog → report → **Schedule**:

| Tab | What you set |
|---|---|
| **General** | Parameter values (with **date expressions** for recurring runs) |
| **Output** | Layout, format, destinations (email, FTP, printer, content server), or **use bursting definition** |
| **Schedule** | Once, Hourly/Minutes, Daily, Weekly, Monthly, Annually, specific dates; start and end; optional **schedule trigger** |
| **Notification** | Email on success, warning, failure, skipped |

**Date expressions** for recurring parameters, so the job doesn't run with a hard-coded date forever. Examples:
`{$SYSDATE()$}`, `{$FIRST_DAY_OF_MONTH()$}`, `{$LAST_DAY_OF_MONTH()$}`, `{$FIRST_DAY_OF_YEAR()$}` (with +/− arithmetic in supported forms).

**Schedule trigger**: a SQL in the data model. When the schedule fires, BIP runs the trigger first. **If it returns no rows, the job is skipped.** This is perfect for the "no file if no payments" requirement.

**Monitoring:**
- **Report Jobs**: manage future and recurring schedules (pause, resume, edit, delete).
- **Report Job History**: status, output, **delivery status per destination**, errors, and republishing output.

### 🎤 Interviewer questions

**Q12.1** *(Basics)* "How do you schedule a report to run every day at 7 PM and email it?"
<details><summary>Model answer</summary>

Catalog → report → **Schedule** → General: parameters → Output: layout, format, add an **email destination** → Schedule: **Daily**, 7 PM, start date → Notification if needed → Submit. Then monitor in **Report Job History**.
</details>

**Q12.2** *(Scenario)* "The job runs daily, but it keeps reporting the **same date's** data."
<details><summary>Model answer</summary>

The date parameter was saved as a **fixed value** at scheduling time. Use a **date expression** like `{$SYSDATE()$}`, or make the SQL use `TRUNC(SYSDATE)` when the parameter is null.
</details>

**Q12.3** *(Drill)* "What's a schedule trigger and when do you use it?"
<details><summary>Model answer</summary>

A data model SQL that BIP evaluates **when the schedule fires**. If it returns rows, the job runs. If not, the job is **skipped**. Use it for "send only when there's data" (positive pay, exception reports) or "run when an upstream process has finished" (e.g. check a status table).
</details>

**Q12.4** *(Security)* "Whose security does a scheduled report use?"
<details><summary>Model answer</summary>

The **user who scheduled it** (the job owner). If an admin schedules a report and emails it to a wide list, recipients see whatever the admin can see. I'd schedule sensitive reports under an appropriate user, or rely on bursting so each recipient gets only their part.
</details>

**Q12.5** *(Production)* "The job owner left the company. What happens to their schedules?"
<details><summary>Model answer</summary>

When the user is deactivated or loses roles, their jobs may **fail** (no access) or run with the wrong security. Before offboarding, **reassign or recreate** critical schedules under a service or functional account and document schedule ownership. Good practice: don't own production schedules with a personal ID.
</details>

**Q12.6** *(Why-not)* "BIP scheduler or ESS job: which would you use for the AP report?"
<details><summary>Model answer</summary>

Use **ESS** when business users should run it from **Scheduled Processes** like any Fusion process, when it should sit in a **job set** after seeded jobs, when job-level privileges should control who can submit it, or when integrations (OIC, ERP Integration Service) must submit it. Use the **BIP scheduler** for simple recurring deliveries, bursting and catalog-centric reporting. (Chapter 13 has the details.)
</details>

---

# CHAPTER 13: ESS jobs and seeded jobs (R5)

### 📖 Story
AP users don't use the BI catalog. They live in **Navigator → Tools → Scheduled Processes**. The AP Manager wants the Open Invoices report **there**, and to run **right after** the seeded **Create Accounting** job.

### 🧠 Concepts

**ESS (Enterprise Scheduler Service)** is Fusion's job engine, the equivalent of **EBS concurrent programs**. Every process you submit from *Scheduled Processes* is an ESS job.

**Seeded (predefined) jobs**: Oracle-delivered ESS jobs. Examples:

| Area | Seeded job examples |
|---|---|
| Financials | Import Payables Invoices, Validate Payables Invoices, Create Accounting, Import AutoInvoice, Post Journals, Load Interface File for Import (FBDI) |
| HCM / Security | Refresh Manager Hierarchy, Synchronize Person Records, Send Pending LDAP Requests, Import User and Role Application Security Data, Autoprovision Roles for All Users |
| Reports | Many seeded reports (e.g. invoice registers, trial balances) are **BIP reports run through ESS jobs** |

**Registering a custom BIP report as an ESS job:**
1. Build and test the report in `/Shared Folders/Custom/Financials/AP/XX_AP_OPEN_INVOICES.xdo`.
2. Setup and Maintenance → **Manage Enterprise Scheduler Job Definitions and Job Sets for Financial, Supply Chain Management, and Related Applications** (there's a matching HCM task for HCM reports).
3. Create a job definition:
   - Display name, name, **application** (e.g. Payables)
   - **Job type: `BIPJobType`**
   - **Report ID**: the path to the `.xdo` under Shared Folders, e.g. `/Custom/Financials/AP/XX_AP_OPEN_INVOICES.xdo`
   - Default output format
   - Tick the option to allow submission from **Scheduled Processes**
4. Define **parameters** in the ESS job. **They're passed to the report's data model parameters in order (positionally)**, so the **sequence must match** the data model's parameter order.
5. Give the job's privilege to the right **role** (custom role) so AP users can submit it.
6. Optionally put it in a **job set** after Create Accounting.

**Customizing a seeded BIP report:**
- Don't edit the seeded report in its predefined folder.
- Use **Customize** on the seeded report. BIP creates a copy under `/Shared Folders/Custom/...` (same relative path), and the seeded ESS job **automatically runs your customized copy**. You can **edit the copy's layout**.
- Alternatively, **copy** it to Custom and register your own ESS job.

**Integration angle:**
- ESS jobs can be submitted externally (e.g. by **OIC**) through the **ERP Integration Service** (`submitESSJobRequest`) or the ERP integrations REST resource.
- BIP reports can be run synchronously by integrations through the **`ExternalReportWSSService`** (`runReport`), which returns the output (base64).

### 🎤 Interviewer questions

**Q13.1** *(Basics, EBS comparison)* "What's an ESS job? What was it in EBS?"
<details><summary>Model answer</summary>

ESS jobs are Fusion's **background processes**, submitted and monitored from **Scheduled Processes**. They're the equivalent of EBS **concurrent programs and request sets**. Job sets ≈ request sets.
</details>

**Q13.2** *(Drill)* "How do you register a BIP report as an ESS job? What are the key fields?"
<details><summary>Model answer</summary>

Setup and Maintenance → *Manage Enterprise Scheduler Job Definitions…* → new job → **Job Type = BIPJobType**, **Report ID = path to .xdo** (without `/Shared Folders`), application, output format, enable submission from Scheduled Processes → add **parameters in the same order** as the data model → grant the job privilege to a role.
</details>

**Q13.3** *(Production)* "ESS job runs, but the report ignores the parameter the user entered."
<details><summary>Model answer</summary>

ESS passes parameters **positionally**. The **order** in the ESS definition doesn't match the data model, or the **data types/date formats** differ (e.g. string vs date), so the report receives NULL or a value in the wrong parameter. Align order and types, then retest.
</details>

**Q13.4** *(Scenario)* "Client wants a small change to the seeded Payables Invoice Register layout. How do you do it without breaking upgrades?"
<details><summary>Model answer</summary>

Use **Customize** on the seeded report. It creates a copy in the **Custom** folder, and the seeded ESS job automatically uses it. Edit the copy's layout (download the RTF, change it, upload). The original stays untouched, so quarterly updates don't overwrite your change. After updates, compare with the new seeded version in case Oracle changed the data model.
</details>

**Q13.5** *(Scenario)* "Run the AP report automatically after Create Accounting finishes."
<details><summary>Model answer</summary>

Create a **job set**: step 1 Create Accounting (seeded), step 2 my ESS report job, running **serially** and only on success. Schedule the job set. Alternatively, an OIC flow can submit Create Accounting, poll for completion, then submit the report.
</details>

**Q13.6** *(Curveball)* "Can an ESS job's output reach the bank SFTP or an email?"
<details><summary>Model answer</summary>

Yes. When submitting or scheduling from Scheduled Processes, the **Advanced → Output** options let you add **destinations** (email, printer, FTP/content server, depending on configuration). For complex per-recipient delivery, use **bursting** or an **integration** (OIC fetching output through ExternalReportWSSService or UCM).
</details>

**Q13.7** *(Integration)* "OIC needs the Open Invoices data every night. How would it get it?"
<details><summary>Model answer</summary>

Two common patterns:
1. **Synchronous**: OIC calls **ExternalReportWSSService.runReport** with the report path and parameters and receives the output in the response. Good for small or medium data.
2. **Asynchronous**: OIC submits an **ESS job** through the ERP Integration Service, waits or polls for completion, then downloads the output (e.g. from UCM). Better for large volumes.
</details>

---

# CHAPTER 14: Migration and ACL (R6)

### 📖 Story
Everything works in DEV. Now TEST, then PROD, without breaking security.

### 🧠 Concepts
- **What to move:** data models, reports, sub-templates, style templates, plus **ESS job definitions** (moved separately through setup migration or recreated) and **role grants** for the job privileges.
- **How:**
  - **Archive** a folder (→ `.catalog` file) → **Unarchive** in the target.
  - Or **Download/Upload** single objects (`.xdoz` for reports, `.xdmz` for data models).
- **Keep permissions** at archive time.
- **Unarchive ACL option:**
  - **Inherit** (default): use the target parent folder's permissions.
  - **Preserve**: keep the original ACL, mapping existing accounts.
  - **Create**: like Preserve, but create missing accounts.
- **Keep the folder paths identical** across environments. ESS Report IDs and sub-template import paths depend on them.

### 🎤 Interviewer questions

**Q14.1** *(Basics)* "How do you migrate a BIP report from DEV to PROD?"
<details><summary>Model answer</summary>

Archive the custom folder (with **Keep permissions** if needed) → download the `.catalog` → in PROD, Unarchive into the same path with the right ACL option → migrate **sub-templates and data models** too → recreate or migrate any **ESS job definitions** and role grants → check data source names → **smoke test** with a business user's login.
</details>

**Q14.2** *(Production)* "After migration, the report runs for admins but ordinary users can't see it."
<details><summary>Model answer</summary>

It was unarchived with **Inherit**, so it took the target folder's ACL, which doesn't include the business role. Or the user's role lacks **Traverse** on a parent folder. Fix the ACL, or re-unarchive with **Preserve** (if archived with permissions). Grant to **roles**, not users.
</details>

**Q14.3** *(Production)* "After migration, the ESS job errors with 'report not found'."
<details><summary>Model answer</summary>

The **Report ID path** in the ESS job doesn't match where the report landed (a different folder or name), or the report wasn't migrated at all. Keep paths identical across environments.
</details>

---

# CHAPTER 15: Performance and troubleshooting master list

### 🧠 Performance tips
- Filter early: **mandatory parameters**, date ranges, indexed columns.
- Avoid `SELECT *`. Fetch only needed columns.
- Avoid functions on indexed columns in WHERE (e.g. `TRUNC(column) = …`).
- Use `EXISTS` instead of large `IN` lists. Use `WITH` clauses for reused subqueries.
- Make sure date-effective filters and `effective_latest_change` are present (fewer rows, no duplicates).
- Move heavy reports to **scheduled-only**. Prefer Excel or CSV for big outputs.
- Don't aggregate 2 lakh rows in RTF. Aggregate in SQL.
- Keep the XML small: remove unused columns from the data set.

### 🔧 Troubleshooting map

| Symptom | Look at |
|---|---|
| Wrong or duplicate rows | SQL: date-effective filters, primary flag, name type, one-to-many joins |
| No data | Parameters (NULL/"All" logic), security filters, date parameter format |
| Blank fields in output | Tag names or case vs XML, context in the template, outdated sample data |
| Template error on the server | Unsupported syntax, missing sub-template, bad XSL in form fields |
| Online timeout | SQL tuning, schedule-only, output format |
| Email not received | Bursting KEY, email server, PARAMETER1, job history delivery status |
| Wrong person's document | Bursting keys or joins, Split By level |
| SFTP failed | Server config in BI Admin, credentials, directory, firewall, SFTP flag (P6) |
| Job skipped | Schedule trigger returned no rows (intended?) |
| ESS ignores parameters | Parameter order and data types |
| Report missing after migration | ACL Inherit vs Preserve, Traverse, path changes |

### 🎤 Interviewer questions

**Q15.1** *(Production)* "A report that ran in 2 minutes now takes 40. Nothing changed, they say. What do you do?"
<details><summary>Model answer</summary>

"Nothing changed" usually means **the data grew** or **parameters changed** (e.g. users now pick "All"). Check:
1. Which parameters were used.
2. Data volume growth.
3. The SQL plan (missing filters, functions on indexed columns).
4. Whether a quarterly update changed underlying views.
5. Output format (PDF of a large volume is slow).

Tune the SQL, add mandatory filters, and move it to scheduled-only with Excel output if needed.
</details>

**Q15.2** *(Ownership)* "Tell me about a report you built end to end."
<details><summary>Model answer (adapt honestly to your training work)</summary>

Use the **STAR** structure with this casebook: *"In my training instance I built a workforce report for HR. Situation: HR needed active employees by department per month. Task: data model plus Excel output with a sheet per department. Action: I wrote date-effective SQL on PER_ALL_PEOPLE_F, PER_ALL_ASSIGNMENTS_M with effective_latest_change, primary flag, and secured list views, added cascading LOVs for legal employer and department, built an Excel template with XDO_METADATA sheet splitting, and scheduled it monthly with date expressions. Result: one workbook per month with no manual work, and I learned to handle duplicate rows from multiple assignment changes."* Say **"in training"** honestly. Depth matters more than whether it was a live project.
</details>

---

# 🔥 FINAL ROUND: Venkat's rapid fire

| # | Question | Crisp answer |
|---|---|---|
| 1 | Data model extension? | `.xdm` |
| 2 | Report extension? | `.xdo` |
| 3 | Sub-template extension? | `.xsb` |
| 4 | Where do custom reports go? | `/Shared Folders/Custom` |
| 5 | HCM data source? | `ApplicationDB_HCM` |
| 6 | Financials data source? | `ApplicationDB_FSCM` |
| 7 | Does BIP SQL apply Fusion data security? | No. Add secured views or BU filters |
| 8 | Current user variable? | `:xdo_user_name` |
| 9 | Bind vs lexical? | `:` safe value vs `&` text substitution |
| 10 | Why `effective_latest_change`? | `_M` tables can have several changes per day |
| 11 | Work email type? | `W1` in `PER_EMAIL_ADDRESSES` |
| 12 | Loop tag in RTF? | `<?for-each:G?>…<?end for-each?>` |
| 13 | Regroup flat data? | `<?for-each-group:G;./KEY?>` + `current-group()` |
| 14 | Restart page numbers per group? | `<?for-each@section:G?>` |
| 15 | Excel field naming? | `XDO_?FIELD?`, groups `XDO_GROUP_?G?` |
| 16 | Excel sheet split? | `XDO_SHEET_?` + `XDO_SHEET_NAME_?` in `XDO_METADATA` |
| 17 | eText types? | Fixed-position, delimiter-based |
| 18 | eText repeat per group? | `<LEVEL>` |
| 19 | XPT built in? | Browser Layout Editor; supports interactive output |
| 20 | Bursting key rule? | `KEY` must equal the Deliver By value |
| 21 | Email P1 / P4? | To / Subject |
| 22 | FTP P6 = true? | SFTP |
| 23 | Bursting runs online? | No. Only when scheduled with the bursting option |
| 24 | Skip job when no data? | Schedule trigger |
| 25 | Recurring date parameter? | Date expressions like `{$SYSDATE()$}` |
| 26 | Scheduled report security? | Job owner's (submitter's) |
| 27 | EBS concurrent program equivalent? | ESS job |
| 28 | ESS job type for BIP? | `BIPJobType` |
| 29 | ESS parameter mapping? | Positional: the order must match |
| 30 | Change a seeded report safely? | **Customize** → copy in Custom, seeded job uses it |
| 31 | Integration to run a BIP report? | `ExternalReportWSSService.runReport` |
| 32 | Integration to submit an ESS job? | ERP Integration Service `submitESSJobRequest` |
| 33 | Unarchive ACL default? | Inherit |
| 34 | Can't see report despite access? | Missing Traverse on parent folders / No Access override |
| 35 | Can see but can't run? | Missing Run Publisher Report / data model / data source access |

---

## 🗺️ One-page memory map

```
DATA SOURCE (ApplicationDB_HCM / FSCM)
   └─► DATA MODEL (.xdm)
         ├─ Data sets (SQL; date-effective; secured views; :xdo_user_name)
         ├─ Parameters (:bind, &lexical, multi-select NULL trick, cascading LOVs)
         ├─ LOVs
         ├─ Links + group functions (master-detail)
         ├─ Bursting (Split By, Deliver By, KEY, TEMPLATE, LOCALE, FORMAT, CHANNEL, P1..P10)
         └─ Schedule trigger (skip if no rows)
   └─► REPORT (.xdo)
         ├─ RTF   → documents (for-each, if/choose, group, @section, sub-templates, XLIFF)
         ├─ Excel → workbooks (XDO_?..?, XDO_GROUP_?..?, XDO_METADATA sheet split)
         ├─ eText → bank/vendor files (LEVEL, NEW RECORD, padding, COUNT/SUM trailer)
         └─ XPT   → interactive browser layouts
   └─► RUN
         ├─ Online (small)
         ├─ BIP Scheduler (date expressions, destinations, notifications, job history)
         └─ ESS job (BIPJobType, Report ID path, positional params, job sets, seeded jobs, Customize)
   └─► DELIVER: Email · SFTP · Printer · Content server · Integration (OIC)
   └─► MIGRATE: Archive/Unarchive (.catalog), Keep permissions, Inherit/Preserve/Create
```