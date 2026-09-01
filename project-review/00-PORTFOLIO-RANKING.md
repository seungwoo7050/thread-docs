# 포트폴리오 프로젝트 상대평가

점수는 코드량이 아니라 직접 구현한 기술적 난도, 설계 판단, 검증 근거, 현재 완성도와 다른 프로젝트 대비 차별성을 함께 반영했다. 분류 기준은 CORE 90점 이상, STRONG SUPPORT 84~89점, SUPPORTING 70~83점, OMIT 69점 이하이다.

`system-evolution`의 언어별 track, Sportsbook의 개별 서비스, Pong의 apps/packages와 Inception의 컨테이너는 상위 시스템의 구현 근거로만 사용하고 별도 프로젝트로 중복 평가하지 않았다.

후행 추가 프로젝트 2개는 같은 기준으로 점수와 기술 등급을 산정하되 기존 18개 랭킹과 선정 결과를 다시 조정하지 않는 **후보(CANDIDATE)**로 분리했다. 후보 표의 C1·C2는 후보끼리의 순서이며 기존 Rank에 편입된 번호가 아니다.

| Rank | Project | Type | Classification | Score / 100 | Best Target Roles | Strongest Engineering Signal | Main Weakness |
|---:|---|---|---|---:|---|---|---|
| 1 | Game Server Systems Evolution | MULTI_MODULE_PROJECT | CORE | 96 | C/C++ / Systems, Backend, General Software | C++20 POSIX/kqueue와 Java 21 Netty로 같은 authoritative session 계약을 구현하고 fixed tick, replay, snapshot, reconnect와 backpressure의 cross-track state 일치를 검증 | 분산 room ownership 단계인 G15~G18은 미구현이며 C++ track은 macOS/kqueue에 한정되고 hosted CI·배포가 없음 |
| 2 | Web Systems Evolution | MULTI_MODULE_PROJECT | CORE | 95 | Backend, Full-stack, General Software | Fastify와 Spring 양쪽에서 PostgreSQL 기반 실행 소유권, lease·idempotency·crash recovery, SSRF 방어와 SSR/hydration 경계를 구현하고 hosted 검증 기록을 보존 | Redis·Kafka·outbox를 다루는 후속 phase는 미구현이고 검증은 통제된 fixture 중심이며 공개 배포가 없음 |
| 3 | ft-irc | INDEPENDENT_PROJECT | CORE | 94 | C/C++ / Systems, Backend, General Software | epoll/kqueue 기반 non-blocking event loop와 연결별 partial I/O, 송신 queue, backpressure 및 connection lifecycle을 직접 관리 | IRC 부분집합만 지원하고 TLS·영속성·server federation이 없으며 event별 작업량 상한은 두지 않음 |
| 4 | miniRT | INDEPENDENT_PROJECT | CORE | 92 | C/C++ / Systems, Graphics, General Software | 해석적 도형 교차, BVH와 결정적 tile 병렬 렌더링을 직접 구현하고 linear/BVH 및 thread 수 간 이미지 동일성을 검증 | 단일 sample 기반의 기본 renderer로 anti-aliasing·texture·mesh·SIMD/GPU 가속이 없음 |
| 5 | Mobile Systems Evolution | MULTI_MODULE_PROJECT | CORE | 90 | Mobile, General Software | Kotlin/Room과 React Native/SQLite에서 local DB를 source of truth로 두고 durable mutation intent, 충돌 처리, process-death 및 background recovery를 구현 | Android에 한정된 좁은 item domain이며 push·deep link·notification phase와 hosted CI·store release가 없음 |
| 6 | Sportsbook Backend Platform | MULTI_MODULE_PROJECT | STRONG SUPPORT | 88 | Backend, Full-stack (backend-heavy), General Software | 불확실한 네트워크 결과와 순서 역전을 durable state machine, idempotency proof, double-entry ledger, outbox와 lease 기반 정정 복구로 처리 | 현재 split-repo 구조가 과거 branch 기반 CI·cold gate 가정과 어긋나 전체 release 검증을 clean clone에서 재현할 수 없음 |
| 7 | Pong Pong | MULTI_MODULE_PROJECT | STRONG SUPPORT | 87 | Full-stack, Backend, Frontend, General Software | 서버 권위형 fixed-step game state, 공유 Zod HTTP/WS 계약, reconnect·snapshot backpressure와 결과 단일 확정을 하나의 monorepo로 연결 | production 사용자 onboarding이 없고 room ownership이 단일 process에 머물며 hosted CI 통과 근거가 없음 |
| 8 | Minishell | INDEPENDENT_PROJECT | STRONG SUPPORT | 85 | C/C++ / Systems, General Software | quote-aware tokenizer·parser·expansion에서 fork/pipe/dup2/wait, heredoc과 부모 builtin의 FD 복구까지 process lifecycle을 직접 연결 | job control, process group·terminal 제어와 완전한 interactive signal recovery를 의도적으로 지원하지 않음 |
| 9 | ft-container | INDEPENDENT_PROJECT | SUPPORTING | 83 | C/C++ / Systems, General Software | allocator 기반 vector 객체 수명과 header-sentinel red-black tree의 회전·재균형·예외 경계를 직접 구현하고 불변식을 검증 | C++98 표준 container 부분집합을 재현한 전형적인 exercise이며 상위 C++ 프로젝트와 신호가 중복됨 |
| 10 | Inception | MULTI_MODULE_PROJECT | SUPPORTING | 81 | Backend, Platform / Infrastructure, General Software | Compose stack의 상태 재조정, secret 격리, backup·restore와 credential rotation의 실패 복구 절차를 직접 구현 | off-the-shelf 단일 host stack이고 runtime 시나리오는 이번 검토에서 재실행하지 않았으며 현재 최상위 `make test`가 자체 wording 검사로 실패함 |
| 11 | Portfolio Site | INDEPENDENT_PROJECT | SUPPORTING | 80 | Frontend, Full-stack, General Software | 검증된 content model을 route별 view model과 다섯 renderer로 분리하고 accessibility·visual·bundle·Lighthouse·container gate를 구성 | 실제 content가 placeholder 상태이고 공개 배포·hosted CI 근거가 없으며 mobile 성능 목표를 모든 route에서 만족하지 않음 |
| 12 | Signal Message Bus | INDEPENDENT_PROJECT | SUPPORTING | 78 | C/C++ / Systems, General Software | signal handler와 일반 실행 문맥을 self-pipe로 분리하고 Unix datagram으로 session·sequence·ACK 제어 protocol을 구현 | 신뢰성의 상당 부분을 datagram이 담당하며 같은 host·ABI의 단일 session만 지원하고 재전송·중복 제거가 없음 |
| 13 | Thread Dining | INDEPENDENT_PROJECT | SUPPORTING | 76 | C/C++ / Systems, General Software | condition variable 시작 barrier, monotonic time, terminal state 선형화와 부분 thread 생성·회수 자원 ledger를 구현 | 고전적인 dining philosophers exercise이고 starvation 부재나 최대 대기 시간을 보장하지 않음 |
| 14 | C++ Foundation | MULTI_MODULE_PROJECT | SUPPORTING | 74 | C/C++ / Systems, General Software | Rule of Three, polymorphic clone, copy-swap, overflow 검사와 template·iterator 경계를 실패 주입 및 속성 검사로 검증 | 여섯 exercise의 학습형 progression이며 C++98·raw-pointer 모델이 다른 C++ 프로젝트와 중복됨 |
| 15 | Stack Sort | INDEPENDENT_PROJECT | SUPPORTING | 72 | C/C++ / Systems, General Software | rank compression과 stable binary radix command generator를 checker·독립 replay oracle·fault test와 함께 구현 | 고전적인 제한 연산 정렬 exercise이고 배열 앞쪽 이동 때문에 물리적 이동 비용이 `O(n² log n)`임 |
| 16 | Buffered Line Reader | INDEPENDENT_PROJECT | SUPPORTING | 70 | C/C++ / Systems | borrowed-FD context와 EOF·AGAIN·ERROR 상태를 분리하고 EINTR, non-blocking read, descriptor reuse와 copy budget을 검증 | 범위가 좁은 단일 기능 library이고 호환용 global API는 thread-safe하지 않으며 `NULL` 원인을 구분하지 못함 |
| 17 | C Foundation | INDEPENDENT_PROJECT | OMIT | 67 | C/C++ / Systems, General Software | 43개 libc·list API에서 overflow, ownership, partial write와 중간 실패 정리를 직접 구현 | 기본 libc 재구현 성격이 강하고 ownership·failure 신호가 상위 systems 프로젝트와 거의 전부 중복됨 |
| 18 | Format Printer | INDEPENDENT_PROJECT | OMIT | 65 | C/C++ / Systems | format을 측정·검증한 뒤 출력하는 two-pass pipeline과 partial write·EINTR 복구를 구현 | 지원 문법이 좁은 printf 부분집합이며 parsing·I/O 신호가 다른 C 프로젝트보다 약하고 중복됨 |

## Recommended Anchor Projects

1. **Game Server Systems Evolution** — 서로 다른 runtime과 추상화에서 동일한 realtime invariant를 구현·비교한 가장 차별화된 systems 프로젝트다.
2. **Web Systems Evolution** — persistence, 인증, worker concurrency, outbound security, frontend 경계와 운영 검증을 하나의 문제에서 연결한다.
3. **ft-irc** — framework 아래에 숨지 않고 socket, readiness event, stream I/O와 connection resource lifecycle을 직접 다룬 근거가 선명하다.
4. **miniRT** — 웹·네트워크와 겹치지 않는 수학, 자료구조, 병렬 처리와 측정 가능한 최적화 판단을 보여준다.
5. **Mobile Systems Evolution** — offline source of truth, process death와 background lifecycle이라는 모바일 고유 문제를 두 구현으로 비교한다.

## Strong Supporting Projects

- **Sportsbook Backend Platform** — 금전 상태와 분산 실패 복구의 깊이는 매우 높지만 현재 release 재현성 문제 때문에 anchor보다 한 단계 낮춘다.
- **Pong Pong** — 사용자가 이해하기 쉬운 realtime full-stack 제품으로, Game Server와 Web Systems의 기술 신호를 제품 흐름에서 보완한다.
- **Minishell** — parser에서 process·FD lifecycle까지 연결되는 순수 C 시스템 구현으로 상위 네트워크 프로젝트와 다른 syscall 대화를 열 수 있다.

## Candidate Projects

| Rank | Project | Type | Classification | Score / 100 | Best Target Roles | Strongest Engineering Signal | Main Weakness |
|---:|---|---|---|---:|---|---|---|
| C1 | Travel Readiness | MULTI_MODULE_PROJECT | CORE | 91 | Backend, Full-stack (data-heavy), General Software | 호출 전 durable attempt부터 content-addressed artifact, ordered parse lineage, sealed candidate와 운영자 통제 publication까지 다중 공공데이터 provenance를 연결하고 비행시간 통계·시간대 기반 검색을 구현 | 로컬 후보로 hosted CI·공개 배포가 없고 runtime mtree pin 불일치와 VoiceOver·전체 module rollback 등 미완료 gate가 남음 |
| C2 | Grocery Seasonality | INDEPENDENT_PROJECT | STRONG SUPPORT | 89 | Backend, Full-stack (data-heavy), General Software | bounded KAMIS 수집과 엄격한 typed parsing을 검토·seal·CAS activation으로 연결하고 PostgreSQL trigger로 append-only provenance와 current pointer 정합성을 방어 | 현재 HEAD 전체 회귀를 재실행하지 못했고 실데이터 검증 범위가 제한적이며 CI·배포와 운영 publication·복구 근거가 없음 |

- **Travel Readiness**의 91점은 기존 점수상 miniRT(92)와 Mobile Systems Evolution(90) 사이에 해당한다. 다만 후행 후보이므로 Anchor 목록과 직무별 본 순서에는 아직 편입하지 않는다.
- **Grocery Seasonality**의 89점은 기존 점수상 Mobile Systems Evolution(90)과 Sportsbook Backend Platform(88) 사이에 해당한다. 기술 등급은 STRONG SUPPORT지만 동일하게 후보 상태를 유지한다.
- Travel의 Django app들은 공통 PostgreSQL·settings·WSGI와 publication contract를 공유하는 `SERVICE_OR_COMPONENT`이고, Grocery의 내부 수집·검토·게시 모듈도 별도 프로젝트로 중복 평가하지 않았다.

## Supporting Projects

- **ft-container** — allocator, 객체 수명과 red-black tree 불변식의 보조 근거로 사용한다.
- **Inception** — 단일 host infrastructure와 상태 재조정·backup/restore 경험을 보완하되 현재 test 한계를 함께 밝힌다.
- **Portfolio Site** — frontend architecture, accessibility와 delivery gate를 보여주지만 실제 content를 채우기 전에는 대표 프로젝트로 내세우지 않는다.
- **Signal Message Bus** — async-signal-safe boundary와 IPC protocol을 묻는 면접에서 짧게 언급한다.
- **Thread Dining** — 동시성 기본기와 resource cleanup의 보조 사례로 사용한다.
- **C++ Foundation** — 객체 소유권·예외 안전성의 학습 progression을 설명하는 배경 자료로 둔다.
- **Stack Sort** — 제한된 연산 모델, rank compression과 독립 oracle 검증을 보여주는 알고리즘 보조 사례다.
- **Buffered Line Reader** — 작은 C 프로젝트 중에서는 non-blocking I/O와 명시적 상태 API를 대표하는 사례로 남긴다.

## Projects to Omit

- **C Foundation** — 구현과 검증은 충실하지만 기본 library 재구현이며 상위 C 프로젝트보다 채용 대화의 폭과 차별성이 낮다.
- **Format Printer** — parsing과 출력 오류 처리는 유효하지만 좁은 printf subset으로 다른 프로젝트의 신호를 반복한다.

## Redundancy Analysis

- **Evolution 계열**은 형식은 같지만 Game Server는 realtime replication, Web은 durable worker와 security, Mobile은 offline lifecycle을 담당하므로 세 프로젝트 모두 유지한다. 소개 첫 문장부터 각 domain 문제를 드러내 동일한 프로젝트의 언어 변형처럼 보이지 않게 한다.
- **Realtime/networking**에서는 Game Server를 cross-runtime architecture 대표, ft-irc를 syscall·event-loop 대표로 사용한다. Pong은 세 번째 systems anchor가 아니라 사용자-facing full-stack 제품으로 배치한다.
- **Backend consistency**에서는 현재 검증 근거가 더 완결된 Web Systems를 anchor로 둔다. Sportsbook은 금전·정정 domain이 차별화되지만 split-repo release gate가 복구되기 전까지 strong support로 유지한다.
- **C/C++ systems**의 대표는 ft-irc와 miniRT다. Minishell은 process·FD 주제로만 추가하고 ft-container, Signal Message Bus, Thread Dining, C++ Foundation은 후속 질문용 근거로 낮춘다.
- **Frontend/infrastructure**에서는 Portfolio Site와 Inception이 각각 고유 신호를 갖지만 현재 completeness·verification gap 때문에 supporting으로 둔다.
- **작은 C library**는 Buffered Line Reader만 supporting 대표로 남기고 C Foundation과 Format Printer는 포트폴리오에서 제외한다.
- **후행 데이터 후보**인 Travel Readiness와 Grocery Seasonality는 Django SSR, 외부 source evidence, 검토·publication과 secret 비저장 경계가 서로 겹친다. 후보 중에는 통계적 비행시간 derivation, 시간대 계산과 세 독립 publication을 결합한 Travel을 대표로 두고, Grocery는 KAMIS typed fact set과 DB trigger 기반 활성화의 보조 사례로 둔다. Web Systems Evolution은 cross-runtime durable worker와 outbound security 대표로 그대로 유지한다.

## Role-oriented Selection

### Frontend Engineer

1. Web Systems Evolution
2. Pong Pong
3. Portfolio Site

Portfolio Site는 실제 content를 채우고 production readiness gate를 통과한 뒤 순위를 다시 올리는 편이 안전하다.

### Full-stack Engineer

1. Web Systems Evolution
2. Pong Pong
3. Sportsbook Backend Platform — backend-heavy 보완 프로젝트
4. Portfolio Site

후보 참고: Travel Readiness → Grocery Seasonality. 둘 다 frontend 표현보다 SSR·공공데이터 게시 경계의 신호가 강하므로 기존 순서를 대체하지 않는다.

### Backend Engineer

1. Web Systems Evolution
2. Sportsbook Backend Platform
3. Game Server Systems Evolution
4. ft-irc
5. Pong Pong

후보 참고: Travel Readiness → Grocery Seasonality. Travel은 data-heavy 공공데이터 대표 후보이고 Grocery는 같은 계열의 보조 후보로 본다.

### C/C++ / Systems Engineer

1. ft-irc
2. Game Server Systems Evolution
3. miniRT
4. Minishell
5. ft-container

### General Software Engineer

1. Game Server Systems Evolution
2. Web Systems Evolution
3. miniRT
4. Mobile Systems Evolution
5. Pong Pong
6. Sportsbook Backend Platform

후보 참고: Travel Readiness → Grocery Seasonality. 기존 여섯 프로젝트의 순서는 동결하고 차후 정식 편입 시에만 다시 비교한다.
