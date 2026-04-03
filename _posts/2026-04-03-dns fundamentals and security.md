---
title: DNS Fundamentals & Security - Mastering Domain Name Resolution in Cloud Environments
date: 2026-02-02 14:30:00 +0300
categories: [Network-Security, DNS, Cloud, TryHackMe, Fundamentals]
tags: [dns, domain-names, record-types, dns-queries, network-protocols, security]
pin: true
image:
  path: /assets/dnssec.jpg
  alt: DNS resolution flow showing hierarchy from root to authoritative nameserver

---

## Why I Studied DNS

Most people think of DNS as "the thing that makes google.com work." But for cybersecurity professionals, DNS is far more — it's a critical infrastructure protocol that's both essential and frequently exploited. Understanding DNS deeply is non-negotiable for anyone in network or cloud security.

The TryHackMe "DNS In Detail" lab forced me past surface-level knowledge into the actual mechanics of domain name resolution, DNS record types, and the security implications of misconfiguration.

![DNS Hierarchy](/assets/dnssec.jpg)
_The hierarchical structure of DNS: Root → TLD → Authoritative Nameservers_

## Lab Objectives

I set out to understand:
1. **DNS hierarchy** and how domain names are structured
2. **Record types** and their specific purposes
3. **The DNS resolution process** step-by-step
4. **Query mechanics** and caching behavior
5. **Practical DNS lookups** using command-line tools

## Understanding DNS Hierarchy

### The Three-Layer Domain Structure

A domain like `admin.tryhackme.com` has three components:

```
admin . tryhackme . com
  ↓        ↓        ↓
 SLD     Second    TLD
(Sub)    Level    (Top)
```

**Subdomain (Admin):** The leftmost label. This is the "subdomain" level where you can create unlimited subdomains. The key constraint: **maximum 63 characters per label, cannot start/end with hyphens.**

Example variations:
```
api.tryhackme.com
admin.tryhackme.com
developer.tryhackme.com
v2-api.tryhackme.com  ✓ (hyphens allowed in middle)
-api.tryhackme.com    ✗ (cannot start with hyphen)
```

You can stack subdomains:
```
backend.api.v2.tryhackme.com
```

But each label must follow the same rules independently.

**Second-Level Domain (TLD):** The rightmost label. For `.com`, the maximum length is **253 characters** for the entire domain name. This seems generous, but when you account for subdomains, it fills up quickly.

**Top-Level Domain (.com):** The authority for all `.com` domains. Other examples:
- `.org` - Organizations
- `.uk` - United Kingdom (country code)
- `.gov` - US Government
- `.io` - British Indian Ocean Territory (but adopted by tech startups)

### Critical Insight: Domain Ownership & Subdomains

Here's where security gets interesting: **The owner of `tryhackme.com` can create ANY subdomain they want.** This means:

- `admin.tryhackme.com` ✓ Controlled by TryHackMe
- `phishing.tryhackme.com` ✓ Also controlled by TryHackMe (even if it's a mistake!)
- `evil.tryhackme.com` ✓ Still under TryHackMe's control

An attacker cannot create `attacker.tryhackme.com` — they would need to compromise TryHackMe's domain registrar or DNS server.

## DNS Record Types: The Language of DNS

DNS records are the actual data stored in nameservers. Different record types serve different purposes:

### A Records (IPv4 Addresses)

```
www.google.com → A Record → 142.251.46.110
```

**What:** Maps domain names to IPv4 addresses  
**Purpose:** The foundational record for accessing websites  
**Example:**
```
@ 300 IN A 192.0.2.1
www 300 IN A 192.0.2.1
```

The `300` is the TTL (Time To Live) in seconds — how long DNS resolvers can cache this record.

### AAAA Records (IPv6 Addresses)

```
www.google.com → AAAA Record → 2607:f8b0:4004:808::200e
```

Same as A records but for IPv6. As IPv6 adoption increases, these become more common. The question in the lab asked: "What record type handles IPv6 addresses?" Answer: **AAAA** (four A's = 128-bit address, vs. A's 32 bits).

### MX Records (Mail Exchange)

```
tryhackme.com → MX Record → mail.tryhackme.com (priority 10)
```

**What:** Directs email to mail servers  
**Purpose:** Tells senders where to deliver email  
**Priority field:** Lower numbers = higher priority
```
@ 300 IN MX 10 mail1.tryhackme.com
@ 300 IN MX 20 mail2.tryhackme.com
```

If `mail1` is down, mail gets routed to `mail2`. This is critical for email resilience.

**Security angle:** Attackers who control MX records can intercept all email for the domain. Email accounts often use "forgot password" flows that send reset links via email.

### TXT Records (Text Data)

```
tryhackme.com → TXT Record → "v=spf1 include:_spf.google.com ~all"
```

**What:** Arbitrary text data  
**Purpose:** Multiple uses — SPF for anti-spoofing, DKIM for email signing, domain verification, etc.

**Real example:**
```
@ 300 IN TXT "v=spf1 include:sendgrid.net ~all"
```

This says: "Only SendGrid is authorized to send email from this domain. Reject others softly (~all)."

Without SPF records, attackers can send email that *appears* to come from your domain (spoofing).

### CNAME Records (Canonical Name / Alias)

```
blog.tryhackme.com → CNAME → www.tryhackme.com
```

**What:** Alias one domain to another  
**Purpose:** Makes one domain point to another's DNS records

Example:
```
blog 300 IN CNAME www.tryhackme.com
```

Now `blog.tryhackme.com` uses the same IP as `www.tryhackme.com`. Common use case: A CDN might tell you to point your subdomain to their CNAME:
```
cdn 300 IN CNAME d12345.cloudfront.net
```

**Security consideration:** CNAMEs can create domain takeover opportunities if not managed carefully. If you CNAME to a domain you don't own, and that domain disappears, your CNAME becomes dead and unusable.

### NS Records (Nameserver)

```
tryhackme.com → NS → ns1.tryhackme.com
```

**What:** Specifies which servers hold the DNS records  
**Purpose:** Delegates authority

If you buy a domain at GoDaddy but host DNS at Cloudflare, your domain's NS records point to Cloudflare's servers. The DNS resolver then queries Cloudflare, not GoDaddy.

```
@ 300 IN NS ns1.cloudflare.com
@ 300 IN NS ns2.cloudflare.com
```

**Security implication:** Whoever controls the NS records controls the domain's DNS. This is why DNS hijacking is so dangerous — an attacker who changes NS records can redirect traffic anywhere.

## The DNS Resolution Process

Here's what happens when you type `www.google.com` into your browser:

### Step 1: Local Recursive Resolver
```
Client PC → ISP DNS Server (8.8.8.8 or 1.1.1.1)
```

Your computer asks its configured DNS server (usually your ISP or public DNS like Cloudflare). This is a **recursive query** — "Get me the answer, I don't care how."

### Step 2: Root Nameserver
```
ISP DNS → Root Nameserver (.com)
"Where are the .com nameservers?"
```

The root nameserver returns the address of the TLD nameserver responsible for `.com` domains.

### Step 3: TLD Nameserver
```
ISP DNS → TLD Nameserver (.com authority)
"Where is google.com?"
```

The TLD server responds with the address of Google's authoritative nameserver.

### Step 4: Authoritative Nameserver
```
ISP DNS → google.com Authoritative NS
"What's the IP for www.google.com?"
```

Google's authoritative server returns the A record:
```
www.google.com → 142.251.46.110
```

### Step 5: Caching & Return
```
ISP DNS → Client PC
```

The resolver caches the result (TTL = 300 seconds, so for 5 minutes) and returns it to your PC.

### The Critical "Interesting Traffic" Concept

The lab taught me that DNS has **interesting traffic** similar to VPNs:

- **Authoritative nameservers** hold the actual records
- **Recursive resolvers** ask questions on behalf of clients
- **TTL (Time To Live)** controls caching duration

A lower TTL means:
- ✓ Changes propagate faster
- ✗ More DNS queries (higher load)

A higher TTL means:
- ✓ Fewer queries (less load)
- ✗ Changes take longer to propagate (if you update DNS, old clients still get stale data)

This is why many cloud providers set TTL to 300-3600 seconds during normal operation, but drop it to 60 seconds before major migrations.

## Practical DNS Queries

### Using nslookup (Windows/Linux/Mac)

```bash
$ nslookup shop.website.thm
Server: 127.8.0.53
Address: 127.8.0.53:53

Non-authoritative answer:
Name: shop.website.thm
Address: 127.8.0.53
```

The "Non-authoritative" answer means this came from a **recursive resolver's cache**, not from the authoritative nameserver itself.

### Using dig (Linux)

```bash
$ dig shop.website.thm +short
127.8.0.53
```

Much cleaner. The lab's practical section had us query specific record types:

```bash
$ dig website.thm MX
; shop.website.thm MX query
; result: mail = 30 alt4.aspmx.l.google.com

$ dig website.thm TXT
; TXT record
text = "THM[70128BA699377F35A951GC2E16D2944FF]"
```

In the lab's practical exercise, I had to:
1. Query the CNAME of `shop.website.thm` → Found `shops.myshopify.com`
2. Get the TXT record value
3. Find the MX record priority
4. Determine the IPv4 address of the A record

This mimics real penetration testing — DNS reconnaissance is often the first phase of an attack.

## Security Implications

### DNS Spoofing

If an attacker intercepts DNS queries (on an unencrypted network), they can respond with fake IP addresses:

```
You: "What's the IP for your-bank.com?"
Attacker intercepts: "It's 192.0.2.1 (attacker's server)"
You: Visit attacker's fake banking site
```

### DNS Poisoning

More sophisticated: An attacker floods a DNS resolver with fake responses, hoping one gets cached:

```
Attacker: "www.google.com is 192.0.2.1, 192.0.2.2, 192.0.2.3..."
(50 fake responses per second)
```

If one gets cached, thousands of users see the wrong IP.

### DNSSEC: Cryptographic Authentication

Modern DNS uses **DNSSEC** (DNS Security Extensions) to sign records:

```
Domain signs its records: "This is tryhackme.com's record, signed with our private key"
Resolver verifies: "Signature is valid, I trust this"
Attacker injects fake record: "Here's a fake record"
Resolver rejects: "Signature doesn't match, rejecting"
```

### DoS Amplification

DNS servers can be weaponized. An attacker sends a small DNS query with a spoofed source IP (your target), and the DNS server responds with a large answer to your target:

```
Attacker (1 KB query) → DNS Server → Target (100 KB response)
```

Multiply by thousands of queries, and the target is flooded.

## Conclusion

DNS is the internet's **phonebook** — but it's also a critical attack surface. By understanding:

- **How domains are structured** (hierarchy, character limits, ownership)
- **What DNS records do** (A, AAAA, MX, TXT, CNAME, NS)
- **How resolution works** (recursive, authoritative, caching)
- **Security risks** (spoofing, poisoning, DoS, hijacking)

I can now:
- Identify misconfigurations during security assessments
- Spot DNS-based attacks in network traffic
- Design DNS policies that balance performance and security
- Troubleshoot DNS issues in cloud deployments

For cloud architects, DNS is even more critical. Misconfigured DNS can lead to:
- Traffic routing to the wrong data center
- SSL certificate validation failures
- Email delivery problems
- Subdomain takeovers (if you CNAME to a service you no longer use)

This lab transformed DNS from "something that just works" into a protocol I understand deeply — and can defend against.

---

**Course:** Cloud and Network Security (C1-2026)  
**Student:** Rita Njoki (CS-CNS11-26074)  
**Lab Date:** Monday, February 2, 2026  
**Status:** ✓ Completed - 112 points, 5 tasks completed, 100% room mastery

*Next: Explore DNSSEC implementation, DNS over HTTPS (DoH), and advanced DNS reconnaissance techniques for penetration testing.*
