# DevThread 개발자 기술면접 워크북 — 마스터 인덱스

이 문서는 현재 GPT 프로젝트에 축적된 Thread 01~14의 작업 기록에서 실제로 확인 가능한 내용만 사용해 면접 포인트를 선별한 기준 문서다. 같은 역량을 반복하는 기록은 대표 Thread에 묶었고, 확인되지 않는 커밋 제목·파일·함수는 만들지 않았다.

## 우선순위 기준

- **S**: 질문과 10~30분 백지 구현 모두 가치가 높다. 반드시 준비한다.
- **A**: 질문 가치가 높고, 핵심 부분을 축소한 구현 문제도 가능하다.
- **B**: 구현보다 설계·개념·검증 전략 설명이 적합하다. S/A 대표 문제에 통합할 수 있다.
- **C**: 증거 수집, 반복 설정, 특정 harness 유지처럼 별도 면접 문제로 만들 가치가 낮다.

## 전체 Thread·커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| B | 01 | 현재 프로젝트 메모리에서 개별 커밋 메시지 미확인 — Thread 제목: E01 Minimal Monitor and Synchronous Check | `CheckRunner.requireFixtureUrl`, `CheckRunner.run`, `scripts/fixture.mjs` | 동기 HTTP 관측의 최소 실패 분류와 정리 | timeout·redirect 금지·body 미보관·disconnect는 기본기지만, Thread 12가 같은 역량을 더 강한 경계로 확장한다. | 중간 | 낮음 | 09, 12 |
| **S** | 02 | `fix: reject overflowing numeric monitor intervals` | `MonitorController.CreateMonitor.fromJson` | **SA-01 — JSON 유형·정수성·overflow·범위 검증** → [상세](01-contracts-persistence-and-transactions.md#sa-01) | Java 숫자 변환과 JSON type을 동시에 다루며, mutation 전에 실패해야 한다는 invariant가 명확하다. | 높음 | 높음 | 01, 03 |
| **A** | 02 | `fix: reject overflowing numeric monitor intervals` | `ApiErrors`, `app/monitors/api.ts` | **SA-02 — 안전한 오류 봉투와 클라이언트 런타임 검증** → [상세](01-contracts-persistence-and-transactions.md#sa-02) | 정적 타입이 네트워크 계약을 보장하지 않는다는 점과 예외 정보 비공개를 함께 묻기 좋다. | 높음 | 중간 | 06, 08, 14 |
| C | 02 | `docs: record E02 numeric overflow repair evidence` | 숫자 overflow 검증 기록; 개별 증거 파일명은 현재 메모리에서 미확인 | 수리 증거와 실행 기록 보존 | 결과 재현에는 유용하지만 핵심 역량은 SA-01에 이미 포함된다. | 낮음 | 낮음 | 02 |
| C | 03 | `API 재시작으로 사라지는 Monitor와 CheckRun 고정 반례 기록` | `PostgresPersistenceTest`의 재시작 반례 | in-memory 상태 소실을 고정하는 반례 | 영속성 도입 동기는 설명할 수 있으나 별도 구현 문제보다 SA-03의 배경으로 적합하다. | 중간 | 낮음 | 01, 03 |
| **A** | 03 | `PostgreSQL 재시작과 스키마 거부의 실제 검증 결과 기록` | `MonitorEntity.fromDomain/toDomain`, `CheckRunEntity.fromDomain/toDomain` | **SA-03 — 도메인·ORM·DB 정규형** → [상세](01-contracts-persistence-and-transactions.md#sa-03) | 시각 정밀도, nullable 결과, `0`과 `null`의 의미를 계층마다 동일하게 유지하는 일반화 가능한 문제다. | 높음 | 중간 | 09, 11 |
| **S** | 03 | `PostgreSQL 재시작과 스키마 거부의 실제 검증 결과 기록` | `MonitorStore`의 `@Transactional` 경계, `MonitorController` | **SA-04 — 짧은 트랜잭션과 외부 I/O 분리** → [상세](01-contracts-persistence-and-transactions.md#sa-04) | connection·lock을 외부 네트워크 대기 동안 점유하지 않는 원리는 이후 worker 구조 전체의 기반이다. | 높음 | 높음 | 01, 09, 10, 11 |
| B | 03 | `PostgreSQL 재시작과 스키마 거부의 실제 검증 결과 기록` | `SchemaCompatibility` | migration 성공과 runtime mapping 호환성의 차이 | startup fail-fast 설계 질문으로 좋지만 축소 구현보다 SA-03의 설명 항목으로 준비하는 편이 낫다. | 높음 | 낮음 | 04, 05 |
| **S** | 04 | `Spring Security 세션 인증과 1시간 수명주기 적용` | `AuthenticationConfig`, `SessionController`, `SessionExpiryFilter` | **SA-05 — 세션 고정 방지와 절대 만료** → [상세](02-authentication-authorization-and-browser-security.md#sa-05) | 로그인 시 session ID 회전, 명시적 SecurityContext 저장, idle·absolute expiry의 차이를 함께 검증할 수 있다. | 높음 | 높음 | 05, 08 |
| B | 04 | `Spring Security 세션 인증과 1시간 수명주기 적용` | `UserAccounts`, `BootstrapUsers`, `SessionController` | bcrypt, credential erase, logout cookie·세션 정리 | 보안 기본기 질문에는 유용하지만 독립 문제보다 SA-05의 lifecycle 꼬리 질문으로 적합하다. | 높음 | 낮음 | 05 |
| **S** | 05 | `test(authz): freeze the two-user IDOR counterexample` | `MonitorStore`, `MonitorController`, `V4__require_monitor_ownership.sql` | **SA-06 — 객체 단위 인가와 owner predicate** → [상세](02-authentication-authorization-and-browser-security.md#sa-06) | 사전 조회만으로 끝내지 않고 모든 read/update/delete에 소유권을 결합하는 핵심 보안 invariant다. | 높음 | 높음 | 03, 07, 10 |
| **A** | 05 | `test(authz): preserve E05 security gates and observation history` | `BrowserOriginFilter.doFilterInternal`, `AuthenticationConfig` | **SA-07 — CSRF와 정확한 Origin의 결합** → [상세](02-authentication-authorization-and-browser-security.md#sa-07) | cookie 기반 인증의 변경 요청 경계를 CSRF token과 source 검증 두 층으로 설명할 수 있다. | 높음 | 중간 | 04, 08 |
| C | 05 | `test(authz): preserve E05 security gates and observation history` | `OwnershipAuthorizationTest.SqlEvidence`, DB snapshot 검사 | SQL·transaction 증거 수집 harness | 보안 회귀 증거로는 좋지만 면접 핵심은 owner predicate와 외부 observable인 404 의미론이다. | 중간 | 낮음 | 03, 05 |
| B | 06 | `test(state): freeze the held-create duplicate submission` | `tests/browser/server-state.spec.ts` | held response로 중복 제출 창 재현 | 실패를 재현하는 테스트 설계는 가치가 있으나 해결 원리는 SA-08에 통합한다. | 높음 | 낮음 | 06, 10 |
| **A** | 06 | `fix(state): own monitor data and pending mutations together` | `app/monitors/use-monitor-state.ts`, 내부 `perform` | **SA-08 — 동기 in-flight 소유권과 mutation state** → [상세](03-client-state-pagination-and-rendering.md#sa-08) | React render 전에 닫히지 않는 중복 창과 서버 권위 데이터 보존을 프레임워크 밖의 상태 문제로 일반화할 수 있다. | 높음 | 중간 | 07, 10 |
| C | 06 | `test(state): preserve E06 response-barrier verification` | `tests/browser/server-state.spec.ts` | response barrier 증거 유지 | 회귀 방지에는 유용하지만 독립 면접 주제는 아니다. | 낮음 | 낮음 | 06 |
| B | 07 | `test(history): freeze E07 pagination counterexample` | `HistoryPaginationTest`, browser history 시나리오 | offset·동률·새 삽입 반례 고정 | 문제 동기를 설명하는 사례이며 해결 자체는 SA-09·SA-10에 통합한다. | 높음 | 낮음 | 07, 13 |
| **S** | 07 | `feat(history): bound owner history with stable cursors` | `HistoryQuery.parse/nextCursor`, `MonitorStore.historyPage`, `V5__index_check_history.sql` | **SA-09 — 복합 keyset pagination** → [상세](03-client-state-pagination-and-rendering.md#sa-09) | 안정 정렬, cursor context binding, limit+1, 누락·중복 방지를 한 문제에서 검증할 수 있다. | 높음 | 높음 | 05, 13 |
| **A** | 07 | `feat(history): restore filter and page state from URLs` | `app/monitors/use-monitor-state.ts`, `app/monitors/api.ts` | **SA-10 — URL 권위 상태와 stale 응답 차단** → [상세](03-client-state-pagination-and-rendering.md#sa-10) | back/forward/reload와 비동기 응답 순서가 만나는 실제 UI 경계다. | 높음 | 중간 | 06, 08 |
| C | 08 | `test(rendering): freeze phase-1 E08 SSR baseline` | `tests/browser/rendering.spec.ts` | SSR 부재·초기 중복 fetch 반례 | 다음 구현의 기준점이지만 별도 면접 문제로는 낮다. | 중간 | 낮음 | 08 |
| **A** | 08 | `feat(rendering): seed request-scoped server state for hydration` | `server-data.ts`, `loadInitialMonitorState`, `monitor-controls.tsx` | **SA-11 — 요청별 SSR 데이터와 hydration 경계** → [상세](03-client-state-pagination-and-rendering.md#sa-11) | 세션 cookie 전달, no-store, 사용자별 데이터 격리, 동일 초기 상태 재사용을 함께 묻기 좋다. | 높음 | 중간 | 04, 05, 07 |
| B | 08 | `test(rendering): verify privacy keyboard and accessibility` | `tests/browser/rendering.spec.ts`, `monitor-controls.tsx` | SSR privacy, keyboard, 접근성 회귀 | 중요한 품질 경계지만 프로젝트 특화 UI 검증 비중이 높아 설명·테스트 전략 준비가 적합하다. | 높음 | 낮음 | 05, 08 |
| **S** | 09 | `feat(worker): persist queued checks and execute interval intents` | `MonitorStore.enqueueCheck`, `CheckQueue.startNext/finish`, `CheckWorker.executeNext` | **SA-12 — durable intent와 API·worker 분리** → [상세](04-queue-idempotency-and-recovery.md#sa-12) | 202 응답 전에 의도를 commit하고 외부 I/O를 별도 process로 옮기는 시스템 경계가 명확하다. | 높음 | 높음 | 03, 10, 11 |
| **S** | 09 | `feat(worker): persist queued checks and execute interval intents` | `CheckQueue.scheduleDue`, `V6__queue_check_execution.sql` | **SA-13 — 예약 슬롯 결정성과 DB 유일성** → [상세](04-queue-idempotency-and-recovery.md#sa-13) | 중복 tick·재기동·다중 scheduler에서도 같은 slot을 하나로 만드는 invariant를 구현하기 좋다. | 높음 | 높음 | 10, 11 |
| B | 09 | `feat(worker): persist queued checks and execute interval intents` | `tests/browser/worker.spec.ts` | 별도 worker process에서 acceptance→terminal 진행 증명 | end-to-end 검증 가치가 있지만 핵심 설계는 SA-12에 포함된다. | 중간 | 낮음 | 09, 11 |
| **S** | 10 | `feat(execution): adopt preserved E10 ownership implementation` | `MonitorStore.enqueueCheck`, `V7__execution_ownership_and_manual_identity.sql` | **SA-14 — 요청 idempotency와 원자적 재사용** → [상세](04-queue-idempotency-and-recovery.md#sa-14) | 재시도·중복 전송을 in-memory flag가 아닌 유일성과 `ON CONFLICT`로 해결하는 대표 문제다. | 높음 | 높음 | 06, 09 |
| **S** | 10 | `feat(execution): adopt preserved E10 ownership implementation` | `CheckQueue.startNext/finish`, `CheckWorker` | **SA-15 — 경쟁 claim과 소유자 fencing** → [상세](04-queue-idempotency-and-recovery.md#sa-15) | `FOR UPDATE SKIP LOCKED`, 상태·소유자 동시 commit, 조건부 완료를 통해 실제 동시성 기본기를 평가한다. | 높음 | 높음 | 09, 11 |
| B | 10 | `feat(ui): adopt preserved E10 manual intent lifecycle` | `app/monitors/use-monitor-state.ts`, `tests/browser/worker.spec.ts` | 202 수락 이후 terminal refresh와 pending 범위 | 클라이언트 표현은 중요하지만 SA-08과 SA-16의 상태 의미론에 묶는 편이 낫다. | 높음 | 낮음 | 06, 11 |
| C | 10 | `test(idempotency): freeze bounded E10 repair diagnostic` | `ExecutionOwnershipTest`와 E10 진단 기록 | bounded repair 증거 | 구현 역량보다 검증 절차 기록 성격이 강하다. | 낮음 | 낮음 | 10 |
| C | 10 | `test(idempotency): send the literal frozen non-ASCII header` | `SessionClient`, idempotency header 테스트 | 비 ASCII header 전달 반례 | 유효성 검사의 한 경계값으로 SA-14에 포함할 수 있으나 독립 문제는 아니다. | 중간 | 낮음 | 10 |
| **S** | 11 | `feat: recover expired worker runs and drain shutdown` | `CheckQueue.finish/recoverExpired`, `CheckWorker.acceptingClaims`, `WorkerProcess` | **SA-16 — lease 만료·늦은 완료 차단·graceful drain** → [상세](04-queue-idempotency-and-recovery.md#sa-16) | crash 이후 소유권과 시간 fencing을 유지하고 재실행 대신 ABORTED를 택한 trade-off를 구현·설명하기 좋다. | 높음 | 높음 | 10, 14 |
| B | 11 | `fix: show aborted executions and refresh terminal history` | `app/monitors/api.ts`, `app/monitors/use-monitor-state.ts` | ABORTED를 terminal 결과로 반영하는 UI | 상태 모델의 소비 문제이며 핵심 invariant는 SA-16에 통합한다. | 중간 | 낮음 | 09, 10 |
| C | 12 | `test(e12): freeze safe outbound baseline and fixtures` | `CheckRunnerTest`, E12 fixture | SSRF·timeout 반례 동결 | 보안 테스트 설계에 유용하지만 구현 포인트는 SA-17·SA-18에 포함된다. | 중간 | 낮음 | 01, 12 |
| **S** | 12 | `feat(e12): adopt preserved outbound destination safeguards` | `OutboundUrl.canonical/requireAddresses/isPublic`, `CheckRunner` | **SA-17 — URL 정규화·DNS pinning·redirect 재검증** → [상세](05-outbound-security-and-resource-lifecycle.md#sa-17) | URL 문자열부터 실제 socket 목적지와 TLS hostname까지 이어지는 대표 SSRF 문제다. | 높음 | 높음 | 01, 09, 11 |
| **S** | 12 | `feat(e12): adopt preserved outbound destination safeguards` | `CheckRunner.run/close`, `Attempt`, 제한 상수 | **SA-18 — total deadline·bounded DNS·socket 정리** → [상세](05-outbound-security-and-resource-lifecycle.md#sa-18) | 시간·thread·queue·header·socket 상한과 cleanup을 한 문제에서 평가할 수 있다. | 높음 | 높음 | 01, 11, 14 |
| **A** | 13 | `test(e20): verify the fixed skewed history plan` | `HistoryQueryPlanTest`, `HistoryPaginationTest.SqlEvidence`, `MonitorStore.historyPage` | **SA-19 — 구조적 EXPLAIN 검증과 no-index 결정** → [상세](06-query-plans-and-production-operations.md#sa-19) | 실행 시간보다 plan 구조를 검증하고 read 이득과 write cost를 함께 판단하는 실무형 성능 질문이다. | 높음 | 중간 | 07, 03 |
| **A** | 14 | `test(e24): freeze the operations and container contract` | `AuthorityHealthIndicator`, `WorkerManagement`, `CheckWorker.acceptingClaims` | **SA-20 — liveness/readiness와 drain 분리** → [상세](06-query-plans-and-production-operations.md#sa-20) | dependency outage를 process restart와 traffic admission으로 분리하는 운영 설계의 핵심이다. | 높음 | 중간 | 09, 11 |
| **A** | 14 | `fix: bound completed HTTP observation exports` | `HttpObservations.method/route`, `ProcessObservations`, `CheckQueue.oldestQueuedAgeSeconds` | **SA-21 — low-cardinality 메트릭과 안전한 구조화 로그** → [상세](06-query-plans-and-production-operations.md#sa-21) | UUID·사용자 입력으로 시계열이 폭증하는 문제와 durable event 이후 계측이라는 의미론을 함께 묻기 좋다. | 높음 | 중간 | 05, 10, 11 |
| B | 14 | `test: add frozen phase-1 production operations observer` | `Dockerfile.api`, `Dockerfile.frontend`, `compose.production.yaml`, `scripts/e24-container.mjs` | non-root/PID 1, SIGTERM, 실제 container 경계 | 운영 설명 가치는 높지만 Docker 설정과 observer 절차가 프로젝트 특화라 SA-20의 꼬리 질문으로 둔다. | 높음 | 낮음 | 11, 14 |
| C | 14 | `test: add frozen phase-1 production operations observer` | `scripts/e24-container.mjs`, E24 evidence | 고정 seed·컨테이너 소유권·증거 정리 harness | 실제 운영 계약을 검증하는 도구지만 면접용 핵심 구현보다 검증 인프라 비중이 크다. | 낮음 | 낮음 | 14 |

## 대표 Thread와 연관 Thread 관계

| 대표 문제군 | 대표 Thread | 연관 Thread | 통합 기준 | 상세 문서 |
| --- | --- | --- | --- | --- |
| 입력·응답 계약 | 02 | 01, 06, 08, 14 | transport에서 받은 값을 신뢰하지 않고 mutation·render 전에 검증한다. | [01-contracts-persistence-and-transactions.md](01-contracts-persistence-and-transactions.md) |
| 영속성·트랜잭션 | 03 | 09, 10, 11 | 정규형과 짧은 transaction이 queue·claim·recovery의 기반이 된다. | [01-contracts-persistence-and-transactions.md](01-contracts-persistence-and-transactions.md) |
| 인증·인가·브라우저 보안 | 04, 05 | 08, 10, 14 | session identity, owner scope, mutation source를 서로 다른 경계로 유지한다. | [02-authentication-authorization-and-browser-security.md](02-authentication-authorization-and-browser-security.md) |
| 클라이언트 상태·페이지네이션 | 06, 07, 08 | 09, 10, 11 | pending ownership, URL selection, cursor, SSR seed가 오래된 응답의 덮어쓰기를 막는다. | [03-client-state-pagination-and-rendering.md](03-client-state-pagination-and-rendering.md) |
| 작업 큐·동시성·복구 | 09, 10, 11 | 03, 06, 14 | durable intent → unique identity → exclusive claim → lease fencing 순으로 invariant를 강화한다. | [04-queue-idempotency-and-recovery.md](04-queue-idempotency-and-recovery.md) |
| 외부 통신 보안·자원 | 12 | 01, 09, 11, 14 | URL policy와 actual connection, total budget, shutdown cleanup을 하나의 outbound boundary로 본다. | [05-outbound-security-and-resource-lifecycle.md](05-outbound-security-and-resource-lifecycle.md) |
| 성능·운영 관측 | 13, 14 | 07, 10, 11 | timing 수치보다 plan 구조, service admission, bounded cardinality와 durable event 의미를 검증한다. | [06-query-plans-and-production-operations.md](06-query-plans-and-production-operations.md) |

## 상세 워크북 파일 지도

| 파일 | 포함 면접 포인트 | 역할 |
| --- | --- | --- |
| [01-contracts-persistence-and-transactions.md](01-contracts-persistence-and-transactions.md) | SA-01~SA-04 | 입력 type/overflow, 안전한 오류 계약, 정규형 mapping, transaction과 I/O 분리 |
| [02-authentication-authorization-and-browser-security.md](02-authentication-authorization-and-browser-security.md) | SA-05~SA-07 | session lifecycle, IDOR 방지, CSRF와 Origin 경계 |
| [03-client-state-pagination-and-rendering.md](03-client-state-pagination-and-rendering.md) | SA-08~SA-11 | 중복 mutation 창, keyset cursor, URL state, request-scoped SSR/hydration |
| [04-queue-idempotency-and-recovery.md](04-queue-idempotency-and-recovery.md) | SA-12~SA-16 | durable queue, schedule uniqueness, idempotency, claim fencing, lease recovery |
| [05-outbound-security-and-resource-lifecycle.md](05-outbound-security-and-resource-lifecycle.md) | SA-17~SA-18 | SSRF 방어, DNS pinning, total deadline, bounded executor와 socket cleanup |
| [06-query-plans-and-production-operations.md](06-query-plans-and-production-operations.md) | SA-19~SA-21 | 구조적 query-plan 검증, readiness/liveness, bounded metrics와 structured logs |

## S/A 완전성 검증

| 면접 포인트 | 우선순위 | 상세 위치 | 상태 |
| --- | --- | --- | --- |
| SA-01 | S | [01 / SA-01](01-contracts-persistence-and-transactions.md#sa-01) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-02 | A | [01 / SA-02](01-contracts-persistence-and-transactions.md#sa-02) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-03 | A | [01 / SA-03](01-contracts-persistence-and-transactions.md#sa-03) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-04 | S | [01 / SA-04](01-contracts-persistence-and-transactions.md#sa-04) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-05 | S | [02 / SA-05](02-authentication-authorization-and-browser-security.md#sa-05) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-06 | S | [02 / SA-06](02-authentication-authorization-and-browser-security.md#sa-06) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-07 | A | [02 / SA-07](02-authentication-authorization-and-browser-security.md#sa-07) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-08 | A | [03 / SA-08](03-client-state-pagination-and-rendering.md#sa-08) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-09 | S | [03 / SA-09](03-client-state-pagination-and-rendering.md#sa-09) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-10 | A | [03 / SA-10](03-client-state-pagination-and-rendering.md#sa-10) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-11 | A | [03 / SA-11](03-client-state-pagination-and-rendering.md#sa-11) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-12 | S | [04 / SA-12](04-queue-idempotency-and-recovery.md#sa-12) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-13 | S | [04 / SA-13](04-queue-idempotency-and-recovery.md#sa-13) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-14 | S | [04 / SA-14](04-queue-idempotency-and-recovery.md#sa-14) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-15 | S | [04 / SA-15](04-queue-idempotency-and-recovery.md#sa-15) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-16 | S | [04 / SA-16](04-queue-idempotency-and-recovery.md#sa-16) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-17 | S | [05 / SA-17](05-outbound-security-and-resource-lifecycle.md#sa-17) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-18 | S | [05 / SA-18](05-outbound-security-and-resource-lifecycle.md#sa-18) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-19 | A | [06 / SA-19](06-query-plans-and-production-operations.md#sa-19) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-20 | A | [06 / SA-20](06-query-plans-and-production-operations.md#sa-20) | 독립적인 상세 워크북 항목으로 작성됨 |
| SA-21 | A | [06 / SA-21](06-query-plans-and-production-operations.md#sa-21) | 독립적인 상세 워크북 항목으로 작성됨 |

대조 결과: S 12개와 A 9개, 총 21개가 모두 상세 문서에 연결되어 있다. 상세 문서 하나에는 2~5개 항목만 배치했으며, B/C 항목은 독립 문제로 늘리지 않고 대표 항목의 꼬리 질문·원본 확인 위치 또는 제외 근거로 남겼다.

## 백지 구현 우선순위

1. SA-15 경쟁 worker claim과 소유자 조건부 완료
2. SA-16 lease 만료·늦은 완료 차단·graceful drain
3. SA-17 URL 정규화·DNS pinning·redirect 재검증
4. SA-18 total deadline·bounded DNS 실행기·socket cleanup
5. SA-14 요청 idempotency와 원자적 재사용
6. SA-09 복합 정렬 keyset cursor
7. SA-04 짧은 transaction과 외부 I/O 분리
8. SA-06 owner predicate 기반 객체 단위 인가
9. SA-13 예약 슬롯 계산과 database uniqueness
10. SA-01 JSON type·정수성·overflow 검증
11. SA-05 session rotation과 absolute expiry
12. SA-12 durable execution intent와 worker 경계
13. SA-19 EXPLAIN JSON 구조 분석기
14. SA-08 동기 in-flight registry와 mutation state
15. SA-20 liveness/readiness 판정 함수
16. SA-21 bounded HTTP observation policy
17. SA-03 domain/entity/database 정규형 mapper
18. SA-07 CSRF·Origin 변경 요청 정책
19. SA-10 URL selection과 stale response guard
20. SA-11 request-scoped SSR 초기 상태 loader
21. SA-02 성공·오류 envelope runtime decoder

## 설명 연습 우선순위

1. SA-17: 검증한 IP로 연결하면서 HTTPS hostname을 보존하는 이유
2. SA-16: lease, database clock, row lock, ABORTED no-replay trade-off
3. SA-15: `SKIP LOCKED`와 claim owner가 각각 보장하는 범위
4. SA-18: 단계별 timeout과 하나의 total deadline 차이
5. SA-14: idempotency key가 같은 Monitor에서는 재사용되고 다른 Monitor에서는 충돌해야 하는 이유
6. SA-20: dependency 장애를 liveness가 아니라 readiness에 반영하는 이유
7. SA-21: metric tag cardinality와 post-commit 계측 의미론
8. SA-19: timing 임계값 대신 plan 구조를 검증하고 새 인덱스를 만들지 않은 판단
9. SA-04: 외부 I/O 동안 transaction을 열어 두지 않는 이유
10. SA-06: foreign ID와 nonexistent ID를 같은 404로 처리하면서도 모든 SQL에 owner 조건이 필요한 이유
11. SA-05: idle timeout과 absolute expiry, session fixation 방지의 차이
12. SA-09: `(finishedAt, id)` 복합 cursor가 동률·새 삽입에서 누락을 막는 방식
13. SA-12: 202 응답 전에 durable intent를 commit해야 하는 이유
14. SA-13: scheduler의 중복 실행을 application flag가 아니라 unique slot으로 막는 이유
15. SA-11: request cookie를 전달하되 사용자 데이터를 공유 cache에 넣지 않는 이유
16. SA-10: URL을 선택 상태의 권위로 두고 오래된 응답을 거부하는 방식
17. SA-03: 첫 API 응답부터 database round-trip 정규형과 일치시켜야 하는 이유
18. SA-07: CSRF token과 exact Origin을 함께 검사하는 방어 계층
19. SA-08: React state 반영 전의 중복 제출 창과 synchronous guard
20. SA-01: JSON 숫자 node와 Java 숫자 범위를 별도로 검증하는 이유
21. SA-02: TypeScript type assertion 대신 runtime response validation이 필요한 이유

## 한 문제로 통합한 Thread 묶음

- **외부 HTTP 경계:** Thread 01의 fixture-only 동기 Check를 Thread 12의 SA-17·SA-18로 통합했다. Thread 09의 worker I/O, Thread 11의 shutdown, Thread 14의 readiness는 연관 경계로만 연결했다.
- **영속 실행 수명주기:** Thread 03의 짧은 transaction을 기반으로 Thread 09의 durable intent·schedule, Thread 10의 idempotency·claim, Thread 11의 lease recovery를 SA-12~SA-16의 연속 문제군으로 통합했다.
- **브라우저 mutation 안전성:** Thread 04의 session, Thread 05의 owner/CSRF/Origin, Thread 06의 중복 제출, Thread 10의 request idempotency를 서로 다른 보안·상태 계층으로 분리하되 중복 역량은 SA-05~SA-08·SA-14에 묶었다.
- **이력 탐색과 성능:** Thread 07의 keyset pagination을 기능 대표로, Thread 13의 실행계획 검증을 성능 대표로 두어 SA-09와 SA-19로 분리했다. 같은 index·limit+1 근거는 상호 연관 위치로 연결했다.
- **클라이언트 선택 상태:** Thread 06의 operation authority, Thread 07의 URL·cursor state, Thread 08의 SSR seed, Thread 11의 terminal refresh를 SA-08·SA-10·SA-11·SA-16에 통합했다.
- **운영 가능성:** Thread 11의 graceful drain과 Thread 14의 readiness·worker active 관측을 SA-20으로 연결하고, committed claim/recovery의 관측 의미는 SA-21에 통합했다.
- **증거·harness 커밋:** 각 Thread의 baseline, frozen counterexample, repair diagnostic, observer·root evidence는 B/C로 낮추고 해당 대표 S/A 문제의 경계 조건과 자가 검증 항목에만 반영했다.
