# DevThread Sportsbook Admin 개발자 기술면접 워크북 — 마스터 인덱스

이 문서는 현재 GPT 프로젝트에 축적된 DevThread 학습 문서와 작업 기록에서 실제로 확인되는 내용만으로 작성한 선별 기준표다. 확인되지 않은 파일명·함수명·클래스명은 적지 않았다. 상세 워크북은 S/A 항목만 다루며, 같은 역량이 반복되는 경우 대표 문제에 통합했다.

## 우선순위 기준

- **S — 반드시 준비:** 실제 질문과 10~30분 직접 구현 모두 가치가 높다.
- **A — 준비 가치가 높음:** 질문 또는 핵심 구현 가능성이 높다.
- **B — 설명 준비 중심:** 구현보다 설계 판단과 개념 설명의 가치가 높다.
- **C — 별도 항목 불필요:** 보일러플레이트, 반복 위임, 설정 고정처럼 독립 면접 문제로 만들 가치가 낮다.

## 전체 Thread/커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
| --- | ---: | --- | --- | --- | --- | --- | --- | --- |
| B | 01 | `build(maven): establish Java 17 baseline`<br>`build(wrapper): add reproducible Maven wrapper` | `pom.xml`, `mvnw`, `.mvn/wrapper/` | 재현 가능한 런타임·빌드 입력 | 런타임과 도구 버전을 고정하는 이유는 설명 가치가 있지만, 이 프로젝트의 구체적 Maven 설정을 백지 구현할 가치는 낮다. | 높음 | 낮음 | 02 |
| B | 01 | `test(maven): resolve shared protocol 1.0.0`<br>`test(checkstyle): verify semantic rule set` | `SharedProtocolDependencyTest`, `CheckstyleConfigurationTest`, `config/checkstyle/checkstyle.xml` | 빌드 계약을 테스트로 고정하는 방법 | 의존성·정적 분석 설정도 회귀 대상이라는 판단은 일반화되지만 구현은 프로젝트 특화다. | 중간 | 낮음 | 02 |
| C | 01 | `feat(app): bootstrap admin API`<br>`chore(config): define runtime defaults` | `AdminApiApplication`, `application.yml` | 애플리케이션 부트스트랩과 기본 설정 | Spring Boot 진입점과 기본 YAML은 보일러플레이트 비중이 높다. | 낮음 | 낮음 | 06, 13 |
| B | 02 | `build(history): enforce archive commit policy`<br>`build(history): traverse the complete commit chain` | `.github/scripts/verify-history.sh`, `HistoryGuardFixture`, `AdminHistoryGuardPolicyTest` | 히스토리를 실행 가능한 정책으로 검증 | 셸의 실패 전파, 전체 커밋 순회, 인접 커밋 검증은 질문 가치가 있으나 아카이브 규칙 자체는 프로젝트 특화다. | 높음 | 중간 | 01 |
| C | 02 | `ci(archive): verify fixed admin release inputs`<br>`chore(release): release admin API 1.0.0`<br>`docs(project): document admin API contracts` | `.github/workflows/admin-ci.yml`, `pom.xml`, `README.md` | 릴리스 경계와 문서화 | 고정 입력과 릴리스 순서는 중요하지만 독립 코딩 문제보다 운영 규칙 설명에 가깝다. | 중간 | 낮음 | 01 |
| S | 03 | `feat(security): parse CIDR address ranges` | `CidrBlock` | [W01 — CIDR 비트 경계 계산](01-security-and-command-identity.md#W01) | IPv4/IPv6 바이트 표현, prefix 경계, DNS 해석 회피를 직접 구현으로 확인할 수 있다. | 높음 | 높음 | 03 |
| S | 03 | `feat(security): resolve trusted client addresses`<br>`feat(security): enforce the admin IP allowlist` | `TrustedProxyResolver`, `IpAllowlistFilter`, `AdminNetworkProperties` | [W01 — 신뢰 프록시 체인과 fail-closed 해석](01-security-and-command-identity.md#W01) | 잘못된 X-Forwarded-For 신뢰는 즉시 인증 우회가 된다. 오른쪽부터 신뢰 경계를 찾는 이유와 malformed chain 처리가 핵심이다. | 높음 | 높음 | 04 |
| A | 04 | `feat(security): require bounded JWT lifetime`<br>`feat(security): validate operator subjects`<br>`feat(security): validate operator role claims`<br>`feat(security): decode verified RS256 tokens` | `AdminJwtTimestampValidator`, `AdminJwtSubjectValidator`, `AdminJwtRoleValidator`, `JwtDecoderConfiguration`, `RsaPublicKeyParser`, `AdminRole` | [W02 — JWT 서명·시간·주체·역할 신뢰 경계](01-security-and-command-identity.md#W02) | "서명 검증 성공"과 "운영자 토큰으로 신뢰 가능"은 다르다. 시간, issuer, subject, role을 함께 설명해야 한다. | 높음 | 중간 | 03, 05 |
| B | 04 | `feat(error): render RFC 7807 security failures` | `Rfc7807Writer` | 보안 실패의 일관된 외부 표현 | 캐시 방지와 민감정보 비노출은 중요하지만 RFC 7807 직렬화 자체는 독립 구현 가치가 낮다. | 중간 | 낮음 | 07, 09 |
| S | 05 | `feat(context): create mutation identities early` | `AdminMutationContextFilter`, `AdminContextArgumentResolver`, `AdminContext` | [W03 — HTTP 시도 식별자 생성 시점과 request lifecycle](01-security-and-command-identity.md#W03) | 인증 뒤, 권한·검증 실패보다 앞에서 한 번 생성해야 실패까지 같은 action ID로 추적할 수 있다. | 높음 | 높음 | 09, 11, 18 |
| S | 05 | `feat(api): require one idempotency header` | `AdminRequestHeaders`, `AdminRequestException` | [W03 — 논리 요청 멱등 키와 물리 시도 action ID 구분](01-security-and-command-identity.md#W03) | 중복 헤더, raw 값 보존, UUID 전용 계약, 재시도 상관관계를 짧은 구현 문제로 만들 수 있다. | 높음 | 높음 | 16, 18, 19, 20 |
| A | 06 | `feat(client): validate isolated downstream credentials`<br>`feat(client): validate downstream HTTP origins` | `DownstreamCredentials`, `DownstreamProperties` | [W04 — 자격 증명·origin 구성의 fail-fast 검증](01-security-and-command-identity.md#W04) | 누락·재사용 키, 비HTTP origin, 무효 timeout을 시작 시점에 거부하는 경계 설계가 일반화된다. | 높음 | 중간 | 04, 15 |
| A | 06 | `feat(client): isolate the Wallet RestClient`<br>`feat(client): isolate the Risk RestClient`<br>`feat(client): isolate the Odds RestClient`<br>`feat(client): isolate the Settlement RestClient`<br>`test(client): prevent cross-client secret leakage` | `DownstreamClientConfiguration`, `DownstreamHeaders`, `CrossClientSecretIsolationTest` | [W04 — 서비스별 HTTP 클라이언트와 비밀값 격리](01-security-and-command-identity.md#W04) | 공유 interceptor 하나의 잘못된 조건문보다 별도 클라이언트가 자격 증명 누출 범위를 구조적으로 줄인다. | 높음 | 중간 | 15, 16, 17, 18, 19, 20 |
| S | 07 | `feat(client): classify ambiguous downstream failures`<br>`feat(client): recognize wrapped read timeouts` | `DownstreamFailureMapper`, `DownstreamStatusException`, `DownstreamUnavailableException` | [W05 — 실패 확실성 분류와 중복 실행 방지](02-boundary-contracts-and-failure.md#W05) | 4xx는 명시적 거절이지만 timeout·5xx·transport는 명령 적용 여부가 불명확하다. 무조건 재시도하면 중복 변경을 만들 수 있다. | 높음 | 높음 | 05, 09, 11 |
| A | 07 | `feat(api): map unknown downstream outcomes` | `AdminExceptionHandler`, `Rfc7807Writer` | [W05 — 내부 실패 분류를 API 상태와 감사 결과로 변환](02-boundary-contracts-and-failure.md#W05) | 502/504와 감사 `UNKNOWN`의 의미를 일관되게 유지하는 시스템 경계 문제다. | 높음 | 중간 | 09, 11 |
| S | 08 | `feat(client): reject malformed success responses` | `DownstreamContract`, `DownstreamContractException` | [W06 — HTTP 성공과 도메인 성공 증명의 분리](02-boundary-contracts-and-failure.md#W06) | 2xx만 확인하면 잘못된 body·status·빈 응답을 성공으로 오인한다. 구조와 의미를 별도 invariant로 검증해야 한다. | 높음 | 높음 | 16, 17, 19, 20 |
| A | 08 | `feat(client): classify malformed success bodies` | `DownstreamFailureMapper` | [W06 — 역직렬화 실패를 계약 위반으로 분류](02-boundary-contracts-and-failure.md#W06) | converter 예외가 transport 오류로 뭉개지지 않도록 원인 사슬을 해석하는 경계가 까다롭다. | 높음 | 중간 | 07 |
| S | 09 | `feat(audit): weave the fail-closed action lifecycle`<br>`feat(error): fail closed when audit finalization fails` | `AuditAspect`, `AuditService`, `AdminExceptionHandler`, `AuditHttpIntegrationTest` | [W08 — 명령 전후 fail-closed 감사 파이프라인](02-boundary-contracts-and-failure.md#W08) | 감사 시작 실패 시 명령을 막고, 종료 기록 실패 시 성공 응답을 내보내지 않는 순서가 시스템의 핵심 안전 속성이다. | 높음 | 높음 | 05, 07, 11 |
| A | 09 | `test(audit): correlate downstream action identities` | `AuditHttpIntegrationTest`, `AuditHttpTestEnvironment` | [W08 — 비동기 지연을 포함한 end-to-end 상태 관찰 테스트](02-boundary-contracts-and-failure.md#W08) | STARTED가 다운스트림 완료 전에 외부에서 보이는지, 헤더·DB·하위 요청 식별자가 일치하는지 검증한다. | 높음 | 중간 | 05, 11 |
| S | 10 | `feat(audit): preserve the V1 audit migration`<br>`feat(audit): migrate to fail-closed lifecycle states` | `V1__audit_log.sql`, `V2__audit_lifecycle.sql`, `AuditV1ChecksumTest`, `AuditMigrationTest` | [W09 — 배포된 스키마의 호환 진화와 DB 제약](03-audit-state-and-concurrency.md#W09) | 이미 적용된 V1을 수정하지 않고 V2로 legacy row를 보존하며 상태 invariant를 DB에 강제하는 실전 마이그레이션 문제다. | 높음 | 높음 | 11, 12, 14 |
| S | 11 | `feat(audit): guard the terminal lifecycle transition`<br>`test(audit): allow one terminal transition under race` | `AuditWriteRepository`, `AuditTerminalRecord`, `AuditTerminalRaceTest` | [W10 — 조건부 UPDATE로 STARTED→terminal 전이 원자화](03-audit-state-and-concurrency.md#W10) | 읽고-검사-쓰기보다 한 SQL 문장의 조건부 갱신이 경쟁 상황에서 exactly-once terminal claim을 보장한다. | 높음 | 높음 | 10, 12 |
| S | 11 | `feat(audit): honor declared response statuses`<br>`feat(audit): order audit before method authorization` | `AuditOutcomeClassifier`, `AuditAspect`, `AuditMethodSecurityOrderingTest` | [W08 — AOP/보안 순서와 결과 분류](02-boundary-contracts-and-failure.md#W08) | 메서드 권한 거절도 감사하려면 advice 순서가 중요하며, 2xx/4xx/불명확 실패의 분류가 감사 의미를 결정한다. | 높음 | 중간 | 04, 07, 09 |
| A | 11 | `feat(audit): surface persistence failure phases` | `AuditPersistenceException`, `AuditService` | [W08 — BEGIN과 COMPLETE 실패의 구분 및 suppressed context](02-boundary-contracts-and-failure.md#W08) | 같은 DB 오류라도 명령 실행 전과 실행 후의 의미가 다르다. 원래 실패를 보존하면서 최종화 실패를 우선하는 판단이 중요하다. | 높음 | 중간 | 09 |
| S | 12 | `feat(audit): claim stale STARTED rows safely` | `AuditWriteRepository.claimStale` | [W11 — SKIP LOCKED 기반 병렬 stale claim](03-audit-state-and-concurrency.md#W11) | 여러 복구 worker가 같은 행을 중복 처리하지 않으면서 오래된 작업을 제한된 batch로 가져오는 전형적인 동시성 문제다. | 높음 | 높음 | 10, 11 |
| A | 12 | `feat(audit): recover stale actions on schedule`<br>`feat(audit): stream recovered stale actions`<br>`test(audit): stream every stale transition once` | `AuditStaleScheduler`, `AuditStaleSchedulerTest` | [W11 — 복구 스케줄러의 실패 격리·진행성·관측성](03-audit-state-and-concurrency.md#W11) | 한 번의 DB/발행 실패가 다음 scan을 막지 않아야 하며, claimed 수와 scan 실패를 별도로 관측해야 한다. | 높음 | 중간 | 13 |
| A | 13 | `feat(audit): define terminal audit event schema`<br>`feat(audit): publish terminal actions best effort` | `AdminActionRecorded.avsc`, `AdminActionPublisher`, `AuditKafkaConfiguration`, `AuditService` | [W12 — 권위 DB와 best-effort 이벤트 투영](03-audit-state-and-concurrency.md#W12) | DB와 Kafka를 한 원자적 결과처럼 취급하지 않고, 권위 소스와 파생 투영의 실패 의미를 분리한 trade-off가 핵심이다. | 높음 | 중간 | 10, 11, 12 |
| A | 13 | `test(audit): contain Kafka publish failures`<br>`test(audit): deliver terminal events to Kafka`<br>`test(audit): publish terminal events through real Kafka` | `AdminActionPublisherFailureTest`, `AdminActionPublisherBrokerTest`, `AdminActionPublisherRealKafkaTest`, `RealKafkaAuditFixture` | [W12 — 동기·비동기 발행 실패와 자원 cleanup 테스트](03-audit-state-and-concurrency.md#W12) | send 호출 자체의 실패와 future 완료 실패가 다르며, producer·consumer·admin 자원 수명도 검증 대상이다. | 높음 | 중간 | 15 |
| A | 14 | `feat(audit): expose lifecycle read models`<br>`feat(audit): expose filtered audit search`<br>`test(audit): page filtered actions newest first` | `AuditLogController`, `AuditLogRepository.search`, `AuditLogView`, `OffsetPage` | [W15 — 안정 정렬·상한·반개구간을 가진 검색 페이지](04-domain-invariants-and-operability.md#W15) | 같은 시각 행의 결정적 순서, size 상한, 시간 구간 검증은 운영 조회 API에서 자주 묻는 경계다. | 높음 | 높음 | 10 |
| B | 14 | `test(load): add audit read fixture`<br>`test(load): prevent persistent load evidence` | `load-test/scenarios/audit-read.js`, `AuditLoadFixtureTest` | 부하 목표와 증거 보관 경계 | p95/p99와 오류율 목표는 설명 가치가 있지만 k6 스크립트 자체를 구현 문제로 만들 필요는 낮다. W15의 설명 항목에 통합한다. | 중간 | 낮음 | 15 |
| A | 15 | `feat(logging): emit redacted structured events`<br>`test(logging): redact structured secrets` | `RedactedEventJsonProvider`, `logback-spring.xml`, `StructuredLoggingTest` | [W16 — 구조화 로그의 비밀값 마스킹과 한계](04-domain-invariants-and-operability.md#W16) | message뿐 아니라 stack trace와 Bearer·labelled secret 변형을 다뤄야 한다. 정규식 기반 방어의 한계도 설명해야 한다. | 높음 | 중간 | 04, 05, 06, 13 |
| A | 16 | `test(wallet): reject incomplete refund proofs`<br>`feat(wallet): delegate refunds with exact proof`<br>`test(wallet): send exact refund request` | `WalletOperationProof`, `WalletOperationResponse`, `WalletCreditPayload`, `WalletClient`, `WalletClientExactRequestTest` | [W07 — 환불 성공 응답의 요청-응답 상관 증명](02-boundary-contracts-and-failure.md#W07) | 금액만 맞는 응답이 아니라 user, currency, ledger reason, operation group, timestamp까지 일치해야 실제 환불 증거가 된다. | 높음 | 높음 | 05, 06, 08 |
| B | 16 | `feat(wallet): define operator refund contract` | `RefundRequest`, `RefundResponse`, `WalletAdminController` | 입력 정규화·권한·감사 메타데이터 | validation annotation과 얇은 controller 위임은 W07의 경계 설명에 포함하면 충분하다. | 중간 | 낮음 | 04, 11 |
| S | 17 | `feat(risk): verify complete limits snapshots`<br>`test(risk): reject incomplete limits snapshots` | `RiskLimitsResponse`, `RiskLimitType`, `RiskLimitPayload` | [W13 — 집합 완전성·중복·scope invariant 검증](04-domain-invariants-and-operability.md#W13) | 정확히 7개 target을 빠짐없이 한 번씩 요구하는 문제는 Set을 이용한 완전성 검증과 교차 필드 조건을 함께 본다. | 높음 | 높음 | 08 |
| A | 17 | `feat(risk): replace one user limit`<br>`feat(risk): clear one user limit override`<br>`test(risk): clear the exact scoped target` | `RiskClient`, `RiskLimitPayload`, `RiskLimitType` | [W13 — 통화가 필요한 타입과 금지되는 타입의 정확한 target 지정](04-domain-invariants-and-operability.md#W13) | type과 currency의 XOR 성격, 정확한 URI·query 구성, 값 상한을 하나의 도메인 invariant로 통합한다. | 높음 | 중간 | 06, 08 |
| C | 17 | `feat(risk): expose limit administration`<br>`test(risk): delegate every limit operation` | `RiskAdminController` | 역할별 controller 위임 | 권한 annotation과 단순 위임은 대표 invariant 문제에 비해 독립 면접 가치가 낮다. | 낮음 | 낮음 | 04, 11 |
| A | 18 | `feat(odds): delegate all market actions`<br>`test(odds): preserve four market headers` | `MarketClient`, `DownstreamHeaders.ADMIN_ACTION_ID`, `MarketStatusPayload` | [W03에 통합 — 같은 멱등 키, 다른 action ID](01-security-and-command-identity.md#W03) | 재시도마다 물리 시도 ID는 달라지고 논리 명령 키는 유지된다는 차이를 실제 하위 요청에서 보여 주는 대표 사례다. | 높음 | 중간 | 05, 06 |
| C | 18 | `feat(odds): expose market suspension`<br>`feat(odds): expose market closure`<br>`feat(odds): expose market reopening` | `MarketAdminController` | 반복되는 상태 변경 endpoint 위임 | 세 endpoint는 같은 공통 경계를 반복한다. 별도 문제로 늘리지 않고 W03에 묶는다. | 낮음 | 낮음 | 05 |
| A | 19 | `feat(settlement): model candidate evidence`<br>`feat(settlement): verify candidate decisions`<br>`test(settlement): validate candidate decisions` | `SettlementCandidateView`, `SettlementCandidateReceipt`, `SettlementRejectionPayload` | [W14에 통합 — 후보 lifecycle 및 결정 영수증 증명](04-domain-invariants-and-operability.md#W14) | 상태와 `accepted`, 결정 시각, 요청 idempotency key와 결과가 서로 일치해야 한다. 단순 역직렬화보다 상태 증명에 가깝다. | 높음 | 중간 | 08, 20 |
| B | 19 | `test(settlement): fetch exact candidate evidence`<br>`test(settlement): send exact candidate approval` | `SettlementClient`, `SettlementClientCandidateQueryTest`, `SettlementClientCandidateApprovalTest` | 정확한 settlement 경계 호출 | 인증 헤더·경로·빈 body 확인은 중요하지만 W04·W06·W14의 결합 사례로 보는 편이 낫다. | 중간 | 낮음 | 06, 08 |
| S | 20 | `feat(settlement): verify revision lifecycle proof` | `SettlementRevisionView`, `SettlementRevisionProof` | [W14 — 다중 상태·lease·wallet 증거의 교차 필드 검증](04-domain-invariants-and-operability.md#W14) | 19개 필드가 독립적으로 유효한지만 보는 것이 아니라 상태별 허용 조합, 시간 순서, retry budget을 검증해야 한다. | 높음 | 높음 | 08, 19 |
| A | 20 | `feat(settlement): verify revision retry receipts`<br>`feat(settlement): retry paused revisions` | `SettlementRetryReceipt`, `SettlementClient.retryRevision` | [W14 — 재시도 요청과 영수증 상관관계](04-domain-invariants-and-operability.md#W14) | QUEUED와 REPLAY의 의미, 시도 횟수, 다음 실행 시각, 요청 key 일치를 검증하는 축소 구현 문제가 가능하다. | 높음 | 높음 | 05, 07, 19 |
| C | 20 | `feat(settlement): expose lifecycle queries`<br>`feat(settlement): expose revision retries` | `SettlementQueryController`, `SettlementRevisionCommandController` | 조회·명령 controller 계약 | 역할·상태코드·위임은 확인 가치가 있으나 대표 상태 증명 문제보다 우선도가 낮다. | 중간 | 낮음 | 04, 11 |

## 대표 면접 포인트와 상세 워크북 위치

| ID | 대표 Thread | 통합한 연관 Thread | 대표 역량 | 상세 문서 |
| --- | --- | --- | --- | --- |
| W01 | 03 | 03 | CIDR, IPv4/IPv6, trusted proxy, fail-closed allowlist | [01-security-and-command-identity.md](01-security-and-command-identity.md#W01) |
| W02 | 04 | 03, 05 | JWT cryptographic/claim trust boundary | [01-security-and-command-identity.md](01-security-and-command-identity.md#W02) |
| W03 | 05 | 18, 09, 11 | action ID, idempotency key, request lifecycle | [01-security-and-command-identity.md](01-security-and-command-identity.md#W03) |
| W04 | 06 | 15, 16, 17, 18, 19, 20 | origin·credential·RestClient isolation | [01-security-and-command-identity.md](01-security-and-command-identity.md#W04) |
| W05 | 07 | 09, 11 | downstream failure certainty and relay | [02-boundary-contracts-and-failure.md](02-boundary-contracts-and-failure.md#W05) |
| W06 | 08 | 16, 17, 19, 20 | structural and semantic success contracts | [02-boundary-contracts-and-failure.md](02-boundary-contracts-and-failure.md#W06) |
| W07 | 16 | 05, 06, 08 | exact wallet refund proof | [02-boundary-contracts-and-failure.md](02-boundary-contracts-and-failure.md#W07) |
| W08 | 09 | 05, 07, 11 | end-to-end fail-closed command/audit pipeline | [02-boundary-contracts-and-failure.md](02-boundary-contracts-and-failure.md#W08) |
| W09 | 10 | 11, 12, 14 | compatible migration and DB constraints | [03-audit-state-and-concurrency.md](03-audit-state-and-concurrency.md#W09) |
| W10 | 11 | 10, 12 | atomic guarded terminal transition | [03-audit-state-and-concurrency.md](03-audit-state-and-concurrency.md#W10) |
| W11 | 12 | 10, 11, 13 | concurrent stale claim and recovery scheduler | [03-audit-state-and-concurrency.md](03-audit-state-and-concurrency.md#W11) |
| W12 | 13 | 10, 11, 12, 15 | authoritative DB and best-effort Kafka projection | [03-audit-state-and-concurrency.md](03-audit-state-and-concurrency.md#W12) |
| W13 | 17 | 08 | complete risk target set and scope invariants | [04-domain-invariants-and-operability.md](04-domain-invariants-and-operability.md#W13) |
| W14 | 20 | 19, 08, 05, 07 | settlement candidate/revision/retry proof | [04-domain-invariants-and-operability.md](04-domain-invariants-and-operability.md#W14) |
| W15 | 14 | 10 | stable bounded audit pagination and load boundary | [04-domain-invariants-and-operability.md](04-domain-invariants-and-operability.md#W15) |
| W16 | 15 | 04, 05, 06, 13 | secret-safe structured logging | [04-domain-invariants-and-operability.md](04-domain-invariants-and-operability.md#W16) |

## S/A 완전성 검증

아래 표는 위 선별표의 모든 S/A 행을 상세 워크북과 대조한 결과다. "통합"은 해당 Thread가 대표 문제의 사례 또는 꼬리 질문으로 명시적으로 포함되었음을 뜻한다.

| 선별 항목 | 상태 | 상세 위치 |
| --- | --- | --- |
| T03 CIDR 파싱 | 독립 상세 항목 | W01 |
| T03 trusted proxy/IP allowlist | W01에 통합 | W01 |
| T04 JWT 신뢰 경계 | 독립 상세 항목 | W02 |
| T05 mutation action ID | 독립 상세 항목 | W03 |
| T05 idempotency header | W03에 통합 | W03 |
| T06 credential/origin 검증 | 독립 상세 항목 | W04 |
| T06 RestClient 격리 | W04에 통합 | W04 |
| T07 ambiguous failure 분류 | 독립 상세 항목 | W05 |
| T07 API mapping | W05에 통합 | W05 |
| T08 malformed success 검증 | 독립 상세 항목 | W06 |
| T08 conversion failure 분류 | W06에 통합 | W06 |
| T09 fail-closed pipeline | 독립 상세 항목 | W08 |
| T09 E2E 상관관계 테스트 | W08에 통합 | W08 |
| T10 migration/DB invariant | 독립 상세 항목 | W09 |
| T11 guarded terminal transition | 독립 상세 항목 | W10 |
| T11 AOP/security ordering | W08에 통합 | W08 |
| T11 persistence phase 구분 | W08에 통합 | W08 |
| T12 stale claim | 독립 상세 항목 | W11 |
| T12 scheduler/streaming | W11에 통합 | W11 |
| T13 authoritative DB/best-effort Kafka | 독립 상세 항목 | W12 |
| T13 Kafka failure/resource test | W12에 통합 | W12 |
| T14 audit search/pagination | 독립 상세 항목 | W15 |
| T15 secret-safe logging | 독립 상세 항목 | W16 |
| T16 wallet refund proof | 독립 상세 항목 | W07 |
| T17 complete limits snapshot | 독립 상세 항목 | W13 |
| T17 set/clear exact target | W13에 통합 | W13 |
| T18 market idempotency/action headers | W03에 통합 | W03 |
| T19 candidate evidence/receipt | W14에 통합 | W14 |
| T20 revision lifecycle proof | 독립 상세 항목 | W14 |
| T20 retry receipt | W14에 통합 | W14 |

**검증 결과:** 고아 S/A 항목 없음. B/C 항목에는 별도 상세 워크북을 만들지 않았다.

## 백지 구현 우선순위

1. W10 — 조건부 UPDATE 기반 원자적 terminal transition
2. W05 — downstream 실패 확실성 분류
3. W01 — CIDR와 trusted proxy chain 해석
4. W13 — 완전한 risk target 집합 검증
5. W14 — settlement revision/retry 상태 증명
6. W03 — action ID와 idempotency key lifecycle
7. W11 — `SKIP LOCKED` 기반 stale claim
8. W06 — 구조·의미 성공 계약 검증
9. W09 — 호환 migration과 DB constraint 설계
10. W07 — wallet refund 요청-응답 증명
11. W15 — 안정 정렬과 상한이 있는 페이지 요청 정규화
12. W02 — JWT claim policy 검증
13. W04 — 서비스별 credential/header 격리
14. W16 — secret redaction 함수
15. W08 — fail-closed orchestration
16. W12 — best-effort projection wrapper

## 설명 연습 우선순위

1. W08 — 왜 감사 BEGIN/COMPLETE가 명령보다 강한 실패 경계를 갖는가
2. W12 — 왜 DB는 권위이고 Kafka는 best effort인가
3. W09 — 이미 배포된 migration을 수정하지 않는 이유와 legacy backfill
4. W05 — 명시적 거절과 결과 불명확 실패를 구분하는 이유
5. W03 — 논리 요청 ID와 물리 시도 ID의 차이
6. W02 — 서명 검증 이후에도 claim 검증이 필요한 이유
7. W14 — 상태별 nullable field 조합을 증명으로 보는 방법
8. W11 — `FOR UPDATE SKIP LOCKED`의 진행성과 공정성 trade-off
9. W04 — 별도 RestClient가 secret leakage를 구조적으로 줄이는 이유
10. W15 — offset pagination의 안정 정렬과 깊은 페이지 비용
11. W16 — 정규식 redaction을 최종 보안 경계로 볼 수 없는 이유
12. W13 — size 검사만으로 완전성을 보장할 수 없는 이유
13. W06 — 2xx와 도메인 성공의 차이
14. W07 — 환불 응답을 증거로 인정하기 위한 최소 필드
15. W10 — SQL 조건부 갱신과 애플리케이션 락의 차이
16. W01 — X-Forwarded-For를 오른쪽부터 읽는 이유

## 한 문제로 통합한 Thread 묶음

- **T03 단일 보안 경계:** CIDR 계산, trusted proxy 해석, IP allowlist를 W01 하나로 통합했다.
- **T05 + T18:** action ID와 idempotency key의 차이를 market 재시도 헤더 사례와 함께 W03으로 통합했다.
- **T06 + T16/T17/T18/T19/T20:** 서비스별 credential/header 차이는 W04가 대표하며 각 도메인 client는 사례로만 사용한다.
- **T07 + T08:** 실패 확실성(W05)과 성공 계약(W06)은 서로 다른 축으로 분리하되, 같은 downstream boundary 묶음으로 배치했다.
- **T09 + T11:** AOP 순서, outcome 분류, BEGIN/COMPLETE 실패를 end-to-end fail-closed 파이프라인 W08로 통합했다.
- **T10 + T11 + T12 + T13:** migration, atomic transition, stale recovery, event projection은 감사 시스템 묶음으로 같은 문서에 두되 각기 독립 면접 포인트를 유지했다.
- **T16 + T08:** wallet proof는 일반 계약 검증의 구체 사례지만 금전 변경의 위험 때문에 W07로 독립 유지했다.
- **T17의 조회·변경:** complete snapshot과 typed/currency target 규칙을 W13 하나로 통합했다.
- **T19 + T20:** candidate evidence, decision receipt, revision lifecycle, retry receipt를 상태 증명 문제 W14로 통합했다.
- **T14 load fixture:** 별도 문제로 만들지 않고 W15의 성능·운영 꼬리 질문에 포함했다.
