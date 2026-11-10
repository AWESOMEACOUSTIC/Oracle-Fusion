# Oracle Fusion ERP — Order to Cash (O2C) Study Guide
### Learn the process, remember the tables, write the queries

> **How to use this guide:** Read Parts 1–3 to get the story and the memory tricks. Study the join map in Part 5, then solve the use cases in Part 6 one at a time, writing each query yourself before looking. Finish with the self-test (Part 8) and revise from the cheat sheet (Part 9).

> **Scope:** Only the O2C tables in your sheet: **Customer (HZ), Order Management (DOO), Shipping (WSH), Receivables (RA / AR)**. GL and Inventory are shared with P2P, so they're covered briefly here and fully in the P2P guide.

> **Tip:** You've already learned P2P. **O2C is P2P in a mirror.** In P2P we are the buyer; in O2C we are the seller. Part 3.1 maps every O2C table to the P2P table you already know.

---

## Table of Contents
1. [O2C in one picture](#part-1--o2c-in-one-picture)
2. [The running story: ABC Corp sells laptops](#part-2--the-running-story-abc-corp-sells-laptops)
3. [Memory tricks for table names](#part-3--memory-tricks-for-table-names)
4. [Every table explained (with sample rows)](#part-4--every-table-explained)
5. [The full join map](#part-5--the-full-join-map)
6. [Use cases with SQL (BI Publisher style)](#part-6--use-cases-with-sql)
7. [Traps that interviewers love](#part-7--traps-interviewers-love)
8. [Self-test (with answers)](#part-8--self-test)
9. [One-page cheat sheet](#part-9--one-page-cheat-sheet)

---

## Part 1 — O2C in one picture

O2C = everything from **"a customer wants to buy"** to **"we got the money and the books are updated."**

```
 ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌───────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
 │ CUSTOMER │──▶│  SALES   │──▶│ SHIPPING │──▶│ INVENTORY │   │ INVOICE  │──▶│ RECEIPT  │──▶│    GL    │
 │   (HZ)   │   │  ORDER   │   │  (WSH)   │   │   (INV)   │   │   (RA)   │   │   (AR)   │   │   (GL)   │
 │          │   │  (DOO)   │   │          │   │ stock ↓   │   │          │   │          │   │          │
 └──────────┘   └──────────┘   └────┬─────┘   └───────────┘   └────▲─────┘   └──────────┘   └──────────┘
  "Who is the     "What they      "Pick, pack,                   "We bill        "Customer     "Accountant
   customer?"      ordered"        ship it"                       them"           pays us"      records it"
                                    └─────── shipped lines become invoice lines (AutoInvoice) ──┘
```

### The O2C process steps

| # | Step | What happens in real life | Module | Main tables |
|---|------|---------------------------|--------|-------------|
| 1 | **Customer setup** | XYZ Retail is created with a bill-to and ship-to address | Customer (TCA) | `HZ_PARTIES`, `HZ_CUST_ACCOUNTS`, `HZ_PARTY_SITES`, `HZ_LOCATIONS`, `HZ_PARTY_SITE_USES`, `HZ_CUST_ACCT_SITES_ALL`, `HZ_CUST_SITE_USES_ALL` |
| 2 | **Sales order** | Order is entered and submitted | Order Management | `DOO_HEADERS_ALL`, `DOO_LINES_ALL` |
| 3 | **Orchestration / scheduling** | Each line is split into fulfillment lines and scheduled from a warehouse | Order Management | `DOO_FULFILL_LINES_ALL` |
| 4 | **Pick release** | The warehouse is told "go pick these items" | Shipping | `WSH_DELIVERY_DETAILS` |
| 5 | **Pick confirm & pack** | Items picked, packed, assigned to a delivery | Shipping | `WSH_DELIVERY_ASSIGNMENTS`, `WSH_NEW_DELIVERIES` |
| 6 | **Ship confirm** | Truck leaves, so stock goes **down** | Shipping + Inventory | `WSH_NEW_DELIVERIES`, (`INV_MATERIAL_TXNS` −qty) |
| 7 | **AutoInvoice** | Shipped lines become a Receivables invoice | Receivables | `RA_CUSTOMER_TRX_ALL`, `RA_CUSTOMER_TRX_LINES_ALL`, `AR_PAYMENT_SCHEDULES_ALL` |
| 8 | **Receipt** | Customer pays us | Receivables | `AR_CASH_RECEIPTS_ALL` |
| 9 | **Apply receipt** | Payment is matched to the invoice, which reduces the balance | Receivables | `AR_RECEIVABLE_APPLICATIONS_ALL` |
| 10 | **Accounting → GL** | Revenue, receivable, cash and COGS journals are posted | General Ledger | `GL_JE_BATCHES`, `GL_JE_HEADERS`, `GL_JE_LINES` |

### Fulfillment line status journey (what users see on the order)
```
Awaiting Shipping → (Picked/Staged) → Shipped → Awaiting Billing → Billed → Closed
```
> Status codes differ slightly by release. Run `SELECT DISTINCT STATUS_CODE FROM DOO_FULFILL_LINES_ALL` in your instance to see the real values.

---

## Part 2 — The running story: ABC Corp sells laptops

This continues the P2P story. ABC Corp **bought** 100 laptops from Dell. Now it **sells** some of them.

> **XYZ Retail** orders **50 laptops at ₹70,000 each** (₹35,00,000).
> - Only 30 are in stock in Bangalore, so the order line is **split into 2 fulfillment lines**: 30 now and 20 later (backordered).
> - The 30 laptops are picked, packed and shipped on **Delivery DEL-501** to XYZ's **Pune warehouse (ship-to)**.
> - AutoInvoice creates **Invoice INV-9001** for 30 × 70,000 = **₹21,00,000**, billed to XYZ's **Mumbai HQ (bill-to)**.
> - XYZ pays **₹15,00,000** (Receipt AR-701) and still owes **₹6,00,000**.

| Thing | Value |
|-------|-------|
| Customer party | XYZ Retail Pvt Ltd (PARTY_ID = 50001) |
| Customer account | ACC-1001 (CUST_ACCOUNT_ID = 60001) |
| Bill-to | Mumbai HQ (SITE_USE_ID = 73001) |
| Ship-to | Pune Warehouse (SITE_USE_ID = 73002) |
| Sales order | SO-2001 (HEADER_ID = 80001) |
| Item | LAPTOP-DELL-5440 (INVENTORY_ITEM_ID = 7001) |
| Delivery | DEL-501 (DELIVERY_ID = 85001) |
| Invoice | INV-9001 (CUSTOMER_TRX_ID = 90001) |
| Receipt | AR-701 (CASH_RECEIPT_ID = 95001) |

---

## Part 3 — Memory tricks for table names

### 3.1 ⭐ The mirror trick — O2C = P2P flipped

Learn this table and you've learned half of O2C already.

| Concept | P2P (we BUY) | O2C (we SELL) |
|---------|--------------|---------------|
| The other party | `POZ_SUPPLIERS_V` | `HZ_PARTIES` + `HZ_CUST_ACCOUNTS` |
| Their address | `POZ_SUPPLIER_SITES_V` | `HZ_CUST_ACCT_SITES_ALL` + `HZ_CUST_SITE_USES_ALL` |
| Order header | `PO_HEADERS_ALL` | `DOO_HEADERS_ALL` |
| Order line | `PO_LINES_ALL` | `DOO_LINES_ALL` |
| Schedule (when/where) | `PO_LINE_LOCATIONS_ALL` | `DOO_FULFILL_LINES_ALL` |
| Goods movement | `RCV_*` (goods come **in**) | `WSH_*` (goods go **out**) |
| Stock effect | `INV_MATERIAL_TXNS` **+qty** | `INV_MATERIAL_TXNS` **−qty** |
| The bill | `AP_INVOICES_ALL` (we **receive** a bill) | `RA_CUSTOMER_TRX_ALL` (we **send** a bill) |
| Bill lines | `AP_INVOICE_LINES_ALL` | `RA_CUSTOMER_TRX_LINES_ALL` |
| Installments / balance | `AP_PAYMENT_SCHEDULES_ALL` | `AR_PAYMENT_SCHEDULES_ALL` |
| The money | `AP_CHECKS_ALL` (money **out**) | `AR_CASH_RECEIPTS_ALL` (money **in**) |
| The "stapler" bridge | `AP_INVOICE_PAYMENTS_ALL` | `AR_RECEIVABLE_APPLICATIONS_ALL` |
| GL source | 'Payables' | 'Receivables' |

### 3.2 Decode the prefix

| Prefix | Read it as | Module |
|--------|------------|--------|
| `HZ_` | Trading Community Architecture (TCA). Often explained as **"Horizontal"**, because one customer/party model is shared **across all modules** | Customer master |
| `DOO_` | **D**istributed **O**rder **O**rchestration | Order Management |
| `WSH_` | **W**arehouse **SH**ipping | Shipping |
| `RA_` | Old Oracle Receivables prefix. Think **"Raise A bill"** | Receivables **transactions** (invoices, credit memos) |
| `AR_` | **A**ccounts **R**eceivable (money **owed to us**) | Receivables **balances and cash** |

> **RA vs AR:** `RA_` tables = the **documents** (invoice and its lines). `AR_` tables = **money tracking** (balances, receipts, applications).

### 3.3 The order's 3 levels: "**Order, Item, Shipment**"

| Level | Table | Question it answers | Story |
|-------|-------|---------------------|-------|
| Header | `DOO_HEADERS_ALL` | **Which order**, which customer? | SO-2001 for XYZ Retail |
| Line | `DOO_LINES_ALL` | **What item**, how many, what price? (what the customer asked for) | 50 laptops @ ₹70,000 |
| Fulfillment line | `DOO_FULFILL_LINES_ALL` | **How we'll ship it**: from where, when, how many per shipment (the real work) | 30 now from BLR + 20 later |

> Mnemonic: **"The customer orders a LINE, the warehouse FULFILLS it in pieces."** One line → one or more fulfillment lines. It's just like one PO line → many schedules.

### 3.4 Shipping's 3 tables: "**Parcel, Label, Van**"

| Table | Think of it as |
|-------|----------------|
| `WSH_DELIVERY_DETAILS` | The **parcel**: one shippable item quantity (comes from a fulfillment line) |
| `WSH_DELIVERY_ASSIGNMENTS` | The **label** that sticks a parcel onto a van |
| `WSH_NEW_DELIVERIES` | The **van / consignment** going to one customer address |

> The flow is **Detail → Assignment → Delivery**. It's a bridge again, just like `AP_INVOICE_PAYMENTS_ALL`. One delivery carries many details, and a detail can be re-assigned or packed into containers.

### 3.5 The customer model: "**Person vs Bank Account**"

This is the most confusing part of O2C, so here's a real-life analogy:

| TCA concept | Real life |
|-------------|-----------|
| **Party** (`HZ_PARTIES`) | **You as a person**: your identity (name). Exists once. |
| **Customer Account** (`HZ_CUST_ACCOUNTS`) | **Your account at a bank**: a business relationship. One person can have many accounts. |
| **Location** (`HZ_LOCATIONS`) | A **physical address** (building, street, city) |
| **Party Site** (`HZ_PARTY_SITES`) | "**This person is linked to this address**" |
| **Party Site Use** (`HZ_PARTY_SITE_USES`) | The **purpose** of that address at party level |
| **Account Site** (`HZ_CUST_ACCT_SITES_ALL`) | "**For this account, we use this address**" |
| **Account Site Use** (`HZ_CUST_SITE_USES_ALL`) | The **purpose** of the address for this account: **BILL_TO**, **SHIP_TO** |

The model is two parallel layers, a **party layer** (who they are) and an **account layer** (how we do business with them):

```
        PARTY LAYER (identity)                    ACCOUNT LAYER (business relationship)
        ─────────────────────                     ─────────────────────────────────────
        HZ_PARTIES  ───────── PARTY_ID ─────────▶ HZ_CUST_ACCOUNTS
            │ PARTY_ID                                 │ CUST_ACCOUNT_ID
        HZ_PARTY_SITES ─────── PARTY_SITE_ID ───▶ HZ_CUST_ACCT_SITES_ALL
            │        │ PARTY_SITE_ID                   │ CUST_ACCT_SITE_ID
            │    HZ_PARTY_SITE_USES               HZ_CUST_SITE_USES_ALL  ◀── transactions point HERE
            │ LOCATION_ID                         (SITE_USE_CODE = BILL_TO / SHIP_TO)
        HZ_LOCATIONS  (the actual address text)
```

> **Golden rule:** Transactions (invoices, orders) store **account-level IDs**: `CUST_ACCOUNT_ID` and `SITE_USE_ID`. **The name lives on the party** (`HZ_PARTIES.PARTY_NAME`). **The address lives on the location** (`HZ_LOCATIONS`). So you always have to "climb" the chain to print a name and address.

### 3.6 Receivables: "**Bill, Balance, Cash, Stapler**"

| Table | Think of it as |
|-------|----------------|
| `RA_CUSTOMER_TRX_ALL` + `_LINES_ALL` | The **bill** we send |
| `AR_PAYMENT_SCHEDULES_ALL` | The **balance sheet of each document**: how much is due and how much is still left |
| `AR_CASH_RECEIPTS_ALL` | The **cash** the customer sent |
| `AR_RECEIVABLE_APPLICATIONS_ALL` | The **stapler** pinning cash to a bill |

### 3.7 The ID rule (same as P2P)

| Table | Primary key |
|-------|-------------|
| HZ_PARTIES | PARTY_ID |
| HZ_CUST_ACCOUNTS | CUST_ACCOUNT_ID |
| HZ_PARTY_SITES | PARTY_SITE_ID |
| HZ_LOCATIONS | LOCATION_ID |
| HZ_PARTY_SITE_USES | PARTY_SITE_USE_ID |
| HZ_CUST_ACCT_SITES_ALL | CUST_ACCT_SITE_ID |
| HZ_CUST_SITE_USES_ALL | SITE_USE_ID ⚠️ (not CUST_SITE_USE_ID) |
| DOO_HEADERS_ALL / LINES / FULFILL | HEADER_ID / LINE_ID / FULFILL_LINE_ID |
| WSH_NEW_DELIVERIES | DELIVERY_ID |
| WSH_DELIVERY_DETAILS | DELIVERY_DETAIL_ID |
| RA_CUSTOMER_TRX_ALL / LINES | CUSTOMER_TRX_ID / CUSTOMER_TRX_LINE_ID |
| AR_PAYMENT_SCHEDULES_ALL | PAYMENT_SCHEDULE_ID |
| AR_CASH_RECEIPTS_ALL | CASH_RECEIPT_ID |
| AR_RECEIVABLE_APPLICATIONS_ALL | RECEIVABLE_APPLICATION_ID |

> **Naming twist:** On invoices, the customer-account column is called `BILL_TO_CUSTOMER_ID`, **not** `CUST_ACCOUNT_ID`. It's "customer" in RA but "cust account" in HZ, and they hold the same value.

---

## Part 4 — Every table explained

> Column names follow the standard Fusion data model. Confirm in your instance with `SELECT * FROM <table> WHERE ROWNUM < 5` in BI Publisher.

---

### 4.1 CUSTOMER (HZ)

#### `HZ_PARTIES` — "Who are they?" (identity)

| PARTY_ID | PARTY_NUMBER | PARTY_NAME | PARTY_TYPE | STATUS |
|---|---|---|---|---|
| 50001 | P-50001 | XYZ Retail Pvt Ltd | ORGANIZATION | A |

**Key columns:** `PARTY_ID`, `PARTY_NUMBER`, `PARTY_NAME`, `PARTY_TYPE` (ORGANIZATION, PERSON, PARTY_RELATIONSHIP), `STATUS` (A = active, I = inactive)
> HZ_PARTIES isn't only customers. Suppliers, employees' contacts, and banks are parties too. That's why `POZ_SUPPLIERS_V` also has a `PARTY_ID`. To get **customers**, go through `HZ_CUST_ACCOUNTS`.

#### `HZ_CUST_ACCOUNTS` — "Our business relationship with them"

| CUST_ACCOUNT_ID | PARTY_ID | ACCOUNT_NUMBER | ACCOUNT_NAME | STATUS | CUSTOMER_TYPE |
|---|---|---|---|---|---|
| 60001 | 50001 | ACC-1001 | XYZ Retail - Domestic | A | R |

**Key columns:** `CUST_ACCOUNT_ID`, `PARTY_ID`, `ACCOUNT_NUMBER`, `ACCOUNT_NAME`, `STATUS`, `CUSTOMER_TYPE` (R = external, I = internal), `CUSTOMER_CLASS_CODE`
**Join (sheet row 50):** `HZ_PARTIES.PARTY_ID = HZ_CUST_ACCOUNTS.PARTY_ID`

#### `HZ_LOCATIONS` — "The physical address"

| LOCATION_ID | ADDRESS1 | CITY | STATE | POSTAL_CODE | COUNTRY |
|---|---|---|---|---|---|
| 70001 | 12 Nariman Point | Mumbai | MH | 400021 | IN |
| 70002 | Plot 5, Chakan MIDC | Pune | MH | 410501 | IN |

**Key columns:** `LOCATION_ID`, `ADDRESS1`–`ADDRESS4`, `CITY`, `STATE`, `POSTAL_CODE`, `COUNTRY`
> A location is **just an address**. It has no idea who uses it. That link is the party site.

#### `HZ_PARTY_SITES` — "This party uses this address"

| PARTY_SITE_ID | PARTY_ID | LOCATION_ID | PARTY_SITE_NUMBER | PARTY_SITE_NAME | IDENTIFYING_ADDRESS_FLAG | STATUS |
|---|---|---|---|---|---|---|
| 71001 | 50001 | 70001 | PS-71001 | Mumbai HQ | Y | A |
| 71002 | 50001 | 70002 | PS-71002 | Pune Warehouse | N | A |

**Joins (sheet rows 52, 53):** `HZ_PARTIES.PARTY_ID = HZ_PARTY_SITES.PARTY_ID`, `HZ_PARTY_SITES.LOCATION_ID = HZ_LOCATIONS.LOCATION_ID`
> `IDENTIFYING_ADDRESS_FLAG = 'Y'` = the party's **main** address.

#### `HZ_PARTY_SITE_USES` — "Purpose at party level"

| PARTY_SITE_USE_ID | PARTY_SITE_ID | SITE_USE_TYPE | STATUS |
|---|---|---|---|
| 71101 | 71001 | BILL_TO | A |
| 71102 | 71002 | SHIP_TO | A |

**Join (sheet row 54):** `HZ_PARTY_SITES.PARTY_SITE_ID = HZ_PARTY_SITE_USES.PARTY_SITE_ID`
> The column here is **`SITE_USE_TYPE`**, but on the account side it's **`SITE_USE_CODE`**. This is a classic trap.

#### `HZ_CUST_ACCT_SITES_ALL` — "This account uses this party site"

| CUST_ACCT_SITE_ID | CUST_ACCOUNT_ID | PARTY_SITE_ID | SET_ID | STATUS |
|---|---|---|---|---|
| 72001 | 60001 | 71001 | 0 | A |
| 72002 | 60001 | 71002 | 0 | A |

**Join (sheet row 55):** `HZ_PARTY_SITES.PARTY_SITE_ID = HZ_CUST_ACCT_SITES_ALL.PARTY_SITE_ID`
> `_ALL` because account sites are **set/BU-specific**: the same address can be set up differently for different business units.

#### `HZ_CUST_SITE_USES_ALL` — "Purpose at account level" ⭐ (transactions point here)

| SITE_USE_ID | CUST_ACCT_SITE_ID | SITE_USE_CODE | LOCATION | PRIMARY_FLAG | STATUS |
|---|---|---|---|---|---|
| 73001 | 72001 | BILL_TO | XYZ-MUM-BILL | Y | A |
| 73002 | 72002 | SHIP_TO | XYZ-PUN-SHIP | Y | A |

**Key columns:** `SITE_USE_ID`, `CUST_ACCT_SITE_ID`, `SITE_USE_CODE` (BILL_TO, SHIP_TO, …), `PRIMARY_FLAG`, `LOCATION` (a short **name** for the site use, **not** the address ⚠️), `STATUS`
**Joins (sheet rows 56, 57):**
- `HZ_CUST_ACCT_SITES_ALL.CUST_ACCT_SITE_ID = HZ_CUST_SITE_USES_ALL.CUST_ACCT_SITE_ID`
- `RA_CUSTOMER_TRX_ALL.BILL_TO_SITE_USE_ID = HZ_CUST_SITE_USES_ALL.SITE_USE_ID` (also SHIP_TO / SOLD_TO)

> **The famous "address chain":** to print an invoice's bill-to address:
> ```
> RA_CUSTOMER_TRX_ALL.BILL_TO_SITE_USE_ID
>    → HZ_CUST_SITE_USES_ALL.SITE_USE_ID        (get CUST_ACCT_SITE_ID)
>    → HZ_CUST_ACCT_SITES_ALL.CUST_ACCT_SITE_ID (get PARTY_SITE_ID)
>    → HZ_PARTY_SITES.PARTY_SITE_ID             (get LOCATION_ID)
>    → HZ_LOCATIONS.LOCATION_ID                 (ADDRESS1, CITY…)
> ```
> Mnemonic: **"Use → Account Site → Party Site → Location"**, or **"U-A-P-L: Up A Pretty Ladder"**.

---

### 4.2 ORDER MANAGEMENT (DOO)

#### `DOO_HEADERS_ALL` — "The order"

| HEADER_ID | ORDER_NUMBER | SOURCE_ORDER_NUMBER | SOLD_TO_PARTY_ID | ORDERED_DATE | STATUS_CODE | SUBMITTED_FLAG | CHANGE_VERSION_NUMBER | ORG_ID |
|---|---|---|---|---|---|---|---|---|
| 80001 | SO-2001 | WEB-777 | 50001 | 20-Nov-26 | OPEN | Y | 1 | 300 |

**Key columns:** `HEADER_ID`, `ORDER_NUMBER`, `SOURCE_ORDER_NUMBER` / `SOURCE_ORDER_SYSTEM` (where the order came from: web, legacy, etc.), `SOLD_TO_PARTY_ID` (→ `HZ_PARTIES`, not in your sheet but very useful), `ORDERED_DATE`, `STATUS_CODE`, `SUBMITTED_FLAG`, `OPEN_FLAG`, `CHANGE_VERSION_NUMBER`, `TRANSACTIONAL_CURRENCY_CODE`, `ORG_ID` (selling BU)

> **⚠️ Versioning trap (Fusion-specific):** When an order is **changed after submission** (a change order), Fusion keeps the old version as a **reference copy**. So one `ORDER_NUMBER` can have **several rows**. Report on the current, submitted version only:
> ```sql
> AND dha.SUBMITTED_FLAG = 'Y'
> AND dha.STATUS_CODE <> 'DOO_REFERENCE'   -- excludes old reference copies
> ```
> (Check `SELECT DISTINCT STATUS_CODE FROM DOO_HEADERS_ALL` to confirm the values in your instance.) HCM has a similar idea: multiple rows per person, where you had to pick the current one with effective dates.

#### `DOO_LINES_ALL` — "What the customer asked for"

| LINE_ID | HEADER_ID | LINE_NUMBER | DISPLAY_LINE_NUMBER | INVENTORY_ITEM_ID | ORDERED_QTY | UNIT_SELLING_PRICE | EXTENDED_AMOUNT | STATUS_CODE |
|---|---|---|---|---|---|---|---|---|
| 81001 | 80001 | 1 | 1 | 7001 | 50 | 70000 | 3500000 | OPEN |

**Key columns:** `LINE_ID`, `HEADER_ID`, `LINE_NUMBER`, `DISPLAY_LINE_NUMBER`, `INVENTORY_ITEM_ID`, `ORDERED_QTY`, `ORDERED_UOM`, `UNIT_SELLING_PRICE`, `EXTENDED_AMOUNT`, `STATUS_CODE`, `OPEN_FLAG`, `CATEGORY_CODE` (ORDER / RETURN)
**Join (sheet row 29):** `DOO_HEADERS_ALL.HEADER_ID = DOO_LINES_ALL.HEADER_ID`

#### `DOO_FULFILL_LINES_ALL` — "How we'll actually ship it" ⭐

| FULFILL_LINE_ID | LINE_ID | HEADER_ID | FULFILL_LINE_NUMBER | FULFILL_ORG_ID | ORDERED_QTY | SHIPPED_QTY | SCHEDULE_SHIP_DATE | STATUS_CODE |
|---|---|---|---|---|---|---|---|---|
| 82001 | 81001 | 80001 | 1 | 101 (BLR) | 30 | 30 | 22-Nov-26 | BILLED |
| 82002 | 81001 | 80001 | 2 | 101 (BLR) | 20 | 0 | 10-Dec-26 | AWAIT_SHIPPING (backordered) |

**Key columns:** `FULFILL_LINE_ID`, `LINE_ID`, `HEADER_ID`, `FULFILL_LINE_NUMBER`, `INVENTORY_ITEM_ID`, `FULFILL_ORG_ID` (warehouse), `ORDERED_QTY`, `SHIPPED_QTY`, `REQUEST_SHIP_DATE`, `SCHEDULE_SHIP_DATE`, `ACTUAL_SHIP_DATE`, `STATUS_CODE`, `OPEN_FLAG`, `CANCELED_FLAG`
**Joins (sheet rows 30, 31):** `DOO_LINES_ALL.LINE_ID = DOO_FULFILL_LINES_ALL.LINE_ID`, `DOO_FULFILL_LINES_ALL.HEADER_ID = DOO_HEADERS_ALL.HEADER_ID`

> **Why it's the most important DOO table:** shipping, invoicing and statuses all happen at **fulfillment-line** level. It's like `PO_LINE_LOCATIONS_ALL` in P2P. Pending to ship = `ORDERED_QTY - NVL(SHIPPED_QTY,0)`.

---

### 4.3 SHIPPING (WSH)

#### `WSH_DELIVERY_DETAILS` — "The parcel" ⭐

| DELIVERY_DETAIL_ID | SOURCE_HEADER_ID | SALES_ORDER_NUMBER | SALES_ORDER_LINE_NUMBER | SALES_ORDER_SHIPMENT_NUMBER | INVENTORY_ITEM_ID | ORGANIZATION_ID | REQUESTED_QUANTITY | SHIPPED_QUANTITY | RELEASED_STATUS |
|---|---|---|---|---|---|---|---|---|---|
| 84001 | 80001 | SO-2001 | 1 | 1 | 7001 | 101 | 30 | 30 | C |
| 84002 | 80001 | SO-2001 | 1 | 2 | 7001 | 101 | 20 | (null) | B |

**Key columns:** `DELIVERY_DETAIL_ID`, `SOURCE_HEADER_ID` (= DOO header), `SALES_ORDER_NUMBER`, `SALES_ORDER_LINE_NUMBER`, `SALES_ORDER_SHIPMENT_NUMBER` (= fulfill line number), `INVENTORY_ITEM_ID`, `ORGANIZATION_ID`, `REQUESTED_QUANTITY`, `SHIPPED_QUANTITY`, `RELEASED_STATUS`, `SUBINVENTORY`, `LOT_NUMBER`

**Joins (sheet rows 34–36):**
- → `DOO_HEADERS_ALL`: `wdd.SOURCE_HEADER_ID = dha.HEADER_ID`
- → `DOO_LINES_ALL`: `wdd.SALES_ORDER_NUMBER = ORDER_NUMBER` **and** `wdd.SALES_ORDER_LINE_NUMBER = dla.LINE_NUMBER`
- → `DOO_FULFILL_LINES_ALL`: `wdd.SALES_ORDER_NUMBER = ORDER_NUMBER` **and** `wdd.SALES_ORDER_SHIPMENT_NUMBER = dfla.FULFILL_LINE_NUMBER`

> `ORDER_NUMBER` lives on `DOO_HEADERS_ALL`. If it isn't on your lines/fulfill lines tables, reach it through the header. In the queries below we use `wdd.SOURCE_HEADER_ID = dfla.HEADER_ID` + line number + shipment number, which is equivalent and safer.
> Many teams also use the ID-based link `wdd.SOURCE_SHIPMENT_ID = dfla.FULFILL_LINE_ID`. Check whether that column is populated in your instance.

**`RELEASED_STATUS` — learn these:**

| Code | Meaning | Memory hook |
|------|---------|-------------|
| N | Not ready to release | **N**ot yet |
| R | **R**eady to release | |
| S | Released to warehouse (pick wave created) | **S**ent to warehouse |
| Y | Staged / pick confirmed | "**Y**es, picked" |
| C | Shipped (ship confirmed) | **C**ompleted |
| B | **B**ackordered (not enough stock) | |
| D | Cancelle**D** | |
| X | Not applicable (non-shippable) | |

#### `WSH_DELIVERY_ASSIGNMENTS` — "The label (bridge)"

| DELIVERY_ASSIGNMENT_ID | DELIVERY_DETAIL_ID | DELIVERY_ID | PARENT_DELIVERY_DETAIL_ID |
|---|---|---|---|
| 84501 | 84001 | 85001 | (null) |
| 84502 | 84002 | (null) | (null) |

**Joins (sheet rows 32, 33):** `WSH_NEW_DELIVERIES.DELIVERY_ID = WSH_DELIVERY_ASSIGNMENTS.DELIVERY_ID`, `WSH_DELIVERY_ASSIGNMENTS.DELIVERY_DETAIL_ID = WSH_DELIVERY_DETAILS.DELIVERY_DETAIL_ID`
> **`DELIVERY_ID` is NULL** = the parcel isn't on any van yet (the backordered 20). Use a **LEFT JOIN** when you want unshipped details too.
> `PARENT_DELIVERY_DETAIL_ID` = packed inside a container (box/pallet).

#### `WSH_NEW_DELIVERIES` — "The van / consignment"

| DELIVERY_ID | NAME (Delivery No.) | STATUS_CODE | ORGANIZATION_ID | INITIAL_PICKUP_DATE | WAYBILL |
|---|---|---|---|---|---|
| 85001 | DEL-501 | CL | 101 | 22-Nov-26 | BLUEDART-9988 |

**Key columns:** `DELIVERY_ID`, `NAME`, `STATUS_CODE` (OP = open, CO = confirmed, IT = in transit, CL = closed/shipped), `ORGANIZATION_ID`, `INITIAL_PICKUP_DATE` (ship date), `ULTIMATE_DROPOFF_DATE`, `WAYBILL`, `ULTIMATE_DROPOFF_LOCATION_ID`

> **Ship confirm** closes the delivery and creates an `INV_MATERIAL_TXNS` row with **negative** quantity (sales order issue), so stock goes down. Same table as P2P, opposite sign.

---

### 4.4 RECEIVABLES (RA / AR)

#### `RA_CUSTOMER_TRX_ALL` — "The bill (header)"

| CUSTOMER_TRX_ID | TRX_NUMBER | TRX_DATE | BILL_TO_CUSTOMER_ID | BILL_TO_SITE_USE_ID | SHIP_TO_CUSTOMER_ID | SHIP_TO_SITE_USE_ID | INVOICE_CURRENCY_CODE | COMPLETE_FLAG | ORG_ID |
|---|---|---|---|---|---|---|---|---|---|
| 90001 | INV-9001 | 23-Nov-26 | 60001 | 73001 | 60001 | 73002 | INR | Y | 300 |

**Key columns:** `CUSTOMER_TRX_ID`, `TRX_NUMBER`, `TRX_DATE`, `BILL_TO_CUSTOMER_ID` / `SHIP_TO_CUSTOMER_ID` / `SOLD_TO_CUSTOMER_ID` (→ `HZ_CUST_ACCOUNTS.CUST_ACCOUNT_ID`), `BILL_TO_SITE_USE_ID` / `SHIP_TO_SITE_USE_ID` / `SOLD_TO_SITE_USE_ID` (→ `HZ_CUST_SITE_USES_ALL.SITE_USE_ID`), `INVOICE_CURRENCY_CODE`, `COMPLETE_FLAG` (Y = completed, so it has accounting and a payment schedule), `ORG_ID`, `CT_REFERENCE`, transaction type ID (`CUST_TRX_TYPE_SEQ_ID` in Fusion → INV / CM / DM)
**Joins (sheet rows 51, 57):** the bill-to / sold-to / ship-to customer IDs → `HZ_CUST_ACCOUNTS.CUST_ACCOUNT_ID`, and the site use IDs → `HZ_CUST_SITE_USES_ALL.SITE_USE_ID`

#### `RA_CUSTOMER_TRX_LINES_ALL` — "Lines on the bill"

| CUSTOMER_TRX_LINE_ID | CUSTOMER_TRX_ID | LINE_NUMBER | LINE_TYPE | INVENTORY_ITEM_ID | QUANTITY_INVOICED | UNIT_SELLING_PRICE | EXTENDED_AMOUNT | LINK_TO_CUST_TRX_LINE_ID | INTERFACE_LINE_ATTRIBUTE1 | INTERFACE_LINE_ATTRIBUTE2 |
|---|---|---|---|---|---|---|---|---|---|---|
| 91001 | 90001 | 1 | LINE | 7001 | 30 | 70000 | 2100000 | (null) | '80001' | '1' |
| 91002 | 90001 | 2 | TAX | (null) | (null) | (null) | 378000 | 91001 | (null) | (null) |

**Key columns:** `CUSTOMER_TRX_LINE_ID`, `CUSTOMER_TRX_ID`, `LINE_NUMBER`, `LINE_TYPE` (**LINE**, TAX, FREIGHT, CHARGES), `INVENTORY_ITEM_ID`, `DESCRIPTION`, `QUANTITY_INVOICED`, `UNIT_SELLING_PRICE`, `EXTENDED_AMOUNT`, `LINK_TO_CUST_TRX_LINE_ID` (tax line → its item line), `SALES_ORDER`, `SALES_ORDER_LINE`, `INTERFACE_LINE_CONTEXT`, `INTERFACE_LINE_ATTRIBUTE1..15`

**Joins (sheet rows 37, 38):**
- → `RA_CUSTOMER_TRX_ALL`: `CUSTOMER_TRX_ID`
- → `DOO_LINES_ALL` (the **AutoInvoice bridge**):
  ```sql
  rctl.INTERFACE_LINE_ATTRIBUTE1 = TO_CHAR(dla.HEADER_ID)     -- order / header mapping
  AND rctl.INTERFACE_LINE_ATTRIBUTE2 = TO_CHAR(dla.LINE_NUMBER)  -- order line number
  ```

> **Why `TO_CHAR`?** The interface attribute columns are **VARCHAR2** (generic text slots). The DOO IDs are **NUMBER**, so you must convert one side.
> **Why "attribute"?** AutoInvoice is generic: it can create invoices from Order Management, Projects, or external systems. Each source stores its own references in these generic slots, and `INTERFACE_LINE_CONTEXT` tells you which source filled them. What goes into each attribute depends on setup, so **check your instance**: pick one known order and run `SELECT INTERFACE_LINE_CONTEXT, INTERFACE_LINE_ATTRIBUTE1, …6, SALES_ORDER, SALES_ORDER_LINE FROM RA_CUSTOMER_TRX_LINES_ALL WHERE CUSTOMER_TRX_ID = …`.
> **Always filter `LINE_TYPE = 'LINE'`** when joining to orders. TAX and FREIGHT lines have no order reference.

#### `AR_PAYMENT_SCHEDULES_ALL` — "Balance of every document" ⭐

| PAYMENT_SCHEDULE_ID | CLASS | CUSTOMER_TRX_ID | CASH_RECEIPT_ID | TRX_NUMBER | DUE_DATE | AMOUNT_DUE_ORIGINAL | AMOUNT_DUE_REMAINING | STATUS | CUSTOMER_ID |
|---|---|---|---|---|---|---|---|---|---|
| 92001 | INV | 90001 | (null) | INV-9001 | 23-Dec-26 | 2478000 | 978000 | OP | 60001 |
| 92002 | PMT | (null) | 95001 | AR-701 | 25-Nov-26 | -1500000 | 0 | CL | 60001 |

**Key columns:** `PAYMENT_SCHEDULE_ID`, `CLASS`, `CUSTOMER_TRX_ID` (filled for invoices/CMs), `CASH_RECEIPT_ID` (filled for receipts), `TRX_NUMBER`, `DUE_DATE`, `AMOUNT_DUE_ORIGINAL`, `AMOUNT_DUE_REMAINING`, `AMOUNT_APPLIED`, `STATUS` (OP = open, CL = closed), `CUSTOMER_ID` (= cust account), `CUSTOMER_SITE_USE_ID`, `GL_DATE_CLOSED`
**Join (sheet row 39):** `AR_PAYMENT_SCHEDULES_ALL.CUSTOMER_TRX_ID = RA_CUSTOMER_TRX_ALL.CUSTOMER_TRX_ID`

| `CLASS` | Meaning |
|---------|---------|
| INV | Invoice |
| CM | Credit memo (negative) |
| DM | Debit memo |
| PMT | Receipt (payment) |
| DEP | Deposit |
| CB | Chargeback |

> **Big difference from AP:** `AR_PAYMENT_SCHEDULES_ALL` holds rows for **invoices AND receipts**, and it has its own single key (`PAYMENT_SCHEDULE_ID`). **Always filter `CLASS`.** Also note that tax is included, so the invoice total 24.78L = 21L + 3.78L tax.

#### `AR_CASH_RECEIPTS_ALL` — "The money that came in"

| CASH_RECEIPT_ID | RECEIPT_NUMBER | RECEIPT_DATE | AMOUNT | CURRENCY_CODE | PAY_FROM_CUSTOMER | STATUS | TYPE |
|---|---|---|---|---|---|---|---|
| 95001 | AR-701 | 25-Nov-26 | 1500000 | INR | 60001 | APP | CASH |

**Key columns:** `CASH_RECEIPT_ID`, `RECEIPT_NUMBER`, `RECEIPT_DATE`, `AMOUNT`, `CURRENCY_CODE`, `PAY_FROM_CUSTOMER` (→ `HZ_CUST_ACCOUNTS.CUST_ACCOUNT_ID`, not in sheet), `CUSTOMER_SITE_USE_ID`, `STATUS`, `TYPE` (CASH = customer payment, MISC = miscellaneous, e.g. interest), `RECEIPT_METHOD_ID`, `REVERSAL_DATE`

| `STATUS` | Meaning |
|----------|---------|
| APP | Applied (fully applied to invoices) |
| UNAPP | Unapplied (money received, not yet matched) |
| UNID | Unidentified (we don't know who paid) |
| REV | Reversed |
| NSF / STOP | Bounced / stopped payment |

#### `AR_RECEIVABLE_APPLICATIONS_ALL` — "The stapler" ⭐

| RECEIVABLE_APPLICATION_ID | CASH_RECEIPT_ID | PAYMENT_SCHEDULE_ID (receipt's own) | APPLIED_PAYMENT_SCHEDULE_ID (invoice's) | APPLIED_CUSTOMER_TRX_ID | AMOUNT_APPLIED | STATUS | APPLICATION_TYPE | DISPLAY |
|---|---|---|---|---|---|---|---|---|
| 96001 | 95001 | 92002 | 92001 | 90001 | 1500000 | APP | CASH | Y |

**Joins (sheet rows 40, 41):**
- `AR_CASH_RECEIPTS_ALL.CASH_RECEIPT_ID = AR_RECEIVABLE_APPLICATIONS_ALL.CASH_RECEIPT_ID`
- `ara.APPLIED_PAYMENT_SCHEDULE_ID = AR_PAYMENT_SCHEDULES_ALL.PAYMENT_SCHEDULE_ID` ⚠️

> **Trap:** The table has **two** payment schedule columns:
> - `PAYMENT_SCHEDULE_ID` = the **receipt's own** schedule row (where the money came **from**)
> - `APPLIED_PAYMENT_SCHEDULE_ID` = the **invoice's** schedule row (where the money went **to**)
>
> Memory hook: "**APPLIED_** = **applied TO**".

**Filters:**
- `STATUS = 'APP'` → real applications (other values: UNAPP = unapplied, ACC = on-account, UNID)
- `DISPLAY = 'Y'` → the **current** application row (when an application is reversed or changed, older rows stay with DISPLAY = 'N')
- `APPLICATION_TYPE` → CASH (receipt → invoice) or CM (credit memo → invoice)

---

### 4.5 GL and Inventory (shared with P2P)

**Accounting story of O2C:**

| Event | Debit | Credit | GL `JE_SOURCE` |
|-------|-------|--------|----------------|
| Ship confirm | Cost of Goods Sold | Inventory | Cost Management |
| Invoice | Receivable | Revenue + Tax | Receivables |
| Receipt | Cash | Receivable | Receivables |

- GL tables and joins are the same as P2P: `GL_JE_BATCHES → GL_JE_HEADERS → GL_JE_LINES`. Filter `JE_SOURCE = 'Receivables'`.
- Inventory: ship confirm creates `INV_MATERIAL_TXNS` rows with **negative** `TRANSACTION_QUANTITY`. Item master = `EGP_SYSTEM_ITEMS_VL` (join on item **+** org).

---

## Part 5 — The full join map

Every O2C join from your sheet:

### Order Management
| # | Table 1 | Table 2 | Join |
|---|---------|---------|------|
| 29 | DOO_HEADERS_ALL | DOO_LINES_ALL | HEADER_ID |
| 30 | DOO_LINES_ALL | DOO_FULFILL_LINES_ALL | LINE_ID |
| 31 | DOO_FULFILL_LINES_ALL | DOO_HEADERS_ALL | HEADER_ID |

### Shipping
| # | Table 1 | Table 2 | Join |
|---|---------|---------|------|
| 32 | WSH_NEW_DELIVERIES | WSH_DELIVERY_ASSIGNMENTS | DELIVERY_ID |
| 33 | WSH_DELIVERY_ASSIGNMENTS | WSH_DELIVERY_DETAILS | DELIVERY_DETAIL_ID |
| 34 | WSH_DELIVERY_DETAILS | DOO_HEADERS_ALL | wdd.SOURCE_HEADER_ID = dha.HEADER_ID |
| 35 | WSH_DELIVERY_DETAILS | DOO_LINES_ALL | wdd.SALES_ORDER_NUMBER = ORDER_NUMBER, wdd.SALES_ORDER_LINE_NUMBER = dla.LINE_NUMBER |
| 36 | WSH_DELIVERY_DETAILS | DOO_FULFILL_LINES_ALL | wdd.SALES_ORDER_NUMBER = ORDER_NUMBER, wdd.SALES_ORDER_SHIPMENT_NUMBER = dfla.FULFILL_LINE_NUMBER |

### Receivables
| # | Table 1 | Table 2 | Join |
|---|---------|---------|------|
| 37 | RA_CUSTOMER_TRX_ALL | RA_CUSTOMER_TRX_LINES_ALL | CUSTOMER_TRX_ID |
| 38 | RA_CUSTOMER_TRX_LINES_ALL | DOO_LINES_ALL | rctl.INTERFACE_LINE_ATTRIBUTE1 = TO_CHAR(dla.HEADER_ID), rctl.INTERFACE_LINE_ATTRIBUTE2 = TO_CHAR(dla.LINE_NUMBER) |
| 39 | AR_PAYMENT_SCHEDULES_ALL | RA_CUSTOMER_TRX_ALL | CUSTOMER_TRX_ID |
| 40 | AR_CASH_RECEIPTS_ALL | AR_RECEIVABLE_APPLICATIONS_ALL | CASH_RECEIPT_ID |
| 41 | AR_RECEIVABLE_APPLICATIONS_ALL | AR_PAYMENT_SCHEDULES_ALL | ara.APPLIED_PAYMENT_SCHEDULE_ID = aps.PAYMENT_SCHEDULE_ID |

### Customer (HZ)
| # | Table 1 | Table 2 | Join |
|---|---------|---------|------|
| 50 | HZ_PARTIES | HZ_CUST_ACCOUNTS | PARTY_ID |
| 51 | RA_CUSTOMER_TRX_ALL | HZ_CUST_ACCOUNTS | BILL_TO_CUSTOMER_ID / SOLD_TO_CUSTOMER_ID / SHIP_TO_CUSTOMER_ID = CUST_ACCOUNT_ID |
| 52 | HZ_PARTIES | HZ_PARTY_SITES | PARTY_ID |
| 53 | HZ_PARTY_SITES | HZ_LOCATIONS | LOCATION_ID |
| 54 | HZ_PARTY_SITES | HZ_PARTY_SITE_USES | PARTY_SITE_ID |
| 55 | HZ_PARTY_SITES | HZ_CUST_ACCT_SITES_ALL | PARTY_SITE_ID |
| 56 | HZ_CUST_ACCT_SITES_ALL | HZ_CUST_SITE_USES_ALL | CUST_ACCT_SITE_ID |
| 57 | RA_CUSTOMER_TRX_ALL | HZ_CUST_SITE_USES_ALL | BILL_TO_SITE_USE_ID / SOLD_TO_SITE_USE_ID / SHIP_TO_SITE_USE_ID = SITE_USE_ID |

### Useful joins NOT in your sheet (good to know)
| Table 1 | Table 2 | Join |
|---------|---------|------|
| DOO_HEADERS_ALL | HZ_PARTIES | dha.SOLD_TO_PARTY_ID = hp.PARTY_ID |
| AR_CASH_RECEIPTS_ALL | HZ_CUST_ACCOUNTS | acr.PAY_FROM_CUSTOMER = hca.CUST_ACCOUNT_ID |
| AR_PAYMENT_SCHEDULES_ALL | HZ_CUST_ACCOUNTS | aps.CUSTOMER_ID = hca.CUST_ACCOUNT_ID |
| AR_RECEIVABLE_APPLICATIONS_ALL | RA_CUSTOMER_TRX_ALL | ara.APPLIED_CUSTOMER_TRX_ID = rct.CUSTOMER_TRX_ID |

### The map as one picture

```
  HZ_PARTIES ──PARTY_ID── HZ_CUST_ACCOUNTS ◀───────────── BILL_TO_CUSTOMER_ID ──────────────┐
     │ PARTY_ID                                                                              │
  HZ_PARTY_SITES ──PARTY_SITE_ID── HZ_CUST_ACCT_SITES_ALL                                    │
     │      │                             │ CUST_ACCT_SITE_ID                                │
     │   HZ_PARTY_SITE_USES          HZ_CUST_SITE_USES_ALL ◀── BILL_TO_SITE_USE_ID ──────────┤
     │ LOCATION_ID                                                                           │
  HZ_LOCATIONS                                                                               │
                                                                                             │
  DOO_HEADERS_ALL ◀──SOURCE_HEADER_ID── WSH_DELIVERY_DETAILS ──DELIVERY_DETAIL_ID──┐         │
     │ HEADER_ID                           ▲ order no + line no / shipment no      │         │
  DOO_LINES_ALL ───────────────────────────┤                              WSH_DELIVERY_      │
     │ LINE_ID      ▲                      │                              ASSIGNMENTS        │
  DOO_FULFILL_LINES_ALL ───────────────────┘                                   │ DELIVERY_ID │
                    │                                                    WSH_NEW_DELIVERIES  │
                    │ INTERFACE_LINE_ATTRIBUTE1/2 = TO_CHAR(HEADER_ID / LINE_NUMBER)         │
                    │                                                                        │
             RA_CUSTOMER_TRX_LINES_ALL ──CUSTOMER_TRX_ID── RA_CUSTOMER_TRX_ALL ──────────────┘
                                                                │ CUSTOMER_TRX_ID
                                                       AR_PAYMENT_SCHEDULES_ALL
                                                                ▲ PAYMENT_SCHEDULE_ID = APPLIED_PAYMENT_SCHEDULE_ID
                                                    AR_RECEIVABLE_APPLICATIONS_ALL
                                                                │ CASH_RECEIPT_ID
                                                        AR_CASH_RECEIPTS_ALL
```

> **The 4 bridges between modules — memorize these:**
> 1. **Order ↔ Shipping:** `WSH_DELIVERY_DETAILS.SOURCE_HEADER_ID` + order/line/shipment numbers
> 2. **Order ↔ Invoice:** `RA_CUSTOMER_TRX_LINES_ALL.INTERFACE_LINE_ATTRIBUTE1/2`
> 3. **Invoice ↔ Customer:** `RA_CUSTOMER_TRX_ALL.BILL_TO_CUSTOMER_ID` / `BILL_TO_SITE_USE_ID`
> 4. **Invoice ↔ Cash:** `AR_RECEIVABLE_APPLICATIONS_ALL.APPLIED_PAYMENT_SCHEDULE_ID`

---

## Part 6 — Use cases with SQL

> Each use case has: **Business question → Tables & why → Query → Filters explained.** Write it yourself first.

---

### 🟢 Level 1 — One module at a time

#### UC-1: Customer master with bill-to and ship-to addresses
**Business question:** "Give me every active customer account with its bill-to and ship-to addresses."
**Tables:** the full HZ chain, Party → Account → Account Site → Site Use, plus Party Site → Location.

```sql
SELECT  hp.PARTY_NAME          AS customer_name,
        hca.ACCOUNT_NUMBER,
        hcsu.SITE_USE_CODE     AS purpose,
        hcsu.PRIMARY_FLAG,
        hl.ADDRESS1,
        hl.CITY,
        hl.STATE,
        hl.POSTAL_CODE
FROM    HZ_PARTIES             hp
JOIN    HZ_CUST_ACCOUNTS       hca  ON hca.PARTY_ID          = hp.PARTY_ID           -- row 50
JOIN    HZ_CUST_ACCT_SITES_ALL hcas ON hcas.CUST_ACCOUNT_ID  = hca.CUST_ACCOUNT_ID
JOIN    HZ_CUST_SITE_USES_ALL  hcsu ON hcsu.CUST_ACCT_SITE_ID = hcas.CUST_ACCT_SITE_ID   -- row 56
JOIN    HZ_PARTY_SITES         hps  ON hps.PARTY_SITE_ID     = hcas.PARTY_SITE_ID        -- row 55
JOIN    HZ_LOCATIONS           hl   ON hl.LOCATION_ID        = hps.LOCATION_ID           -- row 53
WHERE   hca.STATUS  = 'A'
AND     hcas.STATUS = 'A'
AND     hcsu.STATUS = 'A'
AND     hcsu.SITE_USE_CODE IN ('BILL_TO','SHIP_TO')
ORDER BY hp.PARTY_NAME, hcsu.SITE_USE_CODE;
```
**Story result:** 2 rows: XYZ Retail / BILL_TO / Mumbai and XYZ Retail / SHIP_TO / Pune.
**Why STATUS on 3 tables:** Each layer can be deactivated on its own. An account can be active while one of its sites is closed.

---

#### UC-2: Show one sales order fully (current version only)
**Business question:** "Show SO-2001 with its lines and fulfillment lines."
**Tables:** `DOO_HEADERS_ALL` → `DOO_LINES_ALL` → `DOO_FULFILL_LINES_ALL`

```sql
SELECT  dha.ORDER_NUMBER,
        dha.ORDERED_DATE,
        dla.DISPLAY_LINE_NUMBER,
        dla.ORDERED_QTY          AS line_qty,
        dla.UNIT_SELLING_PRICE,
        dfla.FULFILL_LINE_NUMBER,
        dfla.ORDERED_QTY         AS fulfill_qty,
        dfla.SHIPPED_QTY,
        dfla.SCHEDULE_SHIP_DATE,
        dfla.STATUS_CODE         AS fulfill_status
FROM    DOO_HEADERS_ALL       dha
JOIN    DOO_LINES_ALL         dla  ON dla.HEADER_ID = dha.HEADER_ID     -- row 29
JOIN    DOO_FULFILL_LINES_ALL dfla ON dfla.LINE_ID  = dla.LINE_ID       -- row 30
WHERE   dha.ORDER_NUMBER   = :P_ORDER_NUMBER
AND     dha.SUBMITTED_FLAG = 'Y'
AND     dha.STATUS_CODE   <> 'DOO_REFERENCE'
ORDER BY dla.LINE_NUMBER, dfla.FULFILL_LINE_NUMBER;
```
**Story result:** 2 rows (30 shipped, 20 awaiting).
**Interview point:** "Why the `DOO_REFERENCE` filter?" → *Change orders keep old versions as reference rows. Without the filter, a changed order shows up twice.*

---

#### UC-3: Backlog — what's ordered but not yet shipped?
**Business question:** "Sales head wants the order backlog: everything not yet shipped."
**Tables:** DOO header + fulfill lines. **No WSH needed**, because the fulfill line tracks `SHIPPED_QTY` (same idea as P2P UC-3).

```sql
SELECT  dha.ORDER_NUMBER,
        dfla.FULFILL_LINE_NUMBER,
        dfla.INVENTORY_ITEM_ID,
        dfla.FULFILL_ORG_ID                          AS warehouse_id,
        dfla.ORDERED_QTY,
        NVL(dfla.SHIPPED_QTY,0)                      AS shipped_qty,
        dfla.ORDERED_QTY - NVL(dfla.SHIPPED_QTY,0)   AS backlog_qty,
        dfla.SCHEDULE_SHIP_DATE,
        CASE WHEN dfla.SCHEDULE_SHIP_DATE < SYSDATE THEN 'LATE' ELSE 'ON TIME' END AS ship_status
FROM    DOO_HEADERS_ALL       dha
JOIN    DOO_FULFILL_LINES_ALL dfla ON dfla.HEADER_ID = dha.HEADER_ID      -- row 31
WHERE   dha.SUBMITTED_FLAG = 'Y'
AND     dha.STATUS_CODE   <> 'DOO_REFERENCE'
AND     dfla.OPEN_FLAG     = 'Y'
AND     NVL(dfla.CANCELED_FLAG,'N') = 'N'
AND     dfla.ORDERED_QTY - NVL(dfla.SHIPPED_QTY,0) > 0;
```
**Story result:** Fulfill line 2 → backlog 20.

---

#### UC-4: What's on each delivery and what stage is each parcel at?
**Business question:** "Warehouse wants delivery DEL-501's contents with readable statuses."
**Tables:** `WSH_NEW_DELIVERIES` → `WSH_DELIVERY_ASSIGNMENTS` → `WSH_DELIVERY_DETAILS`

```sql
SELECT  wnd.NAME                 AS delivery_number,
        wnd.STATUS_CODE          AS delivery_status,
        wnd.INITIAL_PICKUP_DATE  AS ship_date,
        wdd.SALES_ORDER_NUMBER,
        wdd.SALES_ORDER_LINE_NUMBER,
        wdd.REQUESTED_QUANTITY,
        wdd.SHIPPED_QUANTITY,
        DECODE(wdd.RELEASED_STATUS,
               'N','Not Ready', 'R','Ready to Release', 'S','Released to Warehouse',
               'Y','Staged',    'C','Shipped',          'B','Backordered',
               'D','Cancelled', 'X','Not Applicable',   wdd.RELEASED_STATUS) AS line_status
FROM    WSH_NEW_DELIVERIES       wnd
JOIN    WSH_DELIVERY_ASSIGNMENTS wda ON wda.DELIVERY_ID        = wnd.DELIVERY_ID          -- row 32
JOIN    WSH_DELIVERY_DETAILS     wdd ON wdd.DELIVERY_DETAIL_ID = wda.DELIVERY_DETAIL_ID   -- row 33
WHERE   wnd.NAME = :P_DELIVERY;
```
**Why DECODE:** Users don't understand 'C' or 'Y'. Decoding codes into labels is standard BI Publisher practice. You've done this in HCM with lookups too.

---

#### UC-5: Invoice with its lines (no tax noise)
**Business question:** "Show invoice INV-9001's item lines, and separately its total tax."
**Tables:** `RA_CUSTOMER_TRX_ALL` → `RA_CUSTOMER_TRX_LINES_ALL`

```sql
SELECT  rct.TRX_NUMBER,
        rct.TRX_DATE,
        rctl.LINE_NUMBER,
        rctl.DESCRIPTION,
        rctl.QUANTITY_INVOICED,
        rctl.UNIT_SELLING_PRICE,
        rctl.EXTENDED_AMOUNT                                   AS line_amount,
        (SELECT NVL(SUM(tax.EXTENDED_AMOUNT),0)
           FROM RA_CUSTOMER_TRX_LINES_ALL tax
          WHERE tax.LINK_TO_CUST_TRX_LINE_ID = rctl.CUSTOMER_TRX_LINE_ID
            AND tax.LINE_TYPE = 'TAX')                          AS tax_amount
FROM    RA_CUSTOMER_TRX_ALL       rct
JOIN    RA_CUSTOMER_TRX_LINES_ALL rctl ON rctl.CUSTOMER_TRX_ID = rct.CUSTOMER_TRX_ID   -- row 37
WHERE   rct.TRX_NUMBER = :P_TRX_NUMBER
AND     rct.COMPLETE_FLAG = 'Y'
AND     rctl.LINE_TYPE = 'LINE';
```
**Story result:** 1 row → line 21,00,000, tax 3,78,000.
**Why the subquery:** Tax is stored as a **separate line** that points to its item line through `LINK_TO_CUST_TRX_LINE_ID`. This puts it on the same row without duplicating item lines.

---

#### UC-6: AR aging — who owes us, and how overdue?
**Business question:** "Finance wants open customer balances bucketed by days overdue."
**Tables:** `AR_PAYMENT_SCHEDULES_ALL` + customer (account → party)

```sql
SELECT  hp.PARTY_NAME            AS customer,
        hca.ACCOUNT_NUMBER,
        aps.TRX_NUMBER,
        aps.CLASS,
        aps.DUE_DATE,
        aps.AMOUNT_DUE_REMAINING,
        TRUNC(SYSDATE) - TRUNC(aps.DUE_DATE) AS days_overdue,
        CASE
          WHEN TRUNC(SYSDATE) - TRUNC(aps.DUE_DATE) <= 0  THEN 'Current'
          WHEN TRUNC(SYSDATE) - TRUNC(aps.DUE_DATE) <= 30 THEN '1-30'
          WHEN TRUNC(SYSDATE) - TRUNC(aps.DUE_DATE) <= 60 THEN '31-60'
          WHEN TRUNC(SYSDATE) - TRUNC(aps.DUE_DATE) <= 90 THEN '61-90'
          ELSE '90+'
        END                       AS aging_bucket
FROM    AR_PAYMENT_SCHEDULES_ALL aps
JOIN    HZ_CUST_ACCOUNTS         hca ON hca.CUST_ACCOUNT_ID = aps.CUSTOMER_ID
JOIN    HZ_PARTIES               hp  ON hp.PARTY_ID         = hca.PARTY_ID         -- row 50
WHERE   aps.STATUS = 'OP'
AND     aps.CLASS IN ('INV','DM','CM')       -- not receipts
AND     aps.AMOUNT_DUE_REMAINING <> 0
ORDER BY hp.PARTY_NAME, aps.DUE_DATE;
```
**Story result:** XYZ Retail / INV-9001 / 9,78,000 / bucket depends on today's date.
**Why include CM:** Open credit memos are negative balances. They **reduce** what the customer owes, so aging must include them.
**Why `<> 0` not `> 0`:** Credit memos are negative.

---

#### UC-7: Receipts and where each was applied
**Business question:** "For each customer receipt, which invoices did it pay, and how much is still unapplied?"
**Tables:** `AR_CASH_RECEIPTS_ALL` → `AR_RECEIVABLE_APPLICATIONS_ALL` → `AR_PAYMENT_SCHEDULES_ALL`

```sql
SELECT  acr.RECEIPT_NUMBER,
        acr.RECEIPT_DATE,
        acr.AMOUNT                AS receipt_amount,
        acr.STATUS                AS receipt_status,
        aps.TRX_NUMBER            AS applied_to_invoice,
        ara.AMOUNT_APPLIED,
        ara.APPLY_DATE
FROM    AR_CASH_RECEIPTS_ALL           acr
LEFT JOIN AR_RECEIVABLE_APPLICATIONS_ALL ara
        ON  ara.CASH_RECEIPT_ID = acr.CASH_RECEIPT_ID                     -- row 40
        AND ara.STATUS  = 'APP'
        AND ara.DISPLAY = 'Y'
LEFT JOIN AR_PAYMENT_SCHEDULES_ALL     aps
        ON  aps.PAYMENT_SCHEDULE_ID = ara.APPLIED_PAYMENT_SCHEDULE_ID     -- row 41
WHERE   acr.TYPE = 'CASH'
AND     acr.REVERSAL_DATE IS NULL
AND     acr.RECEIPT_DATE BETWEEN :P_FROM_DATE AND :P_TO_DATE
ORDER BY acr.RECEIPT_NUMBER;
```
**Why LEFT JOIN + filters in ON:** A fully **unapplied** receipt has no APP row. It must still appear (with null invoice) so finance can chase it.
**Why `APPLIED_PAYMENT_SCHEDULE_ID`:** You want the **invoice** the money went to, not the receipt's own schedule.

---

### 🟡 Level 2 — Crossing two modules

#### UC-8: Ordered vs shipped per order line (Order ↔ Shipping)
**Business question:** "For each order line, compare ordered qty with what shipping actually shipped."
**Tables:** DOO header/lines + `WSH_DELIVERY_DETAILS` (bridge #1)

```sql
SELECT  dha.ORDER_NUMBER,
        dla.DISPLAY_LINE_NUMBER,
        dla.ORDERED_QTY,
        NVL(SUM(CASE WHEN wdd.RELEASED_STATUS = 'C' THEN wdd.SHIPPED_QUANTITY END),0) AS shipped_qty,
        NVL(SUM(CASE WHEN wdd.RELEASED_STATUS = 'B' THEN wdd.REQUESTED_QUANTITY END),0) AS backordered_qty
FROM    DOO_HEADERS_ALL      dha
JOIN    DOO_LINES_ALL        dla ON dla.HEADER_ID = dha.HEADER_ID
LEFT JOIN WSH_DELIVERY_DETAILS wdd
        ON  wdd.SOURCE_HEADER_ID        = dha.HEADER_ID                  -- row 34
        AND wdd.SALES_ORDER_LINE_NUMBER = dla.LINE_NUMBER                -- row 35
WHERE   dha.ORDER_NUMBER   = :P_ORDER_NUMBER
AND     dha.SUBMITTED_FLAG = 'Y'
AND     dha.STATUS_CODE   <> 'DOO_REFERENCE'
GROUP BY dha.ORDER_NUMBER, dla.DISPLAY_LINE_NUMBER, dla.ORDERED_QTY;
```
**Story result:** ordered 50, shipped 30, backordered 20.
**Why CASE on RELEASED_STATUS:** Like P2P's RCV_TRANSACTIONS trap. Delivery details exist **before** shipping (released, staged), so summing everything overstates what was shipped.

---

#### UC-9: Which order did this invoice come from? (Invoice ↔ Order)
**Business question:** "For invoice INV-9001, show the sales order and line behind each invoice line."
**Tables:** RA lines → DOO lines via the AutoInvoice bridge (bridge #2)

```sql
SELECT  rct.TRX_NUMBER,
        rctl.LINE_NUMBER        AS inv_line,
        rctl.EXTENDED_AMOUNT,
        dha.ORDER_NUMBER,
        dla.DISPLAY_LINE_NUMBER AS order_line,
        dla.ORDERED_QTY
FROM    RA_CUSTOMER_TRX_ALL       rct
JOIN    RA_CUSTOMER_TRX_LINES_ALL rctl ON rctl.CUSTOMER_TRX_ID = rct.CUSTOMER_TRX_ID
JOIN    DOO_LINES_ALL             dla
        ON  rctl.INTERFACE_LINE_ATTRIBUTE1 = TO_CHAR(dla.HEADER_ID)      -- row 38
        AND rctl.INTERFACE_LINE_ATTRIBUTE2 = TO_CHAR(dla.LINE_NUMBER)
JOIN    DOO_HEADERS_ALL           dha  ON dha.HEADER_ID = dla.HEADER_ID
WHERE   rct.TRX_NUMBER = :P_TRX_NUMBER
AND     rctl.LINE_TYPE = 'LINE';
```
> **Before using this in a real report:** confirm what your instance stores in ATTRIBUTE1/2 (Part 4.4). If it stores the **order number** instead of the header ID, change the right-hand side to `dha.ORDER_NUMBER`. Some setups also fill `rctl.SALES_ORDER` / `SALES_ORDER_LINE`, which is an easier join.

---

#### UC-10: Invoice with bill-to customer name and full address (Invoice ↔ Customer)
**Business question:** "Print-ready invoice header: number, date, customer name, bill-to address."
**Tables:** RA header → account → party (name), and RA header → site use → account site → party site → location (address). This is **"Up A Pretty Ladder"**.

```sql
SELECT  rct.TRX_NUMBER,
        rct.TRX_DATE,
        hp.PARTY_NAME         AS bill_to_customer,
        hca.ACCOUNT_NUMBER,
        hl.ADDRESS1,
        hl.CITY,
        hl.STATE,
        hl.POSTAL_CODE,
        hl.COUNTRY
FROM    RA_CUSTOMER_TRX_ALL    rct
JOIN    HZ_CUST_ACCOUNTS       hca  ON hca.CUST_ACCOUNT_ID    = rct.BILL_TO_CUSTOMER_ID    -- row 51
JOIN    HZ_PARTIES             hp   ON hp.PARTY_ID            = hca.PARTY_ID               -- row 50
JOIN    HZ_CUST_SITE_USES_ALL  hcsu ON hcsu.SITE_USE_ID       = rct.BILL_TO_SITE_USE_ID    -- row 57  (U)
JOIN    HZ_CUST_ACCT_SITES_ALL hcas ON hcas.CUST_ACCT_SITE_ID = hcsu.CUST_ACCT_SITE_ID     -- row 56  (A)
JOIN    HZ_PARTY_SITES         hps  ON hps.PARTY_SITE_ID      = hcas.PARTY_SITE_ID         -- row 55  (P)
JOIN    HZ_LOCATIONS           hl   ON hl.LOCATION_ID         = hps.LOCATION_ID            -- row 53  (L)
WHERE   rct.TRX_NUMBER = :P_TRX_NUMBER;
```
**Story result:** INV-9001 / XYZ Retail Pvt Ltd / 12 Nariman Point, Mumbai.
**Exercise:** change it to print the **ship-to** address. *(Swap `BILL_TO_SITE_USE_ID` → `SHIP_TO_SITE_USE_ID`. Joining the same HZ tables twice with different aliases gives both addresses in one row.)*

---

### 🔴 Level 3 — End-to-end (interview favourites)

#### UC-11: ⭐ The grand O2C trace — Order → Shipment → Invoice → Receipt
**Business question:** "Give me the full life story of SO-2001."

```sql
SELECT  hp.PARTY_NAME             AS customer,
        dha.ORDER_NUMBER,
        dla.DISPLAY_LINE_NUMBER   AS order_line,
        dfla.FULFILL_LINE_NUMBER,
        dfla.ORDERED_QTY          AS fulfill_qty,
        wnd.NAME                  AS delivery,
        wdd.SHIPPED_QUANTITY,
        rct.TRX_NUMBER            AS invoice,
        rctl.EXTENDED_AMOUNT      AS invoiced_amount,
        aps.AMOUNT_DUE_REMAINING  AS invoice_balance,
        acr.RECEIPT_NUMBER,
        ara.AMOUNT_APPLIED        AS paid_amount
FROM    DOO_HEADERS_ALL       dha
JOIN    HZ_PARTIES            hp   ON hp.PARTY_ID   = dha.SOLD_TO_PARTY_ID        -- not in sheet
JOIN    DOO_LINES_ALL         dla  ON dla.HEADER_ID = dha.HEADER_ID
JOIN    DOO_FULFILL_LINES_ALL dfla ON dfla.LINE_ID  = dla.LINE_ID
-- shipping (may not have happened yet)
LEFT JOIN WSH_DELIVERY_DETAILS     wdd ON wdd.SOURCE_HEADER_ID            = dfla.HEADER_ID
                                      AND wdd.SALES_ORDER_LINE_NUMBER     = dla.LINE_NUMBER
                                      AND wdd.SALES_ORDER_SHIPMENT_NUMBER = dfla.FULFILL_LINE_NUMBER
                                      AND wdd.RELEASED_STATUS             = 'C'
LEFT JOIN WSH_DELIVERY_ASSIGNMENTS wda ON wda.DELIVERY_DETAIL_ID = wdd.DELIVERY_DETAIL_ID
LEFT JOIN WSH_NEW_DELIVERIES       wnd ON wnd.DELIVERY_ID        = wda.DELIVERY_ID
-- invoice (may not exist yet)
LEFT JOIN RA_CUSTOMER_TRX_LINES_ALL rctl ON rctl.INTERFACE_LINE_ATTRIBUTE1 = TO_CHAR(dla.HEADER_ID)
                                        AND rctl.INTERFACE_LINE_ATTRIBUTE2 = TO_CHAR(dla.LINE_NUMBER)
                                        AND rctl.LINE_TYPE = 'LINE'
LEFT JOIN RA_CUSTOMER_TRX_ALL       rct  ON rct.CUSTOMER_TRX_ID = rctl.CUSTOMER_TRX_ID
LEFT JOIN AR_PAYMENT_SCHEDULES_ALL  aps  ON aps.CUSTOMER_TRX_ID = rct.CUSTOMER_TRX_ID
-- cash (may not have come in yet)
LEFT JOIN AR_RECEIVABLE_APPLICATIONS_ALL ara ON ara.APPLIED_PAYMENT_SCHEDULE_ID = aps.PAYMENT_SCHEDULE_ID
                                            AND ara.STATUS  = 'APP'
                                            AND ara.DISPLAY = 'Y'
LEFT JOIN AR_CASH_RECEIPTS_ALL      acr  ON acr.CASH_RECEIPT_ID = ara.CASH_RECEIPT_ID
WHERE   dha.ORDER_NUMBER   = :P_ORDER_NUMBER
AND     dha.SUBMITTED_FLAG = 'Y'
AND     dha.STATUS_CODE   <> 'DOO_REFERENCE'
ORDER BY dla.LINE_NUMBER, dfla.FULFILL_LINE_NUMBER;
```

**Story result (simplified):**

| fulfill | qty | delivery | shipped | invoice | invoiced | balance | receipt | paid |
|---|---|---|---|---|---|---|---|---|
| 1 | 30 | DEL-501 | 30 | INV-9001 | 21L | 9.78L | AR-701 | 15L |
| 2 | 20 | (null) | (null) | INV-9001 ⚠️ | 21L ⚠️ | 9.78L | AR-701 | 15L |

**Explaining it in an interview:**
1. **Start from the order**: INNER JOINs inside DOO (it must exist), **LEFT JOINs** for everything that happens later.
2. **Filters in ON clauses** of LEFT JOINs (`RELEASED_STATUS`, `LINE_TYPE`, `STATUS`, `DISPLAY`), so unshipped or unpaid rows aren't dropped.
3. **⚠️ Notice row 2:** fulfill line 2 wasn't shipped, yet it shows INV-9001. That's because the invoice join is at **order line** level (the attributes carry the line number, not the fulfill line). Both fulfill lines share line 1, so both "see" the invoice. **This is fan-out.** In a real report, either aggregate shipping and invoicing separately, or link invoices at fulfill-line level if your AutoInvoice attributes carry the fulfill line ID. Spotting and explaining this impresses interviewers.
4. **Never SUM amounts** from this query. Multiple receipts × multiple fulfill lines multiply rows. See UC-13 for the right way.

---

#### UC-12: Shipped but not invoiced (revenue leakage check)
**Business question:** "Month-end: what did we ship that hasn't been billed yet?"
**Tables:** fulfill lines + `NOT EXISTS` on RA lines

```sql
SELECT  dha.ORDER_NUMBER,
        dla.DISPLAY_LINE_NUMBER,
        dfla.FULFILL_LINE_NUMBER,
        dfla.SHIPPED_QTY,
        dfla.ACTUAL_SHIP_DATE,
        dfla.STATUS_CODE
FROM    DOO_HEADERS_ALL       dha
JOIN    DOO_LINES_ALL         dla  ON dla.HEADER_ID = dha.HEADER_ID
JOIN    DOO_FULFILL_LINES_ALL dfla ON dfla.LINE_ID  = dla.LINE_ID
WHERE   dha.SUBMITTED_FLAG = 'Y'
AND     dha.STATUS_CODE   <> 'DOO_REFERENCE'
AND     NVL(dfla.SHIPPED_QTY,0) > 0
AND     NOT EXISTS (
          SELECT 1
          FROM   RA_CUSTOMER_TRX_LINES_ALL rctl
          WHERE  rctl.INTERFACE_LINE_ATTRIBUTE1 = TO_CHAR(dla.HEADER_ID)
          AND    rctl.INTERFACE_LINE_ATTRIBUTE2 = TO_CHAR(dla.LINE_NUMBER)
          AND    rctl.LINE_TYPE = 'LINE');
```
**Why NOT EXISTS instead of LEFT JOIN … IS NULL:** Both work. `NOT EXISTS` reads like the business question ("no invoice line exists") and **can't fan out**.
**Why it matters:** Shipped but unbilled = **revenue we earned but haven't claimed**. It's the mirror of P2P's "received not invoiced".

---

#### UC-13: Customer statement — total billed, received, outstanding (fan-out done right)
**Business question:** "Per customer: total invoiced, total cash applied, and current balance."
**Technique:** **Aggregate each module separately, then join the totals.** This is the fix for the fan-out problem in UC-11 (and P2P UC-13).

```sql
WITH billed AS (                         -- one row per customer
  SELECT  aps.CUSTOMER_ID,
          SUM(aps.AMOUNT_DUE_ORIGINAL)  AS total_billed,
          SUM(aps.AMOUNT_DUE_REMAINING) AS balance
  FROM    AR_PAYMENT_SCHEDULES_ALL aps
  WHERE   aps.CLASS IN ('INV','DM','CM')
  GROUP BY aps.CUSTOMER_ID
),
received AS (                            -- one row per customer
  SELECT  acr.PAY_FROM_CUSTOMER          AS CUSTOMER_ID,
          SUM(ara.AMOUNT_APPLIED)        AS total_applied
  FROM    AR_CASH_RECEIPTS_ALL           acr
  JOIN    AR_RECEIVABLE_APPLICATIONS_ALL ara ON ara.CASH_RECEIPT_ID = acr.CASH_RECEIPT_ID
  WHERE   ara.STATUS  = 'APP'
  AND     ara.DISPLAY = 'Y'
  AND     acr.REVERSAL_DATE IS NULL
  GROUP BY acr.PAY_FROM_CUSTOMER
)
SELECT  hp.PARTY_NAME,
        hca.ACCOUNT_NUMBER,
        NVL(b.total_billed,0)   AS total_billed,
        NVL(r.total_applied,0)  AS total_received,
        NVL(b.balance,0)        AS outstanding
FROM    HZ_CUST_ACCOUNTS hca
JOIN    HZ_PARTIES       hp ON hp.PARTY_ID = hca.PARTY_ID
LEFT JOIN billed         b  ON b.CUSTOMER_ID = hca.CUST_ACCOUNT_ID
LEFT JOIN received       r  ON r.CUSTOMER_ID = hca.CUST_ACCOUNT_ID
WHERE   hca.STATUS = 'A'
ORDER BY outstanding DESC;
```
**Story result:** XYZ Retail → billed 24.78L, received 15L, outstanding 9.78L. ✅ Billed − received = outstanding.
**Interview line:** *"Each CTE returns one row per customer, so the final joins are 1-to-1 and nothing gets multiplied."*

---

## Part 7 — Traps interviewers love

| # | Trap | What goes wrong | The fix |
|---|------|-----------------|---------|
| 1 | DOO order versions | Same order shows 2–3 times after a change order | `SUBMITTED_FLAG='Y'` and exclude `DOO_REFERENCE` |
| 2 | Customer name from `HZ_CUST_ACCOUNTS` | `ACCOUNT_NAME` isn't the legal name | Name = `HZ_PARTIES.PARTY_NAME` |
| 3 | Joining invoice to HZ on `PARTY_ID` | RA stores the **account** ID | `BILL_TO_CUSTOMER_ID = CUST_ACCOUNT_ID` |
| 4 | `BILL_TO_SITE_USE_ID = PARTY_SITE_ID` | Wrong table, wrong level | Site use ID → `HZ_CUST_SITE_USES_ALL.SITE_USE_ID` |
| 5 | `SITE_USE_TYPE` vs `SITE_USE_CODE` | Column not found | TYPE = party side, CODE = account side |
| 6 | Thinking `HZ_CUST_SITE_USES_ALL.LOCATION` is the address | It's just a short name | Address = `HZ_LOCATIONS` via party site |
| 7 | Forgetting `LINE_TYPE='LINE'` on RA lines | Tax/freight lines added in or fail to match | Filter LINE, link tax via `LINK_TO_CUST_TRX_LINE_ID` |
| 8 | Interface attribute join without `TO_CHAR` | Implicit conversion errors / slow | `= TO_CHAR(dla.HEADER_ID)` |
| 9 | `AR_PAYMENT_SCHEDULES_ALL` without CLASS filter | Receipts mixed with invoices | `CLASS IN ('INV','CM','DM')` or `'PMT'` |
| 10 | Applications joined on `PAYMENT_SCHEDULE_ID` | You get the receipt's row, not the invoice's | Use `APPLIED_PAYMENT_SCHEDULE_ID` |
| 11 | Applications without `STATUS='APP'` / `DISPLAY='Y'` | Unapplied / reversed amounts counted | Filter both |
| 12 | Summing all `WSH_DELIVERY_DETAILS` | Released/backordered counted as shipped | `RELEASED_STATUS = 'C'` for shipped |
| 13 | INNER JOIN to `WSH_DELIVERY_ASSIGNMENTS`→deliveries | Unassigned details vanish | LEFT JOIN (DELIVERY_ID can be NULL) |
| 14 | Credit memos filtered with `> 0` | Negative CMs dropped | Use `<> 0` |
| 15 | Joining order + shipments + invoices + receipts then SUM | Fan-out multiplies amounts | Aggregate separately (CTEs), join totals |
| 16 | `_ALL` tables without BU filter | Mixed business units | Filter `ORG_ID` / BU parameter |

---

## Part 8 — Self-test

**Q1.** What's the difference between a Party and a Customer Account? Which one does an invoice store?
<details><summary>Answer</summary>

Party = the identity (person or organization, stored once). Account = a business relationship with us (one party can have many). Invoices store the **account** (`BILL_TO_CUSTOMER_ID = CUST_ACCOUNT_ID`). The name comes from the party.
</details>

**Q2.** Write the chain from `RA_CUSTOMER_TRX_ALL` to the bill-to city.
<details><summary>Answer</summary>

`rct.BILL_TO_SITE_USE_ID` → `HZ_CUST_SITE_USES_ALL.SITE_USE_ID` → `.CUST_ACCT_SITE_ID` → `HZ_CUST_ACCT_SITES_ALL` → `.PARTY_SITE_ID` → `HZ_PARTY_SITES` → `.LOCATION_ID` → `HZ_LOCATIONS.CITY`. "Up A Pretty Ladder."
</details>

**Q3.** Which DOO table is the O2C equivalent of `PO_LINE_LOCATIONS_ALL`, and why?
<details><summary>Answer</summary>

`DOO_FULFILL_LINES_ALL`. Like a PO schedule, it splits a line into pieces by **when/where** to ship, and it tracks progress (`SHIPPED_QTY`).
</details>

**Q4.** A report shows sales order SO-2001 twice with different statuses. What's the likely cause and fix?
<details><summary>Answer</summary>

Order versioning: a change order left a reference copy. Filter `SUBMITTED_FLAG = 'Y'` and `STATUS_CODE <> 'DOO_REFERENCE'`.
</details>

**Q5.** What do `RELEASED_STATUS` values S, Y, C and B mean?
<details><summary>Answer</summary>

S = released to warehouse, Y = staged/pick confirmed, C = shipped, B = backordered.
</details>

**Q6.** `AR_RECEIVABLE_APPLICATIONS_ALL` has `PAYMENT_SCHEDULE_ID` and `APPLIED_PAYMENT_SCHEDULE_ID`. Which one do you join to find the invoice?
<details><summary>Answer</summary>

`APPLIED_PAYMENT_SCHEDULE_ID` ("applied **to**"). `PAYMENT_SCHEDULE_ID` is the receipt's own schedule row.
</details>

**Q7.** Why does the RA → DOO join need `TO_CHAR`?
<details><summary>Answer</summary>

`INTERFACE_LINE_ATTRIBUTE1..15` are generic VARCHAR2 columns used by AutoInvoice for any source; DOO IDs are NUMBERs.
</details>

**Q8.** How do you find open invoices only (not receipts) in `AR_PAYMENT_SCHEDULES_ALL`?
<details><summary>Answer</summary>

`CLASS = 'INV'` (add CM/DM for full balance) and `STATUS = 'OP'` / `AMOUNT_DUE_REMAINING <> 0`.
</details>

**Q9.** Write a query: list all unapplied receipts (money received but not matched to any invoice).
<details><summary>Answer</summary>

```sql
SELECT  acr.RECEIPT_NUMBER, acr.RECEIPT_DATE, acr.AMOUNT
FROM    AR_CASH_RECEIPTS_ALL acr
WHERE   acr.TYPE = 'CASH'
AND     acr.REVERSAL_DATE IS NULL
AND     NOT EXISTS (SELECT 1
                    FROM   AR_RECEIVABLE_APPLICATIONS_ALL ara
                    WHERE  ara.CASH_RECEIPT_ID = acr.CASH_RECEIPT_ID
                    AND    ara.STATUS  = 'APP'
                    AND    ara.DISPLAY = 'Y');
```
(Shortcut: `acr.STATUS = 'UNAPP'` catches fully unapplied ones. The NOT EXISTS version shows you understand the tables.)
</details>

**Q10.** Map these to their P2P twins: `AR_CASH_RECEIPTS_ALL`, `AR_RECEIVABLE_APPLICATIONS_ALL`, `WSH_DELIVERY_DETAILS`, `RA_CUSTOMER_TRX_ALL`.
<details><summary>Answer</summary>

`AP_CHECKS_ALL`, `AP_INVOICE_PAYMENTS_ALL`, `RCV_SHIPMENT_LINES` / `RCV_TRANSACTIONS`, `AP_INVOICES_ALL`.
</details>

**Q11.** Interviewer: "In the grand trace query, an unshipped fulfill line shows an invoice number. Is the query wrong?"
<details><summary>Answer</summary>

It's fan-out. The invoice is linked at **order line** level, and both fulfillment lines belong to the same order line, so both rows pick up the invoice. Fix it by aggregating shipping and invoicing separately, or by linking at fulfill-line level if AutoInvoice carries the fulfill line ID.
</details>

**Q12.** Give the GL entries for shipment, invoice and receipt in O2C.
<details><summary>Answer</summary>

Ship: Dr COGS / Cr Inventory. Invoice: Dr Receivable / Cr Revenue (+Tax). Receipt: Dr Cash / Cr Receivable.
</details>

---

## Part 9 — One-page cheat sheet

```
PROCESS:   Customer → Order → Fulfill/Schedule → Pick Release → Pick Confirm → Ship → AutoInvoice → Receipt → Apply → GL

MIRROR:    O2C = P2P flipped
           PO_HEADERS/LINES/LINE_LOCATIONS  ↔  DOO_HEADERS/LINES/FULFILL_LINES
           RCV_* (in)                       ↔  WSH_* (out)
           AP_INVOICES / LINES              ↔  RA_CUSTOMER_TRX / LINES
           AP_PAYMENT_SCHEDULES             ↔  AR_PAYMENT_SCHEDULES
           AP_CHECKS / AP_INVOICE_PAYMENTS  ↔  AR_CASH_RECEIPTS / AR_RECEIVABLE_APPLICATIONS

CUSTOMER   Party = person/identity      Account = bank account (relationship)
  (HZ)     HZ_PARTIES ─PARTY_ID─ HZ_CUST_ACCOUNTS          (name = PARTY_NAME)
           HZ_PARTIES ─PARTY_ID─ HZ_PARTY_SITES ─LOCATION_ID─ HZ_LOCATIONS (address)
           HZ_PARTY_SITES ─PARTY_SITE_ID─ HZ_PARTY_SITE_USES      (SITE_USE_TYPE)
           HZ_PARTY_SITES ─PARTY_SITE_ID─ HZ_CUST_ACCT_SITES_ALL
           HZ_CUST_ACCT_SITES_ALL ─CUST_ACCT_SITE_ID─ HZ_CUST_SITE_USES_ALL (SITE_USE_CODE BILL_TO/SHIP_TO)
           Address chain: U → A → P → L  "Up A Pretty Ladder"

ORDER      DOO_HEADERS_ALL  HEADER_ID     ORDER_NUMBER, SOLD_TO_PARTY_ID; SUBMITTED_FLAG='Y', ≠ DOO_REFERENCE
  (DOO)    DOO_LINES_ALL    LINE_ID       what customer asked; ORDERED_QTY, UNIT_SELLING_PRICE
           DOO_FULFILL_LINES_ALL FULFILL_LINE_ID  how we ship; SHIPPED_QTY, FULFILL_ORG_ID, STATUS

SHIPPING   Parcel → Label → Van
  (WSH)    WSH_DELIVERY_DETAILS  ─DELIVERY_DETAIL_ID─ WSH_DELIVERY_ASSIGNMENTS ─DELIVERY_ID─ WSH_NEW_DELIVERIES
           RELEASED_STATUS: N R S Y C B D X   (C = shipped, B = backordered)
           → DOO: SOURCE_HEADER_ID = HEADER_ID; + SALES_ORDER_LINE_NUMBER / SALES_ORDER_SHIPMENT_NUMBER

INVOICE    RA_CUSTOMER_TRX_ALL ─CUSTOMER_TRX_ID─ RA_CUSTOMER_TRX_LINES_ALL (LINE_TYPE='LINE')
  (RA)     BILL_TO_CUSTOMER_ID → HZ_CUST_ACCOUNTS     BILL_TO_SITE_USE_ID → HZ_CUST_SITE_USES_ALL
           → DOO: INTERFACE_LINE_ATTRIBUTE1 = TO_CHAR(HEADER_ID), ATTRIBUTE2 = TO_CHAR(LINE_NUMBER)

CASH       AR_PAYMENT_SCHEDULES_ALL   PAYMENT_SCHEDULE_ID; CLASS INV/CM/DM/PMT; AMOUNT_DUE_REMAINING; STATUS OP/CL
  (AR)     AR_CASH_RECEIPTS_ALL       CASH_RECEIPT_ID; PAY_FROM_CUSTOMER; STATUS APP/UNAPP/UNID/REV
           AR_RECEIVABLE_APPLICATIONS_ALL  CASH_RECEIPT_ID; APPLIED_PAYMENT_SCHEDULE_ID ("applied TO")
                                           STATUS='APP', DISPLAY='Y'

GL         Ship: Dr COGS / Cr Inventory   Invoice: Dr AR / Cr Revenue+Tax   Receipt: Dr Cash / Cr AR
           JE_SOURCE = 'Receivables' (invoice, receipt), 'Cost Management' (COGS)

GOLDEN RULES
  1. Current order version only          6. RA lines: LINE_TYPE = 'LINE'
  2. Name from PARTY, IDs from ACCOUNT    7. AR PS: always filter CLASS
  3. SITE_USE_ID ≠ PARTY_SITE_ID          8. Apps: APPLIED_PS_ID, APP, DISPLAY='Y'
  4. Shipped = RELEASED_STATUS 'C'        9. LEFT JOIN later steps; filters in ON
  5. TO_CHAR on interface attributes     10. Aggregate per module, then join totals
```

---

*With both P2P and O2C done, a good next step is a **combined "inventory in vs out" query**: purchases (P2P +qty) and sales (O2C −qty) from `INV_MATERIAL_TXNS` for one item, reconciled with on-hand. It ties both cycles together and is a strong interview talking point.*