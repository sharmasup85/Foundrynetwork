---
title: Foundry Account Outbound Restrictions — BYO VNet vs Managed VNet
---

# Foundry Account Outbound Restrictions — BYO VNet vs Managed VNet

Based on the [Microsoft Learn documentation](https://learn.microsoft.com/azure/foundry/how-to/managed-virtual-network) and [configure-private-link](https://learn.microsoft.com/azure/foundry/how-to/configure-private-link) pages.

---

## Managed Virtual Network (Preview) — Outbound Restrictions

Microsoft provisions and manages the VNet. You choose one of these isolation modes:

| Outbound Mode | Behavior |
|---|---|
| **Allow internet outbound** | All outbound traffic permitted. Cannot downgrade to Disabled after enabling. |
| **Allow only approved outbound** | Outbound restricted to: **service tags**, **managed private endpoints**, and optional **FQDN rules** (ports 80/443 only) enforced via a managed Azure Firewall. Cannot upgrade to Allow internet outbound after setting this. |
| **Disabled** | No managed VNet. Use public outbound or supply your own VNet. |

### Outbound rule types (Allow Only Approved Outbound)

| Rule Type | Details |
|---|---|
| **Private Endpoint** | Managed private endpoints to Azure services (Storage, Cosmos DB, AI Search, Key Vault, etc.). Created via `az rest` CLI. |
| **Service Tag** | Azure service tags (e.g., `AzureActiveDirectory`) — some are auto-added. |
| **FQDN** | Specific FQDNs allowed on ports 80/443 only. Triggers creation of a managed Azure Firewall (Standard or Basic SKU). |

### Key constraints

- Cannot bring your own firewall — a managed firewall is auto-provisioned per Foundry account
- No firewall reuse across accounts
- No logging of outbound traffic
- Outbound rules managed via **Azure CLI only** (`az rest`)
- Private endpoints to Cosmos DB and AI Search are **not** auto-created (only Storage is)
- **MCP tools private FQDNs are NOT honored** (Limitation #8) — must use public MCP tools
- Evaluations compute security not supported
- Irreversible — cannot disable after enabling

---

## Custom (BYO) VNet with Network Injection (GA) — Outbound Restrictions

You bring your own VNet and control outbound traffic entirely.

| Aspect | Detail |
|---|---|
| **Outbound control** | Fully customer-managed via your own Azure Firewall, NSGs, UDRs, NVAs |
| **Network architecture** | Hub-and-spoke supported (firewall in hub, Foundry in spoke, peered) |
| **Subnet requirement** | Delegated to `Microsoft.App/environments`, minimum /27 |
| **Private endpoints** | Customer creates private endpoints to Storage, Cosmos DB, AI Search separately |
| **MCP tools** | **Private MCP fully supported** — traffic flows through your VNet subnet |
| **Egress inspection** | Full control — route through Azure Firewall or NVA for inspection and logging |

### What you must configure for outbound restrictions

1. **NSGs** on subnets to restrict outbound by destination IP/service tag/port
2. **UDRs** to force-tunnel outbound traffic through your firewall
3. **Azure Firewall / NVA rules** for application-layer (FQDN) and network-layer filtering
4. **Private endpoints** to all dependent Azure PaaS services (Storage, Cosmos DB, AI Search, Key Vault)
5. **Private DNS zones** linked to your VNet for proper FQDN → private IP resolution

---

## Side-by-Side Comparison

| Aspect | Managed VNet (Preview) | BYO VNet (GA) |
|---|---|---|
| **Who manages the VNet** | Microsoft | Customer |
| **Outbound modes** | 3 modes (internet / approved only / disabled) | Fully custom via firewall + NSG + UDR |
| **Firewall** | Managed (cannot BYO) | Customer-managed (full control) |
| **FQDN rules** | Ports 80/443 only | Any port via your firewall |
| **Private MCP tools** | ❌ Not supported | ✅ Supported |
| **Outbound traffic logging** | ❌ Not supported | ✅ Supported (your firewall logs) |
| **Evaluations compute security** | ❌ Not supported | ✅ Supported |
| **On-premises access** | Via Application Gateway only | VPN Gateway / ExpressRoute / Bastion |
| **Complexity** | Low (Microsoft manages) | Higher (subnet delegation, DNS, firewall config) |
| **Reversibility** | Irreversible once enabled | Flexible |
| **Deployment** | Bicep template only | Portal, Bicep, or Terraform |
| **Status** | Public Preview (no SLA) | GA |

---

**Bottom line:** BYO VNet gives full outbound control including private MCP support and traffic logging, at the cost of more setup complexity. Managed VNet is simpler but has notable gaps — especially for MCP tools and outbound observability.

---

## References

- [Configure managed virtual network for Microsoft Foundry projects](https://learn.microsoft.com/azure/foundry/how-to/managed-virtual-network)
- [How to configure network isolation for Microsoft Foundry](https://learn.microsoft.com/azure/foundry/how-to/configure-private-link)
- [How to use a virtual network with the Azure AI Agent Service](https://learn.microsoft.com/azure/ai-services/agents/how-to/virtual-networks)

*Last updated: April 14, 2026*
