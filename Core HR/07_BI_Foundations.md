# BI Publisher for Oracle HCM — Phase 1: Foundations

> **Goal of this phase:** Be able to clearly explain *what* BI Publisher is, *why* a project uses it, *when* to choose it over OTBI or HCM Extracts, *how* it works internally, and *where* things live inside Oracle Fusion.

**Topics in this phase**
1. What BI Publisher is and why it is used
2. Typical business requirements
3. BIP vs OTBI vs HCM Extracts
4. BIP architecture
5. Navigation and the BI Catalog
6. Roles and access

---

## 1. What is BI Publisher?

### Simple definition
**BI Publisher (BIP)** is Oracle's **reporting and document-generation tool**. It takes data from a source (usually a SQL query on the Fusion database), converts that data into **XML**, and then applies a **template (layout)** to produce a **formatted output** like PDF, Excel, CSV, HTML, or a text file.

> **One line to remember:**
> **BI Publisher = Data (XML) + Template (Layout) → Formatted Output**

### Name history (interviewers sometimes ask this)
| Period | Name |
|---|---|
| Early (E-Business Suite days) | **XML Publisher** |
| Later | **BI Publisher (BIP)** |
| Current (Fusion, recent releases) | **Oracle Analytics Publisher** |

All three names refer to the same product. In projects and interviews, people still mostly say **"BIP"** or **"BI Publisher"**.

### Where it sits in Oracle Fusion HCM
- BI Publisher is **embedded** inside Oracle Fusion Applications. You don't install it separately.
- It reads data from the same Fusion database that HCM uses, through a data source called **`ApplicationDB_HCM`**.
- Many **seeded (Oracle-delivered) documents** in Fusion are themselves built on BIP, such as payslips, some letters, and several standard reports.

### Why do we use BI Publisher? (Core reasons)

| Reason | What it means technically |
|---|---|
| **Pixel-perfect formatting** | You control exact layout: fonts, logos, tables, page breaks, headers, footers. |
| **Separation of data and layout** | The SQL (data model) and the look (template) are separate objects. You can change one without touching the other. |
| **One data model, many layouts** | The same data model can feed a PDF letter, an Excel sheet, and a text file. |
| **Multiple output formats** | PDF, Excel, CSV, HTML, RTF, Text (fixed-width / delimited), XML. |
| **Full SQL flexibility** | You can query almost any table, write complex joins, sub-queries, and functions. |
| **Scheduling** | Reports can run automatically (daily, weekly, monthly). |
| **Bursting** | One report run can be split into many outputs and delivered to many people (e.g., one letter per employee). |
| **Delivery channels** | Output can go to email, SFTP/FTP, UCM (content server), or printer. |
| **High volume** | Handles large data sets better than interactive tools. |

> **Interview-ready line:**
> "We use BI Publisher when the requirement needs **formatted, pixel-perfect documents or structured files**, built with **custom SQL**, often **scheduled and delivered** automatically — something interactive tools like OTBI are not designed for."

---

## 2. Typical Business Requirements for BIP in HCM

When a client asks for a report, these are the kinds of requirements that point to BI Publisher:

### a) Formatted documents for employees
- Offer letters, promotion letters, salary revision letters, experience/relieving letters
- Employee profile documents
- Payslip-style documents

**Why BIP:** Needs company logo, fixed layout, signatures, page formatting → **RTF template**.

### b) Operational reports for HR teams
- Active employee list with department, job, location, manager
- New hires / terminations between two dates
- Headcount by department, location, or business unit
- Employees with missing data (no email, no manager, no bank details)

**Why BIP:** Complex SQL logic, parameters, grouping and totals → **RTF or Excel template**.

### c) Outbound interface files to third parties
- Bank payment files
- Benefits provider / insurance vendor files
- Files for external payroll vendors, background check companies, etc.

**Why BIP:** Vendor demands an exact file format (fixed-width or delimited, with header and trailer records) → **eText template**, delivered via **SFTP**, often **encrypted (PGP)**.

### d) Statutory / compliance reports
- Government or regulatory reports with a fixed format
- Audit reports (who changed what, when)

### e) Automated, scheduled distribution
- Weekly headcount report emailed to HR managers
- Monthly letter sent individually to each employee → **Bursting**

> **Quick rule to remember:**
> If the requirement says **"formatted"**, **"file"**, **"send to vendor"**, **"email to each employee"**, or **"complex logic not available in OTBI"** → think **BI Publisher**.

---

## 3. BI Publisher vs OTBI vs HCM Extracts ⭐

This is one of the **most frequently asked** interview questions. Learn each tool's identity first, then the comparison.

### OTBI — Oracle Transactional Business Intelligence
- A **self-service, ad hoc, real-time** reporting tool.
- Users **drag and drop** columns from **Subject Areas** (pre-built business views like "Workforce Management – Worker Assignment Real Time").
- **No SQL needed.** Business users can build their own analyses.
- **Security is applied automatically** through the BI semantic layer — users only see data their roles allow.
- Output: interactive tables, pivot tables, charts, **dashboards**.
- **Limitations:** limited to columns available in subject areas, limited formatting, not designed to create structured files or pixel-perfect documents.

### HCM Extracts
- An **HCM-specific tool for outbound data integration** (sending HCM data to other systems).
- Built using **Database Items (DBIs)** and **User Entities** — not by writing raw SQL.
- Filtering logic is written with **Fast Formulas** or criteria in the extract definition.
- Supports **Changes Only (delta) extraction** — sends only records that changed since the last run. **This is its biggest advantage.**
- Designed for **high-volume** data and runs as a **flow** (scheduled process).
- **Uses a BI Publisher template for output formatting** (e.g., an eText template for the final file). So Extracts and BIP work *together*.

### BI Publisher
- **SQL-based**, highly flexible, **pixel-perfect** reporting and file generation.
- Needs **technical skills** (SQL + template design).
- Can query almost any table, including areas not exposed in OTBI subject areas.
- **Security is NOT automatic** when you query base tables. The developer must handle data security (e.g., using **secured views** or security-based conditions).

### Comparison Table

| Point | **OTBI** | **BI Publisher** | **HCM Extracts** |
|---|---|---|---|
| Main purpose | Ad hoc analysis, dashboards | Formatted reports & files | Outbound data integration |
| Who builds it | Business users / functional | Technical developers | Technical / techno-functional |
| How data is fetched | Subject areas (drag & drop) | **SQL query** (data model) | DBIs / User Entities + Fast Formula |
| SQL needed? | No | **Yes** | No (mostly) |
| Formatting | Limited | **Pixel-perfect** | Uses **BIP template** for output |
| Output | Tables, charts, dashboards | PDF, Excel, CSV, Text, HTML | Mostly files (XML → formatted via BIP) |
| Data security | **Automatic** (role-based) | **Developer must handle it** | Built into extract framework |
| Changes-only (delta) | No | Not built-in (must code it) | **Yes, built-in** |
| Volume | Small–medium | Medium–large | **Very large** |
| Scheduling / delivery | Basic (agents) | **Yes** — schedule, burst, email, SFTP | **Yes** — as flows, with delivery |
| Best for | "Show me data quickly" | "Give me a formatted document/file" | "Send HCM data to another system regularly" |

### How to decide (simple decision path)
1. **Business user wants to explore data or see a dashboard?** → **OTBI**
2. **Need a formatted document, complex SQL, or a custom-format file?** → **BI Publisher**
3. **Need a regular, high-volume outbound integration, especially with changes-only?** → **HCM Extracts** (with a BIP template for the file format)

> **Interview-ready line:**
> "OTBI is for real-time self-service analysis using subject areas with automatic security. BI Publisher is for pixel-perfect reports and structured files using custom SQL, where the developer handles security. HCM Extracts is for high-volume outbound integrations using DBIs and supports changes-only extraction — and it uses a BI Publisher template to format its output."

### Extra point that impresses interviewers
- **BIP can use OTBI as a data source.** A data model can be built on an OTBI subject area instead of SQL. This is useful when you want OTBI's automatic security but BIP's formatting.
- **HCM Extracts depend on BIP** for output formatting, so knowing BIP templates helps with Extracts too.

---

## 4. BI Publisher Architecture

### The flow

```
 ┌──────────────────┐     ┌────────────────────┐     ┌──────────────────┐     ┌──────────────────┐
 │   DATA SOURCE    │ ──► │    DATA MODEL      │ ──► │   XML DATA       │ ──► │    TEMPLATE      │
 │ ApplicationDB_HCM│     │ (.xdm)             │     │ (generated at    │     │ (RTF / Excel /   │
 │ (Fusion database)│     │ SQL, parameters,   │     │  run time)       │     │  eText ...)      │
 └──────────────────┘     │ LOVs, groups       │     └──────────────────┘     └────────┬─────────┘
                          └────────────────────┘                                       │
                                                                                       ▼
                                                    ┌──────────────────────────────────────────────┐
                                                    │   REPORT (.xdo) = Data Model + Layout(s)     │
                                                    │   → OUTPUT: PDF / Excel / CSV / Text / HTML  │
                                                    │   → DELIVERY: Online / Email / SFTP / UCM    │
                                                    └──────────────────────────────────────────────┘
```

### The main building blocks

| Component | File / Object | What it does |
|---|---|---|
| **Data Source** | e.g., `ApplicationDB_HCM` | The database connection the data model queries. |
| **Data Model** | **`.xdm`** | Holds the SQL query (data sets), parameters, LOVs, links (master–detail), groups, bursting definitions. **Produces XML data.** |
| **Template / Layout** | `.rtf`, `.xls`, eText `.rtf`, etc. | Defines how the output looks. Reads the XML tags and places values in the layout. |
| **Report** | **`.xdo`** | The object users run. It **links one data model with one or more layouts** and stores report properties. |
| **Sub Template** | Catalog object | Reusable piece of layout (e.g., common company header) shared across reports. |
| **Style Template** | Catalog object | Common styles (fonts, table styles) applied across reports. |

### What happens when a report runs (step by step)
1. User runs the **Report (.xdo)** and enters **parameter** values.
2. The report calls its **Data Model (.xdm)**.
3. The **data engine** runs the SQL with the parameter values against `ApplicationDB_HCM`.
4. The result is converted into **XML** (structured as groups and elements).
5. The **layout/formatting engine** merges the XML with the selected **template**.
6. The final **output** (PDF, Excel, etc.) is generated.
7. The output is **shown online** or **delivered** (email, SFTP, UCM) — immediately or through a schedule.

### Why XML is in the middle
- XML is a **neutral, structured format**. Because data always becomes XML first, **any template type** can read it.
- This is what makes **"one data model → many layouts"** possible.
- When a report shows blank output, the first debugging step is to **check the XML** — if the data is in the XML, the problem is in the template; if not, the problem is in the data model/SQL.

> **Interview-ready line:**
> "BI Publisher separates data and presentation. The data model (.xdm) runs SQL against ApplicationDB_HCM and generates XML. The report (.xdo) links that data model to one or more templates. At run time, the engine merges the XML with the template to produce the output, which can be viewed online, scheduled, burst, or delivered."

---

## 5. Navigation and the BI Catalog

### How to open BI Publisher in Fusion
- **Navigator → Tools → Reports and Analytics → Browse Catalog**
  (This opens the BI Catalog, which holds both OTBI and BIP objects.)
- Or directly through the URL: `https://<your-fusion-pod>/analytics` (BI Catalog) or `https://<your-fusion-pod>/xmlpserver` (BI Publisher directly).

### What is the BI Catalog?
The **catalog** is the **folder-based storage** where all BI objects are saved — data models, reports, sub templates, style templates, and OTBI analyses/dashboards.

### Key folders

| Folder | What it contains | Can you edit it? |
|---|---|---|
| **My Folders** | Your personal objects. Only you can see them. | Yes (for personal testing). |
| **Shared Folders** | Objects shared across users. | Depends on folder. |
| **Shared Folders → Human Capital Management** (and other product folders) | **Seeded / Oracle-delivered** reports and data models. | **No — never edit these.** |
| **Shared Folders → Custom** | All **client-specific custom** reports and data models. | **Yes — this is where project work goes.** |

### Why we never edit seeded objects ⭐
- Oracle delivers **quarterly updates**. During an update, seeded content can be **overwritten**, and your changes are lost.
- Seeded objects are protected by Oracle's design.
- **Correct approach:** Use the **"Customize"** option (or copy) on a seeded report. This creates a copy in **`/Shared Folders/Custom`** under the same folder path. Make changes in the copy. Fusion then runs the customized version.

### Typical project folder structure (example)
```
/Shared Folders/Custom/
   └── HCM/
        ├── Data Models/
        │     └── XX_Active_Employees_DM.xdm
        └── Reports/
              └── XX_Active_Employees_Report.xdo
```
- Projects usually follow a **naming convention**, e.g., a client prefix like `XX_` or `<CLIENT>_`, and suffixes `_DM` for data models and `_RPT` for reports.
- Keep data models and reports in an organized folder structure so migration across environments is easy.

> **Interview-ready line:**
> "All custom development goes under /Shared Folders/Custom. We never modify seeded content because quarterly updates can overwrite it. To change a seeded report, we use the Customize option, which creates a copy in the Custom folder with the same path."

---

## 6. Roles and Access

### Main BI roles

| Role | What the user can do |
|---|---|
| **BI Consumer** | **Run and view** reports and analyses they have access to. |
| **BI Author** | **Create and edit** data models, reports, and analyses (plus everything BI Consumer can do). |
| **BI Administrator** | **Full control** — manage data sources, server settings, catalog permissions, scheduler, and all objects. |

- In Fusion, these BI roles are usually **inherited by job roles** (for example, an HCM technical/integration role may include BI Author capabilities).
- A **BIP developer** on a project typically needs **BI Author** (and sometimes BI Administrator in lower environments).

### Catalog permissions (object level)
Access is also controlled **per folder or per object** in the catalog:

| Permission | Meaning |
|---|---|
| **Read** | See the object. |
| **Run Publisher Report** | Run the BIP report. |
| **Schedule Publisher Report** | Schedule the report. |
| **View Publisher Output** | View outputs of scheduled runs. |
| **Write** | Edit the object. |
| **Delete** | Delete the object. |
| **Full Control** | All of the above, plus change permissions. |

### Important points to remember
- To **run** a report, the user needs permission on the **report** *and* on its **data model** (and the data source access behind it).
- Permissions are usually set at the **folder level** in `/Custom` and inherited by objects inside.
- **Data security is a separate concern:** having access to run a report does **not** automatically restrict *which employees' data* the user sees. In BIP, that is controlled by the **SQL** (e.g., using secured views). This links to Phase 2.

> **Interview-ready line:**
> "BI Consumer can run and view, BI Author can create and edit, and BI Administrator has full control. On top of roles, catalog permissions on folders and objects decide who can read, run, schedule, or edit a report. Data-level security in BIP is handled inside the SQL, not by these permissions."

---

## Quick Revision Sheet (read this before an interview)

1. **BIP formula:** Data (XML) + Template → Formatted Output.
2. **Names:** XML Publisher → BI Publisher → Oracle Analytics Publisher.
3. **Data source for HCM:** `ApplicationDB_HCM`.
4. **File types:** Data Model = **.xdm**, Report = **.xdo**.
5. **Key strength:** Separation of data and layout; one data model, many layouts.
6. **Outputs:** PDF, Excel, CSV, HTML, RTF, Text, XML.
7. **Delivery:** Online, Email, SFTP/FTP, UCM, Printer.
8. **OTBI:** self-service, subject areas, no SQL, automatic security, dashboards.
9. **BIP:** SQL-based, pixel-perfect, files and documents, developer handles security.
10. **HCM Extracts:** outbound integration, DBIs, Fast Formulas, **changes-only**, high volume, **uses BIP template** for output.
11. **Debug rule:** Check the XML first. Data in XML → template issue. No data in XML → SQL/data model issue.
12. **Custom work goes in:** `/Shared Folders/Custom`. Never edit seeded objects. Use **Customize**.
13. **Roles:** BI Consumer (run) → BI Author (create) → BI Administrator (full control).
14. **Run access:** needs permission on report **and** data model.

---

## Practice Interview Questions with Answers

**Q1. What is BI Publisher?**
BI Publisher is Oracle's reporting and document-generation tool. It runs a data model (usually SQL) to generate XML data, then merges that XML with a template such as RTF, Excel, or eText to produce formatted output like PDF, Excel, or text files. Output can be viewed online, scheduled, burst, or delivered via email, SFTP, or UCM.

**Q2. What is the difference between a data model and a report in BIP?**
The data model (.xdm) defines *what data* to fetch — SQL data sets, parameters, LOVs, links, and groups — and produces XML. The report (.xdo) defines *how to present it* — it links a data model to one or more layouts and holds report properties. One data model can be reused by multiple reports.

**Q3. When would you choose BIP over OTBI?**
When the requirement needs pixel-perfect formatting, a structured file format (like a bank file), complex SQL logic or tables not available in OTBI subject areas, or scheduled delivery and bursting. OTBI is better when business users need real-time, self-service analysis or dashboards.

**Q4. When would you choose HCM Extracts over BIP?**
For regular, high-volume outbound integrations to third-party systems, especially when only changed records should be sent (changes-only). HCM Extracts have built-in delta logic and run as flows, and they use a BIP template to format the final file.

**Q5. Is data security automatic in BI Publisher?**
No. When BIP queries base tables directly, role-based data security is not applied automatically. The developer must apply it, typically by using secured views or security conditions in the SQL. OTBI applies security automatically through its subject areas.

**Q6. Why should we not modify seeded reports?**
Seeded content can be overwritten during Oracle's quarterly updates, so changes would be lost. The correct method is to use the Customize option, which copies the report into /Shared Folders/Custom with the same path, and make changes there.

**Q7. A report runs but shows no data. How do you start debugging?**
First check the XML data generated by the data model. If the XML has data, the issue is in the template (for example, tag names or loop group names don't match). If the XML is empty, the issue is in the SQL or the parameter values passed.

**Q8. What roles are needed to create and run BIP reports?**
BI Author to create and edit data models and reports; BI Consumer to run and view them; BI Administrator for full control including data sources and permissions. Users also need catalog permissions on both the report and its data model.

**Q9. Can BI Publisher use OTBI as a data source?**
Yes. A data model can use an OTBI subject area as a data set instead of SQL. This gives OTBI's built-in security with BIP's formatting and delivery features.

**Q10. What types of outputs and delivery options does BIP support?**
Outputs: PDF, Excel, CSV, HTML, RTF, Text, and XML. Delivery: online view, email, FTP/SFTP, UCM (content server), and printer. It also supports scheduling and bursting.

---
