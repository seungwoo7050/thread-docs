## `docs: improve README with project visuals`

diff --git a/README.md b/README.md
index 29348f6..e1d84ef 100644
--- a/README.md
+++ b/README.md
@@ -1,20 +1,209 @@
-# irc-relay-server
+# IRC Relay Server
 
-단일 프로세스에서 여러 IRC 클라이언트의 연결과 메시지를 중계하는 C++17 서버를
-구현한다. 개발 과정에서는 운영체제별 이벤트 API를 공통 경계 뒤에 두고, 연결 수명과
-프로토콜 상태의 소유자를 명확하게 유지한다.
+IPv4 논블로킹 소켓을 단일 event loop에서 처리하는 C++17 IRC relay server입니다. Linux에서는 `epoll`, macOS에서는 `kqueue`를 사용하며 연결별 입력 버퍼, 송신 대기열, 등록 상태, nickname, channel, timeout과 rate limit를 메모리에 보관합니다.
 
-## 초기 개발 규약
+전체 IRC 사양을 구현하는 서버는 아닙니다. 현재 지원하는 command와 numeric, 연결 수명, backpressure, 종료 절차와 보안상 비범위를 공개 계약으로 고정합니다.
 
-- C++17과 `-Wall -Wextra -Werror`를 기본 컴파일 계약으로 사용한다.
-- Linux와 macOS의 논블로킹 이벤트 처리 경로를 함께 고려한다.
-- 소켓, 버퍼, 사용자 및 채널 상태는 수명과 정리 책임이 드러나는 객체가 소유한다.
-- 기능, 수정, 테스트, 문서와 CI 변경은 가능한 한 독립된 커밋으로 남긴다.
-- 각 개발 커밋은 깨끗한 상태에서 컴파일하고, 존재하는 검사를 모두 통과해야 한다.
-- 비밀번호나 생성된 빌드 산출물은 저장소에 기록하지 않는다.
+![논블로킹 event loop와 연결별 backpressure 구조](docs/images/event-loop-and-backpressure.svg)
 
-## 예정 범위
+## 한눈에 보기
 
-첫 개발 범위는 TCP 연결 수락, IRC 프레임 처리, 사용자 등록, 개인·채널 메시지 중계와
-기본 채널 상태 관리다. 서버 간 연동, 영속 저장소, TLS 종료와 전체 IRC 표준 구현은
-초기 범위에 포함하지 않는다.
+| 항목 | 내용 |
+| --- | --- |
+| 실행 파일 | `irc-relay-server` |
+| 언어 | C++17 |
+| 네트워크 | IPv4 TCP, 논블로킹 socket |
+| event backend | Linux `epoll`, macOS `kqueue` |
+| 상태 저장 | 프로세스 메모리 |
+| 지원 기능 | 등록, nickname, channel, 메시지, mode 일부, timeout, rate limit, metrics |
+| 주요 검증 | 송신 상태, 수명, TCP 계약, 다중 peer와 느린 수신자 진행 검사 |
+
+## 빌드와 실행
+
+```sh
+make
+./irc-relay-server 6667 relay-secret
+```
+
+Makefile은 `uname -s`가 `Linux` 또는 `Darwin`일 때 해당 event backend를 선택합니다. 기본 컴파일 조건은 다음과 같습니다.
+
+```text
+-std=c++17 -Wall -Wextra -Werror -g -Iinclude
+```
+
+전체 실행 형식은 다음과 같습니다.
+
+```text
+irc-relay-server <port> <password>
+  [--idle-timeout=120]
+  [--ping-timeout=30]
+  [--registration-timeout=30]
+  [--rate-limit=24:3]
+  [--max-pending-bytes=1048576]
+  [--max-connections=256]
+```
+
+| 옵션 | 의미 |
+| --- | --- |
+| `port` | 부호와 공백 없는 십진수 `1..65535` |
+| `password` | 등록 전 `PASS`에서 비교할 서버 비밀번호 |
+| `--idle-timeout` | 활동이 없는 연결에 PING을 보내기 전 시간 |
+| `--ping-timeout` | PONG을 기다리는 시간 |
+| `--registration-timeout` | 등록이 끝나지 않은 연결의 제한 시간 |
+| `--rate-limit=C:S` | S초 동안 허용할 command 수 C |
+| `--max-pending-bytes` | 연결별 미전송 논리 바이트 상한 |
+| `--max-connections` | 동시 연결 수 상한, 0은 무제한 |
+
+정상적으로 listen을 시작하면 표준 오류에 구조화된 시작 기록을, 표준 출력에 listen port를 기록합니다.
+
+```text
+event=server_started port=6667
+Listening on port 6667
+```
+
+## 지원 프로토콜
+
+등록 전에는 다음 command를 처리합니다.
+
+```text
+PASS  NICK  USER  PING  PONG  QUIT
+```
+
+등록 뒤에는 다음 부분집합을 제공합니다.
+
+```text
+PRIVMSG  JOIN  PART  TOPIC  KICK  INVITE
+MODE     LIST  NAMES METRICS
+```
+
+channel mode는 `+i`, `+t`, `+o`를 지원합니다. user mode 변경은 구현하지 않습니다. 입력은 CRLF와 LF-only frame을 모두 받고 출력은 CRLF로 정규화합니다.
+
+정확한 line 길이, 등록 전이, command별 인자와 numeric, channel 이름 비교, 초대와 operator 규칙은 [프로토콜 계약](docs/protocol-contract.md)에 정리되어 있습니다.
+
+## 요청 처리 과정
+
+```text
+listen socket
+  -> accept를 EAGAIN까지 반복
+  -> Connection 생성과 event backend 등록
+  -> recv를 EAGAIN까지 반복
+  -> 연결 입력 버퍼에 추가
+  -> 완성된 line 분리
+  -> command 파싱과 등록 상태 확인
+  -> nickname·channel 상태 변경
+  -> 응답을 연결별 송신 대기열에 추가
+  -> write-ready에서 send를 EAGAIN까지 반복
+  -> 완료된 바이트 제거
+  -> 오류·timeout·QUIT 시 모든 연결 상태 정리
+```
+
+`Server`는 socket과 event 준비 상태를 처리하고, 애플리케이션 계층은 IRC command와 사용자·channel 상태를 처리합니다. 외부에서 관찰되는 릴리스 범위는 실행 파일의 CLI, TCP wire, 표준 출력·오류와 종료 상태입니다. `include/*.hpp`는 설치용 ABI가 아니라 실행 파일 내부 모듈 경계입니다.
+
+## 연결과 소유권
+
+연결 객체는 `unique_ptr<Connection>`으로 소유합니다. event에서 받은 파일 디스크립터는 callback 사이에 연결이 제거될 수 있으므로 필요할 때 현재 연결 맵에서 다시 조회합니다.
+
+연결을 닫을 때는 다음 상태를 함께 제거합니다.
+
+- event backend 등록
+- socket 파일 디스크립터
+- nickname 인덱스
+- 참여 channel과 operator 상태
+- invite 상태
+- 입력 버퍼와 송신 대기열
+- timeout과 rate limit 기록
+
+callback에서 잡힌 `std::exception`은 연결 단위로 기록하고 해당 연결을 정리한 뒤 서버를 계속 실행할 수 있습니다. non-standard exception과 소멸 중 예외까지 격리한다고 주장하지 않습니다.
+
+## backpressure와 자원 상한
+
+`max-pending-bytes`는 아직 전송하지 않은 논리 바이트 수를 제한합니다. `std::string` capacity, allocator 부가 정보와 프로세스 RSS 전체의 상한은 아닙니다.
+
+다음 경계도 남아 있습니다.
+
+- `accept`, `recv`, `send`는 한 ready event에서 별도 처리량 제한 없이 `EAGAIN`까지 반복합니다.
+- 계속 준비된 한 연결이 다른 연결보다 오래 실행되는 경우의 공정성 상한이 없습니다.
+- 한 read callback은 완성된 모든 frame을 먼저 `vector`에 모으므로 callback 전체의 임시 메모리를 line limit만으로 제한하지 못합니다.
+- 정상 종료 요청 뒤 남은 출력은 write-ready를 기다리지만 연결별 drain 제한 시간은 없습니다.
+- socket 또는 event 오류에서는 남은 출력을 버릴 수 있습니다.
+
+## timeout과 rate limit
+
+- 등록이 끝나지 않은 연결은 registration timeout으로 제거합니다.
+- idle timeout이 지나면 PING을 보내고 ping timeout 동안 PONG을 기다립니다.
+- rate limit는 command 시각 deque를 사용해 지정한 시간 창의 명령 수를 제한합니다.
+- `rate-limit=0:SECONDS`는 거부만 끄고 시각 deque 갱신은 계속합니다.
+
+모든 시간 제한과 rate limit 상태는 서버 메모리에만 존재하며 재시작 뒤 복구하지 않습니다.
+
+## 종료
+
+SIGINT 또는 SIGTERM을 받으면 signal handler는 종료 요청 상태만 남깁니다. event loop가 일반 실행 문맥에서 다음 절차를 수행합니다.
+
+```text
+새 종료 요청 확인
+  -> 제한된 poll 반복으로 기존 연결 출력 drain 시도
+  -> listen socket과 연결 정리
+  -> event backend 해제
+  -> 정상 종료 상태 반환
+```
+
+유예는 `pollOnce(50)` 최대 8회의 wait budget입니다. listen socket이 그동안 열려 있어 새 연결과 명령을 받을 수 있으며 400ms wall-clock 종료를 보장하지 않습니다.
+
+## 네트워크 신뢰 범위
+
+서버는 기본 IPv4 `0.0.0.0`에 bind하고 TLS를 종료하지 않습니다. CLI 비밀번호와 클라이언트의 `PASS`는 암호화되지 않은 TCP를 통과합니다.
+
+이 저장소에는 다음 기능이 없습니다.
+
+- TLS 인증서와 암호화
+- 저장된 비밀번호 hash와 계정 데이터베이스
+- SASL
+- 외부 인증 공급자
+- 네트워크 접근 제어
+
+신뢰할 수 없는 네트워크에서는 별도의 TLS endpoint와 방화벽·접근 통제가 필요합니다.
+
+## 검증
+
+```sh
+make test
+make connection-test
+make unit
+make application-test
+make event-test
+```
+
+`make test`는 다음 검사를 포함합니다.
+
+- 연결별 송신 대기열과 짧은 `send`
+- server와 application 객체 수명
+- 등록·nickname·channel command 계약
+- TCP smoke와 wire 응답
+- 160개 peer 동시 연결
+- 느린 수신자가 있을 때 다른 연결의 진행
+- timeout과 rate limit 상태
+- 연결 제거 뒤 nickname·channel 정리
+
+고정된 유한 시나리오는 지속적인 fast sender가 event loop를 독점하는 경우, 모든 IRC numeric, 모든 allocation 실패, non-standard exception, 실제 파일 디스크립터 재사용과 wall-clock 성능 상한을 증명하지 않습니다.
+
+## 문서
+
+- [프로토콜 계약](docs/protocol-contract.md)
+- [socket에서 channel 메시지까지](architecture/socket-to-channel-message.md)
+- [연결 상태와 보호 수명](architecture/connection-state-and-protection-lifecycle.md)
+- [이식 가능한 event loop](architecture/portable-event-loop.md)
+- [종료·지표·실행 계약](architecture/shutdown-metrics-and-runtime-contract.md)
+
+## 제한 사항
+
+- IPv6와 서버 간 federation을 지원하지 않습니다.
+- channel·message 상태를 재시작 뒤 보존하지 않습니다.
+- 전체 IRC command, numeric, casemapping과 user mode를 구현하지 않습니다.
+- ban, key, limit mode와 message history가 없습니다.
+- 처리량 quantum이 없어 모든 연결에 대한 엄격한 공정성을 보장하지 않습니다.
+- 장기 ABI 호환 라이브러리와 설치 규칙을 제공하지 않습니다.
+
+## 프로젝트 배경
+
+이 저장소는 42의 `ft_irc`에서 출발했습니다. 현재 구현은 C++17로 재설계되었으며 Linux `epoll`과 macOS `kqueue`, 연결별 backpressure, 등록·idle timeout, rate limit, 구조화된 상태 정리와 다중 peer 회귀 검사를 추가했습니다.
diff --git a/docs/images/event-loop-and-backpressure.svg b/docs/images/event-loop-and-backpressure.svg
new file mode 100644
index 0000000..74222c4
--- /dev/null
+++ b/docs/images/event-loop-and-backpressure.svg
@@ -0,0 +1,19 @@
+<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1200 620" role="img" aria-labelledby="title desc">
+  <title id="title">IRC event loop and per-connection backpressure</title>
+  <desc id="desc">Ready events drive accept, receive, parse, application state, queued responses, and nonblocking send. Each connection owns buffers, registration, nickname, channels, and time limits.</desc>
+  <defs><marker id="arrow" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L9,3 z" fill="#66d4ff"/></marker>
+  <style>.bg{fill:#0b1020}.box{fill:#151d31;stroke:#344463;stroke-width:2}.accent{fill:#112d3c;stroke:#66d4ff;stroke-width:2}.warn{fill:#3a2c17;stroke:#f6c665;stroke-width:2}.line{stroke:#66d4ff;stroke-width:3;fill:none;marker-end:url(#arrow)}.t{fill:#f4f7ff;font:600 22px -apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif}.s{fill:#abb9d2;font:16px -apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif}.m{fill:#dce8ff;font:16px ui-monospace,SFMono-Regular,Menlo,monospace}.label{fill:#66d4ff;font:600 15px -apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif}</style></defs>
+  <rect class="bg" width="1200" height="620" rx="24"/><text class="t" x="55" y="60">One event loop, isolated connection state</text>
+  <rect class="box" x="55" y="115" width="170" height="100" rx="15"/><text class="t" x="92" y="157">TCP peers</text><text class="s" x="83" y="188">nonblocking IPv4</text>
+  <rect class="accent" x="300" y="100" width="210" height="130" rx="16"/><text class="t" x="347" y="148">ready set</text><text class="m" x="341" y="181">epoll / kqueue</text><text class="s" x="341" y="207">fd looked up again</text>
+  <rect class="box" x="585" y="100" width="220" height="130" rx="16"/><text class="t" x="644" y="148">receive</text><text class="m" x="624" y="181">recv → line frames</text><text class="s" x="624" y="207">until EAGAIN</text>
+  <rect class="box" x="880" y="100" width="265" height="130" rx="16"/><text class="t" x="929" y="148">IRC application</text><text class="s" x="923" y="181">registration · nick · channel</text><text class="s" x="923" y="207">command and numeric</text>
+  <path class="line" d="M225 165 H290"/><path class="line" d="M510 165 H575"/><path class="line" d="M805 165 H870"/>
+  <rect class="box" x="880" y="335" width="265" height="135" rx="16"/><text class="t" x="918" y="382">pending output</text><text class="s" x="919" y="414">logical byte accounting</text><text class="m" x="919" y="445">max-pending-bytes</text>
+  <rect class="accent" x="585" y="335" width="220" height="135" rx="16"/><text class="t" x="654" y="382">send</text><text class="s" x="625" y="414">advance by bytes written</text><text class="m" x="625" y="445">until EAGAIN</text>
+  <path class="line" d="M1012 230 V325"/><path class="line" d="M880 402 H815"/><path class="line" d="M585 402 H405 V240"/>
+  <text class="label" x="425" y="389">write-ready interest</text>
+  <rect class="warn" x="55" y="335" width="390" height="210" rx="18"/><text class="t" x="90" y="380">Per-connection ownership</text>
+  <text class="m" x="90" y="420">input buffer · output queue</text><text class="m" x="90" y="453">registration · nickname · channels</text><text class="m" x="90" y="486">idle/ping timeout · rate window</text><text class="s" x="90" y="522">close removes every index and event registration</text>
+  <rect class="box" x="585" y="520" width="560" height="55" rx="14"/><text class="s" x="623" y="554">Bounded slow receivers; no strict fairness or drain deadline.</text>
+</svg>
