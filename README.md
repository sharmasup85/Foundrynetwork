# Foundry Network Isolation & MCP Tool Findings

All the learnings of Foundry networking, outbound restrictions, and MCP tool private FQDN behavior.

---

## Investigation Process

Searched Microsoft Learn documentation for the latest information on Foundry account outbound restrictions and MCP tool networking behavior using the following queries:

1. **Microsoft Docs Search** — `Azure AI Foundry account outbound network restrictions managed virtual network`
2. **Microsoft Docs Search** — `Azure AI Foundry MCP tools private endpoints FQDN outbound rules networking`

Full pages fetched and reviewed:

- [Configure managed virtual network for Microsoft Foundry projects](https://learn.microsoft.com/azure/foundry/how-to/managed-virtual-network)
- [How to configure network isolation for Microsoft Foundry](https://learn.microsoft.com/azure/foundry/how-to/configure-private-link)
- [Connect agents to Model Context Protocol servers](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/model-context-protocol)

---

## Foundry Account Outbound Restrictions and MCP Tool Private FQDN Behavior

### The Observation Is Correct — This Is a Known, Documented Limitation

Per the current [Microsoft Learn documentation for managed virtual networks](https://learn.microsoft.com/azure/foundry/how-to/managed-virtual-network#limitations), **Limitation #8** explicitly states:

> **End-to-end network isolation for Agent MCP tools with managed virtual network is currently not supported.** Use public MCP tools with managed network isolation Foundry.

This means that when you use the **managed virtual network** (preview) isolation mode on a Foundry account, private FQDNs for MCP tool servers are **not honored** by the Foundry account's outbound restrictions. MCP tool traffic does not flow through the managed network's private endpoint or FQDN outbound rules.

### The Two Isolation Approaches Behave Differently for MCP

| Isolation Approach | Private MCP Support | Doc Reference |
|---|---|---|
| **Managed Virtual Network** (preview) | **Not supported** — MCP tools must be public | [Managed VNet Limitations #8](https://learn.microsoft.com/azure/foundry/how-to/managed-virtual-network#limitations) |
| **Custom (BYO) VNet** with network injection (GA) | **Supported** — Private MCP flows through your VNet subnet | [Agent tools with network isolation](https://learn.microsoft.com/azure/foundry/how-to/configure-private-link#agent-tools-with-network-isolation) |

The [configure-private-link](https://learn.microsoft.com/azure/foundry/how-to/configure-private-link#agent-tools-with-network-isolation) page confirms that **MCP Tool (Private MCP)** is supported in the **BYO VNet** model with traffic flowing "through your VNet subnet." For private MCP, you must deploy your MCP server on Azure Container Apps with internal-only ingress on a dedicated MCP subnet delegated to `Microsoft.App/environments`.

### Foundry Account Outbound Restriction Modes (Managed VNet)

| Outbound Mode | Description |
|---|---|
| **Allow internet outbound** | All outbound traffic to the internet is permitted. |
| **Allow only approved outbound** | Outbound restricted using **service tags**, **managed private endpoints**, and optional **FQDN rules** (ports 80/443 only) enforced via a managed Azure Firewall. |
| **Disabled** | No managed VNet; use public outbound or supply your own VNet. |

Key restrictions under "Allow Only Approved Outbound":
- FQDN outbound rules support **only ports 80 and 443**
- A managed Azure Firewall is auto-provisioned (cannot bring your own)
- Each Foundry account gets its own firewall (no sharing)
- Outbound rules are created via **Azure CLI only** (`az rest`)
- Private endpoints must be created to Cosmos DB, Storage, and AI Search for Standard BYO agents

### Has This Changed?

Yes — the **managed virtual network** feature is currently in **public preview** and the MCP tool limitation is a known gap in this preview. If you previously tested private MCP FQDNs working with outbound restrictions, you were likely using the **custom (BYO) VNet with network injection** approach (GA), where private MCP is fully supported through your own subnet. The managed VNet preview introduced a **different** networking architecture (Microsoft-managed VNet with managed private endpoints) that does **not yet** support MCP tool private connectivity.

### Workaround

If you need private MCP with network isolation today, use the **custom (BYO) VNet** approach with:
- The [19-hybrid-private-resources-agent-setup](https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/19-hybrid-private-resources-agent-setup) Bicep template
- A dedicated MCP subnet delegated to `Microsoft.App/environments`
- Your MCP server deployed on Azure Container Apps with **internal-only ingress**

---

## BYO VNet Requirements for Private MCP

1. **Standard Agent setup** with BYO resources (Storage, Cosmos DB, AI Search) — not Basic Agent
2. **Public network access set to Disabled** on the Foundry resource
3. **VNet injection subnet** delegated to `Microsoft.App/environments` (minimum /27)
4. **Dedicated MCP subnet** for the private MCP server (also delegated to `Microsoft.App/environments`)
5. **Private endpoints** created separately to AI Search, Storage, and Cosmos DB
6. Use the [19-hybrid-private-resources-agent-setup](https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/19-hybrid-private-resources-agent-setup) Bicep template

### MCP Server Deployment

Deploy the MCP server on **Azure Container Apps with internal-only ingress** on the dedicated MCP subnet.

---

## Troubleshooting Private MCP on BYO VNet

If private MCP FQDNs are not being honored on BYO VNet, check:

- **DNS resolution** — Ensure private DNS zones are configured and linked to the VNet; run `nslookup` from inside the VNet to confirm the MCP server FQDN resolves to a private IP
- **NSG rules** — Verify outbound traffic from the agent subnet to the MCP server private IP on port 443 is allowed
- **Firewall rules** — If using Azure Firewall or NVA, confirm agent → MCP server traffic is permitted
- **Managed identity RBAC** — The agent's managed identity must have appropriate roles on services the MCP tools access
- **Internal-only ingress** — Confirm the Container App hosting the MCP server has ingress set to `internal` (not `external`)

---

## Agent Tool Support in Network-Isolated Environments

| Tool | Support Status | Traffic Flow |
|---|---|---|
| MCP Tool (Private MCP) | ✅ Supported (BYO VNet only) | Through your VNet subnet |
| Azure AI Search | ✅ Supported | Through private endpoint |
| Code Interpreter | ✅ Supported | Microsoft backbone network |
| Function Calling | ✅ Supported | Microsoft backbone network |
| Bing Grounding | ✅ Supported | Public endpoint |
| Websearch | ✅ Supported | Public endpoint |
| SharePoint Grounding | ✅ Supported | Public endpoint |
| Foundry IQ (preview) | ✅ Supported | Via MCP |
| Fabric Data Agent | ❌ Not supported | Under development |
| Logic Apps | ❌ Not supported | Under development |
| File Search | ❌ Not supported | Under development |
| OpenAPI tool | ❌ Not supported | Under development |
| Azure Functions | ❌ Not supported | Under development |

---

## References

- [Configure managed virtual network for Microsoft Foundry projects](https://learn.microsoft.com/azure/foundry/how-to/managed-virtual-network)
- [How to configure network isolation for Microsoft Foundry](https://learn.microsoft.com/azure/foundry/how-to/configure-private-link)
- [Connect agents to MCP servers](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/model-context-protocol)
- [19-hybrid-private-resources-agent-setup Bicep template](https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/19-hybrid-private-resources-agent-setup)
- [18-managed-virtual-network-preview Bicep template](https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/18-managed-virtual-network-preview)

*Last updated: April 14, 2026*
