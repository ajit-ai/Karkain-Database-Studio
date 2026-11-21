# Karkain Database Studio — First Implementation Backlog

## First vertical slice

**Slice goal.** Prove the smallest real administration workflow end to end: from a config-selected database, execute a SQL script that creates a table and inserts rows inside an explicit transaction, commit it, then run a SELECT and print a deterministic result. This exercises dispatch, SQL execution, transaction control and diagnostics — no empty scaffolding.

**Karkain modules/files expected.**

- `src/main.kark` — config-driven command dispatch (DBS-CLI).
- `src/sql/splitter.kark` — SQL statement splitter (DBS-SQL).
- `src/exec/executor.kark` — `db2` CREATE/INSERT/SELECT execution via `db` (DBS-EXEC).
- `src/txn/control.kark` — `db_begin`/`db_commit`/`db_rollback` wrapper (DBS-TXN).
- `src/diag/log.kark` — run record emission (DBS-DIAG).
- `configs/default.karkcfg` — database path, script path, transaction policy, verbosity.

**Exact input.** A script `examples/setup.sql`:

```
CREATE TABLE item (id INT, name TEXT);
INSERT INTO item VALUES (1, 'alpha');
INSERT INTO item VALUES (2, 'beta');
SELECT id, name FROM item;
```

and config selecting `script=examples/setup.sql`, `policy=explicit`, `database=examples/demo.db2`.

**Exact expected observable output** (stdout, deterministic):

```
stmt=1 CREATE status=ok
stmt=2 INSERT status=ok rows=1
stmt=3 INSERT status=ok rows=1
BEGIN ok
COMMIT ok
stmt=4 SELECT status=ok rows=2
1|alpha
2|beta
status=ok
```

plus one DBS-DIAG run record with command=`script`, statements=4, status=`ok`.

**Verification command pattern (Karkain v1.2.0).** From the repository root, with the verified toolchain configuration applied:

```
karkain lint src/main.kark
karkain run src/main.kark --target native-x86_64-windows
karkain build src/main.kark -o karkain-database-studio --target native-x86_64-windows
.\karkain-database-studio
```

The built executable's stdout must match the golden output above byte-for-byte.

## Backlog items

| ID | Feature | Files/modules | Dependencies | Acceptance criteria | Tests | Complexity | Blocking Karkain capability | Definition of done |
|---|---|---|---|---|---|---|---|---|
| DBS-001 | Config-driven CLI dispatch | `src/main.kark`, `configs/default.karkcfg` | none | Reads config, selects `script`, ignores empty `get_args()` | Unit, E2E | M | `get_args()` empty; `system()` shell wrapper | Command runs from config |
| DBS-002 | SQL statement splitter | `src/sql/splitter.kark` | DBS-001 | Splits on semicolons, preserves quoted text | Unit | M | No SQL parser stdlib (hand-written) | Fixture script splits into 4 statements |
| DBS-003 | Core statement execution | `src/exec/executor.kark` | DBS-002 | CREATE/INSERT/SELECT round-trip in `db2` | Integration | M | `db2` subset, no joins/external drivers | Rows retrievable via SELECT |
| DBS-004 | Explicit transaction control | `src/txn/control.kark` | DBS-003 | BEGIN/COMMIT/ROLLBACK state machine correct | Integration, failure | M | transactions PARTIAL-atomic | Commit persists, rollback reverts |
| DBS-005 | Per-statement status output | `src/exec/executor.kark` | DBS-003 | Status and affected rows printed in order | E2E | S | none | Golden stdout matches |
| DBS-006 | Script failure policy | `src/txn/control.kark` | DBS-004 | Failing statement triggers rollback + abort | Failure/recovery | M | none | Prior state restored on failure |
| DBS-007 | Prepared statements | `src/exec/prepared.kark` | DBS-003 | `db_prepare`/`db_execute_stmt` reuse works | Unit, integration | M | prepared statements PRESENT/unverified | Repeated execute returns correct rows |
| DBS-008 | Schema inspection | `src/schema/inspect.kark` | DBS-003 | Lists tables, columns, row counts | Unit, integration | M | no metadata stdlib beyond `db2` | Inspection output matches fixture |
| DBS-009 | Append-only audit log | `src/audit/log.kark` | DBS-005 | Every statement + outcome logged | Observability, E2E | M | none | Audit record complete per run |
| DBS-010 | Backup with checksum | `src/backup/backup.kark` | DBS-004 | Copies db2 file + `sha256` checksum | Integration | L | atomic write PARTIAL | Backup verifies against source |
| DBS-011 | Verified restore | `src/backup/restore.kark` | DBS-010 | Refuses mismatch; reopen-and-inspect passes | Failure/recovery | L | atomic write PARTIAL | Restored DB readable and correct |
| DBS-012 | Delimited bulk load | `src/exec/load.kark` | DBS-003, DBS-004 | Loads CSV rows in one transaction | Integration | M | no CSV stdlib (hand-written) | Loaded rows match source |
| DBS-013 | Actual executable E2E | all | DBS-001..DBS-012 | Built `.exe` reproduces all goldens | Actual-compiled-executable | M | Go-engine/C-free target required | Binary E2E pass |

## Dependency order and gates

1. DBS-001 → DBS-002 → DBS-003 → DBS-004 → DBS-005 (Gate 1: vertical slice runs with commit).
2. DBS-006, DBS-007, DBS-008 (Gate 2: failure policy, prepared statements, inspection).
3. DBS-009 (Gate 3: full audit).
4. DBS-010 → DBS-011 (Gate 4: backup/restore verified).
5. DBS-012, DBS-013 (Gate 5: bulk load, binary verification and packaging).

## Status

All items: **NOT STARTED**. Implementation must not begin until all designs and repository initialization are complete. No tests have been run for this application.
