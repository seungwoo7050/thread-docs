# DevThread 개발자 기술면접 워크북 — 마스터 인덱스

## 범위와 선별 원칙

이 인덱스는 현재 GPT 프로젝트에 축적된 DevThread 01~15의 커밋 학습 기록에서 실제로 확인되는 내용만 사용한다. 같은 역량을 여러 Thread가 반복한 경우 가장 대표적인 지점을 상세 문제로 삼고, 나머지는 연관 Thread 또는 통합 근거로 연결했다. 파일명·함수명·클래스명은 작업 기록에서 확인되는 경우에만 적었다.

우선순위의 의미는 다음과 같다.

- **S**: 질문과 10~30분 직접 구현 모두 준비해야 하는 핵심 지점
- **A**: 질문 가치가 높고, 핵심 구현 또는 테스트 구현 가능성이 큰 지점
- **B**: 구현보다 설계 의도·제약·trade-off 설명을 준비할 지점
- **C**: 별도 면접 문제로 만들기보다 맥락만 알고 있으면 되는 지점

상세 워크북 ID는 `P01`~`P17`이며, 선별 표의 S/A 행은 모두 해당 ID와 파일 경로를 가진다.

## 상세 워크북 구성

| 파일 | 포함 면접 포인트 | 역할 |
| --- | --- | --- |
| `01-event-loop-and-lifetime.md` | P01~P03 | 이벤트 백엔드 정규화, 이벤트 루프 상태 전이, 콜백·실패 경계의 객체 수명과 롤백 |
| `02-stream-io-and-protocol.md` | P04~P06 | 논블로킹 수신, 부분 송신과 backpressure, IRC 문법 파싱·직렬화 |
| `03-state-invariants-and-authorization.md` | P07~P10 | 등록·nickname 색인, channel invariant와 정리, 권한 검사, 대상 해석과 fan-out |
| `04-time-and-resource-protection.md` | P11~P14 | 오버플로 안전 파싱, heartbeat 상태 기계, rate limit, 계층형 자원 제한 |
| `05-shutdown-and-verification.md` | P15~P17 | graceful shutdown, 결정적 실패 주입, 실제 TCP 계약·부하 검증 |

## 전체 Thread/커밋 선별 결과

동일 기능을 완성하는 연속 커밋은 한 행에 묶었다. `핵심 면접 주제`의 P 번호는 상세 워크북 위치를 뜻한다.

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| S | 01 | `feat(event): kqueue 관심 상태 등록 구현` · `feat(event): kqueue 준비 이벤트 변환 구현` · `feat(event): epoll 관심 상태 등록 구현` · `feat(event): epoll 준비 이벤트 변환 구현` | `include/EventManager.hpp`, `src/KqueueEventManager.cpp`, `src/EpollEventManager.cpp`, `EventInterest`, `Event`, `EventManager` | **P01** 플랫폼 이벤트를 공통 계약으로 정규화 — `01-event-loop-and-lifetime.md` | OS별 플래그·오류·EOF 의미를 공통 추상화로 보존하는 문제는 시스템 경계 설계와 오류 모델을 함께 묻기 좋다. | 높음 | 높음 | 03, 15 |
| C | 01 | `build(project): 플랫폼별 IRC 서버 빌드 구성` | `Makefile` | 플랫폼별 소스 선택과 빌드 조건 | 필요한 설정이지만 핵심 구현 판단보다 빌드 배선 비중이 크다. | 낮음 | 낮음 | 15 |
| S | 02 | `feat(connection): IRC 입력 프레임 추출 구현` · `feat(connection): 논블로킹 수신 상태 처리` | `include/Connection.hpp`, `src/Connection.cpp`, `Connection::extractLines`, `Connection::readAvailable` | **P04** TCP 스트림에서 완전한 IRC 프레임 추출 — `02-stream-io-and-protocol.md` | TCP 조각화·병합, `EINTR`, `EAGAIN`, EOF, 길이 제한을 동시에 다루며 네트워크 기본기를 직접 확인할 수 있다. | 높음 | 높음 | 03, 04, 09 |
| S | 02 | `feat(connection): 부분 송신 대기열 처리` · `feat(buffer): 송신 대기열 크기 제한` | `src/Connection.cpp`, `src/ConnectionLimits.hpp`, `Connection::queueRaw`, `Connection::queueLine`, `Connection::flushPending`, `Connection::pendingBytes`, `Connection::wantsWrite` | **P05** 부분 송신과 연결별 backpressure — `02-stream-io-and-protocol.md` | 논블로킹 `send`의 부분 진행, offset, 쓰기 관심 등록, 대기열 상한과 실패 시 불변조건을 묻는 대표 문제다. | 높음 | 높음 | 03, 10, 13 |
| S | 03 | `feat(server): 논블로킹 연결 수락과 등록 구현` · `feat(server): 준비 이벤트 루프 구동` · `feat(server): 클라이언트 입력 이벤트 전달` · `feat(server): 송신 큐와 쓰기 관심 상태 연결` · `feat(server): 연결 해제와 오류 정리 구현` | `include/Server.hpp`, `src/Server.cpp`, `Server::pollOnce`, `Server::acceptReadyClients`, `Server::handleClientEvent`, `Server::refreshInterest`, `Server::disconnect` | **P02** 단일 이벤트 루프의 관심 상태와 종료 전이 — `01-event-loop-and-lifetime.md` | read/write/error/hangup 순서와 close-drain 정책이 전체 서버의 진행성·정합성을 결정한다. | 높음 | 높음 | 01, 02, 10, 14 |
| S | 04 | `feat(parser): IRC 메시지 값과 직렬화 정의` · `feat(parser): IRC 한 줄 구문 해석 구현` | `include/IrcMessage.hpp`, `src/IrcMessage.cpp`, `src/Replies.cpp`, `IrcMessage::parseLine`, `IrcMessage::toLine`, `Replies` | **P06** IRC 한 줄 문법 파싱과 대칭 직렬화 — `02-stream-io-and-protocol.md` | prefix·command·middle/trailing parameter·CRLF를 엄격히 처리하는 작은 파서로 문자열 경계 처리 능력을 확인할 수 있다. | 높음 | 높음 | 02, 09, 14 |
| B | 04 | `feat(parser): 버퍼에서 IRC 프레임 소비` | `src/IrcMessage.cpp`, `IrcMessage::consumeBuffer` | 스트림 framing과 문법 파싱의 책임 분리 | 이후 연결 계층 framing과 역량이 겹친다. 별도 문제보다 P04/P06에서 경계 설계를 설명하는 편이 낫다. | 중간 | 낮음 | 02 |
| S | 05 | `feat(client): 연결별 등록 상태 저장` · `feat(client): 닉네임 색인 관리` · `feat(registration): PASS 인증 상태 처리` · `feat(registration): 닉네임 검증과 색인 갱신` · `feat(registration): USER 정보와 환영 응답 연결` | `src/ClientRegistry.hpp`, `src/ClientRegistry.cpp`, `src/RegistrationCommands.cpp`, `ClientState`, `ClientRegistry`, `IrcApplication::maybeRegister` | **P07** 등록 상태 기계와 nickname 보조 색인 — `03-state-invariants-and-authorization.md` | 여러 명령의 순서가 자유로운 상태 기계와 `fd↔nickname` 색인의 원자적 갱신을 함께 묻는다. | 높음 | 높음 | 06, 09, 10, 12 |
| S | 06 | `feat(channel): 채널 상태 계약 정의` · `feat(channel): 구성원과 운영자 상태 관리` · `feat(channel): 주제·초대·모드와 이름 규칙 구현` | `include/Channel.hpp`, `src/Channel.cpp`, `Channel` | **P08** channel membership·operator·invite invariant — `03-state-invariants-and-authorization.md` | 운영자는 반드시 구성원이어야 하고, 탈퇴·삭제가 여러 집합을 함께 갱신해야 하는 전형적인 상태 정합성 문제다. | 높음 | 높음 | 07, 08, 10 |
| S | 06 | `feat(channel): 구성원 정리와 식별자 변경 방송` | `src/ApplicationSupport.cpp`, `src/RegistrationCommands.cpp`, `IrcApplication::removeClientState`, `IrcApplication::partAllChannels`, `IrcApplication::eraseChannelIfEmpty` | **P08** 연결 종료·nickname 변경 시 역색인과 channel 정리 — `03-state-invariants-and-authorization.md` | 연결 하나의 제거가 nickname 색인·여러 channel·공통 peer 알림에 연쇄 영향을 주므로 누락·중복·순서 오류가 생기기 쉽다. | 높음 | 높음 | 05, 08, 10 |
| A | 07 | `feat(channel): JOIN 채널 입장 처리` · `feat(channel): PART 채널 퇴장 처리` · `feat(channel): KICK 구성원 제거 처리` · `feat(channel): 채널 운영자 모드 변경` | `src/ChannelCommands.cpp`, `IrcApplication::handleJoin`, `handlePart`, `handleTopic`, `handleKick`, `handleInvite`, `handleMode`, `handleChannelMode` | **P09** 권한 검사와 다단계 channel 명령 — `03-state-invariants-and-authorization.md` | 존재 확인, 구성원 여부, operator 권한, 대상 상태, 방송과 상태 변경 순서를 명확히 설명할 수 있어야 한다. | 높음 | 중간 | 06, 10 |
| B | 07 | `feat(room): 채널 목록 조회 구현` · `test(client): 채널 목록 응답 검증` | `src/ChannelCommands.cpp`, `IrcApplication::handleList`, `IrcApplication::handleNames`, `tools/irc_smoke_client.py` | LIST/NAMES 응답 계약과 반복 송신 중단 | 프로토콜 계약 설명에는 가치가 있지만 핵심 자료구조·알고리즘은 다른 항목과 겹친다. | 중간 | 낮음 | 10, 14 |
| A | 08 | `feat(message): 등록 사용자의 개인 메시지 전달` · `feat(channel): 채널 탐색과 대상 해석 지원` · `feat(message): 채널 대상 메시지 방송` | `src/MessagingCommands.cpp`, `src/ApplicationSupport.cpp`, `IrcApplication::handlePrivmsg`, `findNick`, `findChannelForCommand`, `broadcastToChannel`, `broadcastToCommon` | **P10** 대상 해석과 수신자 fan-out·중복 제거 — `03-state-invariants-and-authorization.md` | direct/channel 대상의 오류 모델, 송신자 제외, 여러 공통 channel에서의 중복 알림 제거를 일반화할 수 있다. | 높음 | 중간 | 05, 06, 07 |
| B | 09 | `feat(app): 등록 전 명령 분배 구현` · `feat(app): 응답과 클라이언트 정리 지원 구현` | `src/IrcApplication.cpp`, `src/IrcApplication.hpp`, `src/ApplicationSupport.cpp`, `src/main.cpp`, `IrcApplication::onLine`, `handleMessage`, `onDisconnect` | socket→parser→application→queue 경계 설명 | 전체 흐름을 설명하는 데 중요하지만 개별 핵심 구현은 P02, P04, P06, P07에 포함된다. | 높음 | 낮음 | 02, 03, 04, 05 |
| A | 09 | `test(smoke): 실제 TCP 등록과 채널 흐름 검증` | `tests/irc_smoke.sh`, `tools/irc_smoke_client.py` | **P17** 실제 TCP에서 관찰 가능한 계약 검증 — `05-shutdown-and-verification.md` | 메모리 내부 단위 테스트만으로 놓치는 framing·응답 순서·다중 peer 상호작용을 검증한다. | 높음 | 중간 | 13, 14, 15 |
| B | 09 | `docs: improve README with project visuals` | `README.md`, `architecture/socket-to-channel-message.md`, `architecture/connection-state-and-protection-lifecycle.md`, `architecture/portable-event-loop.md`, `architecture/shutdown-metrics-and-runtime-contract.md` | 보안·신뢰 경계와 비목표 설명 | TLS 없음, 평문 PASS, 프로세스 메모리 상태, 엄격한 공정성 비보장 등 한계를 숨기지 않고 설명할 수 있어야 한다. | 높음 | 낮음 | 13, 14 |
| C | 09 | `docs: improve README with project visuals` | `docs/images/event-loop-and-backpressure.svg` | 문서 시각화 | 설계 전달에는 도움이 되지만 별도 기술면접 준비 지점으로 만들 필요는 낮다. | 낮음 | 낮음 | 01, 02, 03 |
| S | 10 | `test(server): 연결 제거와 이벤트 등록 실패 경로 검증` | `tests/server_lifetime_test.cpp`, `src/Server.cpp`, `FakeEventManager`, `registrationRollbackTest`, `connectCallbackLifetimeTest`, `lineCallbackLifetimeTest`, `interestUpdateRollbackTest`, `queueLimitCloseTest` | **P03** 콜백 중 제거와 이벤트 등록 실패의 롤백 — `01-event-loop-and-lifetime.md` | 콜백이 현재 객체를 삭제하거나 이벤트 백엔드 갱신이 실패할 수 있다는 가정 아래 dangling reference와 반쯤 등록된 상태를 막아야 한다. | 높음 | 높음 | 03, 15 |
| S | 10 | `test(app): 작은 송신 한도에서 상태 정리 검증` · `fix(app): 응답 실패 뒤 명령 처리를 중단` | `tests/application_lifetime_test.cpp`, `src/ChannelCommands.cpp`, `src/ApplicationSupport.cpp`, `IrcApplication::sendRaw`, `sendNumeric`, `sendNames`, `broadcastMode` | **P03/P09 통합** 송신 실패가 곧 연결 제거일 때 후속 로직 중단 — `01-event-loop-and-lifetime.md`, `03-state-invariants-and-authorization.md` | 출력 함수가 단순 부수효과가 아니라 객체 수명을 끝낼 수 있는 경계다. 안정 키로 재탐색하고 이후 변이를 중단해야 한다. | 높음 | 높음 | 03, 06, 07 |
| S | 11 | `fix(config): 서버 크기 옵션을 오버플로 없이 해석` · `test(config): 크기 옵션 경계와 오류 입력 검증` | `src/RuntimeConfig.cpp`, `src/RuntimeConfig.hpp`, `RuntimeConfig::parsePort`, `RuntimeConfig::parseSize`, `parseUnsignedDecimal` | **P11** 오버플로 전 검사하는 십진수 파서 — `04-time-and-resource-protection.md` | 부호·공백·빈 문자열·최댓값·오버플로를 라이브러리 부작용 없이 처리하는 짧고 면접 친화적인 구현 문제다. | 높음 | 높음 | 13, 14 |
| A | 11 | `feat(throttle): 클라이언트별 명령 호출 횟수 제한` | `src/ClientRegistry.hpp`, `src/IrcApplication.cpp`, `ClientState::commandWindow`, `IrcApplication::recordCommand` | **P13** sliding-window rate limit — `04-time-and-resource-protection.md` | 시간 구간 만료, 경계 시각, 연결별 격리, 메모리 상한과 정책 trade-off를 묻기 좋다. | 높음 | 높음 | 12, 13 |
| B | 11 | `feat(config): 기본 실행 인자 해석 모듈 구성` | `src/RuntimeConfig.hpp`, `src/RuntimeConfig.cpp`, `RuntimeConfig::parseOptions`, `RuntimeConfig::printUsage` | 런타임 정책과 서버 설정 경계 | 옵션 나열보다 파싱 실패 계약과 정책 주입 방식을 설명하는 정도가 적절하다. | 중간 | 낮음 | 12, 13 |
| A | 12 | `feat(registration): 등록 대기 시간 제한` · `feat(heartbeat): 유휴 연결에 PING을 보내고 응답 대기` · `fix(heartbeat): 단조 시계와 토큰으로 응답 대기 상태 관리` | `src/ClientRegistry.hpp`, `src/IrcApplication.cpp`, `src/RegistrationCommands.cpp`, `IrcApplication::onTick`, `maintainClient`, `handlePong`, `MonotonicClock`, `pendingPongToken` | **P12** 단조 시계 기반 timeout·heartbeat 상태 기계 — `04-time-and-resource-protection.md` | wall clock 변경을 피하고, 이전 PONG이나 임의 PONG이 현재 probe를 해제하지 못하도록 상태와 토큰을 검증해야 한다. | 높음 | 높음 | 05, 11, 14 |
| B | 12 | `test(client): 유휴 연결의 PING·PONG 흐름 검증` · `refactor(command): 명령 처리 시각 기록 통합` | `tools/irc_smoke_client.py`, `tests/irc_smoke.sh`, `src/IrcApplication.cpp` | 시간 기반 기능의 관찰 가능성 | 테스트 자체는 P12와 P17에서 함께 다루며 별도 문제로 늘릴 필요는 없다. | 중간 | 낮음 | 15 |
| A | 13 | `feat(buffer): 송신 대기열 크기 제한` · `feat(server): 최대 연결 수 제한` | `src/Connection.cpp`, `src/ConnectionLimits.hpp`, `src/Server.cpp`, `src/RuntimeConfig.cpp`, `Server::acceptReadyClients` | **P14** 연결별·서버 전체 계층형 자원 제한 — `04-time-and-resource-protection.md` | 한 연결의 메모리 폭주와 전체 fd 고갈은 다른 층의 문제이며, 거절 시 정리와 기존 연결의 진행을 보존해야 한다. | 높음 | 중간 | 02, 03, 11 |
| A | 13 | `test(event): 160개 연결과 느린 수신자 처리 공정성 검증` | `tests/irc_event_fairness.py`, `check_many_connections`, `check_slow_receiver_isolation` | **P14/P17 통합** slow receiver 격리와 진행성 검증 — `04-time-and-resource-protection.md`, `05-shutdown-and-verification.md` | 느린 수신자가 event loop를 막지 않는다는 시스템 성질을 실제 소켓과 여러 peer로 검증한다. | 높음 | 중간 | 02, 03, 15 |
| B | 14 | `test(client): 서버 지표 명령 검증` | `src/Server.cpp`, `src/IrcApplication.cpp`, `src/ApplicationSupport.cpp`, `Server::Metrics`, `AppMetrics`, `Server::metrics`, `METRICS` 처리 | 지표의 소유권·증가 시점·안정된 직렬화 | 운영 관점의 질문 가치는 높지만 카운터 추가 자체를 별도 백지 구현 문제로 만들 필요는 낮다. | 높음 | 낮음 | 09, 15 |
| A | 14 | `feat(shutdown): 종료 전 송신 대기열 처리` | `src/main.cpp`, `src/IrcApplication.cpp`, `Server::pollOnce`, `Server::stop`, `IrcApplication::shutdown` | **P15** bounded drain을 포함한 graceful shutdown — `05-shutdown-and-verification.md` | 종료 신호 처리, 새 작업 차단, ERROR 전송, 제한된 drain, 강제 정리의 순서와 보장 범위를 설명해야 한다. | 높음 | 중간 | 03, 10, 15 |
| A | 14 | `test(irc): 실행 조건과 오류 동작 계약 검증` | `tests/irc_contract.py`, `check_cli_contract`, `check_wire_contract`, `validate_shutdown_log` | **P17** CLI·wire·로그 순서 계약 — `05-shutdown-and-verification.md` | 정확한 CRLF·numeric·응답 순서·종료 로그 순서를 외부 관찰자로 검증하는 계약 테스트 설계를 묻기 좋다. | 높음 | 중간 | 09, 11, 12, 15 |
| A | 15 | `test(server): 연결 제거와 이벤트 등록 실패 경로 검증` · `test(app): 작은 송신 한도에서 상태 정리 검증` | `tests/server_lifetime_test.cpp`, `tests/application_lifetime_test.cpp`, `FakeEventManager`, `CapturedStderr` | **P16** 결정적 실패 주입과 수명 테스트 — `05-shutdown-and-verification.md` | 운영체제의 우연한 실패를 기다리지 않고 특정 호출만 실패시켜 rollback·cleanup·무관 상태 보존을 검증한다. | 높음 | 높음 | 10 |
| A | 15 | `test(event): 160개 연결과 느린 수신자 처리 공정성 검증` | `tests/irc_event_fairness.py`, `selectors.DefaultSelector`, `Peer` | **P17** 다중 연결·slow receiver 시스템 검증 — `05-shutdown-and-verification.md` | 단위 테스트가 보장하지 못하는 실제 scheduler·socket buffer·다중 연결 상호작용을 검증한다. | 높음 | 중간 | 02, 13 |
| B | 15 | `ci: Linux·macOS 회귀와 새니타이저 자동화` | `.github/workflows/cpp-ft-irc-ci.yml`, `Makefile` | epoll/kqueue 교차 플랫폼 회귀와 ASan·UBSan | 테스트 전략 설명에는 중요하지만 워크플로 문법 자체는 면접 핵심 구현이 아니다. | 중간 | 낮음 | 01, 10, 13 |
| C | 15 | `ci: harden cross-platform verification` | `.github/workflows/cpp-ft-irc-ci.yml`, `Makefile`, `tests/irc_smoke.sh` | 도구 버전 고정, 권한 축소, locale/timezone 고정, 실행기 override | 재현성 강화 세부는 유익하지만 별도 문제보다 CI 설명의 근거로 충분하다. | 낮음 | 낮음 | 14 |

## 대표 면접 포인트와 연관 Thread

| ID | 대표 Thread | 통합·연관 Thread | 상세 파일 |
| --- | --- | --- | --- |
| P01 | 01 | 03, 15 | `01-event-loop-and-lifetime.md` |
| P02 | 03 | 01, 02, 10, 14 | `01-event-loop-and-lifetime.md` |
| P03 | 10 | 03, 06, 07, 15 | `01-event-loop-and-lifetime.md` |
| P04 | 02 | 03, 04, 09 | `02-stream-io-and-protocol.md` |
| P05 | 02 | 03, 10, 13 | `02-stream-io-and-protocol.md` |
| P06 | 04 | 02, 09, 14 | `02-stream-io-and-protocol.md` |
| P07 | 05 | 06, 09, 10, 12 | `03-state-invariants-and-authorization.md` |
| P08 | 06 | 05, 07, 08, 10 | `03-state-invariants-and-authorization.md` |
| P09 | 07 | 06, 10 | `03-state-invariants-and-authorization.md` |
| P10 | 08 | 05, 06, 07 | `03-state-invariants-and-authorization.md` |
| P11 | 11 | 13, 14 | `04-time-and-resource-protection.md` |
| P12 | 12 | 05, 11, 14 | `04-time-and-resource-protection.md` |
| P13 | 11 | 12, 13 | `04-time-and-resource-protection.md` |
| P14 | 13 | 02, 03, 11, 15 | `04-time-and-resource-protection.md` |
| P15 | 14 | 03, 10, 15 | `05-shutdown-and-verification.md` |
| P16 | 15 | 10 | `05-shutdown-and-verification.md` |
| P17 | 15 | 09, 11, 12, 13, 14 | `05-shutdown-and-verification.md` |

## S/A 완전성 검증

| S/A 선별 행 | 상태 | 상세 워크북 반영 |
| --- | --- | --- |
| Thread 01 이벤트 백엔드 등록·변환 | 독립 항목 | P01 |
| Thread 02 입력 framing·수신 | 독립 항목 | P04 |
| Thread 02 부분 송신·대기열 제한 | 독립 항목 | P05 |
| Thread 03 이벤트 루프·관심 상태 | 독립 항목 | P02 |
| Thread 04 파서·직렬화 | 독립 항목 | P06 |
| Thread 05 등록·nickname 색인 | 독립 항목 | P07 |
| Thread 06 channel invariant | 독립 항목 | P08 |
| Thread 06 연결·nickname 정리 | P08에 명시적으로 통합 | P08 |
| Thread 07 channel 권한 명령 | 독립 항목 | P09 |
| Thread 08 대상 해석·fan-out | 독립 항목 | P10 |
| Thread 09 실제 TCP smoke | P17에 명시적으로 통합 | P17 |
| Thread 10 server 수명·rollback | 독립 항목 | P03 |
| Thread 10 응답 실패 뒤 중단 | P03과 P09에 명시적으로 통합 | P03, P09 |
| Thread 11 오버플로 안전 파싱 | 독립 항목 | P11 |
| Thread 11 rate limit | 독립 항목 | P13 |
| Thread 12 timeout·heartbeat | 독립 항목 | P12 |
| Thread 13 계층형 자원 제한 | 독립 항목 | P14 |
| Thread 13 slow receiver 검증 | P14와 P17에 명시적으로 통합 | P14, P17 |
| Thread 14 graceful shutdown | 독립 항목 | P15 |
| Thread 14 CLI·wire·로그 계약 | P17에 명시적으로 통합 | P17 |
| Thread 15 실패 주입 테스트 | 독립 항목 | P16 |
| Thread 15 다중 연결 시스템 테스트 | P17에 명시적으로 통합 | P17 |

## 백지 구현 우선순위

1. **P05** 부분 송신 대기열과 backpressure
2. **P04** TCP 스트림 framing과 논블로킹 수신
3. **P11** 오버플로 안전 십진수 파서
4. **P03** 콜백 중 제거와 등록 실패 rollback
5. **P07** 등록 상태 기계와 nickname 보조 색인
6. **P08** channel membership·operator invariant와 연결 정리
7. **P02** 이벤트 관심 상태와 close-drain 전이
8. **P06** IRC 한 줄 파서와 직렬화
9. **P12** heartbeat timeout 상태 기계
10. **P13** sliding-window rate limit
11. **P01** 이벤트 백엔드 결과 정규화
12. **P10** fan-out 수신자 중복 제거
13. **P15** bounded graceful shutdown coordinator
14. **P16** 실패 주입 테스트 대역
15. **P14** 계층형 admission·queue 정책
16. **P09** 권한 검사를 포함한 channel 명령 전이
17. **P17** 실제 TCP 계약 테스트 harness

## 설명 연습 우선순위

1. **P03** 왜 콜백과 송신 함수가 객체 수명 경계인지
2. **P02** read/write/error/hangup 처리 순서와 close-drain 보장
3. **P05** 연결별 backpressure가 slow receiver를 격리하는 방식과 한계
4. **P08** 여러 자료구조에 걸친 invariant와 cleanup 순서
5. **P12** `steady_clock`과 PONG token 검증이 필요한 이유
6. **P15** graceful shutdown에서 보장하는 것과 보장하지 못하는 것
7. **P17** 단위·계약·부하·교차 플랫폼 검증이 각각 잡는 결함
8. **P01** epoll/kqueue 차이를 공통 이벤트로 숨길 때 잃지 말아야 할 정보
9. **P14** 연결별 제한과 서버 전체 제한을 분리하는 이유
10. **P09** 권한 검사, 방송, 상태 변경 사이의 실패 경계
11. **P07** 등록 완료 조건과 nickname 색인의 원자성
12. **P06** framing과 grammar parsing을 분리하는 이유
13. **P13** 고정 창·sliding window·token bucket trade-off
14. **P10** 공통 channel 방송의 중복·누락 방지
15. **P11** 변환 후 범위 검사보다 변환 전 오버플로 검사가 안전한 이유

## 한 문제로 통합한 Thread 묶음

1. **P01**: Thread 01의 epoll/kqueue 구현 + Thread 03의 공통 이벤트 소비 + Thread 15의 교차 플랫폼 검증
2. **P02**: Thread 03 이벤트 루프 + Thread 02 송수신 상태 + Thread 10 수명 안전성 + Thread 14 종료 drain
3. **P03**: Thread 10 server/application 수명 테스트 + Thread 03 콜백 경계 + Thread 07 다단계 명령 중단
4. **P04·P06**: Thread 02 TCP framing + Thread 04 IRC 문법 파싱 + Thread 09 split-frame smoke
5. **P05·P14**: Thread 02 연결별 송신 대기열 + Thread 13 계층형 제한·slow receiver 격리
6. **P07·P08**: Thread 05 등록·nickname 색인 + Thread 06 channel membership·disconnect cleanup
7. **P09**: Thread 07 권한 명령 + Thread 10 응답 실패 뒤 재탐색·중단
8. **P10**: Thread 08 대상 해석·fan-out + Thread 06 공통 peer dedup·cleanup
9. **P12·P13**: Thread 12 timeout·heartbeat + Thread 11 명령 호출 제한 + Thread 05 연결별 상태
10. **P15**: Thread 14 graceful shutdown + Thread 03 close-drain 상태 + Thread 10 안전한 제거
11. **P16**: Thread 10 실패 경계 회귀 테스트 + Thread 15 결정적 검증 계층
12. **P17**: Thread 09 TCP smoke + Thread 13 event fairness + Thread 14 CLI·wire·shutdown 계약 + Thread 15 Linux/macOS 회귀
