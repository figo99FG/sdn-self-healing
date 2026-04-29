# AI-Driven Self-Healing SDN Controller
### Multi-Autonomous System BGP Retail Topology with Zero-Trust Enforcement

> **BSc Networking & Security — Final Year Project (CST3590)**  
> Middlesex University London · Figo Figo · 2025–2026

---

<div align="center">

![Python](https://img.shields.io/badge/Python-3.x-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Ryu](https://img.shields.io/badge/Ryu-SDN_Controller-00ADEF?style=for-the-badge)
![OpenFlow](https://img.shields.io/badge/OpenFlow-1.3-orange?style=for-the-badge)
![GCP](https://img.shields.io/badge/GCP-Cloud_Healing-4285F4?style=for-the-badge&logo=google-cloud&logoColor=white)
![scikit-learn](https://img.shields.io/badge/scikit--learn-Isolation_Forest-F7931E?style=for-the-badge&logo=scikit-learn&logoColor=white)

</div>

---

## Results at a Glance

| Metric | Target | Result |
|--------|--------|--------|
| BGP sessions established | 8 | **8 / 8 ✅** |
| Zero-trust flows enforced | 19 | **19 / 19 ✅** |
| AI false positive quarantines | 0 | **0 ✅** |
| AI hosts trained | 9 | **9 / 9 ✅** |
| VRRP failover time | < 200 ms | **101 ms mean ✅** |
| GCP self-healing detection | < 30 s | **~15 s ✅** |

---

## Overview

This project implements a fully autonomous SDN controller that manages a **four-Autonomous-System BGP retail network** — and heals itself when things go wrong.

The system combines three layers of intelligence:

- **AI anomaly detection** — per-host Isolation Forest models trained on live flow stats; abnormal hosts are automatically quarantined with a DROP flow and released after 90 seconds
- **Zero-trust micro-segmentation** — OpenFlow rules enforce strict department boundaries (HR / IT / Marketing); cross-department traffic is denied and logged at the switch level
- **GCP self-healing** — a cloud monitor probes a live retail Flask app every 5 seconds; on failure it emails an alert and SSHes into the GCP VM to restart nginx automatically

Everything runs in **Mininet** on a single Linux machine, with a live GCP VM acting as the retail cloud endpoint.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Ryu SDN Controller                   │
│   • OpenFlow 1.3 flow management                        │
│   • Per-host Isolation Forest (AI)                      │
│   • Zero-trust policy enforcement                       │
│   • Quarantine & auto-release logic                     │
└───────────┬─────────────────────────────────────────────┘
            │ OpenFlow 1.3 / TCP 6633
            ▼
┌─────────────────────────────────────────────────────────┐
│                   Mininet Topology                      │
│                                                         │
│  AS100 London          AS200 Core (Route Reflector)     │
│  ├─ r7 (gateway)       ├─ r2 (RR)                       │
│  ├─ lon_hr             ├─ r5 (dist-Manchester)          │
│  ├─ lon_it             └─ r6 (dist-Liverpool)           │
│  └─ lon_mkt                                             │
│                                                         │
│  AS300 Manchester      AS400 Liverpool                  │
│  ├─ r1 MASTER          ├─ r0 MASTER                     │
│  ├─ r3 BACKUP          ├─ r4 BACKUP                     │
│  ├─ man_hr             ├─ liv_hr                        │
│  ├─ man_it             ├─ liv_it                        │
│  └─ man_mkt            └─ liv_mkt                       │
│                                                         │
│  8 BGP sessions · VRRP failover · FRRouting daemons     │
└─────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────┐
│                    GCP Compute Engine                   │
│   Flask retail app · nginx · systemd                    │
│   figo-retail.duckdns.org                               │
│   Auto-healed via SSH on failure detection              │
└─────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| SDN Controller | [Ryu](https://ryu-sdn.org/) + OpenFlow 1.3 |
| Network Emulation | [Mininet](http://mininet.org/) |
| BGP / OSPF Routing | [FRRouting (FRR)](https://frrouting.org/) |
| AI Anomaly Detection | scikit-learn — Isolation Forest |
| Cloud Infrastructure | Google Cloud Platform (e2-micro) |
| Cloud Monitoring | GCP Cloud Monitoring (custom metrics) |
| Retail App | Flask + nginx (TLS via DuckDNS) |
| Alerting | Gmail SMTP |

---

## Zero-Trust Policy Matrix

Traffic is enforced at the OpenFlow switch level. Same-department allows; cross-department drops and logs.

|  | HR | IT | Marketing |
|--|----|----|-----------|
| **HR** | ✅ Allow | ❌ Deny | ❌ Deny |
| **IT** | ❌ Deny | ✅ Allow | ❌ Deny |
| **Marketing** | ❌ Deny | ❌ Deny | ✅ Allow |

BGP port 179 is always permitted via priority-500 flows regardless of department.

---

## AI Quarantine Logic

Each of the 9 hosts gets its own **Isolation Forest** model, retrained every 10 polling cycles (polled every 5 s on `FlowStatsReply`).

Quarantine (90-second DROP flow) triggers when **any** of:

- `denied_pkts ≥ 50` per interval (hard threshold)  
- `unique_destinations ≥ 40` per interval  
- Isolation Forest score `< −0.15` (after ≥ 8 training samples with ≥ 3 non-zero)

Feature vector: `pps · bps · unique_dst · new_flows · denied_pkts`

---

## GCP Self-Healing Loop

```
ai_engine.py probes /health every 5 s
        │
        ├─ down_streak ≥ 3 (slow fail)
        │       OR
        └─ fastfail_streak ≥ 4 (instant fail)
                │
                ▼
        Email alert → ff332@live.mdx.ac.uk
                │
                ▼
        remediate.py → SSH → systemctl restart nginx
                │
                ▼
        Verify HTTP 200 → 45 s cooldown
```

---

## Project Structure

```
sdn-self-healing/
├── controller.py        # Ryu SDN controller — flows, zero-trust, AI, quarantine
├── ai_engine.py         # GCP health monitor + self-healing trigger
├── gcp_topology.py      # Mininet topology — all hosts, switches, routers
├── launch_all.py        # Orchestrates full startup sequence
├── bootstrap_bgp.sh     # Generates FRR configs, brings up BGP sessions
├── remediate.py         # SSHes into GCP VM, restarts nginx/Flask
├── vrrp_watchdog.py     # Pings MASTER, promotes BACKUP on 3 failures
├── cloud_monitor.py     # Pushes 8 custom metrics to GCP Cloud Monitoring
├── evaluate.py          # Automated test suite (BGP, zero-trust, failover, AI)
├── start_demo.sh        # Opens 5 terminal tabs, starts everything in order
└── trigger_quarantine.py# Force-triggers AI quarantine for demo/testing
```

---

## Getting Started

### Prerequisites

```bash
# Install Mininet
sudo apt-get install mininet

# Install FRRouting
sudo apt-get install frr frr-pythontools

# Install Ryu
pip install ryu

# Install Python dependencies
pip install scikit-learn flask requests paramiko google-cloud-monitoring
```

### Run

```bash
# Clean up any previous state
sudo mn -c
sudo pkill -f ryu-manager
sudo pkill -f bgpd
sudo pkill -f zebra
sudo systemctl stop frr
sudo rm -f /tmp/tab*.sh

# Start everything (opens 5 terminal tabs)
bash start_demo.sh
```

Tabs start in order: Ryu (0 s) → Mininet (6 s) → BGP (18 s) → AI Engine (25 s) → Tests

### Verify

```bash
# Check all 8 BGP sessions are Established
sudo vtysh --vty_socket /tmp/r2 -c "show bgp summary"

# Run the automated evaluation suite
sudo python3 evaluate.py
```

---

## Demo Walkthrough

1. **BGP** — verify 8/8 sessions `Established` on r2 route reflector  
2. **AI training** — slow cross-site pings until `trained=9/9`  
3. **Zero-trust** — `lon_hr → man_hr` ✅ vs `lon_hr → lon_mkt` ❌  
4. **AI quarantine** — run `trigger_quarantine.py`, watch DROP flow install + 90s auto-release  
5. **VRRP failover** — `sudo ip link set r1-s0 down` → ~101ms recovery  
6. **GCP self-heal** — stop nginx on GCP VM → detection + email + auto-restart  
7. **Evaluate** — `sudo python3 evaluate.py` → full pass  

---

## Six Objectives

| # | Objective | Result |
|---|-----------|--------|
| O1 | Multi-AS BGP topology (4 ASes, 8 routers) | **8/8 sessions** |
| O2 | AI anomaly detection (Isolation Forest) | **0 false positives** |
| O3 | Zero-trust micro-segmentation (OpenFlow) | **19/19 enforced** |
| O4 | VRRP-style failover (< 200 ms) | **101 ms mean** |
| O5 | GCP self-healing (detect + remediate) | **~15 s detection** |
| O6 | Automated evaluation suite | **Full pass** |

---

## Known Gotchas

- Fast pings to the **same** department don't trigger quarantine (`denied_pkts` stays 0) — use cross-dept traffic or `trigger_quarantine.py`
- AI needs ≥ 8 samples per host; seed with slow pings before expecting quarantine
- `start_demo.sh` must **not** be run with `sudo` — `/tmp/tab*.sh` becomes root-owned and breaks the next run
- BGP needs ~30 s after Mininet starts — the startup script has built-in delays
- If a quarantined host won't recover, either wait 90 s or restart the controller

---

## Academic Context

**Module:** CST3590 — Final Year Project  
**Institution:** Middlesex University London  
**Degree:** BSc Networking & Security  
**Student:** Figo Figo (M00971570)  
**Year:** 2025–2026  

---

<div align="center">
  <sub>Built with Ryu · Mininet · FRRouting · scikit-learn · GCP</sub>
</div>
