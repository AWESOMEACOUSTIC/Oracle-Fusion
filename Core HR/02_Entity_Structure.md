# Oracle Fusion Core HR — Enterprise Structure Study Guide

## How to use this guide

Enterprise Structure is the skeleton of Oracle Fusion HCM: every worker you hire, every payslip, every report hangs off it. If you can draw it on a whiteboard with a real company and explain why each box exists, you will clear most Core HR interviews.

Read it in this order:

1. **The Nike story** — the whole picture in plain words, the way you would explain it to an interviewer.
2. **Each component** — what it is, why it exists, one-line Nike example.
3. **Setup order and how to create each one** — the actual tasks you run in Setup and Maintenance.
4. **Edge cases** — the stuff that breaks on real projects (interviewers love these).
5. **Interview Q&A** — basic to tricky, with answers you can say out loud.
6. **Cheat sheet** — revise this the night before.

A quick note on the example: Nike is used only as a teaching story. The legal entity names, countries and numbers below are made up to explain the concepts — they are not Nike's actual Oracle setup.

## The Nike story: explaining Enterprise Structure to an interviewer

Think of Enterprise Structure as answering four questions about a company: **Who is the company? Who legally employs people? How is it managed? Where and in what roles do people work?** Here's how I'd walk an interviewer through it using Nike.

**"Let's say Nike is moving its HR onto Oracle Fusion."**

**1. The Enterprise — "the whole company"** The top box is just **Nike**. One enterprise for the whole organization. It's not a legal company, it's the umbrella. At this level we set company-wide rules, like how person numbers are generated and which employment model to use by default.

**2. Divisions — "how the bosses split the business"** Nike's management thinks in terms of brands: **Nike Brand, Jordan Brand, Converse**. Each of those is a Division. A division is a management grouping, not a legal one. Jordan Brand might have its own president and P&L, but that doesn't make it a separate company in the eyes of the law.

**3. Legal Entities — "the actual registered companies"** To operate in a country, Nike registers a company there. Imagine **Nike USA Inc.**, **Nike India Pvt Ltd** and **Nike UK Ltd**. Each one can sign contracts, pay tax, and get sued. These are Legal Entities.

Now HCM puts "hats" on these legal entities:

- **Legal Employer** — the entity whose name is on your offer letter. If you join Nike in Bengaluru, your legal employer is Nike India Pvt Ltd.
- **Payroll Statutory Unit (PSU)** — the entity responsible for paying people and filing statutory stuff like PF and income tax returns. In India, Nike India Pvt Ltd is usually both the legal employer and the PSU.
- **Tax Reporting Unit (TRU)** — a smaller registration under the PSU used for tax reporting. In India, think of separate registrations for Professional Tax in Karnataka and Maharashtra.
- **Legislative Data Group (LDG)** — the "country rulebook" for payroll: India LDG (rupees, Indian rules), US LDG (dollars), UK LDG (pounds). Every PSU is tied to one LDG.

**4. Business Units — "teams that run a piece of the business"** Inside India, Nike might run **Nike India Retail** (the stores) and **Nike India Corporate** (the head office). Business units are management units. In HCM they mostly control two things: **which reference data people see** (jobs, locations, departments) and **security** (an HR person can be given access to just one BU's workers).

**5. Reference Data Sets — "shared lists"** Nike doesn't want to create "Sales Associate" separately for every country. So there's a **COMMON** set for global stuff, and an **India Set** for India-only things like Mumbai store locations. Each BU is told which set it uses by default.

**6. Departments, Locations — "where you sit"**

- **Department**: Retail Operations, Marketing, HR, Supply Chain. It's the team you belong to and has a manager.
- **Location**: the physical address — Bengaluru HQ, Mumbai Flagship Store.

**7. Jobs, Positions, Grades — "what you do and what you're paid"**

- **Job**: a generic role — *Store Manager*, *Sales Associate*, *Product Designer*.
- **Position**: a specific seat of a job in a specific place — *Store Manager, Mumbai Flagship Store*, headcount 1. If it's filled, you can't hire a second person into it (unless headcount allows).
- **Grade**: the pay level — G1 to G8 — with salary ranges.

**So when Priya joins as Store Manager in Mumbai, her assignment says:** Legal Employer = Nike India Pvt Ltd, Business Unit = Nike India Retail, Department = Retail Operations, Location = Mumbai Flagship Store, Job = Store Manager, Position = Store Manager – Mumbai Flagship, Grade = G5. Every one of those values came from the Enterprise Structure we set up.

**The one-line wrap-up for the interviewer:** *"Legal entities answer who employs and pays you; divisions and business units answer who manages you; departments, locations, jobs, positions and grades answer where you sit and what you do."*

The diagram below shows the same picture.

&#91;embedded content: Nike enterprise structure · legal, management and workforce sides\]

Read it top-down: the enterprise is the umbrella, the three columns are built separately, and every hire's assignment picks one value from each.

## Each component, explained simply

Every component falls into one of three buckets: **legal** (who employs and pays), **management** (who runs and controls), or **workforce** (where you sit and what you do). Keep this in your head and nothing feels random.

| Bucket | Components | Nike example |
| --- | --- | --- |
| Top | Enterprise | Nike |
| Legal | Legal Entity, Legal Employer, PSU, TRU, LDG | Nike India Pvt Ltd, India LDG |
| Management | Division, Business Unit, Reference Data Set | Jordan Brand, Nike India Retail, India Set |
| Workforce | Department, Location, Job, Position, Grade, Trees | Retail Ops, Mumbai Store, Store Manager, G5 |

### Enterprise

The top-level organization — the whole company. You normally have **one enterprise per Fusion instance**.

What you configure here (task: **Manage Enterprise HCM Information**):

- **Person number generation** — Manual, Automatic prior to submission, or Automatic upon final save.
- **Employment model** — the default for the whole company (e.g. 2-Tier Single Assignment). Legal employers can override it.
- **Worker number generation** — optional extra number per work relationship.
- **Position synchronization** — whether assignments automatically copy job, department, location, grade etc. from the position.

*Nike:* "Nike" is the enterprise. Company-wide rule: person numbers are generated automatically.

### Division

A **management** grouping of the business — by brand, product line or region. It's optional, and it has no legal meaning.

- Created with **Manage Divisions**.
- Divisions are **date-effective** (you can track changes over time).
- In Financials, a division is often tied to a chart of accounts segment; in HCM it's mostly for reporting and grouping.

*Nike:* Nike Brand, Jordan Brand, Converse.

### Legal Entity (LE)

A real, registered company that can own assets, sign contracts, pay tax and be sued.

- Created with **Manage Legal Entity**.
- Needs a **legal address** (which needs **geography** data for that country loaded first), a **legal jurisdiction** and registration details.
- When you create an LE, you tick whether it's a **Legal Employer** and/or a **Payroll Statutory Unit**.
- A **main legal reporting unit** is created along with it.

*Nike:* Nike India Pvt Ltd.

### Legal Employer

An LE that **employs** people. Every work relationship (and so every employee) belongs to exactly one legal employer.

- Set up with **Manage Legal Entity HCM Information**.
- Here you can **override** the enterprise's employment model and worker number method, and set working hours.
- Moving someone to another legal employer = a **Global Transfer** (new work relationship), not a simple assignment change.

*Nike:* Priya's offer letter is from Nike India Pvt Ltd → that's her legal employer.

### Payroll Statutory Unit (PSU)

An LE responsible for **paying people and statutory reporting** (social insurance, year-end tax filings).

- One PSU can cover **many legal employers** (e.g. a group that files PF centrally).
- One LE can be **both** a legal employer and a PSU — very common.
- Each PSU is linked to **exactly one LDG**.

*Nike:* Nike India Pvt Ltd is also the PSU.

### Tax Reporting Unit (TRU)

The HCM name for a **Legal Reporting Unit** used for tax and social insurance reporting. When an LE is marked as a PSU, its legal reporting units become available as TRUs.

- Created with **Manage Legal Reporting Unit**, then **Manage Legal Reporting Unit HCM Information**.
- A TRU always sits under a PSU.

*Nike:* Separate Professional Tax registrations for Karnataka and Maharashtra.

### Legislative Data Group (LDG)

A **country rulebook** for payroll: it holds a country (legislation code), a currency, and all payroll objects — elements, payrolls, payment methods, balances.

- Created with **Manage Legislative Data Groups** — usually one per country.
- One LDG → many PSUs. One PSU → only one LDG.
- Even if you don't run Oracle Payroll, you still need LDGs because absences, benefits and elements use them.

*Nike:* India LDG (INR), US LDG (USD), UK LDG (GBP).

### Business Unit (BU)

A **management** unit that does some work for the company and is controlled by a manager. In Financials, BUs perform "business functions" like Payables. In HCM, they matter for three things:

1. **Every assignment needs a BU.**
2. **Reference data** — the BU's default set decides which jobs, departments, locations and grades a user can pick.
3. **Security** — data roles can restrict HR users to specific BUs.

- Created with **Manage Business Unit**. BUs are **not date-effective** — you make them active or inactive.

*Nike:* Nike India Retail, Nike India Corporate.

### Reference Data Set

A **bucket of shared setup data**, so the same job or location can be reused across BUs, or kept private to one BU.

- Created with **Manage Reference Data Sets**. A predefined **COMMON** set is shared with everyone.
- Linked to BUs with **Manage Business Unit Set Assignment** (each BU gets a default set).
- In HCM, **jobs, locations, departments and grades** are "set-enabled" — they live in a set.

*Nike:* "Sales Associate" job in COMMON (used everywhere); Mumbai and Bengaluru locations in India Set.

### Department

The **team** a person belongs to, usually with a manager and a cost center.

- Created with **Manage Departments** (or the Departments quick action). Date-effective and set-enabled.
- Can be arranged in a **Department Tree** for reporting and security.
- Can be linked to a company and cost center in Financials for costing.

*Nike:* Retail Operations, Marketing, HR.

### Location

The **physical address** where people work.

- Created with **Manage Locations**. Date-effective and set-enabled.
- Used on assignments, positions, and for things like time zone and work schedules.

*Nike:* Bengaluru HQ, Mumbai Flagship Store.

### Job

A **generic role** — the "what", not the "where".

- Created with **Manage Jobs**. Date-effective and set-enabled.
- Can have a **job family** (e.g. Retail), **job function**, and a list of **valid grades**.

*Nike:* Store Manager, Sales Associate, Product Designer.

### Position

A **specific seat** — a job in a particular department, location and BU, with a headcount and FTE.

- Created with **Manage Positions**. Date-effective. A position belongs to a **BU** (not a set).
- Has a **hiring status** (e.g. Proposed, Approved, Frozen) — you can only hire into an approved one.
- With **position synchronization** on, the assignment inherits job, department, location, grade and manager from the position.
- Not every company uses positions. Rigid, budget-controlled companies (government, banks) love them; fast-hiring retail often uses jobs only.

*Nike:* Store Manager – Mumbai Flagship, headcount 1.

### Grade (plus Grade Ladder, Grade Rate)

The **pay level**.

- **Grade**: G1–G8, optionally with **steps**. Date-effective, set-enabled.
- **Grade Rate**: the salary range (min/mid/max) for a grade.
- **Grade Ladder**: an ordered list of grades for progression.

*Nike:* G5 = store managers, range ₹12–18 lakh (made-up numbers).

### Organization Trees and Hierarchies

Ways to arrange things into a hierarchy for **reporting, security and approvals**:

- **Department tree** — departments under parent departments.
- **Organization tree / Division tree** — broader org rollups.
- **Position hierarchy** — positions reporting to positions.
- Don't confuse these with the **manager (supervisor) hierarchy**, which comes from the manager field on each assignment.

### Supporting setup you'll hear about

- **Geographies** — country structure (country → state → city → postal code). Load these before creating addresses, or address validation fails.
- **Legal Jurisdictions and Legal Authorities** — the law area (e.g. India income tax) and the government body (e.g. Income Tax Department) that LEs and TRUs register with.
- **Enterprise Structures Configurator (ESC)** — a guided tool that builds enterprise, divisions, LEs, BUs and ledgers together. Mostly used in Financials-led implementations at the very start; after that, you maintain things with the individual tasks.

## Setup order: what you build first and why

The golden rule: **build the thing that others point to first.** A department points to a location, a position points to a job and a department, a PSU points to an LDG — so those come earlier. Interviewers often ask "what's the sequence?" and this list is the answer.

All of this is done in **Setup and Maintenance**, mainly under the **Workforce Deployment** offering. You can also just search the task name in Setup and Maintenance. (Day-to-day workforce structures like departments, jobs and positions also have quick actions under **My Client Groups → Workforce Structures** in newer releases.)

1. **Geographies** — load the country structure (country, state, city, postal code). *Why first:* every address you create later is validated against it.
2. **Enterprise HCM Information** — person number method, employment model, position sync.
3. **Legal Jurisdictions and Legal Authorities** — many countries come predefined; check and add if needed.
4. **Legal Addresses** — the registered address for each legal entity.
5. **Legislative Data Groups** — one per country. *Why before LE HCM info:* the PSU must be mapped to an LDG.
6. **Legal Entities** — create, tick Legal Employer and/or PSU, add registrations.
7. **Legal Entity HCM Information** — legal employer details (employment model override, working hours) and PSU details (map to LDG).
8. **Legal Reporting Units / TRU HCM Information** — tax reporting registrations.
9. **Divisions** — optional management groupings.
10. **Reference Data Sets** — e.g. COMMON plus country sets.
11. **Business Units** and **Business Unit Set Assignment** — create BUs, give each a default set.
12. **Locations** — set-enabled, so the set must exist first.
13. **Departments** (and **Department Trees**) — departments reference locations.
14. **Grades, Grade Rates, Grade Ladders** — jobs reference valid grades.
15. **Job Families and Jobs**.
16. **Positions** — need job, department, location, BU and grade to already exist.
17. **Trees and hierarchies** — position hierarchy, organization trees, for reporting and security.

### Which tool to use

| Tool | What it's for | When you'd use it |
| --- | --- | --- |
| Setup tasks (UI) | Create records one by one | Small volumes, legal structures, first-time setup |
| Enterprise Structures Configurator (ESC) | Guided build of enterprise, divisions, LEs, BUs, ledgers together | Start of a Financials-led implementation |
| Rapid implementation spreadsheets | Fill an Excel template, upload | Quick bulk setup of enterprise and workforce structures |
| HCM Data Loader (HDL) | Load .dat files (Location, Organization, Job, Grade, Position…) | Large volumes, data migration, ongoing mass changes |
| HCM Spreadsheet Data Loader (HSDL) | Excel front end over HDL | Business users doing mass updates |

**Interview tip:** If asked "how did you load 3,000 positions?", the answer is HDL (Position.dat), loading in dependency order — locations, departments, grades, jobs, then positions.

## How to create each component, step by step

For each one: the task name, what you fill in, and the Nike values. Field labels can differ slightly by release, so focus on **what** and **why**, not on memorizing every label.

**Before you start — Update vs Correct.** Most HCM structures are date-effective. When you edit one, Fusion asks:

- **Update** = "from this date onwards it's different" — keeps history. *Example:* Marketing department moves to a new location from 1 April.
- **Correct** = "it was always wrong, fix it" — overwrites the record without new history. *Example:* you typed "Markting".

Interviewers ask this a lot.

### 1. Enterprise HCM Information

**Task:** Manage Enterprise HCM Information → Edit (Update or Correct).

- Person Number Generation Method → *Automatic prior to submission* (the default) or *Automatic upon final save* or *Manual*.
- Initial Person Number → e.g. 100001 (or highest legacy number + 1 if migrating).
- Employment Model → *2 Tier – Single Assignment* (default) for Nike globally.
- Worker Number Generation → optional.
- Position Synchronization → turn on if Nike is position-driven, and pick which attributes sync (department, job, location, grade, manager).

### 2. Division

**Task:** Manage Divisions → Create.

- Effective Start Date → keep it early (e.g. 01-Jan-2000).
- Name → *Jordan Brand*. Location → optional.
- Division-related financial info (chart of accounts segment) only if Financials uses it.

### 3. Legislative Data Group

**Task:** Manage Legislative Data Groups → Create.

- Name → *India LDG*. Country → *India*. Currency → *INR*.
- Cost allocation structure → only if payroll costing needs it.

### 4. Legal Entity

**Task:** Manage Legal Entity → Create.

- Country → *India*. Name → *Nike India Pvt Ltd*. Legal Entity Identifier → a unique code.
- Tick **Legal Employer** ✔ and **Payroll Statutory Unit** ✔.
- Registration → identifying jurisdiction, registration number, legal address (created from geography data).
- Effective From → early date.

### 5. Legal Entity HCM Information

**Task:** Manage Legal Entity HCM Information → select the LE as scope → Edit.

- **Legal Employer tab:** employment model (override if needed), worker number generation, standard working hours.
- **Payroll Statutory Unit tab:** Associated LDG → *India LDG*; fiscal year start date.

### 6. Legal Reporting Unit / Tax Reporting Unit

**Tasks:** Manage Legal Reporting Unit → Create (scope = the LE), then Manage Legal Reporting Unit HCM Information.

- Name → *Nike India – Karnataka PT*. Registration with the right jurisdiction and legal authority.
- HCM info → country-specific tax details used by payroll.

### 7. Reference Data Set

**Task:** Manage Reference Data Sets → New.

- Set Code → *IND*. Set Name → *India Set*. (COMMON already exists.)

### 8. Business Unit + Set Assignment

**Tasks:** Manage Business Unit → Create; then Manage Business Unit Set Assignment.

- Name → *Nike India Retail*. Location, Manager → optional.
- Set assignment → Default Set = *India Set*. You can override per object (e.g. jobs come from COMMON, locations from India Set).
- Business functions (Payables, Procurement…) are a Financials thing — an HCM-only project doesn't need them.

### 9. Location

**Task:** Manage Locations → Create.

- Effective Start Date → **early** (see edge cases).
- Set → *India Set*. Name → *Mumbai Flagship Store*. Code → *MUM-FLG*. Status → Active.
- Address → validated against geography.

### 10. Department

**Task:** Manage Departments → Create.

- Effective Start Date → **early**. Name → *Retail Operations*. Set → *India Set*.
- Location → *Mumbai Flagship Store*. Manager → optional.
- Company / Cost Center → for costing, if Financials is used.
- Optionally add it to a **Department Tree** under a parent like *India Operations*.

### 11. Grades, Grade Rates, Grade Ladders

**Tasks:** Manage Grades → Create; Manage Grade Rates; Manage Grade Ladders.

- Grade → Set *COMMON*, Code *G5*, Name *Grade 5*. Steps → optional.
- Grade Rate → INR range for G5 (min/mid/max).
- Grade Ladder → G1 → G8 in order.

### 12. Job

**Task:** Manage Jobs → Create.

- Set → *COMMON*. Name → *Store Manager*. Code → *STR-MGR*.
- Job Family → *Retail*. Job Function, Management Level → optional.
- Valid Grades → G4, G5, G6.

### 13. Position

**Task:** Manage Positions → Create.

- Business Unit → *Nike India Retail*. Name → *Store Manager – Mumbai Flagship*.
- Job → *Store Manager*. Department → *Retail Operations*. Location → *Mumbai Flagship Store*.
- Hiring Status → *Approved*. Position Type → *Single Incumbent* (one person) or *Shared* (several people).
- FTE / Headcount → 1. Valid grades / Entry grade → G5.

**Then hire Priya** — her assignment picks Nike India Pvt Ltd, Nike India Retail, and the position. With position sync on, job, department, location and grade fill in automatically.

## Edge cases and gotchas from real projects

These are the "tell me about a problem you faced" answers. Each one has what happens, why, and the fix.

### Dates and history

1. **The start-date trap.** You create the *Retail Operations* department today. Then data migration loads Priya with a 2018 hire date → error: the department doesn't exist on that date.
   - *Why:* structures are date-effective; an assignment can only use something valid on its date.
   - *Fix:* create locations, departments, jobs, grades and positions with a very early start date (many projects use 01-01-1951). Pushing dates back later is painful, so decide this on day one.
2. **Update vs Correct mistake.** Someone uses *Correct* to change a department's location from 1 April. Now history says it was always at the new location, and old reports are wrong.
   - *Fix:* use *Update* for real business changes; *Correct* only for typos.
3. **Future-dated changes.** A department is updated with a 1 Jan next-year change, and then someone tries to Correct today's record. Fusion warns that future rows exist.
   - *Fix:* check the history first; you may need to change the future row too.

### Enterprise settings you can't casually change

4. **Changing the employment model after go-live.** Switching between 2-tier and 3-tier is allowed **only if no work relationships exist** in that enterprise or legal employer ([Oracle docs](https://docs.oracle.com/en/cloud/saas/human-resources/faigh/when-to-select-the-employment-model.html)). Single → Multiple Assignments is fine any time; some other switches (like to a contract-based model) also need no existing work relationships ([apps2fusion](https://apps2fusion.com/old/oracle-fusion-online-training/fusion-applications/oracle-fusion-hcm/677-employment-model-in-oracle-fusion-hcm)).
   - *Lesson:* decide the employment model in the design phase.
5. **Worker number method.** A legal employer can switch to **manual** worker numbers any time, but to **automatic** only if it has no employee or contingent worker work relationships yet ([Oracle docs](https://docs.oracle.com/en/cloud/saas/human-resources/faigh/considerations-for-using-worker-numbers.html)).
6. **Legacy person numbers during migration.** Nike wants old employee IDs kept. You don't need to switch to Manual — keep the method automatic, pass the legacy number in the HDL Worker file, and set the **Initial Person Number** to highest legacy number + 1 so new hires don't clash ([Oracle docs](https://docs.oracle.com/en/cloud/saas/human-resources/faqpr/can-i-load-legacy-person-numbers-when-the-person-number-generation-method-is-set-to-automatic.html)).
7. **Gaps in person numbers.** With *Automatic prior to submission*, a number is used up even if the hire is cancelled, so gaps appear ([Oracle docs](https://docs.oracle.com/en/cloud/saas/human-resources/faigh/person-number-generation-methods.html)). If the client hates gaps, use *Automatic upon final save* (number appears only after approval).
8. **Fixing one wrong person number.** Temporarily switch the method to Manual, correct the number, then switch back to the original automatic method — and don't hire anyone in between.

### Legal structure traps

9. **Moving someone to another legal entity.** Priya moves from Nike India Pvt Ltd to Nike UK Ltd. You can't just change a field on her assignment.
   - *Why:* the legal employer belongs to the **work relationship**, not the assignment.
   - *Fix:* a **Global Transfer** (or Change Legal Employer) ends the old work relationship and creates a new one. If the PSU changes, payroll relationship changes too.
10. **One person, two countries.** A designer works part-time for Nike US and Nike UK at the same time → **two work relationships**, one per legal employer.
11. **PSU linked to the wrong LDG.** The PSU for India gets mapped to the US LDG. Payroll objects now live in the wrong currency and rules. Fixing this after payroll data exists is very hard → check the mapping before any payroll setup.
12. **Missing geography.** Creating a legal address or location in a new country fails validation.
    - *Fix:* load or enable that country's geography structure first.

### Business units and reference data

13. **"The job doesn't show up in the list!"** — the most common support ticket. An HR user in *Nike India Retail* can't find the job *Visual Merchandiser*.
    - *Why:* the job sits in the *UK Set*, and the India BU only sees its default set (*India Set*) plus COMMON.
    - *Fix:* move the job to COMMON, or change the BU's set assignment for jobs.
14. **BU vs legal employer confusion.** They are independent. One BU can have workers from several legal employers, and one legal employer can have workers in several BUs. Don't assume BU = company.
15. **BUs aren't date-effective.** Renaming a BU changes the name everywhere, including in old history. If the client needs history, create a new BU and move people.

### Positions

16. **Position is full.** Hiring a second Store Manager into a headcount-1 position triggers a warning or error (depending on setup). Increase headcount, use a Shared position, or create a new one.
17. **Position not approved.** You can't hire into a position whose hiring status is *Proposed* or *Frozen*.
18. **Position sync side effects.** With sync on, updating the position's department from 1 May also moves every incumbent's department from 1 May. And if HR manually changes a synced field on an assignment, it stops inheriting that value.
19. **Grade not in the list.** If valid grades are defined on the job or position, only those grades can be picked.

### Inactivating and deleting

20. **Can't close a department.** Trying to inactivate *Marketing* while people are still assigned to it is blocked or warned → transfer the workers first, then end-date or inactivate it.
21. **Can't delete a legal entity or BU.** Once used in transactions, you inactivate rather than delete.

### Trees and hierarchies

22. **Tree changes don't show up.** A new department added to the department tree doesn't appear in reports or security. Check the tree version is **Active** and that the flattening processes have run. For manager-based security, the **Refresh Manager Hierarchy** process may need to run.

## Interview questions and answers

Answers are written the way you'd say them out loud. Practice saying them, not just reading them.

### Basic

**Q1. What is Enterprise Structure in Oracle Fusion HCM?** It's the framework that models the company inside Fusion — its legal side (legal entities, legal employers, PSUs), its management side (divisions, business units), and its workforce side (departments, locations, jobs, positions, grades). Every worker transaction uses it.

**Q2. What is an enterprise?** The top-level organization representing the whole company, usually one per instance. It holds company-wide HCM settings like person number generation and the default employment model.

**Q3. What's the difference between a legal entity and a legal employer?** A legal entity is any registered company. A legal employer is a legal entity that actually employs workers. Every legal employer is a legal entity, but a holding company that employs nobody is a legal entity without being a legal employer.

**Q4. What is a PSU?** A Payroll Statutory Unit is the legal entity responsible for paying workers and statutory reporting. One PSU can cover several legal employers, and it is tied to exactly one LDG.

**Q5. What is an LDG?** A Legislative Data Group groups payroll data by country — legislation, currency, and payroll objects like elements and payrolls. Usually one per country.

**Q6. What is a TRU?** A Tax Reporting Unit is a legal reporting unit used for tax and social insurance reporting. It sits under a PSU. Example: separate state registrations for Professional Tax in India.

**Q7. What is a division?** A management grouping — by brand, product line or region — with no legal standing. Nike Brand vs Jordan Brand.

**Q8. What is a business unit in HCM?** A management unit. In HCM it's required on every assignment, it decides which reference data sets a user sees, and it's used for data security.

**Q9. Difference between job and position?** A job is a generic role like *Store Manager*. A position is a specific instance of that job in a particular department, location and BU, with headcount — *Store Manager, Mumbai Flagship, headcount 1*. Many positions can share one job.

**Q10. Difference between department and location?** Department is the team you belong to; location is the physical place you work. Two people in the Marketing department can sit in Mumbai and Bengaluru.

**Q11. What is a reference data set?** A bucket that holds shared setup data like jobs, locations, departments and grades, so they can be shared across business units (COMMON set) or kept private to some.

### Intermediate

**Q12. What's the setup sequence?** Geographies → enterprise HCM info → jurisdictions and legal addresses → LDGs → legal entities → LE HCM info (legal employer, PSU-to-LDG) → TRUs → divisions → reference data sets → business units and set assignment → locations → departments → grades → jobs → positions → trees. The rule: create what others reference first.

**Q13. What's the employment model? Where do you set it?** It defines how work relationships, (optionally) employment terms, and assignments relate. Two-tier = work relationship + assignment; three-tier adds employment terms. Set at enterprise level in Manage Enterprise HCM Information, overridable per legal employer in Manage Legal Entity HCM Information. Default is 2-tier single assignment.

**Q14. Can one legal entity be both a legal employer and a PSU?** Yes — that's the most common setup for a single-country company. Nike India Pvt Ltd employs people and also pays them and files statutory returns.

**Q15. Relationship between PSU, LDG and legal employer?** Legal employers roll up to a PSU. Each PSU links to one LDG. One LDG can serve many PSUs. TRUs sit under the PSU.

**Q16. What are set-enabled objects in HCM?** Jobs, locations, departments and grades (plus grade rates and ladders). Positions are not set-enabled — they belong to a business unit.

**Q17. What is position synchronization?** A setting (enterprise or legal employer level) that makes assignments inherit attributes like job, department, location, grade and manager from the position. When the position changes, incumbents change with it.

**Q18. Update vs Correct?** Update creates a new dated row and keeps history — for real changes. Correct overwrites the current row — for mistakes.

**Q19. How do you load structures in bulk?** HCM Data Loader with .dat files — Location, Organization (for departments), Job, Grade, Position — loaded in dependency order. For a quick initial build, rapid implementation spreadsheets also work.

**Q20. What's the person number generation method and its options?** An enterprise setting: Manual, Automatic prior to submission (default — number shown during hire, gaps possible if cancelled), Automatic upon final save (number assigned only after approval).

### Tricky and scenario-based

**Q21. Nike has two brands that are separate companies legally, but managed as one. How do you model it?** Two legal entities (legal employers), one division. Legal structure follows the law; management structure follows how leadership runs it. They don't have to match.

**Q22. A user says a job isn't appearing in the job list during hire. What do you check?** The job's reference data set vs the BU's set assignment. If the job is in a set not assigned to that BU (and not COMMON), it won't show. Also check the job is active as of the hire date.

**Q23. Migration fails: "department not valid on assignment start date." Why?** The department's effective start date is after the worker's hire date. Create structures with an early start date (like 1951) or correct the department's start date before loading.

**Q24. An employee moves from India to the UK. What happens in Fusion?** It's a global transfer — the India work relationship ends and a new one starts with the UK legal employer, with a new assignment. Payroll moves to the UK PSU/LDG. It's not a simple assignment change because the legal employer lives on the work relationship.

**Q25. Can you change the employment model after go-live?** Partly. Switching between 2-tier and 3-tier is only allowed if there are no work relationships in that enterprise or legal employer. Some switches, like single to multiple assignments, are allowed any time. So it's a design-phase decision.

**Q26. Is a business unit the same as a legal entity?** No. BU is management, LE is legal. One BU can include workers from multiple legal employers, and one legal employer's workers can be spread across BUs.

**Q27. You want legacy employee IDs kept but new hires numbered automatically. How?** Keep the method automatic, load legacy numbers through HDL, and set the initial person number to the highest legacy number + 1.

**Q28. When would you recommend positions vs jobs only?** Positions when the client needs headcount and budget control or a fixed hierarchy — government, banks, healthcare. Jobs only when hiring is fast and flexible — retail stores, startups. Positions add maintenance overhead.

**Q29. A client wants to delete a business unit created by mistake that has workers. What do you say?** You can't delete it once used. Move workers to the correct BU (date-effective update), then inactivate the wrong BU.

**Q30. Why does an HCM-only client still need an LDG if they use a third-party payroll?** Because elements, absences, benefits and many country rules still live in LDG context, and PSUs must be mapped to one.

**Q31. The department tree was updated but reports still show the old structure. Why?** The new tree version may not be active, or the flattening processes haven't run. Activate the version and run them.

**Q32. Tell me how you'd explain enterprise structure to a business user in 30 seconds.** "Legal entities answer who employs and pays you. Divisions and business units answer who manages you. Departments, locations, jobs, positions and grades answer where you sit and what you do. The enterprise is the umbrella over all of it."

## Cheat sheet for quick revision

| Component | One-liner | Key task | Remember |
| --- | --- | --- | --- |
| Enterprise | The whole company | Manage Enterprise HCM Information | Person number method, default employment model, position sync |
| Division | Management grouping (brand, region) | Manage Divisions | No legal meaning, optional, date-effective |
| Legal Entity | Registered company | Manage Legal Entity | Needs legal address → geography first |
| Legal Employer | LE that employs people | Manage Legal Entity HCM Information | Lives on the work relationship; change = global transfer |
| PSU | LE that pays and files statutory | Manage Legal Entity HCM Information | Many legal employers → one PSU; one PSU → one LDG |
| TRU | Tax registration under a PSU | Manage Legal Reporting Unit (HCM Information) | e.g. state Professional Tax |
| LDG | Country payroll rulebook | Manage Legislative Data Groups | One per country, holds currency |
| Business Unit | Management unit | Manage Business Unit | Required on assignment; drives sets + security; not date-effective |
| Reference Data Set | Bucket of shared setup data | Manage Reference Data Sets | COMMON is shared; BU Set Assignment decides visibility |
| Department | Your team | Manage Departments | Set-enabled, date-effective, early start date |
| Location | Physical address | Manage Locations | Set-enabled, date-effective |
| Job | Generic role | Manage Jobs | Set-enabled; valid grades |
| Position | Specific seat with headcount | Manage Positions | Belongs to a BU; hiring status; sync |
| Grade | Pay level | Manage Grades | Grade rates = salary range; ladders = progression |

**Five lines to say if you blank out:**

1. Legal = who employs and pays; management = who runs; workforce = where and what.
2. Legal employer → PSU → LDG. TRUs sit under the PSU.
3. BU picks the reference data set; COMMON is shared.
4. Build what others reference first; start dates early.
5. Employment model and number methods are design-phase decisions.

## Sources

Checked against Oracle documentation as of the date above:

- [When to Select the Employment Model](https://docs.oracle.com/en/cloud/saas/human-resources/faigh/when-to-select-the-employment-model.html)
- [Person Number Generation Methods](https://docs.oracle.com/en/cloud/saas/human-resources/faigh/person-number-generation-methods.html)
- [Loading legacy person numbers with automatic generation](https://docs.oracle.com/en/cloud/saas/human-resources/faqpr/can-i-load-legacy-person-numbers-when-the-person-number-generation-method-is-set-to-automatic.html)
- [Considerations for Using Worker Numbers](https://docs.oracle.com/en/cloud/saas/human-resources/faigh/considerations-for-using-worker-numbers.html)
- [Manage Legal Entities (HCM information, PSU to LDG)](https://docs.oracle.com/en/cloud/saas/human-resources/24b/fapid/manage-legal-entities.html)
- [Examples of HCM Organization Models for Payroll](https://docs.oracle.com/en/cloud/saas/human-resources/faipb/examples-of-hcm-organization-models-for-payroll.html)
- [Employment model change rules (apps2fusion)](https://apps2fusion.com/old/oracle-fusion-online-training/fusion-applications/oracle-fusion-hcm/677-employment-model-in-oracle-fusion-hcm)
