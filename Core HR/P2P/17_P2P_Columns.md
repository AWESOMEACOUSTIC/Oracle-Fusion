# P2P — Important Columns Memory Guide
### Ask me any P2P table → I can say why it matters and its key columns in 10 seconds

> **How this guide makes columns stick (3 layers):**
> 1. **One recall template** (`I-L-P-Q-S-D-A`): every table's columns are recalled in the **same 7 slots**, so you never start from a blank mind.
> 2. **The column story**: the life of one PO told **event by event**, showing exactly **which row is created** and **which column changes**. A column is easy to remember when you know **the event that fills it**.
> 3. **Table cards**: for each table: why it matters, its "10-second answer", and the columns in the 7 slots.
>
> No made-up analogies here. Everything is the actual technical behaviour.
>
> ⚠️ Column names follow the standard Fusion data model. Confirm in your instance with `SELECT * FROM <table> WHERE ROWNUM < 5`.

---

## Table of Contents
1. [Column-name grammar (read a column, know its meaning)](#1-column-name-grammar)
2. [The recall template: I-L-P-Q-S-D-A](#2-the-recall-template-i-l-p-q-s-d-a)
3. [The two golden rules of P2P columns](#3-the-two-golden-rules-of-p2p-columns)
4. [The column story: life of PO-1001, event by event](#4-the-column-story)
5. [Table cards (all 21 tables)](#5-table-cards)
6. [Lookalike columns that trap people](#6-lookalike-columns-that-trap-people)
7. [Rapid-fire drill](#7-rapid-fire-drill)
8. [One-page revision sheet](#8-one-page-revision-sheet)

---

## 1. Column-name grammar

Before memorizing anything, learn to **read** a column name. The **suffix** tells you what kind of data it holds.

| If the column ends with… | It holds… | Example |
|--------------------------|-----------|---------|
| `_ID` | An **internal number** (system key). Users never see it. Used for **joins**. | `PO_HEADER_ID`, `VENDOR_ID` |
| `SEGMENT1`, `_NUM`, `_NUMBER` | The **number the user sees** on screen | `SEGMENT1` (PO no.), `INVOICE_NUM`, `RECEIPT_NUM`, `CHECK_NUMBER` |
| `_FLAG` | **Y / N** (sometimes a few letters) | `PAY_SITE_FLAG`, `HOLD_FLAG` |
| `_CODE`, `_LOOKUP_CODE` | A value from a **fixed list** (lookup) | `TYPE_LOOKUP_CODE` = STANDARD/BLANKET |
| `_STATUS`, `STATUS_` | Where the record is in its **lifecycle** | `DOCUMENT_STATUS`, `STATUS_LOOKUP_CODE` |
| `QUANTITY_…` | A quantity; on PO tables these are **running totals** updated by later events | `QUANTITY_RECEIVED`, `QUANTITY_BILLED` |
| `AMOUNT`, `_AMOUNT` | Money | `INVOICE_AMOUNT`, `AMOUNT_REMAINING` |
| `_DATE` | A date | `NEED_BY_DATE`, `DUE_DATE` |
| `CODE_COMBINATION_ID` / `_CCID` | A **GL account** (the full account string like 01-510-6100-000) | `CODE_COMBINATION_ID` |
| `ORG_ID`, `_BU_ID` | A **Business Unit** (who owns the transaction) | `ORG_ID`, `PRC_BU_ID` |
| `ORGANIZATION_ID`, `_ORG_ID` on RCV/INV | An **Inventory Organization** (a warehouse/plant) | `SHIP_TO_ORGANIZATION_ID` |

> ⚠️ **ORG_ID ≠ ORGANIZATION_ID.** `ORG_ID` = Business Unit (financial owner). `ORGANIZATION_ID` = inventory org (physical warehouse). This is a classic interview trick.

**Every table also has the 4 WHO columns:** `CREATED_BY`, `CREATION_DATE`, `LAST_UPDATED_BY`, `LAST_UPDATE_DATE`. They're never "important columns" on their own, but they're always there if someone asks "who created this PO / when".

---

## 2. The recall template: I-L-P-Q-S-D-A

When someone names a table, **walk these 7 slots in order**. Almost every P2P column falls into one of them.

| Slot | Stands for | Question to ask yourself |
|------|-----------|--------------------------|
| **I** | **Identity** | What's the primary key, and what number does the user see? |
| **L** | **Links** (to parent rows) | Which parent IDs does it carry? |
| **P** | **Parties & Places** | Which supplier / site / buyer / BU / warehouse? |
| **Q** | **Quantities & amounts** | What's counted or charged? |
| **S** | **Status & flags** | What lifecycle state / Y-N controls? |
| **D** | **Dates** | Which business dates? |
| **A** | **Accounting** | Which GL account / posted / period? |

**Mnemonic:** **"I Love Paneer, Quite Spicy, Daily, Always."**

Not every table fills all 7 slots. A missing slot is also useful to know (e.g. `RCV_SHIPMENT_HEADERS` has **no** PO link, which is a classic question).

---

## 3. The two golden rules of P2P columns

These two rules explain about half of all the important columns.

### Rule 1 — Children POINT to parents (ID columns)
A lower-level row stores its parent's ID.
`PO_LINES_ALL.PO_HEADER_ID`, `PO_DISTRIBUTIONS_ALL.LINE_LOCATION_ID`, `RCV_TRANSACTIONS.PO_DISTRIBUTION_ID`, `AP_INVOICE_DISTRIBUTIONS_ALL.PO_DISTRIBUTION_ID`…

### Rule 2 — Parents COUNT what happened later (QUANTITY_* / AMOUNT_* columns)
When a later event happens (receipt, delivery, invoice, payment), the system **updates running totals on the earlier document**:

| Later event | Updates which earlier table | Column(s) |
|-------------|-----------------------------|-----------|
| RECEIVE | `PO_LINE_LOCATIONS_ALL` (schedule) | `QUANTITY_RECEIVED` |
| ACCEPT / REJECT (inspection) | `PO_LINE_LOCATIONS_ALL` | `QUANTITY_ACCEPTED`, `QUANTITY_REJECTED` |
| DELIVER | `PO_DISTRIBUTIONS_ALL` | `QUANTITY_DELIVERED` |
| Invoice matched | `PO_LINE_LOCATIONS_ALL` **and** `PO_DISTRIBUTIONS_ALL` | `QUANTITY_BILLED` (and amount billed) |
| Cancel | schedule & distribution | `QUANTITY_CANCELLED` |
| Payment | `AP_INVOICES_ALL` | `AMOUNT_PAID`, `PAYMENT_STATUS_FLAG` |
| Payment | `AP_PAYMENT_SCHEDULES_ALL` | `AMOUNT_REMAINING`, `PAYMENT_STATUS_FLAG` |

> **Why does RECEIVE update the schedule but DELIVER updates the distribution?**
> Receiving happens at the **dock**, for a schedule (when/where the goods were due). Delivering moves goods to their **final destination and charge account**, and destination + account live on the **distribution**. That's why `QUANTITY_RECEIVED` is on the schedule and `QUANTITY_DELIVERED` is on the distribution.

**Short form:** **"Children point up. Parents count down."**

---

## 4. The column story

**Life of PO-1001**: ABC Corp buys **100 laptops @ ₹50,000** from **Dell India**. 60 go to Bangalore (split 30 IT / 30 HR), 40 to Chennai. Only the Bangalore 60 arrive, get invoiced and paid.

Read each event as: **what happened → rows created (➕) → columns changed (✏️)**. This is the main memory tool. When you recall the event, the columns come with it.

---

### Event 0 — Supplier is set up
➕ `POZ_SUPPLIERS_V`: VENDOR_ID = 9001, **SEGMENT1** = 10045 (supplier number), **VENDOR_NAME** = Dell India, **ENABLED_FLAG** = Y
➕ `POZ_SUPPLIER_SITES_V`: VENDOR_SITE_ID = 9101, **VENDOR_SITE_CODE** = DELL-BLR, **PRC_BU_ID** = 300, **PURCHASING_SITE_FLAG** = Y, **PAY_SITE_FLAG** = Y

> Remember: a site must have `PURCHASING_SITE_FLAG = Y` to receive POs and `PAY_SITE_FLAG = Y` to receive payments.

---

### Event 1 — Buyer creates the PO (not yet approved)
➕ `PO_HEADERS_ALL`: PO_HEADER_ID = 5001, **SEGMENT1** = PO-1001, **TYPE_LOOKUP_CODE** = STANDARD, **VENDOR_ID** = 9001, **VENDOR_SITE_ID** = 9101, **AGENT_ID** = buyer, **PRC_BU_ID** = 300, **CURRENCY_CODE** = INR, **DOCUMENT_STATUS** = `INCOMPLETE`
➕ `PO_LINES_ALL`: PO_LINE_ID = 5101, **LINE_NUM** = 1, **ITEM_ID** = 7001, **ITEM_DESCRIPTION**, **CATEGORY_ID**, **QUANTITY** = 100, **UNIT_PRICE** = 50000, **UOM_CODE** = Ea
➕ `PO_LINE_LOCATIONS_ALL` ×2:
| LINE_LOCATION_ID | SHIP_TO_ORGANIZATION_ID | QUANTITY | NEED_BY_DATE | RECEIPT_REQUIRED_FLAG | INSPECTION_REQUIRED_FLAG | QUANTITY_RECEIVED | QUANTITY_BILLED |
|---|---|---|---|---|---|---|---|
| 5201 | 101 (BLR) | 60 | 15-Nov | Y | Y | **0** | **0** |
| 5202 | 102 (CHN) | 40 | 30-Nov | Y | N | **0** | **0** |

➕ `PO_DISTRIBUTIONS_ALL` ×3:
| PO_DISTRIBUTION_ID | LINE_LOCATION_ID | QUANTITY_ORDERED | CODE_COMBINATION_ID | DESTINATION_TYPE_CODE | QUANTITY_DELIVERED | QUANTITY_BILLED |
|---|---|---|---|---|---|---|
| 5301 | 5201 | 30 | 11001 (IT) | INVENTORY | **0** | **0** |
| 5302 | 5201 | 30 | 11002 (HR) | INVENTORY | **0** | **0** |
| 5303 | 5202 | 40 | 11001 (IT) | INVENTORY | **0** | **0** |

> Notice: all the "counter" columns start at **0**. The rest of the story fills them.

---

### Event 2 — PO is submitted and approved
✏️ `PO_HEADERS_ALL.DOCUMENT_STATUS`: `INCOMPLETE` → `PENDING APPROVAL` → **`OPEN`**
✏️ `PO_HEADERS_ALL.APPROVED_DATE` gets filled

> **OPEN = approved and active.** Later life: `CLOSED FOR RECEIVING`, `CLOSED FOR INVOICING`, `CLOSED`, `FINALLY CLOSED`, or `CANCELED`.

---

### Event 3 — 60 laptops arrive at the Bangalore dock (RECEIVE)
➕ `RCV_SHIPMENT_HEADERS`: SHIPMENT_HEADER_ID = 3001, **RECEIPT_NUM** = RCPT-3001, **RECEIPT_SOURCE_CODE** = VENDOR, **VENDOR_ID**, **VENDOR_SITE_ID**, **SHIP_TO_ORG_ID** = 101
➕ `RCV_SHIPMENT_LINES`: SHIPMENT_LINE_ID = 3101, **ITEM_ID** = 7001, **QUANTITY_RECEIVED** = 60, **PO_HEADER_ID / PO_LINE_ID / PO_LINE_LOCATION_ID**, **SHIPMENT_LINE_STATUS_CODE** = FULLY RECEIVED
➕ `RCV_TRANSACTIONS`: TRANSACTION_ID = 4001, **TRANSACTION_TYPE** = `RECEIVE`, **QUANTITY** = 60, **PARENT_TRANSACTION_ID** = -1 (no parent; it's the first action), **DESTINATION_TYPE_CODE** = RECEIVING
✏️ `PO_LINE_LOCATIONS_ALL[5201].QUANTITY_RECEIVED`: 0 → **60** *(Rule 2: parent counts)*

> The receipt **header** has no PO ID, because one receipt can cover many POs. PO IDs appear from the **lines** down.

---

### Event 4 — Quality inspection passes (ACCEPT)
➕ `RCV_TRANSACTIONS`: TRANSACTION_ID = 4002, **TRANSACTION_TYPE** = `ACCEPT`, **QUANTITY** = 60, **PARENT_TRANSACTION_ID** = 4001
✏️ `PO_LINE_LOCATIONS_ALL[5201].QUANTITY_ACCEPTED`: 0 → **60**

> Inspection only happens because `INSPECTION_REQUIRED_FLAG = Y` (4-way match). If rejected, `TRANSACTION_TYPE = REJECT` and `QUANTITY_REJECTED` goes up instead.

---

### Event 5 — Laptops are delivered into stock (DELIVER)
➕ `RCV_TRANSACTIONS` ×2 (one per distribution):
| TRANSACTION_ID | TYPE | QUANTITY | PARENT | PO_DISTRIBUTION_ID | SUBINVENTORY | INV_TRANSACTION_ID |
|---|---|---|---|---|---|---|
| 4003 | DELIVER | 30 | 4002 | 5301 | STORES | 9501 |
| 4004 | DELIVER | 30 | 4002 | 5302 | STORES | 9502 |

➕ `INV_MATERIAL_TXNS` ×2: TRANSACTION_ID = 9501 / 9502, **INVENTORY_ITEM_ID** = 7001, **ORGANIZATION_ID** = 101, **SUBINVENTORY_CODE** = STORES, **TRANSACTION_QUANTITY** = **+30**, **TRANSACTION_SOURCE_TYPE_ID** = purchase order, **TRANSACTION_DATE**
➕/✏️ `INV_ONHAND_QUANTITIES_DETAIL`: rows for item 7001 / org 101 / STORES, **PRIMARY_TRANSACTION_QUANTITY** total = **60**
✏️ `PO_DISTRIBUTIONS_ALL[5301].QUANTITY_DELIVERED`: 0 → **30**
✏️ `PO_DISTRIBUTIONS_ALL[5302].QUANTITY_DELIVERED`: 0 → **30**

> This one event touches **4 tables**: RCV_TRANSACTIONS (action), INV_MATERIAL_TXNS (stock history), INV_ONHAND (stock balance), PO_DISTRIBUTIONS (counter).
> `INV_TRANSACTION_ID` on the RCV row stores the ID of the inventory row it created.

---

### Event 6 — Dell's invoice is entered and matched to the receipt
➕ `AP_INVOICES_ALL`: INVOICE_ID = 8001, **INVOICE_NUM** = DELL-INV-778, **INVOICE_TYPE_LOOKUP_CODE** = STANDARD, **VENDOR_ID / VENDOR_SITE_ID**, **INVOICE_AMOUNT** = 30,00,000, **INVOICE_CURRENCY_CODE** = INR, **INVOICE_DATE**, **GL_DATE**, **TERMS_ID**, **SOURCE** (Manual / spreadsheet / import), **ORG_ID** = 300, **AMOUNT_PAID** = 0, **PAYMENT_STATUS_FLAG** = `N`
➕ `AP_INVOICE_LINES_ALL`: INVOICE_ID = 8001, **LINE_NUMBER** = 1, **LINE_TYPE_LOOKUP_CODE** = ITEM, **AMOUNT** = 30,00,000, **QUANTITY_INVOICED** = 60, **UNIT_PRICE** = 50000, **PO_HEADER_ID / PO_LINE_ID / PO_LINE_LOCATION_ID**, **MATCH_TYPE** = ITEM_TO_RECEIPT
➕ `AP_INVOICE_DISTRIBUTIONS_ALL` ×2: **INVOICE_LINE_NUMBER** = 1, **AMOUNT** = 15,00,000 each, **DIST_CODE_COMBINATION_ID**, **PO_DISTRIBUTION_ID** = 5301 / 5302, **RCV_TRANSACTION_ID** = 4003 / 4004, **MATCH_STATUS_FLAG** = (null, not yet validated), **POSTED_FLAG** = `N`
➕ `AP_PAYMENT_SCHEDULES_ALL` ×2 (terms = 2 installments): **PAYMENT_NUM** = 1 / 2, **DUE_DATE**, **GROSS_AMOUNT** = 15,00,000, **AMOUNT_REMAINING** = 15,00,000, **PAYMENT_STATUS_FLAG** = `N`
✏️ `PO_LINE_LOCATIONS_ALL[5201].QUANTITY_BILLED`: 0 → **60**
✏️ `PO_DISTRIBUTIONS_ALL[5301/5302].QUANTITY_BILLED`: 0 → **30** each

> Payment schedules are created from the **payment terms** (`TERMS_ID`). That's why one invoice can have several due dates.

---

### Event 7 — Invoice is validated
✏️ `AP_INVOICE_DISTRIBUTIONS_ALL.MATCH_STATUS_FLAG`: null → **`A`** (validated)

> If validation finds a mismatch (price/qty/receipt), the flag becomes **`T`** (tested, has holds) and the hold is recorded in `AP_HOLDS_ALL` (outside our sheet). A held invoice cannot be paid.

---

### Event 8 — Invoice accounting is created
✏️ `AP_INVOICE_DISTRIBUTIONS_ALL.POSTED_FLAG`: N → **`Y`**
✏️ `AP_INVOICE_DISTRIBUTIONS_ALL.ACCOUNTING_EVENT_ID`: filled (link to Subledger Accounting)

---

### Event 9 — Dell is paid, installment 1
➕ `AP_CHECKS_ALL`: CHECK_ID = 6001, **CHECK_NUMBER** = CHK-6001, **CHECK_DATE**, **AMOUNT** = 15,00,000, **CURRENCY_CODE**, **VENDOR_ID / VENDOR_SITE_ID**, **PAYMENT_METHOD_CODE** = EFT, **STATUS_LOOKUP_CODE** = `NEGOTIABLE`
➕ `AP_INVOICE_PAYMENTS_ALL`: **CHECK_ID** = 6001, **INVOICE_ID** = 8001, **PAYMENT_NUM** = 1, **AMOUNT** = 15,00,000, **ACCOUNTING_DATE**, **REVERSAL_FLAG** = N
✏️ `AP_PAYMENT_SCHEDULES_ALL[8001, 1].AMOUNT_REMAINING`: 15,00,000 → **0**; **PAYMENT_STATUS_FLAG** → `Y`
✏️ `AP_INVOICES_ALL[8001].AMOUNT_PAID`: 0 → **15,00,000**; **PAYMENT_STATUS_FLAG**: N → **`P`** (partially paid)

### Event 10 — Installment 2 paid, bank clears payment 1
➕ `AP_CHECKS_ALL` (CHK-6002) + `AP_INVOICE_PAYMENTS_ALL` (PAYMENT_NUM = 2)
✏️ `AP_INVOICES_ALL.AMOUNT_PAID` → **30,00,000**; **PAYMENT_STATUS_FLAG** → **`Y`** (fully paid)
✏️ `AP_CHECKS_ALL[6001].STATUS_LOOKUP_CODE`: NEGOTIABLE → **`CLEARED`**; **CLEARED_DATE** filled

> If a payment is voided instead: `STATUS_LOOKUP_CODE = VOIDED`, `VOID_DATE` filled, and a **negative reversal row** is added in `AP_INVOICE_PAYMENTS_ALL` (`REVERSAL_FLAG = Y`).

---

### Event 11 — Transfer to GL and posting
➕ `GL_JE_BATCHES`: **NAME**, **STATUS** = `U` (unposted)
➕ `GL_JE_HEADERS`: **JE_SOURCE** = Payables, **JE_CATEGORY** = Purchase Invoices / Payments, **LEDGER_ID**, **PERIOD_NAME** = Nov-26, **CURRENCY_CODE**, **ACTUAL_FLAG** = A
➕ `GL_JE_LINES`: **CODE_COMBINATION_ID**, **ENTERED_DR/CR**, **ACCOUNTED_DR/CR**, **DESCRIPTION**
✏️ After posting: `GL_JE_BATCHES.STATUS` → **`P`**, `POSTED_DATE` filled

---

### The whole story compressed into one table

| # | Event | New rows in | Columns that change |
|---|-------|-------------|---------------------|
| 0 | Supplier set up | POZ_SUPPLIERS_V, POZ_SUPPLIER_SITES_V | — |
| 1 | PO created | PO_HEADERS, LINES, LINE_LOCATIONS, DISTRIBUTIONS | DOCUMENT_STATUS = INCOMPLETE |
| 2 | PO approved | — | DOCUMENT_STATUS → OPEN, APPROVED_DATE |
| 3 | RECEIVE | RCV_SHIPMENT_HEADERS, LINES, RCV_TRANSACTIONS | PLL.QUANTITY_RECEIVED |
| 4 | ACCEPT | RCV_TRANSACTIONS | PLL.QUANTITY_ACCEPTED |
| 5 | DELIVER | RCV_TRANSACTIONS, INV_MATERIAL_TXNS, INV_ONHAND | PDA.QUANTITY_DELIVERED, RT.INV_TRANSACTION_ID |
| 6 | Invoice matched | AP_INVOICES, LINES, DISTRIBUTIONS, PAYMENT_SCHEDULES | PLL & PDA.QUANTITY_BILLED |
| 7 | Validated | — | AID.MATCH_STATUS_FLAG → A |
| 8 | Accounted | — | AID.POSTED_FLAG → Y |
| 9–10 | Paid | AP_CHECKS, AP_INVOICE_PAYMENTS | APS.AMOUNT_REMAINING → 0, AIA.AMOUNT_PAID, PAYMENT_STATUS_FLAG N→P→Y, CHECKS.STATUS → CLEARED |
| 11 | To GL | GL_JE_BATCHES, HEADERS, LINES | BATCH.STATUS U → P |

> **Practice:** cover the right column and recite it from the event name. Once you can do that, you know the important columns of all 21 tables.

---

## 5. Table cards

Format of each card:
- **Why it matters**: what it's for and what you couldn't answer without it.
- **10-second answer**: what to say if someone asks "tell me about this table".
- **Columns in I-L-P-Q-S-D-A order.** ⭐ = the must-say columns.

---

### 5.1 `POZ_SUPPLIERS_V`
**Why it matters:** The master list of suppliers. Every PO, receipt, invoice and payment refers to a supplier through `VENDOR_ID`.
**10-second answer:** *"One row per supplier. Key is VENDOR_ID, supplier number is SEGMENT1, name is VENDOR_NAME. It links to HZ_PARTIES through PARTY_ID, and ENABLED_FLAG / END_DATE_ACTIVE say if it's active."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `VENDOR_ID` | Internal supplier key |
| I | ⭐ `SEGMENT1` | Supplier number shown to users |
| I | ⭐ `VENDOR_NAME` | Supplier name |
| L | `PARTY_ID` | Link to `HZ_PARTIES` (TCA party record) |
| P | `VENDOR_TYPE_LOOKUP_CODE` | Supplier type (e.g. supplier, employee, tax authority) |
| S | ⭐ `ENABLED_FLAG` | Y = enabled |
| D | `START_DATE_ACTIVE`, ⭐ `END_DATE_ACTIVE` | Active date range |

---

### 5.2 `POZ_SUPPLIER_SITES_V`
**Why it matters:** A supplier can have many addresses/branches. **The site decides whether you can order from it and whether you can pay it.** Sites are defined per procurement BU.
**10-second answer:** *"One row per supplier site per procurement BU. Key VENDOR_SITE_ID, child of VENDOR_ID. VENDOR_SITE_CODE is the site name. PURCHASING_SITE_FLAG allows POs, PAY_SITE_FLAG allows payments."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `VENDOR_SITE_ID` | Internal site key |
| I | ⭐ `VENDOR_SITE_CODE` | Site name shown to users |
| L | ⭐ `VENDOR_ID` | Parent supplier |
| P | ⭐ `PRC_BU_ID` | Procurement BU that owns this site |
| P | `LOCATION_ID` / `PARTY_SITE_ID` | Address link |
| S | ⭐ `PURCHASING_SITE_FLAG` | Y = POs can be sent here |
| S | ⭐ `PAY_SITE_FLAG` | Y = invoices can be paid here |
| D | `INACTIVE_DATE` | Site deactivated from this date |

---

### 5.3 `PO_HEADERS_ALL`
**Why it matters:** The **contract-level** record of a purchase: who we buy from, who the buyer is, currency, status. Every P2P report that says "by PO" starts here.
**10-second answer:** *"One row per PO. PO_HEADER_ID is the key, SEGMENT1 is the PO number. It holds supplier and site (VENDOR_ID, VENDOR_SITE_ID), buyer (AGENT_ID), procurement BU (PRC_BU_ID), currency, type (TYPE_LOOKUP_CODE) and status (DOCUMENT_STATUS)."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `PO_HEADER_ID` | Internal PO key |
| I | ⭐ `SEGMENT1` | PO number shown to users |
| P | ⭐ `VENDOR_ID`, ⭐ `VENDOR_SITE_ID` | Supplier and site |
| P | ⭐ `AGENT_ID` | Buyer (person who owns the PO) |
| P | ⭐ `PRC_BU_ID` | Procurement BU |
| P | `REQ_BU_ID`, `BILLTO_BU_ID`, `SOLDTO_LE_ID` | Requisitioning BU, bill-to BU, buying legal entity |
| Q | `CURRENCY_CODE`, `RATE` | PO currency and exchange rate |
| S | ⭐ `TYPE_LOOKUP_CODE` | STANDARD / BLANKET / CONTRACT |
| S | ⭐ `DOCUMENT_STATUS` | INCOMPLETE, PENDING APPROVAL, OPEN, CLOSED…, CANCELED |
| D | `CREATION_DATE`, `APPROVED_DATE` | Created / approved |
| — | `TERMS_ID` | Payment terms (defaults to the invoice) |

> **No quantity or amount on the header.** Totals come from the lines.

---

### 5.4 `PO_LINES_ALL`
**Why it matters:** **What** we buy and **at what price**. Price lives only here.
**10-second answer:** *"One row per item line on a PO. Key PO_LINE_ID, parent PO_HEADER_ID, LINE_NUM is the line number. Holds ITEM_ID (or just ITEM_DESCRIPTION for services), CATEGORY_ID, QUANTITY, UNIT_PRICE and UOM_CODE."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `PO_LINE_ID` | Internal line key |
| I | ⭐ `LINE_NUM` | Line number shown (1, 2, 3…) |
| L | ⭐ `PO_HEADER_ID` | Parent PO |
| P | ⭐ `ITEM_ID` | Item (= `EGP_SYSTEM_ITEMS_B.INVENTORY_ITEM_ID`); can be null for description-only lines |
| P | ⭐ `ITEM_DESCRIPTION` | What's being bought, in text |
| P | `CATEGORY_ID` | Purchasing category (e.g. IT hardware) |
| P | `LINE_TYPE_ID` | Goods (quantity-based) vs services (amount-based) |
| Q | ⭐ `QUANTITY` | Total ordered on this line |
| Q | ⭐ `UNIT_PRICE` | Price per unit |
| Q | `UOM_CODE` | Unit of measure |
| S | `LINE_STATUS` | Line-level status |

> Line amount = `QUANTITY × UNIT_PRICE`. There's no stored "line amount" you must use.

---

### 5.5 `PO_LINE_LOCATIONS_ALL` (the schedule) ⭐⭐
**Why it matters:** **When and where** goods must arrive, the **match rule** (2/3/4-way), and the **running counters** for received, accepted and billed. It's the most useful PO table for operational reports (open POs, late deliveries, received-not-billed).
**10-second answer:** *"The PO schedule. One row per delivery split of a line. Key LINE_LOCATION_ID. It holds ship-to org, need-by and promised dates, the match flags RECEIPT_REQUIRED_FLAG and INSPECTION_REQUIRED_FLAG, and the counters QUANTITY, QUANTITY_RECEIVED, QUANTITY_ACCEPTED, QUANTITY_BILLED and QUANTITY_CANCELLED."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `LINE_LOCATION_ID` | Internal schedule key |
| I | `SHIPMENT_NUM` | Schedule number shown |
| L | ⭐ `PO_LINE_ID`, `PO_HEADER_ID` | Parent line / header |
| P | ⭐ `SHIP_TO_ORGANIZATION_ID` | Inventory org (warehouse) receiving the goods |
| P | `SHIP_TO_LOCATION_ID` | Ship-to address |
| Q | ⭐ `QUANTITY` | Quantity due on this schedule |
| Q | ⭐ `QUANTITY_RECEIVED` | Counter, updated by RECEIVE |
| Q | `QUANTITY_ACCEPTED` / `QUANTITY_REJECTED` | Counters, updated by inspection |
| Q | ⭐ `QUANTITY_BILLED` | Counter, updated by invoice matching |
| Q | `QUANTITY_CANCELLED` | Cancelled quantity |
| S | ⭐ `RECEIPT_REQUIRED_FLAG` | Y = receipt needed before paying (3-way) |
| S | ⭐ `INSPECTION_REQUIRED_FLAG` | Y = inspection needed (4-way) |
| S | `MATCH_OPTION` | Invoice matches to PO or to receipt |
| D | ⭐ `NEED_BY_DATE` | Date the business needs it |
| D | `PROMISED_DATE` | Date the supplier promised |

> **Formulas to remember:**
> Pending to receive = `QUANTITY − QUANTITY_RECEIVED − QUANTITY_CANCELLED`
> Received not billed = `QUANTITY_RECEIVED − QUANTITY_BILLED`
> Match: both flags N → **2-way**, receipt Y → **3-way**, receipt Y + inspection Y → **4-way**

---

### 5.6 `PO_DISTRIBUTIONS_ALL` ⭐⭐
**Why it matters:** **Who pays**: the GL charge account, destination (stock or expense), project. The lowest PO level, and **the level AP invoices and receiving deliveries link to**.
**10-second answer:** *"One row per accounting split of a schedule. Key PO_DISTRIBUTION_ID, carries all three parent IDs. CODE_COMBINATION_ID is the charge account, DESTINATION_TYPE_CODE says inventory or expense. Counters QUANTITY_ORDERED, QUANTITY_DELIVERED, QUANTITY_BILLED. AP invoice distributions match to it on PO_DISTRIBUTION_ID."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `PO_DISTRIBUTION_ID` | Internal distribution key |
| I | `DISTRIBUTION_NUM` | Distribution number shown |
| L | ⭐ `LINE_LOCATION_ID`, `PO_LINE_ID`, `PO_HEADER_ID` | All parents |
| L | `REQ_DISTRIBUTION_ID` | Link back to the requisition distribution |
| P | ⭐ `DESTINATION_TYPE_CODE` | INVENTORY (goes to stock) / EXPENSE (consumed directly) |
| P | `DELIVER_TO_LOCATION_ID`, `DELIVER_TO_PERSON_ID` | Final drop-off place / requester |
| P | `PROJECT_ID` | Project being charged (if any) |
| Q | ⭐ `QUANTITY_ORDERED` | Quantity on this split |
| Q | ⭐ `QUANTITY_DELIVERED` | Counter, updated by DELIVER |
| Q | ⭐ `QUANTITY_BILLED` | Counter, updated by invoice matching |
| Q | `QUANTITY_CANCELLED` | Cancelled |
| A | ⭐ `CODE_COMBINATION_ID` | GL charge account (cost center lives inside it) |

> If anyone says **"by cost center / department / account"**, the answer starts at this table.

---

### 5.7 `RCV_SHIPMENT_HEADERS`
**Why it matters:** The **receipt document**: one arrival of goods from a supplier, with the receipt number users search by.
**10-second answer:** *"One row per receipt. Key SHIPMENT_HEADER_ID, RECEIPT_NUM is the receipt number. Holds supplier and site, the receiving org SHIP_TO_ORG_ID, and RECEIPT_SOURCE_CODE (VENDOR for PO receipts). No PO ID, because a receipt can cover many POs."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `SHIPMENT_HEADER_ID` | Internal receipt key |
| I | ⭐ `RECEIPT_NUM` | Receipt number shown |
| I | `SHIPMENT_NUM` | Supplier's shipment/ASN number (if sent) |
| P | ⭐ `VENDOR_ID`, `VENDOR_SITE_ID` | Supplier |
| P | ⭐ `SHIP_TO_ORG_ID` | Receiving inventory org |
| S | ⭐ `RECEIPT_SOURCE_CODE` | VENDOR = from supplier (others = internal transfers) |
| D | `CREATION_DATE`, `SHIPPED_DATE`, `EXPECTED_RECEIPT_DATE` | Receipt / shipped / expected dates |
| — | ❌ **no PO_HEADER_ID** | Classic interview question |

---

### 5.8 `RCV_SHIPMENT_LINES`
**Why it matters:** **What item** arrived on the receipt and **which PO schedule** it belongs to.
**10-second answer:** *"One row per item line on a receipt. Key SHIPMENT_LINE_ID, parent SHIPMENT_HEADER_ID. It holds ITEM_ID, QUANTITY_SHIPPED and QUANTITY_RECEIVED, the PO links (PO_HEADER_ID, PO_LINE_ID, PO_LINE_LOCATION_ID), and SHIPMENT_LINE_STATUS_CODE."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `SHIPMENT_LINE_ID` | Internal key |
| I | `LINE_NUM` | Receipt line number |
| L | ⭐ `SHIPMENT_HEADER_ID` | Parent receipt |
| L | ⭐ `PO_HEADER_ID`, `PO_LINE_ID`, ⭐ `PO_LINE_LOCATION_ID` | Link to the PO |
| P | ⭐ `ITEM_ID`, `TO_ORGANIZATION_ID` | Item, receiving org |
| Q | `QUANTITY_SHIPPED`, ⭐ `QUANTITY_RECEIVED`, `UOM_CODE` | Quantities |
| S | ⭐ `SHIPMENT_LINE_STATUS_CODE` | EXPECTED / PARTIALLY RECEIVED / FULLY RECEIVED |
| S | `SOURCE_DOCUMENT_CODE` | PO (source document type) |

---

### 5.9 `RCV_TRANSACTIONS` ⭐⭐
**Why it matters:** The **history of every action** on received goods. It's the only place you can see *when* goods were received, accepted, rejected, delivered, returned or corrected. AP links here for 3-way match, and inventory links here for stock.
**10-second answer:** *"One row per receiving action. Key TRANSACTION_ID. TRANSACTION_TYPE is RECEIVE, ACCEPT, REJECT, DELIVER, RETURN TO VENDOR or CORRECT. PARENT_TRANSACTION_ID chains the actions. It holds QUANTITY, TRANSACTION_DATE, DESTINATION_TYPE_CODE, all PO links down to PO_DISTRIBUTION_ID, and INV_TRANSACTION_ID for the stock movement."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `TRANSACTION_ID` | Internal key (AP stores it as `RCV_TRANSACTION_ID`) |
| L | ⭐ `PARENT_TRANSACTION_ID` | The previous action in the chain (-1 for the first RECEIVE) |
| L | `SHIPMENT_HEADER_ID`, `SHIPMENT_LINE_ID` | Receipt header / line |
| L | ⭐ `PO_HEADER_ID`, `PO_LINE_ID`, `PO_LINE_LOCATION_ID`, ⭐ `PO_DISTRIBUTION_ID` | PO links (distribution mainly on DELIVER rows) |
| L | ⭐ `INV_TRANSACTION_ID` | Link to the stock movement in `INV_MATERIAL_TXNS` |
| P | `VENDOR_ID`, `ORGANIZATION_ID`, `SUBINVENTORY`, `LOCATOR_ID` | Supplier, warehouse, storage area, rack |
| Q | ⭐ `QUANTITY`, `UOM_CODE` | Quantity for this action |
| S | ⭐ `TRANSACTION_TYPE` | RECEIVE / ACCEPT / REJECT / TRANSFER / DELIVER / RETURN TO VENDOR / RETURN TO RECEIVING / CORRECT |
| S | ⭐ `DESTINATION_TYPE_CODE` | RECEIVING (still at dock) / INVENTORY / EXPENSE |
| D | ⭐ `TRANSACTION_DATE` | When the action happened |

> **Always filter `TRANSACTION_TYPE`** before summing. One receipt line has several rows.

---

### 5.10 `EGP_SYSTEM_ITEMS_B` / `_TL` / `_VL`
**Why it matters:** The **item master**: what the item is, its unit, and what it's allowed to do (be purchased, stocked, sold). Defined **per inventory org**.
**10-second answer:** *"Item master, one row per item per inventory org, so the key is INVENTORY_ITEM_ID plus ORGANIZATION_ID. ITEM_NUMBER is the item code, PRIMARY_UOM_CODE the unit, INVENTORY_ITEM_STATUS_CODE the status, and flags like PURCHASING_ENABLED_FLAG and STOCK_ENABLED_FLAG control behaviour. Description comes from _TL; reports use _VL."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `INVENTORY_ITEM_ID` + ⭐ `ORGANIZATION_ID` | **Composite key** (always join on both) |
| I | ⭐ `ITEM_NUMBER` | Item code shown to users |
| I | ⭐ `DESCRIPTION` (in `_TL` / `_VL`) | Item description (translated) |
| P | `ITEM_TYPE` | Item type (finished good, purchased item…) |
| Q | ⭐ `PRIMARY_UOM_CODE` | Base unit of measure |
| S | ⭐ `INVENTORY_ITEM_STATUS_CODE` | Active / Inactive / etc. |
| S | `PURCHASING_ENABLED_FLAG` | Can be put on a PO |
| S | `STOCK_ENABLED_FLAG`, `INVENTORY_ITEM_FLAG` | Can be stocked / tracked in inventory |
| S | `CUSTOMER_ORDER_ENABLED_FLAG` | Can be sold (O2C side) |
| — | `LANGUAGE`, `SOURCE_LANG` (in `_TL`) | Which language the description is in |

> `_B` = base columns (IDs, flags). `_TL` = translated text per language. `_VL` = `_B` + `_TL` for the session language. Use `_VL` in reports.

---

### 5.11 `INV_MATERIAL_TXNS`
**Why it matters:** The **history of every stock movement**: receipts into stock, issues, transfers, sales shipments. It answers "how and when did stock change?"
**10-second answer:** *"One row per inventory movement. Key TRANSACTION_ID. Holds item, org, subinventory and locator, TRANSACTION_QUANTITY (positive in, negative out), TRANSACTION_DATE, and the type and source columns that say what caused it, e.g. a PO receipt. Receiving links to it via RCV_TRANSACTIONS.INV_TRANSACTION_ID."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `TRANSACTION_ID` | Internal key |
| P | ⭐ `INVENTORY_ITEM_ID`, ⭐ `ORGANIZATION_ID` | Item + org |
| P | ⭐ `SUBINVENTORY_CODE`, `LOCATOR_ID` | Storage area / rack-bin |
| P | `TRANSFER_ORGANIZATION_ID`, `TRANSFER_SUBINVENTORY` | Destination for transfers |
| Q | ⭐ `TRANSACTION_QUANTITY` | **+ in / − out**, in `TRANSACTION_UOM` |
| Q | `PRIMARY_QUANTITY` | Same quantity in the item's primary UOM |
| S | ⭐ `TRANSACTION_TYPE_ID` | Specific type (PO receipt, sales issue, subinventory transfer…) |
| S | `TRANSACTION_ACTION_ID` | Physical action (receipt, issue, transfer) |
| S | ⭐ `TRANSACTION_SOURCE_TYPE_ID` | What caused it (purchase order, sales order, account…) |
| L | `TRANSACTION_SOURCE_ID` | ID of that source document |
| D | ⭐ `TRANSACTION_DATE` | When stock moved |

---

### 5.12 `INV_ONHAND_QUANTITIES_DETAIL`
**Why it matters:** **Current stock balance**, split by where it sits (subinventory, locator, lot).
**10-second answer:** *"Current on-hand stock, stored in fragments, one row per item, org, subinventory, locator and lot (and per receipt layer). PRIMARY_TRANSACTION_QUANTITY is the quantity, so you always SUM and GROUP BY. DATE_RECEIVED helps with stock aging, and CREATE_TRANSACTION_ID points to the movement that created the row."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | `ONHAND_QUANTITIES_ID` | Internal key of the fragment |
| P | ⭐ `INVENTORY_ITEM_ID`, ⭐ `ORGANIZATION_ID` | Item + org |
| P | ⭐ `SUBINVENTORY_CODE`, `LOCATOR_ID`, `LOT_NUMBER` | Where exactly it sits |
| Q | ⭐ `PRIMARY_TRANSACTION_QUANTITY` | Quantity in this fragment (**SUM it**) |
| Q | `TRANSACTION_UOM_CODE` | Unit |
| L | `CREATE_TRANSACTION_ID`, `UPDATE_TRANSACTION_ID` | Material transaction that created / last updated it |
| D | `DATE_RECEIVED` | When this stock arrived (for aging / FIFO) |

---

### 5.13 `AP_INVOICES_ALL` ⭐
**Why it matters:** The **supplier's bill**: what we owe, to whom, in which currency, and how much has been paid. Every AP report starts here.
**10-second answer:** *"One row per supplier invoice. Key INVOICE_ID, INVOICE_NUM is the supplier's bill number. Holds supplier and site, INVOICE_TYPE_LOOKUP_CODE, INVOICE_AMOUNT and AMOUNT_PAID, currency, INVOICE_DATE and GL_DATE, TERMS_ID, PAYMENT_STATUS_FLAG (Y/N/P), CANCELLED_DATE, SOURCE, and ORG_ID for the BU."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `INVOICE_ID` | Internal key |
| I | ⭐ `INVOICE_NUM` | Supplier's invoice number |
| P | ⭐ `VENDOR_ID`, `VENDOR_SITE_ID` | Supplier and pay site |
| P | ⭐ `ORG_ID` | Business Unit |
| L | `PO_HEADER_ID` | Filled when the whole invoice is for one PO (lines are the reliable link) |
| Q | ⭐ `INVOICE_AMOUNT` | Total bill amount |
| Q | ⭐ `AMOUNT_PAID` | Paid so far (updated by payments) |
| Q | `INVOICE_CURRENCY_CODE`, `PAYMENT_CURRENCY_CODE`, `EXCHANGE_RATE` | Currencies |
| Q | `CANCELLED_AMOUNT` | Amount at cancellation |
| S | ⭐ `INVOICE_TYPE_LOOKUP_CODE` | STANDARD / CREDIT / DEBIT / PREPAYMENT |
| S | ⭐ `PAYMENT_STATUS_FLAG` | N = unpaid, P = partially paid, Y = fully paid |
| S | ⭐ `SOURCE` | How it came in (Manual, spreadsheet/FBDI import, e-invoice…) |
| S | `WFAPPROVAL_STATUS` | Invoice approval workflow status |
| D | ⭐ `INVOICE_DATE` | Date on the bill |
| D | ⭐ `GL_DATE` | Accounting date (decides the period) |
| D | ⭐ `CANCELLED_DATE` | Filled = invoice cancelled |
| — | `TERMS_ID`, `TERMS_DATE`, `PAYMENT_METHOD_CODE` | Payment terms, terms start date, how to pay |

> `INVOICE_DATE` vs `GL_DATE`: the supplier's date vs the date the accounting lands on. Month-end reports use **GL_DATE**.

---

### 5.14 `AP_INVOICE_LINES_ALL`
**Why it matters:** **What's on the bill**: item, freight, tax and misc lines, and which **PO schedule/receipt** each item line is matched to.
**10-second answer:** *"One row per invoice line. Composite key INVOICE_ID plus LINE_NUMBER. LINE_TYPE_LOOKUP_CODE is ITEM, TAX, FREIGHT or MISCELLANEOUS. Holds AMOUNT, QUANTITY_INVOICED and UNIT_PRICE, the PO links, RCV_TRANSACTION_ID for receipt matching, and MATCH_TYPE."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `INVOICE_ID` + ⭐ `LINE_NUMBER` | **Composite key** (no single line ID) |
| L | ⭐ `PO_HEADER_ID`, `PO_LINE_ID`, ⭐ `PO_LINE_LOCATION_ID`, `PO_DISTRIBUTION_ID` | PO match |
| L | `RCV_TRANSACTION_ID` | Receipt match |
| P | `INVENTORY_ITEM_ID`, `DESCRIPTION` | Item / text |
| Q | ⭐ `AMOUNT` | Line amount |
| Q | ⭐ `QUANTITY_INVOICED`, `UNIT_PRICE` | Billed qty and price |
| S | ⭐ `LINE_TYPE_LOOKUP_CODE` | ITEM / TAX / FREIGHT / MISCELLANEOUS |
| S | ⭐ `MATCH_TYPE` | ITEM_TO_PO / ITEM_TO_RECEIPT / NOT_MATCHED |
| S | `DISCARDED_FLAG`, `CANCELLED_FLAG` | Line discarded / cancelled |
| D | `ACCOUNTING_DATE` | Line accounting date |

---

### 5.15 `AP_INVOICE_DISTRIBUTIONS_ALL` ⭐⭐
**Why it matters:** The **accounting of the invoice**: which account each rupee hits. It's **the link to PO distributions and receipts**, and it carries the **validation** and **accounted** flags.
**10-second answer:** *"One row per accounting split of an invoice line. Joins to lines on INVOICE_ID plus INVOICE_LINE_NUMBER. Holds AMOUNT, DIST_CODE_COMBINATION_ID, PO_DISTRIBUTION_ID and RCV_TRANSACTION_ID, MATCH_STATUS_FLAG for validation (A = validated), POSTED_FLAG for accounted, and REVERSAL_FLAG."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `INVOICE_DISTRIBUTION_ID` | Internal key |
| I | `DISTRIBUTION_LINE_NUMBER` | Number within the line |
| L | ⭐ `INVOICE_ID`, ⭐ `INVOICE_LINE_NUMBER` | Parent line (**name differs from LINE_NUMBER!**) |
| L | ⭐ `PO_DISTRIBUTION_ID` | Bridge to PO |
| L | ⭐ `RCV_TRANSACTION_ID` | Bridge to receipt (= `RCV_TRANSACTIONS.TRANSACTION_ID`) |
| L | `PARENT_REVERSAL_ID` | Points to the original row it reverses |
| Q | ⭐ `AMOUNT`, `QUANTITY_INVOICED` | Amount / qty on this split |
| S | ⭐ `MATCH_STATUS_FLAG` | null/N = never validated, **A = validated**, T = tested with holds |
| S | ⭐ `POSTED_FLAG` | Y = accounting created |
| S | ⭐ `REVERSAL_FLAG` | Y = reversed row (exclude from totals) |
| S | `LINE_TYPE_LOOKUP_CODE` | ITEM / TAX / FREIGHT… |
| D | `ACCOUNTING_DATE`, `PERIOD_NAME` | Accounting date / GL period |
| A | ⭐ `DIST_CODE_COMBINATION_ID` | GL account charged |
| A | `ACCOUNTING_EVENT_ID` | Link to Subledger Accounting event |

---

### 5.16 `AP_PAYMENT_SCHEDULES_ALL`
**Why it matters:** **When** each part of an invoice is due and **how much is still left**. Aging, cash forecasting and the payment run all read this table.
**10-second answer:** *"One row per installment of an invoice, composite key INVOICE_ID plus PAYMENT_NUM, created from the payment terms. Holds DUE_DATE, GROSS_AMOUNT, AMOUNT_REMAINING, discount date and amount, PAYMENT_STATUS_FLAG and HOLD_FLAG."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `INVOICE_ID` + ⭐ `PAYMENT_NUM` | Composite key (installment number) |
| Q | ⭐ `GROSS_AMOUNT` | Installment amount |
| Q | ⭐ `AMOUNT_REMAINING` | Still unpaid (updated by payments) |
| Q | `DISCOUNT_AMOUNT_AVAILABLE` | Early-payment discount available |
| S | ⭐ `PAYMENT_STATUS_FLAG` | N / P / Y for this installment |
| S | ⭐ `HOLD_FLAG` | Y = this installment is held from payment |
| S | `PAYMENT_METHOD_CODE` | How it'll be paid |
| D | ⭐ `DUE_DATE` | When it's due |
| D | `DISCOUNT_DATE` | Pay by this date to get the discount |

---

### 5.17 `AP_CHECKS_ALL`
**Why it matters:** The **payment document itself** (cheque, EFT or wire), with its bank status. Used for payment registers, voids and bank reconciliation.
**10-second answer:** *"One row per payment, whatever the method. Key CHECK_ID, CHECK_NUMBER is the payment number. Holds AMOUNT, CHECK_DATE, currency, supplier and site, PAYMENT_METHOD_CODE, PAYMENT_TYPE_FLAG, STATUS_LOOKUP_CODE (NEGOTIABLE, CLEARED, RECONCILED, VOIDED), and VOID_DATE / CLEARED_DATE."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `CHECK_ID` | Internal key |
| I | ⭐ `CHECK_NUMBER` | Payment number shown |
| P | ⭐ `VENDOR_ID`, `VENDOR_SITE_ID`, `VENDOR_NAME` | Payee |
| P | `BANK_ACCOUNT_NAME`, `ORG_ID` | Our bank account it was paid from, BU |
| Q | ⭐ `AMOUNT`, `CURRENCY_CODE` | Total payment amount |
| Q | `CLEARED_AMOUNT` | Amount cleared by the bank |
| S | ⭐ `PAYMENT_METHOD_CODE` | CHECK / EFT / WIRE |
| S | `PAYMENT_TYPE_FLAG` | How it was created: A = payment process request (batch run), M = manual, Q = quick, R = refund |
| S | ⭐ `STATUS_LOOKUP_CODE` | NEGOTIABLE / CLEARED / RECONCILED / VOIDED |
| D | ⭐ `CHECK_DATE` | Payment date |
| D | `CLEARED_DATE`, ⭐ `VOID_DATE` | Bank clearing / void date |

> `AMOUNT` here is the **whole payment**. For the amount applied to **one invoice**, use `AP_INVOICE_PAYMENTS_ALL.AMOUNT`.

---

### 5.18 `AP_INVOICE_PAYMENTS_ALL`
**Why it matters:** The **link between payments and invoice installments** (many-to-many). The only table that tells you *which invoice a payment paid, and how much of it*.
**10-second answer:** *"Bridge between AP_CHECKS_ALL and AP_PAYMENT_SCHEDULES_ALL. One row per payment applied to one invoice installment. Holds CHECK_ID, INVOICE_ID and PAYMENT_NUM, the AMOUNT applied, DISCOUNT_TAKEN, ACCOUNTING_DATE, and REVERSAL_FLAG for voids."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `INVOICE_PAYMENT_ID` | Internal key |
| L | ⭐ `CHECK_ID` | Which payment |
| L | ⭐ `INVOICE_ID` + ⭐ `PAYMENT_NUM` | Which invoice installment |
| L | `REVERSAL_INV_PMT_ID` | The row this one reverses (void) |
| Q | ⭐ `AMOUNT` | Amount applied to this invoice |
| Q | `DISCOUNT_TAKEN` | Discount taken on this payment |
| S | ⭐ `REVERSAL_FLAG` | Y = reversal row (void) |
| S | `POSTED_FLAG` | Payment accounting created |
| D | `ACCOUNTING_DATE`, `PERIOD_NAME` | Accounting date / period |

---

### 5.19 `GL_JE_BATCHES` (sheet says GL_BATCHES)
**Why it matters:** A **group of journals** posted together. Posting status is tracked here.
**10-second answer:** *"One row per journal batch. Key JE_BATCH_ID. Holds NAME, STATUS (U unposted, P posted), DEFAULT_PERIOD_NAME, POSTED_DATE, and running debit/credit totals."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `JE_BATCH_ID`, ⭐ `NAME` | Key / batch name |
| Q | `RUNNING_TOTAL_DR`, `RUNNING_TOTAL_CR` | Batch totals |
| S | ⭐ `STATUS` | U = unposted, P = posted |
| S | `ACTUAL_FLAG` | A = actual, B = budget, E = encumbrance |
| D | ⭐ `DEFAULT_PERIOD_NAME`, `POSTED_DATE` | Period / posting date |

### 5.20 `GL_JE_HEADERS`
**Why it matters:** One **journal**, which tells you **where it came from** (Payables) and **what kind** it is (invoices vs payments), in which ledger and period.
**10-second answer:** *"One row per journal, child of a batch. Key JE_HEADER_ID. JE_SOURCE (Payables) and JE_CATEGORY (Purchase Invoices, Payments) identify P2P journals. Also LEDGER_ID, PERIOD_NAME, CURRENCY_CODE, STATUS and ACTUAL_FLAG."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `JE_HEADER_ID`, `NAME` | Key / journal name |
| L | ⭐ `JE_BATCH_ID` | Parent batch |
| P | ⭐ `LEDGER_ID` | Which ledger (set of books) |
| Q | `CURRENCY_CODE`, `RUNNING_TOTAL_DR/CR` | Currency / totals |
| S | ⭐ `JE_SOURCE` | Payables / Receivables / Cost Management / Manual… |
| S | ⭐ `JE_CATEGORY` | Purchase Invoices / Payments / … |
| S | `STATUS`, `ACTUAL_FLAG` | Posted? Actual vs budget |
| D | ⭐ `PERIOD_NAME`, `DEFAULT_EFFECTIVE_DATE`, `POSTED_DATE` | Period / accounting date / posted date |

### 5.21 `GL_JE_LINES`
**Why it matters:** The actual **debits and credits** against accounts. The lowest level of the ledger.
**10-second answer:** *"One row per journal line. Key JE_HEADER_ID plus JE_LINE_NUM. Holds CODE_COMBINATION_ID, ENTERED_DR/CR in transaction currency and ACCOUNTED_DR/CR in ledger currency, DESCRIPTION, and EFFECTIVE_DATE."*

| Slot | Column | Meaning |
|------|--------|---------|
| I | ⭐ `JE_HEADER_ID` + ⭐ `JE_LINE_NUM` | Composite key |
| P | `LEDGER_ID` | Ledger |
| Q | ⭐ `ENTERED_DR`, `ENTERED_CR` | Amounts in the **transaction** currency |
| Q | ⭐ `ACCOUNTED_DR`, `ACCOUNTED_CR` | Amounts in the **ledger (functional)** currency |
| D | `EFFECTIVE_DATE`, `PERIOD_NAME` | Accounting date / period |
| A | ⭐ `CODE_COMBINATION_ID` | GL account |
| — | `DESCRIPTION` | Line description |

> **Entered vs accounted:** a USD invoice in an INR ledger has ENTERED in USD and ACCOUNTED in INR. Financial reports use **ACCOUNTED**.

---

## 6. Lookalike columns that trap people

| Looks similar | The difference |
|---------------|----------------|
| `SEGMENT1` | PO number (PO_HEADERS), supplier number (POZ_SUPPLIERS), an account segment (GL_CODE_COMBINATIONS) |
| `ORG_ID` vs `ORGANIZATION_ID` | Business Unit vs inventory org (warehouse) |
| `PRC_BU_ID` vs `ORG_ID` | Procurement BU on POs / sites vs BU on AP documents |
| `LINE_NUM` vs `LINE_NUMBER` | PO_LINES uses `LINE_NUM`; AP_INVOICE_LINES uses `LINE_NUMBER` |
| `LINE_NUMBER` vs `INVOICE_LINE_NUMBER` | Same value: on AP lines it's `LINE_NUMBER`, on AP distributions it's `INVOICE_LINE_NUMBER` |
| `LINE_LOCATION_ID` vs `PO_LINE_LOCATION_ID` | Same value: in PO_LINE_LOCATIONS it's the key; in RCV/AP tables it carries the `PO_` prefix |
| `TRANSACTION_ID` vs `RCV_TRANSACTION_ID` | Same value: key of RCV_TRANSACTIONS vs how AP stores it |
| `TRANSACTION_ID` vs `INV_TRANSACTION_ID` | Key of INV_MATERIAL_TXNS vs how RCV_TRANSACTIONS stores it |
| `QUANTITY_RECEIVED` vs `QUANTITY_DELIVERED` | Received at dock (schedule) vs delivered to destination (distribution) |
| `QUANTITY` (PO_LINES) vs `QUANTITY` (PLL) vs `QUANTITY_ORDERED` (PDA) | Line total vs schedule share vs distribution share |
| `INVOICE_DATE` vs `GL_DATE` | Supplier's bill date vs accounting date |
| `INVOICE_AMOUNT` vs `AMOUNT_PAID` vs `AMOUNT_REMAINING` | Bill total vs paid so far (invoice) vs left per installment (schedule) |
| `AP_CHECKS_ALL.AMOUNT` vs `AP_INVOICE_PAYMENTS_ALL.AMOUNT` | Whole payment vs part applied to one invoice |
| `PAYMENT_STATUS_FLAG` (invoice) vs (schedule) | Whole invoice vs one installment |
| `MATCH_STATUS_FLAG` vs `POSTED_FLAG` | Validated? vs accounted? |
| `ENTERED_DR` vs `ACCOUNTED_DR` | Transaction currency vs ledger currency |
| `CODE_COMBINATION_ID` vs `DIST_CODE_COMBINATION_ID` | PO charge account vs invoice charge account (both are GL accounts) |

---

## 7. Rapid-fire drill

Cover the right column. Answer out loud.

| # | Question | Answer |
|---|----------|--------|
| 1 | Column that tells if a PO is approved? | <details><summary>show</summary>`PO_HEADERS_ALL.DOCUMENT_STATUS = 'OPEN'`</details> |
| 2 | Where is the unit price? | <details><summary>show</summary>`PO_LINES_ALL.UNIT_PRICE`</details> |
| 3 | Which column tells 3-way vs 2-way? | <details><summary>show</summary>`PO_LINE_LOCATIONS_ALL.RECEIPT_REQUIRED_FLAG` (+ `INSPECTION_REQUIRED_FLAG` for 4-way)</details> |
| 4 | Which event increases `QUANTITY_DELIVERED`, and on which table? | <details><summary>show</summary>DELIVER, on `PO_DISTRIBUTIONS_ALL`</details> |
| 5 | Where is the PO's charge account? | <details><summary>show</summary>`PO_DISTRIBUTIONS_ALL.CODE_COMBINATION_ID`</details> |
| 6 | What does `PARENT_TRANSACTION_ID = -1` mean? | <details><summary>show</summary>The first action (RECEIVE) in the receiving chain</details> |
| 7 | Which RCV column links to the stock movement? | <details><summary>show</summary>`RCV_TRANSACTIONS.INV_TRANSACTION_ID`</details> |
| 8 | Key of the item master? | <details><summary>show</summary>`INVENTORY_ITEM_ID` + `ORGANIZATION_ID`</details> |
| 9 | Sign of `TRANSACTION_QUANTITY` for a PO delivery? | <details><summary>show</summary>Positive (stock in)</details> |
| 10 | Column for current stock, and what you must do with it | <details><summary>show</summary>`INV_ONHAND_QUANTITIES_DETAIL.PRIMARY_TRANSACTION_QUANTITY`, SUM it</details> |
| 11 | How do you know an invoice is cancelled? | <details><summary>show</summary>`AP_INVOICES_ALL.CANCELLED_DATE IS NOT NULL`</details> |
| 12 | Which date decides the accounting period of an invoice? | <details><summary>show</summary>`AP_INVOICES_ALL.GL_DATE`</details> |
| 13 | Column that shows an invoice came via spreadsheet/FBDI import? | <details><summary>show</summary>`AP_INVOICES_ALL.SOURCE`</details> |
| 14 | `PAYMENT_STATUS_FLAG = 'P'` means? | <details><summary>show</summary>Partially paid</details> |
| 15 | Composite key of AP invoice lines? | <details><summary>show</summary>`INVOICE_ID` + `LINE_NUMBER`</details> |
| 16 | Validated invoice distribution has which flag value? | <details><summary>show</summary>`MATCH_STATUS_FLAG = 'A'`</details> |
| 17 | Where do you find how much is still due and when? | <details><summary>show</summary>`AP_PAYMENT_SCHEDULES_ALL.AMOUNT_REMAINING`, `DUE_DATE`</details> |
| 18 | How do you identify a voided payment? | <details><summary>show</summary>`AP_CHECKS_ALL.STATUS_LOOKUP_CODE = 'VOIDED'` / `VOID_DATE`</details> |
| 19 | Which table says "this payment paid this invoice installment"? | <details><summary>show</summary>`AP_INVOICE_PAYMENTS_ALL` (CHECK_ID, INVOICE_ID, PAYMENT_NUM)</details> |
| 20 | How do you find P2P journals in GL? | <details><summary>show</summary>`GL_JE_HEADERS.JE_SOURCE = 'Payables'` (+ JE_CATEGORY)</details> |

---

## 8. One-page revision sheet

```
TEMPLATE:  I-L-P-Q-S-D-A  "I Love Paneer, Quite Spicy, Daily, Always"
           Identity · Links · Parties/Places · Quantities · Status · Dates · Accounting
RULES:     Children POINT up (…_ID)     Parents COUNT down (QUANTITY_*, AMOUNT_*)

POZ_SUPPLIERS_V        VENDOR_ID, SEGMENT1, VENDOR_NAME, PARTY_ID, ENABLED_FLAG, END_DATE_ACTIVE
POZ_SUPPLIER_SITES_V   VENDOR_SITE_ID, VENDOR_SITE_CODE, VENDOR_ID, PRC_BU_ID, PURCHASING_SITE_FLAG, PAY_SITE_FLAG

PO_HEADERS_ALL         PO_HEADER_ID, SEGMENT1, VENDOR_ID, VENDOR_SITE_ID, AGENT_ID, PRC_BU_ID,
                       TYPE_LOOKUP_CODE, DOCUMENT_STATUS, CURRENCY_CODE
PO_LINES_ALL           PO_LINE_ID, PO_HEADER_ID, LINE_NUM, ITEM_ID, ITEM_DESCRIPTION, QUANTITY, UNIT_PRICE, UOM_CODE
PO_LINE_LOCATIONS_ALL  LINE_LOCATION_ID, SHIP_TO_ORGANIZATION_ID, NEED_BY_DATE, QUANTITY,
                       QUANTITY_RECEIVED/ACCEPTED/BILLED/CANCELLED, RECEIPT_REQUIRED_FLAG, INSPECTION_REQUIRED_FLAG
PO_DISTRIBUTIONS_ALL   PO_DISTRIBUTION_ID, CODE_COMBINATION_ID, DESTINATION_TYPE_CODE,
                       QUANTITY_ORDERED/DELIVERED/BILLED

RCV_SHIPMENT_HEADERS   SHIPMENT_HEADER_ID, RECEIPT_NUM, VENDOR_ID, SHIP_TO_ORG_ID, RECEIPT_SOURCE_CODE  (no PO id!)
RCV_SHIPMENT_LINES     SHIPMENT_LINE_ID, ITEM_ID, QUANTITY_RECEIVED, PO_*_IDs, SHIPMENT_LINE_STATUS_CODE
RCV_TRANSACTIONS       TRANSACTION_ID, PARENT_TRANSACTION_ID, TRANSACTION_TYPE, QUANTITY, TRANSACTION_DATE,
                       DESTINATION_TYPE_CODE, PO_DISTRIBUTION_ID, INV_TRANSACTION_ID

EGP_SYSTEM_ITEMS_VL    INVENTORY_ITEM_ID + ORGANIZATION_ID, ITEM_NUMBER, DESCRIPTION, PRIMARY_UOM_CODE,
                       INVENTORY_ITEM_STATUS_CODE, PURCHASING_ENABLED_FLAG, STOCK_ENABLED_FLAG
INV_MATERIAL_TXNS      TRANSACTION_ID, ITEM+ORG, SUBINVENTORY_CODE, LOCATOR_ID, TRANSACTION_QUANTITY(±),
                       TRANSACTION_TYPE_ID, TRANSACTION_SOURCE_TYPE_ID, TRANSACTION_DATE
INV_ONHAND_QTY_DETAIL  ITEM+ORG, SUBINVENTORY_CODE, LOCATOR_ID, LOT_NUMBER, PRIMARY_TRANSACTION_QUANTITY (SUM!), DATE_RECEIVED

AP_INVOICES_ALL        INVOICE_ID, INVOICE_NUM, VENDOR_ID, ORG_ID, INVOICE_TYPE_LOOKUP_CODE, INVOICE_AMOUNT,
                       AMOUNT_PAID, PAYMENT_STATUS_FLAG(N/P/Y), INVOICE_DATE, GL_DATE, CANCELLED_DATE, SOURCE
AP_INVOICE_LINES_ALL   INVOICE_ID+LINE_NUMBER, LINE_TYPE_LOOKUP_CODE, AMOUNT, QUANTITY_INVOICED, PO_*_IDs, MATCH_TYPE
AP_INVOICE_DISTS_ALL   INVOICE_DISTRIBUTION_ID, INVOICE_LINE_NUMBER, AMOUNT, DIST_CODE_COMBINATION_ID,
                       PO_DISTRIBUTION_ID, RCV_TRANSACTION_ID, MATCH_STATUS_FLAG(A), POSTED_FLAG, REVERSAL_FLAG
AP_PAYMENT_SCHEDULES   INVOICE_ID+PAYMENT_NUM, DUE_DATE, GROSS_AMOUNT, AMOUNT_REMAINING, PAYMENT_STATUS_FLAG, HOLD_FLAG
AP_CHECKS_ALL          CHECK_ID, CHECK_NUMBER, CHECK_DATE, AMOUNT, VENDOR_ID, PAYMENT_METHOD_CODE, STATUS_LOOKUP_CODE, VOID_DATE
AP_INVOICE_PAYMENTS    CHECK_ID, INVOICE_ID+PAYMENT_NUM, AMOUNT, ACCOUNTING_DATE, REVERSAL_FLAG

GL_JE_BATCHES          JE_BATCH_ID, NAME, STATUS(U/P), DEFAULT_PERIOD_NAME
GL_JE_HEADERS          JE_HEADER_ID, JE_BATCH_ID, LEDGER_ID, PERIOD_NAME, JE_SOURCE, JE_CATEGORY
GL_JE_LINES            JE_HEADER_ID+JE_LINE_NUM, CODE_COMBINATION_ID, ENTERED_DR/CR, ACCOUNTED_DR/CR

EVENTS → COLUMNS
  Approve → DOCUMENT_STATUS=OPEN        Receive → PLL.QUANTITY_RECEIVED      Accept → PLL.QUANTITY_ACCEPTED
  Deliver → PDA.QUANTITY_DELIVERED, INV_MATERIAL_TXNS +qty, ONHAND up, RT.INV_TRANSACTION_ID
  Invoice match → PLL/PDA.QUANTITY_BILLED, payment schedules created
  Validate → MATCH_STATUS_FLAG=A        Account → POSTED_FLAG=Y
  Pay → AMOUNT_REMAINING↓, AMOUNT_PAID↑, PAYMENT_STATUS_FLAG N→P→Y    Bank → STATUS NEGOTIABLE→CLEARED
  To GL → batch STATUS U→P
```