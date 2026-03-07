# Supply Chain Intelligence Demo — End-to-End Setup Guide

## Complete Multi-Agent Orchestrator with Snowflake Cortex, Microsoft Fabric, and AI Foundry

This guide walks you through the **entire setup** — from creating the Snowflake database to deploying a multi-agent orchestrator in Microsoft AI Foundry that routes queries across Snowflake Cortex (via MCP) and Microsoft Fabric Data Agents.

---

## Architecture Overview

```
                    ┌─────────────────────────┐
                    │   User (Teams / Web /    │
                    │   Copilot / API)         │
                    └────────┬────────────────┘
                             │
                    ┌────────▼────────────────┐
                    │  Microsoft AI Foundry    │
                    │  Orchestrator Agent      │
                    │  (GPT-5.2 / GPT-5.1)    │
                    └────┬──────────┬─────────┘
                         │          │
            ┌────────────▼──┐  ┌───▼─────────────┐
            │  Snowflake    │  │  Fabric Data     │
            │  MCP Server   │  │  Agent           │
            │  (SSE)        │  │  (Lakehouse)     │
            └───────┬───────┘  └───┬──────────────┘
                    │              │
        ┌───────────▼──┐    ┌─────▼────────────┐
        │ Cortex Agent │    │ Lakehouse Tables  │
        │ - Analyst    │    │ - freight_costs   │
        │ - Search x3  │    │ - customer_returns│
        └──────────────┘    └──────────────────┘
```

**What each layer does:**

| Layer | Platform | Purpose |
|---|---|---|
| **AI Foundry Orchestrator** | Microsoft Azure | Routes user questions to the right data source |
| **Snowflake MCP Server** | Snowflake | Exposes Cortex Agent as an MCP tool over SSE |
| **Cortex Agent** | Snowflake | Orchestrates Cortex Analyst (SQL) + 3 Cortex Search services |
| **Fabric Data Agent** | Microsoft Fabric | NL2SQL over Lakehouse tables (freight costs, customer returns) |

---

## Prerequisites

| Requirement | Details |
|---|---|
| **Snowflake account** | With ACCOUNTADMIN role (or equivalent privileges) and Cortex features enabled |
| **Azure subscription** | With access to Microsoft Foundry (formerly Azure AI Foundry) |
| **Microsoft Fabric** | F2 or higher capacity (or Power BI Premium P1+) with Fabric enabled |
| **Azure CLI** | Installed and authenticated (`az login`) — for SDK-based agent creation |
| **Python 3.10+** | For SDK-based agent creation (optional) |

---

## File Inventory

All setup files are in the `setup/` directory. The two CSV files are in the project root.

| File | Purpose |
|---|---|
| `setup/01_database_and_warehouse.sql` | Creates database and warehouse |
| `setup/02_create_tables.sql` | Creates all 13 tables and a stage |
| `setup/03_load_structured_data.sql` | Loads 7 structured tables (~1,100 rows) |
| `setup/04_load_semi_structured_data.sql` | Loads 3 semi-structured tables (~160 rows of JSON) |
| `setup/05_load_unstructured_data.sql` | Loads 3 unstructured text tables (~63 rows) |
| `setup/06_cortex_search_services.sql` | Creates 3 Cortex Search services |
| `setup/07_semantic_view.sql` | Creates semantic view for Cortex Analyst |
| `setup/08_cortex_agent.sql` | Creates Cortex Agent with 4 tools |
| `setup/09_mcp_server.sql` | Creates MCP Server exposing the agent |
| `setup/10_foundry_instructions.md` | Orchestrator agent instructions (paste into Foundry) |
| `transportation_freight_costs.csv` | Freight cost data for Fabric Lakehouse (40 rows) |
| `customer_returns_complaints.csv` | Customer returns data for Fabric Lakehouse (40 rows) |

---

# Phase 1: Snowflake Setup (Scripts 01–09)

Run these SQL scripts **in order** in a Snowflake worksheet or via SnowSQL.

---

## Step 1 — Create Database and Warehouse

**Script:** `setup/01_database_and_warehouse.sql`

Creates:
- **Database:** `SUPPLY_CHAIN_DEMO`
- **Warehouse:** `SUPPLY_CHAIN_WH` (X-Small, auto-suspend 60 seconds)

```sql
-- Key commands in the script:
CREATE OR REPLACE DATABASE SUPPLY_CHAIN_DEMO;
CREATE OR REPLACE WAREHOUSE SUPPLY_CHAIN_WH
  WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE;
```

Run the full script, then confirm:
```sql
SHOW DATABASES LIKE 'SUPPLY_CHAIN_DEMO';
SHOW WAREHOUSES LIKE 'SUPPLY_CHAIN_WH';
```

---

## Step 2 — Create Tables

**Script:** `setup/02_create_tables.sql`

Creates 13 tables across three categories and a stage for semantic models:

### Structured Tables (7)

| Table | Key Columns | Notes |
|---|---|---|
| `SUPPLIERS` | SUPPLIER_ID (PK), SUPPLIER_NAME, COUNTRY, RELIABILITY_SCORE | 20 global suppliers |
| `PRODUCTS` | PRODUCT_ID (PK), PRODUCT_NAME, CATEGORY, SUPPLIER_ID (FK) | 50 products across 9 categories |
| `WAREHOUSES` | WAREHOUSE_ID (PK), WAREHOUSE_NAME, CITY, STATE, CAPACITY_SQFT | 8 US warehouses |
| `INVENTORY` | INVENTORY_ID (auto), PRODUCT_ID, WAREHOUSE_ID, QUANTITY_ON_HAND, REORDER_POINT | 162 stock records |
| `PURCHASE_ORDERS` | PO_ID (PK), SUPPLIER_ID, PRODUCT_ID, ORDER_QTY, DELAY_DAYS | 200 orders |
| `SHIPMENTS` | SHIPMENT_ID (PK), PO_ID, CARRIER, CURRENT_STATUS, ACTUAL_DELIVERY_DATE | 160 shipments |
| `STORE_SALES` | SALE_ID (PK), PRODUCT_ID, STORE_ID, CHANNEL, QUANTITY_SOLD, TOTAL_REVENUE | 500 sales records |

### Semi-Structured Tables (3) — VARIANT columns with JSON

| Table | Key Columns | Notes |
|---|---|---|
| `SHIPMENT_UPDATES` | UPDATE_ID (auto), SHIPMENT_ID, UPDATE_PAYLOAD (VARIANT) | Status change events |
| `IOT_SENSOR_LOGS` | LOG_ID (auto), WAREHOUSE_ID, SENSOR_PAYLOAD (VARIANT) | Temperature, humidity readings |
| `DELIVERY_TRACKING_EVENTS` | EVENT_ID (auto), SHIPMENT_ID, EVENT_PAYLOAD (VARIANT) | Delivery milestones |

### Unstructured Tables (3) — Free text

| Table | Key Columns | Notes |
|---|---|---|
| `SUPPLIER_EMAILS` | EMAIL_ID (PK), SUPPLIER_ID, SUBJECT, EMAIL_BODY | 30 supplier communications |
| `LOGISTICS_INCIDENT_REPORTS` | REPORT_ID (PK), INCIDENT_TYPE, REPORT_TEXT | 18 incident narratives |
| `WAREHOUSE_INSPECTION_NOTES` | INSPECTION_ID (PK), WAREHOUSE_ID, INSPECTION_NOTES, OVERALL_RATING | 15 inspection findings |

Also creates:
```sql
CREATE OR REPLACE STAGE SEMANTIC_MODELS
  DIRECTORY = (ENABLE = TRUE);
```

---

## Step 3 — Load Structured Data

**Script:** `setup/03_load_structured_data.sql`

Loads ~1,100 rows across the 7 structured tables:

| Table | Row Count |
|---|---|
| SUPPLIERS | 20 |
| PRODUCTS | 50 |
| WAREHOUSES | 8 |
| INVENTORY | 162 |
| PURCHASE_ORDERS | 200 |
| SHIPMENTS | 160 |
| STORE_SALES | 500 |

---

## Step 4 — Load Semi-Structured Data

**Script:** `setup/04_load_semi_structured_data.sql`

Loads JSON data using `PARSE_JSON()` into VARIANT columns:

| Table | Approx. Rows |
|---|---|
| SHIPMENT_UPDATES | ~55 |
| IOT_SENSOR_LOGS | ~55 |
| DELIVERY_TRACKING_EVENTS | ~50 |

Example of how data is inserted:
```sql
INSERT INTO SHIPMENT_UPDATES (SHIPMENT_ID, UPDATE_TIMESTAMP, UPDATE_PAYLOAD)
VALUES ('SHP-001', '2025-01-03 08:00:00',
  PARSE_JSON('{"status":"In Transit","location":"Los Angeles, CA","carrier":"UPS",...}'));
```

---

## Step 5 — Load Unstructured Data

**Script:** `setup/05_load_unstructured_data.sql`

Loads free-text content:

| Table | Row Count | Content |
|---|---|---|
| SUPPLIER_EMAILS | 30 | Communications about pricing, delays, quality issues |
| LOGISTICS_INCIDENT_REPORTS | 18 | Detailed narratives of logistics incidents |
| WAREHOUSE_INSPECTION_NOTES | 15 | Facility inspection findings and ratings |

---

## Step 6 — Create Cortex Search Services

**Script:** `setup/06_cortex_search_services.sql`

Creates 3 Cortex Search services for semantic search over unstructured text:

| Service Name | Source Table | Search Column | Attributes |
|---|---|---|---|
| `SUPPLIER_COMMS_SEARCH` | SUPPLIER_EMAILS | EMAIL_BODY | SUPPLIER_ID, SUBJECT, EMAIL_DATE |
| `INCIDENT_REPORTS_SEARCH` | LOGISTICS_INCIDENT_REPORTS | REPORT_TEXT | INCIDENT_TYPE, SEVERITY, REPORT_DATE |
| `WAREHOUSE_INSPECTIONS_SEARCH` | WAREHOUSE_INSPECTION_NOTES | INSPECTION_NOTES | WAREHOUSE_ID, INSPECTOR_NAME, OVERALL_RATING |

All services use:
- **Warehouse:** SUPPLY_CHAIN_WH
- **Refresh interval:** 1 hour (`TARGET_LAG = '1 hour'`)

> **Important:** Wait ~2–3 minutes after running this script for the search services to finish indexing before testing.

Verify:
```sql
SHOW CORTEX SEARCH SERVICES IN SUPPLY_CHAIN_DEMO.PUBLIC;
```

---

## Step 7 — Create Semantic View

**Script:** `setup/07_semantic_view.sql`

Creates the semantic view `SUPPLY_CHAIN_ANALYTICS` using the Cortex Analyst (CA) extension. This is what powers natural-language-to-SQL queries.

**Semantic view components:**

| Component | Count | Details |
|---|---|---|
| **Tables** | 7 | All structured tables |
| **Relationships** | 8 | Foreign key joins between tables |
| **Facts** | 20 | Numeric columns (quantities, costs, scores, etc.) |
| **Dimensions** | 38 | Categorical columns (names, statuses, dates, etc.) |
| **Metrics** | 16 | Pre-defined aggregations (total revenue, avg delay, etc.) |
| **Verified queries** | 5 | Known-good question-to-SQL mappings for accuracy |

Example verified query:
```
"What are the top 5 suppliers by total purchase order value?"
→ SELECT s.SUPPLIER_NAME, SUM(po.TOTAL_COST) AS total_po_value
  FROM PURCHASE_ORDERS po JOIN SUPPLIERS s ON po.SUPPLIER_ID = s.SUPPLIER_ID
  GROUP BY s.SUPPLIER_NAME ORDER BY total_po_value DESC LIMIT 5;
```

---

## Step 8 — Create Cortex Agent

**Script:** `setup/08_cortex_agent.sql`

Creates `SUPPLY_CHAIN_AGENT` with 4 tools:

| Tool Name | Type | Purpose | Data Source |
|---|---|---|---|
| `Analyst` | CORTEX_ANALYST | Natural language to SQL | Semantic view (7 structured tables) |
| `SupplierEmailSearch` | CORTEX_SEARCH | Search supplier emails | SUPPLIER_COMMS_SEARCH |
| `IncidentSearch` | CORTEX_SEARCH | Search incident reports | INCIDENT_REPORTS_SEARCH |
| `InspectionSearch` | CORTEX_SEARCH | Search inspection notes | WAREHOUSE_INSPECTIONS_SEARCH |

Configuration:
- **Model:** auto (Snowflake selects the best model)
- **Budget:** 16,000 tokens per response

Verify:
```sql
SHOW AGENTS IN SUPPLY_CHAIN_DEMO.PUBLIC;
```

---

## Step 9 — Create MCP Server

**Script:** `setup/09_mcp_server.sql`

Creates `SUPPLY_CHAIN_MCP_SERVER` that exposes the Cortex Agent via the Model Context Protocol (MCP):

```sql
CREATE OR REPLACE MCP SERVER SUPPLY_CHAIN_MCP_SERVER
  AGENT = SUPPLY_CHAIN_AGENT
  TOOLS = (
    TYPE = CORTEX_AGENT_RUN
  );
```

**SSE Endpoint:**
```
https://<YOUR_ACCOUNT>.snowflakecomputing.com/api/v2/databases/SUPPLY_CHAIN_DEMO/schemas/PUBLIC/mcp-servers/SUPPLY_CHAIN_MCP_SERVER/sse
```

Replace `<YOUR_ACCOUNT>` with your Snowflake account identifier (e.g., `SFSEAPAC-BSURESH`).

Verify:
```sql
SHOW MCP SERVERS IN SUPPLY_CHAIN_DEMO.PUBLIC;
```

---

## Verify Phase 1

After running all 9 scripts, run these verification queries:

```sql
-- Check all tables have data
SELECT 'SUPPLIERS' AS tbl, COUNT(*) AS cnt FROM SUPPLY_CHAIN_DEMO.PUBLIC.SUPPLIERS
UNION ALL SELECT 'PRODUCTS', COUNT(*) FROM SUPPLY_CHAIN_DEMO.PUBLIC.PRODUCTS
UNION ALL SELECT 'WAREHOUSES', COUNT(*) FROM SUPPLY_CHAIN_DEMO.PUBLIC.WAREHOUSES
UNION ALL SELECT 'INVENTORY', COUNT(*) FROM SUPPLY_CHAIN_DEMO.PUBLIC.INVENTORY
UNION ALL SELECT 'PURCHASE_ORDERS', COUNT(*) FROM SUPPLY_CHAIN_DEMO.PUBLIC.PURCHASE_ORDERS
UNION ALL SELECT 'SHIPMENTS', COUNT(*) FROM SUPPLY_CHAIN_DEMO.PUBLIC.SHIPMENTS
UNION ALL SELECT 'STORE_SALES', COUNT(*) FROM SUPPLY_CHAIN_DEMO.PUBLIC.STORE_SALES
UNION ALL SELECT 'SUPPLIER_EMAILS', COUNT(*) FROM SUPPLY_CHAIN_DEMO.PUBLIC.SUPPLIER_EMAILS
UNION ALL SELECT 'INCIDENT_REPORTS', COUNT(*) FROM SUPPLY_CHAIN_DEMO.PUBLIC.LOGISTICS_INCIDENT_REPORTS
UNION ALL SELECT 'INSPECTION_NOTES', COUNT(*) FROM SUPPLY_CHAIN_DEMO.PUBLIC.WAREHOUSE_INSPECTION_NOTES;

-- Check Cortex Search services
SHOW CORTEX SEARCH SERVICES IN SUPPLY_CHAIN_DEMO.PUBLIC;

-- Check agent
SHOW AGENTS IN SUPPLY_CHAIN_DEMO.PUBLIC;

-- Check MCP server
SHOW MCP SERVERS IN SUPPLY_CHAIN_DEMO.PUBLIC;
```

Expected row counts:

| Table | Rows |
|---|---|
| SUPPLIERS | 20 |
| PRODUCTS | 50 |
| WAREHOUSES | 8 |
| INVENTORY | 162 |
| PURCHASE_ORDERS | 200 |
| SHIPMENTS | 160 |
| STORE_SALES | 500 |
| SUPPLIER_EMAILS | 30 |
| LOGISTICS_INCIDENT_REPORTS | 18 |
| WAREHOUSE_INSPECTION_NOTES | 15 |

---

# Phase 2: Microsoft Fabric Setup

In this phase, you create a Fabric workspace with a Lakehouse containing the two CSV files that are **not** in Snowflake — transportation freight costs and customer returns/complaints.

---

## Step 10 — Create a Fabric Workspace

1. Go to [https://app.fabric.microsoft.com](https://app.fabric.microsoft.com) and sign in
2. In the left navigation, select **Workspaces** > **New workspace**
3. Name it: `SupplyChainDemo`
4. Under **License mode**, select a capacity that supports Fabric (Trial, Premium, or Fabric F2+)
5. Click **Apply**

---

## Step 11 — Create a Lakehouse

1. Inside the `SupplyChainDemo` workspace, click **+ New item**
2. Search for **Lakehouse** and select it
3. Name it: `SupplyChainLakehouse`
4. Click **Create**
5. Wait for the Lakehouse to provision (~30 seconds)

---

## Step 12 — Upload CSV Files to the Lakehouse

You have two CSV files in the project root directory:

| File | Description | Rows | Key Columns |
|---|---|---|---|
| `transportation_freight_costs.csv` | Freight rates, carrier invoices, detention charges, damage claims | 40 | freight_id, shipment_id, carrier, total_freight_cost, invoice_status, damage_claim_amount, carrier_rating |
| `customer_returns_complaints.csv` | Return reasons, RMA tracking, refunds, satisfaction scores | 40 | return_id, product_name, supplier_name, is_supplier_defect, complaint_severity, customer_satisfaction_score, sla_breached |

**Upload steps:**

1. In the Lakehouse explorer, click the **Files** folder
2. Click the **...** (ellipsis) menu next to **Files** > **New subfolder** > name it `supply_chain_data`
3. Click the **...** menu next to `supply_chain_data` > **Upload** > **Upload files**
4. Select `transportation_freight_costs.csv` and upload it
5. Repeat for `customer_returns_complaints.csv`

---

## Step 13 — Load CSV Files into Delta Tables

For each CSV file:

1. In the Lakehouse explorer, navigate to **Files** > `supply_chain_data`
2. Click the **...** menu next to `transportation_freight_costs.csv`
3. Select **Load to Tables** > **New table**
4. Table name: `transportation_freight_costs`
5. Click **Load**
6. Repeat for `customer_returns_complaints.csv` with table name: `customer_returns_complaints`

---

## Step 14 — Verify the Lakehouse Tables

1. In the Lakehouse explorer, expand **Tables** > **dbo** and confirm both tables appear
2. Click each table to preview data
3. Switch to **SQL analytics endpoint** (top-right dropdown) and run:

```sql
-- Verify freight data
SELECT carrier, COUNT(*) AS shipment_count,
       AVG(total_freight_cost) AS avg_cost,
       SUM(CASE WHEN on_time = 'No' THEN 1 ELSE 0 END) AS late_count
FROM transportation_freight_costs
GROUP BY carrier
ORDER BY avg_cost DESC;

-- Verify returns data
SELECT reason_category, COUNT(*) AS return_count,
       SUM(refund_amount) AS total_refunds,
       AVG(customer_satisfaction_score) AS avg_satisfaction
FROM customer_returns_complaints
GROUP BY reason_category
ORDER BY return_count DESC;
```

---

## Step 15 — Create a Fabric Data Agent

1. In the `SupplyChainDemo` workspace, click **+ New item**
2. Search for **Data Agent** and select **Fabric data agent** (Preview)
3. Name it: `SupplyChainFreightReturnsAgent`
4. Click **Create**

### Add the Lakehouse as a Data Source

1. In the OneLake catalog that appears, find and select `SupplyChainLakehouse`
2. Click **Add**
3. In the Explorer pane, expand the lakehouse and select both tables:
   - `transportation_freight_costs`
   - `customer_returns_complaints`

### Add Data Agent Instructions

Click the **Data agent instructions** button (top right) and paste:

```
You are a Supply Chain Data Agent specializing in transportation costs and customer returns.

Data Source: SupplyChainLakehouse with two tables:

1. transportation_freight_costs — Contains freight shipment details including:
   carrier, origin_city, destination_warehouse_id, service_level, weight_kg,
   freight_rate_per_kg, fuel_surcharge_pct, total_freight_cost, invoice_status,
   detention_hours, detention_charge, damage_claim_amount, on_time, carrier_rating, notes

2. customer_returns_complaints — Contains customer return details including:
   product_name, category, return_reason, reason_category, quantity_returned,
   refund_amount, return_channel, condition_on_return, rma_status, supplier_name,
   is_supplier_defect, complaint_severity, customer_satisfaction_score,
   sla_days, actual_resolution_days, sla_breached, resolution_type, agent_notes

Guidelines:
- For freight cost questions, use the transportation_freight_costs table
- For return/complaint questions, use the customer_returns_complaints table
- Always include relevant context: trends, comparisons, totals
- Flag critical issues: disputed invoices, safety issues, SLA breaches
- When asked about suppliers, cross-reference is_supplier_defect field
- Carrier ratings range from 1-5 (5 = best)
- Complaint severity: Critical > High > Medium > Low
```

### Add Example Queries

Click **Example queries** and add these question-SQL pairs:

**Question 1:** Which carriers have the highest freight costs per kg?
```sql
SELECT carrier,
       COUNT(*) AS total_shipments,
       ROUND(AVG(freight_rate_per_kg), 2) AS avg_rate_per_kg,
       ROUND(AVG(total_freight_cost), 2) AS avg_total_cost,
       ROUND(AVG(carrier_rating), 1) AS avg_rating
FROM transportation_freight_costs
GROUP BY carrier
ORDER BY avg_rate_per_kg DESC
```

**Question 2:** Which suppliers have the most product defect returns?
```sql
SELECT supplier_name,
       COUNT(*) AS total_returns,
       SUM(refund_amount) AS total_refund_amount,
       ROUND(AVG(customer_satisfaction_score), 1) AS avg_satisfaction,
       SUM(CASE WHEN complaint_severity = 'Critical' THEN 1 ELSE 0 END) AS critical_count
FROM customer_returns_complaints
WHERE is_supplier_defect = 'Yes'
GROUP BY supplier_name
ORDER BY total_returns DESC
```

**Question 3:** Show disputed or outstanding freight invoices
```sql
SELECT freight_id, carrier, origin_city, total_freight_cost,
       invoice_number, invoice_status, damage_claim_amount, notes
FROM transportation_freight_costs
WHERE invoice_status IN ('Disputed', 'Outstanding')
ORDER BY total_freight_cost DESC
```

### Test the Data Agent

Use the chat interface to test:
- "Which carriers have the most late deliveries?"
- "Show me all safety issue returns"
- "What is the total damage claim amount by carrier?"
- "Which suppliers have the highest return rates?"

### Publish the Data Agent

1. Click **Publish** in the top toolbar
2. The agent is now available for integration with AI Foundry

---

# Phase 3: Microsoft AI Foundry — Orchestrator Agent

In this phase, you create the orchestrator agent in AI Foundry that connects to both the Snowflake MCP Server and the Fabric Data Agent.

---

## Step 16 — Create an AI Foundry Project

1. Go to [https://ai.azure.com](https://ai.azure.com) (Microsoft Foundry portal)
2. Click **+ Create project** (or select an existing project)
3. Configure:
   - **Project name:** `SupplyChainOrchestrator`
   - **Region:** `East US 2` (recommended — widest model availability including GPT-5.2)
   - **Resource group:** Create new or select existing
4. Click **Create**
5. Wait for provisioning to complete

---

## Step 17 — Deploy the Latest OpenAI Model

1. In your Foundry project, go to **Models + endpoints** in the left navigation
2. Click **+ Deploy model** > **Deploy base model**
3. Search for and select **gpt-5.2** (latest as of March 2026)
4. Configure deployment:
   - **Deployment name:** `gpt-5-2-supply-chain` (or your preferred name)
   - **Deployment type:** Global Standard
   - **Tokens per minute rate limit:** Start with 30K (adjust based on usage)
5. Click **Deploy**
6. Note the **deployment name** — you will need it when creating the agent

> **Model availability (March 2026):**
>
> | Model | Best For | Regions |
> |---|---|---|
> | **gpt-5.2** | Most capable, best reasoning | East US 2, Sweden Central, South Central US |
> | **gpt-5.1** | Strong all-around | East US, East US 2, Canada East, Japan East, UK South, Sweden Central, Switzerland North |
> | **gpt-5** | Good balance of cost/performance | Most regions |
> | **gpt-5-mini** | Cost-effective, good for testing | Most regions |

---

## Step 18 — Set Up OAuth Authentication for Snowflake MCP

Run in Snowflake (as ACCOUNTADMIN):

```sql
-- Create OAuth security integration for AI Foundry
CREATE OR REPLACE SECURITY INTEGRATION foundry_mcp_oauth
  TYPE = OAUTH
  OAUTH_CLIENT = CUSTOM
  ENABLED = TRUE
  OAUTH_CLIENT_TYPE = 'CONFIDENTIAL'
  OAUTH_REDIRECT_URI = 'https://ai.azure.com/oauth/callback'
  OAUTH_ISSUE_REFRESH_TOKENS = TRUE
  OAUTH_REFRESH_TOKEN_VALIDITY = 86400;

-- Retrieve client credentials (save these securely)
SELECT SYSTEM$SHOW_OAUTH_CLIENT_SECRETS('FOUNDRY_MCP_OAUTH');

-- Get the OAuth client ID
DESCRIBE SECURITY INTEGRATION foundry_mcp_oauth;
-- Look for OAUTH_CLIENT_ID in the output
```

> **Important:** Save the `OAUTH_CLIENT_ID` and `OAUTH_CLIENT_SECRET` securely. You will need them in Step 20.

---

## Step 19 — Create the Orchestrator Agent

### Option A: Via the Foundry Portal (No Code)

1. In your Foundry project, go to **Agents** in the left navigation
2. Click **+ New agent**
3. Configure:
   - **Name:** `SupplyChainOrchestrator`
   - **Model:** Select your deployed model (e.g., `gpt-5-2-supply-chain`)
   - **Instructions:** Paste the full content from `setup/10_foundry_instructions.md`
4. Under **Tools**, you will add the Snowflake MCP Server (Step 20) and Fabric Data Agent (Step 21)

### Option B: Via Python SDK

Install dependencies:
```bash
pip install azure-ai-projects azure-identity python-dotenv
```

Create a `.env` file in your project root:
```
PROJECT_ENDPOINT=https://<your-foundry-resource>.services.ai.azure.com/api/projects/<your-project>
MODEL_DEPLOYMENT_NAME=gpt-5-2-supply-chain
```

Create `create_agent.py`:
```python
import os
from dotenv import load_dotenv
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import PromptAgentDefinition, MCPTool

load_dotenv()

project_client = AIProjectClient(
    endpoint=os.environ["PROJECT_ENDPOINT"],
    credential=DefaultAzureCredential(),
)

# Read the orchestrator instructions
with open("setup/10_foundry_instructions.md", "r") as f:
    instructions = f.read()

# Define the Snowflake MCP tool
snowflake_mcp_tool = MCPTool(
    server_label="supply-chain-snowflake",
    server_url="https://SFSEAPAC-BSURESH.snowflakecomputing.com/api/v2/databases/SUPPLY_CHAIN_DEMO/schemas/PUBLIC/mcp-servers/SUPPLY_CHAIN_MCP_SERVER",
    require_approval="never",
)

# Create the agent
agent = project_client.agents.create_version(
    agent_name="SupplyChainOrchestrator",
    definition=PromptAgentDefinition(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        instructions=instructions,
        tools=[snowflake_mcp_tool],
    ),
)

print(f"Agent created: id={agent.id}, name={agent.name}, version={agent.version}")
```

Run:
```bash
az login
python create_agent.py
```

---

## Step 20 — Add Snowflake MCP Server as a Tool

### Create a Connection in AI Foundry

1. In your Foundry project, go to **Management** > **Connected resources** (or **Settings** > **Connections**)
2. Click **+ New connection**
3. Select **Custom** connection type
4. Configure:
   - **Name:** `snowflake-mcp-connection`
   - **Access:** Project-level
   - **Authentication type:** OAuth 2.0
   - **Client ID:** (from Step 18)
   - **Client Secret:** (from Step 18)
   - **Token endpoint:** `https://SFSEAPAC-BSURESH.snowflakecomputing.com/oauth/token-request`
   - **Authorization endpoint:** `https://SFSEAPAC-BSURESH.snowflakecomputing.com/oauth/authorize`
5. Click **Save**

### Add the MCP Server Tool to the Agent

1. Go to your agent **SupplyChainOrchestrator**
2. Click **Tools** in the agent configuration
3. Click **+ Add tool** > **MCP Server**
4. Configure:
   - **Server label:** `supply-chain-snowflake`
   - **Server URL:** `https://SFSEAPAC-BSURESH.snowflakecomputing.com/api/v2/databases/SUPPLY_CHAIN_DEMO/schemas/PUBLIC/mcp-servers/SUPPLY_CHAIN_MCP_SERVER`
   - **Connection:** Select `snowflake-mcp-connection`
   - **Require approval:** `never` (for demo; use `always` in production)
5. Click **Save**

### MCP Server Endpoint Reference

```
Endpoint URL:
https://SFSEAPAC-BSURESH.snowflakecomputing.com/api/v2/databases/SUPPLY_CHAIN_DEMO/schemas/PUBLIC/mcp-servers/SUPPLY_CHAIN_MCP_SERVER

SSE Endpoint (for streaming):
https://SFSEAPAC-BSURESH.snowflakecomputing.com/api/v2/databases/SUPPLY_CHAIN_DEMO/schemas/PUBLIC/mcp-servers/SUPPLY_CHAIN_MCP_SERVER/sse

Tool exposed:
  - supply-chain-agent (Cortex Agent with Analyst + 3 Search services)

Supported operations:
  - tools/list    — Discover available tools
  - tools/call    — Invoke a tool (Analyst query or Search)

Authentication:
  - OAuth 2.0 (recommended for production)
  - Programmatic Access Token (PAT) for development/testing
```

---

## Step 21 — Connect Fabric Data Agent to the Foundry Orchestrator

1. In your Foundry project, go to your **SupplyChainOrchestrator** agent
2. Click **Tools** > **+ Add tool**
3. Select **Fabric** from the available integrations
   - If Fabric appears as a connected service, select your workspace `SupplyChainDemo`
   - Select the `SupplyChainFreightReturnsAgent`
4. Click **Add**

> **Alternative — A2A Protocol:** If direct Fabric integration is not available in your region, you can use the Agent-to-Agent (A2A) protocol:
> 1. In **Tools** > **+ Add tool** > **A2A endpoint**
> 2. Provide the Fabric Data Agent's endpoint URL
> 3. Configure authentication (Entra ID)

---

## Step 22 — Query Routing Rules

The orchestrator instructions (`setup/10_foundry_instructions.md`) contain routing logic that directs queries to the right data source:

| Query Topic | Routed To | Tool Used |
|---|---|---|
| Suppliers, reliability scores | Snowflake | MCP → Cortex Analyst |
| Products, categories | Snowflake | MCP → Cortex Analyst |
| Inventory, stock levels, reorder points | Snowflake | MCP → Cortex Analyst |
| Purchase orders, delays | Snowflake | MCP → Cortex Analyst |
| Shipments, carriers, delivery status | Snowflake | MCP → Cortex Analyst |
| Store sales, revenue, channels | Snowflake | MCP → Cortex Analyst |
| Supplier emails | Snowflake | MCP → Cortex Search (SUPPLIER_COMMS_SEARCH) |
| Incident reports | Snowflake | MCP → Cortex Search (INCIDENT_REPORTS_SEARCH) |
| Warehouse inspections | Snowflake | MCP → Cortex Search (WAREHOUSE_INSPECTIONS_SEARCH) |
| Freight costs, carrier invoices, damage claims | Fabric | Data Agent (NL2SQL) |
| Customer returns, RMAs, complaints, refunds | Fabric | Data Agent (NL2SQL) |
| Cross-platform analysis | Both | MCP + Data Agent → synthesized response |

---

# Phase 4: Testing and Validation

---

## Step 23 — Test the Multi-Agent Orchestrator

In your Foundry agent's chat interface, test these queries:

### Snowflake-routed queries (via MCP)

1. "Which suppliers have reliability scores below 0.7?"
2. "What products are at risk of stockout in the next 5 days?"
3. "Show me all purchase orders delayed by more than 10 days"
4. "Search supplier emails about pricing changes"
5. "Show me warehouse inspection reports with Poor ratings"
6. "What incidents involved temperature violations?"

### Fabric-routed queries

7. "Which carriers have the highest freight costs per kg?"
8. "Show me all disputed freight invoices"
9. "What products have the most customer returns due to defects?"
10. "Which suppliers have critical safety issues in customer returns?"

### Cross-platform queries (both data sources)

11. "Which suppliers have both high delivery delays AND high return rates?"
    - Expected: Agent queries Snowflake for delay data AND Fabric for return data, then synthesizes
12. "Compare Shenzhen Fast Supply's performance across shipments, POs, and customer complaints"
    - Expected: Agent queries Snowflake for PO/shipment data AND Fabric for return data

### Expected routing behavior

| Query Type | Data Source | Tool Used |
|---|---|---|
| Inventory levels | Snowflake | MCP → Cortex Analyst |
| Supplier emails | Snowflake | MCP → Cortex Search |
| Freight costs | Fabric | Data Agent (NL2SQL) |
| Customer returns | Fabric | Data Agent (NL2SQL) |
| Cross-platform | Both | MCP + Data Agent |

---

# Phase 5: Production Deployment (Optional)

---

## Step 24 — Expose via Microsoft Teams / Copilot

AI Foundry agents can be published to Microsoft 365 Copilot and Teams:

1. In Foundry, go to your agent > **Deploy**
2. Select **Microsoft 365 Copilot** or **Teams**
3. Follow the Copilot Studio integration prompts
4. Your supply chain team can now ask questions directly in Teams

## Step 25 — API Access

For programmatic access, use the Foundry API:

```bash
# Authenticate
AZURE_AI_AUTH_TOKEN=$(az account get-access-token \
  --resource "https://ai.azure.com" --query accessToken -o tsv)

# Query the orchestrator agent
curl -X POST "https://<YOUR-FOUNDRY-RESOURCE>.services.ai.azure.com/api/projects/<PROJECT>/openai/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $AZURE_AI_AUTH_TOKEN" \
  -d '{
    "agent_reference": {"type": "agent_reference", "name": "SupplyChainOrchestrator"},
    "input": [{"role": "user", "content": "Which suppliers are causing delivery delays and have the most customer complaints?"}]
  }'
```

---

# Complete Data Summary

## Snowflake Data (via Cortex Agent + MCP)

| Category | Tables | Total Rows |
|---|---|---|
| Structured | SUPPLIERS, PRODUCTS, WAREHOUSES, INVENTORY, PURCHASE_ORDERS, SHIPMENTS, STORE_SALES | ~1,100 |
| Semi-structured | SHIPMENT_UPDATES, IOT_SENSOR_LOGS, DELIVERY_TRACKING_EVENTS | ~160 |
| Unstructured | SUPPLIER_EMAILS, LOGISTICS_INCIDENT_REPORTS, WAREHOUSE_INSPECTION_NOTES | 63 |

## Fabric Lakehouse Data (via Data Agent)

| Table | Rows | Key Data |
|---|---|---|
| `transportation_freight_costs` | 40 | Freight IDs FRT-001 to FRT-040, 6 carriers (UPS, FedEx, DHL, USPS, Maersk, XPO Logistics), invoice statuses (Paid, Disputed, Outstanding), detention charges, damage claims |
| `customer_returns_complaints` | 40 | Return IDs RET-001 to RET-040, linked to Snowflake suppliers/products, reason categories (Product Defect, Shipping Damage, Safety Issue, Fulfillment Error, Product Quality, Customer Misuse), severity levels, SLA tracking |

## Key Cross-Platform Insights the Orchestrator Can Surface

- **Shenzhen Fast Supply** (Supplier 7): Low reliability in Snowflake + highest return count in Fabric (4 returns, including 2 Critical safety issues)
- **Casablanca Imports** (Supplier 16): Customs issues in Snowflake + worst carrier rating (1.5–1.8) and safety incident in Fabric returns
- **Maersk**: Frequent ocean delays in Snowflake shipments + disputed invoices and damage claims in Fabric freight data
- **MidWest Industrial** (Supplier 8): Generally reliable in Snowflake but 2 SLA breaches in Fabric returns

---

# Quick Reference Checklist

| # | Task | Platform | Script / Reference |
|---|---|---|---|
| 1 | Create database and warehouse | Snowflake | `01_database_and_warehouse.sql` |
| 2 | Create 13 tables | Snowflake | `02_create_tables.sql` |
| 3 | Load structured data (1,100 rows) | Snowflake | `03_load_structured_data.sql` |
| 4 | Load semi-structured JSON data (~160 rows) | Snowflake | `04_load_semi_structured_data.sql` |
| 5 | Load unstructured text data (63 rows) | Snowflake | `05_load_unstructured_data.sql` |
| 6 | Create 3 Cortex Search services | Snowflake | `06_cortex_search_services.sql` |
| 7 | Create semantic view (Cortex Analyst) | Snowflake | `07_semantic_view.sql` |
| 8 | Create Cortex Agent (4 tools) | Snowflake | `08_cortex_agent.sql` |
| 9 | Create MCP Server | Snowflake | `09_mcp_server.sql` |
| 10 | Create Fabric workspace | Fabric | Step 10 |
| 11 | Create Lakehouse | Fabric | Step 11 |
| 12 | Upload CSV files | Fabric | Step 12 |
| 13 | Load CSVs into Delta tables | Fabric | Step 13 |
| 14 | Verify Lakehouse tables | Fabric | Step 14 |
| 15 | Create Fabric Data Agent | Fabric | Step 15 |
| 16 | Create AI Foundry project | Foundry | Step 16 |
| 17 | Deploy GPT-5.2 model | Foundry | Step 17 |
| 18 | Set up Snowflake OAuth for MCP | Snowflake | Step 18 |
| 19 | Create orchestrator agent | Foundry | Step 19 |
| 20 | Add Snowflake MCP as tool | Foundry | Step 20 |
| 21 | Connect Fabric Data Agent | Foundry | Step 21 |
| 22 | Verify routing rules | — | Step 22 |
| 23 | Test multi-agent orchestrator | Foundry | Step 23 |

---

# Troubleshooting

| Issue | Solution |
|---|---|
| MCP tool call times out | Snowflake MCP non-streaming timeout is 50 seconds. Simplify queries or increase warehouse size |
| OAuth token errors | Verify `OAUTH_CLIENT_ID` and secret. Ensure security integration is `ENABLED = TRUE` |
| Fabric Data Agent returns no results | Check that tables are loaded in Lakehouse (not just in Files folder). Refresh the explorer |
| Model not available in region | Switch to `East US 2` or `Sweden Central` for widest GPT-5.x availability |
| Agent doesn't route correctly | Review instructions in `10_foundry_instructions.md` — ensure routing rules are explicit |
| MCP hostname issues | Use hyphens (-) not underscores (_) in hostnames for MCP server connections |
| Cortex Search returns no results | Wait 2–3 minutes after creating search services for indexing to complete |
| Semi-structured queries fail | Ensure VARIANT columns are queried with proper path notation (e.g., `SENSOR_PAYLOAD:temperature`) |
| Semantic view errors | Check that all 7 structured tables have data before creating the semantic view |

---

# Cleanup

```sql
-- Remove all Snowflake objects
DROP DATABASE IF EXISTS SUPPLY_CHAIN_DEMO;
DROP WAREHOUSE IF EXISTS SUPPLY_CHAIN_WH;

-- Remove OAuth integration (if created)
DROP SECURITY INTEGRATION IF EXISTS foundry_mcp_oauth;
```

For Fabric:
1. Delete the `SupplyChainFreightReturnsAgent` data agent
2. Delete the `SupplyChainLakehouse` lakehouse
3. Delete the `SupplyChainDemo` workspace

For AI Foundry:
1. Delete the `SupplyChainOrchestrator` agent
2. Delete the model deployment
3. Delete the project (if no longer needed)

---

# References

- [Snowflake Managed MCP Server Documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp)
- [Snowflake QuickStart: AI Foundry + MCP](https://www.snowflake.com/en/developers/guides/getting-started-with-ai-foundry-and-the-snowflake-managed-mcp/)
- [Microsoft Foundry Agent Service Quickstart](https://learn.microsoft.com/en-us/azure/foundry/quickstarts/get-started-code)
- [Microsoft Foundry: Connect to MCP Server Endpoints](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/model-context-protocol)
- [Create a Fabric Data Agent](https://learn.microsoft.com/en-us/fabric/data-science/how-to-create-data-agent)
- [Fabric Data Agent End-to-End Tutorial](https://learn.microsoft.com/en-us/fabric/data-science/data-agent-end-to-end-tutorial)
- [Create a Fabric Lakehouse](https://learn.microsoft.com/en-us/fabric/data-engineering/tutorial-build-lakehouse)
- [Integrating AI Foundry with Snowflake Cortex Agents (Blog)](https://medium.com/snowflake/integrating-ai-foundry-with-snowflake-cortex-agents-55855c96211c)
- [Foundry Models: Region Availability](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure)
