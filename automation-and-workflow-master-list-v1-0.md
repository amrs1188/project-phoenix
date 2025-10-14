# Project Phoenix: Automation & Workflow Master List (v1.0)
* **Date:** October 14, 2025
* **Purpose:** This document provides a comprehensive summary and description of every automated workflow required to run the Project Phoenix system, separated by platform (GHL and n8n).

## Part 1: GHL Internal Automations (The Arena's Local Rules 🏟️)

These workflows must be created **inside each** GHL sub-account. They handle tasks that are specific to that team's environment.

### **1.1. Workflow: Fresh Lead Capture & Notify**
* **Name:** `Fresh Lead Capture & Notify`
* **Trigger:** `Facebook Lead Form Submitted` (or other ad platform triggers).
* **Description:** This is the "Welcome Committee" for brand-new leads. Its job is to:
    1.  **Tag the Contact:** Applies the `lead_type:fresh` tag.
    2.  **Set the Source:** Updates the `Lead Source` custom field (e.g., "Meta Ads - Project X").
    3.  **Initial Assignment:** Assigns the lead to an agent on the **local team** using GHL's internal round-robin feature for maximum speed.
    4.  **Notify the Referee:** Its final, critical action is to call a **Custom Webhook**, sending the new lead's complete data to the n8n "Real-Time Registrar" workflow to be logged in the Master DB.

### **1.2. Workflow Series: 30-Day Cool Down**
* **Name Example:** `Cool Down: Qualifying to Pending`
* **Trigger:** `Pipeline Stage Changed` (e.g., To "Qualifying").
* **Description:** This is a **series of "Housekeeper" workflows** that prevent leads from stalling indefinitely in an agent's pipeline. For example:
    * A workflow triggers when a lead enters "Qualifying." It waits 30 days.
    * After 30 days, it checks if the lead is *still* in "Qualifying."
    * If yes, it automatically moves the lead to the "Pending" stage and adds a system note.
    * This pattern is replicated for other stages (e.g., a separate workflow for `Pending` -> `Lost`).

### **1.3. Workflow: Notify n8n Referee on Stage Change**
* **Name:** `Notify n8n Referee on Stage Change`
* **Trigger:** `Pipeline Stage Changed` (filtered for the main sales pipeline).
* **Description:** This is a simple but vital workflow. Its **only job** is to act as the real-time "eyes and ears" for our n8n Referee.
    1.  It triggers every time a lead is moved from one stage to another.
    2.  Its only action is to `Execute Webhook`, sending a small data package with the contact ID and the new stage ID to our n8n "Real-Time Referee" workflow.

---
## Part 2: n8n Controller Workflows (The Referee's Brain 🧠)

These are the four core, powerful workflows that live in our self-hosted n8n instance. They control the global logic of the entire system.

### **2.1. Workflow A: Real-Time Registrar**
* **Trigger:** Webhook (listens for the call from GHL's "Fresh Lead Capture" workflow).
* **Description:** The system's official record-keeper for new leads. When it receives data from GHL, it:
    1.  Creates the permanent record for the person in the `master_contacts` table.
    2.  Creates the initial assignment record in the `assignments` table, linking the lead to the agent and setting the initial 48-hour timer.

### **2.2. Workflow D: Timer & Status Janitor**
* **Trigger:** Schedule (runs every hour).
* **Description:** The system's automated housekeeper. It ensures the "royal rumble" never stops.
    1.  It runs an SQL query to find all `assignments` where the timer has expired.
    2.  For each expired assignment, it updates the `master_contacts` record to set a `priority_level`, making that lead available for re-dispatch.
    3.  It then updates the original `assignments` record, changing its status to `Lost` to close it out.

### **2.3. Workflow B: The Morning Dispatch**
* **Trigger:** Schedule (runs daily at 12:10 AM).
* **Description:** The main distribution engine for all recycled leads.
    1.  It queries the database for all leads with a `priority_level` set, sorting them from hottest to coldest.
    2.  It reads the `routing_rules` to understand which teams get which leads.
    3.  It intelligently assigns leads one by one, checking each potential agent's capacity (the 60-lead cap) and their history (ensuring they don't get the same lead twice).
    4.  For each successful assignment, it creates the contact in the correct GHL sub-account and creates a new `assignments` record in the database.

### **2.4. Workflow C: The Real-Time Referee**
* **Trigger:** Webhook (listens for the call from GHL's "Notify n8n Referee" workflow).
* **Description:** The live-action umpire for the competition.
    1.  It receives the signal that a lead's stage has changed.
    2.  It uses a Switch node to determine if the change is significant:
        * **If "Surrender" (moved to `Lost`):** It updates the database to immediately put the lead back into the dispatch pool.
        * **If "Win" (moved to `Pre-Booking`):** It declares a winner in the database and then sends API commands to all other competing agents' GHL accounts to automatically move their version of the lead to "Lost" and add a system note.
