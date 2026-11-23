# 🛡️ The Gatekeeper's Handbook
### Oracle Fusion Cloud Security: from basics to interview scenarios (HCM + Financials + BI)


---

## 0. The one-picture mental model

```
                    ┌───────────────────────────────┐
   WHO is it?       │   USER  (Security Console)    │
                    └──────────────┬────────────────┘
                                   │ gets roles (manually or via ROLE MAPPINGS)
                    ┌──────────────▼────────────────┐
   WHAT can I DO?   │  FUNCTION SECURITY            │  privileges inside roles
                    ├───────────────────────────────┤
   ON WHICH DATA?   │  DATA SECURITY                │  data security policies +
                    │                               │  HCM security profiles /
                    │                               │  ERP data access (BU, ledger)
                    ├───────────────────────────────┤
   WHICH REPORTS?   │  BI CATALOG SECURITY (ACL)    │  permissions on folders/reports
                    └───────────────────────────────┘
```

Every security question is one of three:
1. **Can the user *do* it?** → function security (roles and privileges)
2. **On *which* records?** → data security (policies, security profiles, data access)
3. **Can they reach the *report object*?** → BI catalog ACL (plus data security *inside* the report)

---

## 1. Role-Based Access Control (RBAC)

Oracle Fusion uses **Role-Based Access Control**. You **never give access to a user directly**. You give the user **roles**, and the **roles carry the access**.

Why it matters:
- 500 HR specialists → change **one role** instead of 500 users.
- Access follows the **job**, not the person. When someone changes jobs, their roles change with them (through role mappings).
- Easier auditing and **segregation of duties**.

---

## 2. The two kinds of security

| Type | Answers the question | Example | Comes from |
|---|---|---|---|
| **Function security** | "What can I **DO**?" (pages, buttons, tasks, work areas) | Can open *Hire an Employee* | **Function security privileges** inside roles |
| **Data security** | "On **WHICH** records?" | Can hire only into the **India** legal employer | **Data security policies** + **HCM security profiles** (HCM) / **data access for users** (ERP) |

**Example:** An HR Specialist in India *can open* the Hire page (function security) but should *only see Indian employees* (data security).

> 🎯 **Interview line:** "Function security decides what I can do. Data security decides which records I can do it on. A user needs both. A privilege without data access shows an empty page, and data access without the privilege means you can't even open the page."

That second sentence answers a classic scenario: *"The user can open the page, but the search returns nothing."* → function security is fine, data security is missing.

---

## 3. The building blocks: three terms people confuse

| Term | What it is | Where it lives |
|---|---|---|
| **Function security privilege** | Permission to perform a function (open a page, run a task). E.g. *Hire Worker* | Inside roles |
| **Data security policy** | Grants an action on an object for a set of rows. E.g. "HR Specialist can **manage** Person records **for the persons in their security profile**" | Inside roles |
| **HCM security profile** | Defines **which set of records** (persons, orgs, positions, LDGs...). E.g. "All persons in India legal employer" | Attached to **data roles** (or directly to **abstract roles**) |

> ⚠️ Don't say "security **policies**" when you mean "security **profiles**." Policies live inside roles. Profiles define the data set in HCM.

---

## 4. Role types

Oracle defines **four role types**: **job roles, abstract roles, duty roles, aggregate privileges**. HCM adds **HCM data roles** on top.

| Role type | What it is | Example | Predefined? | Can you create it? | Assign to a user? |
|---|---|---|---|---|---|
| **Aggregate privilege** | One function privilege **+** its related data security policies, bundled | *Promote Worker* aggregate privilege | All predefined | ❌ Can't create, edit or copy | ❌ No |
| **Duty role** | A logical group of related privileges for part of a job | *Manage Worker Documents* duty | Yes | ✅ Yes | ❌ No, only inside job/abstract roles |
| **Job role** | Based on the **job you do** | Human Resource Specialist, HR Analyst | Yes | ✅ Yes | ✅ Yes |
| **Abstract role** | Based on **who you are**, not your job | Employee, Line Manager, Contingent Worker | Yes | ✅ Yes | ✅ Yes |
| **HCM data role** | **One job role + security profiles** (the data scope) | HR Specialist – India | ❌ **None predefined** (enterprise-specific) | ✅ Yes | ✅ Yes |

### 4.1 Abstract role
- Given because of **who you are** in the organization.
- An **employee** gets the **Employee** abstract role and can access their own personal details, payslips and so on.
- A manager can hold **both** Employee **and** Line Manager.
- Predefined codes: `ORA_PER_EMPLOYEE_ABSTRACT`, `ORA_PER_LINE_MANAGER_ABSTRACT`, `ORA_PER_CONTINGENT_WORKER_ABSTRACT`.
- **Abstract roles have no automatic data access.** Without security profiles, an employee can open the Directory but searches return nothing. So you **assign security profiles directly to abstract roles**.

### 4.2 Duty role
- A **collection of related privileges** (and possibly other duties and aggregate privileges).
- Can inherit **other duty roles** and **aggregate privileges**.
- Never assigned directly to users.

### 4.3 Job role
- Based on the user's **job responsibilities**.
- Inherits **mostly aggregate privileges**, plus a few **duty roles**, and is also granted some privileges and policies **directly**.

### 4.4 Aggregate privilege
- **One** function privilege + its data security policies.
- Named after the privilege it contains (Promote Worker → Promote Worker).
- **Cannot inherit any role.** It is always a leaf.
- All predefined. You can't create, edit or copy them, but you can **include** them in custom job, abstract and duty roles.
- When you copy a job or abstract role, aggregate privileges are **not copied**. The copied role simply becomes a member of the same aggregate privileges.

### 4.5 HCM data role
- **Exactly one job role** + **one security profile per object type** (person, public person, organization, position, LDG, document type...).
- Created through **Setup and Maintenance → Users and Security → Assign Security Profiles to Role** (opens *Manage Data Roles and Security Profiles*). You need the **IT Security Manager** role.
- When you pick the job role, Fusion **automatically detects** which object types it needs profiles for.
- Data roles are for **job roles**. **Abstract roles** get security profiles **assigned directly**.

> ⚠️ **Oracle recommends NOT assigning security profiles directly to job roles.** Users with the same job role usually need **different** data (an India HR specialist vs. a US one). If you assign profiles to a job role and later use that job role inside a data role, users may get **more data than intended**.

---

## 5. The role hierarchy (verified against Oracle docs)

```
USER
 │
 ├── HCM DATA ROLE  (e.g. "HR Specialist – India")
 │      ├── JOB ROLE  (exactly one)
 │      └── SECURITY PROFILES  (one per object type)
 │
 ├── JOB ROLE / ABSTRACT ROLE   ◄── what users are assigned
 │      ├── AGGREGATE PRIVILEGES   (most of the access)
 │      ├── DUTY ROLES             (a few)
 │      │      ├── other DUTY ROLES
 │      │      └── AGGREGATE PRIVILEGES
 │      └── privileges + data security policies granted DIRECTLY
 │
 └── ABSTRACT ROLE (Employee / Line Manager / Contingent Worker)
        └── SECURITY PROFILES assigned DIRECTLY

BOTTOM OF EVERYTHING: function security privileges + data security policies
```

**Bottom to top:**
1. **Privileges + data security policies**: the real bottom.
2. **Aggregate privileges**: one privilege + its policies; can't inherit anything.
3. **Duty roles**: many privileges; can inherit duties and aggregate privileges.
4. **Job and abstract roles**: what users get.
5. **HCM data role**: a job role + security profiles.

The hierarchy is deliberately **flat**. Because most access now comes through aggregate privileges, there are few levels, which makes it easier to manage.

> 🎯 **Interview line:** "Privileges and data security policies are the base. Aggregate privileges bundle one privilege with its data policies. They're predefined and can't inherit anything. Duty roles group multiple privileges and can inherit aggregate privileges and other duties. Job and abstract roles sit on top and are what we assign to users. For data access in HCM, we create a data role from one job role plus security profiles, one per object type. For abstract roles like Employee, we assign security profiles directly."

---

## 6. Predefined vs custom roles

- Predefined roles have a **role code starting with `ORA_`**, e.g. `ORA_PER_HUMAN_RESOURCE_SPECIALIST_JOB`.
- Suffixes tell you the type: `_JOB`, `_ABSTRACT`, `_DUTY`.
- You **cannot edit** predefined roles. To change one, **copy** it and edit the copy. The copy is your **custom role**.
- **Never use the `ORA_` prefix** on custom roles. Use a client prefix like `XX_` or `CTS_`.
- Oracle **updates predefined roles every quarter** (quarterly updates).

### 6.1 Copy Role: the two options (very common interview question)

| Option | What gets copied | Inherited roles point to | Good | Bad |
|---|---|---|---|---|
| **Copy top role** | Only the top role | Oracle's **original** duty roles | Keeps receiving Oracle's quarterly updates **inside** those inherited duties | You can't change the inherited duty roles themselves |
| **Copy top role and inherited roles** | Top role + every role under it | **New custom copies** | You can change anything in the hierarchy | No more Oracle updates for any of the copies; more to maintain |

**Gotchas:**
- With **Copy top role**, if Oracle adds a privilege **inside an inherited duty**, you get it. If Oracle adds a **brand-new duty to the predefined top role**, your copy **doesn't** get it, because your top role's hierarchy is yours now.
- After every quarterly update, use **Security Console → Compare Roles** to compare your custom role with the updated predefined one.
- After creating or copying roles, run **Import User and Role Application Security Data** so the new role is available to HCM (role mappings, data roles) and BI.
- Data access is set up **separately** from the Security Console copy. After copying a job role, set up its data role or security profiles for the **new** role through *Assign Security Profiles to Role*.

---

## 7. HCM security profiles

### 7.1 Security profile types (object types)

| Profile type | Secures | Example |
|---|---|---|
| **Person** | Persons you can **manage** (edit, transact on) | Persons in India legal employer |
| **Public Person** | Persons you can **see/search** in directories and lists of values | All workers (so you can pick a manager from another country) |
| **Organization** | Departments, BUs, legal employers | India organizations only |
| **Position** | Positions | Positions in Finance BU |
| **Legislative Data Group (LDG)** | LDGs (payroll data partition) | India LDG |
| **Payroll** | Payrolls | India Monthly payroll |
| **Payroll Flow** | Payroll flows | Flows of India payrolls |
| **Document Type** | Document records types | Passports, visas |
| **Country** | Countries | India |

Others exist for specific products (e.g. job requisitions in Recruiting).

### 7.2 Person vs Public Person (classic question)
- **Person security profile** = whose records you can **manage**. HR Specialist – India can edit only Indian employees.
- **Public Person security profile** = whose records you can **see** in directories, searches and lists of values. During a hire in India, HR may need to pick a **manager based in the US**.

### 7.3 Ways to secure persons in a person security profile
- **By area of responsibility (AOR)**: dynamic. HR rep is assigned responsibility for a department, location, BU or legal employer, and sees the people in it. When responsibilities change, access changes automatically.
- **By manager hierarchy**: line managers see their direct and indirect reports.
- **By workforce structures**: through organization, position or legal employer criteria.
- **By custom criteria**: SQL-based conditions for unusual rules.
- **Own record handling**: you can control access to the user's own record, so an HR specialist can't edit **their own salary**. This is SoD in HCM.

### 7.4 Organization security profile
Secure organizations by **organization hierarchy**, by a **list of organizations**, or by **classification** (e.g. all departments).

### 7.5 Predefined "View All" profiles
Oracle provides profiles like **View All People**, **View All Organizations**, **View All Positions**. They're handy for admins, but dangerous on normal users. If a user's profile is View All, even secured views in BI will return everyone.

---

## 8. Financials (ERP) data access: different from HCM

HCM secures data with **security profiles**. Financials and procurement mostly secure data by **business unit, ledger or data access set**, assigned per **user + role**:

- Task: **Manage Data Access for Users**. You give a user a role *for* a specific BU or ledger, e.g. *Accounts Payable Specialist* for *India BU*.
- Stored in **`FUN_USER_ROLE_DATA_ASGNMNTS`** (user GUID, role, BU/org, ledger, active flag...).
- That's why the same AP specialist can see invoices of India BU but not US BU.

| | HCM | Financials / P2P / O2C |
|---|---|---|
| Data scope defined by | Security profiles (person, org, position...) | BU / ledger / data access set per user + role |
| Setup task | Assign Security Profiles to Role (data roles) | Manage Data Access for Users |
| BI Publisher filter | Secured list views (`PER_..._SECURED_LIST_V`) | `FUN_USER_ROLE_DATA_ASGNMNTS` + `PER_USERS` + `:xdo_user_name` |

---

## 9. Users and role provisioning

### 9.1 How users get created
- On **hire**, Fusion can create the user account automatically. This is controlled by **User and Role Provisioning** settings in **Manage Enterprise HCM Information** (create accounts automatically, provision roles, and so on).
- User name rules and password policy are managed in **Security Console → User Categories**.
- If a user isn't created, common reasons are missing work email (if the user name is email-based), provisioning switched off, or requests stuck in LDAP.

### 9.2 Role provisioning and role mappings
**Role provisioning** = automatically giving roles to users using **role mappings**.

A **role mapping** says: *if a person's assignment matches these conditions, then give them these roles.*

A role mapping has three parts:
1. **Conditions**: assignment attributes such as legal employer, BU, department, job, position, grade, location, assignment type, system person type, assignment status, **Manager with Reports**, manager type, etc.
2. **Roles**: which roles to give.
3. **Role provisioning options**:

| Option | Meaning |
|---|---|
| **Autoprovision** | Role is **added automatically** when conditions match and **removed automatically** when they stop matching |
| **Requestable** | A manager or HR can add the role for someone who matches |
| **Self-requestable** | The user can ask for the role themselves |

### 9.3 Behavior rules (memorize these)
- **Autoprovisioned roles** are removed **as soon as** the person no longer meets the conditions (e.g. a transfer to another department).
- **Manually added roles** are **not** removed on a transfer. They're removed automatically only when **all work relationships are terminated**.
- If a termination is **reversed**, autoprovisioned roles **come back**, but manually added roles **don't**. You have to add them again.
- Creating a **new** mapping doesn't automatically give roles to **existing** users. Run **Autoprovision Roles for All Users**, or use the autoprovision action on the mapping or on the individual user.
- **Line Manager** is usually autoprovisioned with the condition **Manager with Reports = Yes**. When the reports move away, the role drops off. If the client maps on the **manager flag** instead, the role stays until the flag is removed.
- For **terminated workers** who still need access (e.g. payslips), create a mapping on **system person type = Ex-Employee / assignment status = Inactive** with a limited custom role.

### 9.4 Key processes (ESS jobs)

| Process | When you need it |
|---|---|
| **Import User and Role Application Security Data** | After creating or copying roles, so HCM and BI see them |
| **Autoprovision Roles for All Users** | After creating or changing a role mapping, to apply it to existing users |
| **Send Pending LDAP Requests** | When user or role requests are stuck, or for future-dated hires |
| **User and Role Access Audit Report** | Auditing who has which roles and privileges |

### 9.5 Useful tables

| Table | Holds |
|---|---|
| `PER_USERS` | Users (USER_ID, USERNAME, PERSON_ID, USER_GUID) |
| `PER_USER_ROLES` | Which user has which role |
| `PER_ROLES_DN` / `PER_ROLES_DN_TL` | Role definitions and translated role names |
| `FUN_USER_ROLE_DATA_ASGNMNTS` | ERP data access (user + role + BU/ledger) |

```sql
-- Users and their roles (verify column names in your instance)
SELECT pu.username,
       prdt.role_name,
       prd.role_common_name
FROM   per_users        pu,
       per_user_roles   pur,
       per_roles_dn     prd,
       per_roles_dn_tl  prdt
WHERE  pu.user_id    = pur.user_id
AND    pur.role_id   = prd.role_id
AND    prd.role_id   = prdt.role_id
AND    prdt.language = USERENV('LANG')
AND    pu.active_flag = 'Y'
ORDER  BY pu.username;
```

---

## 10. Segregation of Duties (SoD)

**Segregation of duties** is a security principle that **prevents one person from controlling every stage of a business process**, which reduces the risk of fraud and errors.

**Classic P2P example:** one employee should **not** independently **create a supplier**, **approve the supplier**, **create an invoice** and **make the payment**. Otherwise they could pay a fake supplier.

**HCM examples:**
- An HR specialist shouldn't be able to **edit their own salary** (own-record handling in the person security profile).
- One person shouldn't both **hire** a worker and **run payroll** for them without review.

How it's enforced:
- **Role design**: keep conflicting duties in different roles.
- **Approvals/workflow**: the creator can't be the approver.
- **Risk Management / Advanced Access Controls**: tools that analyze role combinations for SoD conflicts.

---

## 11. BI security: three layers

```
Layer 1: CATALOG ACL     → Can the user SEE / OPEN / RUN the report object?
Layer 2: DATA SECURITY   → When it runs, WHICH rows come back?
Layer 3: DATA SOURCE     → Is the user allowed to use the JDBC data source?
```

### 11.1 BI Publisher bypasses data security ⚠️
- BIP data models run **plain SQL on base tables** through a **data source connection** (e.g. `ApplicationDB_HCM`, `ApplicationDB_FSCM`), **not as the logged-in user**.
- So a query on `PER_ALL_PEOPLE_F` returns **all records**, even ones the user isn't allowed to see.
- **You must add the security yourself** in the SQL.

**HCM: use the secured list views:**

| View | Use for |
|---|---|
| `PER_PERSON_SECURED_LIST_V` | Persons the user can access |
| `PER_ASSIGNMENT_SECURED_LIST_V` | Assignments the user can access |
| `PER_DEPARTMENT_SECURED_LIST_V` | Departments |
| `PER_POSITION_SECURED_LIST_V` | Positions |
| `PER_LEGAL_EMPLOYER_SECURED_LIST_V` | Legal employers |

These are **views** (not tables). They return only rows inside the **current user's** security profiles.

```sql
SELECT papf.person_number,
       ppnf.full_name
FROM   per_all_people_f   papf,
       per_person_names_f ppnf
WHERE  papf.person_id = ppnf.person_id
AND    ppnf.name_type = 'GLOBAL'
AND    TRUNC(SYSDATE) BETWEEN papf.effective_start_date AND papf.effective_end_date
AND    TRUNC(SYSDATE) BETWEEN ppnf.effective_start_date AND ppnf.effective_end_date
AND    EXISTS (SELECT 1
               FROM   per_person_secured_list_v sec
               WHERE  sec.person_id = papf.person_id);
```

**Financials (P2P / O2C): there are no HCM-style secured views**, so filter using the user's BU access:

```sql
-- Restrict AP invoices to BUs the logged-in user has access to
SELECT aia.invoice_num,
       aia.invoice_amount,
       aia.org_id
FROM   ap_invoices_all aia
WHERE  aia.org_id IN (
         SELECT furda.org_id
         FROM   fun_user_role_data_asgnmnts furda,
                per_users                   pu
         WHERE  furda.user_guid   = pu.user_guid
         AND    furda.active_flag = 'Y'
         AND    pu.username       = :xdo_user_name
       );
```

- `:xdo_user_name` is a **built-in BIP system variable**: the user name of the person running the report.
- `:xdo_user_name` + `PER_USERS.PERSON_ID` is also how you build **"my own data"** reports (e.g. an employee viewing only their own record).

**Scheduling gotcha:** a scheduled or burst report runs with the security context of the **user who scheduled it**, not each recipient.

### 11.2 OTBI applies security automatically
- OTBI (Analyses/Dashboards on **subject areas**) identifies the **logged-in user's roles** and applies **function security** (which subject areas they can use) and **data security** automatically.
- That's why OTBI is "secure by default" and BIP is "secure only if you build it in."

| | BI Publisher | OTBI |
|---|---|---|
| Source | SQL on base tables | Subject areas (semantic layer) |
| Data security | ❌ Not automatic: add secured views / filters | ✅ Automatic from user's roles |
| Best for | Pixel-perfect, bursting, outbound files, complex SQL | Self-service analysis, dashboards |

### 11.3 Access Control List (ACL) on catalog objects
An **ACL** is a list of **accounts (roles/users) + permissions**, stored **on each catalog object** (folder, report, data model). To access an object, the user needs the right entry in its ACL.

**ACL vs data security:**
- ACL on a folder: **Can** the user open the *Payroll Register* report?
- Data security: when they run it, **which employees'** records do they see?

**Permission bundles:**

| Bundle | What it allows |
|---|---|
| **Full Control** | Everything, including changing permissions and ownership |
| **Modify** | Read, write, delete |
| **Open** | Read and use (open/view; on folders, browse into them) |
| **No Access** | Explicit block. **It overrides other grants.** |
| **Custom** | Pick individual permissions |

**Granular permissions worth knowing:** Read, **Traverse**, Write, Delete, Change Permissions, Set Ownership, **Run Publisher Report**, **Schedule Publisher Report**, **View Publisher Output**.

**Key gotchas:**
- A user can **see** a report but can't **run** it without **Run Publisher Report**.
- A user with access to a report still can't **reach** it without **Traverse** on every **parent folder**.
- The report needs access to its **data model** too. Running the report reads the data model.
- **No Access** from any role beats a grant from another role.
- **Best practice:** grant permissions to **roles**, not individual users. Keep custom content under **`/Shared Folders/Custom`**, because predefined folders can be overwritten during updates.

### 11.4 Migrating reports and what happens to the ACL
Reports are built in **DEV**, then moved to **TEST** and **PROD** using **Archive → .catalog file → Unarchive**.

When archiving, tick **Keep permissions** (and optionally Keep timestamps) if you want the ACL carried with the object.

The **ACL option in the Unarchive dialog** decides what happens in the new folder:

| Option | Meaning |
|---|---|
| **Inherit** (default) | Takes permissions from the **new parent folder** |
| **Preserve** | Keeps the **original** permissions, mapping accounts where they exist |
| **Create** | Like Preserve, but **creates and maps** missing accounts as needed |

**Gotchas:**
- Preserve and Create only work if the archive was created with **Keep permissions**.
- Migrate **both the report and its data model**, and make sure the **data source** exists in the target.

---

## 12. Troubleshooting playbook (symptom → where to look)

| Symptom | Most likely layer | Check |
|---|---|---|
| User can't sign in at all | User account | Exists? Suspended? Pending LDAP? User category or password policy? |
| New hire has **no roles** | Provisioning | User exists → pending LDAP → future-dated hire → mapping conditions vs assignment → Autoprovision ticked → mapping start date → role imported → run Autoprovision Roles |
| Can't see a **menu / task / page** | Function security | Role missing the privilege or aggregate privilege |
| Can open page but **search returns nothing** | Data security | No security profile (abstract role without profiles, or data role missing a profile type) |
| Sees **too much** data | Data security | View All profile, profile assigned directly to job role *and* data role, or BIP SQL without secured views |
| Can't pick a person from another country in a list of values | Public person profile | Public person security profile too narrow |
| Legal employer missing on **Hire** screen | Setup / data security | Not designated legal employer, effective date, organization security profile |
| Role gone after **transfer** | Provisioning | It was autoprovisioned and conditions no longer match |
| Role **didn't come back** after termination reversal | Provisioning | It was manually added. Add it again |
| New custom role not in role mapping or data role lists | Sync | Run Import User and Role Application Security Data |
| Report visible but **Run** fails | BI ACL / data source | Run Publisher Report, data model access, data source access |
| Report **not visible** | BI ACL | Traverse on parent folders, No Access override, role not synced to BI |
| BIP shows **all countries** | BIP SQL | Base tables without secured list views / BU filter |
| AP report shows **other BUs' invoices** | BIP SQL | No `FUN_USER_ROLE_DATA_ASGNMNTS` + `:xdo_user_name` filter |

---

## 13. Scenario questions with model answers

### Q1. A new HR Specialist in Pune joined yesterday. The role mapping exists, but she has no roles. What do you check, in order?
1. **User account exists?** Security Console → Users. If not, check *User and Role Provisioning* settings and whether user name generation failed (e.g. missing email).
2. **Pending LDAP requests?** Run or check **Send Pending LDAP Requests**.
3. **Future-dated hire?** Roles are provisioned only when the start date arrives.
4. **Mapping conditions vs her assignment**: legal employer, BU, department, job, location, assignment type, person type, assignment status. Is **Autoprovision** ticked? Does the **mapping start date** come before her hire?
5. **Role synced?** For a new custom role: *Import User and Role Application Security Data*.
6. **Mapping newer than the hire?** Run **Autoprovision Roles for All Users** (or autoprovision for her user).

> "First I confirm the user exists and nothing is stuck in LDAP, then whether the hire is future-dated, then I compare every mapping condition with her assignment, and finally whether autoprovisioning needs to be rerun."

### Q2. A retiring employee must view payslips for one year after termination. Design it.
- Custom **abstract role** (e.g. `XX_FORMER_EMPLOYEE`) with **only** the payslip-viewing access, not the full Employee role.
- **Role mapping** for terminated workers: system person type = Ex-Employee, assignment status = Inactive, Autoprovision.
- Data scope: **own record only**.
- Login: work email or SSO may be disabled, so use a **personal email** for user name and password reset.
- **One-year limit:** a role mapping can't express "terminated within the last 12 months." Use a periodic process (scheduled report or HCM Extract to find users terminated over a year ago, then remove the role or suspend the user manually or through HDL).

### Q3. Custom BIP report looks right for you, but an India HR user sees all countries. Why and how to fix?
- BIP SQL on **base tables** ignores data security, because it runs through the data source, not as the user. It looked right for you because you probably have broad access.
- Fix: add `EXISTS` on **`PER_PERSON_SECURED_LIST_V`** / **`PER_ASSIGNMENT_SECURED_LIST_V`** (see 11.1).
- If it still shows everything, check the user's **person security profile**. It may be View All.
- Scheduled reports run as the **scheduler's** security.

### Q4. You used **Copy top role** on HR Specialist and removed a duty. Oracle adds a privilege to an inherited duty next quarter. Do you get it? What about a full copy?
- **Copy top role:** **Yes.** Your role still points to Oracle's duty, so privilege changes inside it flow through. The removed duty stays removed. But a **new duty added to the predefined top role** won't reach your copy. Find it with **Compare Roles**.
- **Copy top role and inherited roles:** **No.** All duties are your copies. Compare and add changes manually every quarter.

### Q5. One user sees the report but Run fails. Another has Open on the report but can't see it at all.
- **Run fails:** missing **Run Publisher Report**, no access to the **data model**, or no access to the **data source**.
- **Can't see it:** no **Traverse** on a **parent folder**, an explicit **No Access** from another role, or the role isn't synced to BI (Import User and Role Application Security Data, then sign in again).

### Q6. A line manager is promoted to a role with no direct reports. Does he lose Line Manager?
- **It depends on the mapping.**
  - Mapping on **Manager with Reports = Yes**: once reports are reassigned, the condition fails → role **removed automatically**.
  - Mapping on the **manager flag**: he **keeps** it until the flag is removed.
  - Role **added manually**: it **stays** (manual roles are removed only when all work relationships end).

---

## 14. Rapid-fire Q&A

| Question | Answer |
|---|---|
| What's RBAC? | Access is given through roles, never directly to users |
| Function vs data security? | What you can do vs which records you can do it on |
| Which roles can be assigned to users? | Job, abstract, data roles. **Not** duty roles or aggregate privileges |
| Can you create an aggregate privilege? | No. Predefined only; can't create, edit or copy |
| Can aggregate privileges inherit roles? | No. They're always leaves |
| What can a duty role inherit? | Other duties and aggregate privileges |
| How many job roles in an HCM data role? | Exactly one |
| How many security profiles per object type in a data role? | One |
| Any predefined HCM data roles? | No. They're enterprise-specific |
| Security profiles on job roles directly? | Possible, but **Oracle recommends against it** |
| Security profiles on abstract roles? | Yes, that's how Employee / Line Manager get data access |
| Person vs Public Person? | Manage vs see/search |
| How do you identify predefined roles? | `ORA_` prefix on the role code |
| Can you edit predefined roles? | No. Copy and edit the copy |
| Autoprovisioned role on transfer? | Removed if conditions no longer match |
| Manual role on transfer? | Stays. Removed only when all work relationships end |
| Termination reversed? | Autoprovisioned roles return; manual ones don't |
| New mapping, existing users? | Run Autoprovision Roles for All Users |
| New custom role not visible in HCM/BI? | Run Import User and Role Application Security Data |
| Why secure BIP manually? | SQL on base tables bypasses Fusion data security |
| HCM secured views? | `PER_PERSON_SECURED_LIST_V`, `PER_ASSIGNMENT_SECURED_LIST_V`, etc. |
| ERP data access table? | `FUN_USER_ROLE_DATA_ASGNMNTS` |
| Current BIP user variable? | `:xdo_user_name` |
| OTBI security? | Automatic, based on the logged-in user's roles |
| ACL? | Accounts + permissions stored on each catalog object |
| Unarchive ACL options? | Inherit (default), Preserve, Create |
| SoD? | No single person controls every stage of a process |

---

## 15. Your 60-second pitch on Fusion security

> "Oracle Fusion uses role-based access control. Users get roles, and roles carry access. There are two kinds of security. Function security, from privileges inside roles, decides what a user can do. Data security, from data security policies and, in HCM, security profiles, decides which records they can do it on.
>
> The hierarchy starts with privileges and data security policies. Aggregate privileges bundle one privilege with its data policies. Duty roles group multiple privileges. Job and abstract roles sit on top and are what we assign to users. In HCM we create a data role from one job role plus security profiles, one per object type. Abstract roles like Employee get security profiles directly.
>
> Predefined roles start with ORA_ and can't be edited, so we copy them. Copy top role keeps Oracle's quarterly updates in the inherited duties, while a full copy gives complete control but no updates. Roles are provisioned automatically through role mappings based on assignment conditions. Autoprovisioned roles are removed when conditions stop matching, while manual roles stay until all work relationships end.
>
> For reporting, OTBI applies security automatically, but BI Publisher SQL bypasses it. So in HCM we use secured list views, and in Financials we filter on the user's BU access using FUN_USER_ROLE_DATA_ASGNMNTS and :xdo_user_name. Report objects themselves are protected by catalog ACLs, and when migrating we choose whether ACLs are inherited, preserved or created. Throughout, we design roles to respect segregation of duties."

---

*Tip: before the interview, open the Security Console in your training instance and look at the HR Specialist role hierarchy, then one role mapping, then one data role. Seeing them once makes every answer above concrete.*