# BI Publisher for Oracle HCM: Phase 4, Templates and Report Creation

> **Goal of this phase:** Turn the XML from your data model into real output: create the report (.xdo), design RTF, Excel and eText templates, choose output formats, and handle translations.

**Topics in this phase**
23. Creating the report and report properties
24. RTF templates in depth
25. Excel templates
26. eText templates
27. Output formats and defaults
28. Translations

**XML used in all examples** (from the Phase 3 data model):
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
      <SALARY>65000</SALARY>
    </G_EMP>
    <G_EMP> ... </G_EMP>
  </G_DEPT>
  <G_DEPT> ... </G_DEPT>
</DATA_DS>
```

---

## 23. Creating the Report and Report Properties

### What is a report?
A **report (.xdo)** is the object users actually **run**. It connects:
- **one data model** (where the data comes from), and
- **one or more layouts/templates** (how the output looks).

### Steps to create a report
1. Catalog → **New → Report**.
2. **Select the data model** (e.g., `/Custom/HCM/Data Models/XX_DEPT_EMP_DM`).
3. Choose how to add a layout:
   - **Use Report Editor**: go straight to the editor (most common on projects).
   - **Guide Me**: a step-by-step wizard for simple table reports.
4. In the report editor, click **Add New Layout**. Three options appear:

| Option | What it does |
|---|---|
| **Create Layout** | Opens BIP's **online Layout Editor** (browser-based, drag-and-drop). Saves as **.xpt**. Good for simple interactive reports. |
| **Upload or Generate Layout → Upload** | Upload a file you designed offline: **RTF**, **Excel**, **eText**, PDF, XSL. **Most common on HCM projects.** |
| **Upload or Generate Layout → Generate** | BIP auto-generates a basic RTF from the data structure, which you can then download and edit. |

5. When uploading, give the layout a **name**, choose the **template file**, the **type** (RTF Template, Excel Template, eText Template …), and the **locale**.
6. **Save** the report in `/Shared Folders/Custom/...` (e.g., `XX_DEPT_EMP_RPT`).

### Layout properties (List view in the report editor)
One report can have **several layouts** (e.g., a PDF letter and an Excel summary). For each layout you set:

| Property | Meaning |
|---|---|
| **Name** | Layout name the user selects |
| **Template File** | The uploaded file |
| **Type** | RTF, Excel, eText, XPT, etc. |
| **Output Formats** | Which formats users may choose (tick or untick) |
| **Default Format** | Format used when the user doesn't choose one |
| **Default Layout** | The layout used by default |
| **Active** | Whether the layout is available |
| **View Online** | Whether the layout can be viewed in the browser (untick for file-only layouts like eText) |

### Report properties (Properties button in the report editor)

| Tab | Important settings |
|---|---|
| **General** | **Run report online** (allow interactive run), **Auto Run** (run immediately when opened), **Show controls**, **Enable Bursting** (Phase 5) |
| **Caching** | Data and document caching (usually off for HCM data, so users always see current data) |
| **Formatting** | Output-specific settings: PDF security, Excel options, CSV delimiter, etc. |
| **Font Mapping** | Map template fonts to fonts available on the server (needed for some non-Latin languages) |
| **Currency Formats** | Currency-specific number formatting |

### Report parameters (Parameters button in the report editor)
- Controls **how data model parameters appear** to the user: show or hide each parameter, change display labels, set report-level default values.
- The parameter itself is still defined in the data model; the report only controls its display.

> **Interview-ready line:**
> "A report (.xdo) links a data model to one or more layouts. In the report editor I add layouts either by creating them in the online layout editor or, more commonly, by uploading RTF, Excel or eText templates designed offline. For each layout I set the allowed output formats, default format and default layout. Report properties control online running, caching, formatting, and enabling bursting."

---

## 24. RTF Templates in Depth ⭐

### What is an RTF template?
An **RTF template** is a **Microsoft Word document** with **BIP tags** placed where data should appear. Word handles the design (fonts, tables, logo, headers, footers), and the tags tell BIP which XML element to print.

- Saved as **.rtf** (Rich Text Format).
- Most widely used BIP template type: letters, formatted reports, multi-section documents.
- Output: PDF, RTF, HTML, Excel, PowerPoint.

### Template Builder for Word
A **Word add-in** that makes RTF template design easier.
- Download from BI Publisher: **Home → Download BI Publisher Tools / Desktop Tools**.
- After installing, Word shows a **BI Publisher** ribbon tab.

| Ribbon option | Use |
|---|---|
| **Load Data → Sample XML** | Load the sample XML saved from the data model (Phase 3) |
| **Insert → Field** | Insert a single element |
| **Insert → Table Wizard** | Build a table with a loop in a few steps |
| **Insert → Repeating Group** | Add a for-each loop (with optional grouping) |
| **Insert → Conditional Region / Conditional Format** | Show, hide or highlight based on a condition |
| **Insert → Chart / Pivot Table** | Charts and pivots from data |
| **Preview** | Preview as PDF, HTML, Excel, RTF using the sample XML |
| **Tools → Validate Template** | Check for syntax errors |
| **Tools → Field Browser** | See and edit all tags in the document in one list |
| **Tools → Translation** | Extract text for translation (Topic 28) |

### How tags are placed
Two ways, both work:
1. **Plain text tags**: type `<?FULL_NAME?>` directly in the document.
2. **Word form fields**: a small grey field that **displays** a short label (e.g., `FULL_NAME`) and holds the real tag in its **help text**. Template Builder uses this method. It keeps tables neat because long code is hidden.

### 1) Printing a value
```
Employee Name: <?FULL_NAME?>
```
- The name must **exactly match** the XML element, including **case** (`FULL_NAME`, not `full_name`).
- The value comes from the **current context**, meaning the group you are looping inside.

### 2) Loops: `for-each`
```
<?for-each:G_EMP?>
   <?PERSON_NUMBER?>  <?FULL_NAME?>
<?end for-each?>
```
- Repeats the content once **per `G_EMP` group** in the XML.

**Loop inside a table row (most common pattern)**

| Person No | Name | Job |
|---|---|---|
| `<?for-each:G_EMP?><?PERSON_NUMBER?>` | `<?FULL_NAME?>` | `<?JOB_NAME?><?end for-each?>` |

- Put **`for-each` in the first cell** and **`end for-each` in the last cell** of the same row. The whole row repeats for every employee.
- The header row stays outside the loop, so it prints once.

**Nested loops (master–detail)**
```
<?for-each:G_DEPT?>
Department: <?DEPARTMENT_NAME?>
   <?for-each:G_EMP?> <?FULL_NAME?> <?end for-each?>
<?end for-each?>
```
- Inner `G_EMP` is relative to the current `G_DEPT`, so each department prints **only its own** employees.

### 3) Navigating the XML path

| Syntax | Meaning |
|---|---|
| `<?FULL_NAME?>` | Element in the current group |
| `<?../DEPARTMENT_NAME?>` | Element in the **parent** group (from inside `G_EMP`) |
| `<?/DATA_DS/P_EFFECTIVE_DATE?>` | **Absolute** path from the root |
| `<?.//G_EMP?>` | Any `G_EMP` below the current node |

### 4) Conditions

**`if`: one condition**
```
<?if:SALARY > 50000?> High Earner <?end if?>
```

**`choose`: multiple conditions (like CASE / if-else)**
```
<?choose:?>
  <?when:SALARY > 100000?> Band A <?end when?>
  <?when:SALARY > 50000?>  Band B <?end when?>
  <?otherwise:?>           Band C <?end otherwise?>
<?end choose?>
```

**Inline if (single line)**
```
<?xdofx:if SALARY > 50000 then 'High' else 'Normal' end if?>
```

**Conditional row highlighting / hiding a column**
- Use **Insert → Conditional Format** in Template Builder. It generates code using `<?if@row:...?>` (applies to the whole table row) or `<?if@column:...?>` (shows or hides an entire column).

**Comparing text values:** strings go in **single quotes**: `<?if:JOB_NAME='Manager'?>`.

### 5) Sorting
```
<?for-each:G_EMP?><?sort:FULL_NAME;'ascending';data-type='text'?>
   ...
<?end for-each?>
```
- `sort` must come **immediately after** `for-each`.
- Use `data-type='number'` for numeric sorting. Otherwise `100` sorts before `20`.

### 6) Grouping in the template: `for-each-group`
Used when the XML is **flat** (no master–detail in the data model) but the output needs grouping.
```
<?for-each-group:G_EMP;./DEPARTMENT_NAME?>
   Department: <?DEPARTMENT_NAME?>
   <?for-each:current-group()?>
       <?FULL_NAME?>
   <?end for-each?>
<?end for-each-group?>
```
- Groups `G_EMP` rows by `DEPARTMENT_NAME`.
- **`current-group()`** = the rows belonging to the current department.

**Data model grouping vs template grouping:**

| Data model grouping (Phase 3) | Template grouping (`for-each-group`) |
|---|---|
| XML is already nested | XML is flat; template does the grouping |
| Faster for large data | Fine for small and medium data |
| Every layout benefits | Logic lives inside this one template |

### 7) Totals and calculations

| Requirement | Tag |
|---|---|
| Sum of all salaries | `<?sum(.//SALARY)?>` |
| Sum within current group | `<?sum(current-group()/SALARY)?>` (inside `for-each-group`) or `<?sum(G_EMP/SALARY)?>` (inside `G_DEPT`) |
| Count of employees | `<?count(G_EMP)?>` |
| Average | `<?sum(SALARY) div count(SALARY)?>` |
| Concatenate | `<?concat(FIRST_NAME,' ',LAST_NAME)?>` |
| Substring | `<?substring(PERSON_NUMBER,1,3)?>` |
| Upper case | `<?xdofx:upper(FULL_NAME)?>` |
| Value if null | `<?xdofx:nvl(JOB_NAME,'Not Assigned')?>` |

- XSL uses **`div`** for division, not `/` (because `/` means a path).

**Variables (e.g., running totals)**
```
<?xdoxslt:set_variable($_XDOCTX, 'v_total', 0)?>
<?for-each:G_EMP?>
   <?xdoxslt:set_variable($_XDOCTX, 'v_total', xdoxslt:get_variable($_XDOCTX,'v_total') + SALARY)?>
   Running total: <?xdoxslt:get_variable($_XDOCTX, 'v_total')?>
<?end for-each?>
```

### 8) Formatting dates and numbers ⭐
Dates arrive in XML in **canonical format** (`2026-06-01T00:00:00.000+05:30`). Format them in the template:

| Requirement | Tag | Output |
|---|---|---|
| Date | `<?format-date:HIRE_DATE;'DD-MON-YYYY'?>` | 01-JUN-2023 |
| Date | `<?format-date:HIRE_DATE;'DD/MM/YYYY'?>` | 01/06/2023 |
| Number with separators | `<?format-number:SALARY;'999G999D99'?>` | 65,000.00 |

- In number masks, **`G`** = group (thousands) separator and **`D`** = decimal separator. They adapt to the user's locale.
- Form fields also have a **Properties** dialog in Template Builder where you choose number or date formats without writing code.
- **Format-date only works if the XML date is in canonical format.** If you converted the date with `TO_CHAR` in SQL, it's now text and should be printed as is.

### 9) Page layout controls

| Requirement | How |
|---|---|
| **New page per record** (e.g., per department) | `<?split-by-page-break:?>` placed just before `<?end for-each?>` |
| **New section per record** (page numbering and headers restart, e.g., one letter per employee) | `<?for-each@section:G_EMP?>` ... `<?end for-each?>` |
| **Headers and footers** | Use normal Word header/footer; tags inside them work |
| **Page numbers** | Insert Word's page number field |
| **Content only on the last page** (e.g., signature block, grand total) | `<?start@last-page:body?>` ... `<?end body?>` |
| **Repeat table header on each page** | Word table property: *Repeat as header row at the top of each page* |
| **Watermark** (e.g., "CONFIDENTIAL", "DRAFT") | Word's built-in watermark feature |

### 10) Images
| Type | How |
|---|---|
| **Static logo** | Insert the image in Word directly |
| **Image from a URL** | Insert a placeholder image; in its **Alt Text** write `url:{IMAGE_URL}` (element holding the URL) or a fixed URL in quotes |
| **Image stored in DB (BLOB)** | Advanced: uses `fo:instream-foreign-object` with base64 data (awareness only) |

### 11) Printing parameter values
Two methods:
1. Enable **Include Parameter Tags** in the data model, then use `<?/DATA_DS/P_EFFECTIVE_DATE?>`.
2. Declare the parameter in the template: `<?param@begin:P_EFFECTIVE_DATE?>` at the top, then use `<?$P_EFFECTIVE_DATE?>`.

### 12) Sub-templates (reusable pieces)
Used when many reports share the same **header, footer, address block, or formatting logic**.

**Step 1:** Create an RTF with one or more named templates:
```
<?template:XX_COMPANY_HEADER?>
   [Company logo]  ABC Pvt Ltd · Chennai
<?end template?>
```
**Step 2:** Upload it to the catalog as a **Sub Template** (e.g., `/Custom/HCM/Common/XX_HEADER_SUB`).

**Step 3:** In each main template, import it and call it:
```
<?import:xdoxsl:///Custom/HCM/Common/XX_HEADER_SUB.xsb?>
<?call-template:XX_COMPANY_HEADER?>
```
- Change the logo **once** in the sub-template, and every report using it is updated.

### Complete RTF example: department-wise employee report

```
                       DEPARTMENT-WISE EMPLOYEE REPORT
                As of: <?format-date:/DATA_DS/P_EFFECTIVE_DATE;'DD-MON-YYYY'?>

<?for-each:G_DEPT?><?sort:DEPARTMENT_NAME?>
Department: <?DEPARTMENT_NAME?>          Total Employees: <?DEPT_EMP_COUNT?>

| Person No                                      | Name          | Job          | Hire Date                                              | Salary                                                         |
|------------------------------------------------|---------------|--------------|--------------------------------------------------------|----------------------------------------------------------------|
| <?for-each:G_EMP?><?sort:FULL_NAME?><?PERSON_NUMBER?> | <?FULL_NAME?> | <?JOB_NAME?> | <?format-date:HIRE_DATE;'DD-MON-YYYY'?>                | <?format-number:SALARY;'999G999D99'?><?end for-each?>          |
|                                                |               |              | Department Total:                                      | <?format-number:sum(G_EMP/SALARY);'999G999D99'?>               |

<?split-by-page-break:?>
<?end for-each?>
```
**What it does:**
- Prints the report date once at the top.
- Loops each department (sorted), one department per page.
- Inside each, loops the employees (sorted by name) as table rows.
- Formats dates and salaries; shows a department salary total.

### RTF troubleshooting ⭐

| Symptom | Likely cause |
|---|---|
| Field prints blank | Tag name or **case** doesn't match XML; wrong context (field outside its loop) |
| Only the first row prints | `for-each` missing, or placed on a different row than `end for-each` |
| Every department shows all employees | Inner loop uses an absolute path (`//G_EMP`) instead of a relative one (`G_EMP`) |
| Error when generating output | Unclosed `for-each`, `if`, or `choose`; run **Validate Template** |
| Date not formatting | Date in XML isn't canonical (converted to text in SQL) |
| Numbers sort wrongly | Missing `data-type='number'` in `sort` |

> **Interview-ready line:**
> "An RTF template is a Word document with BIP tags. I load sample XML into Template Builder for Word, insert fields, and use for-each loops (usually across a table row) to repeat data. I use if and choose for conditions, sort and for-each-group for ordering and grouping, sum and count for totals, and format-date and format-number for formatting. For page control I use split-by-page-break or for-each@section, and I put shared headers in sub-templates so they can be reused across reports."

---

## 25. Excel Templates

### What is an Excel template?
An **Excel workbook** where **cells are mapped to XML elements** using special **named ranges**. At run time, BIP fills the cells and repeats rows for each record.

### Why use an Excel template instead of RTF with Excel output?

| RTF template → Excel output | Excel template |
|---|---|
| Converted from a document layout; may produce merged cells and odd formatting | **Native Excel**: proper rows and columns |
| No Excel formulas | **Excel formulas work** (SUM, VLOOKUP, etc.) |
| One sheet | **Multiple sheets**, including one sheet per group (e.g., per department) |
| Good enough for simple tables | Best when users will **filter, sort, or calculate** in Excel |

### Key rules
- Save the template as **Excel 97-2003 Workbook (.xls)**.
- Mapping is done through **named ranges** (Formulas → Name Manager in Excel).
- **Template Builder for Excel** (another add-in from BI Publisher Desktop Tools) can create these names automatically.

### Naming conventions ⭐

| Named range | Meaning |
|---|---|
| **`XDO_?FULL_NAME?`** | This cell shows the `FULL_NAME` element |
| **`XDO_GROUP_?G_EMP?`** | This range (usually a full row) **repeats** for every `G_EMP` |

**Example layout**

| | A | B | C |
|---|---|---|---|
| 1 | **Person No** | **Name** | **Job** |
| 2 | `1001` *(named `XDO_?PERSON_NUMBER?`)* | `Anita` *(named `XDO_?FULL_NAME?`)* | `Accountant` *(named `XDO_?JOB_NAME?`)* |

- Name **row 2 (A2:C2)** as `XDO_GROUP_?G_EMP?`, and row 2 repeats for every employee.
- Sample values in the cells are just placeholders; they are replaced at run time.

### The `XDO_METADATA` sheet
Every Excel template needs a sheet named **`XDO_METADATA`** (usually hidden). It has:

**a) Header section (required)**

| A | B |
|---|---|
| Version | |
| Extractor Version | |
| Template Code | |
| **Template Type** | **TYPE_EXCEL_TEMPLATE** |
| Preprocess XSLT File | |
| Last Modified Date | |
| Last Modified By | |

**b) Data Constraints section (optional, for extra logic)**
Here you apply XSL logic to a mapped cell:

| A | B |
|---|---|
| **Data Constraints:** | |
| `XDO_?FULL_NAME?` | `<?concat(FULL_NAME,' (',PERSON_NUMBER,')')?>` |
| `XDO_?HIRE_DATE?` | `<?format-date:HIRE_DATE;'DD-MON-YYYY'?>` |

### Multiple sheets: one per group
Add these in the **Data Constraints** section:

| A | B |
|---|---|
| `XDO_SHEET_?` | `<?.//G_DEPT?>` |
| `XDO_SHEET_NAME_?` | `<?DEPARTMENT_NAME?>` |

- Creates **one sheet per department**, named after the department.

### Excel formulas
- Normal Excel formulas keep working. For a total below the repeating row, use `=SUM(C2:C2)`. Because BIP inserts rows inside the range, the formula range **expands automatically**.

> **Interview-ready line:**
> "An Excel template is a .xls workbook where cells are mapped to XML elements using named ranges: XDO_?ELEMENT? for single fields and XDO_GROUP_?GROUP? for repeating rows. It needs an XDO_METADATA sheet with the template type set to TYPE_EXCEL_TEMPLATE, and a Data Constraints section for extra logic, like XDO_SHEET_? to create one sheet per group. I use it over RTF when users need native Excel with formulas or multiple sheets."

---

## 26. eText Templates ⭐

### What is an eText template?
An **eText template** produces **plain text files** in an **exact format** required by external systems: **bank payment files, benefits vendor files, payroll interface files, government files**.

- Built in an **RTF (Word) document**, but it contains **no visual design**, only **tables of commands**.
- Output is a **text file** (`.txt`, `.csv`, `.dat`).
- **HCM Extracts** commonly use eText templates for their output files.

### Two types

| Type | Description | Example line |
|---|---|---|
| **FIXED_POSITION_BASED** | Each field has a **fixed start position and length**; padded with spaces or zeros | `1001      ANITA RAO           0000065000` |
| **DELIMITER_BASED** | Fields separated by a **delimiter** (comma, pipe, tab) | `1001\|Anita Rao\|65000` |

### Structure of an eText template
An eText RTF contains:
1. **One setup table**: global file settings
2. **One or more format tables**: one per **record type** (header, detail, trailer)

### 1) Setup table (global commands)

| Command | Example value | Meaning |
|---|---|---|
| **TEMPLATE TYPE** | `FIXED_POSITION_BASED` or `DELIMITER_BASED` | Type of file |
| **OUTPUT CHARACTER SET** | `utf-8` | Encoding of the file |
| **NEW RECORD CHARACTER** | `Carriage Return`, `Linefeed` | Line ending between records |
| **CASE CONVERSION** | `UPPER` | Convert all text to upper case |
| **DEFINE SEQUENCE** | (block) | Generates running sequence numbers |
| **DEFINE LEVEL** | (block) | Creates a custom level from XML (advanced) |
| **INVALID CHARACTERS / REPLACEMENT CHARACTERS** | | Remove or replace characters the vendor doesn't accept |

### 2) Format tables (record definitions)

Each table starts with:
- **`<LEVEL>`**: the **XML group** to loop over (e.g., `DATA_DS` for once per file, `G_EMP` for once per employee)
- **`<NEW RECORD>`**: a name for this record type (e.g., `HEADER_REC`)

**Columns in a fixed-position format table:**

| Column | Meaning |
|---|---|
| **`<POSITION>`** | Start position of the field (1, 11, 31 …) |
| **`<LENGTH>`** | Field length |
| **`<FORMAT>`** | Data type and mask: `Alpha`, `Number`, `Date, YYYYMMDD` |
| **`<PAD>`** | Padding: `L, '0'` (zeros on the left) or `R, ' '` (spaces on the right) |
| **`<DATA>`** | XML element, a constant in quotes, or an expression |
| **`<COMMENTS>`** | Notes for developers (not in output) |

**Delimiter-based tables** use `<MAXIMUM LENGTH>`, `<FORMAT>`, `<DATA>`, and `<COMMENTS>`. The delimiter (e.g., `'|'`) is placed as a constant value **between** the data fields.

### Example: fixed-position employee file
**Requirement:**
- Header record: record type `H`, file date `YYYYMMDD`
- Detail record per employee: type `D`, person number (10, zero-padded), name (30, space-padded), salary (12, zero-padded, no decimals)
- Trailer record: type `T`, total record count, total salary

**Setup table**

| Command | Value |
|---|---|
| TEMPLATE TYPE | FIXED_POSITION_BASED |
| OUTPUT CHARACTER SET | utf-8 |
| NEW RECORD CHARACTER | Carriage Return |
| CASE CONVERSION | UPPER |

**Header**

| `<LEVEL>` | DATA_DS | | | | |
|---|---|---|---|---|---|
| **`<POSITION>`** | **`<LENGTH>`** | **`<FORMAT>`** | **`<PAD>`** | **`<DATA>`** | **`<COMMENTS>`** |
| `<NEW RECORD>` | HEADER_REC | | | | |
| 1 | 1 | Alpha | R, ' ' | 'H' | Record type |
| 2 | 8 | Date, YYYYMMDD | | SYSDATE | File date |

**Detail**

| `<LEVEL>` | G_EMP | | | | |
|---|---|---|---|---|---|
| **`<POSITION>`** | **`<LENGTH>`** | **`<FORMAT>`** | **`<PAD>`** | **`<DATA>`** | **`<COMMENTS>`** |
| `<NEW RECORD>` | DETAIL_REC | | | | |
| 1 | 1 | Alpha | R, ' ' | 'D' | Record type |
| 2 | 10 | Alpha | L, '0' | PERSON_NUMBER | Person number |
| 12 | 30 | Alpha | R, ' ' | FULL_NAME | Name |
| 42 | 12 | Number | L, '0' | SALARY | Salary |

**Trailer** (at `DATA_DS` level, after the details)

| `<LEVEL>` | DATA_DS | | | | |
|---|---|---|---|---|---|
| `<NEW RECORD>` | TRAILER_REC | | | | |
| 1 | 1 | Alpha | R, ' ' | 'T' | Record type |
| 2 | 8 | Number | L, '0' | COUNT(DETAIL_REC) | Number of detail records |
| 10 | 15 | Number | L, '0' | SUM(SALARY) | Total salary |

**Sample output**
```
H20261108
D0000001001ANITA RAO                     000000065000
D0000001002RAVI KUMAR                    000000052000
T00000002000000000117000
```
*(The exact function syntax for counts and sums can vary; check Oracle's eText reference for your release and look at existing eText templates on the project.)*

### Common eText functions and expressions
- **Aggregates:** `SUM`, `COUNT`, `MIN`, `MAX` (used in trailer records)
- **Text and null handling:** `SUBSTR`, `NVL`, `REPLACE`, `LENGTH`
- **Logic:** `IF ... THEN ... ELSE ... END IF`
- **Dates:** `SYSDATE`, `TO_DATE`
- **Sequences:** `SEQUENCE_NUMBER` (defined through `DEFINE SEQUENCE`)

### eText tips ⭐
- Get the vendor's **file specification** (field positions, lengths, formats, padding) before you start. The template is a direct translation of that spec.
- Enable **Include Empty Tags for Null Elements** in the data model, so missing values still produce padded blanks and **positions don't shift**.
- Do heavy logic (formatting, codes, date conversion) **in SQL**. Keep eText for positioning.
- In layout properties, untick **View Online**. eText output is a file, not something to view in the browser.
- Files often go to **SFTP** with **PGP encryption** (Phase 5).

> **Interview-ready line:**
> "eText templates generate plain text files in exact formats for banks and vendors. They're built in RTF but contain only command tables: a setup table with the template type (fixed-position or delimiter-based), character set and record separator, and format tables, one per record type, each starting with a LEVEL tied to an XML group. In fixed-position tables I define position, length, format, padding and data for every field, and I build header and trailer records with counts and sums. HCM Extracts also use eText templates for their outbound files."

---

## 27. Output Formats and Defaults

### Which template produces which output

| Template type | Output formats |
|---|---|
| **RTF** | PDF, RTF, HTML, Excel, PowerPoint |
| **Excel template** | Excel only |
| **eText** | Text (eText) only |
| **Layout Editor (.xpt)** | Interactive, HTML, PDF, Excel, CSV, RTF, PowerPoint |
| **Any layout** | **Data (XML)** and **Data (CSV)**: raw data output, no layout |

### Choosing the right output

| Requirement | Best output |
|---|---|
| Letters, official documents, anything printed or signed | **PDF** (layout can't be edited) |
| Data users will filter, sort, or analyse | **Excel** |
| Raw data for loading into another system | **CSV** or **Data (CSV)** |
| Interface files to banks or vendors | **eText (Text)** |
| Quick on-screen view | **HTML** or **Interactive** |
| Editable document | **RTF** |

### Controlling formats
- In **layout properties**: tick only the formats users should have, and set a **Default Format**.
- When a report has **multiple layouts**, set one as the **Default Layout**.
- **Data (XML)** is very useful for **debugging**. You can see exactly what the template receives.

> **Interview-ready line:**
> "RTF templates can produce PDF, RTF, HTML, Excel and PowerPoint; Excel templates produce Excel; eText produces text files; and any layout can output raw data as XML or CSV. I choose PDF for formal documents, Excel for analysis, CSV for data loads and eText for interface files, and I restrict and default the formats in the layout properties."

---

## 28. Translations

Two different things get translated:

### a) Data translation
- Names stored in HCM (job, department, grade) are translated through **`_TL` tables** and `USERENV('LANG')` (Phase 2).
- **Nothing to do in the template.**

### b) Template text translation
The fixed text in the template (titles, column headings, labels like "Department:").

| Method | How it works | When to use |
|---|---|---|
| **XLIFF translation** | Extract the template's text into an **XLIFF file** (Template Builder → Tools → Translation, or from the catalog), translate it, and upload it for each **locale**. BIP uses one template plus language files. | Same layout in many languages. Layout changes only need doing once. |
| **Localized templates** | Create a **separate RTF per language** and upload each one with its locale. | When the layout itself differs by country (e.g., different letter formats). |

- BIP picks the translation based on the **user's report language/locale** preference.
- With XLIFF, if no translation exists for a locale, the **base template** is used.

> **Interview-ready line:**
> "Data translation is handled by the _TL tables in SQL. For template text, I either use XLIFF, where the text is extracted, translated and uploaded per locale against one template, or localized templates, with a separate RTF per language when the layout differs. BIP chooses the version based on the user's locale."

---

## Quick Revision Sheet

1. **Report (.xdo)** = data model + one or more layouts. Add layout: **Create** (online, .xpt), **Upload** (RTF, Excel, eText), **Generate** (auto RTF).
2. **Layout properties:** output formats, default format, default layout, active, view online.
3. **RTF** = Word + BIP tags. Template Builder for Word: Load XML, Insert Field, Table Wizard, Preview, Validate.
4. **Value:** `<?ELEMENT?>` (case-sensitive). **Parent:** `<?../ELEMENT?>`. **Absolute:** `<?/DATA_DS/ELEMENT?>`.
5. **Loop:** `<?for-each:G_EMP?>` … `<?end for-each?>`. In tables: first cell / last cell of the same row.
6. **Conditions:** `if`, `choose/when/otherwise`, inline `xdofx:if`. Row and column conditions: `if@row`, `if@column`.
7. **Sort:** `<?sort:ELEMENT;'ascending';data-type='number'?>` right after `for-each`.
8. **Group in template:** `<?for-each-group:G_EMP;./DEPT?>` + `current-group()`.
9. **Totals:** `sum()`, `count()`, division with `div`. **Variables:** `xdoxslt:set_variable / get_variable`.
10. **Format:** `<?format-date:DATE;'DD-MON-YYYY'?>`, `<?format-number:NUM;'999G999D99'?>`. Dates must be canonical.
11. **Pages:** `split-by-page-break`, `for-each@section`, `start@last-page:body`.
12. **Sub-template:** `<?template:NAME?>` … `<?end template?>`; `<?import:xdoxsl:///path.xsb?>` + `<?call-template:NAME?>`.
13. **Excel template:** `.xls`, `XDO_?ELEMENT?`, `XDO_GROUP_?GROUP?`, `XDO_METADATA` sheet (`TYPE_EXCEL_TEMPLATE`), `XDO_SHEET_?` for multiple sheets. Native formulas work.
14. **eText:** RTF with command tables. Setup table (TEMPLATE TYPE, character set, record character) + format tables (`<LEVEL>`, `<NEW RECORD>`, position, length, format, pad, data). Fixed-position vs delimiter-based. Header, detail, trailer.
15. **Outputs:** RTF → PDF/RTF/HTML/Excel/PPT; Excel → Excel; eText → Text; any → Data XML/CSV.
16. **Translations:** data via `_TL`; template text via XLIFF or localized templates.

---

## Practice Interview Questions with Answers

**Q1. What is the difference between a data model and a layout?**
The data model defines what data is fetched and how it's structured as XML. The layout (template) defines how that XML is presented. A report links one data model to one or more layouts, so the same data can produce a PDF letter and an Excel sheet.

**Q2. How do you repeat a table row for every employee in an RTF template?**
Put `<?for-each:G_EMP?>` in the first cell of the row and `<?end for-each?>` in the last cell of the same row. The entire row repeats for each G_EMP group in the XML.

**Q3. In a master–detail report, every department shows all employees. Why?**
The inner loop probably uses an absolute path like `//G_EMP`, which selects every employee in the document. It should use the relative path `G_EMP`, so it loops only within the current department.

**Q4. What is the difference between `for-each` and `for-each-group`?**
`for-each` loops over every occurrence of a group as it appears in the XML. `for-each-group` regroups flat data by a key (e.g., department) inside the template, and `current-group()` gives the rows in each group. If the data model already nests the data, a simple nested `for-each` is enough.

**Q5. How do you format a date in an RTF template?**
`<?format-date:HIRE_DATE;'DD-MON-YYYY'?>`. This works when the date is in canonical XML format. If the SQL converted it to text with TO_CHAR, it prints as is.

**Q6. How do you start each employee's letter on a new page with its own page numbering?**
Use `<?for-each@section:G_EMP?>`. It creates a new section for each employee, so headers, footers and page numbering restart. `split-by-page-break` only adds a page break.

**Q7. How do you show text only on the last page?**
Wrap it in `<?start@last-page:body?>` and `<?end body?>`. This is typically used for signature blocks or grand totals.

**Q8. What is a sub-template and why use it?**
A reusable RTF piece stored in the catalog, such as a company header. It defines named templates with `<?template:NAME?>`, and main templates import it with `<?import:xdoxsl:///path.xsb?>` and call it with `<?call-template:NAME?>`. Updating it once updates every report that uses it.

**Q9. When would you use an Excel template instead of RTF with Excel output?**
When users need native Excel: proper cell structure, working Excel formulas, multiple sheets (such as one per department), and data they'll filter or analyse. RTF to Excel is a converted document layout and can produce merged cells.

**Q10. How do you map data in an Excel template?**
With named ranges: `XDO_?ELEMENT?` maps a cell to an element, and `XDO_GROUP_?GROUP?` marks a range that repeats for each group occurrence. The workbook is saved as .xls and must include an XDO_METADATA sheet with Template Type set to TYPE_EXCEL_TEMPLATE.

**Q11. What is an eText template and where is it used?**
A template for generating plain text files in exact formats, used for bank payment files, vendor and benefits files, and other interfaces. It's an RTF containing command tables rather than a visual layout. HCM Extracts commonly use eText templates for their output.

**Q12. What's the difference between fixed-position and delimiter-based eText?**
In fixed-position files, each field starts at a fixed position with a fixed length, padded with zeros or spaces. In delimiter-based files, fields are separated by a character such as a comma or pipe, and each field has a maximum length.

**Q13. What are the main parts of an eText template?**
A setup table with global commands (template type, output character set, new record character, case conversion, sequences) and format tables. Each format table starts with a LEVEL linked to an XML group and a NEW RECORD name, and defines each field's position, length, format, padding and data. Header, detail and trailer records are separate tables.

**Q14. In a fixed-position file, fields after a blank value are shifted. How do you fix it?**
Enable "Include Empty Tags for Null Elements" in the data model so null values still produce tags, and make sure each field has a defined length and padding. Using NVL in SQL to supply defaults also helps.

**Q15. Which output formats are available for RTF, Excel and eText templates?**
RTF: PDF, RTF, HTML, Excel and PowerPoint. Excel templates: Excel. eText: text files. Any layout can also output raw Data (XML) or Data (CSV).

**Q16. How do you handle multiple languages in a BIP report?**
Data in the user's language comes from _TL tables filtered by USERENV('LANG'). For template text, use XLIFF translation files per locale against one template, or separate localized templates when the layout differs by country. BIP selects the version from the user's localeBIP.