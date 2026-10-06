---
id: fourjs-ifx-to-pg
name: fourjs-ifx-to-pg
description: Migrate Genero BDL (.4gl) source code from Informix to PostgreSQL. Covers SQL syntax changes, connection switching, SERIAL columns, cursor and locking differences, transaction error handling, type compatibility, and portable coding patterns.
version: "1.6"
tags:
  - migration
  - postgresql
  - informix
  - sql
category: operations
keywords:
  - MATCHES
  - LIKE
  - "FOR UPDATE"
  - "WITH HOLD"
  - "WHERE CURRENT OF"
  - "BEGIN WORK"
  - "COMMIT WORK"
  - SAVEPOINT
  - SERIAL
  - "db_get_last_serial"
  - "fgl_db_driver_type"
  - "OUTER JOIN"
  - SUBSTRING
  - "CONNECT TO"
  - DISCONNECT
  - DATABASE
  - SCHEMA
  - "SET CONNECTION"
  - "ISOLATION LEVEL"
  - "SET LOCK MODE"
  - WHENEVER
  - "SQLSTATE 25P02"
  - dbmpgs
  - ifxemul
  - "FGL_LENGTH_SEMANTICS"
  - "year()"
  - "month()"
  - "day()"
  - EXTRACT
  - NVL
  - COALESCE
  - DECODE
  - TODAY
  - CURRENT_DATE
  - MDY
  - MAKE_DATE
  - "INTO TEMP"
  - "WITH NO LOG"
  - "CREATE TEMP TABLE"
  - "SELECT INTO TEMP"
  - ROWID
  - ctid
sections:
  - id: migration-overview
    title: Migration Overview
  - id: sql-syntax-changes
    title: SQL Syntax Changes
  - id: scalar-functions-and-date-expressions
    title: Informix Scalar Functions and Date Expressions
  - id: temporary-tables
    title: Temporary Tables (SELECT INTO TEMP)
  - id: database-connection-switching
    title: Database Connection Switching
  - id: serial-and-sequences
    title: SERIAL Columns and Sequences
  - id: cursor-and-locking
    title: Cursor Behavior and Locking
  - id: transactions
    title: Transaction Error Handling
  - id: type-compatibility
    title: Variable and Column Type Compatibility
  - id: portable-code-patterns
    title: Portable Code Patterns
  - id: common-agent-mistakes
    title: Common Agent Mistakes
related:
  - fourjs-sql-programming
  - fourjs-error-handling
  - fourjs-database-schema-design
  - fourjs-sql-best-practices
last_verified: "2026-06-01"
genero_version: "5.00"
---

# Informix to PostgreSQL — BDL Code Migration

Patterns and rules for migrating Genero BDL `.4gl` source code from IBM
Informix to PostgreSQL.

**Scope:** This skill covers changes to BDL source code only — SQL statements
embedded in `.4gl` files, connection management, cursor logic, transaction
handling, and variable/column type alignment. It does not cover DDL schema
migration, data transfer, or database administration tasks.

**Schema file:** When a project schema file (`.sql` or `.sch`) is available,
use it to cross-check BDL variable types against column types. PostgreSQL is
significantly stricter than Informix about type mismatches, so verifying
alignment before testing saves time.

---

## Migration Overview

### Scope of BDL Changes

The following categories of changes are required:

1. **SQL syntax** — Informix-specific clauses inside SQL strings
2. **Connection switching** — `DATABASE` / `CONNECT TO` / `DISCONNECT` usage
3. **SERIAL columns** — `INSERT` with value `0` must be replaced
4. **Cursors and locking** — `WITH HOLD`, `FOR UPDATE`, `SET LOCK MODE`
5. **Transactions** — PostgreSQL aborts the entire transaction on any SQL error
6. **Type compatibility** — PostgreSQL rejects implicit type coercion
7. **Portable patterns** — `fgl_db_driver_type()` for code that must run on both

### Quick Detection Commands

```bash
# Find MATCHES operator in SQL strings
grep -rn "MATCHES" src/

# Find SQL substrings using Informix bracket notation
grep -rn "\[[0-9]" src/

# Find Informix OUTER JOIN syntax
grep -rn -i "outer(" src/

# Find INSERT with zero for SERIAL
grep -rn -i "values\s*(0," src/

# Find SET LOCK MODE
grep -rn -i "set lock mode" src/

# Find WITH HOLD cursors
grep -rn -i "with hold" src/

# Find DATABASE instruction at runtime
grep -rn "^\s*DATABASE\s" src/

# Find double-quoted strings in dynamic SQL
grep -rn 'LET.*sql.*".*".*"' src/
```

---

## SQL Syntax Changes

### MATCHES → LIKE

`MATCHES` is Informix-specific and not part of ANSI SQL. The ODI driver
partially converts simple MATCHES patterns but cannot handle all cases,
especially trailing spaces. Each SQL statement using MATCHES must be reviewed
individually.

**Wildcard mapping:**

| Informix MATCHES | SQL LIKE | PostgreSQL ~ (POSIX regex) |
|---|---|---|
| `*` (any sequence) | `%` | `.*` |
| `?` (any single char) | `_` | `.` |
| `[Mm]` (character class) | not possible | `[Mm]` |

```4gl
-- Informix
SELECT * FROM employee WHERE emp_name MATCHES '*cke*'
SELECT * FROM employee WHERE emp_name MATCHES '?ona??'
SELECT * FROM employee WHERE emp_name MATCHES '[Mm]in*'

-- ANSI LIKE (portable)
SELECT * FROM employee WHERE emp_name LIKE '%cke%'
SELECT * FROM employee WHERE emp_name LIKE '_ona__'

-- PostgreSQL ~ operator (when LIKE cannot express the pattern)
SELECT * FROM employee WHERE emp_name ~ 'cke'
SELECT * FROM employee WHERE emp_name ~* '.ona..'    -- case-insensitive
SELECT * FROM employee WHERE emp_name ~ '[Mm]in.*'
```

Note: `~` is case-sensitive; `~*` is case-insensitive.

### SQL Substring `col[x,y]` → `SUBSTRING(col,x,y)`

Informix is the only engine that uses bracket notation for string extraction
inside SQL statements. Replace every occurrence.

```4gl
-- Informix
SELECT * FROM employee WHERE emp_name[1,1] = 'M'
UPDATE employee SET shortcode = emp_name[1,3] WHERE emp_id = ?

-- PostgreSQL / ANSI SQL
SELECT * FROM employee WHERE SUBSTRING(emp_name,1,1) = 'M'
UPDATE employee SET shortcode = SUBSTRING(emp_name,1,3) WHERE emp_id = ?
```

### Outer Join → ANSI JOIN

Informix-specific outer join syntax is not supported by PostgreSQL. All
join queries must be converted to ANSI syntax.

```4gl
-- Informix
SELECT * FROM employee, OUTER(address_list)
  WHERE employee.emp_id = address_list.emp_id

-- ANSI SQL (PostgreSQL)
SELECT * FROM employee
  LEFT OUTER JOIN address_list ON employee.emp_id = address_list.emp_id
```

For multi-table outer joins, each `OUTER(t)` becomes one `LEFT OUTER JOIN`
clause.

### Row Limiting: `SELECT FIRST` / `SKIP`

The ODI driver auto-converts `SKIP n FIRST m` to `OFFSET n ROWS FETCH FIRST m
ROWS ONLY` **only when the parameters are plain integer literals**. When
parameters are variables or SQL parameters the conversion does not happen and
the query fails.

`LIMIT` placed after `ORDER BY` is **never** auto-converted and must always
be rewritten manually.

```4gl
-- Informix (auto-converted by ODI when n and m are integer literals)
SELECT SKIP 10 FIRST 20 emp.* FROM emp ORDER BY emp_name

-- Informix LIMIT after ORDER BY — NOT auto-converted, must change manually
SELECT emp.* FROM emp ORDER BY emp_name LIMIT 10

-- ANSI SQL (works on both, always safe)
SELECT emp.* FROM emp ORDER BY emp_name
  OFFSET 10 ROWS FETCH FIRST 20 ROWS ONLY
```

When the row limit comes from a BDL variable, always write ANSI SQL:

```4gl
DEFINE l_limit INTEGER
LET l_limit = 20

-- WRONG: ODI will not convert when a variable is used
PREPARE st FROM "SELECT FIRST ? * FROM emp"
EXECUTE st USING l_limit

-- CORRECT
PREPARE st FROM "SELECT * FROM emp FETCH FIRST ? ROWS ONLY"
EXECUTE st USING l_limit
```

**Critical: static SQL vs dynamic SQL**

`FETCH FIRST n ROWS ONLY` is valid **only inside dynamic SQL strings** passed
to `PREPARE`. The BDL compiler (`fglcomp`) rejects it in static SQL contexts
(`DECLARE CURSOR FOR`, `SELECT INTO`, etc.) because it parses only Informix
SQL dialect for static statements — error `-6609` is raised.

For a **static** `DECLARE CURSOR FOR` with a literal row limit, keep
`SELECT FIRST n`. The ODI driver auto-converts the literal at runtime.
Switch to a `PREPARE` + dynamic string only when the limit comes from a
variable or when the query must explicitly use ANSI syntax.

```4gl
-- WRONG: fglcomp error -6609 — FETCH FIRST is not valid in static BDL SQL
DECLARE cc CURSOR FOR
  SELECT emp_id, emp_name FROM employee
   ORDER BY emp_name
   FETCH FIRST 200 ROWS ONLY       -- compiler rejects this

-- CORRECT for static DECLARE (literal limit): keep SELECT FIRST n
-- ODI auto-converts the literal 200 to FETCH FIRST 200 ROWS ONLY at runtime
DECLARE cc CURSOR FOR
  SELECT FIRST 200 emp_id, emp_name FROM employee
   ORDER BY emp_name

-- CORRECT for dynamic SQL (variable limit or explicit ANSI): use PREPARE
PREPARE st FROM
  "SELECT emp_id, emp_name FROM employee" ||
  " ORDER BY emp_name FETCH FIRST ? ROWS ONLY"
DECLARE cc CURSOR FOR st
OPEN cc USING l_limit
```

### String Delimiters in Dynamic SQL

The `fglcomp` compiler automatically replaces double-quote string literals
with single quotes in **static** SQL at compile time. In **dynamically built**
SQL strings this conversion does not happen.

```4gl
-- Static SQL — fglcomp fixes this automatically, no change needed
SELECT * FROM employee WHERE emp_name = "Miller"

-- Dynamic SQL — double quotes reach PostgreSQL and cause an error
-- PostgreSQL interprets "Miller" as a column/object name, not a string
LET sql = "SELECT * FROM employee WHERE emp_name = " ||
          '"Miller"'          -- WRONG for PostgreSQL

-- CORRECT: use single quotes in dynamic SQL
LET sql = "SELECT * FROM employee WHERE emp_name = 'Smith'"

-- CORRECT: use a parameterized query (best practice for all databases)
PREPARE st FROM "SELECT * FROM employee WHERE emp_name = ?"
EXECUTE st USING "Miller"
```

As a transitional workaround, set `dbi.database.dbname.ifxemul.dblquotes = true`
in FGLPROFILE to have the ODI auto-replace double quotes with single quotes.
This cannot distinguish between quoted strings and quoted object names, so
manual review is the safe long-term fix.

### ROWID Pseudo-Column → ctid

Informix `ROWID` is a physical row identifier of type `INTEGER`. PostgreSQL's
equivalent is `ctid`, of type `tid` — a `(page, offset)` tuple e.g. `(0,42)`.
The ODI driver does **not** auto-convert `ROWID` references.

#### Simple SELECT

```4gl
-- Informix
SELECT ROWID FROM tabla

-- PostgreSQL
SELECT ctid FROM tabla
```

#### WHERE clause (re-fetching a row by physical address)

```sql
-- Informix
SELECT campo1 FROM tabla WHERE ROWID = v_rowid

-- PostgreSQL
SELECT campo1 FROM tabla WHERE ctid = v_ctid::tid
```

#### BDL cursor pattern (most common use case)

`ROWID` is often fetched inside a cursor loop and used later to re-locate
the row for an UPDATE without `FOR UPDATE`. Migrate as follows:

```4gl
-- Informix (BDL)
DEFINE v_rowid   INTEGER
DEFINE v_campo1  CHAR(20)

DECLARE c CURSOR FOR SELECT ROWID, campo1 FROM tabla
FOREACH c INTO v_rowid, v_campo1
  -- ... process row ...
  UPDATE tabla SET campo1 = v_campo1 WHERE ROWID = v_rowid
END FOREACH

-- PostgreSQL (BDL)
-- ctid has no native BDL type; store as VARCHAR
DEFINE v_ctid    VARCHAR(20)
DEFINE v_campo1  CHAR(20)

DECLARE c CURSOR FOR SELECT ctid::VARCHAR, campo1 FROM tabla
FOREACH c INTO v_ctid, v_campo1
  -- ... process row ...
  UPDATE tabla SET campo1 = v_campo1 WHERE ctid = v_ctid::tid
END FOREACH
```

#### Key differences

| Aspect | Informix `ROWID` | PostgreSQL `ctid` |
|--------|-----------------|-------------------|
| Type | `INTEGER` | `tid` — cast with `::tid` in WHERE |
| BDL variable type | `INTEGER` | `VARCHAR(20)` or `CHAR(20)` |
| Format | simple integer | `(page,offset)` string, e.g. `(0,42)` |
| Stability | stable within normal DML | **changes on UPDATE of the same row**, `VACUUM FULL`, `CLUSTER` |
| Safe across transactions | yes | unreliable — do not store between commits |

#### Recommendation: prefer a primary key

`ctid` is not stable across `UPDATE` operations. If the application stores
`ROWID` between commits or uses it after an UPDATE on the same row, the correct
migration is to add a proper primary key and use it instead:

```sql
ALTER TABLE tabla ADD COLUMN id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY;
```

Use `ctid` only for within-transaction scroll-and-update cursor loops that do
not commit inside the loop.

---

## Informix Scalar Functions and Date Expressions

The Informix scalar functions below are **not auto-converted by the ODI driver**.
Every occurrence inside a SQL string — whether static (`DECLARE CURSOR FOR`,
`SELECT INTO`) or dynamic (`LET sql = ... PREPARE st FROM sql`) — must be
replaced manually.

**Exception:** `TODAY` and `CURRENT` in **static** BDL SQL are converted by the
ODI at runtime. In **dynamic** SQL strings they are not, and must be written
explicitly.

### Function Reference Table

| Informix | PostgreSQL | Notes |
|---|---|---|
| `year(col)` | `EXTRACT(year FROM col)` | Returns `FLOAT8`; cast to `INTEGER` if assigned to an integer variable |
| `month(col)` | `EXTRACT(month FROM col)` | Same cast note |
| `day(col)` | `EXTRACT(day FROM col)` | Same cast note |
| `weekday(col)` | `EXTRACT(dow FROM col)` | 0 = Sunday in both |
| `TODAY` *(static SQL)* | Converted by ODI automatically | No change needed |
| `TODAY` *(dynamic SQL string)* | `CURRENT_DATE` | Must be explicit when built with `LET` |
| `CURRENT` *(static SQL)* | Converted by ODI automatically | No change needed |
| `CURRENT` *(dynamic SQL string)* | `CURRENT_TIMESTAMP` | Must be explicit when built with `LET` |
| `MDY(month, day, year)` | `MAKE_DATE(year, month, day)` | **Argument order differs** — Informix is (m,d,y), PostgreSQL is (y,m,d) |
| `DATE(expr)` | `CAST(expr AS DATE)` | Or use `expr::DATE` shorthand |
| `NVL(a, b)` | `COALESCE(a, b)` | PostgreSQL has no `NVL` |
| `FIRSTNN(a, b)` | `COALESCE(a, b)` | Informix "first not null"; PostgreSQL uses `COALESCE` |
| `DECODE(e, v1,r1, v2,r2, ..., def)` | `CASE WHEN e=v1 THEN r1 WHEN e=v2 THEN r2 ... ELSE def END` | No direct equivalent |
| `EXTEND(dt, qualifier)` | `DATE_TRUNC('unit', dt)` or `CAST(dt AS type)` | Depends on the qualifier — see note below |

### Quick-Detection Grep

```bash
grep -rn -i "year(\|month(\|day(\|weekday(\|nvl(\|decode(\|firstnn(\|MDY(\|extend(\|TODAY\|CURRENT_DATE" src/
```

### Examples

```4gl
-- year() / month() / day()
-- WRONG
LET ptxt = "SELECT year(entry_date) yr, month(entry_date) mon FROM ledger_line WHERE ..."
critstr("year(entry_date)", ">=", "", xdate_from, "")

-- CORRECT
LET ptxt = "SELECT EXTRACT(year FROM entry_date) yr, EXTRACT(month FROM entry_date) mon FROM ledger_line WHERE ..."
critstr("EXTRACT(year FROM entry_date)", ">=", "", xdate_from, "")

-- TODAY in static SQL — ODI auto-converts, no change needed
WHERE val_date = TODAY

-- TODAY in dynamic SQL string — must be written explicitly
-- WRONG
LET sql = "SELECT * FROM journal WHERE val_date = TODAY"

-- CORRECT
LET sql = "SELECT * FROM journal WHERE val_date = CURRENT_DATE"

-- MDY — argument order is DIFFERENT
-- WRONG (Informix order)
LET sql = "INSERT INTO event_log (event_date) VALUES (MDY(6, 15, 2024))"

-- CORRECT (PostgreSQL order: year, month, day)
LET sql = "INSERT INTO event_log (event_date) VALUES (MAKE_DATE(2024, 6, 15))"

-- NVL
-- WRONG
LET ptxt = "SELECT NVL(amount, 0) FROM movement"

-- CORRECT
LET ptxt = "SELECT COALESCE(amount, 0) FROM movement"

-- DECODE
-- WRONG
LET ptxt = "SELECT DECODE(entry_type,'c',amount,-amount) FROM ledger"

-- CORRECT
LET ptxt = "SELECT CASE WHEN entry_type='c' THEN amount ELSE -amount END FROM ledger"
```

### EXTEND Qualifier Mapping

Informix `EXTEND(dt, YEAR TO MONTH)` truncates a DATETIME to year-month precision.

```4gl
-- Informix
EXTEND(dt, YEAR TO MONTH)   -- truncate to year-month
EXTEND(dt, YEAR TO DAY)     -- truncate to date

-- PostgreSQL
DATE_TRUNC('month', dt)     -- truncate to year-month
DATE_TRUNC('day',   dt)     -- truncate to date (same as CAST AS DATE)
```

---

## Temporary Tables (SELECT INTO TEMP)

### The Problem

Informix supports creating a temporary table from a SELECT in a single
statement. PostgreSQL does not have this syntax.

```sql
-- Informix — single statement
SELECT col1, col2 INTO TEMP mytable WITH NO LOG
  FROM source
 WHERE ...
 ORDER BY col1
```

### Solution: CREATE TEMP TABLE … AS SELECT

The direct PostgreSQL replacement is `CREATE TEMP TABLE … AS SELECT …`.
`WITH NO LOG` simply does not exist in PostgreSQL — temp tables are always
session-local and implicitly unlogged. Remove it.

```sql
-- PostgreSQL
CREATE TEMP TABLE mytable AS
  SELECT col1, col2
    FROM source
   WHERE ...
   ORDER BY col1
```

### In Dynamic SQL (PREPARE + EXECUTE)

When the SQL is built as a string, the whole statement must be restructured.

```4gl
-- Informix dynamic SQL
LET ptxt = "SELECT col1, col2 FROM source WHERE ...",
           " ORDER BY col1 INTO TEMP mytable WITH NO LOG"
PREPARE st FROM ptxt
EXECUTE st
FREE st

-- PostgreSQL — restructured as CREATE TEMP TABLE ... AS SELECT
LET ptxt = "CREATE TEMP TABLE mytable AS",
           " SELECT col1, col2 FROM source WHERE ...",
           " ORDER BY col1"
PREPARE st FROM ptxt
EXECUTE st
FREE st
```

Note: `ORDER BY` inside `CREATE TABLE AS SELECT` is syntactically valid in
PostgreSQL but the table has no inherent row order — a subsequent `SELECT`
without `ORDER BY` may return rows in any order.

### DROP TABLE: Add IF EXISTS

Code that drops the temp table before recreating it to ensure a clean state
should use `IF EXISTS` on PostgreSQL. Without it, PostgreSQL raises an error
if the table does not exist (Informix is guarded by `WHENEVER ERROR CONTINUE`).

```4gl
-- Informix pattern (guarded by WHENEVER ERROR CONTINUE)
WHENEVER ERROR CONTINUE
DROP TABLE mytable
WHENEVER ERROR STOP   -- or WHENEVER_ERROR_CALL

-- PostgreSQL-portable — never errors if table absent
DROP TABLE IF EXISTS mytable
```

### CREATE TEMP TABLE WITH NO LOG (DDL Form)

If the code creates a temp table with an explicit column list using `WITH NO LOG`,
just remove the clause.

```sql
-- Informix
CREATE TEMP TABLE mytable (col1 CHAR(10), col2 INTEGER) WITH NO LOG;

-- PostgreSQL
CREATE TEMP TABLE mytable (col1 CHAR(10), col2 INTEGER);
```

### Persistent Unlogged Tables

For non-temporary tables, Informix `WITH NO LOG` maps to PostgreSQL
`CREATE UNLOGGED TABLE`. This is only relevant for persistent tables, not
temp tables.

```sql
-- Informix
CREATE TABLE audit_log (id SERIAL, msg VARCHAR(200)) WITH NO LOG;

-- PostgreSQL
CREATE UNLOGGED TABLE audit_log (id SERIAL, msg VARCHAR(200));
```

---

### External Schemas (Cross-Database Queries)

Informix supports `database:table` inline cross-database notation. PostgreSQL
does not support this syntax. Two migration paths exist:

**Path A — Separate PostgreSQL database (use `CONNECT TO` + `SET CONNECTION`)**

When the referenced database is a separate PostgreSQL instance, apply Pattern 1b:
open a named connection once at startup, wrap each cross-database query with
`SET CONNECTION`, and close the connection at exit.

```4gl
-- Informix: inline cross-database reference
select * into rec.* from otherdb:tablename where ...

-- PostgreSQL migration (Pattern 1b):

-- Step 1 — open the second connection ONCE at program/module init
CONNECT TO "otherdb" AS "conn_otherdb"
SET CONNECTION "maindb"           -- restore active connection

-- Step 2 — wrap each cross-database query with SET CONNECTION.
--          CRITICAL: place SET CONNECTION back AFTER all sqlca checks.
--          SET CONNECTION is a SQL-level instruction that may overwrite sqlca;
--          checking sqlca after the switch tests the connection switch, not the query.
SET CONNECTION "conn_otherdb"
select * into rec.* from tablename where ...   -- no prefix needed
IF sqlca.sqlcode != 0 THEN        -- check sqlca WHILE still on conn_otherdb
  ...
END IF
SET CONNECTION "maindb"           -- restore AFTER the IF block

-- Special case: if the IF block contains a RETURN/EXIT, restore the connection
-- inside the branch BEFORE returning so the caller is not left on the wrong connection:
SET CONNECTION "conn_otherdb"
select * into rec.* from tablename where ...
IF sqlca.sqlcode = 100 THEN
  SET CONNECTION "maindb"         -- restore before early exit
  RETURN
END IF
SET CONNECTION "maindb"           -- normal path

-- NO DISCONNECT between uses — the connection stays open (zero cost to switch back).
-- Disconnecting and reconnecting on every call defeats the purpose of Pattern 1b.

-- Step 3 — DISCONNECT only when the connection is truly no longer needed
--          (program exit or final function that uses it)
DISCONNECT "conn_otherdb"
SET CONNECTION "maindb"           -- make the remaining connection active explicitly
```

**When to DISCONNECT:** Only when the connection will never be used again —
at program end (`DISCONNECT ALL`) or when a module that owns the connection is
being torn down. Never disconnect between individual uses.

**Critical rule after DISCONNECT:** Always call `SET CONNECTION` on the connection
that should remain active immediately after `DISCONNECT`. After a `DISCONNECT` the
runtime may leave the active-connection pointer undefined; making it explicit
avoids "no connection" errors on the next SQL statement.

**`DEFINE … RECORD LIKE otherdb:table.*` (compile-time, not runtime):**
`CONNECT TO` / `SET CONNECTION` are runtime-only and do not help for compile-time
type resolution. Use explicit field types or a `SCHEMA` declaration instead.

**Path B — Foreign Data Wrapper (FDW)**

When the remote database cannot be reached directly (network/firewall) or when
consolidating access through one PostgreSQL server, define a foreign schema via
`CREATE FOREIGN DATA WRAPPER` / `CREATE FOREIGN TABLE`. Once the foreign tables
exist, no BDL code change is needed beyond removing the `dbname:` prefix.

---

## Database Connection Switching

### The Core Difference

In Informix, `DATABASE newdb` seamlessly closes the current connection and
opens a new one. In PostgreSQL, a connection cannot switch databases in-place
— each database is a separate server entity requiring a distinct physical
connection.

### Pattern 1: Sequential Database Switch

`DISCONNECT CURRENT` + `CONNECT TO` tears down and rebuilds the TCP connection
on every switch (authentication, session init, network round-trips). Use this
only when the second database is accessed at most once during the program run or
when the server has very tight connection limits.

```4gl
-- Informix — implicit close and reopen
DATABASE maindb
-- work on maindb
DATABASE archivedb    -- silently closes maindb, opens archivedb
-- work on archivedb
CLOSE DATABASE

-- PostgreSQL-portable minimal form — one connection at a time
CONNECT TO "maindb" USER username USING password
-- work on maindb
DISCONNECT CURRENT
CONNECT TO "archivedb" USER username USING password
-- work on archivedb
DISCONNECT CURRENT
```

### Pattern 1b: Persistent Named Connections + SET CONNECTION (Preferred)

When switching between databases more than once, open both connections once
with named aliases and use `SET CONNECTION` to switch. The runtime changes
only an internal session pointer — no TCP teardown, no re-authentication, no
server-side session allocation on each switch.

```4gl
-- Open both connections once, with named aliases
CONNECT TO "maindb"   AS "conn_main"
CONNECT TO "archivedb" AS "conn_secondary"

-- Switch is near-zero cost
SET CONNECTION "conn_secondary"
-- work on archivedb

SET CONNECTION "conn_main"
-- work on maindb

DISCONNECT ALL   -- closes both at the end
```

The `DATABASE` instruction still works with PostgreSQL in Genero (it closes
and reconnects), but only if each name has a FGLPROFILE entry. The named
`CONNECT TO` + `SET CONNECTION` pattern is the recommended portable form
when switching is frequent.

### Pattern 2: `CONNECT TO DEFAULT` — Replace with Explicit Name

`CONNECT TO DEFAULT`, `SET CONNECTION DEFAULT`, and `DISCONNECT DEFAULT` are
Informix-specific and fail on PostgreSQL.

```4gl
-- Informix-specific — fails on PostgreSQL
CONNECT TO DEFAULT
SET CONNECTION DEFAULT
DISCONNECT DEFAULT

-- PostgreSQL-portable
CONNECT TO "mydb" USER username USING password
SET CONNECTION "conn_mydb"
DISCONNECT "conn_mydb"
```

### Pattern 3: `DATABASE` Before `MAIN` → `SCHEMA`

`DATABASE` used above a `MAIN` block serves two purposes in Informix: compile-
time schema resolution (enabling `DEFINE LIKE table.*`) and an implicit runtime
connection. This implicit connection is undesirable for portability. Use
`SCHEMA` for compile-time schema only and connect explicitly at runtime.

```4gl
-- Informix — creates implicit runtime connection as side-effect
DATABASE maindb

MAIN
  DEFINE rec RECORD LIKE employee.*
  ...
END MAIN

-- PostgreSQL-portable — compile-time schema only
SCHEMA maindb

MAIN
  DEFINE rec RECORD LIKE employee.*
  CONNECT TO "maindb" USER username USING password
  ...
END MAIN
```

### Pattern 4: Multi-Session Simultaneous Access

`CONNECT TO … AS name` + `SET CONNECTION` is already portable. No structural
change is needed — only remove `DEFAULT` keyword usages.

```4gl
CONNECT TO "maindb"   USER u USING p AS "conn_main"
CONNECT TO "archivedb" USER u USING p AS "conn_secondary"

SET CONNECTION "conn_main"
-- work on maindb

SET CONNECTION "conn_secondary"
-- work on archivedb

DISCONNECT ALL
```

### Connection Instruction Reference

| Informix | PostgreSQL-portable |
|---|---|
| `DATABASE maindb` (runtime, one-shot) | `DISCONNECT CURRENT` + `CONNECT TO "maindb"` |
| `DATABASE maindb` (runtime, repeated) | `CONNECT TO "maindb" AS "conn_main"` once; then `SET CONNECTION "conn_main"` |
| `CLOSE DATABASE` | `DISCONNECT CURRENT` |
| `CONNECT TO DEFAULT` | `CONNECT TO "maindb" USER u USING p` |
| `SET CONNECTION DEFAULT` | `SET CONNECTION "conn_main"` |
| `DISCONNECT DEFAULT` | `DISCONNECT CURRENT` or `DISCONNECT ALL` |
| `DATABASE maindb` (before MAIN) | `SCHEMA maindb` + explicit `CONNECT TO` at runtime |

**Performance note:** `SET CONNECTION` changes only an in-process pointer —
no network traffic, no authentication, no server-side session allocation.
`DISCONNECT` + `CONNECT TO` pays the full TCP + auth cost on every switch.
Prefer `SET CONNECTION` whenever a connection will be used more than once.

---

## SERIAL Columns and Sequences

### The Problem: INSERT with Value 0

In Informix, inserting `0` into a SERIAL column triggers the auto-increment
mechanism and the actual inserted value is the next sequence number. In
PostgreSQL, `0` is inserted **literally** — no error is raised, the sequence
is not advanced, and the row receives `0` as its key. This is a silent data
integrity risk.

```4gl
-- Informix: 0 triggers auto-increment, actual value from sequence
INSERT INTO employee
  (emp_id, emp_name, location)
  VALUES (0, "Acme Corp", "Chicago")

-- PostgreSQL: 0 is stored literally — WRONG
INSERT INTO employee
  (emp_id, emp_name, location)
  VALUES (0, "Acme Corp", "Chicago")
```

### Solution 1: ANSI Mode — Omit the SERIAL Column

The safest and most portable fix is to exclude the SERIAL column from both
the column list and the values list. PostgreSQL (and Informix) will then use
the sequence automatically.

```4gl
-- ANSI SQL — portable across all databases
INSERT INTO employee
  (emp_name, location)
  VALUES ("Acme Corp", "Chicago")
```

Use `fglcomp -S` to inspect the SQL that the compiler generates from static
`INSERT INTO table VALUES rec.*` to verify the SERIAL column is excluded.

### Solution 2: INSERT with Record (Syntax 3)

When inserting a full record with `INSERT INTO table VALUES rec.*`, the
compiler automatically omits the SERIAL column in the generated SQL. This
is the recommended pattern.

```4gl
DEFINE rec RECORD LIKE employee.*
LET rec.emp_name = "Acme Corp"
LET rec.location     = "Chicago"
-- rec.emp_id is not set (or set to 0 from INITIALIZE) — compiler omits it
INSERT INTO employee VALUES rec.*
```

### Solution 3: PostgreSQL DEFAULT

When the column must appear in the INSERT, use `DEFAULT` as the value.

```4gl
INSERT INTO employee
  (emp_id, emp_name, location)
  VALUES (DEFAULT, "Acme Corp", "Chicago")
```

### Reading the Last Generated Serial

Do not rely solely on `sqlca.sqlerrd[2]` emulation for reading the last
inserted serial value — it may not work correctly for all configurations.
Use `db_get_last_serial()` from the `fgldbutl` module, which is portable
across drivers.

**Signature** (from `fgldbutl`):
```
FUNCTION db_get_last_serial(emultype STRING, tabname STRING) RETURNS BIGINT
```

- `emultype` — serial emulation type. Use `"native"` for PostgreSQL (and modern
  Informix with native sequences). Other values: `"native2"`, `"regtable"`,
  `"trigseq"` (for older emulation setups).
- `tabname` — name of the table just inserted into (lowercase, case-sensitive).

Must add `IMPORT FGL fgldbutl` at the top of the `.4gl` module.

```4gl
IMPORT FGL fgldbutl
...
DEFINE new_id INTEGER

INSERT INTO employee (emp_name, location) VALUES (l_emp_name, l_city)

LET new_id = db_get_last_serial("native", "employee")
```

---

## Cursor Behavior and Locking

### FOR UPDATE — Lock Release Timing

Informix holds a `FOR UPDATE` lock until the cursor is closed, independently
of transaction boundaries. PostgreSQL releases the lock when the transaction
ends OR the cursor is closed — whichever happens first.

Review code that relies on a `FOR UPDATE` lock surviving a `COMMIT WORK` or
`ROLLBACK WORK` while the cursor remains open.

```4gl
-- Risk pattern: lock expected to survive COMMIT
DECLARE cc CURSOR FOR
  SELECT * FROM invoice WHERE status = 'P' FOR UPDATE
OPEN cc
FETCH cc INTO rec.*          -- lock acquired on Informix

COMMIT WORK                  -- Informix: lock kept; PostgreSQL: lock released

UPDATE invoice SET status = 'X'
  WHERE CURRENT OF cc        -- may fail on PostgreSQL if lock was released
```

Refactor to use the primary key for updates, keeping the update inside the
same transaction as the fetch.

### SET LOCK MODE — Bypass Informix-Specific SQL

`SET LOCK MODE TO WAIT n / NOT WAIT` is Informix-specific. The ODI converts
the BDL-level wait setting, but raw SQL strings executing `SET LOCK MODE` must
be guarded so they do not run against PostgreSQL.

By default, PostgreSQL waits indefinitely for a locked record. Error -244
(locked record) is returned on the `FETCH` or `FOREACH`, not on `OPEN`.
Isolate those statements in a transaction or SAVEPOINT to allow recovery.

```4gl
-- Guard Informix-only lock mode SQL
IF fgl_db_driver_type() = "ifx" THEN
  EXECUTE IMMEDIATE "SET LOCK MODE TO NOT WAIT"
END IF
```

### WHERE CURRENT OF — Use Primary Key Instead

`WHERE CURRENT OF` is supported by PostgreSQL but is fragile across
transaction boundaries. Refactor to use the primary key of the fetched row.

```4gl
-- Informix pattern (supported but not recommended)
UPDATE invoice SET status = 'X' WHERE CURRENT OF cc

-- Portable replacement using primary key
UPDATE invoice SET status = 'X' WHERE inv_id = rec.inv_id
```

### WITH HOLD — Behavior on ROLLBACK

In Informix, a `WITH HOLD` cursor stays open regardless of how the
transaction ends. In PostgreSQL, a `WITH HOLD` cursor stays open after
`COMMIT WORK` but is **closed** on `ROLLBACK WORK`. Any code that reads
a `WITH HOLD` cursor after a rollback must add explicit re-open logic.

```4gl
DECLARE cc CURSOR WITH HOLD FOR SELECT * FROM invoice

OPEN cc
BEGIN WORK
FETCH cc INTO rec.*
-- ... processing fails, must ROLLBACK
ROLLBACK WORK

-- Informix: cursor still open, FETCH continues
-- PostgreSQL: cursor is now CLOSED, next FETCH raises an error

-- Portable fix: re-open after rollback when using WITH HOLD
ROLLBACK WORK
OPEN cc   -- safe to re-open; already-fetched position is lost
```

### WITH HOLD + FOR UPDATE — Not Supported in PostgreSQL

Declaring a cursor as both `WITH HOLD` and `FOR UPDATE` is not supported in
PostgreSQL. Refactor by separating the read pass (WITH HOLD) from locking
updates (a separate transaction using the primary key).

```4gl
-- Informix only — not supported on PostgreSQL
DECLARE cc CURSOR WITH HOLD FOR
  SELECT * FROM invoice FOR UPDATE

-- Portable refactor: two-step approach
DECLARE cc CURSOR WITH HOLD FOR
  SELECT * FROM invoice            -- read cursor, no FOR UPDATE

FOREACH cc INTO rec.*
  BEGIN WORK
  -- Lock the specific row by PK for update
  SELECT * FROM invoice WHERE inv_id = rec.inv_id FOR UPDATE
  -- ... make changes
  COMMIT WORK
END FOREACH
```

### ISOLATION LEVEL

Standard isolation levels (`DIRTY READ`, `COMMITTED READ`, `REPEATABLE READ`,
`SERIALIZABLE`) are auto-converted by the ODI driver. No code change is
required for the `SET ISOLATION TO` statement itself. Use `READ COMMITTED`
(equivalent to `COMMITTED READ`) as the recommended default for portability.

### Scrollable Cursors

PostgreSQL supports `SCROLL` cursors natively. The PGS driver automatically
declares server-side scrollable cursors when `DECLARE c1 SCROLL CURSOR` is
used. No code change is required.

---

## Transaction Error Handling

### The Core Difference

PostgreSQL **aborts the entire transaction** when any SQL error occurs inside a
`BEGIN WORK` / `COMMIT WORK` block. After the error, no further SQL executes
until the transaction is rolled back. Informix only rolls back what the
programmer explicitly reverts, allowing execution to continue after an error
within a transaction.

```4gl
-- This Informix pattern is BROKEN on PostgreSQL
WHENEVER ERROR CONTINUE
BEGIN WORK
INSERT INTO tab1 VALUES (1, 'abc')
INSERT INTO tab1 VALUES (1, 'abc')  -- duplicate key error
IF sqlca.sqlcode < 0 THEN
   INSERT INTO tab1 VALUES (2, 'abc')  -- Informix: ok; PostgreSQL: SQLSTATE 25P02
END IF
COMMIT WORK
```

On PostgreSQL, the third INSERT fails with SQLSTATE `25P02`:
`"current transaction is aborted, commands ignored until end of transaction block"`.

### Solution 1: TRY/CATCH (Recommended)

Wrap the transaction in `TRY`/`CATCH`. Any SQL error causes the `CATCH` block
to execute `ROLLBACK WORK`. This is the cleanest pattern and works on all
databases.

```4gl
BEGIN WORK
TRY
  INSERT INTO invoice (emp_id, amount) VALUES (?, ?)
    USING l_emp_id, l_amount
  INSERT INTO inv_line (inv_id, item_code, qty) VALUES (?, ?, ?)
    USING l_inv_id, l_item_code, l_qty
  COMMIT WORK
CATCH
  ROLLBACK WORK
  DISPLAY "Transaction failed: ", sqlca.sqlcode
END TRY
```

### Solution 2: SAVEPOINT for Partial Protection

When the transaction must survive an expected error on a specific statement,
protect that statement with a `SAVEPOINT`. On error, roll back to the
savepoint and continue; on success, release the savepoint.

```4gl
WHENEVER ERROR CONTINUE
BEGIN WORK

SAVEPOINT sp1
INSERT INTO tab1 VALUES (1, 'abc')   -- may fail on duplicate key
IF sqlca.sqlcode < 0 THEN
  ROLLBACK WORK TO SAVEPOINT sp1    -- undo only this statement
  INSERT INTO tab1 VALUES (2, 'abc') -- try alternative
END IF
RELEASE SAVEPOINT sp1

COMMIT WORK
```

Important: Do not use the `UNIQUE` keyword when declaring savepoints. Always
specify the savepoint name in `ROLLBACK TO SAVEPOINT`. Do not use
`RELEASE SAVEPOINT` after a rollback.

### Solution 3: sql_protect() / sql_unprotect() Helper Pattern

A pair of helper functions that create and release a named savepoint only
when connected to PostgreSQL, making the pattern transparent to callers.

```4gl
FUNCTION sql_protect()
  IF fgl_db_driver_type() != "pgs" THEN
    RETURN
  END IF
  SAVEPOINT _sql_protect_
END FUNCTION

FUNCTION sql_unprotect()
  IF fgl_db_driver_type() != "pgs" THEN
    RETURN
  END IF
  IF sqlca.sqlcode < 0 THEN
    ROLLBACK WORK TO SAVEPOINT _sql_protect_
  ELSE
    RELEASE SAVEPOINT _sql_protect_
  END IF
END FUNCTION

-- Usage
WHENEVER ERROR CONTINUE
BEGIN WORK
CALL sql_protect()
INSERT INTO tab1 VALUES (1, 'abc')
CALL sql_unprotect()
IF sqlca.sqlcode < 0 THEN
  INSERT INTO tab1 VALUES (2, 'abc')
END IF
COMMIT WORK
```

### Audit Checklist for Transactions

When reviewing a module's transaction blocks:

1. Find every `BEGIN WORK` / `COMMIT WORK` pair
2. Check if `WHENEVER ERROR CONTINUE` is active inside the block
3. Check if `sqlca.sqlcode` is tested after individual statements within the TX
4. If execution continues after an error inside the TX → apply SAVEPOINT or
   refactor to TRY/CATCH
5. If no error is expected inside the TX → wrap in TRY/CATCH for clean rollback

---

## Variable and Column Type Compatibility

### PostgreSQL Is Strict About Type Mismatches

PostgreSQL raises SQLSTATE `42883` when a WHERE clause or assignment compares
or assigns values of incompatible types. Informix performs implicit coercion
silently.

```4gl
DEFINE l_emp_id INTEGER

-- This works on Informix (implicit coercion) but fails on PostgreSQL
-- if employee.emp_id is CHAR(5)
SELECT emp_name INTO l_emp_name
  FROM employee
  WHERE emp_id = l_emp_id    -- SQLSTATE 42883: operator does not exist: char = integer
```

Fix: ensure the BDL variable type matches the column type declared in the schema.

```4gl
DEFINE l_emp_id CHAR(5)     -- matches the column type

SELECT emp_name INTO l_emp_name
  FROM employee
  WHERE emp_id = l_emp_id   -- correct type alignment
```

### Using the Schema File to Verify Types

When a schema file (`.sql` or `.sch`) is available, compare the `DEFINE`
variable type against the column type for every variable used as a SQL
predicate or in an INTO clause. Common mismatches to check:

| BDL variable type | Column type | Result on PostgreSQL |
|---|---|---|
| `INTEGER` | `CHAR(n)` | SQLSTATE 42883 |
| `STRING` | `DATE` | SQLSTATE 42883 |
| `DECIMAL` | `INTEGER` | SQLSTATE 42883 |
| `CHAR(n)` | `INTEGER` | SQLSTATE 42883 |
| `INTEGER` / `SMALLINT` | `DATE` | SQLSTATE 42883 — Informix stores DATE as an integer internally and coerces silently; PostgreSQL has no implicit integer→date cast |
| `CHAR(n)` with non-ISO format | `DATE` | Runtime cast error — Informix accepts DBDATE-formatted strings (e.g. `"01/15/2024"`); PostgreSQL requires ISO format `'2024-01-15'` |

### DATE Column Compared with INTEGER or Non-ISO String

Informix stores `DATE` values internally as integers (days since 31-Dec-1899)
and performs implicit coercion in comparisons. PostgreSQL has no such coercion
— every comparison must be type-compatible.

**INTEGER / SMALLINT compared with a DATE column:**

```4gl
-- WRONG: works on Informix (implicit coercion), fails on PostgreSQL
DEFINE l_days  INTEGER
LET l_days = 45291          -- some day offset
SELECT * INTO rec.* FROM invoice WHERE inv_date = l_days
-- PostgreSQL: SQLSTATE 42883 "operator does not exist: date = integer"

-- CORRECT: use a DATE variable
DEFINE l_date  DATE
LET l_date = l_days         -- BDL converts integer to DATE
SELECT * INTO rec.* FROM invoice WHERE inv_date = l_date
```

**CHAR / STRING with non-ISO date format compared with a DATE column:**

Informix accepts DBDATE-formatted strings (e.g. `"01/15/2024"` for MM/DD/YYYY)
directly against DATE columns. PostgreSQL requires ISO 8601 format
(`'2024-01-15'`) or an explicit `CAST`.

```4gl
-- WRONG: DBDATE-format string — may silently fail or raise error on PostgreSQL
SELECT * INTO rec.* FROM invoice WHERE inv_date = "01/15/2024"

-- CORRECT: use a DATE variable (BDL handles the conversion from DBDATE internally)
DEFINE l_date DATE
LET l_date = "01/15/2024"   -- BDL parses using DBDATE setting
SELECT * INTO rec.* FROM invoice WHERE inv_date = l_date

-- CORRECT for dynamic SQL: use ISO format string or a bind parameter
PREPARE st FROM "SELECT * FROM invoice WHERE inv_date = ?"
EXECUTE st USING l_date
```

### DECIMAL Must Always Have Precision and Scale

Due to implementation differences between Informix and PostgreSQL, `DECIMAL`
without `(p,s)` behaves inconsistently. Always specify both values.

```4gl
-- Avoid
DEFINE amount DECIMAL
LET amount = 123.45

-- Always specify precision and scale
DEFINE amount DECIMAL(16,2)
```

### NCHAR / NVARCHAR in BDL DEFINE

If the Informix schema used `NCHAR` or `NVARCHAR` columns, BDL variables
defined with `DEFINE rec RECORD LIKE table.*` inherit those types. Since
PostgreSQL has no `NCHAR`/`NVARCHAR`, the schema must be updated to `VARCHAR`
before extracting a `.sch` file for PostgreSQL compilation.

### FGL_LENGTH_SEMANTICS=CHAR for UTF-8 Environments

Informix uses byte-length semantics: `CHAR(20)` means 20 bytes. PostgreSQL
uses character-length semantics. When migrating to a UTF-8 database, set the
environment variable `FGL_LENGTH_SEMANTICS=CHAR` so that `CHAR(n)` and
`VARCHAR(n)` sizes in BDL are interpreted as character counts, matching
PostgreSQL's semantics.

```bash
export FGL_LENGTH_SEMANTICS=CHAR
```

---

## Portable Code Patterns

### fgl_db_driver_type() Branching

Use `fgl_db_driver_type()` to select database-specific SQL at runtime.
Driver codes: `"ifx"` = IBM Informix, `"pgs"` = PostgreSQL.

```4gl
DEFINE sql STRING

CASE fgl_db_driver_type()
  WHEN "ifx"
    LET sql = "SELECT FIRST 10 * FROM invoice"
  WHEN "pgs"
    LET sql = "SELECT * FROM invoice FETCH FIRST 10 ROWS ONLY"
END CASE

PREPARE st FROM sql
```

### Bypassing Informix-Only SQL Statements

Wrap any Informix-specific session commands so they are skipped on PostgreSQL.

```4gl
-- SET LOCK MODE is Informix-specific
IF fgl_db_driver_type() = "ifx" THEN
  EXECUTE IMMEDIATE "SET LOCK MODE TO NOT WAIT"
END IF

-- UPDATE STATISTICS is Informix-specific
IF fgl_db_driver_type() = "ifx" THEN
  EXECUTE IMMEDIATE "UPDATE STATISTICS FOR TABLE invoice"
END IF

-- Informix SPL stored procedure calls
IF fgl_db_driver_type() = "ifx" THEN
  EXECUTE PROCEDURE ifx_specific_proc(l_param)
END IF
```

### Error Code Mapping for PostgreSQL

When connected to PostgreSQL, native errors that cannot be mapped to an
Informix error code are reported as `sqlca.sqlcode = -6372`. The native
PostgreSQL error code is always available in `sqlca.sqlerrd[2]`.

Centralise SQL error classification to handle both drivers:

```4gl
CONSTANT SQLERRTYPE_LOCK  = 1
CONSTANT SQLERRTYPE_PKEY  = 2
CONSTANT SQLERRTYPE_FATAL = 3

FUNCTION classify_sql_error() RETURNS INTEGER
  -- Locked record: Informix -244, PostgreSQL native 55P03 / lock_timeout
  IF sqlca.sqlcode = -244
  OR sqlca.sqlerrd[2] = 55 THEN   -- PostgreSQL lock_not_available class
    RETURN SQLERRTYPE_LOCK
  END IF
  -- Duplicate primary key: Informix -239/-268, PostgreSQL 23505
  IF sqlca.sqlcode = -239
  OR sqlca.sqlcode = -268
  OR sqlca.sqlerrd[2] = 23505 THEN
    RETURN SQLERRTYPE_PKEY
  END IF
  RETURN SQLERRTYPE_FATAL
END FUNCTION
```

### sql_protect() / sql_unprotect() — Reusable SAVEPOINT Helpers

See **Transaction Error Handling** section for the implementation. These
helpers allow existing code with `WHENEVER ERROR CONTINUE` patterns to run
correctly on PostgreSQL with minimal structural change.

---

## Common Agent Mistakes

### INSERT with 0 into a SERIAL Column

```4gl
-- WRONG: inserts literal 0 in PostgreSQL; sequence not advanced
INSERT INTO employee (emp_id, emp_name) VALUES (0, "Miller")

-- CORRECT: omit SERIAL column (ANSI mode)
INSERT INTO employee (emp_name) VALUES ("Miller")
```

### Calling db_get_last_serial() with Wrong Arity

`db_get_last_serial` requires **two** arguments. Calling it with zero arguments
compiles but fails at link time with error -6200 (parameter count mismatch).
`IMPORT FGL fgldbutl` is also required.

```4gl
-- WRONG: zero arguments — linker error -6200
LET new_id = db_get_last_serial()

-- CORRECT
IMPORT FGL fgldbutl
...
INSERT INTO employee (emp_name, location) VALUES (l_emp_name, l_location)
LET new_id = db_get_last_serial("native", "employee")
```

Use `"native"` as the emulation type for PostgreSQL with SERIAL/SEQUENCE columns.
The second argument is the table name (lowercase, case-sensitive).

### Double Quotes in Dynamically Built SQL

```4gl
-- WRONG: PostgreSQL interprets "Miller" as an object name, not a string
LET sql = 'SELECT * FROM employee WHERE emp_name = "Miller"'

-- CORRECT: use single quotes
LET sql = "SELECT * FROM employee WHERE emp_name = 'Smith'"

-- BEST: parameterized query
PREPARE st FROM "SELECT * FROM employee WHERE emp_name = ?"
EXECUTE st USING "Miller"
```

### FETCH FIRST in Static BDL SQL (Compiler Error -6609)

`FETCH FIRST n ROWS ONLY` is ANSI SQL but the BDL compiler only accepts
Informix SQL syntax for static statements. Writing it inside a
`DECLARE CURSOR FOR` block causes a compile-time error (-6609 "grammatical
error … expecting IDENT").

```4gl
-- WRONG: fglcomp -6609 — not valid in static BDL SQL
DECLARE cc CURSOR FOR
  SELECT emp_id, emp_name FROM employee
   ORDER BY emp_name
   FETCH FIRST 200 ROWS ONLY

-- CORRECT for a static cursor with a literal limit:
-- keep SELECT FIRST n — ODI auto-converts the literal at runtime
DECLARE cc CURSOR FOR
  SELECT FIRST 200 emp_id, emp_name FROM employee
   ORDER BY emp_name

-- CORRECT for a dynamic-SQL cursor (variable limit, or explicit ANSI):
PREPARE st FROM
  "SELECT emp_id, emp_name FROM employee" ||
  " ORDER BY emp_name FETCH FIRST ? ROWS ONLY"
DECLARE cc CURSOR FOR st
OPEN cc USING l_limit
```

Rule of thumb: `FETCH FIRST … ROWS ONLY` belongs only in strings passed to
`PREPARE`. In static `DECLARE CURSOR FOR` blocks, use `SELECT FIRST n`.

### LIMIT After ORDER BY Not Auto-Converted by ODI

```4gl
-- WRONG: ODI never converts LIMIT after ORDER BY
PREPARE st FROM "SELECT * FROM invoice ORDER BY inv_date LIMIT 10"

-- CORRECT: use FETCH FIRST
PREPARE st FROM "SELECT * FROM invoice ORDER BY inv_date FETCH FIRST 10 ROWS ONLY"
```

### Row-Limiting With Variable Parameters Not Auto-Converted

```4gl
-- WRONG: ODI only converts literal integers, not variables
PREPARE st FROM "SELECT FIRST ? * FROM invoice"
EXECUTE st USING l_limit

-- CORRECT
PREPARE st FROM "SELECT * FROM invoice FETCH FIRST ? ROWS ONLY"
EXECUTE st USING l_limit
```

### WITH HOLD + FOR UPDATE Combination

```4gl
-- WRONG: not supported in PostgreSQL, raises runtime error
DECLARE cc CURSOR WITH HOLD FOR SELECT * FROM invoice FOR UPDATE

-- CORRECT: separate read cursor from PK-based locking update
DECLARE cc CURSOR WITH HOLD FOR SELECT * FROM invoice
```

### WHENEVER ERROR CONTINUE Inside a Transaction

```4gl
-- WRONG: works on Informix but breaks on PostgreSQL
--        after first error, all subsequent SQL raises SQLSTATE 25P02
WHENEVER ERROR CONTINUE
BEGIN WORK
INSERT INTO tab1 VALUES (1, 'abc')
INSERT INTO tab1 VALUES (1, 'abc')  -- error
IF sqlca.sqlcode < 0 THEN
  INSERT INTO tab1 VALUES (2, 'abc')  -- PostgreSQL: TX already aborted
END IF
COMMIT WORK

-- CORRECT: use TRY/CATCH or SAVEPOINT (see Transactions section)
```

### Checking sqlca.sqlcode Alone for PostgreSQL-Specific Errors

```4gl
-- INCOMPLETE: unmapped PostgreSQL errors appear as -6372
IF sqlca.sqlcode = -244 THEN   -- may miss PostgreSQL lock errors
  ...
END IF

-- CORRECT: also check the native code in sqlerrd[2]
IF sqlca.sqlcode = -244
OR (sqlca.sqlcode = -6372 AND sqlca.sqlerrd[2] = ...) THEN
  ...
END IF
```

### CONNECT TO DEFAULT / SET CONNECTION DEFAULT

```4gl
-- WRONG: Informix-specific, fails on PostgreSQL
CONNECT TO DEFAULT
SET CONNECTION DEFAULT
DISCONNECT DEFAULT

-- CORRECT: always use explicit names
CONNECT TO "mydb" USER username USING password
SET CONNECTION "conn_mydb"
DISCONNECT "conn_mydb"
```

### DATABASE Before MAIN Creates an Implicit Connection

```4gl
-- PROBLEMATIC: creates implicit runtime connection as side-effect
DATABASE maindb    -- intended for schema resolution only

MAIN
  DEFINE rec RECORD LIKE employee.*
  ...
END MAIN

-- CORRECT: SCHEMA for compile-time only; explicit CONNECT at runtime
SCHEMA maindb

MAIN
  DEFINE rec RECORD LIKE employee.*
  CONNECT TO "maindb" USER username USING password
  ...
END MAIN
```

### Informix Date/Time Functions in Dynamic SQL

`year()`, `month()`, `day()`, `NVL()`, `DECODE()`, `MDY()` are Informix scalar
functions that the ODI driver does **not** auto-convert. Using them in a dynamic
SQL string (built with `LET` and passed to `PREPARE`) causes a PostgreSQL runtime
error.

```4gl
-- WRONG: year()/month() not available in PostgreSQL
LET ptxt = "SELECT year(entry_date), month(entry_date) FROM ledger_line WHERE ..."
PREPARE st FROM ptxt

-- CORRECT
LET ptxt = "SELECT EXTRACT(year FROM entry_date), EXTRACT(month FROM entry_date) FROM ledger_line WHERE ..."
PREPARE st FROM ptxt

-- WRONG: NVL not available in PostgreSQL
LET ptxt = "SELECT NVL(amount, 0) FROM movement"

-- CORRECT
LET ptxt = "SELECT COALESCE(amount, 0) FROM movement"

-- WRONG: TODAY in a dynamic SQL string is not converted by ODI
LET ptxt = "SELECT * FROM journal WHERE val_date = TODAY"

-- CORRECT
LET ptxt = "SELECT * FROM journal WHERE val_date = CURRENT_DATE"

-- NOTE: MDY argument order differs!
-- Informix: MDY(month, day, year)  →  PostgreSQL: MAKE_DATE(year, month, day)
```

`TODAY` and `CURRENT` in **static** BDL SQL (`DECLARE CURSOR FOR`, `SELECT INTO`)
are auto-converted by the ODI — no change needed there.

### SELECT INTO TEMP … WITH NO LOG Not Supported

PostgreSQL has no `SELECT … INTO TEMP tablename WITH NO LOG` statement. The
entire SQL must be restructured as `CREATE TEMP TABLE tablename AS SELECT …`.

```4gl
-- WRONG: syntax error on PostgreSQL
LET ptxt = "SELECT col1, col2 FROM source",
           " ORDER BY col1 INTO TEMP mytable WITH NO LOG"
PREPARE st FROM ptxt
EXECUTE st

-- CORRECT: CREATE TEMP TABLE ... AS SELECT
LET ptxt = "CREATE TEMP TABLE mytable AS",
           " SELECT col1, col2 FROM source",
           " ORDER BY col1"
PREPARE st FROM ptxt
EXECUTE st

-- Also update the preceding DROP TABLE:
-- WRONG (raises error on PostgreSQL if table absent)
DROP TABLE mytable

-- CORRECT
DROP TABLE IF EXISTS mytable
```

### Type Mismatch Between BDL Variable and Column

```4gl
-- WRONG: if employee.emp_id is CHAR(5), this fails on PostgreSQL with
--        SQLSTATE 42883 "operator does not exist: character varying = integer"
DEFINE l_emp_id INTEGER
SELECT emp_name INTO l_emp_name FROM employee WHERE emp_id = l_emp_id

-- CORRECT: variable type must match the column type
DEFINE l_emp_id CHAR(5)
SELECT emp_name INTO l_emp_name FROM employee WHERE emp_id = l_emp_id
```

### Using ROWID Without Converting to ctid

`ROWID` is not auto-converted by the ODI driver. Every occurrence in SQL
strings — static or dynamic — must be changed to `ctid`.

```4gl
-- WRONG: ROWID not recognised by PostgreSQL
DEFINE v_rowid  INTEGER
DECLARE c CURSOR FOR SELECT ROWID, campo1 FROM tabla
FOREACH c INTO v_rowid, v_campo1
  UPDATE tabla SET campo1 = v_campo1 WHERE ROWID = v_rowid
END FOREACH

-- CORRECT
DEFINE v_ctid   VARCHAR(20)   -- tid has no native BDL type
DECLARE c CURSOR FOR SELECT ctid::VARCHAR, campo1 FROM tabla
FOREACH c INTO v_ctid, v_campo1
  UPDATE tabla SET campo1 = v_campo1 WHERE ctid = v_ctid::tid
END FOREACH
```

If the application stores `ROWID` values across transaction boundaries or
after UPDATEs on the same row, add a proper primary key to the table and
use that instead — `ctid` is not stable in those scenarios.
