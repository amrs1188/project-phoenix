# Project Phoenix: The Definitive Blueprint (v8.0 FINAL)
* **Date:** October 19, 2025
* **Status:** Final Blueprint - Awaiting Implementation
* **Authored By:** Gemini, in collaboration with the Project Visionary

## Part 1: The Vision - Project Proposal

### 1.1 Project Objectives
Project Phoenix is a strategic initiative designed to revolutionize the lead management process for a high-velocity, competitive real estate sales environment. The primary objectives are to:
* **Migrate** from a traditional, manual Excel sheet-based system to a powerful, automated CRM infrastructure.
* **Enable** full system accessibility and lead management from mobile devices, empowering agents to act instantly.
* **Track** the responsiveness, activity, and success of hundreds of competing agents with data-driven precision.
* **Facilitate** a high-pace, "royal rumble" business model, where speed-to-action is paramount and the most effective agents are rewarded.
* **Ensure** consistent daily lead flow to active agents via a defined quota system.

### 1.2 The Core Philosophy: The "High-Velocity Royal Rumble"
This system embodies an aggressive, performance-driven approach. Leads are assets that must be actioned rapidly. Stagnation is unacceptable. The system creates a competitive marketplace where:
* **Fresh Leads** are given priority circulation within their originating team.
* **All Leads** operate on accelerated timelines.
* **Recycled Leads** are intelligently distributed globally based on priority and a daily quota.
* The first agent to achieve the "Pre-Booking" milestone definitively wins the lead, ending the competition.

### 1.3 Executive Summary
Project Phoenix is a three-part automated system using **GoHighLevel (GHL)** as the agent-facing "arena," a central **PostgreSQL** database as the "master record," and a self-hosted **n8n** instance as the intelligent "referee." It employs a **"Local Recycle First"** strategy for new leads and a **"Weighted Priority Quota"** system for distributing older leads globally. All lead assignments operate on standardized fast timers (24h/3d). The system enforces a "first to Pre-Booking wins" rule via the n8n Referee.

---
## Part 2: Definitions - Lead States & Cycles

Understanding the different states and cycles of a lead is crucial.

### 2.1 Lead Types
* **Fresh Lead:** A brand new lead acquired directly from an ad source (e.g., Meta, TikTok) and initially assigned within its originating GHL sub-account. Stored in `master_contacts` with `lead_type = 'Fresh'`.
* **Recycle Lead:** A lead that was once "Fresh" but has completed its local recycle loop within the originating sub-account without being won. Stored in `master_contacts` with `lead_type = 'Recycle'`. Legacy leads start as Recycle leads.

### 2.2 Recycle Cycles
* **Local Recycle:** The process specific to **Fresh Leads**. When a Fresh Lead's timer expires, it is flagged (e.g., `needs_local_recycle=true` in DB) and offered sequentially to other eligible agents *only within the original sub-account*. This cycle repeats until all local agents have had a chance.
* **Global Recycle:** The process applying to **Recycle Leads**. When a Recycle Lead's timer expires (or it's manually surrendered), it is assigned a `priority_level` in the database and enters the global pool, making it eligible for assignment to *any* suitable agent across *any* sub-account during the "Morning Dispatch."

---
## Part 3: The Ecosystem - Architecture & Technology

### 3.1 The Three Pillars
* **PostgreSQL (The Master Record 🏛️):** The single source of truth for all lead data, agent info, assignment history, routing rules, configurations (like quotas), and lead state.
* **n8n (The Referee 🧠):** The central brain executing all logic: registering Fresh leads, managing timers, orchestrating local and global recycling, applying quotas, checking capacity/history, communicating with GHL APIs, and enforcing win/loss conditions.
* **GHL (The Arena 🥊):** The agent's workspace for managing assigned leads through the pipeline. Internal automations handle initial Fresh lead capture and basic cool-down movements.

### 3.2 Technology Stack
* **Database:** PostgreSQL
* **Automation Controller:** n8n (Self-hosted)
* **CRM:** GoHighLevel (GHL)

---
## Part 4: The Complete Lead Lifecycle & System Logic

This details the journey and the rules governing it.

### 4.1 Standardized Fast Timers
To maintain high velocity, all active assignments adhere to strict, standardized timers:
* **Stages 1 & 2 (`New Lead`, `Contacting`):** **24 hours** to advance.
* **Stages 3-6 (`Qualifying`, `Pending`, `Incomplete`, `Joint-Loan`):** **3 days** to advance.

### 4.2 Genesis & Local Recycle (Fresh Leads)
1.  **Capture (GHL):** Ad lead lands in originating sub-account AAA, assigned to Agent A.
2.  **Notify & Register (GHL -> n8n):** GHL webhook sends data to n8n (Workflow A), which logs the lead (`lead_type='Fresh'`) and creates the first assignment for Agent A (24h timer starts).
3.  **Timer Expires (Agent A Fails):** Janitor (Workflow D) detects expiration. Because `lead_type='Fresh'`:
    * Sets Agent A's assignment `status='Lost'`.
    * Flags lead in `master_contacts`: `needs_local_recycle=true`, `original_location_id='AAA'`. **No `priority_level` is set.**
4.  **Morning Dispatch (Workflow B) - Local Check:** Finds the locally flagged lead. Searches *only* agents in location 'AAA'. Finds Agent B (next eligible).
5.  **Local Assignment:** Assigns lead to Agent B (via GHL API), creates new assignment record (Fresh timer starts), removes `needs_local_recycle` flag.
6.  **Loop Continues:** If Agent B's timer expires, the Janitor flags it again. Morning Dispatch finds Agent C in AAA, etc.
7.  **Local Exhaustion -> Transition:** Morning Dispatch searches AAA but finds no more eligible agents. It performs the critical transition:
    * Removes `needs_local_recycle` flag.
    * Updates `master_contacts`: `lead_type` changes from `Fresh` to **`Recycle`**.
    * Sets **`priority_level = 2`** (Warm) to enter the global pool.

### 4.3 Global Recycle & Weighted Priority Quota (Recycle Leads)
1.  **Entry into Global Pool:** A lead becomes eligible for Global Recycle when:
    * A Fresh lead completes its local cycle (set to P2).
    * A Recycle lead's timer expires (Janitor sets P1, P2, or P3).
    * An agent manually surrenders a Recycle lead (Referee sets P1, P2, or P3).
    * Legacy leads are imported (set to P3).
2.  **Morning Dispatch (Workflow B) - Global Distribution:** Runs daily at 12:10 AM.
    * **Initialize Counters:** Resets daily assignment counts for all agents to zero.
    * **Get Leads:** Fetches all leads with a `priority_level`, sorted P1 -> P2 -> P3.
    * **Process Leads:** Loops through the sorted list one lead at a time.
    * **Find Team & Agents:** Determines the correct team via `routing_rules`, gets the roster.
    * **Intelligent Assignment Loop:** Checks agents on the roster one-by-one:
        * **Quota Check:** Has agent received their daily quota (e.g., 30 leads) *this run*? (Respecting weighted percentages: try to fill P1 target first, then P2, then P3). If yes, skip.
        * **History Check:** Has agent ever had this lead? If yes, skip.
        * **Capacity Check:** Does agent have < 60 leads (Stages 1&2)? If no, skip.
    * **Delivery:** If an agent passes: Assigns lead (via GHL API), creates `assignments` record (Recycle timer starts), removes `priority_level`, increments agent's daily counter.
3.  **Continues:** Until all priority leads are processed or all eligible agents hit quota/capacity.

### 4.4 Priority Level Definitions (v3.0)
* **Priority 1 (Hottest 🔥):** Set by Janitor/Referee for **Recycle leads** returned from stages `Qualifying`, `Pending`, `Incomplete`, or `Joint-Loan`.
* **Priority 2 (Warm ☀️):** Set by Janitor/Referee for **Recycle leads** returned from the `Contacting` stage. Set by **Morning Dispatch** when a **Fresh lead transitions** to become a global **Recycle lead**.
* **Priority 3 (Cold ❄️):** Set by Janitor/Referee for **Recycle leads** returned from the `New Lead` stage. All **legacy leads** imported via CSV.

### 4.5 The Finish Line: The "Win" Condition & Referee Protocol 🏆
1.  **Trigger:** Any agent moves any lead to **Stage 7: Pre-Booking**.
2.  **Referee Acts (Workflow C):** Receives webhook signal.
3.  **Identify Winner:** Uses webhook data (`contact_id`, `location_id`) to query GHL API, find the `assignedTo` user ID, look up the internal `agent_id`.
4.  **Declare Winner:** Updates the winning `assignments` record (`status='Won'`).
5.  **Find & Notify Losers:** Queries DB for all other `Active` assignments for that `contact_id`. Loops through them:
    * Looks up correct API Token using `ghl_configs` table.
    * Gets the specific Opportunity ID via GHL API.
    * Sends GHL API command to move loser's Opportunity to "Lost".
    * Sends GHL API command to add system note.
    * Updates loser's `assignments` record in DB (`status='Lost'`).

### 4.6 The Surrender: Manual "Lost" Trigger
1.  **Trigger:** Agent moves lead to `Lost` stage.
2.  **Referee Acts (Workflow C):** Receives webhook. Routes to "Surrender" path.
3.  **Update Assignment:** Updates the specific `assignments` record (`status='Lost'`, `current_stage='Lost'`, `closed_at` set).
4.  **Recycle Immediately:** Updates `master_contacts` record, setting appropriate `priority_level` based on the stage the lead was in before being lost.

---
## Part 5: The Arena - GHL Configuration

### 5.1 The Sales Pipeline (11 Stages)
* 1: New Lead
* 2: Contacting
* 3: Qualifying
* 4: Pending
* 5: Incomplete
* 6: Joint-Loan
* 7: **Pre-Booking (Win Stage)**
* 8: Confirmed Booking
* 9: Won - Confirmed Sales
* 10: Lost
* 11: Unqualified

### 5.2 Essential GHL Automations (Per Sub-Account)
* **Fresh Lead Capture & Notify:** Captures ads, tags, assigns locally, calls n8n Custom Webhook.
* **30-Day Cool Down Series:** Automatically moves stalled leads `Qualifying` -> `Pending` -> `Lost`.
* **Notify n8n Referee on Stage Change:** Simple workflow using standard Webhook action triggered on any stage change, sending data to n8n Workflow C.

---
## Part 6: n8n Controller Workflows - Detailed Objectives

### 6.1 Workflow A: Real-Time Registrar
* **Objective:** Officially log new Fresh Leads into the master system and start their initial timer.
* **Trigger:** Webhook. **Process:** Receives data -> Inserts into `master_contacts` -> Inserts into `assignments` (with Fresh timer).

### 6.2 Workflow D: Timer & Status Janitor
* **Objective:** Continuously monitor all active assignments, identify expired timers, and initiate the correct recycling process (flag for local or set global priority).
* **Trigger:** Schedule (hourly). **Process:** Finds expired assignments -> Checks `lead_type` -> If 'Fresh', flags for local recycle in `master_contacts` & closes assignment -> If 'Recycle', sets `priority_level` in `master_contacts` & closes assignment.

### 6.3 Workflow B: The Morning Dispatch
* **Objective:** Intelligently distribute all available leads daily, respecting local recycle priority, global priority weighting, daily quotas, agent capacity, and history.
* **Trigger:** Schedule (daily @ 12:10 AM). **Process:** Initializes counters -> Processes local recycles first (finds next local agent, assigns if found, transitions to global if not) -> Processes global recycles (gets P1/P2/P3 leads, loops through agents checking quota/history/capacity, assigns via API, updates DB & counters).

### 6.4 Workflow C: The Real-Time Referee
* **Objective:** Instantly enforce the "Win" and "Surrender" rules based on real-time stage changes from GHL.
* **Trigger:** Webhook. **Process:** Receives stage change -> Switch routes based on stage -> If 'Lost', updates assignment & sets `priority_level` -> If 'Pre-Booking', identifies winner, updates winner's assignment, finds losers, updates losers in GHL (via API) & DB.

---
## Part 7: The Master Record - PostgreSQL Schema (Updates Required)

* **Table: `master_contacts`:**
    * **Add Column:** `needs_local_recycle` (BOOLEAN, default FALSE)
    * **Add Column:** `original_location_id` (VARCHAR(100), nullable)
    * *Existing Columns:* `contact_id`, `phone`, `email`, `first_name`, `last_name`, `lead_type` ('Fresh'/'Recycle'), `priority_level` (1/2/3), `lead_source`, custom fields..., `notes_history`, `created_at`.
* **Table: `agents`:** `agent_id`, `agent_name`, `ghl_user_id`, `ghl_location_id`, `team_id`, `is_active`.
* **Table: `routing_rules`:** `rule_id`, `team_id`, `rule_type`, `rule_value`.
* **Table: `assignments`:** `assignment_id`, `contact_id` (FK), `agent_id` (FK), `status` ('Active'/'Won'/'Lost'), `current_stage`, `ghl_contact_id`, `timer_expires_at`, `assigned_at`, `closed_at`.
* **Table: `ghl_configs`:** `location_id` (PK), `cf_project_state_id`, custom field IDs..., `auth_header_value`.
* **(Optional but Recommended) Table: `system_settings`:**
    * Could store global settings like `daily_quota` (INT, default 30), timer durations, etc., making them easily configurable without editing workflows.

---
## Part 8: Implementation Roadmap Summary
*(Refer to Detailed Checklist v2.0 for specifics)*
1.  **Phase 1: Foundation:** Setup DB (including schema updates), configure GHL (including updated automations), setup n8n.
2.  **Phase 2: Automation Build:** Build & Unit Test n8n Workflows A, D, B (including quota logic), C (including revised Win path).
3.  **Phase 3: Go-Live:** Migrate legacy data (set P3), End-to-End Pilot Test, Full Rollout & Monitoring.
