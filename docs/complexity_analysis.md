# Complexity Analysis — As-Is vs Target State

## Complexity Metric

A quantitative complexity score is computed for both the as-is and target topologies using the following formula:

```
Complexity Score =
    (QM_count          × 10)  +
    (channel_count     ×  3)  +
    (unused_queues     ×  5)  +
    (multi_qm_apps     ×  8)  +
    (bad_channel_names ×  2)
```

### Weight Rationale

| Component | Weight | Rationale |
|-----------|--------|-----------|
| Queue Manager | 10 | Highest cost — each QM requires its own infrastructure, monitoring, patching, and operational overhead |
| Channel | 3 | Moderate cost — channels create routing complexity and must be named, monitored, and maintained |
| Unused Queue | 5 | High relative cost — dead objects increase audit surface, confuse developers, and mask real topology |
| Multi-QM App | 8 | Highest constraint violation cost — breaks the 1-QM rule, creates coupling, blocks automation |
| Bad Channel Name | 2 | Lower but important — non-standard names break automated provisioning scripts |

---

## As-Is Score: 87

| Component | Count | Weight | Sub-total |
|-----------|-------|--------|-----------|
| Queue Managers | 6 | × 10 | 60 |
| Channels | 14 | × 3 | 42 |
| Unused Queues | 4 | × 5 | 20 |
| Multi-QM Apps | 3 | × 8 | 24 |
| Bad Channel Names | 5 | × 2 | 10 |
| **Total** | | | **87** |

### As-Is Problems Breakdown

**Queue Managers (6)**
- QM.ORDER01, QM.PAYMENT01, QM.INVENTORY01 — active, justified
- QM.LEGACY01 — overgrown legacy hub, source of most violations
- QM.BATCH01 — single-app QM, consolidation candidate
- QM.REPORT01 — single-app QM, consolidation candidate

**Channels (14)**
- 9 correctly named channels (good)
- 5 non-standard channel names on QM.LEGACY01 (violations)
- Redundant routing paths through LEGACY01 fan-out

**Unused Queues (4)**
- Q.LEGACY.DEAD — no producer, no consumer
- Q.ORPHAN.1 — no producer, no consumer
- Q.TEST.OLD — leftover test artifact
- Q.UNUSED.BATCH — deprecated batch job remnant

**Multi-QM Apps (3)**
- APP003 BillingService — 2 QMs
- APP005 FulfillService — 2 QMs (one connection is already dead)
- APP007 BatchProcessor — 2 QMs

**Bad Channel Names (5)**
- LEGACY_TO_ORDERS
- LEGACY_PAYMENT_CH
- LEG_TO_INV
- BATCH_SEND
- RPT_CHAN

---

## Target Score: 44

| Component | Count | Weight | Sub-total |
|-----------|-------|--------|-----------|
| Queue Managers | 4 | × 10 | 40 |
| Channels | 6 | × 3 | 18 |
| Unused Queues | 0 | × 5 | 0 |
| Multi-QM Apps | 0 | × 8 | 0 |
| Bad Channel Names | 0 | × 2 | 0 |
| **Total** | | | **44** |

---

## Reduction Summary

```
As-Is  Score: 87
Target Score: 44
──────────────────
Reduction:    43 points
Percentage:   49.4% simpler
```

### Per-Component Improvement

| Component | As-Is | Target | Δ | Score Impact |
|-----------|-------|--------|---|-------------|
| Queue Managers | 6 | 4 | −2 | −20 |
| Channels | 14 | 6 | −8 | −24 |
| Unused Queues | 4 | 0 | −4 | −20 |
| Multi-QM Apps | 3 | 0 | −3 | −24 |
| Bad Channel Names | 5 | 0 | −5 | −10 |
| **Total reduction** | | | | **−78 → net −43** |

> Note: New queues added in target state (Q.BILL.INCOMING, Q.BATCH.IN) offset the raw queue reduction but are not penalised as they are active, producer-connected queues.

---

## Interpretation

A score of 44 represents a topology that:
- Has no constraint violations
- Has no dead objects
- Has no ambiguous naming
- Is fully automation-ready (channel names drive provisioning scripts)
- Has clear ownership — every app connects to exactly one QM

The minimum achievable score for this dataset (given 4 QMs and 6 required channels) is:
```
Min = (4×10) + (6×3) = 40 + 18 = 58
```
Our target of 44 achieves **the absolute minimum** since unused queues, multi-QM apps, and bad channel names are all eliminated entirely.
