# Azure Networking, NSGs & Routing Lab

## Overview

This hands-on Microsoft Azure lab demonstrates how to design and configure a virtual network with multiple subnets, associate Network Security Groups (NSGs), and examine Azure routing behavior.

The lab emphasizes network segmentation, traffic filtering, system routes, and security validation.

**Status:** Network infrastructure deployed and configuration verified in Azure Portal. VM-to-VM connectivity testing has not been performed.

## Learning Objectives

- Create and organize Azure networking resources.
- Understand VNet and subnet address planning using CIDR.
- Associate NSGs with individual subnets.
- Evaluate NSG rules using priority and matching conditions.
- Understand Azure system routes versus user-defined routes.
- Identify security gaps caused by default NSG rules.
- Document infrastructure accurately, including testing limitations.

## Azure Resources

| Resource | Name | Configuration |
|---|---|---|
| Resource group | `rg-azure-network-lab` | East US |
| Virtual network | `vent-azure-network-lab` | `10.0.0.0/16` |
| Web subnet | `subnet-web` | `10.0.1.0/24` |
| Application subnet | `subnet-app` | `10.0.2.0/24` |
| Web NSG | `nsg-web-lab` | Associated with web subnet |
| Application NSG | `nsg-app-lab` | Associated with app subnet |
| Route table | `rt-web-lab` | Associated with web subnet |

The resource group also contains a storage account and user-assigned managed identity used for a separate identity and storage security lab.

## Network Architecture

```text
Resource Group: rg-azure-network-lab
|
+-- VNet: vent-azure-network-lab
    Address Space: 10.0.0.0/16
    |
    +-- subnet-web: 10.0.1.0/24
    |   +-- NSG: nsg-web-lab
    |   +-- Route Table: rt-web-lab
    |
    +-- subnet-app: 10.0.2.0/24
        +-- NSG: nsg-app-lab
        +-- No custom route table
```

Both subnets are part of the same VNet. Azure provides system routes for communication within the VNet, subject to applicable network security controls.

No virtual machines were deployed in this lab.

## Implementation

### 1. Virtual Network and Subnets

Created the virtual network `vent-azure-network-lab` with address space `10.0.0.0/16`.

Created two non-overlapping subnets:

| Subnet | Address prefix | Usable IPv4 addresses |
|---|---|---|
| `subnet-web` | `10.0.1.0/24` | 251 |
| `subnet-app` | `10.0.2.0/24` | 251 |

Each `/24` contains 256 IPv4 addresses. Azure reserves five addresses per subnet.

Both subnets were configured with the private subnet setting enabled, meaning default outbound internet access is not provided.

### 2. Network Security Groups

Created and associated:

- `nsg-web-lab` with `subnet-web`
- `nsg-app-lab` with `subnet-app`

Configured an inbound rule in `nsg-app-lab`:

| Property | Value |
|---|---|
| Rule | `Allow-Web-To-App-8080` |
| Priority | 100 |
| Source | `10.0.1.0/24` |
| Destination | `10.0.2.0/24` |
| Protocol | TCP |
| Destination port | 8080 |
| Action | Allow |

The rule permits matching TCP 8080 traffic from the web subnet to the application subnet.

### 3. NSG Default Rules and Security Finding

The application NSG also contains Azure's default inbound rules:

| Priority | Rule | Action |
|---|---|---|
| 65000 | `AllowVnetInBound` | Allow |
| 65001 | `AllowAzureLoadBalancerInBound` | Allow |
| 65500 | `DenyAllInBound` | Deny |

**Security finding:** The custom TCP 8080 allow rule does not automatically deny other traffic between the two subnets.

For example, traffic from `subnet-web` to `subnet-app` on TCP 9090 could match the default `AllowVnetInBound` rule.

**Proposed improvement:** Add a custom inbound deny rule at priority 110 for other traffic from `10.0.1.0/24` to `10.0.2.0/24`, after the priority 100 TCP 8080 allow rule.

This proposed deny rule has not been deployed.

## Azure Routing

Created `rt-web-lab` and associated it with `subnet-web`.

The route table currently contains **zero user-defined routes**.

Azure still provides system routes, including a route for the VNet address space. Therefore, an empty custom route table does not automatically prevent communication between the two subnets.

A user-defined route (UDR) can be added when traffic must follow a customized path, such as through a network virtual appliance.

No NAT Gateway was configured for either subnet.

## Verification Results

| Verification | Result |
|---|---|
| Resource group and networking resources exist | Verified in Azure Portal |
| Both subnet address ranges | Verified |
| NSG associations for both subnets | Verified |
| Web subnet route table association | Verified |
| Application NSG custom TCP 8080 rule | Verified |
| Application NSG default inbound rules | Verified |
| No user-defined routes in `rt-web-lab` | Verified |
| VM-to-VM connectivity | Not tested |
| Effective NSG rules on a VM NIC | Not tested |
| Effective routes on a VM NIC | Not tested |
| TCP 8080 application connectivity | Not tested |

Verification was performed through Azure Portal configuration inspection rather than live packet-flow testing.

## Key Lessons Learned

1. A VNet can contain multiple non-overlapping subnets.
2. NSGs evaluate rules in ascending priority order, and the first matching rule determines the action.
3. An allow rule for one port does not automatically deny other ports.
4. Azure system routes remain available when an associated route table has no custom routes.
5. A NAT Gateway is not required for communication between subnets within the same VNet.
6. A configured network path is not proof of successful application connectivity.
7. Accurate documentation distinguishes deployed configuration from tested behavior.

## Next Steps

- Evaluate whether a more restrictive NSG policy is appropriate.
- Add sanitized Azure Portal screenshots showing subnet associations, NSG rules, and the route table.
- Deploy low-cost test workloads only after confirming subscription availability and pricing.
- Test allowed and denied network flows when workloads are available.
- Inspect effective routes and NSG rules on deployed network interfaces.
- Clean up lab resources when they are no longer needed to avoid unnecessary charges.

## Security and Cost Considerations

This lab uses private IPv4 address ranges and subnet-level NSGs. No VM, NAT Gateway, Azure Firewall, or network virtual appliance was deployed.

Azure resource configurations should be reviewed regularly for unnecessary access. Screenshots published to GitHub should exclude account email addresses, subscription identifiers, access keys, tokens, and other sensitive information.

**Lab methodology:** Plan → Build → Secure → Validate → Troubleshoot → Document.
