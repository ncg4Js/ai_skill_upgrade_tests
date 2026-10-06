# Proposed answer-key items 28–45 (new baseline, 2026-10-05)

**Superseded 2026-10-06:** the items are now in the answer key, corrected (35, 39, 40, 41, 45) and with 46–47 added. Use the key, not this file.

Written by the assessor for the user. The user adds these to `tripwires_expected.md` (the assessor never edits it).
They cover the 8 areas in the author's note of 2026-10-05. Every expected answer is checked in the GI manual; the page is in brackets.
"Leave" = the driver handles it, so the code must not be changed (the skill may still mention it if the remark is correct).

New files: `src/tw_sql3.4gl`, `src/tw_types.4gl`, `src/tw_txn.4gl` (all in prog_main) and `src/tw_dflt.4gl` (new program prog_dflt).
Changed: `Makefile`, `README.md` (five programs), `src/tw_main.4gl` (6 new CALLs after line 18; key items 1–27 cite functions, not tw_main lines, so nothing shifts).

## Translated by the driver, inside or outside its limits (area 1)
28. `col[x,y]`: tw_sql3 list_customer_codes
    - line 11, in WHERE → leave (translated) [c_fgl_odiagpgs_024]
    - line 15, in ORDER BY → fix: SUBSTRING/SUBSTR [_024: "rewrite ORDER BY"]
    - line 22, as UPDATE target → fix (rebuild the value with SUBSTR and ||) [_024: "rewrite UPDATE"]
29. Row limiting: tw_sql3
    - line 25, `SKIP 2 FIRST 3` literals → leave [_046]
    - line 33, `FIRST ?` → fix: not converted with a parameter [_046]
30. Double quotes: tw_sql3
    - line 42, string literal in a dynamic string → leave (`ifxemul.dblquotes` defaults to true); recommending single quotes is fine [_026, c_fgl_Connections_015]
    - line 49, double-quoted column names → fix: remove the quotes (with dblquotes they become string literals and the query returns the text 'name'/'city') [_026]
31. CURRENT: tw_sql3
    - line 98, `CURRENT YEAR TO SECOND` → leave [_005, c_fgl_sql_programming_081]
    - line 100, `CURRENT` without qualifiers → fix: a DATETIME variable as parameter [_005: "will not be converted"]
32. Date arithmetic: tw_sql3
    - line 106, static `odate > TODAY - 30` → leave (DATE minus INTEGER works in PostgreSQL) [_005]
    - line 110, `odate < ? + 30` → fix: do the arithmetic in BDL, pass the date [c_fgl_sql_programming_081 "Date-time expressions with parameters"]
33. Integers used as dates: tw_sql3
    - line 116, `odate > 46000` → fix [c_fgl_sql_programming_081 "DATEs as a number of days"]
    - line 120–121, INTEGER variable bound to a DATE column → fix: DATE variable [same]
34. `INTO TEMP`: already item 8 (no new trap).

## Informix SQL functions (area 2)
35. Static SQL, tw_sql3 line 76: `NVL`, `YEAR`, `MONTH` → fix (COALESCE, EXTRACT) [no PostgreSQL page lists them; c_fgl_odiagora_005 lists them as Informix-only date functions]
36. In strings, tw_sql3 lines 82–83 (`DECODE`, `WEEKDAY`, `MDY`) and 95 (`UNITS DAY`) → fix (CASE, EXTRACT(dow), MAKE_DATE with y,m,d order, `+ INTERVAL '30 days'` or `+ 30`)
37. Decoys: must NOT be changed. Settled by test 2026-10-05 (`/home/genero/work/tests/func_test/pgs`, out.txt, PostgreSQL through dbmpgs):
    - line 56, `SUBSTR(name,1,3)` → leave: sent unchanged, works (out.txt:44) [c_fgl_sql_programming_089, _024]
    - line 90, `DATE(logtime) = TODAY` → leave: `DATE()` sent unchanged, `TODAY` translated to `current_date` even in a PREPARE string (out.txt:140), count correct

## Error handled inside a transaction (area 3)
38. tw_txn save_region lines 7–16: WHENEVER ERROR CONTINUE + INSERT, then on error UPDATE in the same transaction → fix: SAVEPOINT around the INSERT (or sql_protect/sql_unprotect), or TRY/CATCH if the logic allows [c_fgl_odiagpgs_013]

## WITH HOLD cursors (area 4)
39. tw_txn line 24–25: `WITH HOLD` + `FOR UPDATE` → fix: SQL error on PostgreSQL; drop WITH HOLD, open inside the transaction, update by key [c_fgl_odiagpgs_033, c_fgl_sql_programming_078]
40. tw_txn line 33–36: WITH HOLD cursor opened after BEGIN WORK, with a ROLLBACK at line 40 → fix: open it outside the transaction (PostgreSQL closes it on rollback) [_033]

## Variable type ≠ column type (area 5)
41. tw_types check_customer_keys:
    - line 10, INTEGER vs `region_code CHAR(2)` → fix
    - line 14, STRING vs `custno INTEGER` → fix
    - line 18, CHAR(10) "01/15/2026" vs `odate DATE` → fix: DATE variable [c_fgl_odiagpgs_028, c_fgl_sql_programming_081]

## Informix-only statements (area 6)
42. tw_txn line 54: `UPDATE STATISTICS` → guard with `fgl_db_driver_type() = "ifx"` (or remove; PostgreSQL uses ANALYZE) [c_fgl_connections_UPDATE_STATISTICS]
43. tw_txn line 57: `EXECUTE PROCEDURE` → must be listed: the SPL procedure must be rewritten for PostgreSQL and called with `CALL` (procedure) or `SELECT` (function). The procedure is not in `sql/`, so the skill cannot rewrite it [c_fgl_sql_programming_031/032, c_fgl_odiagifx_011]

## DEFAULT connection (areas 7 and 8)
44. tw_dflt line 4: `CONNECT TO DEFAULT` → fix or list: Informix-only (connects to INFORMIXSERVER without a database) [c_fgl_Connections_042]
45. tw_dflt lines 9–10: `SET CONNECTION DEFAULT`, `DISCONNECT DEFAULT` → fix with the named session, or remove with line 4 [c_fgl_Connections_043, _044]. prog_dflt also counts as a program for item 18 (mixed modes): it uses only CONNECT TO, so it is not mixed.

## Notes for the user
- Not covered by a trap: `INTO TEMP` outside limits, `FIRST` in a subquery (I could not confirm Informix accepts it), `EXTEND`.
- Item 37 settled by test (both leave). Item 43 needs no test: the procedure does not exist in the project.
- The test refutes a claim in the GI reference skill "fourjs-ifx-to-pg" v1.6: `TODAY` in a dynamic SQL string IS translated by the driver.
