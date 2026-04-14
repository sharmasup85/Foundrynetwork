# Foundry Network Isolation & MCP Tools

## Pages

- [Outbound Restrictions — BYO VNet vs Managed VNet](outbound-restrictions.md)

---

## MCP Tool Private FQDNs — Known Limitation

Per [Managed VNet Limitations #8](https://learn.microsoft.com/azure/foundry/how-to/managed-virtual-network#limitations):

> **End-to-end network isolation for Agent MCP tools with managed virtual network is currently not supported.**

| Approach | Private MCP | Status |
|---|---|---|
| Managed VNet (preview) | ❌ Not supported | Preview |
| BYO VNet + network injection | ✅ Supported — flows through your subnet | GA |

If you previously saw private MCP FQDNs working, you were likely using **BYO VNet** (GA). The **managed VNet** (preview) uses a different architecture that does not yet support MCP private connectivity.

---

## Workaround: Use BYO VNet for Private MCP

1. Standard Agent setup with BYO resources (Storage, Cosmos DB, AI Search)
2. Public network access **Disabled** on the Foundry resource
3. VNet injection subnet delegated to `Microsoft.App/environments` (min /27)
4. Dedicated MCP subnet (same delegation)
5. MCP server on Azure Container Apps with **internal-only ingress**
6. Deploy via [19-hybrid-private-resources-agent-setup](https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/19-hybrid-private-resources-agent-setup)

---

## Troubleshooting (BYO VNet)

| Check | What to verify |
|---|---|
| DNS | Private DNS zones linked to VNet; FQDN resolves to private IP |
| NSGs | Outbound to MCP server IP on port 443 allowed |
| Firewall | Agent → MCP server traffic permitted |
| RBAC | Agent managed identity has required roles |
| Ingress | Container App set to `internal` not `external` |

---

## Agent Tool Support (Network-Isolated)

| Tool | Status | Traffic |
|---|---|---|
| MCP Tool (Private) | ✅ BYO VNet only | Your subnet |
| AI Search | ✅ | Private endpoint |
| Code Interpreter | ✅ | Microsoft backbone |
| Function Calling | ✅ | Microsoft backbone |
| Bing / Websearch / SharePoint | ✅ | Public endpoint |
| Foundry IQ | ✅ | Via MCP |
| File Search, Logic Apps, OpenAPI, Azure Functions, Fabric | ❌ | Under development |

---

## References

- [Managed virtual network for Foundry](https://learn.microsoft.com/azure/foundry/how-to/managed-virtual-network)
- [Network isolation for Foundry](https://learn.microsoft.com/azure/foundry/how-to/configure-private-link)
- [MCP tool integration](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/model-context-protocol)
- [19-hybrid-private-resources-agent-setup](https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/19-hybrid-private-resources-agent-setup)

*Last updated: April 14, 2026*
