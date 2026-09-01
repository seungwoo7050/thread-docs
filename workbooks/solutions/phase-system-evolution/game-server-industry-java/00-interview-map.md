# DevThreadEvolution GameServer JAVA — 기술면접 워크북 맵

## 문서 범위

이 워크북은 현재 GPT 프로젝트에서 확인할 수 있는 G01~G14 DevThread 문서와 커밋 작업 기록만 사용한다. 원격 저장소 상태, 현재 브랜치의 추가 코드, 문서에 나타나지 않은 클래스·함수·설정은 전제로 삼지 않는다.

선별 기준은 "프로젝트 설명을 외우는가"가 아니라 다음을 확인하는 데 둔다.

- 언어·런타임과 네트워크 I/O의 기본 원리를 이해하는가
- 상태 전이와 invariant를 코드로 보존할 수 있는가
- 메모리·버퍼·세션·스레드의 소유권과 수명을 끝까지 설명할 수 있는가
- 중복, 손실, 지연, 재접속, 과부하처럼 정상 경로 밖의 조건을 다룰 수 있는가
- 제한된 시간 안에 핵심 문제를 축소해 구현하고 trade-off를 설명할 수 있는가

우선순위는 다음 의미로 사용한다.

- **S**: 질문과 직접 구현 모두 준비해야 한다.
- **A**: 준비 가치가 높다. 대표 문제에 독립적으로 포함하거나 S 문제에 명시적으로 통합한다.
- **B**: 구현보다 설계·검증 방법을 설명하는 연습이 중요하다.
- **C**: 다른 대표 문제에 흡수되므로 별도 준비 항목을 만들 필요가 낮다.

## 전체 Thread/커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
|---|---|---|---|---|---|---|---|---|
| C | G01 | `feat: establish single-room Netty arena baseline` | `src/main/java/arena/ArenaServer.java`<br>`Room.java`<br>`CompleteFrame.java`<br>`ArenaMain.java`<br>`ScenarioRunner.java`<br>`TcpClient.java` | 단일 authoritative Room, bounded queue, 종료 순서 | 프로젝트의 토대지만 핵심 역량이 G03의 owner confinement, G04의 clock, G12의 outbound ownership으로 더 선명하게 드러난다. 별도 문제로 만들면 중복이 크다. | 중간 — 연관 문제의 배경 질문 | 낮음 — 별도 구현 제외 | G02, G03, G04, G12 |
| S | G02 | `feat: add bounded incremental TCP framing` | `src/main/java/arena/CompleteFrame.java`<br>`CompleteFrame.channelRead0`<br>`channelInactive`<br>`exceptionCaught`<br>`handlerRemoved`<br>`releaseBuffer`<br>`src/test/java/arena/CompleteFrameTest.java`<br>`FramingScenario.java` | TCP 스트림 framing, 점진 파서 상태, `ByteBuf` 수명, 오류 분류 | 패킷 경계와 메시지 경계를 구분하고 fragmentation/coalescing, EOF, 최대 크기, 버퍼 해제를 동시에 다룬다. 네트워크 기본기와 리소스 안전성을 직접 확인하기 좋다. | 매우 높음 | 매우 높음 | G01, G09, G12 |
| S | G03 | `test: verify identity lifecycle and room ownership` | `src/test/java/arena/IdentityScenario.java`<br>`RoomTest.frozenG03IdentityLifecycle`<br>`src/main/java/arena/ArenaServer.java`<br>`Room.java`<br>`Room.assertOwner` | Connection·Session·Player·Room 경계, single-owner state, bounded mailbox, lifecycle | 락을 늘리는 대신 한 owner가 상태를 직렬화하는 이유, 네트워크 콜백과 게임 상태 경계, ID 권한 검증, mailbox 포화 시 동작을 함께 묻기 좋다. | 매우 높음 | 높음 | G01, G05, G11, G13 |
| S | G04 | `feat: add bounded monotonic fixed clock` | `src/main/java/arena/FixedTickClock.java`<br>`ArenaServer.runClockIteration`<br>`src/main/java/arena/ClockScenario.java` | monotonic fixed-step accumulator, drift, bounded catch-up, overload 표현 | 실시간 시뮬레이션에서 wall clock을 배제하고 backlog를 보존하면서도 한 번의 iteration이 독점하지 않게 하는 전형적인 설계 문제다. | 매우 높음 | 매우 높음 | G01, G13 |
| S | G05 | `feat: validate sequence and tick input ordering` | `src/main/java/arena/Room.java`<br>`Room.accept`<br>`Room.tick`<br>`Json.sequence`<br>`Json.targetTick`<br>`CompleteFrame.protocolError`<br>`SequenceScenario.java` | unsigned sequence, 중복·충돌·stale, target tick window, per-tick 선택 | idempotency와 순서 보장, 정확한 정수 파싱, 상태 불변성, bounded pending queue를 한 문제로 확인할 수 있다. | 매우 높음 | 매우 높음 | G03, G06, G07, G09 |
| A | G06 | `feat: bound per-tick input validation attempts` | `src/main/java/arena/Room.java`<br>`Room.beginInputValidation`<br>`ArenaServer` INPUT 처리 경계<br>`AuthorityScenario.java`<br>`RoomTest.frozenG06AuthorityAbuse` | authoritative validation 순서, per-player abuse bound, 실패 시 상태 보존 | 단순 rate limiter가 아니라 "어느 검증 뒤·어느 검증 앞에서 카운트하는가"가 오류 우선순위와 권한 격리를 결정한다. G05 admission 문제에 통합하는 편이 역량 중복을 줄인다. | 높음 | 높음 — G05에 통합 | G05, G03 |
| S | G07 | `feat: capture bounded replay and canonical tick hashes` | `src/main/java/arena/ReplayLog.java`<br>`Room.canonicalRecord`<br>`Room.stateHash`<br>`ArenaServer.replayArtifact`<br>`src/test/java/arena/ReplayVerifier.java`<br>`ReplayScenario.java`<br>`ReplayTool.java` | 결정적 canonicalization, state hash, replay event boundary, bounded artifact | 재현성은 이벤트 목록만 저장한다고 얻어지지 않는다. 초기 상태, admission 시점, 정렬, 직렬화 규약, artifact 불완전성을 모두 명시해야 한다. | 매우 높음 | 높음 | G05, G06, G08, G14 |
| S | G08 | `feat: publish acknowledged full and delta snapshots` | `src/main/java/arena/SnapshotStream.java`<br>`SnapshotStream.next`<br>`SnapshotStream.acknowledge`<br>`src/test/java/arena/SnapshotScenario.java`<br>`SnapshotStreamTest.java` | FULL·DELTA, acknowledged base, immutable retention, removals | 복제 프로토콜의 핵심인 base contract, 변경 집합, 삭제, 보존 한도, 정렬된 wire 표현을 작은 상태 머신으로 설명·구현할 수 있다. | 매우 높음 | 매우 높음 | G07, G10, G11, G12 |
| A | G09 | `feat(udp): add bounded realtime data and independent endpoint binding` | `src/main/java/arena/UdpData.java`<br>`UdpData.channelRead0`<br>`UdpData.bytes`<br>`ArenaServer.handleUdp`<br>`ArenaServer.handleCommand`<br>`TcpClient.sendTcp` / `sendUdp`<br>`UdpBoundaryTest.java`<br>`UdpScenario.java`<br>`UdpFaultProxy.java` | TCP control/UDP data 경계, endpoint binding, one-time token, datagram bound | transport 분리의 장점보다 인증되지 않은 UDP 발신자를 세션에 안전하게 귀속시키는 문제가 중요하다. 크기 제한과 owner dispatch 이전 drop도 구현 가치가 높다. | 매우 높음 | 높음 | G02, G05, G11, G12 |
| S | G10 | `feat: preserve snapshot ACK watermarks and schedule FULL recovery` | `src/main/java/arena/SnapshotStream.java`<br>`SnapshotStream.Retained`<br>`SnapshotStream.next`<br>`SnapshotStream.acknowledge`<br>`src/test/java/arena/AckScenario.java`<br>`SnapshotStreamTest.java` | ACK watermark 단조성, hash 검증, unknown·expired ACK, scheduled FULL resync | 손실 자체와 복구 필요를 구분하고, 잘못된 ACK가 base로 채택되거나 watermark를 되돌리지 못하게 하는 invariant가 명확하다. G08 문제에 통합한다. | 매우 높음 | 매우 높음 — G08에 통합 | G08, G11, G12 |
| S | G11 | `feat(java): preserve sessions across bounded reconnect grace` | `src/main/java/arena/ArenaServer.java`의 `sessions`, `recoverableSessions`<br>`SnapshotStream.java`<br>`src/test/java/arena/ReconnectScenario.java`<br>`TcpClient.reconnectFrom` | bounded reconnect grace, resume token rotation, endpoint 폐기, 즉시 FULL resync | 인증정보의 일회성, 만료 경계, provisional session 정리, 게임 상태 보존과 transport 상태 초기화를 분리해야 한다. 보안·상태·복구를 함께 묻기 좋다. | 매우 높음 | 높음 | G03, G08, G09, G10, G12 |
| S | G12 | `feat: bound outbound snapshot ownership` | `src/main/java/arena/OutboundQueue.java`<br>`OutboundQueue.admit`<br>`flushWhenReady`<br>`close`<br>`view`<br>`ArenaServer.Peer.outbound`<br>`src/test/java/arena/BackpressureScenario.java`<br>`OutboundQueueTest.java` | slow consumer, bounded outbound work, snapshot coalescing, Netty-owned buffer | 가장 직접적인 메모리·수명 문제다. queued와 in-flight 소유권을 구분하고, supersede·close·write completion마다 정확히 한 번 해제해야 한다. | 매우 높음 | 매우 높음 | G01, G02, G08, G09, G10, G14 |
| S | G13 | `feat: isolate bounded many-room scheduling` | `src/main/java/arena/RoomRuntime.java`<br>`ArenaServer` Room registry·scheduler 경계<br>`src/test/java/arena/ManyRoomScenario.java` | per-Room runtime, fair quantum, shared monotonic sample, hot-room isolation | 단일 owner 모델을 여러 Room으로 확장할 때 공정성, bounded work, routing authority, 실패 격리가 어떻게 바뀌는지 확인할 수 있다. | 매우 높음 | 높음 | G03, G04, G12, G14 |
| B | G14 | `G14: verify bounded realtime-core release workload` | `src/test/java/arena/ReleaseScenario.java`<br>`ReleaseMetrics.java`<br>`SnapshotReadiness.java`<br>`scripts/g14_release_load.py`<br>`scripts/g14_load_slot.py`<br>`track` | 고정 부하 검증, 측정 범위, JFR lifecycle, release artifact 검증 | 새 코어 알고리즘보다 "무엇을 측정했고 무엇을 측정하지 않았는가", warm-up·재시도 없는 고정 workload, profiler·자식 프로세스 cleanup을 설명하는 항목이다. | 높음 | 낮음 — 설계·검증 설명 중심 | G07, G12, G13 |

## 대표 면접 포인트와 상세 워크북 위치

| ID | 대표 Thread | 통합한 Thread | 대표 면접 포인트 | 상세 문서 |
|---|---|---|---|---|
| W01 | G02 | — | 점진적 TCP frame decoder와 `ByteBuf` 수명 | [`01-protocol-and-resource-lifecycle.md`](01-protocol-and-resource-lifecycle.md) |
| W02 | G09 | — | TCP control/UDP data 경계와 endpoint binding | [`01-protocol-and-resource-lifecycle.md`](01-protocol-and-resource-lifecycle.md) |
| W03 | G12 | — | slow consumer용 bounded outbound queue와 buffer ownership | [`01-protocol-and-resource-lifecycle.md`](01-protocol-and-resource-lifecycle.md) |
| W04 | G03 | G01 | single-owner command gateway, identity, lifecycle, mailbox bound | [`02-concurrency-clock-and-scheduling.md`](02-concurrency-clock-and-scheduling.md) |
| W05 | G04 | — | monotonic fixed-step accumulator와 bounded catch-up | [`02-concurrency-clock-and-scheduling.md`](02-concurrency-clock-and-scheduling.md) |
| W06 | G13 | — | many-Room fair scheduling과 hot-room failure isolation | [`02-concurrency-clock-and-scheduling.md`](02-concurrency-clock-and-scheduling.md) |
| W07 | G05 | G06 | 입력 sequence·tick window·idempotency·abuse bound | [`03-command-authority-and-replay.md`](03-command-authority-and-replay.md) |
| W08 | G07 | — | canonical state hash와 bounded replay | [`03-command-authority-and-replay.md`](03-command-authority-and-replay.md) |
| W09 | G08 | G10 | FULL·DELTA snapshot, ACK watermark, scheduled resync | [`04-replication-and-recovery.md`](04-replication-and-recovery.md) |
| W10 | G11 | — | reconnect grace, resume credential rotation, 즉시 FULL 복구 | [`04-replication-and-recovery.md`](04-replication-and-recovery.md) |

## 대표 Thread와 연관 Thread 관계

| 대표 문제 | 대표로 삼은 이유 | 연관 Thread에서 가져온 범위 | 통합하지 않은 범위 |
|---|---|---|---|
| W04 — G03 owner/identity | G03이 G01의 single-owner 전제를 실제 identity·lifecycle·mailbox 경계로 검증한다. | G01의 bounded owner queue, 종료 순서, authoritative Room 전제를 배경으로 포함한다. | G01 parser는 W01, clock은 W05, outbound는 W03에서 다룬다. |
| W07 — G05/G06 admission | sequence·duplicate·tick window와 validation rate는 같은 INPUT admission 상태 머신의 전후 단계다. | G06의 per-player 4회 검증 한도, 오류 우선순위, inactive/foreign actor 격리를 포함한다. | G06의 이동·TAG 기존 규칙은 꼬리 질문으로만 남긴다. |
| W09 — G08/G10 replication | G10의 ACK 복구 규칙은 G08의 retained base contract 없이는 독립적으로 설명할 수 없다. | retained hash, watermark 단조성, unknown·expired·mismatch ACK, `resyncPending`을 포함한다. | G11 reconnect가 만드는 새 transport/session lifecycle은 W10으로 분리한다. |
| W02 ↔ W10 | 둘 다 credential·endpoint를 다루지만 문제의 중심이 다르다. | W02는 최초 UDP endpoint binding, W10은 disconnect 이후 credential rotation과 session takeover를 다룬다. | 한 문제로 합치면 네트워크 경계와 reconnect 상태 머신이 지나치게 커지므로 분리한다. |
| W03 ↔ W09 | snapshot 생성과 snapshot 전송은 서로 다른 책임이다. | W09는 어떤 FULL/DELTA를 만들지, W03은 만들어진 buffer를 얼마나 소유하고 언제 버릴지를 다룬다. | 생성 규약과 transport ownership을 한 구현 문제로 섞지 않는다. |
| W05 ↔ W06 | G13 scheduler가 Room별 `FixedTickClock`을 사용하지만 공정성 계층이 추가된다. | W05는 단일 accumulator, W06은 여러 Room 사이의 quantum·deadline·격리를 다룬다. | clock 수학과 registry/routing을 별도 구현으로 유지한다. |

## S/A 완전성 검증

| Thread | 우선순위 | 상태 | 상세 위치 또는 통합 대상 |
|---|---|---|---|
| G02 | S | 독립적인 상세 워크북 항목 | W01 — `01-protocol-and-resource-lifecycle.md` |
| G03 | S | 독립적인 상세 워크북 항목 | W04 — `02-concurrency-clock-and-scheduling.md` |
| G04 | S | 독립적인 상세 워크북 항목 | W05 — `02-concurrency-clock-and-scheduling.md` |
| G05 | S | 독립적인 상세 워크북 항목 | W07 — `03-command-authority-and-replay.md` |
| G06 | A | 대표 문제에 명시적으로 통합 | W07의 validation ordering·rate bound |
| G07 | S | 독립적인 상세 워크북 항목 | W08 — `03-command-authority-and-replay.md` |
| G08 | S | 독립적인 상세 워크북 항목 | W09 — `04-replication-and-recovery.md` |
| G09 | A | 독립적인 상세 워크북 항목 | W02 — `01-protocol-and-resource-lifecycle.md` |
| G10 | S | 대표 문제에 명시적으로 통합 | W09의 ACK watermark·FULL recovery |
| G11 | S | 독립적인 상세 워크북 항목 | W10 — `04-replication-and-recovery.md` |
| G12 | S | 독립적인 상세 워크북 항목 | W03 — `01-protocol-and-resource-lifecycle.md` |
| G13 | S | 독립적인 상세 워크북 항목 | W06 — `02-concurrency-clock-and-scheduling.md` |

S/A 항목 12개는 모두 독립 항목 또는 명시적 통합 상태다. G01은 W04·W05·W03의 배경으로 분해됐고, G14는 설명 연습용 B 항목으로 남겨 상세 구현 문제를 만들지 않았다.

## 백지 구현 우선순위

1. **W07 — G05/G06 입력 admission 상태 머신**: sequence, duplicate, tick window, rate bound, 불변성까지 한 번에 점검한다.
2. **W01 — G02 점진적 TCP frame decoder**: fragmentation/coalescing, 최대 길이, EOF, recoverable/terminal 오류를 구현한다.
3. **W03 — G12 bounded outbound queue**: ordered/FULL/DELTA 슬롯과 in-flight 소유권을 분리한다.
4. **W09 — G08/G10 snapshot stream**: retained base, delta, ACK watermark, scheduled FULL 복구를 구현한다.
5. **W05 — G04 fixed-step accumulator**: monotonic time, 잔여 backlog, catch-up cap을 구현한다.
6. **W04 — G03/G01 owner mailbox gateway**: network thread와 state owner 경계, identity 검증, bounded admission을 구현한다.
7. **W06 — G13 many-Room scheduler**: per-Room quantum과 deadline failure isolation을 구현한다.
8. **W10 — G11 resume registry**: grace 만료, one-time credential rotation, session takeover를 구현한다.
9. **W08 — G07 bounded replay log**: canonical bytes와 overflow latch, close cleanup을 구현한다.
10. **W02 — G09 UDP endpoint registry**: one-time bind token과 sender endpoint 검증을 구현한다.

## 설명 연습 우선순위

1. **G12**: queued buffer와 transport-owned in-flight buffer의 수명, supersede·close·completion 책임.
2. **G13**: shared time sample, per-Room quantum, hot-room 격리, 운영 overload와 게임 lifecycle의 분리.
3. **G08/G10**: delta base가 왜 "마지막 전송"이 아니라 "유효하게 ACK된 retained snapshot"이어야 하는지.
4. **G11**: 게임 상태는 보존하고 transport·credential·snapshot history는 교체하는 이유.
5. **G07**: 결정성에 필요한 canonicalization 규약과 admission boundary 기록.
6. **G03/G01**: single-owner를 선택한 이유, mailbox overflow 정책, Connection·Session·Player·Room의 권한 경계.
7. **G09**: UDP source spoofing과 session claim을 동시에 검증하는 순서, 독립 port binding.
8. **G14**: 측정 범위, release JAR 사용 확인, JFR lifecycle, warm-up·재시도 없는 고정 workload의 해석 한계.
9. **G04**: wall clock 대신 monotonic source를 쓰는 이유와 backlog를 버리지 않는 trade-off.
10. **G05/G06**: duplicate·stale·conflict·rate exceeded가 accepted state를 바꾸지 않아야 하는 이유.

## 한 문제로 통합한 Thread 묶음

1. **G01 + G03 → W04**: authoritative single-owner 전제와 identity/lifecycle/mailbox 검증.
2. **G05 + G06 → W07**: 입력 sequence·tick window·idempotency와 per-tick abuse bound.
3. **G08 + G10 → W09**: FULL·DELTA base contract와 ACK watermark·scheduled FULL recovery.
