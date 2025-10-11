# Project Phoenix: Lead Lifecycle & Flow Cycle (v1.0)
* **Date:** October 12, 2025
* **Purpose:** This document provides a detailed, step-by-step explanation of a lead's journey through the Project Phoenix system, from creation to its final outcome.

## Introduction: The Life of a Lead in the Royal Rumble

Every lead that enters this system is on a high-speed journey. It will be presented to multiple agents over its lifetime, with the system acting as an intelligent and relentless referee, always pushing for a conclusion. The goal is constant momentum. A lead is never allowed to go "stale." This document breaks down that journey.

---

## 1. The Spark: New Lead Ingestion (Workflow A)

This is the moment a fresh, high-value lead is born.

**Trigger:** A real-time webhook is received by n8n from a lead source (e.g., Facebook Lead Ads, Website Form).

**The Play-by-Play:**
1.  **n8n Catches the Lead:** The workflow instantly captures the lead's data (name, phone, email, source).
2.  **Database Logging (The Master Record):**
    * n8n's first action is to connect to the PostgreSQL database.
    * It creates a new entry in the `master_contacts` table. This is the lead's permanent, unique record, identified by their phone number.
3.  **Find the Next Champion (Agent Selection):**
    * n8n queries the `agents` table and the `routing_rules` table to determine the correct team for this lead.
    * It runs its round-robin logic for that team, checking each agent's capacity (is `Stage 1 + Stage 2 < 60`?).
    * It explicitly excludes any agent who has been assigned this lead before by checking the `assignments` table.
4.  **The "Check-Out" (Push to GHL):**
    * Once a valid agent is selected, n8n uses the GHL API to perform a "Create/Update" action in that agent's sub-account.
    * GHL's de-duplication logic ensures that if the contact already exists, it's reactivated and moved to Stage 1. If not, it's created.
5.  **Finalizing the Assignment (Updating the Referee's Scorecard):**
    * n8n receives the `ghl_contact_id` back from the GHL API.
    * It creates a new row in the `assignments` table in PostgreSQL.

**Resulting Database State:**
* `master_contacts`: Contains the permanent record for "John Doe".
* `assignments`: A new row exists showing John Doe is assigned to Agent A.
    * `contact_id`: (links to John Doe)
    * `agent_id`: (links to Agent A)
    * `status`: 'Active'
    * `current_stage`: 'New Lead'
    * `timer_expires_at`: (Current Time + 48 hours)

---

## 2. The First Round: Exclusive Window (Stages 1 & 2)

The agent now has an exclusive, 48-hour window to make progress.

* **Agent's Goal:** Move the lead from `New Lead` -> `Contacting` -> `Qualifying` within 48 hours.
* **The Clock is Ticking:** The `timer_expires_at` value in the `assignments` table is the hard deadline.

### **Scenario A: The Agent Succeeds!**
The agent moves the lead to Stage 3 (Qualifying). The n8n Referee (Workflow C) detects this stage change via webhook and updates the `assignments` table, setting a new 10-day timer. The agent has successfully passed the first round!

### **Scenario B: The 48-Hour Timer Expires...**
The n8n "Timer Janitor" (Workflow D) runs and finds the expired assignment.

**The Play-by-Play:**
1.  **Flag the Assignment:** The Janitor finds the assignment for John Doe & Agent A is overdue.
2.  **Update Master DB:** n8n updates the `master_contacts` record, setting its `priority_level` (e.g., Priority 3 if from New Lead, Priority 2 if from Contacting).
3.  **Cool Down Initiated:** The system now knows this lead is back in the pool for re-dispatch in the next Morning Dispatch (Workflow B). Simultaneously, the GHL automation for Agent A begins the 30-day "cool down" process that will eventually move their copy to `Lost`.

---

## 3. The Rumble Begins: Simultaneous Competition (Stages 3-6)

This is where the action heats up. A lead can be actively worked by multiple agents at once.

* **The 10-Day Progress Timer:** An agent has 10 days in these stages to advance the lead to Pre-Booking.
* **The Re-Dispatch Loop:** If the 10-day timer expires, the exact same process as above happens (`Scenario B`). The lead is put back into the pool with a **Priority 1 (Hottest)** rating.
* **The Result:** Agent A could still have the lead in their "Qualifying" stage, while Agent B receives the same lead as a "New Lead." The race is on.

---

## 4. The Finish Line: The "Win" Condition 🏆 (Stage 7)

This is the championship moment.

**Trigger:** Agent B successfully moves the lead to **Stage 7: Pre-Booking**.

**The Referee's Actions (Workflow C - Real-Time):**
1.  **Webhook Received:** n8n gets a real-time notification: "John Doe moved to Pre-Booking in Agent B's account."
2.  **Identify the Person:** n8n looks up John Doe's permanent record in `master_contacts`.
3.  **Find All Competitors:** n8n queries the `assignments` table for ALL `Active` assignments for John Doe. It finds Agent A and Agent B.
4.  **Declare the Winner:**
    * It updates Agent B's assignment row: `status` -> `'Won'`.
    * It updates Agent A's assignment row: `status` -> `'Lost'`.
5.  **End the Race in GHL:** n8n immediately sends an API command to **Agent A's GHL account**.
    * The command finds the contact for John Doe.
    * It moves the contact to **Stage 10: Lost**.
    * It adds a system note: *"Opportunity closed by another representative."*

The competition for this lead is now officially over.

---

## 5. The End of the Road: Agent Surrender (Manual "Lost")

An agent can manually give up on a lead at any time.

**Trigger:** Agent A moves a lead to **Stage 10: Lost**.

**The Referee's Actions (Workflow C - Real-Time):**
1.  **Webhook Received:** n8n gets a real-time notification: "John Doe moved to Lost in Agent A's account."
2.  **Update the Scorecard:** n8n finds the active assignment for Agent A and updates its `status` to `'Lost'`.
3.  **Immediate Re-circulation:** n8n updates the `master_contacts` record for John Doe, setting its `priority_level` and making it immediately available for the next distribution cycle.

This ensures that when an agent gives up, the lead's velocity is maintained, and it gets back into the rumble as quickly as possible.
