# Karkain Database Studio — Product Specification

**Document status:** Design COMPLETE. Research = COMPLETE. Repository creation = COMPLETE. Application implementation = NOT STARTED. Application compile/execute verification = NOT STARTED (toolchain path itself VERIFIED on Karkain v1.2.0).


## A. Product definition

**Product name:** Karkain Database Studio ("Database Studio").

**Purpose and problem.** Administering an embedded local database requires repeatable, scriptable control over schema, SQL execution, transactions and durability. GUI consoles are heavyweight, vendor-specific and poorly suited to CI or offline admin automation. Karkain's `db2` engine supports a transactional SQL subset, but there is no focused administrative tool around it.

**Vision.** A small, single-binary, headless administration companion for the embedded `db2` database: create schema, run SQL, control transactions and back up/restore — all deterministic and auditable.

**Intended users / segments.** DBAs and backend developers working with an embedded local database; CI pipelines needing schema setup and transaction verification; offline/admin automation operators; learners studying SQL and transaction semantics.

**Usage scenarios.** (1) Create a schema and load seed rows from a SQL script. (2) Run a statement batch and get per-statement status. (3) Begin a transaction, apply updates and roll back on error. (4) Back up a database file and restore it with verification.

**Market positioning.** A scriptable, headless admin tool for the embedded Karkain database — a complement to GUI administration consoles, not a replacement for server-grade tools.

**Primary value proposition.** Zero-runtime, deterministic administration of `db2` with explicit transaction control, prepared statements, audit logging and backup/restore in one native binary.

**Explicit non-goals / boundaries.** Not a data exploration or visualization workspace (Karkain Data Studio); not BI/reporting (Karkain Analytics Studio); no external SQL drivers, joins, PostgreSQL/SQL Server parity; no GUI/web/multi-user mode; no TLS/WebSocket/UDP. These follow Karkain v1.2.0 capability limits.

## B. Functional architecture

**Module inventory.**

| Module | Responsibility |
|---|---|
| DBS-CLI | Config-driven command dispatch and admin command surface. |
| DBS-SQL | SQL statement splitter, script runner and per-statement status. |
| DBS-EXEC | `db2` statement execution: CREATE/INSERT/SELECT/UPDATE/DELETE/DROP via `db`. |
| DBS-TXN | Explicit transaction control (`db_begin`/`db_commit`/`db_rollback`) and failure policy. |
| DBS-PREP | Prepared statement lifecycle (`db_prepare`/`db_execute_stmt`). |
| DBS-SCHEMA | Schema inspection: tables, columns, row counts, metadata. |
| DBS-BACKUP | Database file backup and verified restore. |
| DBS-AUDIT | Append-only audit log of statements and outcomes. |
| DBS-DIAG | Structured diagnostics and exit codes. |

**Core workflows.** Select database (config) → inspect schema → run SQL script → review per-statement results → commit or roll back → audit record. Backup workflow: quiesce → copy database file → checksum → log. Restore workflow: verify backup → write target → reopen and inspect.

**Admin workflows.** Configure target database path, backup directory, default transaction policy (auto-commit vs explicit) and failure action (abort/rollback). List and inspect databases managed by the tool.

**Error/recovery workflows.** A failing statement aborts the batch; DBS-TXN rolls back the open transaction; the failed statement and error are written to DBS-AUDIT. Restore verifies the backup before overwriting a target and refuses on mismatch.

**Roles & permissions.** Single-user, process-local. File-system permissions are the only access boundary; no authentication/authorization model.

**Configuration/customization.** Local config file declares database path, backup path, delimiter/quote for any data import, transaction policy and output verbosity. No machine-specific paths in repository docs.

**Import/export.** Import: SQL script files; optional delimited-text bulk load into `db2`. Export: SQL result sets to delimited text; backup files for full database state.

**Integration requirements.** Invoked by CI as a compiled binary; reads/writes local database files via `db` and `io`; no network integration in MVP.

## C. Competitor analysis

Researched competitors are **pgAdmin**, **SQL Server Management Studio (SSMS)** and **DBeaver**, consistent with `docs/competitor-analysis/COMPETITORS.md` and `domain2-data.md`. Labels: **[VERIFIED]** official-page fact (accessed 2026-11-21), **[INTERPRETATION]** synthesis, **[PROPOSAL]** Karkain recommendation, **UNKNOWN — not verified**.

**[VERIFIED]** pgAdmin is the official PostgreSQL administration and development platform with query tool, graphical EXPLAIN, grant wizard, backup/restore and job scheduling; SSMS is Microsoft's Windows-only utility for SQL Server with Object Explorer, script editors and query plans; DBeaver is a universal explorer spanning SQL/NoSQL/Big Data with driver management. **[INTERPRETATION]** pgAdmin and SSMS are deep but vendor-specific; DBeaver is broad but shallower per engine. All are GUI-first.

**Feature comparison matrix (Karkain = proposed).**

| Capability | Karkain Database Studio (proposed) | pgAdmin | SSMS | DBeaver |
|---|---|---|---|---|
| Deep Postgres/SQL Server admin | No | PostgreSQL | SQL Server | Moderate |
| GUI admin console | Blocked | Yes | Yes | Yes |
| CLI/TUI scriptable admin | Yes | Limited | Limited | CLI edition (paid) |
| Embedded local DB + transactions | Yes (`db2`) | No | No | No |
| Cross-DBMS breadth | No | No | No | Yes |
| Web/multi-user mode | No | Yes | No | CloudBeaver |
| Graphical query plans / grant wizards | No | Yes | Yes | Partial |

**Platform/deployment.** **[VERIFIED]** pgAdmin runs on Windows/Linux/macOS in desktop or server mode and ships as builds/Wheel/Docker/RPM/DEB; SSMS is Windows-only (Visual Studio 2026 shell, v18.x); DBeaver offers desktop plus web/hybrid editions. Karkain Database Studio: one native Windows binary (toolchain VERIFIED), Linux/macOS cross-build artifacts present but execution unverified.

**Licensing.** **[VERIFIED]** SSMS is a free Microsoft download; DBeaver Community is open source with paid editions (Lite $113/user/yr, Enterprise $255/user/yr, Ultimate $510/user/yr, accessed 2026-11-21). pgAdmin's exact licence text **UNKNOWN — not verified**. No price is invented. Karkain Database Studio is Apache-2.0.

**Strengths/limitations.** **[VERIFIED/INTERPRETATION]** Incumbents offer deep object editing and graphical tuning but carry web/JVM/Visual Studio footprints and vendor lock-in. Karkain offers scriptable, headless admin of a local embedded DB but cannot match their depth.

**Market gaps / differentiation.** **[INTERPRETATION]** CI-friendly, headless administration of embedded databases is underserved. **[PROPOSAL]** Differentiate on deterministic transaction control, audit logging and backup/restore for `db2` in a single binary — complementarity, not parity.

**Features deliberately not copied.** Graphical query plans, grant/RLS wizards, multi-user web mode, external DBMS drivers, AI assistants.

## D. Technical architecture

**Application architecture.** DBS-CLI dispatches admin commands to DBS-SQL/DBS-EXEC/DBS-TXN/DBS-PREP/DBS-SCHEMA, with DBS-BACKUP for durability operations and DBS-AUDIT/DBS-DIAG observing all actions.

**Component boundaries.** SQL parsing is separated from execution; transaction policy is centralized so every execution path honours commit/rollback rules; backup is isolated so file operations are testable independently.

**Process/service boundaries.** Single process, no daemon in MVP. A future read-only admin service could wrap the same modules with `http` building blocks.

**UI architecture.** The UI is CLI/TUI/service-first because Karkain v1.2.0 has no native GUI toolkit (BLOCKED); any GUI is a planned, blocked layer contingent on ecosystem capability and is not claimed. The "SQL editor" is a script/buffer workflow driven by files and config, not a graphical editor widget.

**Domain model.** Database, SqlStatement, SqlBatch, Transaction, PreparedStatement, SchemaObject, Backup, AuditEntry — see section E.

**Data storage strategy.** State lives in the embedded `db2` database file; backup files are copies with a checksum; audit is an append-only local log. No external database server is used.

**API/interface architecture.** Internal Karkain module APIs; external interface is the command/config surface and CLI text output. No REST API in MVP.

**Event/message architecture.** No event bus; execution is synchronous and deterministic to preserve transaction ordering.

**Logging/diagnostics/observability.** DBS-AUDIT records each statement with outcome and affected-row count; DBS-DIAG emits command-level run records and exit codes; build-time lint/check used.

**Configuration management.** Versioned local config file for database/backup paths, transaction policy and verbosity.

**Plugin/extension architecture.** A statement-handler registry keyed by SQL verb; new verbs (within the `db2` subset) are Karkain module additions.

## E. Data design

**Main entities.** `Database` (path, state); `SqlStatement` (text, verb, ordinal); `SqlBatch` (ordered statements, transaction policy); `Transaction` (id, state, start time); `PreparedStatement` (name, parameter count); `SchemaObject` (type, name, columns); `Backup` (source, destination, checksum, timestamp); `AuditEntry` (command, statement, outcome, rows).

**Relationships.** A SqlBatch owns ordered SqlStatements and zero or one Transaction; a Transaction scopes many executions; a PreparedStatement is created and reused within a session; a Backup references one Database; AuditEntries are appended per execution.

**Ownership and validation.** All state is owned by the single process. Validation: statements checked against the `db2` SQL subset before execution; identifiers validated before DDL; transaction state machine rejects commit/rollback when no transaction is open.

**Lifecycle and persistence.** Databases are opened, modified and closed; transactions move open→committed/rolled-back; backups are created, verified and retained; audit logs are append-only.

**Transactions, backup/restore.** Every modifying batch runs under an explicit transaction; failure triggers `db_rollback`. Backup is taken when no transaction is open; restore verifies the checksum before writing the target. OS-level atomic-write is not verified, so restore correctness is validated by reopen-and-inspect, not assumed.

**Migration.** No schema format predates v1.2.0; config and backup headers carry version fields for future migration.

**Import/export formats.** SQL script files (UTF-8 text), delimited-text bulk load, result export to delimited text, and full backup files with checksum.

**Audit history.** Append-only audit log of every statement and outcome, plus transaction boundaries, providing a DBA audit trail.

**Storage decision.** A database is inherent to this product: the embedded `db2` file is both the subject of administration and the source of transactional guarantees. Audit and backup use files to keep the database portable.

## F. Karkain implementation architecture

**Required Karkain capabilities / compiler.** Karkain v1.2.0 only; front-end lint and Go-engine C-free native Windows build verified. No compiler modifications.

**stdlib actually present (per capability register).** `db` (PRESENT/PARTIAL: `db2` SQL subset CREATE/INSERT/SELECT/UPDATE/DELETE/DROP, `db_begin/commit/rollback`, `db_prepare`/`db_execute_stmt`; no joins/external drivers), `io` (PRESENT: `io_read_all`, `io_write_file`, `io_append_file`), `collections` (PRESENT), `crypto` (PRESENT: `sha256` for backup checksums), `encoding` (PRESENT), `string`/`config` (PRESENT, unverified inventory), `system`/`env`/`process`/`time` (PARTIAL shell wrappers).

**Required ecosystem components.** None beyond v1.2.0 stdlib and compiler; optional `karkain test`/`lint`/`ide info`.

**Missing dependencies.** No full JSON DOM; no external SQL drivers/joins; no GUI toolkit; no portable process/time APIs; no verified OS atomic-write.

**Native implementation opportunities.** SQL statement splitter, transaction state machine, backup checksum/verify and schema inspection can all be pure Karkain over `db`/`io`/`crypto`.

**FFI/OS integration needs.** None required for MVP; no FFI is justified, so none is proposed. Avoid PowerShell-backed `filesystem` in favour of `io`.

**Build/packaging.** `karkain build src/main.kark -o karkain-database-studio` with the verified Go-engine/C-free native Windows target; single-binary distribution.

**Explicit blockers (recorded, not faked).** GUI BLOCKED; embedded SQL subset with no joins/external drivers; crash-atomic file writes PARTIAL; `get_args()` empty and `system()` returns exit code not stdout, so config-file dispatch is mandatory until probed; TLS/WebSocket/UDP BLOCKED. Each belongs to language/ecosystem work; no alternate language may be used as a workaround.

## G. Non-functional requirements

Targets use workload + measurement method (~4 GB RAM); PLANNED acceptance goals, not measured results.

| Area | Target (workload → method) |
|---|---|
| Performance | Execute a 1,000-statement insert batch → time to commit; report p50 of 5 runs. |
| Latency | Single prepared-statement execute → elapsed time to result. |
| Memory | 1,000-statement batch → peak RSS sampled from OS tool. |
| CPU | Batch workload → average CPU% over run. |
| Scalability | Batch sizes 100/1k/10k → near-linear time growth recorded. |
| Reliability | Inject a failing statement mid-batch → rollback restores prior state, audit complete. |
| Security | Local-only, no network surface; DDL identifier validation verified by review. |
| Privacy | All processing local; no telemetry (log inspection). |
| Portability | Windows build VERIFIED; produce Linux/macOS cross artifacts (execution unverified). |
| Maintainability | Each module lint-clean (`karkain lint`). |
| Accessibility | Deterministic text output, no color dependency. |
| Observability | Every command emits an audit record and a DBS-DIAG run record. |
| Recovery | Restore from backup after simulated corruption → reopen-and-inspect passes. |

## H. Compatibility and portability

**Supported OS.** Windows toolchain VERIFIED; Linux and macOS NOT VERIFIED. **Architectures.** x86-64 verified; others unverified. **Karkain v1.2.0 compatibility matrix.** v1.2.0 only; no support for 1.1.0 or earlier. **Binary/package formats.** Native Windows `.exe`; source + config distribution. **Install/uninstall.** Copy/remove binary and config; database files are user data. **Config/data migration.** Versioned fields; no pre-v1.2.0 backward compatibility required. **Upgrade/rollback.** Replace binary; database files remain compatible if format unchanged. **Cross-platform differences.** Avoid PowerShell-backed `filesystem`; use `io`. **Unsupported features.** GUI, external SQL/joins, multi-user/web mode, TLS/WebSocket/UDP.

## I. Testing strategy

All tests are **PLANNED**; none have been run for this application.

- **Unit:** SQL splitter, transaction state machine, checksum utils → exact expected results on fixtures.
- **Integration:** create→insert→select→update→delete→drop round-trip → expected states.
- **E2E:** config-driven script execution → golden stdout and audit log.
- **Platform-specific:** Windows native execution; cross-target artifact creation.
- **Security:** no network calls; DDL validation review.
- **Performance:** batch timings per section G recorded.
- **Regression:** golden outputs frozen after each milestone.
- **Compatibility:** lint/build with v1.2.0 only.
- **Actual-compiled-executable:** run built `.exe` for E2E.
- **Failure/recovery:** failing statement and corrupted backup → rollback and verified restore.

## J. Implementation roadmap

**MVP.** DBS-CLI, DBS-SQL basic runner, DBS-EXEC core statements, DBS-TXN explicit control, DBS-DIAG.
**Core functional release.** DBS-PREP prepared statements, DBS-SCHEMA inspection, DBS-AUDIT log, script failure policy.
**Advanced release.** DBS-BACKUP verified backup/restore, delimited bulk load, result export.
**Production-readiness release.** Hardened validation, recovery/performance tests, cross-target packaging.

**Feature dependencies.** Execution depends on dispatch; transaction control depends on execution; prepared statements depend on execution; backup depends on a quiescent (no open transaction) state; audit depends on all command paths.

**Milestones/gates.** Gate 1: compiled binary runs a SQL script with commit/rollback. Gate 2: prepared statements and schema inspection. Gate 3: backup/restore with verification. Gate 4: full test pass and packaging. No dates assigned.

**Risks/mitigations.** `get_args()` empty → config dispatch (mitigated). No atomic write → verify-by-reopen (mitigated). SQL subset/no joins → scope limit (accepted). No GUI → CLI/TUI by design (accepted).
