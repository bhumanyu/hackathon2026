# MQ Hackathon — Intelligent MQ Topology Simplification & Automation

> **IBM MQ Hackathon Submission**  
> Track: Intelligent MQ Topology Simplification & Automation

---

## Overview

Enterprise IBM MQ environments grow organically over years — accumulating redundant queue managers, orphaned queues, non-standard channel names, and tightly coupled application dependencies. This project demonstrates how to apply **intelligent reasoning** to analyse an as-is MQ topology, enforce architectural constraints, and produce a clean, simplified, automation-ready target state.

### Results

| Metric | As-Is | Target | Change |
|--------|-------|--------|--------|
| Queue Managers | 6 | 4 | ▼ 2 removed |
| Channels | 14 | 6 | ▼ 8 removed |
| Total Queues | 18 | 10 | ▼ 8 removed |
| Unused Queues | 4 | 0 | ▼ 4 cleaned |
| Multi-QM Apps | 3 | 0 | ▼ all fixed |
| Bad Channel Names | 5 | 0 | ▼ all fixed |
| **Complexity Score** | **87** | **44** | **▼ 49% simpler** |

---

## Complexity Formula

```
Score = (QM_count × 10) + (channel_count × 3) + (unused_queues × 5)
        + (multi_qm_apps × 8) + (bad_channel_names × 2)

As-Is  = (6×10) + (14×3) + (4×5) + (3×8) + (5×2) = 87
Target = (4×10) + (6×3)  + (0×5) + (0×8) + (0×2) = 44
```

---

## Core Constraints Enforced

| # | Constraint | Status |
|---|-----------|--------|
| 1 | Exactly 1 QM per AppID — apps connect only to their own queue manager | ✅ All 8 apps compliant |
| 2 | Producers write to remoteQ → xmitq → sender channel (never directly) | ✅ Enforced |
| 3 | Channel names must follow `fromQM.toQM` convention | ✅ All 6 channels renamed/fixed |
| 4 | Consumers read only from local queues fed by channels | ✅ Enforced |

---

## Project Structure

```
mq-hackathon/
├── README.md
├── data/
│   ├── asis/                        # As-is input CSVs (hackathon input)
│   │   ├── queue_managers.csv
│   │   ├── queues.csv
│   │   ├── channels.csv
│   │   └── applications.csv
│   └── target/                      # Target-state output CSVs
│       ├── target_queue_managers.csv
│       ├── target_queues.csv
│       ├── target_channels.csv
│       └── target_applications.csv
├── tools/
│   ├── mq_master_dashboard.html     # Main submission app (open in browser)
│   ├── mq_intelligent_agent.html    # Standalone AI agent tool
│   └── mq_topology_validation.html  # Topology visualizer + validation
└── docs/
    ├── design_decisions.md          # Full design decision documentation
    └── complexity_analysis.md       # Complexity metric explanation
```

---

## Tools

### 1. Master Dashboard (`tools/mq_master_dashboard.html`)
The primary submission tool. Open in any browser — no server needed.

**Sections:**
- **Overview** — KPI cards, score comparison, constraint verification, change timeline
- **Topology** — Interactive SVG network diagram (As-Is / Target / Side-by-Side). Click any node to inspect.
- **Agent** — Claude-powered live analysis: streams reasoning in a terminal UI, flags violations, generates decisions
- **Validate** — Human-in-the-loop review: step through each agent proposal, approve or reject
- **Export** — Download all 4 target-state CSVs

### 2. Intelligent Agent (`tools/mq_intelligent_agent.html`)
Standalone Claude API-powered tool. Reads the as-is CSV data, reasons about topology constraints, and outputs the full target state with explanations. Includes graceful fallback if API is unavailable.

### 3. Topology Visualizer (`tools/mq_topology_validation.html`)
Focused topology diagram + validation flow for demo purposes.

---

## How to Run

1. **Clone the repo**
   ```bash
   git clone https://github.com/YOUR_USERNAME/mq-hackathon.git
   cd mq-hackathon
   ```

2. **Open the main dashboard**
   ```bash
   open tools/mq_master_dashboard.html
   # or double-click the file in your file explorer
   ```

3. **No server, no dependencies** — pure HTML/CSS/JS. Works offline with fallback data.

4. **To use live Claude AI analysis** — the Agent page calls the Anthropic API automatically. Ensure internet access is available.

---

## Design Decisions Summary

### 1. Decommission QM.BATCH01 → absorbed into QM.INVENTORY01
QM.BATCH01 served only APP007 which itself violated the 1-QM rule. After migrating APP007 to INVENTORY01, BATCH01 is empty. Inventory is the natural home for batch replenishment jobs.

### 2. Decommission QM.REPORT01 → absorbed into QM.ORDER01
Single-app (APP008), single-queue QM receiving data via channel from ORDER01. No justification for a dedicated production QM at this scale.

### 3. Fix APP003 BillingService → QM.ORDER01 only
BillingService was reading from QM.PAYMENT01, violating the 1-QM constraint. Billing is downstream of Orders — ORDER01 is the correct home. A new local queue `Q.BILL.INCOMING` receives payment data via the existing PAYMENT→ORDER channel.

### 4. Fix APP005 FulfillService → QM.INVENTORY01 only
FulfillService had a dead connection to QM.LEGACY01 reading `Q.LEGACY.OUT`, which had no producer. The connection was already non-functional — removing it costs nothing.

### 5. Fix APP007 BatchProcessor → QM.INVENTORY01 only
BatchProcessor was split across LEGACY01 and BATCH01. Since BATCH01 is decommissioned, APP007 consolidates to INVENTORY01 where batch inventory logic naturally belongs.

### 6. Remove 4 unused queues + fix 5 bad channel names
Dead queues and non-standard channel names increase audit surface, break automated provisioning, and add operational risk with zero functional benefit.

---

## Judging Criteria — Coverage

| Criterion | How We Address It |
|-----------|------------------|
| All constraints correctly enforced | Verified in Agent + Validate tabs; all 4 rules pass |
| Clear and explainable design decisions | 6 documented decisions with reasoning in Agent tab |
| Demonstrated reduction in complexity | 87 → 44 score (49%) with formula breakdown |
| Applicability to real enterprise MQ | Uses actual remoteQ/xmitq/channel patterns; output is automation-ready CSV |
| Target-state topology dataset (CSV) | 4 clean CSVs in `data/target/` |
| As-is vs target complexity analysis | Formula + per-metric breakdown in Overview and Export |
| Topology visualizations | Interactive SVG in Topology tab (As-Is / Target / Side-by-Side) |
| Design and decision documentation | `docs/design_decisions.md` + in-app explanations |

---

## Team

IBM MQ Hackathon · 2025
