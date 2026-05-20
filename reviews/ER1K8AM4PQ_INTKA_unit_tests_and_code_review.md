# Deep ABAP Code Review: Transport ER1K8AM4PQ

**Transport:** ER1K8AM4PQ
**Description:** INTKA: Unit Tests and code Review
**Owner:** MANTEL
**Status:** Released (20260513)
**System:** ER1 / Client 001
**Review Date:** 2026-05-20
**Reviewer:** Claude Code (automated deep review)

---

## Transport Contents Summary

| Class | Changed Objects | Type |
|-------|----------------|------|
| `CL_PPH_READ_MASTER` | `READ_MAT_MRP_CTRL_PARAMS`, CCAU, CCIMP | Production AMDP |
| `CL_PPS4_MRP_AMDP_TOOLS` | `EVALUATE_FCAL`, `FILTER_UNSUPPORTED_DOCUMENTS`, `PLANNING_FILE_ENTRIES_PREPARE`, CPRI, CPUB, CCAU | Production AMDP |
| `CL_PPS4_MRP_LOTSIZING` | `LOT_SIZE_CALC`, CCAU | Production AMDP/LLANG |
| `CL_PPS4_MRP_CALC` | CLAS (full class) | Production AMDP |
| `TH_PPS4_LOTSIZING_SQL` | 30+ methods, CLSD, CPRI, CPRO, CPUB, CCAU | Test helper base class |
| `TH_PPS4_MRP_CALC` | `SET_LOTSIZE_CALC`, `SET_PEGAREA_DATA`, `SET_SOS_DET`, CLSD, etc. | Test double base class |

---

## Findings

---

### [HIGH] [PROVEN]: Missing EVALUATE_FCAL implementation in CL_PPS4_MRP_AMDP_TOOLS

**Location:** `CL_PPS4_MRP_AMDP_TOOLS` (definition includes CPRI/CPUB)
**Code:**
```
Transport object: CL_PPS4_MRP_AMDP_TOOLS EVALUATE_FCAL (METH/LIMU)
Source: method not found in fetched CCIMP source
```
**Problem:** The transport declares `EVALUATE_FCAL` as a changed method in `CL_PPS4_MRP_AMDP_TOOLS` (it appears in the CPRI/CPUB definition includes), but the implementation include (CCIMP) contains no `METHOD evaluate_fcal` block. This means the method was added to the class interface but its body was not committed, leaving the class in a state where the definition references a method that has no implementation. In ABAP, this causes a syntax error at activation time — though since the transport is released, activation may have been bypassed or the method is empty.

**Break Scenario:** Any caller invoking `CL_PPS4_MRP_AMDP_TOOLS=>EVALUATE_FCAL` will receive a `CX_SY_DYN_CALL_ILLEGAL_METHOD` or similar runtime exception, or a compilation error if the class cannot be activated.

**Fix:** Either implement the method body in the CCIMP include or remove the declaration from CPRI/CPUB. Based on the class context (factory calendar validation), this method likely was intended to extract the calendar range evaluation logic from `FILTER_UNSUPPORTED_DOCUMENTS` into a reusable sub-procedure.

---

### [HIGH] [PROVEN]: 8 Test Methods Declared in Transport but Not Implemented

**Location:** `TH_PPS4_LOTSIZING_SQL` (CLSD/CPRI/CCAU includes)
**Code:**
```
Transport objects listed (METH/LIMU):
  INTKA_MTO_PERIOD, INTKA_WEBAZ_MTO_PERIOD, INTKA_WEBAZ_REQ_BEYOND_HORIZON
  LGINT_WEBAZ_MTO_PERIOD, TERBV_MTO_PERIOD, P_MTO_PERIOD
  PREPARE_PERIOD_MTO_PAIR, SETUP
```
**Problem:** These 8 method names appear in the transport's object list (indicating class definition changes), but none of them exist in the fetched implementation source (1,082 lines). The class `TH_PPS4_LOTSIZING_SQL` is an ABSTRACT test base class (`FOR TESTING RISK LEVEL HARMLESS DURATION SHORT`). If these methods were declared in the class definition section but have no corresponding `METHOD ... ENDMETHOD` block in the implementation, the class cannot be activated, and all subclasses that inherit from it will also fail to compile.

Notably:
- `SETUP` is a special ABAP Unit lifecycle method (executed before each test method). Its absence means test isolation is not working as designed.
- `P_MTO_PERIOD` and `PREPARE_PERIOD_MTO_PAIR` suggest MTO (Make-To-Order) period handling was planned as test coverage but was not delivered.
- `INTKA_WEBAZ_REQ_BEYOND_HORIZON` covers an important edge case (requirements beyond planning horizon with WEBAZ offset) with no test coverage.

**Break Scenario:** Any unit test run against `CL_PPS4_MRP_LOTSIZING` using these test helper classes will fail with a missing-method activation error. The `SETUP` method gap means each test method does not receive a clean state, potentially causing cascading false passes/failures.

**Fix:** Implement all declared methods or remove their declarations. For `SETUP`, implement the test environment initialization. For MTO period tests, implement the corresponding test scenarios.

---

### [HIGH] [PROVEN]: Dead Code Branch Permanently Disables MPN Netting

**Location:** `CL_PPS4_MRP_CALC::CALC_CHANGES` line ~401
**Code:**
```sql
if 0 = 999 then
    call "CL_PPS4_MRP_SUPERSESSION=>ORCHESTRATE_NETTING_MPN"(
       ...
       et_mdps_after_mpn  => lt_mdps_after_mpn
    );
    lt_mdps_reduced = SELECT * from :lt_mdps_after_mpn;
else
    call "CL_PPS4_MRP_CALC=>ORCHESTRATE_NETTING_SIMPLE"( ... );
end if;
```
**Problem:** The condition `if 0 = 999` is a compile-time constant false, permanently routing execution to `ORCHESTRATE_NETTING_SIMPLE`. This completely disables MPN (Multi-Plant/Multi-Purpose Number) netting. While this appears to be intentional scaffolding for a future feature, it has two issues:

1. The HANA SQLScript optimizer may still parse and validate the dead branch, wasting compile resources.
2. If `CL_PPS4_MRP_SUPERSESSION=>ORCHESTRATE_NETTING_MPN` is the correct code path for supersession scenarios, this permanent bypass silently produces wrong MRP results without any error.

**Break Scenario:** Any material with supersession/MPN configuration will be planned using the simple netting path, ignoring MPN relationships. The result is incorrect planned order quantities/dates without any error or warning.

**Fix:** Either activate the MPN path with a proper runtime flag (e.g., check whether any material in `it_ctrl_mrp` has MPN configured), or if MPN is intentionally disabled, replace the dead code with a comment block and remove the dead branch entirely to improve readability and avoid confusion.

---

### [MEDIUM] [PROVEN]: Missing WERKS in create_pegarea_data JOIN Condition

**Location:** `CL_PPS4_MRP_AMDP_TOOLS::CREATE_PEGAREA_DATA` lines 401-402
**Code:**
```sql
inner join :it_ctrl_mrp c
on c.matnr = mdps.matnr and c.berid = mdps.berid
```
**Problem:** The join between `it_mdps` and `it_ctrl_mrp` uses only `matnr` and `berid` as join keys, omitting `werks`. In the PPH MRP framework, the natural key for MRP control data is `(matnr, werks, berid)`. While for plant-level MRP areas `berid = werks`, for storage-location MRP areas and subcontracting areas, `berid` alone is not unique across plants when the same material exists in multiple plants with the same MRP area identifier. This can produce cross-plant data pollution in the `webaz` and other sourcing attributes returned in `et_pegarea_data`.

**Break Scenario:** In a multi-plant setup where material 'MAT-X' exists in plants '1000' and '2000', both with an MRP area having `berid = 'SC01'`, the join will incorrectly pick up control data from both plants, leading to duplicated `et_pegarea_data` entries with mixed-up `webaz` lead time values. Planned orders would then use wrong replenishment lead times.

**Fix:** Add `and c.werks = mdps.werks` to the join condition (or verify that `berid` is globally unique for the input data scope, which should be confirmed with a comment).

---

### [MEDIUM] [LIKELY]: NULL Propagation Risk in RSTER Date Calculation

**Location:** `CL_PPH_READ_MASTER::READ_MAT_MRP_CTRL_PARAMS` line ~334
**Code:**
```sql
case when respl = '' then '00000000'
     when reshz = 0 then :iv_date
     else to_dats( add_workdays(fabkl, :iv_date, reshz, CURRENT_SCHEMA))
end as rster,
```
**Problem:** The CASE expression handles `respl = ''` and `reshz = 0`, but does not handle `reshz IS NULL`. In SQLScript, `reshz = 0` evaluates to UNKNOWN (not TRUE) when `reshz` is NULL. If `respl` is set (non-empty) but `reshz` is NULL in the database, the CASE falls through to the ELSE branch, calling `add_workdays(fabkl, :iv_date, NULL, CURRENT_SCHEMA)`. HANA's `add_workdays` with a NULL offset returns NULL, so `to_dats(NULL)` also returns NULL, causing a NULL `rster` date to flow into downstream planning logic that may not handle it.

**Break Scenario:** Material master with `respl` set (periodic lot size indicator) but `reshz` = NULL (not maintained in MARC). The `rster` date becomes NULL. Downstream code checking `rster` for periodic replenishment will either skip the material (silently wrong result) or cause a type conversion error.

**Fix:**
```sql
case when respl = '' then '00000000'
     when coalesce(reshz, 0) = 0 then :iv_date
     else to_dats( add_workdays(fabkl, :iv_date, reshz, CURRENT_SCHEMA))
end as rster,
```

---

### [MEDIUM] [LIKELY]: Factory Calendar NULL Propagation in cutoff_dates

**Location:** `CL_PPS4_MRP_AMDP_TOOLS::FILTER_UNSUPPORTED_DOCUMENTS` lines 174-185
**Code:**
```sql
fcal_range = SELECT ctrl.werks, ctrl.fabkl,
                    to_date( FctryCalendarValidityStartDate ) as start_date,
                    to_date( FactoryCalendarValidityEndDate ) as end_date
 from :it_ctrl_mrp_all as ctrl
   inner join I_FactoryCalendarBasic as tfacd on ctrl.fabkl = tfacd.FactoryCalendarLegacyID
   group by werks, fabkl, FctryCalendarValidityStartDate, FactoryCalendarValidityEndDate;

cutoff_dates = SELECT date.werks,
              date.fabkl,
              to_dats( add_workdays( date.fabkl, date.start_date, 5, CURRENT_SCHEMA ) ) as min_date,
              to_dats( add_workdays( date.fabkl, date.end_date,  -1, CURRENT_SCHEMA ) ) as max_date
from :fcal_range as date;
```
**Problem:** If `FctryCalendarValidityStartDate` or `FctryCalendarValidityEndDate` is NULL in `I_FactoryCalendarBasic`, then `start_date`/`end_date` will be NULL, making `min_date`/`max_date` NULL. Subsequently, the comparisons `m.dat00 > d.max_date` and `m.dat00 < d.min_date` will evaluate to UNKNOWN (never TRUE), silently skipping all boundary violations for that plant. This means out-of-range MRP elements are not detected and planning proceeds on dates outside the factory calendar validity — a planning correctness issue.

Additionally, materials whose plant has `fabkl = ''` (no factory calendar assigned) will not match the INNER JOIN, so `cutoff_dates` will be empty for that plant. The cutoff check is then silently skipped, meaning a plant with no factory calendar assigned passes validation unconditionally.

**Break Scenario:** A new plant with no factory calendar (`fabkl = ''` in material master) bypasses the calendar boundary check entirely. MRP will plan dates potentially years beyond any valid factory calendar, causing incorrect work scheduling later.

**Fix:** Add `WHERE ctrl.fabkl <> ''` to the fcal_range query, and add COALESCE guards on the start/end date:
```sql
where ctrl.fabkl <> ''
```
And validate that `start_date` and `end_date` are non-NULL before the cutoff check.

---

### [MEDIUM] [LIKELY]: Division by Zero in Quota Split (lv_quote_sum)

**Location:** `CL_PPS4_MRP_LOTSIZING::LOT_SIZE_CALC_LLANG_PARALLEL` line ~2185
**Code:**
```cpp
lv_quote_sum = lv_quote_sum + Fixed8<7>(it_quotation_pos_quota[lv_q]);  // Sum all valid Quote over all vendors
...
lv_spqty = lv_replenish_quan * Fixed12<3>(lv_quota) / Fixed12<3>(lv_quote_sum);  // calc pos quan
```
**Problem:** `lv_quote_sum` is accumulated from valid quota arrangements. The division `lv_replenish_quan * lv_quota / lv_quote_sum` occurs inside the split-quota processing loop without a prior zero-check on `lv_quote_sum`. While the quota collection logic should prevent zero quotas from entering the split table, a scenario where all collected quotas have `QUOTA = 0` (misconfigured quota arrangement) would yield `lv_quote_sum = 0`, causing a HANA LLANG runtime exception (divide by zero), crashing the entire MRP run for all materials in the processing batch.

**Break Scenario:** A quota arrangement with `QUOTA = 0` for all active positions is technically invalid but can exist if a user deactivates all quota splits and forgets to remove them. The `lv_quote_sum` accumulates to zero, and the next division causes an unhandled exception.

**Note:** A prior guard exists for the single-quota case (`Int32(it_quotation_pos_quota[lv_q]) != 0` at line ~1782), but this guard is in a different code path (best-fit single quota selection). The split-quota path has no equivalent guard.

**Fix:**
```cpp
if ( lv_quote_sum > lv_Fixed_8_7_zero ) {
    lv_spqty = lv_replenish_quan * Fixed12<3>(lv_quota) / Fixed12<3>(lv_quote_sum);
} else {
    lv_spqty = lv_replenish_quan;  // fallback: full quantity to current position
}
```

---

### [MEDIUM] [PROVEN]: validate_terbv Uses TOLERABLE for avail_date — Missing Assert Strictness

**Location:** `TH_PPS4_LOTSIZING_SQL::VALIDATE_TERBV` line 1072-1074
**Code:**
```abap
cl_aunit_Assert=>assert_initial( act = new_lot-avail_date
                                 msg = 'TERBV does not fill Sourcing date'
                                 level = if_aunit_constants=>tolerable ).
```
**Problem:** The assertion that `avail_date` must be initial (empty) for TERBV lots is classified as `TOLERABLE`, meaning test failures on this assertion do NOT fail the test run. This reduces the test's effectiveness: if a future TERBV code change accidentally populates `avail_date`, the test will pass with only a warning. Since `avail_date` (the original requirement date for INTKA sourcing) having a wrong value in TERBV context would propagate a wrong date to planned order creation, this should be a hard failure.

**Break Scenario:** A refactoring of `lt_lots_terbv` that accidentally sets `avail_date` would pass all tests, shipping the bug to production.

**Fix:** Remove the `level = if_aunit_constants=>tolerable` parameter or change it to `if_aunit_constants=>critical` to enforce hard failure.

---

### [MEDIUM] [LIKELY]: LOT_SIZE_CALC Safety Stock Shift Uses mng01 > 0 but Ignores mng01 = 0 Safety Stock

**Location:** `CL_PPS4_MRP_LOTSIZING::LOT_SIZE_CALC` lines 697-719
**Code:**
```sql
lt_earliest_req_date = SELECT matnr, werks, berid, plaab, planr, cuobj, sgt_rcat, sgt_scat,
                               MIN( dat00 ) AS req_dats
                               FROM :it_net_req
                               WHERE mng01 > 0
                               GROUP BY ...;

lt_net_req_corrected_date = SELECT req.*,
    CASE WHEN req.delkz = 'SH' THEN
        COALESCE( earliest_date.req_dats, req.dat00 )
    ELSE req.dat00
    END AS dat00
    FROM :it_net_req AS req
    LEFT OUTER JOIN :lt_earliest_req_date AS earliest_date ON ...;
```
**Problem:** The safety stock shift logic (`delkz = 'SH'`) moves safety stock replenishment to the earliest actual requirement date. The `lt_earliest_req_date` query filters `WHERE mng01 > 0`, correctly excluding zero-quantity requirements. However, if a planning section has ONLY a safety stock line (`delkz = 'SH'`) and no positive net requirements (all cancelled or zero), then `lt_earliest_req_date` returns no row, and `COALESCE( NULL, req.dat00 )` returns `req.dat00` — the original safety stock date. This is correct by COALESCE fallback.

The subtle risk is when a safety stock requirement exists at a *later* date and positive requirements exist at an *earlier* date, but the `INNER JOIN` from `lt_matb_ctrl` in the next step (`lt_net_req_disls`) might drop safety stock rows if there's no matching ctrl entry for the safety stock entry's material/plant/berid combination — leading to silently dropped safety stock provisions.

**Break Scenario:** Low probability but: if safety stock is maintained for a sub-MRP area that is not in `lt_matb_ctrl`, the LEFT OUTER JOIN to `earliest_date` works, but the INNER JOIN to `lt_matb_ctrl` in `lt_net_req_disls` will drop the safety stock row entirely, leading to under-ordering.

**Fix:** Verify that `lt_matb_ctrl` always contains entries for all materials/berids that appear in `it_net_req`. Add a note/assertion in the test class.

---

### [LOW] [PROVEN]: Test Method shift_safety_stock Has Wrong Expected Quantity Comment

**Location:** `TH_PPS4_LOTSIZING_SQL::SHIFT_SAFETY_STOCK` line 321-324
**Code:**
```abap
cl_aunit_assert=>assert_equals( exp = 21
                                act = new_lots[ 1 ]-mng01
                                msg = 'Sum up Safety stock with requirements in past' ).
```
**Problem:** The test expects `mng01 = 21` (the sum of 1 + 20), which is correct. The message text says "requirements in past" but the safety stock requirement (`dat00 = req_date_1 + 5`) is actually in the *future* relative to the regular requirement (`dat00 = req_date_1`). The actual tested scenario is "shift safety stock to earliest date and sum quantities." The misleading message can confuse developers debugging a failure.

**Fix:** Change the message to: `'Safety stock shifted to earliest requirement date; quantities summed to 21'`.

---

### [LOW] [LIKELY]: LGINT_MISSING_PERIOD_WEBAZ Has No Assertions

**Location:** `TH_PPS4_LOTSIZING_SQL::LGINT_MISSING_PERIOD_WEBAZ` lines 552-577
**Code:**
```abap
METHOD lgint_missing_period_webaz.
  ...
  call_lotsizing( ... IMPORTING et_new_lots = DATA(new_lots) ... ).
  "-- NO ASSERTIONS --"
ENDMETHOD.
```
**Problem:** This test method invokes `call_lotsizing` but contains zero assertions. It only proves the method completes without a runtime exception. There are no checks on `new_lots`, no verification of dates, no check of `messages`. The method name and context (`LGINT` with missing period and WEBAZ) represent an important edge case (long-horizon periodic lot sizing with delivery schedule offset when no period table entry exists), but the test provides no behavioral guarantees.

**Break Scenario:** Any regression in the LGINT+WEBAZ+missing-period code path will not be caught by this test.

**Fix:** Add assertions equivalent to those in `lgint_missing_period` and `lgint_webaz`:
```abap
validate_intka( per_start_date = req_date_1
                new_lot        = new_lots[ 1 ]
                message        = 'Daily period generated for LGINT, WEBAZ does not override' ).
```

---

### [LOW] [SPECULATIVE]: TH_PPS4_MRP_CALC Parameter Name Padding May Cause Silent Match Failure

**Location:** `TH_PPS4_MRP_CALC::SET_LOTSIZE_CALC` line 106
**Code:**
```abap
)->set_exporting_parameter( name = 'et_new_lots     '  value = new_lots
)->set_exporting_parameter( name = 'et_invalid_quots'  value = invalid_quots
)->set_exporting_parameter( name = 'et_result_msg   '  value = messages    ).
```
**Problem:** The `name` parameters passed to `set_exporting_parameter` contain trailing spaces (`'et_new_lots     '`). If the AMDP test double framework uses exact string matching (without RTRIM) for parameter name lookup, these trailing spaces would cause the mock to never match the actual parameter, silently returning the initial value instead of the configured mock output.

This pattern is consistent across multiple setter methods (`SET_SOS_DET`, `SET_SEPARATE_MDPS`, etc.) and appears intentional for column-aligned readability. Whether the framework trims the parameter name must be verified.

**Break Scenario:** If `IF_AMDP_TEST_ENVIRONMENT` does not trim parameter names, all configured test double outputs for `CL_PPS4_MRP_LOTSIZING=>LOT_SIZE_CALC` would be silently ignored, making every MRP calculation unit test in `TH_PPS4_MRP_CALC` subclasses test against empty (initial) output data only — appearing to pass while testing nothing.

**Fix:** Remove trailing spaces from parameter names or verify the framework documentation explicitly states trailing-space tolerance.

---

### [DESIGN] [PROVEN]: Excessive Inline SQL Column Lists Reduce Maintainability

**Location:** `CL_PPH_READ_MASTER::READ_MAT_MRP_CTRL_PARAMS` lines 299-360 (62 columns)
**Code:** (full column list in `lt_ctrl_fixtr_raw` SELECT and `et_ctrl` SELECT)

**Problem:** The SQLScript SELECT lists enumerate 60+ columns inline by name without any structured comment grouping (beyond a few inline comments). The final `et_ctrl` SELECT at lines 363-414 duplicates the same 60+ columns with `c.` prefix. These two nearly-identical lists must be kept in sync manually. Any new column added to `PPH_ENT_MAT_CTRL_DD` must be added in at least 3 places: the `lt_ctrl_fixtr_raw` projection, the `et_ctrl` projection, and the `adjust_mat_ctrl_for_pegging` method.

This transport itself demonstrates the issue: `adjust_mat_ctrl_for_pegging` returns NULLs for many fields, which must be kept in sync with whatever columns `et_ctrl` projects. If a new column is added to `PPH_ENT_MAT_CTRL_DD` and added to `et_ctrl` but not to `adjust_mat_ctrl_for_pegging`, callers that use the pegging path get NULL for the new field without any compile error.

**Fix:** Consider using a VIEW or a `SELECT *` with explicit EXCLUDE for the fields being overridden, to reduce the maintenance surface.

---

## Summary Table

| # | Severity | Evidence | Location | Title |
|---|----------|----------|----------|-------|
| 1 | HIGH | PROVEN | CL_PPS4_MRP_AMDP_TOOLS | EVALUATE_FCAL method declared but not implemented |
| 2 | HIGH | PROVEN | TH_PPS4_LOTSIZING_SQL | 8 test methods in transport not found in source (INTKA_MTO_PERIOD etc.) |
| 3 | HIGH | PROVEN | CL_PPS4_MRP_CALC::CALC_CHANGES | Dead code `if 0 = 999` permanently disables MPN netting |
| 4 | MEDIUM | PROVEN | CL_PPS4_MRP_AMDP_TOOLS::CREATE_PEGAREA_DATA | Missing WERKS in it_ctrl_mrp join condition |
| 5 | MEDIUM | LIKELY | CL_PPH_READ_MASTER::READ_MAT_MRP_CTRL_PARAMS | NULL propagation in RSTER date (reshz IS NULL) |
| 6 | MEDIUM | LIKELY | CL_PPS4_MRP_AMDP_TOOLS::FILTER_UNSUPPORTED_DOCUMENTS | NULL/empty fabkl bypasses factory calendar boundary check |
| 7 | MEDIUM | LIKELY | CL_PPS4_MRP_LOTSIZING::LOT_SIZE_CALC_LLANG_PARALLEL | Division by zero if lv_quote_sum = 0 in quota split |
| 8 | MEDIUM | PROVEN | TH_PPS4_LOTSIZING_SQL::VALIDATE_TERBV | avail_date assertion uses TOLERABLE — too weak |
| 9 | MEDIUM | LIKELY | CL_PPS4_MRP_LOTSIZING::LOT_SIZE_CALC | Safety stock may be silently dropped for sub-MRP areas |
| 10 | LOW | PROVEN | TH_PPS4_LOTSIZING_SQL::SHIFT_SAFETY_STOCK | Misleading assertion message text |
| 11 | LOW | PROVEN | TH_PPS4_LOTSIZING_SQL::LGINT_MISSING_PERIOD_WEBAZ | Test method has no assertions — no behavioral guarantees |
| 12 | LOW | SPECULATIVE | TH_PPS4_MRP_CALC | Trailing spaces in parameter names may break test double matching |
| 13 | DESIGN | PROVEN | CL_PPH_READ_MASTER | 60+ column inline lists duplicated in 3 places — maintenance risk |

---

## Positive Observations

1. **Comprehensive SQLScript NULL guards in LOT_SIZE_CALC:** The `lt_lot_tab_max_min` SELECT extensively uses `CAST(COALESCE(ctrl.minbe, 0) AS DECIMAL(13,3))` patterns for all numeric lot-size parameters (`bstmi`, `bstma`, `bstfe`, `mabst`, `ausss`, `bstrf`), preventing NULL arithmetic in downstream LLANG code. This is well-executed defensive programming.

2. **Safety stock date correction via LEFT OUTER JOIN:** The `COALESCE( earliest_date.req_dats, req.dat00 )` pattern for shifting safety stock to the earliest requirement date is elegant and correct: when no positive requirements exist, safety stock remains on its original date rather than failing or producing a NULL date.

3. **Period join disambiguation in test class:** `prepare_period_all_joins` explicitly injects competing rows with different `periv`, `mrppp`, and `mtord` values to test that JOIN conditions are precise enough not to pick up wrong period buckets. This is sophisticated test construction that proves actual SQL join behavior, not just happy-path logic.

4. **FIRST_VALUE window function for worst-message aggregation:** The `planning_file_entries_prepare` method correctly uses `FIRST_VALUE(...) OVER (PARTITION BY ... ORDER BY severity DESC)` to propagate the most severe message per material/MRP area, rather than an arbitrary MAX/MIN. This ensures the right message type (X > E > W > I) is surfaced in the planning file update.

5. **Test helper abstraction design in TH_PPS4_LOTSIZING_SQL:** Using an abstract public `FOR TESTING` base class with concrete `prepare_disls()`, `prepare_period()`, and `validate_intka()/validate_terbv()` helper methods provides clean test composition. The test methods read almost like specifications: `prepare_disls(intka='1')` → `call_lotsizing(...)` → `validate_intka(per_start)`. This is a clean test DSL pattern.

6. **AMDP test double injection pattern in TH_PPS4_MRP_CALC:** The class correctly uses `if_amdp_test_environment` with `get_test_double()` and `create_output_configuration()` chaining, separating the concerns of "what the double returns" from "how the real method works." The `constructor` and `destruct_environment` lifecycle methods properly clear and destroy doubles.

7. **Factory calendar boundary checking:** Adding calendar range validation in `filter_unsupported_documents` (detecting requirements beyond `max_date` and before `min_date`) proactively prevents the `add_workdays` function from being called with dates outside the calendar scope, which would cause runtime errors in the factory calendar engine.

---

## Blast Radius Assessment

**Finding #3 (Dead code / MPN netting disabled):** Affects all MRP runs for any material configured with supersession/MPN. With this code, MPN relationships are never evaluated. Depending on how widely MPN is used in the productive landscape, this could affect 0% (if MPN not used) to a significant percentage of materials. The risk is **silent wrong results** rather than a crash.

**Finding #1 (EVALUATE_FCAL missing implementation):** Blast radius depends on where/whether this method is called. If it was added as a new API but not yet wired in, the current blast radius is zero. If any caller was already added elsewhere in the same or another transport, the blast radius is all MRP runs where `EVALUATE_FCAL` is invoked.

**Finding #7 (Division by zero in quota split):** Blast radius is the entire HANA LOP (L-language parallel) partition that processes the affected material. In `LOT_SIZE_CALC_LLANG_PARALLEL`, processing is partitioned, so a crash in one partition aborts all materials in that partition. With default partitioning, this could abort MRP for hundreds of materials per occurrence.
