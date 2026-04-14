# Foundry Network Isolation & MCP Tool Findings

All the learnings of Foundry networking, outbound restrictions, and MCP tool private FQDN behavior.

## Key Finding: MCP Tool Private FQDNs and Foundry Outbound Restrictions

MCP tool private FQDNs behave differently depending on which network isolation approach is used.

### Managed Virtual Network (Preview) — Private MCP NOT Supported

Per [Microsoft Learn - Managed VNet Limitations #8](https://learn.microsoft.com/azure/foundry/how-to/managed-virtual-network#limitations):

> **End-to-end network isolation for Agent MCP tools with managed virtual network is currently not supported.** Use public MCP tools with managed network isolation Foundry.

Private FQDNs for MCP tool servers are **not honored** by the Foundry account's outbound restrictions when using the managed VNet. MCP tool traffic does not flow through the managed network's private endpoint or FQDN outbound rules.

### Custom (BYO) VNet with Network Injection (GA) — Private MCP Supported

Per [Microsoft Learn - Agent tools with network isolation](https://learn.microsoft.com/azure/foundry/how-to/configure-private-link#agent-tools-with-network-isolation):

Private MCP is fully supported with BYO VNet. Traffic flows through your VNet subnet.

| Isolation Approach | Private MCP Support | Status |
|---|---|---|
| Managed Virtual Network (preview) | **Not supported** — MCP tools must be public | Preview |
| Custom (BYO) VNet with network injection | **Supported** — traffic flows through VNet subnet | GA |

---

## Foundry Account Outbound Restriction Modes (Managed VNet)

| Outbound Mode | Description |
|---|---|
| Allow internet outbound | All outbound traffic to the internet is permitted. |
| Allow only approved outbound | Outbound restricted using service tags, managed private endpoints, and optional FQDN rules (ports 80/443 only) enforced via managed Azure Firewall. |
| Disabled | No managed VNet; use public outbound or supply your own VNet. |

### Key Constraints Under "Allow Only Approved Outbound"

- FQDN outbound rules support **only ports 80 and 443**
- A managed Azure Firewall is auto-provisioned (cannot bring your own)
- Each Foundry account gets its own firewall (no sharing)
- Outbound rules are created via **Azure CLI only** (`az rest`)
- Private endpoints must be created to Cosmos DB, Storage, and AI Search for Standard BYO agents

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
