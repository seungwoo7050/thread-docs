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

  **모범답변:** 이 프로젝트에서는 collection을 1~100개의 고유 scope로 제한하므로, 작은 partition은 실패한 scope만 다시 처리하기 쉽지만 fetch·parse·part 감사 레코드와 요청 수가 늘어납니다. 큰 partition은 요청 오버헤드는 줄지만 한 번의 재시도 비용, 응답 메모리, 파싱 시간과 실패 영향 범위가 커집니다. 일반적으로 provider 제한과 트랜잭션 크기를 넘지 않는 범위에서 재시도 단위가 충분히 작도록 정합니다.

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
    if not 1 <= len(scopes) <= 100:
        raise ValueError("partition count must be between 1 and 100")

    source_contracts = {
        (scope.source_configuration_id, scope.dataset, scope.mode, scope.collection_kind)
        for scope in scopes
    }
    if len(source_contracts) != 1:
        raise ValueError("all partitions must use one source contract")

    windows = {(scope.window_start, scope.window_end) for scope in scopes}
    if any(start > end for start, end in windows) or len(windows) != 1:
        # 원본 계획처럼 한 collection의 모든 partition은 같은 기간을 공유한다.
        raise ValueError("partitions must share one valid collection window")

    keys = [scope.canonical_key for scope in scopes]
    hashes = [scope.scope_sha256 for scope in scopes]
    if len(set(keys)) != len(keys) or len(set(hashes)) != len(hashes):
        raise ValueError("duplicate partition scope")

    ordered = tuple(sorted(scopes, key=lambda scope: scope.canonical_key))
    parts = tuple((ordinal, scope) for ordinal, scope in enumerate(ordered, start=1))
    # 원본 계약은 ordinal 순서 자체를 위치로 삼아 scope hash 문자열 목록만 해시한다.
    manifest_payload = [scope.scope_sha256 for scope in ordered]
    manifest_sha256 = sha256(
        json.dumps(
            manifest_payload,
            separators=(",", ":"),
        ).encode("ascii")
    ).hexdigest()

    first = ordered[0]
    return CollectionPlan(
        source_configuration_id=first.source_configuration_id,
        collection_kind=first.collection_kind,
        window_start=first.window_start,
        window_end=first.window_end,
        parts=parts,
        partition_manifest_sha256=manifest_sha256,
    )
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

  **모범답변:** 프로젝트는 fetch·parse·typed fact 저장을 ordinal별 part로 나누고, 이미 저장된 part의 동일 replay를 기존 결과로 수렴시킵니다. 현재 orchestration이 계획을 다시 순회하더라도 성공 part를 덮어쓰지 않으며 실패 지점과 감사 증거가 scope별로 분리됩니다. 일반적으로 scheduler가 part 단위 재시도를 지원하면 실제 재요청 범위도 같은 경계로 줄일 수 있습니다.

- 정렬된 manifest가 completion gate에 주는 가치

  **모범답변:** 프로젝트의 `partition_manifest_sha256`은 ordinal 순서의 scope hash 문자열 목록을 해시하고, completion도 part를 ordinal 순으로 읽어 같은 목록을 재계산합니다. ordinal은 목록 위치에 암묵적으로 고정되므로 part 수가 같아도 누락·교체·순서 변경을 탐지합니다. 일반화된 계약에서 version이나 ordinal을 payload에 추가할 수 있지만, 그것은 기존 hash와 호환되지 않는 명시적 계약 변경입니다.

- partition granularity의 요청 수 대 재시도 비용 trade-off

  **모범답변:** partition이 작으면 요청·artifact·parse run·DB row 수와 외부 API overhead가 증가하지만 실패 시 다시 처리할 데이터가 적습니다. 크게 잡으면 정상 경로의 overhead는 줄지만 timeout, 메모리 사용량, 재시도 비용과 단일 실패의 영향이 커집니다.

- plan과 execution을 분리한 이유

  **모범답변:** 프로젝트는 fetch 전에 collection id, source, 공통 window, part 수와 ordered manifest를 먼저 고정합니다. 실행이 중단되거나 재시도되어도 “무엇을 수집하기로 했는지”가 바뀌지 않아, 새 scope를 기존 collection에 섞지 않고 계획 replay와 범위 변경을 구분할 수 있습니다.

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

  **모범답변:** 프로젝트의 `persist_*_part`와 `start_historical_part`는 collection·parse run·part·typed facts를 한 `transaction.atomic` 경계에서 기록하므로 빈 part나 고아 fact가 남지 않습니다. 일반적으로 이 방식은 원자성을 높이는 대신 fact가 많을수록 잠금 보유 시간, WAL과 rollback 비용이 커지므로 partition 상한으로 트랜잭션 크기를 제한해야 합니다.

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
    planned = {
        ordinal: scope for ordinal, scope in collection.plan.parts
    }.get(candidate.ordinal)
    if planned is None or planned.scope_sha256 != candidate.partition_scope_sha256:
        raise ValueError("part is outside the collection plan")

    attempt = candidate.fetch_attempt
    parse = candidate.parse_run
    if (
        attempt.state != "SUCCEEDED"
        or attempt.source_configuration_id != collection.source_configuration_id
        or attempt.request_scope_sha256 != planned.scope_sha256
        or parse.status != "VALIDATED"
        or not parse.result_sha256
        or parse.artifact_id != attempt.artifact_id
    ):
        raise ValueError("part provenance does not match the planned fetch")
    if (
        candidate.collection_kind != collection.kind
        or candidate.code_manifest_sha256 != collection.code_manifest_sha256
        or parse.total_row_count != parse.accepted_row_count
        or parse.accepted_row_count != len(candidate.facts)
        or candidate.fact_count != len(candidate.facts)
    ):
        raise ValueError("part metadata does not reconcile")

    for fact in candidate.facts:
        if fact.kind != collection.kind or not planned.matches_fact(fact):
            raise ValueError("fact does not match the planned semantic scope")
        if collection.kind == "MONTHLY":
            in_window = collection.month_min <= fact.year_month <= collection.month_max
        else:
            in_window = collection.date_min <= fact.survey_date <= collection.date_max
        if not in_window:
            raise ValueError("fact is outside the collection window")
        if fact.kind == "MARKET_DAILY" and fact.market.region_id != fact.region_id:
            raise ValueError("market does not belong to the fact region")

    def canonical_fact(fact: TypedFact) -> tuple[object, ...]:
        common = (
            fact.series.series_identity_sha256,
            fact.region.region_code,
        )
        if fact.kind == "MONTHLY":
            key = (*common, fact.year_month)
        elif fact.kind == "REGIONAL_DAILY":
            key = (*common, fact.survey_date.isoformat())
        elif fact.kind == "MARKET_DAILY":
            key = (*common, fact.market.market_code, fact.survey_date.isoformat())
        else:
            raise ValueError("unknown typed fact kind")

        if fact.kind in {"MONTHLY", "REGIONAL_DAILY"}:
            values = (
                format(fact.provider_mean, "f"),
                format(fact.provider_low, "f"),
                format(fact.provider_high, "f"),
            )
        else:
            values = (format(fact.provider_price, "f"),)
        return (*key, *values, fact.source_row_sha256)

    def replay_fingerprint(part: StoredPart, facts: Sequence[TypedFact]) -> tuple[object, ...]:
        return (
            part.collection_id,
            part.ordinal,
            part.partition_scope_sha256,
            part.parse_run_id,
            part.parse_run.result_sha256,
            part.fact_count,
            tuple(sorted(canonical_fact(fact) for fact in facts)),
        )

    candidate_fingerprint = (
        collection.id,
        candidate.ordinal,
        candidate.partition_scope_sha256,
        candidate.parse_run.id,
        candidate.parse_run.result_sha256,
        candidate.fact_count,
        tuple(sorted(canonical_fact(fact) for fact in candidate.facts)),
    )

    with store.atomic():
        locked = store.lock_collection(collection.id)
        by_ordinal = store.part_by_ordinal(locked.id, candidate.ordinal)
        by_parse = store.part_by_parse_run(candidate.parse_run.id)
        if by_ordinal is not None and by_parse is not None and by_ordinal.id != by_parse.id:
            raise ValueError("ordinal and parse run identify different stored parts")
        existing = by_ordinal or by_parse
        if existing is not None:
            stored_facts = store.facts_for_part(existing.id)
            if replay_fingerprint(existing, stored_facts) != candidate_fingerprint:
                raise ValueError("part replay conflicts with stored evidence")
            return existing

        # 원본도 replay 확인 뒤에만 STARTED를 요구한다. 완료 후 동일 replay는 읽기만 한다.
        if locked.state != "STARTED":
            raise ValueError("new parts require a started collection")
        part = store.insert_part(locked.id, candidate)
        store.insert_facts(part.id, candidate.facts)
        return part
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

  **모범답변:** collection의 source configuration·kind·code manifest, planned ordinal/scope, 성공한 fetch의 artifact/request scope, validated parse의 result hash를 모두 대조해야 다른 실행의 정상 산출물이 잘못 연결되는 것을 막을 수 있습니다. 프로젝트는 이 연결을 서비스와 DB trigger 양쪽에서 확인합니다.

- part/fact 원자성의 필요성

  **모범답변:** part만 있고 facts가 없거나 facts 일부만 남으면 completion의 count 대사가 실패하고 replay도 애매해집니다. `transaction.atomic` 안에서 parse 완료, part insert와 fact insert를 묶으면 어느 fact 생성에서 실패해도 전체 묶음이 롤백됩니다.

- replay 충돌을 update로 해결하지 않는 이유

  **모범답변:** 같은 ordinal 또는 parse identity에 다른 hash·scope·count가 나타난 것은 단순 최신값이 아니라 비결정성이나 잘못된 provenance의 증거입니다. update로 덮으면 최초 감사 증거를 잃으므로, 원본은 완전 동일 replay만 반환하고 다른 내용은 실패시킵니다.

- 대량 fact transaction 크기와 atomicity trade-off

  **모범답변:** 큰 트랜잭션은 묶음 단위 원자성은 강하지만 lock 시간, 메모리, WAL, 실패 시 rollback 비용이 증가합니다. 이 프로젝트의 bounded partition을 원자 단위로 유지하고 collection 전체는 별도 completion에서 대사하는 방식이 그 균형점입니다.

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

  **모범답변:** 프로젝트에서는 completion이 부모 collection을 `SELECT ... FOR UPDATE`로 잠그고, part/fact insert trigger는 같은 부모를 `FOR SHARE`로 잠근 뒤 STARTED를 확인합니다. 따라서 insert가 먼저면 completion이 기다린 뒤 그 row까지 대사하고, completion이 먼저면 insert가 기다렸다가 VALIDATED 상태를 보고 실패합니다. 일반적으로 검증 읽기와 상태 전이를 같은 트랜잭션·잠금 순서에 넣어 TOCTOU를 제거합니다.

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
    ordered_parts = tuple(sorted(parts, key=lambda part: part.ordinal))
    expected_ordinals = tuple(range(1, collection.expected_part_count + 1))
    if tuple(part.ordinal for part in ordered_parts) != expected_ordinals:
        raise ValueError("collection parts are incomplete or non-contiguous")

    manifest_payload = [part.partition_scope_sha256 for part in ordered_parts]
    manifest_sha256 = sha256(
        json.dumps(
            manifest_payload,
            separators=(",", ":"),
        ).encode("ascii")
    ).hexdigest()
    if manifest_sha256 != collection.partition_manifest_sha256:
        raise ValueError("partition manifest does not match the plan")

    part_ids = {part.id for part in ordered_parts}
    if len(part_ids) != len(ordered_parts):
        raise ValueError("duplicate collection part")
    for part in ordered_parts:
        if part.collection_id != collection.id:
            raise ValueError("part does not belong to this collection")
        parse = part.parse_run
        attempt = part.fetch_attempt
        if (
            parse.status != "VALIDATED"
            or not parse.result_sha256
            or attempt.state != "SUCCEEDED"
            or attempt.source_configuration_id != collection.source_configuration_id
            or attempt.artifact_id != parse.artifact_id
            or attempt.request_scope_sha256 != part.partition_scope_sha256
            or part.code_manifest_sha256 != collection.code_manifest_sha256
        ):
            raise ValueError("part parse provenance does not match the collection")

    facts_by_part = {part.id: [] for part in ordered_parts}
    for fact in facts:
        if (
            fact.collection_id != collection.id
            or fact.collection_part_id not in part_ids
            or fact.kind != collection.kind
        ):
            raise ValueError("fact does not belong to this collection and part")
        if collection.kind == "MONTHLY":
            in_window = collection.month_min <= fact.year_month <= collection.month_max
        else:
            in_window = collection.date_min <= fact.survey_date <= collection.date_max
        if not in_window:
            raise ValueError("fact is outside the collection window")
        facts_by_part[fact.collection_part_id].append(fact)

    for part in ordered_parts:
        actual = len(facts_by_part[part.id])
        if actual != part.fact_count or actual != part.parse_run.accepted_row_count:
            raise ValueError("part, parse and typed fact counts do not reconcile")
    accepted = sum(part.fact_count for part in ordered_parts)
    if accepted != len(facts):
        raise ValueError("collection fact count does not reconcile")

    # 각 parse result는 parser가 semantic key로 정렬한 canonical row payload의 hash다.
    # 원본 completion은 그 hash들을 ordinal 순으로 다시 묶어 collection 결과를 만든다.
    result_payload = {
        "kind": collection.kind,
        "parts": [
            {
                "ordinal": part.ordinal,
                "partition_scope_sha256": part.partition_scope_sha256,
                "parse_result_sha256": part.parse_run.result_sha256,
                "fact_count": part.fact_count,
            }
            for part in ordered_parts
        ],
    }
    result_sha256 = sha256(
        json.dumps(
            result_payload,
            ensure_ascii=True,
            sort_keys=True,
            separators=(",", ":"),
        ).encode("ascii")
    ).hexdigest()

    if collection.state == "VALIDATED":
        if (
            collection.accepted_row_count != accepted
            or collection.result_sha256 != result_sha256
        ):
            raise ValueError("completion replay conflicts with stored result")
        return ValidatedCollection.from_state(collection)
    if collection.state != "STARTED":
        raise ValueError("only a started collection can be completed")

    # 실제 저장 계층은 이 결과의 상태와 완료 필드를 한 트랜잭션에서 기록한다.
    return ValidatedCollection(
        source=collection,
        state="VALIDATED",
        accepted_row_count=accepted,
        out_of_scope_row_count=sum(
            part.parse_run.out_of_scope_row_count for part in ordered_parts
        ),
        quarantined_row_count=0,
        result_sha256=result_sha256,
    )
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

  **모범답변:** part 성공은 하나의 scope가 올바른 fetch·parse·facts로 저장됐다는 뜻일 뿐, 계획된 다른 ordinal의 존재까지 보장하지 않습니다. collection completion은 모든 ordinal과 ordered manifest를 한꺼번에 확인한 뒤에만 공개 검수 가능한 VALIDATED 상태를 만듭니다.

- count conservation을 여러 층에서 확인하는 이유

  **모범답변:** `part.fact_count`, parse run의 `accepted_row_count`, 실제 typed fact 수는 서로 다른 단계가 기록한 같은 양의 증거입니다. 세 값이 같아야 parse 뒤 유실, 중복 insert, 잘못된 part 연결을 탐지할 수 있고, 다른 kind 모델에 fact가 섞이지 않았는지도 별도로 확인해야 합니다.

- canonical result hash가 review/publication에 연결되는 방식

  **모범답변:** collection review의 APPROVE는 collection result hash와 partition manifest hash를 exact evidence로 저장합니다. 이후 publication seal은 승인된 세 collection의 실제 typed facts를 다시 canonical하게 해시하므로, 수집 완료 증거와 공개 snapshot 증거가 단계별로 연결됩니다.

- 큰 collection 완료 시 잠금 범위와 처리 시간 trade-off

  **모범답변:** 부모와 parts를 잠근 채 count 조회와 hash 계산을 하면 강한 일관성을 얻지만, collection이 클수록 insert 대기와 트랜잭션 시간이 늘어납니다. 그래서 원본은 collection당 part 수를 제한하고, 장기적으로는 잠금 안에서 검증할 요약 증거를 미리 축적하되 동일한 대사 강도를 유지하는 방식이 필요합니다.

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

  **모범답변:** 프로젝트의 insert trigger는 부모 collection을 `FOR SHARE`로 잡아 애플리케이션 completion의 `SELECT FOR UPDATE`와 충돌시킵니다. insert가 공유 잠금을 먼저 얻으면 completion이 commit까지 기다리고, completion이 배타 잠금을 먼저 얻으면 insert가 기다린 뒤 갱신된 VALIDATED를 읽어 거부됩니다. 일반적으로 경쟁하는 쓰기 경로가 같은 부모를 호환되지 않는 lock mode와 동일한 순서로 잡게 하면 결과를 한 실행 순서로 직렬화할 수 있습니다.

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
    with repository.atomic():
        # 모든 쓰기 경로의 첫 잠금은 부모 collection이다: parent -> parts -> facts.
        collection = repository.lock_collection_for_update(collection_id)
        if collection.state == "VALIDATED":
            return FinalizedCollection.from_state(collection)
        if collection.state != "STARTED":
            raise ValueError("only a started collection can be finalized")

        parts = repository.load_parts(collection_id, for_update=True)
        # 부모 잠금이 새 insert를 막고 DB guard가 기존 fact의 mutation을 막으므로 읽기면 충분하다.
        facts = repository.load_facts(collection_id)
        validated = complete_collection(collection, parts, facts)

        updated = repository.mark_validated_if_started(
            collection_id=collection_id,
            accepted_row_count=validated.accepted_row_count,
            out_of_scope_row_count=validated.out_of_scope_row_count,
            quarantined_row_count=validated.quarantined_row_count,
            result_sha256=validated.result_sha256,
        )
        if updated != 1:
            raise RuntimeError("collection finalization did not advance exactly once")
        return repository.load_finalized(collection_id)
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

  **모범답변:** 서비스 검증은 이해하기 쉬운 오류와 정상 ORM 경로의 계약을 담당하지만 `bulk_update`, raw SQL, 다른 프로세스는 모델 `save()`를 우회할 수 있습니다. 프로젝트의 trigger는 그런 경로에도 append-only, STARTED membership, kind/window 관계를 강제하는 최종 신뢰 경계입니다.

- lock mode 선택이 동시성에 미치는 영향

  **모범답변:** insert끼리는 부모의 `FOR SHARE`를 함께 보유할 수 있어 병렬성을 유지하지만, completion의 배타 잠금과는 충돌합니다. 초기 `FOR KEY SHARE`는 key를 바꾸지 않는 update와 양립할 수 있어 직렬화가 약했기 때문에, 보강 migration은 `FOR SHARE`를 사용해 상태 update와 확실히 충돌시켰습니다.

- serializable isolation 대신 명시적 row lock을 택할 때의 trade-off

  **모범답변:** 명시적 부모 row lock은 보호 대상과 lock 순서가 드러나고 collection별 동시성은 유지하며 재시도도 적습니다. 반면 모든 쓰기 경로가 같은 프로토콜을 따라야 합니다. Serializable은 더 일반적인 이상 현상을 막지만 abort·재시도 비용과 원인 분석 부담이 커질 수 있습니다.

- 동시성 테스트를 결정적으로 만드는 방법

  **모범답변:** 서로 다른 DB connection을 쓰는 두 thread를 barrier/event로 특정 잠금 획득 지점에 세운 뒤 반대 작업을 시작합니다. insert-first와 finalize-first를 각각 강제하고, lock timeout으로 실제 대기를 확인한 다음 최종 row·상태를 검증하면 단순 반복 기반 race test보다 결정적입니다.

### 원본 확인 위치

- Thread 10
- 파일 `grocery/migrations/0022_guard_historical_collection_membership.py`
- DB 함수 `grocery_guard_historical_collection_part`, `grocery_guard_historical_monthly_fact`, `grocery_guard_historical_regional_fact`, `grocery_guard_historical_market_fact`
- 구성 요소 `complete_historical_collection`
- 연관 위치: Thread 10의 collection write 직렬화 보강 migration과 concurrency test
