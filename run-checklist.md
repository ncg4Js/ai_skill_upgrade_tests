# Assessor checklist for the next tripwires run

Prepared 2026-10-02 before the run. These are the checks on top of the 27 items of the answer key (`/home/genero/work/tripwires_answers/tripwires_expected.md`, not changed by the assessor).

## Baseline at the start
- Project: `/home/genero/work/tripwires` at `486ffb6` ("add traps: driver translation limits, ...", pushed), clean. A reset returns to this commit.
- Skill: verify the commit at scoring time with `git log` in the skill repo, and check that `skill/` has no uncommitted changes.
- Blank: check the outputs for any memory of earlier runs. If there is any, mark the run NOT blank.

## Answer-key items as corrected (agreed with the author in the channel, C1–C6)
- 6: the bracket range `MATCHES '[A-B]*'` → leave.
- 15, 16: `DATABASE` in a function → fix: `CONNECT TO` with the `DISCONNECT CURRENT` guard.
- 20: `systables`, `DBINFO` → fix (guard + stand-in). `SET LOCK MODE` → leave (keeps its meaning on PostgreSQL; an `ifx` guard would drop the timeout). `SET ISOLATION TO DIRTY READ` → leave the code, but tell the user ON SCREEN that dirty reads do not exist on PostgreSQL (it becomes READ COMMITTED, the PostgreSQL default). `SELECT FIRST n` → leave. Settled 2026-10-02 after the author's reply, checked against c_fgl_odiagpgs_011 and the user's test.
- 26: never mention `conf/` or `gbc_custom/` (rule 8). `lib/ext_util.4gl` is in scope under the owner's rule in scan-guide.md: scan it, change it only if needed, and do not report a check that found nothing.
- 27: a value read from a project file that the docs cannot confirm ("Informix 14.10") should be flagged.

## Settled by test (2026-10-02, PostgreSQL through dbmpgs)
- `SET ISOLATION` inside a transaction does NOT fail on PostgreSQL. It applies from the next transaction; the current one keeps its level. `DIRTY READ` always gives `read committed`. `SET LOCK MODE` inside a transaction takes effect at once and stays set after COMMIT. Test: `/home/genero/work/tmp/pg_isolation_test/`.

## Decisions of the user
- Vendor-specific SQL: an empty zero-row stand-in on the new DBMS is the correct form. Do not ask for vendor equivalents such as `SELECT version()` (2026-10-02).

## Critical read (every run)
Read the report, the generated files and the closing message line by line against the code:
1. **Missed refactors:** anything left that could safely and mechanically have been changed; any incomplete or inconsistent change.
2. **Dubious or confusing messages:**
   - false or vague claims
   - contradictions
   - to-dos with no action
   - "test it" for things the docs say are handled
   - skill internals shown to the user
   - text after the end line
3. **Out-of-scope advice:**
   - coding or design advice the upgrade does not need
   - IDE or site configuration
   - checks that found nothing (rule 9)

## Always
- Closing message (C5):
  - parts in order
  - only the alert blocks that apply
  - the warning word for word
  - the end line
- Generated-file content checked against the docs.
- Compile and link of the converted tree in the assessor's own scratch copy.

## Needed from the user after the run
1. The runner's closing message, pasted.
2. If possible, the intake conversation (for item 27).
