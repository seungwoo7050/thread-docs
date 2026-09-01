# ft-irc (IRC Relay Server)

## 1. 이력서용 프로젝트 설명

C++17로 등록·채널·메시징 기능을 제공하는 IRC 부분 프로토콜 서버를 구현했습니다.  
Linux `epoll`과 macOS `kqueue`를 공통 인터페이스로 추상화하고, 단일 non-blocking event loop에서 연결별 입력 버퍼와 송신 대기열을 관리했습니다.  
partial send·`EINTR`·`EAGAIN`에서도 전송 위치를 보존하며, 대기열 상한·등록/유휴 timeout·rate limit와 연결 종료 시 채널 상태 정리를 구현했습니다.  
단위·수명·TCP 프로토콜 검사와 160개 동시 peer 및 한 느린 수신자 시나리오를 로컬 테스트로 검증했습니다.

## 2. 30초 프로젝트 소개

이 프로젝트는 C++17로 만든 IRC 부분 프로토콜 서버입니다. Linux의 `epoll`과 macOS의 `kqueue`를 공통 인터페이스로 묶고, 단일 non-blocking event loop에서 연결을 처리했습니다. 연결별 송신 큐가 partial send와 `EAGAIN` 이후에도 전송 위치를 보존하도록 설계한 점이 핵심이며, 파일 디스크립터와 채널 상태를 함께 정리하는 수명 관리에 집중했습니다.

## 3. 2분 프로젝트 소개

목표는 IRC 전체 표준을 재현하는 것이 아니라, 등록과 채널 메시징에 필요한 부분 프로토콜을 non-blocking 서버 구조로 구현하는 것이었습니다. 소켓과 readiness event는 `Server`가, nickname·channel·operator 상태는 애플리케이션 계층이 담당합니다. Linux `epoll`과 macOS `kqueue`를 같은 인터페이스 뒤에 두고, `accept`와 `recv`는 `EAGAIN`까지 처리합니다. 송신은 연결별 버퍼와 offset으로 partial send와 `EINTR`를 이어 처리하고, `EAGAIN`이면 남은 데이터를 다음 event까지 보존합니다. callback이 연결을 제거할 수 있어 연결은 `unique_ptr`로 소유하되 callback 이후에는 파일 디스크립터로 다시 조회했습니다. 연결 종료 때 nickname과 channel 인덱스도 함께 제거합니다. 대기열 상한과 등록·유휴 timeout도 연결 상태에 포함하고, 종료 signal은 요청만 기록한 뒤 일반 문맥에서 출력 drain과 정리를 수행합니다. 로컬 `make test`에서 송신 상태, 객체 수명, TCP 명령 계약과 160개 peer·한 느린 수신자 시나리오가 통과했습니다. 다만 지원 범위는 IRC 부분집합이며 TLS, 영속 저장, federation이 없습니다. event별 처리량 quantum도 없어 모든 부하에서 엄격한 공정성을 보장하지 않습니다.
