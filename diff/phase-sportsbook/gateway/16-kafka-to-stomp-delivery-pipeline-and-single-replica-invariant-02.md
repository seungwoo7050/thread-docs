## `docs(project): document API gateway`

diff --git a/README.md b/README.md
index 6fc6ef7..2671c56 100644
--- a/README.md
+++ b/README.md
@@ -1,23 +1,61 @@
 # Sportsbook API Gateway
 
-The gateway is the public HTTP and WebSocket boundary for the sportsbook platform. It will verify
-user access tokens, apply distributed request limits, proxy approved API routes, and fan out live
-updates from Kafka to STOMP clients.
+The gateway is the public HTTP and WebSocket boundary for the sportsbook platform. It validates
+user JWTs, enforces distributed request limits, proxies an allowlist of HTTP routes, and projects
+validated Kafka events to STOMP clients. It does not own betting, wallet, odds, settlement, or risk
+state.
 
-## Contract boundary
+## Runtime contract
 
-The service is intended to expose only explicitly approved public routes. Internal service headers
-are not part of the public contract and must never be trusted when supplied by a client.
+- Java 17 and `com.sportsbook:shared-protocol:1.0.0`
+- RS256 JWT verification with a required expiry and canonical UUID subject
+- Redis-backed limits of 120 requests per minute for authenticated users and 60 per minute for
+  anonymous client addresses
+- 500 ms downstream connect timeout and 3 s downstream read timeout
+- Public odds streams and owner-scoped bet updates over STOMP
+- Raw Avro consumption from four Kafka inputs with a same-partition dead-letter topic for each
+- Exactly one gateway replica
 
-The gateway consumes the shared protocol library for common value, error, and event contracts. It
-does not own betting, wallet, risk, odds, or settlement state.
+The gateway is intentionally stateless with respect to bets and result revisions. Redis stores only
+rate-limit buckets. WebSocket sessions and expiry tasks are local to the running process.
 
-## Runtime baseline
+## Build
 
-The service targets Java 17 and is built with Maven. Runtime configuration is supplied through the
-environment; credentials and private key material do not belong in the repository.
+The shared protocol artifact must be available to Maven before building the gateway.
 
-## Current status
+```sh
+./mvnw clean verify
+```
 
-This branch currently contains the project introduction only. Build configuration and runtime
-behavior are added in subsequent development commits.
+The Maven wrapper uses Maven 3.9.11. Integration tests require a working Docker daemon for the
+Redis container used by the rate-limit tests.
+
+## Run
+
+Supply the three required runtime values through the runtime environment:
+
+- `GATEWAY_JWT_PUBLIC_KEY`: an RSA public key in `PUBLIC KEY` PEM form, at least 2048 bits
+- `GATEWAY_BETTING_API_KEY`: the gateway-specific betting credential, at least 32 characters
+- `GATEWAY_WALLET_API_KEY`: the gateway-specific wallet credential, at least 32 characters
+
+The two downstream credentials must be different. A servlet deployment fails during startup when
+either credential is missing or weak, or when the same value is reused for both services.
+
+Then start the packaged service:
+
+```sh
+java -jar target/gateway-1.0.0.jar
+```
+
+The default HTTP port is `8080`. Redis defaults to `localhost:6379`, Kafka to `localhost:9092`,
+and downstream service locations to their local development ports. Production deployments should
+set every dependency location explicitly and must provision all input and dead-letter topics before
+starting the service.
+
+## Documentation
+
+- [Architecture](docs/architecture.md)
+- [HTTP contract](docs/http-contract.md)
+- [Realtime contract](docs/realtime-contract.md)
+- [Operations](docs/operations.md)
+- [Build and use](docs/build-and-use.md)
diff --git a/docs/architecture.md b/docs/architecture.md
new file mode 100644
index 0000000..9bb1c65
--- /dev/null
+++ b/docs/architecture.md
@@ -0,0 +1,109 @@
+# Gateway Architecture
+
+## Responsibility
+
+The gateway owns the platform's public transport boundary. Its responsibilities are deliberately
+narrow:
+
+1. authenticate users at HTTP and STOMP entry points;
+2. remove caller-controlled trust headers;
+3. apply Redis-backed request limits;
+4. proxy only approved HTTP method and path combinations;
+5. validate raw Avro events before projecting them to WebSocket clients; and
+6. quarantine records that cannot be delivered safely.
+
+Business state remains in the downstream services. The gateway has no database, does not call the
+risk service, and does not keep the latest state or revision for a bet.
+
+## Request path
+
+```text
+HTTP client
+    |
+    v
+trusted-header removal -> JWT security -> rate limit -> exact route
+                                                     |
+                           +-------------------------+-------------------------+
+                           |                         |                         |
+                           v                         v                         v
+                    betting service            wallet service             odds feed
+                  identity + betting key    identity + wallet key        public reads
+```
+
+The trusted-header filter runs at the outside edge and hides all case variants of
+`X-User-Id`, `X-User-Roles`, `X-Internal-Service`, and `X-Internal-Api-Key`. Route filters remove
+those headers again before constructing downstream identity. They also remove the external bearer
+token from every downstream request.
+
+Authenticated betting and wallet calls receive `X-User-Id` derived from the verified JWT. If the
+token has roles, the gateway supplies them as a comma-separated `X-User-Roles` value. Both private
+downstream routes receive `X-Internal-Service: gateway` plus their distinct configured API key.
+Servlet startup validates the two keys together and rejects a shared value. Public odds-feed calls
+receive no identity or internal credential headers.
+
+## Realtime path
+
+```text
+Kafka input (raw key and value)
+    |
+    v
+strict Avro decode -> key and payload validation -> local STOMP projection
+    |                                                   |
+    |                                                   +-> public event topic
+    |                                                   +-> owner user queue
+    v
+same-partition DLT after a permanent failure or exhausted transient retries
+```
+
+Kafka keys and values remain byte arrays until the event contract validates them. The gateway uses
+the generated record types from `shared-protocol` and rejects malformed binary data, trailing
+bytes, invalid identities, or key/payload mismatches.
+
+The STOMP broker is process-local. Authenticated sessions are associated with the JWT subject, and
+terminal bet updates are sent through Spring's user destination mapping. Odds updates are broadcast
+on an event-specific public topic.
+
+## State and delivery boundaries
+
+Redis contains distributed token buckets under gateway-owned key prefixes. Redis is not an identity
+or business-state source. If Redis is unavailable, the current request is admitted and the failure
+is counted; later requests resume distributed limiting after connectivity returns.
+
+WebSocket sessions and token-expiry schedules exist only in process memory. The gateway does not
+offer durable WebSocket subscriptions or replay. Kafka delivery and WebSocket delivery may produce
+duplicates around failures, so clients must tolerate repeated projections. Bet clients use revision
+numbers as described in the realtime contract.
+
+## Single-replica deployment
+
+Gateway 1.0 runs as exactly one replica. Kafka consumer groups divide partitions among consumers,
+while the simple STOMP broker knows only the clients connected to its own process. Multiple replicas
+would therefore allow one process to consume an update whose subscriber is connected to another.
+Deployment configuration must enforce a replica count of one.
+
+Scaling beyond one replica requires a shared broker or another cross-instance fan-out design and is
+outside this contract.
+
+## Dependency boundaries
+
+| Dependency | Gateway use | Failure behavior |
+|---|---|---|
+| shared protocol | Money, problem, and Avro event types | Required at build time |
+| betting service | Bet placement and reads | 502 on connection or I/O failure; 504 on read timeout |
+| wallet service | Authenticated balance read | Same proxy failure mapping |
+| odds feed | Anonymous event and odds reads | Same proxy failure mapping |
+| Redis | Distributed rate-limit buckets | Request is admitted on Redis failure |
+| Kafka | Four event inputs and four dead-letter outputs | Failed source offset is retained if DLT publication fails |
+
+There is no direct gateway-to-risk contract.
+
+## Operational surface
+
+Only `health`, `info`, and `prometheus` Actuator endpoints are exposed. Liveness and readiness
+report application availability state; they do not include Redis, Kafka, or downstream reachability.
+This prevents a dependency outage from creating an automatic restart loop and makes dependency
+alerts a metrics and logs concern.
+
+Structured logs are emitted as one JSON object per line. The stable service field is `gateway`, and
+only `traceId` and `spanId` are admitted from MDC. Recognizable credential patterns in formatted
+messages and emitted stack traces are replaced with `[REDACTED]`.
diff --git a/docs/build-and-use.md b/docs/build-and-use.md
new file mode 100644
index 0000000..991504f
--- /dev/null
+++ b/docs/build-and-use.md
@@ -0,0 +1,100 @@
+# Build and Use
+
+## Prerequisites
+
+- JDK 17
+- a working Docker daemon for Redis integration tests
+- `com.sportsbook:shared-protocol:1.0.0` available from the configured Maven repositories
+
+The checked-in wrapper downloads Maven 3.9.11.
+
+## Verify
+
+Run the complete build from the gateway project root:
+
+```sh
+./mvnw clean verify
+```
+
+The verify lifecycle compiles with Java release 17, runs unit and integration tests, checks Google
+Java Format compliance, and applies Checkstyle validation. Tests cover JWT and header boundaries,
+exact routing, Redis limiting, STOMP authorization and expiry, raw Avro contracts, Kafka retry and
+DLT behavior, fan-out isolation, operational endpoints, structured logging, and concurrent identity
+isolation.
+
+The Redis integration suite starts a Redis container. Embedded Kafka tests create their own brokers
+and topics.
+
+## Package
+
+The same verify command produces an executable Spring Boot artifact:
+
+```text
+target/gateway-1.0.0.jar
+```
+
+The artifact identity is `com.sportsbook:gateway:1.0.0`. Its runtime dependency set contains
+`com.sportsbook:shared-protocol:1.0.0`.
+
+## Start locally
+
+Make Redis, Kafka, betting, wallet, and odds-feed endpoints available, then inject the required
+runtime values through the process environment:
+
+- `GATEWAY_JWT_PUBLIC_KEY`
+- `GATEWAY_BETTING_API_KEY`
+- `GATEWAY_WALLET_API_KEY`
+
+Start from source:
+
+```sh
+./mvnw spring-boot:run
+```
+
+Or start the packaged artifact:
+
+```sh
+java -jar target/gateway-1.0.0.jar
+```
+
+Do not put these required values in a tracked environment file. The full configuration inventory is
+in [Operations](operations.md).
+
+## Basic checks
+
+After startup, application availability and build identity are available without a token:
+
+```sh
+curl --fail http://localhost:8080/actuator/health/liveness
+curl --fail http://localhost:8080/actuator/health/readiness
+curl --fail http://localhost:8080/actuator/info
+```
+
+Prometheus can scrape:
+
+```text
+GET /actuator/prometheus
+```
+
+These probes report process availability, not Redis, Kafka, or downstream reachability.
+
+## Runtime use
+
+Use HTTP bearer authentication for private REST routes. Supply the bearer token in the STOMP
+`CONNECT` frame for the private bet queue. Anonymous clients may use public event and odds HTTP
+reads and may subscribe to canonical event destinations on the odds stream.
+
+See [HTTP contract](http-contract.md) for exact routes and [Realtime contract](realtime-contract.md)
+for STOMP destinations, Kafka inputs, DLT behavior, and client revision handling.
+
+## Deployment checklist
+
+- Run exactly one replica.
+- Inject the RSA public key and distinct betting and wallet credentials through the deployment
+  secret system; servlet startup rejects credential reuse.
+- Set production dependency URIs and broker locations explicitly.
+- Replace the default WebSocket origin patterns.
+- Configure trusted proxy CIDRs only for controlled direct peers.
+- Provision each DLT with the same partition count as its source and at least seven days of
+  retention.
+- Scrape Prometheus and collect standard-output JSON logs.
diff --git a/docs/http-contract.md b/docs/http-contract.md
new file mode 100644
index 0000000..c7e6cba
--- /dev/null
+++ b/docs/http-contract.md
@@ -0,0 +1,136 @@
+# HTTP Contract
+
+## Authentication
+
+Private routes require an HTTP bearer token signed with RS256. The configured RSA public key must
+use X.509 `PUBLIC KEY` PEM encoding and have a modulus of at least 2048 bits. The decoder accepts a
+literal multiline PEM or an environment value containing escaped newline sequences.
+
+Every accepted token has:
+
+- an `exp` claim that is still in the future, with no clock-skew allowance; and
+- a nonblank `sub` equal to the lowercase canonical string form of a UUID.
+
+The `roles` claim is optional. When present, it must be an array of at most 16 distinct strings.
+Every role must match `[A-Z][A-Z0-9_]{0,31}`. The gateway does not evaluate issuer or audience and
+does not maintain a token revocation store.
+
+Authentication failure returns `401 GATEWAY_UNAUTHORIZED`. An authenticated request outside the
+method/path allowlist returns `403 GATEWAY_FORBIDDEN`. Error dispatches are allowed through the
+security boundary so that a public-route proxy failure is not replaced by an authentication error.
+
+## Public routes
+
+| External request | Authentication | Downstream request |
+|---|---|---|
+| `POST /api/v1/bets` | Required | `POST /internal/v1/bets` on betting |
+| `GET /api/v1/bets` | Required | `GET /internal/v1/bets` on betting |
+| `GET /api/v1/bets/{betId}` | Required | Same suffix under `/internal/v1/bets` |
+| `GET /api/v1/wallet/balance` | Required | `GET /internal/v1/wallet/accounts/{sub}/balance` |
+| `GET /api/v1/events` | Anonymous allowed | Same path on odds feed |
+| `GET /api/v1/events/{eventId}` | Anonymous allowed | Same path on odds feed |
+| `GET /api/v1/odds/{eventId}/{marketId}/{selectionId}` | Anonymous allowed | Same path on odds feed |
+
+No other application route or method is exposed. The gateway treats `{betId}` as an opaque path
+segment; the betting service remains responsible for validating that resource identifier.
+
+For the bet collection read, the gateway overwrites any caller-supplied `userId` query value with
+the verified JWT subject. Other query parameters are retained. The individual bet path is forwarded
+unchanged after the public prefix is replaced.
+
+## Header boundary
+
+The following inbound headers are always hidden before authentication and routing, using
+case-insensitive name matching:
+
+- `X-User-Id`
+- `X-User-Roles`
+- `X-Internal-Service`
+- `X-Internal-Api-Key`
+
+Every downstream request also has `Authorization` removed. The gateway then adds only the headers
+required by that route:
+
+| Header | Betting | Wallet balance | Public odds feed |
+|---|---|---|---|
+| `X-User-Id` | Verified `sub` | Verified `sub` | Not sent |
+| `X-User-Roles` | Verified roles, when nonempty | Verified roles, when nonempty | Not sent |
+| `X-Internal-Service` | `gateway` | `gateway` | Not sent |
+| `X-Internal-Api-Key` | Configured betting key | Configured wallet key | Not sent |
+
+Betting and wallet credentials are required to be distinct. Servlet startup rejects a missing,
+short, or shared value without including either secret in the failure. The gateway injects each
+only on its corresponding route after removing any caller-supplied internal headers.
+
+## Request and response relay
+
+The proxy retains request bodies and ordinary application headers, including `Idempotency-Key`.
+It also retains an inbound `traceparent` only when there is exactly one valid version `00` value
+with lowercase hexadecimal, nonzero trace and span identifiers, and flags `00` or `01`. Invalid or
+ambiguous values are removed. When tracing has an active span, the gateway can replace a removed or
+absent value with that span's valid `traceparent`.
+
+Downstream status, body, content type, `Location`, and `Retry-After` are relayed. A downstream
+application error therefore remains the downstream service's contract rather than being rewritten
+as a gateway error.
+
+The HTTP client uses these default bounds:
+
+| Stage | Default | Environment setting |
+|---|---:|---|
+| Connection establishment | 500 ms | `GATEWAY_DOWNSTREAM_CONNECT_TIMEOUT` |
+| Response read | 3 s | `GATEWAY_DOWNSTREAM_READ_TIMEOUT` |
+
+A connection or other downstream I/O failure returns `502 GATEWAY_BAD_GATEWAY`. A read timeout
+returns `504 GATEWAY_TIMEOUT`.
+
+## Gateway problem response
+
+Gateway-owned failures use `application/problem+json` and the shared protocol problem shape:
+
+```json
+{
+  "type": "https://sportsbook/errors/upstream-timeout",
+  "title": "Gateway Timeout",
+  "status": 504,
+  "errorCode": "GATEWAY_TIMEOUT",
+  "detail": "An upstream service timed out.",
+  "instance": "/api/v1/events/example",
+  "correlationId": "generated-or-active-trace-id"
+}
+```
+
+| Status | Error code | Meaning |
+|---:|---|---|
+| 401 | `GATEWAY_UNAUTHORIZED` | A private route lacks valid authentication |
+| 403 | `GATEWAY_FORBIDDEN` | The method or path is not allowed |
+| 429 | `GATEWAY_RATE_LIMITED` | The distributed token bucket rejected the request |
+| 502 | `GATEWAY_BAD_GATEWAY` | The downstream connection or I/O path failed |
+| 504 | `GATEWAY_TIMEOUT` | The downstream read exceeded its bound |
+
+The correlation ID is the active trace ID when one exists; otherwise it is a newly generated UUID.
+
+## Request limiting
+
+The gateway applies one token bucket to each application request when limiting is enabled; Actuator
+and error-dispatch paths are excluded. Authenticated requests use the canonical JWT subject, and
+anonymous requests use a client address.
+
+| Class | Default capacity | Redis key prefix |
+|---|---:|---|
+| Authenticated user | 120 per minute | `gateway:ratelimit:user:` |
+| Anonymous address | 60 per minute | `gateway:ratelimit:ip:` |
+
+Successful Redis-backed decisions include `X-RateLimit-Remaining`. A rejection returns 429 with
+`X-RateLimit-Remaining: 0` and an integer-seconds `Retry-After` value of at least one.
+
+The trusted-proxy CIDR list is empty by default. When the socket peer is not trusted, the gateway
+ignores `X-Forwarded-For`. When the peer is trusted, it parses every hop and walks from right to
+left to select the first untrusted address. An invalid hop or a chain containing only trusted hops
+falls back to the socket peer.
+
+Redis connection and command operations are bounded at 300 ms and 500 ms respectively. A Redis
+failure admits the current request without a remaining-token header and increments
+`gateway.ratelimit.fail.open`. The next requests continue attempting to use the distributed limit.
+
+Invalid capacities, refill periods, or trusted CIDRs prevent application startup.
diff --git a/docs/operations.md b/docs/operations.md
new file mode 100644
index 0000000..dc428af
--- /dev/null
+++ b/docs/operations.md
@@ -0,0 +1,169 @@
+# Gateway Operations
+
+## Deployment requirements
+
+Run exactly one gateway replica with Java 17. Before startup:
+
+1. make Redis and Kafka reachable;
+2. provision the four Kafka inputs and their four DLTs;
+3. configure the betting, wallet, and odds-feed base URIs;
+4. inject the RSA public key and distinct betting and wallet API keys from the deployment secret
+   system; and
+5. configure the allowed WebSocket origins.
+
+The service does not perform a startup reachability check for Redis, Kafka, or downstream services.
+Configuration shape and required runtime values are validated during application creation. A
+servlet deployment fails startup if the betting and wallet keys are equal; failure text does not
+render either value.
+
+## Required runtime values
+
+| Environment variable | Requirement |
+|---|---|
+| `GATEWAY_JWT_PUBLIC_KEY` | X.509 `PUBLIC KEY` PEM for an RSA key of at least 2048 bits |
+| `GATEWAY_BETTING_API_KEY` | Nonblank betting secret of at least 32 characters, distinct from the wallet key |
+| `GATEWAY_WALLET_API_KEY` | Nonblank wallet secret of at least 32 characters, distinct from the betting key |
+
+The public key may contain literal newlines or escaped `\n` sequences. Do not place these values in
+tracked configuration, command examples, logs, or diagnostic bundles. Changing any required value
+requires a process restart because decoders and downstream authenticators are constructed at startup.
+
+## HTTP and downstream configuration
+
+| Environment variable | Default | Validation or use |
+|---|---|---|
+| `GATEWAY_HTTP_PORT` | `8080` | Public HTTP and WebSocket port |
+| `GATEWAY_BETTING_URI` | `http://localhost:8082` | Betting HTTP base URI |
+| `GATEWAY_WALLET_URI` | `http://localhost:8081` | Wallet HTTP base URI |
+| `GATEWAY_ODDS_FEED_URI` | `http://localhost:8085` | Odds-feed HTTP base URI |
+| `GATEWAY_DOWNSTREAM_CONNECT_TIMEOUT` | `500ms` | Downstream connection bound |
+| `GATEWAY_DOWNSTREAM_READ_TIMEOUT` | `3s` | Downstream response-read bound |
+| `GATEWAY_WS_ALLOWED_ORIGINS` | local HTTP origin patterns | Comma-separated Spring origin patterns |
+
+Each downstream base URI must be an absolute `http` or `https` URI with a host and root path. User
+information, a non-root path, query, and fragment are rejected.
+
+The process uses graceful shutdown with a 20-second Spring lifecycle phase timeout.
+
+## Redis and rate-limit configuration
+
+| Environment variable | Default |
+|---|---|
+| `GATEWAY_REDIS_HOST` | `localhost` |
+| `GATEWAY_REDIS_PORT` | `6379` |
+| `GATEWAY_REDIS_USERNAME` | empty |
+| `GATEWAY_REDIS_PASSWORD` | empty |
+| `GATEWAY_REDIS_DATABASE` | `0` |
+| `GATEWAY_REDIS_SSL` | `false` |
+| `GATEWAY_RATELIMIT_ENABLED` | `true` |
+| `GATEWAY_RATELIMIT_USER_CAPACITY` | `120` |
+| `GATEWAY_RATELIMIT_USER_PERIOD` | `1m` |
+| `GATEWAY_RATELIMIT_IP_CAPACITY` | `60` |
+| `GATEWAY_RATELIMIT_IP_PERIOD` | `1m` |
+| `GATEWAY_TRUSTED_PROXY_CIDRS` | empty |
+
+Trusted proxy CIDRs are a comma-separated list. Leave the list empty unless the gateway's direct
+network peers are controlled proxies that overwrite forwarding headers.
+
+Redis connection establishment is bounded at 300 ms and commands at 500 ms. The client reconnects
+automatically but rejects commands while disconnected. Requests fail open during those errors and
+increment `gateway.ratelimit.fail.open`.
+
+## Kafka configuration
+
+| Environment variable | Default |
+|---|---|
+| `GATEWAY_KAFKA_BOOTSTRAP` | `localhost:9092` |
+| `GATEWAY_TOPIC_ODDS_CHANGED` | `odds.changed` |
+| `GATEWAY_TOPIC_BET_SETTLED` | `bet.settled.v1` |
+| `GATEWAY_TOPIC_BET_VOIDED` | `bet.voided.v1` |
+| `GATEWAY_TOPIC_BET_RESOLUTION_REVISED` | `bet.resolution.revised.v1` |
+| `GATEWAY_KAFKA_RETRY_INTERVAL` | `1s` |
+| `GATEWAY_KAFKA_RETRY_ATTEMPTS` | `2` |
+| `GATEWAY_KAFKA_MAX_BLOCK_MS` | `5000` |
+| `GATEWAY_KAFKA_REQUEST_TIMEOUT_MS` | `5000` |
+| `GATEWAY_KAFKA_DELIVERY_TIMEOUT_MS` | `10000` |
+| `GATEWAY_KAFKA_DLT_WAIT_TIMEOUT` | `11s` |
+| `GATEWAY_KAFKA_DLT_TIMEOUT_BUFFER` | `1s` |
+
+Input topic names must be nonblank, distinct, and must not end in `.DLT`. Retry attempts may be zero
+or greater; durations must be positive.
+
+For every configured input topic, provision `<input>.DLT` with the same partition count and at least
+seven days of retention. The recoverer does not perform a separate destination-partition preflight;
+the producer sends directly to the exact source partition number. If the DLT has fewer partitions,
+publication fails and the source offset remains uncommitted. Topic auto-creation is not an
+operational substitute for explicit provisioning.
+
+The consumer groups are fixed as `gateway-odds` and `gateway-bets`. Auto-commit is disabled and the
+listener acknowledgment mode is `RECORD`. Keep the gateway deployment at one replica.
+
+## Health and metrics
+
+These anonymous Actuator endpoints are exposed:
+
+| Endpoint | Purpose |
+|---|---|
+| `/actuator/health` | Aggregate application availability |
+| `/actuator/health/liveness` | `livenessState` only |
+| `/actuator/health/readiness` | `readinessState` only |
+| `/actuator/info` | Build group, artifact, name, and version |
+| `/actuator/prometheus` | Prometheus scrape |
+
+Health responses hide components and details. The liveness and readiness groups do not include
+Redis, Kafka, betting, wallet, or odds-feed checks. Monitor those dependencies separately.
+
+All Micrometer metrics receive `service="gateway"`. The rate-limit fail-open metric is exported to
+Prometheus as `gateway_ratelimit_fail_open_total`. Alert on sustained increases because admitted
+traffic is no longer receiving distributed enforcement during that interval.
+
+Build info excludes build time and does not expose source-control metadata.
+
+## Structured logs
+
+Logs go to standard output as newline-delimited JSON. The configured fields are:
+
+- `@timestamp`
+- `level`
+- `logger_name`
+- `message`
+- `service`, fixed to `gateway`
+- `traceId` and `spanId` when present in MDC
+- `stack_trace` when an exception is attached
+
+Root and application loggers use `INFO`; the Apache Kafka logger uses `WARN`. Only `traceId` and
+`spanId` are admitted from MDC.
+
+The event provider replaces recognizable bearer tokens and values labelled as authorization,
+internal API key, API key, password, or token with `[REDACTED]` in both formatted messages and
+emitted stack traces. This is a final guard, not permission to log credentials. Avoid placing
+secrets in message arguments or exception text.
+
+## Failure response guide
+
+| Symptom | Gateway behavior | Operator action |
+|---|---|---|
+| Redis unavailable | Requests admitted; fail-open counter rises | Restore Redis and confirm the counter stops increasing |
+| Downstream connection refused | HTTP 502 problem response | Check the configured URI, network, and target process |
+| Downstream read bound exceeded | HTTP 504 problem response | Check target saturation and request processing |
+| Invalid Avro or key contract | Immediate same-partition DLT publication | Inspect raw evidence, correct the producer or data, then use controlled replay |
+| Transient event delivery failure | Two retries, then DLT | Correct the delivery path and inspect the DLT |
+| DLT unavailable | Source offset retained and record redelivered | Restore the DLT with matching partitions before resuming normal flow |
+| Authenticated WebSocket reaches `exp` | Socket closed with code 1008 | Client obtains a new token and reconnects |
+
+## DLT handling
+
+Treat a DLT record as sensitive operational evidence because its raw application headers and value
+are retained. Restrict read and produce permissions accordingly.
+
+After correcting the root cause:
+
+1. read the record from the exact DLT;
+2. verify that the target source topic and same-numbered partition exist;
+3. remove only framework recovery, exception, delivery-attempt, and deserializer-exception headers;
+4. retain the raw key, value, and application headers;
+5. publish to the tail of the paired source partition and wait for acknowledgment; and
+6. confirm normal processing without changing the gateway consumer-group offsets.
+
+`DltReplayRecordFactory` implements the record transformation but does not expose an HTTP endpoint,
+consume DLT records, or publish them automatically.
diff --git a/docs/realtime-contract.md b/docs/realtime-contract.md
new file mode 100644
index 0000000..dfbba4e
--- /dev/null
+++ b/docs/realtime-contract.md
@@ -0,0 +1,158 @@
+# Realtime Contract
+
+## STOMP endpoints
+
+The gateway exposes native WebSocket STOMP handshakes at:
+
+- `/ws/v1/odds` for public odds clients; and
+- `/ws/v1/bets` for authenticated bet-status clients.
+
+Both handshake paths pass through the same STOMP authorization policy. The default allowed origins
+are local HTTP origins on any port for `localhost` and `127.0.0.1`; deployments set
+`GATEWAY_WS_ALLOWED_ORIGINS` to the required origin patterns.
+
+Transport limits are fixed:
+
+| Limit | Value |
+|---|---:|
+| Inbound message | 64 KiB |
+| Send buffer | 512 KiB |
+| Send time | 10 s |
+
+## CONNECT authentication
+
+Authentication is carried in the STOMP `CONNECT` or `STOMP` frame as exactly one native
+`Authorization: Bearer <token>` header. The token uses the same decoder and claim validator as HTTP.
+
+The header may be omitted for an anonymous odds session. A malformed, repeated, invalid, or expired
+bearer header rejects the connection. An authenticated session is registered under the canonical
+UUID `sub` and may subscribe to its user destination.
+
+Each authenticated connection receives an expiry task for its JWT `exp`. At expiry the gateway
+closes the underlying WebSocket immediately with close code 1008. An early disconnect cancels the
+task. Anonymous sessions have no token-expiry task.
+
+## Client commands and destinations
+
+Clients may use only `CONNECT` or `STOMP`, `SUBSCRIBE`, `UNSUBSCRIBE`, and `DISCONNECT`. Every client
+`SEND` is rejected, as are other client commands.
+
+| Destination | Authentication | Rule |
+|---|---|---|
+| `/topic/odds/{eventId}` | Optional | `eventId` must be one lowercase canonical UUID segment |
+| `/user/queue/bets` | Required | Delivery is scoped to the authenticated JWT subject |
+
+No wildcard, nested, application, or alternate queue destination is accepted.
+
+## Odds update
+
+An `OddsChanged` event is projected to `/topic/odds/{eventId}` with this JSON shape:
+
+| Field | Type | Source |
+|---|---|---|
+| `eventId` | string | event identity |
+| `marketId` | string | market identity |
+| `selectionId` | string | selection identity |
+| `previousOdds` | string | previous normalized decimal odds |
+| `newOdds` | string | new normalized decimal odds |
+| `changedAt` | ISO-8601 instant | event time |
+
+The event, market, and selection identities must all be canonical UUIDs. The raw Kafka key must be
+strict UTF-8 and equal `eventId`.
+
+## Bet status update
+
+Terminal bet events are projected to `/user/queue/bets` for the event's `userId`. The JSON fields
+are:
+
+```text
+betId, userId, eventId, status, result, amount, reason,
+revisionId, revisionNumber, updatedAt
+```
+
+`amount` has the shared Money shape `{ "amount": <integer>, "currency": <code> }`.
+
+| Source event | `status` | `result` | `amount` | `reason` | `revisionId` | `revisionNumber` | `updatedAt` |
+|---|---|---|---|---|---|---:|---|
+| `BetSettled` | `SETTLED` | settlement result | payout | `null` | `null` | `0` | `settledAt` |
+| `BetVoided` | `VOIDED` | `null` | refund | void reason | `null` | `null` | `voidedAt` |
+| `BetResolutionRevised` | `SETTLED` | new result | new payout | `null` | actual revision ID | actual revision number | `revisedAt` |
+
+The raw key for settled and voided events must equal `eventId`. The raw key for a resolution
+revision must equal `betId`. All identity fields must be canonical UUIDs.
+
+`BetVoided` is limited to whole-slip lifecycle or administrative voids. The retained
+`MARKET_VOID` wire enum is rejected as a permanent contract failure: a market void is represented
+by `BetSettled` with `status=SETTLED` and `result=VOID`.
+
+A revision must have a number of at least one, matching previous and new payout currencies, and a
+`revisedAt` value that is not before `sourceResultSettledAt`.
+
+## Client state rule
+
+The gateway does not store a latest revision. Each client keeps the greatest revision number seen
+for each bet. Initial settlement is logical revision zero. Once a positive revision has been
+applied, the client ignores duplicate revisions, lower revisions, and a late revision-zero
+settlement for that bet.
+
+Voided updates have no revision number and must be handled according to the client's terminal-state
+rules.
+
+WebSocket delivery is live and nondurable. A reconnect does not replay missed messages. Consumers
+must refresh authoritative state over HTTP when continuity is uncertain.
+
+## Kafka inputs
+
+Gateway 1.0 consumes four raw Avro binary streams:
+
+| Default input | Group | Expected record | Required key |
+|---|---|---|---|
+| `odds.changed` | `gateway-odds` | `OddsChanged` | `eventId` |
+| `bet.settled.v1` | `gateway-bets` | `BetSettled` | `eventId` |
+| `bet.voided.v1` | `gateway-bets` | `BetVoided` | `eventId` |
+| `bet.resolution.revised.v1` | `gateway-bets` | `BetResolutionRevised` | `betId` |
+
+Both key and value deserializers return `byte[]`. The value must contain exactly one binary Avro
+record of the expected generated type; extra trailing bytes are a contract failure.
+
+The listener uses record acknowledgment with Kafka auto-commit disabled. A successful projection
+allows that record's offset to advance. Duplicate delivery remains possible around process or broker
+failures, so clients cannot use arrival count as business state.
+
+## Dead-letter boundary
+
+Each input has a dead-letter topic formed by appending uppercase `.DLT`:
+
+- `odds.changed.DLT`
+- `bet.settled.v1.DLT`
+- `bet.voided.v1.DLT`
+- `bet.resolution.revised.v1.DLT`
+
+Decode and event-contract failures bypass transient retries and go directly to the matching DLT.
+With the default settings, other delivery failures are retried twice at one-second intervals, for
+no more than three listener attempts before quarantine.
+
+The dead-letter producer uses raw byte serializers, `acks=all`, and Kafka idempotence. Publication
+uses the source partition and preserves the original key, value, and application headers. Recovery
+metadata includes original topic, partition, offset, consumer group, and exception information.
+
+DLT publication is fail-closed: without a separate destination-partition preflight, the producer
+sends to the exact source partition number, and a send failure is raised back to the listener
+container. The source offset is not committed, and the source record is eligible for redelivery.
+The publisher has bounded block, request, delivery, result-wait, and buffer times documented in
+[Operations](operations.md).
+
+## Manual replay
+
+There is no automatic replay consumer. After the record's cause is corrected, an operator-controlled
+tool can use `DltReplayRecordFactory` to prepare a source record:
+
+1. accept only a record from one of the four exact configured DLT names;
+2. select the paired source topic and the same partition;
+3. clone the raw key and value;
+4. preserve application headers, including duplicate and null-valued headers;
+5. remove Kafka recovery, exception, delivery-attempt, and deserializer-exception headers; and
+6. publish the result at the tail of that source partition and wait for broker acknowledgment.
+
+The factory creates the `ProducerRecord`; it does not send it. The operator must not reset the
+`gateway-odds` or `gateway-bets` consumer-group offsets.
