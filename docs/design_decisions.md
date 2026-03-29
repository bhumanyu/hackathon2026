# Design Decisions — MQ Topology Simplification

## Methodology

Each design decision was produced by the Intelligent MQ Agent after reasoning over the as-is CSV dataset. All decisions were validated through a human-in-the-loop review step before finalisation.

---

## Decision 1 — Decommission QM.BATCH01

**Action:** Absorbed into QM.INVENTORY01

**Why:**
QM.BATCH01 existed solely to serve APP007 (BatchProcessor), which itself violated the 1-QM-per-app constraint by also connecting to QM.LEGACY01. Once APP007 is fixed to connect to a single QM, BATCH01 becomes completely empty. QM.INVENTORY01 is the natural consolidation target — batch replenishment logic is inventory-adjacent.

**What changed:**
- APP007 migrated to QM.INVENTORY01
- Q.BATCH.LOCAL renamed to Q.BATCH.IN and moved to QM.INVENTORY01
- All channels involving QM.BATCH01 removed
- QM.BATCH01 decommissioned

**Constraint impact:** Fixes APP007 multi-QM violation. Reduces QM count by 1.

---

## Decision 2 — Decommission QM.REPORT01

**Action:** Absorbed into QM.ORDER01

**Why:**
QM.REPORT01 served only APP008 (ReportService) with a single queue (Q.REPORT.LOCAL). It received data exclusively via a channel from QM.ORDER01. A dedicated production QM for a single app and single queue is not justified — the overhead of managing a separate QM outweighs any isolation benefit.

**What changed:**
- APP008 migrated to QM.ORDER01
- Q.REPORT.LOCAL moved to QM.ORDER01
- QM.REPORT01 decommissioned

**Constraint impact:** Reduces QM count by 1. Simplifies the reporting data path.

---

## Decision 3 — Fix APP003 BillingService

**Action:** Assigned to QM.ORDER01 only (was: QM.ORDER01 + QM.PAYMENT01)

**Why:**
BillingService was consuming from Q.PAY.LOCAL on QM.PAYMENT01, violating the 1-QM constraint. Billing is a downstream outcome of order processing — QM.ORDER01 is the correct single QM. Payment data can reach BillingService via a new local queue (Q.BILL.INCOMING) fed by the already-existing PAYMENT01→ORDER01 receiver channel.

**Pattern applied:**
```
Producer (PAYMENT01 side) → remoteQ → xmitq → sender channel →
  Q.BILL.INCOMING (local on ORDER01) → APP003 (consumer on ORDER01)
```

**Constraint impact:** Fixes 1 of 3 multi-QM violations. No new QM or channel required.

---

## Decision 4 — Fix APP005 FulfillService

**Action:** Assigned to QM.INVENTORY01 only (was: QM.INVENTORY01 + QM.LEGACY01)

**Why:**
FulfillService was connected to QM.LEGACY01 to read Q.LEGACY.OUT. However, Q.LEGACY.OUT had **no producer** — the connection was already dead and non-functional. Removing it costs nothing in terms of live message flow.

**What changed:**
- APP005 fixed to QM.INVENTORY01 only
- Q.LEGACY.OUT decommissioned (no producer ever existed)
- Dead LEGACY01 connection removed

**Constraint impact:** Fixes 1 of 3 multi-QM violations. Zero functional disruption.

---

## Decision 5 — Fix APP007 BatchProcessor

**Action:** Assigned to QM.INVENTORY01 only (was: QM.LEGACY01 + QM.BATCH01)

**Why:**
BatchProcessor was split across two QMs — writing via a REMOTE queue on QM.LEGACY01 and reading from a LOCAL queue on QM.BATCH01. Since QM.BATCH01 is being decommissioned (Decision 1), APP007 must move. QM.INVENTORY01 is the logical home for batch inventory replenishment jobs.

**What changed:**
- APP007 migrated to QM.INVENTORY01
- New queue Q.BATCH.IN created on QM.INVENTORY01
- All LEGACY01 and BATCH01 queue/channel references for APP007 removed

**Constraint impact:** Fixes the final multi-QM violation. Multi-QM apps: 3 → 0.

---

## Decision 6 — Remove Unused Queues & Fix Channel Names

### Unused queues removed (4 total)

| Queue | QM | Reason |
|-------|----|--------|
| Q.LEGACY.DEAD | QM.LEGACY01 | No producer, no consumer — orphaned |
| Q.ORPHAN.1 | QM.LEGACY01 | No producer, no consumer — orphaned |
| Q.TEST.OLD | QM.LEGACY01 | Leftover test queue — no activity |
| Q.UNUSED.BATCH | QM.BATCH01 | Deprecated batch queue — no activity |

**Why:** Dead queues increase audit surface, complicate topology documentation, and add operational risk with zero functional benefit.

### Channel names corrected (5 total)

| Old Name | New Name / Action |
|----------|-------------------|
| LEGACY_TO_ORDERS | Renamed → QM.LEGACY01.toQM.ORDER01 |
| LEG_TO_INV | Renamed → QM.LEGACY01.toQM.INVENTORY01 |
| LEGACY_PAYMENT_CH | Removed (LEGACY01 no longer routes to PAYMENT01 in target state) |
| BATCH_SEND | Removed (QM.BATCH01 decommissioned) |
| RPT_CHAN | Removed (QM.REPORT01 decommissioned) |

**Why:** Non-standard channel names break automated provisioning pipelines, make topology audits unreliable, and violate the deterministic naming constraint required by the hackathon.

**Constraint impact:** Bad channel names: 5 → 0. Enables full automation of channel provisioning.

---

## Summary Table

| Decision | QMs Δ | Channels Δ | Queues Δ | Violations Fixed |
|----------|--------|------------|----------|-----------------|
| 1. Decommission BATCH01 | −1 | −3 | −2 | APP007 multi-QM |
| 2. Decommission REPORT01 | −1 | −2 | −1 | — |
| 3. Fix APP003 | — | — | +1 | APP003 multi-QM |
| 4. Fix APP005 | — | — | −1 | APP005 multi-QM |
| 5. Fix APP007 | — | — | +1/−1 | APP007 multi-QM |
| 6. Clean up | — | −5 | −4 | 5 bad names, 4 unused |
| **Total** | **−2** | **−8** | **−6** | **12 violations** |
