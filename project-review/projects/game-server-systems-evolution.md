# Game Server Systems Evolution

## 1. 이력서용 프로젝트 설명

동일한 authoritative arena server 규약을 C++20/POSIX·kqueue와 Java 21/Netty 두 트랙으로 구현했습니다.  
C++에서는 file descriptor RAII, bounded frame parser, move-only outbound buffer와 mailbox를 직접 관리하고, Java에서는 Netty 위에 동일한 protocol·room ownership 규칙을 구성했습니다.  
TCP control과 UDP realtime 경로, fixed tick, deterministic replay, full/delta snapshot, ACK·재접속·backpressure·room별 scheduling을 구현했습니다.  
보존된 교차 검증에서 32개 room·128명·600 tick 동안 19,200개 canonical state hash 쌍의 일치와 bounded queue 및 자원 해제를 확인했습니다.

## 2. 30초 프로젝트 소개

이 프로젝트는 실시간 게임 서버 규약을 C++과 Java로 구현해 동일한 불변식을 확인한 작업입니다. C++에서는 POSIX socket과 kqueue로 자원 수명을 직접 관리하고, Java에서는 Netty 위에 같은 room ownership을 구성했습니다. TCP control과 UDP realtime을 분리하고 replay와 snapshot ACK로 재접속·backpressure 상황을 검증했습니다.

## 3. 2분 프로젝트 소개

이 프로젝트는 연결 장애와 느린 client가 있어도 authoritative state와 자원 수명을 설명할 수 있는 실시간 세션 서버를 목표로 했습니다. 하나의 contract로 C++20과 Java 21 구현을 독립적으로 만들었습니다. C++은 POSIX socket과 kqueue 위에서 descriptor를 RAII로 감싸고 frame parser, move-only write buffer와 bounded mailbox를 직접 관리했습니다. Java는 Netty event loop를 쓰되 connection·session·room identity와 snapshot stream은 domain이 소유하도록 했습니다. control은 TCP, input과 snapshot은 UDP로 분리하고, UDP 손실에는 full/delta snapshot, ACK와 full resync로 대응했습니다. tick catch-up과 outbound queue에도 상한을 뒀습니다. 두 구현의 replay는 canonical hash로 비교했습니다. 보존된 G07 근거에서 200-tick 실행 다섯 번이 일치했고 변형 input은 37번째 tick에서 차이를 잡았습니다. G14 고정 workload는 32개 room·128명·600 tick의 19,200개 hash 쌍과 자원 해제를 확인했습니다. 이번 검토에서는 workload를 다시 돌리지 않고 handoff evidence·tag·history의 무결성만 검사했으므로 일반 throughput 결과로 말하지 않습니다. 범위는 단일 서버 realtime core까지이고 분산 room ownership은 미구현이며, C++은 macOS kqueue에 고정돼 있습니다.
