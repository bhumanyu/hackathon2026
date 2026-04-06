# Architecture
## Intelligent MQ Topology Simplification & Automation

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [High-Level Architecture](#2-high-level-architecture)
3. [Component Architecture](#3-component-architecture)
4. [Data Architecture](#4-data-architecture)
5. [Agent Architecture](#5-agent-architecture)
6. [MQ Topology Architecture](#6-mq-topology-architecture)
7. [Application Flow](#7-application-flow)
8. [Module Breakdown](#8-module-breakdown)
9. [Design Patterns Used](#9-design-patterns-used)
10. [Security Considerations](#10-security-considerations)
11. [Scalability & Extensibility](#11-scalability--extensibility)

---

## 1. System Overview

The system is a **single-page, browser-based application** that combines AI-powered topology reasoning with interactive visualisation and human-in-the-loop validation. It takes IBM MQ as-is topology data (CSV), analyses it against architectural constraints, and produces a simplified, automation-ready target state.

```
┌────────────────────────────────────────────────────────────────┐
│                        USER (MQ Architect)                     │
└────────────────────────────┬───────────────────────────────────┘
                             │  Opens HTML file in browser
                             ▼
┌────────────────────────────────────────────────────────────────┐
│                  MQ Hackathon Application                      │
│              (Zero-dependency browser app)                     │
│                                                                │
│   ┌──────────────────────────────────────────────────────┐     │
│   │                  Navigation Shell                    │     │
│   │   Overview │ Topology │ Agent │ Validate │ Export    │     │
│   └──────────────────────────────────────────────────────┘     │
│                             │                                  │
│   ┌─────────┬───────────────┼───────────────┬────────────┐     │
│   │Overview │  Topology     │    Agent       │  Validate  │     │
│   │Dashboard│  Visualizer   │  (Claude API)  │  H-i-L     │     │
│   └─────────┴───────────────┴───────┬───────┴────────────┘     │
│                                     │                          │
└─────────────────────────────────────┼──────────────────────────┘
                                      │ HTTPS / REST
                                      ▼
                     ┌────────────────────────────┐
                     │    Anthropic Claude API     │
                     │  claude-sonnet-4-*          │
                     │  Topology reasoning engine  │
                     └────────────────────────────┘
```

---

## 2. High-Level Architecture

### Architecture Style

The application follows a **client-side single-page architecture** with an external AI service integration.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        PRESENTATION LAYER                           │
│                                                                     │
│   ┌───────────┐  ┌──────────────┐  ┌──────────┐  ┌─────────────┐   │
│   │ Overview  │  │  Topology    │  │  Agent   │  │  Validate   │   │
│   │ Dashboard │  │  SVG Graph   │  │ Terminal │  │  Proposals  │   │
│   └───────────┘  └──────────────┘  └──────────┘  └─────────────┘   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                         LOGIC LAYER                                 │
│                                                                     │
│  ┌─────────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │ Topology Engine │  │ Complexity   │  │  Validation Engine     │  │
│  │ (SVG renderer)  │  │ Calculator   │  │  (Proposal manager)    │  │
│  └─────────────────┘  └──────────────┘  └────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │              Intelligent Agent (Claude API Client)           │   │
│  │   Prompt builder │ API caller │ Response parser │ Fallback   │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                          DATA LAYER                                 │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │   In-memory dataset (embedded CSV → parsed JS objects)      │    │
│  │                                                             │    │
│  │   ASIS_DATA          │   TARGET_DATA (generated)            │    │
│  │   queue_managers[]   │   target_queue_managers[]            │    │
│  │   queues[]           │   target_queues[]                    │    │
│  │   channels[]         │   target_channels[]                  │    │
│  │   applications[]     │   target_applications[]              │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Architecture

The application is composed of **5 independent page components**, each mounted inside a shared navigation shell.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Navigation Shell                             │
│   Manages active page state · Renders sidebar/topbar               │
└────┬────────────┬──────────────┬─────────────┬──────────────────────┘
     │            │              │             │              │
     ▼            ▼              ▼             ▼              ▼
┌─────────┐ ┌──────────┐ ┌───────────┐ ┌──────────┐ ┌───────────────┐
│Overview │ │Topology  │ │  Agent    │ │Validate  │ │    Export     │
│         │ │          │ │           │ │          │ │               │
│KPI cards│ │SVG render│ │Term. UI   │ │Proposals │ │CSV previews   │
│Score    │ │Node click│ │API call   │ │Approve / │ │Download btns  │
│Timeline │ │3 views   │ │Violations │ │Reject    │ │Formula recap  │
│Formula  │ │Legend    │ │Decisions  │ │Log       │ │               │
└─────────┘ └──────────┘ └───────────┘ └──────────┘ └───────────────┘
```

### Component Responsibilities

| Component | Responsibility | Key Functions |
|-----------|---------------|---------------|
| **Overview** | Executive summary of the topology transformation | Renders KPIs, score comparison, constraint checklist, change timeline |
| **Topology** | Interactive network graph of QM connectivity | `buildAsis()`, `buildTarget()`, `drawEdge()`, `drawNode()`, `showNodeInfo()` |
| **Agent** | AI-powered analysis via Claude API | `runAgent()`, streaming terminal logger, response parser |
| **Validate** | Step-by-step human review of agent proposals | `renderProp()`, `vDecide()`, decision log builder |
| **Export** | Download target-state CSVs | `buildExport()`, `dlCSV()`, `dlAll()` |

---

## 4. Data Architecture

### Data Model

All data is represented as structured CSV that maps directly to typed JavaScript objects in memory.

```
┌──────────────────────────────────────────────────────────────────┐
│                        DATA MODEL                                │
│                                                                  │
│  QueueManager                  Queue                            │
│  ├── QM_ID (PK)                ├── Queue_Name (PK)              │
│  ├── Region                    ├── QM_ID (FK → QueueManager)    │
│  ├── Neighborhood              ├── Type (LOCAL|REMOTE|XMIT)     │
│  └── Description               ├── HasProducer (bool)           │
│                                ├── HasConsumer (bool)           │
│                                ├── RemoteQM (FK, nullable)      │
│                                └── Notes                        │
│                                                                  │
│  Channel                       Application                      │
│  ├── Channel_Name (PK)         ├── App_ID (PK)                  │
│  ├── FromQM (FK → QueueMgr)    ├── App_Name                     │
│  ├── ToQM   (FK → QueueMgr)    ├── QM_IDs (FK[], pipe-sep)      │
│  ├── Type (SENDER|RECEIVER)    ├── Role                         │
│  └── NamingConventionOK (bool) ├── Queues_Used (FK[], pipe-sep) │
│                                └── SingleQM_Compliant (bool)    │
└──────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
Embedded CSV Strings (in JS constants)
         │
         ├──► As-Is Dataset (ASIS_DATA object)
         │         │
         │         ├──► Complexity calculator  ──► Score: 87
         │         │
         │         ├──► Topology renderer  ──► SVG As-Is graph
         │         │
         │         └──► Claude API prompt  ──► Agent reasoning
         │                                          │
         │                                          ▼
         │                               Structured JSON response
         │                                          │
         │                               ┌──────────▼──────────┐
         │                               │  violations[]        │
         │                               │  decisions[]         │
         │                               │  target_qm_summary[] │
         │                               │  output_csvs{}       │
         │                               └──────────┬──────────┘
         │                                          │
         ├──► Target Dataset (TARGET_CSVS object)◄──┘
         │         │
         │         ├──► Complexity calculator  ──► Score: 44
         │         │
         │         ├──► Topology renderer  ──► SVG Target graph
         │         │
         │         └──► Export  ──► Downloadable CSVs
         │
         └──► Validation proposals (PROPOSALS constant)
                   │
                   └──► Human decisions[]  ──► Decision log
```

### CSV Structure Specification

#### As-Is Input Format

```
queue_managers.csv
  QM_ID, Region, Neighborhood, Description

queues.csv
  Queue_Name, QM_ID, Type, HasProducer, HasConsumer, RemoteQM, Notes

channels.csv
  Channel_Name, FromQM, ToQM, Type, NamingConventionOK, Notes

applications.csv
  App_ID, App_Name, QM_IDs, Role, Queues_Used, SingleQM_Compliant, Notes
```

#### Target Output Format (same structure + Change column)

```
target_queue_managers.csv
  QM_ID, Region, Neighborhood, Description, Change

target_queues.csv
  Queue_Name, QM_ID, Type, HasProducer, HasConsumer, RemoteQM, Change

target_channels.csv
  Channel_Name, FromQM, ToQM, Type, NamingConventionOK, Change

target_applications.csv
  App_ID, App_Name, QM_ID, Role, Queues_Used, SingleQM_Compliant, Change
```

> **Design decision:** Target CSVs use identical column structure to input CSVs (plus one `Change` column) so they can feed directly into existing provisioning pipelines without schema changes.

---

## 5. Agent Architecture

The Intelligent Agent is the core AI component. It uses a structured prompt → response pattern to reason over the MQ topology.

```
┌─────────────────────────────────────────────────────────────────┐
│                    AGENT PIPELINE                               │
│                                                                 │
│  Step 1: Build System Prompt                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • 4 core constraints (explicit rules)                  │    │
│  │  • Complexity formula (scoring algorithm)               │    │
│  │  • Expected JSON output schema (strict)                 │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                     │
│  Step 2: Build User Prompt                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • Full as-is CSV data (all 4 files embedded)           │    │
│  │  • Instruction to apply all 4 constraints               │    │
│  │  • Instruction to return JSON only                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                     │
│  Step 3: API Call                                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  POST https://api.anthropic.com/v1/messages             │    │
│  │  model: claude-sonnet-4-*                               │    │
│  │  max_tokens: 4000                                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                     │
│  Step 4: Parse Response                                         │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • Strip markdown fences (```json ... ```)              │    │
│  │  • JSON.parse()                                         │    │
│  │  • Fallback: regex extract JSON block                   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                     │
│  Step 5: Render Results                                         │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • Violations tab (severity-tagged cards)               │    │
│  │  • Decisions tab (expandable cards with reasoning)      │    │
│  │  • Topology table (target QM summary)                   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  Step 6: Fallback (if API unavailable)                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • Pre-computed VIOLATIONS[] constant                   │    │
│  │  • Pre-computed DECISIONS[] constant                    │    │
│  │  • Pre-computed TGT_QMS[] constant                      │    │
│  │  • Graceful degradation — demo never breaks             │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### Agent Response Schema

```json
{
  "summary": {
    "asis_score": 87,
    "target_score": 44,
    "reduction_pct": 49,
    "asis_breakdown": {
      "qm_count": 6,
      "channel_count": 14,
      "unused_queues": 4,
      "multi_qm_apps": 3,
      "bad_channels": 5
    },
    "target_breakdown": {
      "qm_count": 4,
      "channel_count": 6,
      "unused_queues": 0,
      "multi_qm_apps": 0,
      "bad_channels": 0
    }
  },
  "violations": [
    {
      "severity": "Critical | Warning | Info",
      "type": "<violation type>",
      "object_id": "<affected object>",
      "description": "<what is wrong>",
      "fix": "<recommended action>"
    }
  ],
  "decisions": [
    {
      "title": "<short title>",
      "why": "<reasoning>",
      "what_changed": "<specific changes>"
    }
  ],
  "target_qm_summary": [
    {
      "qm_id": "<id>",
      "region": "<region>",
      "apps": "<app list>",
      "queues": 5,
      "channels_out": 2,
      "status": "expanded | unchanged | slimmed | new",
      "status_color": "green | blue | amber | purple"
    }
  ],
  "output_csvs": {
    "queue_managers": "<full CSV string>",
    "queues": "<full CSV string>",
    "channels": "<full CSV string>",
    "applications": "<full CSV string>"
  }
}
```

### Prompt Engineering Strategy

| Technique | Application |
|-----------|------------|
| **Role priming** | System prompt establishes Claude as an IBM MQ architect |
| **Explicit constraints** | All 4 rules listed verbatim — no inference required |
| **Schema-first output** | JSON schema provided upfront — model follows structure |
| **JSON-only instruction** | "Respond ONLY with valid JSON, no markdown, no preamble" |
| **Formula inclusion** | Complexity formula embedded so model uses exact same calculation |
| **Graceful fallback** | Pre-computed answers loaded if API call fails |

---

## 6. MQ Topology Architecture

### As-Is Topology

The as-is environment has 6 queue managers with 14 channels. QM.LEGACY01 acts as an uncontrolled hub.

```
                    ┌──────────────────┐
                    │   QM.ORDER01     │
                    │   US-EAST / NYC  │
                    │   APP001, APP003 │
                    └───┬──────────┬───┘
                        │          │
          ┌─────────────┘          └──────────────┐
          │ (good channel, bidir)                  │ (good channel, bidir)
          ▼                                        ▼
┌──────────────────┐                   ┌──────────────────────┐
│  QM.PAYMENT01    │                   │   QM.INVENTORY01     │
│  US-EAST / NYC   │                   │   US-WEST / LAX      │
│  APP002, APP003  │                   │   APP004, APP005     │
└────────┬─────────┘                   └──────────┬───────────┘
         │                                         │
         │ (good channel)                          │ (good channel)
         ▼                                         ▼
         │          ┌───────────────────┐           │
         │          │   QM.LEGACY01     │           │
         │          │   US-EAST / NYC   │           │
         │          │   ⚠ VIOLATION     │           │
         └─────────►│   APP005,006,007  │◄──────────┘
                    └──┬──┬──┬──┬──┬───┘
                       │  │  │  │  │
              5 bad channels (dashed)
              to: ORDER01, PAYMENT01,
                  INVENTORY01, BATCH01,
                  REPORT01

         ┌─────────────┘      └───────────────────┐
         ▼                                         ▼
┌──────────────────┐                   ┌──────────────────────┐
│   QM.BATCH01     │                   │   QM.REPORT01        │
│   EU-WEST / LON  │──────────────────►│   EU-WEST / LON      │
│   APP007         │  (good channel)   │   APP008             │
└──────────────────┘                   └──────────────────────┘

Legend:
  ──────►  Good channel (correctly named)
  - - - ►  Bad channel (non-standard name, violation)
  ⚠        Constraint violation (multi-QM apps or bad names)
```

### Target Topology

After simplification: 4 queue managers, 6 channels, all constraints satisfied.

```
                    ┌──────────────────────────┐
                    │       QM.ORDER01          │
                    │       US-EAST / NYC       │
                    │  ✓ Expanded               │
                    │  APP001, APP003, APP008   │
                    └───────┬─────────┬─────────┘
                            │         │
             ───────────────┘         └───────────────
            │  (good bidir channel)                   │  (good bidir channel)
            ▼                                         ▼
┌──────────────────┐                   ┌──────────────────────────┐
│   QM.PAYMENT01   │                   │     QM.INVENTORY01       │
│   US-EAST / NYC  │                   │     US-WEST / LAX        │
│   ✓ Unchanged    │                   │     ✓ Expanded           │
│   APP002         │                   │     APP004, APP005, APP007│
└──────────────────┘                   └──────────────────────────┘
                                                     ▲
                                                     │ (renamed channel)
                    ┌──────────────────────────┐      │
                    │       QM.LEGACY01         │──────┘
                    │       US-EAST / NYC       │
                    │  ✓ Slimmed                │──────────────────►
                    │  APP006                   │  (renamed channel)
                    └──────────────────────────┘  to QM.ORDER01

Legend:
  ──────►  Good channel (fromQM.toQM convention)
  ✓        Constraint compliant
  (removed) QM.BATCH01 and QM.REPORT01 decommissioned
```

### Channel Naming Convention

All channels must follow the `fromQM.toQM` pattern:

```
CORRECT:   QM.ORDER01.toQM.PAYMENT01
           QM.PAYMENT01.toQM.ORDER01
           QM.LEGACY01.toQM.INVENTORY01

INCORRECT: LEGACY_TO_ORDERS        ← renamed to QM.LEGACY01.toQM.ORDER01
           LEG_TO_INV              ← renamed to QM.LEGACY01.toQM.INVENTORY01
           LEGACY_PAYMENT_CH       ← removed (no longer needed)
           BATCH_SEND              ← removed (QM.BATCH01 decommissioned)
           RPT_CHAN                ← removed (QM.REPORT01 decommissioned)
```

### Producer → Consumer Message Flow (remoteQ Pattern)

```
Producer Application (e.g. OrderService on QM.ORDER01)
         │
         │  Writes to
         ▼
┌─────────────────────────┐
│   Q.ORDER.REMOTE        │  ← remoteQ (LOCAL type, RemoteQM = QM.PAYMENT01)
│   Type: REMOTE          │
│   On: QM.ORDER01        │
└────────────┬────────────┘
             │  MQ routes via
             ▼
┌─────────────────────────┐
│   Q.ORDER.XMIT          │  ← Transmission queue (XMIT type)
│   Type: XMIT            │
│   On: QM.ORDER01        │
└────────────┬────────────┘
             │  Picked up by
             ▼
┌─────────────────────────────────────────┐
│   QM.ORDER01.toQM.PAYMENT01             │  ← Sender Channel (fromQM.toQM name)
│   Type: SENDER                          │
│   From: QM.ORDER01 → To: QM.PAYMENT01  │
└────────────┬────────────────────────────┘
             │  Delivers to
             ▼
┌─────────────────────────┐
│   Q.PAY.LOCAL           │  ← Local Queue on destination QM
│   Type: LOCAL           │
│   On: QM.PAYMENT01      │
└────────────┬────────────┘
             │  Read by
             ▼
Consumer Application (e.g. PaymentService on QM.PAYMENT01)
```

---

## 7. Application Flow

### End-to-End User Journey

```
User opens HTML file in browser
         │
         ▼
[OVERVIEW TAB]
  · KPI cards load from in-memory constants
  · Score comparison rendered
  · Constraint checklist displayed
  · Change timeline rendered
         │
         │  User clicks "Topology" tab
         ▼
[TOPOLOGY TAB]
  · buildAsis() called → renders SVG As-Is graph
  · User toggles to "Target" view
  · buildTarget() called → renders SVG Target graph
  · User clicks a QM node → showNodeInfo() displays detail panel
  · User toggles "Side-by-Side" → both SVGs rendered
         │
         │  User clicks "Agent" tab
         ▼
[AGENT TAB]
  · User clicks "Run Agent Analysis"
  · runAgent() starts:
      → Terminal streams load messages
      → Anthropic API called with full dataset
      → Response parsed → violations, decisions rendered
      → Fallback: pre-computed data used if API fails
  · User browses Violations / Decisions / Topology sub-tabs
         │
         │  User clicks "Validate" tab
         ▼
[VALIDATE TAB]
  · renderProp(0) renders first proposal
  · User reads reasoning + impact analysis
  · User clicks "Approve" → vDecide(idx, true)
      → Decision logged
      → renderProp(idx+1) renders next proposal
  · Repeat for all 6 proposals
  · Completion banner appears → "Go to Export"
         │
         │  User clicks "Export" tab
         ▼
[EXPORT TAB]
  · buildExport() renders all 4 CSV previews
  · User clicks individual "⬇" buttons or "Download All"
  · CSVs downloaded as files
  · Decisions log shown at bottom
```

### Validation State Machine

```
         ┌─────────┐
         │  IDLE   │  (page loads, step 0 of 6)
         └────┬────┘
              │ renderProp(0)
              ▼
         ┌─────────┐
         │PROPOSAL │◄──────────────────────────────────┐
         │DISPLAYED│                                   │
         └────┬────┘                                   │
              │                                        │
       ┌──────┼──────┐                                 │
       │      │      │                                 │
       ▼      ▼      ▼                                 │
   [Approve][Reject][Skip]                             │
       │      │      │                                 │
       └──────┴──────┘                                 │
              │ vDecide(idx, true|false|null)           │
              │ Log entry added                         │
              │ step++                                  │
              │                                         │
              ├─── step < 6 ──────────────────────────►┘
              │
              └─── step === 6
                        │
                        ▼
                  ┌───────────┐
                  │ COMPLETE  │  (banner shown, Export enabled)
                  └───────────┘
```

---

## 8. Module Breakdown

### Topology Rendering Engine

Responsible for drawing QM network graphs as inline SVG.

```
buildAsis(svgEl, clickFn)
  ├── drawEdge(svg, posMap, 'ORDER01', 'PAYMENT01', '#60a5fa', false, bidir=true)
  ├── drawEdge(svg, posMap, 'LEGACY01', 'ORDER01',  '#f5c842', dashed=true, false)
  ├── ... (14 edges total for as-is)
  └── ASIS_NODES.forEach(n => drawNode(svg, n, ASIS_POS[n.id], clickFn))

buildTarget(svgEl, clickFn)
  ├── drawEdge(...)  ← 6 edges for target
  └── TARGET_NODES.forEach(...)

drawEdge(svg, posMap, fromId, toId, color, dashed, bidir)
  ├── ep(cx, cy, targetCx, targetCy)     ← compute edge attachment point on rect boundary
  ├── mkMarker(svg, color)               ← create/reuse SVG arrowhead marker
  └── mkPath(x1, y1, x2, y2, ox, oy)    ← quadratic Bezier curve path

drawNode(svg, node, pos, clickFn)
  ├── <rect> with fill/stroke from node.status
  ├── <text> label (IBM Plex Mono, 11px)
  └── <text> subtitle (region + status, 9px)
```

### Complexity Calculator

```
function calcScore(qm, ch, unused, multiQM, badNames) {
  return (qm * 10) + (ch * 3) + (unused * 5) + (multiQM * 8) + (badNames * 2)
}

As-Is:  calcScore(6, 14, 4, 3, 5) = 87
Target: calcScore(4,  6, 0, 0, 0) = 44
```

### Terminal Logger

Streams analysis steps to a terminal-style UI during agent execution.

```
tlog(text, class, prompt)  → appends a line with prompt glyph and styled text
tblank()                   → adds vertical spacing between log sections
tCursor()                  → appends a blinking cursor block (active state)
tRemCursor()               → removes cursor (when response received)
delay(ms)                  → Promise-based sleep for streaming effect
```

---

## 9. Design Patterns Used

### 1. Graceful Degradation

The Agent component works in two modes:

```
Try: Call Anthropic API
  └── Success: Render live AI response
  └── Failure: Load pre-computed VIOLATIONS[], DECISIONS[], TGT_QMS[]
                → User sees correct analysis regardless of API availability
```

### 2. Separation of Data and Presentation

All data is stored in clearly named constants at the top of each file. Rendering functions accept data as parameters and have no side effects beyond DOM updates.

```javascript
// Data (constants, defined once)
const ASIS_NODES = [ ... ]
const TARGET_NODES = [ ... ]
const PROPOSALS = [ ... ]
const TARGET_CSVS = { ... }

// Rendering (pure functions, no globals mutated)
function buildAsis(svgEl, clickFn) { ... }
function renderProp(idx) { ... }
function buildExport() { ... }
```

### 3. Event-Driven Navigation

Page switching is handled by a single `page(id, btn)` function that activates/deactivates DOM sections without any routing library.

```javascript
function page(id, btn) {
  document.querySelectorAll('.page').forEach(p => p.classList.remove('active'))
  document.querySelectorAll('.sb-btn').forEach(b => b.classList.remove('active'))
  document.getElementById('p-' + id).classList.add('active')
  if (btn) btn.classList.add('active')
  if (id === 'export') buildExport()  // lazy init
}
```

### 4. Lazy Initialisation

The Export tab's CSV previews are only built when the tab is first opened — not on page load. This keeps the initial render fast.

### 5. SVG Coordinate System

The topology diagram uses a fixed `viewBox="0 0 660 H"` coordinate system. All node positions are defined as centre points `{cx, cy}` with the node half-dimensions (NW=140, NH=46) computed at draw time.

```javascript
const ASIS_POS = {
  ORDER01:    { cx: 165, cy:  90 },
  PAYMENT01:  { cx: 460, cy:  90 },
  INVENTORY01:{ cx: 110, cy: 265 },
  LEGACY01:   { cx: 310, cy: 345 },
  BATCH01:    { cx: 510, cy: 235 },
  REPORT01:   { cx: 582, cy: 135 },
}
```

Edge attachment points are computed dynamically so arrows always meet the node boundary regardless of direction:

```javascript
function ep(cx, cy, tcx, tcy) {
  const dx = tcx - cx, dy = tcy - cy
  const t = Math.min(NW/2 / Math.abs(dx || 0.001), NH/2 / Math.abs(dy || 0.001))
  return { x: cx + dx * t, y: cy + dy * t }
}
```

---

## 10. Security Considerations

| Concern | Approach |
|---------|----------|
| **API key exposure** | No API key is stored in the HTML file. The Anthropic platform injects the key server-side. In standalone deployments, key management must be handled separately (environment variable, secrets manager). |
| **Data privacy** | All topology data is processed in the browser. No data is sent anywhere except to the Anthropic API during the agent call. |
| **Input validation** | The tool uses embedded mock data. In a future version with user CSV uploads, input sanitisation and size limits must be applied before parsing. |
| **XSS** | No user-supplied content is rendered as innerHTML. All dynamic content uses `textContent` or structured DOM creation. |
| **HTTPS** | The Anthropic API is called over HTTPS only. No plaintext API calls are made. |
| **Offline mode** | The fallback pre-computed data ensures the tool never requires a network call to be functional for demo purposes. |

---

## 11. Scalability & Extensibility

### Adding New Constraints

New architectural rules can be added by:

1. Adding the rule description to the system prompt in `runAgent()`
2. Adding detection logic to the pre-computed `VIOLATIONS[]` fallback array
3. Adding a corresponding proposal to `PROPOSALS[]` in the Validate tab

### Supporting Real CSV Upload

To support user-uploaded CSV files instead of embedded mock data:

```
1. Add <input type="file" accept=".csv"> elements to a new "Upload" tab
2. Use FileReader API to read file contents
3. Parse CSV using a simple split('\n') + split(',') parser
4. Replace ASIS_DATA constants with the parsed objects
5. Re-run buildAsis() and runAgent() with the new data
```

### Adding More QMs

The topology visualiser supports any number of QMs. Add entries to `ASIS_NODES` and `ASIS_POS` with `{cx, cy}` positions within the `viewBox="0 0 660 H"` coordinate space.

### Connecting to Live MQ

Future integration with IBM MQ REST API:

```
GET https://<mq-host>:9443/ibmmq/rest/v3/admin/qmgr          → Queue managers
GET https://<mq-host>:9443/ibmmq/rest/v3/admin/qmgr/{qm}/queue → Queues
GET https://<mq-host>:9443/ibmmq/rest/v3/admin/qmgr/{qm}/channel → Channels
```

The response data would be transformed into the CSV format and passed to the existing analysis pipeline.

---

*IBM MQ Hackathon 2025 · Built with IBM Plex · Powered by Anthropic Claude*
