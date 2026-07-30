# Task 4 Part A — Passive Reconnaissance: dodopayments.tech

## Methodology
Passive-only techniques per rules of engagement: DNS resolution, certificate
transparency (crt.sh), and general web search. No active scanning, fuzzing,
or automated tooling directed at any dodopayments.tech host.

## Findings

### DNS Resolution
`dodopayments.tech` resolves to:
- 104.18.10.178
- 104.18.11.178
- 2606:4700:83b1:4cfa:75fe:66a:894b:d274 (IPv6)

These IPs fall within **Cloudflare's** published IP ranges. This indicates
the domain is fronted by Cloudflare — a CDN/reverse-proxy — meaning the
true origin server IP is not directly discoverable through DNS alone.
This is itself a defensive control worth noting: it prevents direct
IP-based targeting/DDoS against origin infrastructure.

### Certificate Transparency (crt.sh)
crt.sh returned intermittent 503 errors during testing (a known
availability issue with the free public service, unrelated to the
target). Repeated attempts with corrected query encoding returned no
indexed certificate history, consistent with dodopayments.tech being a
recently-provisioned or low-traffic domain (likely a dedicated
assessment/lab domain rather than long-lived production infrastructure).

### General Web Presence
No indexed subdomains, cached pages, or public references to
dodopayments.tech were found via web search. This contrasts sharply with
Dodo Payments' actual production domain (dodopayments.com), which has
extensive public presence (docs.dodopayments.com, test.dodopayments.com,
customer.dodopayments.com, active marketing/support content, app store
listings, etc.) — suggesting dodopayments.tech is intentionally isolated
from the company's main public-facing infrastructure, likely provisioned
specifically for candidate assessment lab environments.

## Risk Observations
- Minimal public footprint is itself a positive security posture:
  fewer discoverable entry points for an external attacker.
- Cloudflare fronting hides origin IPs, reducing direct-attack surface
  and providing baseline DDoS/WAF protection.
- Without an authorized lab-specific subdomain (per invite), no further
  passive enumeration was possible within the assessment's scope and
  time constraints. An attacker in the same position would likely pivot
  to alternative reconnaissance vectors: employee OSINT (LinkedIn,
  GitHub org repos), third-party breach databases, or waiting for the
  domain's public certificate to be indexed over time.

## Scope Compliance
No active scanning, fuzzing, or exploitation was performed against any
dodopayments.tech host, per the assessment's explicit rules of engagement.
