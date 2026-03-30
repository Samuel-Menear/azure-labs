# Lab 05 – Implement Intersite Connectivity

## Overview
This lab explores communication between virtual networks through virtual network peering,
Network Watcher diagnostics, PowerShell connectivity testing, and custom user-defined routes.
It simulates a common enterprise scenario where core IT services need to communicate securely
with a separate manufacturing network — a pattern applicable to production/development
separation or subsidiary isolation.

> **Note:** VMs deployed to **West Europe, Zone 3** rather than East US as specified in the
> lab default — region and zone selection has no impact on the peering, routing, or
> connectivity behaviour demonstrated. VM size changed from Standard_D2s_v3 to
> **Standard_B2ls_v2** for cost efficiency. Networking behaviour is identical at either size.

## Architecture

![Architecture diagram showing CoreServicesVnet and ManufacturingVnet connected via peering with a perimeter subnet and custom route](diagram.png)

## Resources Deployed

### Virtual Machines and Virtual Networks

| Resource | Name | Value |
|---|---|---|
| Resource Group | az104-rg5 | — |
| VM | CoreServicesVM | West Europe, Zone 3, B2ls_v2 |
| VNet | CoreServicesVnet | 10.0.0.0/16 |
| Subnet | Core | 10.0.0.0/24 |
| Subnet | perimeter | 10.0.1.0/24 |
| VM | ManufacturingVM | West Europe, Zone 3, B2ls_v2 |
| VNet | ManufacturingVnet | 172.16.0.0/16 |
| Subnet | Manufacturing | 172.16.0.0/24 |

### Peerings

| Peering Link Name | Direction |
|---|---|
| CoreServicesVnet-to-ManufacturingVnet | Core → Manufacturing |
| ManufacturingVnet-to-CoreServicesVnet | Manufacturing → Core |

### Custom Routing

| Resource | Name | Value |
|---|---|---|
| Route Table | rt-CoreServices | Associated with Core subnet |
| Route | PerimetertoCore | 10.0.0.0/16 via virtual appliance 10.0.1.7 |

---

## Key Concepts Demonstrated

### VNets do not communicate by default
Before peering was configured, Network Watcher's Connection Troubleshoot confirmed the two
VMs were unreachable from each other despite being in the same subscription and resource
group. This is the expected default — VNets are isolated boundaries and connectivity must
be explicitly enabled.

### Virtual network peering
Peering was configured bidirectionally from CoreServicesVnet, which automatically creates
both peering links in a single operation. Once connected, peered VNets appear as one for
routing purposes and traffic travels across the Microsoft backbone — not the public internet.
The peering status must show **Connected** on both sides before traffic flows.

### Network Watcher — Connection Troubleshoot
Network Watcher's Connection Troubleshoot tool was used to verify connectivity pre and
post peering. It tests reachability between two resources and reports the path status,
hop-by-hop where possible. This is the primary diagnostic tool for Azure network
connectivity issues and a core AZ-104 exam topic.

### PowerShell connectivity testing via Run Command
Post-peering connectivity was verified from ManufacturingVM using the **Run Command**
feature in the portal — no RDP or public IP required. The command used:

```powershell
Test-NetConnection <CoreServicesVM private IP> -port 3389
```

Run Command executes scripts directly on a VM through the Azure agent, making it useful
for diagnostics when public access is intentionally blocked. A successful result confirms
peering is functioning at the network layer.

### User-defined routes (UDR) and virtual network appliances
By default, Azure uses system routes to handle traffic between subnets. Task 6 overrides
this by creating a custom route table (`rt-CoreServices`) that directs traffic destined
for the core services VNet (10.0.0.0/16) via a **virtual appliance** at 10.0.1.7 in the
perimeter subnet.

This is the standard pattern for inserting a Network Virtual Appliance (NVA) — such as
Azure Firewall, a third-party firewall VM, or an IDS — into the traffic path between
network segments. The NVA address (10.0.1.7) is a placeholder representing a future
appliance to be deployed in the perimeter subnet.

Key UDR behaviour worth knowing:
- Lower-specificity system routes are overridden by matching user-defined routes
- Route tables must be **associated with a subnet** to take effect — creating the table
  alone does nothing
- **Propagate gateway routes: No** was set, preventing on-premises routes from a VPN
  gateway being automatically pushed to this subnet

---

## Infrastructure as Code

Export the resource group template after completing the lab for reuse:

```bash
az group export --name az104-rg5 > template.json
```

To redeploy from scratch:

```bash
az group create --name az104-rg5 --location westeurope

az deployment group create \
  --resource-group az104-rg5 \
  --template-file template.json \
  --parameters @parameters.json
```

To convert ARM template to Bicep:

```bash
az bicep decompile --file template.json
```

## Cleanup

```bash
az group delete --name az104-rg5 --yes --no-wait
```

---

## Lab Source
[AZ-104 Lab 05 – Implement Intersite Connectivity](https://microsoftlearning.github.io/AZ-104-MicrosoftAzureAdministrator/Instructions/Labs/LAB_05-Implement_Intersite_Connectivity.html)
