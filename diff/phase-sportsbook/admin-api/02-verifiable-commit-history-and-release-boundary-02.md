## `chore(release): release admin API 1.0.0`

diff --git a/pom.xml b/pom.xml
index 5f8c370..7330e8d 100644
--- a/pom.xml
+++ b/pom.xml
@@ -19,7 +19,7 @@
 
     <groupId>com.sportsbook</groupId>
     <artifactId>admin-api</artifactId>
-    <version>0.1.0-SNAPSHOT</version>
+    <version>1.0.0</version>
     <packaging>jar</packaging>
 
     <name>admin-api</name>


## `docs(project): document admin API contracts`

diff --git a/README.md b/README.md
index 3d0bc0c..19db53e 100644
--- a/README.md
+++ b/README.md
@@ -1,4 +1,225 @@
 # Sportsbook Admin API
 
-Operator-facing control-plane API for the sportsbook archive project.
+Operator-facing control-plane API for sportsbook services. The service authenticates operators,
+enforces network and role policy, delegates commands through isolated service credentials, and
+records mutation outcomes in a fail-closed PostgreSQL audit trail.
 
+## Runtime contract
+
+- Java 17 / class-file major version 61
+- Spring Boot 3.2.11
+- PostgreSQL 16
+- Kafka with raw Avro audit events
+- `com.sportsbook:shared-protocol:1.0.0`
+- Default HTTP port: `8090`
+
+The archive build fixes `shared-protocol` to commit
+`f9de6bc1e533761ab4bb1454d8d4ab8175cdf001`.
+
+## Authentication and network policy
+
+Every `/admin/**` request must originate from `ADMIN_IP_ALLOWLIST` and carry a verified bearer
+JWT.
+
+JWT requirements:
+
+- RS256 only
+- SPKI RSA public key of at least 2048 bits
+- required, currently valid `exp` claim with zero clock skew
+- valid `nbf` when present
+- nonblank `sub`, at most 128 code points, with no surrounding whitespace or control characters
+- exact string `role`: `ADMIN`, `TRADER`, `CS`, or `READONLY`
+- exact issuer match when `ADMIN_JWT_ISSUER` is configured
+
+The service is stateless and does not create HTTP sessions. `X-Forwarded-For` is honored only when
+the direct peer belongs to `ADMIN_TRUSTED_PROXY_CIDRS`; malformed or ambiguous proxy chains fail
+closed.
+
+For each authenticated `POST`, `PATCH`, or `DELETE` under `/admin/v1/`, the service generates one
+UUIDv7 action identity and returns it as `X-Admin-Action-Id`, including request or authorization
+failures reached after authentication.
+
+## API
+
+All paths require the authentication and IP policy above.
+
+| Method and path | Roles | Contract |
+| --- | --- | --- |
+| `POST /admin/v1/wallet/{userId}/refund` | `ADMIN`, `CS` | Requires exactly one `Idempotency-Key`. Body: `{amount,currency,reason}` with positive amount and a 1–256 character reason. Returns `200` with `{operationGroupId,actionId}`. |
+| `GET /admin/v1/risk/users/{userId}/limits` | all roles | Returns the complete typed limit snapshot. |
+| `PATCH /admin/v1/risk/users/{userId}/limits` | `ADMIN`, `TRADER` | Body: `{type,currency,value}`. Returns empty `204`. |
+| `DELETE /admin/v1/risk/users/{userId}/limits/{type}?currency=` | `ADMIN`, `TRADER` | Currency is required for stake limits and forbidden for `SELECTIONS_PER_MINUTE`. Returns empty `204`. |
+| `POST /admin/v1/events/{eventId}/markets/{marketId}/suspend` | `ADMIN`, `TRADER` | Requires `Idempotency-Key`; body `{reason}` with 1–256 characters. Returns empty `202`. |
+| `POST /admin/v1/events/{eventId}/markets/{marketId}/close` | `ADMIN`, `TRADER` | Same contract as suspend. |
+| `POST /admin/v1/events/{eventId}/markets/{marketId}/reopen` | `ADMIN`, `TRADER` | Same contract as suspend. |
+| `GET /admin/v1/settlements/result-candidates/{candidateId}` | all roles | Returns typed candidate evidence. |
+| `GET /admin/v1/settlements/revisions/{revisionId}` | all roles | Returns typed revision and wallet evidence. |
+| `POST /admin/v1/settlements/result-candidates/{candidateId}/approve` | `ADMIN`, `TRADER` | Requires a UUID `Idempotency-Key`. Returns a verified receipt with `200`. |
+| `POST /admin/v1/settlements/result-candidates/{candidateId}/reject` | `ADMIN`, `TRADER` | Requires a UUID `Idempotency-Key`; body `{reason}` with 1–256 printable characters. Returns a verified receipt with `200`. |
+| `POST /admin/v1/settlements/revisions/{revisionId}/retry` | `ADMIN`, `TRADER` | Requires a UUID `Idempotency-Key`. Returns a verified retry receipt with `202`. |
+| `GET /admin/v1/audit-logs` | all roles | Filters by optional ISO-8601 `from`, `to`, and `actor`; supports `page` and `size`, capped at 200. |
+| `GET /admin/v1/audit-logs/{actionId}` | all roles | Returns the exact audit action or `404`. |
+
+Risk limit types are `STAKE_DAILY`, `STAKE_WEEKLY`, `STAKE_MONTHLY`, and
+`SELECTIONS_PER_MINUTE`. Stake limits require `KRW` or `USD`; the selection-rate limit has no
+currency. Risk values must be nonnegative and no greater than JavaScript's maximum safe integer.
+
+There are no settlement void or replay endpoints.
+
+## Downstream isolation
+
+The four caller keys are mandatory, at least 32 characters, distinct within this service, and
+injected only into their dedicated `RestClient`.
+
+| Service | Base URL | Caller key | Authentication headers |
+| --- | --- | --- | --- |
+| Wallet | `ADMIN_WALLET_BASE_URL` | `ADMIN_WALLET_API_KEY` | `X-Internal-Service: admin-api`, `X-Internal-Api-Key` |
+| Risk | `ADMIN_RISK_BASE_URL` | `ADMIN_RISK_API_KEY` | `X-Internal-Service: admin-api`, `X-Internal-Api-Key` |
+| Odds | `ADMIN_ODDS_FEED_BASE_URL` | `ADMIN_ODDS_FEED_API_KEY` | `X-Internal-Service: admin-api`, `X-Internal-Api-Key` |
+| Settlement | `ADMIN_SETTLEMENT_BASE_URL` | `ADMIN_SETTLEMENT_API_KEY` | `X-Service-Name: admin-api`, `X-API-Key` |
+
+Wallet refunds call `POST /internal/v1/wallet/transactions/credit`. The external idempotency key is
+forwarded unchanged, while the internal body fixes `source=HOUSE_POOL` and `reason=REFUND`. A
+successful response is accepted only when its operation group, user, amount and currency, ledger
+reason, and timestamp form complete matching proof.
+
+Risk commands call the exact `/internal/v1/risk/limits/{userId}` resource. Odds commands forward
+the idempotency key and generated `X-Admin-Action-Id`. Settlement mutation keys must be UUIDs and
+are forwarded unchanged.
+
+Downstream `4xx` responses are relayed. Timeouts become `504 GATEWAY_TIMEOUT`; other unavailable
+outcomes become `502 BAD_GATEWAY`; malformed success responses become
+`502 DOWNSTREAM_CONTRACT_VIOLATION`. Sensitive headers and credential-like values are redacted
+from structured logs.
+
+## Fail-closed audit lifecycle
+
+PostgreSQL is the authoritative audit store.
+
+1. Before an audited downstream call, a separate transaction inserts a `STARTED` row containing
+   the UUIDv7 action ID, actor, role, action, target, reason, trace ID, and start time.
+2. Failure to insert `STARTED` prevents downstream execution and returns `503 AUDIT_UNAVAILABLE`
+   with the same action ID.
+3. Completion performs one guarded `STARTED`-to-terminal update.
+4. Failure to finalize returns `503 AUDIT_FINALIZATION_FAILED`; an earlier application failure is
+   retained as suppressed context.
+5. Terminal rows are copied best effort to Kafka topic `admin.action`, keyed by actor ID and encoded
+   with the `AdminActionRecorded` Avro schema. Kafka failure never replaces the authoritative
+   database result.
+
+For audited mutations that reach the method boundary, terminal classification is:
+
+- successful `2xx`: `SUCCESS` with the exact response status
+- explicit `4xx` or authorization denial: `FAILED`
+- timeout, downstream `5xx`, malformed success, or unexpected failure: `UNKNOWN`
+
+A scheduler scans every 30 seconds by default. `STARTED` rows older than five minutes are claimed
+with `FOR UPDATE SKIP LOCKED`, transitioned once to `UNKNOWN`, and published. Defaults are
+configurable with:
+
+- `ADMIN_AUDIT_TOPIC`
+- `ADMIN_AUDIT_STALE_AFTER`
+- `ADMIN_AUDIT_STALE_SCAN_INTERVAL`
+- `ADMIN_AUDIT_STALE_BATCH_SIZE`
+
+## Database migrations
+
+Admin API owns only `audit_log`.
+
+- `V1__audit_log.sql` creates the original audit table and indexes; its checksum is locked.
+- `V2__audit_lifecycle.sql` adds the fail-closed lifecycle, terminal timestamps, constraints, and
+  the partial stale-row index.
+- Flyway supports both a clean V1-to-V2 installation and upgrade of an existing V1 database while
+  preserving legacy completion evidence.
+
+Hibernate validates the schema; it does not create or update it.
+
+## Configuration
+
+Production deployments should explicitly set:
+
+```text
+ADMIN_JWT_PUBLIC_KEY
+ADMIN_JWT_ISSUER
+ADMIN_IP_ALLOWLIST
+ADMIN_TRUSTED_PROXY_CIDRS
+
+ADMIN_DB_URL
+ADMIN_DB_USER
+ADMIN_DB_PASSWORD
+ADMIN_KAFKA_BOOTSTRAP
+
+ADMIN_WALLET_BASE_URL
+ADMIN_RISK_BASE_URL
+ADMIN_ODDS_FEED_BASE_URL
+ADMIN_SETTLEMENT_BASE_URL
+
+ADMIN_WALLET_API_KEY
+ADMIN_RISK_API_KEY
+ADMIN_ODDS_FEED_API_KEY
+ADMIN_SETTLEMENT_API_KEY
+```
+
+`ADMIN_JWT_PUBLIC_KEY` accepts an inline PEM with real newlines or escaped `\n`. Downstream
+connection and read timeouts default to `200ms` and `2s`; override them with
+`ADMIN_DOWNSTREAM_CONNECT_TIMEOUT` and `ADMIN_DOWNSTREAM_READ_TIMEOUT`.
+
+## Health and observability
+
+- `GET /actuator/health/liveness` checks application liveness.
+- `GET /actuator/health/readiness` contains only `readinessState` and PostgreSQL health.
+- `GET /actuator/prometheus` exposes metrics.
+- Health component details are hidden.
+
+Kafka and downstream services are intentionally not readiness dependencies because Kafka
+publication is best effort and downstream availability is evaluated per request. Readiness becomes
+`DOWN` when PostgreSQL is unavailable.
+
+Audit metrics include:
+
+- `admin.audit.stale.claimed`
+- `admin.audit.stale.scan.failure`
+- `admin.audit.publish.failure`
+
+## Build and run locally
+
+Prerequisites are JDK 17, Docker, and the fixed `shared-protocol` checkout. Docker is required by
+the PostgreSQL and Kafka verification tests.
+
+Install the shared artifact into an isolated Maven repository:
+
+```bash
+cd /path/to/shared-protocol
+./mvnw -Dmaven.repo.local=/tmp/admin-api-m2 -DskipTests install
+```
+
+Verify Admin API:
+
+```bash
+cd /path/to/admin-api
+./mvnw -Dmaven.repo.local=/tmp/admin-api-m2 clean verify
+```
+
+After configuring PostgreSQL, Kafka, JWT, network policy, downstream URLs, and four distinct caller
+keys, run either:
+
+```bash
+./mvnw -Dmaven.repo.local=/tmp/admin-api-m2 spring-boot:run
+```
+
+or:
+
+```bash
+java -jar target/admin-api-1.0.0.jar
+```
+
+## Release artifact
+
+The release build produces:
+
+```text
+target/admin-api-1.0.0.jar
+```
+
+It is an executable Java 17 Spring Boot JAR and depends on exactly
+`com.sportsbook:shared-protocol:1.0.0`.
