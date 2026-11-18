# Oracle BI Publisher: Bursting Guide

> Built on the same Supplier Invoice data model from your previous guide. Read the parts in order: Parts 1–4 build the concept, Part 5 fixes your query, Parts 6–7 handle multi-language and multi-template bursting, Part 8 is scheduling, and Part 9 is situational questions.

---

## Part 1: What Is Bursting and Why Do We Need It?

### The problem
Your report shows invoices grouped by supplier site. Say the AP team wants **each supplier site to receive only its own invoices, by email, as a PDF**.

**Without bursting:** someone runs the report once per site, saves the PDF, opens Outlook, attaches it and sends it. For 200 sites that's 200 manual runs.

**With bursting:** run the report **once**. BI Publisher:
1. **cuts** the output into pieces (one per site),
2. **formats** each piece (template, language, PDF/Excel),
3. **delivers** each piece to its own recipient (email/FTP/printer…).

> **One-line definition:** bursting = one report run → split into many personalized documents → each delivered to the right recipient automatically.

### A simple way to picture it

```
           YOUR XML DATA  (one big cake)
 ┌────────────────────────────────────────────┐
 │ Site ACME-NYC     │ Site ACME-LON │ Site X │
 └────────────────────────────────────────────┘
        │                    │           │
   SPLIT BY  = where to cut the cake into slices
        │                    │           │
   DELIVER BY + BURSTING QUERY = the address label on each slice
        ▼                    ▼           ▼
   📧 nyc@acme.com     📧 lon@acme.com   📧 x@x.com
```

---

## Part 2: The Bursting Screen, Field by Field

| Field | What it means | Your value |
|---|---|---|
| **Name** | Name of this bursting definition (you'll select it in report properties) | e.g. `Burst_By_Site` |
| **Type** | Where delivery instructions come from; almost always **SQL Query** | SQL Query |
| **Data Source** | DB connection for the bursting query | ApplicationDB_FSCM |
| **Split By** | XML element used to **cut** the data into separate documents | `/DATA_DS/G_1/VENDOR_SITE_CODE` |
| **Deliver By** | XML element whose value is **matched with the `KEY` column** of the bursting query to find delivery details | `/DATA_DS/G_1/VENDOR_SITE_CODE` |
| **Enable Consolidated Output** | Also produce **one combined output** of all the pieces (saved in job history), in addition to the individual bursted files | Leave unchecked unless someone wants a single copy of everything |
| **Group Data by Split Key Values** | If the same split value appears in **non-adjacent** places in the XML, group it into one document instead of creating two | Leave unchecked: your data model already groups by site, so each site appears once |
| **SQL Query** | The "address book": one row per delivery, with fixed column names | Your query |

---

## Part 3: Split By vs Deliver By (the Confusing Part)

### 3.1 The definitions

**Split By: "How many documents?"**
Every **distinct value** of this element becomes **one separate document**.
- Split by VENDOR_SITE_CODE → one PDF per site.
- Split by INVOICE_ID → one PDF per invoice.

**Deliver By: "Which row of the bursting query is the delivery instruction for this document?"**
For each document, BIP takes the value of the Deliver By element and looks for the bursting query row(s) where **`KEY` = that value**. That row says where the document goes (email, template, format, file name…).

```
   Document for site "ACME-NYC"
        │
        │  Deliver By value = "ACME-NYC"
        ▼
   Bursting query rows ──► find KEY = 'ACME-NYC'
        │
        ▼
   TEMPLATE=T1, FORMAT=PDF, EMAIL=nyc@acme.com  → send
```

### 3.2 The golden rules

1. **The `KEY` column must return the same values as the Deliver By element.** If Deliver By is VENDOR_SITE_CODE, KEY must be the vendor site code. They must match exactly (case, spaces, data type).
2. **Deliver By must be at the same level as Split By or above it (a parent).** Each document must carry exactly **one** Deliver By value.
   - ✅ Split by Invoice, Deliver by Site: each invoice belongs to one site.
   - ❌ Split by Site, Deliver by Invoice: one site document holds many invoices, so BIP can't tell which one to use.
3. **Pick elements from a repeating group.** Split By points to an element inside a group (G_1, G_2…) that repeats once per value you want to cut on.
4. **Number of emails = number of documents × matching KEY rows.**
   - 1 matching row → delivered once.
   - 0 matching rows → **not delivered**.
   - 2 matching rows → **delivered twice** (useful on purpose for email + FTP; a bug if accidental).

### 3.3 When are they the same, and when are they different?

**Most of the time they're the same** (your case): "one document per site, sent to that site".

They differ when **what you cut on** is not **who you send to**:

| Scenario | Split By | Deliver By | KEY in query | Result |
|---|---|---|---|---|
| Each site gets its own statement | Site | Site | site code | 1 email per site |
| Each **invoice** as its own PDF, sent to the site's email | Invoice ID | Site | site code | A site with 5 invoices gets **5 emails** |
| Each **site** document sent to the **supplier's** head-office email | Site | Supplier | vendor id/name | Supplier with 3 sites gets **3 emails**, one per site |
| Payslip per employee | Person ID | Person ID | person id | 1 email per employee |
| Department summary to the department manager | Department | Department | department id | 1 email per department (KEY row carries the manager's email) |

> **Subtle point (good for interviews):** in "split by Invoice, deliver by Site", all 5 invoice PDFs use the **same** KEY row, so they get the **same file name and subject**. If each invoice needs its own name (`INV-1001.pdf`), use **Deliver By = Invoice ID** and write the bursting query at invoice level (KEY = invoice_id), joining to the site for the email address.

### 3.4 How to decide, in two questions

```
Q1. "One document per WHAT?"          → that's your SPLIT BY
Q2. "Whose delivery details apply?"  → that's your DELIVER BY
     (Usually the same thing. If not, Deliver By must be the same level or a parent.)
     Then write the bursting query so KEY = the Deliver By value.
```

---

## Part 4: The Bursting Query, Column by Column

The column **aliases are fixed names** that BIP recognizes. The values can be hard-coded or come from tables.

### 4.1 Common columns (for every channel)

| Column | Meaning | Example values |
|---|---|---|
| **KEY** | Must match the **Deliver By** element value | `vendor_site_code` |
| **TEMPLATE** | **Layout name** as defined in the report (not the file name). Case-sensitive | `'T1'` |
| **LOCALE** | Language/territory of the template translation; also drives date and number formatting | `'en-US'`, `'fr-FR'`, `'de-DE'` |
| **OUTPUT_FORMAT** | Output type | `'PDF'`, `'RTF'`, `'HTML'`, `'EXCEL'`, `'XLSX'`, `'CSV'`, `'TEXT'` (the template must support the format) |
| **SAVE_OUTPUT** | `'true'` = keep a copy in **Report Job History** so you can view or republish it | `'true'` |
| **DEL_CHANNEL** | Delivery method | `'EMAIL'`, `'FTP'`, `'FILE'`, `'PRINT'`, `'FAX'`, `'WEBDAV'`, `'WCC'` (Content Server/UCM) |
| **OUTPUT_NAME** | File name of the generated document | `'Invoices_' \|\| vendor_site_code \|\| '.pdf'` |
| **TIMEZONE** *(optional)* | Time zone for dates | `'Asia/Calcutta'` |
| **CALENDAR** *(optional)* | Calendar type | `'GREGORIAN'` |
| **PARAMETER1 – PARAMETER10** | Meaning **depends on DEL_CHANNEL** | see 4.2 |

### 4.2 What PARAMETER1–10 mean per channel

**EMAIL** (your case):

| Param | Meaning |
|---|---|
| PARAMETER1 | **To** (multiple addresses: comma-separated) |
| PARAMETER2 | **Cc** |
| PARAMETER3 | **From** |
| PARAMETER4 | **Subject** |
| PARAMETER5 | **Body** |
| PARAMETER6 | **Attachment**: `'true'` attaches the file. **Must be `'true'` for PDF**, otherwise the PDF isn't attached |
| PARAMETER7 | **Reply-To** |
| PARAMETER8 | **Bcc** |

**FTP / SFTP:**

| Param | Meaning |
|---|---|
| PARAMETER1 | Server name (as configured by the admin in BIP delivery settings) |
| PARAMETER2 | Username |
| PARAMETER3 | Password |
| PARAMETER4 | Remote directory |
| PARAMETER5 | Remote file name |
| PARAMETER6 | `'true'` = secure (SFTP) |

**FILE:** PARAMETER1 = directory, PARAMETER2 = file name.
**PRINT:** PARAMETER1 = printer group, PARAMETER2 = printer, PARAMETER3 = copies, PARAMETER4 = sides, PARAMETER5 = tray, PARAMETER6 = page range.
**FAX:** PARAMETER1 = fax server, PARAMETER2 = fax number.

> Servers (email, FTP, printers) must be set up by an admin under BIP **Administration → Delivery**. Your query only refers to them by name.

---

## Part 5: Your Query, Explained and Improved

### 5.1 What your query does, line by line

```sql
SELECT vendor_site_code                     KEY,            -- matches Deliver By (site code)
       'T1'                                 TEMPLATE,       -- use layout named T1
       'en-US'                              LOCALE,         -- English (US) version
       'PDF'                                OUTPUT_FORMAT,  -- generate PDF
       'true'                               SAVE_OUTPUT,    -- keep copy in job history
       'EMAIL'                              DEL_CHANNEL,    -- send by email
       'Invoices_' || vendor_site_code || '.pdf' OUTPUT_NAME, -- e.g. Invoices_ACME-NYC.pdf
       'jagannath.tripathy@gmail.com'       PARAMETER1,     -- To
       ''                                   PARAMETER2,     -- Cc (none)
       'no_reply@cognizant.com'             PARAMETER3,     -- From
       'Invoices for ' || vendor_site_code  PARAMETER4,     -- Subject
       'Hi ' || vendor_site_code || ', ...' PARAMETER5,     -- Body
       'true'                               PARAMETER6,     -- attach the PDF
       'no_reply@cognizant.com'             PARAMETER7,     -- Reply-To
       ''                                   PARAMETER8      -- Bcc (none)
FROM   poz_supplier_sites_v
```

At runtime:
1. The data model runs with your parameters (e.g. supplier = ACME). The XML has one G_1 per site.
2. **Split By** `/DATA_DS/G_1/VENDOR_SITE_CODE` → one document per site (ACME-NYC, ACME-LON).
3. For each document, **Deliver By** value "ACME-NYC" is matched to `KEY = 'ACME-NYC'` in your query.
4. That row says: template T1, PDF, email to jagannath…, subject "Invoices for ACME-NYC", attach the PDF.
5. Same for ACME-LON.

**For training this works**, and everything goes to one test inbox, which is a sensible way to test.

### 5.2 Problems a reviewer would spot

| # | Issue | What goes wrong | Fix |
|---|---|---|---|
| 1 | **No WHERE clause**: returns **every site of every supplier** | Wasteful, and **site codes aren't unique across suppliers** (many suppliers have a site called `MAIN` or `HQ`). If your supplier has site `MAIN` and 40 other suppliers do too, **KEY 'MAIN' matches 41 rows, so the same PDF is emailed 41 times** | Filter to the report's supplier: `WHERE vendor_id = :P_VENDOR_ID` (bursting queries can use the data model's parameters) |
| 2 | Matching on a **code** instead of an **ID** | Codes can repeat; IDs are unique | Add `ai.VENDOR_SITE_ID` to the INV data set, split/deliver by it, and use `vendor_site_id KEY` |
| 3 | **Hard-coded recipient** | Fine for testing; in production each site should get its own email | Take the email from the supplier site/contact data (see 5.3) |
| 4 | Body says "Hi ACME-NYC" | Greets a site code, not a company | Use the supplier name |
| 5 | Body has line breaks typed inside the string | Works, but is fragile when copying the query | Use `CHR(10)` for new lines |

### 5.3 Production-style version

```sql
SELECT vs.vendor_site_code                                   KEY,
       'T1'                                                  TEMPLATE,
       'en-US'                                               LOCALE,
       'PDF'                                                 OUTPUT_FORMAT,
       'true'                                                SAVE_OUTPUT,
       'EMAIL'                                               DEL_CHANNEL,
       'Invoices_' || vs.vendor_site_code || '.pdf'          OUTPUT_NAME,
       vs.email_address                                      PARAMETER1,  -- To: site's email*
       'ap.team@yourcompany.com'                             PARAMETER2,  -- Cc: AP team keeps a copy
       'no_reply@cognizant.com'                              PARAMETER3,  -- From
       'Invoice statement - ' || v.vendor_name
             || ' (' || vs.vendor_site_code || ')'           PARAMETER4,  -- Subject
       'Dear ' || v.vendor_name || ',' || CHR(10) || CHR(10) ||
       'Please find attached the invoice statement for site '
             || vs.vendor_site_code || '.' || CHR(10) || CHR(10) ||
       'Regards,' || CHR(10) || 'Accounts Payable Team'      PARAMETER5,  -- Body
       'true'                                                PARAMETER6,  -- attach
       'ap.team@yourcompany.com'                             PARAMETER7,  -- Reply-To
       NULL                                                  PARAMETER8   -- Bcc
FROM   poz_supplier_sites_v vs,
       poz_suppliers_v      v
WHERE  vs.vendor_id = v.vendor_id
AND    vs.vendor_id = :P_VENDOR_ID          -- only this report's supplier → no duplicate KEYs
AND    vs.email_address IS NOT NULL         -- no email = nothing to deliver
```

\* *Where the supplier email lives varies by setup: site communication email, supplier contacts, or the remittance advice email (`IBY_EXTERNAL_PAYEES_ALL.REMIT_ADVICE_EMAIL`). Check the column names in your instance before relying on `email_address`.*

### 5.4 Template for writing any bursting query

```sql
SELECT <same value as Deliver By element>   KEY,
       <layout name>                        TEMPLATE,
       <locale>                             LOCALE,
       <format>                             OUTPUT_FORMAT,
       'true'                               SAVE_OUTPUT,
       <channel>                            DEL_CHANNEL,
       <file name>                          OUTPUT_NAME,
       <PARAMETER1..n as per channel>
FROM   <table that has one row per recipient + their contact details>
WHERE  <restrict to the same population as the report> 
```

---

## Part 6: Bursting with Different Languages

### Situation
Your suppliers are in India, France and Germany. Each should get the statement in **its own language**, with local date and number formats (`12/11/2026` vs `12.11.2026`, `1,000.50` vs `1.000,50`).

### What you need
1. **Translations of the template.** In the report, the layout T1 needs locale versions, either:
   - **separate localized RTF files** uploaded for the same layout with locale `fr-FR`, `de-DE`; or
   - **XLIFF translation files** (translate the template's text strings and upload them per locale).
2. **The LOCALE column** in the bursting query, driven by data (the supplier's country or language).

If no translation exists for a locale, BIP typically falls back to the base template, so always keep a default (e.g. `en-US`).

### Query

```sql
SELECT vs.vendor_site_code                                  KEY,
       'T1'                                                 TEMPLATE,
       CASE loc.country
            WHEN 'FR' THEN 'fr-FR'
            WHEN 'DE' THEN 'de-DE'
            WHEN 'ES' THEN 'es-ES'
            ELSE 'en-US'
       END                                                  LOCALE,          -- ← the key change
       'PDF'                                                OUTPUT_FORMAT,
       'true'                                               SAVE_OUTPUT,
       'EMAIL'                                              DEL_CHANNEL,
       'Invoices_' || vs.vendor_site_code || '.pdf'         OUTPUT_NAME,
       vs.email_address                                     PARAMETER1,
       NULL                                                 PARAMETER2,
       'no_reply@cognizant.com'                             PARAMETER3,
       CASE loc.country                                                       -- subject in their language
            WHEN 'FR' THEN 'Relevé de factures - '
            WHEN 'DE' THEN 'Rechnungsübersicht - '
            ELSE 'Invoice statement - '
       END || v.vendor_name                                 PARAMETER4,
       CASE loc.country                                                       -- body in their language
            WHEN 'FR' THEN 'Bonjour,' || CHR(10) || 'Veuillez trouver ci-joint vos factures.'
            WHEN 'DE' THEN 'Guten Tag,' || CHR(10) || 'Anbei finden Sie Ihre Rechnungen.'
            ELSE 'Hello,' || CHR(10) || 'Please find your invoices attached.'
       END                                                  PARAMETER5,
       'true'                                               PARAMETER6
FROM   poz_supplier_sites_v vs,
       poz_suppliers_v      v,
       hz_locations         loc
WHERE  vs.vendor_id   = v.vendor_id
AND    vs.location_id = loc.location_id          -- site address → country
AND    vs.vendor_id   = :P_VENDOR_ID
```

**Key takeaways:**
- **LOCALE** picks *which translation of the template* is used and *how dates and numbers are formatted*.
- The **email subject and body** aren't translated by the template. You translate them yourself in the query (CASE, or a mapping/lookup table on bigger projects).
- **Cleaner on real projects:** keep a small mapping (country → locale, subject, body) in a lookup or custom table and join to it, instead of long CASE statements.

---

## Part 7: Bursting with Different Templates

### Situation
One data model, several layouts in the report:

| Layout name | Purpose |
|---|---|
| `T_STANDARD` | Normal PDF statement |
| `T_INDIA_GST` | Indian suppliers: extra GST columns/legal text |
| `T_EXCEL` | Suppliers who want Excel to reconcile in their own systems |

Each recipient should get the right one.

### Rules
1. Every template name used must **exist as a layout in the report** (exact name, case-sensitive).
2. **OUTPUT_FORMAT must suit the template**: an Excel template → `'XLSX'`/`'EXCEL'`; an RTF can give PDF/RTF/Excel/HTML.
3. Make **OUTPUT_NAME's extension match** the format.

### Query

```sql
SELECT vs.vendor_site_code                                       KEY,
       CASE
            WHEN loc.country = 'IN'               THEN 'T_INDIA_GST'
            WHEN vs.preferred_format = 'EXCEL'    THEN 'T_EXCEL'      -- *see note
            ELSE 'T_STANDARD'
       END                                                       TEMPLATE,       -- ← the key change
       'en-US'                                                   LOCALE,
       CASE
            WHEN vs.preferred_format = 'EXCEL' THEN 'XLSX'
            ELSE 'PDF'
       END                                                       OUTPUT_FORMAT,  -- must match template
       'true'                                                    SAVE_OUTPUT,
       'EMAIL'                                                   DEL_CHANNEL,
       'Invoices_' || vs.vendor_site_code ||
       CASE WHEN vs.preferred_format = 'EXCEL' THEN '.xlsx' ELSE '.pdf' END
                                                                 OUTPUT_NAME,    -- extension matches
       vs.email_address                                          PARAMETER1,
       NULL                                                      PARAMETER2,
       'no_reply@cognizant.com'                                  PARAMETER3,
       'Invoice statement - ' || v.vendor_name                   PARAMETER4,
       'Please find your invoice statement attached.'            PARAMETER5,
       'true'                                                    PARAMETER6
FROM   poz_supplier_sites_v vs,
       poz_suppliers_v      v,
       hz_locations         loc
WHERE  vs.vendor_id   = v.vendor_id
AND    vs.location_id = loc.location_id
AND    vs.vendor_id   = :P_VENDOR_ID
```

\* *`preferred_format` is illustrative. A standard table won't have it. On a project, this preference usually comes from a descriptive flexfield (DFF) attribute on the supplier site, or a custom lookup table mapping site → format.*

### Combining both (language and template)
TEMPLATE, LOCALE and OUTPUT_FORMAT are **independent columns**, so you can drive all three from data in the same query: e.g. a French supplier gets `T_STANDARD` in `fr-FR` as PDF, and an Indian supplier gets `T_INDIA_GST` in `en-US` as PDF.

### Same document to two destinations (bonus)
Need the PDF **emailed to the supplier** and **dropped on an SFTP server** for an archive? Return **two rows with the same KEY** using `UNION ALL`:

```sql
SELECT vs.vendor_site_code KEY, 'T1' TEMPLATE, 'en-US' LOCALE, 'PDF' OUTPUT_FORMAT,
       'true' SAVE_OUTPUT, 'EMAIL' DEL_CHANNEL,
       'Invoices_' || vs.vendor_site_code || '.pdf' OUTPUT_NAME,
       vs.email_address PARAMETER1, NULL PARAMETER2, 'no_reply@cognizant.com' PARAMETER3,
       'Your invoices' PARAMETER4, 'Attached.' PARAMETER5, 'true' PARAMETER6
FROM   poz_supplier_sites_v vs
WHERE  vs.vendor_id = :P_VENDOR_ID

UNION ALL

SELECT vs.vendor_site_code KEY, 'T1', 'en-US', 'PDF',
       'true', 'FTP',
       'Invoices_' || vs.vendor_site_code || '.pdf',
       'ARCHIVE_SFTP'      PARAMETER1,   -- server name configured by admin
       'ftp_user'          PARAMETER2,
       'ftp_password'      PARAMETER3,   -- don't hard-code real passwords; follow project standards
       '/outbound/ap'      PARAMETER4,   -- remote directory
       'Invoices_' || vs.vendor_site_code || '.pdf' PARAMETER5,  -- remote file name
       'true'              PARAMETER6    -- secure (SFTP)
FROM   poz_supplier_sites_v vs
WHERE  vs.vendor_id = :P_VENDOR_ID
```

This is the same mechanism that caused the duplicate-email bug in Part 5.2: **two rows with the same KEY means two deliveries.** Here it's on purpose.

---

## Part 8: Step-by-Step Setup and Scheduling

### Step 1: Prepare the data model
- Make sure the element you'll split and deliver by is in the XML (ideally an **ID**).
- **View Data → Save as Sample Data.**

### Step 2: Create the report and layout(s)
- Create Report → select the data model → upload/create layout(s).
- Note the **exact layout names** (e.g. `T1`). That's what goes in the TEMPLATE column.

### Step 3: Define bursting (in the data model)
1. Open the data model → left panel → **Bursting** → click **+**.
2. **Name:** `Burst_By_Site`. **Type:** SQL Query. **Data Source:** ApplicationDB_FSCM.
3. **Split By:** `/DATA_DS/G_1/VENDOR_SITE_CODE`.
4. **Deliver By:** `/DATA_DS/G_1/VENDOR_SITE_CODE`.
5. Paste the bursting SQL → **Save** the data model.

### Step 4: Enable bursting on the report
1. Open the report → **Edit** → **Properties** (top right).
2. **General** tab → check **Enable Bursting** → select `Burst_By_Site`.
3. OK → **Save** the report.

### Step 5: Schedule the report
> Bursting **only happens when the report is scheduled.** Viewing the report online just shows the normal output.

1. In the catalog, find the report → **More → Schedule** (or open the report → **Actions → Schedule**).
2. **General** tab: enter the **parameter values** (e.g. supplier = ACME).
3. **Output** tab: check **"Use Bursting Definition to Determine Output and Delivery Destination"**. This tells BIP to take template, format and delivery from your bursting query instead of from this screen.
4. **Schedule** tab: choose the frequency:
   - **Once** (now or at a date/time), for testing
   - **Daily / Weekly / Monthly / Hourly**, e.g. every Monday 9 AM for weekly supplier statements
   - set start and end dates
5. **Notification** tab *(optional)*: email yourself when the job **succeeds / fails / has warnings**.
6. **Submit** → give the job a name (e.g. `Weekly_Supplier_Invoice_Burst`).

### Step 6: Monitor
1. **Report Job History** (catalog header → Report Job History) → find your job.
2. Open it to see **status**, and **each bursted document** with its delivery status (Success/Failed).
3. If `SAVE_OUTPUT = 'true'`, you can **view or download** each output and **republish** failed deliveries.
4. **Report Jobs** lists *future/recurring* schedules, where you can edit, suspend or delete them.

> In Fusion, reports can also be registered as **ESS jobs** and submitted from **Scheduled Processes**. The bursting definition still lives in the data model.

### Testing checklist
- [ ] Test with **your own email** in PARAMETER1 first (as you did, which was the right call).
- [ ] Run the bursting SQL on its own: **KEY values must match** the split values in your XML (open the sample XML and compare).
- [ ] No duplicate KEYs (unless intended).
- [ ] TEMPLATE names exactly match layout names.
- [ ] PARAMETER6 = `'true'` for PDF attachments.
- [ ] Schedule **Once** → check job history → then set the real recurrence.

---

## Part 9: Situational Questions (Practice Like an Interview)

**S1. Each supplier site should receive only its own invoices. What are Split By and Deliver By?**
Both = vendor site (preferably VENDOR_SITE_ID). KEY = vendor site id/code. One document per site, delivered using that site's row.

**S2. AP wants every invoice as a separate PDF named after the invoice number, emailed to the supplier site.**
Split By = INVOICE_ID (invoice group). Deliver By = **INVOICE_ID**, with the bursting query at invoice level (`FROM ap_invoices_all ai JOIN poz_supplier_sites_v vs ...`, KEY = invoice_id, OUTPUT_NAME = invoice_num || '.pdf', email from the site). Deliver By = site would also send them, but every file would get the same name and subject from the single site row.

**S3. A supplier with 3 sites wants one email per site, but all sent to their head-office email.**
Split By = site (3 documents). Deliver By = supplier (parent level). KEY = vendor_id, with the head-office email. The head office gets 3 emails.

**S4. The supplier wants one combined document for all their sites.**
Split By = supplier (the group containing VENDOR_NAME/VENDOR_ID). Deliver By = supplier. The sites appear inside the one document.

**S5. Can Split By be "Site" and Deliver By be "Invoice"? Why not?**
No. Deliver By must be at the same level as Split By or a parent. A site document contains many invoices, so there's no single invoice value to match against KEY.

**S6. The job succeeded but nobody got an email. What do you check?**
(1) Report was scheduled with "Use Bursting Definition…" checked. (2) KEY values actually match the Deliver By values (case, spaces, code vs ID, number vs string). (3) Recipient column isn't NULL. (4) Email server configured and From address allowed. (5) Bursting enabled in report properties. (6) Job history delivery status/error message.

**S7. A supplier complained they received the same email 12 times.**
The bursting query returns **12 rows with that KEY**, e.g. KEY = site code like `MAIN` shared across suppliers, or a join to contacts returning multiple rows. Fix: key on unique IDs, filter to the report population, and use DISTINCT or fix the join. If they truly have many contacts, put all addresses comma-separated in PARAMETER1 in one row (e.g. `LISTAGG(email, ',')`).

**S8. Some sites received emails, others didn't.**
Those sites have no matching KEY row: missing email (filtered out), site not returned by the query's WHERE, or code mismatch. Compare the distinct split values in the XML with the KEY list.

**S9. Same PDF must go to the supplier by email and to an SFTP archive.**
Two rows per KEY via `UNION ALL`: one with DEL_CHANNEL `'EMAIL'`, one with `'FTP'` (PARAMETER6 `'true'` for secure).

**S10. French suppliers need French documents and French date formats.**
Add French translations (localized RTF or XLIFF) to the layout. In the bursting query, set `LOCALE = 'fr-FR'` for French suppliers (CASE on country or a mapping table). Translate the subject and body in the query too.

**S11. Indian suppliers need a GST layout; others get the standard one.**
Add both layouts to the report. In the bursting query, `TEMPLATE = CASE WHEN country = 'IN' THEN 'T_INDIA_GST' ELSE 'T_STANDARD' END`.

**S12. Some suppliers want Excel, others PDF.**
Drive TEMPLATE, OUTPUT_FORMAT and OUTPUT_NAME's extension from a per-site preference (e.g. a DFF on the supplier site). Make sure the template supports the chosen format.

**S13. The PDF is generated but the email arrives without the attachment.**
PARAMETER6 isn't `'true'`.

**S14. Can the bursting query use the report's parameters?**
Yes. Data model parameters like `:P_VENDOR_ID` can be referenced, which is how you keep the bursting population the same as the report's.

**S15. Why does nothing get burst when I click View Report?**
Bursting runs only on **scheduled** jobs with the bursting option selected on the Output tab.

**S16. Payslips must go to each employee's work email. (HCM version.)**
Split By = PERSON_ID, Deliver By = PERSON_ID. KEY = person_id from `PER_EMAIL_ADDRESSES` (work email type, valid for today) joined to the person. Same concept, different module.

---

## Quick Revision Card

```
SPLIT BY      → one document per distinct value       ("how do I cut?")
DELIVER BY    → value matched to KEY                   ("whose address label?")
KEY           → must equal Deliver By value exactly
Deliver By    → same level as Split By, or a parent
0 KEY rows    → not delivered | 1 → once | 2+ → multiple deliveries
TEMPLATE      → layout NAME in the report (case-sensitive)
LOCALE        → which translation + date/number format
OUTPUT_FORMAT → must be supported by the template
EMAIL params  → P1 To, P2 Cc, P3 From, P4 Subject, P5 Body, P6 Attach('true'), P7 Reply-To, P8 Bcc
FTP params    → P1 Server, P2 User, P3 Pwd, P4 Dir, P5 File, P6 Secure
Bursting runs → only when SCHEDULED with "Use Bursting Definition…" checked
Enable it     → Report → Properties → Enable Bursting → choose definition
```