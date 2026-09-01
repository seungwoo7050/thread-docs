# 분할 수집·collection 정합성·동시성

partition plan에서 collection completion과 DB 직렬화까지의 invariant를 다룬다.

> 이 문서는 정답 코드를 제공하지 않는다. 백지 구현은 원본을 다시 보기 전에 수행한다.

## P14. [Thread 09 / `feat(history): plan ordered source partitions`] 결정적 partition 계획과 manifest

**우선순위:** A

### 면접 질문

- 역사 수집을 하나의 큰 요청이 아니라 ordered partition으로 나눈 이유는 무엇인가요?
- partition manifest가 ordinal과 scope hash를 함께 고정해야 하는 이유는 무엇인가요?
- 동일 범위 재계획과 범위 변경을 어떻게 구분하나요?
- 꼬리 질문: partition 크기를 너무 작게 또는 크게 잡을 때의 운영 trade-off는 무엇인가요?

### 30초 모범 답변

대량 역사 수집은 요청·재시도·감사 범위를 제한하기 위해 partition으로 나눕니다. 계획은 semantic scope를 안정적으로 정렬해 1부터 연속 ordinal을 부여하고, 그 순서와 scope hash 전체를 manifest에 묶습니다. 동일 입력은 같은 plan과 hash를 재현하지만 하나의 범위라도 바뀌면 다른 collection plan이 되어 이전 part와 섞이지 않습니다.

### 답변 핵심 키워드

`partition plan`, `contiguous ordinal`, `scope hash`, `manifest`, `determinism`, `bounded retry`

### 백지 구현

**구현 목표**

unordered scope 목록을 검증해 안정적인 ordered partition plan과 manifest hash를 만든다.

**인터페이스 또는 함수 시그니처**

```python
def plan_partitions(scopes: Sequence[PartitionScope]) -> CollectionPlan:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: dataset/mode와 기간·지역 등의 semantic scope 목록
- 출력: ordinal 1..N의 immutable parts와 manifest SHA-256

**반드시 만족해야 할 조건**

- 모든 scope가 같은 source configuration과 collection kind에 속한다.
- scope 중복을 허용하지 않는다.
- canonical scope key로 안정적으로 정렬한다.
- ordinal은 1부터 빈틈없이 부여한다.
- manifest는 contract version과 ordered `(ordinal, scope_sha256)`를 포함한다.
- 빈 plan은 거부한다.

**경계 조건**

- 입력 순서가 역순인 경우
- 월 경계·연도 경계
- 선행 0 region code
- 최대 part 수 경계

**실패 조건**

- 중복 scope
- 서로 다른 dataset/mode 혼합
- 겹치거나 잘못된 기간
- part 수 상한 초과

**제약**

- 네트워크 호출이나 DB 저장은 하지 않는다.
- O(n log n) 이내로 구현한다.
- 15~20분 구현 크기다.

### 구현 후 자가 검증

- [ ] 입력 순서를 바꿔도 plan/hash가 같다.
- [ ] scope 하나가 바뀌면 hash가 바뀐다.
- [ ] ordinal에 gap이 없다.
- [ ] 중복 scope가 조용히 제거되지 않는다.
- [ ] source kind 혼합이 검출된다.

### 구현 후 설명할 것

- partitioning이 failure domain을 줄이는 방식
- 정렬된 manifest가 completion gate에 주는 가치
- partition granularity의 요청 수 대 재시도 비용 trade-off
- plan과 execution을 분리한 이유

### 원본 확인 위치

- Thread 09
- 커밋 `feat(history): plan ordered source partitions`
- 파일 `grocery/historical_collection_plans.py`
- 구성 요소 `plan_historical_collection`
- 연관 Thread 03, 10, 20

## P15. [Thread 09 / `feat(history): persist regional collection parts`] 계획된 part의 provenance 검증과 idempotent 저장

**우선순위:** A

### 면접 질문

- collection part를 저장할 때 parse run만 검증하면 부족한 이유는 무엇인가요?
- planned scope와 실제 fetch attempt의 request scope가 같아야 하는 이유는 무엇인가요?
- 동일 ordinal replay와 충돌 replay를 어떻게 구분하나요?
- 꼬리 질문: part와 typed facts를 같은 트랜잭션에 넣는 장단점은 무엇인가요?

### 30초 모범 답변

part는 계획, fetch, artifact, parse, typed fact를 잇는 provenance 경계입니다. collection의 kind/source/code manifest와 planned scope가 실제 attempt의 request scope, validated parse, fact 범위와 모두 맞아야 합니다. 같은 ordinal과 scope의 완전 동일 replay만 허용하고 parse hash나 fact count가 다르면 충돌로 실패하며, part와 fact 저장은 원자적으로 묶어 빈 part나 고아 fact를 막습니다.

### 답변 핵심 키워드

`provenance chain`, `planned scope`, `validated parse`, `idempotent part`, `atomic facts`, `conflict replay`

### 백지 구현

**구현 목표**

계획된 collection part와 typed fact 묶음을 검증해 원자적으로 저장하는 축소 함수를 구현한다.

**인터페이스 또는 함수 시그니처**

```python
def persist_part(
    collection: CollectionState,
    candidate: PartCandidate,
    store: PartStore,
) -> StoredPart:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: STARTED collection, ordinal/scope/parse/facts를 가진 후보, 저장소
- 출력: 새 또는 동일 replay로 확인된 stored part

**반드시 만족해야 할 조건**

- ordinal과 scope는 collection plan에 정확히 존재해야 한다.
- fetch request scope와 planned scope가 일치해야 한다.
- parse run은 validated이며 같은 artifact/source에서 나와야 한다.
- fact kind·기간·region/market 관계가 collection kind와 맞아야 한다.
- fact count와 accepted row count가 계약상 일치해야 한다.
- part와 facts는 전부 저장되거나 전부 저장되지 않아야 한다.
- 완전 동일 replay만 허용한다.

**경계 조건**

- 첫 part와 마지막 part
- fact count 1
- 동일 replay
- market fact의 region-market 관계 경계

**실패 조건**

- 계획에 없는 scope/ordinal
- validated가 아닌 parse
- 다른 source configuration provenance
- 동일 ordinal의 다른 result hash/count
- facts 일부 저장 후 실패

**제약**

- collection이 VALIDATED 이후에는 part를 추가하지 않는다.
- 기존 part를 update하지 않는다.
- 20~25분 구현 크기다.

### 구현 후 자가 검증

- [ ] 정상 part와 facts가 함께 저장된다.
- [ ] 동일 replay가 중복 facts를 만들지 않는다.
- [ ] scope drift가 검출된다.
- [ ] 중간 실패 후 part/fact 수가 모두 원래대로다.
- [ ] collection kind와 fact kind 불일치가 통과하지 않는다.

### 구현 후 설명할 것

- provenance chain의 각 식별자를 대조하는 이유
- part/fact 원자성의 필요성
- replay 충돌을 update로 해결하지 않는 이유
- 대량 fact transaction 크기와 atomicity trade-off

### 원본 확인 위치

- Thread 09
- 커밋 `feat(history): persist regional collection parts`
- 커밋 `feat(history): persist market collection parts`
- 파일 `grocery/historical_daily_generation.py`
- 구성 요소 `persist_regional_part`, `persist_market_part`, `_validate_regional_scope`, `resolve_historical_market`

## P16. [Thread 10 / `feat(history): reconcile collection parts`] collection 완료의 전역 대사와 결정적 결과 hash

**우선순위:** S

### 면접 질문

- 각 part가 개별적으로 성공했다고 해서 collection 전체를 VALIDATED로 바꿀 수 없는 이유는 무엇인가요?
- 완료 시 expected part count, contiguous ordinal, manifest, parse count, typed fact count를 모두 대사하는 이유는 무엇인가요?
- collection result hash는 어떤 순서와 데이터 계약을 기반으로 해야 하나요?
- 꼬리 질문: completion 중 새 part insert가 들어오는 race는 어떻게 막습니까?

### 30초 모범 답변

collection 완료는 전역 invariant를 확인하는 단일 gate입니다. 계획된 part가 정확히 모두 있고 ordinal과 manifest가 일치하며, 각 part의 validated parse accepted count와 실제 typed fact 수·provenance가 맞을 때만 상태를 바꿉니다. facts를 canonical key로 정렬해 contract version과 함께 result hash를 계산하고, collection row와 part 쓰기 경계를 잠가 completion과 추가 insert가 직렬화되도록 합니다.

### 답변 핵심 키워드

`global reconciliation`, `manifest equality`, `count conservation`, `canonical fact hash`, `atomic state transition`, `serialization`

### 백지 구현

**구현 목표**

STARTED collection과 parts/facts를 받아 모든 전역 invariant를 검증하고 VALIDATED 결과를 만든다.

**인터페이스 또는 함수 시그니처**

```python
def complete_collection(
    collection: CollectionState,
    parts: Sequence[CollectionPart],
    facts: Sequence[TypedFact],
) -> ValidatedCollection:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: 계획 정보가 있는 STARTED collection, persisted parts, typed facts
- 출력: accepted count와 result hash를 가진 immutable VALIDATED collection

**반드시 만족해야 할 조건**

- part 수가 expected count와 정확히 같다.
- ordinal이 1..N으로 연속이고 scope 순서가 planned manifest와 같다.
- 모든 part의 parse가 VALIDATED이며 같은 source/code manifest에 묶인다.
- part fact_count 합, parse accepted count 합, 실제 fact 수가 일치한다.
- 각 fact가 정확히 하나의 part와 collection에 속하고 kind/window invariant를 만족한다.
- canonical fact ordering으로 deterministic result hash를 계산한다.
- 상태 전이와 완료 필드 저장은 원자적이다.

**경계 조건**

- part가 하나뿐인 collection
- 입력 parts/facts 순서가 무작위인 경우
- 완료 replay
- 큰 count 합계의 정수 경계

**실패 조건**

- part 누락·중복·ordinal gap
- manifest mismatch
- count 하나라도 불일치
- 다른 kind/window의 fact 혼입
- 이미 다른 결과로 완료된 collection

**제약**

- 누락 part를 추정하거나 빈 part로 보충하지 않는다.
- 완료 후 parts/facts를 수정하지 않는다.
- 30분 이내 핵심 reconciliation 로직을 구현한다.

### 구현 후 자가 검증

- [ ] 정상 unordered 입력이 동일한 result hash를 만든다.
- [ ] part 하나 누락 시 상태가 STARTED로 남는다.
- [ ] fact count drift가 검출된다.
- [ ] 동일 completion replay가 결과를 바꾸지 않는다.
- [ ] 다른 hash replay가 기존 완료를 덮지 않는다.
- [ ] 실패 시 completed_at/result_hash 일부만 남지 않는다.

### 구현 후 설명할 것

- part-level 성공과 collection-level 완전성의 차이
- count conservation을 여러 층에서 확인하는 이유
- canonical result hash가 review/publication에 연결되는 방식
- 큰 collection 완료 시 잠금 범위와 처리 시간 trade-off

### 원본 확인 위치

- Thread 10
- 커밋 `feat(history): reconcile collection parts`
- 파일 `grocery/historical_collections.py`
- 구성 요소 `partition_manifest_sha256`, `complete_historical_collection`
- 연관 Thread 09, 11, 22

## P17. [Thread 10 / collection write 직렬화 보강 커밋(제목은 현재 기록에서 확인되지 않음)] DB append-only guard와 completion-versus-insert 동시성

**우선순위:** S

### 면접 질문

- ORM `save()` override만으로 append-only를 보장할 수 없는 우회 경로에는 무엇이 있나요?
- part/fact insert와 collection completion이 동시에 실행될 때 어떤 잘못된 최종 상태가 가능한가요?
- 부모 row의 lock mode를 이용해 insert와 completion을 직렬화하는 원리는 무엇인가요?
- 꼬리 질문: trigger 안의 공유 잠금과 애플리케이션의 `SELECT FOR UPDATE`가 어떻게 상호작용합니까?

### 30초 모범 답변

bulk update, raw SQL, 다른 서비스 계정은 모델 메서드를 우회하므로 append-only와 membership invariant를 DB trigger로도 강제해야 합니다. fact insert는 STARTED 부모와 kind/window/part 관계를 확인하며 부모와 잠금 관계를 만들고, completion은 부모를 배타적으로 잠가 기존 insert가 끝난 뒤 검증하거나 완료 후 새 insert를 실패시킵니다. 그래서 VALIDATED인데 검증 뒤에 fact가 추가되는 상태를 막습니다.

### 답변 핵심 키워드

`DB trigger`, `append-only`, `row lock`, `shared lock`, `SELECT FOR UPDATE`, `serialization`, `TOCTOU`

### 백지 구현

**구현 목표**

제공된 repository 추상화 위에서 part insert와 collection finalize가 동시에 실행되어도 “VALIDATED 이후 insert 없음” invariant를 만족하는 잠금 프로토콜을 구현한다.

**인터페이스 또는 함수 시그니처**

```python
def finalize_collection(
    collection_id: UUID,
    repository: CollectionRepository,
) -> FinalizedCollection:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: collection id와 명시적 lock/조회/상태전이 메서드를 가진 repository
- 출력: 직렬화된 완료 결과

**반드시 만족해야 할 조건**

- finalize는 부모 collection을 잠근 상태에서 parts/facts를 읽고 검증한다.
- insert 경로는 부모가 STARTED인지 확인하는 잠금 관계를 가져야 한다.
- 둘 중 어느 순서로 실행해도 최종 상태는 insert 포함 후 완료 또는 완료 후 insert 실패 중 하나다.
- deadlock을 피하도록 모든 경로의 lock 순서를 고정한다.
- update/delete는 DB 수준에서 거부된다고 가정하고 서비스도 시도하지 않는다.

**경계 조건**

- insert가 먼저 잠금을 가진 경우
- finalize가 먼저 잠금을 가진 경우
- 동시에 여러 finalize가 들어오는 경우
- 검증 실패 후 잠금 해제

**실패 조건**

- 검증 읽기와 상태 update 사이에 insert가 끼어드는 경우
- 경로마다 lock 순서가 달라 deadlock이 가능한 경우
- VALIDATED collection에 insert가 성공하는 경우
- trigger 우회 경로가 열려 있는 경우

**제약**

- 실제 SQL 정답을 제공하지 않고 repository primitive를 사용한다.
- 테스트에서는 barrier/event로 두 thread의 interleaving을 제어한다.
- 25~30분 구현 및 동시성 테스트 설계를 목표로 한다.

### 구현 후 자가 검증

- [ ] insert-before-finalize interleaving에서 insert가 결과에 포함된다.
- [ ] finalize-before-insert interleaving에서 insert가 실패한다.
- [ ] 두 finalize가 서로 다른 결과를 기록하지 않는다.
- [ ] 예외 경로에서 lock/transaction이 정리된다.
- [ ] DB trigger를 우회한 raw mutation 테스트가 거부된다.
- [ ] lock 순서가 문서화되어 있다.

### 구현 후 설명할 것

- 애플리케이션 검증과 DB trigger의 신뢰 경계 차이
- lock mode 선택이 동시성에 미치는 영향
- serializable isolation 대신 명시적 row lock을 택할 때의 trade-off
- 동시성 테스트를 결정적으로 만드는 방법

### 원본 확인 위치

- Thread 10
- 파일 `grocery/migrations/0022_guard_historical_collection_membership.py`
- DB 함수 `grocery_guard_historical_collection_part`, `grocery_guard_historical_monthly_fact`, `grocery_guard_historical_regional_fact`, `grocery_guard_historical_market_fact`
- 구성 요소 `complete_historical_collection`
- 연관 위치: Thread 10의 collection write 직렬화 보강 migration과 concurrency test
