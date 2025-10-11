# Project Phoenix: Implementation Roadmap & Checklist (v1.0)
* **Date:** October 12, 2025
* **Purpose:** This document provides a comprehensive, trackable checklist for the complete implementation of the Project Phoenix system.

## Introduction: The Path to Launch

This roadmap is divided into three major phases: **Foundation**, **Automation Build**, and **Go-Live**. Each phase contains a detailed checklist of tasks. Mark each item as complete as you progress to track our journey to launching the most powerful lead management system in the game.

---
## Phase 1: Foundation (Setup) 🏛️
*The goal of this phase is to build and configure all the core infrastructure.*

### 1.1 PostgreSQL Database Setup
- [ ] **Deploy PostgreSQL Server:** Ensure a stable PostgreSQL instance is running and accessible.
- [ ] **Create Database:** Run the SQL command to create the `phoenix_db` database.
- [ ] **Create User:** Run the SQL command to create the `phoenix_user` and grant all necessary privileges to `phoenix_db`.
- [ ] **Execute Schema Script:** Run the final SQL script to create all four master tables: `master_contacts`, `agents`, `routing_rules`, and `assignments`.
- [ ] **Initial Data Population:**
    - [ ] Populate the `agents` table with the initial list of all participating agents, including their correct `ghl_user_id` and `ghl_location_id`.
    - [ ] Populate the `routing_rules` table with the initial project/location rules for each team.

### 1.2 n8n Controller Setup
- [ ] **Install Server Prerequisites:** Install Docker and Docker Compose on the Hostinger VPS.
- [ ] **Create Project Directory:** Create the `n8n-phoenix` directory on the server.
- [ ] **Create `docker-compose.yml` File:** Create the file with the correct configuration.
- [ ] **Create and Secure `.env` File:** Create the `.env` file and populate it with all database credentials and the n8n subdomain.
- [ ] **Launch n8n Container:** Run `docker-compose up -d` to start the n8n instance.
- [ ] **Configure DNS & SSL:** Point your n8n subdomain's DNS A record to the server IP and ensure SSL (HTTPS) is correctly configured.
- [ ] **Create n8n Credentials:** Inside the n8n UI, create a separate "Header Auth" credential for each GHL sub-account's API key.

### 1.3 GHL Sub-Account Configuration
*(This entire section must be repeated for **EACH** participating GHL sub-account)*
- [ ] **Configuration:**
    - [ ] Generate and securely store the sub-account's **API Key**.
    - [ ] Create the **11-stage sales pipeline** exactly as defined in the blueprint.
    - [ ] Create all required **Custom Fields** under the correct groups.
    - [ ] Confirm the **"Allow Duplicate Contact"** setting is turned **OFF**.
- [ ] **Internal Automations (Workflows):**
    - [ ] Build the "Fresh Lead Capture & Notify" workflow.
    - [ ] Build the series of "30-Day Cool Down" workflows (e.g., Qualifying -> Pending, Pending -> Lost).

---
## Phase 2: Automation Build (Development) 🧠
*The goal of this phase is to build the four core n8n workflows that run the system.*

### 2.1 Workflow A: Real-Time Registrar
- [ ] Build the webhook-triggered workflow logic.
- [ ] Test by submitting a lead through a connected GHL ad form.
- [ ] Verify the lead is correctly created in the `master_contacts` and `assignments` tables.

### 2.2 Workflow D: The Timer & Status Janitor
- [ ] Build the scheduled workflow logic that queries the `assignments` table for expired timers.
- [ ] Test by manually setting a `timer_expires_at` value in the database to a past date and running the workflow.
- [ ] Verify the lead's `priority_level` is correctly updated in `master_contacts`.

### 2.3 Workflow B: The Morning Dispatch
- [ ] Build the scheduled workflow logic (reads rules, gets priority leads, checks capacity, assigns).
- [ ] Test with a sample set of "Recycled" leads in the database.
- [ ] Verify leads are correctly assigned to the right agents in GHL and the `assignments` table is updated.

### 2.4 Workflow C: The Real-Time Referee
- [ ] Build the webhook-triggered workflow logic to handle the "Win" and "Surrender" conditions.
- [ ] **Test "Win" Condition:** Move a test lead to "Pre-Booking" in one GHL account.
- [ ] Verify the competing lead in another GHL account is automatically moved to "Lost" with the correct system note.
- [ ] **Test "Surrender" Condition:** Manually move a test lead to "Lost."
- [ ] Verify the lead is correctly updated in the Master DB and made available for the next dispatch.

---
## Phase 3: Go-Live (Deployment) 🚀
*The goal of this phase is to migrate legacy data, perform final testing, and launch the system.*

### 3.1 Initial Data Migration
- [ ] **Prepare Legacy Data:** Clean and format the 50,000 legacy leads into the master CSV file.
- [ ] **Set Initial Status:** Ensure all leads in the CSV have `lead_type` set to `Recycle` and `current_status` set to `Pending_Initial_Distribution`.
- [ ] **Perform Bulk Import:** Import the clean CSV file directly into the PostgreSQL `master_contacts` table.

### 3.2 End-to-End Testing (Pilot Team)
- [ ] Select a pilot team (e.g., Team A) for the first live test.
- [ ] Run the special "Initial Distribution" n8n workflow for a small, controlled batch of legacy leads (e.g., 500 leads) for the pilot team.
- [ ] Monitor all system workflows (Registrar, Referee, Janitor, Dispatch) for 3-5 days.
- [ ] Collect feedback from the pilot team on lead quality and system behavior.
- [ ] Make any final adjustments or bug fixes to the n8n workflows.

### 3.3 Full Rollout
- [ ] Announce the official "Go-Live" date to all team leaders.
- [ ] Run the "Initial Distribution" workflow in larger, controlled batches for all remaining teams over a period of several days.
- [ ] Enable all scheduled n8n workflows to run on their automatic schedules.
- [ ] Closely monitor system health and execution logs for the first full week of operation.
- [ ] **Celebrate the launch of Project Phoenix! 🎉**
