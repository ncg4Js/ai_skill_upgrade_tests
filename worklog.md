# Worklog — Genero Upgrade Skill, tester (assessor)

## Project
- Role: ASSESSOR instance. Scores runs of the genero-upgrade skill; never runs or edits the skill.
- Test project: /home/genero/work/tripwires (trap cases, built by another Claude instance; Informix 14.10, Genero 3.21 style)
- Answer key: /home/genero/work/tripwires_answers/tripwires_expected.md (outside the project, never shown to the runner)
- Scorecard: /home/genero/work/tmp/ai_skill_score.md (recreated after each evaluation)
- Author channel (only one): /home/genero/work/tmp/ai-writer-cooms.md
- Times are local: `TZ=America/Los_Angeles`.
- New entries are added when the user says "take a note", "register this" or similar.

## Log

### 2026-10-02
- 2026-10-02 10:57 Worklog created. State so far today:
  - reviewed the tripwires project
  - fixed the Makefile (fglform output now goes to `bin/`)
  - added `.vscode/settings.json` (its `FGLDBDRIVER` line is bogus, still to remove)
  - scored run 01 (author in-session run): 26/26 scorable items correct, 0 misses, item 27 unscorable; the answer key is out of date on items 15, 16 and 20
  - scorecard rewritten in the author's R5 format
  - an independent blank run (tripwires-02) is in progress, waiting for the user's go to score it
- 2026-10-02 11:00 Scored the independent run (report 17:47 UTC, skill `70ba50b` + uncommitted edits): 26/26 scorable, item 27 unscorable. Scorecard moved to /home/genero/work/tmp at the user's request, and rewritten without references to other runs (every scorecard is independent from now on).
- 2026-10-02 11:30 The author's counter-review (C1–C6) arrived in `ai-writer-cooms.md`. All six accepted after checking. Score revised to 25/27: item 26 partial (mention of `lib/`), item 27 miss (rule gap). The generated files are now assessed for content, not only for leaks.
- 2026-10-02 12:00 The user set the rules: `ai-writer-cooms.md` is the only channel with the author, in both directions; "reset the test" = `git clean -fd` + `git restore .` in tripwires. Stopped reading `ai_writer_requests.md`.
- 2026-10-02 12:20 The user committed the README target line: the baseline is now `7082e22`. An unscored run (not blank) was discarded at the user's request, and my copy of its output was deleted without being read.
- 2026-10-02 12:40 Scored the run with report 19:27 UTC (skill `b815f0b`, possibly `3220ad1`): 26/27, item 26 partial. Summary posted in the channel.
- 2026-10-02 12:55 Scored the run with report 19:47 UTC (skill `19a1579`, D87: every source file in scope): 27/27, but NOT blank (the runner's notes after the end line refer to "the last run"). Item 26 passed: `lib/` is now in scope by the owner's rule. Summary posted in the channel.
- 2026-10-02 13:30 The user found that I missed many issues in that final report: I checked mechanics (lines, counts, compile, leaks), not quality and scope. New standing rule: a critical line-by-line read of the report, the generated files and the screen output against the code, looking for missed refactors, dubious or confusing messages, and out-of-scope advice. Saved to memory and to `run-checklist.md`.
- 2026-10-02 13:43 Handover written: `handson_20261002.md` in this folder.
- 2026-10-02 14:20 Scored the run with report 20:40 UTC (skill `4e4999e`), with the critical read: answer key 26/27 (item 20 partial); code otherwise correct; 8 dubious messages, 4 out of scope. Item 20 corrected after the user's Partner Portal input, checked in GI: `SET ISOLATION TO DIRTY READ` must be handled (it becomes READ COMMITTED on PostgreSQL), and `SET LOCK MODE` should be guarded for portability. My earlier "leave" was wrong. New memory rule: verify every rule in GI before scoring.
- 2026-10-02 14:20 Tested the Portal's claim that `SET ISOLATION` fails inside a transaction on PostgreSQL (`/home/genero/work/tmp/pg_isolation_test/iso_test.4gl`, `out.txt`). Refuted: no error; it applies from the next transaction. DIRTY READ always gives `read committed`. SET LOCK MODE inside a transaction applies at once. Recorded in the scorecard and the checklist.
- 2026-10-02 14:43 Scored the run with report 21:30 UTC (skill `8928799`): answer key 26/27, item 20 partial. The author agreed with D1–D6 and O1–O2. M1 settled: no code change for SET LOCK MODE / SET ISOLATION on PostgreSQL (a guard would drop the timeout), but the screen must tell the user about DIRTY READ. M2 closed by the user: empty stand-ins are correct.
- 2026-10-02 14:43 Handover for tomorrow written: `handson_20261003.md`. The project is not reset (it holds the scored 21:30 run).

### 2026-10-05
- 2026-10-05 13:35 Moved `fourjs-ifx-to-postgresql-migration.md` (a GI skill, "fourjs-ifx-to-pg" v1.6) from `tripwires/docs/` (untracked) to this folder, so the runner cannot see it; removed the empty `docs/`. Handover for a new chat written: `handson_20261005.md`.
- 2026-10-05 14:18 New baseline `486ffb6` pushed: traps for the 8 areas of the author's note (tw_sql3, tw_types, tw_txn, new program prog_dflt), each expected answer checked in GI; proposed key items 28–45 in `proposed_key_items_20261005.md`. PostgreSQL test `tests/func_test/pgs`: SUBSTR and DATE()/TODAY work unchanged (TODAY translated even in a PREPARE string). Local tripwires fast-forwarded and set to track origin/main. Handover for tomorrow: `handson_20261006.md`.

### 2026-10-06
- 2026-10-06 13:31 With the user's go, added items 28–47 to the answer key (it stopped at 27). Changes from the proposal, each checked in GI or by test: 35 (EXTRACT does not compile in static SQL, -6609, fglcomp 6.00.02 → dynamic SQL or list), 31 (CURRENT with qualifiers also correct, _005), 39 (list: the cursor spans many commits, removing WITH HOLD breaks it, _033), 40/41 (fix or list), 45 (must stay consistent with line 4). New: 46 (SERIAL setval in the reference script, _008), 47 (== in BDL, = in SQL). run-checklist baseline set to 486ffb6. Password redaction in func_test dropped (user). The scorecard is now written after every test for the writer to read.
