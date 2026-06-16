---
title: "OSINT Recon Methodology: A Structured Approach to Target Discovery"
description: "A systematic methodology for passive and semi-passive reconnaissance — from initial domain enumeration through certificate transparency, social media pivoting, and infrastructure mapping."
date: 2026-03-05
tags:
  - OSINT
  - recon
  - pentesting
  - enumeration
author: newbiefromcoma
---

## Recon Is Where Engagements Are Won

Poor recon leads to shallow attacks. The classic mistake is starting with Nmap against a single IP while the real attack surface lives on forgotten subdomains, third-party SaaS integrations, and shadow IT assets the client forgot they owned.

This writeup documents a repeatable recon methodology — structured, passive-first, tool-agnostic. The goal is asset coverage before any active probing begins.

## Phase 1: Seed the Graph

Every recon engagement starts with a small set of known anchors: typically the primary domain, the organization name, and any known IP ranges from the scope document.

From these seeds, you grow the graph by pivoting on relationships:
- Domain → subdomains, DNS history, registrar
- IP → reverse DNS, ASN, netblock owner, hosting history
- Organization name → job listings, LinkedIn, GitHub orgs, app store listings
- Email → breach databases, social platforms, domain registration

**Never start active scanning until you have exhausted passive pivots.** You'll find assets that active scanning would miss, and you'll understand the target's architecture before making noise.

## Phase 2: Certificate Transparency

Certificate Transparency (CT) logs are one of the most underutilized sources in recon. Every public TLS certificate issued is logged by CAs to public CT logs. You get subdomain enumeration without sending a single DNS query to the target.

**crt.sh query:**
```
https://crt.sh/?q=%.example.com&output=json
```

**Extract unique names from JSON:**
```bash
curl -s "https://crt.sh/?q=%.example.com&output=json" | \
  jq -r '.[].name_value' | \
  sed 's/\*\.//g' | \
  sort -u
```

CT logs reveal:
- Internal subdomains the target didn't intend to expose
- Wildcard cert domains (useful for guessing patterns)
- Historical subdomains that may still have live services
- Third-party services (e.g., `jira.example.com`, `jenkins.example.com`)

**Additional CT sources:**
- `censys.io` — certificate search with filtering
- `shodan.io/domain/example.com` — ports + certs combined
- `google.com/transparencyreport/` — certificate transparency viewer

## Phase 3: DNS Intelligence

### Passive DNS

Passive DNS databases record DNS resolution history. This reveals:
- Historical IPs (identify infrastructure migration, find old servers)
- Subdomains that no longer resolve but may still have services
- Pattern recognition across large organizations

**Tools:**
- RiskIQ PassiveTotal (now Microsoft Defender)
- SecurityTrails (`securitytrails.com/domain/example.com`)
- DNSDB (Farsight)
- `virustotal.com` — Relations tab on any domain

### DNS Zone Transfer (AXFR)

If the nameserver is misconfigured, a zone transfer dumps the entire DNS zone:

```bash
dig axfr example.com @ns1.example.com
```

Rarely works against hardened targets, but worth testing on internal nameservers discovered during the engagement. Internal DNS zones frequently contain infrastructure maps the client didn't expect to expose.

### DNS Enumeration via Brute Force

When passive methods hit their limit, subdomain brute force fills gaps:

```bash
# gobuster with wordlist
gobuster dns -d example.com -w /opt/wordlists/subdomains-top1million.txt -t 50

# amass
amass enum -d example.com -passive
amass enum -d example.com -brute -w /opt/wordlists/all.txt

# subfinder (aggregates multiple sources)
subfinder -d example.com -silent
```

**Wordlist sources:**
- `SecLists/Discovery/DNS/` — multiple tiers from 10k to 10M entries
- `commonspeak2` — derived from real web crawl data

## Phase 4: ASN and IP Range Discovery

Knowing the target's IP space before active scanning prevents you from missing assets behind CDNs and lets you target internal infrastructure directly.

**Find ASNs by organization:**
```bash
# Using whois
whois -h whois.radb.net -- '-i origin AS12345' | grep route

# bgp.he.net — search org name for ASN
# https://bgp.he.net/search?search[search]=TARGET&commit=Search
```

**Expand ASN to IP prefixes:**
```bash
# All prefixes announced by ASN
curl -s "https://api.bgpview.io/asn/12345/prefixes" | jq '.data.ipv4_prefixes[].prefix'
```

**Reverse DNS on CIDR blocks:**
```bash
# dnsx for fast reverse lookups
echo "203.0.113.0/24" | dnsx -ptr -resp
```

**Shodan org search:**
```
org:"Target Corporation" http.status:200
```

This surfaces internet-exposed services across all IP space, not just what's in the scope document.

## Phase 5: GitHub and Code Intelligence

Source code leaks are consistently high-value findings. Developers commit secrets, internal documentation, and architecture details to public repos — often without realizing the repo is public.

### Search Patterns

```
# GitHub web search operators
org:targetname password
org:targetname secret
org:targetname api_key
org:targetname BEGIN RSA PRIVATE KEY
filename:.env DB_PASSWORD
filename:config.php password
filename:credentials.json
```

**Tools:**
- `trufflehog` — entropy analysis + pattern matching across git history
  ```bash
  trufflehog github --org=targetname
  ```
- `gitleaks` — secret detection in local repos
- `gitrob` — organization-wide GitHub scanning
- GitHub Code Search API (authenticated, higher rate limits)

### What to Look For

Beyond passwords, code repos reveal:
- Internal hostnames (API endpoints, internal service names)
- Technology stack and framework versions
- Third-party service integrations (S3 buckets, Slack webhooks, Stripe keys)
- Internal documentation in comments and READMEs
- Developer usernames → cross-reference with breach databases

## Phase 6: Social Engineering Intelligence (Passive)

### LinkedIn

Job postings are an intelligence goldmine. A posting for "Senior AWS Security Engineer who can manage GuardDuty, Config, and Security Hub" tells you the cloud provider, security tooling, and implies gaps in current coverage. A role for "Network Engineer with Palo Alto and Cisco ASA experience" maps the firewall stack.

**Data to extract:**
- Employee names → generate username patterns (firstlast, first.last, fl)
- Job titles → understand hierarchy and responsibility split
- Technology mentions in job descriptions
- Office locations

### LinkedIn Username Generation

Common enterprise username formats:
```
john.doe
jdoe
johndoe
john_doe
j.doe
```

Generate systematically:
```python
def gen_usernames(first, last, domain):
    patterns = [
        f"{first}.{last}",
        f"{first[0]}{last}",
        f"{first}{last}",
        f"{first[0]}.{last}",
        f"{first}_{last}",
    ]
    return [f"{u}@{domain}" for u in patterns]
```

Feed into Office 365 username validation (`/common/oauth2/token` timing attack or `autodiscover.xml`) to enumerate valid accounts without triggering lockout.

### Shodan and Censys

Both index internet-facing services continuously. Passive — you're reading their cache, not touching the target.

**Shodan queries:**
```
hostname:example.com                    # all indexed
hostname:example.com port:8080          # specific ports
ssl.cert.subject.CN:example.com         # by certificate CN
org:"Example Corp"                      # by org
product:"Apache Tomcat" org:"Example"   # specific product
```

**Censys queries (v2 API):**
```
parsed.names: example.com
autonomous_system.name: "Example Corp"
```

Both tools give you service banners, TLS cert details, and HTTP response data — all passively.

## Phase 7: Breach Data and Credential Intelligence

Credential exposure from prior breaches is an active attack vector. Leaked passwords often reused; email:password pairs enable password spraying against O365, VPN portals, and SSO providers.

**Sources:**
- `dehashed.com` — indexed breach data by email, username, IP, domain
- `haveibeenpwned.com` API — check email exposure status
- `intelx.io` — broader OSINT including breach data, pastes, TOR
- `spycloud.com` — business-focused, ATO prevention data

**Responsible use:** Credential data in breach databases should be used only for demonstrating exposure to clients and validating password policies — never for unauthorized authentication.

## Building the Attack Map

After phases 1–7, consolidate findings into a structured map before any active scanning:

```
Target: example.com
├── Subdomains (CT + brute): 47 unique, 31 resolving
│   ├── api.example.com      → AWS IP, port 443, TLS expired
│   ├── jenkins.example.com  → port 8080, no auth (!)
│   ├── staging.example.com  → different IP block than prod
│   └── legacy.example.com   → historical IP, may still live
├── IP Ranges
│   ├── 203.0.113.0/24 (ASN 12345, primary hosting)
│   └── 198.51.100.0/28 (AWS, us-east-1)
├── GitHub: org/example-corp
│   └── private key committed 2022-11-03 (SHA abc123)
├── LinkedIn: 850 employees, 12 postings
│   └── Security stack: CrowdStrike, Okta, Palo Alto
└── Breach data: 23 email addresses in HIBP
```

Active scanning begins from this map — targeted, purposeful, with minimal noise.

## Toolchain Summary

| Phase | Tool | Notes |
|---|---|---|
| CT logs | crt.sh, Censys | Passive, no target contact |
| DNS history | SecurityTrails | Requires free account |
| Subdomain enum | subfinder, amass, gobuster | Amass is slowest but most thorough |
| IP/ASN | bgpview.io, Shodan | Shodan requires API key for automation |
| Code recon | trufflehog, gitleaks | Always check git history, not just HEAD |
| Social | LinkedIn, job boards | Manual, highest signal |
| Breach data | dehashed, HIBP | Verify scope allows credential checks |

Recon is iterative. Every discovery is a new seed. An IP found via ASN lookup resolves to a hostname that reveals a new subdomain. A GitHub commit message references an internal service name. Follow the graph until it stops yielding new nodes.
