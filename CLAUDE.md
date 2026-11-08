# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose of this clone: LEARNING

This is a local clone of `apache/fineract` used for **study, not production work**. The user is a senior backend engineer (Java/Spring Boot, Kafka, PostgreSQL, payments/wallets background) learning how a core banking system works: loan lifecycle, interest and schedules, savings, double-entry GL, close-of-business batch jobs, multi-tenancy, security.

- Default mode is explain and trace: follow a flow through the code, name the classes involved with `file:line`, draw sequence diagrams, state the accounting entries an action produces. Verify in the code before asserting how something works.
- Do not modify source, commit, or push unless asked. Experiments go on a local branch.
- Study roadmap lives in this repo root: `fineract_topics.md` (one linear checklist, steps 1-52). Log verified findings there.
- Fineract has no card issuing, VCN or tokenization. Don't look for them.

## Build requirements

- **JDK 25** is required (`languageVersion = JavaLanguageVersion.of(25)` in `build.gradle`). The shell default here is JDK 21 (sdkman `current`), which will not work. `25.0.4-tem` is installed: `sdk use java 25.0.4-tem` before running Gradle.
- Gradle wrapper only (`./gradlew`). `gradle.properties` sets `-Xmx12g` for the daemon, so it needs real RAM.
- PostgreSQL is the supported database. MariaDB/MySQL work but are deprecated.
- `./gradlew` runs Spotless and Checkstyle as part of the normal build and fails on violations.

## Commands

```bash
# One-time: create the two databases (PostgreSQL on localhost:5432)
./gradlew createPGDB -PdbName=fineract_tenants
./gradlew createPGDB -PdbName=fineract_default

./gradlew devRun                    # run the server, HTTPS on 8443; wait for "APACHE FINERACT IS READY"
./gradlew bootRun -Dspring.profiles.active=test   # same, with the test profile (needed for E2E)
./gradlew clean bootJar             # fat JAR in fineract-provider/build/libs
./gradlew spotlessApply             # auto-fix formatting (also fixes some Checkstyle issues)
./gradlew spotlessCheck
./gradlew rat                       # Apache license header audit

# Unit tests only (~5 min, ~1,500 tests, no server needed)
./gradlew test -x :twofactor-tests:test -x :oauth2-tests:test -x :integration-tests:test -x :fineract-e2e-tests-runner:test

# Single unit test class / method (module-qualified to avoid running every module)
./gradlew :fineract-provider:test --tests CommandSourceServiceTest
./gradlew :fineract-provider:test --tests "*Loan*"
```

Health check and smoke call (default login `mifos` / `password`, tenant `default`):

```bash
curl --insecure https://localhost:8443/fineract-provider/actuator/health
curl --insecure https://localhost:8443/fineract-provider/api/v1/clients \
  -H 'Fineract-Platform-TenantId: default' -H 'Authorization: Basic bWlmb3M6cGFzc3dvcmQ='
```

Swagger UI: `https://localhost:8443/fineract-provider/swagger-ui/index.html`.

### Integration and E2E tests need a live server

`:integration-tests`, `:twofactor-tests`, `:oauth2-tests` and the Cucumber `:fineract-e2e-tests-runner` do **not** start Fineract. Start one first (`devRun`/`bootRun`, or the docker compose files) or every API test fails.

```bash
./gradlew :integration-tests:test --tests ClientLoanIntegrationTest
./gradlew :integration-tests:test --tests ClientLoanIntegrationTest.testLoanRepayment

# Cucumber E2E; INITIALIZATION_ENABLED seeds test data on a fresh DB
cd fineract-e2e-tests-runner && INITIALIZATION_ENABLED=true ../gradlew cucumber
```

Point the suites at a non-default server with `BACKEND_PROTOCOL`, `BACKEND_HOST`, `BACKEND_PORT`. Reading the integration tests is the best executable documentation of API flows (create client, product, loan, approve, disburse, repay).

## Architecture

Fineract is a **headless** core banking server: one HTTPS REST API, no bundled UI. The Mifos X Angular UI (`openMF/web-app`) is a separate project that calls this API. Docs in `fineract-doc/src/docs/en/chapters/architecture/` (`system.adoc`, `principles.adoc`, `loan-locking.adoc`, `batch-jobs.adoc`, `reliable-event-framework.adoc`, `persistence.adoc`) are worth reading before tracing code.

### Module layout (Gradle multi-module, see `settings.gradle`)

Domain code is split across modules, but `fineract-provider` is still the assembly point and holds much of the legacy portfolio code:

- `fineract-provider`: Spring Boot app (`ServerApplication`), `application.properties`, Liquibase changelogs, and legacy `portfolio/*` (client, group, savings, loanproduct, shares...) and `accounting/*` packages.
- `fineract-core`: shared infrastructure: command pipeline, tenant routing, jobs/Spring Batch glue, business date, events, security base.
- `fineract-loan`, `fineract-savings`, `fineract-accounting`, `fineract-charge`, `fineract-tax`, `fineract-investor`, `fineract-loan-origination`, `fineract-progressive-loan*`, `fineract-working-capital-loan`: extracted domain modules. A feature's code may be split between these modules and `fineract-provider`, so grep both.
- `fineract-cob`: Close of Business (end-of-day) Spring Batch jobs.
- `fineract-command*` (`-jdbc`, `-async`, `-audit`, `-disruptor`, `-test`): newer command-processing infrastructure alongside the legacy one in `fineract-core`.
- `fineract-client`, `fineract-client-feign`: generated SDKs from the OpenAPI descriptor. Backward compatibility of the API is checked automatically.
- `fineract-war`, `custom/`: WAR packaging and the custom-module extension point.

### Request path: write commands vs queries

Command Query Separation is the central pattern. State-changing calls do not go straight to a service:

1. A REST resource (Jakarta JAX-RS `*ApiResource`, e.g. `LoansApiResource`) builds a `CommandWrapper` via `CommandWrapperBuilder` (action + entity + permission code).
2. `PortfolioCommandSourceWritePlatformService` -> `SynchronousCommandProcessingService` (`fineract-core/.../commands/service/`) persists the command to the `CommandSource` table first. This gives the audit trail, **maker-checker** (command held until approved), and **idempotency** (key via `IdempotencyKeyResolver`).
3. `CommandHandlerProvider` resolves the handler (`@CommandType(entity, action)`), which calls the write service, which mutates the domain entities and, for money-moving actions, posts journal entries.
4. Reads go through separate read-platform services returning DTOs, never through the command source.

Business events are raised from the write side and published via the reliable event framework (Avro schemas in `fineract-avro-schemas`) to Kafka or ActiveMQ when enabled. Hooks are a second, older notification mechanism.

### Multi-tenancy

One database per tenant plus a tenant-store DB (`fineract_tenants`) holding connection details. The `Fineract-Platform-TenantId` header is resolved in the security filter chain (`TenantAwareBasicAuthenticationFilter` in `fineract-security`), and `RoutingDataSource` (`fineract-core/.../core/service/database/`) picks the tenant's DataSource per request. Liquibase changelogs live in `fineract-provider/src/main/resources/db/changelog/` with separate `tenant-store/` and `tenant/` trees.

### Accounting

Loan, savings and share actions post double-entry journal entries through `AccountingProcessorFor*` classes (e.g. `AccountingProcessorForLoan`) using `AccountingProcessorHelper` (`fineract-accounting`). Which GL accounts are hit depends on the product's accounting rule (none / cash / accrual periodic / accrual upfront) and its account mappings. Tracing "what entries does this action make" means: write service -> accounting service -> processor for that account type.

### Instance types, business date, COB

The same binary runs as write, read and/or batch instance, set by configuration (all three by default). The platform has its own **business date** (not wall clock) that COB advances. Loan COB runs as partitioned Spring Batch jobs; loans being processed are protected by rows in `m_loan_account_locks` (soft lock = inline COB on write, hard lock = HTTP 409). See `loan-locking.adoc`.

### Naming and conventions worth knowing

- Tables use `m_` prefix (`m_loan`, `m_savings_account`, `m_client`); accounting tables use `acc_gl_*`.
- Extension points are custom modules/plugin JARs, hooks, custom command handlers and configurable COB business steps. Don't add origination workflow or scoring inside the core.
- Commit message style upstream: `FINERACT-<jira id>: Imperative description`.
