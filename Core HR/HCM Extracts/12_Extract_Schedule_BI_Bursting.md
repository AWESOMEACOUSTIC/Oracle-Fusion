# HCM Extract From Design to Delivered File
### Submit, Scheduling, BI Publisher Report & Bursting
---

## 1. The full picture in one diagram

Split everything into two phases:
- **Design time** is what you set up **once**.
- **Run time** is what happens **every time** the extract runs.

```
DESIGN TIME (you set up once)
──────────────────────────────
1. Design the extract   → data groups, records, attributes, rules
2. Validate + Save      → Oracle also creates a FLOW (same name as extract)
3. Build the BI report  → template (RTF / eText) on the extract data model
4. Deliver tab          → link the report + choose output & destination
                          (+ bursting, if needed)

RUN TIME (every run)
──────────────────────────────
5. Submit / Schedule    → you press "go", or a schedule presses it for you
        ↓
6. Extract task         → reads HCM data → creates XML
        ↓
7. Report task          → BI Publisher takes the XML + your template → file
        ↓                  (bursting splits it into many files here)
8. Delivery             → SFTP / UCM / Email
        ↓
9. Check results        → View Extract Results / flow status / logs
```

**Memory line:** *"Design → Validate → Report → Deliver tab → Submit → Extract → Format → Send → Check."*

---

## 2. Validate and Save (design is ready)

Before anything can run:
- **Validate**: Oracle checks the design and compiles your formulas.
- **Save**: Oracle saves it and **creates a flow with the same name** as your extract.

### What is a flow?
A **flow** is a to-do list of tasks that run in order. For an extract, the to-do list is:

```
Task 1: Extract the data     (makes XML)
Task 2: Report + Delivery    (makes the file and sends it)
```

You never run an extract "alone". You always run its **flow**. That's why the submit screen asks for a **flow instance name**.

---

## 3. Build the BI Publisher report

### Why do we need BI Publisher?
The extract only produces **XML**, which is raw data. A vendor usually wants:
- a **CSV** or **pipe-separated** file
- a **fixed-width** text file (each field at an exact position)
- an **Excel** or **PDF** file

**BI Publisher turns the XML into that file.**

### What you create in BI Publisher
1. **A report** in a custom folder, for example `/Custom/HCM/XX_Emp_Report`.
2. **Data model:** Oracle's ready-made one for extracts, **`globalReportsDataModel`**.
   - You write **no SQL** here.
   - It simply hands your extract's XML to the template.
3. **A template:**

| Template | Used for | Output |
|---|---|---|
| **eText** | Files for other systems | CSV, pipe-separated, fixed-width text |
| **RTF** | Files for humans to read | PDF, Excel, Word, HTML |

The template is just instructions like:
*"For each person → print person number, comma, name, new line."*

### Important point
BI Publisher **does not read HCM tables** here. It only **formats the XML** the extract already created.

---

## 4. The Deliver tab (connects extract + report)

This is where you tell the extract which report to use and where to send the result. You fill in a **delivery option**:

| Field | What you put | Example |
|---|---|---|
| **Report** | Path of your BI report | `/Custom/HCM/XX_Emp_Report.xdo` |
| **Template** | Which template in that report | `CSV_Template` |
| **Output format** | Type of file | Text / PDF / Excel |
| **Output name** | File name | `Employee_File` |
| **Delivery type** | Where the file goes | SFTP, WebCenter Content (UCM), Email, None |
| **Bursting node** | *(Optional)* how to split output | see Section 8 |

### Notes
- One extract can have **more than one delivery option**, for example a CSV to SFTP **and** a PDF by email.
- SFTP and email servers are **not** set up in the extract. They're set up once by an admin in **BI Publisher Administration**. The extract just picks them.

### Testing tip
First set the delivery type to **None** (or just check the XML in View Extract Results). Make sure the **data** is correct before worrying about files and SFTP.

---

## 5. Submit (when the extract actually runs)

Everything before this was **setup**. **Submit = "run it now".**

### Steps
**My Client Groups → Data Exchange → Submit Extracts**
1. **Pick your extract.**
2. **Flow instance name**: a unique name for this run, for example `Emp_Extract_22Nov_Run1`. It must be **different every time**.
3. **Parameters:**
   - **Effective Date**: the "as of" date for the data (mandatory)
   - **Changes Only**: if your extract supports delta files
   - any custom parameters you added
4. **When to run:**
   - **As soon as possible**: runs now (used while **testing**)
   - **Using a schedule**: runs automatically (used in **real projects**)
5. **Submit.**

---

## 6. Scheduling (so nobody has to press the button)

In real projects, a vendor wants the file **every day / week / month** automatically.

When submitting, choose **Using a schedule**:

| Setting | Example |
|---|---|
| Frequency | Daily / Weekly / Monthly |
| Start date & time | 23-Nov-2026, 6:00 AM |
| End date | Far in the future (or when the interface ends) |

From then on, **Oracle submits the flow by itself** at that time.

### ⚠️ The most common scheduling trap
If the Effective Date is typed as a **fixed date** (like 22-Nov-2026), then **every** scheduled run uses 22-Nov, so the vendor gets the **same old data every day**.

**Fix:** the Effective Date must follow the **system date** (today). In the flow's parameter settings, Effective Date is **bound to the system date**. Then:

```
Run on 23-Nov → data as of 23-Nov
Run on 24-Nov → data as of 24-Nov
...
```

### Scheduling + Changes Only
For daily **delta** files:
- First run = **full** data (baseline)
- Every later run = only **what changed since the last run**
- This only works properly if the **effective date moves forward** each run (the same trap above).

---

## 7. What happens behind the scenes after Submit

```
Flow instance starts
   │
   ├── Task 1: EXTRACT
   │     • finds the employees (root data group)
   │     • splits them into chunks and runs in parallel (threading)
   │     • reads data through child data groups
   │     • runs your rules / fast formulas
   │     • applies filters
   │     • creates XML + saves (archives) the result
   │     • (Changes Only → compares with last run)
   │
   └── Task 2: REPORT / DELIVERY
         • sends the XML to BI Publisher
         • BI Publisher applies your template → final file
         • bursting splits it here (if configured)
         • file delivered to SFTP / UCM / Email
```

**Key point:** BI Publisher only joins at **Task 2**.

---

## 8. Bursting (one output becomes many)

### Without bursting
You get **one file** with **all** employees.

### With bursting
BI Publisher **splits** the output into **many files**, one for each value of something you choose.

You choose *where to split* using the **bursting node**: you tell it **which repeating level in the XML** to split on.

### Example
Your XML has 3 departments:
```xml
<DEPARTMENT> Finance ... 50 employees </DEPARTMENT>
<DEPARTMENT> HR      ... 20 employees </DEPARTMENT>
<DEPARTMENT> IT      ... 80 employees </DEPARTMENT>
```

| Setup | Result |
|---|---|
| No bursting | **1 file**, all 150 employees |
| Bursting node = DEPARTMENT | **3 files**: Finance (50), HR (20), IT (80) |
| Bursting node = PERSON | **150 files**, one per employee |

### Why do people use bursting?
| Need | Burst on |
|---|---|
| One file per legal employer, because each goes to a different payroll vendor | Legal employer level |
| One file per department, emailed to that department's manager | Department level |
| One letter or statement per employee | Person level |

Bursting can also send **each piece to a different place**, for example each department's file to a different email address, when that's configured in the delivery options.

### Important: the XML must be shaped for bursting
You can only burst on a **level that exists and repeats** in your XML. If you want one file per department, your extract design needs a **department level** (a data group) that groups the employees.

**Memory line:** *"Bursting = one run, split into many outputs, each going to the right place."*

---

## 9. Checking results and troubleshooting

### Where to look
| Where | What you see |
|---|---|
| **View Extract Results** | Each run, its status, the XML and output file |
| **Checklist / flow status** | Which task passed or failed (extract or report) |
| **Scheduled Processes** | Background jobs and their **log files** |

### Simple troubleshooting rule (find the layer)
```
XML is wrong              → problem in EXTRACT DESIGN (filters, rules, data groups, parameters)
XML right, file wrong     → problem in BI TEMPLATE (XML paths, format)
File right, not received  → problem in DELIVERY SETUP (SFTP / email server, path, credentials)
```

---

## 10. Scenario questions (try to answer before reading)

### Scenario 1
**The extract is scheduled daily, but the vendor says they get the same data every day.**
<details><summary>Answer</summary>

The Effective Date was entered as a fixed date, so every run uses that same date. Bind the Effective Date to the **system date** in the flow's parameters so each run uses that day's date. If Changes Only is used, the baseline/comparison also depends on the date moving forward.
</details>

### Scenario 2
**The extract completed successfully, but the CSV file is empty.**
<details><summary>Answer</summary>

Check the **XML** in View Extract Results first.
- XML is empty → the problem is in the extract (wrong effective date, filters too strict, or security means the user can't see the data).
- XML has data but the file is empty → the template's XML paths don't match the tag names, for example a tag name was changed after the template was built.
</details>

### Scenario 3
**The client has 3 legal employers, each with a different payroll vendor. Each vendor must get only their own employees.**
<details><summary>Answer</summary>

Use **bursting** with the bursting node at the **legal employer** level. One run produces one file per legal employer, and each file can be sent to the right vendor. The extract design must have a legal employer level in the XML to burst on. (Another option is separate extracts with filters, but bursting does it in one run.)
</details>

### Scenario 4
**You're building a new extract and want to test it quickly without sending anything to the vendor.**
<details><summary>Answer</summary>

Submit with **As soon as possible**, use a filter or small population if possible, and set the delivery type to **None**, or just check the XML in **View Extract Results**. Only after the data is correct, attach the template and test the real delivery.
</details>

### Scenario 5
**The file is generated correctly (you can see it in View Extract Results), but it never reaches the vendor's SFTP server.**
<details><summary>Answer</summary>

The problem is in the **delivery layer**: check the SFTP server setup in BI Publisher Administration (host, port, user, password/key), the remote folder path, and the delivery option's delivery type and server selection. Check the report/delivery task's log for the error.
</details>

### Scenario 6
**The vendor wants a CSV file on SFTP, and HR also wants a PDF of the same data by email.**
<details><summary>Answer</summary>

Create **two delivery options** on the same extract:
1. eText CSV template → SFTP
2. RTF template → PDF → Email

Both run from the same extract XML in one submission.
</details>

### Scenario 7
**Each employee should receive their own letter by email.**
<details><summary>Answer</summary>

Use an **RTF** template (for the letter layout), output **PDF**, and **burst at the person level** so each employee gets a separate document. The email address for each burst comes from the data, so the extract must include each person's email.
</details>

### Scenario 8
**The submit fails with an error about the flow instance name.**
<details><summary>Answer</summary>

Flow instance names must be **unique**. You probably reused the name of an earlier run. Give a new name, for example by adding the date and time.
</details>

### Scenario 9
**Daily delta file: the first run sent everyone. Is that a bug?**
<details><summary>Answer</summary>

No. The first Changes Only run has nothing to compare with, so it creates the **baseline** and sends full data. From the second run onward, only changes are sent.
</details>

---

## 11. Rapid revision Q&A

| Question | Answer |
|---|---|
| What gets created when you save an extract? | A flow with the same name |
| What are the two main tasks in an extract flow? | Extract task (XML) and Report/Delivery task (file) |
| Which data model is used for the BI report? | `globalReportsDataModel` |
| Which template for a CSV or fixed-width file? | eText |
| Which template for a PDF or Excel for humans? | RTF |
| Where do you link the BI report to the extract? | Deliver tab, in the delivery options |
| Where is the SFTP server set up? | BI Publisher Administration |
| Where do you submit an extract? | Data Exchange → Submit Extracts |
| What must be unique on every submit? | Flow instance name |
| Two "when to run" options? | As soon as possible, or Using a schedule |
| How do you make scheduled runs use today's data? | Bind Effective Date to the system date |
| What is bursting? | Splitting one output into many, by a repeating XML level |
| What decides where bursting splits? | The bursting node |
| Where do you check the output? | View Extract Results |
| Where do you check logs? | Scheduled Processes / flow task logs |

---

## 12. Interview answer (say it like this)

> "Once the extract design is ready, I validate and save it, which also creates a flow with the same name. Then I create a BI Publisher report on the globalReportsDataModel with an eText or RTF template, and link it in the extract's delivery options along with the output format and delivery type, such as SFTP or UCM. Then I submit the extract through Submit Extracts with the effective date, either immediately for testing or on a schedule for production, with the effective date bound to the system date so each run picks up current data. At run time the flow runs the extract task to generate XML, then the report task, where BI Publisher applies the template and delivers the file. If bursting is configured, BI Publisher splits the output at the bursting node, for example one file per department or legal employer, and can deliver each one to a different destination. Finally I monitor it through View Extract Results and the flow status."