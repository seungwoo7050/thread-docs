## `docs(project): document wallet 1.0 contracts`

diff --git a/README.md b/README.md
index e5a4179..ab5a62c 100644
--- a/README.md
+++ b/README.md
@@ -1,21 +1,52 @@
 # Sportsbook Wallet Service
 
-The wallet service owns user balances and the append-only double-entry ledger for the
-sportsbook backend. It is the only service allowed to mutate available or locked funds.
+The wallet service is the authoritative owner of sportsbook account balances, locked funds,
+matched double-entry ledger entries, durable operation outcomes, settlement adjustments, and
+wallet integration events. No other service may mutate a wallet account.
 
-## Responsibilities
+PostgreSQL owns correctness. Redis is a best-effort idempotency hint, and Kafka publication is
+driven from the PostgreSQL outbox.
 
-- keep available and locked balances in one currency per account;
-- record every money movement as a balanced debit-credit pair;
-- make caller retries resolve to one durable operation outcome;
-- publish wallet integration events through a transactional outbox;
-- expose internal account, transfer, and settlement-adjustment APIs.
+## Prerequisites
 
-## Runtime
+- Java 17.
+- Maven 3.9.11, provided by `./mvnw`.
+- `com.sportsbook:shared-protocol:1.0.0` installed in the local Maven repository before building
+  the wallet.
+- PostgreSQL for an application run; Redis is optional, and Kafka is required when outbox delivery
+  is enabled.
+- Docker for both verification commands.
 
-The service uses Java 17, Spring Boot, PostgreSQL, Redis, Kafka, Avro, and Maven. PostgreSQL
-owns correctness. Redis is only an optional replay hint, and Kafka publication is driven from
-the database outbox.
+## Run
 
-Build and runtime instructions are added as the executable project is introduced. The final
-project documentation records the API, security, recovery, and operational contracts.
+Provide five distinct internal API keys of at least 32 characters. Keep them in the runtime secret
+manager; never store them in the repository.
+
+```bash
+env \
+  WALLET_PLATFORM_API_KEY='<environment-provided secret>' \
+  WALLET_GATEWAY_API_KEY='<environment-provided secret>' \
+  WALLET_BETTING_SERVICE_API_KEY='<environment-provided secret>' \
+  WALLET_SETTLEMENT_SERVICE_API_KEY='<environment-provided secret>' \
+  WALLET_ADMIN_API_KEY='<environment-provided secret>' \
+  ./mvnw spring-boot:run
+```
+
+The remaining environment settings and their exact defaults are listed in
+[Operations](docs/operations.md).
+
+## Verify
+
+```bash
+./mvnw clean verify
+./mvnw -Psemantic-gates clean verify
+```
+
+Both commands are container-backed. The semantic profile selects the tagged PostgreSQL 16,
+Redis 7, and Kafka 3.8 subset.
+
+## Contracts
+
+- [Architecture and invariants](docs/architecture.md)
+- [Internal HTTP API](docs/internal-api.md)
+- [Operations and observability](docs/operations.md)
diff --git a/docs/architecture.md b/docs/architecture.md
new file mode 100644
index 0000000..74e49bc
--- /dev/null
+++ b/docs/architecture.md
@@ -0,0 +1,135 @@
+# Architecture and invariants
+
+## Ownership and authoritative state
+
+PostgreSQL is the sole correctness authority for accounts, operations, adjustments, ledger entries,
+outbox events, recovery debt, and delivery leases. The wallet service is the only component allowed
+to mutate `available` or `locked` funds. Redis and Kafka never decide whether a monetary operation
+committed.
+
+Each user account has one currency and two nonnegative buckets:
+
+- `available`: funds that can be withdrawn or moved into a bet;
+- `locked`: funds reserved by a debit while the bet is open.
+
+The accepted currencies are `KRW` and `USD`. `available + locked` must fit in a signed 64-bit
+integer and may not exceed `Long.MAX_VALUE`. `HOUSE` and `EXTERNAL_PAYMENT` are reserved ledger
+counterparties and cannot be opened as user accounts.
+
+## Monetary writes and the ledger
+
+A successful money movement commits the account mutation, operation outcome, and matched ledger
+pair in one PostgreSQL transaction. The two ledger rows share the operation group, amount,
+currency, business reason, and timestamp, and use opposite ledger sides. A transaction or
+infrastructure failure rolls back the write. A recognized business rejection commits no balance or
+ledger mutation, but does commit immutable `REJECTED` operation facts, an adjustment proof when
+applicable, and a debit-failed outbox event for a rejected debit.
+
+The ledger reasons are:
+
+- `DEPOSIT` and `WITHDRAW` for external payment movement;
+- `BET_DEBIT` for available-to-locked movement;
+- `BET_PAYOUT` and `BET_REFUND` for credits;
+- `BET_FORFEIT` for locked funds moved to the house;
+- `BET_ADJUSTMENT` for settlement corrections.
+
+Integrity scans verify account snapshots, ledger topology, operation groups, recovery queues,
+adjustment outcomes, failure facts, semantic fingerprints, and adjustment ledger pairs against one
+repeatable-read database view.
+
+## Idempotency and stored outcomes
+
+Every monetary POST is identified by exactly one `Idempotency-Key`. The first writer is serialized
+with a PostgreSQL advisory lock scoped to its transaction. A versioned canonical binary encoding of
+the authenticated caller and all semantic request fields is hashed with SHA-256 and persisted with
+the operation.
+
+When the same key and semantic fields are submitted again, the service returns the stored operation
+outcome. An adjustment can return a stored `BLOCKED` proof and later an `APPLIED` proof after
+recovery. Reusing the key with different semantic fields returns `WALLET_IDEMPOTENCY_CONFLICT`.
+Stored rejection facts include their status, title, detail, code, and any balance or
+expected-currency fact; they are not rebuilt from mutable account state.
+
+Authentication, route authorization, malformed JSON, header validation, and semantic credit
+authorization occur before a durable operation is created. Those failures do not consume an
+idempotency key. A retryable PostgreSQL availability failure also leaves the key free when no
+operation committed.
+
+Redis contains only a 24-hour best-effort marker at `idempotency:wallet:<key>`. A missing marker or
+an unavailable Redis instance always falls back to PostgreSQL. Redis loss cannot change a committed
+outcome.
+
+## Settlement adjustment and recovery
+
+An adjustment identifies a `(betId, revisionNumber)` pair and carries nonnegative previous and new
+payout snapshots in the same currency. The delta must be nonzero.
+
+- A positive delta credits available funds immediately.
+- A negative delta applies immediately when the account can pay and has no blocked queue head.
+- Otherwise, the negative delta becomes a `BLOCKED` proof with a positive per-account
+  `queue_sequence`; the absolute delta is added to recovery debt.
+
+Recovery debt sets `outboundFrozen`. Only withdraw and debit enforce this freeze. Deposits, credits,
+forfeits, and settlement inflows remain available. A deposit, credit, or positive adjustment wakes
+the queue head in the same transaction as the inflow; it never runs recovery inline. This lock
+ordering prevents a missed wake between an inflow and the worker.
+
+The worker processes one due account in its own transaction:
+
+1. lock a due account with `FOR UPDATE SKIP LOCKED`;
+2. lock its first `BLOCKED` proof by `queue_sequence`;
+3. lock the linked operation;
+4. either defer the head or apply the full correction.
+
+Insufficient funds change only `retry_count`, `next_attempt_at`, and the proof observation time. No
+balance, ledger row, or operation status changes. Sufficient funds atomically write the full two-leg
+`BET_ADJUSTMENT` transfer, move the proof from `BLOCKED` to `APPLIED`, move the operation from
+`BLOCKED_FUNDS` to `SUCCEEDED`, reduce debt, and wake the next head. The account unfreezes only when
+debt reaches zero. Partial application and queue bypass are invalid states.
+
+## Transactional outbox
+
+Debit and credit transactions append their event in the same PostgreSQL transaction as the wallet
+outcome. Adjustments do not emit these events.
+
+| Topic | Avro record |
+| --- | --- |
+| `wallet.debited.v1` | `com.sportsbook.protocol.event.WalletDebited` |
+| `wallet.debit-failed.v1` | `com.sportsbook.protocol.event.WalletDebitFailed` |
+| `wallet.credited.v1` | `com.sportsbook.protocol.event.WalletCredited` |
+
+The binary Avro contracts come from `shared-protocol` 1.0.0. Every Kafka record uses the user UUID
+string as its key and has one US-ASCII `event-id` header containing the outbox event UUID. There is
+no schema header.
+
+The outbox assigns a monotonic `stream_sequence` per `(topic, partition_key)`. A row is claimable
+only when no preceding unpublished row exists for that stream. Claims use a short
+`REQUIRES_NEW` transaction with `FOR UPDATE SKIP LOCKED`, so independent user streams can proceed
+concurrently.
+
+Kafka send and broker acknowledgement occur outside the claim transaction. Only after the broker
+acknowledges the send does a separate transaction mark the row published. Both publication and
+retry completion require the exact owner and `lease_version`, preventing a stale worker from
+completing a lease taken by another process. Expired leases can be claimed again.
+
+Delivery is at-least-once. A process failure after broker acknowledgement and before the database
+mark can produce a duplicate, so consumers must deduplicate by `event-id`. Failures are retried
+without an attempt limit, using exponential delay capped at 60 seconds. Automatic publication is
+off by default and requires deliberate operational enablement.
+
+## Time and causal order
+
+Lifecycle and event observations such as `created_at`, `updated_at`, `requested_at`,
+`completed_at`, `queued_at`, `applied_at`, `published_at`, and event occurrence time are not causal
+ordering authorities. Clock movement between transactions may make a later observation numerically
+earlier than a prior observation.
+
+Causal state and ordering come from operation and adjustment status, account version,
+`queue_sequence`, `stream_sequence`, and `lease_version`. Operational deadlines use the PostgreSQL
+clock:
+
+- outbox `available_at` and `lease_until`;
+- adjustment `next_attempt_at`.
+
+Queries compare those deadlines with the database clock. Business correctness never depends on
+cross-transaction timestamp chronology.
diff --git a/docs/internal-api.md b/docs/internal-api.md
new file mode 100644
index 0000000..74940d0
--- /dev/null
+++ b/docs/internal-api.md
@@ -0,0 +1,219 @@
+# Internal HTTP API
+
+All wallet routes are under `/internal/v1/wallet`. The boundary is stateless and closed: an
+authenticated request with an unlisted method or path receives `403`, while a request without valid
+credentials receives `401`.
+
+## Authentication
+
+Send exactly one value for each header:
+
+```http
+X-Internal-Service: <caller wire name>
+X-Internal-Api-Key: <environment-provided secret>
+```
+
+The caller wire name is case-sensitive and is not trimmed.
+
+| Caller | Wire name |
+| --- | --- |
+| Platform | `platform` |
+| Gateway | `gateway` |
+| Betting | `betting-service` |
+| Settlement | `settlement-service` |
+| Admin | `admin-api` |
+
+Missing one header, duplicating either header, using an unknown caller, or presenting a bad key
+returns `401 WALLET_AUTHENTICATION_REQUIRED`. Valid credentials for a forbidden route or credit
+meaning return `403 WALLET_ACCESS_DENIED`.
+
+Anonymous `GET` access is limited to `/actuator/health`, `/actuator/health/**`, and
+`/actuator/prometheus`. Platform credentials are required for `/actuator`, `/actuator/**`, and all
+other management endpoints.
+
+## Route capabilities
+
+| Method and path | Allowed caller | Success response |
+| --- | --- | --- |
+| `POST /internal/v1/wallet/accounts` | Platform | `200 AccountResponse` |
+| `GET /internal/v1/wallet/accounts/{userId}/balance` | Platform, Gateway | `200 BalanceResponse` |
+| `POST /internal/v1/wallet/transactions/deposit` | Platform | `200 WalletOperationResponse` |
+| `POST /internal/v1/wallet/transactions/withdraw` | Platform | `200 WalletOperationResponse` |
+| `POST /internal/v1/wallet/transactions/debit` | Betting | `200 WalletOperationResponse` |
+| `GET /internal/v1/wallet/transactions/debit/{betId}` | Betting | `200 WalletOperationResponse` |
+| `POST /internal/v1/wallet/transactions/credit` | Betting, Settlement, Admin | `200 WalletOperationResponse` |
+| `POST /internal/v1/wallet/transactions/forfeit` | Settlement | `200 WalletOperationResponse` |
+| `POST /internal/v1/wallet/transactions/adjustment` | Settlement | `200` or `202 AdjustmentProofResponse` |
+| `GET /internal/v1/wallet/transactions/adjustment/{revisionId}` | Settlement | `200 AdjustmentProofResponse` |
+
+Account, debit, and adjustment lookups return their specific `404` problem when the requested
+resource does not exist.
+
+Debit GET returns `200 WalletOperationResponse` for a stored success, the stored ProblemDetail and
+status for a stored rejection, and `404 WALLET_OPERATION_NOT_FOUND` only when no matching debit
+operation exists.
+
+## Credit semantic allowlist
+
+Route access alone is not enough for a credit. Exactly these five caller, source, and reason
+combinations are allowed:
+
+| Caller | `source` | `reason` |
+| --- | --- | --- |
+| `betting-service` | `USER_LOCKED` | `REFUND` |
+| `settlement-service` | `USER_LOCKED` | `VOID` |
+| `settlement-service` | `USER_LOCKED` | `REFUND` |
+| `settlement-service` | `HOUSE_POOL` | `PAYOUT` |
+| `admin-api` | `HOUSE_POOL` | `REFUND` |
+
+Every other combination returns `403 WALLET_ACCESS_DENIED` before execution.
+
+## Request identity
+
+Every transaction POST requires exactly one `Idempotency-Key` header. Account creation is the only
+POST in this API that does not use it. A general key must be nonblank printable ASCII, contain at
+most 128 characters, and is used exactly as supplied without trimming or normalization. A missing,
+duplicate, or invalid key returns `400 WALLET_INVALID_REQUEST` without calling the wallet service.
+
+Debit identity has an additional rule: the POST key is the bet ID and must be a canonical lowercase
+UUID string. The GET `{betId}` path must use the same canonical form. Uppercase, abbreviated, or
+malformed UUID text returns `400`.
+
+An adjustment key must exactly equal:
+
+```text
+settlement:revision:<revisionId>
+```
+
+The UUID text is the canonical form of the `revisionId` in the JSON body.
+
+Submitting the same key and semantic fields again returns the stored operation outcome. Submitting
+different semantic fields under that key returns `409 WALLET_IDEMPOTENCY_CONFLICT`.
+
+## JSON contracts
+
+`Money` requires these value fields:
+
+```json
+{"amount": 100, "currency": "KRW"}
+```
+
+`currency` is `KRW` or `USD`. Transaction amounts must be strictly positive.
+
+### Requests
+
+| Request | Required fields | Additional rules |
+| --- | --- | --- |
+| Open account | `userId`, `currency` | Both required; a reserved system UUID is invalid. |
+| Deposit, withdraw, debit, forfeit | `userId`, `amount` | Both required; `amount` is positive. |
+| Credit | `userId`, `amount`, `source`, `reason` | All required; `source` is `USER_LOCKED` or `HOUSE_POOL`; `reason` is `PAYOUT`, `VOID`, or `REFUND`. |
+| Adjustment | `revisionId`, `betId`, `revisionNumber`, `userId`, `previousPayout`, `newPayout` | All object fields required; revision number is at least 1; payout amounts are nonnegative, share a currency, and have a nonzero delta; the user cannot be a reserved system account. |
+
+### Account response
+
+`AccountResponse` contains exactly:
+
+1. `userId`
+2. `currency`
+3. `available`
+4. `locked`
+5. `outboundFrozen`
+6. `version`
+7. `createdAt`
+8. `updatedAt`
+
+### Balance response
+
+`BalanceResponse` contains exactly:
+
+1. `userId`
+2. `available`
+3. `locked`
+4. `total`
+5. `outboundFrozen`
+
+Account and balance responses intentionally omit recovery debt, queue sequence, and freeze timing.
+
+### Operation response
+
+`WalletOperationResponse` contains exactly:
+
+1. `operationGroupId`
+2. `userId`
+3. `amount`
+4. `reason`
+5. `at`
+
+The `reason` is the committed ledger reason, such as `DEPOSIT`, `WITHDRAW`, `BET_DEBIT`,
+`BET_PAYOUT`, `BET_REFUND`, or `BET_FORFEIT`.
+
+### Adjustment proof response
+
+`AdjustmentProofResponse` contains exactly these 14 fields:
+
+1. `revisionId`
+2. `betId`
+3. `revisionNumber`
+4. `userId`
+5. `previousPayout`
+6. `newPayout`
+7. `deltaAmount`
+8. `currency`
+9. `status`
+10. `queueSequence`
+11. `operationGroupId`
+12. `queuedAt`
+13. `appliedAt`
+14. `nextAttemptAt`
+
+`deltaAmount` is signed. `status` is `APPLIED`, `BLOCKED`, or `REJECTED`. The nullable fields
+`queueSequence`, `operationGroupId`, `queuedAt`, `appliedAt`, and `nextAttemptAt` remain present as
+JSON `null` when absent. Retry count, idempotency key, and persistence observation fields are not
+exposed.
+
+## Adjustment HTTP semantics
+
+- An immediately applied POST returns `200` and no `Location` header.
+- A blocked negative adjustment returns `202` with
+  `Location: /internal/v1/wallet/transactions/adjustment/{revisionId}`.
+- A durable rejection returns its stored problem status and body. The POST response contains no
+  proof representation or `Location`; the stored `REJECTED` proof remains available through GET.
+- A repeated POST after worker completion returns the `APPLIED` proof with `200`.
+- GET returns any `APPLIED`, `BLOCKED`, or `REJECTED` proof with `200`; an absent proof returns
+  `404 WALLET_ADJUSTMENT_NOT_FOUND`.
+
+## Problem details
+
+Errors use `application/problem+json` and RFC 9457 fields:
+
+- `type`
+- `title`
+- `status`
+- `detail`
+- `instance`
+- `errorCode`
+
+`instance` contains the request path without its query string. A stored business rejection may add
+`balance` or `expectedCurrency`; no request credential, idempotency key, database diagnostic, or
+exception text is reflected.
+
+| Status | `errorCode` | Title | `type` |
+| --- | --- | --- | --- |
+| 400 | `WALLET_INVALID_REQUEST` | Invalid wallet request | `https://sportsbook/errors/wallet/invalid-request` |
+| 401 | `WALLET_AUTHENTICATION_REQUIRED` | Authentication required | `https://sportsbook/errors/wallet/authentication-required` |
+| 403 | `WALLET_ACCESS_DENIED` | Wallet access denied | `https://sportsbook/errors/wallet/access-denied` |
+| 404 | `WALLET_ACCOUNT_NOT_FOUND` | Account not found | `https://sportsbook/errors/wallet/account-not-found` |
+| 404 | `WALLET_OPERATION_NOT_FOUND` | Wallet operation not found | `https://sportsbook/errors/wallet/operation-not-found` |
+| 404 | `WALLET_ADJUSTMENT_NOT_FOUND` | Wallet adjustment not found | `https://sportsbook/errors/wallet/adjustment-not-found` |
+| 409 | `WALLET_IDEMPOTENCY_CONFLICT` | Idempotency key conflict | `https://sportsbook/errors/wallet/idempotency-conflict` |
+| 422 | `WALLET_CURRENCY_MISMATCH` | Currency mismatch | `https://sportsbook/errors/wallet/currency-mismatch` |
+| 422 | `WALLET_INSUFFICIENT_BALANCE` | Insufficient balance | `https://sportsbook/errors/wallet/insufficient-balance` |
+| 422 | `WALLET_AMOUNT_OUT_OF_RANGE` | Amount out of range | `https://sportsbook/errors/wallet/amount-out-of-range` |
+| 423 | `WALLET_ACCOUNT_RECOVERY_BLOCKED` | Wallet account blocked for recovery | `https://sportsbook/errors/wallet/account-recovery-blocked` |
+| 500 | `WALLET_INTERNAL_ERROR` | Internal server error | `https://sportsbook/errors/wallet/internal-error` |
+| 503 | `WALLET_BUSY` | Wallet temporarily busy | `https://sportsbook/errors/wallet/busy` |
+
+`503 WALLET_BUSY` includes `Retry-After: 1`. Retryable PostgreSQL connection, availability, lock,
+timeout, deadlock, and serialization failures use this response. A transient failure can be retried
+with the same key and does not claim that key when no operation committed. Permanent database
+failures remain an opaque `500 WALLET_INTERNAL_ERROR`.
diff --git a/docs/operations.md b/docs/operations.md
new file mode 100644
index 0000000..c49ee21
--- /dev/null
+++ b/docs/operations.md
@@ -0,0 +1,162 @@
+# Operations and observability
+
+## Runtime prerequisites
+
+The service runs on Java 17. The Maven wrapper uses Maven 3.9.11, and
+`com.sportsbook:shared-protocol:1.0.0` must be installed in the local Maven repository before the
+wallet build starts.
+
+An application instance requires PostgreSQL. Flyway applies the four wallet migrations at startup
+and Hibernate validates the mapped schema. Redis is optional and best effort. Kafka is required
+when outbox delivery is enabled; publication is inactive until explicitly enabled. The complete
+integration verification path uses all three dependencies.
+
+## Environment configuration
+
+These are the complete environment mappings declared by `application.yml`. Do not place API keys
+in files or command logs. Supply only `<environment-provided secret>` values through the runtime
+secret manager.
+
+### Internal credentials
+
+| Variable | Default | Contract |
+| --- | --- | --- |
+| `WALLET_PLATFORM_API_KEY` | none | Required Platform secret. |
+| `WALLET_GATEWAY_API_KEY` | none | Required Gateway secret. |
+| `WALLET_BETTING_SERVICE_API_KEY` | none | Required Betting secret. |
+| `WALLET_SETTLEMENT_SERVICE_API_KEY` | none | Required Settlement secret. |
+| `WALLET_ADMIN_API_KEY` | none | Required Admin secret. |
+
+All five values must be distinct, nonblank, and at least 32 characters. Startup fails if any value
+is absent or violates that contract.
+
+### Database and scheduler
+
+| Variable | Default | Purpose |
+| --- | --- | --- |
+| `WALLET_SCHEDULER_POOL_SIZE` | `4` | Shared scheduled-task pool size. |
+| `WALLET_DB_URL` | `jdbc:postgresql://localhost:5432/wallet` | JDBC URL. |
+| `WALLET_DB_USER` | `wallet` | Database user. |
+| `WALLET_DB_PASSWORD` | `wallet` | Database password. Replace outside a local environment. |
+| `WALLET_DB_CONNECTION_TIMEOUT_MS` | `2000` | Hikari connection acquisition timeout in milliseconds. |
+| `WALLET_DB_LOCK_TIMEOUT` | `2s` | PostgreSQL session `lock_timeout`. |
+| `WALLET_DB_STATEMENT_TIMEOUT` | `5s` | PostgreSQL session `statement_timeout`. |
+
+The fixed pool bounds are 20 maximum connections and 5 minimum idle connections. JDBC timestamps
+use UTC, and Open Session in View is disabled.
+
+### Network dependencies and HTTP
+
+| Variable | Default | Purpose |
+| --- | --- | --- |
+| `WALLET_KAFKA_BOOTSTRAP` | `localhost:9092` | Kafka bootstrap servers. |
+| `WALLET_REDIS_HOST` | `localhost` | Redis host. |
+| `WALLET_REDIS_PORT` | `6379` | Redis port. |
+| `WALLET_HTTP_PORT` | `8081` | HTTP listener port. |
+
+Redis operations use a fixed 200 ms timeout. HTTP shutdown is graceful.
+
+### Integrity, outbox, and recovery scheduling
+
+| Variable | Default | Purpose |
+| --- | --- | --- |
+| `WALLET_INTEGRITY_ENABLED` | `true` | Enable periodic integrity scans. |
+| `WALLET_INTEGRITY_POLL_INTERVAL` | `PT30S` | Delay between scans. |
+| `WALLET_OUTBOX_ENABLED` | `false` | Enable outbox delivery and backlog sampling. |
+| `WALLET_RECOVERY_ENABLED` | `true` | Enable blocked-adjustment recovery. |
+| `WALLET_RECOVERY_POLL_INTERVAL` | `PT1S` | Delay between recovery polls. |
+| `WALLET_RECOVERY_RETRY_BASE` | `PT1S` | Initial insufficient-funds delay. |
+| `WALLET_RECOVERY_RETRY_CAP` | `PT60S` | Maximum insufficient-funds delay. |
+
+Outbox scheduling is intentionally off by default. A production deployment publishes wallet events
+only after an operator deliberately sets `WALLET_OUTBOX_ENABLED=true` and confirms the Kafka
+destination, consumer contracts, and alerting.
+
+## Advanced Spring properties
+
+These settings are Spring properties, not additional environment mappings declared by
+`application.yml`.
+
+| Property | Default | Purpose |
+| --- | --- | --- |
+| `wallet.outbox.owner` | `${HOSTNAME:wallet-service}-${random.uuid}` | Nonblank process identity, at most 128 characters, used for lease fencing. |
+| `wallet.outbox.poll-interval` | `PT1S` | Delay between claims. |
+| `wallet.outbox.batch-size` | `20` | Maximum rows per claim. |
+| `wallet.outbox.max-in-flight` | `100` | Process-wide asynchronous send bound. |
+| `wallet.outbox.lease-duration` | `PT30S` | Claim lease duration. |
+| `wallet.outbox.retry-base` | `PT1S` | Initial delivery retry delay. |
+| `wallet.outbox.retry-cap` | `PT60S` | Maximum delivery retry delay. |
+| `wallet.outbox.metrics-interval` | `PT5S` | Backlog gauge sampling delay. |
+
+The publisher requires the lease duration to exceed Kafka completion bounds plus its safety margin.
+Kafka producer settings are `acks=all`, idempotence enabled, at most 5 in-flight requests per
+connection, a 5 second delivery timeout, a 5 second maximum blocking time, and a 4 second request
+timeout. Delivery retries have no attempt limit and use the configured capped exponential delay.
+
+Recovery uses a 5 second transaction timeout. An insufficient-funds attempt changes only proof
+retry metadata and keeps the account, ledger, and operation state untouched.
+
+## Health endpoints
+
+The management base path is `/actuator`.
+
+- Anonymous GET: `/actuator/health`, `/actuator/health/**`, `/actuator/prometheus`.
+- Platform-authenticated access: `/actuator`, `/actuator/info`, `/actuator/metrics`, and every other
+  management route.
+
+Health details are shown only to an authorized caller. The `walletIntegrityHealth` component
+behaves as follows:
+
+| State | Meaning |
+| --- | --- |
+| `UNKNOWN` | No scan has completed; detail `reason=integrity_not_checked`. |
+| `DOWN` | The scan failed; detail `reason=integrity_scan_failed`. |
+| `DOWN` | A completed scan found one or more drift facts. |
+| `UP` | The latest completed scan found zero drift facts. |
+
+A completed scan includes `lastCheckedAt` and `driftCount` in the component details.
+
+## Metrics
+
+Micrometer registers these outbox meters:
+
+- `wallet.outbox.claimed`
+- `wallet.outbox.published`
+- `wallet.outbox.retried`
+- `wallet.outbox.fenced.completion`
+- `wallet.outbox.lease.takeovers`
+- `wallet.outbox.pending`
+- `wallet.outbox.leased`
+- `wallet.outbox.oldest.pending.seconds`
+
+Integrity meters are:
+
+- `wallet.integrity.account.snapshot.drift`
+- `wallet.integrity.account.orphan.ledgers`
+- `wallet.integrity.operation.group.drift`
+- `wallet.integrity.recovery.queue.drift`
+- `wallet.integrity.adjustment.outcome.drift`
+- `wallet.integrity.adjustment.failure.drift`
+- `wallet.integrity.adjustment.fingerprint.drift`
+- `wallet.integrity.adjustment.ledger.drift`
+- `wallet.integrity.total.drift`
+- `wallet.integrity.scan.failed`
+- `wallet.integrity.last.checked.epoch.seconds`
+
+Prometheus renders the dotted Micrometer IDs with underscores; counter names also receive the
+`_total` suffix, while gauge names do not. The integrity gauges describe the latest completed
+repeatable-read scan; a scrape does not query PostgreSQL.
+
+## Verification
+
+Both verification commands require Docker:
+
+```bash
+./mvnw clean verify
+./mvnw -Psemantic-gates clean verify
+```
+
+The default command runs the complete suite. The semantic profile selects the tagged wallet,
+recovery, outbox, security, and live HTTP checks. The container-backed checks use PostgreSQL 16,
+Redis 7, and Kafka 3.8 and exercise schema migrations, database concurrency, broker delivery,
+recovery, and idempotent HTTP behavior.
