# 개발자 기술면접 워크북 마스터 인덱스

이 인덱스는 현재 GPT 프로젝트에 축적된 DevThread 01~23 문서와 커밋 작업 기록에서 **실제로 확인된 정보만** 사용해 작성했다. 동일 역량이 여러 Thread에서 반복되면 대표 지점 하나를 상세 문제로 삼고 나머지는 연관 Thread로 묶었다. 파일·함수·클래스·컴포넌트 이름은 기록에서 확인된 경우에만 적었다.

우선순위 기준은 다음과 같다.

- **S**: 질문과 10~30분 직접 구현 모두 반드시 준비할 가치가 높다.
- **A**: 질문 가치가 높고, 축소된 핵심 구현도 충분히 나올 수 있다.
- **B**: 직접 구현보다 설계 선택·trade-off·운영 의미를 설명하는 준비가 중요하다.
- **C**: 반복 적용이나 화면·설정 wiring 비중이 커 별도 면접 항목으로 분리할 필요가 낮다.

선별 결과는 **S 11개, A 14개, B 11개, C 2개**다. 상세 워크북은 S/A 25개만 작성했다.

## 전체 Thread·커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread | 상세 워크북 |
|---|---|---|---|---|---|---|---|---|---|
| S | 09 | `feat(game): 서버 주도 퐁 물리 갱신`<br>`test(game): 결정적 simulation 검증`<br>`test(game): versioned match replay fixture 추가` | `apps/api/src/game/pongSimulation.ts`<br>`PongSimulation`, `step`, `initialState`, `cloneState`<br>`apps/api/src/game/fixtures/replay-v1.json` | **IM-16 · 서버 권위형 결정적 시뮬레이션** | 순수 상태 전이, 충돌·득점 경계, 동일 입력 재현성은 언어 기본기·상태 invariant·테스트 설계를 한 번에 확인한다. | 높음 | 높음 | 10, 13, 14 | [04](04-simulation-input-and-ai.md) |
| S | 15 | `feat(db): 경기 확정 command 계약 정의`<br>`feat(db): PostgreSQL 경기 결과 중복 생성을 차단`<br>`refactor(game): 경기 결과 확정 boundary 사용` | `packages/db/migrations/003_match_finalization.sql`<br>`packages/db/src/index.ts`<br>`FinalizeMatchCommand`, `finalizeMatch`, `assertFinalizeMatchCommand` | **IM-07 · 멱등한 경기 확정과 원자적 부수 효과** | 결과 행, rating, 이력, 토너먼트 연결을 하나의 트랜잭션과 idempotency key로 묶는 핵심 데이터 정합성 문제다. | 높음 | 높음 | 07, 16, 21, 22 | [02](02-database-invariants-and-lifecycle.md) |
| S | 12 | `feat(game): 게임 방 상태를 RoomSession에 연결`<br>`test(game): reconnect 복구 동작 검증` | `apps/api/src/game/roomSession.ts`<br>`RoomSession`<br>`apps/api/src/gameHub.reconnect.test.ts` | **IM-11 · 경기 방 상태 머신과 재접속 유예** | 대기·진행·일시정지·재접속·몰수패 전이를 명시해 중복 종료와 타이머 누수를 막는 전형적인 lifecycle 문제다. | 높음 | 높음 | 07, 22 | [03](03-realtime-state-and-resources.md) |
| S | 13 | `test(game): snapshot replacement와 congestion 검증`<br>`fix(game): callback 지연을 snapshot congestion으로 오판하지 않음` | `apps/api/src/game/latestSnapshotBuffer.ts`<br>`LatestSnapshotBuffer`<br>`apps/api/src/game/latestSnapshotBuffer.test.ts` | **IM-14 · 최신 상태 전송과 WebSocket backpressure** | 느린 소비자에서 메모리를 제한하고 오래된 snapshot을 버리되 실제 socket 혼잡과 callback 지연을 구분해야 한다. | 높음 | 높음 | 12, 21, 23 | [03](03-realtime-state-and-resources.md) |
| S | 14 | `feat(game): room별 input sequence 중복을 차단`<br>`feat(game): 입력 순서와 rate limit 보호` | `apps/api/src/game/inputGate.ts`<br>`InputGate`, `InputGateDecision`<br>`apps/web/src/game/gameInput.ts` | **IM-17 · 입력 순서 보장과 token bucket** | 재전송·역순 입력·스팸을 하나의 경계에서 처리하며, stale 입력이 rate-limit 자원을 소모하지 않는 순서까지 설명할 수 있다. | 높음 | 높음 | 08, 09, 12 | [04](04-simulation-input-and-ai.md) |
| S | 11 | `refactor(game): rating 기반 closest-pair queue 구현`<br>`refactor(game): AI fallback과 reservation lifecycle 구현` | `apps/api/src/game/matchmaker.ts`<br>`Matchmaker`, `findClosestOpponent`, `MATCHMAKER_AI_FALLBACK_MS` | **IM-19 · closest-pair 매칭과 예약 수명주기** | 자료구조·탐색 복잡도뿐 아니라 예약 중복, fallback 경쟁, disconnect 정리까지 함께 검증할 수 있다. | 높음 | 높음 | 10, 12, 16 | [05](05-matchmaking-finalization-and-tournaments.md) |
| S | 06 | `feat(db): PostgreSQL WebSocket ticket 저장 추가`<br>`feat(auth): ticket 기반 WebSocket 인증 연결`<br>`test(auth): WebSocket ticket 경계 검증` | `packages/db/migrations/002_ws_tickets.sql`<br>`packages/db/src/index.ts` · `createWsTicket`, `consumeWsTicket`<br>`apps/api/src/wsTicket.ts` | **IM-03 · 일회용 WebSocket 티켓 인증** | 브라우저 WebSocket 제약, 비밀값 저장, TTL, 원자적 단일 소비, 인증 전 버퍼 제한을 모두 포함하는 보안 경계다. | 높음 | 높음 | 03, 07, 08 | [01](01-contracts-auth-and-security.md) |
| S | 13 | `feat(game): fixed-step scheduler 추가`<br>`feat(game): fixed-step scheduler를 GameHub에 연결` | `apps/api/src/game/fixedStepScheduler.ts`<br>`FixedStepAccumulator`, `FixedStepScheduler` | **IM-13 · 고정 시간 간격 실행과 지연 누적 제한** | event loop 지연을 시뮬레이션 시간과 분리하고 catch-up 상한으로 spiral of death를 막는 런타임 기본기다. | 높음 | 높음 | 09, 21, 23 | [03](03-realtime-state-and-resources.md) |
| S | 17 | `feat(db): memory friendship invariant 적용`<br>`test(db): friendship와 tournament 경쟁 상태 검증` | `packages/db/migrations/004_friendship_tournament_invariants.sql`<br>`packages/db/src/index.ts` · `requestFriend`, `acceptFriend`, `listFriends` | **IM-08 · 방향 없는 관계의 정규화와 멱등 전이** | 정규화된 pair key, self 관계 차단, 역방향 요청 병합, concurrent retry를 일반적인 관계 모델링 문제로 확장할 수 있다. | 높음 | 높음 | 05, 19 | [02](02-database-invariants-and-lifecycle.md) |
| S | 04 | `feat(db): test database reset target guard 추가`<br>`feat(db): test schema reset과 migration 실행 연결`<br>`test(db): test database reset guard 검증` | `packages/db/src/testReset.ts`<br>`resolveTestResetTarget`, `resetTestDatabase`<br>`packages/db/src/postgres.integration.test.ts` | **IM-06 · 파괴적 테스트 DB 작업의 다중 안전장치** | 환경 변수 하나에 의존하지 않고 DB 이름·schema·connection option을 교차 검증하는 방어적 I/O 설계다. | 높음 | 높음 | 21, 23 | [02](02-database-invariants-and-lifecycle.md) |
| S | 16·17 | `feat(db): PostgreSQL tournament 참가를 원자화`<br>`test(db): friendship와 tournament 경쟁 상태 검증` | `packages/db/src/index.ts` · `joinTournament`, `ensureFinalMatch`<br>`packages/db/src/postgres.integration.test.ts` | **IM-09 · 마지막 자리 경쟁과 대진표 단일 생성** | check-then-insert 경쟁을 제거하고 정원·seed·대진표 단일 생성 invariant를 DB와 트랜잭션으로 보장한다. | 높음 | 높음 | 11, 15 | [02](02-database-invariants-and-lifecycle.md) |
| A | 02 | `feat(api): typed HTTP 오류 boundary 추가`<br>`feat(shared): 모든 HTTP request schema를 strict하게 정의` | `packages/shared/src/http.ts` · `defineHttpRequestContract`, `jsonHttpRequestContracts`<br>`apps/api/src/httpBoundary.ts` · `ApiHttpError`, `parseHttpRequest` | **IM-01 · 런타임 HTTP 계약과 오류 경계** | TypeScript 타입 소거 이후의 입력을 검증하고 안정적 오류 envelope로 변환하는 API 경계 설계다. | 높음 | 중간 | 03, 06, 18 | [01](01-contracts-auth-and-security.md) |
| A | 03 | `feat(protocol): versioned WebSocket event codec 연결` | `packages/shared/src/ws.ts`<br>`clientEventSchema`, `serverEventSchema`, `parseClientEvent`, `parseServerEvent`, `encodeServerEvent` | **IM-02 · 버전이 있는 WebSocket 이벤트 프로토콜** | 양방향 메시지의 discriminated union, strict validation, 호환성 실패 정책을 설명하거나 작은 codec을 구현하기 좋다. | 높음 | 중간 | 02, 06, 14, 18 | [01](01-contracts-auth-and-security.md) |
| A | 07 | `fix(db): 차단 감사 기록을 원자적으로 저장`<br>`test(auth): 계정 정지의 기존 WebSocket 차단 검증` | `packages/db/src/index.ts` · `setUserBan`<br>`apps/api/src/gameHub.ts` · `GameHub.revokeUser` | **IM-04 · 원자적 감사 기록과 실시간 권한 회수** | DB 상태와 감사 로그의 원자성, 이미 연결된 세션의 즉시 폐기, room·queue 자원 정리를 연결하는 인가 문제다. | 높음 | 중간 | 06, 12, 15 | [01](01-contracts-auth-and-security.md) |
| A | 08 | `feat(guest): guest resource lease 수명주기 추가`<br>`test(guest): 위조 client address 거부` | `apps/api/src/guestAccess.ts`<br>`GuestAccess`, `GuestAccessError`<br>`apps/api/src/guest-demo.test.ts` | **IM-05 · 서명된 게스트 신원과 교체 가능한 lease** | TTL, HMAC, per-IP/global 제한, connection replacement 시 generation 기반 release를 다루는 보안·리소스 문제다. | 높음 | 중간 | 06, 11, 12 | [01](01-contracts-auth-and-security.md) |
| A | 18 | `fix(game): 매치 채팅의 좌석과 audience 검증`<br>`test(game): 타 경기방 채팅 주입 차단 검증` | `packages/shared/src/ws.ts`<br>`packages/db/src/index.ts` · `assertChatRoom`, `createChatMessage`<br>`apps/api/src/gameHub.chat.test.ts` | **IM-10 · 다층 채팅 scope와 audience 검증** | payload 형태, 저장 invariant, 현재 방 좌석, broadcast audience를 서로 다른 신뢰 경계에서 반복 검증한다. | 높음 | 중간 | 02, 03, 12 | [02](02-database-invariants-and-lifecycle.md) |
| A | 12 | `feat(game): WebSocket heartbeat 추가`<br>`feat(game): 사용자별 active connection 교체` | `apps/api/src/game/heartbeat.ts` · `ConnectionHeartbeat`<br>`apps/api/src/gameHub.ts` · `clientsByUser` | **IM-12 · 연결 생존성과 단일 active connection** | half-open connection 탐지, 타이머 idempotency, 새 연결이 이전 연결을 대체할 때의 소유권·정리 순서를 묻기 좋다. | 높음 | 중간 | 07, 08 | [03](03-realtime-state-and-resources.md) |
| A | 22 | `feat(game): 새 작업 차단과 active room drain 추가`<br>`feat(ops): graceful shutdown 절차 추가`<br>`fix(runtime): container 종료 유예를 room drain과 정렬` | `apps/api/src/gameHub.ts` · `beginDrain`, `close`, `finishDrain`<br>`apps/api/src/gracefulShutdown.ts` · `installGracefulShutdown`<br>`docker-compose.yml` | **IM-15 · drain과 graceful shutdown** | 신규 작업 차단, 진행 중 작업 대기, signal 중복, 상한 초과 강제 종료, 플랫폼 유예 시간 정렬을 설명해야 한다. | 높음 | 중간 | 12, 15, 21 | [03](03-realtime-state-and-resources.md) |
| A | 10 | `refactor(game): 결정적 정수 난수 생성기 추가`<br>`refactor(game): rating 기반 Pong AI 정책 분리` | `apps/api/src/game/pongAi.ts`<br>`SeededIntegerPrng`, `PongAi`, `PongAiSnapshot`, `profileFor`, `predictedBallY` | **IM-18 · 재현 가능한 PRNG와 난이도 정책** | 난수 상태 snapshot, 동일 seed 재현, rating→정책 매핑과 예측 오차를 작은 구현 문제로 만들 수 있다. | 중간 | 높음 | 09, 11 | [04](04-simulation-input-and-ai.md) |
| A | 15 | `refactor(game): 경기 결과 확정 boundary 사용` | `apps/api/src/gameHub.ts`<br>`finishRoom`, `finalizeRoom`, room의 `finishing` 상태 | **IM-20 · 단일 in-flight 확정과 재시도** | 여러 종료 신호를 하나의 Promise로 합치고 저장 실패 시 상태를 되돌려 retry 가능하게 만드는 비동기 single-flight 패턴이다. | 높음 | 중간 | 13, 16, 21, 22 | [05](05-matchmaking-finalization-and-tournaments.md) |
| A | 16 | `feat(tournament): 대진 경기 schema 추가` | `packages/db/src/index.ts` · `getTournamentMatch`, `startTournamentMatch`, `completeTournamentMatch`<br>`apps/api/src/gameHub.ts` · `joinTournamentMatch`, `leaveTournamentWaiters` | **IM-21 · 토너먼트 예약과 실패 보상** | DB에서 경기 자원을 예약한 뒤 room 생성이 실패하면 보상해야 하는 짧은 saga와 소유권 전이 문제다. | 높음 | 중간 | 11, 15, 17 | [05](05-matchmaking-finalization-and-tournaments.md) |
| A | 20 | `refactor(web): query key와 retry 정책 정의`<br>`refactor(web): session query와 cache invalidation 추가`<br>`test(web): query cache key·retry·invalidation 검증` | `apps/web/src/lib/query.ts` · `queryKeys`, `mutationInvalidations`, `expireSession`<br>`apps/web/src/components/QueryProvider.tsx` | **IM-22 · 서버 상태 캐시 정합성과 세션 만료** | exact invalidation, retry 분류, in-flight 응답과 사용자 전환 사이의 데이터 혼합을 다루는 프런트엔드 상태 문제다. | 높음 | 중간 | 02, 19 | [06](06-cache-observability-and-faults.md) |
| A | 21 | `feat(db): migration set 상태 검사 추가`<br>`feat(db): repository readiness 경계 추가`<br>`feat(ops): liveness와 readiness endpoint 추가` | `packages/db/src/migrator.ts` · `compareMigrationSets`, `inspectMigrationSet`<br>`packages/db/src/index.ts` · `checkReadiness`<br>`apps/api/src/app.ts` | **IM-23 · liveness, readiness, migration 집합 비교** | 프로세스 생존과 트래픽 수용 가능성을 구분하고 pending·diverged migration을 단순 count가 아닌 집합으로 판정한다. | 높음 | 중간 | 04, 22, 23 | [06](06-cache-observability-and-faults.md) |
| A | 21 | `feat(metrics): repository operation 측정 추가`<br>`fix(db): idle connection pool 오류에서 복구`<br>`test(db): 안전한 connection pool 오류 처리 검증` | `apps/api/src/observability.ts` · `ApiMetrics`, `instrumentRepository`<br>`packages/db/src/poolError.ts` · `installPostgresPoolErrorHandler` | **IM-24 · 비동기 계측과 실패해도 안전한 오류 경계** | sync throw와 rejected Promise를 모두 계측하고, EventEmitter 오류와 reporter 실패가 프로세스 밖으로 새지 않게 해야 한다. | 높음 | 중간 | 02, 22, 23 | [06](06-cache-observability-and-faults.md) |
| A | 23 | `test(load): fault recovery 검사 자동화`<br>`test(load): fault scenario 설정과 report 검증` | `tests/load/fault-scenario.mjs` · `createFaultScenarioConfig`, `runFaultScenario`, `formatFaultReport`<br>`tests/load/toxiproxy-control.mjs` | **IM-25 · 안전한 장애 주입과 복구 판정** | 실험 대상을 loopback으로 제한하고 baseline→장애→복구를 관측하며 cleanup을 반드시 수행하는 운영 테스트 설계다. | 높음 | 중간 | 21, 22 | [06](06-cache-observability-and-faults.md) |
| B | 01 | `build(shared): production package artifact 구성`<br>`build(db): production package artifact 구성`<br>`build(app): API와 Web production artifact 구성` | `package.json`, `tsconfig.base.json`<br>`packages/shared/tsconfig.build.json`, `packages/db/tsconfig.build.json`, `apps/api/tsconfig.build.json`<br>`apps/web/next.config.mjs`, `tests/build-artifacts.mjs` | **IM-26 · TypeScript 모노레포의 source/runtime artifact 경계** | NodeNext ESM, export map, 빌드 순서와 standalone 산출물 설명 가치는 높지만 제한 시간 내 직접 구현보다는 설계 설명에 가깝다. | 중간 | 낮음 | 22, 23 | — |
| B | 04 | `refactor(db): SQL migration lifecycle 분리`<br>`refactor(db): migration과 seed CLI 연결`<br>`fix(api): startup seed 생성을 제거` | `packages/db/src/migrator.ts` · `SqlMigrationProvider`, `migrateDatabase`<br>`packages/db/src/cli.ts` | **IM-27 · migration·seed·startup 책임 분리** | 운영 lifecycle과 재현성 설명에는 좋지만 핵심 알고리즘보다 배포 절차와 책임 배치가 중심이다. | 중간 | 낮음 | 01, 21, 22 | — |
| B | 05 | `refactor(db): dashboard와 friendship 조회 경계 정렬`<br>`test(db): database row mapping contract 검증` | `packages/db/src/rowMappers.ts`<br>`toPublicUser`, `toSessionUser`, `toMatchSummary`, `toFriendSummary`, `toTournamentSummary` | **IM-28 · PostgreSQL·memory 저장소의 canonical mapping** | port-adapter parity와 표현 계층 분리는 설명 가치가 있으나 mapper 자체는 반복 구현 비중이 크다. | 중간 | 낮음 | 04, 15, 17, 19 | — |
| C | 08 | `feat(web): guest login API와 middleware 연결`<br>`feat(web): guest play presentation 적용` | `apps/web/src/lib/demoPolicy.ts`<br>`apps/web/src/middleware.ts`<br>`apps/web/src/lib/demoPolicy.test.ts` | **IM-29 · 체험 모드 표시·탐색 정책** | 서버 격리 정책을 UI에 일관되게 반영하는 작업이지만, 별도 면접 항목으로는 화면 분기와 정책 wiring 비중이 높다. | 낮음 | 낮음 | 06, 08 | — |
| B | 10 | `feat(db): rating 구간별 NPC 상대 저장`<br>`feat(game): NPC 상대를 경기 방에 연결` | `packages/db/src/index.ts` · `listNpcOpponents`<br>`apps/api/src/gameHub.ts` | **IM-30 · rating 구간별 NPC 데이터와 방 연결** | 데이터 모델과 게임 orchestration 설명은 가능하지만 핵심 판단은 IM-18의 결정적 AI 정책에 통합하는 편이 낫다. | 중간 | 낮음 | 09, 11 | — |
| B | 12 | `refactor(web): game connection 상태 reducer 분리`<br>`refactor(play): 자동 경기 진입을 connection hook으로 전환`<br>`fix(web): 중단된 game reconnect 복구` | `apps/web/src/game/gameConnection.ts`<br>`apps/web/src/game/useGameConnection.ts`<br>`apps/web/src/game/GameSocketClient.ts` | **IM-31 · 프런트엔드 연결 reducer와 hook 경계** | 상태 전이와 side effect 분리 설명은 중요하지만 핵심 lifecycle 질문은 서버의 RoomSession 항목이 더 대표적이다. | 중간 | 낮음 | 11, 13, 14 | — |
| B | 13 | `refactor(web): PongCanvas snapshot state 렌더링` | `apps/web/src/components/PongCanvas.tsx` | **IM-32 · snapshot 보간과 렌더 cadence** | 클라이언트 부드러운 표시의 trade-off는 설명할 가치가 있으나 이 프로젝트의 핵심 권위와 성능 문제는 IM-13·14가 대표한다. | 중간 | 낮음 | 09, 12 | — |
| B | 18 | `feat(lobby): 실시간 로비 지표 API 추가`<br>`feat(lobby): 연결 중인 WebSocket 사용자 목록 추가` | `apps/api/src/gameHub.ts` · `liveStats`, `onlinePlayers`, `broadcastPresence`<br>`packages/shared/src/http.ts` · `LobbyStats` | **IM-33 · in-memory presence와 최근 대기시간 통계** | 실시간 상태의 원천과 표본 창을 설명할 수 있지만 직접 구현 문제로는 프로젝트 특화 비중이 높다. | 중간 | 낮음 | 11, 12, 21 | — |
| B | 19 | `feat(db): 순위 조회 구현`<br>`feat(web): 플레이어 대시보드 구현`<br>`fix(dashboard): 빈 rating history를 정확히 표시` | `packages/db/src/index.ts` · `listRecentMatches`, `getDashboard`, `listLeaderboard`, `bestWinningStreak`, `percentage`<br>`apps/web/src/app/dashboard/page.tsx` | **IM-34 · 전적·순위·대시보드 read model** | 정렬·필터·연승 계산·빈 이력 처리 설명은 유효하지만 핵심 invariant나 동시성보다 읽기 모델 조립 비중이 크다. | 중간 | 낮음 | 05, 15, 20 | — |
| C | 20 | `refactor(web): dashboard와 leaderboard를 query cache로 전환`<br>`refactor(web): tournament 조회와 mutation을 query cache로 전환` | `apps/web/src/app/dashboard/page.tsx`<br>`apps/web/src/app/leaderboard/page.tsx`<br>`apps/web/src/app/tournaments/page.tsx` | **IM-35 · 화면별 query cache 전환 반복 작업** | 핵심 판단은 IM-22의 key·retry·무효화 정책에 이미 포함되며 화면별 변환은 반복 적용에 가깝다. | 낮음 | 낮음 | 19, 20 | — |
| B | 21 | `feat(metrics): runtime gauge registry 추가`<br>`feat(metrics): event-loop lag 측정 추가` | `apps/api/src/observability.ts` · `ApiMetrics`<br>`apps/api/src/app.ts` · `/metrics` | **IM-36 · 메트릭 taxonomy와 저카디널리티 label** | 운영 설계 설명은 중요하지만 계측 wrapper와 안전한 오류 경계는 IM-24가 더 구체적인 구현 포인트다. | 높음 | 낮음 | 13, 23 | — |
| B | 22 | `build(docker): production API image 구성`<br>`build(docker): production Web image 구성`<br>`test(docker): production container contract 검증`<br>`fix(config): production에서 영속 저장소 요구` | `apps/api/Dockerfile`, `apps/web/Dockerfile`<br>`docker-compose.yml`, `Caddyfile`<br>`apps/api/src/env.ts` · `readEnv`<br>`tests/docker-production.test.mjs` | **IM-37 · production container·network·persistence 계약** | non-root, one-shot migration, internal metrics, secret 요구, 영속 저장소 같은 배포 판단을 설명하기 좋지만 코드 구현형은 아니다. | 높음 | 낮음 | 01, 21, 23 | — |
| B | 23 | `ci(repo): typecheck·unit·build workflow 추가`<br>`test(db): PostgreSQL integration 환경과 계약 추가`<br>`ci(repo): 정적 계약 검사 실행` | `.github/workflows/ci.yml`<br>`packages/db/src/postgres.integration.test.ts`<br>`tests/ci-contract.test.mjs`, `tests/load/load-harness.test.mjs` | **IM-38 · 계층형 검증과 CI 분리** | unit·integration·smoke·browser·load·fault 층의 목적과 비용을 설명하는 항목이며 개별 테스트 구현은 다른 대표 문제에 흡수된다. | 높음 | 낮음 | 04, 21, 22 | — |

## 상세 워크북 파일 안내

| 파일 | 포함 항목 | 역할 |
|---|---|---|
| [01-contracts-auth-and-security.md](01-contracts-auth-and-security.md) | IM-01~05 | HTTP·WebSocket 계약, 일회용 인증 티켓, 계정 정지, 게스트 신원·lease |
| [02-database-invariants-and-lifecycle.md](02-database-invariants-and-lifecycle.md) | IM-06~10 | 파괴적 DB 작업 보호, 멱등 트랜잭션, 관계·정원 경쟁, 채팅 권한 |
| [03-realtime-state-and-resources.md](03-realtime-state-and-resources.md) | IM-11~15 | room 상태 머신, heartbeat, fixed-step, backpressure, graceful drain |
| [04-simulation-input-and-ai.md](04-simulation-input-and-ai.md) | IM-16~18 | 결정적 시뮬레이션, ordered input gate, 재현 가능한 PRNG·AI |
| [05-matchmaking-finalization-and-tournaments.md](05-matchmaking-finalization-and-tournaments.md) | IM-19~21 | matchmaking reservation, single-flight 확정, 토너먼트 실패 보상 |
| [06-cache-observability-and-faults.md](06-cache-observability-and-faults.md) | IM-22~25 | query cache, readiness, 계측·오류 경계, 장애 주입·복구 판정 |

## 대표 Thread와 연관 Thread 관계

| 대표 면접 포인트 | 대표 Thread | 함께 검토할 Thread | 통합 이유 |
|---|---:|---|---|
| IM-01 런타임 HTTP 계약 | 02 | 03, 06, 18, 20 | 외부 입력을 타입으로 신뢰하지 않고 각 프로토콜 경계에서 검증하는 공통 원리 |
| IM-03 WebSocket 티켓 | 06 | 03, 07, 08 | 프로토콜 버전, 인증 replay 방지, 계정·게스트 수명주기가 같은 handshake 경계에서 만남 |
| IM-04 실시간 권한 회수 | 07 | 06, 12, 15 | DB 권한 변경 이후 기존 세션·queue·room·확정 작업의 정리까지 이어짐 |
| IM-06 테스트 DB 보호 | 04 | 21, 23 | migration 상태, 통합 테스트 격리, 파괴적 명령의 안전성이 하나의 테스트 lifecycle을 형성 |
| IM-07 경기 결과 확정 | 15 | 07, 16, 21, 22 | idempotency, transaction, tournament 연결, 재시도, 종료 drain을 잇는 영속화 경계 |
| IM-09 토너먼트 정원 경쟁 | 16·17 | 11, 15 | 참가 경쟁, 예약, 대진표 단일 생성, 경기 확정이 같은 invariant를 공유 |
| IM-10 채팅 권한 | 18 | 02, 03, 12 | 런타임 schema, 저장 invariant, 현재 room 소유권, broadcast audience를 계층별로 검증 |
| IM-11 RoomSession | 12 | 07, 22 | 재접속·몰수패·권한 회수·shutdown이 모두 방 상태와 타이머 수명주기에 의존 |
| IM-13·14 실시간 실행·전송 | 13 | 09, 12, 21, 23 | 시뮬레이션 cadence, event-loop 지연, snapshot congestion, 부하 검증이 하나의 성능 경계 |
| IM-16 결정적 simulation | 09 | 10, 13, 14 | AI 난수, fixed-step, ordered input가 동일 replay를 보존해야 함 |
| IM-19 matchmaking | 11 | 10, 12, 16 | AI fallback, 연결 종료, 토너먼트 예약과 같은 리소스 경쟁 패턴을 공유 |
| IM-22 query cache | 20 | 02, 17, 19 | HTTP 계약과 mutation 영향 범위, read model, 사용자 세션 경계를 프런트 캐시에 반영 |
| IM-23·24 운영 상태·관측 | 21 | 04, 13, 22, 23 | migration, DB, event loop, shutdown, fault test가 배포 가능성과 복구 가능성을 함께 설명 |
| IM-25 fault scenario | 23 | 21, 22 | 관측 가능한 readiness와 안전한 cleanup이 있어야 장애 실험의 성공·복구를 판정할 수 있음 |

## S/A 상세 문서 완전성 검증

다음 표를 기준으로 마스터 인덱스의 모든 S/A 항목을 상세 문서와 대조했다. `통합 상세 항목`은 둘 이상의 Thread를 하나의 문제로 의도적으로 합친 경우다.

| ID | 우선순위 | 상태 | 상세 파일 | 대표/통합 범위 |
|---|---|---|---|---|
| IM-01 | A | 독립 상세 항목 | [01-contracts-auth-and-security.md](01-contracts-auth-and-security.md) | Thread 02 대표, 03·06·18 연관 |
| IM-02 | A | 독립 상세 항목 | [01-contracts-auth-and-security.md](01-contracts-auth-and-security.md) | Thread 03 대표 |
| IM-03 | S | 통합 상세 항목 | [01-contracts-auth-and-security.md](01-contracts-auth-and-security.md) | Thread 06 인증, Thread 03 버전 선검증, Thread 08 게스트 티켓 경계 |
| IM-04 | A | 통합 상세 항목 | [01-contracts-auth-and-security.md](01-contracts-auth-and-security.md) | Thread 07 DB·감사, Thread 12 연결·room 정리 |
| IM-05 | A | 통합 상세 항목 | [01-contracts-auth-and-security.md](01-contracts-auth-and-security.md) | Thread 08 대표, Thread 06·12 인증·연결 연관 |
| IM-06 | S | 통합 상세 항목 | [02-database-invariants-and-lifecycle.md](02-database-invariants-and-lifecycle.md) | Thread 04 보호 장치, Thread 23 통합 테스트 lifecycle |
| IM-07 | S | 통합 상세 항목 | [02-database-invariants-and-lifecycle.md](02-database-invariants-and-lifecycle.md) | Thread 15 대표, Thread 16 tournament 부수 효과 포함 |
| IM-08 | S | 독립 상세 항목 | [02-database-invariants-and-lifecycle.md](02-database-invariants-and-lifecycle.md) | Thread 17 friendship invariant |
| IM-09 | S | 통합 상세 항목 | [02-database-invariants-and-lifecycle.md](02-database-invariants-and-lifecycle.md) | Thread 16·17 원자적 참가와 경쟁 테스트 |
| IM-10 | A | 통합 상세 항목 | [02-database-invariants-and-lifecycle.md](02-database-invariants-and-lifecycle.md) | Thread 18 대표, Thread 02·03 계약 경계 포함 |
| IM-11 | S | 통합 상세 항목 | [03-realtime-state-and-resources.md](03-realtime-state-and-resources.md) | Thread 12 대표, Thread 22 drain 연관 |
| IM-12 | A | 독립 상세 항목 | [03-realtime-state-and-resources.md](03-realtime-state-and-resources.md) | Thread 12 heartbeat·connection ownership |
| IM-13 | S | 통합 상세 항목 | [03-realtime-state-and-resources.md](03-realtime-state-and-resources.md) | Thread 13 대표, Thread 09 simulation cadence |
| IM-14 | S | 통합 상세 항목 | [03-realtime-state-and-resources.md](03-realtime-state-and-resources.md) | Thread 13 대표, Thread 21·23 관측·부하 검증 연관 |
| IM-15 | A | 통합 상세 항목 | [03-realtime-state-and-resources.md](03-realtime-state-and-resources.md) | Thread 22 대표, Thread 12 room lifecycle, Thread 15 finalization 대기 |
| IM-16 | S | 통합 상세 항목 | [04-simulation-input-and-ai.md](04-simulation-input-and-ai.md) | Thread 09 대표, Thread 13·14 시간·입력 경계 연관 |
| IM-17 | S | 독립 상세 항목 | [04-simulation-input-and-ai.md](04-simulation-input-and-ai.md) | Thread 14 input gate |
| IM-18 | A | 통합 상세 항목 | [04-simulation-input-and-ai.md](04-simulation-input-and-ai.md) | Thread 10 대표, Thread 09 replay·11 fallback 연관 |
| IM-19 | S | 통합 상세 항목 | [05-matchmaking-finalization-and-tournaments.md](05-matchmaking-finalization-and-tournaments.md) | Thread 11 대표, Thread 10 AI·12 connection lifecycle 연관 |
| IM-20 | A | 통합 상세 항목 | [05-matchmaking-finalization-and-tournaments.md](05-matchmaking-finalization-and-tournaments.md) | Thread 15 비동기 coordinator, Thread 22 drain 연관 |
| IM-21 | A | 통합 상세 항목 | [05-matchmaking-finalization-and-tournaments.md](05-matchmaking-finalization-and-tournaments.md) | Thread 16 대표, Thread 11 reservation·15 확정 연관 |
| IM-22 | A | 통합 상세 항목 | [06-cache-observability-and-faults.md](06-cache-observability-and-faults.md) | Thread 20 대표, Thread 02 계약·19 read model 연관 |
| IM-23 | A | 통합 상세 항목 | [06-cache-observability-and-faults.md](06-cache-observability-and-faults.md) | Thread 21 대표, Thread 04 migration·22 배포 연관 |
| IM-24 | A | 통합 상세 항목 | [06-cache-observability-and-faults.md](06-cache-observability-and-faults.md) | Thread 21 대표, Thread 23 verification 연관 |
| IM-25 | A | 통합 상세 항목 | [06-cache-observability-and-faults.md](06-cache-observability-and-faults.md) | Thread 23 대표, Thread 21 readiness·22 배포 lifecycle 연관 |

## 백지 구현 우선순위

1. **최우선**: IM-16 결정적 시뮬레이션, IM-07 멱등 경기 확정, IM-11 RoomSession, IM-14 snapshot backpressure, IM-17 InputGate.
2. **상위**: IM-19 matchmaking reservation, IM-03 일회용 WebSocket 티켓, IM-13 fixed-step scheduler, IM-08 canonical friendship, IM-06 테스트 DB reset guard, IM-09 토너먼트 정원 경쟁.
3. **중상위**: IM-20 single-flight finalization, IM-05 guest lease, IM-21 토너먼트 예약 보상, IM-23 migration set 비교, IM-10 채팅 scope 검증.
4. **보완**: IM-24 계측·안전한 오류 경계, IM-22 query cache, IM-25 fault scenario, IM-18 PRNG·AI, IM-01 HTTP 경계, IM-02 WebSocket codec, IM-04 권한 회수, IM-12 heartbeat, IM-15 drain.

## 설명 연습 우선순위

1. **최우선**: IM-03 인증 티켓, IM-07 exactly-once 영속화, IM-11 재접속 상태 머신, IM-14 backpressure, IM-15 graceful drain.
2. **상위**: IM-04 실시간 권한 회수, IM-19 matchmaking 경쟁, IM-20 single-flight·retry, IM-23 readiness, IM-24 관측 경계, IM-25 장애 주입.
3. **중상위**: IM-05 guest lease, IM-09 토너먼트 정원 경쟁, IM-17 ordered input, IM-16 결정성, IM-22 캐시 정합성, IM-21 실패 보상.
4. **보완**: IM-10 채팅 audience, IM-13 fixed-step, IM-18 결정적 AI, IM-01 HTTP 계약, IM-02 protocol versioning, IM-08 canonical pair, IM-12 heartbeat, IM-06 파괴적 명령 보호.

## 한 문제로 통합한 Thread 묶음

1. **프로토콜·인증 handshake**: Thread 03 + 06 + 08 → IM-03. 버전 검증, 일회용 소비, 게스트·등록 사용자 인증 경계를 하나의 문제로 통합했다.
2. **권한 변경과 연결 정리**: Thread 07 + 12 → IM-04. 원자적 감사 기록과 이미 연결된 사용자의 queue·room·socket 회수를 함께 다룬다.
3. **테스트 DB lifecycle**: Thread 04 + 23 → IM-06. reset target 검증, schema 격리, migration 실행, cleanup 실패 처리를 묶었다.
4. **경기 결과 exactly-once**: Thread 15 + 16 → IM-07. match·rating·history·tournament 반영을 하나의 idempotent transaction 문제로 통합했다.
5. **토너먼트 참가 경쟁**: Thread 16 + 17 → IM-09. 마지막 자리 경쟁, 연속 seed, 준결승 단일 생성 invariant를 함께 묶었다.
6. **채팅 권한 경계**: Thread 02 + 03 + 18 → IM-10. HTTP/WS schema, 저장 scope, 현재 좌석과 audience 검증을 합쳤다.
7. **room lifecycle과 종료**: Thread 12 + 22 → IM-11·15. 재접속 유예와 graceful drain을 같은 상태·타이머 수명주기 축으로 다룬다.
8. **실시간 실행·전송 성능**: Thread 09 + 13 + 21 + 23 → IM-13·14·16. 결정적 step, catch-up 제한, backpressure, 관측·부하 검증을 연결했다.
9. **AI fallback matchmaking**: Thread 10 + 11 + 12 → IM-18·19. 결정적 AI 정책, fallback 시점, 예약과 disconnect 정리를 하나의 흐름으로 묶었다.
10. **경기 종료 coordinator**: Thread 15 + 22 → IM-20. 단일 in-flight 저장, retry, drain 대기를 같은 비동기 완료 경계로 통합했다.
11. **토너먼트 경기 saga**: Thread 11 + 15 + 16 → IM-21. 경기 예약, room 생성, 실패 보상, 최종 확정을 이어서 본다.
12. **클라이언트 서버 상태**: Thread 02 + 19 + 20 → IM-22. 응답 계약, read model, query key·session cache 정합성을 통합했다.
13. **배포 준비와 장애 복구**: Thread 04 + 21 + 22 + 23 → IM-23·24·25. migration·DB readiness, 메트릭, container lifecycle, fault recovery를 하나의 운영 검증 축으로 묶었다.
