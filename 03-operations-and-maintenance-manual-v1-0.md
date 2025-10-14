# Project Phoenix: Operations & Maintenance Manual (v1.0)
* **Date:** October 12, 2025
* **Purpose:** This document provides the standard operating procedures (SOPs) for managing, monitoring, and troubleshooting the Project Phoenix system. It is the go-to guide for system administrators.

## Part 1: System Health Monitoring (The Control Tower 📡)

You need to know if the system is running smoothly. Here’s how you check the vital signs.

### **1.1 The n8n Executions List**
This is your primary health dashboard.
* **Location:** In your n8n dashboard, click on "Executions."
* **What to Look For:**
    * ✅ **Success:** A long list of green "Succeeded" executions is a healthy sign. It means leads are being processed correctly.
    * ❌ **Error:** A red "Failed" execution is an immediate call to action. Click on it to see exactly which step failed and why.

### **1.2 Automated Error Notifications (The Alarm System)**
We will build a dedicated "Error Workflow" in n8n.
* **How it Works:** This special workflow is configured to automatically run whenever *any other workflow fails*.
* **Action:** Its only job is to send an immediate, detailed notification email to the system administrator.
* **Benefit:** You don't have to constantly watch the dashboard. The system will tell you the moment something goes wrong.

---
## Part 2: Common Administrative Tasks (The Playbooks 📘)

These are your step-by-step guides for the most common administrative tasks.

### **Playbook 1: Onboarding a New Agent**
1.  **In GHL:** Create the new user in the appropriate sub-account. Navigate to their profile and copy their **User ID** from the URL bar.
2.  **In PostgreSQL:** Connect to the `phoenix_db` with your admin tool (e.g., DBeaver).
3.  **Execute SQL Command:** Run the following command, replacing the placeholder values. This officially adds the agent to the distribution roster.
    ```sql
    INSERT INTO agents (agent_name, ghl_user_id, ghl_location_id, team_id, is_active) 
    VALUES ('New Agent Name', 'GHL_USER_ID_HERE', 'GHL_SUB_ACCOUNT_ID_HERE', 'Team-C', TRUE);
    ```

### **Playbook 2: Adding a New Routing Rule**
When you start a new project for a specific location (e.g., "Klang Valley Project" for Team D).
1.  **In PostgreSQL:** Connect to the `phoenix_db`.
2.  **Execute SQL Command:** Run the following command to teach the system the new rule.
    ```sql
    INSERT INTO routing_rules (team_id, rule_type, rule_value) 
    VALUES ('Team-D', 'project_state', 'Klang Valley');
    ```

### **Playbook 3: Adding a New Team / Sub-Account**
1.  **In GHL:** Configure the new sub-account (pipeline, fields, etc.) and get its **API Key**.
2.  **In n8n:** Create a new "Header Auth" credential for the new team's API key.
3.  **In n8n:** Update the relevant workflows to include the new team's credentials in their logic loops.
4.  **In PostgreSQL:** Add the new agents to the `agents` table using Playbook 1.

---
## Part 3: Troubleshooting Guide (First Aid Kit 🚑)

| Problem | Possible Cause | What to Check First |
| :--- | :--- | :--- |
| **Fresh Leads are not being registered.** | GHL webhook is misconfigured or failing. | 1. Check the GHL workflow for the "Fresh Lead Capture" trigger. 2. Verify the webhook URL is correct. 3. Check the "Real-Time Registrar" workflow in n8n for any failed executions. |
| **Recycled Leads were not distributed.** | The scheduled n8n workflow failed. | 1. Check the "Morning Dispatch" workflow execution log in n8n for errors. 2. Verify the database connection from n8n is active. |
| **An agent "won" but other leads weren't lost.**| The "Referee" workflow failed for a specific agent. | 1. Check the "Real-Time Referee" workflow execution log. 2. The most likely cause is an incorrect or expired API key for one of the "losing" agents' sub-accounts. |

---
## Part 4: Backup & Recovery (The Insurance Policy 🛡️)

* **Database Backups (Critical):**
    * Use your hosting provider's (Hostinger's) built-in database management tools to schedule **automated daily backups** of the `phoenix_db`.
    * Set a retention policy of at least **14 days**.
* **n8n Workflow Backups:**
    * After making any significant changes to a workflow, it is a best practice to download the workflow's JSON file from the n8n UI.
    * To do this, open the workflow, click the three dots (`...`) in the top right, and select "Download." Store these JSON files in a secure location.
