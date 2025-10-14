# Project Phoenix: Automation & Workflow Master List (v2.0)
* **Date:** October 15, 2025
* **Purpose:** This document provides a comprehensive summary and detailed description of every automated workflow required to run the Project Phoenix system, separated by platform (GHL and n8n).

## Part 1: GHL Internal Automations (The Arena's Local Rules 🏟️)

These workflows must be created **inside each** GHL sub-account. They handle tasks specific to that team's environment.

### **1.1. Workflow: Fresh Lead Capture & Notify**
* **Name:** `Fresh Lead Capture & Notify`
* **Trigger:** `Facebook Lead Form Submitted` (or other ad platform triggers).
* **Description:** The "Welcome Committee" for brand-new leads.
    1.  **Tags the Contact:** Applies the `lead_type:fresh` tag.
    2.  **Sets the Source:** Updates the `Lead Source` custom field.
    3.  **Initial Assignment:** Uses GHL's built-in `Assign to User` action (with round-robin) to give the lead to a local agent instantly.
    4.  **Notifies the Referee:** The final action is to call a **Custom Webhook**, sending a clean JSON package of the new lead's data to the n8n "Real-Time Registrar" workflow.

### **1.2. Workflow Series: 30-Day Cool Down**
* **Name Example:** `Cool Down: Qualifying to Pending`
* **Trigger:** `Pipeline Stage Changed` (e.g., To "Qualifying").
* **Description:** A **series of "Housekeeper" workflows** that prevent leads from stalling. For example, the "Qualifying" workflow:
    1.  Triggers when a lead enters the "Qualifying" stage.
    2.  It then enters a `Wait` step for 30 days.
    3.  After 30 days, an `If/Else` condition checks if the lead is *still* in the "Qualifying" stage.
    4.  If yes, it automatically moves the lead to the "Pending" stage and adds a system note.
    5.  This pattern is replicated for other cool-down transitions: `Contacting` -> `Pending`, `Pending` -> `Lost`, etc.

### **1.3. Workflow: Notify n8n Referee on Stage Change**
* **Name:** `Notify n8n Referee on Stage Change`
* **Trigger:** `Pipeline Stage Changed` (filtered for the "Project Phoenix Sales" pipeline).
* **Description:** The real-time "eyes and ears" for our n8n Referee.
    1.  It triggers every time a lead is moved from one stage to another.
    2.  Its only action is to use the standard `Execute Webhook` action to send a data package (containing the `contact_id`, `pipelineStageId_from`, `pipelineStageId_to`, etc.) to our n8n "Real-Time Referee" workflow.

---
## Part 2: n8n Controller Workflows (The Referee's Brain 🧠)

These are the four core, powerful workflows that live in our self-hosted n8n instance.

### **2.1. Workflow A: Real-Time Registrar**
* **Trigger:** Webhook (listens for the call from GHL's "Fresh Lead Capture" workflow).
* **Description:** The system's official record-keeper for new leads.
    1.  A **Webhook** node receives the new Fresh Lead data from GHL.
    2.  A **Postgres node (Insert)** creates the permanent record for the person in the `master_contacts` table, giving them a unique, internal `contact_id`.
    3.  A second **Postgres node (Insert)** creates the initial record in the `assignments` table, linking the `contact_id` to the `agent_id`, setting the `status` to `Active`, and calculating the initial 48-hour `timer_expires_at` value.

### **2.2. Workflow D: Timer & Status Janitor**
* **Trigger:** Schedule (runs every hour).
* **Description:** The system's automated housekeeper that ensures the "royal rumble" never stops.
    1.  A **Schedule** node triggers the workflow.
    2.  A **Postgres node (Execute Query)** runs a `SELECT` query to find all records in the `assignments` table where `status = 'Active'` and `timer_expires_at < NOW()`.
    3.  A second **Postgres node (Update)** updates the `master_contacts` table for each found lead, setting its `priority_level` (e.g., to `1`, `2`, or `3`).
    4.  A third **Postgres node (Update)** updates the original `assignments` record, changing its `status` to `Lost` and setting the `closed_at` timestamp.

### **2.3. Workflow B: The Morning Dispatch**
* **Trigger:** Schedule (runs daily at 12:10 AM).
* **Description:** The main distribution engine for all recycled leads. This is a multi-step, looping process for each lead.
    1.  A **Schedule** node triggers the workflow.
    2.  A **Postgres node (Execute Query)** gets the day's "to-do list": all leads from `master_contacts` with a `priority_level`, sorted from hottest to coldest.
    3.  For each lead in the list, the workflow then performs a series of lookups:
        * A **Postgres node (Select)** finds the correct `team_id` from the `routing_rules` table based on the lead's `project_state`.
        * Another **Postgres node (Select)** gets a full roster of all `Active` agents for that `team_id`.
    4.  The workflow then begins an **Intelligent Assignment Loop** for each agent on the roster, performing two checks:
        * **History Check:** A **Postgres node (Select)** checks the `assignments` table to see if the agent has ever had this lead before.
        * **Capacity Check:** An **HTTP Request node** makes an API call to GHL to check if the agent has fewer than 60 leads in their initial pipeline stages.
    5.  Once an agent passes all checks, the **Delivery Actions** trigger:
        * An **HTTP Request node** sends a command to the GHL API to create the contact in the winning agent's sub-account.
        * A final **Postgres node (Insert)** creates a new record in the `assignments` table for this new assignment.

### **2.4. Workflow C: The Real-Time Referee**
* **Trigger:** Webhook (listens for the call from GHL's "Notify n8n Referee" workflow).
* **Description:** The live-action umpire for the competition.
    1.  A **Webhook** node receives the signal that a lead's stage has changed.
    2.  A **Switch** node inspects the new stage name to decide which path to take.
    3.  **If "Surrender" (stage is `Lost`):** The workflow follows a simple path with a single **Postgres (Update)** node that sets the lead's `priority_level` in `master_contacts`, making it available for re-dispatch.
    4.  **If "Win" (stage is `Pre-Booking`):** The workflow follows a more complex path:
        * It updates the database to mark the winning assignment.
        * It queries the database to find all other active, competing assignments for that same person.
        * It then loops through these "losing" assignments, sending an **HTTP Request** to each respective GHL sub-account to move their lead to the "Lost" stage and add a system note.
        * Finally, it updates the database to mark the losing assignments as `Lost`.
