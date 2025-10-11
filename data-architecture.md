# Project Phoenix: Data Architecture Document (v2.0)
* **Date:** October 12, 2025
* **Status:** Final - Ready for Implementation
* **Purpose:** This document is the single source of truth for all data structures within the Project Phoenix ecosystem. It provides the definitive schema for the GoHighLevel (GHL) CRM frontend and the PostgreSQL Master Database backend. This is a technical reference for system administrators and developers.

## Part 1: GoHighLevel (GHL) Data Structure

This section outlines the agent-facing data structures that must be configured within each GHL sub-account. Consistency across all sub-accounts is critical for the system to function correctly.

### 1.1 Sales Pipeline & Stages
The pipeline is designed to provide a granular view of the lead's journey.

| Stage # | Stage Name | Purpose & Key Actions |
| :--- | :--- | :--- |
| 1 | **New Lead** | Holding pen for all freshly assigned leads. The 48-hour "Exclusive Window" timer starts here. |
| 2 | **Contacting** | Agent is actively attempting first contact (call, WhatsApp). The 48-hour timer continues. |
| 3 | **Qualifying** | Active financial assessment, needs analysis. The 10-day "Progress" timer starts here. |
| 4 | **Pending** | Lead has gone silent or is unresponsive after initial contact. 10-day "Progress" timer is active. |
| 5 | **Incomplete** | Lead is providing documents, but they are incomplete. 10-day "Progress" timer is active. |
| 6 | **Joint-Loan** | A joint-loan application is being explored. 10-day "Progress" timer is active. |
| 7 | **Pre-Booking** | **The "Win" stage.** Complete documents have been collected and submitted to the external ERP system. |
| 8 | **Confirmed Booking**| The ERP system has confirmed the booking is valid. |
| 9 | **Won - Confirmed Sales**| The final stage for a successful deal (e.g., SPA signed). |
| 10 | **Lost** | Agent has manually lost the lead, or the system has automatically closed the opportunity. |
| 11 | **Unqualified** | Lead is deemed unqualified but may become qualified in the future. |

### 1.2 GHL Custom Fields
These fields must be created within the GHL settings to capture all essential lead information.

**Group 1: Property Interest**
*This group captures what the client is looking for.*

| Field Name | GHL Field Type | Purpose & Notes |
| :--- | :--- | :--- |
| **Project Site** | Dropdown | The specific housing project the lead is interested in. |
| **Project State** | Dropdown | The state where the project is located (e.g., Johor, Perak). |
| **House Type** | Dropdown | e.g., Single-Storey Terrace, Condominium, Semi-D. |
| **Budget Range** | Dropdown | e.g., < 300k, 300k-500k, 500k-700k, > 700k. |
| **Bumiputera Status** | Dropdown (Yes/No) | Critical for Malaysian property sales to identify eligibility for Bumi lots/discounts. |

**Group 2: Financial Qualification**
*This group captures the client's financial ability to purchase.*

| Field Name | GHL Field Type | Purpose & Notes |
| :--- | :--- | :--- |
| **Employment Sector** | Dropdown | e.g., Government, Private Sector, Self-Employed. Important for loan risk assessment. |
| **Job Title** | Text | The lead's job title or profession. |
| **Monthly Gross Income** | Number (Currency) | The lead's personal monthly income before deductions. |
| **Household Gross Income**| Number (Currency) | The combined income used for the loan application (e.g., with a spouse). |
| **Loan Pre-Qualification** | Dropdown | Options: `Not Attempted`, `Pending`, `Qualified`, `Not Qualified`. |
| **CTOS/CCRIS Status** | Dropdown | Options: `No Issues`, `Minor Issues`, `Major Issues`, `Unknown`. |

---
## Part 2: PostgreSQL Master Database Schema

This is the definitive backend structure. It is the system's brain and permanent memory.

### 2.1 Table: `master_contacts`
The master record of every unique person who has ever entered the system. One person, one record.

| Column Name | Data Type | Description / Purpose |
| :--- | :--- | :--- |
| `contact_id` | `SERIAL PRIMARY KEY` | Unique, auto-incrementing ID for each person. This is the **Primary Key (PK)**. |
| `phone` | `VARCHAR(25) UNIQUE`| The lead's primary phone number. Must be unique, as it's the master identifier. |
| `email` | `VARCHAR(255)` | The lead's email address. Can be NULL. |
| `first_name` | `VARCHAR(100)` | Lead's first name. |
| `last_name` | `VARCHAR(100)` | Lead's last name. |
| `lead_type` | `VARCHAR(50)` | Current master type: `Fresh` or `Recycle`. |
| `priority_level`| `INT` | Priority for re-dispatch: `1` (Hottest) to `3` (Coldest). Can be NULL. |
| `lead_source` | `VARCHAR(100)` | Original source of the lead (e.g., 'Facebook Ads', 'Legacy Import'). |
| `project_site` | `VARCHAR(100)`| Maps to GHL custom field. Can be NULL. |
| `project_state` | `VARCHAR(50)` | Maps to GHL custom field. Can be NULL. |
| `house_type` | `VARCHAR(100)`| Maps to GHL custom field. Can be NULL. |
| `budget_range` | `VARCHAR(50)` | Maps to GHL custom field. Can be NULL. |
| `bumiputera_status`| `BOOLEAN` | Maps to GHL custom field. Can be NULL. |
| `employment_sector`| `VARCHAR(50)` | Maps to GHL custom field. Can be NULL. |
| `job_title` | `VARCHAR(100)`| Maps to GHL custom field. Can be NULL. |
| `monthly_income` | `DECIMAL(10, 2)`| Maps to GHL custom field. Can be NULL. |
| `household_income` | `DECIMAL(10, 2)`| Maps to GHL custom field. Can be NULL. |
| `loan_prequalification`| `VARCHAR(50)` | Maps to GHL custom field. Can be NULL. |
| `ctos_status` | `VARCHAR(50)` | Maps to GHL custom field. Can be NULL. |
| `notes_history` | `TEXT` | A compiled, anonymized history of all notes from previous assignments. |
| `created_at` | `TIMESTAMPTZ` | Timestamp of when the contact was first created in our system. |

### 2.2 Table: `agents`
The complete roster of all agents eligible to receive leads.

| Column Name | Data Type | Description / Purpose |
| :--- | :--- | :--- |
| `agent_id` | `SERIAL PRIMARY KEY`| Unique, auto-incrementing ID for each agent. This is the **PK**. |
| `agent_name` | `VARCHAR(100)` | Full name of the agent. |
| `ghl_user_id` | `VARCHAR(100)` | The agent's User ID from the GHL API. **Crucial for assignment.** |
| `ghl_location_id`| `VARCHAR(100)` | The ID of the GHL sub-account this agent belongs to. |
| `team_id` | `VARCHAR(50)` | The team the agent belongs to (e.g., 'Team A', 'Team B-1'). |
| `is_active` | `BOOLEAN` | Controls if an agent can receive leads. `FALSE` for vacation, etc. |

### 2.3 Table: `routing_rules`
Defines the business logic for distributing leads to specific teams.

| Column Name | Data Type | Description / Purpose |
| :--- | :--- | :--- |
| `rule_id` | `SERIAL PRIMARY KEY` | Unique ID for each rule. This is the **PK**. |
| `team_id` | `VARCHAR(50)` | The team the rule applies to (e.g., 'Team B-1'). |
| `rule_type` | `VARCHAR(50)` | The lead attribute to filter on (e.g., `project_state`). |
| `rule_value` | `VARCHAR(100)`| The value to match (e.g., `Johor`). |

### 2.4 Table: `assignments`
The referee's scorecard. This is the most dynamic table, tracking every single assignment of a contact to an agent.

| Column Name | Data Type | Description / Purpose |
| :--- | :--- | :--- |
| `assignment_id` | `SERIAL PRIMARY KEY`| Unique ID for this specific assignment. This is the **PK**. |
| `contact_id` | `INT` | **Foreign Key (FK)** referencing `master_contacts.contact_id`. |
| `agent_id` | `INT` | **FK** referencing `agents.agent_id`. |
| `status` | `VARCHAR(50)` | The outcome of this assignment: `Active`, `Won`, `Lost`. |
| `current_stage` | `VARCHAR(100)`| The current pipeline stage name in GHL (e.g., 'Qualifying'). |
| `ghl_contact_id` | `VARCHAR(100)` | The unique contact ID from the GHL API for *this specific assignment*. |
| `timer_expires_at`| `TIMESTAMPTZ` | The deadline for the agent to take action. Can be NULL. |
| `assigned_at` | `TIMESTAMPTZ` | The exact timestamp of when this assignment was made. |
| `closed_at` | `TIMESTAMPTZ` | Timestamp of when the status changed to `Won` or `Lost`. Can be NULL. |

---
## Part 3: Data Integrity & Synchronization Rules

* **Handling of Blank Fields:** The system is built to be robust. A blank or empty custom field in GHL will be stored as a `NULL` value in the corresponding PostgreSQL column. This ensures that data synchronization will not fail due to incomplete agent input
