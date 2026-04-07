---
title: Building Secure Site-to-Site VPNs with IPsec - A Hands-On Lab Experience
date: 2026-03-07 10:00:00 +0300
categories: [Network-Security, VPN, IPsec, Cisco, Lab]
tags: [ipsec, vpn, isakmp, crypto, packet-tracer, network-encryption]
pin: true
image:
  path: /assets/ipsec.jpg
  alt: IPsec Site-to-Site VPN tunnel connecting two remote networks

---

## Why I Built This Lab

As a cybersecurity student, I needed to understand how organizations securely connect remote offices and data centers. **[IPsec](https://en.wikipedia.org/wiki/IPsec)-based site-to-site VPNs** are the backbone of enterprise network security, encrypting and authenticating all traffic between branch offices. This lab gave me hands-on experience configuring a real VPN tunnel from scratch.

![IPsec VPN Architecture](/assets/ipsec.jpg)
_A secure encrypted tunnel connecting LAN to LAN across an untrusted network_

## Lab Objectives

I aimed to:
1. **Configure ISAKMP Phase 1** (IKE) for secure tunnel establishment
2. **Configure IPsec Phase 2** for data encryption and integrity
3. **Define ACLs** to identify "interesting traffic" that triggers encryption
4. **Test tunnel activation** by sending data between remote networks
5. **Verify selective encryption** - only specified traffic gets encrypted

## Network Topology

I set up a Packet Tracer environment with:

- **R1 (Router 1)** - First site with LAN: 192.168.1.0/24
- **R3 (Router 3)** - Remote site with LAN: 192.168.3.0/24
- **PC-A & PC-C** - Client machines on respective LANs
- **Serial WAN links** - Connecting the routers across an untrusted network

The challenge: Encrypt only traffic between these two LANs while leaving other traffic unencrypted.

## VPN Configuration Breakdown

### Phase 1: ISAKMP (Internet Key Exchange)

**[ISAKMP](https://en.wikipedia.org/wiki/Internet_Key_Exchange)** is where the VPN tunnel is negotiated and established. Think of it as the "handshake" that sets up the secure channel.

**Configuration on R1:**

```
crypto isakmp policy 10
  encryption aes 256
  authentication pre-share
  group 5
  exit
crypto isakmp key vpnpa55 address 10.2.2.2
```

**Key decisions I made:**
- **AES-256 encryption** - Strong, industry-standard encryption
- **Pre-shared key (PSK)** - Simple but effective for site-to-site; both routers share secret "vpnpa55"
- **DH Group 5** - Diffie-Hellman group for key exchange (production networks use DH 14+)
- **Peer address 10.2.2.2** - R3's WAN interface IP

**R3's reciprocal configuration:**
```
crypto isakmp policy 10
  encryption aes 256
  authentication pre-share
  group 5
  exit
crypto isakmp key vpnpa55 address 10.1.1.2
```

The shared key and matching parameters allow both routers to authenticate each other.

### Phase 2: IPsec (Data Encryption)

Once the tunnel is established, IPsec encrypts the actual data using parameters negotiated in Phase 2.

**Transform Set (defines encryption & integrity):**
```
crypto ipsec transform-set VPN-SET esp-aes esp-sha-hmac
```

This means:
- **ESP (Encapsulating Security Payload)** - Encrypts the data payload
- **AES** - Encryption algorithm
- **SHA-HMAC** - Ensures data hasn't been tampered with

**Crypto Map (binds everything together):**
```
crypto map VPN-MAP 10 ipsec-isakmp
  description VPN connection to R3
  set peer 10.2.2.2
  set transform-set VPN-SET
  match address 110
  exit
```

This crypto map says: "When traffic matches ACL 110, use this transform set and peer to encrypt it."

### The Critical ACL: "Interesting Traffic"

This is the part most people get wrong. The ACL defines **what gets encrypted** — without it, nothing happens.

```
access-list 110 permit ip 192.168.1.0 0.0.0.255 192.168.3.0 0.0.0.255
```

**Translation:** "Traffic coming FROM the 192.168.1.0/24 network TO the 192.168.3.0/24 network is interesting and must be encrypted."

On R3, the ACL is reversed:
```
access-list 110 permit ip 192.168.3.0 0.0.0.255 192.168.1.0 0.0.0.255
```

**This is critical:** ACLs must be **bidirectional mirrors** of each other. If R1 says "encrypt 192.168.1 → 192.168.3," then R3 must say "encrypt 192.168.3 → 192.168.1." Mismatched ACLs are a common configuration mistake.

### Binding to the Interface

The crypto map only works when applied to an outgoing interface:

```
interface s0/0/0
  crypto map VPN-MAP
```

This tells R1 to apply the VPN policy to outbound traffic on Serial 0/0/0.

## Testing & Validation

### Before Encryption (No Traffic)

```
R1# show crypto ipsec sa
Packets encapsulated: 0
Packets encrypted: 0
Packets decapsulated: 0
Packets decrypted: 0
```

The tunnel exists but hasn't seen any "interesting traffic" yet.

### Triggering the Tunnel

I pinged from PC-A (192.168.1.5) to PC-C (192.168.3.5):

```
PC-A> ping 192.168.3.5
Reply from 192.168.3.5: bytes=32 time=25ms TTL=64
```

This immediately triggered ISAKMP negotiation:
1. Phase 1: R1 and R3 authenticate using the pre-shared key
2. Phase 2: They agree on encryption parameters
3. The tunnel becomes active
4. Ping packets are encrypted and forwarded

### After Encryption (Active Tunnel)

```
R1# show crypto ipsec sa
Packets encapsulated: 4
Packets encrypted: 4
Packets decapsulated: 4
Packets decrypted: 4
```

Success! The tunnel encrypted our ping packets.

### Testing Non-Interesting Traffic

I then pinged from PC-A to PC-B (192.168.2.5 - a different network):

```
PC-A> ping 192.168.2.5
Reply from 192.168.2.5: bytes=32 time=10ms TTL=64
```

And checking the tunnel:

```
R1# show crypto ipsec sa
Packets encapsulated: 4  ← Still 4 (unchanged!)
Packets encrypted: 4
```

The packet count didn't increase because PC-B is not in our ACL. This demonstrates the **principle of selective encryption** — only traffic matching the ACL gets encrypted.

## Key Learnings

### 1. **Interesting Traffic is Everything**

The ACL isn't just a firewall rule; it's the entire policy for what the VPN protects. If you don't define it, nothing gets encrypted — and if you define it wrong, the VPN won't work at all.

### 2. **Phase 1 and Phase 2 Must Match**

I made a mistake early on: R1 had `group 5` while I initially forgot it on R3. Result? The tunnel wouldn't establish. **Mismatch in ANY ISAKMP parameter breaks the tunnel.**

### 3. **ACLs Must Be Bidirectional Mirrors**

A common error:
- R1: `access-list 110 permit ip 192.168.1.0 0.0.0.255 192.168.3.0 0.0.0.255` ✓
- R3: `access-list 110 permit ip 192.168.1.0 0.0.0.255 192.168.3.0 0.0.0.255` ✗ (WRONG!)

Correct:
- R1: `access-list 110 permit ip 192.168.1.0 0.0.0.255 192.168.3.0 0.0.0.255` ✓
- R3: `access-list 110 permit ip 192.168.3.0 0.0.0.255 192.168.1.0 0.0.0.255` ✓

### 4. **Crypto Maps Bind Configuration to Reality**

Phase 1 and Phase 2 are just settings. The crypto map is what actually *applies* them to traffic on an interface. Without the crypto map binding, you have a configured but non-functional VPN.

### 5. **DH Group Limitations in Simulation**

Packet Tracer only supports DH Group 5. Production networks use DH 14 (2048-bit) or higher. This is a security consideration — larger DH groups resist brute-force attacks better.

## Real-World Applications

This lab directly applies to enterprise scenarios:

**Branch Office Connectivity** — A retail company securely connects 50 branch stores to headquarters using site-to-site IPsec VPNs.

**Data Center Failover** — Two geographically separated data centers encrypt all replication traffic across the internet.

**Mergers & Acquisitions** — Securely integrate two companies' networks during an acquisition without exposing sensitive data.

**Cloud Hybrid Networks** — Connect on-premises infrastructure to cloud VPCs using IPsec tunnels.

## Common Troubleshooting Scenarios

| Problem | Cause | Solution |
|---------|-------|----------|
| Tunnel won't establish | ISAKMP parameters don't match | Verify encryption, authentication, and DH group on both routers |
| Tunnel established but no data flows | ACL mismatch | Ensure ACLs are mirrors and correctly identify interesting traffic |
| Tunnel drops after 1 hour | Aggressive Phase 1 timeout | Increase ISAKMP keepalive timers |
| Traffic not encrypted but tunnel is up | Wrong interface binding | Verify crypto map is applied to the correct outgoing interface |

## Conclusion

Site-to-site VPNs are deceptively complex. What looks like four simple configurations (ACL, ISAKMP, IPsec transform, crypto map) actually represents a complete security framework. Getting it right requires understanding *why* each piece exists:

- **ACLs** define security policy
- **ISAKMP** builds the secure channel
- **IPsec** encrypts the actual data
- **Crypto maps** apply policy to interfaces

By building this lab, I moved from theoretical understanding ("VPNs encrypt traffic") to practical competency ("I can configure, test, and troubleshoot IPsec tunnels"). This is invaluable for anyone working in network security.

---

**Course:** Cloud and Network Security (C1-2026)  
**Student:** Rita Njoki (CS-CNS11-26074)  
**Lab Date:** Saturday, March 7, 2026  
**Status:** ✓ Completed with 100% tunnel verification

*Next: Explore dynamic VPNs with IPsec in IKEv2, multi-site VPN meshes, and VPN failover scenarios.*
