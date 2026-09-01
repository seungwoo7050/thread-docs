## `docs(project): document risk 1.0 contracts`

diff --git a/README.md b/README.md
index dd16194..1f718d6 100644
--- a/README.md
+++ b/README.md
@@ -1,13 +1,69 @@
 # Risk Service
 
-Risk Service owns sportsbook admission policy. It evaluates user limits and suspicious activity,
-then reserves capacity before a bet can debit funds.
+Risk Service owns atomic sportsbook admission, short-lived capacity reservations, user limit
+overrides, and accepted-bet reconciliation. It is a Java 17 Spring Boot service published as
+`com.sportsbook:risk-service:1.0.0` and uses `com.sportsbook:shared-protocol:1.0.0` for shared value
+objects, errors, and Avro records.
 
-The service is intentionally small at its boundary:
+Redis is the authoritative runtime state for admission and reservation decisions. Kafka supplies
+accepted-bet facts and carries advisory risk signals. The service does not keep a relational
+database or a transactional Kafka outbox.
 
-- Redis is the authoritative store for limits, reservations, and recent risk history.
-- Kafka carries accepted bet facts and non-authoritative risk signals.
-- Internal HTTP APIs expose reservation lifecycle, limit administration, and diagnostics.
-- Shared Protocol supplies the value objects and Avro records exchanged with other services.
+## What the service owns
 
-The implementation targets Java 17 and runs as a Spring Boot service.
+- Atomic admission against the configured single-bet, daily, weekly, monthly, and per-minute
+  selection limits.
+- Suspicious-pattern evaluation for rapid betting, sudden stake increases, and repeated
+  selections.
+- Idempotent reservation, commit, and release operations keyed by `betId` and an opaque request
+  fingerprint.
+- Per-user limit overrides managed by `admin-api`.
+- Reconciliation of `BetPlacedRequested` records from `bet.placed.v1`, including first-seen
+  projection when an accepted event arrives without a retained reservation.
+
+Risk signals from the diagnostic evaluator are best effort. Reservation admission returns its
+flags and rejections directly but does not publish those decisions as Kafka signals.
+
+## Runtime dependencies
+
+- Temurin-compatible JDK 17
+- Standalone Redis 7
+- Kafka broker reachable through `KAFKA_BOOTSTRAP`
+
+The Lua operations intentionally span bet-scoped and user-scoped keys, so Redis Cluster is not a
+supported topology. Redis availability and persistence policy are deployment responsibilities;
+the service itself does not add a second state store.
+
+## Build and run
+
+The build requires the `shared-protocol:1.0.0` artifact in the configured Maven repository.
+
+```bash
+./mvnw clean verify
+```
+
+Set three distinct internal secrets of at least 32 characters, then start the executable JAR:
+
+```bash
+export INTERNAL_BETTING_SERVICE_API_KEY='replace-with-a-distinct-secret-at-least-32-characters'
+export INTERNAL_ADMIN_API_KEY='replace-with-another-distinct-secret-at-least-32-characters'
+export INTERNAL_PLATFORM_API_KEY='replace-with-a-third-distinct-secret-at-least-32-characters'
+export REDIS_HOST=localhost
+export REDIS_PORT=6379
+export KAFKA_BOOTSTRAP=localhost:9092
+java -jar target/risk-service-1.0.0.jar
+```
+
+The default HTTP port is `8083`. `/actuator/health`, its probe paths, and
+`/actuator/prometheus` are anonymous; every internal API requires caller-specific credentials.
+
+## Contracts and operations
+
+- [Runtime and consistency boundaries](architecture/runtime-and-consistency-boundaries.md)
+- [Redis keyspace and reservation lifecycle](architecture/redis-keyspace-and-reservation-lifecycle.md)
+- [Internal API and event contracts](architecture/internal-api-and-event-contracts.md)
+- [Operations](docs/operations.md)
+
+Configuration defaults live in
+[`src/main/resources/application.yml`](src/main/resources/application.yml). Credentials must be
+provided through environment variables and must not be stored in configuration files.
diff --git a/architecture/internal-api-and-event-contracts.md b/architecture/internal-api-and-event-contracts.md
new file mode 100644
index 0000000..1eeafd9
--- /dev/null
+++ b/architecture/internal-api-and-event-contracts.md
@@ -0,0 +1,157 @@
+# Internal API and event contracts
+
+All internal requests use JSON, with identifiers represented as UUID strings. Kafka identifiers
+must be lowercase canonical UUID strings. Monetary amounts are positive integer `Money.amount`
+values paired with `KRW` or `USD`; the service does not perform decimal or exchange-rate
+conversion.
+
+## Authentication and ownership
+
+Every internal request supplies both headers:
+
+```text
+X-Internal-Service: <caller>
+X-Internal-Api-Key: <caller-specific secret>
+```
+
+| Caller | Owned operations |
+| --- | --- |
+| `betting-service` | Reservation admission, commit, and release |
+| `admin-api` | Read, set, and clear user limit overrides |
+| `platform` | Non-reserving risk check and any additionally exposed actuator endpoint |
+
+Secrets come from `INTERNAL_BETTING_SERVICE_API_KEY`, `INTERNAL_ADMIN_API_KEY`, and
+`INTERNAL_PLATFORM_API_KEY`. Startup fails when a secret is missing, blank, shorter than 32
+characters, or equal to another caller's secret. The application keeps SHA-256 digests and uses a
+constant-time comparison.
+
+Missing or invalid credentials return `401`. A valid caller using another caller's route returns
+`403`. Health endpoints and Prometheus are anonymous.
+
+## Endpoint inventory
+
+| Method and path | Caller | Success |
+| --- | --- | --- |
+| `POST /internal/v1/risk/reservations` | `betting-service` | `200` with an approved lease or retained rejection |
+| `PUT /internal/v1/risk/reservations/{betId}/commit` | `betting-service` | `204` for applied or matching replay |
+| `DELETE /internal/v1/risk/reservations/{betId}` | `betting-service` | `204` for applied, matching replay, missing, or tombstoned state |
+| `GET /internal/v1/risk/limits/{userId}` | `admin-api` | `200` with every effective limit |
+| `PATCH /internal/v1/risk/limits/{userId}` | `admin-api` | `204` after replacing one override |
+| `DELETE /internal/v1/risk/limits/{userId}/{type}` | `admin-api` | `204` after clearing one override |
+| `POST /internal/v1/risk/check` | `platform` | `200` with a point-in-time diagnostic decision |
+
+Request validation failures use the shared `ProblemDetail` JSON shape with
+`errorCode=VALIDATION_FAILED`. Conflicting reuse of a reservation `betId` uses `DUPLICATE_BET` and
+`409`. Commit of missing or tombstoned state uses `RISK_RESERVATION_NOT_FOUND` and `404`; release of
+an already committed reservation uses `RISK_RESERVATION_COMMITTED` and `409`. Unhandled failures
+are rendered as an opaque `INTERNAL_ERROR` without exposing the exception.
+
+## Candidate and reservation payloads
+
+Reservation and diagnostic check use the same request body:
+
+```json
+{
+  "userId": "10000000-0000-4000-8000-000000000001",
+  "betId": "20000000-0000-4000-8000-000000000001",
+  "stake": {"amount": 1000, "currency": "KRW"},
+  "selectionIds": ["30000000-0000-4000-8000-000000000001"]
+}
+```
+
+There must be one to 15 unique selection IDs. `stake.amount` must be in
+`1..9007199254740991`.
+
+An approved reservation returns its expiry and opaque token:
+
+```json
+{
+  "approved": true,
+  "replayed": false,
+  "patterns": [],
+  "reservationState": "RESERVED",
+  "expiresAt": "2026-08-21T10:02:00Z",
+  "reservationToken": "0000000000000000000000000000000000000000000000000000000000000000"
+}
+```
+
+A policy rejection is also a `200` application result. It has `approved=false`, a
+`rejectionReason`, any pattern flags, and no reservation token. Repeating the same request returns
+the retained result with `replayed=true` while its lifecycle exists.
+
+Commit requires the returned token:
+
+```http
+PUT /internal/v1/risk/reservations/{betId}/commit
+X-Risk-Reservation-Token: <reservationToken>
+```
+
+The diagnostic response contains `approved`, optional `rejectionReason`, optional limit details,
+and `patterns`. It does not create a lease and must not replace reservation admission.
+
+## Limit administration
+
+`GET /internal/v1/risk/limits/{userId}` returns seven entries: daily, weekly, and monthly stake
+limits for both currencies, plus the currency-neutral selection limit. Each entry identifies its
+`POLICY` or `OVERRIDE` source.
+
+Set one monetary override:
+
+```json
+{"type":"STAKE_DAILY","currency":"KRW","value":750000}
+```
+
+Set the selection override by omitting currency:
+
+```json
+{"type":"SELECTIONS_PER_MINUTE","value":20}
+```
+
+Values may be zero and must not exceed `9007199254740991`. Clearing a monetary override requires
+the matching query parameter, for example:
+
+```http
+DELETE /internal/v1/risk/limits/{userId}/STAKE_DAILY?currency=KRW
+```
+
+Clearing `SELECTIONS_PER_MINUTE` requires currency to be omitted.
+
+## Accepted-bet input
+
+| Property | Contract |
+| --- | --- |
+| Topic | `bet.placed.v1` by default |
+| Kafka key | Canonical `userId` string |
+| Value | Plain Avro binary `com.sportsbook.protocol.event.BetPlacedRequested` with no registry framing |
+| Consumer group | `risk.bet-placed-consumer` by default |
+
+The consumer validates canonical IDs, a shared-protocol idempotency key of at most 128 printable
+ASCII characters, selection identity and odds, positive exactly representable exposure, and a
+selection count from one to 15. The key must not be blank and selection IDs must be unique.
+
+- `SINGLE` has exactly one selection and no system fields.
+- `MULTIPLE` has at least two selections and no system fields; exposure is the event stake.
+- `SYSTEM` supplies matching `systemTotalSelections` and a valid `systemMinWins`; exposure is the
+  event stake multiplied by `C(totalSelections, systemMinWins)`.
+
+The consumer observation time is used for rolling windows. `requestedAt` is validated and carried
+through decoding but is not used as the Redis score.
+
+Permanent failures go to `bet.placed.v1.DLT` with the original key and payload and an ASCII
+`risk-dlt-reason` header. Values are `MALFORMED_EVENT`, `KEY_MISMATCH`, `FINGERPRINT_MISMATCH`, or
+`TERMINAL_RESERVATION`.
+
+## Risk signal output
+
+| Topic | Kafka key | Plain Avro value | Delivery role |
+| --- | --- | --- | --- |
+| `risk.limit.violated` | `userId` | `RiskLimitViolated` | Best-effort diagnostic signal for daily stake or per-minute selection violations |
+| `risk.pattern.suspected` | `userId` | `RiskPatternSuspected` | Best-effort diagnostic signal for rapid betting, sudden stake increase, or repeated selection |
+
+Signals are emitted by `POST /internal/v1/risk/check`; reservation admission does not publish its
+decision. Single-bet, weekly, and monthly diagnostic rejections have no corresponding shared
+risk-limit signal in this service. Signal topics and input/DLT topics are configurable under
+`risk.topics`.
+
+See [Runtime and consistency boundaries](runtime-and-consistency-boundaries.md) for acknowledgment
+and retry behavior.
diff --git a/architecture/redis-keyspace-and-reservation-lifecycle.md b/architecture/redis-keyspace-and-reservation-lifecycle.md
new file mode 100644
index 0000000..17cbe2f
--- /dev/null
+++ b/architecture/redis-keyspace-and-reservation-lifecycle.md
@@ -0,0 +1,89 @@
+# Redis keyspace and reservation lifecycle
+
+All UUID components are lowercase canonical UUID strings. Currency suffixes in keys are lowercase.
+Amounts and aggregate counts are base-10 integers limited to Redis Lua's exact range,
+`0..9007199254740991`; candidate stakes must be positive.
+
+## Key inventory
+
+| Key pattern | Type | Contents and lifetime |
+| --- | --- | --- |
+| `risk:reservation:<betId>` | hash | Request fingerprint, identity, stake, selections, pattern result, lifecycle state, and timestamps. Its TTL is the configured reservation retention. |
+| `risk:reservations:active` | string | Global count of active reservations. It is deleted when the count reaches zero. |
+| `risk:reservations:user:{<userId>}:bets` | sorted set | Active `betId` members scored by reservation time. |
+| `risk:reservations:user:{<userId>}:stakes:<currency>:entries` | sorted set | Active `<betId>|<amount>` members. |
+| `risk:reservations:user:{<userId>}:stakes:<currency>:sum` | string | Exact aggregate paired with the active stake entries. |
+| `risk:reservations:user:{<userId>}:selections:entries` | sorted set | Active `<betId>|<selectionCount>` members. |
+| `risk:reservations:user:{<userId>}:selections:sum` | string | Exact aggregate paired with the active selection entries. |
+| `risk:reservations:user:{<userId>}:selection:<selectionId>` | sorted set | Active bet IDs containing one selection. |
+| `risk:limit:{<userId>}:<dimension>:entries` | sorted set | Committed `<betId>|<amount>` or `<betId>|<selectionCount>` members scored by commit time. |
+| `risk:limit:{<userId>}:<dimension>:sum` | string | Exact aggregate paired with one committed window. |
+| `risk:limit:override:{<userId>}` | hash | Administrative limits. Fields are `<LIMIT_TYPE>:<CURRENCY>` or `SELECTIONS_PER_MINUTE`; they have no service-assigned TTL. |
+| `risk:history:{<userId>}:bets` | sorted set | Confirmed bet IDs used by rapid-betting evaluation. |
+| `risk:history:{<userId>}:stakes:<currency>` | sorted set | Bounded confirmed `<betId>|<amount>` samples for sudden-stake evaluation. |
+| `risk:history:{<userId>}:selection:<selectionId>` | sorted set | Confirmed bet IDs used by repeated-selection evaluation. |
+| `risk:event:fingerprint:<betId>` | string | Fingerprint of a first-seen accepted event. Its TTL is the reservation retention. |
+
+Committed dimensions are `stake-daily:<currency>`, `stake-weekly:<currency>`,
+`stake-monthly:<currency>`, and `selections-per-minute`. Their windows are one day, seven days,
+30 days, and one minute respectively. Non-empty committed keys receive a TTL equal to the window
+plus five minutes. Confirmed pattern keys receive the configured idle retention, seven days by
+default; stake samples are also capped at 100 by default.
+
+Active aggregate keys do not have independent TTLs. Expired footprints are cleaned while a script
+examines that user's active set, and the lifecycle hash remains the evidence needed to perform the
+cleanup. Operational Redis eviction must therefore be disabled for these keys.
+
+## Fingerprint and token
+
+The reservation token is the lowercase SHA-256 fingerprint of a canonical request. Version
+`risk-reservation-v1`, user ID, bet ID, stake amount, currency, and sorted selection IDs are fed to
+the digest as length-prefixed UTF-8 fields. Selection order therefore does not change the token,
+while any semantic request change does.
+
+The token is an idempotency binding, not a caller credential. It is returned only for approved
+reservations and is required in `X-Risk-Reservation-Token` for commit. Authentication still uses
+the caller-specific internal headers.
+
+## Lifecycle
+
+The configured lease is two minutes and retention is 32 days by default.
+
+```text
+absent ── reserve approved ──> RESERVED ── commit with token ──> COMMITTED
+   │                              │
+   └─ reserve rejected ───────> REJECTED
+                                  │
+RESERVED ── release ──────────> RELEASED
+RESERVED ── lazy expiry ──────> EXPIRED
+```
+
+- Repeating reserve with the same fingerprint returns the retained result with `replayed=true`.
+- Reusing a retained `betId` with a different fingerprint returns a conflict.
+- Repeating commit on `COMMITTED` succeeds as a replay. Commit on a missing, expired, released, or
+  rejected lifecycle is reported as not found at the HTTP boundary.
+- Repeating release on `RELEASED` succeeds as a replay. Release of missing or tombstoned state is
+  also an idempotent HTTP success; release of `COMMITTED` is a conflict.
+- After the lifecycle TTL expires, the service no longer has a reservation replay record for that
+  `betId`.
+
+An approved reserve creates active capacity. Commit moves the full stake and full selection count
+from active aggregates into all applicable committed windows and confirmed pattern facts. Release
+or expiry removes the active capacity without adding committed facts. No partial stake or partial
+selection reservation is supported.
+
+## Pattern state
+
+Admission evaluates confirmed facts together with unexpired active reservations, so concurrent
+candidates cannot all ignore one another. The candidate receives zero or more flags:
+
+- `SUSPECT` and `REVIEW` are advisory and do not by themselves reject admission.
+- `BLOCK` rejects admission and stores the rejection for bounded replay.
+
+If an already-accepted Kafka fact has no retained lifecycle, accepted projection adds its full
+exposure directly to committed counters and confirmed pattern facts. It does not create an active
+reservation or retroactively reject the accepted bet.
+
+See [Internal API and event contracts](internal-api-and-event-contracts.md) for transition status
+mapping and [Runtime and consistency boundaries](runtime-and-consistency-boundaries.md) for the
+supported Redis topology.
diff --git a/architecture/runtime-and-consistency-boundaries.md b/architecture/runtime-and-consistency-boundaries.md
new file mode 100644
index 0000000..f0613dc
--- /dev/null
+++ b/architecture/runtime-and-consistency-boundaries.md
@@ -0,0 +1,85 @@
+# Runtime and consistency boundaries
+
+Risk Service has two state-changing boundaries: standalone Redis Lua scripts and Kafka consumer
+offset acknowledgment. Keeping those boundaries explicit is necessary when evaluating failure
+behavior.
+
+## Admission and reservation
+
+`POST /internal/v1/risk/reservations` runs one `risk-reserve.lua` invocation. The script validates
+the relevant key types and aggregates, removes expired active footprints for the user, evaluates
+configured limits and pattern rules, and either records a rejection or creates a reservation with
+all active capacity footprints. There is no interval in which the reservation exists without its
+active stake and selection totals.
+
+The authoritative admission calculation includes both committed rolling counters and unexpired
+active reservations. The standalone diagnostic endpoint reads the same categories of state but
+does not reserve capacity; its result can therefore become stale immediately and must not be used
+as authorization to debit funds.
+
+Commit and release are separate atomic Lua operations:
+
+- Commit validates the opaque token, removes the active footprints, adds the stake and selection
+  counts to rolling committed windows, records confirmed pattern facts, and marks the lifecycle
+  `COMMITTED`.
+- Release removes active footprints and marks the lifecycle `RELEASED`.
+- Expired reservations are removed lazily by reservation and snapshot scripts. Their lifecycle is
+  retained as `EXPIRED` for replay behavior.
+
+Scripts validate the keys and aggregates required for each transition and return an error on an
+inconsistency. Expired-footprint cleanup is a deliberate side effect of admission and snapshot
+reads and may occur even when the candidate is rejected. Script errors fail the HTTP request or
+Kafka reconciliation; there is no fallback that approves a candidate without Redis.
+
+## Redis topology and persistence
+
+Reservation scripts touch `risk:reservation:<betId>` together with keys tagged by `userId`.
+Because these keys are intentionally in different hash slots, the supported deployment is a
+single Redis server, not Redis Cluster. The application configures a two-second connect timeout
+and a two-second command timeout.
+
+The guarantees above come from atomic Redis script execution. Survival across host loss depends on
+the operator's Redis persistence, replication, and restore configuration. Risk Service has no
+database journal and does not rebuild Redis state from Kafka.
+
+## Accepted-bet reconciliation
+
+The `risk.bet-placed-consumer` group reads `BetPlacedRequested` records from `bet.placed.v1` with
+manual immediate acknowledgment.
+
+1. The consumer validates Avro, identifiers, Kafka key, slip shape, exposure, and idempotency key.
+2. It tries to commit a matching reservation using the canonical reservation fingerprint.
+3. If no lifecycle exists, it atomically projects the accepted bet into committed counters and
+   pattern facts, guarded by `risk:event:fingerprint:<betId>`.
+4. It acknowledges the source record only after reconciliation succeeds.
+
+The accepted-event fingerprint marker makes matching redelivery a replay while the marker is
+retained. A different fingerprint for the same `betId` is a permanent failure. The marker uses the
+reservation retention period, so this is a bounded idempotency window rather than an unbounded
+ledger.
+
+Unhandled Redis, Kafka, or application failures remain unacknowledged and are retried every second
+without an attempt limit. A repeatedly failing record can therefore hold its source partition.
+
+Malformed input, key mismatch, fingerprint mismatch, and terminal reservation state are sent to
+`bet.placed.v1.DLT`. The publisher waits up to ten seconds for the broker acknowledgment before the
+source offset is acknowledged. A process failure between those two acknowledgments can produce a
+duplicate DLT record; downstream DLT handling must tolerate duplicates.
+
+## Advisory risk signals
+
+The diagnostic evaluator submits limit and pattern signals asynchronously to Kafka. Submission or
+delivery failures are counted and logged, but they do not reverse the diagnostic result and are
+not retried by an outbox. Reservation admission returns flags to its caller and does not publish
+signals. These topics are observational signals, not a state-transfer mechanism.
+
+## Availability boundary
+
+The readiness group contains Spring's `readinessState`, Redis health, and a Kafka metadata query.
+The Kafka indicator waits at most two seconds for the cluster ID. Readiness reports `DOWN` when
+either dependency check fails, but an already-started process can still receive traffic unless the
+runtime removes it from service.
+
+See [Operations](../docs/operations.md) for probe and metric endpoints and
+[Redis keyspace and reservation lifecycle](redis-keyspace-and-reservation-lifecycle.md) for the
+retained state model.
diff --git a/docs/operations.md b/docs/operations.md
new file mode 100644
index 0000000..467a297
--- /dev/null
+++ b/docs/operations.md
@@ -0,0 +1,131 @@
+# Operations
+
+## Required configuration
+
+| Environment variable | Default | Purpose |
+| --- | --- | --- |
+| `INTERNAL_BETTING_SERVICE_API_KEY` | none | Required `betting-service` credential, at least 32 characters |
+| `INTERNAL_ADMIN_API_KEY` | none | Required `admin-api` credential, at least 32 characters |
+| `INTERNAL_PLATFORM_API_KEY` | none | Required `platform` credential, at least 32 characters |
+| `REDIS_HOST` | `localhost` | Standalone Redis host |
+| `REDIS_PORT` | `6379` | Standalone Redis port |
+| `KAFKA_BOOTSTRAP` | `localhost:9092` | Kafka bootstrap servers |
+| `SERVER_PORT` | `8083` | HTTP port |
+| `OTEL_TRACES_SAMPLER_ARG` | `0.1` | OpenTelemetry sampling probability |
+
+The three secrets must be distinct. The service refuses to start when credential validation
+fails. Do not pass secrets as command-line arguments or place them in tracked configuration.
+
+Policy, retention, consumer group, and topic defaults are in
+[`application.yml`](../src/main/resources/application.yml). Reservation retention must exceed both
+the lease and the 30-day monthly counter window.
+
+## Build
+
+Use JDK 17 and the checked-in Maven 3.9.11 wrapper. The Maven repository must already contain
+`com.sportsbook:shared-protocol:1.0.0`.
+
+```bash
+java -version
+./mvnw -version
+./mvnw -B clean verify
+```
+
+`verify` runs unit and integration tests, Spotless, and Checkstyle, then builds the executable
+Spring Boot JAR.
+
+## Start and stop
+
+Supply secrets through the runtime's secret mechanism. A local shell can start the packaged
+service as follows:
+
+```bash
+export INTERNAL_BETTING_SERVICE_API_KEY='replace-with-a-distinct-secret-at-least-32-characters'
+export INTERNAL_ADMIN_API_KEY='replace-with-another-distinct-secret-at-least-32-characters'
+export INTERNAL_PLATFORM_API_KEY='replace-with-a-third-distinct-secret-at-least-32-characters'
+export REDIS_HOST=localhost REDIS_PORT=6379
+export KAFKA_BOOTSTRAP=localhost:9092 SERVER_PORT=8083
+java -jar target/risk-service-1.0.0.jar
+```
+
+Use the process supervisor's normal termination signal and readiness draining. The service has no
+local state volume to flush, but Redis and Kafka must be operated according to their own shutdown
+procedures.
+
+## Health and readiness
+
+```bash
+curl -fsS http://localhost:8083/actuator/health
+curl -fsS http://localhost:8083/actuator/health/liveness
+curl -fsS http://localhost:8083/actuator/health/readiness
+```
+
+The readiness response is `UP` only when application readiness, Redis, and Kafka metadata checks
+are up. The Kafka check has a two-second budget. Health details are intentionally hidden.
+
+Health, probe paths, and Prometheus are anonymous so infrastructure can scrape them. Internal API
+credentials must not be sent to those endpoints.
+
+## Metrics
+
+Prometheus metrics are exposed at:
+
+```bash
+curl -fsS http://localhost:8083/actuator/prometheus
+```
+
+Application meter IDs and bounded tags are:
+
+| Meter ID | Tags | Meaning |
+| --- | --- | --- |
+| `risk.check.latency` | none | Diagnostic check duration |
+| `risk.limit.violations` | `reason` | Diagnostic limit rejection count |
+| `risk.pattern.flags` | `rule`, `action` | Diagnostic pattern match count |
+| `risk.signal.delivery` | `outcome` | Best-effort signal delivery callback result |
+| `risk.reservation.lua.latency` | `operation` | Redis script duration for reserve, commit, accepted projection, or release |
+| `risk.reservation.expirations` | none | Expired reservations cleaned during admission |
+| `risk.reservation.requests` | `result` | Created, rejected, conflict, or replay admission result |
+| `risk.reservation.transitions` | `operation`, `result` | Commit and release result count |
+| `risk.bet.placed.reconciliation` | `result` | Accepted-event reconciliation result count |
+| `risk_bet_placed_dlt_total` | `reason` | Permanent accepted-event failures sent to DLT |
+
+Micrometer converts dotted meter IDs to Prometheus snake-case names and adds the normal counter or
+timer suffixes. No meter uses user, bet, selection, or reservation token as a tag.
+
+Operationally significant conditions include readiness failure, a rising failed signal count, DLT
+records, fingerprint or terminal reconciliation results, persistent consumer lag, and reservation
+script latency. Alert thresholds belong to the deployment's traffic and latency objectives.
+
+## Correctness gate
+
+The release gate requires Docker, `curl`, `jq`, JDK 17, a Maven repository populated with the
+shared protocol artifact, and three distinct test secrets. It starts isolated Redis 7 and Kafka,
+builds the service, waits for readiness, checks concurrent admission and lifecycle replay, and
+asserts that core reservation metrics are present.
+
+```bash
+export RISK_MAVEN_REPO=/absolute/path/to/isolated-maven-repository
+export INTERNAL_BETTING_SERVICE_API_KEY='gate-betting-secret-at-least-32-characters'
+export INTERNAL_ADMIN_API_KEY='gate-admin-secret-at-least-32-characters'
+export INTERNAL_PLATFORM_API_KEY='gate-platform-secret-at-least-32-characters'
+bash load-test/run-gate.sh
+```
+
+The runner uses a temporary output directory and removes its containers and volume on exit. Gate
+responses, metrics, and service logs are diagnostic output and are not project artifacts.
+
+## Incident boundaries
+
+- Redis failure makes admission, reservation lifecycle, limit administration, and accepted-event
+  reconciliation unavailable. Do not bypass the service with a default approval.
+- Kafka input reconciliation retries unhandled failures without a limit. Inspect the blocking
+  source record and dependency health before resetting offsets.
+- Advisory risk signal failure does not roll back an admission result. Use the delivery metrics
+  and logs to assess missing notifications.
+- Deleting or evicting lifecycle and active aggregate keys can break replay and consistency checks.
+  Restore Redis as a coherent dataset; do not repair individual aggregate keys while traffic is
+  active.
+
+The detailed state and event semantics are documented in
+[Redis keyspace and reservation lifecycle](../architecture/redis-keyspace-and-reservation-lifecycle.md)
+and [Runtime and consistency boundaries](../architecture/runtime-and-consistency-boundaries.md).
