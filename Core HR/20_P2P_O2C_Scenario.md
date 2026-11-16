# P2P & O2C — Scenario-Based Interview Prep
### "Here's a business problem. Which tables, which joins, and how would you write it?"

> **Who's asking:** a senior ERP platform developer with ~20 years of experience. They won't ask *"What is PO_LINE_LOCATIONS_ALL?"* They'll describe a **real business problem** (an angry supplier, a month-end close, an audit finding) and watch **how you think**.
>
> **What they're grading:**
> 1. Can you turn **business words** into **tables**?
> 2. Do you know the **grain** (what one row means)?
> 3. Do you pick the **right join path** and avoid **double counting**?
> 4. Do you think about **status filters, cancelled/reversed data, BU security and performance**?
> 5. Can you handle the **follow-up twist**?
>
> **How to use this file:** read Part 1 (the method + table cards). Then for each scenario, **cover the solution** (it's folded under "Show solution"), write your answer on paper, and then compare.

---

## Table of Contents
- [Part 1 — The method](#part-1--the-method)
  - [1.1 The 5-step scenario attack](#11-the-5-step-scenario-attack)
  - [1.2 Keyword → table dictionary](#12-keyword--table-dictionary)
  - [1.3 Table memory cards (what each table holds)](#13-table-memory-cards)
- [Part 2 — 10 P2P scenarios](#part-2--10-p2p-scenarios)
- [Part 3 — 10 O2C scenarios](#part-3--10-o2c-scenarios)
- [Part 4 — Senior-level follow-ups that apply to any scenario](#part-4--senior-level-follow-ups-that-apply-to-any-scenario)
- [Part 5 — Practice tracker](#part-5--practice-tracker)

---

## Part 1 — The method

### 1.1 The 5-step scenario attack

Say these steps **out loud** in the interview. Seniors love structured thinking.

| Step | What you say | Example (for "supplier says they're not paid") |
|------|--------------|-----------------------------------------------|
| **1. Underline the nouns** | "The key business objects here are…" | supplier, delivery, invoice, payment |
| **2. Map nouns → tables** | "That maps to…" | POZ_SUPPLIERS_V, RCV_*, AP_INVOICES_ALL, AP_CHECKS_ALL |
| **3. Decide the grain** | "One row of my output is one…" | one invoice (or one PO schedule) |
| **4. Walk the join path** | "I'd go from X to Y through…" | PO → RCV_TRANSACTIONS → AP dist → AP invoice → AP_INVOICE_PAYMENTS → AP_CHECKS |
| **5. Filters & traps** | "I'd exclude… and watch out for…" | cancelled invoices, voided checks, reversed rows, fan-out, BU filter |

> 💡 **Grain is the #1 thing seniors check.** If you say *"one row per invoice"* and then join to distributions without aggregating, they'll catch it.

### 1.2 Keyword → table dictionary

When you **hear** these words in a scenario, **think** of these tables.

#### P2P
| If you hear… | Think |
|--------------|-------|
| supplier, vendor, supplier number, pay site, inactive supplier | `POZ_SUPPLIERS_V`, `POZ_SUPPLIER_SITES_V` |
| PO number, buyer, PO status, PO type, currency | `PO_HEADERS_ALL` |
| item on PO, unit price, ordered quantity, category | `PO_LINES_ALL` |
| **need-by date, promised date, ship-to org, pending qty, received qty, billed qty, 2/3/4-way match** | `PO_LINE_LOCATIONS_ALL` (schedule) |
| **charge account, cost center, department, project, expense vs inventory** | `PO_DISTRIBUTIONS_ALL` |
| receipt number, GRN, receipt date, "goods arrived" | `RCV_SHIPMENT_HEADERS` |
| **received / accepted / rejected / delivered / returned / corrected, inspection** | `RCV_TRANSACTIONS` |
| item code, item description, UOM, item status | `EGP_SYSTEM_ITEMS_VL` |
| stock movement, stock history, "when did stock go up/down" | `INV_MATERIAL_TXNS` |
| stock in hand, current stock, available qty | `INV_ONHAND_QUANTITIES_DETAIL` |
| supplier invoice number, invoice amount, invoice type, credit memo, cancelled invoice | `AP_INVOICES_ALL` |
| freight/tax/misc line on invoice, matched to PO line | `AP_INVOICE_LINES_ALL` |
| **validated, accounted/posted, invoice charge account, matched to PO/receipt** | `AP_INVOICE_DISTRIBUTIONS_ALL` |
| **due date, overdue, amount remaining, installment, hold on payment** | `AP_PAYMENT_SCHEDULES_ALL` |
| cheque, EFT, wire, payment number, voided, cleared | `AP_CHECKS_ALL` |
| "which invoices did this payment pay", partial payment | `AP_INVOICE_PAYMENTS_ALL` |
| journal, posted, period, ledger, debit/credit | `GL_JE_BATCHES`, `GL_JE_HEADERS`, `GL_JE_LINES` |

#### O2C
| If you hear… | Think |
|--------------|-------|
| customer name, organization, person | `HZ_PARTIES` |
| customer account, account number, customer type | `HZ_CUST_ACCOUNTS` |
| address, city, state, postal code | `HZ_LOCATIONS` (via `HZ_PARTY_SITES`) |
| **bill-to, ship-to, primary address** | `HZ_CUST_SITE_USES_ALL` (+ `HZ_CUST_ACCT_SITES_ALL`) |
| order number, order date, order status, source system, change order | `DOO_HEADERS_ALL` |
| ordered item, ordered qty, selling price, return line | `DOO_LINES_ALL` |
| **warehouse, schedule ship date, shipped qty, backorder, split shipment, billing status** | `DOO_FULFILL_LINES_ALL` |
| **pick release, picked, staged, backordered, shipped (per item)** | `WSH_DELIVERY_DETAILS` |
| delivery number, tracking/waybill, ship date, truck/consignment | `WSH_NEW_DELIVERIES` |
| "which items are on which delivery", packing | `WSH_DELIVERY_ASSIGNMENTS` |
| invoice / credit memo / debit memo number, transaction date | `RA_CUSTOMER_TRX_ALL` |
| invoice line, tax line, freight line, qty invoiced | `RA_CUSTOMER_TRX_LINES_ALL` |
| **open balance, due date, overdue, aging, outstanding** | `AR_PAYMENT_SCHEDULES_ALL` |
| customer payment, receipt, cheque/UPI received, unapplied, unidentified, bounced | `AR_CASH_RECEIPTS_ALL` |
| **"applied to which invoice", on-account, credit memo applied** | `AR_RECEIVABLE_APPLICATIONS_ALL` |

### 1.3 Table memory cards

**Grain** = what **one row** means. **Must-know columns** = enough to recognize the table in a scenario.

#### P2P cards

| Table | Grain (1 row =) | Must-know columns |
|-------|-----------------|-------------------|
| `POZ_SUPPLIERS_V` | one supplier | VENDOR_ID, SEGMENT1 (supplier no.), VENDOR_NAME, PARTY_ID, ENABLED_FLAG, END_DATE_ACTIVE |
| `POZ_SUPPLIER_SITES_V` | one supplier site per procurement BU | VENDOR_SITE_ID, VENDOR_ID, VENDOR_SITE_CODE, PRC_BU_ID, PURCHASING_SITE_FLAG, PAY_SITE_FLAG, INACTIVE_DATE |
| `PO_HEADERS_ALL` | one PO | PO_HEADER_ID, SEGMENT1 (PO no.), TYPE_LOOKUP_CODE, VENDOR_ID, VENDOR_SITE_ID, AGENT_ID (buyer), DOCUMENT_STATUS, PRC_BU_ID, CURRENCY_CODE, CREATION_DATE |
| `PO_LINES_ALL` | one item line on a PO | PO_LINE_ID, PO_HEADER_ID, LINE_NUM, ITEM_ID, ITEM_DESCRIPTION, CATEGORY_ID, QUANTITY, UNIT_PRICE, UOM_CODE |
| `PO_LINE_LOCATIONS_ALL` | one **schedule** (when/where) | LINE_LOCATION_ID, PO_LINE_ID, PO_HEADER_ID, SHIP_TO_ORGANIZATION_ID, NEED_BY_DATE, PROMISED_DATE, QUANTITY, QUANTITY_RECEIVED, QUANTITY_ACCEPTED, QUANTITY_BILLED, QUANTITY_CANCELLED, RECEIPT_REQUIRED_FLAG, INSPECTION_REQUIRED_FLAG |
| `PO_DISTRIBUTIONS_ALL` | one **charge account split** | PO_DISTRIBUTION_ID, LINE_LOCATION_ID, PO_LINE_ID, PO_HEADER_ID, CODE_COMBINATION_ID, QUANTITY_ORDERED, QUANTITY_DELIVERED, QUANTITY_BILLED, DESTINATION_TYPE_CODE |
| `RCV_SHIPMENT_HEADERS` | one receipt (truck) | SHIPMENT_HEADER_ID, RECEIPT_NUM, VENDOR_ID, VENDOR_SITE_ID, RECEIPT_SOURCE_CODE, SHIP_TO_ORG_ID, CREATION_DATE |
| `RCV_SHIPMENT_LINES` | one received item line | SHIPMENT_LINE_ID, SHIPMENT_HEADER_ID, ITEM_ID, QUANTITY_RECEIVED, PO_HEADER_ID, PO_LINE_ID, PO_LINE_LOCATION_ID, SHIPMENT_LINE_STATUS_CODE |
| `RCV_TRANSACTIONS` | one **action** on received goods | TRANSACTION_ID, PARENT_TRANSACTION_ID, TRANSACTION_TYPE, TRANSACTION_DATE, QUANTITY, DESTINATION_TYPE_CODE, PO_HEADER_ID, PO_LINE_ID, PO_LINE_LOCATION_ID, PO_DISTRIBUTION_ID, INV_TRANSACTION_ID |
| `EGP_SYSTEM_ITEMS_VL` | one item **per org** | INVENTORY_ITEM_ID, ORGANIZATION_ID, ITEM_NUMBER, DESCRIPTION, PRIMARY_UOM_CODE, INVENTORY_ITEM_STATUS_CODE |
| `INV_MATERIAL_TXNS` | one stock movement | TRANSACTION_ID, INVENTORY_ITEM_ID, ORGANIZATION_ID, SUBINVENTORY_CODE, LOCATOR_ID, TRANSACTION_QUANTITY (+/−), TRANSACTION_DATE, TRANSACTION_SOURCE_TYPE_ID |
| `INV_ONHAND_QUANTITIES_DETAIL` | one **fragment** of current stock | INVENTORY_ITEM_ID, ORGANIZATION_ID, SUBINVENTORY_CODE, LOCATOR_ID, LOT_NUMBER, PRIMARY_TRANSACTION_QUANTITY |
| `AP_INVOICES_ALL` | one supplier invoice | INVOICE_ID, INVOICE_NUM, VENDOR_ID, VENDOR_SITE_ID, INVOICE_TYPE_LOOKUP_CODE, INVOICE_AMOUNT, AMOUNT_PAID, PAYMENT_STATUS_FLAG, INVOICE_DATE, GL_DATE, CANCELLED_DATE, ORG_ID |
| `AP_INVOICE_LINES_ALL` | one invoice line | INVOICE_ID, LINE_NUMBER, LINE_TYPE_LOOKUP_CODE, AMOUNT, QUANTITY_INVOICED, UNIT_PRICE, PO_HEADER_ID, PO_LINE_ID, PO_LINE_LOCATION_ID, MATCH_TYPE |
| `AP_INVOICE_DISTRIBUTIONS_ALL` | one accounting split of an invoice line | INVOICE_DISTRIBUTION_ID, INVOICE_ID, INVOICE_LINE_NUMBER, AMOUNT, DIST_CODE_COMBINATION_ID, PO_DISTRIBUTION_ID, RCV_TRANSACTION_ID, MATCH_STATUS_FLAG, POSTED_FLAG, REVERSAL_FLAG |
| `AP_PAYMENT_SCHEDULES_ALL` | one installment of an invoice | INVOICE_ID, PAYMENT_NUM, DUE_DATE, GROSS_AMOUNT, AMOUNT_REMAINING, PAYMENT_STATUS_FLAG, HOLD_FLAG |
| `AP_CHECKS_ALL` | one payment (cheque/EFT/wire) | CHECK_ID, CHECK_NUMBER, CHECK_DATE, AMOUNT, VENDOR_ID, VENDOR_SITE_ID, PAYMENT_METHOD_CODE, STATUS_LOOKUP_CODE, VOID_DATE |
| `AP_INVOICE_PAYMENTS_ALL` | one "payment ↔ invoice installment" link | INVOICE_PAYMENT_ID, CHECK_ID, INVOICE_ID, PAYMENT_NUM, AMOUNT, ACCOUNTING_DATE, REVERSAL_FLAG |
| `GL_JE_BATCHES` / `HEADERS` / `LINES` | batch / journal / journal line | JE_BATCH_ID, STATUS · JE_HEADER_ID, LEDGER_ID, PERIOD_NAME, JE_SOURCE, JE_CATEGORY · JE_LINE_NUM, CODE_COMBINATION_ID, ACCOUNTED_DR/CR |

#### O2C cards

| Table | Grain (1 row =) | Must-know columns |
|-------|-----------------|-------------------|
| `HZ_PARTIES` | one party (org/person) | PARTY_ID, PARTY_NUMBER, PARTY_NAME, PARTY_TYPE, STATUS |
| `HZ_CUST_ACCOUNTS` | one customer account | CUST_ACCOUNT_ID, PARTY_ID, ACCOUNT_NUMBER, ACCOUNT_NAME, CUSTOMER_TYPE, STATUS |
| `HZ_LOCATIONS` | one physical address | LOCATION_ID, ADDRESS1–4, CITY, STATE, POSTAL_CODE, COUNTRY |
| `HZ_PARTY_SITES` | one party ↔ address link | PARTY_SITE_ID, PARTY_ID, LOCATION_ID, PARTY_SITE_NUMBER, IDENTIFYING_ADDRESS_FLAG, STATUS |
| `HZ_PARTY_SITE_USES` | one purpose of a party site | PARTY_SITE_USE_ID, PARTY_SITE_ID, SITE_USE_TYPE, STATUS |
| `HZ_CUST_ACCT_SITES_ALL` | one account ↔ party site link | CUST_ACCT_SITE_ID, CUST_ACCOUNT_ID, PARTY_SITE_ID, SET_ID, STATUS |
| `HZ_CUST_SITE_USES_ALL` | one purpose of an account site | SITE_USE_ID, CUST_ACCT_SITE_ID, SITE_USE_CODE (BILL_TO/SHIP_TO), PRIMARY_FLAG, LOCATION (name), STATUS |
| `DOO_HEADERS_ALL` | one order **version** | HEADER_ID, ORDER_NUMBER, SOURCE_ORDER_NUMBER, SOLD_TO_PARTY_ID, ORDERED_DATE, STATUS_CODE, SUBMITTED_FLAG, CHANGE_VERSION_NUMBER, ORG_ID |
| `DOO_LINES_ALL` | one ordered item line | LINE_ID, HEADER_ID, LINE_NUMBER, INVENTORY_ITEM_ID, ORDERED_QTY, UNIT_SELLING_PRICE, EXTENDED_AMOUNT, CATEGORY_CODE (ORDER/RETURN), STATUS_CODE |
| `DOO_FULFILL_LINES_ALL` | one **shipment split** of a line | FULFILL_LINE_ID, LINE_ID, HEADER_ID, FULFILL_LINE_NUMBER, FULFILL_ORG_ID, ORDERED_QTY, SHIPPED_QTY, SCHEDULE_SHIP_DATE, ACTUAL_SHIP_DATE, STATUS_CODE, OPEN_FLAG, CANCELED_FLAG |
| `WSH_DELIVERY_DETAILS` | one shippable item quantity | DELIVERY_DETAIL_ID, SOURCE_HEADER_ID, SALES_ORDER_NUMBER, SALES_ORDER_LINE_NUMBER, SALES_ORDER_SHIPMENT_NUMBER, INVENTORY_ITEM_ID, ORGANIZATION_ID, REQUESTED_QUANTITY, SHIPPED_QUANTITY, RELEASED_STATUS |
| `WSH_DELIVERY_ASSIGNMENTS` | one detail ↔ delivery link | DELIVERY_DETAIL_ID, DELIVERY_ID, PARENT_DELIVERY_DETAIL_ID |
| `WSH_NEW_DELIVERIES` | one delivery (consignment) | DELIVERY_ID, NAME, STATUS_CODE, ORGANIZATION_ID, INITIAL_PICKUP_DATE, WAYBILL |
| `RA_CUSTOMER_TRX_ALL` | one invoice / CM / DM | CUSTOMER_TRX_ID, TRX_NUMBER, TRX_DATE, BILL_TO_CUSTOMER_ID, BILL_TO_SITE_USE_ID, SHIP_TO_SITE_USE_ID, PREVIOUS_CUSTOMER_TRX_ID, COMPLETE_FLAG, ORG_ID |
| `RA_CUSTOMER_TRX_LINES_ALL` | one invoice line (item/tax/freight) | CUSTOMER_TRX_LINE_ID, CUSTOMER_TRX_ID, LINE_TYPE, QUANTITY_INVOICED, UNIT_SELLING_PRICE, EXTENDED_AMOUNT, LINK_TO_CUST_TRX_LINE_ID, INTERFACE_LINE_ATTRIBUTE1/2 |
| `AR_PAYMENT_SCHEDULES_ALL` | one balance row per document/installment | PAYMENT_SCHEDULE_ID, CLASS, CUSTOMER_TRX_ID, CASH_RECEIPT_ID, CUSTOMER_ID, DUE_DATE, AMOUNT_DUE_ORIGINAL, AMOUNT_DUE_REMAINING, STATUS |
| `AR_CASH_RECEIPTS_ALL` | one customer payment | CASH_RECEIPT_ID, RECEIPT_NUMBER, RECEIPT_DATE, AMOUNT, PAY_FROM_CUSTOMER, STATUS, TYPE, REVERSAL_DATE |
| `AR_RECEIVABLE_APPLICATIONS_ALL` | one "money → document" application | RECEIVABLE_APPLICATION_ID, CASH_RECEIPT_ID, PAYMENT_SCHEDULE_ID, APPLIED_PAYMENT_SCHEDULE_ID, APPLIED_CUSTOMER_TRX_ID, AMOUNT_APPLIED, APPLY_DATE, STATUS, DISPLAY, APPLICATION_TYPE |

> ⚠️ Column names follow the standard Fusion data model. Always run a quick `SELECT * … WHERE ROWNUM < 5` in your instance before trusting a column. Saying this in the interview is a **plus**, because it shows real-world caution.

---

## Part 2 — 10 P2P scenarios

Difficulty: 🟢 warm-up · 🟡 standard · 🔴 senior-level twist

---

### P2P-1 🟡 "The supplier says they weren't paid"

> **Interviewer:** *"A supplier calls the AP helpdesk and says: 'We delivered 500 units against PO 10234 three weeks ago, and we still haven't been paid.' You're the developer. Build me something that tells the helpdesk **exactly where this PO is stuck**."*

**What they're really testing:** Can you trace the full P2P chain and turn it into a **diagnosis**, not just a dump of data?

<details><summary><b>Show solution</b></summary>

**Step 1–2: Nouns → tables**
| Noun | Table |
|------|-------|
| supplier | `POZ_SUPPLIERS_V` |
| PO 10234 | `PO_HEADERS_ALL` (SEGMENT1) |
| delivered 500 units | `RCV_TRANSACTIONS` (RECEIVE / DELIVER) |
| invoice | `AP_INVOICE_DISTRIBUTIONS_ALL` → `AP_INVOICES_ALL` |
| why not paid | `AP_PAYMENT_SCHEDULES_ALL` (HOLD_FLAG, AMOUNT_REMAINING), dist `MATCH_STATUS_FLAG` |
| paid? | `AP_INVOICE_PAYMENTS_ALL` → `AP_CHECKS_ALL` |

**Step 3: Grain.** One row per **PO schedule**, with a "stuck at" status column.

**Step 4: The diagnosis ladder.** Say this to the interviewer. It's what they want to hear:
```
Is the PO approved?            → PO_HEADERS_ALL.DOCUMENT_STATUS
Was anything received?         → PO_LINE_LOCATIONS_ALL.QUANTITY_RECEIVED > 0
Was an invoice entered?        → QUANTITY_BILLED > 0 / AP dist exists for the PO_DISTRIBUTION_ID
Is the invoice validated?      → AP_INVOICE_DISTRIBUTIONS_ALL.MATCH_STATUS_FLAG = 'A'
Is it on hold?                 → AP_PAYMENT_SCHEDULES_ALL.HOLD_FLAG = 'Y'
Is it due yet?                 → AP_PAYMENT_SCHEDULES_ALL.DUE_DATE
Was a payment made / voided?   → AP_INVOICE_PAYMENTS_ALL + AP_CHECKS_ALL.STATUS_LOOKUP_CODE
```

**Step 5: SQL**
```sql
WITH inv AS (   -- one row per PO distribution with invoice facts, aggregated
  SELECT  aid.PO_DISTRIBUTION_ID,
          MIN(aia.INVOICE_NUM)                                        AS invoice_num,
          MIN(aia.INVOICE_ID)                                         AS invoice_id,
          MAX(CASE WHEN NVL(aid.MATCH_STATUS_FLAG,'N') <> 'A' THEN 1 ELSE 0 END) AS not_validated
  FROM    AP_INVOICE_DISTRIBUTIONS_ALL aid
  JOIN    AP_INVOICES_ALL aia ON aia.INVOICE_ID = aid.INVOICE_ID
  WHERE   aia.CANCELLED_DATE IS NULL
  AND     NVL(aid.REVERSAL_FLAG,'N') = 'N'
  GROUP BY aid.PO_DISTRIBUTION_ID
)
SELECT  ps.VENDOR_NAME,
        pha.SEGMENT1                 AS po_number,
        pha.DOCUMENT_STATUS,
        plla.QUANTITY                AS ordered,
        plla.QUANTITY_RECEIVED       AS received,
        plla.QUANTITY_BILLED         AS billed,
        inv.invoice_num,
        apsa.DUE_DATE,
        apsa.HOLD_FLAG,
        apsa.AMOUNT_REMAINING,
        CASE
          WHEN pha.DOCUMENT_STATUS <> 'OPEN'                 THEN '1. PO not open/approved'
          WHEN NVL(plla.QUANTITY_RECEIVED,0) = 0             THEN '2. Not received in system'
          WHEN inv.invoice_id IS NULL                        THEN '3. No invoice entered'
          WHEN inv.not_validated = 1                         THEN '4. Invoice not validated'
          WHEN apsa.HOLD_FLAG = 'Y'                          THEN '5. Payment on hold'
          WHEN apsa.AMOUNT_REMAINING > 0 AND apsa.DUE_DATE > SYSDATE THEN '6. Not due yet'
          WHEN apsa.AMOUNT_REMAINING > 0                     THEN '7. Due, not paid. Check payment run'
          ELSE '8. Paid'
        END                          AS stuck_at
FROM    PO_HEADERS_ALL        pha
JOIN    POZ_SUPPLIERS_V       ps   ON ps.VENDOR_ID         = pha.VENDOR_ID
JOIN    PO_LINE_LOCATIONS_ALL plla ON plla.PO_HEADER_ID    = pha.PO_HEADER_ID
JOIN    PO_DISTRIBUTIONS_ALL  pda  ON pda.LINE_LOCATION_ID = plla.LINE_LOCATION_ID
LEFT JOIN inv                      ON inv.PO_DISTRIBUTION_ID = pda.PO_DISTRIBUTION_ID
LEFT JOIN AP_PAYMENT_SCHEDULES_ALL apsa ON apsa.INVOICE_ID  = inv.invoice_id
WHERE   pha.SEGMENT1 = :P_PO_NUMBER;
```

**Traps to mention:**
- `QUANTITY_RECEIVED` on the schedule = **system** receipt. If it's 0 but the supplier delivered, the warehouse **didn't record the receipt**. That's the most common real cause.
- Exclude cancelled invoices and reversed distributions.
- One invoice can have many installments, so `apsa` may give multiple rows. That's fine for a diagnostic.

**Likely follow-ups:**
- *"Where would you see **why** it's on hold?"* → `AP_HOLDS_ALL` (hold reason, release status). It's outside our sheet, but naming it shows depth.
- *"How do you know the invoice is accounted?"* → `AP_INVOICE_DISTRIBUTIONS_ALL.POSTED_FLAG = 'Y'`.
</details>

---

### P2P-2 🟡 "Month-end accrual: received but not invoiced"

> **Interviewer:** *"It's month-end. The controller says: 'We received goods worth crores that suppliers haven't billed yet. I need to book an accrual. Give me the list **by supplier and charge account**, with the value.'"*

**Testing:** Do you pick the **distribution** level (because they said *charge account*) and use the quantity columns instead of rebuilding them from receipts?

<details><summary><b>Show solution</b></summary>

**Key insight:** "by charge account" means the grain must be `PO_DISTRIBUTIONS_ALL` (that's where `CODE_COMBINATION_ID` lives). The distribution already tracks `QUANTITY_DELIVERED` and `QUANTITY_BILLED`.

| Noun | Table |
|------|-------|
| supplier | `POZ_SUPPLIERS_V` |
| received | `PO_DISTRIBUTIONS_ALL.QUANTITY_DELIVERED` |
| not invoiced | `PO_DISTRIBUTIONS_ALL.QUANTITY_BILLED` |
| value | `PO_LINES_ALL.UNIT_PRICE` |
| charge account | `PO_DISTRIBUTIONS_ALL.CODE_COMBINATION_ID` |

**Grain:** one row per supplier + charge account.

```sql
SELECT  ps.VENDOR_NAME,
        pda.CODE_COMBINATION_ID                                         AS charge_account_id,
        pha.CURRENCY_CODE,
        SUM( (NVL(pda.QUANTITY_DELIVERED,0) - NVL(pda.QUANTITY_BILLED,0)) * pla.UNIT_PRICE ) AS accrual_value
FROM    PO_HEADERS_ALL       pha
JOIN    POZ_SUPPLIERS_V      ps  ON ps.VENDOR_ID     = pha.VENDOR_ID
JOIN    PO_LINES_ALL         pla ON pla.PO_HEADER_ID = pha.PO_HEADER_ID
JOIN    PO_DISTRIBUTIONS_ALL pda ON pda.PO_LINE_ID   = pla.PO_LINE_ID
WHERE   pha.PRC_BU_ID = :P_BU_ID
AND     NVL(pda.QUANTITY_DELIVERED,0) > NVL(pda.QUANTITY_BILLED,0)
GROUP BY ps.VENDOR_NAME, pda.CODE_COMBINATION_ID, pha.CURRENCY_CODE
ORDER BY accrual_value DESC;
```

**Traps:**
- **Currency:** don't sum INR and USD together. Group by currency (or convert using the PO's rate).
- **Point in time:** these quantity columns are **current** values. For "as of 31-Oct", you'd rebuild from `RCV_TRANSACTIONS` (DELIVER up to the date) and AP distributions (accounting date up to the date).
- To show the account **segments** (cost center etc.) you need `GL_CODE_COMBINATIONS`, which is outside the sheet but worth naming.

**Follow-ups:**
- *"Why not use `PO_LINE_LOCATIONS_ALL`?"* → It has no charge account. One schedule can split into 2 cost centers.
- *"Does Fusion already do this?"* → Yes, receipt accrual accounting through Cost Management / SLA. Your report is a **reconciliation** view.
</details>

---

### P2P-3 🔴 "Audit: we paid for more than we received"

> **Interviewer:** *"Internal audit found that for some PO lines **we paid the supplier for more quantity than we actually received**, even though those POs required receipts. Write me the exception report."*

**Testing:** 3-way match understanding, `RECEIPT_REQUIRED_FLAG`, comparing **billed vs received**, and including only **paid** amounts.

<details><summary><b>Show solution</b></summary>

| Noun | Table |
|------|-------|
| POs that required receipt | `PO_LINE_LOCATIONS_ALL.RECEIPT_REQUIRED_FLAG = 'Y'` |
| received qty | `PO_LINE_LOCATIONS_ALL.QUANTITY_RECEIVED` |
| invoiced qty | `AP_INVOICE_LINES_ALL.QUANTITY_INVOICED` (by PO_LINE_LOCATION_ID) |
| paid | `AP_INVOICES_ALL.PAYMENT_STATUS_FLAG IN ('Y','P')` |

**Grain:** one row per PO schedule where invoiced qty > received qty.

```sql
WITH billed AS (
  SELECT  ail.PO_LINE_LOCATION_ID,
          SUM(ail.QUANTITY_INVOICED)   AS qty_invoiced,
          SUM(ail.AMOUNT)              AS amt_invoiced,
          LISTAGG(DISTINCT aia.INVOICE_NUM, ', ') WITHIN GROUP (ORDER BY aia.INVOICE_NUM) AS invoices
  FROM    AP_INVOICE_LINES_ALL ail
  JOIN    AP_INVOICES_ALL      aia ON aia.INVOICE_ID = ail.INVOICE_ID
  WHERE   aia.CANCELLED_DATE IS NULL
  AND     aia.PAYMENT_STATUS_FLAG IN ('Y','P')          -- paid / partially paid
  AND     ail.LINE_TYPE_LOOKUP_CODE = 'ITEM'
  AND     NVL(ail.DISCARDED_FLAG,'N') = 'N'
  GROUP BY ail.PO_LINE_LOCATION_ID
)
SELECT  ps.VENDOR_NAME, pha.SEGMENT1 AS po_number, pla.LINE_NUM, plla.SHIPMENT_NUM,
        plla.QUANTITY_RECEIVED,
        b.qty_invoiced,
        b.qty_invoiced - NVL(plla.QUANTITY_RECEIVED,0)                  AS excess_qty,
        (b.qty_invoiced - NVL(plla.QUANTITY_RECEIVED,0)) * pla.UNIT_PRICE AS excess_value,
        b.invoices
FROM    PO_LINE_LOCATIONS_ALL plla
JOIN    billed                b   ON b.PO_LINE_LOCATION_ID = plla.LINE_LOCATION_ID
JOIN    PO_LINES_ALL          pla ON pla.PO_LINE_ID        = plla.PO_LINE_ID
JOIN    PO_HEADERS_ALL        pha ON pha.PO_HEADER_ID      = plla.PO_HEADER_ID
JOIN    POZ_SUPPLIERS_V       ps  ON ps.VENDOR_ID          = pha.VENDOR_ID
WHERE   plla.RECEIPT_REQUIRED_FLAG = 'Y'
AND     b.qty_invoiced > NVL(plla.QUANTITY_RECEIVED,0);
```

**Traps:**
- **Returns:** if goods were returned (RETURN TO VENDOR), `QUANTITY_RECEIVED` goes down. That's correct, because you shouldn't pay for returned goods.
- Use `QUANTITY_ACCEPTED` instead when the PO is **4-way** (`INSPECTION_REQUIRED_FLAG = 'Y'`).
- Credit memos raised later may have fixed the overpayment. A good report nets those off (credit memos are negative invoice lines matched to the same PO).

**Follow-ups:**
- *"How could this even happen if the system enforces 3-way match?"* → Tolerances (qty received tolerance), holds manually released, or a receipt that was later corrected/returned after payment. Mentioning **hold release** and **tolerances** is senior-level.
</details>

---

### P2P-4 🔴 "Find duplicate supplier invoices"

> **Interviewer:** *"We suspect duplicate payments. Same supplier, same amount, but the invoice numbers are like `INV-1001`, `INV1001`, `inv 1001`. The system didn't block them. Find them."*

**Testing:** Self-join thinking, data normalization, date windows, and performance awareness.

<details><summary><b>Show solution</b></summary>

| Noun | Table |
|------|-------|
| supplier, invoice number, amount | `AP_INVOICES_ALL` (self-join) |
| payments | `AP_INVOICE_PAYMENTS_ALL` → `AP_CHECKS_ALL` |

**Grain:** one row per **suspected duplicate pair**.

```sql
WITH inv AS (
  SELECT  INVOICE_ID, INVOICE_NUM, VENDOR_ID, INVOICE_AMOUNT, INVOICE_DATE, AMOUNT_PAID,
          REGEXP_REPLACE(UPPER(INVOICE_NUM), '[^A-Z0-9]', '')  AS norm_num   -- remove spaces, dashes, slashes
  FROM    AP_INVOICES_ALL
  WHERE   CANCELLED_DATE IS NULL
  AND     INVOICE_TYPE_LOOKUP_CODE = 'STANDARD'
  AND     INVOICE_DATE >= ADD_MONTHS(TRUNC(SYSDATE), -12)       -- limit the scan
)
SELECT  ps.VENDOR_NAME,
        a.INVOICE_NUM  AS invoice_1, a.INVOICE_DATE AS date_1, a.AMOUNT_PAID AS paid_1,
        b.INVOICE_NUM  AS invoice_2, b.INVOICE_DATE AS date_2, b.AMOUNT_PAID AS paid_2,
        a.INVOICE_AMOUNT
FROM    inv a
JOIN    inv b
        ON  b.VENDOR_ID      = a.VENDOR_ID
        AND b.INVOICE_AMOUNT = a.INVOICE_AMOUNT
        AND b.norm_num       = a.norm_num
        AND b.INVOICE_ID     > a.INVOICE_ID          -- each pair once, no self-match
JOIN    POZ_SUPPLIERS_V ps ON ps.VENDOR_ID = a.VENDOR_ID
ORDER BY a.INVOICE_AMOUNT DESC;
```

**Why `b.INVOICE_ID > a.INVOICE_ID`:** with `<>` you'd get every pair twice (A-B and B-A). `>` gives each pair once.

**Traps / senior talking points:**
- **Performance:** `REGEXP_REPLACE` on a big table can't use an index. So limit by date, and do the normalization **once** in a CTE, not inside the join condition twice.
- **Variations:** also check same supplier + same amount + invoice dates within 7 days, even when the numbers differ completely (a fuzzier rule).
- **Different supplier records for the same company** (duplicate supplier master!) won't be caught by `VENDOR_ID`. Mention matching on supplier tax registration/bank account as a further check.

**Follow-ups:**
- *"How would you prevent this, not just detect it?"* → Supplier/invoice-number uniqueness validation in Payables setup, plus a scheduled BIP report delivered to the AP manager (bursting/scheduling).
</details>

---

### P2P-5 🟡 "Stock doesn't match receipts"

> **Interviewer:** *"The warehouse says: 'Receiving shows we delivered 1,000 units of item LAP-100 to stores this month, but on-hand only went up by 800.' How do you investigate with data?"*

**Testing:** The RCV ↔ INV bridge, on-hand aggregation, and knowing that stock also goes **out**.

<details><summary><b>Show solution</b></summary>

| Noun | Table |
|------|-------|
| delivered to stores | `RCV_TRANSACTIONS` (TRANSACTION_TYPE = 'DELIVER', DESTINATION_TYPE_CODE = 'INVENTORY') |
| inventory effect of that delivery | `INV_MATERIAL_TXNS` via `rt.INV_TRANSACTION_ID` |
| all other movements | `INV_MATERIAL_TXNS` (issues, transfers, sales) |
| on-hand | `INV_ONHAND_QUANTITIES_DETAIL` |
| item | `EGP_SYSTEM_ITEMS_VL` |

**Investigation in 3 queries (say it like this):**

**Q1. Did every DELIVER create an inventory transaction?**
```sql
SELECT  rt.TRANSACTION_ID, rt.TRANSACTION_DATE, rt.QUANTITY, rt.INV_TRANSACTION_ID
FROM    RCV_TRANSACTIONS    rt
JOIN    EGP_SYSTEM_ITEMS_VL esi ON esi.INVENTORY_ITEM_ID = (SELECT rsl.ITEM_ID FROM RCV_SHIPMENT_LINES rsl
                                                           WHERE rsl.SHIPMENT_LINE_ID = rt.SHIPMENT_LINE_ID)
                               AND esi.ORGANIZATION_ID   = rt.ORGANIZATION_ID
LEFT JOIN INV_MATERIAL_TXNS imt ON imt.TRANSACTION_ID   = rt.INV_TRANSACTION_ID
WHERE   esi.ITEM_NUMBER = :P_ITEM
AND     rt.TRANSACTION_TYPE = 'DELIVER'
AND     rt.DESTINATION_TYPE_CODE = 'INVENTORY'
AND     rt.TRANSACTION_DATE >= TRUNC(SYSDATE,'MM')
AND     imt.TRANSACTION_ID IS NULL;          -- delivers with NO stock movement
```

**Q2. What else moved the stock this month (the other 200)?**
```sql
SELECT  imt.TRANSACTION_SOURCE_TYPE_ID,
        SUM(CASE WHEN imt.TRANSACTION_QUANTITY > 0 THEN imt.TRANSACTION_QUANTITY END) AS qty_in,
        SUM(CASE WHEN imt.TRANSACTION_QUANTITY < 0 THEN imt.TRANSACTION_QUANTITY END) AS qty_out
FROM    INV_MATERIAL_TXNS   imt
JOIN    EGP_SYSTEM_ITEMS_VL esi ON esi.INVENTORY_ITEM_ID = imt.INVENTORY_ITEM_ID
                               AND esi.ORGANIZATION_ID   = imt.ORGANIZATION_ID
WHERE   esi.ITEM_NUMBER = :P_ITEM
AND     imt.TRANSACTION_DATE >= TRUNC(SYSDATE,'MM')
GROUP BY imt.TRANSACTION_SOURCE_TYPE_ID;
```

**Q3. Current on-hand** (`SUM(PRIMARY_TRANSACTION_QUANTITY)` grouped by org/subinventory).

**The answer the interviewer wants:**
*"On-hand change = all ins − all outs, not just receipts. The 200 gap is usually sales shipments / issues out of stock, returns to vendor, or DELIVERs that went to a different org/subinventory. Q2 proves which."*

**Traps:** item join on **item + org**; on-hand must be **summed**; returns (`RETURN TO RECEIVING`) reduce stock too.

**Follow-up:** *"How do you get on-hand **as of** a past date?"* → Current on-hand minus the sum of `INV_MATERIAL_TXNS` quantities after that date (roll-back method).
</details>

---

### P2P-6 🟡 "Budget view: committed vs invoiced vs paid"

> **Interviewer:** *"A department head asks: 'For my cost center, for every PO, show me how much I **committed**, how much has been **invoiced**, and how much is actually **paid**.' Be careful, I've seen many developers get these numbers wrong."*

**Testing:** **Fan-out.** "I've seen developers get this wrong" is a hint.

<details><summary><b>Show solution</b></summary>

| Number | Source |
|--------|--------|
| committed | `PO_DISTRIBUTIONS_ALL.QUANTITY_ORDERED × PO_LINES_ALL.UNIT_PRICE` |
| invoiced | `AP_INVOICE_DISTRIBUTIONS_ALL.AMOUNT` (by PO_DISTRIBUTION_ID) |
| paid | `AP_INVOICE_PAYMENTS_ALL.AMOUNT` |
| cost center | `PO_DISTRIBUTIONS_ALL.CODE_COMBINATION_ID` (segment via `GL_CODE_COMBINATIONS`) |

**Why naive joins fail:** PO dist → many invoice dists → each invoice → many payments. Joining all and summing multiplies amounts.

**Correct pattern:** aggregate each measure **to the same grain (PO_HEADER_ID)** in its own CTE, then join.
```sql
WITH committed AS (
  SELECT pda.PO_HEADER_ID, SUM(pda.QUANTITY_ORDERED * pla.UNIT_PRICE) AS committed_amt
  FROM   PO_DISTRIBUTIONS_ALL pda
  JOIN   PO_LINES_ALL pla ON pla.PO_LINE_ID = pda.PO_LINE_ID
  WHERE  pda.CODE_COMBINATION_ID IN (SELECT CODE_COMBINATION_ID FROM GL_CODE_COMBINATIONS
                                     WHERE SEGMENT2 = :P_COST_CENTER)   -- segment # depends on COA
  GROUP BY pda.PO_HEADER_ID
),
invoiced AS (
  SELECT pda.PO_HEADER_ID, SUM(aid.AMOUNT) AS invoiced_amt
  FROM   AP_INVOICE_DISTRIBUTIONS_ALL aid
  JOIN   PO_DISTRIBUTIONS_ALL pda ON pda.PO_DISTRIBUTION_ID = aid.PO_DISTRIBUTION_ID
  JOIN   AP_INVOICES_ALL aia      ON aia.INVOICE_ID = aid.INVOICE_ID
  WHERE  aia.CANCELLED_DATE IS NULL
  AND    NVL(aid.REVERSAL_FLAG,'N') = 'N'
  GROUP BY pda.PO_HEADER_ID
),
po_invoices AS (   -- distinct invoices per PO (avoid counting an invoice twice)
  SELECT DISTINCT pda.PO_HEADER_ID, aid.INVOICE_ID
  FROM   AP_INVOICE_DISTRIBUTIONS_ALL aid
  JOIN   PO_DISTRIBUTIONS_ALL pda ON pda.PO_DISTRIBUTION_ID = aid.PO_DISTRIBUTION_ID
),
paid AS (
  SELECT pi.PO_HEADER_ID, SUM(aipa.AMOUNT) AS paid_amt
  FROM   po_invoices pi
  JOIN   AP_INVOICE_PAYMENTS_ALL aipa ON aipa.INVOICE_ID = pi.INVOICE_ID
  JOIN   AP_CHECKS_ALL aca            ON aca.CHECK_ID    = aipa.CHECK_ID
  WHERE  aca.STATUS_LOOKUP_CODE <> 'VOIDED'
  GROUP BY pi.PO_HEADER_ID
)
SELECT  pha.SEGMENT1 AS po_number,
        c.committed_amt,
        NVL(i.invoiced_amt,0) AS invoiced_amt,
        NVL(p.paid_amt,0)     AS paid_amt
FROM    committed c
JOIN    PO_HEADERS_ALL pha ON pha.PO_HEADER_ID = c.PO_HEADER_ID
LEFT JOIN invoiced i ON i.PO_HEADER_ID = c.PO_HEADER_ID
LEFT JOIN paid     p ON p.PO_HEADER_ID = c.PO_HEADER_ID;
```

**Senior caveat to say out loud:** *"Paid is at invoice level. If one invoice covers 2 POs, payments can't be split exactly per PO without prorating by invoice distribution amount. I'd either prorate or show paid at invoice level."* This caveat is often what separates good answers from great ones.
</details>

---

### P2P-7 🟡 "Supplier scorecard: on-time delivery and quality"

> **Interviewer:** *"Procurement wants a quarterly supplier scorecard: **% of schedules delivered on or before the need-by date**, and **% of received quantity rejected in inspection**."*

**Testing:** Using `RCV_TRANSACTIONS` **types** and **dates**, aggregating at the right level.

<details><summary><b>Show solution</b></summary>

| Measure | Source |
|---------|--------|
| need-by date | `PO_LINE_LOCATIONS_ALL.NEED_BY_DATE` |
| actual receipt date | first `RCV_TRANSACTIONS.TRANSACTION_DATE` where type = 'RECEIVE' |
| rejected qty | `RCV_TRANSACTIONS` type = 'REJECT' |
| supplier | `PO_HEADERS_ALL.VENDOR_ID` → `POZ_SUPPLIERS_V` |

```sql
WITH sched AS (    -- one row per schedule
  SELECT  pha.VENDOR_ID,
          plla.LINE_LOCATION_ID,
          plla.NEED_BY_DATE,
          MIN(CASE WHEN rt.TRANSACTION_TYPE = 'RECEIVE' THEN rt.TRANSACTION_DATE END) AS first_receipt_date,
          SUM(CASE WHEN rt.TRANSACTION_TYPE = 'RECEIVE' THEN rt.QUANTITY ELSE 0 END)  AS qty_received,
          SUM(CASE WHEN rt.TRANSACTION_TYPE = 'REJECT'  THEN rt.QUANTITY ELSE 0 END)  AS qty_rejected
  FROM    PO_HEADERS_ALL        pha
  JOIN    PO_LINE_LOCATIONS_ALL plla ON plla.PO_HEADER_ID       = pha.PO_HEADER_ID
  JOIN    RCV_TRANSACTIONS      rt   ON rt.PO_LINE_LOCATION_ID  = plla.LINE_LOCATION_ID
  WHERE   rt.TRANSACTION_DATE BETWEEN :P_FROM AND :P_TO
  GROUP BY pha.VENDOR_ID, plla.LINE_LOCATION_ID, plla.NEED_BY_DATE
)
SELECT  ps.VENDOR_NAME,
        COUNT(*)                                                           AS schedules,
        ROUND(100 * SUM(CASE WHEN TRUNC(s.first_receipt_date) <= TRUNC(s.NEED_BY_DATE) THEN 1 ELSE 0 END)
                  / COUNT(*), 1)                                           AS on_time_pct,
        ROUND(100 * SUM(s.qty_rejected) / NULLIF(SUM(s.qty_received),0), 2) AS reject_pct
FROM    sched s
JOIN    POZ_SUPPLIERS_V ps ON ps.VENDOR_ID = s.VENDOR_ID
WHERE   s.first_receipt_date IS NOT NULL
GROUP BY ps.VENDOR_NAME
ORDER BY on_time_pct;
```

**Talking points:** `NULLIF` avoids divide-by-zero. Use `PROMISED_DATE` if the supplier committed to a different date (fairer to suppliers). Partial deliveries: "on time" could mean **fully** received by need-by. Ask the business which definition they want (seniors like it when you clarify the requirement).
</details>

---

### P2P-8 🟢 "Treasury: cash needed next 30 days"

> **Interviewer:** *"Treasury wants to know how much cash they need for supplier payments in the next 30 days, by week and currency. Only include invoices that are actually **payable**."*

**Testing:** `AP_PAYMENT_SCHEDULES_ALL` as the right source, and defining "payable".

<details><summary><b>Show solution</b></summary>

"Actually payable" = not cancelled, validated, not on hold, amount remaining > 0.

```sql
SELECT  TRUNC(apsa.DUE_DATE,'IW')        AS week_starting,
        aia.INVOICE_CURRENCY_CODE,
        SUM(apsa.AMOUNT_REMAINING)       AS cash_needed,
        COUNT(DISTINCT aia.INVOICE_ID)   AS invoices
FROM    AP_PAYMENT_SCHEDULES_ALL apsa
JOIN    AP_INVOICES_ALL          aia ON aia.INVOICE_ID = apsa.INVOICE_ID
WHERE   aia.CANCELLED_DATE IS NULL
AND     aia.ORG_ID = :P_BU_ID
AND     apsa.AMOUNT_REMAINING > 0
AND     NVL(apsa.HOLD_FLAG,'N') = 'N'
AND     apsa.DUE_DATE BETWEEN TRUNC(SYSDATE) AND TRUNC(SYSDATE) + 30
AND     NOT EXISTS (SELECT 1 FROM AP_INVOICE_DISTRIBUTIONS_ALL aid      -- every dist validated
                    WHERE  aid.INVOICE_ID = aia.INVOICE_ID
                    AND    NVL(aid.MATCH_STATUS_FLAG,'N') <> 'A')
GROUP BY TRUNC(apsa.DUE_DATE,'IW'), aia.INVOICE_CURRENCY_CODE
ORDER BY week_starting;
```

**Talking points:** `TRUNC(date,'IW')` = ISO week start (Monday). `NOT EXISTS` for "all distributions validated" is cleaner than joining and grouping. Overdue amounts (due date < today) are also cash needed. **Ask** whether to include them (yes, usually).
</details>

---

### P2P-9 🔴 "A payment was voided. What's the impact?"

> **Interviewer:** *"Yesterday a ₹50 lakh EFT to a supplier bounced and AP voided it. Show me which invoices are open again, and how I'd confirm the reversal hit the GL."*

**Testing:** How voids look in data (reversal rows), the AP payment bridge, and AP → GL thinking.

<details><summary><b>Show solution</b></summary>

**How a void looks in the tables:**
- `AP_CHECKS_ALL`: `STATUS_LOOKUP_CODE = 'VOIDED'`, `VOID_DATE` filled
- `AP_INVOICE_PAYMENTS_ALL`: the **original** row stays, and a **new reversal row** with negative amount is created (`REVERSAL_FLAG = 'Y'`)
- `AP_PAYMENT_SCHEDULES_ALL.AMOUNT_REMAINING` goes back up; `AP_INVOICES_ALL.PAYMENT_STATUS_FLAG` goes back to 'N'/'P'

```sql
SELECT  aca.CHECK_NUMBER, aca.VOID_DATE,
        aia.INVOICE_NUM,
        aipa.PAYMENT_NUM,
        aipa.AMOUNT,
        aipa.REVERSAL_FLAG,
        apsa.AMOUNT_REMAINING        AS now_remaining,
        aia.PAYMENT_STATUS_FLAG      AS now_status
FROM    AP_CHECKS_ALL            aca
JOIN    AP_INVOICE_PAYMENTS_ALL  aipa ON aipa.CHECK_ID    = aca.CHECK_ID
JOIN    AP_INVOICES_ALL          aia  ON aia.INVOICE_ID   = aipa.INVOICE_ID
JOIN    AP_PAYMENT_SCHEDULES_ALL apsa ON apsa.INVOICE_ID  = aipa.INVOICE_ID
                                     AND apsa.PAYMENT_NUM = aipa.PAYMENT_NUM
WHERE   aca.STATUS_LOOKUP_CODE = 'VOIDED'
AND     aca.VOID_DATE >= TRUNC(SYSDATE) - 1
ORDER BY aca.CHECK_NUMBER, aia.INVOICE_NUM, aipa.REVERSAL_FLAG;
```

**GL confirmation (talking points):**
- At GL level: `GL_JE_HEADERS` with `JE_SOURCE = 'Payables'`, `JE_CATEGORY = 'Payments'` (or reconciled payments) in the void-date period. There should be an entry reversing *Dr Liability / Cr Cash*.
- **Exact drill-down** from a payment to its GL lines goes through Subledger Accounting (`XLA_AE_HEADERS`/`XLA_AE_LINES`) and GL import references. **Name these.** A 20-year veteran will respect that you know AP doesn't link directly to GL.

**Trap:** if someone summed `AP_INVOICE_PAYMENTS_ALL.AMOUNT` without considering reversal rows, the net is still correct (+50L −50L = 0). But if they **filtered `REVERSAL_FLAG = 'N'`** they'd see the payment as still valid. Explain which you want: **net** (no filter, sum) or **current valid payments** (exclude both the original and the reversal, i.e. exclude voided checks).
</details>

---

### P2P-10 🟢 "Inactive suppliers with open business"

> **Interviewer:** *"Before we deactivate 200 old suppliers, master data wants to know which of them still have **open POs** or **unpaid invoices**. Also flag any payment made to a site that is **not a pay site**."*

**Testing:** Supplier master structure + EXISTS checks + a control check.

<details><summary><b>Show solution</b></summary>

```sql
SELECT  ps.SEGMENT1 AS supplier_no, ps.VENDOR_NAME,
        CASE WHEN EXISTS (SELECT 1 FROM PO_HEADERS_ALL pha
                          WHERE pha.VENDOR_ID = ps.VENDOR_ID
                          AND   pha.DOCUMENT_STATUS = 'OPEN')                 THEN 'Y' ELSE 'N' END AS has_open_po,
        CASE WHEN EXISTS (SELECT 1 FROM AP_INVOICES_ALL aia
                          WHERE aia.VENDOR_ID = ps.VENDOR_ID
                          AND   aia.CANCELLED_DATE IS NULL
                          AND   aia.PAYMENT_STATUS_FLAG IN ('N','P'))         THEN 'Y' ELSE 'N' END AS has_unpaid_inv
FROM    POZ_SUPPLIERS_V ps
WHERE   ps.SEGMENT1 IN (SELECT supplier_no FROM <uploaded_list>)   -- or a parameter list
;

-- Control check: payments to non-pay sites
SELECT  aca.CHECK_NUMBER, aca.CHECK_DATE, aca.AMOUNT, pss.VENDOR_SITE_CODE
FROM    AP_CHECKS_ALL        aca
JOIN    POZ_SUPPLIER_SITES_V pss ON pss.VENDOR_SITE_ID = aca.VENDOR_SITE_ID
WHERE   NVL(pss.PAY_SITE_FLAG,'N') = 'N'
AND     aca.STATUS_LOOKUP_CODE <> 'VOIDED';
```

**Why EXISTS here:** we only need yes/no, and EXISTS **stops at the first match**, which is faster than joining and counting, and it can't fan out.
</details>

---

## Part 3 — 10 O2C scenarios

---

### O2C-1 🟡 "Customer says: billed for 100, received 60"

> **Interviewer:** *"A customer escalates: 'We ordered 100 units on order 55012, received only 60, but your invoice charges us for 100.' Show me the data that proves who's right."*

**Testing:** Comparing **ordered vs shipped vs invoiced** across DOO, WSH and RA using the bridges.

<details><summary><b>Show solution</b></summary>

| Noun | Table |
|------|-------|
| ordered 100 | `DOO_LINES_ALL.ORDERED_QTY` |
| received 60 (= we shipped) | `WSH_DELIVERY_DETAILS` (RELEASED_STATUS='C', SHIPPED_QUANTITY) / `DOO_FULFILL_LINES_ALL.SHIPPED_QTY` |
| invoice charges 100 | `RA_CUSTOMER_TRX_LINES_ALL.QUANTITY_INVOICED` |

**Grain:** one row per order line, with three quantities side by side. Aggregate each **separately**.

```sql
WITH shipped AS (
  SELECT  wdd.SOURCE_HEADER_ID, wdd.SALES_ORDER_LINE_NUMBER,
          SUM(wdd.SHIPPED_QUANTITY) AS shipped_qty
  FROM    WSH_DELIVERY_DETAILS wdd
  WHERE   wdd.RELEASED_STATUS = 'C'
  GROUP BY wdd.SOURCE_HEADER_ID, wdd.SALES_ORDER_LINE_NUMBER
),
invoiced AS (
  SELECT  rctl.INTERFACE_LINE_ATTRIBUTE1 AS hdr, rctl.INTERFACE_LINE_ATTRIBUTE2 AS line_no,
          SUM(rctl.QUANTITY_INVOICED)   AS invoiced_qty,
          LISTAGG(DISTINCT rct.TRX_NUMBER, ', ') WITHIN GROUP (ORDER BY rct.TRX_NUMBER) AS invoices
  FROM    RA_CUSTOMER_TRX_LINES_ALL rctl
  JOIN    RA_CUSTOMER_TRX_ALL rct ON rct.CUSTOMER_TRX_ID = rctl.CUSTOMER_TRX_ID
  WHERE   rctl.LINE_TYPE = 'LINE'
  GROUP BY rctl.INTERFACE_LINE_ATTRIBUTE1, rctl.INTERFACE_LINE_ATTRIBUTE2
)
SELECT  dha.ORDER_NUMBER, dla.DISPLAY_LINE_NUMBER,
        dla.ORDERED_QTY,
        NVL(s.shipped_qty,0)  AS shipped_qty,
        NVL(i.invoiced_qty,0) AS invoiced_qty,
        i.invoices,
        CASE WHEN NVL(i.invoiced_qty,0) > NVL(s.shipped_qty,0) THEN 'BILLED MORE THAN SHIPPED' END AS flag
FROM    DOO_HEADERS_ALL dha
JOIN    DOO_LINES_ALL   dla ON dla.HEADER_ID = dha.HEADER_ID
LEFT JOIN shipped  s ON s.SOURCE_HEADER_ID = dha.HEADER_ID AND s.SALES_ORDER_LINE_NUMBER = dla.LINE_NUMBER
LEFT JOIN invoiced i ON i.hdr = TO_CHAR(dla.HEADER_ID)     AND i.line_no = TO_CHAR(dla.LINE_NUMBER)
WHERE   dha.ORDER_NUMBER   = :P_ORDER
AND     dha.SUBMITTED_FLAG = 'Y'
AND     dha.STATUS_CODE   <> 'DOO_REFERENCE';
```

**Talking points:**
- Normally Fusion invoices **only what's shipped** (billing happens after ship confirm). If invoiced > shipped, suspect a **manual invoice**, or an item/line set up as "bill on order" (non-shippable/service).
- Also check **credit memos** already raised (CLASS = 'CM' in `AR_PAYMENT_SCHEDULES_ALL`, or `RA_CUSTOMER_TRX_ALL.PREVIOUS_CUSTOMER_TRX_ID` = this invoice).
- **Customer might be right in a different way:** shipped 100 but 40 lost in transit, so check `WSH_NEW_DELIVERIES` status / waybill for proof of delivery.
</details>

---

### O2C-2 🟢 "Sales backlog with value"

> **Interviewer:** *"The Sales VP wants the open order backlog: everything booked but not shipped, with **value**, and highlight anything past its scheduled ship date."*

**Testing:** Fulfill-line level thinking, versions, cancelled lines, value calculation.

<details><summary><b>Show solution</b></summary>

```sql
SELECT  hp.PARTY_NAME                               AS customer,
        dha.ORDER_NUMBER,
        dla.DISPLAY_LINE_NUMBER,
        dfla.FULFILL_LINE_NUMBER,
        dfla.FULFILL_ORG_ID,
        dfla.ORDERED_QTY - NVL(dfla.SHIPPED_QTY,0)  AS open_qty,
        (dfla.ORDERED_QTY - NVL(dfla.SHIPPED_QTY,0)) * dla.UNIT_SELLING_PRICE AS open_value,
        dha.TRANSACTIONAL_CURRENCY_CODE             AS currency,
        dfla.SCHEDULE_SHIP_DATE,
        CASE WHEN dfla.SCHEDULE_SHIP_DATE < TRUNC(SYSDATE) THEN 'PAST DUE' END AS flag
FROM    DOO_HEADERS_ALL       dha
JOIN    HZ_PARTIES            hp   ON hp.PARTY_ID  = dha.SOLD_TO_PARTY_ID
JOIN    DOO_LINES_ALL         dla  ON dla.HEADER_ID = dha.HEADER_ID
JOIN    DOO_FULFILL_LINES_ALL dfla ON dfla.LINE_ID  = dla.LINE_ID
WHERE   dha.SUBMITTED_FLAG = 'Y'
AND     dha.STATUS_CODE   <> 'DOO_REFERENCE'
AND     dfla.OPEN_FLAG     = 'Y'
AND     NVL(dfla.CANCELED_FLAG,'N') = 'N'
AND     dla.CATEGORY_CODE  = 'ORDER'                 -- exclude return lines
AND     dfla.ORDERED_QTY > NVL(dfla.SHIPPED_QTY,0)
AND     dha.ORG_ID = :P_BU_ID;
```

**Talking points:** price from the **line**, quantity from the **fulfill line** (the split). Exclude RMA/return lines. Currency per order. Never sum across currencies.
</details>

---

### O2C-3 🔴 "Shipped but not invoiced at month-end"

> **Interviewer:** *"Finance says revenue looks low this month. I suspect goods were shipped but never invoiced. Find them, and tell me **why** they might not have been invoiced."*

**Testing:** NOT EXISTS across the AutoInvoice bridge **plus** knowledge of where invoicing can fail.

<details><summary><b>Show solution</b></summary>

```sql
SELECT  dha.ORDER_NUMBER, dla.DISPLAY_LINE_NUMBER, dfla.FULFILL_LINE_NUMBER,
        dfla.SHIPPED_QTY, dfla.ACTUAL_SHIP_DATE, dfla.STATUS_CODE,
        TRUNC(SYSDATE) - TRUNC(dfla.ACTUAL_SHIP_DATE) AS days_since_ship
FROM    DOO_HEADERS_ALL       dha
JOIN    DOO_LINES_ALL         dla  ON dla.HEADER_ID = dha.HEADER_ID
JOIN    DOO_FULFILL_LINES_ALL dfla ON dfla.LINE_ID  = dla.LINE_ID
WHERE   dha.SUBMITTED_FLAG = 'Y'
AND     dha.STATUS_CODE   <> 'DOO_REFERENCE'
AND     NVL(dfla.SHIPPED_QTY,0) > 0
AND     dfla.ACTUAL_SHIP_DATE < TRUNC(SYSDATE,'MM') + INTERVAL '1' MONTH   -- shipped up to month end
AND     NOT EXISTS (SELECT 1
                    FROM   RA_CUSTOMER_TRX_LINES_ALL rctl
                    WHERE  rctl.INTERFACE_LINE_ATTRIBUTE1 = TO_CHAR(dla.HEADER_ID)
                    AND    rctl.INTERFACE_LINE_ATTRIBUTE2 = TO_CHAR(dla.LINE_NUMBER)
                    AND    rctl.LINE_TYPE = 'LINE')
ORDER BY days_since_ship DESC;
```

**The "why" (this is what earns the points):**
1. **AutoInvoice hasn't run yet** (it's a scheduled process).
2. **AutoInvoice rejected the line.** Lines wait in the interface table (`RA_INTERFACE_LINES_ALL`) with errors (e.g. invalid bill-to, closed GL period, missing tax setup). Outside the sheet, but name it.
3. **Billing integration hasn't transferred it** from Order Management to Receivables (the fulfill line status will still show awaiting billing).
4. The line is **non-billable** by design (free sample / no-charge item).

**Performance note:** the `TO_CHAR(dla.HEADER_ID)` is applied to the DOO side, so an index on `INTERFACE_LINE_ATTRIBUTE1` (if one exists) can still be used. Wrapping the RA column in `TO_NUMBER()` would break index use **and** could fail on non-numeric attribute values from other sources.
</details>

---

### O2C-4 🟡 "Collections: aging and net exposure"

> **Interviewer:** *"The collections manager wants aging by customer, with 0–30, 31–60, 61–90 and 90+ buckets. Also show **net exposure**. Some customers have paid money that hasn't been matched yet."*

**Testing:** `AR_PAYMENT_SCHEDULES_ALL` classes, including **credit memos and unapplied receipts**.

<details><summary><b>Show solution</b></summary>

**Key insight:** `AR_PAYMENT_SCHEDULES_ALL` holds **both** invoices (positive remaining) **and** receipts/CMs (negative remaining when unapplied). So net exposure = sum of all open rows.

```sql
SELECT  hp.PARTY_NAME, hca.ACCOUNT_NUMBER,
        SUM(CASE WHEN aps.CLASS IN ('INV','DM') AND TRUNC(SYSDATE) - aps.DUE_DATE <= 0  THEN aps.AMOUNT_DUE_REMAINING END) AS current_amt,
        SUM(CASE WHEN aps.CLASS IN ('INV','DM') AND TRUNC(SYSDATE) - aps.DUE_DATE BETWEEN 1  AND 30 THEN aps.AMOUNT_DUE_REMAINING END) AS b_1_30,
        SUM(CASE WHEN aps.CLASS IN ('INV','DM') AND TRUNC(SYSDATE) - aps.DUE_DATE BETWEEN 31 AND 60 THEN aps.AMOUNT_DUE_REMAINING END) AS b_31_60,
        SUM(CASE WHEN aps.CLASS IN ('INV','DM') AND TRUNC(SYSDATE) - aps.DUE_DATE BETWEEN 61 AND 90 THEN aps.AMOUNT_DUE_REMAINING END) AS b_61_90,
        SUM(CASE WHEN aps.CLASS IN ('INV','DM') AND TRUNC(SYSDATE) - aps.DUE_DATE > 90  THEN aps.AMOUNT_DUE_REMAINING END) AS b_90_plus,
        SUM(CASE WHEN aps.CLASS IN ('CM','PMT')  THEN aps.AMOUNT_DUE_REMAINING END)   AS unapplied_credits,  -- negative
        SUM(aps.AMOUNT_DUE_REMAINING)                                                 AS net_exposure
FROM    AR_PAYMENT_SCHEDULES_ALL aps
JOIN    HZ_CUST_ACCOUNTS hca ON hca.CUST_ACCOUNT_ID = aps.CUSTOMER_ID
JOIN    HZ_PARTIES       hp  ON hp.PARTY_ID         = hca.PARTY_ID
WHERE   aps.STATUS = 'OP'
GROUP BY hp.PARTY_NAME, hca.ACCOUNT_NUMBER
HAVING  SUM(aps.AMOUNT_DUE_REMAINING) <> 0
ORDER BY net_exposure DESC;
```

**Talking points:**
- **Currency:** `AMOUNT_DUE_REMAINING` is in the transaction currency. Multi-currency customers need grouping by currency or conversion to functional currency (`ACCTD_AMOUNT_DUE_REMAINING` exists for that, so mention it).
- **As-of-date aging** (e.g. aging as of 31-Oct) can't use `STATUS`/remaining directly. You rebuild it from original amounts minus applications dated ≤ as-of-date. Seniors often ask this as a follow-up.
</details>

---

### O2C-5 🟡 "Customer: 'I paid ₹5 lakh, why am I still overdue?'"

> **Interviewer:** *"A customer says they paid ₹5,00,000 last week by NEFT, but our statement still shows them overdue. Walk me through how you'd investigate using data."*

**Testing:** Receipt statuses, applications, and the **many ways cash can be 'received but not applied'**.

<details><summary><b>Show solution</b></summary>

**The diagnosis ladder:**
```
1. Did the receipt get created at all?          → AR_CASH_RECEIPTS_ALL by AMOUNT/date/RECEIPT_NUMBER(UTR)
2. Under which customer?                        → PAY_FROM_CUSTOMER (maybe a different account / UNID = unidentified!)
3. Is it applied?                               → STATUS: APP / UNAPP / UNID
4. If applied, to which invoices?               → AR_RECEIVABLE_APPLICATIONS_ALL (STATUS='APP', DISPLAY='Y')
5. Or is it parked on-account?                  → applications with STATUS = 'ACC'
6. Was it reversed (bounced)?                   → REVERSAL_DATE / STATUS='REV'
```

```sql
SELECT  acr.RECEIPT_NUMBER, acr.RECEIPT_DATE, acr.AMOUNT, acr.STATUS AS receipt_status,
        acr.REVERSAL_DATE,
        hca_pay.ACCOUNT_NUMBER         AS receipt_customer,
        ara.STATUS                     AS application_status,     -- APP / UNAPP / ACC
        ara.AMOUNT_APPLIED,
        aps.TRX_NUMBER                 AS applied_to,
        aps.CUSTOMER_ID                AS applied_to_customer
FROM    AR_CASH_RECEIPTS_ALL acr
LEFT JOIN HZ_CUST_ACCOUNTS hca_pay ON hca_pay.CUST_ACCOUNT_ID = acr.PAY_FROM_CUSTOMER
LEFT JOIN AR_RECEIVABLE_APPLICATIONS_ALL ara ON ara.CASH_RECEIPT_ID = acr.CASH_RECEIPT_ID
                                            AND ara.DISPLAY = 'Y'
LEFT JOIN AR_PAYMENT_SCHEDULES_ALL aps ON aps.PAYMENT_SCHEDULE_ID = ara.APPLIED_PAYMENT_SCHEDULE_ID
WHERE   acr.AMOUNT = 500000
AND     acr.RECEIPT_DATE >= TRUNC(SYSDATE) - 14;
```

Note: **no** `STATUS='APP'` filter on applications here. For investigation, we **want** to see UNAPP and ACC rows.

**Typical answers:**
- Receipt is **UNID** (no customer identified) → cash team must identify it.
- Applied to the **wrong customer** account (`applied_to_customer` ≠ the complaining customer) → reapply.
- Sitting **on-account / unapplied** → apply to the overdue invoices.
- **Reversed**: the NEFT was returned.
</details>

---

### O2C-6 🟢 "Customer-facing invoice print (BIP template)"

> **Interviewer:** *"Build the data model for our customer invoice PDF: invoice header, **bill-to and ship-to** addresses, item lines with their tax."*

**Testing:** HZ address chain **twice** (different aliases), tax line linking.

<details><summary><b>Show solution</b></summary>

```sql
SELECT  rct.TRX_NUMBER, rct.TRX_DATE, rct.INVOICE_CURRENCY_CODE,
        hp.PARTY_NAME                         AS customer_name,
        -- bill-to
        bl.ADDRESS1 AS bill_addr1, bl.CITY AS bill_city, bl.POSTAL_CODE AS bill_pin,
        -- ship-to
        sl.ADDRESS1 AS ship_addr1, sl.CITY AS ship_city, sl.POSTAL_CODE AS ship_pin,
        rctl.LINE_NUMBER, rctl.DESCRIPTION, rctl.QUANTITY_INVOICED, rctl.UNIT_SELLING_PRICE,
        rctl.EXTENDED_AMOUNT,
        (SELECT NVL(SUM(t.EXTENDED_AMOUNT),0) FROM RA_CUSTOMER_TRX_LINES_ALL t
          WHERE t.LINK_TO_CUST_TRX_LINE_ID = rctl.CUSTOMER_TRX_LINE_ID
            AND t.LINE_TYPE = 'TAX')          AS line_tax
FROM    RA_CUSTOMER_TRX_ALL       rct
JOIN    RA_CUSTOMER_TRX_LINES_ALL rctl ON rctl.CUSTOMER_TRX_ID = rct.CUSTOMER_TRX_ID AND rctl.LINE_TYPE = 'LINE'
JOIN    HZ_CUST_ACCOUNTS          hca  ON hca.CUST_ACCOUNT_ID  = rct.BILL_TO_CUSTOMER_ID
JOIN    HZ_PARTIES                hp   ON hp.PARTY_ID          = hca.PARTY_ID
-- bill-to chain: U → A → P → L
JOIN    HZ_CUST_SITE_USES_ALL     bsu  ON bsu.SITE_USE_ID       = rct.BILL_TO_SITE_USE_ID
JOIN    HZ_CUST_ACCT_SITES_ALL    bas  ON bas.CUST_ACCT_SITE_ID = bsu.CUST_ACCT_SITE_ID
JOIN    HZ_PARTY_SITES            bps  ON bps.PARTY_SITE_ID     = bas.PARTY_SITE_ID
JOIN    HZ_LOCATIONS              bl   ON bl.LOCATION_ID        = bps.LOCATION_ID
-- ship-to chain (LEFT: some invoices have no ship-to, e.g. services)
LEFT JOIN HZ_CUST_SITE_USES_ALL   ssu  ON ssu.SITE_USE_ID       = rct.SHIP_TO_SITE_USE_ID
LEFT JOIN HZ_CUST_ACCT_SITES_ALL  sas  ON sas.CUST_ACCT_SITE_ID = ssu.CUST_ACCT_SITE_ID
LEFT JOIN HZ_PARTY_SITES          sps  ON sps.PARTY_SITE_ID     = sas.PARTY_SITE_ID
LEFT JOIN HZ_LOCATIONS            sl   ON sl.LOCATION_ID        = sps.LOCATION_ID
WHERE   rct.TRX_NUMBER = :P_TRX_NUMBER
AND     rct.COMPLETE_FLAG = 'Y';
```

**BIP talking points:** In a real BIP data model you'd split this into **two data sets**, header (one row) and lines (many), linked by `CUSTOMER_TRX_ID`, so the RTF template can loop lines under one header. Bursting by `TRX_NUMBER` lets you email each customer their own PDF.
</details>

---

### O2C-7 🟡 "Warehouse dashboard"

> **Interviewer:** *"The warehouse manager wants: (a) deliveries shipped yesterday per warehouse with line counts, and (b) lines **stuck** in 'released to warehouse' for more than 2 days."*

**Testing:** WSH table roles, RELEASED_STATUS, WHO columns.

<details><summary><b>Show solution</b></summary>

```sql
-- (a) shipped yesterday
SELECT  wnd.ORGANIZATION_ID,
        COUNT(DISTINCT wnd.DELIVERY_ID)        AS deliveries,
        COUNT(wdd.DELIVERY_DETAIL_ID)          AS lines_shipped,
        SUM(wdd.SHIPPED_QUANTITY)              AS units
FROM    WSH_NEW_DELIVERIES       wnd
JOIN    WSH_DELIVERY_ASSIGNMENTS wda ON wda.DELIVERY_ID        = wnd.DELIVERY_ID
JOIN    WSH_DELIVERY_DETAILS     wdd ON wdd.DELIVERY_DETAIL_ID = wda.DELIVERY_DETAIL_ID
WHERE   wdd.RELEASED_STATUS = 'C'
AND     wnd.INITIAL_PICKUP_DATE >= TRUNC(SYSDATE) - 1
AND     wnd.INITIAL_PICKUP_DATE <  TRUNC(SYSDATE)
GROUP BY wnd.ORGANIZATION_ID;

-- (b) stuck in released-to-warehouse
SELECT  wdd.ORGANIZATION_ID, wdd.SALES_ORDER_NUMBER, wdd.SALES_ORDER_LINE_NUMBER,
        wdd.REQUESTED_QUANTITY, wdd.LAST_UPDATE_DATE,
        TRUNC(SYSDATE) - TRUNC(wdd.LAST_UPDATE_DATE) AS days_stuck
FROM    WSH_DELIVERY_DETAILS wdd
WHERE   wdd.RELEASED_STATUS = 'S'
AND     wdd.LAST_UPDATE_DATE < SYSDATE - 2;
```

**Talking points:**
- `COUNT(DISTINCT DELIVERY_ID)` because joining to details repeats each delivery per line.
- Date filter as a **range** (`>= … AND < …`) instead of `TRUNC(INITIAL_PICKUP_DATE) = TRUNC(SYSDATE-1)`, because wrapping the column in TRUNC blocks index use. **Seniors notice this.**
- `LAST_UPDATE_DATE` is a standard "WHO" column (CREATED_BY, CREATION_DATE, LAST_UPDATED_BY, LAST_UPDATE_DATE) present on almost every Fusion table.
- Container rows (`PARENT_DELIVERY_DETAIL_ID`) can inflate line counts. Filter to item rows if packing is used.
</details>

---

### O2C-8 🟡 "Customer master data quality"

> **Interviewer:** *"Master data team asks: (a) find **active customer accounts with no active primary BILL_TO**, which will break invoicing, and (b) find **possible duplicate customers** (same name, same city)."*

**Testing:** HZ structure depth, NOT EXISTS, and normalization.

<details><summary><b>Show solution</b></summary>

```sql
-- (a) accounts without an active primary bill-to
SELECT  hp.PARTY_NAME, hca.ACCOUNT_NUMBER
FROM    HZ_CUST_ACCOUNTS hca
JOIN    HZ_PARTIES       hp ON hp.PARTY_ID = hca.PARTY_ID
WHERE   hca.STATUS = 'A'
AND     NOT EXISTS (
          SELECT 1
          FROM   HZ_CUST_ACCT_SITES_ALL hcas
          JOIN   HZ_CUST_SITE_USES_ALL  hcsu ON hcsu.CUST_ACCT_SITE_ID = hcas.CUST_ACCT_SITE_ID
          WHERE  hcas.CUST_ACCOUNT_ID = hca.CUST_ACCOUNT_ID
          AND    hcas.STATUS = 'A'
          AND    hcsu.STATUS = 'A'
          AND    hcsu.SITE_USE_CODE = 'BILL_TO'
          AND    hcsu.PRIMARY_FLAG  = 'Y');

-- (b) possible duplicate customers
WITH c AS (
  SELECT  hp.PARTY_ID, hp.PARTY_NAME, hl.CITY,
          REGEXP_REPLACE(UPPER(hp.PARTY_NAME), '(PVT|PRIVATE|LTD|LIMITED|[^A-Z0-9])', '') AS norm_name
  FROM    HZ_PARTIES hp
  JOIN    HZ_PARTY_SITES hps ON hps.PARTY_ID = hp.PARTY_ID AND hps.IDENTIFYING_ADDRESS_FLAG = 'Y'
  JOIN    HZ_LOCATIONS   hl  ON hl.LOCATION_ID = hps.LOCATION_ID
  WHERE   hp.STATUS = 'A'
  AND     EXISTS (SELECT 1 FROM HZ_CUST_ACCOUNTS hca WHERE hca.PARTY_ID = hp.PARTY_ID)  -- customers only
)
SELECT  a.PARTY_NAME, b.PARTY_NAME AS possible_duplicate, a.CITY
FROM    c a JOIN c b ON b.norm_name = a.norm_name
                    AND UPPER(b.CITY) = UPPER(a.CITY)
                    AND b.PARTY_ID > a.PARTY_ID;
```

**Talking points:** Status must be checked at **each layer** (account, account site, site use). Removing "PVT/LTD" catches "ABC Pvt Ltd" vs "ABC Private Limited". `EXISTS` on `HZ_CUST_ACCOUNTS` limits to customers, because HZ_PARTIES also contains suppliers, contacts and banks.
</details>

---

### O2C-9 🔴 "Order-to-cash cycle time"

> **Interviewer:** *"The CFO wants the average **order-to-cash cycle time** per customer for last quarter: order → ship, ship → invoice, invoice → cash. How would you build it, and what will go wrong?"*

**Testing:** Multi-stage timestamps, choosing **first/last** events, and avoiding fan-out when a line has many shipments/receipts.

<details><summary><b>Show solution</b></summary>

**Design (say this first):** compute **one row per order line** with 4 dates, each from its own aggregated subquery:
| Stage date | Source | Rule |
|------------|--------|------|
| order date | `DOO_HEADERS_ALL.ORDERED_DATE` | |
| ship date | `MAX(DOO_FULFILL_LINES_ALL.ACTUAL_SHIP_DATE)` per line | fully shipped = last shipment |
| invoice date | `MIN(RA_CUSTOMER_TRX_ALL.TRX_DATE)` per line | first invoice |
| cash date | `MAX(AR_RECEIVABLE_APPLICATIONS_ALL.APPLY_DATE)` on that invoice | fully paid = last application |

```sql
WITH ship AS (
  SELECT LINE_ID, MAX(ACTUAL_SHIP_DATE) AS ship_date
  FROM   DOO_FULFILL_LINES_ALL WHERE ACTUAL_SHIP_DATE IS NOT NULL GROUP BY LINE_ID
),
inv AS (
  SELECT rctl.INTERFACE_LINE_ATTRIBUTE1 AS hdr, rctl.INTERFACE_LINE_ATTRIBUTE2 AS line_no,
         MIN(rct.TRX_DATE) AS inv_date, MIN(rct.CUSTOMER_TRX_ID) AS trx_id
  FROM   RA_CUSTOMER_TRX_LINES_ALL rctl
  JOIN   RA_CUSTOMER_TRX_ALL rct ON rct.CUSTOMER_TRX_ID = rctl.CUSTOMER_TRX_ID
  WHERE  rctl.LINE_TYPE = 'LINE'
  GROUP BY rctl.INTERFACE_LINE_ATTRIBUTE1, rctl.INTERFACE_LINE_ATTRIBUTE2
),
cash AS (
  SELECT aps.CUSTOMER_TRX_ID, MAX(ara.APPLY_DATE) AS cash_date
  FROM   AR_PAYMENT_SCHEDULES_ALL aps
  JOIN   AR_RECEIVABLE_APPLICATIONS_ALL ara ON ara.APPLIED_PAYMENT_SCHEDULE_ID = aps.PAYMENT_SCHEDULE_ID
  WHERE  ara.STATUS = 'APP' AND ara.DISPLAY = 'Y'
  AND    aps.STATUS = 'CL'                                  -- only fully paid invoices
  GROUP BY aps.CUSTOMER_TRX_ID
)
SELECT  hp.PARTY_NAME,
        ROUND(AVG(s.ship_date - dha.ORDERED_DATE),1) AS avg_order_to_ship,
        ROUND(AVG(i.inv_date  - s.ship_date),1)      AS avg_ship_to_invoice,
        ROUND(AVG(c.cash_date - i.inv_date),1)       AS avg_invoice_to_cash,
        ROUND(AVG(c.cash_date - dha.ORDERED_DATE),1) AS avg_total_cycle
FROM    DOO_HEADERS_ALL dha
JOIN    HZ_PARTIES      hp  ON hp.PARTY_ID  = dha.SOLD_TO_PARTY_ID
JOIN    DOO_LINES_ALL   dla ON dla.HEADER_ID = dha.HEADER_ID
JOIN    ship s ON s.LINE_ID = dla.LINE_ID
JOIN    inv  i ON i.hdr = TO_CHAR(dla.HEADER_ID) AND i.line_no = TO_CHAR(dla.LINE_NUMBER)
JOIN    cash c ON c.CUSTOMER_TRX_ID = i.trx_id
WHERE   dha.SUBMITTED_FLAG = 'Y' AND dha.STATUS_CODE <> 'DOO_REFERENCE'
AND     dha.ORDERED_DATE BETWEEN :P_FROM AND :P_TO
GROUP BY hp.PARTY_NAME;
```

**What will go wrong (they asked!):**
- **Survivorship bias:** INNER JOINs keep only fully completed cycles. Orders still unpaid are excluded, which makes the cycle look **shorter** than reality. Mention it and offer an "open items age" companion metric.
- One invoice covering many orders, or one line invoiced in several invoices, needs MIN/MAX rules (agree them with the business).
- Credit memos / write-offs close an invoice without cash. Exclude `APPLICATION_TYPE = 'CM'` if you want **cash** only.
</details>

---

### O2C-10 🔴 "Returns: RMA → credit memo → applied"

> **Interviewer:** *"A customer returned 10 defective units. Show me the return order, the credit memo created for it, and whether that credit was **applied to the original invoice** or is still sitting open."*

**Testing:** Return lines in DOO, credit memos in RA/AR (CLASS='CM'), `PREVIOUS_CUSTOMER_TRX_ID`, and CM applications.

<details><summary><b>Show solution</b></summary>

| Noun | Table / column |
|------|----------------|
| return order | `DOO_LINES_ALL.CATEGORY_CODE = 'RETURN'` |
| credit memo | `RA_CUSTOMER_TRX_ALL` whose `AR_PAYMENT_SCHEDULES_ALL.CLASS = 'CM'` |
| link CM → original invoice | `RA_CUSTOMER_TRX_ALL.PREVIOUS_CUSTOMER_TRX_ID` (CM points to the invoice it credits) |
| CM applied? | `AR_RECEIVABLE_APPLICATIONS_ALL` with `APPLICATION_TYPE = 'CM'` (`CUSTOMER_TRX_ID` = CM, `APPLIED_CUSTOMER_TRX_ID` = invoice) |
| still open? | CM's `AR_PAYMENT_SCHEDULES_ALL.AMOUNT_DUE_REMAINING <> 0` |

```sql
SELECT  dha.ORDER_NUMBER             AS return_order,
        dla.DISPLAY_LINE_NUMBER,
        dla.ORDERED_QTY              AS returned_qty,
        cm.TRX_NUMBER                AS credit_memo,
        orig.TRX_NUMBER              AS original_invoice,
        cm_ps.AMOUNT_DUE_ORIGINAL    AS cm_amount,          -- negative
        cm_ps.AMOUNT_DUE_REMAINING   AS cm_unapplied,
        ara.AMOUNT_APPLIED,
        applied_inv.TRX_NUMBER       AS applied_to_invoice
FROM    DOO_HEADERS_ALL dha
JOIN    DOO_LINES_ALL   dla ON dla.HEADER_ID = dha.HEADER_ID AND dla.CATEGORY_CODE = 'RETURN'
LEFT JOIN RA_CUSTOMER_TRX_LINES_ALL cml ON cml.INTERFACE_LINE_ATTRIBUTE1 = TO_CHAR(dla.HEADER_ID)
                                       AND cml.INTERFACE_LINE_ATTRIBUTE2 = TO_CHAR(dla.LINE_NUMBER)
                                       AND cml.LINE_TYPE = 'LINE'
LEFT JOIN RA_CUSTOMER_TRX_ALL      cm    ON cm.CUSTOMER_TRX_ID   = cml.CUSTOMER_TRX_ID
LEFT JOIN RA_CUSTOMER_TRX_ALL      orig  ON orig.CUSTOMER_TRX_ID = cm.PREVIOUS_CUSTOMER_TRX_ID
LEFT JOIN AR_PAYMENT_SCHEDULES_ALL cm_ps ON cm_ps.CUSTOMER_TRX_ID = cm.CUSTOMER_TRX_ID
                                        AND cm_ps.CLASS = 'CM'
LEFT JOIN AR_RECEIVABLE_APPLICATIONS_ALL ara ON ara.CUSTOMER_TRX_ID = cm.CUSTOMER_TRX_ID
                                            AND ara.APPLICATION_TYPE = 'CM'
                                            AND ara.STATUS = 'APP' AND ara.DISPLAY = 'Y'
LEFT JOIN RA_CUSTOMER_TRX_ALL      applied_inv ON applied_inv.CUSTOMER_TRX_ID = ara.APPLIED_CUSTOMER_TRX_ID
WHERE   dha.ORDER_NUMBER = :P_RETURN_ORDER;
```

**Talking points:**
- For **cash** applications the bridge is keyed by `CASH_RECEIPT_ID`. For **credit memo** applications it's keyed by the CM's `CUSTOMER_TRX_ID`. Same table, two uses, and that's the senior-level insight here.
- If `cm_unapplied <> 0` → the credit is **on the customer's account** and can be applied to a future invoice or refunded.
- Return lines also create an inventory transaction (stock **+**) when received back. That ties O2C back to `INV_MATERIAL_TXNS`.
- Column names around RMA→CM can vary by setup, so verify `INTERFACE_LINE_ATTRIBUTE` usage for return lines in your instance.
</details>

---

## Part 4 — Senior-level follow-ups that apply to any scenario

These are the questions a 20-year veteran asks **after** you've written the query. Have a 2–3 line answer ready for each.

<details><summary><b>1. "Your query is slow on production. What do you check?"</b></summary>

- Am I filtering on **indexed columns** (IDs, numbers, dates) without wrapping them in functions (`TRUNC(col)`, `UPPER(col)`, `TO_CHAR(col)`)? Use ranges instead.
- Did I add a **date range / BU** filter to limit the scan?
- Am I joining big tables **before** aggregating? Aggregate in CTEs first.
- `EXISTS` instead of `IN`/`DISTINCT` for yes/no checks.
- Avoid `SELECT *` and unnecessary views (some `_V` views join many tables internally).
- Use **bind parameters** (`:P_…`) so the plan is reused.
- Look at the explain plan for full table scans on large tables (AP/AR dists, INV txns).
</details>

<details><summary><b>2. "These are _ALL tables. What about security?"</b></summary>

`_ALL` tables contain every business unit. A raw SQL data model in BI Publisher does **not** automatically restrict rows to what the user is allowed to see, so I add a **BU parameter** (`ORG_ID`, `PRC_BU_ID`) and, where needed, restrict the parameter's list of values to the user's authorized BUs (or use secured views). Without that, a user could see another BU's data. This is a real audit finding in many projects.
</details>

<details><summary><b>3. "How does AP/AR data reach the GL? Can you join AP_INVOICES_ALL to GL_JE_LINES directly?"</b></summary>

No direct join. Subledgers create accounting through **Subledger Accounting (SLA)**: `XLA_AE_HEADERS` / `XLA_AE_LINES` (with `XLA_TRANSACTION_ENTITIES` mapping back to the source document). Then **Transfer to GL** creates journals in `GL_JE_HEADERS`/`GL_JE_LINES` (`JE_SOURCE` = Payables/Receivables), linked via `GL_IMPORT_REFERENCES`. For reconciliation you go source → XLA → GL.
</details>

<details><summary><b>4. "How do you find which table a field on the Fusion screen comes from?"</b></summary>

- Oracle's published **Tables and Views** guides for Fusion (per module, per release).
- OTBI: find the column in a **subject area** and check its physical mapping/logical SQL.
- Ask a functional consultant for the business meaning, then confirm with a sample SQL in BIP comparing UI values with table values for one known record.

Fusion has no direct DB access; you query through BIP data models.
</details>

<details><summary><b>5. "Why did you use a CTE / subquery instead of joining everything?"</b></summary>

To control **grain**. Each CTE returns one row per key (per PO, per invoice, per customer), so the final joins are 1-to-1 and amounts don't multiply (fan-out). It also makes the query readable and testable stage by stage.
</details>

<details><summary><b>6. "How would you deliver this report to 200 customers / suppliers?"</b></summary>

BIP **bursting**: the data model has a bursting query that splits output by a key (customer/supplier) and routes each part to its email/FTP. Schedule it via the BIP scheduler (or ESS job). Keep the data model parameter-driven.
</details>

<details><summary><b>7. "What's the difference between a current-state report and an as-of-date report?"</b></summary>

Current-state uses running totals/statuses (`QUANTITY_RECEIVED`, `AMOUNT_DUE_REMAINING`, `STATUS`). As-of-date must be **rebuilt from transactions** dated ≤ the as-of date (receipts, applications, payments), because the running columns only show *today*. Seniors ask this about aging and accruals.
</details>

<details><summary><b>8. "What's one thing you'd validate with the business before building any of these?"</b></summary>

**Definitions.** For example: "on-time" (need-by or promised? full or partial?), "paid" (payment created or cleared at bank?), "shipped" (ship confirm or proof of delivery?), "revenue" (invoiced or recognized?). Wrong definitions = correct SQL, wrong report.
</details>

---

## Part 5 — Practice tracker

Do each scenario **without looking**, then compare. Tick the box when you can explain it aloud in under 3 minutes.

| # | Scenario | Key tables | Core trick | ✅ |
|---|----------|-----------|------------|----|
| P2P-1 | Supplier not paid | PO → RCV → AP dist → AP PS → AP payments | Diagnosis ladder with CASE | ☐ |
| P2P-2 | Accrual: received not invoiced | PO_DISTRIBUTIONS (delivered − billed) | Grain = distribution (charge account) | ☐ |
| P2P-3 | Paid more than received | PLL + AP lines | Receipt-required flag, billed > received | ☐ |
| P2P-4 | Duplicate invoices | AP_INVOICES self-join | Normalize, `id > id` | ☐ |
| P2P-5 | Stock ≠ receipts | RCV_TRANSACTIONS, INV_MATERIAL_TXNS, onhand | RCV↔INV bridge, ins vs outs | ☐ |
| P2P-6 | Committed / invoiced / paid | PO dist, AP dist, AP payments | Fan-out → CTE per measure | ☐ |
| P2P-7 | Supplier scorecard | PLL + RCV_TRANSACTIONS | MIN date, CASE by type, NULLIF | ☐ |
| P2P-8 | Cash forecast | AP_PAYMENT_SCHEDULES | "Payable" definition, NOT EXISTS | ☐ |
| P2P-9 | Voided payment | AP_CHECKS, AP_INVOICE_PAYMENTS | Reversal rows, AP→XLA→GL | ☐ |
| P2P-10 | Inactive suppliers | POZ + EXISTS | EXISTS for yes/no, pay-site control | ☐ |
| O2C-1 | Billed > shipped | DOO, WSH, RA lines | 3 quantities, separate aggregation | ☐ |
| O2C-2 | Backlog value | DOO fulfill lines | Qty from fulfill, price from line | ☐ |
| O2C-3 | Shipped not invoiced | DOO + NOT EXISTS RA | AutoInvoice failure reasons | ☐ |
| O2C-4 | Aging + net exposure | AR_PAYMENT_SCHEDULES | CLASS: INV vs CM/PMT | ☐ |
| O2C-5 | "I paid but still overdue" | AR receipts, applications | UNAPP/UNID/ACC/REV ladder | ☐ |
| O2C-6 | Invoice print | RA + HZ ×2 | Address chain twice, tax link | ☐ |
| O2C-7 | Warehouse dashboard | WSH ×3 | RELEASED_STATUS, date ranges | ☐ |
| O2C-8 | Customer data quality | HZ | Status at each layer, normalize | ☐ |
| O2C-9 | Cycle time | DOO, RA, AR apps | MIN/MAX per stage, survivorship bias | ☐ |
| O2C-10 | Returns → CM → applied | DOO return, RA CM, AR apps | CM applications keyed by CUSTOMER_TRX_ID | ☐ |
