# Project Phoenix: Automation & Workflow Master List (v4.0)
* **Date:** October 19, 2025
* **Purpose:** This document provides a comprehensive summary and detailed description of every automated workflow required to run the Project Phoenix system, separated by platform (GHL and n8n). This aligns with the Definitive Blueprint v8.0.

## Part 1: GHL Internal Automations (The Arena's Local Rules 🏟️)

These workflows must be created **inside each** GHL sub-account. They handle tasks specific to that team's environment.

### **1.1. Workflow: Fresh Lead Capture & Notify**
* **Name:** `Fresh Lead Capture & Notify`
* **Trigger:** `Facebook Lead Form Submitted` (or other ad platform triggers).
* **Objective:** To instantly capture a new lead from an ad, assign it to a local agent for immediate action, and register its existence with the central n8n controller. This workflow is optimized for maximum speed-to-lead.
* **Detailed Process:**
    1.  Triggers upon ad form submission.
    2.  Tags the Contact: `lead_type:fresh`.
    3.  Sets the `Lead Source` custom field.
    4.  Assigns to a local agent via GHL's internal round-robin (`Assign to User`).
    5.  Calls a **Custom Webhook**, sending a clean JSON package of the lead's data (including `contact_id`, `location_id`, basic details, and source) to the n8n "Real-Time Registrar" workflow.
* **Outcome:** A new contact exists in the GHL sub-account, assigned to an agent in the "New Lead" stage. n8n has received the data to log it in the Master DB and start the 24-hour timer.

### **1.2. Workflow Series: 30-Day Cool Down**
* **Name Example:** `Cool Down: Qualifying to Pending`
* **Trigger:** `Pipeline Stage Changed` (e.g., To "Qualifying").
* **Objective:** To prevent leads from stagnating indefinitely in an agent's active pipeline by automatically de-escalating them after 30 days of inactivity, keeping the active view clean. (Note: This is separate from the n8n timers that trigger recycling).
* **Detailed Process:**
    1.  Triggers when a lead enters a specific stage (e.g., "Qualifying").
    2.  Waits for 30 days.
    3.  Checks (If/Else) if the lead is *still* in that same stage.
    4.  If yes, moves the Opportunity to the next "cool down" stage (e.g., "Pending") and adds a system note.
* **Outcome:** Stalled leads are visually de-escalated in the agent's GHL pipeline over a longer timeframe. This complements, but does not replace, the faster n8n-driven recycling timers.

### **1.3. Workflow: Notify n8n Referee on Stage Change**
* **Name:** `Notify n8n Referee on Stage Change`
* **Trigger:** `Pipeline Stage Changed` (filtered only for the "Project Phoenix Sales" pipeline).
* **Objective:** To provide the central n8n Referee with real-time updates on every lead movement within the sales pipeline, enabling the "Win" and "Surrender" rules.
* **Detailed Process:**
    1.  Triggers on any stage change within the designated pipeline.
    2.  Uses the standard **"Execute Webhook"** action (Method: POST) to send a data package (containing `contact_id`, `location_id`, stage names/IDs) to the n8n "Real-Time Referee" workflow's unique Production URL.
* **Outcome:** n8n is instantly aware of all stage changes, allowing immediate enforcement of competition rules.

---
## Part 2: n8n Controller Workflows (The Referee's Brain 🧠)

These are the four core workflows in the self-hosted n8n instance, controlling the global logic.

### **2.1. Workflow A: Real-Time Registrar**
* **Name:** `SYSTEM: Real-Time Registrar`
* **Trigger:** Webhook (listens for GHL's "Fresh Lead Capture" workflow).
* **Objective:** To officially log new Fresh Leads into the master database, creating their permanent record and initiating the standardized fast timer (24 hours).
* **Detailed Process:**
    1.  A **Webhook** node (`ReceiveFreshLeadData`) receives the JSON package from GHL.
    2.  A **Postgres (Insert)** node (`CreateMasterContactRecord`) creates the record in `master_contacts`, setting `lead_type='Fresh'` and storing fields like phone, name, source, `original_location_id`.
    3.  A second **Postgres (Insert)** node (`CreateInitialAssignment`) creates the first record in the `assignments` table, linking `contact_id` to the `agent_id`, setting `status='Active'`, `current_stage='New Lead'`, storing `ghl_contact_id`, and calculating `timer_expires_at` (NOW + 24 hours).
* **Outcome:** The Fresh Lead is officially registered in PostgreSQL. Its first 24-hour performance timer is active.

### **2.2. Workflow D: Timer & Status Janitor**
* **Name:** `SYSTEM: Timer & Status Janitor`
* **Trigger:** Schedule (runs every hour).
* **Objective:** To continuously monitor all active assignments, identify expired timers (24h for Stages 1&2, 3d for Stages 3-6), and initiate the correct recycling process based on lead type (flag Fresh for local recycle, set priority for Recycle leads).
* **Detailed Process:**
    1.  A **Schedule** node triggers the workflow.
    2.  A **Postgres (Execute Query)** node (`FindExpiredAssignments`) finds all `assignments` records where `status = 'Active'` and `timer_expires_at < NOW()`, joining with `master_contacts` to get the `lead_type`.
    3.  The workflow loops through each expired assignment.
    4.  An **IF** node (`CheckLeadType`) checks if `lead_type` is 'Fresh' or 'Recycle'.
        * **If 'Fresh' (True Path):** A **Postgres (Update)** node (`FlagForLocalRecycle`) updates the `master_contacts` record: sets `needs_local_recycle=true` and stores `original_location_id`. It does **NOT** set `priority_level`.
        * **If 'Recycle' (False Path):** A **Postgres (Update)** node (`SetGlobalPriority`) updates the `master_contacts` record: sets the `priority_level` (1, 2, or 3 based on the `current_stage` the timer expired in).
    5.  A final **Postgres (Update)** node (`CloseExpiredAssignment`) runs regardless of the IF path, updating the original `assignments` record: sets `status='Lost'` and fills `closed_at`.
* **Outcome:** Expired assignments are closed. Fresh leads are flagged for local recycling. Recycle leads are prioritized and placed in the global pool.

### **2.3. Workflow B: The Morning Dispatch**
* **Name:** `SYSTEM: The Morning Dispatch`
* **Trigger:** Schedule (runs daily at 12:10 AM).
* **Objective:** To intelligently distribute all available leads (prioritizing local recycles, then global weighted priorities) according to the daily quota, agent capacity, and assignment history.
* **Detailed Process:**
    1.  **Schedule** trigger fires.
    2.  **(New) Initialize Counters:** Resets internal counters for daily quota per agent (e.g., using n8n Static Data or a temporary DB table).
    3.  **(New) Process Local Recycles:**
        * Queries `master_contacts` for leads with `needs_local_recycle=true`.
        * Loops through these leads. For each:
            * Queries `agents` for active agents in the `original_location_id`.
            * Loops through local agents, checking history (`assignments` table).
            * Finds the first eligible local agent.
            * If found: Performs **Delivery Actions** (see step 6), removes `needs_local_recycle` flag, **stops loop for this lead**.
            * If **NO** eligible local agent found: Removes `needs_local_recycle` flag, updates `master_contacts` `lead_type` to `Recycle`, sets `priority_level=2`.
    4.  **Process Global Recycles:**
        * **Get Leads:** Queries `master_contacts` for all leads with a `priority_level`, sorted P1 -> P2 -> P3.
        * Loops through these leads. For each:
            * **Find Team & Agents:** Looks up `team_id` via `routing_rules`, gets roster from `agents`.
            * **Intelligent Assignment Loop (Global):** Loops through agents on the roster:
                * **Quota Check:** Has agent hit daily quota (e.g., 30) *this run*? (Applying weighted logic: try to fill P1 target, then P2, then P3 within the total quota). If yes, skip.
                * **History Check:** Has agent ever had this lead? If yes, skip.
                * **Capacity Check:** Does agent have < 60 leads (Stages 1&2)? (Uses **HTTP Request** node with dynamic auth via `ghl_configs` lookup). If no, skip.
                * If agent passes all checks: Proceed to Delivery.
    5.  **Delivery Actions (Used by both Local & Global):**
        * **Get Config:** Looks up `ghl_configs` for the agent's location to get API token and field IDs.
        * **Create Contact:** Uses **HTTP Request (POST)** with dynamic auth and field IDs to create/update contact in GHL, assigning to the agent.
        * **Create Opportunity:** Uses **HTTP Request (POST)** with dynamic auth to create the opportunity in the "New Lead" stage.
        * **Create Assignment Record:** Uses **Postgres (Insert)** to create the new `assignments` record (with correct 24h/3d timer based on stage).
        * **Update Master Contact:** Uses **Postgres (Update)** to remove `priority_level`.
        * **Increment Quota Counter:** Updates the agent's counter for this run.
        * **Stop Loop:** A **NoOp ("Execute Once")** node stops processing further agents for *this specific lead*.
* **Outcome:** Local recycles are handled first. Global recycles are distributed according to priority, quota, capacity, and history. Agents start the day with a managed workload.

### **2.4. Workflow C: The Real-Time Referee**
* **Name:** `SYSTEM: Real-Time Referee`
* **Trigger:** Webhook (listens for GHL's "Notify n8n Referee" workflow).
* **Objective:** To instantly enforce the "Win" and "Surrender" rules based on real-time stage changes, ensuring data integrity and fair competition.
* **Detailed Process:**
    1.  A **Webhook** node (`ReceiveGHLStageChange`) receives the stage change data.
    2.  A **Switch** node (`CheckStageType`) routes based on the new stage name.
    3.  **If "Surrender" (stage is `Lost`):**
        * **Postgres (Update)** node (`UpdateAssignmentOnSurrender`) updates the `assignments` record (`status='Lost'`, `current_stage='Lost'`, `closed_at` set).
        * **Postgres (Update)** node (`MarkContactForRedispatch`) updates `master_contacts` to set the appropriate `priority_level` based on the stage the lead was in *before* being lost.
    4.  **If "Win" (stage is `Pre-Booking`):**
        * **HTTP Request (GET)** node (`GetOpportunityOwner`) queries GHL API to confirm the actual owner (`assignedTo`) of the opportunity that triggered the webhook.
        * **Postgres (Select)** node (`FindWinningAgentInternalID`) looks up the internal `agent_id` for the GHL owner.
        * **Postgres (Update)** node (`DeclareWinningAssignment`) updates the winning `assignments` record (`status='Won'`).
        * **Postgres (Execute Query)** node (`FindCompetingAssignments`) finds all other `Active` assignments for the same `contact_id`, joining with `agents` to get `ghl_location_id`.
        * Loops through each losing assignment:
            * **Postgres (Select)** node (`GetGHLConfigs`) fetches API token and field IDs for the loser's location.
            * **HTTP Request (GET)** node (`GetLoserOpportunityID`) finds the specific Opportunity ID in the loser's GHL.
            * **HTTP Request (PUT)** node (`MoveLoserToLostInGHL`) uses dynamic auth to move the loser's Opportunity to "Lost".
            * **HTTP Request (POST)** node (`AddSystemNoteToLoser`) uses dynamic auth to add the system note.
            * **Postgres (Update)** node (`UpdateLosingAssignmentDB`) updates the loser's `assignments` record (`status='Lost'`).
* **Outcome:** A "Win" immediately and cleanly ends the competition for all other agents. A "Surrender" immediately puts the lead back into the global pool with the correct priority.
