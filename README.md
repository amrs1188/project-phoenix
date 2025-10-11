# Project Phoenix: The Definitive Blueprint (v7.0 FINAL)
* **Date:** October 12, 2025
* **Status:** Final Blueprint - Awaiting Implementation
* **Authored By:** Gemini, in collaboration with the Project Visionary

## Part 1: The Vision - Project Proposal

### 1.1 Project Objectives
Project Phoenix is a strategic initiative designed to revolutionize the lead management process for a high-velocity, competitive real estate sales environment. The primary objectives are to:
* **Migrate** from a traditional, manual Excel sheet-based system to a powerful, automated CRM infrastructure.
* **Enable** full system accessibility and lead management from mobile devices, empowering agents to act instantly.
* **Track** the responsiveness, activity, and success of hundreds of competing agents with data-driven precision.
* **Facilitate** a high-pace, "royal rumble" business model, where speed-to-action is paramount and the most effective agents are rewarded.

### 1.2 The Core Philosophy: The "Royal Rumble"
This system is built to support a unique and aggressive business model. Unlike a traditional CRM that nurtures leads within a single team, Project Phoenix creates a competitive marketplace. A lead that stalls with one agent is not considered "dead"; it is a fresh opportunity for another. The system is designed to create, manage, and officiate this competition, ensuring that every lead is given the maximum number of chances to be closed by the most motivated and effective agent. Speed, persistence, and skill are the keys to victory.

### 1.3 Executive Summary
Project Phoenix is a three-part automated system. It uses **GoHighLevel (GHL)** as the agent-facing "arena," a central **PostgreSQL** database as the "master record," and a self-hosted **n8n** instance as the intelligent "referee." The system's engine is the **"Simultaneous Competition" Protocol**, where multiple agents can be assigned to the same lead over time, creating a high-urgency environment. The first agent to advance a lead to the "Pre-Booking" stage wins, and the system automatically closes the opportunity for all other competitors.

---
## Part 2: The Ecosystem - Architecture & Technology

### 2.1 The Three Pillars
* **PostgreSQL (The Master Record 🏛️):** The robust, central database. It is the single source of truth for every person, every assignment, and all business rules.
* **n8n (The Referee 🧠):** The central brain. n8n executes all logic: registering new leads, dispatching opportunities, checking timers, managing competition, and updating the database.
* **GHL (The Arena 🥊):** The agent's workspace. Each agent/team operates in their own GHL sub-account where they manage their assigned leads through the sales pipeline.

### 2.2 Technology Stack
* **Database:** PostgreSQL
* **Automation Controller:** n8n (Self-hosted)
* **CRM:** GoHighLevel (GHL)

---
## Part 3: The Complete Lead Lifecycle

This is the detailed, moment-by-moment journey of a lead in the Project Phoenix system.

### 3.1 Genesis: The Fresh Lead Flow
This is the *only* entry point for brand-new leads.
1.  **Capture (In GHL):** A team leader's ad (Meta, TikTok) generates a lead. GHL's native integration captures the contact directly inside the team's specific sub-account.
2.  **Internal GHL Automation:** An internal GHL workflow triggers instantly. It tags the contact (`lead_type:fresh`), sets the source, and assigns it to an agent within that team.
3.  **Notify the Referee:** The final step of the GHL workflow is to call a webhook, sending the new contact's data to the n8n controller.
4.  **Register in Master DB:** n8n receives the webhook and creates the permanent record in the PostgreSQL `master_contacts` table and a new entry in the `assignments` table, officially starting the 48-hour timer.

### 3.2 The First Round: Exclusive Window (Stages 1 & 2)
* **Timer:** A strict **48-hour** timer is active.
* **Capacity:** An agent's combined lead count in these two stages cannot exceed **60**.
* **If Timer Expires:** The n8n "Janitor" workflow detects the expired assignment. It updates the Master DB, making the lead available for re-dispatch as a "Recycled Lead." The original agent's copy of the lead begins its 30-day "cool down" journey inside their GHL.

### 3.3 The Rumble: Simultaneous Competition (Stages 3-6)
* **Timer:** A **10-day** "progress" timer is active in the Qualifying, Pending, Incomplete, and Joint-Loan stages.
* **The Action:** If the 10-day timer expires, the lead is sent back to the Master DB pool for re-dispatch to a *new* agent.
* **The Competition:** The original agent's lead remains active in their pipeline. At this point, two or more agents can be actively working on the same lead in their respective GHL accounts.

### 3.4 The Finish Line: The "Win" Condition & Referee Protocol 🏆
1.  **The Trigger:** An agent successfully moves a lead to **Stage 7: Pre-Booking**.
2.  **The Referee Acts:** The n8n "Referee" workflow catches this via a real-time webhook.
3.  **Declare Winner:** n8n locks in the winning agent in the `assignments` table in the Master DB.
4.  **End the Race:** n8n immediately sends API commands to the GHL accounts of all other competing agents to **automatically move their version of the lead to Stage 10: Lost** and adds a system note: *"Opportunity closed by another representative."*

### 3.5 The Surrender: Manual "Lost" Trigger
* If an agent manually moves a lead to the "Lost" stage, a webhook fires to the n8n Referee.
* n8n instantly updates the Master DB, making the lead available for immediate re-circulation. This ensures maximum lead velocity.

### 3.6 The Priority System
Recycled leads are prioritized to ensure the most valuable opportunities are worked first.
* **Priority 1 (Hottest):** Returned from Stages 3-6 (Qualifying, etc.)
* **Priority 2 (Warm):** Returned from Stage 2 (Contacting)
* **Priority 3 (Cold):** Returned from Stage 1 (New Lead)

---
## Part 4: The Arena - GHL Configuration

### 4.1 The Sales Pipeline
| Stage # | Stage Name |
| :--- | :--- |
| 1 | New Lead |
| 2 | Contacting |
| 3 | Qualifying |
| 4 | Pending |
| 5 | Incomplete |
| 6 | Joint-Loan |
| 7 | **Pre-Booking (Win Stage)** |
| 8 | Confirmed Booking |
| 9 | Won - Confirmed Sales |
| 10| Lost |
| 11| Unqualified |

### 4.2 Essential GHL Automations
The following workflows must be built *inside each* GHL sub-account:
* **Fresh Lead Capture & Notify:** Captures from ads, tags, assigns internally, and calls the n8n webhook.
* **Auto-Note on Stage Change:** Adds a system note whenever a lead is moved automatically.
* **30-Day Cool Down Timers:** A series of workflows that move leads from `Qualifying` -> `Pending` -> `Lost` if they remain inactive for 30 days in each stage.

---
## Part 5: The n8n Controller - Workflow Blueprints

Four core workflows must be built in n8n to run the system.

* **Workflow A: Real-Time Registrar:** A webhook-triggered workflow that listens for new Fresh Leads from GHL and registers them in the Master DB.
* **Workflow B: The Morning Dispatch:** A scheduled workflow (runs at 12:10 AM) that checks agent capacity and distributes all available "Recycled Leads" based on the priority system.
* **Workflow C: The Real-Time Referee:** A webhook-triggered workflow that listens for stage changes. It enforces the "Win" condition and handles "Manual Lost" surrenders.
* **Workflow D: The Timer & Status Janitor:** A scheduled workflow that runs frequently (e.g., every hour) to find assignments with expired timers and trigger the re-dispatch process.

---
## Part 6: The Master Record - PostgreSQL Schema

* **Table: `master_contacts`:** The master record of every unique person. Key columns: `contact_id`, `phone`, `email`, `lead_type` (`Fresh`/`Recycle`), `priority_level`.
* **Table: `agents`:** The roster of all competing agents. Key columns: `agent_id`, `agent_name`, `ghl_user_id`, `team_id`, `is_active`.
* **Table: `routing_rules`:** Defines which teams get leads based on certain criteria (e.g., location).
* **Table: `assignments`:** The referee's scorecard. Tracks every assignment of a contact to an agent. Key columns: `assignment_id`, `contact_id` (FK), `agent_id` (FK), `status` (`Active`/`Won`/`Lost`), `current_stage`, `timer_expires_at`.

---
## Part 7: Implementation Roadmap

1.  **Phase 1: Foundation (Setup)**
    * Deploy the PostgreSQL database and create the schema.
    * Configure all GHL sub-accounts with the pipeline, fields, and internal automations.
2.  **Phase 2: Automation Build (Development)**
    * Build, test, and deploy the four core n8n workflows in sequence.
3.  **Phase 3: Go-Live (Deployment)**
    * Perform end-to-end system testing with a pilot team.
    * Full rollout to all teams.
