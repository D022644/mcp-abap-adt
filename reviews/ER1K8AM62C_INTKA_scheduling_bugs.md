# Deep Code Review: Transport `ER1K8AM62C`

**Transport:** ER1K8AM62C — *"INTKA: Repair small scheduling bugs"*
**Developer:** MANTEL | **Released:** 2026-05-04 | **Target:** /SSCB_S4P/
**Review Date:** 2026-05-20
**Classification:** Critical path — HANA MRP planning engine (S/4HANA AMDP)
**If this breaks:** MRP run produces wrong planned order dates for materials with periodic lot sizing and INTKA=1 (planning calendar interpretation mode)

---

## Objects in Transport

| Class | Changed Methods |
|---|---|
| `CL_PPH_READ_MASTER` | `CALC_FIXTR_PERIODIC`, `CALC_TIME_PHASED_PARAMETERS`, `READ_MAT_CTRL_PARAMS`, `READ_MAT_MRP_CTRL_PARAMS` |
| `CL_PPS4_MRP_LOTSIZING` | `LOT_SIZE_CALC` |
| `CL_PPS4_MRP_CALC` | `PREPARE_DOCUMENTS`, `CREATE_INHOUSE_PRODUCTION`, `CREATE_SUBCONTRACING`, `CREATE_EXTERNAL_DOCUMENTS`, `CREATE_STOCK_TRANSFER` |
| `CL_PPS4_MRP_CHECKS` | `READ_AND_CHECK` |
| `CL_PPS4_MRP_EXTPROC` | `EXT_PROC_HANDLE`, `EXT_PROC_SCHED`, `EXT_PROC_SCHED_STD` |
| `CL_PPS4_MRP_INHPROD` | `INHOUSE_HANDLE`, `PLAF_SCHED`, `PLAF_SCHED_STD` |
| `CL_PPS4_MRP_SUBCON` | `SUBCON_HANDLE` |

All methods are **AMDP (ABAP Managed Database Procedures)** in HDB SQLScript — they execute directly on the HANA database engine.

---

## Phase 1: Context

INTKA is the "Interpretation of planning calendar events" field (domain `INTKA`, package `MD`). Value `'1'` means: the planned receipt date (`avail_date`) **is** the delivery/sourcing date — i.e., the period start date is the procurement date, not the requirements date. This inverts the normal backward-scheduling logic.

The transport introduces a consistent `intka_active` flag propagated from lot sizing through to order creation, replacing ad-hoc `intka = '1'` checks scattered across methods.

---

## Findings

### [HIGH] [LIKELY]: `lt_net_req_lfdat` — LFDAT Only Calculated for Two Lot Size Codes, Missing Coverage

**Location:** `CL_PPS4_MRP_LOTSIZING=>LOT_SIZE_CALC` ~line 802
**Code:**
```sql
lt_net_req_lfdat = select req.*,
    case when ( req.loskz = 'K' AND req.intka = '1' )
           or ( req.lglkz = 'K' AND req.lgint = '1' ) then
              TO_DATS(add_workdays(req.fabkl, req.dat00, -req.webaz, CURRENT_SCHEMA))
         end as lfdat
    from :lt_net_req_disls as req;
```
**Problem:** `lfdat` is only computed for `loskz='K'` (period lot size) with `intka='1'` OR `lglkz='K'` with `lgint='1'`. For all other lot size procedures, `lfdat` is `NULL`. Downstream in `lt_net_req_multi_period`, `lfdat` is used as the join key to the period table for INTKA-active rows:
```sql
AND per_lf.day_date = req.lfdat
```
**Break Scenario:** If a material has `intka='1'` but uses a lot size procedure other than `'K'` (e.g., `'WB'` = replenish to maximum, `'FX'` = fixed lot size), the `lfdat` is `NULL`, the LEFT OUTER JOIN finds no period match, and the INTKA date logic silently falls back to `dat00` — producing an incorrect (non-INTKA-adjusted) order date. Not caught by any error handler.

**Mitigation:** The comment implies this is intentional — `lfdat` is only relevant for periodic lot sizes (`loskz='K'`). Whether `intka='1'` can validly coexist with non-periodic lot sizes is a business configuration question. If it can, this is a silent data quality bug.

---

### [HIGH] [LIKELY]: INTKA Period JOIN in `lt_terbv` Uses `f.lfdat` — But `lfdat` May Be NULL After Forward Scheduling

**Location:** `CL_PPS4_MRP_EXTPROC=>EXT_PROC_SCHED_STD` ~line 1408
**Code:**
```sql
and  ( ( p.day_date = f.lfdat and m.intka = '1')
  or ( p.day_date = f.avail_date and m.intka <> '1') )
```
**Problem:** The variable `f` comes from `lt_forw_result` (output of `EXT_PROC_SCHED_FORW_STD`). If `EXT_PROC_SCHED_FORW_STD` does not populate `lfdat` for all rows, the condition `p.day_date = f.lfdat` evaluates to `p.day_date = NULL`, which is always FALSE in SQL — the period is never joined for INTKA rows that went through forward scheduling, and the fallback `f.avail_date` is used instead.

**Break Scenario:** Materials that trigger forward scheduling (in-past requirements) with `intka='1'` may get the wrong period matched, leading to incorrect `avail_date` adjustment.

**Verification needed:** Confirm whether `EXT_PROC_SCHED_FORW_STD` always populates `lfdat` for rows with `intka='1'`.

---

### [MEDIUM] [LIKELY]: `CREATE_STOCK_TRANSFER` — INTKA Zeroes Out `webaz` Contingent on `intka_active` Propagation

**Location:** `CL_PPS4_MRP_CALC=>CREATE_STOCK_TRANSFER` ~line 1523
**Code:**
```sql
case when intka.intka_active = '1'
  then 0
  else l.webaz
end as webaz,
```
**Problem:** For INTKA-active stock transfer requisitions, the GR processing time (`webaz`) is forced to 0. This is correct per INTKA semantics (avail_date = delivery date, webaz already subtracted in scheduling). However, if `it_ctrl_lot_periodic` does not correctly propagate `intka_active='1'` for stock transfer scenarios (e.g., if the periodic lot sizing step was skipped), `webaz` will NOT be zeroed, and the delivery date will shift by the GR processing time — double-counting the webaz already accounted for in scheduling.

**Mitigation:** `it_ctrl_lot_periodic` is always populated upstream in `PREPARE_DOCUMENTS` via `LOT_SIZE_CALC`. Unless `LOT_SIZE_CALC` is bypassed, this should be safe. Same pattern exists in `CREATE_INHOUSE_PRODUCTION` and `CREATE_EXTERNAL_DOCUMENTS`.

---

### [MEDIUM] [PROVEN]: `PLAF_SCHED_BACK_INTKA` — Non-Working Day Offset Applied to Planning Calendar Dates

**Location:** `CL_PPS4_MRP_INHPROD=>PLAF_SCHED_BACK_INTKA` ~line 2260
**Code:**
```sql
case
  when m.dzeit > 0 then
    case
      when add_workdays(m.fabkl, m.pedtr, 0, :iv_schema) = m.pedtr
        then add_workdays(m.fabkl, m.pedtr, -m.dzeit, :iv_schema)
      else
        add_workdays(m.fabkl, m.pedtr, -m.dzeit -1, :iv_schema)
    end
  else m.pedtr
end as psttr
```
**Problem:** The `-dzeit - 1` branch adjusts for `pedtr` falling on a non-working day. For INTKA mode, `pedtr = avail_date` (the planning calendar delivery date). Planning calendar dates **are** working days by definition. The non-working day branch should therefore never be reached — but if a misconfigured factory calendar causes `pedtr` to appear as a non-working day, the production start date (`psttr`) will be calculated one extra day too early.

---

### [MEDIUM] [LIKELY]: `CALC_FIXTR_PERIODIC` — Lot-to-Lot Horizon Type Excluded, INTKA in MTO Segments May Not Get Periodic Fixtr

**Location:** `CL_PPH_READ_MASTER=>CALC_FIXTR_PERIODIC` ~line 447
**Code:**
```sql
et_fixtr_periodic = SELECT matnr, berid, loskz, terbv,
    case when terbv = '3' or terbv = '4'
           then per_last_workday
         when day_date = per_first_workday
           then per_first_workday
         else TO_DATS(add_workdays(fabkl, per_last_workday, 1, CURRENT_SCHEMA))
    end as fixtr_periodic
  from :lt_fxhor_period
    where horizon_type > '0';   -- excludes lot-to-lot
```
**Problem:** The `WHERE horizon_type > '0'` filter excludes lot-to-lot entries (`horizon_type = '0'`). These cover materials in sales order (`plaab IN ('20','22')`) planning segments without customer-specific lot sizing. For these, `fixtr_periodic` is absent from the output, and `READ_MAT_MRP_CTRL_PARAMS` uses the non-periodic `c.fixtr` instead.

This is likely intentional for lot-to-lot. However, if INTKA is expected to work in MTO scenarios, these materials silently skip periodic fixtr calculation.

---

### [LOW] [SPECULATIVE]: `CALC_TIME_PHASED_PARAMETERS` — `add_workdays(..., -0, ...)` Is a No-Op

**Location:** `CL_PPH_READ_MASTER=>CALC_TIME_PHASED_PARAMETERS` ~line 494
**Code:**
```sql
to_dats(add_workdays( ctrl.fabkl, add_workdays( ctrl.fabkl, rhy.new_rhdat, -0, CURRENT_SCHEMA ), ctrl.mtwzt, CURRENT_SCHEMA ))
```
**Problem:** `add_workdays(..., -0, ...)` is arithmetically identical to `add_workdays(..., 0, ...)`. Negative zero is the same as zero. The intent appears to be "normalize to next working day", which `0` achieves. The `-0` is confusing to readers but harmless.

---

### [LOW] [LIKELY]: `READ_MAT_MRP_CTRL_PARAMS` — Redundant `NO_JOIN_THRU_AGGR` Hint

**Location:** `CL_PPH_READ_MASTER=>READ_MAT_MRP_CTRL_PARAMS` ~line 391
**Code:**
```sql
WITH HINT ( NO_JOIN_THRU_AGGR )
```
The same hint already appears on the `lt_ctrl_cds` query earlier in the same method. Redundant hints are harmless but indicate copy-paste without review.

---

## Summary Table

| # | Severity | Evidence | Finding |
|---|----------|----------|---------|
| 1 | HIGH | LIKELY | `lfdat` only computed for `loskz='K'`; INTKA materials with other lot sizes silently fall back to `dat00` |
| 2 | HIGH | LIKELY | INTKA period JOIN in `lt_terbv` uses `f.lfdat` which may be NULL after forward scheduling |
| 3 | MEDIUM | LIKELY | INTKA zeroes `webaz` in stock transfer — depends on `intka_active` propagation being correct |
| 4 | MEDIUM | PROVEN | `PLAF_SCHED_BACK_INTKA` applies `-dzeit-1` for non-working pedtr; planning calendar dates are always working days but misconfiguration could trigger this |
| 5 | MEDIUM | LIKELY | `CALC_FIXTR_PERIODIC` skips lot-to-lot horizon type — INTKA in MTO segments may not get periodic fixtr |
| 6 | LOW | SPECULATIVE | `add_workdays(..., -0, ...)` is a no-op, confusing but harmless |
| 7 | LOW | LIKELY | Redundant `NO_JOIN_THRU_AGGR` hint in `READ_MAT_MRP_CTRL_PARAMS` |

---

## Positive Observations

- **Clean `intka_active` propagation architecture:** The new flag `intka_active` (derived in `LOT_SIZE_CALC`) is consistently passed through `it_ctrl_lot_periodic` to all document creation methods. Significant improvement over ad-hoc `intka='1'` checks scattered through scheduling code.
- **INTKA-specific scheduling path isolated:** `PLAF_SCHED_BACK_INTKA` correctly implements the "avail_date = pedtr" semantic as a dedicated procedure, cleanly separated from standard backward scheduling.
- **Period JOIN conditions are consistent:** The `ON` clause for joining `it_period_tab` uses the same multi-condition pattern (`disls`, `werks`, `fabkl`, `periv`, `mrppp`, `day_date`, `mtord`) consistently across `LOT_SIZE_CALC`, `EXT_PROC_SCHED_STD`, and `PLAF_SCHED_STD`, reducing the risk of subtle period-matching bugs.
- **`terbv_active` correctly isolated from `intka_active`** in `lt_lots_terbv` vs `lt_lots_sourcing_date` UNION — these two date-adjustment modes are kept separate, preventing interaction bugs.
- **`CALC_FIXTR_PERIODIC` guards its INNER JOIN correctly** — only materials with `losvf='P'` (periodic lot sizing) AND valid `fixtr > 0` enter the computation, preventing null-date propagation.
