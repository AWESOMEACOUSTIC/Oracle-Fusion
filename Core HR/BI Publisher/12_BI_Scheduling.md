# BI Publisher for Oracle HCM: Phase 5, Delivery and Automation

> **Goal of this phase:** Make reports run **without a person clicking "View"**: schedule them, run them as Fusion scheduled processes, deliver them by email or SFTP, split one run into many outputs with bursting, encrypt files, and understand how HCM Extracts use BIP for their output.

**Topics in this phase**
29. Scheduling reports (BIP scheduler and ESS scheduled processes)
30. Delivery channels
31. Bursting
32. PGP encryption for outbound files
33. HCM Extracts + BIP

---

## 29. Scheduling Reports ⭐

### Why schedule?
- Reports that are **heavy** (large data) should not run online, where they can time out.
- Reports needed **regularly** (daily, weekly, monthly) should run automatically.
- **Bursting and delivery** (email, SFTP) only happen through a **scheduled job**.

There are **two ways** to schedule a BIP report in Fusion HCM:

| Method | Where | Who usually uses it |
|---|---|---|
| **BIP Scheduler** | BI Catalog → report → **Schedule** | Developers, BI admins, users with BI access |
| **ESS Scheduled Process** | **Navigator → Tools → Scheduled Processes** | HR business users (familiar Fusion screen) |

---

### A) BIP Scheduler

**How to open:** Catalog → find the report → **More → Schedule** (or open the report → **Actions → Schedule**).

The schedule page has **four tabs**:

#### 1. General tab
- The **report** being scheduled.
- **Parameter values** for this run.

**Tip:** Scheduled reports usually use **date expressions as default values** (from Phase 3), e.g., `{$FIRST_DAY_OF_MONTH()$}` and `{$LAST_DAY_OF_MONTH()$}`. Each monthly run then picks up the correct month without anyone changing the dates.

#### 2. Output tab

| Setting | Meaning |
|---|---|
| **Use Bursting Definition to Determine Output & Delivery Destination** | Tick this for bursting reports (Topic 31). Output and delivery then come from the bursting SQL. |
| **Output Name** | File name of the output. Can include **date codes**, e.g., `XX_NEW_HIRES_%y%m%d` gives `XX_NEW_HIRES_20261109`. |
| **Layout** | Which layout to use |
| **Format** | PDF, Excel, Text, etc. |
| **Locale / Time Zone** | Language and time zone for the output |
| **Save Output** | Keep a copy of the output in BIP, to view later from job history |
| **Save Data for Republishing** | Keep the XML, so the output can later be regenerated with a different layout or format **without rerunning the SQL** |
| **Destination** | Where to deliver: Email, FTP/SFTP, Content Server (UCM), Printer (Topic 30). One output can go to **multiple destinations**. |

**Common date codes in Output Name**

| Code | Meaning |
|---|---|
| `%y` | Year (4 digits) |
| `%m` | Month |
| `%d` | Day |
| `%H` / `%M` / `%S` | Hour / Minute / Second |

#### 3. Schedule tab

| Frequency | Example |
|---|---|
| **Once** | Run now, or once at a specific date and time |
| **Hourly / Minute** | Every 2 hours |
| **Daily** | Every day at 6:00 AM |
| **Weekly** | Every Monday |
| **Monthly** | 1st of every month |
| **Annually** | Every 1st April |
| **Specific Dates** | A list of chosen dates |

- Set a **start date/time** and an optional **end date**.
- A **schedule trigger** (Phase 3) can be attached here, so the job runs **only if** the trigger SQL returns rows.

#### 4. Notification tab
- Send an **email notification** when the job completes with **Success**, **Warning**, **Failed**, or **Skipped** status.
- Useful so the support team learns about failures immediately.

Click **Submit** and give the job a name.

### Monitoring BIP scheduled jobs

| Screen | What it shows | Actions |
|---|---|---|
| **Report Jobs** | **Future and recurring** schedules | Edit, pause, resume, delete a schedule |
| **Report Job History** | **Completed and running** job instances | View status, open output, see delivery status, view error details, **republish** |

**Job statuses:** Running, Success, Warning, Failed, Skipped (schedule trigger returned no rows), Canceled.

---

### B) ESS Scheduled Process ⭐

**ESS (Enterprise Scheduler Service)** is the Fusion framework behind **Navigator → Tools → Scheduled Processes**. All seeded Fusion processes (payroll runs, HCM loads, etc.) run through ESS.

**Why convert a BIP report into an ESS job?**
- HR users can run it from the **standard Scheduled Processes screen**, without going into the BI Catalog.
- It can be **combined with other jobs** in **job sets**.
- Users get the **standard Fusion features**: parameters, scheduling, output, log, notifications.
- Access is controlled through **Fusion security**.

**Steps to create an ESS job for a BIP report**
1. **Setup and Maintenance** → search the task **"Manage Enterprise Scheduler Job Definitions and Job Sets for Human Capital Management and Related Applications"**.
2. In **Job Definitions**, click **Create**.
3. Fill in:

| Field | Value / Meaning |
|---|---|
| **Display Name** | Name users see, e.g., `XX Department Wise Employee Report` |
| **Name** | Internal name, e.g., `XX_DEPT_EMP_REPORT` |
| **Path** | Folder in the job hierarchy, e.g., `/xx/custom/` |
| **Job Application Name** | The HCM application (e.g., `HcmCore`) |
| **Job Type** | **`BIPJobType`** (this makes it a BIP report job) |
| **Report ID** | Path of the report **starting from `/Custom`**, ending in `.xdo`, e.g., `/Custom/HCM/Reports/XX_DEPT_EMP_RPT.xdo` (do **not** include "Shared Folders") |
| **Default Output Format** | PDF, Excel, etc. |
| **Enable submission from Scheduled Processes** | Ticked |

4. In the **Parameters** section, create **one ESS parameter per data model parameter**:
   - **Same order** as the parameters in the data model. ESS passes values **by position**, not by name.
   - **Matching data types** (String, Date, Number).
   - Parameter prompt (label), required or optional, default value.
   - For drop-downs, ESS parameters need a **List of Values Source** (set up separately). Simple ESS jobs often use text or date parameters.
5. **Save.**
6. Make sure the right roles **have access** to run the job. This is part of the project's security setup.
7. Users run it from **Scheduled Processes → Schedule New Process** → search the display name → enter parameters → **Advanced** (for schedule and output options) → **Submit**.

**After the run:**
- Status (Succeeded, Error, Warning) and output are visible in **Scheduled Processes**.
- Output and log files are attached to the process.
- Output can be **republished** in a different format.

**Common ESS mistakes**

| Problem | Cause |
|---|---|
| Wrong data in output | ESS parameters in a **different order** than the data model parameters |
| "Report not found" error | **Report ID** includes `/Shared Folders` or has a typo or wrong case |
| Date parameter not applied | Data type mismatch between ESS parameter and data model parameter |
| Job not visible to user | User's role doesn't have access to the job |

> **Interview-ready line:**
> "Reports can be scheduled through the BIP scheduler, where I set parameters, output name with date codes, layout and format, delivery destinations, frequency and notifications, then monitor them in Report Job History. For business users, I create an ESS job definition with job type BIPJobType and the report ID path starting from /Custom, with parameters in the same order and data types as the data model, so users can run it from Scheduled Processes."

---

## 30. Delivery Channels

### Available channels

| Channel | Use | Example |
|---|---|---|
| **Email** | Send output as an attachment | Weekly headcount report to HR managers |
| **FTP / SFTP** | Place a file on a server | Bank or vendor interface file (SFTP is the secure, standard choice) |
| **Content Server (UCM / WebCenter Content)** | Store files in Fusion's content server | Files used by other Fusion processes or picked up by integration tools (e.g., Oracle Integration Cloud) |
| **Printer** | Send directly to a configured printer | Rare in cloud projects |

### Setup: who does what
- **Servers are configured by the BI Administrator**: email server, SFTP server details (host, port, credentials, keys), content server. This is done in **BI Administration → Delivery** configuration.
- Each configured server gets a **name**. Developers and users **refer to that name** when scheduling or bursting. They don't enter host details in reports.

### Email delivery fields
| Field | Example |
|---|---|
| **To / CC / BCC** | `hr.team@company.com` (multiple addresses separated by commas) |
| **Reply-To** | `hr.support@company.com` |
| **Subject** | `Monthly New Hires Report` |
| **Message** | Body text |
| **Attach output** | Yes (output as an attachment) |

### SFTP delivery fields
| Field | Example |
|---|---|
| **Server** | Configured server name, e.g., `XX_BANK_SFTP` |
| **Remote Directory** | `/inbound/payments/` |
| **Remote File Name** | `PAYFILE_%y%m%d.txt` |

### Content Server (UCM) delivery
- Files are placed under a **security group and account**, e.g., the `hcm$/dataloader$/export$` account used for HCM data exports.
- Integration tools (like **OIC**) can then pick up the file from UCM.

> **Interview-ready line:**
> "BIP can deliver output by email, FTP/SFTP, the content server (UCM), or printer. The BI Administrator configures the servers in delivery settings, and in schedules or bursting we refer to those configured server names. Email is used for reports to people, SFTP for vendor and bank interface files, and UCM when files are consumed by other Fusion processes or integration tools."

---

## 31. Bursting ⭐ (very important for interviews)

### What is bursting?
**Bursting = one report run → many separate outputs → each delivered to a different destination.**

Without bursting, a report run produces **one** output for **everyone**.
With bursting, BIP **splits the data** by a key (e.g., employee, department, manager) and creates **one output per key value**, each with its own **layout, format, and delivery**.

### Typical HCM bursting requirements

| Requirement | Split by | Deliver to |
|---|---|---|
| Each employee gets their own salary revision letter | Employee (`PERSON_ID`) | Employee's work email |
| Each manager gets a report of their own team | Manager (`MANAGER_ID`) | Manager's email |
| Each department head gets their department's headcount | Department | Department head's email |
| Each legal entity's file goes to a different SFTP folder | Legal entity | SFTP folder per entity |

### Where bursting is defined
Bursting is defined in the **data model** (left pane → **Bursting**), then **enabled in the report**, then **used in a schedule**.

### Step 1: Create the bursting definition (in the data model)

| Field | Meaning |
|---|---|
| **Name** | e.g., `XX_LETTER_BURST` |
| **Type** | SQL Query |
| **Data Source** | `ApplicationDB_HCM` |
| **Split By** | XML element that **splits** the data into chunks, e.g., `/DATA_DS/G_EMP/PERSON_ID` |
| **Deliver By** | XML element whose value is **matched with `KEY`** in the bursting query to find the delivery details. Usually the same as Split By. |
| **SQL Query** | The **bursting query** (below) |

**Split By vs Deliver By**
- **Split By** decides **how the data is cut**: one chunk per value.
- **Deliver By** decides **which delivery row** (from the bursting query) applies to each chunk.
- In most HCM cases both are the same element (e.g., `PERSON_ID`).
- The XML data should be **sorted by the Split By element**, so all rows for one key are together.

### Step 2: Write the bursting query
The bursting query returns **one row per key**, with **fixed column aliases** that BIP understands:

| Column alias | Meaning | Example |
|---|---|---|
| **`KEY`** | Must match the **Deliver By** element value | `papf.person_id` |
| **`TEMPLATE`** | **Layout name** (as named in the report) | `'XX_REVISION_LETTER'` |
| **`LOCALE`** | Language/locale | `'en-US'` |
| **`OUTPUT_FORMAT`** | Output type | `'PDF'`, `'EXCEL'`, `'RTF'` |
| **`DEL_CHANNEL`** | Delivery channel | `'EMAIL'`, `'FTP'`, `'SFTP'`, `'WCC'` (content server), `'PRINT'` |
| **`OUTPUT_NAME`** | File name of each output | `'Letter_' \|\| papf.person_number` |
| **`SAVE_OUTPUT`** | Keep the output in job history | `'true'` |
| **`TIMEZONE`** | Optional time zone | `'Asia/Calcutta'` |
| **`PARAMETER1` … `PARAMETER10`** | Channel-specific delivery details (below) | |

### PARAMETER meanings by channel ⭐

**EMAIL**

| Parameter | Meaning |
|---|---|
| **PARAMETER1** | To (email address) |
| **PARAMETER2** | CC |
| **PARAMETER3** | From |
| **PARAMETER4** | Subject |
| **PARAMETER5** | Message body |
| **PARAMETER6** | Attachment (`'true'`) |
| **PARAMETER7** | Reply-To |
| **PARAMETER8** | BCC |

**FTP / SFTP**

| Parameter | Meaning |
|---|---|
| **PARAMETER1** | Server name (as configured by the BI Admin) |
| **PARAMETER2** | Username |
| **PARAMETER3** | Password |
| **PARAMETER4** | Remote directory |
| **PARAMETER5** | Remote file name |
| **PARAMETER6** | Secure (`'true'` for SFTP) |

- Don't hardcode passwords in the SQL. Follow the project's standard for credentials, which are normally kept in the configured server setup.
- Parameters for other channels (content server, printer) differ. Check Oracle's bursting documentation when you need them.

### Example: email each employee their own letter

**Data model data set (`G_EMP`)** returns one row per employee with `PERSON_ID`, `PERSON_NUMBER`, `FULL_NAME`, `NEW_SALARY`, etc., sorted by `PERSON_ID`.

**Bursting definition**
- Split By: `/DATA_DS/G_EMP/PERSON_ID`
- Deliver By: `/DATA_DS/G_EMP/PERSON_ID`

**Bursting SQL**
```sql
SELECT papf.person_id                               AS "KEY",
       'XX_REVISION_LETTER'                         AS TEMPLATE,
       'en-US'                                      AS LOCALE,
       'PDF'                                        AS OUTPUT_FORMAT,
       'EMAIL'                                      AS DEL_CHANNEL,
       'Salary_Revision_' || papf.person_number     AS OUTPUT_NAME,
       'true'                                       AS SAVE_OUTPUT,
       pea.email_address                            AS PARAMETER1,  -- To
       NULL                                         AS PARAMETER2,  -- CC
       'hr.noreply@company.com'                     AS PARAMETER3,  -- From
       'Your Salary Revision Letter'                AS PARAMETER4,  -- Subject
       'Dear Employee, please find your letter attached.' AS PARAMETER5,  -- Body
       'true'                                       AS PARAMETER6   -- Attach output
FROM   per_all_people_f     papf
JOIN   per_email_addresses  pea
       ON  pea.person_id  = papf.person_id
       AND pea.email_type = 'W1'
       AND TRUNC(SYSDATE) BETWEEN pea.date_from AND NVL(pea.date_to, DATE '4712-12-31')
WHERE  TRUNC(SYSDATE) BETWEEN papf.effective_start_date AND papf.effective_end_date
```
- `"KEY"` is in double quotes to keep the alias exactly as `KEY`.
- Only keys that **exist in the report data** get an output. Other rows in the bursting query are ignored.
- Employees **without a work email** get no delivery row, so their letter isn't sent. Requirements should cover what happens to them (e.g., send to HR instead using `NVL`).

### Step 3: Enable bursting in the report
- Open the report → **Properties → General** → tick **Enable Bursting** → choose the bursting definition (`XX_LETTER_BURST`).

### Step 4: Schedule the report
- Schedule it → **Output tab** → tick **Use Bursting Definition to Determine Output & Delivery Destination**.
- Submit.

**Bursting runs only through a scheduled job** (BIP scheduler or ESS). Viewing the report online doesn't burst.

### Step 5: Verify
- **Report Job History** → open the job → each burst output is listed with its **delivery status** (success or failure per recipient).

### Bursting troubleshooting ⭐

| Symptom | Likely cause |
|---|---|
| No outputs generated | `KEY` values don't match the **Deliver By** element values (e.g., person_id vs person_number, or a data type difference) |
| One big output instead of many | **Split By** path is wrong or points to the wrong group |
| Output blank or error per burst | **TEMPLATE** value doesn't exactly match the **layout name** in the report |
| Some employees didn't receive email | No email row for them in the bursting SQL, or the email address is NULL or invalid |
| Mixed data in an output | Data not **sorted** by the Split By element, or Split By chosen at the wrong level |
| Bursting not happening at all | Bursting not enabled in report properties, or "Use Bursting Definition" not ticked in the schedule, or the report was viewed online |
| Email delivery failed | Email server not configured, or From address not allowed by the mail server |

> **Interview-ready line:**
> "Bursting splits one report run into multiple outputs and delivers each separately, for example a letter emailed to each employee. I define it in the data model with a Split By element, a Deliver By element, and a bursting SQL that returns KEY, TEMPLATE, LOCALE, OUTPUT_FORMAT, DEL_CHANNEL, OUTPUT_NAME and PARAMETER1 to 10, where the parameters depend on the channel. For email, PARAMETER1 is To, PARAMETER3 is From, PARAMETER4 is Subject, PARAMETER5 is the body, and PARAMETER6 is the attachment flag. KEY must match the Deliver By value. Then I enable bursting in the report properties and schedule it with the bursting option ticked, because bursting only works through scheduling."

---

## 32. PGP Encryption for Outbound Files

### Why encrypt?
Interface files (bank payments, salary data, national IDs) contain **highly sensitive data**. SFTP secures the **transfer**, but banks and vendors usually also require the **file itself** to be encrypted, so it stays protected even after it lands on their server.

### How PGP works (the basics)
PGP uses a **key pair**:

| Key | Owner | Used for |
|---|---|---|
| **Public key** | Shared openly by the **receiver** (bank or vendor) | **Encrypting** the file |
| **Private key** | Kept secret by the **receiver** | **Decrypting** the file |

**Flow:**
1. The vendor sends their **public key** to the project.
2. The public key is **uploaded into Fusion**.
3. BIP **encrypts** the output file with that public key before delivery.
4. Only the vendor, with their **private key**, can **decrypt** it.

### Signed vs unsigned
| Mode | Meaning |
|---|---|
| **Encrypted (unsigned)** | File is encrypted with the vendor's public key |
| **Encrypted + signed** | File is also **signed** with **our** private key, so the vendor can **verify it really came from us** and wasn't changed |

### Where it's set up (Fusion)
- **For BIP delivery:** the **BI Administrator** uploads the vendor's public key in BI Administration (in the security area for **PGP keys**) and enables PGP encryption on the delivery setup (e.g., the SFTP server configuration).
- **For HCM Extracts:** encryption is chosen in the extract's **delivery options** (encryption mode such as PGP Signed or PGP Unsigned, plus the key).
- Exact screen names differ slightly by release, and key management is usually handled by the **admin or security team**.

### File naming with encryption
Encrypted files usually end in **`.pgp`** or **`.gpg`**, e.g., `PAYFILE_20261109.txt.pgp`. Confirm the expected name with the vendor.

> **Interview-ready line:**
> "For sensitive outbound files we use PGP encryption. The vendor shares their public key, which is uploaded into Fusion, and BIP or the HCM Extract encrypts the file with it before SFTP delivery, so only the vendor can decrypt it with their private key. If the vendor needs to verify the sender, we also sign the file with our private key."

---

## 33. HCM Extracts + BIP ⭐

### Recap: what is an HCM Extract?
An **HCM-specific tool for outbound integrations** (from Phase 1):
- Built using **data groups**, **records**, and **attributes** based on **User Entities** and **Database Items (DBIs)**, not raw SQL.
- Filters with **criteria** and **Fast Formulas**.
- Supports **Changes Only** (delta) extraction.
- Generates **XML** as its raw output.

**The extract produces XML, but it has no layout of its own. A BIP template formats the XML into the final file.**

### HCM Extract structure (quick view)

| Component | Meaning |
|---|---|
| **Extract Definition** | The overall extract (name, type, parameters) |
| **Data Group** | A block of data based on a **User Entity** (e.g., person or assignment); like a data set |
| **Connection** | Parent–child link between data groups (like master–detail) |
| **Record** | A group of attributes inside a data group |
| **Attribute** | One field, usually a **DBI** (e.g., person number) or a fast formula result |
| **Criteria** | Filters (e.g., only active employees, a legal entity) |
| **Delivery Options** | Output format, BIP template, delivery type, encryption |

The data group, record and attribute **names become XML tags** in the extract output.

### How HCM Extracts use BIP: steps ⭐
1. **Define the extract** (data groups, records, attributes, criteria) in **Data Exchange → Extract Definitions**.
2. **Run the extract once** and **download its XML output**. This is your **sample XML**.
3. **Design the template** (usually **eText**, sometimes RTF or Excel) using that XML. Template tags must match the **extract's XML tag names**.
4. **Create a BIP report** that uses the seeded data model **`globalReportsDataModel`** (in `/Shared Folders/Human Capital Management/Payroll/Data Models/`).
   - This data model is a **placeholder**: at run time, the extract **passes its own XML** to the report.
   - Upload your template as a layout in this report. Save it in `/Custom/...`.
5. In the extract's **Delivery Options**, set:

| Field | Value |
|---|---|
| **Output Format** | e.g., eText, Excel, PDF, XML |
| **Report** (template path) | e.g., `/Custom/HCM/Extracts/XX_BENEFITS_RPT.xdo` |
| **Template Name** | The **layout name** in that report |
| **Output Name** | File name |
| **Delivery Type** | Email, FTP/SFTP, WebCenter Content (UCM), or None |
| **Encryption** | PGP mode and key, if required |

6. **Validate / refresh** the extract definition so the changes are compiled.
7. **Submit the extract** (through its flow: Data Exchange → Submit Extracts, or Scheduled Processes). Choose parameters like **effective date** and **Changes Only**.
8. **Check the output** in the flow results: the generated file and delivery status.

### HCM Extract vs pure BIP report for an interface

| Point | Pure BIP (SQL + eText) | HCM Extract + BIP template |
|---|---|---|
| Data fetching | Custom SQL | User Entities and DBIs |
| Changes only | Must code it manually in SQL | **Built-in** |
| Security | Developer handles it | Built into the extract framework |
| Volume | Medium–large | **Very large** |
| Flexibility | **Very high** (any table, any logic) | Limited to what UEs, DBIs and fast formulas offer |
| Output formatting | BIP template | **BIP template** |
| Run as | BIP schedule or ESS job | Extract **flow** |

**Choosing between them:**
- **HCM Extract** when the integration is **regular**, **high volume**, needs **changes only**, or is **payroll-related**.
- **Pure BIP** when the logic needs **custom SQL** that DBIs can't provide, or for smaller, simpler files.

> **Interview-ready line:**
> "An HCM Extract produces XML using data groups, records and attributes based on user entities and DBIs, but it has no layout of its own. To produce the final file, I run the extract once to get sample XML, design an eText or RTF template on it, create a BIP report using the seeded globalReportsDataModel, and set the report path, template name, output format, delivery type and encryption in the extract's delivery options. At run time the extract passes its XML to that report, and BIP formats and delivers the file."

---

## End-to-End Example: Monthly New Hires File to a Vendor

**Requirement:** Every month, send a pipe-delimited file of last month's new hires to a background-check vendor over SFTP, encrypted with PGP. Also email HR a PDF summary.

| Step | What you do | Phase |
|---|---|---|
| 1 | SQL on `PER_PERIODS_OF_SERVICE` (`DATE_START` between parameters), person, names, assignment; date-effective as of hire date | 2 |
| 2 | Data model with `P_FROM_DATE` / `P_TO_DATE` (defaults: first and last day of the **previous** month), sample XML saved; **Include Empty Tags for Null Elements** ticked | 3 |
| 3 | Report with two layouts: **eText** (delimiter-based, pipe) and **RTF** (PDF summary) | 4 |
| 4 | Admin configures the vendor **SFTP server** and uploads the vendor's **PGP public key** | 5 |
| 5 | Schedule (or ESS job): **monthly on the 1st**, two outputs: eText → **SFTP** (`NEWHIRES_%y%m%d.txt`, encrypted); RTF/PDF → **email** to HR | 5 |
| 6 | Optional **schedule trigger**: run only if there were new hires last month | 3 |
| 7 | **Notification** on failure to the support team | 5 |
| 8 | Monitor in **Report Job History** / **Scheduled Processes** | 5 |

---

## Quick Revision Sheet

1. **Schedule** for heavy, regular, or delivered reports. Bursting and delivery **need a schedule**.
2. **BIP Scheduler tabs:** General (parameters) → Output (layout, format, name, destinations, bursting option) → Schedule (frequency) → Notification.
3. **Output name date codes:** `%y %m %d`.
4. **Save Data for Republishing** = keep XML to regenerate output without rerunning SQL.
5. **Report Jobs** = future schedules. **Report Job History** = runs, outputs, delivery status.
6. **ESS job:** Job Type **`BIPJobType`**, **Report ID from `/Custom/...xdo`** (no "Shared Folders"), parameters in the **same order and types** as the data model, run from **Scheduled Processes**.
7. **Delivery channels:** Email, FTP/SFTP, Content Server (UCM), Printer. Servers configured by the **BI Admin**; reports refer to **server names**.
8. **Bursting** = one run → many outputs → many destinations. Defined in the **data model** → enabled in **report properties** → used in **schedule**.
9. **Split By** = how data is split. **Deliver By** = element matched to **KEY**.
10. **Bursting SQL aliases:** `KEY, TEMPLATE, LOCALE, OUTPUT_FORMAT, DEL_CHANNEL, OUTPUT_NAME, SAVE_OUTPUT, PARAMETER1–10`.
11. **Email params:** P1 To, P2 CC, P3 From, P4 Subject, P5 Body, P6 Attachment, P7 Reply-To, P8 BCC.
12. **SFTP params:** P1 Server, P2 User, P3 Password, P4 Directory, P5 File name, P6 Secure.
13. **TEMPLATE = layout name** exactly. **KEY = Deliver By value** exactly.
14. **PGP:** vendor's public key encrypts, vendor's private key decrypts; signing proves the sender.
15. **HCM Extract + BIP:** extract XML → template built on extract XML → report on **`globalReportsDataModel`** → set report path and template in **Delivery Options**.

---

## Practice Interview Questions with Answers

**Q1. Why would you schedule a report instead of running it online?**
Large reports can time out online, regular reports should run automatically, and bursting and delivery to email or SFTP only happen through a scheduled job.

**Q2. What does "Save Data for Republishing" do?**
It saves the XML data from the run, so the output can later be regenerated in another layout or format from Report Job History without running the SQL again.

**Q3. How do you make a BIP report available in Scheduled Processes?**
Create an ESS job definition in Setup and Maintenance (Manage Enterprise Scheduler Job Definitions for HCM) with job type BIPJobType and the report ID set to the .xdo path starting from /Custom. Add ESS parameters in the same order and data types as the data model parameters, and make sure the right roles can access the job.

**Q4. Why must ESS parameters be in the same order as the data model parameters?**
ESS passes parameter values by position, not by name. If the order differs, values go to the wrong bind variables and the output is wrong.

**Q5. What delivery channels does BIP support?**
Email, FTP/SFTP, the content server (UCM/WebCenter Content), and printer. The BI Administrator configures the servers, and schedules or bursting queries refer to them by the configured server name.

**Q6. What is bursting?**
Splitting one report run into multiple outputs based on a key, such as employee or manager, with each output getting its own layout, format and delivery destination. A common example is emailing each employee their own letter.

**Q7. What is the difference between Split By and Deliver By?**
Split By is the element used to cut the XML into separate chunks, one per value. Deliver By is the element whose value is matched with the KEY column in the bursting query to find each chunk's delivery details. They are usually the same element.

**Q8. Which columns does a bursting query return?**
KEY, TEMPLATE, LOCALE, OUTPUT_FORMAT, DEL_CHANNEL, OUTPUT_NAME, SAVE_OUTPUT, optionally TIMEZONE and CALENDAR, and PARAMETER1 to PARAMETER10, whose meaning depends on the delivery channel.

**Q9. For email bursting, what do the parameters mean?**
PARAMETER1 is To, PARAMETER2 CC, PARAMETER3 From, PARAMETER4 Subject, PARAMETER5 message body, PARAMETER6 attachment flag, PARAMETER7 Reply-To, and PARAMETER8 BCC.

**Q10. The bursting job ran but no outputs were generated. What do you check?**
That KEY values match the Deliver By element values (same column and data type), that the Split By path points to the correct group, that the TEMPLATE value exactly matches the layout name, that bursting is enabled in report properties, and that "Use Bursting Definition" was ticked in the schedule.

**Q11. Can you burst a report by viewing it online?**
No. Bursting only runs through a scheduled job, from the BIP scheduler or an ESS process.

**Q12. Some employees didn't receive their letters. Why?**
Most likely they have no work email or a null email in the bursting query, so no delivery row was created for them. Job history shows per-output delivery status. The fix is to handle missing emails, for example sending those letters to HR.

**Q13. How does PGP encryption work for an outbound file?**
The vendor shares their public key, which is uploaded into Fusion. BIP or the extract encrypts the file with it before delivery, and only the vendor can decrypt it with their private key. If signing is required, the file is also signed with our private key so the vendor can verify the sender.

**Q14. How does an HCM Extract use BI Publisher?**
The extract generates XML from data groups, records and attributes. A BIP report based on the seeded globalReportsDataModel holds the template (usually eText), built from the extract's sample XML. The extract's delivery options point to that report path and template name, along with output format, delivery type and encryption. At run time the extract passes its XML to the report, which formats and delivers the file.

**Q15. When would you choose an HCM Extract over a BIP report for an interface?**
When the integration is regular and high-volume, needs changes-only extraction, or is payroll-related, because extracts have built-in delta logic and security. A BIP report is better when the logic needs custom SQL that DBIs can't provide, or for smaller files.

**Q16. What is a schedule trigger and when would you use it here?**
A SQL query attached to a schedule. The job runs only if the query returns rows. For example, a monthly new hires file runs only if there were new hires, which avoids sending empty files to the vendor.

---

**Next: Phase 6, Project Readiness** (performance tuning, troubleshooting, migration across environments, documentation, practice reports, and a final interview question bank).