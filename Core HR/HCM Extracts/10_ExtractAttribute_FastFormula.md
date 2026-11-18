# HCM Extract Attribute Types & Fast Formula — Study Guide

> **How to read this guide:** Part A covers extract attribute types and when to choose each one. Part B covers Fast Formula from zero, with examples and edge cases. Part C answers the key question: *"How do I know what inputs my formula gets and what it must return?"*
>
> **Note on names:** Database item (DBI) names in the examples, like `PER_ASG_NORMAL_HOURS`, are **illustrative**. Always search the exact DBI name in your pod: in the Fast Formula editor's Database Items list, or in the user entity used by your data group. Labels can also differ slightly between the classic **Define Extracts** page and the newer **Extract Definitions** page. For example, "Rule" may appear as "Fast Formula".

---

# PART A — Extract Attribute Types

## A1. The six attribute types at a glance

| # | Attribute Type | Where the value comes from | One-line memory hook |
|---|---|---|---|
| 1 | **Database Item Group** | A DBI from the data group's User Entity | "Take the field directly from HCM" |
| 2 | **String** | A fixed constant you type | "Same value for every row" |
| 3 | **Parameter** | A runtime parameter value | "Value given when the extract is submitted" |
| 4 | **Record Calculation** | An expression using other attributes of the **same record** | "Calculate within one row" |
| 5 | **Summary Calculation** | An aggregate (COUNT, SUM…) across **records/rows** | "Calculate across many rows" |
| 6 | **Rule** | An **Extract Rule Fast Formula** | "Custom logic in code" |

---

## A2. Each type explained with an example

### 1. Database Item Group (DBI)
**What:** Outputs the value of a DBI that belongs to the data group's User Entity.

**Significance:**
- **Fastest and simplest.** No custom code, and the value comes straight from Oracle's secured route.
- **Date-effective automatically.** It returns the value as of the extract's effective date.
- **First choice** whenever the field already exists as a DBI.

**Example:** In the Person data group, the attribute `PERSON_NUMBER` uses the Person Number DBI.
```xml
<PERSON_NUMBER>1001</PERSON_NUMBER>
```

**Choose it when:** the value you need already exists as a DBI in the data group's user entity.

---

### 2. String
**What:** A fixed text value that is the same for every row.

**Significance:**
- Used for **record identifiers, file codes and constant flags** that the receiving system expects.
- No data lookup, so it costs no processing.

**Examples:**
- Record type indicator: `D` for detail rows, `H` for header, `T` for trailer
- Source system code: `ORACLE_HCM`
- Vendor client ID: `ACME001`

```xml
<RECORD_TYPE>D</RECORD_TYPE>
```

**Choose it when:** the value never changes from row to row.

---

### 3. Parameter
**What:** Outputs the value of an extract parameter passed at submission.

**Significance:**
- Lets the **same extract** print run-specific information without code changes.
- Common in **header records**.

**Example:** A header record shows the run's effective date.
```xml
<FILE_DATE>2026-11-06</FILE_DATE>
```

**Choose it when:** the value depends on the run, not on the employee. Examples are the effective date, a run type, or the legal employer chosen at submission.

---

### 4. Record Calculation
**What:** An expression that combines or calculates using **other attributes in the same record**.

**Significance:**
- Lets you derive values **without writing a fast formula**.
- It only sees attributes of the **same record**. It cannot look up new data.

**Examples:**
- Full name = First Name `||` ' ' `||` Last Name
- Annual salary = Monthly salary × 12

**Choose it when:** the result can be built only from attributes already present in the same record.

---

### 5. Summary Calculation
**What:** An aggregate across rows: COUNT, SUM, MIN, MAX of an attribute from another record or data group.

**Significance:**
- Used mostly in **trailer/footer records** that receiving systems use to check the file. They compare row counts and totals.

**Examples:**
- Trailer record: `TOTAL_RECORDS` = COUNT of `PERSON_NUMBER`
- Payroll interface trailer: `TOTAL_AMOUNT` = SUM of `PAY_AMOUNT`

```xml
<TOTAL_RECORDS>2450</TOTAL_RECORDS>
```

**Choose it when:** you need totals or counts over all extracted rows.

> Counts and totals can also be done in the **eText template** with `COUNT()` / `SUM()`. Use a Summary Calculation when the total must exist in the XML itself, for example when several templates or deliveries use it.

---

### 6. Rule (Extract Rule Fast Formula)
**What:** The value is returned by a fast formula of type **Extract Rule**.

**Significance:**
- **Most flexible.** It can use IF/ELSE logic, read DBIs that are not in the data group's user entity (if the contexts allow), call functions, use lookups and do date maths.
- **Most expensive.** It runs once for every row, so heavy formulas slow down large extracts.
- Needs to be **compiled** and maintained.

**Examples:**
- Employment category: return `FT` if weekly hours ≥ 40, else `PT`
- Years of service from the hire date
- Map an Oracle code to a vendor code (Oracle `ACTIVE` → vendor `A`)

**Choose it when:** none of the other five types can produce the value.

---

## A3. Decision flow — which attribute type to choose?

```
Is the value the same for every row?             ──Yes──► STRING
            │No
Is it a runtime value given at submission?       ──Yes──► PARAMETER
            │No
Does a DBI give exactly this value?              ──Yes──► DATABASE ITEM
            │No
Can it be built from attributes in the same record? ─Yes─► RECORD CALCULATION
            │No
Is it a count/sum/min/max across rows?           ──Yes──► SUMMARY CALCULATION
            │No
                                                         ► RULE (Fast Formula)
```

**Rule of thumb:** *Cheapest option first. Fast formula last.*

---

## A4. For which attribute types do we write a Fast Formula?

| Attribute type | Do *you* write a Fast Formula? |
|---|---|
| Database Item Group | ❌ No |
| String | ❌ No |
| Parameter | ❌ No |
| Record Calculation | ❌ No. You write an **expression**; the extract engine compiles it internally |
| Summary Calculation | ❌ No. You choose an aggregate function and attribute |
| **Rule** | ✅ **Yes — Extract Rule formula type** |

**Other places in HCM Extracts where you write a fast formula, outside attributes:**
| Formula type | Used for |
|---|---|
| **Extract Criteria** | Data group filtering logic too complex for simple criteria |
| **Extract Record** | Record-level processing (rarely needed) |

**Behind the scenes:** Oracle also **auto-generates** formulas for each record when you validate or compile the extract. This is why the Extract Execution Tree shows a formula compile status. You do not write those yourself.

### Interview answer
> "We write a fast formula only for attributes of type Rule, using the Extract Rule formula type. Database item, string and parameter attributes need no code. Record and summary calculations use expressions or aggregate functions that the engine compiles internally. Outside attributes, we can also use Extract Criteria formulas for complex filtering. I choose the cheapest type first: string, parameter, DBI, then calculations, and a fast formula only when nothing else works, because it runs for every row."

---

# PART B — Fast Formula from Zero

## B1. What is Fast Formula?

**Fast Formula is Oracle HCM's own simple programming language for writing business rules without changing Oracle's code.** The calling process runs the formula. That process can be an extract, payroll, absence, a validation, or person number generation. The process provides the context, the formula runs, and it returns named outputs.

**Key idea — the formula type is a contract:**
```
Calling process ──(contexts + inputs)──► YOUR FORMULA ──(named outputs)──► Calling process
```
The **formula type** decides:
1. Which **contexts** are available (for example PERSON_ID, EFFECTIVE_DATE)
2. Which **DBIs** you can use (only those whose contexts are available)
3. Which **inputs** are passed, if any
4. Which **output names** the calling process reads

---

## B2. Data types

| Type | Example value | Notes |
|---|---|---|
| **NUMBER** | `40`, `12.5` | Default type of undeclared inputs |
| **TEXT** | `'FT'` | Always in **single quotes** |
| **DATE** | `'2026/11/06 00:00:00' (date)` | Date literal format with the `(date)` suffix |

**Array types** (value type first, index type second):
`NUMBER_NUMBER`, `TEXT_NUMBER`, `DATE_NUMBER`, `NUMBER_TEXT`, `TEXT_TEXT`, `DATE_TEXT`.
For example, `TEXT_NUMBER` is an array of text values indexed by number.

---

## B3. Kinds of variables

| Kind | Meaning | Can you change it? | Example |
|---|---|---|---|
| **Local variable** | Created inside the formula | Yes | `l_hours = 40` |
| **Input variable** | Passed by the calling process | Read (can reassign locally) | `INPUTS ARE p_amount` |
| **Database Item** | Value fetched from the database | ❌ Read-only | `PER_ASG_NORMAL_HOURS` |
| **Context** | Run information (person, date…) | Read with `GET_CONTEXT`; change temporarily with `CHANGE_CONTEXTS` | `PERSON_ID` |
| **Global value** | Constant defined once for many formulas | ❌ Read-only | a company-wide rate |
| **Output variable** | Returned at the end | — | `RETURN RULE_VALUE` |

**Rules about local variables:**
- Type is set by the **first assignment** and cannot change later.
- Must be **assigned before use**. Initialise them at the top of the formula.

---

## B4. Formula structure (always in this order)

```
/* 1. Header comment: name, type, purpose, author, date */

/* 2. ALIAS (optional)  - short name for a long DBI */
ALIAS PER_ASG_NORMAL_HOURS AS l_std_hours

/* 3. DEFAULT FOR - default values for DBIs and inputs */
DEFAULT FOR PER_ASG_NORMAL_HOURS IS 0

/* 4. INPUTS ARE - only if the formula type passes inputs */
INPUTS ARE p_value (TEXT)

/* 5. Logic - assignments, IF, loops, function calls */
l_result = ' '

/* 6. RETURN - named outputs expected by the formula type */
RETURN l_result
```

**Syntax essentials:**
- Comments: `/* ... */`
- No semicolons at line ends
- Text joining uses `||`
- Group multiple statements in IF or loops with **parentheses `( )`**
- Compare with `=`, `<>` (or `!=`), `>`, `<`, `>=`, `<=`, `LIKE`, `NOT LIKE`
- Combine conditions with `AND`, `OR`, `NOT`

---

## B5. Key statements

### IF / ELSE
```
IF l_hours >= 40 THEN
   l_cat = 'FT'
ELSE IF l_hours >= 20 THEN
   l_cat = 'PT'
ELSE
   l_cat = 'CASUAL'
```

Several statements inside one branch need parentheses:
```
IF l_status = 'ACTIVE' THEN
(
   l_flag  = 'A'
   l_count = l_count + 1
)
```

### WAS DEFAULTED
This tells you whether a DBI had no data, so the default value was used.
```
IF PER_ASG_NORMAL_HOURS WAS DEFAULTED THEN
   l_hours_text = 'MISSING'
```

### WHILE loop
```
i = 1
WHILE i <= 5 LOOP
(
   l_total = l_total + i
   i = i + 1
)
```

### CHANGE_CONTEXTS
Temporarily changes a context, such as the date, so DBIs are read as of another date.
```
CHANGE_CONTEXTS(EFFECTIVE_DATE = l_prev_month_date)
(
   l_old_hours = PER_ASG_NORMAL_HOURS
)
```

### GET_CONTEXT
Reads a context value, with a default if the context is not set.
```
l_person_id = GET_CONTEXT(PERSON_ID, 0)
l_eff_date  = GET_CONTEXT(EFFECTIVE_DATE, '4712/12/31 00:00:00' (date))
```

---

## B6. Common functions (most used)

| Category | Functions |
|---|---|
| Text | `UPPER`, `LOWER`, `INITCAP`, `SUBSTR`, `INSTR`, `LENGTH`, `LPAD`, `RPAD`, `LTRIM`, `RTRIM`, `REPLACE`, `||` |
| Number | `ROUND`, `TRUNC`, `FLOOR`, `ABS`, `GREATEST`, `LEAST` |
| Date | `ADD_DAYS`, `ADD_MONTHS`, `ADD_YEARS`, `DAYS_BETWEEN`, `MONTHS_BETWEEN`, `LAST_DAY` |
| Conversion | `TO_CHAR`, `TO_NUMBER`, `TO_DATE`, `TO_TEXT` |
| Null check | `ISNULL` (returns `'Y'` / `'N'`) |
| Context | `GET_CONTEXT`, `CHANGE_CONTEXTS` |
| Lookups / tables | `GET_LOOKUP_MEANING`, `GET_TABLE_VALUE` (user-defined tables) |
| Calling other formulas | `CALL_FORMULA`, or `SET_INPUT` + `EXECUTE` + `GET_OUTPUT` |
| Logging | `PAY_INTERNAL_LOG_WRITE` (writes to the process log when logging is enabled) |

---

## B7. Examples (Extract Rule type, from simple to advanced)

> Every Extract Rule formula must **return `RULE_VALUE`**. Make its data type match the attribute's **Data Type** (Text, Number or Date). Many projects return **text** and set the attribute to Text, then format the value in the template.

### Example 1 — Simple text logic: Full-time or Part-time
```
/******************************************************
 Name    : XX_EXT_EMP_CATEGORY
 Type    : Extract Rule
 Purpose : Return FT if weekly hours >= 40 else PT
******************************************************/
DEFAULT FOR PER_ASG_NORMAL_HOURS IS 0

RULE_VALUE = 'PT'                        /* initialise output first */
l_hours    = PER_ASG_NORMAL_HOURS

IF l_hours >= 40 THEN
   RULE_VALUE = 'FT'

RETURN RULE_VALUE
```
**Edge cases handled:** missing hours default to 0, which gives `PT`. RULE_VALUE is always set.
**Edge case NOT handled:** hours might be stored per day or per month rather than per week. Check the frequency DBI before comparing.

---

### Example 2 — Detecting missing data with WAS DEFAULTED
```
/* Name: XX_EXT_EMP_CATEGORY_V2   Type: Extract Rule */
DEFAULT FOR PER_ASG_NORMAL_HOURS IS 0

RULE_VALUE = ' '

IF PER_ASG_NORMAL_HOURS WAS DEFAULTED THEN
   RULE_VALUE = 'UNKNOWN'
ELSE IF PER_ASG_NORMAL_HOURS >= 40 THEN
   RULE_VALUE = 'FT'
ELSE
   RULE_VALUE = 'PT'

RETURN RULE_VALUE
```
**Why better:** it separates "no data" from "real value of 0". The vendor gets `UNKNOWN` instead of wrongly getting `PT`.

---

### Example 3 — Date maths: Years of service
```
/* Name: XX_EXT_YEARS_OF_SERVICE   Type: Extract Rule */
DEFAULT FOR PER_PERSON_ENTERPRISE_HIRE_DATE IS '4712/12/31 00:00:00' (date)

RULE_VALUE = '0'
l_eff_date  = GET_CONTEXT(EFFECTIVE_DATE, '4712/12/31 00:00:00' (date))
l_hire_date = PER_PERSON_ENTERPRISE_HIRE_DATE

IF l_hire_date WAS DEFAULTED THEN
   RULE_VALUE = '0'
ELSE IF l_hire_date > l_eff_date THEN
   RULE_VALUE = '0'                       /* future hire: no service yet */
ELSE
(
   l_years    = TRUNC(MONTHS_BETWEEN(l_eff_date, l_hire_date) / 12)
   RULE_VALUE = TO_CHAR(l_years)
)

RETURN RULE_VALUE
```
**Edge cases handled:** missing hire date, future-dated hire, and partial years (`TRUNC` gives completed years only).

**Note:** `WAS DEFAULTED` is checked on the DBI, which is the correct way. Comparing to the default date also works, but it is less clear.

---

### Example 4 — Date output: Probation end date (hire date + 6 months)
```
/* Name: XX_EXT_PROBATION_END   Type: Extract Rule */
DEFAULT FOR PER_PERSON_ENTERPRISE_HIRE_DATE IS '4712/12/31 00:00:00' (date)

RULE_VALUE = ' '

IF PER_PERSON_ENTERPRISE_HIRE_DATE WAS NOT DEFAULTED THEN
   RULE_VALUE = TO_CHAR(ADD_MONTHS(PER_PERSON_ENTERPRISE_HIRE_DATE, 6), 'YYYY-MM-DD')

RETURN RULE_VALUE
```
**Edge case:** `ADD_MONTHS` on the 31st. For example, 31-Aug + 6 months gives the **last day of February** (28 or 29). Confirm that this matches the business rule.

---

### Example 5 — Mapping Oracle values to vendor codes
```
/* Name: XX_EXT_STATUS_MAP   Type: Extract Rule */
DEFAULT FOR PER_ASG_STATUS_TYPE IS 'NONE'

RULE_VALUE = 'X'                          /* safe fallback for unexpected values */
l_status   = UPPER(LTRIM(RTRIM(PER_ASG_STATUS_TYPE)))

IF l_status = 'ACTIVE' THEN
   RULE_VALUE = 'A'
ELSE IF l_status = 'SUSPENDED' THEN
   RULE_VALUE = 'S'
ELSE IF l_status = 'INACTIVE' THEN
   RULE_VALUE = 'T'

RETURN RULE_VALUE
```
**Edge cases handled:** text comparison is **case-sensitive**, so the value is converted with `UPPER`. Spaces are trimmed. Any new status code that Oracle or the customer adds later falls back to `X`, so the formula never fails.

---

### Example 6 — Value as of another date (CHANGE_CONTEXTS)
Gets the hours as of the previous month end, to report a change.
```
/* Name: XX_EXT_PREV_HOURS   Type: Extract Rule */
DEFAULT FOR PER_ASG_NORMAL_HOURS IS 0

RULE_VALUE = '0'
l_eff_date  = GET_CONTEXT(EFFECTIVE_DATE, '4712/12/31 00:00:00' (date))
l_prev_date = LAST_DAY(ADD_MONTHS(l_eff_date, -1))

CHANGE_CONTEXTS(EFFECTIVE_DATE = l_prev_date)
(
   l_prev_hours = PER_ASG_NORMAL_HOURS
)

RULE_VALUE = TO_CHAR(l_prev_hours)
RETURN RULE_VALUE
```
**Edge cases:** the assignment may not have existed last month. The default 0 handles that, but consider `WAS DEFAULTED` if 0 is a valid value. Also, the context change applies **only inside the parentheses**.

---

### Example 7 — Looping an array DBI (multiple rows)
Some DBIs return **many rows**, for example all phone numbers. These are array DBIs.
```
/* Name: XX_EXT_ALL_PHONES   Type: Extract Rule */
DEFAULT_DATA_VALUE FOR PER_PHONE_NUMBERS_ARR IS ' '   /* array default */

RULE_VALUE = ' '
l_list = ' '
i = PER_PHONE_NUMBERS_ARR.FIRST(-1)                  /* -1 if array is empty */

WHILE PER_PHONE_NUMBERS_ARR.EXISTS(i) LOOP
(
   IF l_list = ' ' THEN
      l_list = PER_PHONE_NUMBERS_ARR[i]
   ELSE
      l_list = l_list || ';' || PER_PHONE_NUMBERS_ARR[i]

   i = PER_PHONE_NUMBERS_ARR.NEXT(i, -1)             /* move forward, else -1 */
)

RULE_VALUE = l_list
RETURN RULE_VALUE
```
**Edge cases handled:** an empty array (the loop never runs) and building a delimited list.
**Danger:** if you forget `i = ...NEXT(...)`, the loop **never ends**.

---

### Example 8 — A different formula type, to show that outputs depend on the type
**Element Input Validation** formula type. It does not return `RULE_VALUE`. It returns `FORMULA_STATUS` and `FORMULA_MESSAGE`.
```
/* Name: XX_VALIDATE_BONUS   Type: Element Input Validation */
INPUTS ARE ENTRY_VALUE (NUMBER)

FORMULA_STATUS  = 'S'
FORMULA_MESSAGE = ' '

IF ENTRY_VALUE < 0 THEN
(
   FORMULA_STATUS  = 'E'
   FORMULA_MESSAGE = 'Bonus cannot be negative'
)
ELSE IF ENTRY_VALUE > 100000 THEN
(
   FORMULA_STATUS  = 'W'
   FORMULA_MESSAGE = 'Bonus is unusually high, please verify'
)

RETURN FORMULA_STATUS, FORMULA_MESSAGE
```
**Takeaway:** the **same language** is used with **different contracts**. The formula type decides the input names and output names.

---

## B8. Edge cases checklist (the interview favourite)

| # | Edge case | What goes wrong | How to handle |
|---|---|---|---|
| 1 | **DBI has no data** | Runtime error saying no data was found for the DBI | Add `DEFAULT FOR` on every DBI you use |
| 2 | **Default hides real meaning** | A default of 0 looks like a real 0 | Check `WAS DEFAULTED` and output something like `UNKNOWN` |
| 3 | **Text default of a space `' '`** | Output contains a blank space, so a fixed-width file shifts or validation fails | Trim, or choose a meaningful default |
| 4 | **DBI returns multiple rows** | "Too many rows" style error | Use an array DBI and loop, or the correct context or level |
| 5 | **Input not declared with type** | Undeclared inputs are **NUMBER** by default, so text inputs fail | `INPUTS ARE p_code (TEXT), p_dt (DATE)` |
| 6 | **Local variable not initialised in all branches** | Compile or runtime error, or an unset output | Initialise every variable and the output at the top |
| 7 | **Type mismatch** | Cannot compare or assign TEXT to NUMBER | Convert with `TO_CHAR`, `TO_NUMBER`, `TO_DATE` |
| 8 | **`TO_NUMBER` on non-numeric text** | Runtime conversion error | Validate first, for example with `ISNULL` or pattern checks |
| 9 | **Division by zero** | Runtime error | `IF l_den <> 0 THEN ...` |
| 10 | **Case-sensitive text compare** | `'Active' <> 'ACTIVE'` | `UPPER()` both sides; trim spaces |
| 11 | **Unexpected new code value** | No branch matches, so the output is blank | Final `ELSE` or an initial fallback value |
| 12 | **Future-dated records** | DBI shows the value as of the effective date only | Use `CHANGE_CONTEXTS` or the correct effective date |
| 13 | **End-of-time dates (4712/12/31)** | Shows as a real date in output | Treat as "no end date" in logic or output blank |
| 14 | **Month-end / leap year maths** | `ADD_MONTHS` from the 31st, or 29-Feb | Test boundary dates explicitly |
| 15 | **Context not available** | DBI cannot be used, or the formula fails at compile or run time | Use DBIs whose contexts the formula type supports; the data group level matters (person vs assignment) |
| 16 | **Infinite loop** | Process hangs or times out | Always move the index in a WHILE loop |
| 17 | **Formula effective date** | Formulas are **date-effective**. If the formula starts after the run's effective date, it is not found | Use an early start date, such as 01-01-1951 |
| 18 | **Changed formula not picked up** | Old logic still runs | Re-compile the formula, then re-validate the extract |
| 19 | **Output too long** | Truncated or misaligned in a fixed-width file | Use `SUBSTR` or `RPAD`/`LPAD` to the exact length (or do it in eText) |
| 20 | **Performance** | Heavy formula × 50,000 rows is slow | Prefer DBIs; avoid unnecessary `CHANGE_CONTEXTS` and loops |

---

# PART C — How Do I Know the Inputs, the Return Value and Its Data Type?

## C1. The core answer
> **You don't decide the inputs and outputs — the formula type does.**
> The calling process (extract, payroll, absence, validation) was built by Oracle to **pass specific contexts and inputs** and to **read specific output names**. Your formula must follow that contract.

## C2. The 5-step method

### Step 1 — Identify the formula type
Ask: *"Which process will call this formula?"*
| Requirement | Formula type |
|---|---|
| Extract attribute value | **Extract Rule** |
| Extract filtering logic | **Extract Criteria** |
| Validate an element entry | **Element Input Validation** |
| Payroll element calculation | **Oracle Payroll** |
| Absence accrual, eligibility and similar | Absence formula types |
| Custom person number generation | **Person Number Generation** |

### Step 2 — Check what the formula type provides
In **My Client Groups → Fast Formulas** (or Setup and Maintenance → Manage Fast Formulas), when you create or edit a formula of that type:
- **Contexts**: the contexts the calling process sets, such as PERSON_ID, HR_ASSIGNMENT_ID and EFFECTIVE_DATE
- **Database Items**: only DBIs whose contexts are available for that type can be used. The editor lets you search them.
- **Functions**: the list of functions you can call

For extracts, the data group level also matters. A Rule attribute in a **person-level** data group may not have an assignment context available. If so, assignment DBIs cannot be resolved there.

### Step 3 — Check Oracle's documentation for the formula type
Oracle documents each formula type: its contexts, its input variables (names and data types) and its **expected return variables**. Sources are:
- Oracle Help Center: *Administering Fast Formulas* and the product-specific guides
- My Oracle Support: the formula-type reference documents (for example for Payroll and Absence)
- Oracle's **predefined/sample formulas** of the same type, which are the best templates to copy

### Step 4 — Decide the inputs
| Situation | What to do |
|---|---|
| Formula type passes **no inputs** (common for Extract Rule) | No `INPUTS ARE`. Get data from **contexts + DBIs** |
| Formula type passes **named inputs** | Declare them **with the exact names** and data types in `INPUTS ARE` |
| **Payroll element formulas** | Input names match the element's **input value names** (for example `PAY_VALUE`, `HOURS`) |
| **Validation formulas** | The entered value is passed as an input (for example `ENTRY_VALUE`) |

Always declare a data type for non-number inputs:
```
INPUTS ARE p_code (TEXT), p_start (DATE), p_amount       /* p_amount = NUMBER by default */
```

### Step 5 — Decide the return name and data type
| Formula type | Return variable(s) | Data type rule |
|---|---|---|
| **Extract Rule** | `RULE_VALUE` | Match the attribute's Data Type (Text, Number, Date) |
| **Element Input Validation** | `FORMULA_STATUS`, `FORMULA_MESSAGE` | Text (`S`/`W`/`E`, message text) |
| **Oracle Payroll** | Names you map in the element's **formula result rules** | Match the target input value's type |
| **Person Number Generation** | `NEXT_NUMBER` | As documented |
| Others | As per that formula type's documentation | As documented |

**Golden rule:** if the return name is wrong, the formula may still **compile**, but the calling process **will not get the value**. It might return blank, error out or ignore the value. Always confirm the exact output name.

## C3. Quick self-check before you write a formula

```
1. What process calls me?              → Formula type
2. Which contexts do I get?            → Contexts list for the type
3. Which DBIs can I read?              → DBIs valid for those contexts
4. Do I receive inputs? Names/types?   → Type docs / element input values
5. What must I return? Name/type?      → Type docs / attribute data type / result rules
6. What if data is missing?            → DEFAULT FOR + WAS DEFAULTED
7. What if values are unexpected?      → Fallback value / final ELSE
```

## C4. Development workflow for an Extract Rule formula
1. **Create** the formula: type **Extract Rule**, effective start date early (for example 01-01-1951), and the correct LDG if one is needed.
2. **Write**: header comment → `DEFAULT FOR` → initialise `RULE_VALUE` → logic → `RETURN RULE_VALUE`.
3. **Compile**, then fix any errors.
4. **Attach** it to an attribute of type **Rule**, and set the attribute Data Type to match.
5. **Validate/compile the extract**.
6. **Test** with a small population and XML output. Include edge-case employees: missing data, future hire, terminated.
7. **Debug** using logs (for example `PAY_INTERNAL_LOG_WRITE` with logging enabled), or by temporarily returning intermediate values in `RULE_VALUE`.

---

## Interview Q&A

**Q1. When would you use a fast formula instead of a DBI attribute?**
> When the value needs logic, such as conditions, mapping, date calculations, or data from another date or context, that a DBI, string, parameter or calculation can't provide. Fast formulas run per row, so I use them only when needed.

**Q2. What does an Extract Rule formula return?**
> It returns `RULE_VALUE`, whose data type should match the attribute's data type.

**Q3. What happens if a DBI returns no data and there is no default?**
> The formula fails at runtime. We always use `DEFAULT FOR`, and `WAS DEFAULTED` when we need to tell missing data apart from real data.

**Q4. How do you know which inputs a formula receives?**
> The formula type defines the contract. I check the contexts and DBIs available for that type in the Fast Formulas page, Oracle's documentation for that formula type, and predefined sample formulas. Extract Rule formulas usually get no inputs and work with contexts and DBIs. Payroll formulas get element input values. Validation formulas get the entered value.

**Q5. What is the default data type of an input variable?**
> NUMBER. Text and date inputs must be declared, for example `INPUTS ARE p_code (TEXT)`.

**Q6. How do you read a value as of a different date?**
> Use `CHANGE_CONTEXTS(EFFECTIVE_DATE = l_date) ( ... )`. DBIs inside the parentheses are evaluated as of that date.

**Q7. How do you handle a DBI that returns multiple rows?**
> Use the array version of the DBI with `DEFAULT_DATA_VALUE`, and loop using `.FIRST`, `.EXISTS` and `.NEXT` in a WHILE loop.

**Q8. You changed a formula but the extract still shows old values. Why?**
> The formula was not recompiled, the extract was not re-validated, or the formula's effective date doesn't cover the run's effective date.

**Q9. Difference between Record Calculation and Rule?**
> A Record Calculation is an expression using attributes of the same record, with no new data access and no custom code. A Rule is an Extract Rule fast formula that can read DBIs, use contexts and apply complex logic.

**Q10. Which attribute type would you use for a trailer record count?**
> Summary Calculation with COUNT, or a `COUNT()` in the eText template.

---

## Final Memory Map

```
ATTRIBUTE TYPES  → String | Parameter | DBI | Record Calc | Summary Calc | Rule
CHOOSE           → Cheapest first, Fast Formula last
FF WRITTEN FOR   → Rule attributes (Extract Rule) + Extract Criteria (+ Extract Record)
FF CONTRACT      → Formula type decides contexts, DBIs, inputs, outputs
DATA TYPES       → NUMBER (default) | TEXT 'quotes' | DATE '...'(date) | arrays
STRUCTURE        → Comment → ALIAS → DEFAULT FOR → INPUTS ARE → Logic → RETURN
EXTRACT RULE     → No inputs usually; contexts + DBIs; RETURN RULE_VALUE
SAFETY           → DEFAULT FOR, WAS DEFAULTED, initialise outputs, final ELSE
DATES            → GET_CONTEXT(EFFECTIVE_DATE), CHANGE_CONTEXTS, ADD_MONTHS edge cases
ARRAYS           → DEFAULT_DATA_VALUE, FIRST(-1), EXISTS, NEXT(i,-1)
DEPLOY           → Compile FF → attach to Rule attr → validate extract → test edge cases
```