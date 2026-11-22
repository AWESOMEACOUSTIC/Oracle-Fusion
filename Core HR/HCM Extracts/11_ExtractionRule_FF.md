# Extract Rule Fast Formula Practice Scenarios

> Work through these **in order**. Each one adds just one new idea.
> For every scenario, cover the solution and try writing it yourself after the **"Think first"** part. Then compare.
>
> ⚠️ Database item names here (like `PER_PERSON_DATE_OF_BIRTH`) are **made-up examples**. On your pod, find the real name in the formula editor's **Database Items** search. Attribute keys used with `DATA_ELEMENTS` (like `'FIRST_NAME'`) are the **Tag Names** of attributes in your record, so check them on your extract.

---

## Before you start: kill the fear

### The 4 questions to ask before writing ANY Extract Rule formula

| # | Question | Answer decides… |
|---|---|---|
| 1 | What should the output look like? | What you put in `RULE_VALUE` (always **text**) |
| 2 | Where does each piece of data come from? | Database item, another attribute, or a parameter |
| 3 | Is it **one value** or a **list**? | Single database item or array database item |
| 4 | What if the data is missing? | Your `DEFAULT` lines and starting value |

### Where data can come from (only 3 places)

| Data comes from… | How you read it | Safety line |
|---|---|---|
| **Database item (one value)** | `PER_SOMETHING` | `DEFAULT FOR PER_SOMETHING IS ...` |
| **Database item (a list)** | `PER_SOMETHING_ARR[i]` with a loop | `DEFAULT_DATA_VALUE FOR PER_SOMETHING_ARR IS ...` |
| **Another attribute in the same record** | `DATA_ELEMENTS['TAG_NAME']` | `DEFAULT FOR DATA_ELEMENTS IS EMPTY_TEXT_TEXT` + `INPUTS ARE DATA_ELEMENTS (TEXT_TEXT)` |
| *(Bonus)* **An extract parameter** | `GET_PARAMETER_VALUE_DATE('EFFECTIVE_DATE')` | None needed |

### TEXT_NUMBER, TEXT_TEXT… what is this?

Array type names are just **two words**:

```
TEXT _ NUMBER
 │       │
 │       └── how you find a box  → by a NUMBER (position 1, 2, 3…)
 └────────── what's inside a box → TEXT
```

| Type | What's inside | How you find a box | Where you'll see it |
|---|---|---|---|
| `TEXT_NUMBER` | Text | Number (1, 2, 3) | Array database items of text, like a list of phone numbers |
| `NUMBER_NUMBER` | Number | Number | Array database items of numbers |
| `DATE_NUMBER` | Date | Number | Array database items of dates |
| `TEXT_TEXT` | Text | Text (a name) | `DATA_ELEMENTS`, where you look up by attribute name |

**The good news:** you **only write a type for INPUTS**. For array database items, Oracle already knows the type; you never write it. And for `DATA_ELEMENTS` it's **always `TEXT_TEXT`**. So in practice you only need to remember:

```
INPUTS ARE DATA_ELEMENTS (TEXT_TEXT)
```

### The full skeleton: sections always go in this order

```
/* Name / Type / Purpose */

DEFAULT FOR <single database item> IS ...              -- 1. single database items
DEFAULT_DATA_VALUE FOR <array database item> IS ...    -- 2. array database items
DEFAULT FOR DATA_ELEMENTS IS EMPTY_TEXT_TEXT           -- 3. input default
INPUTS ARE DATA_ELEMENTS (TEXT_TEXT)                   -- 4. inputs

RULE_VALUE = <starting answer>                         -- 5. starting answer

<read data into l_ variables>                          -- 6. read
<IF / loops>                                           -- 7. logic

RETURN RULE_VALUE                                      -- 8. return
```
Skip any section you don't need. A formula with only database items has no sections 3 and 4.

---

# LEVEL 1 — Single database items only

## Scenario 1: Age band

**Requirement:** Vendor wants an age band: `UNDER_25`, `25_TO_40`, `ABOVE_40`. If no date of birth is recorded, send `UNKNOWN`.

**Think first**
1. Output: text, one of 4 values
2. Data: date of birth → **database item**; effective date → **parameter**
3. One value → **single database item**
4. Missing date of birth → `UNKNOWN`

```
/* Name: XX_AGE_BAND   Type: Extract Rule */

DEFAULT FOR PER_PERSON_DATE_OF_BIRTH IS '4712/12/31 00:00:00' (date)

RULE_VALUE = 'UNKNOWN'                                   /* starting answer */

l_eff_date = GET_PARAMETER_VALUE_DATE('EFFECTIVE_DATE')  /* extract run date */

IF PER_PERSON_DATE_OF_BIRTH WAS NOT DEFAULTED THEN
(
   l_age = TRUNC(MONTHS_BETWEEN(l_eff_date, PER_PERSON_DATE_OF_BIRTH) / 12)

   IF l_age < 25 THEN
      RULE_VALUE = 'UNDER_25'
   ELSE IF l_age <= 40 THEN
      RULE_VALUE = '25_TO_40'
   ELSE
      RULE_VALUE = 'ABOVE_40'
)

RETURN RULE_VALUE
```

**What to notice**
- `WAS NOT DEFAULTED` means "only if a real date of birth exists".
- `ELSE IF l_age <= 40` doesn't need `>= 25`, because the first IF already caught everyone under 25.
- `l_age` is only used for comparison, so no `TO_CHAR` is needed. We return fixed text values.

**Try it yourself:** change the bands to `UNDER_30` / `30_TO_50` / `ABOVE_50`.

---

## Scenario 2: Gender code mapping

**Requirement:** Oracle stores gender as `M`, `F`, `ORA_D`, etc. The vendor only accepts `M`, `F` or `X` (anything else).

**Think first**
1. Output: `M`, `F` or `X`
2. Data: gender → **database item**
3. One value → **single**
4. Missing → `X`

```
/* Name: XX_GENDER_CODE   Type: Extract Rule */

DEFAULT FOR PER_PERSON_SEX IS 'X'

RULE_VALUE = 'X'                          /* anything unexpected → X */

l_sex = UPPER(PER_PERSON_SEX)             /* safe against 'm' vs 'M' */

IF l_sex = 'M' THEN
   RULE_VALUE = 'M'
ELSE IF l_sex = 'F' THEN
   RULE_VALUE = 'F'

RETURN RULE_VALUE
```

**What to notice**
- The starting value `'X'` acts as the "else" for **every** unexpected code. No final ELSE is needed.
- `UPPER` protects you from case differences, because text comparison is case-sensitive.

**Try it yourself:** the vendor wants `MALE` / `FEMALE` / `OTHER` instead.

---

## Scenario 3: Probation status (two dates)

**Requirement:** Probation is 6 months from hire date. Send `ON_PROBATION` if the effective date is still inside 6 months, else `CONFIRMED`. If hire date is missing, send `NO_HIRE_DATE`.

**Think first**
1. Output: 3 possible text values
2. Data: hire date → **database item**; effective date → **parameter**
3. Single
4. Missing → `NO_HIRE_DATE`

```
/* Name: XX_PROBATION_STATUS   Type: Extract Rule */

DEFAULT FOR PER_PERSON_HIRE_DATE IS '4712/12/31 00:00:00' (date)

RULE_VALUE = 'NO_HIRE_DATE'

l_eff_date = GET_PARAMETER_VALUE_DATE('EFFECTIVE_DATE')

IF PER_PERSON_HIRE_DATE WAS NOT DEFAULTED THEN
(
   l_prob_end = ADD_MONTHS(PER_PERSON_HIRE_DATE, 6)

   IF l_eff_date < l_prob_end THEN
      RULE_VALUE = 'ON_PROBATION'
   ELSE
      RULE_VALUE = 'CONFIRMED'
)

RETURN RULE_VALUE
```

**What to notice**
- `ADD_MONTHS` creates a new date variable `l_prob_end`.
- Dates can be compared directly with `<`, `>`, `=`.
- Edge case: hire date 31-Aug + 6 months gives end of February.

**Try it yourself:** also send `NOT_JOINED` if the hire date is **after** the effective date (future joiner).

---

# LEVEL 2 — Inputs only (other attributes in the record)

## Scenario 4: Name format "LAST, First"

**Requirement:** Your record already has attributes **First Name** (tag `FIRST_NAME`) and **Last Name** (tag `LAST_NAME`). The vendor wants `KUMAR, Ravi`. If last name is missing, send just the first name.

**Think first**
1. Output: text
2. Data: both names are **already attributes** → **inputs**
3. `DATA_ELEMENTS` is a list looked up by name → `TEXT_TEXT`
4. Missing last name → first name only; both missing → blank

```
/* Name: XX_NAME_FORMAT   Type: Extract Rule */

DEFAULT FOR DATA_ELEMENTS IS EMPTY_TEXT_TEXT
INPUTS ARE DATA_ELEMENTS (TEXT_TEXT)

RULE_VALUE = ' '
l_first    = ' '
l_last     = ' '

IF DATA_ELEMENTS.EXISTS('FIRST_NAME') THEN
   l_first = DATA_ELEMENTS['FIRST_NAME']

IF DATA_ELEMENTS.EXISTS('LAST_NAME') THEN
   l_last = DATA_ELEMENTS['LAST_NAME']

IF l_last <> ' ' THEN
   RULE_VALUE = UPPER(l_last) || ', ' || l_first
ELSE
   RULE_VALUE = l_first

RETURN RULE_VALUE
```

**What to notice**
- No database items, so there's no `DEFAULT FOR <database item>`. The only default is for `DATA_ELEMENTS`.
- Always check `.EXISTS('KEY')` before reading `DATA_ELEMENTS['KEY']`.
- `l_first` and `l_last` are set to `' '` at the top, so they always have a value even if the attribute is missing.

**Try it yourself:** format it as `Ravi KUMAR` instead.

---

## Scenario 5: Pad Person Number to 10 digits

**Requirement:** Your record has **Person Number** (tag `PERSON_NUMBER`). The vendor needs exactly 10 characters, left-padded with zeros: `1001` → `0000001001`.

**Think first**
1. Output: 10-character text
2. Data: Person Number is an attribute → **input**
3. `TEXT_TEXT`
4. Missing → `0000000000`

```
/* Name: XX_PADDED_PERSON_NUMBER   Type: Extract Rule */

DEFAULT FOR DATA_ELEMENTS IS EMPTY_TEXT_TEXT
INPUTS ARE DATA_ELEMENTS (TEXT_TEXT)

RULE_VALUE = '0000000000'

IF DATA_ELEMENTS.EXISTS('PERSON_NUMBER') THEN
   RULE_VALUE = LPAD(DATA_ELEMENTS['PERSON_NUMBER'], 10, '0')

RETURN RULE_VALUE
```

**What to notice**
- `LPAD(text, length, character)` pads on the **left**. `RPAD` pads on the right.
- Edge case: if the person number is **longer** than 10, `LPAD` **cuts it**. Ask the vendor what they want in that case.

**Try it yourself:** pad to 8 characters with spaces on the right (`RPAD`).

---

# LEVEL 3 — Database item AND input together (the one you fear)

> **The trick:** don't think of it as "mixing". Just write **both** safety sections and read each piece of data from where it lives. That's it.

## Scenario 6: Employee code = Country + Person Number

**Requirement:** Build `IN-1001`, where `IN` is the legislation code of the assignment (**database item**) and `1001` is the Person Number (**attribute in record**).

**Think first**
1. Output: `<country>-<person number>`
2. Data: country → **database item**; person number → **input**
3. Both are single values (`DATA_ELEMENTS` is the only "list" and it's always `TEXT_TEXT`)
4. Missing country → `XX`; missing person number → blank output

```
/* Name: XX_EMPLOYEE_CODE   Type: Extract Rule */

/* 1. single database item default */
DEFAULT FOR PER_ASG_LEGISLATION_CODE IS 'XX'

/* 3 + 4. input default + inputs */
DEFAULT FOR DATA_ELEMENTS IS EMPTY_TEXT_TEXT
INPUTS ARE DATA_ELEMENTS (TEXT_TEXT)

/* 5. starting answer */
RULE_VALUE = ' '

/* 6. read */
l_country = PER_ASG_LEGISLATION_CODE              /* from database item */

/* 7. logic */
IF DATA_ELEMENTS.EXISTS('PERSON_NUMBER') THEN
   RULE_VALUE = l_country || '-' || DATA_ELEMENTS['PERSON_NUMBER']   /* from input */

RETURN RULE_VALUE
```

**What to notice**
- Two `DEFAULT FOR` lines: one for the database item and one for `DATA_ELEMENTS`. Each data source gets its own safety line.
- The database item is read directly by name; the input is read with `['KEY']`.

**Try it yourself:** produce `IN/1001/E`, where `E` is a fixed string.

---

## Scenario 7: Service band, but only for employees

**Requirement:** Your record has **Person Type** (tag `PERSON_TYPE`). If it's not `EMP`, send `NA`. For employees, use hire date (**database item**) to send `NEW` (< 1 year), `MID` (1–5 years) or `SENIOR` (> 5 years).

**Think first**
1. Output: `NA` / `NEW` / `MID` / `SENIOR`
2. Data: person type → **input**; hire date → **database item**; effective date → **parameter**
3. All single
4. Missing hire date → `NA`

```
/* Name: XX_SERVICE_BAND   Type: Extract Rule */

DEFAULT FOR PER_PERSON_HIRE_DATE IS '4712/12/31 00:00:00' (date)
DEFAULT FOR DATA_ELEMENTS IS EMPTY_TEXT_TEXT
INPUTS ARE DATA_ELEMENTS (TEXT_TEXT)

RULE_VALUE = 'NA'
l_type     = ' '

IF DATA_ELEMENTS.EXISTS('PERSON_TYPE') THEN
   l_type = UPPER(DATA_ELEMENTS['PERSON_TYPE'])

IF l_type = 'EMP' AND PER_PERSON_HIRE_DATE WAS NOT DEFAULTED THEN
(
   l_eff_date = GET_PARAMETER_VALUE_DATE('EFFECTIVE_DATE')
   l_years    = MONTHS_BETWEEN(l_eff_date, PER_PERSON_HIRE_DATE) / 12

   IF l_years < 1 THEN
      RULE_VALUE = 'NEW'
   ELSE IF l_years <= 5 THEN
      RULE_VALUE = 'MID'
   ELSE
      RULE_VALUE = 'SENIOR'
)

RETURN RULE_VALUE
```

**What to notice**
- The **input** decides *whether* to calculate; the **database item** gives *what* to calculate with.
- One IF combines both checks with `AND`.
- `l_years` keeps decimals here (no `TRUNC`), so 0.9 years counts as `NEW`.

**Try it yourself:** also treat `CWK` (contingent worker) as `NA_CWK` instead of `NA`.

---

# LEVEL 4 — Array database items

> Remember the loop pattern:
> ```
> i = ARR.FIRST(-1)
> WHILE ARR.EXISTS(i) LOOP
> (
>    ... use ARR[i] ...
>    i = ARR.NEXT(i, -1)
> )
> ```

## Scenario 8: Count phone numbers

**Requirement:** Send how many phone numbers the person has, for example `3`.

**Think first**
1. Output: a number as text
2. Data: phone numbers → **database item**
3. A **list** → **array database item**
4. No phones → `0`

```
/* Name: XX_PHONE_COUNT   Type: Extract Rule */

DEFAULT_DATA_VALUE FOR PER_PHONE_NUMBERS_ARR IS ' '

l_count = 0
i = PER_PHONE_NUMBERS_ARR.FIRST(-1)

WHILE PER_PHONE_NUMBERS_ARR.EXISTS(i) LOOP
(
   l_count = l_count + 1
   i = PER_PHONE_NUMBERS_ARR.NEXT(i, -1)
)

RULE_VALUE = TO_CHAR(l_count)          /* number → text */
RETURN RULE_VALUE
```

**What to notice**
- We don't even read `ARR[i]`; we just count the boxes.
- `TO_CHAR` because `RULE_VALUE` must be text.
- No phones: the loop is skipped, so the output is `0`.

**Try it yourself:** count only phones that are not blank (`ARR[i] <> ' '`).

---

## Scenario 9: Find the work phone (two arrays side by side)

**Requirement:** You have two array database items from the same source:
- `PER_PHONE_TYPES_ARR` holds phone types (for example `W1` = work, `H1` = home; check the codes in your pod)
- `PER_PHONE_NUMBERS_ARR` holds the numbers

Box 1 of both arrays belongs to the same phone, and so on. Send the **work phone**, or `NO_WORK_PHONE`.

**Think first**
1. Output: one phone number or `NO_WORK_PHONE`
2. Data: two **array database items**
3. Lists, matched by **the same index `i`**
4. No work phone → `NO_WORK_PHONE`

```
/* Name: XX_WORK_PHONE   Type: Extract Rule */

DEFAULT_DATA_VALUE FOR PER_PHONE_TYPES_ARR   IS ' '
DEFAULT_DATA_VALUE FOR PER_PHONE_NUMBERS_ARR IS ' '

RULE_VALUE = 'NO_WORK_PHONE'
i = PER_PHONE_TYPES_ARR.FIRST(-1)

WHILE PER_PHONE_TYPES_ARR.EXISTS(i) LOOP
(
   IF PER_PHONE_TYPES_ARR[i] = 'W1' THEN
   (
      RULE_VALUE = PER_PHONE_NUMBERS_ARR[i]     /* same i = same phone */
      i = -1                                    /* found it → force loop to stop */
   )
   ELSE
      i = PER_PHONE_TYPES_ARR.NEXT(i, -1)
)

RETURN RULE_VALUE
```

**What to notice**
- **Same index `i` in both arrays** means the same phone record.
- `i = -1` is a trick to **stop early**: on the next check, `EXISTS(-1)` is false, so the loop ends.
- `i` changes in **both** branches (found → -1, not found → NEXT), so there's no infinite loop.

**Try it yourself:** find the home phone (`H1`) instead, and if there's no home phone, fall back to the work phone.

---

## Scenario 10: First 3 non-blank emails, comma-separated

**Requirement:** From the email array, send up to 3 email addresses separated by commas. Skip blank ones.

**Think first**
1. Output: text list
2. Data: emails → **array database item**
3. List
4. None → blank; more than 3 → only first 3

```
/* Name: XX_TOP3_EMAILS   Type: Extract Rule */

DEFAULT_DATA_VALUE FOR PER_EMAIL_ADDRESSES_ARR IS ' '

RULE_VALUE = ' '
l_list  = ' '
l_count = 0
i = PER_EMAIL_ADDRESSES_ARR.FIRST(-1)

WHILE PER_EMAIL_ADDRESSES_ARR.EXISTS(i) AND l_count < 3 LOOP
(
   l_email = PER_EMAIL_ADDRESSES_ARR[i]

   IF l_email <> ' ' THEN                       /* skip blanks */
   (
      IF l_count = 0 THEN
         l_list = l_email
      ELSE
         l_list = l_list || ',' || l_email

      l_count = l_count + 1
   )

   i = PER_EMAIL_ADDRESSES_ARR.NEXT(i, -1)      /* OUTSIDE the IF: always move */
)

RULE_VALUE = l_list
RETURN RULE_VALUE
```

**What to notice**
- The WHILE has **two conditions**: a box exists **and** we have fewer than 3.
- `l_count = 0` decides "first item, so no comma", which is cleaner than checking `l_list = ' '`.
- `NEXT` is **outside** the blank-check IF. If it were inside, a blank email would freeze the loop forever.

**Try it yourself:** separate with `;` and allow up to 5.

---

# LEVEL 5 — Everything together

## Scenario 11: Preferred contact

**Requirement:**
1. If the record's **Work Email** attribute (tag `WORK_EMAIL`) has a value, send it.
2. Else send the **first non-blank phone** from the phone array database item.
3. Else send `NO_CONTACT`.

**Think first**
1. Output: email, phone or `NO_CONTACT`
2. Data: work email → **input**; phones → **array database item**
3. Input (`TEXT_TEXT`) + array
4. Nothing → `NO_CONTACT`

```
/* Name: XX_PREFERRED_CONTACT   Type: Extract Rule */

/* 2. array database item default */
DEFAULT_DATA_VALUE FOR PER_PHONE_NUMBERS_ARR IS ' '

/* 3 + 4. input default + inputs */
DEFAULT FOR DATA_ELEMENTS IS EMPTY_TEXT_TEXT
INPUTS ARE DATA_ELEMENTS (TEXT_TEXT)

/* 5. starting answer */
RULE_VALUE = 'NO_CONTACT'
l_email    = ' '

/* 6. read input */
IF DATA_ELEMENTS.EXISTS('WORK_EMAIL') THEN
   l_email = DATA_ELEMENTS['WORK_EMAIL']

/* 7. logic */
IF l_email <> ' ' THEN
   RULE_VALUE = l_email                         /* step 1: email wins */
ELSE
(
   i = PER_PHONE_NUMBERS_ARR.FIRST(-1)          /* step 2: search phones */

   WHILE PER_PHONE_NUMBERS_ARR.EXISTS(i) LOOP
   (
      IF PER_PHONE_NUMBERS_ARR[i] <> ' ' THEN
      (
         RULE_VALUE = PER_PHONE_NUMBERS_ARR[i]
         i = -1                                 /* found → stop */
      )
      ELSE
         i = PER_PHONE_NUMBERS_ARR.NEXT(i, -1)
   )
)
/* step 3: if nothing found, RULE_VALUE is still 'NO_CONTACT' */

RETURN RULE_VALUE
```

**What to notice**
- The skeleton order is the same as always: array default → input default → inputs → starting answer → read → logic → return.
- The "fallback chain" (email → phone → NO_CONTACT) works because `RULE_VALUE` starts with the **last** fallback, and each step only overwrites it if it finds something.
- This one formula uses almost everything: input, array, `.EXISTS`, early stop, blank check.

---

# Bonus practice (solutions below — try first!)

**P1.** Send `Y` if the person is 60 or older on the effective date, else `N`. Missing date of birth → `N`.

**P2.** Your record has **Department Name** (tag `DEPT_NAME`). Send the first 10 characters in UPPER case. Missing → `NODEPT`.

**P3.** From the phone number array, send the **last** non-blank phone. *(Hint: you can walk through all phones and keep overwriting.)*

---

### Solutions

**P1**
```
/* Name: XX_SENIOR_FLAG   Type: Extract Rule */
DEFAULT FOR PER_PERSON_DATE_OF_BIRTH IS '4712/12/31 00:00:00' (date)

RULE_VALUE = 'N'
l_eff_date = GET_PARAMETER_VALUE_DATE('EFFECTIVE_DATE')

IF PER_PERSON_DATE_OF_BIRTH WAS NOT DEFAULTED THEN
(
   IF MONTHS_BETWEEN(l_eff_date, PER_PERSON_DATE_OF_BIRTH) / 12 >= 60 THEN
      RULE_VALUE = 'Y'
)

RETURN RULE_VALUE
```

**P2**
```
/* Name: XX_DEPT_SHORT   Type: Extract Rule */
DEFAULT FOR DATA_ELEMENTS IS EMPTY_TEXT_TEXT
INPUTS ARE DATA_ELEMENTS (TEXT_TEXT)

RULE_VALUE = 'NODEPT'

IF DATA_ELEMENTS.EXISTS('DEPT_NAME') THEN
   RULE_VALUE = UPPER(SUBSTR(DATA_ELEMENTS['DEPT_NAME'], 1, 10))

RETURN RULE_VALUE
```

**P3**
```
/* Name: XX_LAST_PHONE   Type: Extract Rule */
DEFAULT_DATA_VALUE FOR PER_PHONE_NUMBERS_ARR IS ' '

RULE_VALUE = ' '
i = PER_PHONE_NUMBERS_ARR.FIRST(-1)

WHILE PER_PHONE_NUMBERS_ARR.EXISTS(i) LOOP
(
   IF PER_PHONE_NUMBERS_ARR[i] <> ' ' THEN
      RULE_VALUE = PER_PHONE_NUMBERS_ARR[i]      /* keep overwriting → last one wins */

   i = PER_PHONE_NUMBERS_ARR.NEXT(i, -1)
)

RETURN RULE_VALUE
```
*(Another way: start from `.LAST(-1)` and move backwards with `.PREVIOUS(i, -1)`, stopping at the first non-blank phone.)*

---

## Quick self-check after each formula

- [ ] Did every **single database item** get `DEFAULT FOR`?
- [ ] Did every **array database item** get `DEFAULT_DATA_VALUE FOR`?
- [ ] If I used `DATA_ELEMENTS`: `DEFAULT FOR DATA_ELEMENTS IS EMPTY_TEXT_TEXT` + `INPUTS ARE DATA_ELEMENTS (TEXT_TEXT)`?
- [ ] Did I check `.EXISTS('KEY')` before reading `DATA_ELEMENTS['KEY']`?
- [ ] Did I set `RULE_VALUE` at the **top**?
- [ ] In every loop, does `i` **always** change on every pass?
- [ ] Is `RULE_VALUE` **text**? (`TO_CHAR` for numbers and dates)
- [ ] Does it end with `RETURN RULE_VALUE`?