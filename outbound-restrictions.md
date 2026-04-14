---
title: Outbound Restrictions — BYO VNet vs Managed VNet
---

# Outbound Restrictions — BYO VNet vs Managed VNet

## Comparison

| | Managed VNet (Preview) | BYO VNet (GA) |
|---|---|---|
| **Managed by** | Microsoft | Customer |
| **Outbound control** | 3 preset modes | Full control (firewall + NSG + UDR) |
| **Firewall** | Auto-provisioned, no BYO | Customer-managed |
| **FQDN rules** | Ports 80/443 only | Any port |
| **Private MCP** | ❌ | ✅ |
| **Outbound logging** | ❌ | ✅ |
| **Evaluations security** | ❌ | ✅ |
| **On-premises access** | Application Gateway only | VPN / ExpressRoute / Bastion |
| **Deployment** | Bicep only | Portal, Bicep, or Terraform |
| **Reversible** | No | Yes |

---

## Managed VNet Outbound Modes

| Mode | Behavior |
|---|---|
| **Allow internet outbound** | All outbound permitted. Irreversible. |
| **Allow only approved outbound** | Restricted to service tags, managed private endpoints, FQDN rules (80/443). Managed Azure Firewall auto-created. |
| **Disabled** | No managed VNet. |

### Rule types (Approved Outbound only)

| Type | Details |
|---|---|
| Private Endpoint | To Azure services (Storage, Cosmos DB, AI Search, Key Vault). Via `az rest` CLI. |
| Service Tag | e.g. `AzureActiveDirectory`. Some auto-added. |
| FQDN | Ports 80/443 only. Triggers Azure Firewall creation. |

### Constraints

- No BYO firewall, no firewall sharing, no outbound logging
- Rules via CLI only (`az rest`)
- Cosmos DB and AI Search private endpoints not auto-created
- **Private MCP not supported** (Limitation #8)
- Irreversible once enabled

---

## BYO VNet Outbound Configuration

You configure:

1. **NSGs** — restrict outbound by IP/service tag/port
2. **UDRs** — force-tunnel through your firewall
3. **Firewall rules** — FQDN and network-layer filtering
4. **Private endpoints** — to Storage, Cosmos DB, AI Search, Key Vault
5. **Private DNS zones** — linked to your VNet

Supports hub-and-spoke architecture with centralized firewall.

---

**Bottom line:** BYO VNet gives full outbound control including private MCP and logging. Managed VNet is simpler but has gaps for MCP tools and observability.

---

## References

- [Managed virtual network for Foundry](https://learn.microsoft.com/azure/foundry/how-to/managed-virtual-network)
- [Network isolation for Foundry](https://learn.microsoft.com/azure/foundry/how-to/configure-private-link)
- [VNet for Agent Service](https://learn.microsoft.com/azure/ai-services/agents/how-to/virtual-networks)

*Last updated: April 14, 2026*
