# Security Console, Users & Roles — Oracle Fusion Core HR

## 1. The big picture: how security works in Fusion HCM

In one line: **a user can only do something if a role gives them the action (function security) AND a security profile gives them the data (data security).** Remember this. Almost every tricky interview question comes back to it.

Fusion uses **RBAC (Role-Based Access Control)**. You never give access to a person directly. You give them roles, and roles carry the access.

### The two kinds of security

| Type | Answers the question | Example | Comes from |
| --- | --- | --- | --- |
| Function security | "What can I DO?" (pages, buttons, tasks) | Can open Hire an Employee | Privileges inside roles |
| Data security | "On WHICH records?" | Can hire only in India legal employer | Data security policies + HCM security profiles |

A simple example: an HR Specialist in India can open the Hire page (function), but should only see Indian employees (data). Miss either one and the user is stuck.

### The 4 role types (learn these cold)

| Role type | What it is | Example | Can you assign it to a user? |
| --- | --- | --- | --- |
| Abstract role | Based on who you are, not your job | Employee, Line Manager, Contingent Worker | Yes |
| Job role | Based on the job you do | Human Resource Specialist, HR Analyst | Yes |
| Duty role | A group of related tasks inside a job | Manage Worker Documents duty | No, it only sits inside job/abstract roles |
| Data role | A job role + a data scope | HR Specialist - India | Yes |

**Privileges** sit at the bottom. A privilege is one single action, like "Hire Worker". Duty roles bundle privileges, and job/abstract roles bundle duty roles.

Think of it like a company ID card. The job role is your designation. Duty roles are your responsibilities. Privileges are the individual doors your card opens. The security profile decides which floors those doors are on.

&#91;embedded content: role hierarchy · user to privileges, plus the data side\]

Read it top down: the user holds roles, a job role inherits duty roles, and duty roles hold privileges. The dashed link is the data side; without it the user opens the page but sees no records.

### Predefined vs custom roles

- Oracle ships **predefined roles**. Their role codes start with `ORA_` (for example `ORA_PER_HUMAN_RESOURCE_SPECIALIST_JOB`).
- You **cannot edit** predefined roles. To change one, you **copy** it and edit the copy. That copy is your **custom role**.
- Oracle updates predefined roles every quarterly release. This matters a lot for copied roles (see section 4).

## 2. Security Console: a quick tour

The Security Console is the one screen where you manage roles and users for all of Fusion (HCM, Finance, SCM, everyone).

**How to open it:** Navigator → Tools → Security Console. You need the **IT Security Manager** role (`ORA_FND_IT_SECURITY_MANAGER_JOB`) to use it.

### The tabs you will see

| Tab | What you do there |
| --- | --- |
| Roles | Search roles, see a role's hierarchy, copy, edit, compare, create roles, simulate the navigator |
| Users | Search users, add user accounts, edit roles on a user, reset passwords, lock/unlock |
| User Categories | Group users and set rules per group: user name format, password policy, notifications |
| Analytics | Reports on roles and users, e.g. which privileges a role has, which users hold a role |
| Certificates | Manage security certificates used for integrations and encryption |
| Single Sign-On | Set up SSO with the client's identity provider |
| Administration | General settings, e.g. default prefix/suffix for copied roles, password and user name policies |

Your client's pod may show a slightly different set depending on the release, but these are the main ones.

### Handy features on the Roles tab

- **Visualize hierarchy:** shows a role as a tree (graph) or a table, so you see all inherited duty roles and privileges.
- **Copy Role:** the safe way to customise a predefined role.
- **Compare Roles:** puts two roles side by side. Great after a quarterly update to see what Oracle added to the original that your copy is missing.
- **Simulate Navigator:** shows which menu items a role would see, without logging in as that user. Very useful for testing.
- **Create Role:** build a new role from scratch (rare in HCM; copying is more common).

### Handy features on the Users tab

- **Add User Account:** create a user directly.
- **Edit:** add or remove roles, change the user name, lock or unlock the account.
- **Reset Password:** sends a reset or lets you set a temporary one.
- **User's roles and privileges:** see everything a user can do, through which role.

## 3. Creating users

In Core HR, most users are created **automatically when you hire someone**. Manual creation is mostly for special users like implementers or integration accounts.

### Linked user vs standalone user

|  | Linked user | Standalone user |
| --- | --- | --- |
| Has a person record? | Yes (employee, contingent worker, etc.) | No |
| Typical use | Real employees | Implementation users, integration users, external support |
| Gets roles by | Role mappings (auto) + manual | Manual only |
| Can use self-service, be in manager hierarchy? | Yes | No |

### The ways to create a user

1. **Hire a person (most common).** If user account creation is switched on, Fusion creates the user when the hire is saved. Roles come from role provisioning rules.
2. **Security Console → Users → Add User Account.** Fill in name, email, user name, password and roles. You can create it with no linked person, or with a basic worker record.
3. **Bulk import.** Security Console's Import Users option, or **HCM Data Loader (HDL)** with the `User.dat` file (for creating accounts for existing persons, changing user names, adding/removing roles in bulk). Hires done through `Worker.dat` also create users if the setting is on.
4. **REST/SCIM APIs** for integrations with an outside identity system.

### The setting that controls automatic user creation

Setup and Maintenance → **Manage Enterprise HCM Information** → User and Role Provisioning section. The key options:

- **User Account Creation:** create a user automatically on hire (on or off).
- **User Account Role Provisioning:** add/remove roles automatically based on role mappings.
- **User Account Maintenance:** what happens to accounts when people leave, e.g. suspend the account on termination.
- **User Account Creation for Terminated Workers:** whether to still create users for people hired and terminated quickly (for example, in a data load of past employees).

### User name and password rules

- The **user name format** is set in **Security Console → User Categories** (for example: email, first.last, person number, or a custom Fast Formula).
- User names must be **unique**. Once a user name exists, you cannot reuse it for someone else, even if the first user is inactive.
- The **password policy** (length, complexity, expiry) is also set per user category.
- Welcome emails with the user name and password reset link go to the work email. **No work email = no welcome email.**

### Step by step: add a user in Security Console

1. Navigator → Tools → Security Console → **Users** tab.
2. Click **Add User Account**.
3. Choose the associated person type (none, or worker).
4. Enter first name, last name, email, user name. Pick the user category.
5. Set a password (or let the system send a reset email).
6. Click **Add Role**, search and add the roles needed.
7. **Save and Close**.
8. Make sure the **Send Pending LDAP Requests** job runs, so the account is pushed to the identity store and can log in.

## 4. Creating custom roles

On real projects you almost always **copy** a predefined role and change the copy. Building from zero is rare and risky, because you might miss privileges Oracle silently needs.

### Copy Role: the two options (very common interview question)

| Option | What gets copied | Inherited roles point to | Good | Bad |
| --- | --- | --- | --- | --- |
| **Copy top role** | Only the top role | Oracle's original duty roles | You keep getting Oracle's quarterly updates to those duty roles | You can't change the inherited duty roles themselves |
| **Copy top role and inherited roles** | Top role + every role under it | New custom copies | You can change anything in the hierarchy | You stop getting Oracle updates for all those copies; more to maintain |

**Rule of thumb:** choose **Copy top role** unless you really need to change something inside an inherited duty role.

### Step by step: copy and edit a role

1. Security Console → **Roles** tab → search for the role, e.g. *Human Resource Specialist*.
2. Click the action menu next to it → **Copy Role**.
3. Pick **Copy top role** (or top role and inherited roles).
4. The copy opens in a step-by-step train:
   1. **Basic Information:** role name, role code, role category (e.g. *HCM - Job Roles*), description. By default the copy gets a suffix like `_CUSTOM`.
   2. **Function Security Policies:** add or remove privileges (what they can do).
   3. **Data Security Policies:** add or remove data rules on database objects (which data).
   4. **Role Hierarchy:** add or remove inherited duty roles.
   5. **Segregation of Duties:** see if the role creates risky combos.
   6. **Users:** optionally add users to the role right here.
   7. **Summary:** review and **Submit and Close**.
5. Run **Import User and Role Application Security Data** so the new role shows up in other screens (like role mappings and security profile assignment).

### Naming tips

- Never start a custom role code with `ORA_`. That prefix is reserved for Oracle.
- Use a clear client prefix, e.g. `XX_HR_SPECIALIST_IN` / "XX HR Specialist India".
- Keep the name and code consistent; teams search by both.

### Removing access from a role

To take a menu away, you usually remove the **privilege** (function security). Hiding a page by personalisation is not security; the user can still reach it another way.

### Keep your custom roles healthy after each quarterly update

Oracle adds new privileges to predefined roles every release. Your copies don't get them automatically (only the parts that still point to Oracle's originals do). After each update, use **Compare Roles** (original vs your copy) and add anything new you need.

## 5. Data access: security profiles, data roles and AOR

Function security lets a user open the page. **Security profiles** decide which records show up on it. In HCM, data security is mostly done with security profiles.

### Main HCM security profiles

| Security profile | Controls access to | Common ways to scope it |
| --- | --- | --- |
| Person | Person and worker records you can manage | By manager hierarchy, area of responsibility, organization, position, legal employer, custom SQL |
| Public person | People you can only search and view basic info for (directory, picking a manager or approver) | Usually "view all" |
| Organization | Departments, legal employers, business units | Organization hierarchy or a list |
| Position | Positions | Position hierarchy or a list |
| Legislative Data Group (LDG) | Payroll-related data per country | List of LDGs |
| Country | Country-specific data | List of countries |
| Document type | Document records (e.g. visas, certificates) | List of document types |
| Payroll / Payroll flow | Payrolls and payroll flows | List |

Oracle gives some ready-made ones like **View All People**, **View All Organizations** and **View Own Record** (used by the Employee role).

A **person security profile can include an organization or position security profile inside it**. For example: "people in departments under the India org tree".

### Two ways to give data access to a job role

1. **Data role (older, classic way).** Create a data role like *HR Specialist - India* that combines the job role *Human Resource Specialist* with India security profiles. You create a separate data role for each scope (India, US, UK...).
2. **Assign security profiles directly to the job/abstract role (newer, more common now).** Use the task **Assign Security Profiles to Role**. Here the role itself carries the profiles, and the profiles are often dynamic (for example, based on AOR), so one role works for every country. Fewer roles to manage.

Both use the same task area in Setup and Maintenance (Assign Security Profiles to Role).

### Area of Responsibility (AOR)

AOR is how you say "Priya is the HR rep for the Chennai location". You assign it on the person's record: **My Client Groups → Person Management → (person) → Areas of Responsibility**.

An AOR has a **responsibility type** (e.g. HR Representative) and a **scope** (department, legal employer, location, business unit, etc.). Then a person security profile "secured by area of responsibility" gives Priya access to exactly those people. When her AOR changes, her data access changes too. No new role needed.

### Manager-based access

A person security profile secured by **manager hierarchy** lets managers see their reports. Key points:

- You choose the manager type (line manager, or others) and how many levels down.
- It depends on the **Refresh Manager Hierarchy** job. If that job is not scheduled, new reporting lines won't show up for managers.

### Quick note on ERP data access

For finance-type roles (business units, ledgers), access is given with **Manage Data Access for Users** in Setup and Maintenance. HCM data mostly uses security profiles instead. Good to know if the interviewer mixes ERP and HCM.

## 6. Role provisioning: how users get roles automatically

Role provisioning means **giving roles to users**. In HCM it's mostly automatic, using **role mappings** (also called role provisioning rules).

### Role mappings

**Where:** Setup and Maintenance → **Manage HCM Role Provisioning Rules** (search the task name).

A role mapping says: *"If a person's assignment matches these conditions, give them these roles."*

- **Conditions** come from the assignment: legal employer, business unit, department, job, position, grade, location, assignment type, assignment status, system person type, user person type, responsibility (AOR), and more.
- For each role in the mapping you tick one or more options:

| Option | Meaning |
| --- | --- |
| Autoprovision | Role is added automatically when conditions match, and removed when they stop matching |
| Requestable | A manager or HR can add this role for someone who matches |
| Self-requestable | The user can ask for this role themselves |

**Example mapping:** Name *Employee Role*, condition *System Person Type = Employee* and *Assignment Status = Active*, role *Employee* with Autoprovision ticked. Now every active employee gets the Employee abstract role.

### Auto vs manual roles: what happens later

- **Autoprovisioned roles** are removed **as soon as** the person no longer meets the conditions (e.g. transfer to another department).
- **Manually added roles** are **not** removed on a transfer. They are removed automatically only when **all** work relationships are terminated.
- If a termination is **reversed**, autoprovisioned roles come back, but manually added roles **do not**. You have to re-add them.

### Key jobs (ESS processes) you must know

| Job | What it does | When to run it |
| --- | --- | --- |
| Send Pending LDAP Requests | Pushes user and role requests to the identity store, including future-dated ones | Schedule it often (e.g. every few hours/daily); run after bulk changes |
| Retrieve Latest LDAP Changes | Pulls changes made directly in the identity store back into Fusion | When user/role data looks out of sync |
| Autoprovision Roles for All Users (Apply Autoprovisioning) | Re-checks all role mappings for all users | After creating or changing a role mapping |
| Import User and Role Application Security Data | Copies users, roles and memberships into Fusion's own tables | After creating custom roles, so they appear in lists of values |
| Refresh Manager Hierarchy | Rebuilds manager-report relationships for manager-based security | Schedule daily |

### Useful reports

- **User and Role Access Audit Report:** who has which role/privilege.
- **User Role Membership Report:** users per role.
- **Inactive Users Report:** users not logging in (audit reviews).
- Security Console **Analytics** tab for quick role and user analysis.

## 7. Edge cases and troubleshooting (real project problems)

When something breaks, ask three questions in this order: **Does the user exist and can they log in? Do they have the right role (function)? Do they have the right data (security profile)?**

| # | Problem | Likely cause | Fix |
| --- | --- | --- | --- |
| 1 | New hire has no user account | User Account Creation is off, or the job didn't run | Check Manage Enterprise HCM Information; run Send Pending LDAP Requests; check its output for errors |
| 2 | User exists but can't log in | Account locked, password expired, or request still pending in LDAP | Unlock / reset password in Security Console; check Send Pending LDAP Requests |
| 3 | Future-dated hire (joins next month) has no roles yet | Normal: roles are given on the start date | Make sure Send Pending LDAP Requests is scheduled; it processes future-dated requests |
| 4 | User has the role but menu/tile is missing | Role lacks the privilege, or session is old | Sign out and in; check privilege in role hierarchy; use Simulate Navigator |
| 5 | User opens the page but search shows no people | Function OK, data missing: no security profile, wrong AOR, or role has no profiles assigned | Check Assign Security Profiles to Role / data role; check the person's AOR |
| 6 | Manager can't see a new direct report | Manager hierarchy not refreshed | Run Refresh Manager Hierarchy; check manager type and levels in the profile |
| 7 | HR rep sees everyone in the company | Role uses View All People profile | Assign a scoped person security profile (AOR, org or legal employer based) |
| 8 | New custom role not in the role mapping list | Fusion tables not refreshed | Run Import User and Role Application Security Data |
| 9 | Role mapping created but nobody got the role | Mapping only applies on the next change to each person | Run Autoprovision Roles for All Users (Apply Autoprovisioning), then Send Pending LDAP Requests |
| 10 | Employee transferred, still has old department's role | That role was added manually, not by mapping | Remove manually, or change it to an autoprovisioned role |
| 11 | Terminated worker can still log in | Termination is future-dated, or User Account Maintenance doesn't suspend users | Check termination date and the setting; run Send Pending LDAP Requests |
| 12 | Termination reversed, user missing some roles | Manually added roles don't come back | Re-add manual roles; autoprovisioned ones return on their own |
| 13 | Rehired employee gets a "user already exists" type issue | Rehire reuses the old user account and user name | Don't create a new user; reactivate/check the existing account |
| 14 | Can't create user, user name taken | User names are unique forever, even for inactive users | Use a different name or follow the naming rule's tie-break (e.g. add a number) |
| 15 | New hire got no welcome email | No work email, or notification is off for that user category | Add email; check user category notification settings |
| 16 | After quarterly update, custom role users lose a new feature | Copied roles don't get Oracle's new privileges | Compare Roles (original vs copy) and add missing privileges |
| 17 | Person with two assignments gets roles from only one | Role mapping matches per assignment | Check each assignment's attributes against the mapping conditions |
| 18 | You try to edit a predefined role | Not allowed | Copy it first, then edit the copy |
| 19 | You try to give a duty role directly to a user | Duty roles can't be assigned to users | Put the duty role inside a custom job role, then assign that |
| 20 | Integration user needs access but has no person record | Standalone user, no HCM person | Give roles manually; data profiles that don't depend on manager/AOR |

### Testing tips that save you

- Always test with a **real test user**, not your admin account. Admin sees everything and hides problems.
- Use **Simulate Navigator** for menus and a test login for data.
- After any security change: sign out, sign back in, and check both the menu **and** the search results.

## 8. Interview questions and answers

Answer like you've done it: say **where** (screen/task), **what** you'd do, and **why**. Short, clear answers beat long ones.

### Level 1: Basics

**Q1. What is the Security Console?** It's the central screen in Fusion to manage users and roles: create/copy roles, see role hierarchies, add users, assign roles, reset passwords. Navigator → Tools → Security Console. You need the IT Security Manager role.

**Q2. What is RBAC?** Role-Based Access Control. Access is given through roles, never directly to a person. Users get roles; roles carry privileges and data access.

**Q3. Name the role types in Fusion.** Abstract roles (Employee, Line Manager), job roles (HR Specialist), duty roles (groups of tasks inside a job role), and data roles (job role + data scope). Privileges sit at the bottom.

**Q4. Function security vs data security?** Function security = what you can do (open a page, click Hire). Data security = which records you can do it on (only India employees). A user needs both.

**Q5. What is a privilege?** The smallest unit of access. One action, like "Hire Worker" or "Manage Person Documentation".

**Q6. Can you edit a predefined role?** No. Predefined roles (codes starting `ORA_`) are locked. You copy them and edit the copy.

**Q7. Abstract role vs job role?** Abstract role is based on who you are (an employee, a manager). Job role is based on what job you do (HR Specialist). Every employee gets Employee; only HR staff get HR Specialist.

**Q8. How is a user created when you hire someone?** If User Account Creation is on (Manage Enterprise HCM Information), the hire creates the user automatically. Role mappings give the roles. Send Pending LDAP Requests pushes it to the identity store.

### Level 2: Intermediate

**Q9. Copy top role vs copy top role and inherited roles?** Top role only: inherited duty roles still point to Oracle's originals, so you keep getting Oracle updates. Top + inherited: everything is copied, you can change anything, but you stop getting Oracle updates for those copies. Default choice: copy top role only.

**Q10. What is a role mapping?** A rule that gives roles based on assignment conditions (legal employer, job, department, person type, assignment status, etc.). Each role can be autoprovisioned, requestable or self-requestable. Task: Manage HCM Role Provisioning Rules.

**Q11. Autoprovision vs requestable vs self-requestable?** Autoprovision: added and removed automatically. Requestable: HR/manager can add it for eligible people. Self-requestable: users ask for it themselves.

**Q12. What is a security profile? Name a few.** A set of rules that defines which data a role can access. Person, public person, organization, position, LDG, country, document type, payroll, payroll flow.

**Q13. Person vs public person security profile?** Person = people whose records you can manage. Public person = people you can find and see basic info on, like in the directory or when picking an approver or manager.

**Q14. What is AOR?** Area of Responsibility. You tell Fusion "this person is the HR rep for this department/legal employer/location". A person security profile secured by AOR then gives them access to exactly those people. Change the AOR, the access changes. No new role needed.

**Q15. Data role vs assigning security profiles directly to a job role?** Data role: one role per scope (HR Specialist India, HR Specialist US), more roles to maintain. Direct assignment: profiles attached to the job role itself, usually dynamic (AOR or manager based), so one role covers everyone. Most new projects prefer the direct approach.

**Q16. Which jobs do you run after creating a custom role and a role mapping?** Import User and Role Application Security Data (so the role shows up in lists), then Autoprovision Roles for All Users (to apply the mapping), then Send Pending LDAP Requests.

**Q17. Linked user vs standalone user?** Linked has a person record (real employee). Standalone has none (implementation or integration users). Standalone users can't use self-service or be part of the manager hierarchy, and get roles only manually.

### Level 3: Tricky / scenario-based

**Q18. A user can open the Person Management page but search shows nothing. Why?** Function security is fine; data security is missing. Check the person security profile on their role (or data role), their AOR if the profile uses AOR, and that the role actually has profiles assigned.

**Q19. A manager can't see a newly hired direct report. What do you check?** First the assignment's manager field is correct. Then run Refresh Manager Hierarchy. Then check the person security profile's manager type and number of levels.

**Q20. An employee transferred to Finance but still has the HR Specialist role. Why?** The role was added manually. Manual roles don't drop on transfer; only autoprovisioned ones do. Remove it manually, and ideally move it to an autoprovisioned mapping.

**Q21. A termination was reversed. What happens to roles?** Autoprovisioned roles come back automatically. Manually added roles don't; they must be re-added.

**Q22. A hire is dated 1st of next month. Will the user have roles today?** The user account can exist, but roles become active on the start date. Send Pending LDAP Requests must be scheduled to process those future-dated requests.

**Q23. You created a role mapping yesterday. Old employees still don't have the role. Why?** Mappings apply when a person's assignment changes. For existing people, run Autoprovision Roles for All Users, then Send Pending LDAP Requests.

**Q24. After the quarterly update, users with your custom role are missing a new Oracle feature. Why and what do you do?** Oracle added the new privilege to the predefined role, but your copy didn't get it (fully copied parts don't get updates). Use Compare Roles between original and copy, then add the missing privilege.

**Q25. Client wants HR reps in India to see only India employees and US reps only US. How would you design it?** Option A: two data roles (HR Specialist India, HR Specialist US) with country-specific person profiles. Option B (cleaner): one custom HR Specialist role with a person security profile secured by AOR, and assign each rep an AOR for their legal employer. Option B scales better as new countries are added.

**Q26. Can you give a duty role directly to a user?** No. Duty roles are building blocks only. Wrap the duty role in a custom job role and assign that.

**Q27. How would you take a single menu item away from a role?** Copy the role (if predefined) and remove the privilege behind that menu item from function security. Don't rely on hiding it by personalisation; that's not security.

**Q28. A rehired employee needs access. Do you create a new user?** No. Rehire reuses the existing user account and user name. Check it's active and roles have been re-provisioned.

**Q29. How do you check what a user can actually do?** Security Console → Users → open the user → see roles and privileges. Use Simulate Navigator on the role, the User and Role Access Audit Report, or log in as a test user.

**Q30. A big data load of 5,000 historical (already terminated) workers. Do you want user accounts for them?** Usually no. Turn off User Account Creation for Terminated Workers in Manage Enterprise HCM Information before loading, so you don't create thousands of useless accounts.

### Bonus: questions to ask the interviewer

- Is the client using data roles, or security profiles assigned directly to job roles?
- Is SSO set up with their own identity provider?
- How do they handle role changes after each quarterly update?

Asking these shows you understand real project concerns.

## 9. Quick revision cheat sheet

Read this the night before the interview.

| Topic | Remember this |
| --- | --- |
| Golden rule | Access = role (what you can do) + security profile (which data) |
| Role layers | User → Job/Abstract role → Duty roles → Privileges |
| Assignable to users | Abstract, job, data roles. Not duty roles |
| Predefined roles | Code starts `ORA_`, can't edit, copy instead |
| Copy options | Top role only = keep Oracle updates. Top + inherited = full control, no updates |
| Security Console | Navigator → Tools → Security Console, needs IT Security Manager |
| Auto user creation | Manage Enterprise HCM Information → User and Role Provisioning |
| User name format | Security Console → User Categories |
| Role mappings | Manage HCM Role Provisioning Rules: autoprovision / requestable / self-requestable |
| Auto vs manual roles | Auto drop on change; manual drop only when all work relationships end |
| Termination reversal | Auto roles return, manual roles don't |
| Data access in HCM | Security profiles: person, public person, org, position, LDG, country, document type, payroll |
| AOR | Person-level responsibility + scope → dynamic data access |
| Push to identity store | Send Pending LDAP Requests (also handles future-dated) |
| New custom role not in lists | Import User and Role Application Security Data |
| New mapping, old users | Autoprovision Roles for All Users |
| Manager can't see reports | Refresh Manager Hierarchy |
| After quarterly update | Compare Roles: original vs custom copy |
| Troubleshooting order | Can they log in? → Right role? → Right data? |
