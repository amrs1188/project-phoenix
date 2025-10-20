# Project Phoenix: Implementation Roadmap & Checklist (v3.0)
* **Status:** In Progress
* **Purpose:** This is the master checklist for the entire implementation of Project Phoenix (aligned with Blueprint v8.0), detailing every task from initial setup to full deployment.

---
## Phase 1: Foundation (Setup) 🏛️
*The goal of this phase is to build and configure all the core infrastructure.*

### 1.1 PostgreSQL Database Setup
- [x] **Deploy PostgreSQL Server:** Ensure a stable PostgreSQL instance is running and accessible.
- [x] **Create Database:** Run the SQL command to create the `phoenix_db` database.
- [x] **Create User:** Run the SQL command to create the `phoenix_user` and grant privileges.
- [ ] **Execute Updated Schema Script:** Run SQL script to create `master_contacts` (with `needs_local_recycle`, `original_location_id`), `agents`, `routing_rules`, `assignments`, and the **new `ghl_configs` table**. *(Optional: `system_settings` table)*.
- [ ] **Initial Configuration Data:**
    - [x] Populate `agents` table with pilot agents (ensure `ghl_location_id` is correct).
    - [x] Populate `routing_rules` table with initial rules.
    - [ ] Populate `ghl_configs` table for pilot sub-account(s) with `location_id`, `auth_header_value`, and necessary Custom Field/Pipeline/Stage IDs retrieved via API or GHL interface.

### 1.2 n8n Controller Setup
- [x] **Install Server Prerequisites:** Docker & Docker Compose installed.
- [x] **Create Project Directory:** `n8n-phoenix` created.
- [x] **Create `docker-compose.yml` File:** Correct configuration applied.
- [x] **Create and Secure `.env` File:** Populated with DB credentials and subdomain.
- [x] **Launch n8n Container:** Instance running.
- [x] **Configure DNS & SSL:** Subdomain points correctly, HTTPS active.
- [ ] **Create n8n Dynamic Credentials:** Inside n8n UI, create "Header Auth" credentials for pilot sub-account(s) following the **`GHL-[LocationID]` naming convention**.

### 1.3 GHL Sub-Account Configuration
*(Repeat for EACH participating sub-account)*
- [x] **Configuration:**
    - [x] Create **Private Integration** and securely store the V2 Access Token.
    - [x] Create the **11-stage sales pipeline**.
    - [x] Create all required **Custom Fields**.
    - [x] Confirm **"Allow Duplicate Contact"** is **OFF**.
- [ ] **Internal Automations (Workflows):**
    - [x] Build the "Fresh Lead Capture & Notify" workflow (using Custom Webhook).
    - [x] Build the series of "30-Day Cool Down" workflows.
    - [x] Build the "Notify n8n Referee on Stage Change" workflow (using standard Webhook and n8n Production URL).

---
## Phase 2: Automation Build (Development) 🧠
*The goal of this phase is to build and test the four core n8n workflows incorporating all v8.0 logic.*

### 2.1 Workflow A: Real-Time Registrar
- [x] **A.1 - Build & Test Webhook Connection:** Webhook node receives data.
- [x] **A.2 - Build Master Record Logic:** Postgres node inserts into `master_contacts`, setting `lead_type='Fresh'`, storing `original_location_id`.
- [x] **A.3 - Build Assignment Logic:** Postgres node inserts into `assignments`, calculating the **correct Fresh timer (24h)**.
- [x] **A.4 - Perform Full End-to-End Test:** Verified correct data creation in DB.

### 2.2 Workflow D: The Timer & Status Janitor
- [x] **D.1 - Build "Find Overdue" Logic:** Schedule trigger + Postgres `Execute Query` finds expired assignments, joining `master_contacts` to get `lead_type`.
- [x] **D.2 - Build Conditional Logic:** Add `IF` node (`CheckLeadType`) branching based on `lead_type`.
- [x] **D.3 - Build "Flag for Local Recycle" Path:** On 'Fresh' path, add Postgres `Update` node (`FlagForLocalRecycle`) to set `needs_local_recycle=true` in `master_contacts`.
- [x] **D.4 - Build "Set Global Priority" Path:** On 'Recycle' path, add Postgres `Update` node (`SetGlobalPriority`) to set `priority_level` (1, 2, or 3 based on stage) in `master_contacts`.
- [x] **D.5 - Build "Close Assignment" Logic:** Add final Postgres `Update` node (`CloseExpiredAssignment`) after the IF to set `status='Lost'` in `assignments`.
- [x] **D.6 - Perform Full End-to-End Test:** Test with both Fresh and Recycle leads, verifying correct flags/priorities are set and assignments closed.

### 2.3 Workflow B: The Morning Dispatch
- [x] **B.1 - Build Initial Setup:** Schedule trigger. *(New)* Add step to initialize/reset daily quota counters.
- [x] **B.2 - Build Local Recycle Logic:** Add Postgres `Select` to find leads with `needs_local_recycle=true`. Loop through them: find next local agent (checking history), assign if found (using **Delivery Actions**), OR transition to global (update `lead_type`, set `priority_level=2`) if no local agents left.
- [ ] **B.3 - Build Global Recycle - Info Gathering:** Add Postgres `Execute Query` to get leads with `priority_level` (P1>P2>P3). Loop through them: find team, get agent roster.
- [ ] **B.4 - Build Global Recycle - Intelligent Loop:** Loop through agents: Check **Quota** (new logic needed), check History, check Capacity (using **HTTP Request** with dynamic auth via `ghl_configs` lookup).
- [ ] **B.5 - Build Delivery Actions:** Add nodes: `GetGHLConfigs` (Postgres Select), `CreateContactInGHL` (HTTP POST), `CreateOpportunityInGHL` (HTTP POST), `CreateAssignmentRecord` (Postgres Insert calculating correct timer), `UpdateMasterContact` (Postgres Update to remove priority), *(New)* Increment agent quota counter, `StopLoopForThisLead` (NoOp - Execute Once).
- [ ] **B.6 - Perform Full End-to-End Test:** Test extensively with local recycle flags, different priorities, quota limits, and capacity limits.

### 2.4 Workflow C: The Real-Time Referee
- [x] **C.1 - Build Trigger & Switch Logic:** Webhook `ReceiveGHLStageChange` + Switch `CheckStageType` routing to 'Lost' or 'Pre-Booking'.
- [ ] **C.2 - Build "Surrender" Path:** Add Postgres `Update` node (`UpdateAssignmentOnSurrender`) to set assignment `status='Lost'`. Add second Postgres `Update` node (`MarkContactForRedispatch`) to set correct `priority_level` in `master_contacts` based on previous stage.
- [ ] **C.3 - Build "Win" Path:**
    - [ ] Add `GetOpportunityOwner` (HTTP GET) to find GHL user ID.
    - [ ] Add `FindWinningAgentInternalID` (Postgres Select) to get internal agent ID.
    - [ ] Build `DeclareWinningAssignment` (Postgres Update) to set assignment `status='Won'`.
    - [ ] Build `FindCompetingAssignments` (Postgres Execute Query) to get list of losers including `ghl_location_id`.
    - [ ] Build loop for losers: `GetGHLConfigs` (Postgres Select), `GetLoserOpportunityID` (HTTP GET), `MoveLoserToLostInGHL` (HTTP PUT using dynamic auth), `AddSystemNoteToLoser` (HTTP POST using dynamic auth), `UpdateLosingAssignmentDB` (Postgres Update).
- [ ] **C.4 - Perform Full End-to-End Test:** Test both "Surrender" (verify priority set) and "Win" conditions (verify winner marked, losers updated in GHL & DB).

---
## Phase 3: Go-Live (Deployment) 🚀
*The goal of this phase is to migrate legacy data, perform final testing, and launch the system.*

### 3.1 Initial Data Migration
- [ ] **Prepare Legacy Data:** Clean CSV, match headers, include new columns.
- [ ] **Set Initial Status:** Set `lead_type = Recycle` and **`priority_level = 3`** for all rows.
- [ ] **Perform Bulk Import:** Import CSV into `master_contacts` via HeidiSQL/pgAdmin.

### 3.2 End-to-End Testing (Pilot Team)
- [ ] Select pilot team. Populate `ghl_configs` for their sub-account. Create their dynamic credentials in n8n.
- [ ] Run **Morning Dispatch** for a small batch of *legacy leads* (P3) for the pilot team. Verify assignment, quota, capacity limits.
- [ ] Manually push a **Fresh Lead** through GHL for the pilot team. Verify registration and timer.
- [ ] Let timers expire for both Fresh (test local flag) and Recycle (test priority setting) via **Janitor**. Verify DB changes.
- [ ] Run **Morning Dispatch** again to test local recycle assignment and prioritized global assignment.
- [ ] Test **Referee** "Surrender" path by moving a lead to Lost. Verify priority set.
- [ ] Test **Referee** "Win" path by creating competing assignments and moving one to Pre-Booking. Verify winner marked, losers updated.
- [ ] Monitor all workflows for 3-5 days. Collect pilot feedback. Make adjustments.

### 3.3 Full Rollout
- [ ] Populate `ghl_configs` for **all** sub-accounts. Create **all** dynamic credentials in n8n. Add **all** agents to `agents` table.
- [ ] Announce Go-Live date.
- [ ] Run **Morning Dispatch** in controlled batches until desired lead volume is active.
- [ ] **Activate** all scheduled n8n workflows (Janitor, Dispatch).
- [ ] Closely monitor system health, execution logs, and agent feedback for the first week.
- [ ] **Celebrate the launch of Project Phoenix! 🎉**
