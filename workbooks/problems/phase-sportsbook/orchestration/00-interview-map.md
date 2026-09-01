# DevThread 개발자 기술면접 워크북 — 마스터 인덱스

## 사용 범위와 증거 경계

이 워크북은 현재 GPT 프로젝트 대화에 남아 있는 17개 Development Thread의 제목, 확인 가능한 대표 커밋, External-State Evidence Packet 요약, 그리고 이전 작업에서 확인된 Markdown 파일명만 사용한다.

- Thread 1~9, 14~17은 프로젝트 기록에 노출된 대표 커밋 메시지를 사용했다.
- Thread 9의 대표 커밋은 `28a36bf8d802 — test(e2e): …`까지만 노출되어 있어 뒤쪽 문구를 복원하지 않았다.
- Thread 10~13은 문서 제목과 실제 Markdown 파일명은 확인되지만 개별 커밋 메시지는 현재 프로젝트 요약에 노출되지 않았다. 표와 상세 문서에서 이를 그대로 표시했다.
- 파일·함수·클래스 이름은 프로젝트 기록에 직접 나타난 경우에만 적었다. 그 밖의 백지 구현 시그니처와 상태명은 원본 코드 복원이 아니라 10~30분 면접 문제로 축소한 연습용 인터페이스다.

## 우선순위 정의

| 우선순위 | 의미 |
|---|---|
| S | 질문과 직접 구현 모두 반드시 준비할 가치가 높다. 프로젝트 밖에서도 재사용되는 핵심 기본기다. |
| A | 질문 또는 핵심 로직 구현 가능성이 높다. S와 함께 상세 워크북으로 준비한다. |
| B | 직접 코딩보다 설계 의도·경계·trade-off 설명이 중요하다. |
| C | 설정 암기, 도구 래퍼, 프로젝트 특수 배선처럼 별도 문제로 만들 효율이 낮다. |

## 전체 Thread·커밋 선별 결과

| ID | 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread | 상세 워크북 |
|---|---:|---:|---|---|---|---|---:|---:|---|---|
| IM-01 | S | 1 | `ca6c5af4f18b — fix(source): resolve archive remote refs` | 로컬 브랜치, `origin` remote-tracking branch, detached source materialization | 가변 Git ref를 불변 커밋 정체성으로 해소하기 | 입력 재현성, 모호성 처리, fail-closed 정책을 함께 검증할 수 있다. | 높음 | 높음 | 16, 17 | [01-release-identity-and-artifacts.md#im-01](01-release-identity-and-artifacts.md#im-01) |
| IM-02 | S | 2 | `f4a48d911ada — build(jars): stage exact release artifacts atomically` | run-owned Maven repository, 임시 generation, 정확히 7개 service JAR, 원자적 게시 | 완전한 아티팩트 집합 검증과 atomic publication | 부분 게시, 이전 세대 보존, 동시 실행, 파일시스템 경계를 묻기 좋다. | 높음 | 높음 | 8, 16 | [01-release-identity-and-artifacts.md#im-02](01-release-identity-and-artifacts.md#im-02) |
| IM-03 | A | 3 | `4b3c66663326 — build(startup): enforce full dependency DAG` | Docker Compose dependency graph: persistence → preflight → application → consumer-assignment → settlement → admin | 기동 DAG, readiness와 단순 process start의 차이 | 위상 정렬, cycle 검출, 의미 기반 준비 상태, 실패 전파를 일반화할 수 있다. | 높음 | 높음 | 5, 14 | [02-runtime-lifecycle-and-security.md#im-03](02-runtime-lifecycle-and-security.md#im-03) |
| IM-04 | A | 4 | `f57b610f2637 — build(postgres): bootstrap service databases` | PostgreSQL database bootstrap, Flyway 실행, Redis별 격리 persistent state | 저장소 격리, idempotent bootstrap, migration 무결성 | 재시작·부분 실패·schema drift·상태 소유권을 함께 다룬다. | 높음 | 중간 | 9, 10, 11 | [03-storage-messaging-and-probes.md#im-04](03-storage-messaging-and-probes.md#im-04) |
| IM-05 | S | 5 | `f9e15158d474 — build(kafka): provision topics without mutation` | Kafka KRaft state, `topic-init`, partition·replication·retention drift | create-if-missing과 기존 리소스 drift 검출 | 편의상 자동 수정하는 방식과 운영 안전성 사이의 trade-off가 선명하다. | 높음 | 높음 | 3, 8, 9, 12 | [03-storage-messaging-and-probes.md#im-05](03-storage-messaging-and-probes.md#im-05) |
| IM-06 | A | 5 | `f9e15158d474 — build(kafka): provision topics without mutation`와 같은 Thread의 consumer-assignment gap | consumer assignment readiness | 인프라가 떠 있는 상태와 실제 소비 준비 상태 구분 | fixed sleep을 피하고 rebalance·timeout·누락 assignment를 설명할 수 있다. | 높음 | 높음 | 3, 8, 12 | [03-storage-messaging-and-probes.md#im-06](03-storage-messaging-and-probes.md#im-06) |
| IM-07 | A | 6 | `43d20c34e2eb — build(gate): generate isolated runtime secrets` | cold-gate별 service key, RSA keypair, PostgreSQL/Grafana password, environment injection | 실행 단위 secret 생성과 최소 노출 경계 | 생성보다 더 어려운 소비자 제한, 로그·evidence 유출, lifecycle 결합을 평가할 수 있다. | 높음 | 중간 | 14, 15 | [02-runtime-lifecycle-and-security.md#im-07](02-runtime-lifecycle-and-security.md#im-07) |
| IM-08 | B | 7 | `aa55201ffca6 — build(observability): compose isolated metrics and logs` | Prometheus, Loki, Grafana, Promtail, project-owned volumes/healthchecks, Docker discovery 권한 | 관측 데이터 plane 격리와 discovery 권한 경계 | 보안·소유권 설명 가치는 높지만 원본 구현은 Compose 배선 비중이 크다. | 높음 | 낮음 | 6, 14, 15 | — |
| IM-09 | C | 7 | `aa55201ffca6 — build(observability): compose isolated metrics and logs` | 개별 관측 스택 서비스 설정과 Compose 배선 | 제품별 설정 암기 | 일반화 가능한 판단보다 도구별 boilerplate 비중이 높다. | 낮음 | 낮음 | 7의 IM-08 | — |
| IM-10 | A | 8 | `269cf445cb2a — build(fixtures): stage executable publisher` | Java 17, shared protocol identity, shaded dependencies, main class | 실행 가능한 fixture JAR 계약 검증 | 런타임 호환성, 패키징 완전성, protocol identity를 배포 전 확인하는 문제로 축소하기 좋다. | 높음 | 높음 | 2, 5, 16 | [03-storage-messaging-and-probes.md#im-10](03-storage-messaging-and-probes.md#im-10) |
| IM-11 | A | 8 | `269cf445cb2a — build(fixtures): stage executable publisher`와 같은 Thread의 live publish/probe gap | Kafka fixture publisher와 probe | 상관관계 기반 live I/O 검증과 timeout | stale record, unrelated traffic, 중복 응답, protocol mismatch를 다룰 수 있다. | 높음 | 높음 | 5, 9, 12 | [05-e2e-oracles-and-faults.md#im-11](05-e2e-oracles-and-faults.md#im-11) |
| IM-12 | S | 9 | `28a36bf8d802 — test(e2e): …` — 현재 요약에 전체 문구 미노출 | 13개 cross-store E2E scenario, Toxiproxy fault mutation과 restoration | 다중 저장소 oracle, 장애 주입, 무조건 복구 | 성공 응답만 보는 테스트를 넘어 상태 정합성·복구·cleanup을 검증한다. | 높음 | 높음 | 4, 5, 8, 10, 11, 12, 13 | [05-e2e-oracles-and-faults.md#im-12](05-e2e-oracles-and-faults.md#im-12) |
| IM-13 | S | 10 | 개별 커밋 메시지는 현재 프로젝트 요약에 미노출 | `10-bet-placement-state-machine-and-failure-recovery-01.md`, `10-bet-placement-state-machine-and-failure-recovery-02.md` | 베팅 배치 상태 머신과 실패 복구 | 상태 전이, 중복 이벤트, 외부 효과, 재시도 invariant는 대표적인 실전 면접 주제다. | 높음 | 높음 | 4, 5, 9, 12 | [04-domain-state-and-recovery.md#im-13](04-domain-state-and-recovery.md#im-13) |
| IM-14 | S | 11 | 개별 커밋 메시지는 현재 프로젝트 요약에 미노출 | `11-settlement-result-candidates-and-revision-recovery-01.md`, `11-settlement-result-candidates-and-revision-recovery-02.md` | 결과 후보 선택과 revision 복구 | 최신성, stale input, 동일 revision 충돌, 복구 후 단조성을 묻기 좋다. | 높음 | 높음 | 9, 12, 13 | [04-domain-state-and-recovery.md#im-14](04-domain-state-and-recovery.md#im-14) |
| IM-15 | S | 12 | 개별 커밋 메시지는 현재 프로젝트 요약에 미노출 | `12-event-ordering-replay-and-dlt-invariants-01.md`, `12-event-ordering-replay-and-dlt-invariants-02.md` | 이벤트 순서, replay, DLT 분류 invariant | 중복·역순·gap·poison event를 서로 다른 실패 종류로 분리하는 기본기를 평가한다. | 높음 | 높음 | 5, 8, 9, 10, 11, 13 | [04-domain-state-and-recovery.md#im-15](04-domain-state-and-recovery.md#im-15) |
| IM-16 | A | 13 | 개별 커밋 메시지는 현재 프로젝트 요약에 미노출 | `13-admin-audit-and-downstream-correlation.md` | 감사 기록과 downstream correlation | 현재 상태와 감사 이력의 차이, causation/correlation, 재처리 추적성을 설명할 수 있다. | 높음 | 중간 | 9, 11, 12, 15, 16 | [04-domain-state-and-recovery.md#im-16](04-domain-state-and-recovery.md#im-16) |
| IM-17 | S | 14 | `5ef2d1349379 — build(gate): own cold release lifecycle` | context 생성, secrets, build, checks, success cleanup, failure cleanup | 소유권 기반 resource lifecycle과 예외 안전성 | 정상·실패·중단 경로에서 누가 무엇을 정리하는지 직접 구현하기 좋다. | 높음 | 높음 | 3, 6, 15, 16 | [02-runtime-lifecycle-and-security.md#im-17](02-runtime-lifecycle-and-security.md#im-17) |
| IM-18 | S | 15 | `627c34edbd44 — build(evidence): redact runtime credentials` | evidence persistence, service key/password, PEM/JWT-like material, redaction과 rejection | 구조·값 기반 비밀 탐지와 fail-closed 저장 | 보안 필터의 누락, 과탐, 중첩 데이터, 오류 메시지 자체의 유출을 점검할 수 있다. | 높음 | 높음 | 6, 13, 14, 16 | [02-runtime-lifecycle-and-security.md#im-18](02-runtime-lifecycle-and-security.md#im-18) |
| IM-19 | A | 16 | `6184fc6137c — build(evidence): record locked release identities` | orchestration SHA, source SHA, lock identity, JAR hash | canonical attestation과 release identity chain | mutable label 대신 실제 실행에서 얻은 식별자를 묶고 재검증하는 설계를 평가한다. | 높음 | 높음 | 1, 2, 14, 15, 17 | [01-release-identity-and-artifacts.md#im-19](01-release-identity-and-artifacts.md#im-19) |
| IM-20 | B | 17 | `f969a81afbda — build(history): expose the archive history guard` | 로컬 repository history를 읽는 CLI, policy verifier, Git/CI control binding | 선형 history 정책의 의미와 enforcement 위치 | 직접 코딩보다 정책 위반 정의, 예외, local/CI 일관성 설명이 중요하다. | 높음 | 중간 | 1, 14, 16 | — |
| IM-21 | C | 17 | `f969a81afbda — build(history): expose the archive history guard` | guard CLI 노출과 command wrapper | CLI 래퍼 구현 자체 | 정책 semantics를 제외한 래퍼 코드는 면접 대비 효율이 낮다. | 낮음 | 낮음 | 17의 IM-20 | — |

## 대표 Thread와 연관 Thread 관계

| 역량 묶음 | 대표 면접 지점 | 대표 Thread | 연관 Thread | 통합 원칙 |
|---|---|---|---|---|
| 릴리스 입력과 증명 | IM-01, IM-02, IM-19 | 1, 2, 16 | 17 | ref 해소 → 완전한 artifact 게시 → identity attestation을 하나의 재현성 사슬로 본다. history 정책은 설명 항목으로 연결한다. |
| 기동·소유권·보안 lifecycle | IM-03, IM-07, IM-17, IM-18 | 3, 14, 15 | 6 | startup ordering, secret 소비 경계, cleanup, evidence 안전성을 하나의 실행 lifecycle로 본다. |
| 저장소와 Kafka 계약 | IM-04, IM-05, IM-06, IM-10 | 4, 5, 8 | 3, 9, 12 | 영속 상태 bootstrap, drift 검출, assignment readiness, 실행 artifact 계약을 운영 전제조건으로 묶는다. |
| 도메인 상태와 복구 | IM-13, IM-14, IM-15, IM-16 | 10, 11, 12, 13 | 5, 9 | 상태 전이, revision, ordering/replay, 감사 상관관계를 하나의 event-driven invariant 묶음으로 본다. |
| live 검증과 장애 오라클 | IM-11, IM-12 | 8, 9 | 4, 5, 10, 11, 12, 13 | 실제 publish/probe와 cross-store state oracle, fault restoration을 별도 테스트 기본기로 준비한다. |
| 관측·히스토리 정책 설명 | IM-08, IM-20 | 7, 17 | 6, 14, 15, 16 | 제품 설정이나 래퍼 구현은 버리고 권한, 격리, enforcement 위치만 설명 연습한다. |

## 상세 워크북 파일 안내

| 파일 | 포함 항목 | 역할 |
|---|---|---|
| [01-release-identity-and-artifacts.md](01-release-identity-and-artifacts.md) | IM-01, IM-02, IM-19 | Git ref 고정, atomic artifact publication, release attestation |
| [02-runtime-lifecycle-and-security.md](02-runtime-lifecycle-and-security.md) | IM-03, IM-07, IM-17, IM-18 | startup DAG, secret 경계, cold-gate cleanup, evidence redaction |
| [03-storage-messaging-and-probes.md](03-storage-messaging-and-probes.md) | IM-04, IM-05, IM-06, IM-10 | PostgreSQL/Flyway/Redis 격리, Kafka provisioning/readiness, executable fixture 계약 |
| [04-domain-state-and-recovery.md](04-domain-state-and-recovery.md) | IM-13, IM-14, IM-15, IM-16 | 상태 머신, revision 복구, ordering/replay/DLT, audit correlation |
| [05-e2e-oracles-and-faults.md](05-e2e-oracles-and-faults.md) | IM-11, IM-12 | live Kafka probe, cross-store oracle, Toxiproxy fault restoration |

## S/A 완전성 검증

| ID | 우선순위 | 상태 | 상세 위치 |
|---|---:|---|---|
| IM-01 | S | 독립 상세 항목 | `01-release-identity-and-artifacts.md#im-01` |
| IM-02 | S | 독립 상세 항목 | `01-release-identity-and-artifacts.md#im-02` |
| IM-03 | A | 독립 상세 항목 | `02-runtime-lifecycle-and-security.md#im-03` |
| IM-04 | A | 독립 상세 항목 | `03-storage-messaging-and-probes.md#im-04` |
| IM-05 | S | 독립 상세 항목 | `03-storage-messaging-and-probes.md#im-05` |
| IM-06 | A | 독립 상세 항목 | `03-storage-messaging-and-probes.md#im-06` |
| IM-07 | A | 독립 상세 항목 | `02-runtime-lifecycle-and-security.md#im-07` |
| IM-10 | A | 독립 상세 항목 | `03-storage-messaging-and-probes.md#im-10` |
| IM-11 | A | 독립 상세 항목 | `05-e2e-oracles-and-faults.md#im-11` |
| IM-12 | S | 독립 상세 항목 | `05-e2e-oracles-and-faults.md#im-12` |
| IM-13 | S | 독립 상세 항목 | `04-domain-state-and-recovery.md#im-13` |
| IM-14 | S | 독립 상세 항목 | `04-domain-state-and-recovery.md#im-14` |
| IM-15 | S | 독립 상세 항목 | `04-domain-state-and-recovery.md#im-15` |
| IM-16 | A | 독립 상세 항목 | `04-domain-state-and-recovery.md#im-16` |
| IM-17 | S | 독립 상세 항목 | `02-runtime-lifecycle-and-security.md#im-17` |
| IM-18 | S | 독립 상세 항목 | `02-runtime-lifecycle-and-security.md#im-18` |
| IM-19 | A | 독립 상세 항목 | `01-release-identity-and-artifacts.md#im-19` |

## 백지 구현 우선순위

1. IM-13 베팅 배치 상태 전이 함수와 중복 이벤트 처리
2. IM-15 event ordering·replay·DLT 분류기
3. IM-02 정확한 artifact 집합 검증과 atomic publication
4. IM-17 소유권 기반 lifecycle scope와 실패 시 cleanup
5. IM-05 Kafka topic create/unchanged/drift 판정기
6. IM-12 fault scope와 cross-store oracle
7. IM-14 revision 후보 reducer
8. IM-01 Git ref 해소와 immutable commit identity 반환
9. IM-18 evidence redaction·rejection 경계
10. IM-03 dependency DAG의 startup wave 계산과 cycle 검출
11. IM-06 consumer assignment readiness 판정
12. IM-19 canonical release attestation 생성
13. IM-11 correlation 기반 live probe 결과 판정
14. IM-04 격리된 store bootstrap plan 생성
15. IM-10 executable fixture contract validator
16. IM-07 secret binding allowlist validator
17. IM-16 audit/correlation event validator

## 설명 연습 우선순위

1. IM-12 왜 HTTP 성공만으로 cross-store E2E를 통과시킬 수 없는가
2. IM-15 duplicate·out-of-order·gap·poison event를 왜 분리해야 하는가
3. IM-17 primary failure를 보존하면서 cleanup failure를 다루는 방법
4. IM-01 mutable ref와 immutable source identity의 차이
5. IM-02 부분 publication을 막는 generation 교체 전략
6. IM-05 기존 Kafka topic을 자동 수정하지 않는 이유
7. IM-18 redaction과 rejection을 동시에 두는 이유
8. IM-03 process start와 semantic readiness의 차이
9. IM-14 revision 단조성과 stale input 처리
10. IM-19 source·lock·artifact identity를 하나의 증거로 묶는 이유
11. IM-07 실행 단위 secret과 최소 소비자 노출
12. IM-04 bootstrap idempotency가 drift 무시를 뜻하지 않는 이유
13. IM-16 audit, correlation, causation의 역할 구분
14. IM-08 observability 격리와 Docker discovery 권한 trade-off
15. IM-20 local guard와 CI enforcement가 같은 정책을 사용해야 하는 이유

## 한 문제로 통합한 Thread 묶음

- **릴리스 재현성 사슬:** Thread 1 + Thread 2 + Thread 16, 설명 연관 Thread 17
- **실행 lifecycle과 보안 경계:** Thread 3 + Thread 6 + Thread 14 + Thread 15
- **Kafka 준비 계약:** Thread 5 + Thread 8, ordering 연관 Thread 12
- **저장소·메시징 통합 검증:** Thread 4 + Thread 5 + Thread 8 + Thread 9
- **도메인 상태 복구:** Thread 10 + Thread 11 + Thread 12 + Thread 13
- **관측 권한 설명:** Thread 7 + Thread 6 + Thread 14 + Thread 15
