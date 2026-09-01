# 검수·봉인·공개 전환·권한 분리

append-only review, immutable revision, CAS pointer, 복구 정책과 control plane을 다룬다.

> 이 문서는 정답 코드를 제공하지 않는다. 백지 구현은 원본을 다시 보기 전에 수행한다.

## P18. [Thread 11 / `feat(history): bind reviews to collections`] append-only review chain과 current-tail 승인

**우선순위:** A

### 면접 질문

- 승인 row를 update하지 않고 supersedes chain으로 남긴 이유는 무엇인가요?
- APPROVE decision을 collection result hash와 partition manifest hash에 묶어야 하는 이유는 무엇인가요?
- 동시에 두 재검수 decision이 같은 tail을 supersede하려 하면 어떻게 막나요?
- 꼬리 질문: UUID replay를 허용하면서 다른 evidence replay는 왜 거부합니까?

### 30초 모범 답변

사람 결정은 감사 기록이므로 수정 대신 새 decision이 이전 tail을 supersede하도록 append-only로 남깁니다. 승인은 단순 collection id가 아니라 검증된 result와 partition manifest의 정확한 hash에 묶어 이후 내용 변경을 승인으로 오인하지 않게 합니다. 현재 tail만 supersede할 수 있고 잠금과 DB guard로 fork를 막으며, 같은 UUID와 동일 evidence replay만 idempotent하게 허용합니다.

### 답변 핵심 키워드

`append-only decision`, `supersedes tail`, `hash-bound approval`, `authorization`, `idempotent UUID`, `fork prevention`

### 백지 구현

**구현 목표**

collection review decision을 append-only chain에 기록하는 순수 또는 메모리 구현을 작성한다.

**인터페이스 또는 함수 시그니처**

```python
def record_review(
    collection: ValidatedCollection,
    decisions: Sequence[ReviewDecision],
    command: ReviewCommand,
) -> ReviewDecision:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: VALIDATED collection, 기존 decision chain, reviewer/UUID/evidence/결정 command
- 출력: 새 decision 또는 동일 replay의 기존 decision

**반드시 만족해야 할 조건**

- reviewer가 active/authenticated이고 정확한 review 권한을 가져야 한다.
- APPROVE는 collection result/partition hash와 정확히 일치해야 한다.
- REJECT와 APPROVE의 필드 shape를 구분한다.
- supersedes가 있으면 같은 collection의 현재 tail이어야 한다.
- 같은 command UUID replay는 모든 필드가 같을 때만 허용한다.
- 기존 decision은 수정·삭제하지 않는다.

**경계 조건**

- 첫 decision
- APPROVE 뒤 재검수 APPROVE
- REJECT 뒤 새 APPROVE
- 동일 replay

**실패 조건**

- 권한 없는 actor
- 다른 collection tail supersede
- 이미 superseded 된 decision 재사용
- 승인 hash mismatch
- 동일 UUID의 evidence 충돌

**제약**

- 결정 chain을 in-place 수정하지 않는다.
- 20분 이내 구현한다.
- 원문 evidence는 받지 않고 SHA-256만 받는다.

### 구현 후 자가 검증

- [ ] 첫 decision과 tail supersede가 정상 동작한다.
- [ ] fork가 허용되지 않는다.
- [ ] hash mismatch가 저장 전 실패한다.
- [ ] 동일 replay와 충돌 replay가 구분된다.
- [ ] 기존 chain이 입력/출력 과정에서 변경되지 않는다.

### 구현 후 설명할 것

- append-only review가 감사·정정에 주는 이점
- 승인을 content hash에 묶는 이유
- tail lock과 unique constraint의 역할
- actor 권한을 서비스와 DB에서 중복 확인하는 이유

### 원본 확인 위치

- Thread 05, Thread 11
- 커밋 `feat(review): append generation decisions`
- 커밋 `feat(history): bind reviews to collections`
- 커밋 `feat(history): authorize collection reviews`
- 파일 `grocery/historical_reviews.py`; 구성 요소 `record_historical_review_decision`

## P19. [Thread 11 / `fix(history): seal only exact reviewed bundles`] 불변 publication 봉인과 canonical fact-set hash

**우선순위:** A

### 면접 질문

- review 승인과 publication seal을 분리한 이유는 무엇인가요?
- 월별·지역별·시장별 세 review가 모두 current이고 서로 달라야 하는 이유는 무엇인가요?
- publication membership hash를 DB row id 나열이 아니라 canonical fact data로 계산할 때의 장점은 무엇인가요?
- 꼬리 질문: copy revision을 hash 또는 unique key에 포함하는 이유는 무엇인가요?

### 30초 모범 답변

review는 source collection 수용 결정이고 seal은 공개 단위의 불변 snapshot 생성이라 역할을 분리했습니다. 세 source review는 각각 올바른 kind, current tail, 동일 code manifest, 정확한 승인 hash를 만족해야 합니다. 멤버십과 typed fact를 안정적으로 정렬해 contract version과 함께 hash하면 DB 생성 순서와 무관하게 동일 내용을 재현할 수 있고, copy revision은 같은 사실의 표시 계약 변경을 별도 revision으로 구분합니다.

### 답변 핵심 키워드

`seal`, `immutable revision`, `current review`, `canonical membership`, `fact-set hash`, `copy revision`

### 백지 구현

**구현 목표**

세 source의 승인 review와 typed facts에서 immutable publication revision을 봉인한다.

**인터페이스 또는 함수 시그니처**

```python
def seal_publication(
    reviews: PublicationReviews,
    facts: Sequence[TypedFact],
    copy_revision: str,
) -> SealedRevision:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: monthly/regional/market current APPROVE review, typed facts, copy revision
- 출력: count·기간·canonical fact-set hash를 가진 sealed revision

**반드시 만족해야 할 조건**

- 세 review id가 서로 다르고 각각 기대한 collection kind에 속한다.
- 각 review가 current tail이며 승인 hash가 collection과 일치한다.
- 세 collection의 code manifest가 동일하다.
- facts는 승인된 collection에 정확히 속하고 중복 semantic key가 없다.
- canonical order와 contract version으로 hash/count/range를 계산한다.
- 동일 내용·copy revision replay는 동일 revision으로 확인하고 충돌은 실패한다.

**경계 조건**

- 각 source에 fact가 하나뿐인 최소 bundle
- facts 입력 순서가 무작위인 경우
- 동일 facts, 다른 copy revision
- seal replay

**실패 조건**

- superseded review
- kind 또는 code manifest mismatch
- 중복 review 사용
- fact count/range mismatch
- 봉인 후 mutation 시도

**제약**

- activation까지 수행하지 않는다.
- 기존 revision을 update하지 않는다.
- 25분 이내 핵심 검증과 canonical hash를 구현한다.

### 구현 후 자가 검증

- [ ] unordered facts가 같은 hash를 만든다.
- [ ] superseded review로 seal되지 않는다.
- [ ] 세 review를 잘못 매핑하면 실패한다.
- [ ] copy revision 변경이 별도 revision으로 구분된다.
- [ ] seal replay가 중복 revision을 만들지 않는다.

### 구현 후 설명할 것

- review와 seal의 책임 분리
- canonical content hash와 row-id hash의 차이
- copy revision을 사실 hash와 분리 또는 결합하는 기준
- seal 시 전체 fact scan 비용과 강한 무결성의 trade-off

### 원본 확인 위치

- Thread 05, Thread 11
- 커밋 `feat(publication): seal immutable revisions`
- 커밋 `feat(history): define publication bundles`
- 커밋 `fix(history): seal only exact reviewed bundles`
- 파일 `grocery/historical_publication_models.py`; 구성 요소 `HistoricalRetailPublicationRevision`, `seal_historical_publication`

## P20. [Thread 05 / `feat(publication): activate revisions atomically`] CAS publication 전환과 event-pointer 원자성

**우선순위:** S

### 면접 질문

- publication channel에 `expected_version`과 `expected_current_revision`을 둘 다 요구한 이유는 무엇인가요?
- activation event insert와 current pointer update를 한 트랜잭션으로 묶지 않으면 어떤 감사 불일치가 생기나요?
- 같은 operation UUID 재실행을 언제 성공 replay로 보고 언제 충돌로 보나요?
- ACTIVATE·ROLLBACK·WITHDRAW의 target/previous shape invariant를 설명해 주세요.
- 꼬리 질문: 낙관적 CAS와 row lock을 함께 쓰는 이유는 무엇인가요?

### 30초 모범 답변

채널 전환은 현재 pointer를 읽고 쓰는 경쟁 작업이므로 예상 version과 예상 current를 모두 대조해 stale operator 명령을 거부합니다. 채널 row를 잠근 뒤 operation shape와 sealed 또는 과거-current 자격을 검증하고, append-only event와 pointer/version 증가를 같은 트랜잭션으로 기록합니다. 같은 operation UUID와 동일 command는 replay로 반환하지만 target·expected 값이 다르면 충돌이며, DB guard도 event와 pointer의 정확한 대응을 강제합니다.

### 답변 핵심 키워드

`compare-and-swap`, `expected version`, `expected current`, `append-only event`, `atomic pointer`, `operation idempotency`, `row lock`

### 백지 구현

**구현 목표**

단일 publication channel에 ACTIVATE/ROLLBACK/WITHDRAW command를 CAS 방식으로 적용하고 append-only event를 생성한다.

**인터페이스 또는 함수 시그니처**

```python
def transition_publication(
    channel: ChannelState,
    history: Sequence[ActivationEvent],
    command: TransitionCommand,
) -> TransitionResult:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: current revision/version을 가진 channel, 기존 event history, operation command
- 출력: 새 channel state와 event 또는 동일 replay 결과

**반드시 만족해야 할 조건**

- command의 expected version/current가 channel과 정확히 일치해야 한다.
- 새 event sequence는 이전 version+1이다.
- event.previous_revision은 전이 전 current와 일치한다.
- ACTIVATE/ROLLBACK은 target이 있고 WITHDRAW는 target이 없다.
- ACTIVATE target은 sealed·eligible, ROLLBACK target은 sealed·previously current이다.
- target은 current와 같을 수 없다.
- event append와 pointer/version 변경은 전부 성공하거나 전부 실패한다.
- operation UUID replay는 모든 command 필드가 같을 때만 허용한다.

**경계 조건**

- version 0, current 없음에서 최초 ACTIVATE
- 활성 상태에서 WITHDRAW
- withdrawn 상태에서 과거 revision으로 ROLLBACK
- 동일 operation replay

**실패 조건**

- stale expected version/current
- unsealed 또는 존재하지 않는 target
- 한 번도 current가 아니었던 rollback target
- operation shape 위반
- 동일 UUID의 다른 command

**제약**

- channel/event 입력을 in-place로 변경하지 않고 새 결과를 만든다.
- 25~30분 내 구현한다.
- recent/historical channel 모두 적용 가능한 일반 규칙으로 작성한다.

### 구현 후 자가 검증

- [ ] 최초 activation의 previous가 None이고 sequence가 1이다.
- [ ] stale CAS가 event나 pointer를 만들지 않는다.
- [ ] WITHDRAW 후 current가 None이며 event에 이전 revision이 남는다.
- [ ] ROLLBACK eligibility가 history로 검증된다.
- [ ] 동일 replay가 sequence를 증가시키지 않는다.
- [ ] 충돌 replay가 기존 history를 변경하지 않는다.

### 구현 후 설명할 것

- version과 current를 함께 비교하는 이유
- event sourcing 전체가 아니라 audit event+current pointer를 선택한 trade-off
- row lock과 CAS가 중복처럼 보여도 둘 다 필요한 이유
- DB trigger가 event-pointer 정합성을 보강하는 방식

### 원본 확인 위치

- Thread 05, Thread 11, Thread 13
- 커밋 `feat(publication): activate revisions atomically`
- 구성 요소 `transition_recent_publication`, `transition_historical_publication`
- 모델 `PublicationChannel`, `PublicationActivation`, `HistoricalRetailPublicationChannel`, `HistoricalRetailPublicationActivation`
- migration `0026_guard_historical_activation_cas.py`

## P21. [Thread 11 / `fix(history): preserve last-known-good rollback`] 현재 승인과 마지막 정상본 rollback의 다른 자격 규칙

**우선순위:** A

### 면접 질문

- review가 supersede된 과거 revision을 새로 ACTIVATE하면 안 되지만 ROLLBACK은 허용할 수 있는 이유는 무엇인가요?
- last-known-good라는 주장을 어떤 history evidence로 제한해야 하나요?
- publication rollback과 application rollback을 같은 개념으로 취급하면 어떤 문제가 생기나요?
- 꼬리 질문: 과거 publication 자체가 보안상 폐기되어야 할 때는 이 정책을 어떻게 확장하겠습니까?

### 30초 모범 답변

새 ACTIVATE는 현재 review 기준을 모두 통과해야 하지만, 장애 복구용 ROLLBACK은 이미 seal되고 실제로 current였던 revision이라는 과거 운영 증거를 근거로 허용합니다. superseded review라는 이유만으로 검증된 마지막 정상본을 복구하지 못하면 가용성이 떨어집니다. 다만 아무 과거 revision이 아니라 activation history에 있는 exact target만 허용하며, 코드 배포 rollback과 publication pointer rollback은 별도 절차로 다룹니다.

### 답변 핵심 키워드

`last-known-good`, `eligibility by operation`, `activation history`, `superseded review`, `availability`, `separate rollback domains`

### 백지 구현

**구현 목표**

operation별 target 자격 정책을 구현한다. ACTIVATE와 ROLLBACK이 서로 다른 review freshness 요구를 갖게 한다.

**인터페이스 또는 함수 시그니처**

```python
def is_transition_target_eligible(
    operation: Operation,
    target: Revision | None,
    history: Sequence[ActivationEvent],
) -> bool:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: operation, target revision, append-only activation history
- 출력: target 자격 여부 또는 구체적 안전 오류

**반드시 만족해야 할 조건**

- WITHDRAW는 target이 없어야 한다.
- ACTIVATE는 sealed이고 current exact reviews를 가진 target만 허용한다.
- ROLLBACK은 sealed이고 동일 channel에서 과거 current였던 target만 허용한다.
- 단순히 revision이 존재한다는 이유로 rollback을 허용하지 않는다.
- 폐기 또는 revoked 상태가 추가되면 두 operation 모두 거부하도록 확장 가능해야 한다.

**경계 조건**

- review가 방금 supersede된 last-known-good
- withdraw 후 rollback
- 한 번도 activate되지 않은 sealed revision
- 현재 revision과 같은 target

**실패 조건**

- ACTIVATE가 superseded review를 우회
- ROLLBACK이 다른 channel history를 사용
- unsealed target
- target 없는 ACTIVATE/ROLLBACK

**제약**

- pointer 변경 자체는 구현하지 않는다.
- 15분 이내 정책 함수로 작성한다.
- history를 수정하지 않는다.

### 구현 후 자가 검증

- [ ] current approved target은 ACTIVATE 가능하다.
- [ ] superseded target은 ACTIVATE 불가다.
- [ ] 같은 superseded target이 과거 current였다면 ROLLBACK 가능하다.
- [ ] 한 번도 current가 아니면 ROLLBACK 불가다.
- [ ] WITHDRAW shape가 분리된다.

### 구현 후 설명할 것

- 검수 freshness와 운영 복구 가능성의 trade-off
- last-known-good를 history로 증명하는 이유
- publication rollback과 application/database rollback의 분리
- 추후 revocation 개념을 추가할 확장 지점

### 원본 확인 위치

- Thread 11
- 커밋 `fix(history): preserve last-known-good rollback`
- 구성 요소 `transition_historical_publication`
- historical activation DB guard
- 연관 Thread 22

## P22. [Thread 12 / 프로덕션 control plane 도입 커밋(제목은 현재 기록에서 확인되지 않음)] 역할 분리, release lock, 외부 인증 경계

**우선순위:** A

### 면접 질문

- reviewer와 publisher actor를 분리하고 exact permission set을 강제한 이유는 무엇인가요?
- control-plane enable flag와 expected release SHA가 인증이 아닌 이유는 무엇인가요?
- 고정 actor에 usable password·group·staff/superuser 권한을 허용하지 않은 이유는 무엇인가요?
- 꼬리 질문: application permission과 role-specific DB credential을 둘 다 써야 하는 이유는 무엇인가요?

### 30초 모범 답변

검수와 공개를 한 주체가 모두 수행하지 못하도록 고정 reviewer/publisher를 분리하고 각 actor의 identity와 exact permission set을 fail-closed로 검증했습니다. enable flag와 release SHA는 잘못된 환경·release의 우발 실행을 막는 lock일 뿐 사용자 인증이 아니므로 실제 production에서는 외부 MFA/IAM과 역할별 DB credential이 필요합니다. actor drift나 추가 권한이 보이면 자동 보정하지 않고 중단합니다.

### 답변 핵심 키워드

`separation of duties`, `least privilege`, `exact permissions`, `release lock`, `external MFA/IAM`, `defense in depth`, `drift detection`

### 백지 구현

**구현 목표**

설정·release SHA·actor registry를 검증해 지정 역할의 operation actor를 반환하는 축소 control-plane resolver를 구현한다.

**인터페이스 또는 함수 시그니처**

```python
def resolve_operation_actor(
    *,
    role: Role,
    expected_release_sha: str,
    runtime: RuntimeContract,
    actors: ActorStore,
) -> OperationActor:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: reviewer/publisher role, expected SHA, runtime flag/DEPLOY_VERSION, actor 저장소
- 출력: exact actor id와 production 여부를 가진 `OperationActor`

**반드시 만족해야 할 조건**

- production operation flag가 명시적으로 활성화되어야 한다.
- expected SHA와 runtime deploy SHA가 모두 정확한 40자리 소문자 hex이고 일치해야 한다.
- role별 고정 username과 exact permission set을 검증한다.
- actor는 active, non-login, non-staff, non-superuser, no-group여야 한다.
- 추가 권한도 drift로 거부한다.
- 오류는 고정 code만 반환하고 actor name/SHA/evidence를 반사하지 않는다.

**경계 조건**

- actor가 존재하지만 permission 하나가 부족한 경우
- 기존 actor의 permission drift
- SHA 대소문자/길이 경계
- local operation과 production operation 분기

**실패 조건**

- flag만 켠 것을 인증으로 간주
- 추가 group/permission drift를 허용
- release SHA mismatch
- actor 누락 또는 로그인 가능 계정
- 오류 메시지에 민감 정보 반사

**제약**

- 실제 MFA/IAM을 구현하지 않고 반드시 외부 경계라고 명시한다.
- 20~25분 구현 크기다.
- 권한을 와일드카드로 검사하지 않는다.

### 구현 후 자가 검증

- [ ] reviewer가 publisher 권한을 얻지 않는다.
- [ ] 추가 권한 하나만 있어도 거부된다.
- [ ] SHA mismatch에서 actor 조회/변경이 일어나지 않는다.
- [ ] 오류가 고정 code로 redacted 된다.
- [ ] resolver가 인증을 수행한다고 오해할 수 있는 이름/설명이 없다.

### 구현 후 설명할 것

- separation of duties와 least privilege
- feature flag/release lock과 인증의 차이
- exact permission 검증의 운영 부담 대 안전성
- application 권한과 DB 권한을 겹쳐 쓰는 defense in depth

### 원본 확인 위치

- Thread 12
- 파일 `grocery/management/control_plane.py`
- 구성 요소 `resolve_operation_actor`, `bootstrap_control_plane_actors`, `OperationActor`, `preflight_operation`, `require_production_operation_environment`
- 커밋 `feat(history): expose collection approval command`
- 파일 `grocery/management/commands/bootstrap_control_plane_actors.py`

## P23. [Thread 13 / `feat(public): connect recent catalog extensions`] recent·historical publication 채널 독립성과 장애 격리

**우선순위:** A

### 면접 질문

- 최근값과 역사값을 하나의 publication revision으로 합치지 않고 독립 channel로 둔 이유는 무엇인가요?
- 상세 화면에서 historical read가 실패할 때 recent 상세까지 503으로 만들지 않은 이유는 무엇인가요?
- 한 응답에 두 publication fact-set header를 넣을 때 어떤 일관성을 검증해야 하나요?
- 꼬리 질문: 두 channel을 원자적으로 같은 시점에 보여 줘야 하는 요구가 생기면 현재 설계의 한계는 무엇인가요?

### 30초 모범 답변

최근값과 역사값은 source cadence·검수·실패 영역이 달라 독립 channel로 공개합니다. recent 상세는 자체 active revision만으로 완전하므로 historical 부가 기능이 실패하면 로그를 남기고 링크나 section만 숨겨 핵심 경로를 유지합니다. 두 데이터를 함께 보여 주는 응답은 각각의 active fact-set hash를 별도 header로 고정하며 요청 중 임의로 다른 revision을 섞지 않습니다.

### 답변 핵심 키워드

`independent channels`, `failure isolation`, `graceful degradation`, `fact-set header`, `bounded consistency`, `optional dependency`

### 백지 구현

**구현 목표**

recent 데이터는 필수, historical 데이터는 선택적 의존성인 상세 상태 로더를 구현한다.

**인터페이스 또는 함수 시그니처**

```python
def load_detail_state(
    load_recent: Callable[[], RecentState],
    load_historical: Callable[[], HistoricalState],
) -> DetailState:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: recent/historical active publication loader
- 출력: recent 필수 내용과 optional historical section/header를 가진 상태

**반드시 만족해야 할 조건**

- recent 없음 또는 무결성 실패는 전체 상세 unavailable이다.
- historical 없음·DB 실패·validation 실패는 historical section만 숨긴다.
- historical failure 내용을 사용자 응답에 반사하지 않는다.
- 각 channel의 fact-set hash를 별도 필드로 유지한다.
- historical row를 recent fallback으로 사용하거나 반대 방향으로 섞지 않는다.

**경계 조건**

- recent만 활성화된 상태
- 두 channel 모두 활성화된 상태
- historical mapping이 해당 recent series에 없는 상태
- historical loader가 예외를 던지는 상태

**실패 조건**

- historical 장애가 recent 응답을 실패시킴
- 서로 다른 channel hash를 하나로 합침
- historical source를 request 중 직접 호출
- 무결성 오류를 빈 데이터로 오인

**제약**

- 실제 template rendering은 하지 않는다.
- 15~20분 구현 크기다.
- catch-all 예외를 사용한다면 안전하게 허용할 범위를 설명한다.

### 구현 후 자가 검증

- [ ] recent-only가 정상 응답한다.
- [ ] historical 실패가 section만 제거한다.
- [ ] recent 실패는 성공으로 위장되지 않는다.
- [ ] 두 hash가 서로 덮어쓰지 않는다.
- [ ] 외부 source 호출이 없다.

### 구현 후 설명할 것

- 채널 독립성이 배포·복구·freshness에 주는 이점
- graceful degradation과 오류 은폐의 경계
- 교차 채널 강한 일관성이 필요할 때의 대안
- optional dependency 예외 범위를 좁히는 방법

### 원본 확인 위치

- Thread 13, Thread 16
- 커밋 `feat(public): connect recent catalog extensions`
- 파일 `grocery/historical_public_read.py`
- 구성 요소 `load_active_historical_publication`, `historical_series_for_recent`
- 파일 `grocery/views.py`; 구성 요소 `detail`
