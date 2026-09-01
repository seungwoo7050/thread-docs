## `docs(project): document settlement service`

diff --git a/README.md b/README.md
index a2a0710..537bb5d 100644
--- a/README.md
+++ b/README.md
@@ -1,3 +1,260 @@
 # Settlement Service
 
-Owns durable bet settlement, lifecycle voiding, corrected-result revisions, and publication of terminal settlement events.
+Settlement Service owns durable bet resolution after Betting has accepted a wager. It consumes
+placements, event lifecycle changes, and official match results; performs the authorized Wallet
+operations; and publishes the terminal resolution records used by Betting and Gateway.
+
+## Runtime contract
+
+- Java 17
+- Spring Boot 3.2
+- PostgreSQL with Flyway migrations V1 through V10
+- Kafka records encoded as raw Avro from `com.sportsbook:shared-protocol:1.0.0`
+- HTTP port 8084 by default
+- At-least-once Kafka processing and publication
+- Database-time, owner-fenced leases for recoverable work
+
+PostgreSQL is the source of truth. Kafka acknowledgements are issued only after the corresponding
+database decision or exact dead-letter publication is durable. Kafka and process memory never
+substitute for persisted monetary evidence.
+
+## Input and output topics
+
+| Direction | Topic | Avro record | Required key |
+| --- | --- | --- | --- |
+| Consume | `bet.placed.v1` | `BetPlacedRequested` | `userId` |
+| Consume | `event.lifecycle` | `EventLifecycle` | `eventId` |
+| Consume | `match.result` | `MatchResult` | `eventId` |
+| Publish | `bet.settled.v1` | `BetSettled` | `eventId` |
+| Publish | `bet.voided.v1` | `BetVoided` | `eventId` |
+| Publish | `bet.resolution.revised.v1` | `BetResolutionRevised` | `betId` |
+
+Every consumed source topic has an uppercase `.DLT` companion. Source and DLT topics must be
+provisioned explicitly with matching partition coverage because topic auto-creation is disabled.
+Permanent key, actor, payload, chronology, or state failures are sent to the exact source
+partition. Infrastructure failures remain eligible for source retry.
+
+Consumers start at `earliest`, use `read_committed`, disable auto commit, and acknowledge records
+manually after the durable boundary.
+
+## Placement intake
+
+Settlement stores accepted wager identity, selections, odds, and exposure from
+`BetPlacedRequested`. Duplicate delivery with the same durable fingerprint is idempotent;
+conflicting reuse is a permanent contract failure.
+
+SINGLE and MULTIPLE placements require null `systemMinWins` and `systemTotalSelections`. SYSTEM
+placements require both fields. A SYSTEM stake is a unit stake, while the monetary exposure is:
+
+```text
+unit stake × C(systemTotalSelections, systemMinWins)
+```
+
+That distinction is preserved in persistence, payout calculation, Wallet operations, and emitted
+resolution snapshots. Combination and money calculations are overflow checked.
+
+Input topics have no global ordering. A lifecycle terminal state or accepted result can arrive
+before its placement. Settlement persists the earlier fact and catches up when the placement
+arrives. Concurrent catch-up and listener execution use row locks and database constraints so a
+bet receives one base resolution attempt.
+
+## Base settlement
+
+An accepted final result fans out to every unresolved placement containing the event. Each
+selection is evaluated from the persisted accepted result, and multi-event wagers may remain
+pending until all required selections are terminal.
+
+When a bet is ready, Settlement persists a complete attempt before calling Wallet. Wallet calls
+use stable idempotency identities derived from the bet and operation purpose. A successful HTTP
+response must contain a complete proof matching the trusted user, amount, operation group,
+currency, reason, and timestamp.
+
+Base settlement may perform these authorized operations:
+
+- return stake through `USER_LOCKED + REFUND`;
+- return a whole-slip void through `USER_LOCKED + VOID`;
+- pay profit through `HOUSE_POOL + PAYOUT`; and
+- forfeit locked exposure through the Settlement-owned forfeit operation.
+
+Once all required Wallet evidence exists, Settlement consumes the still-valid lease, records the
+terminal bet state with one database timestamp, and inserts the outbox event in the same
+transaction. An expired or replaced owner cannot finalize or delete another worker's attempt.
+
+`MatchResult.finalStatus=VOIDED` is an ordinary market result. It produces `BetSettled` with bet
+status `SETTLED`, result `VOID`, and normal payout rules. `BetVoided` is reserved for a whole-slip
+terminal action caused by event `CANCELLED`, event `POSTPONED`, or an authenticated administrative
+decision. `MARKET_VOID` remains a wire enum symbol for compatibility but is not a valid produced
+`BetVoided` reason.
+
+## Result corrections
+
+Every result delivery is stored as a durable candidate. Candidate identity includes its source
+event and result fingerprint. The accepted base candidate creates logical revision 0. A later
+valid candidate may replace the currently accepted candidate and fan out an immutable revision
+plan to each eligible `SETTLED` bet.
+
+A correction is rejected before replay or ordering classification when its
+`sourceResultSettledAt` is after `revisedAt`. Whole-slip `VOIDED`, `REJECTED`, and other
+non-`SETTLED` bets are not revised. A normally settled `VOID` result remains eligible for a later
+correction.
+
+The plan fixes the candidate, previous resolution, new resolution, revision number, payout delta,
+Wallet identity, and final event payload before any external call. A stale target is locked and
+rechecked. Only a newly inserted plan can execute directly; recovery always reloads the persisted
+plan.
+
+Positive and negative payout deltas use the Wallet adjustment contract with the stable key:
+
+```text
+settlement:revision:<revisionId>
+```
+
+A zero delta never calls Wallet. A nonzero delta can finalize only with an exact `APPLIED` proof.
+HTTP 202 `BLOCKED` retains its positive queue sequence and queue timestamps. Authoritative Wallet
+`REJECTED` is the only semantic rejection. Timeouts, malformed responses, unexpected statuses,
+and lost responses never invent a business verdict.
+
+## Revision recovery
+
+Ambiguous and scheduled work is reclaimed with PostgreSQL database time, `FOR UPDATE SKIP LOCKED`,
+bounded batches, and owner-fenced leases. Recovery performs Wallet GET first. It may repeat the
+same POST only after the exact adjustment-not-found 404 and only under the original idempotency
+identity.
+
+Automatic execution is bounded to 12 attempts with capped exponential backoff:
+
+- a plan without durable Wallet proof becomes `EXHAUSTED` and pauses;
+- a plan with durable `BLOCKED` proof remains `BLOCKED`, preserves its queue identity, and pauses;
+- a Wallet-supplied semantic rejection becomes `REJECTED`; and
+- an `APPLIED` proof permits atomic revision and outbox finalization.
+
+The correction catch-up scanner handles valid candidates that raced with placement or base
+settlement. Repeated, sequential, and concurrent delivery preserves monotonic revision numbers and
+the immutable event identity.
+
+## Administrative API
+
+All routes below require exactly one of each header:
+
+```text
+X-Service-Name: admin-api
+X-API-Key: <SETTLEMENT_ADMIN_API_KEY>
+```
+
+Mutation routes also require a UUID `Idempotency-Key`. The configured admin secret must contain at
+least 32 characters and is redacted from configuration rendering. Missing credentials return 401;
+duplicates or invalid credentials return 403. Error responses use safe `application/problem+json`
+details and do not expose dependency bodies or secrets.
+
+| Method | Route | Purpose |
+| --- | --- | --- |
+| GET | `/internal/admin/result-candidates/{candidateId}` | Inspect candidate decision state |
+| GET | `/internal/admin/revisions/{revisionId}` | Inspect revision and Wallet evidence |
+| POST | `/internal/admin/result-candidates/{candidateId}/approve` | Approve an eligible pending candidate |
+| POST | `/internal/admin/result-candidates/{candidateId}/reject` | Reject a pending candidate with a reason |
+| POST | `/internal/admin/revisions/{revisionId}/retry` | Queue paused revision recovery |
+
+Admin actions are append-only and bind their idempotency key to a request fingerprint. Exact
+replay returns the prior outcome; semantic key reuse conflicts. Candidate approval is predecessor
+fenced. Rejection reasons must contain 1 to 256 printable characters.
+
+Revision retry never calls Wallet on the HTTP thread. It queues eligible paused work. An
+`EXHAUSTED` no-proof plan returns to due `PENDING` work at attempt 0. A paused `BLOCKED` plan keeps
+its proof and queue schedule. The recovery worker then claims attempt 1 and applies the same
+GET-first rule before any possible POST. Replaying the admin request does not reset attempts.
+
+## Wallet authentication
+
+Settlement sends exactly:
+
+```text
+X-Internal-Service: settlement-service
+X-Internal-Api-Key: <SETTLEMENT_WALLET_API_KEY>
+```
+
+`SETTLEMENT_WALLET_API_KEY` must be nonblank, at least 32 characters, and different from
+`SETTLEMENT_ADMIN_API_KEY`; duplicate values fail startup. The Wallet deployment must configure
+the matching `WALLET_SETTLEMENT_SERVICE_API_KEY`. The client has its own base URL, credential
+attachment, connect timeout, and read timeout. Those timeouts must be positive and no greater than
+five seconds.
+
+## Configuration
+
+| Environment variable | Default | Meaning |
+| --- | --- | --- |
+| `SETTLEMENT_HTTP_PORT` | `8084` | HTTP and actuator port |
+| `SETTLEMENT_ADMIN_API_KEY` | required | Admin control-plane secret |
+| `SETTLEMENT_WALLET_API_KEY` | required | Settlement-to-Wallet secret |
+| `SETTLEMENT_WALLET_BASE_URL` | `http://localhost:8081` | Canonical Wallet origin |
+| `SETTLEMENT_WALLET_CONNECT_TIMEOUT` | `PT2S` | Wallet connect timeout |
+| `SETTLEMENT_WALLET_READ_TIMEOUT` | `PT5S` | Wallet read timeout |
+| `SETTLEMENT_CORRECTION_WINDOW` | `PT24H` | Automatic correction acceptance window |
+| `SETTLEMENT_RECOVERY_INTERVAL` | `PT1S` | Recovery and catch-up cadence |
+| `SETTLEMENT_LEASE_DURATION` | `PT30S` | Durable work lease |
+| `SETTLEMENT_BATCH_SIZE` | `100` | Maximum work claimed per scan |
+| `SETTLEMENT_OUTBOX_INTERVAL` | `PT1S` | Outbox publication cadence |
+| `SETTLEMENT_WORKERS_ENABLED` | `true` | Enable scheduled recovery and publication workers |
+| `SETTLEMENT_TOPIC_BET_PLACED` | `bet.placed.v1` | Placement source topic |
+| `SETTLEMENT_TOPIC_MATCH_RESULT` | `match.result` | Result source topic |
+| `SETTLEMENT_TOPIC_EVENT_LIFECYCLE` | `event.lifecycle` | Lifecycle source topic |
+| `SETTLEMENT_TOPIC_BET_SETTLED` | `bet.settled.v1` | Base settlement output |
+| `SETTLEMENT_TOPIC_BET_REVISED` | `bet.resolution.revised.v1` | Revision output |
+| `SETTLEMENT_TOPIC_BET_VOIDED` | `bet.voided.v1` | Whole-slip void output |
+
+Use standard Spring Boot datasource and Kafka bootstrap configuration for PostgreSQL and Kafka.
+Hibernate validates the schema only after Flyway has applied the append-only migrations.
+
+## Workers and shutdown
+
+Base recovery, lifecycle catch-up, result catch-up, correction recovery, and outbox publication use
+isolated schedulers so one slow flow cannot starve the others. Runtime worker enablement can be
+disabled in test or controlled environments. Shutdown is graceful with a bounded 20-second Spring
+phase, and every worker executor has a bounded termination policy.
+
+Claims use random owner identities, lease tokens, database timestamps, and stale-owner fences.
+The PostgreSQL claim path supports concurrent workers through `SKIP LOCKED`; each row is returned
+to at most one live owner for that lease generation.
+
+## Health and metrics
+
+Actuator exposes `health`, `info`, and `prometheus`. Kubernetes-style liveness and readiness probes
+are enabled. Readiness includes the custom `settlementDependencies` indicator, which verifies the
+database and reports durable backlog details without exposing credentials.
+
+Prometheus metrics include:
+
+- `settlement_operations_total{flow,outcome}` for base, lifecycle, revision, and admin outcomes;
+- `settlement_operation_duration_seconds{flow}` for bounded flow timings; and
+- `settlement_backlog{kind}` for pending bets, blocked revisions, exhausted revisions, and
+  unpublished outbox rows.
+
+Metric labels are bounded constants rather than request or entity identifiers.
+
+## Database migrations
+
+Flyway migrations are append-only:
+
+- V1: bet read model
+- V3: transactional outbox
+- V4: accepted match result
+- V5: base settlement attempts
+- V6: durable event lifecycle
+- V7: result candidates
+- V8: source revision identity
+- V9: immutable settlement revision plans and Wallet evidence
+- V10: append-only administrative actions
+
+Released migrations must never be edited in place. Add V11 or later for a future schema change.
+
+## Build and verification
+
+Install the exact shared protocol artifact, provide Docker for PostgreSQL integration tests, then
+run the project with Java 17:
+
+```bash
+./mvnw clean verify
+```
+
+The verify lifecycle applies formatting and style gates, unit and contract tests, PostgreSQL 16
+integration and concurrency tests, migration immutability checks, packaging, and the archive
+history guard. CI checks out the fixed shared-protocol commit and uses an isolated Maven repository.
