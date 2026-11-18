# Core HR – BI Publisher SQL Use Cases (Notes)

## Table reference

### Suffix conventions

| Suffix | Meaning | Filter you must add |
| --- | --- | --- |
| `_F` | Date-effective. Same key, one row per date range. | `:P_AS_OF_DATE BETWEEN effective_start_date AND effective_end_date` |
| `_M` | Date-effective, several changes allowed on one day. | The date filter above **plus** `effective_latest_change = 'Y'` |
| `_TL` | Translated names, one row per installed language. | `language = USERENV('LANG')` |
| `_VL` | View of base + `_TL` in the session language. | No language filter needed |
| none | Not date-tracked. | Its own dates (`DATE_FROM`/`DATE_TO`, `DATE_START`/`ACTUAL_TERMINATION_DATE`) |

### Tables used

| Table | One row per | Columns you will use |
| --- | --- | --- |
| PER_ALL_PEOPLE_F | person per date range | PERSON_ID, PERSON_NUMBER, PRIMARY_EMAIL_ID |
| PER_PERSON_NAMES_F | person per name type per date range | NAME_TYPE (`GLOBAL` or a legislation such as `US`, `IN`), FIRST_NAME, LAST_NAME, FULL_NAME, DISPLAY_NAME, TITLE, PRE_NAME_ADJUNCT |
| PER_PEOPLE_LEGISLATIVE_F | person per legislation per date range | LEGISLATION_CODE, SEX, MARITAL_STATUS |
| PER_EMAIL_ADDRESSES | email address (not date-tracked) | EMAIL_ADDRESS_ID, EMAIL_TYPE (`W1` work, `H1` home), EMAIL_ADDRESS, DATE_FROM, DATE_TO |
| PER_PERIODS_OF_SERVICE | work relationship, hire to termination | PERIOD_OF_SERVICE_ID, DATE_START (hire date), ACTUAL_TERMINATION_DATE, PERIOD_TYPE, LEGAL_ENTITY_ID |
| PER_ALL_ASSIGNMENTS_M | assignment (and employment terms) per change | ASSIGNMENT_ID, ASSIGNMENT_NUMBER, PERIOD_OF_SERVICE_ID, ASSIGNMENT_TYPE, PRIMARY_FLAG, ASSIGNMENT_STATUS_TYPE, ORGANIZATION_ID, JOB_ID, POSITION_ID, GRADE_ID, EFFECTIVE_SEQUENCE, EFFECTIVE_LATEST_CHANGE, ACTION_CODE |
| PER_ASSIGNMENT_SUPERVISORS_F | manager link per date range | ASSIGNMENT_ID, MANAGER_ID (the manager's PERSON_ID), MANAGER_ASSIGNMENT_ID, MANAGER_TYPE (`LINE_MANAGER`) |
| HR_ORGANIZATION_UNITS_F_TL | org name per language | ORGANIZATION_ID, NAME, LANGUAGE |
| HR_ORG_UNIT_CLASSIFICATIONS_F | org classification | ORGANIZATION_ID, CLASSIFICATION_CODE (`DEPARTMENT`) |
| PER_JOBS_F_TL | job name per language | JOB_ID, NAME |
| HR_ALL_POSITIONS_F_TL | position name per language | POSITION_ID, NAME |
| PER_GRADES_F_TL | grade name per language | GRADE_ID, NAME |

### Assignment columns

- **ASSIGNMENT_TYPE**: `E` employee, `C` contingent worker, `N` nonworker, `P` pending worker. `ET`, `CT`, `NT`, `PT` are the employment-terms rows stored in the same table. Filtering `= 'E'` keeps employee assignments and drops terms rows.
- **ASSIGNMENT_STATUS_TYPE**: `ACTIVE`, `INACTIVE`, `SUSPENDED`. After a termination the assignment does not disappear; a new slice with `INACTIVE` is added.
- **PRIMARY_FLAG**: `Y` on the person's primary assignment. It stays `Y` on inactive slices and on a contingent worker's assignment.

---

## Use Case 1 – As of person number

```sql
SELECT papf.person_number,
       papf.person_id,
       ppnf.first_name,
       ppnf.last_name,
       ppnf.display_name
FROM   per_all_people_f papf
LEFT JOIN per_person_names_f ppnf
       ON  ppnf.person_id = papf.person_id
       AND ppnf.name_type = 'GLOBAL'
       AND :P_AS_OF_DATE BETWEEN ppnf.effective_start_date AND ppnf.effective_end_date
WHERE  :P_AS_OF_DATE BETWEEN papf.effective_start_date AND papf.effective_end_date
ORDER BY papf.person_number
```

**Reasoning**

I will get person number and person_id from per_all_people_f and first_name and last_name from per_person_names_f. I used left join because the requirement said to include all persons. Now there are two types of name_type and i.e Global and Local. So i will join the rows with the second table on this three condition. The reason of using this condition in join is to preserve the records. If i add this in where then the left join will act as inner join and it will remove the rows. And then i have added the filter to give me the row if the provided date is between the effective start and end date.

---

## Use Case 2 – Active primary employees

```sql
SELECT papf.person_number,
       paam.assignment_number,
       hout.name            AS organization_name,
       paam.organization_id
FROM   per_all_assignments_m paam

JOIN   per_all_people_f papf
       ON  papf.person_id = paam.person_id
       AND :P_AS_OF_DATE BETWEEN papf.effective_start_date AND papf.effective_end_date
LEFT JOIN hr_organization_units_f_tl hout
       ON  hout.organization_id = paam.organization_id
       AND hout.language = USERENV('LANG')
       AND :P_AS_OF_DATE BETWEEN hout.effective_start_date AND hout.effective_end_date
WHERE  :P_AS_OF_DATE BETWEEN paam.effective_start_date AND paam.effective_end_date
AND    paam.effective_latest_change = 'Y'
AND    paam.assignment_type         = 'E'
AND    paam.primary_flag            = 'Y'
AND    paam.assignment_status_type  = 'ACTIVE'
ORDER BY papf.person_number
```

**Reasoning**

Considering the assignments table as the main table as major of the filter on columns belongs to the assignments table. And since i need person number i have to join it with the per_all_people_f table and then a left join with organization table because there can be some rows who might not have matching organization translation record for selected language and date and then add all the specified filters in the where clause.

---

## Use Case 3 – Employee names

```sql
-- with CTE
WITH sel AS (
  SELECT paam.person_id, paam.assignment_number
  FROM   per_all_assignments_m paam
  WHERE  :P_AS_OF_DATE BETWEEN paam.effective_start_date AND paam.effective_end_date
  AND    paam.effective_latest_change = 'Y'
  AND    paam.assignment_type         = 'E'
  AND    paam.primary_flag            = 'Y'
  AND    paam.assignment_status_type  = 'ACTIVE'
)
SELECT papf.person_number,
       ppnf.full_name,
       sel.assignment_number
FROM   sel
JOIN   per_all_people_f papf
       ON  papf.person_id = sel.person_id
       AND :P_AS_OF_DATE BETWEEN papf.effective_start_date AND papf.effective_end_date
JOIN   per_person_names_f ppnf
       ON  ppnf.person_id = sel.person_id
       AND ppnf.name_type = 'GLOBAL'
       AND :P_AS_OF_DATE BETWEEN ppnf.effective_start_date AND ppnf.effective_end_date
ORDER BY ppnf.full_name

-- without cte
SELECT papf.person_number,
       ppnf.full_name,
       paam.assignment_number
FROM per_all_assignments_m paam
JOIN per_all_people_f papf
  ON papf.person_id = paam.person_id
 AND :P_AS_OF_DATE BETWEEN papf.effective_start_date
                       AND papf.effective_end_date
JOIN per_person_names_f ppnf
  ON ppnf.person_id = paam.person_id
 AND ppnf.name_type = 'GLOBAL'
 AND :P_AS_OF_DATE BETWEEN ppnf.effective_start_date
                       AND ppnf.effective_end_date
WHERE :P_AS_OF_DATE BETWEEN paam.effective_start_date
                        AND paam.effective_end_date
  AND paam.effective_latest_change = 'Y'
  AND paam.assignment_type = 'E'
  AND paam.primary_flag = 'Y'
  AND paam.assignment_status_type = 'ACTIVE'
ORDER BY ppnf.full_name;
```

**Reasoning**

The important point here is that the report is about employees with active assignments and not about every person in the system.

---

## Use Case 4 – Department Selection

```sql
-- for the dropdown

SELECT DISTINCT hout.name
FROM   hr_organization_units_f_tl hout
JOIN   hr_org_unit_classifications_f houc
       ON  houc.organization_id     = hout.organization_id
       AND houc.classification_code = 'DEPARTMENT'
       AND TRUNC(SYSDATE) BETWEEN houc.effective_start_date AND houc.effective_end_date
WHERE  hout.language = USERENV('LANG')
AND    TRUNC(SYSDATE) BETWEEN hout.effective_start_date AND hout.effective_end_date
ORDER BY 1

-- main data set
WITH sel AS (
  SELECT paam.person_id, paam.assignment_number, paam.organization_id
  FROM   per_all_assignments_m paam
  WHERE  :P_AS_OF_DATE BETWEEN paam.effective_start_date AND paam.effective_end_date
  AND    paam.effective_latest_change = 'Y'
  AND    paam.assignment_type         = 'E'
  AND    paam.primary_flag            = 'Y'
  AND    paam.assignment_status_type  = 'ACTIVE'
)
SELECT papf.person_number,
       sel.assignment_number,
       hout.name AS department_name
FROM   sel
JOIN   per_all_people_f papf
       ON  papf.person_id = sel.person_id
       AND :P_AS_OF_DATE BETWEEN papf.effective_start_date AND papf.effective_end_date
LEFT JOIN hr_organization_units_f_tl hout
       ON  hout.organization_id = sel.organization_id
       AND hout.language = USERENV('LANG')
       AND :P_AS_OF_DATE BETWEEN hout.effective_start_date AND hout.effective_end_date
WHERE  (:P_DEPARTMENT_NAME IS NULL OR hout.name = :P_DEPARTMENT_NAME)
ORDER BY hout.name, papf.person_number
```

**Reasoning**

First i need to create a drop down option so that user can select the departments. Here i have used two tables hr_organization_units_f_tl (holds the name of the department) and hr_organization_units_f (it holds the department classification code)

---

## Use Case 5 – Missing Job Setup

```sql
WITH sel AS (
  SELECT paam.person_id, paam.assignment_id, paam.assignment_number,
         paam.job_id, paam.position_id, paam.grade_id
  FROM   per_all_assignments_m paam
  WHERE  :P_AS_OF_DATE BETWEEN paam.effective_start_date AND paam.effective_end_date
  AND    paam.effective_latest_change = 'Y'
  AND    paam.assignment_type         = 'E'
  AND    paam.primary_flag            = 'Y'
  AND    paam.assignment_status_type  = 'ACTIVE'
  AND job_id is null
)
SELECT papf.person_number,
       sel.assignment_number,
       hapt.name       AS position_name,
       pgt.name        AS grade_name,
       mgrn.full_name  AS line_manager_name
FROM   sel
JOIN   per_all_people_f papf
       ON  papf.person_id = sel.person_id
       AND :P_AS_OF_DATE BETWEEN papf.effective_start_date AND papf.effective_end_date
LEFT JOIN hr_all_positions_f_tl hapt
       ON  hapt.position_id = sel.position_id
       AND hapt.language = USERENV('LANG')
       AND :P_AS_OF_DATE BETWEEN hapt.effective_start_date AND hapt.effective_end_date
LEFT JOIN per_grades_f_tl pgt
       ON  pgt.grade_id = sel.grade_id
       AND pgt.language = USERENV('LANG')
       AND :P_AS_OF_DATE BETWEEN pgt.effective_start_date AND pgt.effective_end_date
LEFT JOIN per_assignment_supervisors_f pasf
       ON  pasf.assignment_id = sel.assignment_id
       AND pasf.manager_type  = 'LINE_MANAGER'
       AND :P_AS_OF_DATE BETWEEN pasf.effective_start_date AND pasf.effective_end_date
LEFT JOIN per_person_names_f mgrn
       ON  mgrn.person_id = pasf.manager_id
       AND mgrn.name_type = 'GLOBAL'
       AND :P_AS_OF_DATE BETWEEN mgrn.effective_start_date AND mgrn.effective_end_date
ORDER BY papf.person_number
```

---

## Use Case 6 – Names formatted for matching

```sql
WITH sel AS (
  SELECT paam.person_id, paam.assignment_id
  FROM   per_all_assignments_m paam
  WHERE  :P_AS_OF_DATE BETWEEN paam.effective_start_date AND paam.effective_end_date
  AND    paam.effective_latest_change = 'Y'
  AND    paam.assignment_type         = 'E'
  AND    paam.primary_flag            = 'Y'
  AND    paam.assignment_status_type  = 'ACTIVE'
),
gender AS (
  SELECT person_id, MAX(sex) AS sex
  FROM   per_people_legislative_f
  WHERE  :P_AS_OF_DATE BETWEEN effective_start_date AND effective_end_date
  GROUP BY person_id
),
pref AS (
  SELECT ppnf.person_id,
         CASE
           WHEN COALESCE(ppnf.title, ppnf.pre_name_adjunct) IS NOT NULL
             THEN RTRIM(UPPER(COALESCE(ppnf.title, ppnf.pre_name_adjunct)), '.') || '.'
           WHEN g.sex = 'M' THEN 'MR.'
           WHEN g.sex = 'F' THEN 'MS.'
         END
         || TRIM(ppnf.first_name || ' ' || ppnf.last_name) AS prefixed_name
  FROM   per_person_names_f ppnf
  LEFT JOIN gender g ON g.person_id = ppnf.person_id
  WHERE  ppnf.name_type = 'GLOBAL'
  AND    :P_AS_OF_DATE BETWEEN ppnf.effective_start_date AND ppnf.effective_end_date
)
SELECT sel.person_id,
       papf.person_number,
       emp.prefixed_name,
       mgrp.person_number  AS manager_person_number,
       mgr.prefixed_name   AS manager_prefixed_name
FROM   sel
JOIN   per_all_people_f papf
       ON  papf.person_id = sel.person_id
       AND :P_AS_OF_DATE BETWEEN papf.effective_start_date AND papf.effective_end_date
JOIN   pref emp ON emp.person_id = sel.person_id
LEFT JOIN per_assignment_supervisors_f pasf
       ON  pasf.assignment_id = sel.assignment_id
       AND pasf.manager_type  = 'LINE_MANAGER'
       AND :P_AS_OF_DATE BETWEEN pasf.effective_start_date AND pasf.effective_end_date
LEFT JOIN per_all_people_f mgrp
       ON  mgrp.person_id = pasf.manager_id
       AND :P_AS_OF_DATE BETWEEN mgrp.effective_start_date AND mgrp.effective_end_date
LEFT JOIN pref mgr ON mgr.person_id = pasf.manager_id
ORDER BY papf.person_number
```

**Reasoning**

1. Here first question is why are we even considering assignment tables even tough we don't have to retrieve any columns from the table. The reason is we have to connect supervisor table to fetch the line_manager_name, hr_all_positions_f_tl for position name and all. We cannot directly connect the table per_all_people_f table with these above tables so we have to use assignments table to join all this. Also, To establish which assignment belongs to the employee, so you can find the supervisor associated with that assignment.
2. My question for you: If we remove both `GROUP BY` and `MAX()`, and person 102 has two matching rows, what happens when we join that result to the employee name table?

---

## Use Case 7 – Headcount by department

```sql
WITH sel AS (
  SELECT paam.person_id, paam.organization_id
  FROM   per_all_assignments_m paam
  WHERE  :P_AS_OF_DATE BETWEEN paam.effective_start_date AND paam.effective_end_date
  AND    paam.effective_latest_change = 'Y'
  AND    paam.assignment_type         = 'E'
  AND    paam.primary_flag            = 'Y'
  AND    paam.assignment_status_type  = 'ACTIVE'
)
SELECT sel.organization_id,
       hout.name                      AS organization_name,
       COUNT(DISTINCT sel.person_id)  AS headcount
FROM   sel
LEFT JOIN hr_organization_units_f_tl hout
       ON  hout.organization_id = sel.organization_id
       AND hout.language = USERENV('LANG')
       AND :P_AS_OF_DATE BETWEEN hout.effective_start_date AND hout.effective_end_date
GROUP BY sel.organization_id, hout.name
ORDER BY headcount DESC
```

---

## Use Case 8 – Departments with at least 2 employees

```sql
WITH sel AS (
  SELECT paam.person_id, paam.organization_id
  FROM   per_all_assignments_m paam
  WHERE  :P_AS_OF_DATE BETWEEN paam.effective_start_date AND paam.effective_end_date
  AND    paam.effective_latest_change = 'Y'
  AND    paam.assignment_type         = 'E'
  AND    paam.primary_flag            = 'Y'
  AND    paam.assignment_status_type  = 'ACTIVE'
)
SELECT hout.name                     AS organization_name,
       COUNT(DISTINCT sel.person_id) AS headcount
FROM   sel
LEFT JOIN hr_organization_units_f_tl hout
       ON  hout.organization_id = sel.organization_id
       AND hout.language = USERENV('LANG')
       AND :P_AS_OF_DATE BETWEEN hout.effective_start_date AND hout.effective_end_date
GROUP BY sel.organization_id, hout.name
HAVING COUNT(DISTINCT sel.person_id) >= 2
ORDER BY headcount DESC
```

---

## Use Case 9 – Percentage of selected employees missing a job, per department

```sql
WITH sel AS (
  SELECT paam.person_id, paam.organization_id, paam.job_id
  FROM   per_all_assignments_m paam
  WHERE  :P_AS_OF_DATE BETWEEN paam.effective_start_date AND paam.effective_end_date
  AND    paam.effective_latest_change = 'Y'
  AND    paam.assignment_type         = 'E'
  AND    paam.primary_flag            = 'Y'
  AND    paam.assignment_status_type  = 'ACTIVE'
)
SELECT hout.name AS organization_name,
       ROUND(
         100 * COUNT(DISTINCT CASE WHEN sel.job_id IS NULL THEN sel.person_id END)
             / NULLIF(COUNT(DISTINCT sel.person_id), 0)
       , 2) AS missing_job_pct
FROM   sel
LEFT JOIN hr_organization_units_f_tl hout
       ON  hout.organization_id = sel.organization_id
       AND hout.language = USERENV('LANG')
       AND :P_AS_OF_DATE BETWEEN hout.effective_start_date AND hout.effective_end_date
GROUP BY sel.organization_id, hout.name
ORDER BY missing_job_pct DESC
```

---

## Use Case 10 – Valid primary work email, or "Missing Email"

```sql
WITH sel AS (
  SELECT paam.person_id
  FROM   per_all_assignments_m paam
  WHERE  :P_AS_OF_DATE BETWEEN paam.effective_start_date AND paam.effective_end_date
  AND    paam.effective_latest_change = 'Y'
  AND    paam.assignment_type         = 'E'
  AND    paam.primary_flag            = 'Y'
  AND    paam.assignment_status_type  = 'ACTIVE'
)
SELECT papf.person_number,
       NVL(pea.email_address, 'Missing Email') AS work_email
FROM   sel
JOIN   per_all_people_f papf
       ON  papf.person_id = sel.person_id
       AND :P_AS_OF_DATE BETWEEN papf.effective_start_date AND papf.effective_end_date
LEFT JOIN per_email_addresses pea
       ON  pea.email_address_id = papf.primary_email_id
       AND pea.email_type = 'W1'
       AND NVL(pea.date_from, DATE '1900-01-01') <= :P_AS_OF_DATE
       AND NVL(pea.date_to,   DATE '4712-12-31') >= :P_AS_OF_DATE
ORDER BY papf.person_number
```

---

## Use Case 11 – Employees with at least one valid work email

```sql
WITH sel AS (
  SELECT paam.person_id
  FROM   per_all_assignments_m paam
  WHERE  :P_AS_OF_DATE BETWEEN paam.effective_start_date AND paam.effective_end_date
  AND    paam.effective_latest_change = 'Y'
  AND    paam.assignment_type         = 'E'
  AND    paam.primary_flag            = 'Y'
  AND    paam.assignment_status_type  = 'ACTIVE'
)
SELECT papf.person_number,
       pea.email_address AS work_email_address
FROM   sel
JOIN   per_all_people_f papf
       ON  papf.person_id = sel.person_id
       AND :P_AS_OF_DATE BETWEEN papf.effective_start_date AND papf.effective_end_date
JOIN   per_email_addresses pea
       ON  pea.person_id  = sel.person_id
       AND pea.email_type = 'W1'
       AND NVL(pea.date_from, DATE '1900-01-01') <= :P_AS_OF_DATE
       AND NVL(pea.date_to,   DATE '4712-12-31') >= :P_AS_OF_DATE
ORDER BY papf.person_number, pea.email_address
```

---

## Use Case 12 – Employees with no valid work email, with their manager

```sql
WITH sel AS (
  SELECT paam.person_id, paam.assignment_id, paam.assignment_number
  FROM   per_all_assignments_m paam
  WHERE  :P_AS_OF_DATE BETWEEN paam.effective_start_date AND paam.effective_end_date
  AND    paam.effective_latest_change = 'Y'
  AND    paam.assignment_type         = 'E'
  AND    paam.primary_flag            = 'Y'
  AND    paam.assignment_status_type  = 'ACTIVE'
)
SELECT papf.person_number,
       sel.assignment_number,
       mgrn.full_name      AS manager_name,
       mgrp.person_number  AS manager_person_number
FROM   sel
JOIN   per_all_people_f papf
       ON  papf.person_id = sel.person_id
       AND :P_AS_OF_DATE BETWEEN papf.effective_start_date AND papf.effective_end_date
LEFT JOIN per_assignment_supervisors_f pasf
       ON  pasf.assignment_id = sel.assignment_id
       AND pasf.manager_type  = 'LINE_MANAGER'
       AND :P_AS_OF_DATE BETWEEN pasf.effective_start_date AND pasf.effective_end_date
LEFT JOIN per_person_names_f mgrn
       ON  mgrn.person_id = pasf.manager_id
       AND mgrn.name_type = 'GLOBAL'
       AND :P_AS_OF_DATE BETWEEN mgrn.effective_start_date AND mgrn.effective_end_date
LEFT JOIN per_all_people_f mgrp
       ON  mgrp.person_id = pasf.manager_id
       AND :P_AS_OF_DATE BETWEEN mgrp.effective_start_date AND mgrp.effective_end_date
WHERE  NOT EXISTS (
         SELECT 1
         FROM   per_email_addresses pea
         WHERE  pea.person_id  = sel.person_id
         AND    pea.email_type = 'W1'
         AND    NVL(pea.date_from, DATE '1900-01-01') <= :P_AS_OF_DATE
         AND    NVL(pea.date_to,   DATE '4712-12-31') >= :P_AS_OF_DATE
       )
ORDER BY papf.person_number
```