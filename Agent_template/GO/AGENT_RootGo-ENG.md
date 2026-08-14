# Repository and Go Server Rules Template

> Purpose: This is a self-contained template that merges repository-root and Go server rules.
> Usage: Copy it to the root of a Go repository and rename it to `AGENTS.md`. If the repository also contains frontend or other services, keep shared rules at the root and add focused `AGENTS.md` files in child directories.

## Scope and Precedence

- These rules apply to this repository and all descendants.
- A deeper `AGENTS.md` may add technical rules. On conflict, the rules closest to the target file win.
- When conventions are unclear, inspect nearby code, tests, the Makefile, README, CONTRIBUTING, and development documentation. Do not substitute memory or guesses for verification.
- Confirmed project requirements and documentation take precedence over general advice. When a skill is used, its workflow rules apply together with this file.

## Language and Communication

- Use Traditional Chinese (Taiwan) by default for user communication, documentation, commit messages, PR titles, and PR descriptions.
- Preserve the language already used by code. Write comments and docstrings in Traditional Chinese and explain why rather than restating each line.
- English is limited to code, paths, URLs, API fields, technical names, and necessary established terms.
- At handoff, report the result first, then root cause, validation evidence, unverified scope, and deployment or rollback impact.
- State unknown information and assumptions explicitly. Never report unverified work as complete.

## Specifications, Requirements, and Task Tracking

- For new features, requirement changes, out-of-scope bug fixes, or requests to plan first, use the SDD workflow and create or update requirements, design, and an unfinished task.
- The project defines the specification workspace. A private workspace may use `$PRIVATE_SPEC_WORKSPACE/<repository>/.specs/<space>/`. Never copy private requirements, operational data, internal identifiers, or business decisions into a public repository.
- For an in-scope change, first confirm its matching task. If no unfinished task exists, add one and wait for user scheduling before implementation.
- Requirements, design, tasks, code, and validation must remain traceable. Do not append new work to a completed task; create a new unfinished task instead.
- Mark a task `Complete` only when every acceptance criterion has actual evidence.

## Change Boundaries and Cross-Layer Contracts

- Inspect nearby code, types, configuration, and tests before changing anything. Follow existing patterns and do not assume libraries, services, or fields exist.
- Keep changes within the user request and task boundary. Do not bundle unrelated refactors or incidental fixes.
- Update backend behavior, OpenAPI, frontend schemas/types, documentation, and the specification together when a shared API contract changes.
- Frontends must not recalculate backend-owned business metrics, report definitions, or authorization outcomes.
- Do not change API semantics, data definitions, authorization behavior, retention policies, or compatibility commitments without confirmation.

## Security and Data Protection

- Do not hardcode or output passwords, tokens, DSNs, API keys, personal data, or other sensitive information.
- Validate external input for format, length, range, enumerations, and authorization.
- Use parameterized queries. Never concatenate user input into SQL, shell commands, file paths, or URLs.
- Error responses and logs must not expose SQL, stack traces, configuration values, or internal topology.
- Use only standard `crypto/*` for cryptography. Assess regexes, path handling, archive extraction, and external URLs for ReDoS, path traversal, Zip Slip, SSRF, and uncontrolled expansion risks.

## Git, Pull Requests, and Merges

- Use feature branches for features and fixes, with `codex/` as the default prefix.
- Do not commit, push, create or edit PRs, rebase, force-push, merge, switch branches, or pull without explicit authorization for the current action.
- Commit and PR text accurately describe verified scope. Do not use “complete” to hide unimplemented or unverified work.
- Before merging, inspect the target branch, PR dependency order, migration and file conflicts, test state, and deployment and rollback impact.
- Do not use `git reset --hard` or other commands that overwrite or delete work without explicit authorization.
- Run `git diff --check` before handoff. After merging, synchronize the local default branch and report the merge commit and pending deployment work.

## Go Server: Before Editing

- Read two or three nearby files and tests in the same package. Preserve existing directories, filenames, types, and error-handling patterns.
- Use `make help`, README, CONTRIBUTING, or development documentation to identify supported format, test, lint, build, and generate commands.
- Fix root causes. Do not hide problems by disabling lint, weakening tests, ignoring errors, or introducing broad type assertions.

## Go Code Style and Error Handling

- New files use the package already used by their directory. Package names are lowercase, single words, and contain no underscores; avoid `util` and `common`.
- Group imports as standard library, third-party packages, and project packages. Do not keep unused imports or introduce import cycles.
- Use tabs, UTF-8 without BOM, one trailing newline, and project-defined formatting, lint, and `go vet` commands.
- Exported types, functions, and constants receive godoc comments following project convention. Comments record only constraints, compatibility, reasons, or exceptions.
- Keep functions focused, use early returns, and avoid unnecessary nested `else` blocks.
- `context.Context` is the first parameter at I/O, HTTP, database, queue, and external-service boundaries, and it must be propagated.
- Do not store `context.Context` in structs or add arbitrary `context.Background()` calls on request paths.
- Handle errors immediately. Add context with `fmt.Errorf("context: %w", err)` when useful, and use `errors.Is` or `errors.As` across layers.
- Expected failures return `error`. Libraries, handlers, and workers do not use `panic`, `log.Fatal`, or `os.Exit` for normal flows.
- Error strings start with lowercase letters and have no trailing punctuation unless project convention differs.
- Follow Go initialism conventions such as `URL`, `HTTP`, `ID`, and `API`.
- Avoid `any`, broad type assertions, mutable global state, and unmanaged goroutines. If unavoidable, document the reason in the smallest scope.

## Concurrency, Resources, and HTTP Clients

- Every goroutine needs an exit mechanism such as `context.Context`, `WaitGroup`, or channel closure and reacts to `ctx.Done()` during shutdown.
- Protect shared mutable state with `sync.Mutex`, `sync.RWMutex`, or a measured lock-free design.
- Place `defer Close()` in the appropriate scope. Do not accumulate defers in large loops; use a nested function when necessary.
- Copy slices or maps when they cross mutable ownership boundaries and could otherwise be shared unexpectedly.
- HTTP clients retain only configuration and reusable `*http.Client` instances, never per-request state.
- Use `http.NewRequestWithContext`, set reasonable timeouts, and close `resp.Body` after reading it.
- Reuse transports. Retry only idempotent operations with backoff, limits, and explicit retryable-error rules.
- Before rereading `req.Body`, create a safe copy and set `GetBody`. Keep `io.Pipe` and multipart writing synchronized with clear close ownership.

## JSON, Time, and APIs

- External input and output types use consistent `json` tags. Use `omitempty` only when the contract permits it.
- Reject unknown external JSON fields when compatibility allows it.
- Store and calculate time in UTC, use RFC3339 for JSON, and format for user time zones only at presentation boundaries.
- Use existing DTOs, validators, error handling, and authorization middleware. Do not create inconsistent handler-level contracts.
- Update OpenAPI, frontend schemas/types, specifications, and tests together with API changes. Obtain approval for breaking changes.
- Apply size and format limits to file input and protect archive handling from Zip Slip and uncontrolled expansion.

## Configuration, DI, and API Documentation

- Configuration management must use `spf13/viper`. Define precedence explicitly: environment variables override config files, and config files override defaults.
- Unmarshal into a typed Config and validate centrally. Do not read Viper throughout business code.
- Optional settings require explicit defaults. Required settings are validated at startup. Test environment mappings, nested keys, and empty values.
- Inject secrets through environment variables, Secrets, or managed configuration services. Do not store them in repositories, examples, or logs.
- Missing or invalid required configuration fails startup explicitly and returns to the unified lifecycle entry point.
- Compose infrastructure and application dependencies in an explicit composition root. Centralize `uber-go/fx` in `cmd/` or a clearly named DI root when used.
- Keep repository and service interfaces small and capability-focused. Keep concrete implementations unexported and use project-convention test doubles.
- Treat OpenAPI as a traceable HTTP contract. In services using `swag`, update Swagger comments and regenerate artifacts with the project command; do not hand-edit generated files.

## When Using gRPC or Domain Events

- Keep proto files in the project-defined location and never hand-edit generated code. Regenerate and commit code after RPC changes.
- gRPC servers honor deadlines and context cancellation, preserve recovery/observability/logging/authentication interceptor order, and implement health checking.
- Map domain errors to correct gRPC statuses and stop with `GracefulStop` or a timeout-bound flow.
- Domain events are immutable structs containing at least an event ID, occurrence time, aggregate ID, and event type; use past-tense names.
- Cross-context asynchronous events use explicit message boundaries. Outbox writes business changes and events in the same transaction.
- Consumers process duplicate events idempotently using event IDs or equivalent de-duplication keys.

## Lifecycle, Logging, and Observability

- Manage HTTP server, worker, consumer, and background-goroutine lifecycles with `github.com/vincent119/commons/graceful`.
- Stop accepting new work before waiting for active work, then close database, cache, queue, trace, and other external resources.
- Do not call `os.Exit` or `log.Fatal` in server goroutines or libraries; return errors to the unified lifecycle entry point.
- All application logging uses `github.com/vincent119/zlogger`. Do not mix standard `log`, `slog`, `zap`, `logrus`, or other loggers.
- Use structured logs and preserve correlation fields such as `trace_id`, `span_id`, `req_id`, and `subsystem`.
- Propagate `context.Context` across boundaries. Use consistent snake-case metric names and unit suffixes such as `_total`, `_seconds`, and `_bytes`.
- Do not use high-cardinality labels such as `user_id`, email, or `trace_id`, and do not include sensitive data in logs or metric labels.

## Testing, Databases, and Migrations

- For new observable behavior or a bug fix, prefer a failing test first, then the smallest fix, then refactoring.
- Every bug fix requires a regression test. Test names describe observable behavior and outcomes rather than private implementation.
- Prefer table-driven tests. Use `t.Helper()` for helpers, `t.Cleanup()` for cleanup, and `-race` when concurrency needs verification.
- Prefer isolated domain, service, parser, validator, transformation, and query-planner tests. Boundary tests cover repositories, HTTP, databases, and external services.
- Add benchmarks and fuzz tests for high-risk algorithms or parsers when appropriate; they are not mandatory for every small change.
- Do not weaken assertions, delete or skip tests, or move core logic outside tests merely to make tests pass.
- Documentation, formatting-only changes, and fully covered mechanical changes do not require test-first work, but still require proportionate checks.
- Never run schema migration in application startup. Production does not use ORM automatic migration such as GORM `AutoMigrate`.
- Use versioned schema migrations and never alter one applied in any environment.
- Measure query plans, data volume, lock risk, and expected benefit before adding indexes, views, materialized views, aggregate tables, or backfills.
- Migrations, backfills, schedules, and rollbacks have traceable deployment procedures. Do not apply production SQL without authorization.

## Verification and Handoff

- Run proportionate format, test, lint, build, generate, and security checks. Cross-layer changes verify both sides of the contract.
- If verification cannot run, state the reason, unverified scope, and risk.
- Update existing documentation for user-facing features, APIs, configuration, deployment, or operations. Do not create duplicate parallel documents.
- Explain and obtain approval before adding dependencies, changing architecture, changing shared configuration, or changing deployment behavior.
- At handoff, list changed files, behavioral impact, validation results, and migration, configuration, or deployment notes.
