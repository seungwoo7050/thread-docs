# GameServerCPP 개발자 기술면접 마스터 맵

## 문서 사용 기준

이 문서는 현재 프로젝트에 축적된 G01~G14 Thread 작업 기록만을 기준으로 작성했다. 원격 저장소나 외부 자료는 사용하지 않았다.

- 정확한 Conventional Commit subject가 기록에 노출된 G01, G02, G04, G08, G10, G11, G12, G14는 그대로 적었다.
- G03, G05, G06, G07, G09, G13은 현재 프로젝트 기록에서 exact subject를 확인할 수 없어 Thread 문서 또는 evidence의 기록 제목으로 명시했다. `feat`, `fix` 같은 접두사는 추측하지 않았다.
- 관련 위치에는 현재 기록에서 실제로 확인되는 파일·함수·클래스·컴포넌트만 적었다.
- 같은 역량을 반복하는 G05·G06과 G08·G10은 대표 문제로 통합했다.
- G14는 production 구현 변경이 아니라 고정 부하·관측 harness가 중심이므로 설명 준비용 B로 분류했다.

## 우선순위 정의와 결과 요약

| 우선순위 | 의미 | Thread 수 |
|---|---|---:|
| S | 질문과 10~30분 직접 구현 모두 반드시 준비할 가치가 높음 | 9 |
| A | 질문 또는 핵심 구현 가능성이 높아 준비 가치가 큼 | 4 |
| B | 직접 구현보다 설계·검증 방법 설명이 중요함 | 1 |
| C | 별도 면접 준비 항목으로 만들 가치가 낮음 | 0 |

## 전체 Thread·커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread | 상세 워크북 |
|---|---|---|---|---|---|---|---|---|---|
| A | G01 | `feat: establish kqueue authoritative arena baseline` | `src/game.hpp`, `src/game.cpp`, `src/transport.hpp`, `src/transport.cpp`; `Room`, `Server`, `Fd`, `Server::shutdown`, `Server::advance_one_tick` | 이동 전용 OS 자원, 멱등 종료, authoritative single-owner baseline | C++ 자원 수명과 종료 순서는 프로젝트 밖에서도 반복되는 기본기다. 다만 queue·owner 설계의 더 대표적인 후속 지점이 G03·G12에 있다. | 높음 | 높음 | G03, G12 | [01 · G01](01-memory-ownership-and-io.md#w01-g01) |
| S | G02 | `feat: decode bounded incremental TCP frames` | `src/transport.hpp`, `src/transport.cpp`; `FrameParser`, `FrameParser::consume`, `FrameParser::finish`, `ParseResult`, `ParseState`, `parse_object`, `request_error` | TCP fragmentation·coalescing, bounded incremental parser, 오류 분류와 peer 격리 | 스트림 I/O의 핵심 원리, 유한 메모리, protocol security, recoverable/terminal 경계가 한 지점에 모여 있다. 직접 구현 문제로도 적합하다. | 매우 높음 | 매우 높음 | G01, G03, G09, G12 | [01 · 증분 파서](01-memory-ownership-and-io.md#w01-g02-parser), [01 · 오류 경계](01-memory-ownership-and-io.md#w01-g02-errors) |
| S | G03 | 기록 제목: Connection, session, player and room ownership | `src/transport.hpp`, `src/transport.cpp`, `src/game.hpp`, `src/game.cpp`, `src/lifecycle_scenario.cpp`; `Server::Mailbox`, `Server::poll_io`, `Server::drain_mailbox`, `Room`, `Connection`, `LifecycleFixture` | 식별자 수명 분리, bounded FIFO mailbox, single-writer state invariant, lifecycle matrix | 네트워크 콜백과 도메인 mutation의 경계, 전역·연결별 queue 상한, 상태기계가 서버 설계의 중심이다. | 매우 높음 | 매우 높음 | G01, G04, G11, G13 | [01 · G03](01-memory-ownership-and-io.md#w01-g03) |
| S | G04 | `feat(time): bound fixed ticks with monotonic accumulation` | `src/game.hpp`, `src/game.cpp`, `src/transport.cpp`, `src/lifecycle_scenario.cpp`; `TickBatch`, `FixedTickAccumulator`, `monotonic_milliseconds`, `Server::run_scheduler`, `Server::advance_one_tick` | monotonic fixed tick, drift 방지, 잔여 시간 보존, bounded catch-up | 짧은 코드 안에서 시간 모델, 정수 invariant, overload·공정성 trade-off를 검증할 수 있어 질문·구현 가치가 모두 높다. | 매우 높음 | 매우 높음 | G03, G13 | [02 · G04](02-time-input-and-determinism.md#w02-g04) |
| S | G05 | 기록 제목: Sequence identity and tick-window admission | `src/game.hpp`, `src/game.cpp`, `src/transport.hpp`, `src/transport.cpp`; `Input`, `InputResult`, `Player`, `Room::input`, `Room::tick`, `decode_input`, `admit_input` | exact sequence identity, duplicate·conflict·stale, target tick window, pending 상한 | idempotency와 상태 mutation 순서, unsigned 경계, deterministic tick 적용을 함께 묻기 좋은 대표 입력 문제다. | 매우 높음 | 매우 높음 | G04, G06, G07, G13 | [02 · G05+G06](02-time-input-and-determinism.md#w02-g05-g06) |
| A | G06 | 기록 제목: Authoritative intent and four-attempt bound | `src/game.hpp`, `src/game.cpp`, `src/transport.cpp`; `Room::begin_input_attempt`, `admit_input`, `input_attempt_high_water` | authenticated owner admission, malformed까지 세는 tick당 abuse bound | G05 admission에 rate gate의 위치와 rejected-state invariant를 더한다. 독립 문제보다 G05의 보안·실패 경계로 통합하는 편이 면접 가치가 높다. | 매우 높음 | 높음 | G02, G05, G13 | [02 · G05에 통합](02-time-input-and-determinism.md#w02-g05-g06) |
| S | G07 | 기록 제목: Accepted-input replay and exact state hash | `src/replay.hpp`, `src/replay.cpp`, `src/transport.hpp`, `src/transport.cpp`, `tests/g07.hpp`, `tests/g07.cpp`; `ReplayLog`, `canonical_state`, `sha256`, `parse_replay_artifact`, `verify_replay`, `ReplayRun` | canonical bytes, accepted-event journal, per-tick hash, 첫 divergence, bounded capture | 결정성·데이터 정합성·진단 가능성·기록 overflow 정책을 일반화할 수 있다. serializer와 verifier 모두 면접 구현으로 축소 가능하다. | 매우 높음 | 매우 높음 | G05, G08, G10, G11, G13 | [02 · G07](02-time-input-and-determinism.md#w02-g07) |
| S | G08 | `feat(snapshot): add acknowledged full and delta replication` | `src/snapshot.hpp`, `src/snapshot.cpp`, `src/transport.hpp`, `src/transport.cpp`; `SnapshotStream`, `SnapshotStream::publish`, `SnapshotStream::acknowledge`, `Server::publish_snapshots`, `Server::start_room`, `run_snapshot_scenario` | FULL materialization retention, DELTA diff·apply, acknowledged base contract | 자료구조·정합성·wire 효율·복구의 trade-off가 명확하다. diff/apply와 ACK 상태기를 각각 직접 구현 문제로 만들 수 있다. | 매우 높음 | 매우 높음 | G07, G10, G11, G12 | [03 · DELTA](03-replication-security-and-reconnect.md#w03-g08-delta), [03 · ACK](03-replication-security-and-reconnect.md#w03-g08-g10-ack) |
| A | G09 | 기록 제목: Bounded UDP data plane / TCP control and UDP data plane | `src/transport.hpp`, `src/transport.cpp`, `tests/g09.hpp`, `tests/g09.cpp`; `UdpClient`, `Server::bind_datagram`, `Server::read_datagrams`, `Server::send_realtime` | TCP-authenticated UDP binding, one-time token, endpoint·epoch 검증, payload bound | connectionless transport의 인증·보안 경계와 consume-after-validation transaction을 묻기 좋다. 실제 socket 구현보다 validator 축소 문제가 적절하다. | 매우 높음 | 높음 | G03, G11, G12 | [03 · G09](03-replication-security-and-reconnect.md#w03-g09) |
| A | G10 | `feat(snapshot): validate monotonic acknowledgements and schedule full resync` | `src/snapshot.hpp`, `src/snapshot.cpp`, `src/transport.cpp`, `tests/g10.hpp`, `tests/g10.cpp`; `SnapshotStream::acknowledge`, `SnapshotStream::publish`, `run_ack_scenario` | monotonic ACK watermark, retention 밖 ACK, hash mismatch, next scheduled FULL | G08 base contract의 오류·복구 정책을 완성한다. 독립 serializer 문제보다 G08 ACK 상태기에 합쳐야 중복 없이 핵심 invariant가 드러난다. | 매우 높음 | 높음 | G07, G08, G11, G12 | [03 · G08에 통합](03-replication-security-and-reconnect.md#w03-g08-g10-ack) |
| S | G11 | `feat(session): add bounded reconnect grace and rotating credentials` | `src/game.hpp`, `src/game.cpp`, `src/transport.hpp`, `src/transport.cpp`, `tests/g11.hpp`, `tests/g11.cpp`; `Server::reconnect`, `Room::reconnect`, `Room::evaluate_start_condition`, `Player::disconnect_deadline`, `Server::publish_snapshot` | executed-tick grace, stable identity 승계, credential rotation, current FULL resync | lifecycle·보안·transaction·실패 원자성이 한 번에 드러난다. deadline 경계와 partial mutation을 직접 묻기 좋다. | 매우 높음 | 매우 높음 | G03, G09, G10, G12 | [03 · G11](03-replication-security-and-reconnect.md#w03-g11) |
| S | G12 | `feat(cpp): bound and coalesce outbound snapshots` | `src/transport.hpp`, `src/transport.cpp`, `tests/g12.hpp`, `tests/g12.cpp`, `tests/g12_queue.cpp`; `OutboundMemory`, `PendingWrite`, `Server::write_datagrams`, `Server::update_udp_write_interest`, `Server::outbound_state`, `OutboundFixture` | slow consumer, bounded control queue, snapshot coalescing, partial write와 move-only buffer lifecycle | 실시간 서버에서 필수적인 backpressure·메모리 상한·비차단 I/O·resource cleanup을 대표한다. 질문과 구현 모두 강하다. | 매우 높음 | 매우 높음 | G01, G08, G10, G11, G13, G14 | [04 · G12](04-backpressure-and-room-scheduling.md#w04-g12) |
| S | G13 | 기록 제목: Many-room scheduler and hot-room isolation | `src/transport.hpp`, `src/transport.cpp`, `tests/g13.hpp`, `tests/g13.cpp`; `RoomContext`, `Server::context`, `Server::room_metrics`, `Server::execute_tick`, `Server::close_overloaded`, `rooms_`, `scheduled_rooms_`, `multi_room_fixture` | Room별 accumulator, bounded service quantum, hot-room overload 종료, failure containment | G03·G04·G12를 다중 tenant 시스템 경계로 확장한다. 공정성·복잡도·격리·cleanup을 종합적으로 평가할 수 있다. | 매우 높음 | 매우 높음 | G03, G04, G12, G14 | [04 · G13](04-backpressure-and-room-scheduling.md#w04-g13) |
| B | G14 | `test(release): verify fixed realtime-core load and resource bounds` | `tests/g14.hpp`, `tests/g14.cpp`; `run_release_load`, `process_sample`, `Lines`, `SnapshotHold`, `OutboundFixture`, `multi_room_fixture` | 고정 workload, 관측 오염 방지, Release·TSan 분리, streaming evidence, resource profiling | production 변경 없이 검증 harness를 고정한 작업이다. 직접 구현보다 성능 실험의 재현성·측정 경계·evidence 상한 설명이 중요하다. | 높음 | 설명 중심 | G07, G12, G13 | — B 항목이므로 상세 문서 없음 |

## 대표 Thread와 연관 Thread 관계

| 대표 면접 지점 | 대표 Thread | 연관 위치·Thread | 관계 |
|---|---|---|---|
| 이동 전용 자원과 종료 | G01 | G03, G12 | G01의 RAII는 독립 문제로 유지하고, owner mutation은 G03, outbound lifetime은 G12가 더 대표적으로 다룬다. |
| TCP 증분 파싱과 오류 격리 | G02 | G01, G03, G09, G12 | G01의 bounded transport를 실제 stream parser로 확장하고, 결과는 G03 mailbox로 넘어간다. UDP는 G09에서 datagram 경계를 별도로 갖는다. |
| 식별자·lifecycle·single owner | G03 | G01, G04, G11, G13 | G04 scheduler, G11 reconnect, G13 RoomContext가 모두 G03의 owner·identity 경계를 전제로 한다. |
| 고정 틱 primitive | G04 | G13 | G04의 O(1) accumulator를 G13이 Room별로 소유해 다중 Room scheduler를 만든다. 두 문제는 추상화 수준이 달라 합치지 않는다. |
| 입력 승인 상태기계 | G05 | G06 | sequence·tick window·pending 상태에 G06의 decoder 전 attempt gate를 붙여 하나의 문제로 통합한다. |
| 결정적 replay | G07 | G05, G08, G10, G11, G13 | accepted event와 canonical hash가 snapshot metadata, reconnect lifecycle, Room별 replay 검증의 기준이 된다. |
| snapshot base·복구 | G08 | G10 | G08의 FULL/DELTA 생성은 독립 구현 문제로 유지하고, ACK 오류와 resync policy는 G10을 합쳐 하나의 상태기계로 만든다. |
| TCP/UDP 전송 보안 | G09 | G03, G11, G12 | G03 session ownership을 UDP endpoint와 묶고, G11이 reconnect 때 credential을 회전하며, G12가 nonblocking outbound를 제한한다. |
| reconnect transaction | G11 | G03, G09, G10 | stable identity, 새 UDP bind, 현재 FULL 복구를 하나의 실패 원자적 lifecycle 문제로 다룬다. |
| slow-consumer backpressure | G12 | G01, G08, G10, G13, G14 | RAII buffer, snapshot 의미, Room 격리, fixed-load 관측이 교차하는 대표 성능·수명 지점이다. |
| 다중 Room 공정성과 격리 | G13 | G03, G04, G12, G14 | single owner, Room별 시간, connection별 backpressure를 instance 수준으로 통합한다. |
| 성능 검증 설계 | G14 | G07, G12, G13 | 구현 문제로 별도 확장하지 않고 replay·outbound·scheduler의 고정 부하 검증 방법을 설명하는 B 항목으로 둔다. |

## 상세 워크북 문서 구성

| 파일 | 포함 면접 포인트 | 항목 수 |
|---|---|---:|
| [01-memory-ownership-and-io.md](01-memory-ownership-and-io.md) | G01 이동 전용 자원, G02 증분 파서, G02 오류 경계, G03 owner mailbox·identity | 4 |
| [02-time-input-and-determinism.md](02-time-input-and-determinism.md) | G04 고정 틱, G05+G06 입력 승인, G07 replay·hash | 3 |
| [03-replication-security-and-reconnect.md](03-replication-security-and-reconnect.md) | G08 DELTA, G08+G10 ACK·resync, G09 UDP binding, G11 reconnect | 4 |
| [04-backpressure-and-room-scheduling.md](04-backpressure-and-room-scheduling.md) | G12 bounded outbound, G13 다중 Room scheduler | 2 |

## S/A 완전성 검증

| S/A Thread | 마스터 상태 | 상세 워크북 대응 | 결과 |
|---|---|---|---|
| G01 | 독립 항목 | `01-memory-ownership-and-io.md` · `w01-g01` | 작성됨 |
| G02 | 독립 항목 2개 | `w01-g02-parser`, `w01-g02-errors` | 작성됨 |
| G03 | 독립 항목 | `w01-g03` | 작성됨 |
| G04 | 독립 항목 | `02-time-input-and-determinism.md` · `w02-g04` | 작성됨 |
| G05 | 대표 독립 항목 | `w02-g05-g06` | 작성됨 |
| G06 | G05 대표 문제에 명시적 통합 | `w02-g05-g06` | 통합됨 |
| G07 | 독립 항목 | `w02-g07` | 작성됨 |
| G08 | 독립 DELTA 항목 + ACK 대표 항목 | `w03-g08-delta`, `w03-g08-g10-ack` | 작성됨 |
| G09 | 독립 항목 | `w03-g09` | 작성됨 |
| G10 | G08 ACK 대표 문제에 명시적 통합 | `w03-g08-g10-ack` | 통합됨 |
| G11 | 독립 항목 | `w03-g11` | 작성됨 |
| G12 | 독립 항목 | `04-backpressure-and-room-scheduling.md` · `w04-g12` | 작성됨 |
| G13 | 독립 항목 | `w04-g13` | 작성됨 |

누락된 S/A 항목은 없다. G14는 B이므로 상세 워크북 작성 대상에서 제외했다.

## 백지 구현 우선순위

1. G02 — 길이 접두사 증분 TCP parser
2. G04 — monotonic fixed-tick accumulator
3. G05+G06 — sequence·tick window·attempt bound 입력 승인 상태기계
4. G08 — FULL materialization 기반 DELTA 생성·적용
5. G12 — bounded control queue·snapshot slot·move-only buffer
6. G13 — Room별 bounded scheduler와 hot-room 종료 지시
7. G11 — grace 내 reconnect와 credential rotation transaction
8. G08+G10 — ACK watermark·retention·next FULL 상태기계
9. G03 — 전역·source별 상한을 가진 FIFO mailbox
10. G07 — canonical serializer와 first-divergence verifier
11. G09 — one-time UDP bind·datagram authorization validator
12. G01 — 이동 전용 OS handle RAII
13. G02 — recoverable/terminal connection action 분류기

## 설명 연습 우선순위

1. G03 — Connection·session·player·room 수명 분리와 single-owner invariant
2. G12 — slow consumer가 simulation·healthy client에 영향을 주지 않는 이유
3. G08+G10 — acknowledged base, monotonic ACK, FULL fallback contract
4. G11 — reconnect의 실패 원자성, stable identity, credential rotation
5. G07 — canonical state 선택과 accepted-event replay 경계
6. G13 — Room별 시간·공정성·hot-room failure containment
7. G02 — TCP stream 경계와 recoverable/terminal 오류 분리
8. G09 — TCP bootstrap과 UDP endpoint 보안 경계
9. G04 — drift, backlog 보존, catch-up cap trade-off
10. G05+G06 — accepted-only mutation과 decoder 전 abuse gate
11. G01 — RAII·멱등 shutdown·signal-safe 종료 경로
12. G14 — fixed workload, 측정 오염 방지, Release·TSan 관측 분리

## 한 문제로 통합한 Thread 묶음

1. **G05 + G06**: sequence identity, duplicate/conflict, target tick window, pending bound, malformed까지 포함하는 tick당 attempt gate를 하나의 입력 승인 상태기계로 통합했다.
2. **G08 + G10**: ACK watermark, retained base 검증, unknown·evicted ACK, hash mismatch, `resync_required`, 다음 scheduled FULL을 하나의 복구 상태기계로 통합했다. G08의 DELTA 생성·적용은 별도 구현 문제로 유지했다.
3. **G01의 일부 역량을 대표 후속 문제에 흡수**: owner mutation 경계는 G03, outbound queue·buffer 수명은 G12를 대표 지점으로 삼았다. 이동 전용 OS 자원 자체는 G01 독립 문제로 남겼다.
