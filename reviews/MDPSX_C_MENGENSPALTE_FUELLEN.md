# Deep Code Review: `MDPSX_C_MENGENSPALTE_FUELLEN`

**Object:** Function Module `MDPSX_C_MENGENSPALTE_FUELLEN`
**Function Group:** M61X (package MD03)
**Purpose:** Fill conversion factors for individual lines in display unit of measure (Mengenspalte = quantity column)
**Classification:** User-facing (MRP display list) | Read-only conversion logic
**Review Date:** 2026-05-13

---

## Phase 1: Context Mapping

This FM is called from the MRP stock/requirements list (MD04/MD05) display layer. It:
1. Saves global state → overrides it with parameters → performs unit conversion → restores global state
2. Converts stock quantities from base UOM to a display UOM
3. Loops through MRP elements to determine batch-specific conversion factors
4. Is a display-only function (flag `aufruf = typ-anzeige`)

**If this throws:** MRP list display crashes for the user — short dump or incorrect quantities shown.

---

## Phase 2: Pattern Recognition

Key patterns observed:
- **Save/Restore pattern** for global variables (lines 37-42, 328-332) — standard technique for function modules that share global state
- **Unit conversion via `MATERIAL_UNIT_CONVERSION`** — standard pattern with proper exception handling
- **`READ TABLE ... WITH KEY`** without BINARY SEARCH — acceptable for small internal tables in display context
- **MODIFY ... INDEX** pattern in LOOP — correct for modifying table during iteration

---

## Phase 3: Findings

### [MEDIUM] [LIKELY]: Division by Zero Risk in Unit Conversion Calculations

**Location:** Lines 76-84
**Code:**
```abap
mt61d-umlmc = mt61d-umlmc * mdsta-umren / mdsta-umrez.
mdkp-eisbe  = mdkp-eisbe  * mdsta-umren / mdsta-umrez.
mdkp-minbe  = mdkp-minbe  * mdsta-umren / mdsta-umrez.
mdkp-hoebe  = mdkp-hoebe  * mdsta-umren / mdsta-umrez.
mdkp-bstmi  = mdkp-bstmi  * mdsta-umren / mdsta-umrez.
mdkp-bstma  = mdkp-bstma  * mdsta-umren / mdsta-umrez.
mdkp-bstfx  = mdkp-bstfx  * mdsta-umren / mdsta-umrez.
mdkp-bstrf  = mdkp-bstrf  * mdsta-umren / mdsta-umrez.
```
**Problem:** If `MATERIAL_UNIT_CONVERSION` succeeds (sy-subrc = 0) but returns `mdsta-umrez = 0` — which should not happen for valid conversions but *can* happen with corrupt master data — all 8 divisions will cause a **COMPUTE_INT_ZERODIVIDE** runtime error.

**Break Scenario:** Material with broken UOM conversion entry (e.g., missing denominator in T006/MARM). The FM at line 64-73 checks sy-subrc but does not validate the actual returned values.

**Fix:**
```abap
IF sy-subrc <> 0.
  MESSAGE ID sy-msgid TYPE sy-msgty NUMBER sy-msgno
          WITH sy-msgv1 sy-msgv2 sy-msgv3 sy-msgv4.
ENDIF.
IF mdsta-umrez IS INITIAL.
  mdsta-umrez = 1.  "Fallback to 1:1 conversion
  mdsta-umren = 1.
ENDIF.
```

**Mitigation:** `MATERIAL_UNIT_CONVERSION` internally validates this, so in practice `umrez = 0` with `sy-subrc = 0` is extremely unlikely. Severity kept at MEDIUM.

---

### [MEDIUM] [LIKELY]: Hard Message Statement Can Cause Unexpected Short Dump

**Location:** Lines 72-73
**Code:**
```abap
IF sy-subrc <> 0.
  MESSAGE ID sy-msgid TYPE sy-msgty NUMBER sy-msgno
          WITH sy-msgv1 sy-msgv2 sy-msgv3 sy-msgv4.
ENDIF.
```
**Problem:** The MESSAGE statement uses `sy-msgty` directly. If the original error returned message type `'A'` (abort) or `'X'` (exit), this will trigger a short dump in the calling program. Since this FM is called in a display context (`aufruf = typ-anzeige`), an abort message here is overly aggressive — the user just wants to view an MRP list.

**Break Scenario:** A material with a deleted UOM triggers exception `meinh_not_found` → system message with type 'A' → short dump in MD04.

**Fix:** Cap message type to 'E' for display context:
```abap
IF sy-subrc <> 0.
  DATA(lv_msgty) = sy-msgty.
  IF lv_msgty CA 'AX'.
    lv_msgty = 'E'.
  ENDIF.
  MESSAGE ID sy-msgid TYPE lv_msgty NUMBER sy-msgno
          WITH sy-msgv1 sy-msgv2 sy-msgv3 sy-msgv4.
ENDIF.
```

---

### [LOW] [SPECULATIVE]: Global State Restoration Missing for `t399d`, `cm61x`, `cm61b`, `plsc`

**Location:** Lines 37-42 (save) vs. 328-332 (restore)
**Code saved:**
```abap
af61x_sav = af61x.
mdsta_sav = mdsta.
mdkp_sav  = mdkp.
mt61d_sav = mt61d.
```
**Code overwritten but NOT saved/restored:**
```abap
MOVE i_t399d TO t399d.    " line 49
MOVE i_cm61x TO cm61x.    " line 50
MOVE i_cm61b TO cm61b.    " line 51
MOVE i_plsc  TO plsc.     " line 52
```
**Code restored:**
```abap
af61x = af61x_sav.
mdsta = mdsta_sav.
mdkp  = mdkp_sav.
mt61d = mt61d_sav.
```

**Problem:** Global variables `t399d`, `cm61x`, `cm61b`, and `plsc` are overwritten with import parameters but never restored. If a caller relies on these globals retaining their values after this FM call, there could be side effects.

**Mitigation:** This is a long-standing pattern (comment "TL 46c" dates it to release 4.6C). The FM signature passes these by value, and the caller likely expects the global to reflect the import values. The original developer's comment *"Wiederherstellen alter globaler Variablen (wozu auch immer)"* ("Restoring old global variables (whatever for)") suggests even they were unsure why some are restored and not others. This is likely intentional but not well-documented.

---

### [LOW] [LIKELY]: `READ TABLE ... WITH KEY` Without SORTED/HASHED or BINARY SEARCH

**Location:** Lines 114, 127, 135, 143, 171-173
**Code (example):**
```abap
READ TABLE mdpsx_sav WITH KEY delkz = wkbst.
```
**Problem:** Linear search on internal table. If `mdpsx_sav` has many entries, this is O(n) per read.

**Mitigation:** In the MRP display context, `mdpsx` typically contains at most a few hundred entries (one per MRP element visible on screen). Performance impact is negligible. This is a style issue, not a bug.

---

### [DESIGN] [LIKELY]: CWM (Catch Weight Management) Activation/Deactivation Span Is Too Wide

**Location:** Lines 86-106 (activate) and lines 153-161 (deactivate)
**Code:**
```abap
" Activate (line 99)
CALL FUNCTION '/CWM/MAME_PQ_BASE_MANIPULATE'
  EXPORTING i_activate = 'X' ...

" ... ~60 lines of stock selection logic ...

" Deactivate (line 156)
CALL FUNCTION '/CWM/MAME_PQ_BASE_MANIPULATE'
  EXPORTING i_deactivate = 'X' ...
```
**Problem:** If any exception occurs between activation and deactivation (e.g., in `select_mard_batch`), the CWM PQ base manipulation remains active, potentially affecting subsequent operations in the same LUW. However, EXCEPTIONS OTHERS = 0 on both calls means they cannot fail themselves.

**Mitigation:** The PERFORMs called between activate/deactivate are read-only stock selection routines. If they dump, the session ends anyway, making the "leaked activate" moot. This is an architectural observation, not a bug.

---

## Summary Table

| # | Severity | Evidence | Finding |
|---|----------|----------|---------|
| 1 | MEDIUM | LIKELY | Division by zero risk if `umrez = 0` returned from unit conversion |
| 2 | MEDIUM | LIKELY | Unguarded `MESSAGE sy-msgty` can trigger abort/short dump in display context |
| 3 | LOW | SPECULATIVE | Incomplete save/restore of global variables (`t399d`, `cm61x`, `cm61b`, `plsc`) |
| 4 | LOW | LIKELY | Linear table reads without BINARY SEARCH (negligible performance impact) |
| 5 | DESIGN | LIKELY | CWM activate/deactivate span covers exception-prone code |

---

## Positive Observations

- **Robust batch conversion handling** (lines 226-273): The code correctly handles the fallback chain: batch-specific conversion → document conversion → material-level conversion. Note 676151 and 900722 are properly implemented.
- **BAdI integration is well-guarded**: The `lv_changed_*` flags from `MURC_GET_MRP_DOCUMENT_DATA` are checked before overwriting BAdI-provided values (lines 276-281).
- **Proper exception handling on MATERIAL_UNIT_CONVERSION** for batch context (lines 224-240): fallback to material-level factors on any error.
- **Clean save/restore pattern** prevents side effects on `af61x`, `mdsta`, `mdkp`, `mt61d` — the most critical globals.
- **Code is well-commented** with SAP note references (676151, 900722, 371477, 3093334) making maintenance history traceable.
