# Project Phoenix: Implementation Roadmap & Checklist (v2.0)
* **Status:** In Progress
* **Purpose:** This is the master checklist for the entire implementation of Project Phoenix, detailing every task from initial setup to full deployment.

---
## Phase 1: Foundation (Setup) 🏛️
*The goal of this phase is to build and configure all the core infrastructure.*

### 1.1 PostgreSQL Database Setup
- [x] **Deploy PostgreSQL Server:** Ensure a stable PostgreSQL instance is running and accessible.
- [x] **Create Database:** Run the SQL command to create the `phoenix_db` database.
- [x] **Create User:** Run the SQL command to create the `phoenix_user` and grant all necessary privileges to `phoenix_db`.
- [x] **Execute Schema Script:** Run the final SQL script to create all four master tables: `master_contacts`, `agents`, `routing_rules`, and `assignments`.
- [x] **Initial Data Population:**
    - [x] Populate the `agents` table with the initial list of pilot agents.
    - [x] Populate the `routing_rules` table with the initial project/location rules.

### 1.2 n8n Controller Setup
- [x] **Install Server Prerequisites:** Install Docker and Docker Compose on the Hostinger VPS.
- [x] **Create Project Directory:** Create the `n8n-phoenix` directory on the server.
- [x] **Create `docker-compose.yml` File:** Create the file with the correct configuration.
- [x] **Create and Secure `.env` File:** Create the `.env` file and populate it with all database credentials and the n8n subdomain.
- [x] **Launch n8n Container:** Run `docker-compose up -d` to start the n8n instance.
- [x] **Configure DNS & SSL:** Point the n8n subdomain's DNS A record to the server IP and ensure SSL (HTTPS) is correctly configured.
- [x] **Create n8n Credentials:** Inside the n8n UI, create a separate "Header Auth" credential for each GHL sub-account's API key.

### 1.3 GHL Sub-Account Configuration
*(This entire section must be repeated for **EACH** participating GHL sub-account)*
- [x] **Configuration:**
    - [x] Generate and securely store the sub-account's **API Key**.
    - [x] Create the **11-stage sales pipeline** exactly as defined in the blueprint.
    - [x] Create all required **Custom Fields** under the correct groups.
    - [x] Confirm the **"Allow Duplicate Contact"** setting is turned **OFF**.
- [ ] **Internal Automations (Workflows):**
    - [ ] Build the "Fresh Lead Capture & Notify" workflow.
    - [ ] Build the series of "30-Day Cool Down" workflows (e.g., Qualifying -> Pending -> Lost).
    - [ ] Build the "Notify n8n Referee on Stage Change" trigger/workflow.

---
## Phase 2: Automation Build (Development) 🧠
*The goal of this phase is to build the four core n8n workflows that run the system.*

### 2.1 Workflow A: Real-Time Registrar
- [x] **A.1 - Build & Test Webhook Connection:** Create the workflow, add the Webhook node, and test the connection with GHL.
- [x] **A.2 - Build the Master Record Logic:** Add the first Postgres node to `Insert` a new row into the `master_contacts` table.
- [x] **A.3 - Build the Assignment Logic:** Add the second Postgres node to `Insert` a new row into the `assignments` table, starting the timer.
- [x] **A.4 - Perform Full End-to-End Test:** Send a lead from GHL and verify data is correctly created in both database tables.

### 2.2 Workflow D: The Timer & Status Janitor
- [x] **D.1 - Build the "Find Overdue" Logic:** Create the workflow with a Schedule trigger and a Postgres node using `Execute a SQL Query` to find all expired assignments.
- [x] **D.2 - Build the "Update" Logic:** Add the two subsequent Postgres `Update` nodes to update the `master_contacts` and `assignments` tables.
- [x] **D.3 - Perform Full End-to-End Test:** Manually create test data in the database and execute the workflow to verify it updates the records correctly.

### 2.3 Workflow B: The Morning Dispatch
- [x] **B.1 - Build the "Information Gathering" Path:** Create the workflow with a Schedule trigger and the three Postgres nodes that get the list of available leads, find the correct team, and get the team's agent roster.
- [ ] **B.2 - Build the "Intelligent Assignment Loop":** Add the logic (Postgres and IF nodes) that loops through the agent roster and performs the "History Check."
- [ ] **B.3 - Add the "Capacity Check":** Add an HTTP Request node inside the loop to make an API call to GHL to check the agent's current lead count.
- [ ] **B.4 - Build the "Delivery" Actions:** Add the final nodes that create the contact in GHL via API and create the new `assignments` record in the database.
- [ ] **B.5 - Perform Full End-to-End Test:** Manually create test data and execute the full workflow to verify a lead is correctly assigned to an available agent in GHL.

### 2.4 Workflow C: The Real-Time Referee
- [ ] **C.1 - Build Trigger & Switch Logic:** Create the workflow with a Webhook trigger and a `Switch` node to route the logic based on the new pipeline stage.
- [ ] **C.2 - Build and Test the "Surrender" Path:** Build the Postgres `Update` node for when a lead is moved to "Lost" and test it.
- [ ] **C.3 - Build the "Win" Path:** Build the more complex series of nodes that finds all competing agents and sends API calls to GHL to close out their opportunities.
- [ ] **C.4 - Perform Full End-to-End Test:** Manually move leads in GHL to test both the "Win" and "Surrender" conditions.

---
## Phase 3: Go-Live (Deployment) 🚀
*The goal of this phase is to migrate legacy data, perform final testing, and launch the system.*

### 3.1 Initial Data Migration
- [ ] **Prepare Legacy Data:** Clean and format the 50,000 legacy leads into the master CSV file with the correct headers.
- [ ] **Set Initial Status:** Ensure all leads in the CSV have `lead_type` set to `Recycle` and **`priority_level` set to `3`**.
- [ ] **Perform Bulk Import:** Import the clean CSV file directly into the PostgreSQL `master_contacts` table.

### 3.2 End-to-End Testing (Pilot Team)
- [ ] Select a pilot team (e.g., Team A) for the first live test.
- [ ] Run a special "Initial Distribution" n8n workflow for a small, controlled batch of legacy leads (e.g., 500 leads) for the pilot team.
- [ ] Monitor all system workflows (Registrar, Referee, Janitor, Dispatch) for 3-5 days.
- [ ] Collect feedback from the pilot team on lead quality and system behavior.
- [ ] Make any final adjustments or bug fixes to the n8n workflows.

### 3.3 Full Rollout
- [ ] Announce the official "Go-Live" date to all team leaders.
- [ ] Run the "Initial Distribution" workflow in larger, controlled batches for all remaining teams over a period of several days.
- [ ] Enable all scheduled n8n workflows to run on their automatic schedules.
- [ ] Closely monitor system health and execution logs for the first full week of operation.
- [ ] **Celebrate the launch of Project Phoenix! 🎉**
