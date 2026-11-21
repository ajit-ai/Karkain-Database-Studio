# Karkain Database Studio

A CLI/TUI database administration and SQL development companion for the embedded Karkain `db2` database, built exclusively on Karkain Language v1.2.0.

## Status

| Area | Status |
|---|---|
| Research | COMPLETE |
| Product design | COMPLETE |
| Repository creation | COMPLETE |
| Implementation | NOT STARTED |
| Compile/execute verification of the application | NOT STARTED (toolchain itself VERIFIED on Karkain v1.2.0) |

## What it does

Karkain Database Studio is a scriptable administration tool for the local embedded `db2` engine. It lets a user create and inspect schema objects, run SQL statements from a script or editor-like buffer, manage explicit transactions (begin/commit/rollback), use prepared statements, and back up or restore a database file. It is the DBA-oriented counterpart to Karkain Data Studio: where Data Studio explores and transforms data, Database Studio manages database structure, transactions and durability. It ships as one native binary for CI and offline administration.

## Who it serves

- DBAs and backend developers managing an embedded local database.
- CI pipelines that need deterministic schema setup and transaction checks.
- Offline/admin automation where no server-grade GUI console is available.
- Learners who want to see SQL and transaction semantics in a small, transparent tool.

## Planned features

All items below are **(planned)**; none is implemented.

- **SQL execution**
  - (planned) Run `db2` SQL subset statements: CREATE, INSERT, SELECT, UPDATE, DELETE, DROP.
  - (planned) Script-file execution with statement-by-statement status output.
  - (planned) Prepared statement execution via `db_prepare`/`db_execute_stmt`.
- **Transaction management**
  - (planned) Explicit `db_begin`/`db_commit`/`db_rollback` control from the command surface.
  - (planned) Transaction status display and automated rollback on script failure.
- **Schema and object management**
  - (planned) Create, list and describe tables and columns.
  - (planned) Schema inspection: table/column metadata and row counts.
- **Backup and restore**
  - (planned) Consistent backup of a `db2` database file.
  - (planned) Restore from a backup into a target path with verification.
- **Auditing and operations**
  - (planned) Append-only audit log of executed statements and outcomes.
  - (planned) Config-file-driven connection/database selection and diagnostics.

## Explicit non-goals

- Not a data exploration/visualization workspace; that is Karkain Data Studio.
- Not a BI/reporting tool; that is Karkain Analytics Studio.
- No external SQL drivers, no joins, no PostgreSQL/SQL Server parity.
- No GUI admin console or web/multi-user mode.
- No TLS, WebSocket or UDP networking.
- No server/role/user administration beyond the local embedded scope.

## Karkain foundation

Exclusive foundation: Karkain Language v1.2.0. All application source will be Karkain. No other implementation language. No compiler modifications.

## Platform support status

- Windows x86-64 toolchain: VERIFIED.
- Linux cross-target output: PRESENT but execution NOT VERIFIED.
- macOS: NOT VERIFIED.

## Build and run status

No application binary exists yet. The intended generic commands are `karkain build src/main.kark -o karkain-database-studio` and `karkain run src/main.kark`. These become active when implementation starts; no compile or run of this application has been performed.

## Known blockers

- No native GUI toolkit in Karkain v1.2.0 (BLOCKED); CLI/TUI/service-first by design.
- `db2` is embedded with an SQL subset; no external SQL drivers and no joins.
- Crash-atomic file writes are PARTIAL (transactions exist, OS atomic-write unverified) — backup/restore must be validated carefully.
- `system`/`env`/`process`/`time` are PARTIAL shell wrappers; `get_args()` returns an empty array, so config-file-driven dispatch is required.
- No TLS/WebSocket/UDP; no concurrent HTTP server framework.

## Repository

GitHub: https://github.com/ajit-ai/Karkain-Database-Studio
Issues: https://github.com/ajit-ai/Karkain-Database-Studio/issues
License: Apache-2.0.
