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

  **모범답변:** 프로젝트에서 UUID는 명령의 idempotency key입니다. 같은 UUID와 collection·actor·결정·evidence hash·승인 hash·supersedes가 모두 같으면 전송 재시도로 보고 기존 row를 반환하지만, 하나라도 다르면 같은 명령의 재시도가 아니라 identity 충돌입니다. 일반적으로 key 중복만 성공 처리하면 서로 다른 결정을 조용히 합칠 수 있으므로 semantic equality까지 확인해야 합니다.

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
    reviewer = command.reviewer
    if (
        reviewer.id is None
        or not reviewer.is_authenticated
        or not reviewer.is_active
        or not reviewer.has_perm("grocery.review_historical_collection")
    ):
        raise PermissionError("an active collection reviewer is required")

    semantic_fields = (
        collection.id,
        command.decision,
        reviewer.id,
        command.reconciliation_report_sha256,
        command.acceptance_evidence_sha256,
        command.reason_code,
        command.approved_result_sha256,
        command.approved_partition_manifest_sha256,
        command.supersedes_id,
    )
    existing = next(
        (decision for decision in decisions if decision.id == command.decision_id),
        None,
    )
    if existing is not None:
        stored_fields = (
            existing.collection_id,
            existing.decision,
            existing.reviewer_id,
            existing.reconciliation_report_sha256,
            existing.acceptance_evidence_sha256,
            existing.reason_code,
            existing.approved_result_sha256,
            existing.approved_partition_manifest_sha256,
            existing.supersedes_id,
        )
        if stored_fields != semantic_fields:
            raise ValueError("review UUID conflicts with stored evidence")
        return existing

    if collection.state != "VALIDATED" or collection.completed_at is None:
        raise ValueError("review requires a validated collection")
    expected_mode = {
        "MONTHLY": "HISTORICAL_MONTHLY",
        "REGIONAL_DAILY": "HISTORICAL_REGIONAL",
        "MARKET_DAILY": "HISTORICAL_MARKET",
    }[collection.kind]
    source = collection.source_configuration
    if source.state != "ACTIVE" or source.publication_mode != expected_mode:
        raise ValueError("collection does not use the matching active source")
    collection_decisions = tuple(
        decision for decision in decisions if decision.collection_id == collection.id
    )
    superseded = {
        decision.supersedes_id
        for decision in collection_decisions
        if decision.supersedes_id is not None
    }
    tails = tuple(
        decision for decision in collection_decisions if decision.id not in superseded
    )
    if len(tails) > 1:
        raise ValueError("review chain has more than one tail")
    expected_tail_id = tails[0].id if tails else None
    if command.supersedes_id != expected_tail_id:
        raise ValueError("a review must supersede the current collection tail")

    if command.decision == "APPROVE":
        if (
            command.approved_result_sha256 != collection.result_sha256
            or command.approved_partition_manifest_sha256
            != collection.partition_manifest_sha256
        ):
            raise ValueError("approved hashes must match the collection")
    elif command.decision == "REJECT":
        if command.approved_result_sha256 or command.approved_partition_manifest_sha256:
            raise ValueError("a rejection cannot carry approval hashes")
    else:
        raise ValueError("unknown review decision")

    # 입력 chain은 건드리지 않고, 저장 계층이 이 새 row만 append한다.
    return ReviewDecision(
        id=command.decision_id,
        collection_id=collection.id,
        decision=command.decision,
        reviewer_id=reviewer.id,
        reconciliation_report_sha256=command.reconciliation_report_sha256,
        acceptance_evidence_sha256=command.acceptance_evidence_sha256,
        reason_code=command.reason_code,
        approved_result_sha256=command.approved_result_sha256,
        approved_partition_manifest_sha256=(
            command.approved_partition_manifest_sha256
        ),
        supersedes_id=command.supersedes_id,
    )
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

  **모범답변:** 잘못된 결정을 update로 지우지 않고 새 decision이 이전 tail을 supersede하게 하면, 최초 판단과 정정 이유·actor·evidence가 모두 남습니다. 현재 효력은 tail로 계산하되 과거 감사 이력은 그대로 재구성할 수 있습니다.

- 승인을 content hash에 묶는 이유

  **모범답변:** collection id만 승인하면 같은 id 아래 내용이 달라졌을 때도 승인이 유효해 보일 수 있습니다. 프로젝트는 `approved_result_sha256`과 `approved_partition_manifest_sha256`을 collection의 현재 값과 exact match시켜, 검토자가 본 결과와 partition 구성만 승인되도록 합니다.

- tail lock과 unique constraint의 역할

  **모범답변:** 서비스는 collection과 기존 decision들을 잠가 두 경쟁자가 같은 tail을 동시에 읽지 못하게 합니다. DB의 root unique와 one-to-one supersedes 제약은 잠금 프로토콜을 우회하거나 경합이 생겨도 두 root나 두 child로 fork되는 것을 최종적으로 막습니다.

- actor 권한을 서비스와 DB에서 중복 확인하는 이유

  **모범답변:** 서비스는 active/authenticated actor와 `review_historical_collection` 권한을 확인해 사용자 의도를 검증합니다. DB trigger는 transaction-local capability와 collection/hash/tail 조건을 다시 확인해 bulk/raw SQL 우회를 막습니다. 둘은 같은 검사를 낭비해 반복하는 것이 아니라 서로 다른 신뢰 경계를 보호합니다.

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

  **모범답변:** 프로젝트는 canonical typed-fact hash와 `public_copy_revision`의 조합을 unique key로 사용하므로, 사실이 같아도 표시 문구 계약이 바뀌면 별도 publication revision으로 구분할 수 있습니다. 반대로 둘이 모두 같으면 seal replay로 기존 revision을 확인합니다. 현재 모델은 지원 copy revision을 `ko-v4`로 고정합니다. 일반적으로 표시 계약이 공개 artifact identity에 영향을 준다면 content hash와 함께 replay key에 포함하되, 지원 revision 변경은 명시적으로 versioning해야 합니다.

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
    def semantic_key(fact: TypedFact) -> tuple[object, ...]:
        common = (
            fact.series.series_identity_sha256,
            fact.region.region_code,
        )
        if fact.kind == "MONTHLY":
            return (*common, fact.year_month)
        if fact.kind == "REGIONAL_DAILY":
            return (*common, fact.survey_date.isoformat())
        if fact.kind == "MARKET_DAILY":
            return (*common, fact.market.market_code, fact.survey_date.isoformat())
        raise ValueError("unknown typed fact kind")

    def canonical_payload(fact: TypedFact) -> tuple[object, ...]:
        key = semantic_key(fact)
        if fact.kind in {"MONTHLY", "REGIONAL_DAILY"}:
            return (
                *key,
                format(fact.provider_mean, "f"),
                format(fact.provider_low, "f"),
                format(fact.provider_high, "f"),
                fact.source_row_sha256,
            )
        return (*key, format(fact.provider_price, "f"), fact.source_row_sha256)

    selected = (
        (reviews.monthly, "MONTHLY"),
        (reviews.regional, "REGIONAL_DAILY"),
        (reviews.market, "MARKET_DAILY"),
    )
    if len({review.id for review, _kind in selected}) != 3:
        raise ValueError("three distinct reviews are required")

    collections = []
    for review, expected_kind in selected:
        collection = review.collection
        if (
            not review.is_current_tail
            or review.decision != "APPROVE"
            or collection.kind != expected_kind
            or collection.state != "VALIDATED"
            or review.approved_result_sha256 != collection.result_sha256
            or review.approved_partition_manifest_sha256
            != collection.partition_manifest_sha256
        ):
            raise ValueError("publication requires a current exact approval")
        collections.append(collection)

    manifests = {collection.code_manifest_sha256 for collection in collections}
    if len(manifests) != 1:
        raise ValueError("reviewed collections use different code manifests")
    code_manifest_sha256 = next(iter(manifests))
    if any(
        collection.date_min is not None
        and collection.date_max is not None
        and (collection.date_max - collection.date_min).days > 30
        for collection in collections[1:]
    ):
        raise ValueError("daily collections must stay within 31 calendar days")
    expected_collection_by_kind = {
        collection.kind: collection.id for collection in collections
    }

    facts_by_kind = {kind: [] for _review, kind in selected}
    semantic_keys = set()
    for fact in facts:
        expected_collection_id = expected_collection_by_kind.get(fact.kind)
        if expected_collection_id is None or fact.collection_id != expected_collection_id:
            raise ValueError("fact is outside the reviewed collections")
        key = (fact.kind, semantic_key(fact))
        if key in semantic_keys:
            raise ValueError("duplicate typed fact semantic key")
        semantic_keys.add(key)
        facts_by_kind[fact.kind].append(fact)

    expected_counts = {
        collection.kind: collection.accepted_row_count for collection in collections
    }
    if {
        kind: len(kind_facts) for kind, kind_facts in facts_by_kind.items()
    } != expected_counts:
        raise ValueError("fact counts do not match reconciled collections")
    if any(fact.series.code_manifest_sha256 != code_manifest_sha256 for fact in facts):
        raise ValueError("fact series identity uses a different code manifest")

    monthly = facts_by_kind["MONTHLY"]
    regional = facts_by_kind["REGIONAL_DAILY"]
    markets = facts_by_kind["MARKET_DAILY"]
    series_sets = tuple({fact.series_id for fact in rows} for rows in (monthly, regional, markets))
    if not series_sets[0] or not (series_sets[0] == series_sets[1] == series_sets[2]):
        raise ValueError("source series sets must match exactly")

    month_max = collections[0].month_max
    last_month = int(month_max[:4]) * 12 + int(month_max[4:]) - 1
    required_months = {
        f"{value // 12:04d}{value % 12 + 1:02d}"
        for value in range(last_month - 35, last_month + 1)
    }
    monthly_coverage = {}
    for fact in monthly:
        monthly_coverage.setdefault((fact.series_id, fact.region_id), set()).add(
            fact.year_month
        )
    if any(
        not any(
            series_id == candidate_series and required_months <= months
            for (candidate_series, _region_id), months in monthly_coverage.items()
        )
        for series_id in series_sets[0]
    ):
        raise ValueError("each series needs one complete recent 36-month region")

    regional_keys = {
        (fact.series_id, fact.region_id, fact.survey_date) for fact in regional
    }
    market_keys = {
        (fact.series_id, fact.region_id, fact.survey_date) for fact in markets
    }
    shared = regional_keys & market_keys
    if any(not any(key[0] == series_id for key in shared) for series_id in series_sets[0]):
        raise ValueError("each series needs a shared regional and market date")

    canonical_facts = {
        "monthly": sorted(canonical_payload(fact) for fact in monthly),
        "regional": sorted(canonical_payload(fact) for fact in regional),
        "markets": sorted(canonical_payload(fact) for fact in markets),
    }
    typed_fact_set_sha256 = sha256(
        json.dumps(
            canonical_facts,
            ensure_ascii=True,
            sort_keys=True,
            separators=(",", ":"),
        ).encode("ascii")
    ).hexdigest()
    shared_dates = {key[2] for key in shared}

    fields = {
        "monthly_review_id": reviews.monthly.id,
        "regional_review_id": reviews.regional.id,
        "market_review_id": reviews.market.id,
        "code_manifest_sha256": code_manifest_sha256,
        "fact_hash_version": "historical-retail-bundle-v1",
        "typed_fact_set_sha256": typed_fact_set_sha256,
        "series_count": len(series_sets[0]),
        "monthly_fact_count": len(monthly),
        "regional_fact_count": len(regional),
        "market_fact_count": len(markets),
        "month_min": collections[0].month_min,
        "month_max": month_max,
        "date_min": min(shared_dates),
        "date_max": max(shared_dates),
        "public_copy_revision": copy_revision,
    }
    # 원본처럼 unique identity만 같다고 replay로 보지 않고 모든 봉인 evidence를 대조한다.
    existing = (
        SealedRevision.objects.select_for_update()
        .filter(
            typed_fact_set_sha256=typed_fact_set_sha256,
            public_copy_revision=copy_revision,
        )
        .first()
    )
    if existing is not None:
        if not existing.sealed or any(
            getattr(existing, field_name) != value
            for field_name, value in fields.items()
        ):
            raise ValueError("publication replay conflicts with stored evidence")
        return existing
    return SealedRevision(**fields, sealed=True)
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

  **모범답변:** review는 개별 source collection의 수집 결과를 사람이 수용했다는 기록이고, seal은 월별·지역별·시장별 승인 세 개를 공개 가능한 하나의 불변 bundle로 조립하는 단계입니다. 분리해야 source별 재검수와 공개 snapshot 생성·activation을 독립적으로 감사할 수 있습니다.

- canonical content hash와 row-id hash의 차이

  **모범답변:** row id는 삽입 환경과 순서에 따라 달라져 같은 사실 집합을 재현하지 못합니다. 원본은 series identity, region/market code, 기간, 정규화한 Decimal과 source row hash를 source별로 정렬해 해시하므로 DB 생성 순서와 무관한 내용 동일성을 얻습니다.

- copy revision을 사실 hash와 분리 또는 결합하는 기준

  **모범답변:** 가격 사실의 동일성만 비교할 때는 copy revision을 typed-fact hash에서 분리하는 편이 재사용에 유리합니다. 다만 공개 artifact의 identity와 replay 판단에는 문구 계약도 영향을 주므로 프로젝트는 `(fact-set hash, copy revision)`을 함께 unique key로 둡니다.

- seal 시 전체 fact scan 비용과 강한 무결성의 trade-off

  **모범답변:** seal은 세 collection의 전체 facts를 잠그고 count·series coverage·기간·canonical hash를 재계산하므로 비용이 들지만, 검수 뒤 누락이나 잘못된 membership을 공개 전에 마지막으로 검출합니다. 규모가 커지면 사전 계산한 immutable chunk hash를 합성할 수 있지만 동일한 completeness 증명이 유지돼야 합니다.

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

  **모범답변:** 프로젝트는 channel row lock으로 같은 channel의 writer를 직렬화하고, expected version/current CAS로 명령을 만든 시점의 운영 의도까지 검증합니다. row lock만 있으면 늦게 도착한 stale 명령도 현재 상태 위에서 실행될 수 있고, CAS만 있으면 읽기·event insert·pointer update 사이 경합 처리가 복잡합니다. 일반적으로 lock은 동시 실행을, CAS precondition은 오래된 의도를 막는 서로 다른 책임을 가집니다.

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
    if type(command.operation_id) is not UUID:
        raise ValueError("operation id must be a UUID")
    if type(command.expected_version) is not int or command.expected_version < 0:
        raise ValueError("expected version must be a non-negative integer")

    semantic_fields = (
        channel.channel_id,
        command.operation,
        command.expected_version + 1,
        command.expected_current_revision_id,
        command.target_revision_id,
        command.publisher_id,
        command.reason_code,
        command.acceptance_evidence_sha256,
    )
    existing = next(
        (event for event in history if event.id == command.operation_id),
        None,
    )
    if existing is not None:
        stored_fields = (
            existing.channel_id,
            existing.operation,
            existing.sequence,
            existing.previous_revision_id,
            existing.target_revision_id,
            existing.publisher_id,
            existing.reason_code,
            existing.acceptance_evidence_sha256,
        )
        if stored_fields != semantic_fields:
            raise ValueError("operation UUID conflicts with stored evidence")
        return TransitionResult(channel=channel, event=existing, replayed=True)

    if (
        channel.version != command.expected_version
        or channel.current_revision_id != command.expected_current_revision_id
    ):
        raise ValueError("publication expectation is stale")
    if command.operation not in {"ACTIVATE", "ROLLBACK", "WITHDRAW"}:
        raise ValueError("unknown publication operation")

    target = command.target_revision
    if command.operation == "WITHDRAW":
        if (
            target is not None
            or command.target_revision_id is not None
            or channel.current_revision_id is None
        ):
            raise ValueError("withdraw requires a current revision and no target")
        resulting_revision_id = None
    else:
        if target is None or target.id == channel.current_revision_id:
            raise ValueError("transition requires a different target")
        if command.target_revision_id != target.id:
            raise ValueError("target id does not match the loaded revision")
        if target.channel_id != channel.channel_id:
            raise ValueError("target belongs to a different publication channel")
        eligible_history = tuple(
            event
            for event in history
            if event.channel_id == channel.channel_id
            and event.sequence <= command.expected_version
        )
        if not is_transition_target_eligible(command.operation, target, eligible_history):
            raise ValueError("publication target is not eligible")
        resulting_revision_id = target.id

    event = ActivationEvent(
        id=command.operation_id,
        channel_id=channel.channel_id,
        operation=command.operation,
        sequence=channel.version + 1,
        previous_revision_id=channel.current_revision_id,
        target_revision_id=resulting_revision_id,
        publisher_id=command.publisher_id,
        reason_code=command.reason_code,
        acceptance_evidence_sha256=command.acceptance_evidence_sha256,
    )
    next_channel = replace(
        channel,
        current_revision_id=resulting_revision_id,
        version=channel.version + 1,
    )
    # 저장 계층은 event append와 이 새 pointer를 하나의 transaction으로 반영한다.
    return TransitionResult(channel=next_channel, event=event, replayed=False)
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

  **모범답변:** version은 withdraw 뒤 다시 같은 revision이 current가 되는 ABA 같은 history 변화를 잡고, expected current는 operator가 어떤 revision을 대체한다고 이해했는지 직접 확인합니다. 둘을 함께 비교해야 같은 pointer 값으로 돌아온 stale command와 잘못 지정한 current를 모두 거부할 수 있습니다.

- event sourcing 전체가 아니라 audit event+current pointer를 선택한 trade-off

  **모범답변:** append-only event는 누가 어떤 근거로 전환했는지 남기고, current pointer는 public read가 history fold 없이 한 row로 현재 revision을 찾게 합니다. 쓰기 때 두 표현의 원자적 정합성을 강제해야 하는 비용을 지불하는 대신 읽기 경로가 단순하고 빠릅니다.

- row lock과 CAS가 중복처럼 보여도 둘 다 필요한 이유

  **모범답변:** row lock은 동시에 실행 중인 writer의 DB interleaving을 직렬화하고, CAS는 lock을 늦게 얻은 명령이 과거에 관찰한 version/current를 전제로 실행되는 것을 거부합니다. 즉 하나는 실행 경쟁, 다른 하나는 stale intent를 막습니다.

- DB trigger가 event-pointer 정합성을 보강하는 방식

  **모범답변:** trigger는 transaction-local operation capability를 확인하고 event의 sequence/previous가 현재 channel과 맞는지 검증합니다. channel update도 version이 정확히 1 증가하고 같은 operation의 event target과 새 pointer가 일치해야 허용하며, deferred constraint가 전체 history의 연속성과 최종 pointer를 다시 확인합니다.

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

  **모범답변:** 현재 모델에는 revocation이 없으므로 확장 시 revision을 삭제하지 않고 별도의 append-only revocation evidence를 추가하겠습니다. ACTIVATE와 ROLLBACK의 공통 sealed 검사 전에 revoked 여부를 서비스와 DB guard 모두에서 거부하고, 이미 current라면 별도 WITHDRAW 절차를 실행합니다. 일반적으로 “과거 current였다”는 가용성 증거가 보안 폐기 결정보다 우선해서는 안 됩니다.

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
    if operation == Operation.WITHDRAW:
        return target is None
    if operation not in {Operation.ACTIVATE, Operation.ROLLBACK}:
        return False
    if target is None or target.sealed_at is None:
        return False

    if operation == Operation.ACTIVATE:
        if target.channel_id == "RECENT_RETAIL":
            review = target.review_decision
            return review.decision == "APPROVE" and review.is_current_tail
        if target.channel_id != "HISTORICAL_RETAIL":
            return False
        expected_kinds = ("MONTHLY", "REGIONAL_DAILY", "MARKET_DAILY")
        target_reviews = tuple(target.reviews)
        if (
            len(target_reviews) != 3
            or len({review.id for review in target_reviews}) != 3
        ):
            return False
        return all(
            review.is_current_tail
            and review.decision == "APPROVE"
            and review.collection.kind == expected_kind
            and review.collection.state == "VALIDATED"
            and review.collection.code_manifest_sha256 == target.code_manifest_sha256
            and review.approved_result_sha256 == review.collection.result_sha256
            and review.approved_partition_manifest_sha256
            == review.collection.partition_manifest_sha256
            for review, expected_kind in zip(target_reviews, expected_kinds, strict=True)
        )

    # 호출자가 expected version까지 잘라 준 history에서도 같은 channel의 실제 target만 인정한다.
    return any(
        event.channel_id == target.channel_id
        and event.operation in {Operation.ACTIVATE, Operation.ROLLBACK}
        and event.target_revision_id == target.id
        for event in history
    )
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

  **모범답변:** 새 ACTIVATE에는 현재 review tail을 요구해 최신 검수 정책을 지키지만, 같은 조건을 ROLLBACK에도 적용하면 review가 갱신된 직후 검증된 마지막 정상본조차 복구하지 못할 수 있습니다. 프로젝트는 rollback에 한해 sealed이고 과거 current였다는 더 좁은 운영 증거를 사용해 가용성을 보존합니다.

- last-known-good를 history로 증명하는 이유

  **모범답변:** revision이 존재하거나 sealed됐다는 사실은 실제 공개 운영에서 정상본으로 사용됐다는 뜻이 아닙니다. 같은 channel의 이전 ACTIVATE/ROLLBACK event에서 exact target으로 등장했을 때만 과거 current였음을 증명할 수 있습니다.

- publication rollback과 application/database rollback의 분리

  **모범답변:** publication rollback은 동일한 실행 코드에서 공개 데이터 pointer만 과거 sealed revision으로 바꿉니다. 애플리케이션 배포나 DB schema rollback은 실행 파일·migration 호환성을 다루는 별도 장애 영역이므로, 하나의 명령으로 묶으면 데이터 호환성과 배포 안전성을 동시에 보장하기 어렵습니다.

- 추후 revocation 개념을 추가할 확장 지점

  **모범답변:** 공통 target gate에 immutable revocation 조회를 추가해 ACTIVATE와 ROLLBACK 모두보다 먼저 거부하고, DB activation guard에도 같은 조건을 둡니다. revocation 자체도 actor·reason·evidence hash를 가진 append-only 기록으로 남겨 과거 activation history를 삭제하지 않는 것이 적절합니다.

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

  **모범답변:** 애플리케이션 권한 검사는 어떤 actor와 operation이 허용되는지 표현하고 감사 가능한 의도를 검증합니다. 역할별 DB credential은 raw SQL, 애플리케이션 취약점이나 잘못된 코드 경로가 생겨도 실제 가능한 SQL 권한을 제한합니다. 프로젝트도 전자를 defense in depth로 구현하고 후자는 production 플랫폼의 MFA/IAM·grant 경계로 명시했으며, 일반적으로 둘 중 하나가 우회돼도 다른 층이 피해 범위를 줄입니다.

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
    contracts = {
        Role.REVIEWER: (
            "grocery-control-reviewer",
            {
                "grocery.review_generation",
                "grocery.review_historical_collection",
            },
        ),
        Role.PUBLISHER: (
            "grocery-control-publisher",
            {
                "grocery.publish_publication",
                "grocery.publish_historical_publication",
            },
        ),
    }
    if role not in contracts:
        raise ControlPlaneError(ControlPlaneCode.ACTOR_CONFLICT)

    production_requested = (
        runtime.control_plane_operations_enabled is True
        or expected_release_sha is not None
    )
    if not production_requested:
        runtime.require_local_phase0_environment()
        actor = actors.get_local_operator()
        return OperationActor(actor=actor, actor_id=actor.id, production=False)

    if (
        runtime.debug is not False
        or runtime.admin_enabled is not False
        or runtime.qa_state_previews_enabled is not False
        or runtime.control_plane_operations_enabled is not True
    ):
        raise ControlPlaneError(ControlPlaneCode.ENVIRONMENT_DENIED)

    release_pattern = re.compile(r"[0-9a-f]{40}\Z")
    if (
        not isinstance(expected_release_sha, str)
        or release_pattern.fullmatch(expected_release_sha) is None
        or not isinstance(runtime.deploy_version, str)
        or release_pattern.fullmatch(runtime.deploy_version) is None
    ):
        raise ControlPlaneError(ControlPlaneCode.RELEASE_SHA_INVALID)
    if expected_release_sha != runtime.deploy_version:
        raise ControlPlaneError(ControlPlaneCode.RELEASE_SHA_MISMATCH)

    username, expected_permissions = contracts[role]
    actor = actors.find_by_username(username)
    if actor is None:
        raise ControlPlaneError(ControlPlaneCode.ACTOR_MISSING)
    if (
        actor.username != username
        or actor.email != ""
        or actor.first_name != ""
        or actor.last_name != ""
        or actor.is_active is not True
        or actor.is_staff is not False
        or actor.is_superuser is not False
        or actor.has_usable_password()
        or actor.groups.exists()
        or set(actor.get_all_permissions()) != expected_permissions
    ):
        # 고정 code만 노출하고 username, SHA, 실제 permission은 반사하지 않는다.
        raise ControlPlaneError(ControlPlaneCode.ACTOR_CONFLICT)
    if not isinstance(actor.id, int) or isinstance(actor.id, bool) or actor.id < 1:
        raise ControlPlaneError(ControlPlaneCode.ACTOR_ID_INVALID)
    return OperationActor(actor=actor, actor_id=actor.id, production=True)
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

  **모범답변:** 프로젝트는 reviewer에게 recent/historical review 권한만, publisher에게 recent/historical publication 전환 권한만 줍니다. 한 고정 actor가 검수와 공개를 모두 끝낼 수 없고 각 역할도 필요한 두 permission만 가져, 계정 오용 시 가능한 작업 범위를 제한합니다.

- feature flag/release lock과 인증의 차이

  **모범답변:** enable flag와 40자리 deploy SHA 일치는 올바른 production 환경과 release에서 명령이 실행되는지만 확인하는 사고 방지 장치입니다. 실행 요청자가 누구인지나 MFA 승인을 증명하지 않으므로, 원본 모듈도 외부 IAM/MFA를 별도 필수 경계로 명시합니다.

- exact permission 검증의 운영 부담 대 안전성

  **모범답변:** 권한 추가가 필요할 때 contract와 bootstrap 절차를 함께 갱신해야 하고 drift 때문에 작업이 중단될 수 있어 운영 부담이 생깁니다. 대신 누군가 편의상 group이나 추가 permission을 붙인 상태를 자동 수용하지 않아 privilege creep을 즉시 탐지합니다.

- application 권한과 DB 권한을 겹쳐 쓰는 defense in depth

  **모범답변:** application 권한은 도메인 actor와 명령 의도를 검사하고, DB grant와 trigger는 ORM 우회·raw SQL·코드 결함에도 가능한 mutation 자체를 제한합니다. 서로 다른 실패 경로를 막기 때문에 함께 사용할 때 단일 경계보다 안전합니다.

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

  **모범답변:** 현재는 recent와 historical pointer가 별도 transaction으로 전환되고 상세 요청도 각각 읽으므로, 둘이 특정 조합으로 함께 공개됐다는 atomic cut을 증명할 수 없습니다. 그런 요구가 생기면 두 exact revision id를 묶는 상위 bundle/super-channel을 한 CAS로 활성화하거나 동일 transaction snapshot에서 결합해야 합니다. 일반적으로 이 선택은 현재의 장애 격리와 독립 배포 장점을 줄입니다.

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
    # recent는 상세 페이지의 필수 publication이므로 실패를 성공/empty로 바꾸지 않는다.
    recent = load_recent()
    if recent is None:
        raise DetailNotFound("active recent publication is missing")

    historical = None
    try:
        candidate = load_historical()
        if candidate is not None and candidate.matches_recent_series:
            historical = candidate
    except (DatabaseError, ObjectDoesNotExist, ValidationError):
        # 원본 detail view도 선택 의존성의 알려진 DB/조회/무결성 실패만 숨긴다.
        log_event(_LOGGER, "ERROR", "public.detail.history_hidden")
        historical = None

    return DetailState(
        recent=recent,
        historical=historical,
        recent_fact_set_sha256=recent.typed_fact_set_sha256,
        historical_fact_set_sha256=(
            historical.typed_fact_set_sha256 if historical is not None else None
        ),
    )
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

  **모범답변:** recent와 historical은 source cadence, 최대 허용 age, review와 fact shape가 다릅니다. 별도 channel이면 history 수집·검수·rollback이 recent 공개 pointer를 건드리지 않고, recent 갱신도 장기 history bundle을 다시 seal할 필요가 없습니다.

- graceful degradation과 오류 은폐의 경계

  **모범답변:** recent detail은 필수 계약이므로 active revision이나 해당 entry가 없으면 404로, 알려진 DB·조회·검증 실패면 감사 로그를 남기는 503 경로로 보냅니다. historical은 보조 링크이므로 그 경계의 알려진 실패만 `public.detail.history_hidden`으로 기록한 뒤 숨기며, recent 실패까지 삼키지는 않습니다.

- 교차 채널 강한 일관성이 필요할 때의 대안

  **모범답변:** recent revision id와 historical revision id를 함께 가리키는 immutable bundle 및 단일 CAS pointer를 추가하면 두 hash의 조합을 원자적으로 공개할 수 있습니다. 또는 같은 DB transaction snapshot에서 두 pointer를 읽을 수 있지만, 독립 activation 사이의 비즈니스적 호환성까지 보장하려면 bundle 검수가 더 명확합니다.

- optional dependency 예외 범위를 좁히는 방법

  **모범답변:** 원본처럼 `DatabaseError`, `ObjectDoesNotExist`, `ValidationError`처럼 예상한 read 실패만 historical 경계에서 잡고 즉시 고정 audit event를 남깁니다. `Exception` 전체를 잡지 않으면 프로그래밍 오류까지 정상 degrade로 위장하지 않으며, recent loader 호출은 이 try 범위 밖에 둡니다.

### 원본 확인 위치

- Thread 13, Thread 16
- 커밋 `feat(public): connect recent catalog extensions`
- 파일 `grocery/historical_public_read.py`
- 구성 요소 `load_active_historical_publication`, `historical_series_for_recent`
- 파일 `grocery/views.py`; 구성 요소 `detail`
