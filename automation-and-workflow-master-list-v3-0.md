# Project Phoenix: Automation & Workflow Master List (v3.0)
* **Date:** October 15, 2025
* **Purpose:** This document provides a comprehensive summary and detailed description of every automated workflow required to run the Project Phoenix system, separated by platform (GHL and n8n).

## Part 1: GHL Internal Automations (The Arena's Local Rules 🏟️)

These workflows must be created **inside each** GHL sub-account. They handle tasks that are specific to that team's environment.

### **1.1. Workflow: Fresh Lead Capture & Notify**
* **Name:** `Fresh Lead Capture & Notify`
* **Trigger:** `Facebook Lead Form Submitted` (or other ad platform triggers).
* **Objective:** To instantly capture a new lead from an ad, assign it to a local agent for immediate action, and register its existence with the central n8n controller. This workflow is optimized for maximum speed-to-lead.
* **Detailed Process:**
    1.  The workflow triggers the moment a person submits their information on a connected ad form.
    2.  An **"Add Contact Tag"** action applies the `lead_type:fresh` tag to identify the lead's state.
    3.  An **"Update Contact Field"** action populates the `Lead Source` custom field (e.g., "Meta Ads - Project X") for tracking.
    4.  An **"Assign to User"** action uses GHL's internal round-robin feature to immediately assign the lead to an active agent on the local team.
    5.  The final, critical action is to call a **"Custom Webhook,"** which sends a clean JSON package of the new lead's data to the n8n "Real-Time Registrar" workflow.
* **Outcome:** A new contact exists in the GHL sub-account and is already in an agent's "New Lead" pipeline. Simultaneously, its data has been sent to n8n to be logged in the Master Database, officially starting the system's master timer.

### **1.2. Workflow Series: 30-Day Cool Down**
* **Name Example:** `Cool Down: Qualifying to Pending`
* **Trigger:** `Pipeline Stage Changed` (e.g., to "Qualifying").
* **Objective:** To prevent leads from stagnating indefinitely in an agent's active pipeline by automatically de-escalating them after a prolonged period of inactivity, thus keeping the active pipeline clean.
* **Detailed Process:**
    1.  A workflow triggers when a lead enters a specific stage (e.g., "Qualifying").
    2.  The workflow immediately enters a **"Wait"** step for a pre-defined period (e.g., 30 days).
    3.  After the wait, an **"If/Else"** condition checks if the lead is *still* in that same stage.
    4.  If it is, the workflow proceeds down the "Yes" path, where an **"Update Opportunity"** action moves the lead to the next "cool down" stage (e.g., from "Qualifying" to "Pending").
    5.  An **"Add Contact Note"** action then adds a system note for audit purposes (e.g., "System: Moved contact to Pending stage...").
* **Outcome:** Stalled leads are automatically moved to less active pipeline stages, ensuring an agent's "Qualifying" view only contains recently active leads. This pattern is replicated for other cool-down transitions (e.g., a separate workflow for `Pending` -> `Lost`).

### **1.3. Workflow: Notify n8n Referee on Stage Change**
* **Name:** `Notify n8n Referee on Stage Change`
* **Trigger:** `Pipeline Stage Changed` (filtered for the "Project Phoenix Sales" pipeline).
* **Objective:** To provide the central n8n Referee with real-time updates on every significant move a lead makes within any agent's sales pipeline, enabling the "royal rumble" rules.
* **Detailed Process:**
    1.  This simple workflow triggers every time an opportunity's pipeline stage is changed.
    2.  Its **only action** is to use the standard **"Execute Webhook"** action, which sends a pre-formatted data package (containing `contact_id`, `pipelineStageId_from`, `pipelineStageId_to`, etc.) to our n8n "Real-Time Referee" workflow.
* **Outcome:** The n8n controller is instantly aware of all stage changes, allowing it to enforce the "Win" and "Surrender" rules of the competition in real-time.

---
## Part 2: n8n Controller Workflows (The Referee's Brain 🧠)

These are the four core workflows that live in our self-hosted n8n instance and control the global logic.

### **2.1. Workflow A: Real-Time Registrar**
* **Trigger:** Webhook (listens for GHL's "Fresh Lead Capture" workflow).
* **Objective:** To serve as the official entry point for all "Fresh Leads" into the master database, creating their permanent record and initiating the system's tracking timers.
* **Detailed Process:**
    1.  A **Webhook** node receives the new Fresh Lead data package from a GHL sub-account.
    2.  A **Postgres (Insert)** node creates the permanent record for the person in the `master_contacts` table, giving them a unique internal `contact_id`.
    3.  A second **Postgres (Insert)** node creates the first record in the `assignments` table, linking the `contact_id` to the `agent_id`, setting the `status` to `Active`, and calculating the initial 48-hour `timer_expires_at` value.
* **Outcome:** The Fresh Lead is officially registered in the PostgreSQL database. Its first 48-hour performance timer is now active and being tracked by the central system.

### **2.2. Workflow D: Timer & Status Janitor**
* **Trigger:** Schedule (runs every hour).
* **Objective:** To automatically and continuously scan the system for stalled leads (expired timers) and officially place them into the re-dispatch pool.
* **Detailed Process:**
    1.  A **Schedule** node triggers the workflow hourly.
    2.  A **Postgres (Execute Query)** node runs a `SELECT` query to find all records in the `assignments` table where `status = 'Active'` and `timer_expires_at < NOW()`.
    3.  The workflow then loops through each expired assignment found.
    4.  A **Postgres (Update)** node updates the corresponding `master_contacts` record to set a `priority_level` (1, 2, or 3, depending on the stage the timer expired in).
    5.  A final **Postgres (Update)** node updates the original `assignments` record, changing its `status` to `Lost` and setting the `closed_at` timestamp.
* **Outcome:** Stalled leads are identified, their original assignment is officially closed, and they are marked with a priority level, making them eligible for the "Morning Dispatch."

### **2.3. Workflow B: The Morning Dispatch**
* **Trigger:** Schedule (runs daily at 12:10 AM).
* **Objective:** To act as the main distribution engine, intelligently assigning all available recycled leads to the most appropriate and available agents.
* **Detailed Process:**
    1.  A **Schedule** node triggers the workflow.
    2.  A **Postgres (Execute Query)** node gets a full list of all leads from `master_contacts` that have a `priority_level` set, sorted from hottest to coldest.
    3.  The workflow loops through each lead and performs a series of information-gathering steps:
        * Looks up the correct `team_id` from the `routing_rules` table.
        * Gets a full roster of all `Active` agents for that team.
    4.  It then begins an **Intelligent Assignment Loop**, checking each agent on the roster one by one against two critical conditions:
        * **History Check:** A database query confirms the agent has never been assigned this lead before.
        * **Capacity Check:** An **HTTP Request** node makes a GHL API call to ensure the agent has fewer than 60 leads in their initial pipeline stages.
    5.  Once an agent passes all checks, the **Delivery Actions** trigger:
        * An **HTTP Request** node sends a command to the GHL API to create the contact in the winning agent's sub-account.
        * A **Postgres (Insert)** node creates a new record in the `assignments` table for this new assignment, starting a new 48-hour timer.
* **Outcome:** All available recycled leads are intelligently assigned to active, eligible agents. Agents start their day with a fresh batch of leads in their GHL "New Lead" pipeline.

### **2.4. Workflow C: The Real-Time Referee**
* **Trigger:** Webhook (listens for GHL's "Notify n8n Referee" workflow).
* **Objective:** To act as the live-action umpire, instantly enforcing the "Win" and "Surrender" rules of the competition to keep the data clean and the game fair.
* **Detailed Process:**
    1.  A **Webhook** node receives a signal that a lead's stage has changed in GHL.
    2.  A **Switch** node inspects the new stage name to decide which path to take.
    3.  **If "Surrender" (stage is `Lost`):** The workflow executes a **Postgres (Update)** node to set a `priority_level` on the `master_contacts` record, immediately making the lead available for re-dispatch. It also updates the `assignments` table.
    4.  **If "Win" (stage is `Pre-Booking`):** The workflow executes a complex sequence:
        * It updates the database to mark the winning agent's assignment as `Won`.
        * It queries the database to find all other active, competing assignments for that same person.
        * It then loops through these "losing" assignments, sending an **HTTP Request** to each respective GHL sub-account to move their lead to the "Lost" stage and add a system note.
        * Finally, it updates the database to mark the losing assignments as `Lost`.
* **Outcome:** A "Win" by one agent results in a clean and immediate end to the competition for all others. A "Surrender" by one agent results in the lead being immediately put back into play.
