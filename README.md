<div align="center">

# Multi-Agent Orchestrator

### Snowflake Cortex MCP + Microsoft AI Foundry + Fabric Data Agents

[![Snowflake](https://img.shields.io/badge/Snowflake-29B5E8?style=for-the-badge&logo=snowflake&logoColor=white)](https://www.snowflake.com)
[![Microsoft Azure](https://img.shields.io/badge/Azure_AI_Foundry-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)](https://ai.azure.com)
[![Microsoft Fabric](https://img.shields.io/badge/Microsoft_Fabric-742774?style=for-the-badge&logo=microsoftazure&logoColor=white)](https://fabric.microsoft.com)
[![OpenAI](https://img.shields.io/badge/GPT--5.2-412991?style=for-the-badge&logo=openai&logoColor=white)](https://openai.com)

[![MCP Protocol](https://img.shields.io/badge/MCP-Model_Context_Protocol-00ADD8?style=flat-square)](https://modelcontextprotocol.io)
[![Cortex Agent](https://img.shields.io/badge/Cortex-Agent_|_Analyst_|_Search-29B5E8?style=flat-square)](https://docs.snowflake.com/en/user-guide/snowflake-cortex)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](https://opensource.org/licenses/MIT)

**A production-ready multi-agent architecture that routes natural language queries across Snowflake and Microsoft Fabric through a unified AI Foundry orchestrator.**

</div>

---

## Highlights

| Feature | Description |
|---|---|
| **Cortex Agent** | Text-to-SQL via Cortex Analyst + semantic search via 3 Cortex Search services |
| **MCP Server** | Exposes the Cortex Agent over the Model Context Protocol (SSE) |
| **AI Foundry Orchestrator** | GPT-5.2 powered agent that routes queries to the right data source |
| **Fabric Data Agent** | NL2SQL over Lakehouse Delta tables for supplementary datasets |
| **Cross-Platform Queries** | Single question can pull data from both Snowflake and Fabric simultaneously |

---

## Architecture

![Architecture](images/architecture.png)

```
                    +---------------------------+
                    |   User (Teams / Web /      |
                    |   Copilot / API)           |
                    +-----------+---------------+
                                |
                    +-----------v---------------+
                    |  Microsoft AI Foundry      |
                    |  Orchestrator Agent         |
                    |  (GPT-5.2)                 |
                    +-----+-------------+-------+
                          |             |
             +------------v---+   +----v----------------+
             |  Snowflake     |   |  Fabric Data        |
             |  MCP Server    |   |  Agent              |
             |  (SSE)         |   |  (Lakehouse)        |
             +-------+-------+   +----+----------------+
                     |                |
         +-----------v--+    +-------v-----------+
         | Cortex Agent |    | Lakehouse Tables  |
         | - Analyst    |    | - freight_costs   |
         | - Search x3  |    | - customer_returns|
         +--------------+    +-------------------+
```

| Layer | Platform | Purpose |
|---|---|---|
| **AI Foundry Orchestrator** | Microsoft Azure | Routes user questions to the right data source |
| **Snowflake MCP Server** | Snowflake | Exposes Cortex Agent as an MCP tool over SSE |
| **Cortex Agent** | Snowflake | Orchestrates Cortex Analyst (SQL) + 3 Cortex Search services |
| **Fabric Data Agent** | Microsoft Fabric | NL2SQL over Lakehouse tables |

---

## Prerequisites

| Requirement | Details |
|---|---|
| **Snowflake account** | ACCOUNTADMIN role with Cortex features enabled |
| **Azure subscription** | Access to Microsoft Foundry (formerly Azure AI Foundry) |
| **Microsoft Fabric** | F2+ capacity (or Power BI Premium P1+) |
| **Azure CLI** | Installed and authenticated (`az login`) |
| **Python 3.10+** | For SDK-based agent creation (optional) |

---

## Quick Start

> **25 steps across 5 phases — from zero to a working multi-agent orchestrator.**

| Phase | Platform | Steps | What You Build |
|:---:|---|---|---|
| **1** | Snowflake | 1-9 | Database, tables, data, Cortex Search, Semantic View, Agent, MCP Server |
| **2** | Fabric | 10-15 | Workspace, Lakehouse, Delta tables, Fabric Data Agent |
| **3** | AI Foundry | 16-22 | GPT-5.2 deployment, orchestrator agent, MCP + Fabric tool wiring |
| **4** | All | 23 | End-to-end testing |
| **5** | Azure | 24-25 | Teams/Copilot deployment, API access |

---

## Repository Structure

```
.
├── README.md                                   # This guide
├── transportation_freight_costs.csv            # Fabric Lakehouse data (40 rows)
├── customer_returns_complaints.csv             # Fabric Lakehouse data (40 rows)
└── setup/
    ├── 00_README.md                            # Extended setup guide (detailed copy)
    ├── 01_database_and_warehouse.sql           # Phase 1, Step 1
    ├── 02_create_tables.sql                    # Phase 1, Step 2
    ├── 03_load_structured_data.sql             # Phase 1, Step 3
    ├── 04_load_semi_structured_data.sql        # Phase 1, Step 4
    ├── 05_load_unstructured_data.sql           # Phase 1, Step 5
    ├── 06_cortex_search_services.sql           # Phase 1, Step 6
    ├── 07_semantic_view.sql                    # Phase 1, Step 7
    ├── 08_cortex_agent.sql                     # Phase 1, Step 8
    ├── 09_mcp_server.sql                       # Phase 1, Step 9
    └── 10_foundry_instructions.md              # Phase 3, Step 19 (agent instructions)
```

---

## Phase 1: Snowflake Setup (Scripts 01-09)

Run these SQL scripts **in order** in a Snowflake SQL worksheet or via SnowSQL.

---

### Step 1 — Create Database and Warehouse

> **Script:** [`setup/01_database_and_warehouse.sql`](setup/01_database_and_warehouse.sql)

Open a Snowflake worksheet and run:

```sql
USE ROLE ACCOUNTADMIN;

CREATE DATABASE IF NOT EXISTS SUPPLY_CHAIN_DEMO;

CREATE WAREHOUSE IF NOT EXISTS SUPPLY_CHAIN_WH
  WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE;

USE DATABASE SUPPLY_CHAIN_DEMO;
USE SCHEMA PUBLIC;
USE WAREHOUSE SUPPLY_CHAIN_WH;
```

**Verify:**
```sql
SHOW DATABASES LIKE 'SUPPLY_CHAIN_DEMO';
SHOW WAREHOUSES LIKE 'SUPPLY_CHAIN_WH';
```

You should see one row for each command confirming the database and warehouse exist.

---

### Step 2 — Create Tables

> **Script:** [`setup/02_create_tables.sql`](setup/02_create_tables.sql)

Run the full script. It creates 13 tables across three categories plus an internal stage:

<details>
<summary><b>Structured Tables (7)</b> — click to expand</summary>

| Table | Key Columns | Purpose |
|---|---|---|
| `SUPPLIERS` | SUPPLIER_ID, SUPPLIER_NAME, COUNTRY, RELIABILITY_SCORE | 20 global suppliers with reliability scoring |
| `PRODUCTS` | PRODUCT_ID, PRODUCT_NAME, CATEGORY, SUPPLIER_ID | 50 products across 9 categories linked to suppliers |
| `WAREHOUSES` | WAREHOUSE_ID, WAREHOUSE_NAME, CITY, CAPACITY_SQFT | 8 US warehouse facilities |
| `INVENTORY` | PRODUCT_ID, WAREHOUSE_ID, QUANTITY_ON_HAND, DAYS_OF_SUPPLY | Stock levels per product per warehouse |
| `PURCHASE_ORDERS` | PO_ID, SUPPLIER_ID, PRODUCT_ID, DELAY_DAYS, STATUS | Orders with expected vs actual delivery tracking |
| `SHIPMENTS` | SHIPMENT_ID, PO_ID, CARRIER, STATUS, ACTUAL_ARRIVAL | Carrier tracking with shipment weight |
| `STORE_SALES` | SALE_ID, PRODUCT_ID, STORE_ID, CHANNEL, REVENUE | Sales across 25 stores and 3 channels |

</details>

<details>
<summary><b>Semi-Structured Tables (3)</b> — VARIANT columns for JSON</summary>

| Table | Key Columns | Purpose |
|---|---|---|
| `SHIPMENT_UPDATES` | SHIPMENT_ID, UPDATE_DATA (VARIANT) | Tracking status change events as JSON |
| `IOT_SENSOR_LOGS` | WAREHOUSE_ID, SENSOR_DATA (VARIANT) | Temperature, humidity, motion sensor readings |
| `DELIVERY_TRACKING_EVENTS` | SHIPMENT_ID, TRACKING_DATA (VARIANT) | Last-mile delivery milestones |

</details>

<details>
<summary><b>Unstructured Text Tables (3)</b></summary>

| Table | Key Columns | Purpose |
|---|---|---|
| `SUPPLIER_EMAILS` | SUPPLIER_ID, SUBJECT, EMAIL_BODY | Supplier communications (delays, pricing, quality) |
| `LOGISTICS_INCIDENT_REPORTS` | INCIDENT_TYPE, SEVERITY, REPORT_TEXT | Equipment failures, safety incidents, security breaches |
| `WAREHOUSE_INSPECTION_NOTES` | WAREHOUSE_ID, INSPECTION_NOTES, OVERALL_RATING | Facility condition reports with ratings |

</details>

The script also creates an internal stage:

```sql
CREATE STAGE IF NOT EXISTS SEMANTIC_MODELS;
```

**Verify:** Run `SHOW TABLES IN SUPPLY_CHAIN_DEMO.PUBLIC;` — you should see 13 tables.

---

### Step 3 — Load Structured Data

> **Script:** [`setup/03_load_structured_data.sql`](setup/03_load_structured_data.sql)

Run the full script. It inserts ~1,100 rows of synthetic supply chain data via `INSERT INTO ... VALUES` statements:

| Table | Row Count | Notes |
|---|---|---|
| SUPPLIERS | 20 | 4 low-reliability suppliers (IDs 4, 7, 13, 17) for demo scenarios |
| PRODUCTS | 50 | 9 categories including perishable items |
| WAREHOUSES | 8 | US warehouses across different states |
| INVENTORY | 162 | ~30% of records at or below reorder point |
| PURCHASE_ORDERS | 200 | ~25% have delivery delays |
| SHIPMENTS | 160 | Multiple carriers (UPS, FedEx, DHL, Maersk, XPO) |
| STORE_SALES | 500 | 3 channels: In-Store, Online, Marketplace |

**Verify:**
```sql
SELECT 'SUPPLIERS' AS tbl, COUNT(*) AS cnt FROM SUPPLIERS
UNION ALL SELECT 'PRODUCTS', COUNT(*) FROM PRODUCTS
UNION ALL SELECT 'WAREHOUSES', COUNT(*) FROM WAREHOUSES
UNION ALL SELECT 'INVENTORY', COUNT(*) FROM INVENTORY
UNION ALL SELECT 'PURCHASE_ORDERS', COUNT(*) FROM PURCHASE_ORDERS
UNION ALL SELECT 'SHIPMENTS', COUNT(*) FROM SHIPMENTS
UNION ALL SELECT 'STORE_SALES', COUNT(*) FROM STORE_SALES;
```

---

### Step 4 — Load Semi-Structured Data

> **Script:** [`setup/04_load_semi_structured_data.sql`](setup/04_load_semi_structured_data.sql)

Run the full script. It inserts ~160 JSON rows using `PARSE_JSON()` into VARIANT columns:

| Table | Approx. Rows | Content |
|---|---|---|
| SHIPMENT_UPDATES | ~55 | Status events: picked_up, in_transit, delayed, customs_cleared, delivered |
| IOT_SENSOR_LOGS | ~55 | Sensor types: temperature, humidity, air quality, motion, door (~15% in alert state) |
| DELIVERY_TRACKING_EVENTS | ~50 | Last-mile events with proof-of-delivery data |

Example of how a JSON row is inserted:
```sql
INSERT INTO SHIPMENT_UPDATES (SHIPMENT_ID, EVENT_TIMESTAMP, UPDATE_DATA)
VALUES (1001, '2025-01-03 08:00:00',
  PARSE_JSON('{"status":"In Transit","location":"Los Angeles, CA","carrier":"UPS",...}'));
```

**Verify:**
```sql
SELECT 'SHIPMENT_UPDATES' AS tbl, COUNT(*) AS cnt FROM SHIPMENT_UPDATES
UNION ALL SELECT 'IOT_SENSOR_LOGS', COUNT(*) FROM IOT_SENSOR_LOGS
UNION ALL SELECT 'DELIVERY_TRACKING', COUNT(*) FROM DELIVERY_TRACKING_EVENTS;
```

---

### Step 5 — Load Unstructured Data

> **Script:** [`setup/05_load_unstructured_data.sql`](setup/05_load_unstructured_data.sql)

Run the full script. It inserts 63 rows of realistic free-text content:

| Table | Row Count | Content Types |
|---|---|---|
| SUPPLIER_EMAILS | 30 | Delay notices, quality alerts, pricing changes, capacity updates |
| LOGISTICS_INCIDENT_REPORTS | 18 | Equipment failures, shipping damage, safety incidents, security breaches |
| WAREHOUSE_INSPECTION_NOTES | 15 | Facility inspections with ratings: Excellent, Good, Fair, Poor |

**Verify:**
```sql
SELECT 'SUPPLIER_EMAILS' AS tbl, COUNT(*) AS cnt FROM SUPPLIER_EMAILS
UNION ALL SELECT 'INCIDENT_REPORTS', COUNT(*) FROM LOGISTICS_INCIDENT_REPORTS
UNION ALL SELECT 'INSPECTION_NOTES', COUNT(*) FROM WAREHOUSE_INSPECTION_NOTES;
```

---

### Step 6 — Create Cortex Search Services

> **Script:** [`setup/06_cortex_search_services.sql`](setup/06_cortex_search_services.sql)

Run the full script. It creates 3 Cortex Search services that enable semantic search over unstructured text:

| Service Name | Source Table | Search Column | Filterable Attributes |
|---|---|---|---|
| `SUPPLIER_COMMS_SEARCH` | SUPPLIER_EMAILS | EMAIL_BODY | SUPPLIER_NAME, SUBJECT, DATE_SENT, SENDER, PRIORITY |
| `INCIDENT_REPORTS_SEARCH` | LOGISTICS_INCIDENT_REPORTS | REPORT_TEXT | INCIDENT_TYPE, SEVERITY, REPORT_DATE, REPORTED_BY, STATUS |
| `WAREHOUSE_INSPECTIONS_SEARCH` | WAREHOUSE_INSPECTION_NOTES | INSPECTION_NOTES | INSPECTION_DATE, INSPECTOR, OVERALL_RATING, FOLLOW_UP_REQUIRED |

All services use `TARGET_LAG = '1 hour'` and warehouse `SUPPLY_CHAIN_WH`.

> **Important:** Wait **2-3 minutes** after running this script for the search services to finish indexing before proceeding.

**Verify:**
```sql
SHOW CORTEX SEARCH SERVICES IN SUPPLY_CHAIN_DEMO.PUBLIC;
```

You should see 3 services listed with `indexing_state` showing `ACTIVE` (or `PROVISIONING` if still building).

---

### Step 7 — Create Semantic View

> **Script:** [`setup/07_semantic_view.sql`](setup/07_semantic_view.sql)

Run the full script. This creates `SUPPLY_CHAIN_ANALYTICS`, a semantic view that powers Cortex Analyst text-to-SQL. It is the most complex object in the setup.

| Component | Count | Details |
|---|---|---|
| **Tables** | 7 | All structured tables (SUPPLIERS through STORE_SALES + WAREHOUSES) |
| **Relationships** | 8 | Foreign key joins (e.g., PRODUCTS.SUPPLIER_ID -> SUPPLIERS.SUPPLIER_ID) |
| **Facts** | 20 | Numeric columns: quantities, costs, scores, weights, delays |
| **Dimensions** | 38 | Categorical columns: names, statuses, dates, regions, channels |
| **Metrics** | 16 | Pre-defined aggregations (total revenue, avg delay, OTD rate, etc.) |
| **Verified queries** | 5 | Known-good question-to-SQL mappings that improve accuracy |

The semantic view also includes a Cortex Analyst (CA) extension with filters (e.g., "delivered orders", "delayed orders"), time dimensions, and onboarding questions.

<details>
<summary><b>Verified queries included</b></summary>

1. **"Which suppliers are causing the most delivery delays?"** — Joins PURCHASE_ORDERS with SUPPLIERS, groups by supplier, orders by avg delay
2. **"What products are at risk of stockouts?"** — Joins INVENTORY with PRODUCTS and WAREHOUSES, classifies stock status as Critical/Low/Adequate
3. **"What is the total revenue by product category?"** — Joins STORE_SALES with PRODUCTS, aggregates revenue
4. **"How are the different carriers performing?"** — Aggregates SHIPMENTS by carrier with transit days
5. **"What is the on-time delivery rate by supplier?"** — Calculates OTD percentage per supplier from PURCHASE_ORDERS

</details>

**Verify:**
```sql
DESCRIBE SEMANTIC VIEW SUPPLY_CHAIN_ANALYTICS;
```

---

### Step 8 — Create Cortex Agent

> **Script:** [`setup/08_cortex_agent.sql`](setup/08_cortex_agent.sql)

Run the full script. It creates `SUPPLY_CHAIN_AGENT` with 4 tools:

| Tool Name | Type | Purpose | Data Source |
|---|---|---|---|
| `Analyst` | `cortex_analyst_text_to_sql` | Natural language to SQL queries | Semantic view (7 structured tables) |
| `SupplierEmailSearch` | `cortex_search` | Search supplier communications | SUPPLIER_COMMS_SEARCH service |
| `IncidentSearch` | `cortex_search` | Search incident reports | INCIDENT_REPORTS_SEARCH service |
| `InspectionSearch` | `cortex_search` | Search warehouse inspections | WAREHOUSE_INSPECTIONS_SEARCH service |

The agent uses `model: auto` for orchestration and has a `budget: 16000` token limit per response.

**Verify:**
```sql
SHOW AGENTS IN SUPPLY_CHAIN_DEMO.PUBLIC;
DESCRIBE AGENT SUPPLY_CHAIN_DEMO.PUBLIC.SUPPLY_CHAIN_AGENT;
```

---

### Step 9 — Create MCP Server

> **Script:** [`setup/09_mcp_server.sql`](setup/09_mcp_server.sql)

Run the full script. It creates a Snowflake-managed MCP Server that exposes the Cortex Agent to external clients:

```sql
CREATE OR REPLACE MCP SERVER SUPPLY_CHAIN_MCP_SERVER
FROM SPECIFICATION $$
tools:
  - title: "Supply Chain Intelligence Agent"
    name: "supply-chain-agent"
    identifier: "SUPPLY_CHAIN_DEMO.PUBLIC.SUPPLY_CHAIN_AGENT"
    type: "CORTEX_AGENT_RUN"
    description: "Supply chain intelligence agent that answers questions about
      suppliers, inventory, purchase orders, shipments, warehouse operations,
      sales, supplier communications, incident reports, and warehouse inspections."
$$;
```

**SSE Endpoint** (you will need this in Phase 3):
```
https://<YOUR_ACCOUNT>.snowflakecomputing.com/api/v2/databases/SUPPLY_CHAIN_DEMO/schemas/PUBLIC/mcp-servers/SUPPLY_CHAIN_MCP_SERVER/sse
```

Replace `<YOUR_ACCOUNT>` with your Snowflake account identifier (e.g., `SFSEAPAC-BSURESH`).

**Verify:**
```sql
SHOW MCP SERVERS IN SUPPLY_CHAIN_DEMO.PUBLIC;
DESCRIBE MCP SERVER SUPPLY_CHAIN_DEMO.PUBLIC.SUPPLY_CHAIN_MCP_SERVER;
```

---

### Verify Phase 1 (All Objects)

Run this comprehensive check after completing all 9 scripts:

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

-- Check Cortex objects
SHOW CORTEX SEARCH SERVICES IN SUPPLY_CHAIN_DEMO.PUBLIC;
SHOW AGENTS IN SUPPLY_CHAIN_DEMO.PUBLIC;
SHOW MCP SERVERS IN SUPPLY_CHAIN_DEMO.PUBLIC;
```

<details>
<summary><b>Expected row counts</b></summary>

| Table | Expected Rows |
|---|---|
| SUPPLIERS | 20 |
| PRODUCTS | 50 |
| WAREHOUSES | 8 |
| INVENTORY | 162 |
| PURCHASE_ORDERS | 200 |
| SHIPMENTS | 160 |
| STORE_SALES | 500 |
| SUPPLIER_EMAILS | 30 |
| INCIDENT_REPORTS | 18 |
| INSPECTION_NOTES | 15 |

Plus: 3 Cortex Search services, 1 Agent, 1 MCP Server.

</details>

---

## Phase 2: Microsoft Fabric Setup

---

### Step 10 — Create a Fabric Workspace

1. Go to [https://app.fabric.microsoft.com](https://app.fabric.microsoft.com) and sign in
2. In the left navigation, select **Workspaces** > **New workspace**
3. Name it: `SupplyChainDemo`
4. Under **License mode**, select a capacity that supports Fabric (Trial, Premium, or Fabric F2+)
5. Click **Apply**

---

### Step 11 — Create a Lakehouse

1. Inside the `SupplyChainDemo` workspace, click **+ New item**
2. Search for **Lakehouse** and select it
3. Name it: `SupplyChainLakehouse`
4. Click **Create**
5. Wait for the Lakehouse to provision (~30 seconds)

---

### Step 12 — Upload CSV Files to the Lakehouse

Two CSV files are included in the project root:

| File | Description | Rows | Key Columns |
|---|---|---|---|
| `transportation_freight_costs.csv` | Freight rates, carrier invoices, detention charges, damage claims | 40 | freight_id, carrier, total_freight_cost, invoice_status, damage_claim_amount, carrier_rating |
| `customer_returns_complaints.csv` | Return reasons, RMA tracking, refunds, satisfaction scores | 40 | return_id, product_name, supplier_name, is_supplier_defect, complaint_severity, sla_breached |

**Upload steps:**

1. In the Lakehouse explorer, click the **Files** folder
2. Click the **...** (ellipsis) menu next to **Files** > **New subfolder** > name it `supply_chain_data`
3. Click the **...** menu next to `supply_chain_data` > **Upload** > **Upload files**
4. Select `transportation_freight_costs.csv` and upload it
5. Repeat for `customer_returns_complaints.csv`

---

### Step 13 — Load CSV Files into Delta Tables

For **each** CSV file:

1. In the Lakehouse explorer, navigate to **Files** > `supply_chain_data`
2. Click the **...** menu next to the CSV file
3. Select **Load to Tables** > **New table**
4. Set the table name:
   - `transportation_freight_costs` for the freight file
   - `customer_returns_complaints` for the returns file
5. Click **Load**
6. Repeat for the second CSV file

---

### Step 14 — Verify the Lakehouse Tables

1. In the Lakehouse explorer, expand **Tables** > **dbo** and confirm both tables appear
2. Click each table to preview the data
3. Switch to the **SQL analytics endpoint** (top-right dropdown) and run these verification queries:

```sql
-- Verify freight data (6 carriers expected)
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

### Step 15 — Create a Fabric Data Agent

1. In the `SupplyChainDemo` workspace, click **+ New item**
2. Search for **Data Agent** and select **Fabric data agent** (Preview)
3. Name it: `SupplyChainFreightReturnsAgent`
4. Click **Create**

#### Add the Lakehouse as a Data Source

1. In the OneLake catalog that appears, find and select `SupplyChainLakehouse`
2. Click **Add**
3. In the Explorer pane, expand the lakehouse and select both tables:
   - `transportation_freight_costs`
   - `customer_returns_complaints`

#### Add Data Agent Instructions

Click the **Data agent instructions** button (top right) and paste:

<details>
<summary><b>Click to expand instructions</b></summary>

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

</details>

#### Add Example Queries

Click **Example queries** and add these question-SQL pairs:

<details>
<summary><b>Query 1:</b> "Which carriers have the highest freight costs per kg?"</summary>

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

</details>

<details>
<summary><b>Query 2:</b> "Which suppliers have the most product defect returns?"</summary>

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

</details>

<details>
<summary><b>Query 3:</b> "Show disputed or outstanding freight invoices"</summary>

```sql
SELECT freight_id, carrier, origin_city, total_freight_cost,
       invoice_number, invoice_status, damage_claim_amount, notes
FROM transportation_freight_costs
WHERE invoice_status IN ('Disputed', 'Outstanding')
ORDER BY total_freight_cost DESC
```

</details>

#### Test the Data Agent

Use the built-in chat interface to test before publishing:
- "Which carriers have the most late deliveries?"
- "Show me all safety issue returns"
- "What is the total damage claim amount by carrier?"

#### Publish the Data Agent

1. Click **Publish** in the top toolbar
2. The agent is now available for integration with AI Foundry

---

## Phase 3: AI Foundry Orchestrator

---

### Step 16 — Create an AI Foundry Project

1. Go to [https://ai.azure.com](https://ai.azure.com) (Microsoft Foundry portal)
2. Click **+ Create project** (or select an existing project)
3. Configure:
   - **Project name:** `SupplyChainOrchestrator`
   - **Region:** `East US 2` (recommended for widest GPT-5.x model availability)
   - **Resource group:** Create new or select existing
4. Click **Create**
5. Wait for provisioning to complete

---

### Step 17 — Deploy the OpenAI Model

1. In your Foundry project, go to **Models + endpoints** in the left navigation
2. Click **+ Deploy model** > **Deploy base model**
3. Search for and select **gpt-5.2**
4. Configure deployment:
   - **Deployment name:** `gpt-5-2-supply-chain`
   - **Deployment type:** Global Standard
   - **Tokens per minute rate limit:** Start with 30K (adjust based on usage)
5. Click **Deploy**
6. Note the **deployment name** — you will need it when creating the agent

> **Model alternatives by region:**
>
> | Model | Regions |
> |---|---|
> | **gpt-5.2** | East US 2, Sweden Central, South Central US |
> | **gpt-5.1** | East US, East US 2, Canada East, Japan East, UK South, Sweden Central, Switzerland North |
> | **gpt-5** | Most regions |
> | **gpt-5-mini** | Most regions (cost-effective for testing) |

---

### Step 18 — Set Up OAuth Authentication for Snowflake MCP

Run in a Snowflake worksheet as ACCOUNTADMIN:

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

-- Retrieve client credentials (save these securely!)
SELECT SYSTEM$SHOW_OAUTH_CLIENT_SECRETS('FOUNDRY_MCP_OAUTH');

-- Get the OAuth client ID
DESCRIBE SECURITY INTEGRATION foundry_mcp_oauth;
-- Look for OAUTH_CLIENT_ID in the output
```

> **Important:** Save the `OAUTH_CLIENT_ID` and `OAUTH_CLIENT_SECRET` from the output. You will need them in Step 20.

---

### Step 19 — Create the Orchestrator Agent

#### Option A: Via the Foundry Portal (No Code)

1. In your Foundry project, go to **Agents** in the left navigation
2. Click **+ New agent**
3. Configure:
   - **Name:** `SupplyChainOrchestrator`
   - **Model:** Select your deployed model (e.g., `gpt-5-2-supply-chain`)
   - **Instructions:** Paste the full content from [`setup/10_foundry_instructions.md`](setup/10_foundry_instructions.md)
4. Under **Tools**, you will add the Snowflake MCP Server (Step 20) and Fabric Data Agent (Step 21)

#### Option B: Via Python SDK

Install dependencies:
```bash
pip install azure-ai-projects azure-identity python-dotenv
```

Create a `.env` file in your project root:
```
PROJECT_ENDPOINT=https://<your-foundry-resource>.services.ai.azure.com/api/projects/<your-project>
MODEL_DEPLOYMENT_NAME=gpt-5-2-supply-chain
```

<details>
<summary><b>create_agent.py</b> — click to expand</summary>

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
    server_url="https://<YOUR_ACCOUNT>.snowflakecomputing.com/api/v2/databases/SUPPLY_CHAIN_DEMO/schemas/PUBLIC/mcp-servers/SUPPLY_CHAIN_MCP_SERVER",
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

</details>

Run:
```bash
az login
python create_agent.py
```

---

### Step 20 — Add Snowflake MCP Server as a Tool

#### Create a Connection in AI Foundry

1. In your Foundry project, go to **Management** > **Connected resources** (or **Settings** > **Connections**)
2. Click **+ New connection**
3. Select **Custom** connection type
4. Configure:
   - **Name:** `snowflake-mcp-connection`
   - **Access:** Project-level
   - **Authentication type:** OAuth 2.0
   - **Client ID:** *(from Step 18)*
   - **Client Secret:** *(from Step 18)*
   - **Token endpoint:** `https://<YOUR_ACCOUNT>.snowflakecomputing.com/oauth/token-request`
   - **Authorization endpoint:** `https://<YOUR_ACCOUNT>.snowflakecomputing.com/oauth/authorize`
5. Click **Save**

#### Add the MCP Server Tool to the Agent

1. Go to your agent **SupplyChainOrchestrator**
2. Click **Tools** in the agent configuration
3. Click **+ Add tool** > **MCP Server**
4. Configure:
   - **Server label:** `supply-chain-snowflake`
   - **Server URL:** `https://<YOUR_ACCOUNT>.snowflakecomputing.com/api/v2/databases/SUPPLY_CHAIN_DEMO/schemas/PUBLIC/mcp-servers/SUPPLY_CHAIN_MCP_SERVER`
   - **Connection:** Select `snowflake-mcp-connection`
   - **Require approval:** `never` (for demo; use `always` in production)
5. Click **Save**

<details>
<summary><b>MCP Server endpoint reference</b></summary>

```
Endpoint URL:
https://<YOUR_ACCOUNT>.snowflakecomputing.com/api/v2/databases/SUPPLY_CHAIN_DEMO/schemas/PUBLIC/mcp-servers/SUPPLY_CHAIN_MCP_SERVER

SSE Endpoint (for streaming):
https://<YOUR_ACCOUNT>.snowflakecomputing.com/api/v2/databases/SUPPLY_CHAIN_DEMO/schemas/PUBLIC/mcp-servers/SUPPLY_CHAIN_MCP_SERVER/sse

Tool exposed:
  - supply-chain-agent (Cortex Agent with Analyst + 3 Search services)

Supported operations:
  - tools/list    — Discover available tools
  - tools/call    — Invoke a tool (Analyst query or Search)

Authentication:
  - OAuth 2.0 (recommended for production)
  - Programmatic Access Token (PAT) for development/testing
```

</details>

---

### Step 21 — Connect Fabric Data Agent to the Foundry Orchestrator

1. In your Foundry project, go to your **SupplyChainOrchestrator** agent
2. Click **Tools** > **+ Add tool**
3. Select **Fabric** from the available integrations
   - If Fabric appears as a connected service, select your workspace `SupplyChainDemo`
   - Select the `SupplyChainFreightReturnsAgent`
4. Click **Add**

> **Alternative (A2A Protocol):** If direct Fabric integration is not available in your region, use Agent-to-Agent:
> 1. In **Tools** > **+ Add tool** > **A2A endpoint**
> 2. Provide the Fabric Data Agent's endpoint URL
> 3. Configure authentication via Entra ID

---

### Step 22 — Verify Query Routing Rules

Confirm the orchestrator instructions (from `setup/10_foundry_instructions.md`) route queries correctly:

| Query Topic | Routed To | Tool Used |
|---|---|---|
| Suppliers, products, inventory, POs, shipments, sales | Snowflake | MCP -> Cortex Analyst |
| Supplier emails, incident reports, inspections | Snowflake | MCP -> Cortex Search |
| Freight costs, carrier invoices, detention charges | Fabric | Data Agent (NL2SQL) |
| Customer returns, complaints, RMAs, SLA breaches | Fabric | Data Agent (NL2SQL) |
| Cross-platform analysis (e.g., delays + returns) | Both | MCP + Data Agent |

---

## Phase 4: Testing

Test these queries in the Foundry agent chat interface. Each query should route to the correct data source automatically.

#### Snowflake-routed queries (via MCP -> Cortex Analyst)

| # | Test Query | What to Expect |
|---|---|---|
| 1 | "Which suppliers have reliability scores below 0.7?" | Returns 4 suppliers (IDs 4, 7, 13, 17) with low reliability |
| 2 | "What products are at risk of stockout in the next 5 days?" | Returns products where days_of_supply <= 5 |
| 3 | "Show me all purchase orders delayed by more than 10 days" | Returns POs with DELAY_DAYS > 10 |
| 4 | "Show me total revenue by sales channel" | Returns revenue for In-Store, Online, and Marketplace |

#### Snowflake-routed queries (via MCP -> Cortex Search)

| # | Test Query | What to Expect |
|---|---|---|
| 5 | "Search supplier emails about pricing changes" | Returns relevant supplier email excerpts about pricing |
| 6 | "What incidents involved temperature violations?" | Returns incident reports mentioning temperature issues |
| 7 | "Show me warehouse inspection reports with Poor ratings" | Returns inspection notes with Poor overall rating |

#### Fabric-routed queries (via Data Agent)

| # | Test Query | What to Expect |
|---|---|---|
| 8 | "Which carriers have the highest freight costs per kg?" | Returns carriers ranked by avg freight_rate_per_kg |
| 9 | "Show me all disputed freight invoices" | Returns rows where invoice_status = 'Disputed' |
| 10 | "What products have the most customer returns due to defects?" | Returns products where is_supplier_defect = 'Yes' |

#### Cross-platform queries (both data sources)

| # | Test Query | What to Expect |
|---|---|---|
| 11 | "Which suppliers have both high delivery delays AND high return rates?" | Agent queries Snowflake for delay data AND Fabric for return data, then synthesizes |
| 12 | "Compare Shenzhen Fast Supply's performance across shipments, POs, and customer complaints" | Agent queries both sources and provides a combined analysis |

---

## Phase 5: Production Deployment (Optional)

### Step 24 — Expose via Microsoft Teams / Copilot

1. In Foundry, go to your agent > **Deploy**
2. Select **Microsoft 365 Copilot** or **Teams**
3. Follow the Copilot Studio integration prompts
4. Your team can now ask supply chain questions directly in Teams

### Step 25 — API Access

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

## Data Summary

<details>
<summary><b>Snowflake Data</b> — via Cortex Agent + MCP (~1,320 total rows)</summary>

| Category | Tables | Rows |
|---|---|---|
| Structured | SUPPLIERS, PRODUCTS, WAREHOUSES, INVENTORY, PURCHASE_ORDERS, SHIPMENTS, STORE_SALES | ~1,100 |
| Semi-structured | SHIPMENT_UPDATES, IOT_SENSOR_LOGS, DELIVERY_TRACKING_EVENTS | ~160 |
| Unstructured | SUPPLIER_EMAILS, LOGISTICS_INCIDENT_REPORTS, WAREHOUSE_INSPECTION_NOTES | 63 |

**Cortex objects:** 3 Search Services + 1 Semantic View + 1 Agent + 1 MCP Server

</details>

<details>
<summary><b>Fabric Lakehouse Data</b> — via Data Agent (80 rows)</summary>

| Table | Rows | Key Data |
|---|---|---|
| `transportation_freight_costs` | 40 | 6 carriers (UPS, DHL, FedEx, Maersk, XPO, USPS), invoice statuses, detention charges, damage claims |
| `customer_returns_complaints` | 40 | Linked to Snowflake suppliers/products, severity levels, SLA tracking |

</details>

---

## Troubleshooting

| Issue | Solution |
|---|---|
| MCP tool call times out | Snowflake MCP non-streaming timeout is 50s. Simplify queries or increase warehouse size |
| OAuth token errors | Verify client ID/secret. Ensure security integration is `ENABLED = TRUE` |
| Fabric Data Agent returns no results | Ensure tables are loaded as Delta tables (not just in Files folder). Refresh explorer |
| Model not available in region | Use East US 2 or Sweden Central for widest GPT-5.x availability |
| Agent doesn't route correctly | Review instructions in `10_foundry_instructions.md` — ensure routing rules are explicit |
| MCP hostname issues | Use hyphens (`-`) not underscores (`_`) in hostnames for MCP server connections |
| Cortex Search returns no results | Wait 2-3 minutes after creation for indexing to complete |
| Semi-structured queries fail | Use path notation for VARIANT columns (e.g., `SENSOR_PAYLOAD:temperature`) |
| Semantic view errors | Check that all 7 structured tables have data before creating the semantic view |
| IP not allowed (error 390420) | Add IP to Snowflake network policy: `ALTER NETWORK POLICY ... SET ALLOWED_IP_LIST = (...)` |

---

## Cleanup

<details>
<summary><b>Remove all resources</b></summary>

**Snowflake:**
```sql
DROP DATABASE IF EXISTS SUPPLY_CHAIN_DEMO;
DROP WAREHOUSE IF EXISTS SUPPLY_CHAIN_WH;
DROP SECURITY INTEGRATION IF EXISTS foundry_mcp_oauth;
```

**Fabric:**
1. Delete the `SupplyChainFreightReturnsAgent` data agent
2. Delete the `SupplyChainLakehouse` lakehouse
3. Delete the `SupplyChainDemo` workspace

**AI Foundry:**
1. Delete the `SupplyChainOrchestrator` agent
2. Delete the model deployment
3. Delete the project (if no longer needed)

</details>

---

## References

| Resource | Link |
|---|---|
| Snowflake Managed MCP Server | [docs.snowflake.com](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp) |
| Snowflake QuickStart: AI Foundry + MCP | [snowflake.com](https://www.snowflake.com/en/developers/guides/getting-started-with-ai-foundry-and-the-snowflake-managed-mcp/) |
| Microsoft Foundry Agent Quickstart | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/foundry/quickstarts/get-started-code) |
| Foundry: Connect to MCP Servers | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/model-context-protocol) |
| Create a Fabric Data Agent | [learn.microsoft.com](https://learn.microsoft.com/en-us/fabric/data-science/how-to-create-data-agent) |
| Create a Fabric Lakehouse | [learn.microsoft.com](https://learn.microsoft.com/en-us/fabric/data-engineering/tutorial-build-lakehouse) |

---

<div align="center">

**Built with** [Snowflake](https://www.snowflake.com) + [Microsoft AI Foundry](https://ai.azure.com) + [OpenAI](https://openai.com) + [MCP](https://modelcontextprotocol.io)

[![GitHub stars](https://img.shields.io/github/stars/curious-bigcat/snowflake-cortex-mcp-foundry-fabric-multi-agent?style=social)](https://github.com/curious-bigcat/snowflake-cortex-mcp-foundry-fabric-multi-agent)

</div>
