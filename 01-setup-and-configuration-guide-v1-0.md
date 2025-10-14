# Project Phoenix: Setup & Configuration Guide (v1.0)
* **Date:** October 12, 2025
* **Purpose:** This document provides the complete, step-by-step instructions for setting up the entire technical foundation for Project Phoenix, including the database, the n8n controller, and the GHL sub-accounts.

## Part 1: Prerequisites (The Shopping List 🛒)

Before you begin, ensure you have the following ready:

* **Server/VPS Access:** Full SSH access to your self-hosted server (e.g., from Hostinger) where n8n will be installed. A server with at least 2GB of RAM is recommended.
* **Domain/Subdomain:** A domain name that will point to your n8n instance (e.g., `n8n.yourcompany.com`). You will need to configure its DNS records to point to your server's IP address.
* **PostgreSQL Admin Tool:** A tool to manage your database. **DBeaver** (free) or **pgAdmin** are excellent choices.
* **GHL Agency Admin Access:** You need top-level administrative access to your GoHighLevel agency account to manage sub-accounts and get API keys.
* **Docker & Docker Compose:** These must be installed on your server. This is the modern standard for deploying applications like n8n.

---
## Part 2: PostgreSQL Database Setup (The Foundation 🏛️)

This is the central brain's memory. We will create the database and all the necessary tables.

1.  **Connect to your PostgreSQL instance** using your admin tool (DBeaver/pgAdmin).
2.  **Create the Database:** Execute the following SQL command to create a new database for our project.
    ```sql
    CREATE DATABASE phoenix_db;
    ```
3.  **Create a Dedicated User:** For security, we'll create a specific user that n8n will use to access the database. **Remember to replace `'YourStrongPasswordHere'` with a secure password.**
    ```sql
    CREATE USER phoenix_user WITH ENCRYPTED PASSWORD 'YourStrongPasswordHere';
    GRANT ALL PRIVILEGES ON DATABASE phoenix_db TO phoenix_user;
    ```
4.  **Create the Tables:** Connect to the new `phoenix_db` as the superuser and execute the entire SQL script below. This will create all four of our master tables with the correct columns and relationships.

    ```sql
    -- Create the Agents Table
    CREATE TABLE agents (
        agent_id SERIAL PRIMARY KEY,
        agent_name VARCHAR(100) NOT NULL,
        ghl_user_id VARCHAR(100) NOT NULL,
        ghl_location_id VARCHAR(100) NOT NULL,
        team_id VARCHAR(50),
        is_active BOOLEAN DEFAULT TRUE
    );

    -- Create the Master Contacts Table
    CREATE TABLE master_contacts (
        contact_id SERIAL PRIMARY KEY,
        phone VARCHAR(25) UNIQUE NOT NULL,
        email VARCHAR(255),
        first_name VARCHAR(100),
        last_name VARCHAR(100),
        lead_type VARCHAR(50) DEFAULT 'Recycle',
        priority_level INT,
        lead_source VARCHAR(100),
        project_site VARCHAR(100),
        project_state VARCHAR(50),
        house_type VARCHAR(100),
        budget_range VARCHAR(50),
        bumiputera_status BOOLEAN,
        employment_sector VARCHAR(50),
        job_title VARCHAR(100),
        monthly_income DECIMAL(10, 2),
        household_income DECIMAL(10, 2),
        loan_prequalification VARCHAR(50),
        ctos_status VARCHAR(50),
        notes_history TEXT,
        created_at TIMESTAMPTZ DEFAULT NOW()
    );

    -- Create the Routing Rules Table
    CREATE TABLE routing_rules (
        rule_id SERIAL PRIMARY KEY,
        team_id VARCHAR(50) NOT NULL,
        rule_type VARCHAR(50) NOT NULL,
        rule_value VARCHAR(100) NOT NULL
    );

    -- Create the Assignments Table (The Referee's Scorecard)
    CREATE TABLE assignments (
        assignment_id SERIAL PRIMARY KEY,
        contact_id INT NOT NULL REFERENCES master_contacts(contact_id),
        agent_id INT NOT NULL REFERENCES agents(agent_id),
        status VARCHAR(50) NOT NULL,
        current_stage VARCHAR(100),
        ghl_contact_id VARCHAR(100) NOT NULL,
        timer_expires_at TIMESTAMPTZ,
        assigned_at TIMESTAMPTZ DEFAULT NOW(),
        closed_at TIMESTAMPTZ
    );
    ```

---
## Part 3: n8n Controller Setup (The Brain Surgery 🧠)

We will use Docker to install and run n8n, as it's clean, reliable, and easy to manage.

1.  **SSH into your server.**
2.  **Create a directory** for your n8n project: `mkdir n8n-phoenix && cd n8n-phoenix`
3.  **Create a `docker-compose.yml` file:** Use a text editor like `nano` to create this file and paste the following content. This tells Docker how to run n8n.
    ```yaml
    version: '3.7'

    services:
      n8n:
        image: n8nio/n8n
        restart: always
        ports:
          - "127.0.0.1:5678:5678"
        environment:
          - DB_TYPE=postgresdb
          - DB_POSTGRESDB_HOST=${DB_HOST}
          - DB_POSTGRESDB_USER=${DB_USER}
          - DB_POSTGRESDB_PASSWORD=${DB_PASSWORD}
          - DB_POSTGRESDB_DATABASE=${DB_DATABASE}
          - DB_POSTGRESDB_PORT=5432
          - N8N_HOST=${SUBDOMAIN}
          - N8N_PROTOCOL=https
          - NODE_ENV=production
          - WEBHOOK_URL=https://${SUBDOMAIN}/
        volumes:
          - n8n_data:/home/node/.n8n

    volumes:
      n8n_data:
    ```
4.  **Create the `.env` file:** This is where you'll store your secrets. Create a file named `.env` in the same directory. **Fill in your actual values.**
    ```env
    # PostgreSQL Database Connection
    DB_HOST=your_database_host_or_ip
    DB_USER=phoenix_user
    DB_PASSWORD=YourStrongPasswordHere
    DB_DATABASE=phoenix_db

    # Subdomain for n8n
    SUBDOMAIN=n8n.yourcompany.com
    ```
5.  **Run n8n:** Execute the command `docker-compose up -d`. Docker will now download and start your n8n instance in the background! You can access it at `https://n8n.yourcompany.com`.

---
## Part 4: GHL Sub-Account Configuration (Setting up the Arena 🥊)

This checklist must be completed for **each and every GHL sub-account** that will participate in the system.

* **[ ] Generate API Key:**
    * Go to `Settings` -> `Company`.
    * Copy the **API Key** for the sub-account. You will need this to create credentials inside n8n later.
* **[ ] Create Custom Fields:**
    * Go to `Settings` -> `Custom Fields`.
    * Carefully create **every single custom field** listed in our `Data Architecture Document`. The names and types must match perfectly.
* **[ ] Build the Sales Pipeline:**
    * Go to `Settings` -> `Pipelines`.
    * Create a new pipeline and build the **exact 11 stages** defined in our blueprint.
* **[ ] Disable Duplicate Contacts:**
    * Go to `Settings` -> `Business Info`.
    * Ensure the toggle for **"Allow Duplicate Contact"** is turned **OFF**.
* **[ ] Gather Agent & Team Info:**
    * For each agent you plan to add to the system, you will need their GHL User ID and the Team they belong to. You will enter this data into the `agents` table in our database.

With this guide, the entire foundation of Project Phoenix can be built. You now have the database schema, the n8n controller ready to run, and a checklist for preparing the GHL arenas. The next step will be to create the **"API & Webhook Reference"** to define how these systems will talk to each other.
