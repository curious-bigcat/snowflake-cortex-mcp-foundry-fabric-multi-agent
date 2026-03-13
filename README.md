<div align="center">

# Multi-Agent Orchestrator

### Snowflake Cortex MCP + Microsoft AI Foundry + Fabric Data Agents

[![Snowflake](https://img.shields.io/badge/Snowflake-29B5E8?style=for-the-badge&logo=snowflake&logoColor=white)](https://www.snowflake.com)
[![Microsoft Azure](https://img.shields.io/badge/Azure_AI_Foundry-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)](https://ai.azure.com)
[![Microsoft Fabric](https://img.shields.io/badge/Microsoft_Fabric-742774?style=for-the-badge&logo=microsoftazure&logoColor=white)](https://fabric.microsoft.com)

[![MCP Protocol](https://img.shields.io/badge/MCP-Model_Context_Protocol-00ADD8?style=flat-square)](https://modelcontextprotocol.io)
[![Cortex Agent](https://img.shields.io/badge/Cortex-Agent_|_Analyst_|_Search-29B5E8?style=flat-square)](https://docs.snowflake.com/en/user-guide/snowflake-cortex)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](https://opensource.org/licenses/MIT)

---

**A multi-agent architecture that routes natural language queries across
Snowflake and Microsoft Fabric through a unified AI Foundry orchestrator.**

[Full Setup Guide](setup/00_README.md)

</div>

---

## Architecture

```
                    ┌─────────────────────────┐
                    │   User (Teams / Web /    │
                    │   Copilot / API)         │
                    └────────┬────────────────┘
                             │
                    ┌────────▼────────────────┐
                    │  Microsoft AI Foundry    │
                    │  Orchestrator Agent      │
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
        │ - Analyst    │    │ - shipments       │
        │ - Search x2  │    │ - store_sales     │
        └──────────────┘    │ - carriers        │
                            │ - delivery_status │
                            │ - incidents       │
                            │ - routes          │
                            └──────────────────┘
```

## How It Works

The orchestrator uses two tools, each with different data. It picks the right tool based on the question.

| Tool | Platform | Data |
|---|---|---|
| **snowflake-mcp-supplychain** | Snowflake | Suppliers, purchase orders, inventory, warehouses, supplier emails, inspection reports |
| **sc_fabric_agent** | Microsoft Fabric | Store sales, shipments, carriers, delivery status, logistics incidents |

For questions that span both domains, the orchestrator calls both tools and combines the results.

## Repository Structure

```
.
├── README.md                          # This file
├── fabric_csv/                        # CSV files for Fabric Lakehouse
│   ├── carriers.csv
│   ├── delivery_status.csv
│   ├── logistics_incidents_fabric.csv
│   ├── shipments.csv
│   ├── store_sales.csv
│   └── transportation_routes.csv
└── setup/
    ├── 00_README.md                   # Full setup guide (start here)
    ├── 01–09_*.sql                    # Snowflake setup scripts
    └── 10_foundry_instructions.md     # AI Foundry orchestrator instructions
```

## Quick Start

| Phase | Platform | What You Build |
|:---:|---|---|
| **1** | Snowflake | Database, tables, data, Cortex Search, Semantic View, Agent, MCP Server |
| **2** | Microsoft Fabric | Workspace, Lakehouse, Delta tables, Fabric Data Agent |
| **3** | AI Foundry | Orchestrator agent with MCP + Fabric tool wiring |
| **4** | All | End-to-end testing |

See the **[Full Setup Guide](setup/00_README.md)** for step-by-step instructions.

## Sample Questions

**Snowflake only:**
- "What products are at critical stockout risk?"
- "Which suppliers have reliability scores below 0.7?"
- "Search supplier emails about quality complaints"

**Fabric only:**
- "Which products have the highest sales?"
- "Which carriers have the most late deliveries?"
- "Show me all high-severity logistics incidents"

**Cross-platform (both tools):**
- "Which high-sales products are at stockout risk?"
- "Which suppliers have both high delivery delays AND low reliability scores?"

---

<div align="center">

### Built With

<a href="https://www.snowflake.com"><img src="https://cdn.simpleicons.org/snowflake/29B5E8" width="40" alt="Snowflake"></a>
&nbsp;&nbsp;&nbsp;
<a href="https://ai.azure.com"><img src="https://img.shields.io/badge/Microsoft-0078D4?style=for-the-badge&logo=microsoft&logoColor=white" alt="Microsoft Azure"></a>
&nbsp;&nbsp;&nbsp;
<a href="https://modelcontextprotocol.io"><img src="https://img.shields.io/badge/MCP-Protocol-00ADD8?style=flat-square" alt="MCP"></a>

---

[![GitHub stars](https://img.shields.io/github/stars/curious-bigcat/snowflake-cortex-mcp-foundry-fabric-multi-agent?style=social)](https://github.com/curious-bigcat/snowflake-cortex-mcp-foundry-fabric-multi-agent)

</div>
