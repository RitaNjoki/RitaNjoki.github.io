---
title: Deploying and Testing Azure Firewall - A Cloud Network Security Lab
date: 2026-03-28 14:00:00 +0300
categories: [Cloud, Network-Security, Azure, Firewall, Lab]
tags: [azure, firewall, network-rules, application-rules, cloud-security]
pin: true
image:
  path: /assets/azure-firewall-lab.jpg
  alt: Azure Firewall deployment and testing in a secure lab environment

---

## Why I Built This Lab

As a Computer Science student focused on **Cloud and Network Security**, I needed hands-on experience with enterprise-grade firewall configurations. Azure Firewall presented the perfect opportunity to understand how modern cloud platforms enforce traffic control, implement security policies, and validate network access rules in real-world scenarios.

![Azure Firewall Architecture](/assets/azure-firewall-architecture.jpg)
_A secured network with Azure Firewall protecting workload subnets_

## Lab Objectives

My goal was to:
1. **Deploy Azure Firewall** in a virtual network environment
2. **Configure application rules** to allow specific FQDN-based access
3. **Establish network rules** for DNS traffic on port 53
4. **Test firewall enforcement** by validating allowed vs. blocked traffic
5. **Understand the relationship** between routing, rules, and security policies

## Architecture Overview

I set up a three-subnet environment with the following components:

- **Azure Virtual Network** (`Test-FW-VN`) - The core network boundary
- **Firewall Subnet** - Hosts the Azure Firewall appliance
- **Workload Subnet** (`Workload-SN`) - Contains the `Srv-Work` Windows Server 2016 VM
- **Jump Subnet** - Contains the `Srv-Jump` bastion host for secure access

### Key Configuration Details

**Firewall Specifications:**
- SKU: Standard
- Rule Type: Classic (Application & Network rules)
- Management: Public IP (`TEST-FW-PIP`)
- Private IP: `10.0.1.4` (noted for routing)

## Task Breakdown

### Task 1: Environment Deployment
I deployed the lab infrastructure using an ARM template, which provisioned a Windows Server 2016 Datacenter VM with all necessary networking components. This automated approach ensured consistency and allowed me to focus on firewall configuration rather than infrastructure setup.

### Task 2: Azure Firewall Deployment
Deploying the firewall was straightforward through the Azure Portal. The key insight here was understanding that Azure Firewall is a **managed service** — not a traditional virtual appliance — but we still reference it as a "virtual appliance" in routing tables for compatibility.

### Task 3: Creating a Default Route
This was critical for the lab's success. I created a route table (`Firewall-route`) and associated it with the `Workload-SN` subnet. The default route (0.0.0.0/0) directed all outbound traffic through the firewall's private IP address, ensuring that every packet leaving the workload subnet passed through our security gateway.

**Route Configuration:**
- Route Name: `FW-DG` (Firewall Default Gateway)
- Destination: `0.0.0.0/0` (all traffic)
- Next Hop Type: Virtual Appliance
- Next Hop Address: Firewall Private IP

### Task 4: Application Rules
I configured an application rule collection (`App-Coll01`) to allow HTTP and HTTPS traffic to `www.bing.com` from the workload subnet (10.0.2.0/24).

**Rule Details:**
- Protocol/Ports: `http:80, https:443`
- Target FQDN: `www.bing.com`
- Action: Allow
- Priority: 200

This rule demonstrates application-layer filtering, where the firewall inspects DNS names rather than just IP addresses.

### Task 5: Network Rules
I created a network rule collection (`Net-Coll01`) allowing UDP traffic on port 53 (DNS) from the workload subnet to two public DNS servers:

**Rule Details:**
- Protocol: UDP
- Source: `10.0.2.0/24`
- Destinations: `209.244.0.3`, `209.244.0.4` (public DNS servers)
- Port: 53

This rule was essential because the firewall blocks DNS by default, and without it, domain name resolution would fail, breaking the application rule.

### Task 6: DNS Configuration
I configured the `Srv-Work` VM's network interface to use the same DNS servers (209.244.0.3 and 209.244.0.4) referenced in the network rule. This ensured DNS queries would follow the firewall's network rule path.

### Task 7: Testing & Validation
The moment of truth — testing whether the firewall actually worked as designed.

**Allowed Traffic (bing.com):**
```
✓ Successfully accessed https://www.bing.com
Status: HTTP 200
Firewall Action: ALLOW (matched App-Coll01 rule)
```

**Blocked Traffic (microsoft.com):**
```
✗ Access attempt to http://www.microsoft.com
Status: Firewall Deny Message
Error: "HTTP request from 10.0.2.4:xxxxx to microsoft.com:80. Action: Deny. No rule matched. Proceeding with default action."
Firewall Action: DENY (no matching rule)
```

This demonstrated the principle of **least privilege** — only explicitly allowed traffic passes through; everything else is blocked by default.

## Key Learnings

### 1. **Routing & Firewall Integration**
The firewall is only effective if traffic actually flows through it. The default route was critical — without it, the firewall would sit idle while traffic bypassed it entirely.

### 2. **Application vs. Network Rules**
- **Application Rules** work at Layer 7 (DNS/FQDN level) — more flexible but slightly more complex
- **Network Rules** work at Layer 4 (IP/Port level) — simpler but less granular
- Both are necessary for complete traffic control

### 3. **DNS Dependency**
Application rules rely on DNS resolution. If DNS fails, the firewall can't resolve FQDNs. The network rule for DNS (port 53) was therefore a prerequisite, not an afterthought.

### 4. **Default Deny Principle**
Azure Firewall implements implicit deny — if no rule matches, traffic is dropped. This is the opposite of many on-premises firewalls and requires a mindset shift toward building allowlists rather than blocklists.

### 5. **Managed Service Benefits**
As a managed service, Azure Firewall handles patching, scaling, and high availability automatically. This is different from managing a firewall appliance on-premises or in a VM.

## Real-World Applications

This lab directly translates to enterprise scenarios:
- **Zero Trust Architecture** — Only allow necessary traffic; block by default
- **Egress Filtering** — Control what leaves your network, not just what enters
- **FQDN-based Policies** — Common in regulated industries where IP addresses change frequently
- **Multi-region Deployments** — Azure Firewall scales across regions seamlessly

## Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| Deployment region errors (`internalservererror`) | Change ARM template region from `eastus` to `centralus` |
| DNS resolution failing | Add explicit network rule for DNS (port 53) |
| Application rule not matching | Verify target FQDN is lowercase; check protocol/port combinations |

## Conclusion

This lab reinforced that **firewalls are not just perimeter security** — they're critical infrastructure components that enforce policy, control access, and protect against data exfiltration. By seeing firsthand how application and network rules interact, how routing drives traffic through the firewall, and how testing validates our security posture, I gained practical knowledge that goes far beyond theoretical understanding.

The successful blocking of microsoft.com while allowing bing.com proved that our security policy was working as intended. This is the foundation of enterprise network security.

---

**Course:** Cloud and Network Security (C1-2026)  
**Student:** Rita Njoki (CS-CNS11-26074)  
**Lab Date:** Saturday, March 28, 2026  
**Status:** ✓ Completed & Validated

*Next steps: Explore advanced firewall features like threat intelligence integration, custom rules, and multi-firewall deployments.*