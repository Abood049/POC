# Recon Automation Skill

## Purpose
Automated reconnaissance pipeline for bug bounty targets, chaining subdomain discovery, DNS resolution, network mapping, live host detection, crawling, and endpoint extraction into a single reproducible workflow with resumable checkpoints.

## Tools Used

### Discovery
- **subfinder** — passive subdomain enumeration
- **chaos-client** — ProjectDiscovery Chaos dataset subdomain retrieval
- **dnsx** — DNS resolution and filtering (A/AAAA/CNAME, wildcard filtering)
- **httpx** — live HTTP/HTTPS probing, status codes, titles, tech fingerprints
- **cloudlist** — cloud asset enumeration (AWS/GCP/Azure) if credentials configured
- **tldfinder** — TLD/root domain discovery for brand-based targets

### Crawling
- **katana** — active web crawler for endpoint/JS/route discovery
- **urlfinder** — passive URL discovery (archive.org, common crawl, etc.)

### Network
- **naabu** — fast port scanning
- **mapcidr** — CIDR expansion/aggregation from ASN or IP lists
- **asnmap** — ASN lookup and IP range mapping
- **cdncheck** — CDN/WAF detection to filter out-of-scope IP ranges
- **tlsx** — TLS/certificate data collection (SANs, issuer, cert-based subdomain discovery)

### OAST
- **interactsh-client** — out-of-band interaction testing (blind SSRF/XXE/RCE detection)

### Utilities
- **notify** — push pipeline progress/results to Slack/Discord/Telegram

## Output Structure

```
recon/
│
├── roots.txt          # root/apex domains in scope
├── subdomains.txt      # all discovered subdomains (deduplicated)
├── alive.txt           # resolved + live hosts
├── ips.txt              # resolved IP addresses
├── cidrs.txt            # expanded CIDR ranges
├── asn.txt               # ASN mapping results
├── ports.txt             # open ports per host
├── tech.txt               # technology fingerprints (httpx -tech-detect)
├── urls.txt                # collected URLs (katana + urlfinder)
├── js.txt                    # discovered JavaScript files
├── endpoints.txt               # extracted API/endpoint paths
├── tls.txt                      # TLS/cert data + cert-based subdomains
├── cdn.txt                       # CDN/WAF-flagged hosts (excluded from active scans)
└── screenshots/                   # visual recon (if screenshot step enabled)
```

## Workflow

```
Target
  │
  ▼
Collect root domains            (tldfinder / manual scope input)
  │
  ▼
Enumerate subdomains             (subfinder + chaos-client)
  │
  ▼
Resolve DNS                       (dnsx)
  │
  ▼
Extract IPs                        (dnsx -resp)
  │
  ▼
ASN Enumeration                     (asnmap)
  │
  ▼
CIDR Expansion                       (mapcidr)
  │
  ▼
Live HTTP                             (httpx)
  │
  ▼
Port Scan                              (naabu)
  │
  ▼
TLS Enumeration                         (tlsx)
  │
  ▼
CDN Detection                            (cdncheck)   → filter out-of-scope IPs
  │
  ▼
Technology Detection                      (httpx -tech-detect)
  │
  ▼
URL Collection                             (urlfinder)
  │
  ▼
Katana Crawl                                (katana)
  │
  ▼
JS Discovery                                 (katana -jc / js.txt)
  │
  ▼
Endpoint Extraction                           (regex/JS parsing on js.txt)
  │
  ▼
Final Deduplication                            (sort -u across all outputs)
  │
  ▼
Output                                          → recon/ + notify push
```

## Engineering Notes
- **Resumability**: checkpoint markers written after each stage so the pipeline can restart mid-run without repeating completed work.
- **Domain-preserving pass**: a separate httpx pass (post-naabu) re-associates open ports with their original hostnames, since naabu deduplicates by IP and loses hostname context when multiple domains share an IP.
- **Scope flags**: `-f` for wildcard/root domains, `-e` for exact-scope hosts, to avoid over-scanning assets not covered by a program's policy.
- **Auto-thresholds**: permutation tools (e.g. alterx-style wordlist generation) scale their output size based on the number of subdomains already discovered, to avoid excessive noise on large scopes.
- **CDN filtering**: hosts flagged by cdncheck are excluded from active/port scanning stages to avoid wasting time on shared infrastructure.
