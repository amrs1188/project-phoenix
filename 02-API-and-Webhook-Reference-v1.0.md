# Project Phoenix: API & Webhook Reference (v1.0)
* **Date:** October 12, 2025
* **Purpose:** This document is the technical reference for all communication between the n8n controller and the GoHighLevel (GHL) sub-accounts. It details the required API calls, webhook triggers, and data structures.

## Part 1: Authentication (The Secret Handshake 🤝)

All communication with the GHL API is secured using an API key.

* **Method:** Bearer Token Authentication.
* **Key Location:** Each GHL sub-account has its own unique API Key, found in `Settings` > `Company`.
* **n8n Configuration:** Inside n8n, you will create a separate **"Header Auth" credential** for each GHL sub-account.
    * Credential Name: `GHL-Team-A`
    * Header Name: `Authorization`
    * Header Value: `Bearer [API_KEY_FOR_TEAM_A]`

This ensures that n8n can securely and independently communicate with every team's arena.

---
## Part 2: n8n -> GHL (Outbound API Calls 📤)

This section details the commands our n8n Referee will send **to** GHL.

### **2.1 Create / Update Contact**
This is the primary command for assigning a recycled lead to an agent. Because GHL's de-duplication is enabled, this single command handles both creating a new contact and reactivating an old one.

* **Endpoint:** `POST https://services.leadconnectorhq.com/contacts/`
* **Purpose:** To assign a lead to an agent in their GHL sub-account.
* **Example Payload (JSON):**
    ```json
    {
      "firstName": "John",
      "lastName": "Doe",
      "email": "john.doe@email.com",
      "phone": "+60123456789",
      "locationId": "AbcLocationIDForTeamB",
      "customFields": [
        {
          "id": "xYzA_CustomField_ID_xYz",
          "value": "Project Mega"
        }
      ],
      "tags": [
        "lead_type:recycle"
      ]
    }
    ```

### **2.2 Update Opportunity Stage**
This command is used by the Referee to move a "losing" competitor's lead to the "Lost" stage.

* **Endpoint:** `PUT https://services.leadconnectorhq.com/opportunities/{opportunityId}`
* **Purpose:** To officially end the race for a non-winning agent.
* **Example Payload (JSON):**
    ```json
    {
      "pipelineStageId": "LostStageIDHere",
      "status": "lost"
    }
    ```

### **2.3 Add a Note to a Contact**
Used to add the system-generated note when a lead is lost due to another agent winning.

* **Endpoint:** `POST https://services.leadconnectorhq.com/contacts/{contactId}/notes`
* **Purpose:** To provide a clear reason why a lead was moved to "Lost".
* **Example Payload (JSON):**
    ```json
    {
      "body": "System Note: Opportunity closed by another representative."
    }
    ```

### **2.4 Get Contact's Notes**
Used to fetch the complete note history from a lead before it is returned to the Master DB, ensuring all agent insights are captured.

* **Endpoint:** `GET https://services.leadconnectorhq.com/contacts/{contactId}/notes`
* **Purpose:** To archive an agent's interaction history in the Master DB.

---
## Part 3: GHL -> n8n (Inbound Webhooks 📥)

This section details the signals GHL will send **to** our n8n controller. These are the triggers that make our real-time system possible.

### **3.1 Webhook: Fresh Lead Created**
This is the webhook that the internal GHL automation will call to notify the n8n Registrar.

* **Trigger:** The last step in the "Fresh Lead Capture" workflow inside GHL.
* **Purpose:** To register a new Fresh Lead into the PostgreSQL Master DB.
* **Expected Payload (JSON):** The GHL workflow should be configured to send a payload containing all the necessary contact data.
    ```json
    {
      "contact_id": "GHLContactIDHere",
      "first_name": "Jane",
      "last_name": "Smith",
      "phone": "+60198765432",
      "email": "jane.smith@email.com",
      "tags": "lead_type:fresh, source:Meta",
      "location_id": "AbcLocationIDForTeamA",
      "custom_fields": {
        "project_site": "Project Alpha",
        "budget_range": "300k-500k"
      }
    }
    ```

### **3.2 Webhook: Contact Pipeline Stage Changed**
This is the most important webhook for our Referee. It's a native trigger within GHL's "Triggers" section.

* **Trigger:** Any time a contact's opportunity is moved from one stage to another.
* **Purpose:** Allows the n8n Referee to detect the "Win" condition (moved to Pre-Booking) and the "Surrender" condition (moved to Lost).
* **Expected Payload (JSON):** GHL sends a standard payload for this event.
    ```json
    {
      "contact_id": "GHLContactIDHere",
      "pipelineId": "PipelineIDHere",
      "pipelineStageId_from": "PreviousStageID",
      "pipelineStageId_to": "NewStageID",
      "date_added": "2025-10-12T17:06:00.000Z"
    }
    ```
With this document, we have a crystal-clear blueprint for the communication network. We know exactly what language our systems will speak, what commands they will give, and what signals they will listen for.

The final piece of foundational documentation is the **"Operations & Maintenance Manual."** Shall we build it?
