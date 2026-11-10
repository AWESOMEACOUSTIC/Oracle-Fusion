# Oracle Fusion ERP — Procure to Pay (P2P) Study Guide
### Learn the process, remember the tables, write the queries

> **How to use this guide:** Read Part 1–3 once to get the story. Study the join map in Part 5, then do the use cases in Part 6 one by one, writing each query yourself before looking at the answer. Finish with the self-test in Part 8. Revise from the cheat sheet in Part 9 before interviews.

> **Scope:** Only the P2P tables in your sheet: **PO, Receiving, Inventory, AP, GL, and Supplier (POZ)**. The O2C tables (OM, Shipping, AR, HZ) are left out on purpose.

---

## Table of Contents
1. [P2P in one picture](#part-1--p2p-in-one-picture)
2. [The running story: ABC Corp buys laptops](#part-2--the-running-story-abc-corp-buys-laptops)
3. [Memory tricks for table names](#part-3--memory-tricks-for-table-names)
4. [Every table explained (with sample rows)](#part-4--every-table-explained)
5. [The full join map](#part-5--the-full-join-map)
6. [Use cases with SQL (BI Publisher style)](#part-6--use-cases-with-sql)
7. [Traps that interviewers love](#part-7--traps-interviewers-love)
8. [Self-test (with answers)](#part-8--self-test)
9. [One-page cheat sheet](#part-9--one-page-cheat-sheet)

---

## Part 1 — P2P in one picture

P2P = everything from **"we need something"** to **"we paid for it and the books are updated."**

```
 ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌───────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
 │ SUPPLIER │──▶│    PO    │──▶│ RECEIPT  │──▶│ INVENTORY │   │ INVOICE  │──▶│ PAYMENT  │──▶│    GL    │
 │  (POZ)   │   │   (PO)   │   │  (RCV)   │   │   (INV)   │   │   (AP)   │   │   (AP)   │   │   (GL)   │
 └──────────┘   └──────────┘   └────┬─────┘   └───────────┘   └────▲─────┘   └──────────┘   └──────────┘
   "Who do we     "What we       "What we    "Stock goes up"    "Supplier       "We pay"     "Accountant
    buy from?"     ordered"       got"                           bills us"                    records it"
                                    └──────────── matched against ──┘
```

### The 7 process steps

| # | Step | What happens in real life | Module | Main tables |
|---|------|---------------------------|--------|-------------|
| 1 | **Supplier setup** | Dell is registered as a supplier with an address (site) | Supplier | `POZ_SUPPLIERS_V`, `POZ_SUPPLIER_SITES_V` |
| 2 | **Requisition** *(not in your sheet)* | An employee asks "I need 10 laptops" | Self Service Procurement | (POR_ tables, outside this guide) |
| 3 | **Purchase Order** | Buyer creates and approves a PO to Dell | Purchasing | `PO_HEADERS_ALL`, `PO_LINES_ALL`, `PO_LINE_LOCATIONS_ALL`, `PO_DISTRIBUTIONS_ALL` |
| 4 | **Receipt** | Laptops arrive; the warehouse receives, inspects and delivers them | Receiving | `RCV_SHIPMENT_HEADERS`, `RCV_SHIPMENT_LINES`, `RCV_TRANSACTIONS` |
| 5 | **Inventory update** | Stock goes up in the warehouse | Inventory | `EGP_SYSTEM_ITEMS_B/_TL/_VL`, `INV_MATERIAL_TXNS`, `INV_ONHAND_QUANTITIES_DETAIL` |
| 6 | **Invoice** | Dell sends a bill; AP matches it to the PO/receipt and validates it | Payables | `AP_INVOICES_ALL`, `AP_INVOICE_LINES_ALL`, `AP_INVOICE_DISTRIBUTIONS_ALL`, `AP_PAYMENT_SCHEDULES_ALL` |
| 7 | **Payment** | ABC Corp pays Dell | Payables | `AP_CHECKS_ALL`, `AP_INVOICE_PAYMENTS_ALL` |
| 8 | **Accounting → GL** | Journals are created and posted to the ledger | General Ledger | `GL_JE_BATCHES`*, `GL_JE_HEADERS`, `GL_JE_LINES` |

\* Your sheet says `GL_BATCHES`. The real Fusion table is **`GL_JE_BATCHES`**. Check this in your instance.

### Matching: 2-way, 3-way, 4-way
Before AP pays an invoice, it checks it against other documents. **The more documents compared, the safer the payment.**

| Match type | Compares | Memory hook |
|------------|----------|-------------|
| **2-way** | PO ↔ Invoice | "Did we order it, at this price?" |
| **3-way** | PO ↔ Receipt ↔ Invoice | "…and did we actually **get** it?" |
| **4-way** | PO ↔ Receipt ↔ Inspection ↔ Invoice | "…and was it **good quality**?" |

This is set on the PO **schedule** (`PO_LINE_LOCATIONS_ALL`) using `RECEIPT_REQUIRED_FLAG` and `INSPECTION_REQUIRED_FLAG`.

---

## Part 2 — The running story: ABC Corp buys laptops

We follow **one purchase** through every table. Each table in Part 4 shows its part of this story.

> **ABC Corp** orders **100 Dell laptops at ₹50,000 each** (₹50,00,000 total).
> - 60 laptops go to the **Bangalore warehouse** and 40 to the **Chennai warehouse**.
> - The Bangalore laptops are charged **50/50** to the **IT** and **HR** departments.
> - Dell ships the 60 Bangalore laptops first. The warehouse **receives, accepts and delivers** them.
> - Dell sends an invoice for those 60 laptops. ABC pays it in **2 installments**.
> - The accountant transfers everything to the **General Ledger**.

The key numbers to keep in your head:

| Thing | Value |
|-------|-------|
| Supplier | Dell India (VENDOR_ID = 9001), site "DELL-BLR" (VENDOR_SITE_ID = 9101) |
| PO number | PO-1001 (PO_HEADER_ID = 5001) |
| Item | LAPTOP-DELL-5440 (INVENTORY_ITEM_ID = 7001) |
| Receipt number | RCPT-3001 (SHIPMENT_HEADER_ID = 3001) |
| Invoice number | DELL-INV-778 (INVOICE_ID = 8001) |
| Payment | Check/EFT CHK-6001 and CHK-6002 |

---

## Part 3 — Memory tricks for table names

### 3.1 Decode the prefix (which module)

| Prefix | Read it as | Module |
|--------|------------|--------|
| `POZ_` | **PO + Z**uppliers (think "Zuppliers") | Supplier master |
| `PO_` | **P**urchase **O**rder | Purchasing |
| `RCV_` | **R**e**C**ei**V**ing | Receiving |
| `EGP_` | **E**nterprise **G**oods/**P**roducts (Product Hub item master) | Items |
| `INV_` | **INV**entory | Inventory |
| `AP_` | **A**ccounts **P**ayable (money **we owe**) | Payables |
| `GL_` | **G**eneral **L**edger | Accounting |

> AP vs AR: **P**ayable = we **P**ay. **R**eceivable = we **R**eceive. P2P only uses AP.

### 3.2 Decode the suffix (what kind of object)

| Suffix | Meaning | Example |
|--------|---------|---------|
| `_ALL` | Holds data for **all Business Units**. Filter by BU or you'll mix companies. | `PO_HEADERS_ALL` |
| `_B` | **B**ase table (IDs, codes, flags) | `EGP_SYSTEM_ITEMS_B` |
| `_TL` | **T**rans**L**ated text (description in each language) | `EGP_SYSTEM_ITEMS_TL` |
| `_VL` | **V**iew = `_B` + `_TL` in the user's **L**anguage | `EGP_SYSTEM_ITEMS_VL` |
| `_V` | A **V**iew (ready-made join) | `POZ_SUPPLIERS_V` |

### 3.3 The "same shape" trick — learn one, get three free

Most P2P documents use the same **Header → Lines → (more detail)** shape. Learn the shape once:

```
PURCHASE ORDER                 RECEIPT                       INVOICE                       JOURNAL
PO_HEADERS_ALL                 RCV_SHIPMENT_HEADERS          AP_INVOICES_ALL               GL_JE_BATCHES
   └─ PO_LINES_ALL                └─ RCV_SHIPMENT_LINES         └─ AP_INVOICE_LINES_ALL       └─ GL_JE_HEADERS
        └─ PO_LINE_LOCATIONS_ALL       └─ RCV_TRANSACTIONS            └─ AP_INVOICE_                  └─ GL_JE_LINES
             └─ PO_DISTRIBUTIONS_ALL                                       DISTRIBUTIONS_ALL
  (4 levels)                     (3 levels)                    (3 levels)                    (3 levels)
```

### 3.4 The PO's 4 levels — "**What, Which, When-Where, Who Pays**"

| Level | Table | Question it answers | Story |
|-------|-------|---------------------|-------|
| Header | `PO_HEADERS_ALL` | **What** deal? (supplier, buyer, currency, status) | PO-1001 to Dell |
| Line | `PO_LINES_ALL` | **Which** item, how many, at what price? | 100 laptops @ ₹50,000 |
| Line Location (= **Schedule**) | `PO_LINE_LOCATIONS_ALL` | **When and where** to deliver? | 60 → Bangalore, 40 → Chennai |
| Distribution | `PO_DISTRIBUTIONS_ALL` | **Who pays** (which account/department)? | Bangalore 60 → 30 IT + 30 HR |

> Mnemonic: **"H-L-S-D → Hungry Lions Steal Donuts"** (Header, Line, Schedule, Distribution).
> Remember: the table is named **LINE_LOCATIONS**, but in the UI it's called the **Schedule**. Interviewers like to ask this.

### 3.5 Receiving's 3 levels — "**Truck, Boxes, Actions**"

| Level | Table | Think of it as |
|-------|-------|----------------|
| `RCV_SHIPMENT_HEADERS` | The **truck** that arrived (receipt number, supplier, date) |
| `RCV_SHIPMENT_LINES` | The **boxes** in the truck (which item, which PO line, quantity) |
| `RCV_TRANSACTIONS` | Every **action** done to the boxes: RECEIVE → ACCEPT/REJECT → DELIVER (or RETURN TO VENDOR, CORRECT) |

### 3.6 Payment's tables — "**Bill, Installments, Cheque, Stapler**"

| Table | Think of it as |
|-------|----------------|
| `AP_INVOICES_ALL` | The **bill** |
| `AP_PAYMENT_SCHEDULES_ALL` | The **installments** (when each part of the bill is due) |
| `AP_CHECKS_ALL` | The **cheque/payment** itself (it doesn't matter if it's EFT or wire, it's still "CHECKS") |
| `AP_INVOICE_PAYMENTS_ALL` | The **stapler** that pins a cheque to the bill(s) it paid |

> Why do we need a "stapler" table? **One payment can pay many invoices**, and **one invoice can be paid by many payments.** That's many-to-many, so a bridge table is needed.

### 3.7 The ID rule — guess the join column

**A table's primary key is usually its singular name + `_ID`.**

| Table | Primary key |
|-------|-------------|
| PO_HEADERS_ALL | PO_HEADER_ID |
| PO_LINES_ALL | PO_LINE_ID |
| PO_LINE_LOCATIONS_ALL | LINE_LOCATION_ID |
| PO_DISTRIBUTIONS_ALL | PO_DISTRIBUTION_ID |
| RCV_SHIPMENT_HEADERS | SHIPMENT_HEADER_ID |
| RCV_SHIPMENT_LINES | SHIPMENT_LINE_ID |
| RCV_TRANSACTIONS | TRANSACTION_ID ⚠️ (not RCV_TRANSACTION_ID) |
| AP_INVOICES_ALL | INVOICE_ID |
| AP_CHECKS_ALL | CHECK_ID |
| GL_JE_BATCHES / HEADERS | JE_BATCH_ID / JE_HEADER_ID |
| POZ_SUPPLIERS_V / SITES_V | VENDOR_ID / VENDOR_SITE_ID |

A **child table holds its parent's ID**. So `PO_LINES_ALL` has `PO_HEADER_ID`, and `PO_DISTRIBUTIONS_ALL` has all three parent IDs (`PO_HEADER_ID`, `PO_LINE_ID`, `LINE_LOCATION_ID`).

---

## Part 4 — Every table explained

For each table: **what it is → story rows → key columns → common filters.**

> Column names below follow the standard Fusion data model. Always confirm with your instance (for example, `SELECT * FROM <table> WHERE ROWNUM < 5` in BI Publisher) because column sets can differ slightly between releases.

---

### 4.1 SUPPLIER (POZ)

#### `POZ_SUPPLIERS_V` — "Who is the supplier?"
One row per supplier.

| VENDOR_ID | SEGMENT1 (Supplier No.) | VENDOR_NAME | PARTY_ID | ENABLED_FLAG |
|-----------|------------------------|-------------|----------|--------------|
| 9001 | 10045 | Dell India | 40001 | Y |

**Key columns:** `VENDOR_ID` (PK), `SEGMENT1` (supplier number), `VENDOR_NAME`, `VENDOR_TYPE_LOOKUP_CODE`, `PARTY_ID`, `END_DATE_ACTIVE`
**Common filters:** active suppliers → `ENABLED_FLAG = 'Y'` and `(END_DATE_ACTIVE IS NULL OR END_DATE_ACTIVE > SYSDATE)`

#### `POZ_SUPPLIER_SITES_V` — "Which address/branch of the supplier?"
One supplier → many sites (Dell Bangalore, Dell Mumbai…). A site is defined **per Procurement BU**.

| VENDOR_SITE_ID | VENDOR_ID | VENDOR_SITE_CODE | PRC_BU_ID | PURCHASING_SITE_FLAG | PAY_SITE_FLAG |
|----------------|-----------|------------------|-----------|----------------------|---------------|
| 9101 | 9001 | DELL-BLR | 300 | Y | Y |

**Key columns:** `VENDOR_SITE_ID` (PK), `VENDOR_ID` (FK), `VENDOR_SITE_CODE`, `PRC_BU_ID`, `PURCHASING_SITE_FLAG` (can we order from this site?), `PAY_SITE_FLAG` (can we pay this site?), `INACTIVE_DATE`

> **Remember:** Supplier = the **company**. Site = the **branch** you order from or pay. **Both IDs appear on PO, Receipt, Invoice and Payment.** That's why your sheet joins POZ to 4 different tables.

---

### 4.2 PURCHASING (PO)

#### `PO_HEADERS_ALL` — "The deal"
One row per PO.

| PO_HEADER_ID | SEGMENT1 (PO No.) | TYPE_LOOKUP_CODE | VENDOR_ID | VENDOR_SITE_ID | AGENT_ID (Buyer) | DOCUMENT_STATUS | CURRENCY_CODE |
|---|---|---|---|---|---|---|---|
| 5001 | PO-1001 | STANDARD | 9001 | 9101 | 1200 | OPEN | INR |

**Key columns:** `PO_HEADER_ID`, `SEGMENT1` (the **PO number** users see ⚠️), `TYPE_LOOKUP_CODE` (STANDARD / BLANKET / CONTRACT), `VENDOR_ID`, `VENDOR_SITE_ID`, `AGENT_ID` (buyer), `DOCUMENT_STATUS`, `PRC_BU_ID` (procurement BU), `CURRENCY_CODE`, `CREATION_DATE`
**Common filters:**
- Standard POs only → `TYPE_LOOKUP_CODE = 'STANDARD'`
- Approved/open POs → `DOCUMENT_STATUS = 'OPEN'` (other values: INCOMPLETE, PENDING APPROVAL, CLOSED, CLOSED FOR RECEIVING, CLOSED FOR INVOICING, FINALLY CLOSED, CANCELED)

> **Trap:** `SEGMENT1` means "PO number" here, "supplier number" in POZ_SUPPLIERS_V, and an "account segment" in GL. It's the same column name but it means something different in each table.

#### `PO_LINES_ALL` — "Which item, how many, what price"

| PO_LINE_ID | PO_HEADER_ID | LINE_NUM | ITEM_ID | ITEM_DESCRIPTION | QUANTITY | UNIT_PRICE | UOM_CODE |
|---|---|---|---|---|---|---|---|
| 5101 | 5001 | 1 | 7001 | Dell Latitude 5440 | 100 | 50000 | Ea |

**Key columns:** `PO_LINE_ID`, `PO_HEADER_ID`, `LINE_NUM`, `ITEM_ID` (= `EGP_SYSTEM_ITEMS_B.INVENTORY_ITEM_ID`), `ITEM_DESCRIPTION`, `QUANTITY`, `UNIT_PRICE`, `UOM_CODE`, `CATEGORY_ID`, `LINE_STATUS`
**Line amount** = `QUANTITY * UNIT_PRICE` → 100 × 50,000 = ₹50,00,000

> A line can have **no ITEM_ID**. Services and one-time purchases are described only by text, so use a **LEFT JOIN** to the items table.

#### `PO_LINE_LOCATIONS_ALL` — "Schedule: when and where" (+ the quantity tracker)

| LINE_LOCATION_ID | PO_LINE_ID | PO_HEADER_ID | SHIPMENT_NUM | SHIP_TO_ORGANIZATION_ID | QUANTITY | QUANTITY_RECEIVED | QUANTITY_BILLED | NEED_BY_DATE | RECEIPT_REQUIRED_FLAG | INSPECTION_REQUIRED_FLAG |
|---|---|---|---|---|---|---|---|---|---|---|
| 5201 | 5101 | 5001 | 1 | 101 (BLR) | 60 | 60 | 60 | 15-Nov-26 | Y | Y |
| 5202 | 5101 | 5001 | 2 | 102 (CHN) | 40 | 0 | 0 | 30-Nov-26 | Y | N |

**Key columns:** `LINE_LOCATION_ID`, `PO_LINE_ID`, `PO_HEADER_ID`, `SHIPMENT_NUM`, `SHIP_TO_ORGANIZATION_ID`, `SHIP_TO_LOCATION_ID`, `QUANTITY`, `QUANTITY_RECEIVED`, `QUANTITY_ACCEPTED`, `QUANTITY_BILLED`, `QUANTITY_CANCELLED`, `NEED_BY_DATE`, `PROMISED_DATE`, `RECEIPT_REQUIRED_FLAG`, `INSPECTION_REQUIRED_FLAG`

> **This is the most useful PO table for reports.** It tracks progress:
> - Pending to receive = `QUANTITY - QUANTITY_RECEIVED - QUANTITY_CANCELLED`
> - Received but not billed = `QUANTITY_RECEIVED - QUANTITY_BILLED`
> - Match type: `RECEIPT_REQUIRED_FLAG='N'` → 2-way; `'Y'` + `INSPECTION_REQUIRED_FLAG='N'` → 3-way; both `'Y'` → 4-way

#### `PO_DISTRIBUTIONS_ALL` — "Who pays (accounting)"

| PO_DISTRIBUTION_ID | LINE_LOCATION_ID | PO_LINE_ID | PO_HEADER_ID | DISTRIBUTION_NUM | QUANTITY_ORDERED | CODE_COMBINATION_ID | DESTINATION_TYPE_CODE |
|---|---|---|---|---|---|---|---|
| 5301 | 5201 | 5101 | 5001 | 1 | 30 | 11001 (IT dept acct) | INVENTORY |
| 5302 | 5201 | 5101 | 5001 | 2 | 30 | 11002 (HR dept acct) | INVENTORY |
| 5303 | 5202 | 5101 | 5001 | 1 | 40 | 11001 | INVENTORY |

**Key columns:** `PO_DISTRIBUTION_ID`, `LINE_LOCATION_ID`, `PO_LINE_ID`, `PO_HEADER_ID`, `DISTRIBUTION_NUM`, `QUANTITY_ORDERED`, `QUANTITY_DELIVERED`, `QUANTITY_BILLED`, `CODE_COMBINATION_ID` (charge account), `DESTINATION_TYPE_CODE` (INVENTORY = goes to stock, EXPENSE = used directly), `REQ_DISTRIBUTION_ID` (link back to requisition)

> **Why it matters:** The distribution is the **lowest level**, so **AP invoice distributions match to it** (`PO_DISTRIBUTION_ID`). Receiving also records it.

---

### 4.3 RECEIVING (RCV)

#### `RCV_SHIPMENT_HEADERS` — "The truck"

| SHIPMENT_HEADER_ID | RECEIPT_NUM | VENDOR_ID | VENDOR_SITE_ID | RECEIPT_SOURCE_CODE | SHIP_TO_ORG_ID | CREATION_DATE |
|---|---|---|---|---|---|---|
| 3001 | RCPT-3001 | 9001 | 9101 | VENDOR | 101 | 14-Nov-26 |

**Key columns:** `SHIPMENT_HEADER_ID`, `RECEIPT_NUM` (what users see), `VENDOR_ID`, `VENDOR_SITE_ID`, `RECEIPT_SOURCE_CODE` (**VENDOR** = from a supplier, INVENTORY/INTERNAL ORDER = internal transfers), `SHIP_TO_ORG_ID`
**Common filter:** P2P receipts only → `RECEIPT_SOURCE_CODE = 'VENDOR'`

> The receipt header has **no PO_HEADER_ID**, because one truck can carry goods for **many POs**. The PO link lives on the **lines** and **transactions**.

#### `RCV_SHIPMENT_LINES` — "The boxes"

| SHIPMENT_LINE_ID | SHIPMENT_HEADER_ID | LINE_NUM | ITEM_ID | QUANTITY_RECEIVED | PO_HEADER_ID | PO_LINE_ID | PO_LINE_LOCATION_ID | PO_DISTRIBUTION_ID | SHIPMENT_LINE_STATUS_CODE |
|---|---|---|---|---|---|---|---|---|---|
| 3101 | 3001 | 1 | 7001 | 60 | 5001 | 5101 | 5201 | (null or 5301) | FULLY RECEIVED |

**Key columns:** `SHIPMENT_LINE_ID`, `SHIPMENT_HEADER_ID`, `ITEM_ID`, `QUANTITY_SHIPPED`, `QUANTITY_RECEIVED`, `PO_HEADER_ID`, `PO_LINE_ID`, `PO_LINE_LOCATION_ID`, `TO_ORGANIZATION_ID`, `SHIPMENT_LINE_STATUS_CODE` (EXPECTED, PARTIALLY RECEIVED, FULLY RECEIVED)

#### `RCV_TRANSACTIONS` — "Every action on the boxes" ⭐ (the most important RCV table)

One box can have **many rows**, one per action. Actions form a **parent → child chain** through `PARENT_TRANSACTION_ID`.

| TRANSACTION_ID | PARENT_TRANSACTION_ID | SHIPMENT_HEADER_ID | SHIPMENT_LINE_ID | TRANSACTION_TYPE | QUANTITY | DESTINATION_TYPE_CODE | PO_HEADER_ID | PO_LINE_LOCATION_ID | PO_DISTRIBUTION_ID |
|---|---|---|---|---|---|---|---|---|---|
| 4001 | -1 | 3001 | 3101 | RECEIVE | 60 | RECEIVING | 5001 | 5201 | (null) |
| 4002 | 4001 | 3001 | 3101 | ACCEPT | 60 | RECEIVING | 5001 | 5201 | (null) |
| 4003 | 4002 | 3001 | 3101 | DELIVER | 30 | INVENTORY | 5001 | 5201 | 5301 |
| 4004 | 4002 | 3001 | 3101 | DELIVER | 30 | INVENTORY | 5001 | 5201 | 5302 |

**Key columns:** `TRANSACTION_ID`, `PARENT_TRANSACTION_ID`, `TRANSACTION_TYPE`, `TRANSACTION_DATE`, `QUANTITY`, `DESTINATION_TYPE_CODE`, `SHIPMENT_HEADER_ID`, `SHIPMENT_LINE_ID`, `PO_HEADER_ID`, `PO_LINE_ID`, `PO_LINE_LOCATION_ID`, `PO_DISTRIBUTION_ID`, `ORGANIZATION_ID`, `SUBINVENTORY`, `INV_TRANSACTION_ID`

**TRANSACTION_TYPE values to remember:**

| Type | Meaning |
|------|---------|
| RECEIVE | Goods arrived at the dock |
| ACCEPT / REJECT | Inspection result (4-way match) |
| TRANSFER | Moved within receiving |
| DELIVER | Moved to final destination (stock or expense) |
| RETURN TO VENDOR | Sent back to supplier |
| RETURN TO RECEIVING | Taken back from stock to receiving |
| CORRECT | Fixing a wrong quantity |

> **Trap 1:** If you `SUM(QUANTITY)` over all rows, you get 60+60+30+30 = **180**, not 60. **Always filter by `TRANSACTION_TYPE`.**
> **Trap 2:** `PO_DISTRIBUTION_ID` is usually filled only on **DELIVER** rows. RECEIVE happens at the schedule level, so it's often null there.
> **Trap 3:** The column is `PO_LINE_LOCATION_ID` here but `LINE_LOCATION_ID` in PO_LINE_LOCATIONS_ALL. Your sheet just says "LINE_LOCATION_ID", so the real join is `rt.PO_LINE_LOCATION_ID = pll.LINE_LOCATION_ID`.

---

### 4.4 INVENTORY (EGP / INV)

#### `EGP_SYSTEM_ITEMS_B` / `_TL` / `_VL` — "The item master"

An item is defined **per organization** (warehouse). The same laptop in Bangalore and Chennai = **2 rows**.

| INVENTORY_ITEM_ID | ORGANIZATION_ID | ITEM_NUMBER | PRIMARY_UOM_CODE | INVENTORY_ITEM_STATUS_CODE |
|---|---|---|---|---|
| 7001 | 101 | LAPTOP-DELL-5440 | Ea | Active |
| 7001 | 102 | LAPTOP-DELL-5440 | Ea | Active |

- `_B`: codes, flags, IDs
- `_TL`: `DESCRIPTION`, `LONG_DESCRIPTION` per `LANGUAGE`
- `_VL`: both together, in your session language. **Use `_VL` in reports** to get the description without handling language yourself.

> **The #1 item trap:** The key is **`INVENTORY_ITEM_ID` + `ORGANIZATION_ID`** together. If you join only on item ID, every row **duplicates once per org**. Your sheet shows exactly this: the join is `INVENTORY_ITEM_ID, ORGANIZATION_ID`.

#### `INV_MATERIAL_TXNS` — "The stock movement diary"
Every movement in or out of stock = one row. A PO delivery creates a **positive** quantity row.

| TRANSACTION_ID | INVENTORY_ITEM_ID | ORGANIZATION_ID | SUBINVENTORY_CODE | LOCATOR_ID | TRANSACTION_QUANTITY | TRANSACTION_SOURCE_TYPE_ID | TRANSACTION_SOURCE_ID | TRANSACTION_DATE |
|---|---|---|---|---|---|---|---|---|
| 9501 | 7001 | 101 | STORES | 601 | 30 | 1 (Purchase Order) | 5001 (PO header) | 14-Nov-26 |
| 9502 | 7001 | 101 | STORES | 601 | 30 | 1 | 5001 | 14-Nov-26 |

**Key columns:** `TRANSACTION_ID`, `INVENTORY_ITEM_ID`, `ORGANIZATION_ID`, `SUBINVENTORY_CODE`, `LOCATOR_ID`, `TRANSACTION_QUANTITY` (+ in / − out), `TRANSACTION_UOM`, `TRANSACTION_TYPE_ID`, `TRANSACTION_SOURCE_TYPE_ID`, `TRANSACTION_SOURCE_ID`, `TRANSACTION_DATE`
**Link to receiving (from your sheet):** `RCV_TRANSACTIONS.INV_TRANSACTION_ID = INV_MATERIAL_TXNS.TRANSACTION_ID`

> Think of it as a **bank statement for stock**: every deposit and withdrawal is a line.

#### `INV_ONHAND_QUANTITIES_DETAIL` — "Current stock balance"

| INVENTORY_ITEM_ID | ORGANIZATION_ID | SUBINVENTORY_CODE | LOCATOR_ID | LOT_NUMBER | PRIMARY_TRANSACTION_QUANTITY | CREATE_TRANSACTION_ID |
|---|---|---|---|---|---|---|
| 7001 | 101 | STORES | 601 | (null) | 30 | 9501 |
| 7001 | 101 | STORES | 601 | (null) | 30 | 9502 |

> **Bank statement vs bank balance:** `INV_MATERIAL_TXNS` is the **statement** (history). `INV_ONHAND_QUANTITIES_DETAIL` is the **balance** (now).
> **Trap:** It's called "DETAIL" because one item can have **many rows** (per subinventory, locator, lot, receipt). **Always `SUM()` and `GROUP BY`.** The on-hand here is 30 + 30 = **60**.

Your sheet joins it to material txns on `LOCATOR_ID`. In real reports, also match **item + org** (and usually subinventory). Otherwise every transaction in the same rack will match.

---

### 4.5 PAYABLES (AP) — Invoices

#### `AP_INVOICES_ALL` — "The bill (header)"

| INVOICE_ID | INVOICE_NUM | VENDOR_ID | VENDOR_SITE_ID | INVOICE_TYPE_LOOKUP_CODE | INVOICE_AMOUNT | AMOUNT_PAID | PAYMENT_STATUS_FLAG | INVOICE_DATE | CANCELLED_DATE | ORG_ID |
|---|---|---|---|---|---|---|---|---|---|---|
| 8001 | DELL-INV-778 | 9001 | 9101 | STANDARD | 3000000 | 3000000 | Y | 16-Nov-26 | (null) | 300 |

**Key columns:** `INVOICE_ID`, `INVOICE_NUM` (supplier's bill number), `VENDOR_ID`, `VENDOR_SITE_ID`, `INVOICE_TYPE_LOOKUP_CODE`, `INVOICE_AMOUNT`, `AMOUNT_PAID`, `INVOICE_CURRENCY_CODE`, `INVOICE_DATE`, `GL_DATE`, `PAYMENT_STATUS_FLAG`, `CANCELLED_DATE`, `TERMS_ID`, `ORG_ID` (Business Unit)

| Filter | Meaning |
|--------|---------|
| `INVOICE_TYPE_LOOKUP_CODE` | STANDARD, CREDIT (credit memo, negative), DEBIT, PREPAYMENT |
| `PAYMENT_STATUS_FLAG` | **Y** = fully paid, **N** = not paid, **P** = partially paid |
| `CANCELLED_DATE IS NULL` | Exclude cancelled invoices |

#### `AP_INVOICE_LINES_ALL` — "What's on the bill"

| INVOICE_ID | LINE_NUMBER | LINE_TYPE_LOOKUP_CODE | AMOUNT | QUANTITY_INVOICED | UNIT_PRICE | PO_HEADER_ID | PO_LINE_ID | PO_LINE_LOCATION_ID | MATCH_TYPE |
|---|---|---|---|---|---|---|---|---|---|
| 8001 | 1 | ITEM | 3000000 | 60 | 50000 | 5001 | 5101 | 5201 | ITEM_TO_RECEIPT |

**Key columns:** `INVOICE_ID` + `LINE_NUMBER` (**composite key**, no single LINE_ID ⚠️), `LINE_TYPE_LOOKUP_CODE` (ITEM, TAX, FREIGHT, MISCELLANEOUS), `AMOUNT`, `QUANTITY_INVOICED`, `UNIT_PRICE`, `PO_HEADER_ID`, `PO_LINE_ID`, `PO_LINE_LOCATION_ID`, `PO_DISTRIBUTION_ID`, `RCV_TRANSACTION_ID`, `MATCH_TYPE`, `DISCARDED_FLAG`

#### `AP_INVOICE_DISTRIBUTIONS_ALL` — "Accounting of the bill" ⭐ (the PO/receipt link)

| INVOICE_DISTRIBUTION_ID | INVOICE_ID | INVOICE_LINE_NUMBER | DISTRIBUTION_LINE_NUMBER | AMOUNT | DIST_CODE_COMBINATION_ID | PO_DISTRIBUTION_ID | RCV_TRANSACTION_ID | MATCH_STATUS_FLAG | POSTED_FLAG |
|---|---|---|---|---|---|---|---|---|---|
| 8101 | 8001 | 1 | 1 | 1500000 | 11001 | 5301 | 4003 | A | Y |
| 8102 | 8001 | 1 | 2 | 1500000 | 11002 | 5302 | 4004 | A | Y |

**Key columns:** `INVOICE_DISTRIBUTION_ID`, `INVOICE_ID`, `INVOICE_LINE_NUMBER`, `DISTRIBUTION_LINE_NUMBER`, `AMOUNT`, `DIST_CODE_COMBINATION_ID`, `PO_DISTRIBUTION_ID`, `RCV_TRANSACTION_ID`, `MATCH_STATUS_FLAG`, `POSTED_FLAG`, `ACCOUNTING_DATE`, `REVERSAL_FLAG`

**The join from your sheet (row 19):**
```sql
AP_INVOICE_LINES_ALL ail
JOIN AP_INVOICE_DISTRIBUTIONS_ALL aid
  ON  aid.INVOICE_ID          = ail.INVOICE_ID
  AND aid.INVOICE_LINE_NUMBER = ail.LINE_NUMBER      -- names differ!
```
> Joining only on `INVOICE_ID` would pair **every line with every distribution** of that invoice. With 3 lines × 5 distributions you'd get 15 rows instead of 5.

| Flag | Values |
|------|--------|
| `MATCH_STATUS_FLAG` | **A** = validated, **N / NULL** = never validated, **T** = tested but failed (has holds) |
| `POSTED_FLAG` | **Y** = accounted & posted, **N** = not yet |
| `REVERSAL_FLAG` | **Y** = this row was reversed (exclude it from totals) |

#### `AP_PAYMENT_SCHEDULES_ALL` — "The installments"

| INVOICE_ID | PAYMENT_NUM | DUE_DATE | GROSS_AMOUNT | AMOUNT_REMAINING | PAYMENT_STATUS_FLAG | HOLD_FLAG |
|---|---|---|---|---|---|---|
| 8001 | 1 | 16-Dec-26 | 1500000 | 0 | Y | N |
| 8001 | 2 | 15-Jan-27 | 1500000 | 0 | Y | N |

**Key columns:** `INVOICE_ID` + `PAYMENT_NUM` (composite key), `DUE_DATE`, `GROSS_AMOUNT`, `AMOUNT_REMAINING`, `PAYMENT_STATUS_FLAG`, `HOLD_FLAG`, `DISCOUNT_DATE`

> **Best table for "what do we still owe and when?"** → `AMOUNT_REMAINING > 0`, ordered by `DUE_DATE`.

---

### 4.6 PAYABLES (AP) — Payments

#### `AP_CHECKS_ALL` — "The payment"
Despite the name, it holds **all** payments: cheque, EFT, wire.

| CHECK_ID | CHECK_NUMBER | AMOUNT | CHECK_DATE | VENDOR_ID | VENDOR_SITE_ID | VENDOR_NAME | PAYMENT_METHOD_CODE | STATUS_LOOKUP_CODE |
|---|---|---|---|---|---|---|---|---|
| 6001 | CHK-6001 | 1500000 | 16-Dec-26 | 9001 | 9101 | Dell India | EFT | CLEARED |
| 6002 | CHK-6002 | 1500000 | 15-Jan-27 | 9001 | 9101 | Dell India | EFT | NEGOTIABLE |

**Key columns:** `CHECK_ID`, `CHECK_NUMBER`, `AMOUNT`, `CHECK_DATE`, `VENDOR_ID`, `VENDOR_SITE_ID`, `VENDOR_NAME`, `PAYMENT_METHOD_CODE`, `STATUS_LOOKUP_CODE` (NEGOTIABLE, CLEARED, RECONCILED, VOIDED), `VOID_DATE`, `CURRENCY_CODE`
**Common filter:** exclude voided payments → `STATUS_LOOKUP_CODE <> 'VOIDED'` (or `VOID_DATE IS NULL`)

#### `AP_INVOICE_PAYMENTS_ALL` — "The stapler (bridge)"

| INVOICE_PAYMENT_ID | CHECK_ID | INVOICE_ID | PAYMENT_NUM | AMOUNT | ACCOUNTING_DATE | REVERSAL_FLAG |
|---|---|---|---|---|---|---|
| 6101 | 6001 | 8001 | 1 | 1500000 | 16-Dec-26 | N |
| 6102 | 6002 | 8001 | 2 | 1500000 | 15-Jan-27 | N |

**Joins (all from your sheet):**
- `→ AP_CHECKS_ALL` on `CHECK_ID`
- `→ AP_INVOICES_ALL` on `INVOICE_ID`
- `→ AP_PAYMENT_SCHEDULES_ALL` on `INVOICE_ID` **AND** `PAYMENT_NUM` (tells you which installment was paid)

---

### 4.7 GENERAL LEDGER (GL)

#### `GL_JE_BATCHES` (sheet: GL_BATCHES) → `GL_JE_HEADERS` → `GL_JE_LINES`

Think of it as: **Folder → Journal → Debit/Credit lines**

| Table | Story row | Key columns |
|-------|-----------|-------------|
| `GL_JE_BATCHES` | Batch "Payables A 1234 Nov-26" | `JE_BATCH_ID`, `NAME`, `STATUS` (**P** = posted, **U** = unposted), `DEFAULT_PERIOD_NAME` |
| `GL_JE_HEADERS` | Journal "Purchase Invoices INR" | `JE_HEADER_ID`, `JE_BATCH_ID`, `LEDGER_ID`, `PERIOD_NAME`, `JE_SOURCE`, `JE_CATEGORY`, `STATUS`, `ACTUAL_FLAG`, `CURRENCY_CODE` |
| `GL_JE_LINES` | Dr Inventory 30L / Cr Liability 30L | `JE_HEADER_ID`, `JE_LINE_NUM`, `CODE_COMBINATION_ID`, `ENTERED_DR`, `ENTERED_CR`, `ACCOUNTED_DR`, `ACCOUNTED_CR`, `DESCRIPTION` |

**Filters for P2P journals:**
- `JE_SOURCE = 'Payables'` → invoices and payments
- `JE_SOURCE = 'Cost Management'` (or Receiving, depending on setup) → receipt accounting
- `JE_CATEGORY` → 'Purchase Invoices', 'Payments'
- `ACTUAL_FLAG = 'A'` → actual amounts (B = budget, E = encumbrance)

> **The accounting story of P2P:**
> 1. Receipt: **Dr** Inventory / **Cr** Receipt Accrual (AP Accrual)
> 2. Invoice: **Dr** Receipt Accrual / **Cr** Supplier Liability
> 3. Payment: **Dr** Supplier Liability / **Cr** Cash
>
> *Outside your sheet:* AP → GL goes through Subledger Accounting (`XLA_` tables), and account segments come from `GL_CODE_COMBINATIONS`. That's useful to know if an interviewer asks "how does AP reach GL?"

---

## Part 5 — The full join map

Every join from your sheet (P2P only), grouped. **Table 1 alias.column = Table 2 alias.column**

### PO internal
| # | Table 1 | Table 2 | Join |
|---|---------|---------|------|
| 2 | PO_HEADERS_ALL | PO_LINES_ALL *(sheet shows "0", a typo)* | PO_HEADER_ID |
| 3 | PO_LINES_ALL | PO_LINE_LOCATIONS_ALL | PO_LINE_ID |
| 4 | PO_LINE_LOCATIONS_ALL | PO_DISTRIBUTIONS_ALL | LINE_LOCATION_ID |
| 5 | PO_HEADERS_ALL | PO_LINE_LOCATIONS_ALL | PO_HEADER_ID |
| 6 | PO_LINES_ALL | PO_DISTRIBUTIONS_ALL | PO_LINE_ID |
| 7 | PO_HEADERS_ALL | PO_DISTRIBUTIONS_ALL | PO_HEADER_ID |

### Receiving internal + to PO
| # | Table 1 | Table 2 | Join |
|---|---------|---------|------|
| 8 | RCV_SHIPMENT_HEADERS | RCV_SHIPMENT_LINES | SHIPMENT_HEADER_ID |
| 9 | RCV_SHIPMENT_LINES | RCV_TRANSACTIONS | SHIPMENT_LINE_ID |
| 10 | RCV_SHIPMENT_HEADERS | RCV_TRANSACTIONS | SHIPMENT_HEADER_ID |
| 11 | RCV_TRANSACTIONS | PO_DISTRIBUTIONS_ALL | PO_DISTRIBUTION_ID |
| 12 | RCV_TRANSACTIONS | PO_HEADERS_ALL | PO_HEADER_ID |
| 13 | RCV_TRANSACTIONS | PO_LINES_ALL | PO_LINE_ID |
| 14 | RCV_TRANSACTIONS | PO_LINE_LOCATIONS_ALL | rt.**PO_LINE_LOCATION_ID** = pll.LINE_LOCATION_ID |

### Inventory
| # | Table 1 | Table 2 | Join |
|---|---------|---------|------|
| 15 | EGP_SYSTEM_ITEMS (_B/_TL/_VL) | INV_MATERIAL_TXNS | INVENTORY_ITEM_ID **+** ORGANIZATION_ID |
| 16 | INV_ONHAND_QUANTITIES_DETAIL | INV_MATERIAL_TXNS | LOCATOR_ID (+ item + org in practice) |
| 17 | RCV_TRANSACTIONS | INV_MATERIAL_TXNS | rt.INV_TRANSACTION_ID = imt.TRANSACTION_ID |

### AP
| # | Table 1 | Table 2 | Join |
|---|---------|---------|------|
| 18 | AP_INVOICES_ALL | AP_INVOICE_LINES_ALL | INVOICE_ID |
| 19 | AP_INVOICE_LINES_ALL | AP_INVOICE_DISTRIBUTIONS_ALL | INVOICE_ID **+** ail.LINE_NUMBER = aid.INVOICE_LINE_NUMBER |
| 20 | AP_INVOICES_ALL | AP_INVOICE_DISTRIBUTIONS_ALL | INVOICE_ID |
| 21 | AP_INVOICE_DISTRIBUTIONS_ALL | PO_DISTRIBUTIONS_ALL | PO_DISTRIBUTION_ID |
| 22 | AP_INVOICE_DISTRIBUTIONS_ALL | RCV_TRANSACTIONS | aid.RCV_TRANSACTION_ID = rt.**TRANSACTION_ID** |
| 23 | AP_PAYMENT_SCHEDULES_ALL | AP_INVOICES_ALL | INVOICE_ID |
| 24 | AP_CHECKS_ALL | AP_INVOICE_PAYMENTS_ALL | CHECK_ID |
| 25 | AP_INVOICE_PAYMENTS_ALL | AP_INVOICES_ALL | INVOICE_ID |
| 26 | AP_INVOICE_PAYMENTS_ALL | AP_PAYMENT_SCHEDULES_ALL | INVOICE_ID **+** PAYMENT_NUM |

### GL
| # | Table 1 | Table 2 | Join |
|---|---------|---------|------|
| 27 | GL_JE_BATCHES (sheet: GL_BATCHES) | GL_JE_HEADERS | JE_BATCH_ID |
| 28 | GL_JE_HEADERS | GL_JE_LINES | JE_HEADER_ID |

### Supplier
| # | Table 1 | Table 2 | Join |
|---|---------|---------|------|
| 42 / 43 | POZ_SUPPLIERS_V / POZ_SUPPLIER_SITES_V | PO_HEADERS_ALL | VENDOR_ID / VENDOR_SITE_ID |
| 44 / 45 | POZ_SUPPLIERS_V / POZ_SUPPLIER_SITES_V | AP_INVOICES_ALL | VENDOR_ID / VENDOR_SITE_ID |
| 46 / 47 | POZ_SUPPLIERS_V / POZ_SUPPLIER_SITES_V | RCV_SHIPMENT_HEADERS | VENDOR_ID / VENDOR_SITE_ID |
| 48 / 49 | POZ_SUPPLIERS_V / POZ_SUPPLIER_SITES_V | AP_CHECKS_ALL | VENDOR_ID / VENDOR_SITE_ID |

### The map as one picture

```
                                  POZ_SUPPLIERS_V ──VENDOR_ID── POZ_SUPPLIER_SITES_V
                                         │  (VENDOR_ID / VENDOR_SITE_ID go to all 4 below)
          ┌──────────────────────────────┼─────────────────────────┬──────────────────────────┐
          ▼                              ▼                         ▼                          ▼
   PO_HEADERS_ALL               RCV_SHIPMENT_HEADERS         AP_INVOICES_ALL ──INVOICE_ID── AP_CHECKS_ALL
     │ PO_HEADER_ID                 │ SHIPMENT_HEADER_ID         │ INVOICE_ID          (via AP_INVOICE_PAYMENTS_ALL
   PO_LINES_ALL                 RCV_SHIPMENT_LINES           AP_INVOICE_LINES_ALL        on CHECK_ID & INVOICE_ID)
     │ PO_LINE_ID                   │ SHIPMENT_LINE_ID           │ INVOICE_ID + LINE_NUMBER        │
   PO_LINE_LOCATIONS_ALL ◀──────RCV_TRANSACTIONS ◀─────────AP_INVOICE_DISTRIBUTIONS_ALL   AP_PAYMENT_SCHEDULES_ALL
     │ LINE_LOCATION_ID     PO_LINE_   │      RCV_TRANSACTION_ID   │                       (INVOICE_ID + PAYMENT_NUM)
   PO_DISTRIBUTIONS_ALL ◀──LOCATION_ID │                           │
          ▲     PO_DISTRIBUTION_ID ────┘                           │
          └───────────────────── PO_DISTRIBUTION_ID ───────────────┘
                                       │ INV_TRANSACTION_ID = TRANSACTION_ID
                                       ▼
   EGP_SYSTEM_ITEMS_VL ──ITEM_ID+ORG_ID── INV_MATERIAL_TXNS ──LOCATOR_ID── INV_ONHAND_QUANTITIES_DETAIL

   GL_JE_BATCHES ──JE_BATCH_ID── GL_JE_HEADERS ──JE_HEADER_ID── GL_JE_LINES
```

> **The 3 "bridges" between modules — memorize these:**
> 1. **PO ↔ Receipt:** `RCV_TRANSACTIONS.PO_*` columns
> 2. **PO ↔ Invoice:** `AP_INVOICE_DISTRIBUTIONS_ALL.PO_DISTRIBUTION_ID`
> 3. **Receipt ↔ Invoice:** `AP_INVOICE_DISTRIBUTIONS_ALL.RCV_TRANSACTION_ID`

---

## Part 6 — Use cases with SQL

> Each use case has: **Business question → Tables & why → Query → Filters explained.**
> Try writing the query yourself first. Use `:P_` bind parameters as you would in BI Publisher.

---

### 🟢 Level 1 — One module at a time

#### UC-1: List all active suppliers with their pay sites
**Business question:** "AP team wants every active supplier and the sites we can pay."
**Tables:** `POZ_SUPPLIERS_V` (company) + `POZ_SUPPLIER_SITES_V` (branches)

```sql
SELECT  ps.SEGMENT1          AS supplier_number,
        ps.VENDOR_NAME       AS supplier_name,
        pss.VENDOR_SITE_CODE AS site,
        pss.PAY_SITE_FLAG,
        pss.PURCHASING_SITE_FLAG
FROM    POZ_SUPPLIERS_V      ps
JOIN    POZ_SUPPLIER_SITES_V pss ON pss.VENDOR_ID = ps.VENDOR_ID
WHERE   ps.ENABLED_FLAG = 'Y'
AND     NVL(ps.END_DATE_ACTIVE, SYSDATE + 1) > SYSDATE
AND     NVL(pss.INACTIVE_DATE,  SYSDATE + 1) > SYSDATE
AND     pss.PAY_SITE_FLAG = 'Y'
ORDER BY ps.VENDOR_NAME, pss.VENDOR_SITE_CODE;
```
**Filter logic:** The `NVL(date, SYSDATE+1) > SYSDATE` pattern means "no end date **or** an end date in the future", i.e. still active. You used the same idea with `EFFECTIVE_END_DATE` in HCM.

---

#### UC-2: Show the complete structure of one PO
**Business question:** "Show me PO-1001 fully: lines, schedules, and charge accounts."
**Tables:** all 4 PO tables. This is the H-L-S-D chain.

```sql
SELECT  pha.SEGMENT1            AS po_number,
        pha.DOCUMENT_STATUS,
        pla.LINE_NUM,
        pla.ITEM_DESCRIPTION,
        pla.UNIT_PRICE,
        plla.SHIPMENT_NUM       AS schedule_num,
        plla.QUANTITY           AS schedule_qty,
        plla.NEED_BY_DATE,
        pda.DISTRIBUTION_NUM,
        pda.QUANTITY_ORDERED    AS dist_qty,
        pda.CODE_COMBINATION_ID AS charge_account_id
FROM    PO_HEADERS_ALL        pha
JOIN    PO_LINES_ALL          pla  ON pla.PO_HEADER_ID      = pha.PO_HEADER_ID
JOIN    PO_LINE_LOCATIONS_ALL plla ON plla.PO_LINE_ID       = pla.PO_LINE_ID
JOIN    PO_DISTRIBUTIONS_ALL  pda  ON pda.LINE_LOCATION_ID  = plla.LINE_LOCATION_ID
WHERE   pha.SEGMENT1 = :P_PO_NUMBER
ORDER BY pla.LINE_NUM, plla.SHIPMENT_NUM, pda.DISTRIBUTION_NUM;
```
**Story result:** 3 rows (BLR-IT 30, BLR-HR 30, CHN 40).
**Why:** Each join goes down **one** level using the parent's ID. This uses sheet rows 2 → 3 → 4.

---

#### UC-3: Open POs — what's still not received?
**Business question:** "Which PO schedules are still waiting for goods?"
**Tables:** `PO_HEADERS_ALL`, `PO_LINES_ALL`, `PO_LINE_LOCATIONS_ALL`. **No receipt tables needed**, because the schedule already tracks `QUANTITY_RECEIVED`.

```sql
SELECT  pha.SEGMENT1  AS po_number,
        pla.LINE_NUM,
        plla.SHIPMENT_NUM,
        plla.QUANTITY,
        plla.QUANTITY_RECEIVED,
        plla.QUANTITY - NVL(plla.QUANTITY_RECEIVED,0) - NVL(plla.QUANTITY_CANCELLED,0) AS pending_qty,
        plla.NEED_BY_DATE,
        CASE WHEN plla.NEED_BY_DATE < SYSDATE THEN 'LATE' ELSE 'ON TIME' END AS delivery_status
FROM    PO_HEADERS_ALL        pha
JOIN    PO_LINES_ALL          pla  ON pla.PO_HEADER_ID = pha.PO_HEADER_ID
JOIN    PO_LINE_LOCATIONS_ALL plla ON plla.PO_LINE_ID  = pla.PO_LINE_ID
WHERE   pha.DOCUMENT_STATUS = 'OPEN'
AND     pha.TYPE_LOOKUP_CODE = 'STANDARD'
AND     plla.QUANTITY - NVL(plla.QUANTITY_RECEIVED,0) - NVL(plla.QUANTITY_CANCELLED,0) > 0;
```
**Story result:** Schedule 2 (Chennai) → pending 40.
**Interview point:** "Why didn't you join RCV tables?" → *Because Fusion keeps running totals on the schedule. Joining RCV_TRANSACTIONS would add complexity and risk double counting.*

---

#### UC-4: Receipt details with every action
**Business question:** "For receipt RCPT-3001, show what was received, accepted and delivered."
**Tables:** `RCV_SHIPMENT_HEADERS` → `RCV_SHIPMENT_LINES` → `RCV_TRANSACTIONS`

```sql
SELECT  rsh.RECEIPT_NUM,
        rsl.LINE_NUM,
        rt.TRANSACTION_ID,
        rt.PARENT_TRANSACTION_ID,
        rt.TRANSACTION_TYPE,
        rt.QUANTITY,
        rt.DESTINATION_TYPE_CODE,
        rt.TRANSACTION_DATE
FROM    RCV_SHIPMENT_HEADERS rsh
JOIN    RCV_SHIPMENT_LINES   rsl ON rsl.SHIPMENT_HEADER_ID = rsh.SHIPMENT_HEADER_ID
JOIN    RCV_TRANSACTIONS     rt  ON rt.SHIPMENT_LINE_ID    = rsl.SHIPMENT_LINE_ID
WHERE   rsh.RECEIPT_NUM = :P_RECEIPT_NUM
AND     rsh.RECEIPT_SOURCE_CODE = 'VENDOR'
ORDER BY rt.TRANSACTION_ID;
```
**Story result:** 4 rows: RECEIVE 60, ACCEPT 60, DELIVER 30, DELIVER 30.
**Why `RECEIPT_SOURCE_CODE = 'VENDOR'`:** Receipts can also come from internal transfers. P2P = supplier receipts only.

---

#### UC-5: Current on-hand stock of an item
**Business question:** "How many Dell laptops do we have in each warehouse right now?"
**Tables:** `INV_ONHAND_QUANTITIES_DETAIL` (balance) + `EGP_SYSTEM_ITEMS_VL` (item name)

```sql
SELECT  esi.ITEM_NUMBER,
        esi.DESCRIPTION,
        ioqd.ORGANIZATION_ID,
        ioqd.SUBINVENTORY_CODE,
        SUM(ioqd.PRIMARY_TRANSACTION_QUANTITY) AS onhand_qty
FROM    INV_ONHAND_QUANTITIES_DETAIL ioqd
JOIN    EGP_SYSTEM_ITEMS_VL          esi
        ON  esi.INVENTORY_ITEM_ID = ioqd.INVENTORY_ITEM_ID
        AND esi.ORGANIZATION_ID   = ioqd.ORGANIZATION_ID     -- BOTH keys!
WHERE   esi.ITEM_NUMBER = :P_ITEM
GROUP BY esi.ITEM_NUMBER, esi.DESCRIPTION, ioqd.ORGANIZATION_ID, ioqd.SUBINVENTORY_CODE;
```
**Story result:** Org 101 / STORES → 60.
**Two traps handled:** (1) the item join uses **item + org**, (2) **SUM** because the "DETAIL" table has many rows per item.

---

#### UC-6: Unpaid invoices and their due dates (AP aging base)
**Business question:** "What do we still owe suppliers, and when is it due?"
**Tables:** `AP_INVOICES_ALL` + `AP_PAYMENT_SCHEDULES_ALL` + `POZ_SUPPLIERS_V`

```sql
SELECT  ps.VENDOR_NAME,
        aia.INVOICE_NUM,
        aia.INVOICE_AMOUNT,
        apsa.PAYMENT_NUM       AS installment,
        apsa.DUE_DATE,
        apsa.AMOUNT_REMAINING,
        TRUNC(SYSDATE) - TRUNC(apsa.DUE_DATE) AS days_overdue
FROM    AP_INVOICES_ALL          aia
JOIN    AP_PAYMENT_SCHEDULES_ALL apsa ON apsa.INVOICE_ID = aia.INVOICE_ID
JOIN    POZ_SUPPLIERS_V          ps   ON ps.VENDOR_ID    = aia.VENDOR_ID
WHERE   aia.CANCELLED_DATE IS NULL
AND     aia.PAYMENT_STATUS_FLAG IN ('N','P')
AND     apsa.AMOUNT_REMAINING > 0
ORDER BY apsa.DUE_DATE;
```
**Why payment schedules and not the invoice?** The invoice tells you the **total**. The schedule tells you **how much is left per installment and when it's due**. Aging reports always use the schedule.

---

#### UC-7: P2P journals in GL
**Business question:** "Show Payables journals posted to GL for the Nov-26 period."
**Tables:** `GL_JE_BATCHES` → `GL_JE_HEADERS` → `GL_JE_LINES`

```sql
SELECT  gjb.NAME         AS batch_name,
        gjh.NAME         AS journal_name,
        gjh.JE_CATEGORY,
        gjl.JE_LINE_NUM,
        gjl.CODE_COMBINATION_ID,
        gjl.ACCOUNTED_DR,
        gjl.ACCOUNTED_CR
FROM    GL_JE_BATCHES gjb
JOIN    GL_JE_HEADERS gjh ON gjh.JE_BATCH_ID  = gjb.JE_BATCH_ID
JOIN    GL_JE_LINES   gjl ON gjl.JE_HEADER_ID = gjh.JE_HEADER_ID
WHERE   gjh.JE_SOURCE   = 'Payables'
AND     gjh.PERIOD_NAME = :P_PERIOD          -- e.g. 'Nov-26'
AND     gjh.ACTUAL_FLAG = 'A'
AND     gjb.STATUS      = 'P';               -- posted only
```
**Check yourself:** For any journal, `SUM(ACCOUNTED_DR) = SUM(ACCOUNTED_CR)`. Debits always equal credits.

---

### 🟡 Level 2 — Crossing two modules

#### UC-8: PO vs Receipt — ordered vs received per PO line
**Business question:** "For each PO line, how much did we order vs actually receive?"
**Tables:** PO tables + `RCV_TRANSACTIONS` (bridge #1)

```sql
SELECT  pha.SEGMENT1   AS po_number,
        pla.LINE_NUM,
        pla.QUANTITY   AS ordered_qty,
        NVL(SUM(CASE WHEN rt.TRANSACTION_TYPE = 'RECEIVE'          THEN rt.QUANTITY END),0)
      - NVL(SUM(CASE WHEN rt.TRANSACTION_TYPE = 'RETURN TO VENDOR' THEN rt.QUANTITY END),0) AS net_received_qty
FROM    PO_HEADERS_ALL   pha
JOIN    PO_LINES_ALL     pla ON pla.PO_HEADER_ID = pha.PO_HEADER_ID
LEFT JOIN RCV_TRANSACTIONS rt ON rt.PO_LINE_ID   = pla.PO_LINE_ID     -- sheet row 13
WHERE   pha.SEGMENT1 = :P_PO_NUMBER
GROUP BY pha.SEGMENT1, pla.LINE_NUM, pla.QUANTITY;
```
**Story result:** Line 1 → ordered 100, net received 60.
**Why LEFT JOIN:** A PO line with **no receipt yet** must still show up (with 0). An INNER JOIN would hide it.
**Why CASE on TRANSACTION_TYPE:** Without it, RECEIVE + ACCEPT + DELIVER gets added up and you'd show 180.
*(This is the simplified version. `CORRECT` rows can also adjust quantities in real data.)*

---

#### UC-9: Receipt to inventory — did the delivery hit stock?
**Business question:** "For each DELIVER on a receipt, show the inventory transaction it created."
**Tables:** `RCV_TRANSACTIONS` → `INV_MATERIAL_TXNS` → `EGP_SYSTEM_ITEMS_VL`

```sql
SELECT  rsh.RECEIPT_NUM,
        rt.TRANSACTION_ID      AS rcv_txn_id,
        rt.QUANTITY            AS delivered_qty,
        imt.TRANSACTION_ID     AS inv_txn_id,
        imt.SUBINVENTORY_CODE,
        imt.TRANSACTION_QUANTITY,
        esi.ITEM_NUMBER
FROM    RCV_SHIPMENT_HEADERS rsh
JOIN    RCV_TRANSACTIONS     rt   ON rt.SHIPMENT_HEADER_ID = rsh.SHIPMENT_HEADER_ID   -- row 10
JOIN    INV_MATERIAL_TXNS    imt  ON imt.TRANSACTION_ID    = rt.INV_TRANSACTION_ID    -- row 17
JOIN    EGP_SYSTEM_ITEMS_VL  esi  ON esi.INVENTORY_ITEM_ID = imt.INVENTORY_ITEM_ID    -- row 15
                                 AND esi.ORGANIZATION_ID   = imt.ORGANIZATION_ID
WHERE   rt.TRANSACTION_TYPE      = 'DELIVER'
AND     rt.DESTINATION_TYPE_CODE = 'INVENTORY'
AND     rsh.RECEIPT_NUM          = :P_RECEIPT_NUM;
```
**Why only DELIVER + INVENTORY:** RECEIVE/ACCEPT don't touch stock (the goods are still at the dock). **EXPENSE** deliveries never create an inventory transaction.

---

#### UC-10: Invoice lines and distributions (with the double-key join)
**Business question:** "Show invoice DELL-INV-778 with each line and how it was charged."
**Tables:** `AP_INVOICES_ALL` → `AP_INVOICE_LINES_ALL` → `AP_INVOICE_DISTRIBUTIONS_ALL`

```sql
SELECT  aia.INVOICE_NUM,
        ail.LINE_NUMBER,
        ail.LINE_TYPE_LOOKUP_CODE,
        ail.AMOUNT              AS line_amount,
        aid.DISTRIBUTION_LINE_NUMBER,
        aid.AMOUNT              AS dist_amount,
        aid.DIST_CODE_COMBINATION_ID,
        aid.MATCH_STATUS_FLAG,
        aid.POSTED_FLAG
FROM    AP_INVOICES_ALL              aia
JOIN    AP_INVOICE_LINES_ALL         ail ON ail.INVOICE_ID = aia.INVOICE_ID
JOIN    AP_INVOICE_DISTRIBUTIONS_ALL aid ON aid.INVOICE_ID          = ail.INVOICE_ID
                                        AND aid.INVOICE_LINE_NUMBER = ail.LINE_NUMBER   -- row 19!
WHERE   aia.INVOICE_NUM = :P_INVOICE_NUM
AND     NVL(aid.REVERSAL_FLAG,'N') = 'N'
ORDER BY ail.LINE_NUMBER, aid.DISTRIBUTION_LINE_NUMBER;
```
**Story result:** 2 rows (15L to IT, 15L to HR).

---

#### UC-11: Payments made to a supplier and the invoices they cleared
**Business question:** "Which invoices did each payment to Dell pay, and which installment?"
**Tables:** `AP_CHECKS_ALL` → `AP_INVOICE_PAYMENTS_ALL` → `AP_INVOICES_ALL` (+ `AP_PAYMENT_SCHEDULES_ALL`)

```sql
SELECT  aca.CHECK_NUMBER,
        aca.CHECK_DATE,
        aca.PAYMENT_METHOD_CODE,
        aca.STATUS_LOOKUP_CODE,
        aia.INVOICE_NUM,
        aipa.PAYMENT_NUM        AS installment_paid,
        apsa.DUE_DATE,
        aipa.AMOUNT             AS amount_applied
FROM    AP_CHECKS_ALL            aca
JOIN    AP_INVOICE_PAYMENTS_ALL  aipa ON aipa.CHECK_ID   = aca.CHECK_ID          -- row 24
JOIN    AP_INVOICES_ALL          aia  ON aia.INVOICE_ID  = aipa.INVOICE_ID       -- row 25
JOIN    AP_PAYMENT_SCHEDULES_ALL apsa ON apsa.INVOICE_ID = aipa.INVOICE_ID       -- row 26
                                     AND apsa.PAYMENT_NUM = aipa.PAYMENT_NUM
WHERE   aca.VENDOR_ID = :P_VENDOR_ID
AND     aca.STATUS_LOOKUP_CODE <> 'VOIDED'
AND     NVL(aipa.REVERSAL_FLAG,'N') = 'N'
ORDER BY aca.CHECK_DATE;
```
**Why `AMOUNT` from AP_INVOICE_PAYMENTS_ALL and not AP_CHECKS_ALL:** One check of ₹10L might pay 3 invoices. The **check amount** is the whole cheque, while the **invoice payment amount** is the part applied to *this* invoice.

---

### 🔴 Level 3 — End-to-end (interview favourites)

#### UC-12: Three-way match view — PO, Receipt and Invoice in one row
**Business question:** "For every invoice distribution, show which PO and which receipt it was matched to."
**Tables:** AP dist → `PO_DISTRIBUTIONS_ALL` (bridge #2) and → `RCV_TRANSACTIONS` (bridge #3)

```sql
SELECT  aia.INVOICE_NUM,
        aid.AMOUNT              AS invoice_dist_amount,
        pha.SEGMENT1            AS po_number,
        pla.LINE_NUM            AS po_line,
        pda.DISTRIBUTION_NUM    AS po_dist,
        rsh.RECEIPT_NUM,
        rt.TRANSACTION_TYPE     AS rcv_txn_type,
        rt.QUANTITY             AS rcv_qty
FROM    AP_INVOICES_ALL              aia
JOIN    AP_INVOICE_DISTRIBUTIONS_ALL aid ON aid.INVOICE_ID         = aia.INVOICE_ID
JOIN    PO_DISTRIBUTIONS_ALL         pda ON pda.PO_DISTRIBUTION_ID = aid.PO_DISTRIBUTION_ID   -- row 21
JOIN    PO_HEADERS_ALL               pha ON pha.PO_HEADER_ID       = pda.PO_HEADER_ID          -- row 7
JOIN    PO_LINES_ALL                 pla ON pla.PO_LINE_ID         = pda.PO_LINE_ID            -- row 6
LEFT JOIN RCV_TRANSACTIONS           rt  ON rt.TRANSACTION_ID      = aid.RCV_TRANSACTION_ID    -- row 22
LEFT JOIN RCV_SHIPMENT_HEADERS       rsh ON rsh.SHIPMENT_HEADER_ID = rt.SHIPMENT_HEADER_ID
WHERE   aia.INVOICE_NUM = :P_INVOICE_NUM;
```
**Why INNER to PO but LEFT to RCV:** If an invoice is PO-matched, the PO **must** exist. A **2-way** matched invoice has **no receipt**, so a LEFT JOIN keeps those rows.
**Shortcut you can mention:** Distributions point to `PO_DISTRIBUTION_ID`, which already carries header, line and schedule IDs. So from one distribution you can reach the whole PO.

---

#### UC-13: ⭐ The grand P2P trace — Supplier → PO → Receipt → Invoice → Payment
**Business question:** "Give me the full life story of PO-1001."
**This is the query to master for interviews.**

```sql
SELECT  ps.VENDOR_NAME,
        pss.VENDOR_SITE_CODE,
        pha.SEGMENT1           AS po_number,
        pla.LINE_NUM           AS po_line,
        pda.DISTRIBUTION_NUM   AS po_dist,
        pda.QUANTITY_ORDERED,
        rsh.RECEIPT_NUM,
        rt.QUANTITY            AS delivered_qty,
        aia.INVOICE_NUM,
        aid.AMOUNT             AS invoiced_amount,
        aca.CHECK_NUMBER,
        aipa.AMOUNT            AS paid_amount
FROM    PO_HEADERS_ALL               pha
JOIN    POZ_SUPPLIERS_V              ps   ON ps.VENDOR_ID           = pha.VENDOR_ID
JOIN    POZ_SUPPLIER_SITES_V         pss  ON pss.VENDOR_SITE_ID     = pha.VENDOR_SITE_ID
JOIN    PO_LINES_ALL                 pla  ON pla.PO_HEADER_ID       = pha.PO_HEADER_ID
JOIN    PO_DISTRIBUTIONS_ALL         pda  ON pda.PO_LINE_ID         = pla.PO_LINE_ID
-- receipt side (may not exist yet)
LEFT JOIN RCV_TRANSACTIONS           rt   ON rt.PO_DISTRIBUTION_ID  = pda.PO_DISTRIBUTION_ID
                                         AND rt.TRANSACTION_TYPE    = 'DELIVER'
LEFT JOIN RCV_SHIPMENT_HEADERS       rsh  ON rsh.SHIPMENT_HEADER_ID = rt.SHIPMENT_HEADER_ID
-- invoice side (may not exist yet)
LEFT JOIN AP_INVOICE_DISTRIBUTIONS_ALL aid ON aid.PO_DISTRIBUTION_ID = pda.PO_DISTRIBUTION_ID
                                         AND aid.RCV_TRANSACTION_ID  = rt.TRANSACTION_ID
                                         AND NVL(aid.REVERSAL_FLAG,'N') = 'N'
LEFT JOIN AP_INVOICES_ALL            aia  ON aia.INVOICE_ID         = aid.INVOICE_ID
                                         AND aia.CANCELLED_DATE IS NULL
-- payment side (may not exist yet)
LEFT JOIN AP_INVOICE_PAYMENTS_ALL    aipa ON aipa.INVOICE_ID        = aia.INVOICE_ID
LEFT JOIN AP_CHECKS_ALL              aca  ON aca.CHECK_ID           = aipa.CHECK_ID
                                         AND aca.STATUS_LOOKUP_CODE <> 'VOIDED'
WHERE   pha.SEGMENT1 = :P_PO_NUMBER
ORDER BY pla.LINE_NUM, pda.DISTRIBUTION_NUM;
```

**Story result (simplified):**

| po_dist | qty | receipt | delivered | invoice | invoiced | check | paid |
|---|---|---|---|---|---|---|---|
| BLR-IT | 30 | RCPT-3001 | 30 | DELL-INV-778 | 15L | CHK-6001 | 15L |
| BLR-IT | 30 | RCPT-3001 | 30 | DELL-INV-778 | 15L | CHK-6002 | 15L |
| BLR-HR | 30 | RCPT-3001 | 30 | DELL-INV-778 | 15L | CHK-6001 | 15L |
| BLR-HR | 30 | RCPT-3001 | 30 | DELL-INV-778 | 15L | CHK-6002 | 15L |
| CHN | 40 | (null) | (null) | (null) | (null) | (null) | (null) |

**Explaining it in an interview:**
1. **Start from the PO** because that's the business question. Everything else hangs off the PO distribution.
2. **INNER JOIN** inside the PO (it must exist). **LEFT JOIN** for receipt, invoice and payment, because these happen **later** and may not have happened yet (the Chennai row shows this).
3. **Filters go in the ON clause** of LEFT JOINs. If you put `aca.STATUS_LOOKUP_CODE <> 'VOIDED'` in the WHERE, unpaid rows (null) would be **removed** and the LEFT JOIN would act like an INNER JOIN.
4. **⚠️ Fan-out warning:** Notice the BLR rows doubled. That's because the invoice was paid by **2 checks**, so every distribution row repeated per check. **Never `SUM(invoiced_amount)` from this query.** It would show ₹60L instead of ₹30L. For totals, aggregate each module **separately** (in subqueries/CTEs) and then join the totals. This is the #1 mistake in P2P reports.

---

#### UC-14: Received but not invoiced (accrual check)
**Business question:** "Finance closing the month: what goods did we receive that the supplier hasn't billed yet?"
**Tables:** `PO_LINE_LOCATIONS_ALL` (it tracks both quantities) + PO header/lines + supplier

```sql
SELECT  ps.VENDOR_NAME,
        pha.SEGMENT1   AS po_number,
        pla.LINE_NUM,
        plla.SHIPMENT_NUM,
        plla.QUANTITY_RECEIVED,
        NVL(plla.QUANTITY_BILLED,0)                               AS qty_billed,
        plla.QUANTITY_RECEIVED - NVL(plla.QUANTITY_BILLED,0)      AS uninvoiced_qty,
        (plla.QUANTITY_RECEIVED - NVL(plla.QUANTITY_BILLED,0)) * pla.UNIT_PRICE AS uninvoiced_value
FROM    PO_HEADERS_ALL        pha
JOIN    PO_LINES_ALL          pla  ON pla.PO_HEADER_ID = pha.PO_HEADER_ID
JOIN    PO_LINE_LOCATIONS_ALL plla ON plla.PO_LINE_ID  = pla.PO_LINE_ID
JOIN    POZ_SUPPLIERS_V       ps   ON ps.VENDOR_ID     = pha.VENDOR_ID
WHERE   plla.QUANTITY_RECEIVED > NVL(plla.QUANTITY_BILLED,0);
```
**Why this matters:** You **owe** money for goods you received even without a bill. Accountants book an **accrual** for this. That's the "Receipt Accrual" in the GL story (Part 4.7).

---

## Part 7 — Traps interviewers love

| # | Trap | What goes wrong | The fix |
|---|------|-----------------|---------|
| 1 | `_ALL` tables without a BU filter | Data from every business unit gets mixed | Filter `PRC_BU_ID` (PO) / `ORG_ID` (AP) or use a BU parameter |
| 2 | Item joined on `INVENTORY_ITEM_ID` only | Rows duplicate once per warehouse | Join on **item + org** |
| 3 | AP lines → dists joined on `INVOICE_ID` only | Lines × distributions explosion | Add `aid.INVOICE_LINE_NUMBER = ail.LINE_NUMBER` |
| 4 | `SUM(QUANTITY)` on `RCV_TRANSACTIONS` | RECEIVE + ACCEPT + DELIVER all counted | Filter `TRANSACTION_TYPE` |
| 5 | Not summing `INV_ONHAND_QUANTITIES_DETAIL` | You show one fragment of the stock | `SUM()` + `GROUP BY` |
| 6 | `SEGMENT1` confusion | Wrong meaning in a different table | PO number (PO), supplier number (POZ), account segment (GL) |
| 7 | AP dist → RCV join on `RCV_TRANSACTION_ID` both sides | Column doesn't exist in RCV_TRANSACTIONS | `aid.RCV_TRANSACTION_ID = rt.TRANSACTION_ID` |
| 8 | RCV → PO schedule join on `LINE_LOCATION_ID` both sides | Column name differs | `rt.PO_LINE_LOCATION_ID = pll.LINE_LOCATION_ID` |
| 9 | Using `AP_CHECKS_ALL.AMOUNT` per invoice | One cheque can pay many invoices | Use `AP_INVOICE_PAYMENTS_ALL.AMOUNT` |
| 10 | Joining invoices AND payments AND receipts together, then SUM | Fan-out multiplies amounts | Aggregate per module first, then join totals |
| 11 | LEFT JOIN with filter in WHERE | LEFT JOIN silently becomes INNER | Put the filter in the `ON` clause |
| 12 | Forgetting cancelled/voided/reversed rows | Totals include cancelled money | `CANCELLED_DATE IS NULL`, `STATUS_LOOKUP_CODE <> 'VOIDED'`, `REVERSAL_FLAG = 'N'` |
| 13 | Calling it "schedule table" | No such table name | Schedule = `PO_LINE_LOCATIONS_ALL` |
| 14 | Looking for PO number on the receipt header | Not there (one truck → many POs) | PO link is on `RCV_SHIPMENT_LINES` / `RCV_TRANSACTIONS` |

---

## Part 8 — Self-test

Try answering first, then expand the answer.

**Q1.** Which table would you use to find "how many laptops are still pending delivery for PO-1001", and why don't you need RCV tables?
<details><summary>Answer</summary>

`PO_LINE_LOCATIONS_ALL`: `QUANTITY - QUANTITY_RECEIVED - QUANTITY_CANCELLED`. The schedule keeps running totals, so receiving tables aren't needed.
</details>

**Q2.** An invoice has 2 lines; line 1 has 3 distributions, line 2 has 2. If you join lines to distributions only on `INVOICE_ID`, how many rows do you get? How many should you get?
<details><summary>Answer</summary>

2 × 5 = **10 rows** (wrong). Correct = **5 rows**, using `INVOICE_ID + LINE_NUMBER = INVOICE_LINE_NUMBER`.
</details>

**Q3.** Name the 3 bridge columns that connect PO, Receiving and AP.
<details><summary>Answer</summary>

1. PO ↔ RCV: `RCV_TRANSACTIONS.PO_HEADER_ID / PO_LINE_ID / PO_LINE_LOCATION_ID / PO_DISTRIBUTION_ID`
2. PO ↔ AP: `AP_INVOICE_DISTRIBUTIONS_ALL.PO_DISTRIBUTION_ID`
3. RCV ↔ AP: `AP_INVOICE_DISTRIBUTIONS_ALL.RCV_TRANSACTION_ID = RCV_TRANSACTIONS.TRANSACTION_ID`
</details>

**Q4.** A receipt line has RECEIVE 50, ACCEPT 50, DELIVER 50. A report shows "received qty = 150". What went wrong?
<details><summary>Answer</summary>

It summed all `RCV_TRANSACTIONS` rows. Filter `TRANSACTION_TYPE = 'RECEIVE'` (minus RETURN TO VENDOR).
</details>

**Q5.** What's the difference between `INV_MATERIAL_TXNS` and `INV_ONHAND_QUANTITIES_DETAIL`?
<details><summary>Answer</summary>

Material txns = **history** of every movement (bank statement). On-hand detail = **current balance** (bank balance), in many fragments, so SUM it.
</details>

**Q6.** Why does `AP_INVOICE_PAYMENTS_ALL` exist? Why not just put CHECK_ID on the invoice?
<details><summary>Answer</summary>

Many-to-many: one payment can pay many invoices, and one invoice can be paid by many payments (installments, partial payments). A bridge table is needed.
</details>

**Q7.** Which table holds the PO's charge account, and which holds the invoice's charge account?
<details><summary>Answer</summary>

PO: `PO_DISTRIBUTIONS_ALL.CODE_COMBINATION_ID`. Invoice: `AP_INVOICE_DISTRIBUTIONS_ALL.DIST_CODE_COMBINATION_ID`.
</details>

**Q8.** A PO schedule has `RECEIPT_REQUIRED_FLAG = 'Y'` and `INSPECTION_REQUIRED_FLAG = 'N'`. What match type is it?
<details><summary>Answer</summary>

**3-way** (PO + Receipt + Invoice).
</details>

**Q9.** Write the join between `EGP_SYSTEM_ITEMS_VL` and `PO_LINES_ALL`. What extra condition do you need, and where does it come from?
<details><summary>Answer</summary>

```sql
esi.INVENTORY_ITEM_ID = pla.ITEM_ID
AND esi.ORGANIZATION_ID = plla.SHIP_TO_ORGANIZATION_ID   -- org comes from the schedule
```
The PO line has no org. The org is on the **schedule** (ship-to organization). Use a LEFT JOIN, because service lines have no item.
</details>

**Q10.** Write a query: total amount paid to each supplier in the last 30 days (exclude voided payments).
<details><summary>Answer</summary>

```sql
SELECT  ps.VENDOR_NAME, SUM(aca.AMOUNT) AS total_paid
FROM    AP_CHECKS_ALL   aca
JOIN    POZ_SUPPLIERS_V ps ON ps.VENDOR_ID = aca.VENDOR_ID      -- sheet row 48
WHERE   aca.CHECK_DATE >= TRUNC(SYSDATE) - 30
AND     aca.STATUS_LOOKUP_CODE <> 'VOIDED'
GROUP BY ps.VENDOR_NAME
ORDER BY total_paid DESC;
```
`AP_CHECKS_ALL.AMOUNT` is fine here because we're summing **whole payments** per supplier and not joining to invoices (no fan-out).
</details>

**Q11.** Interviewer: "In UC-13, why did you put `TRANSACTION_TYPE = 'DELIVER'` in the ON clause instead of WHERE?"
<details><summary>Answer</summary>

Because `rt` is LEFT JOINed. A WHERE filter on `rt` would remove POs with no receipt (where `rt` columns are NULL), which turns it into an INNER JOIN. In the ON clause, it only restricts which RCV rows get attached.
</details>

**Q12.** Write the order of the 7 P2P steps and one table for each.
<details><summary>Answer</summary>

Supplier (`POZ_SUPPLIERS_V`) → Requisition (outside sheet) → PO (`PO_HEADERS_ALL`) → Receipt (`RCV_TRANSACTIONS`) → Inventory (`INV_MATERIAL_TXNS`) → Invoice (`AP_INVOICES_ALL`) → Payment (`AP_CHECKS_ALL`) → GL (`GL_JE_HEADERS`)
</details>

---

## Part 9 — One-page cheat sheet

```
PROCESS:   Supplier → (Req) → PO → Receipt → Inventory → Invoice → Payment → GL

SUPPLIER   POZ_SUPPLIERS_V (company, VENDOR_ID)  →  POZ_SUPPLIER_SITES_V (branch, VENDOR_SITE_ID)
           Both IDs sit on: PO header, Receipt header, Invoice, Check

PO         H-L-S-D  "Hungry Lions Steal Donuts"  =  What / Which / When-Where / Who pays
           PO_HEADERS_ALL        PO_HEADER_ID        SEGMENT1 = PO number, DOCUMENT_STATUS
           PO_LINES_ALL          PO_LINE_ID          ITEM_ID, QUANTITY, UNIT_PRICE
           PO_LINE_LOCATIONS_ALL LINE_LOCATION_ID    = SCHEDULE; QTY_RECEIVED / QTY_BILLED; match flags
           PO_DISTRIBUTIONS_ALL  PO_DISTRIBUTION_ID  CODE_COMBINATION_ID; AP matches HERE

RECEIVING  Truck → Boxes → Actions
           RCV_SHIPMENT_HEADERS  SHIPMENT_HEADER_ID  RECEIPT_NUM, VENDOR_ID (no PO id!)
           RCV_SHIPMENT_LINES    SHIPMENT_LINE_ID    PO ids, ITEM_ID
           RCV_TRANSACTIONS      TRANSACTION_ID      TYPE: RECEIVE/ACCEPT/REJECT/DELIVER/RTV/CORRECT
                                                     PO_LINE_LOCATION_ID, PO_DISTRIBUTION_ID, INV_TRANSACTION_ID

INVENTORY  Item = ITEM_ID + ORG_ID (always both!)
           EGP_SYSTEM_ITEMS_B/_TL/_VL   (_VL for reports)
           INV_MATERIAL_TXNS            = history (statement)   ← rt.INV_TRANSACTION_ID
           INV_ONHAND_QUANTITIES_DETAIL = balance (SUM it!)

AP         Bill → Lines → Distributions  |  Installments  |  Cheque  |  Stapler
           AP_INVOICES_ALL              INVOICE_ID       PAYMENT_STATUS_FLAG Y/N/P, CANCELLED_DATE
           AP_INVOICE_LINES_ALL         INVOICE_ID + LINE_NUMBER
           AP_INVOICE_DISTRIBUTIONS_ALL + INVOICE_LINE_NUMBER; PO_DISTRIBUTION_ID; RCV_TRANSACTION_ID
                                        MATCH_STATUS_FLAG A/N/T, POSTED_FLAG, REVERSAL_FLAG
           AP_PAYMENT_SCHEDULES_ALL     INVOICE_ID + PAYMENT_NUM; DUE_DATE, AMOUNT_REMAINING
           AP_CHECKS_ALL                CHECK_ID; STATUS_LOOKUP_CODE (≠ VOIDED)
           AP_INVOICE_PAYMENTS_ALL      CHECK_ID + INVOICE_ID + PAYMENT_NUM (bridge)

GL         GL_JE_BATCHES → GL_JE_HEADERS → GL_JE_LINES
           JE_BATCH_ID     JE_HEADER_ID    JE_SOURCE='Payables', ACTUAL_FLAG='A', DR = CR

BRIDGES    PO↔RCV  : RCV_TRANSACTIONS.PO_*
           PO↔AP   : AP_INVOICE_DISTRIBUTIONS_ALL.PO_DISTRIBUTION_ID
           RCV↔AP  : AP_INVOICE_DISTRIBUTIONS_ALL.RCV_TRANSACTION_ID = RCV_TRANSACTIONS.TRANSACTION_ID
           RCV↔INV : RCV_TRANSACTIONS.INV_TRANSACTION_ID = INV_MATERIAL_TXNS.TRANSACTION_ID

MATCHING   2-way PO+Inv | 3-way +Receipt | 4-way +Inspection   (flags on PO_LINE_LOCATIONS_ALL)

GOLDEN RULES
  1. Filter BU on _ALL tables          5. LEFT JOIN later steps; filters in ON
  2. Item = item + org                 6. Exclude cancelled / voided / reversed
  3. AP dist join = id + line number   7. Don't SUM across fan-out joins
  4. Filter RCV by TRANSACTION_TYPE    8. Onhand → SUM + GROUP BY
```
