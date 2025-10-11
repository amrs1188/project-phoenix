# Project Phoenix: System Proposal & Architectural Blueprint (v6.0)
* **Date:** October 12, 2025
* **Status:** Final Blueprint - Awaiting Implementation

## Part 1: Project Proposal & Executive Summary

### 1.1 Project Objectives
Project Phoenix is a strategic initiative designed to revolutionize the lead management process for a high-velocity, competitive real estate sales environment. The primary objectives are to:
* **Migrate** from a traditional, manual Excel sheet-based system to a powerful, automated CRM infrastructure.
* **Enable** full system accessibility and lead management from mobile devices, empowering agents to act instantly.
* **Track** the responsiveness, activity, and success of hundreds of competing agents with data-driven precision.
* **Facilitate** a high-pace, "royal rumble" business model, where speed-to-action is paramount and the most effective agents are rewarded.

### 1.2 Executive Summary
Project Phoenix is a three-part automated system designed to manage a competitive lead marketplace. It uses **GoHighLevel (GHL)** as the agent-facing "arena," a central **PostgreSQL** database as the "master record," and a self-hosted **n8n** instance as the intelligent "referee" that enforces the rules of the game.

The core philosophy is the **"Simultaneous Competition" Protocol**. Leads are dispatched to agents for a limited time. If progress isn't made, the opportunity is cloned and dispatched to a *new* agent, allowing multiple, independent agents to compete for the same lead. The first agent to advance the lead to the "Pre-Booking" stage wins, and the system automatically closes the opportunity for all other competitors. This creates maximum urgency and increases the probability of a sale.

### 1.3 Key Performance Indicators (KPIs)
The success of Project Phoenix will be measured by:
* **Lead Velocity:** The time it takes for a returned lead to be re-dispatched to a new agent.
* **Agent Engagement Rate:** The percentage of leads actioned by agents before the first timer expires.
* **Close Rate per Agent:** The ultimate measure of an agent's effectiveness within the system.

---
## Part 2: System Architecture & Technology

### 2.1 The Three Pillars
* **PostgreSQL (The Master Record 🏛️):** The robust, central database. It is the single source of truth for every person, every assignment, and all business rules.
* **n8n (The Referee 🧠):** The central brain. n8n executes all logic: dispatching leads, checking timers, managing competition, prioritizing opportunities, and updating the database.
* **GHL (The Arena 🥊):** The agent's workspace. Each agent/team operates in their own GHL sub-account where they manage their assigned leads through the sales pipeline.

### 2.2 Technology Stack
* **Database:** PostgreSQL
* **Automation Controller:** n8n (Self-hosted)
* **CRM:** GoHighLevel (GHL)

---
## Part 3: The GHL Arena - Pipeline & Configuration

### 3.1 The Sales Pipeline Stages
This 11-stage pipeline provides a granular view of the lead's journey.

| Stage # | Stage Name | Purpose |
| :--- | :--- | :--- |
| 1 | **New Lead** | Freshly assigned leads land here. |
| 2 | **Contacting** | Agent is actively attempting first contact. |
| 3 | **Qualifying** | Active financial assessment and needs analysis. |
| 4 | **Pending** | Lead has gone silent or is unresponsive. |
| 5 | **Incomplete** | Lead is providing documents, but they are incomplete. |
| 6 | **Joint-Loan** | A joint-loan application is being explored. |
| 7 | **Pre-Booking** | **The "Win" stage.** Documents are complete and submitted to ERP. |
| 8 | **Confirmed Booking**| ERP system has confirmed the booking. |
| 9 | **Won - Confirmed Sales**| The final stage for a successful deal. |
| 10 | **Lost** | Agent has manually lost the lead, or the system has closed it. |
| 11 | **Unqualified** | Lead is deemed unqualified but may become qualified in the future. |

### 3.2 Core GHL Automations
The following workflows must be configured *within* GHL:
* **Note Automation:** Every time a lead is moved from one stage to another automatically, an internal note must be added (e.g., "System: Moved contact to Pending stage due to inactivity.").
* **30-Day Cool Down Timers:** For leads in stages 3, 4, 5, and 6, a 30-day "Wait" step is required. If the lead is still in that stage after 30 days, it is automatically moved to the next "cool down" stage (e.g., Qualifying -> Pending, Pending -> Lost).

---
## Part 4: The Core Logic - The "Royal Rumble" Engine

This is the operational heart of Project Phoenix.

### 4.1 The "Simultaneous Competition" Protocol
The system is designed to allow multiple agents to work the same lead if it is not closed within the initial timeframes. This maximizes opportunities and rewards the most effective closers.

### 4.2 Phase 1: Exclusive Assignment (Stages 1 & 2)
* **Timer:** A strict **48-hour** timer is active in these stages.
* **Capacity:** Distribution is limited by a combined capacity of **60 leads** across these two stages. An agent at capacity will not receive their daily quota of 30 leads.
* **Action:** If the 48-hour timer expires, the lead's data is sent to the Master DB to be re-dispatched to a new agent. The original agent's lead begins its "Cool Down" lifecycle (moving to Pending after 30 days, then Lost).

### 4.3 Phase 2: The Rumble Begins (Stages 3, 4, 5, 6)
* **Timer:** A **10-day** "progress" timer is active in these stages.
* **Action:** If the 10-day timer expires without the lead moving forward, the lead's data is sent to the Master DB for re-dispatch. The original agent's lead remains active, and a new agent now enters the competition.

### 4.4 The "Win" Condition & Referee Protocol 🏆
This is the rule that governs the end of the race.
1.  When any agent moves a lead into **Stage 7: Pre-Booking**, a real-time trigger fires to the n8n Referee.
2.  n8n instantly locks in the winning agent in the Master DB.
3.  n8n immediately sends commands to the GHL accounts of **all other competing agents** who have an active version of this lead.
4.  The command automatically moves the lead to the **Stage 10: Lost** for all non-winning competitors and adds a system note: *"Opportunity closed by another representative."*

### 4.5 Lead Priority System
Recycled leads are not treated equally. A `priority_level` will be set in the database upon return.
* **Priority 1 (Hottest):** Leads returned from Stages 3-6 (Qualifying, Pending, etc.).
* **Priority 2 (Warm):** Leads returned from Stage 2 (Contacting).
* **Priority 3 (Cold):** Leads returned from Stage 1 (New Lead).
The daily distribution workflow will always dispatch the highest-priority leads first.

---
## Part 5: The Master Database - PostgreSQL Schema

* **Table: `master_contacts`:** The master record of every unique person (identified by phone number).
* **Table: `agents`:** The roster of all competing agents and their GHL IDs.
* **Table: `routing_rules`:** The logic for assigning leads to specific teams (e.g., based on location).
* **Table: `assignments`:** The active state machine. This table tracks every single assignment of a contact to an agent, including its current `status` (`Active`, `Won`, `Lost`), the GHL stage, and timer deadlines. This is the referee's scorecard.

---
## Part 6: The n8n Controller - Workflow Blueprints

To execute this logic, four core workflows will be built in n8n.

* **Workflow A: Real-Time Fresh Lead Ingestion:** Captures new leads, logs them, and performs the initial "check-out" to an agent.
* **Workflow B: The Morning Dispatch:** The daily scheduled workflow that runs at 12:10 AM. It checks agent capacity, prioritizes recycled leads, and distributes the daily quota.
* **Workflow C: The Real-Time Referee:** A webhook-based workflow that listens for stage changes in GHL. It is responsible for enforcing the "Win" condition.
* **Workflow D: The Timer & Status Janitor:** A scheduled workflow that runs frequently (e.g., every hour) to check for expired timers in the `assignments` table and trigger the re-dispatch process.

---
## Conclusion & Next Steps

This document outlines a complete, robust, and powerful system designed specifically for a high-velocity, competitive sales environment. It provides clear tracking, creates urgency, and rewards performance.

The next phase is **Implementation**. The immediate next steps are:
1.  Set up the PostgreSQL database with the defined schema.
2.  Configure the GHL sub-accounts with the specified pipeline and custom fields.
3.  Begin development of the n8n workflows, starting with Workflow A.
