# Self-Hosted External Attack Surface & Threat Intel Lab

A lightweight, low-noise external monitoring and attack surface validation stack built to run alongside—and independently verify—commercial off-the-shelf security tools.

## Why Build This?

Commercial EASM and threat intelligence feeds are valuable, but they frequently lag behind fast-moving external changes:
- Newly registered lookalike domains and TLS certificates issued hours before a phishing or billing-scam campaign launches.
- Unmonitored subdomains, staging endpoints, or dangling CNAMEs (`200 → 404`) spun up outside normal change control.
- Hardcoded API keys or internal domain references pushed to public repositories or third-party integration forks.
- Niche dark web and Telegram chatter that requires brand-specific proximity rules rather than noisy, generic regex feeds.

This blueprint pairs deterministic Python automation, open-source recon tools, and local LLM triage into a self-hosted pipeline that alerts **only on net-new, verified deltas**—at near-zero recurring SaaS cost.

---

## Architecture Overview

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                        Intel Worker VM (Linux)                          │
│                                                                         │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌───────────────┐  │
│  │  brand-monitor/      │  │  code-monitor/       │  │ easm-monitor/ │  │
│  │  crt.sh + DNS        │  │  TruffleHog          │  │ bbot + httpx  │  │
│  │  Lookalike Sweep     │  │  (--only-verified)   │  │ State Diffing │  │
│  └──────────┬───────────┘  └──────────┬───────────┘  └───────┬───────┘  │
│             │                         │                      │          │
│             └─────────────────────────┼──────────────────────┘          │
│                                       ▼                                 │
│                     Local State & SHA-256 Deduplication                 │
│               (seen.json | seen_secrets.json | baseline.json)           │
└───────────────────────────────────────┬─────────────────────────────────┘
                                        │ Net-New Verified Deltas Only
                                        ▼
                        Discord Webhook / Daily PDF Digest
                                        ▲
                                        │ SSH Port Forwarding (-L)
┌───────────────────────────────────────┴─────────────────────────────────┐
│                       Analyst Workstation (macOS)                       │
│  • Isolated Tor SOCKS5 Browser Profile (DNS-leak hardened)              │
│  • Outbound Firewall (LuLu) + Persistence Monitoring (BlockBlock)       │
│  • Local MoE LLM Triage (Zero cloud token cost / zero data egress)      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Repository Structure

```text
ciso-external-intel-blueprint/
├── brand-monitor/
│   └── monitor.py            # Daily Certificate Transparency (crt.sh) & DNS lookalike monitor
├── code-monitor/
│   └── scan_code.py          # Daily TruffleHog org & public domain code secret scanner
├── easm-monitor/
│   └── easm_diff.py          # Weekly stateful bbot + httpx subdomain & endpoint delta monitor
├── mac-workstation/
│   └── zshrc_helpers.zsh     # SSH tunnel & isolated, DNS-leak-hardened Chrome-over-Tor helpers
├── .env.example              # Environment variable template
├── crontab.example           # Automated daily/weekly schedule
└── README.md
```

---

## Components

### 1. Lookalike Domain & Certificate Transparency Monitor (`brand-monitor/monitor.py`)
- **What it does:** Queries `crt.sh` Certificate Transparency logs for third-party TLS certificates containing your brand names and runs multi-threaded DNS resolution across common payment, portal, login, and support permutations (`-pay`, `-portal`, `-billing`, `-login`, `-support`, `-secure`).
- **Noise control:** Records a baseline in `seen.json` on first run and ignores verified corporate domains (`LEGIT_DOMAINS`), alerting only when a **new** lookalike resolves or a **new** third-party certificate is issued.

### 2. Verified Public Code & Secret Leak Monitor (`code-monitor/scan_code.py`)
- **What it does:** Runs `TruffleHog --only-verified` across your organization's public GitHub handles (`GITHUB_ORGS`) and queries GitHub's Code Search API for public repositories referencing your corporate domains.
- **Noise control:** Enforces `--only-verified` (confirming credentials are live), skips massive unrelated community blocklists/mirrors (`size < 15000` KB), and deduplicates findings via SHA-256 fingerprints in `seen_secrets.json`.

### 3. Stateful Weekly EASM Delta Monitor (`easm-monitor/easm_diff.py`)
- **What it does:** Runs passive `bbot` subdomain enumeration (`--flags passive --exclude-modules dnsbrute`) followed by rate-limited `httpx` fingerprinting (`status_code`, `title`, `tech`) across your domain portfolio.
- **Noise control:** Stores a canonical snapshot in `baseline.json` and alerts Discord **only** when:
  - A brand-new subdomain appears.
  - A new live HTTP/HTTPS endpoint is exposed.
  - An existing endpoint shifts HTTP status (e.g., `403 → 200` exposure or `200 → 404` dangling CNAME takeover candidate).

### 4. Analyst Workstation Isolation (`mac-workstation/zshrc_helpers.zsh`)
- **`ti` helper:** Establishes SSH local port forwards (`-L`) to internal web dashboards and Tor SOCKS5 (`9050`) without exposing management interfaces to the LAN.
- **`chrometor` helper:** Launches an isolated Google Chrome user profile (`--user-data-dir`) locked to `socks5://127.0.0.1:9050` with `--host-resolver-rules="MAP * ~NOTFOUND , EXCLUDE 127.0.0.1"` so local DNS fallback is impossible if the tunnel drops.

---

## Quick Start

### Prerequisites
- Linux worker host (Ubuntu/Debian) with **Python 3.10+**, **Docker**, and **`trufflehog`** installed (`/usr/local/bin/trufflehog`).
- Recon container or local installation of **`bbot`** and **`httpx`** for `easm-monitor`.
- A **Discord Webhook URL** for real-time alert delivery.
- An **unscoped GitHub Classic Personal Access Token** (zero permission checkboxes selected = public read-only access for GitHub Code Search rate limits).

### Installation

1. **Configure Environment Variables**
   Copy `.env.example` into each monitor directory (`brand-monitor/.env`, `code-monitor/.env`, `easm-monitor/.env`) and populate your values:
   ```bash
   cp .env.example brand-monitor/.env
   cp .env.example code-monitor/.env
   cp .env.example easm-monitor/.env
   ```

2. **Customize Target Brands & Domains**
   Edit `BRANDS`, `LEGIT_DOMAINS`, and `DOMAINS` at the top of:
   - `brand-monitor/monitor.py`
   - `code-monitor/scan_code.py`
   - `easm-monitor/easm_diff.py`

3. **Run Initial Baselines Manually**
   Run each script once interactively to populate its initial state file (`seen.json`, `seen_secrets.json`, `baseline.json`) without triggering initial alert floods:
   ```bash
   python3 -u brand-monitor/monitor.py
   python3 -u code-monitor/scan_code.py
   python3 -u easm-monitor/easm_diff.py
   ```

4. **Schedule Automated Cron Jobs**
   Install the schedule from `crontab.example` (`crontab -e`):
   ```cron
   30 4 * * * /usr/bin/python3 -u \(HOME/code-monitor/scan_code.py >>\)HOME/code-monitor/cron.log 2>&1
   15 6 * * * /usr/bin/python3 -u HOME/brand-monitor/monitor.py >> HOME/brand-monitor/cron.log 2>&1
   0 2 * * 0 /usr/bin/python3 -u \(HOME/easm-monitor/easm_diff.py >>\)HOME/easm-monitor/cron.log 2>&1
   ```

---

## Lessons Learned & Operational Best Practices

1. **Alert Fatigue Kills Independent Monitoring:** Flat keyword watchlists containing generic terms (`CVE`, `credentials`, `AKIA`, `BEGIN PRIVATE KEY`) will bury you in noise. Anchor dark web/Telegram rules to exact brand variations and proximity windows, and enforce `--only-verified` for secret scanning.
2. **Track Deltas, Not Snapshots:** A list of 500 subdomains is only useful once; knowing within days when subdomain #501 spins up—or when an existing endpoint flips from `200` to `404`—is where real defensive leverage lives.
3. **Least-Privilege API Tokens:** Never store a GitHub token with `repo` (private repository) scope on a recon server. An unscoped classic token provides zero private access while still authenticating public Code Search API queries.
4. **Harden the Analyst Endpoint:** Pair `.onion` research with an isolated browser profile, an outbound application firewall (such as Objective-See's **LuLu**), and persistence monitoring (**BlockBlock**).

---

## Disclaimer

This toolkit is intended strictly for authorized defensive security monitoring, external attack surface management, and brand protection of domains and assets you own or are explicitly authorized to assess.
