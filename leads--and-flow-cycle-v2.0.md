# Project Phoenix: The Definitive Lead Lifecycle & Flow Cycle (v2.0)
* **Date:** October 12, 2025
* **Purpose:** This document provides a detailed, step-by-step explanation of a lead's complete journey through the Project Phoenix system, incorporating the "Simultaneous Competition" protocol and the decentralized Fresh Lead entry model.

## Introduction: The Two States of a Lead

A lead in Project Phoenix exists in one of two states:
* **Fresh Lead:** A brand-new lead that has just entered the ecosystem and is on its first-ever assignment.
* **Recycled Lead:** A lead that was once "Fresh" but failed to be moved to the "Pre-Booking" stage within its initial timeframes. Once a lead becomes "Recycled," it remains so for the rest of its lifecycle.

This document will trace the full path, from Fresh to Recycled, and through the competitive cycle.

---
## 1. Genesis: The Fresh Lead Flow (The Only Entry Point)
This is the moment a brand-new, high-value lead is born.

**Trigger:** A team leader's ad campaign (e.g., on Meta) generates a lead.

**The Play-by-Play:**
1.  **Capture (Inside GHL):** GHL's native integration **instantly** captures this lead and creates a new contact **directly inside the specific team's GHL sub-account.**
2.  **Internal GHL Automation:** A workflow inside that sub-account immediately triggers. It is responsible for:
    * Tagging the contact (`lead_type:fresh`).
    * Setting the `lead_source` custom field.
    * Assigning the lead to an agent on that team using GHL's internal round-robin.
3.  **Notify the Referee (The Hand-off to n8n):** The **final step** of this GHL workflow is to call a webhook, sending the new contact's complete data to our n8n controller.
4.  **Master Registration (n8n's Job):** The n8n "Registrar" workflow receives the data. Its sole job is to:
    * Create the permanent record in the PostgreSQL `master_contacts` table.
    * Create the first entry in the `assignments` table, linking this lead to the agent and officially starting the **48-hour "Exclusive Window" timer.**

**Result:** The Fresh Lead is now live in the agent's GHL and is officially on the referee's radar.

---
## 2. The First Round: The Exclusive Window (Stages 1 & 2)

The agent now has a protected, 48-hour window to make progress.

* **Agent's Goal:** Move the lead from `New Lead` -> `Contacting` -> `Qualifying` within 48 hours.
* **The Clock is Ticking:** The `timer_expires_at` value in the `assignments` table is the hard deadline.

### **Scenario A: The Agent Succeeds!**
The agent moves the lead to Stage 3 (Qualifying). The n8n Referee detects this and updates the `assignments` table, setting a new 10-day "Progress" timer. The agent has successfully passed the first round and maintains their exclusive assignment.

### **Scenario B: The 48-Hour Timer Expires...**
The n8n "Janitor" workflow finds the expired assignment.

**The Play-by-Play:**
1.  **Lead Becomes "Recycled":** The Janitor updates the lead's permanent record in `master_contacts`, changing its `lead_type` from `Fresh` to **`Recycle`**.
2.  **Set Priority:** The lead's `priority_level` is set (e.g., Priority 3 if from New Lead).
3.  **Enter the Pool:** The lead is now officially in the master pool, ready to be distributed to a *new* agent in the next "Morning Dispatch."
4.  **Initiate "Cool Down":** For the original agent, their copy of the lead begins its 30-day "cool down" cycle inside their GHL, which will eventually move it to the `Lost` stage.

---
## 3. The Rumble Begins: The Recycled Lead Flow
This is the core competitive cycle of the system.

**Trigger:** The n8n "Morning Dispatch" workflow runs at 12:10 AM.

**The Play-by-Play:**
1.  **Prioritize:** The workflow queries the Master DB for all available recycled leads, sorting them by `priority_level` (Hottest first).
2.  **Distribute:** It assigns these leads one by one, respecting agent capacity (60 leads in Stage 1 & 2) and ensuring an agent never gets the same lead twice.
3.  **The Rumble:** Once a recycled lead is assigned, it follows the **"Simultaneous Competition"** rules:
    * **10-Day Progress Timer (Stages 3-6):** The assigned agent has a 10-day window to advance the lead.
    * **If Timer Expires:** The lead is returned to the Master DB pool to be given to yet another agent, while the current agent's copy remains in their pipeline (starting its own "cool down" cycle). This is how multiple agents can end up working the same lead at the same time.

---
## 4. The Finish Line: The "Win" Condition 🏆

This is the rule that ends the competition for a specific lead.

**Trigger:** Any agent, at any time, successfully moves a lead into **Stage 7: Pre-Booking**.

**The Referee's Actions (Real-Time):**
1.  **Webhook Received:** n8n gets an instant notification of the stage change.
2.  **Declare Winner:** n8n finds the winning agent's record in the `assignments` table and updates their `status` to **`Won`**.
3.  **End the Race for Others:** n8n finds all other `Active` assignments for this same lead in the database. For each one, it:
    * Updates their `status` in the database to **`Lost`**.
    * Sends an immediate API command to their GHL to **automatically move the contact to Stage 10: Lost** and add the note: *"Opportunity closed by another representative."*

The competition for this lead is now over.
