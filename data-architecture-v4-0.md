# Project Phoenix: Data Architecture Document (v4.0)
* **Date:** October 20, 2025
* **Status:** Final - Aligned with Blueprint v8.0 & Quota Logic
* **Purpose:** This document is the single source of truth for all data structures within the Project Phoenix ecosystem. It provides the definitive schema for the GoHighLevel (GHL) CRM frontend and the PostgreSQL Master Database backend. This is a technical reference for system administrators and developers.

## Part 1: GoHighLevel (GHL) Data Structure

This section outlines the agent-facing data structures that must be configured within each GHL sub-account. Consistency across all sub-accounts is critical.

### 1.1 Sales Pipeline & Stages
The pipeline tracks the lead's journey. Standardized fast timers (24h/3d) apply to all active assignments.

| Stage # | Stage Name | Purpose & Key Actions | Timer Logic |
| :--- | :--- | :--- | :--- |
| 1 | **New Lead** | Holding pen for all freshly assigned leads. | **24-Hour Timer STARTS.** |
| 2 | **Contacting** | Agent is actively attempting first contact. | **24-Hour Timer CONTINUES.** |
| 3 | **Qualifying** | Active financial assessment, needs analysis. | **3-Day Timer STARTS.** |
| 4 | **Pending** | Lead has gone silent or is unresponsive. | **3-Day Timer CONTINUES.** |
| 5 | **Incomplete** | Lead is providing documents, but they are incomplete. | **3-Day Timer CONTINUES.** |
| 6 | **Joint-Loan** | A joint-loan application is being explored. | **3-Day Timer CONTINUES.** |
| 7 | **Pre-Booking** | **The "Win" stage.** Complete documents submitted to ERP. | **Timer STOPS.** |
| 8 | **Confirmed Booking**| ERP system has confirmed the booking. | **Timer STOPS.** |
| 9 | **Won - Confirmed Sales**| Final stage for a successful deal. | **Timer STOPS.** |
| 10 | **Lost** | Agent has manually lost the lead, or the system has closed it. | **Timer STOPS.** |
| 11 | **Unqualified** | Lead is unqualified but may become qualified later. | **Timer STOPS.** |

### 1.2 GHL Custom Fields
These fields must be created within the GHL settings (**Contact** model).

**Group 1: Property Interest**
*Captures what the client is looking for.*

| Field Name | GHL Field Type | Notes | Field ID (Example Only) |
| :--- | :--- | :--- | :--- |
| Project Site | Dropdown | Specific housing project | `lFDEtHTjkX9NlyJfnzRq` |
| Project State | Dropdown | State where project is located | `kuCEEVcD2nZuOqjbREGG` |
| House Type | Dropdown | e.g., Terrace, Condo, Semi-D | `y1QlfcfYVTavYJWuDFkP` |
| Budget Range | Dropdown | e.g., < 300k, 300k-500k | `iXhlXLFOPp85taLZ1DDL` |
| Bumiputera Status | Dropdown (Yes/No) | For Bumi lot eligibility | `qdF4phfgUnIBcUI3nwmU` |
| Home Purchase Number | Dropdown | `First`, `Second`, `Third or more` | `J5TvCbkkThTjv9kErTsJ` |

**Group 2: Financial Qualification**
*Captures the client's financial ability.*

| Field Name | GHL Field Type | Notes | Field ID (Example Only) |
| :--- | :--- | :--- | :--- |
| Employment Sector | Dropdown | e.g., Gov, Private, Self-Employed | `0vQmQT9CKbQIWNnPHV7j` |
| Job Title | Text | Lead's profession | `1FXYLCZ5cvXQzUYBvFHe` |
| Monthly Gross Income | Number (Currency) | Personal income | `U25dP1MxnELcao35H0Bv` |
| Household Gross Income | Number (Currency) | Combined income for loan | `1ERR8m8o7FEzgD6WxNxz` |
| Loan Pre-Qualification | Dropdown | `Not Attempted`, `Pending`, `Qualified`, `Not Qualified` | `Eh5eYTRjsPP4DhbssbsM` |
| CTOS/CCRIS Status | Dropdown | `No Issues`, `Minor Issues`, `Major Issues`, `Unknown` | `IZioHDQ6Vb9KXEe3GCb7` |
| KWSP Status | Dropdown | `Eligible`, `Not Eligible`, `To Be Checked` | `xdxrGV6hZfFS0cp09mjX` |

**Group 3: Lead Metadata**
*System and source information.*

| Field Name | GHL Field Type | Notes | Field ID (Example Only) |
| :--- | :--- | :--- | :--- |
| Lead Source | Text | Original source (e.g., Meta Ads) | `a9qxRuOMrLjSVZYtqLJP` |
| City | Text | City of the lead or project | *(Need ID if created)* |
| Income Statement | Dropdown | Yes/No | `zgwduv3VPhSEBRli7jjw` |

*(Note: Actual Field IDs must be retrieved via API/URL for each sub-account and stored in the `ghl_configs` table).*

---
## Part 2: PostgreSQL Master Database Schema

This defines the structure of the central database.

### 2.1 Table: `master_contacts`
The master record for every unique person.

| Column Name | Data Type | Description / Purpose |
| :--- | :--- | :--- |
| `contact_id` | `SERIAL PRIMARY KEY` | Internal unique ID for the person (PK). |
| `phone` | `VARCHAR(25) UNIQUE NOT NULL` | Master identifier. |
| `email` | `VARCHAR(255)` | Lead's email. |
| `first_name` | `VARCHAR(100)` | Lead's first name. |
| `last_name` | `VARCHAR(100)` | Lead's last name. |
| `lead_type` | `VARCHAR(50) NOT NULL` | Current state: `Fresh` or `Recycle`. |
| `priority_level` | `INT` | Global dispatch priority (1-3) for Recycle leads. NULL if Fresh or Active/Locally Recycling. |
| `needs_local_recycle` | `BOOLEAN DEFAULT FALSE` | Flag indicating if a Fresh lead needs local reassignment. |
| `original_location_id` | `VARCHAR(100)` | Stores the GHL Location ID where a Fresh lead originated. Used for local recycle. NULL otherwise. |
| `lead_source` | `VARCHAR(100)` | Original source. |
| `project_site` | `VARCHAR(100)` | Maps to GHL custom field. |
| `project_state` | `VARCHAR(50)` | Maps to GHL custom field. |
| `house_type` | `VARCHAR(100)` | Maps to GHL custom field. |
| `budget_range` | `VARCHAR(50)` | Maps to GHL custom field. |
| `bumiputera_status` | `BOOLEAN` | Maps to GHL custom field. |
| `home_purchase_number` | `VARCHAR(50)` | Maps to GHL custom field. |
| `employment_sector` | `VARCHAR(50)` | Maps to GHL custom field. |
| `job_title` | `VARCHAR(100)` | Maps to GHL custom field. |
| `monthly_income` | `DECIMAL(10, 2)` | Maps to GHL custom field. |
| `household_income` | `DECIMAL(10, 2)` | Maps to GHL custom field. |
| `loan_prequalification` | `VARCHAR(50)` | Maps to GHL custom field. |
| `ctos_status` | `VARCHAR(50)` | Maps to GHL custom field. |
| `kwsp_status` | `VARCHAR(50)` | Maps to GHL custom field. |
| `city` | `VARCHAR(100)` | Maps to GHL custom field. |
| `income_statement` | `VARCHAR(50)` | Maps to GHL custom field. |
| `notes_history` | `TEXT` | Compiled, anonymized notes history. |
| `created_at` | `TIMESTAMPTZ DEFAULT NOW()` | Timestamp of initial system entry. |

### 2.2 Table: `agents`
The roster of all agents eligible to receive leads.

| Column Name | Data Type | Description / Purpose |
| :--- | :--- | :--- |
| `agent_id` | `SERIAL PRIMARY KEY` | Internal unique ID for the agent (PK). |
| `agent_name` | `VARCHAR(100) NOT NULL` | Agent's full name. |
| `ghl_user_id` | `VARCHAR(100) NOT NULL` | Agent's User ID from GHL API. |
| `ghl_location_id` | `VARCHAR(100) NOT NULL` | The GHL Location/Sub-Account ID the agent belongs to. **(FK Link to `ghl_configs`)** |
| `team_id` | `VARCHAR(50)` | Internal team identifier (e.g., 'Team A-1'). Used by `routing_rules`. |
| `is_active` | `BOOLEAN DEFAULT TRUE` | Controls lead distribution eligibility. |

### 2.3 Table: `routing_rules`
Defines logic for assigning leads to specific teams based on lead attributes.

| Column Name | Data Type | Description / Purpose |
| :--- | :--- | :--- |
| `rule_id` | `SERIAL PRIMARY KEY` | Unique ID for the rule (PK). |
| `team_id` | `VARCHAR(50) NOT NULL` | The internal `team_id` the rule applies to. |
| `rule_type` | `VARCHAR(50) NOT NULL` | The `master_contacts` column to filter on (e.g., `project_state`). |
| `rule_value` | `VARCHAR(100) NOT NULL` | The value to match (e.g., `Johor`). |

### 2.4 Table: `assignments`
The referee's scorecard, tracking every assignment instance.

| Column Name | Data Type | Description / Purpose |
| :--- | :--- | :--- |
| `assignment_id` | `SERIAL PRIMARY KEY` | Unique ID for this assignment instance (PK). |
| `contact_id` | `INT NOT NULL REFERENCES master_contacts(contact_id)` | Link to the person (FK). |
| `agent_id` | `INT NOT NULL REFERENCES agents(agent_id)` | Link to the assigned agent (FK). |
| `status` | `VARCHAR(50) NOT NULL` | Outcome: `Active`, `Won`, `Lost`. |
| `current_stage` | `VARCHAR(100)` | GHL pipeline stage name at last update/assignment. |
| `ghl_contact_id` | `VARCHAR(100) NOT NULL` | The GHL Contact ID within the assigned sub-account. |
| `timer_expires_at` | `TIMESTAMPTZ` | Deadline calculated based on assignment time and stage rules (24h/3d). |
| `assigned_at` | `TIMESTAMPTZ DEFAULT NOW()` | Timestamp of assignment creation. |
| `closed_at` | `TIMESTAMPTZ` | Timestamp when status changed to `Won` or `Lost`. |

### 2.5 Table: `ghl_configs`
The central dictionary storing configuration specific to each GHL Location/Sub-Account.

| Column Name | Data Type | Description / Purpose |
| :--- | :--- | :--- |
| `location_id` | `VARCHAR(100) PRIMARY KEY` | The GHL Location/Sub-Account ID (PK). |
| `auth_header_value` | `TEXT` | The full `Bearer [V2_Token]` string for API authentication. |
| `cf_project_state_id` | `VARCHAR(100)` | GHL Custom Field ID for "Project State". |
| `cf_city_id` | `VARCHAR(100)` | GHL Custom Field ID for "City". |
| `cf_home_purchase_number_id` | `VARCHAR(100)` | GHL Custom Field ID for "Home Purchase Number". |
| `cf_kwsp_status_id` | `VARCHAR(100)` | GHL Custom Field ID for "KWSP Status". |
| `pipeline_id` | `VARCHAR(100)` | GHL Pipeline ID for the main sales pipeline. |
| `stage_new_lead_id` | `VARCHAR(100)` | GHL Pipeline Stage ID for "New Lead". |
| `stage_lost_id` | `VARCHAR(100)` | GHL Pipeline Stage ID for "Lost". |
| *(Add columns for other required Custom Field/Stage IDs)* | ... | ... |

### **2.6 Table: `daily_quota_tracker` (NEW)**
*Tracks the number of leads assigned to each agent per day by Workflow B.*

| Column Name | Data Type | Description / Purpose |
| :--- | :--- | :--- |
| `agent_id` | `INT NOT NULL` | **FK** referencing `agents.agent_id`. Part of the Composite PK. |
| `assignment_date` | `DATE NOT NULL DEFAULT CURRENT_DATE` | The date the assignments occurred. Part of the Composite PK. |
| `assigned_count` | `INT NOT NULL DEFAULT 0` | The number of leads assigned to this agent on this date. |
| *Constraint* | `PRIMARY KEY (agent_id, assignment_date)` | Ensures only one row per agent per day. |

### 2.7 (Optional) Table: `system_settings`
A table to store global system parameters.

| Column Name | Data Type | Description / Purpose |
| :--- | :--- | :--- |
| `setting_key` | `VARCHAR(50) PRIMARY KEY` | Name of the setting (e.g., `daily_quota`). |
| `setting_value` | `VARCHAR(255)` | Value of the setting (e.g., `30`). |

---
## Part 3: Data Integrity & Synchronization Rules

* **Master Source:** PostgreSQL is the absolute source of truth.
* **GHL De-duplication:** GHL's native "prevent duplicate contact" feature must be enabled.
* **Handling Blanks:** NULL values from GHL are acceptable and stored as NULL in PostgreSQL.
* **Quota Tracking:** The `daily_quota_tracker` table is updated exclusively by the n8n "Morning Dispatch" workflow.
