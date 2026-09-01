# 복제·전송 보안·재접속 면접 워크북

이 문서는 G08~G11에서 확인되는 FULL/DELTA 복제, ACK watermark와 resynchronization, TCP/UDP 경계, reconnect grace와 credential rotation을 묶는다. G08과 G10은 하나의 snapshot base contract를 단계적으로 완성하므로 관련 항목에서 명시적으로 통합했다.

---

<a id="w03-g08-delta"></a>

## [G08 / `feat(snapshot): add acknowledged full and delta replication`] FULL·DELTA 생성과 적용 계약

### 면접 질문

서버가 DELTA를 만들 때 wire에는 변경분만 보내면서도 retained base는 FULL materialization으로 보관한 이유는 무엇입니까? 삭제된 player를 빈 상태나 connectivity 값으로 추론하지 않고 `removed_player_ids`로 명시한 이유는 무엇입니까?

꼬리 질문:

- "직전 발행본"이 아니라 "클라이언트가 실제 ACK한 retained snapshot"을 base로 잡아야 하는 이유는 무엇입니까?
  - 모범답변: 전송 성공이나 발행 순서는 클라이언트 적용 성공을 보장하지 않는다. ACK했고 서버 retention에도 남아 있는 sequence만 양쪽이 공유한다고 확인된 materialized base다.
- DELTA에 unchanged player row를 포함하지 않았는지 어떻게 검증합니까?
  - 모범답변: ACK base와 current의 player ID별 visible row를 독립 비교해 동일한 row가 `players`에 없는지 검사한다. 이어 `apply(base, delta)`가 current projection과 정확히 같은지도 확인한다.
- FULL과 DELTA를 적용한 replica가 authoritative visible projection과 같다는 테스트는 production serializer를 재사용하면 왜 약해집니까?
  - 모범답변: 생성과 검증이 같은 버그를 공유하면 둘 다 같은 잘못된 결과를 내고 테스트가 통과할 수 있다. 테스트 replica는 wire contract만 해석하는 별도 구현으로 changed·removed를 적용해야 한다.
- 상태 hash는 FULL wire payload의 hash입니까, 전체 authoritative canonical state의 hash입니까?
  - 모범답변: 원본에서는 `ReplayLog`와 같은 전체 authoritative canonical state의 SHA-256이다. visible wire projection이나 sparse DELTA 표현과 분리돼 simulation 의미를 검증한다.
- player 순서와 removal 순서를 고정하지 않으면 어떤 결정성 문제가 생깁니까?
  - 모범답변: 같은 map 내용도 iteration 순서에 따라 wire byte와 테스트 artifact가 달라질 수 있다. 원본의 player map과 removal 생성은 ID 순서를 안정적으로 유지해 동일 상태가 동일 표현이 되게 한다.
- retained entry를 wire DELTA 그대로 저장하면 다음 DELTA 생성에서 어떤 비용과 오류가 생깁니까?
  - 모범답변: 다음 ACK base를 만들 때 이전 delta chain을 재적용해야 하고 한 조각이 없거나 제거를 잘못 합치면 잘못된 current가 된다. 매 sequence의 FULL materialization을 보존하면 임의 retained ACK와 즉시 비교할 수 있다.
- retention 32가 찼을 때 어떤 항목을 제거하고 어떤 client feedback에 대비해야 합니까?
  - 모범답변: 가장 오래된 materialization을 제거하고 최신 32개만 남긴다. client가 evicted sequence를 ACK하면 watermark로 채택하지 않고 `ACK_OUTSIDE_RETENTION`을 latch해 다음 publication을 FULL로 만든다.

### 30초 모범 답변

DELTA는 ACK로 확인된 base와 현재 visible state의 차이만 보내지만, 서버 retention에는 각 sequence의 완전한 materialization을 둬야 임의의 retained ACK를 즉시 비교할 수 있습니다. 변경·추가 player는 정렬된 row로, 삭제는 정렬된 ID 목록으로 명시해 적용 결과가 하나로 결정되게 합니다. 클라이언트는 실제 보유한 base에만 DELTA를 적용하고, `apply(base, delta)`가 authoritative visible projection과 정확히 같아야 합니다. state hash는 wire 변경분이 아니라 전체 canonical authoritative state를 가리켜 복제 표현과 simulation 의미를 분리합니다.

### 답변 핵심 키워드

acknowledged base, full materialization retention, sparse delta, explicit removal, deterministic order, independent replica, visible projection, canonical state hash

### 백지 구현

#### 구현 목표

정렬된 FULL visible state 두 개로부터 DELTA를 생성하고, base에 DELTA를 적용해 current state를 재구성한다.

#### 인터페이스 또는 함수 시그니처

```cpp
struct VisiblePlayer {
    std::string player_id;
    int slot;
    int x;
    int y;
    Direction direction;
    int score;
    std::string connectivity;

    bool operator==(const VisiblePlayer&) const = default;
};

struct FullSnapshot {
    std::uint64_t snapshot_seq;
    std::int64_t tick;
    std::string room_id;
    std::uint64_t owner_epoch;
    std::string status;
    std::string state_hash;
    std::vector<VisiblePlayer> players;
};

struct DeltaSnapshot {
    std::uint64_t snapshot_seq;
    std::uint64_t base_snapshot_seq;
    std::int64_t tick;
    std::string room_id;
    std::uint64_t owner_epoch;
    std::string state_hash;
    std::vector<VisiblePlayer> changed_players;
    std::vector<std::string> removed_player_ids;
};

DeltaSnapshot make_delta(const FullSnapshot& base,
                         const FullSnapshot& current);

FullSnapshot apply_delta(const FullSnapshot& base,
                         const DeltaSnapshot& delta,
                         std::string current_status);
```

#### 입력과 출력

- 입력: 동일 room·owner stream에 속한 base FULL과 current FULL
- 출력: 변경·추가·삭제만 담은 DELTA, 또는 DELTA를 적용한 새로운 FULL materialization

#### 반드시 만족해야 할 조건

- 입력 player ID는 유일해야 하며 결과는 `player_id` 오름차순이다.
- current에 새로 생기거나 필드가 달라진 player만 `changed_players`에 포함한다.
- base에만 있던 player ID는 `removed_player_ids`에 정확히 한 번 포함한다.
- unchanged player는 DELTA에 포함하지 않는다.
- DELTA의 base sequence는 실제 base와 일치해야 한다.
- room ID와 owner epoch가 다르면 실패한다.
- `apply_delta(base, make_delta(base, current))`의 visible 내용은 current와 같아야 한다.
- 입력 객체는 변경하지 않는다.
- state hash는 current의 전체 authoritative state를 가리키는 metadata로 그대로 전달한다.

#### 경계 조건

- 양쪽 player가 모두 없음
- 모두 unchanged
- 한 명 추가, 한 명 제거, 같은 ID의 여러 필드 변경
- 모든 player 제거
- base와 current player 입력 순서가 뒤섞인 경우
- 중복 player ID
- snapshot sequence가 역행하거나 base보다 작지 않은 current가 아닌 경우
- room ID·owner epoch 불일치

#### 실패 조건

- 존재하지 않던 ID를 removal로 적용하거나 removal과 changed에 동시에 넣으면 실패다.
- unchanged row가 DELTA에 남으면 요구사항 위반이다.
- base를 찾을 수 없는데 빈 state에서 DELTA를 적용하면 실패다.
- 적용 결과의 player ID가 중복되면 실패다.

#### 필요한 제약

- 20~30분 범위다.
- 입력 vector 정렬 여부를 신뢰할지 검증할지 선택하고 설명한다.
- serialization과 network 전송은 범위에서 제외한다.
- 목표 복잡도는 정렬된 입력이면 O(B+C), 아니면 O((B+C) log(B+C)) 이내다.

### 구현 후 자가 검증

- 정상 경로: unchanged·changed·added·removed가 섞인 예제로 round-trip 확인
- 경계값: 빈 state, 전원 제거, 전원 추가 확인
- 실패 경로: 중복 ID와 stream identity 불일치 확인
- 상태 변화: base 객체가 적용 과정에서 수정되지 않는지 확인
- invariant: changed와 removed ID 집합이 겹치지 않고 각각 정렬·유일한지 확인
- 중복·누락 처리: current의 모든 ID가 적용 결과에 정확히 한 번 존재하는지 확인
- 시간·공간 복잡도: map을 쓸 때와 정렬 vector two-pointer의 trade-off 설명
- 요구사항 충족: round-trip 결과가 current visible projection과 정확히 같은지 확인

### 구현 후 설명할 것

1. retained base를 FULL materialization으로 저장한 이유
   - 모범답변: ACK는 최신 발행본이 아닐 수 있으므로 retained 어느 sequence든 완전한 visible state로 바로 비교할 수 있어야 한다. delta chain 재생 비용과 missing-link 오류도 피한다.
2. 삭제를 명시적인 ID 목록으로 보낸 이유
   - 모범답변: row 부재나 connectivity 값만으로 삭제를 추론하면 적용 구현마다 결과가 달라질 수 있다. 정렬된 `removed_player_ids`가 base에서 어떤 key를 제거할지 명시한다.
3. ACK된 base와 단순 최신 발행본의 차이
   - 모범답변: 최신 발행본은 서버가 만들었다는 사실만 있고 client 보유 여부는 모른다. ACK된 retained base는 양쪽이 같은 sequence를 실제로 공유한다는 확인이 있다.
4. canonical state hash와 wire delta 표현을 분리한 이유
   - 모범답변: FULL과 DELTA는 같은 authoritative state를 다른 전송 효율로 표현할 수 있다. hash를 sparse wire bytes가 아니라 canonical state에 매기면 표현 종류와 관계없이 정합성을 비교할 수 있다.
5. retention 상한과 FULL fallback의 관계
   - 모범답변: 상한은 stream당 메모리를 제한하는 대신 오래된 ACK base를 잃을 수 있다. 그 경우 잘못된 DELTA를 만들지 않고 다음 scheduled FULL로 새 base를 세운다.

### 원본 확인 위치

- Thread: G08
- 커밋 메시지: `feat(snapshot): add acknowledged full and delta replication`
- 파일: `src/snapshot.hpp`, `src/snapshot.cpp`, `src/transport.hpp`, `src/transport.cpp`, `tests/g07.hpp`, `tests/g07.cpp`
- 함수·클래스·컴포넌트: `SnapshotStream`, `SnapshotStream::publish`, `SnapshotStream::acknowledge`, `SnapshotStream::metrics`, `Server::publish_snapshots`, `Server::start_room`, `run_snapshot_scenario`
- 관련 Thread: G07의 canonical state hash, G10의 ACK 검증·FULL resync, G12의 snapshot coalescing

---

<a id="w03-g08-g10-ack"></a>

## [G08 / `feat(snapshot): add acknowledged full and delta replication` + G10 / `feat(snapshot): validate monotonic acknowledgements and schedule full resync`] ACK watermark와 다음 scheduled FULL

### 면접 질문

클라이언트가 unknown sequence, retention 밖의 과거 sequence, hash mismatch, 또는 `resync_required`를 ACK로 보냈을 때 서버 watermark를 어떻게 처리해야 합니까? 왜 즉시 임의의 snapshot을 보내지 않고 "다음 scheduled publication을 FULL로 만들 이유"를 latch했습니까?

꼬리 질문:

- 유효하지만 현재 watermark보다 낮은 ACK가 도착하면 rollback해야 합니까?
  - 모범답변: rollback하지 않는다. retained hash가 맞아도 늦게 도착한 feedback일 뿐이며 이미 더 최신 base를 확인한 사실을 취소할 이유가 없다.
- 아직 발행하지 않은 sequence 999를 ACK한 뒤 나중에 sequence 999가 발행되면 유효 base가 될 수 있습니까?
  - 모범답변: 될 수 없다. ACK 시점에 retained publication이 아니므로 `UNKNOWN_ACK`이며 watermark에는 기록하지 않는다. 미래에 같은 숫자가 발행돼도 과거의 검증되지 않은 feedback을 소급 적용하지 않는다.
- retention에서 evict된 ACK와 미래 unknown ACK를 같은 오류로 취급할 수 있습니까?
  - 모범답변: 둘 다 watermark를 바꾸지 않고 FULL을 예약하지만 관측 reason은 나눈다. 원본은 `sequence <= last_sequence`이면 `ACK_OUTSIDE_RETENTION`, 0 또는 미래면 `UNKNOWN_ACK`로 기록해 원인을 진단한다.
- hash mismatch가 authoritative state를 변경하면 안 되는 이유는 무엇입니까?
  - 모범답변: ACK는 client replica에 대한 feedback이지 game command가 아니다. mismatch는 복제 복구를 예약할 뿐 Room simulation, tick, input state를 변경해서는 안 된다.
- client가 로컬 base를 잃었지만 서버 watermark는 정상이라면 어떤 feedback이 필요합니까?
  - 모범답변: ACK에 `resync_required=true`를 보내 server가 `RESYNC_REQUIRED`를 latch하게 한다. 유효 ACK의 watermark는 유지할 수 있지만 다음 ordinary publication은 FULL이어야 한다.
- resync reason을 FULL 발행 시점에 소비하고 바로 지우는 이유는 무엇입니까?
  - 모범답변: reason이 필요로 한 복구 work가 실제로 생성된 시점이기 때문이다. 미리 지우면 FULL이 누락될 수 있고, 남겨 두면 이후 모든 publication이 불필요하게 FULL이 된다.
- 여러 오류 reason이 연속으로 오면 어떤 우선순위 또는 관측 정책이 필요합니까?
  - 모범답변: publication 전 가장 최근에 확인된 구체 reason으로 갱신하고 pending 여부는 유지하는 정책을 사용할 수 있다. 원본도 `UNKNOWN_ACK`, `ACK_OUTSIDE_RETENTION`, `HASH_MISMATCH`, `RESYNC_REQUIRED`를 새 사건으로 덮어쓰되 watermark는 건드리지 않는다.

### 30초 모범 답변

ACK는 실제 retained publication과 선택적 hash가 일치할 때만 watermark를 앞으로 이동시킵니다. 미래 unknown ACK, retention 밖 ACK, hash mismatch는 현재 watermark를 바꾸지 않고 next FULL reason만 latch하며, 낮은 유효 ACK도 rollback시키지 않습니다. `resync_required` 역시 gameplay나 authoritative state를 건드리지 않고 다음 정규 publication을 FULL로 선택하게 합니다. reason을 발행 시점에 소비하면 별도 즉시 전송으로 cadence와 outbound bound를 깨지 않으면서 클라이언트를 복구할 수 있습니다.

### 답변 핵심 키워드

retained ACK, monotonic watermark, no rollback, unknown vs evicted, optional hash validation, resync latch, scheduled FULL, feedback does not simulate

### 백지 구현

#### 구현 목표

retained snapshot metadata와 ACK를 검증해 monotonic watermark와 pending resync reason을 관리하고, 다음 publication이 FULL인지 DELTA인지 결정하는 상태기를 구현한다.

#### 인터페이스 또는 함수 시그니처

```cpp
enum class ResyncReason {
    None,
    UnknownAck,
    AckOutsideRetention,
    HashMismatch,
    ResyncRequired,
    NoRetainedBase,
    Periodic,
    RoomStatus,
    RoomStart
};

struct SnapshotMeta {
    std::uint64_t sequence;
    std::string state_hash;
};

struct AckRequest {
    std::uint64_t sequence;
    std::optional<std::string> state_hash;
    bool resync_required;
};

struct AckOutcome {
    std::optional<std::uint64_t> watermark;
    ResyncReason pending_reason;
    bool advanced;
};

enum class PublicationKind { Full, Delta };

struct PublicationDecision {
    PublicationKind kind;
    std::optional<std::uint64_t> base_sequence;
    ResyncReason full_reason;
};

class SnapshotAckState {
public:
    explicit SnapshotAckState(std::size_t retention_capacity)
        : retention_capacity_(retention_capacity) {
        if (retention_capacity_ == 0)
            throw std::invalid_argument("retention capacity must be positive");
    }

    void retain(SnapshotMeta meta) {
        if (!retained_.empty() && meta.sequence <= retained_.back().sequence)
            throw std::logic_error("snapshot sequence must increase");
        last_published_ = meta.sequence;
        if (retained_.size() == retention_capacity_) retained_.pop_front();
        retained_.push_back(std::move(meta));
    }

    AckOutcome acknowledge(const AckRequest& request) {
        const auto found = std::find_if(retained_.begin(), retained_.end(),
            [&](const SnapshotMeta& item) {
                return item.sequence == request.sequence;
            });
        if (found == retained_.end()) {
            pending_reason_ = request.sequence == 0 ||
                                      request.sequence > last_published_
                                  ? ResyncReason::UnknownAck
                                  : ResyncReason::AckOutsideRetention;
            return {watermark_, pending_reason_, false};
        }
        if (request.state_hash && *request.state_hash != found->state_hash) {
            pending_reason_ = ResyncReason::HashMismatch;
            return {watermark_, pending_reason_, false};
        }
        if (request.resync_required)
            pending_reason_ = ResyncReason::ResyncRequired;

        const bool advanced = !watermark_ || request.sequence > *watermark_;
        if (advanced) watermark_ = request.sequence;
        return {watermark_, pending_reason_, advanced};
    }

    PublicationDecision decide_next(bool room_running,
                                    bool periodic_boundary,
                                    std::uint64_t next_sequence) {
        auto reason = pending_reason_;
        if (reason == ResyncReason::None && next_sequence == 1)
            reason = ResyncReason::RoomStart;
        else if (reason == ResyncReason::None && periodic_boundary)
            reason = ResyncReason::Periodic;
        else if (reason == ResyncReason::None && !room_running)
            reason = ResyncReason::RoomStatus;

        const bool retained_base = watermark_ &&
            std::any_of(retained_.begin(), retained_.end(),
                [&](const SnapshotMeta& item) {
                    return item.sequence == *watermark_;
                });
        if (reason == ResyncReason::None && !retained_base)
            reason = ResyncReason::NoRetainedBase;

        if (reason != ResyncReason::None) {
            // 실제 FULL decision이 만들어질 때만 오류 latch를 소비한다.
            pending_reason_ = ResyncReason::None;
            return {PublicationKind::Full, std::nullopt, reason};
        }
        return {PublicationKind::Delta, watermark_, ResyncReason::None};
    }

private:
    std::size_t retention_capacity_;
    std::deque<SnapshotMeta> retained_;
    std::uint64_t last_published_ = 0;
    std::optional<std::uint64_t> watermark_;
    ResyncReason pending_reason_ = ResyncReason::None;
};
```

#### 입력과 출력

- 입력: 발행 완료 snapshot metadata, client ACK, 다음 publication의 room·periodic 조건
- 출력: monotonic watermark와 pending reason, FULL/DELTA 및 DELTA base

#### 반드시 만족해야 할 조건

- retention은 최신 N개 metadata만 보존한다.
- ACK sequence가 retained에 없으면 watermark를 바꾸지 않는다.
- sequence가 아직 발행되지 않았거나 0인 경우와 과거 evicted sequence를 구분한다.
- 선택적 hash가 제공됐고 다르면 watermark를 바꾸지 않는다.
- `resync_required`는 유효 ACK 여부와 별개로 next FULL 필요를 표시할 수 있어야 하며 정책을 명시한다.
- 유효 ACK는 기존 watermark보다 큰 경우에만 앞으로 이동한다.
- 낮은 유효 ACK는 rollback하지 않는다.
- DELTA는 현재 watermark가 retention에 남아 있을 때만 선택한다.
- pending reason, room start, periodic boundary, non-running status, missing base는 FULL을 선택한다.
- pending 오류 reason은 실제 FULL publication decision에서 소비된다.
- ACK나 publication feedback은 외부 simulation state를 변경하지 않는다.

#### 경계 조건

- 첫 publication 전 ACK
- sequence 0, 미래 sequence, 정확히 최신 sequence
- retention 첫 항목과 바로 이전 evicted 항목
- valid high ACK 후 valid low ACK
- 같은 ACK 반복
- hash 생략, hash 일치, hash 불일치
- 여러 resync reason이 FULL 전에 연속 도착
- pending reason이 있는 동시에 periodic boundary
- watermark base가 retention에서 eviction된 직후

#### 실패 조건

- unknown ACK가 watermark가 되면 실패다.
- late ACK가 watermark를 낮추면 실패다.
- hash mismatch가 valid base로 채택되면 실패다.
- FULL reason을 decision 전에 지우거나 FULL 후에도 무한히 유지하면 실패다.
- base가 retention에 없는데 DELTA를 선택하면 실패다.

#### 필요한 제약

- 20~25분 범위다.
- snapshot payload 생성은 범위에서 제외한다.
- reason 충돌 우선순위는 코드와 설명에서 일관되면 된다.
- retention lookup 자료구조 선택과 상한 N의 의미를 설명해야 한다.

### 구현 후 자가 검증

- 정상 경로: retained ACK가 watermark를 전진시키고 다음 DELTA base가 되는지 확인
- 경계값: retention 경계 안팎의 ACK를 구분하는지 확인
- 실패 경로: 미래 ACK, hash mismatch, missing base가 FULL reason을 latch하는지 확인
- 상태 변화: high ACK 뒤 low ACK에서 watermark 유지 확인
- invariant: watermark가 있다면 과거에 실제 발행된 sequence이며 절대 감소하지 않는지 확인
- 중복·누락 처리: 같은 오류 ACK 반복이 무한한 별도 publication을 만들지 않는지 확인
- 보안·정합성: client feedback이 authoritative state 객체에 접근하지 않는지 확인
- 시간·공간 복잡도: retention capacity에 대해 lookup과 eviction 비용 설명
- 요구사항 충족: reason이 다음 FULL에서 한 번 소비되는지 확인

### 구현 후 설명할 것

1. ACK watermark를 monotonic하게 유지한 이유
   - 모범답변: 더 최신 retained state를 client가 보유한다고 확인한 뒤 늦은 ACK가 그 사실을 취소하면 DELTA base가 불필요하게 과거로 돌아간다. `max(current, valid_ack)`만 적용하면 reorder에도 안정적이다.
2. unknown ACK와 evicted ACK를 나눈 이유
   - 모범답변: 둘 다 복구 동작은 FULL이지만 전자는 잘못되거나 미래인 feedback이고 후자는 정상 지연이 retention보다 길어진 경우다. 관측 reason을 나누면 protocol abuse와 capacity tuning 문제를 구분할 수 있다.
3. 즉시 FULL 대신 다음 scheduled FULL을 선택한 이유
   - 모범답변: ACK가 별도 publication cadence와 sequence를 만들지 않아 outbound work 상한을 지킬 수 있다. 프로젝트는 reason만 latch하고 다음 정상 snapshot 생성에서 FULL로 소비한다.
4. state hash 검증이 stale·corrupt replica를 탐지하는 방식
   - 모범답변: retained sequence가 같아도 client가 보고한 canonical state hash가 다르면 같은 base를 가진 것이 아니다. watermark를 유지하고 FULL을 예약해 잘못된 DELTA 적용을 막는다.
5. replication feedback과 simulation mutation을 분리한 경계
   - 모범답변: ACK state는 connection별 retention·watermark·resync reason만 변경한다. Room model, accepted input, executed tick에는 접근하지 않아 client feedback이 authoritative gameplay를 조작할 수 없다.

### 원본 확인 위치

- 대표 Thread: G08
- 통합 Thread: G10
- G08 커밋 메시지: `feat(snapshot): add acknowledged full and delta replication`
- G10 커밋 메시지: `feat(snapshot): validate monotonic acknowledgements and schedule full resync`
- 파일: `src/snapshot.hpp`, `src/snapshot.cpp`, `src/transport.cpp`, `tests/g10.hpp`, `tests/g10.cpp`
- 함수·클래스·컴포넌트: `SnapshotStream::acknowledge`, `SnapshotStream::publish`, `SnapshotStream::metrics`, `run_ack_scenario`
- 관련 Thread: G07의 state hash, G11의 reconnect 후 FULL, G12의 scheduled snapshot coalescing

---

<a id="w03-g09"></a>

## [G09 / 기록 제목: Bounded UDP data plane / TCP control and UDP data plane] TCP control·UDP data 경계와 one-time bind credential

### 면접 질문

이미 TCP session이 있는데 UDP endpoint를 별도 one-time token으로 bind한 이유는 무엇입니까? token을 언제 소비해야 하며, observed endpoint·session·owner epoch·만료를 어떤 순서로 검증해야 합니까?

꼬리 질문:

- UDP source address만 보고 player를 식별하면 어떤 공격과 오동작이 가능합니까?
  - 모범답변: NAT 재매핑·port 재사용으로 다른 client가 같은 endpoint처럼 보이거나 공격자가 source를 spoof해 player를 사칭할 수 있다. TCP-authenticated session이 발급한 credential과 실제 observed endpoint를 함께 묶어야 한다.
- 실패한 bind가 token을 소비하면 어떤 가용성 문제가 생깁니까?
  - 모범답변: 잘못된 epoch·endpoint 또는 일시 오류 한 번으로 정상 client가 같은 credential로 재시도하지 못한다. 모든 검증 뒤 endpoint 저장과 token clear를 한 commit으로 해야 실패 경로가 상태를 보존한다.
- 성공한 token을 다른 endpoint에서 재사용해 binding을 옮길 수 있게 하면 어떤 문제가 생깁니까?
  - 모범답변: token 탈취자가 기존 session의 data plane을 자기 endpoint로 가져갈 수 있고, 지연된 packet의 routing도 모호해진다. 프로젝트는 이미 bind됐거나 token이 비어 있으면 재bind를 거부한다.
- 다른 session의 유효 token을 제출했을 때 어느 session으로 오류를 돌려보내야 합니까?
  - 모범답변: token이 주장하는 대상이 아니라 실제 TCP/owner attribution이 확인된 connection의 control channel로 안정된 오류를 보내야 한다. credential 자체나 다른 session 존재 여부를 응답에 노출하지 않는다.
- UDP 입력 전송은 UDP로, 제어 오류는 TCP로 보낸 구조의 장단점은 무엇입니까?
  - 모범답변: realtime data는 head-of-line blocking을 피하고 control 오류는 신뢰성 있는 인증 채널로 전달할 수 있다. 대신 두 transport의 routing·credential·순서가 분리돼 wrong-transport와 endpoint 검증이 추가로 필요하다.
- 정확히 1,200바이트는 받고 1,201바이트는 거부하려면 수신 버퍼를 왜 상한+1로 잡을 수 있습니까?
  - 모범답변: 버퍼가 정확히 상한이면 더 큰 datagram이 truncate돼 원래 크기를 구분하기 어려울 수 있다. 1,201바이트까지 관찰하고 `count > 1200` 또는 `MSG_TRUNC`를 보면 decoder 전에 확실히 거부할 수 있다.
- expiry가 발급 후 5,000ms일 때 정확히 경계 시각은 유효합니까?
  - 모범답변: 원본 조건은 `now - issued < 5000`이므로 정확히 5,000ms부터 만료다. subtraction 비교를 써 `issued + ttl` overflow도 피한다.

### 30초 모범 답변

UDP는 연결 상태가 없으므로 TCP로 인증된 session과 관찰된 UDP endpoint를 일회용 credential로 묶어야 합니다. bind는 token 존재·session 소유·owner epoch·미사용 상태·만료 전·아직 미결합·observed endpoint를 모두 검증한 뒤에만 원자적으로 token을 소비하고 endpoint를 저장합니다. 실패는 credential을 보존해 정상 재시도를 막지 않고, 성공 token은 재사용이나 endpoint 이동을 허용하지 않습니다. payload는 상한+1로 읽어 oversize를 확실히 탐지하며, data-plane 입력과 control-plane 오류 경계를 분리합니다.

### 답변 핵심 키워드

connectionless transport, TCP-authenticated bootstrap, one-time token, consume-after-validation, endpoint binding, expiry boundary, owner epoch, max+1 oversize detection

### 백지 구현

#### 구현 목표

TCP에서 발급된 one-time UDP bind token을 검증·소비하고, 이후 UDP datagram이 해당 session·player·endpoint·transport 계약을 만족하는지 승인한다.

#### 인터페이스 또는 함수 시그니처

```cpp
struct Endpoint {
    std::string address;
    std::uint16_t port;
    bool operator==(const Endpoint&) const = default;
};

struct BindCredential {
    std::string token;
    std::string session_id;
    std::uint64_t owner_epoch;
    std::int64_t issued_ms;
    bool consumed;
};

struct SessionTransportState {
    std::string session_id;
    std::string player_id;
    std::uint64_t owner_epoch;
    std::optional<Endpoint> udp_endpoint;
};

enum class BindCode {
    Bound,
    Invalid,
    Expired,
    AlreadyBound
};

BindCode bind_udp(SessionTransportState& session,
                  BindCredential& credential,
                  std::string_view presented_token,
                  const Endpoint& observed_endpoint,
                  std::int64_t now_ms,
                  std::int64_t ttl_ms);

enum class DatagramCode {
    Accepted,
    NotBound,
    WrongEndpoint,
    WrongOwnerEpoch,
    Oversized,
    WrongTransport
};

DatagramCode authorize_datagram(const SessionTransportState& session,
                                const Endpoint& observed_endpoint,
                                std::uint64_t owner_epoch,
                                std::size_t payload_bytes,
                                bool received_over_udp);
```

#### 입력과 출력

- 입력: session transport 상태, bind credential, 관찰된 source endpoint·현재 monotonic 시간, datagram metadata
- 출력: 안정된 bind·datagram 승인 코드와 성공 시 갱신된 상태

#### 반드시 만족해야 할 조건

- credential token과 session ID, owner epoch가 모두 일치해야 한다.
- 성공 전까지 credential과 session state를 변경하지 않는다.
- 만료 판정은 `now_ms >= issued_ms + ttl_ms` 경계를 overflow 없이 처리한다.
- 이미 소비한 token은 다시 쓸 수 없다.
- 이미 bind된 session은 다른 endpoint로 이동하지 않는다.
- 성공 시 token 소비와 endpoint 저장이 하나의 논리적 commit이다.
- bind 전 datagram은 거부한다.
- bind 후 observed endpoint가 다르면 거부한다.
- payload 1,200바이트까지 허용하고 그보다 크면 거부한다.
- data-plane 입력이 TCP로 들어온 경우 `WrongTransport`다.
- 실패한 bind가 다른 session이나 gameplay state를 변경하지 않는다.

#### 경계 조건

- 만료 직전 4,999ms와 정확히 5,000ms
- unknown token, 다른 session token, 잘못된 owner epoch
- 성공 후 같은 endpoint 재시도와 다른 endpoint 재시도
- bind 전 INPUT, bind 후 PING
- payload 0, 1,200, 1,201
- `issued_ms + ttl_ms` overflow 가능성
- session은 존재하지만 player가 아직 할당되지 않은 상태

#### 실패 조건

- 검증 중간에 token을 먼저 소비하면 실패다.
- 다른 session의 token으로 endpoint가 연결되면 실패다.
- wrong endpoint datagram이 owner mailbox에 들어가면 실패다.
- oversize payload를 정상 decoder까지 전달하면 실패다.

#### 필요한 제약

- 15~25분 범위다.
- token 생성·암호학적 entropy와 실제 socket API는 범위에서 제외한다.
- 오류 전달 채널은 반환 코드로만 표현한다.
- 동시 호출은 owner에서 직렬화된다고 가정한다.

### 구현 후 자가 검증

- 정상 경로: 유효 token으로 bind 후 같은 endpoint datagram 승인 확인
- 경계값: TTL 직전/정확한 만료, 1,200/1,201바이트 확인
- 실패 경로: unknown·reused·other-session·wrong-epoch·wrong-endpoint 확인
- 상태 변화: 모든 실패 전후 credential와 endpoint가 동일한지 확인
- invariant: bind된 session은 endpoint 하나만 가지며 consumed token은 재사용되지 않는지 확인
- 중복 처리: 성공 token 재제출이 endpoint 이동을 만들지 않는지 확인
- 보안: 공격자가 token 또는 source address 하나만 알아서는 다른 session을 탈취할 수 없는지 검토
- 요구사항 충족: wrong transport가 gameplay admission 전에 차단되는지 확인

### 구현 후 설명할 것

1. TCP session과 UDP endpoint를 별도 credential로 연결한 이유
   - 모범답변: TCP는 인증된 연결 문맥이 있지만 UDP source는 자체 세션이 없다. TCP에서만 받은 일회용 token을 UDP observed endpoint와 owner에서 결합해 두 plane의 identity를 안전하게 연결한다.
2. validation 완료 후 token을 소비해야 하는 이유
   - 모범답변: token·session·epoch·TTL·미결합 상태를 모두 확인하기 전에 소비하면 실패한 요청이 정상 재시도를 막는다. 원본은 유효 조건을 하나로 계산한 뒤 endpoint 설정과 token clear를 연속 commit한다.
3. endpoint 이동을 자동 허용하지 않은 trade-off
   - 모범답변: NAT 변경 시 재bind 절차가 필요해 편의는 줄지만, 탈취 token이나 지연 packet이 binding을 몰래 옮기는 것을 막는다. reconnect에서는 old endpoint를 버리고 새 credential로 명시적으로 bind한다.
4. payload 상한+1 수신 방식의 의미
   - 모범답변: 실제 허용 최대보다 한 바이트 더 관찰해 oversize를 정상 최대와 구분하고 JSON parsing·mailbox allocation 전에 버린다. 이는 메모리와 CPU 공격 면을 transport 경계에서 제한한다.
5. control-plane과 data-plane 오류 전달을 나눈 이유
   - 모범답변: UDP 오류 응답은 spoofing·손실·증폭 위험이 있고 신뢰하기 어렵다. 인증된 TCP connection에 오류를 귀속시키면 안정적으로 전달할 수 있지만, data packet routing과 control identity의 매칭 검증이 선행돼야 한다.

### 원본 확인 위치

- Thread: G09
- 기록 제목: Bounded UDP data plane / TCP control and UDP data plane (Conventional Commit subject는 현재 프로젝트 기록에서 확인되지 않음)
- 파일: `src/transport.hpp`, `src/transport.cpp`, `tests/g09.hpp`, `tests/g09.cpp`
- 함수·클래스·컴포넌트: `UdpClient`, `Server::bind_datagram`, `Server::read_datagrams`, `Server::send_realtime`, `decode_datagram`, `encode_datagram`
- 관련 Thread: G03의 session·player ownership, G11의 credential rotation, G12의 UDP readiness·bounded send

---

<a id="w03-g11"></a>

## [G11 / `feat(session): add bounded reconnect grace and rotating credentials`] 재접속 transaction과 현재 상태 FULL 복구

### 면접 질문

연결이 끊긴 player를 즉시 LEFT로 제거하지 않고 실행 tick 기준 grace를 둔 이유는 무엇입니까? reconnect 성공 시 stable identity는 보존하면서 resume token과 UDP bind token을 회전시키는 과정을 어떻게 원자적으로 처리합니까?

꼬리 질문:

- wall time이 아니라 `Room::executed_ticks()`로 grace를 판정한 이유는 무엇입니까?
  - 모범답변: reconnect 가능 기간이 simulation pause나 scheduler 지연, wall clock 보정에 흔들리지 않고 게임 진행량과 일치해야 한다. 원본은 disconnect 때 `executed_ticks + 200`을 저장한다.
- `executed_ticks == disconnect_deadline`은 성공입니까, 만료입니까?
  - 모범답변: 만료다. `Room::reconnect`와 `Server::reconnect` 모두 현재 tick이 deadline 이상이면 거부하므로 마지막 허용 상태는 `deadline - 1`이다.
- 새 TCP 연결의 provisional HELLO session을 그대로 남기면 어떤 identity 중복이 생깁니까?
  - 모범답변: 한 connection에 provisional session과 stable session이 함께 active하거나 registry가 두 session을 같은 player로 routing할 수 있다. 성공 commit에서 provisional credential을 폐기하고 connection identity를 stable session으로 교체해야 한다.
- reconnect 전에 player가 이미 connected라면 왜 거부해야 합니까?
  - 모범답변: 두 transport가 한 player를 동시에 소유하는 takeover가 되어 input·snapshot routing이 분기된다. reconnect는 DISCONNECTED이고 유효 deadline이 있는 player에만 허용한다.
- 기존 UDP endpoint를 자동 승계하지 않고 새 bind를 요구한 이유는 무엇입니까?
  - 모범답변: 새 TCP connection은 네트워크 경로가 바뀌었을 수 있고 old endpoint에는 지연 packet이나 공격자 traffic이 남는다. endpoint를 비우고 회전된 bind token으로 observed source를 다시 증명한다.
- resume token을 성공 전에 회전시키거나 일부 상태만 갱신하면 어떤 복구 불능 상태가 생깁니까?
  - 모범답변: token은 무효화됐는데 player connection 이전이 실패하거나, player는 connected인데 새 credential이 없어지는 부분 commit이 생긴다. 원본은 모든 identity·deadline 검증과 새 token 준비 후 Room reconnect와 connection 상태를 한 owner 흐름에서 갱신한다.
- reconnect 직후 과거 tick의 snapshot이 아니라 현재 canonical state의 FULL을 보내야 하는 이유는 무엇입니까?
  - 모범답변: disconnect 동안 tick과 다른 player 상태가 진행됐고 client의 retained base는 신뢰할 수 없다. 새 UDP bind 뒤 현재 STOP/CONNECTED 상태에서 canonical hash를 다시 계산해 FULL로 새 base를 만든다.
- current FULL을 발행해도 과거 tick state hash가 바뀌지 않아야 하는 이유는 무엇입니까?
  - 모범답변: 과거 replay hash는 이미 끝난 tick의 immutable 증거다. reconnect는 현재 connectivity와 transport를 바꾸지만 과거 simulation을 재실행하거나 journal hash를 다시 쓰지 않는다.

### 30초 모범 답변

grace는 일시적 transport 손실과 논리적 player 종료를 분리하며, simulation과 같은 executed tick을 기준으로 해야 pause나 wall clock 조정에 흔들리지 않습니다. reconnect는 room·stable session·player·resume token·미접속 상태·`current_tick < deadline`을 모두 검증한 뒤, 새 connection으로 identity를 옮기고 provisional session과 옛 UDP binding을 폐기합니다. 그 commit 안에서 resume token과 bind token을 모두 회전해 replay를 막고, 새 UDP bind 뒤 현재 authoritative state를 FULL로 보냅니다. 실패 경로는 어떤 credential이나 player state도 부분 변경하지 않아야 합니다.

### 답변 핵심 키워드

disconnect grace, simulation-time deadline, stable identity, provisional identity retirement, token rotation, atomic commit, old endpoint invalidation, current FULL resync

### 백지 구현

#### 구현 목표

실행 tick 기반 grace 안에서 resume request를 검증하고, stable player/session을 새 connection으로 이전하면서 credential을 회전하는 transaction을 구현한다.

#### 인터페이스 또는 함수 시그니처

```cpp
struct PlayerResumeState {
    std::string player_id;
    std::string stable_session_id;
    bool connected;
    std::optional<std::uint64_t> disconnect_deadline_tick;
    std::uint64_t connection_id;
    std::uint64_t last_accepted_seq;
    std::uint64_t last_snapshot_seq;
};

struct ResumeRecord {
    std::string resume_token;
};

struct ProvisionalConnection {
    std::uint64_t connection_id;
    std::string provisional_session_id;
    std::optional<Endpoint> udp_endpoint;
    std::string bind_token;
    bool full_after_bind;
};

struct ReconnectRequest {
    std::string room_id;
    std::string stable_session_id;
    std::string resume_token;
};

enum class ReconnectCode {
    Reconnected,
    Invalid,
    Expired
};

struct ReconnectResult {
    ReconnectCode code;
    std::optional<std::string> new_resume_token;
    std::optional<std::string> new_bind_token;
    bool require_full_after_bind;
};

ReconnectResult reconnect(
    std::string_view expected_room_id,
    std::uint64_t executed_tick,
    PlayerResumeState& player,
    ResumeRecord& resume,
    ProvisionalConnection& connection,
    const ReconnectRequest& request,
    const std::function<std::string()>& new_token);
```

#### 입력과 출력

- 입력: room·player·resume·새 provisional connection 상태, reconnect request, 현재 executed tick, token generator
- 출력: 안정된 결과 코드, 성공 시 새 credential과 FULL 필요 여부

#### 반드시 만족해야 할 조건

- room ID, stable session, resume token이 모두 일치해야 한다.
- player가 이미 connected면 거부한다.
- disconnect deadline이 없으면 거부한다.
- `executed_tick >= deadline`이면 만료다.
- 모든 검증과 새 token 생성 준비가 끝나기 전 기존 상태를 변경하지 않는다.
- 성공 시 player의 stable ID·session·last accepted sequence는 보존한다.
- player connection ID는 새 connection으로 교체하고 connected 상태가 된다.
- provisional session은 stable session으로 대체되어 별도 active identity로 남지 않는다.
- 기존 UDP endpoint는 승계하지 않고 비운다.
- resume token과 bind token은 모두 새 값으로 회전한다.
- 이전 credential은 성공 직후 사용할 수 없어야 한다.
- `full_after_bind`를 설정해 현재 상태 FULL이 필요함을 표시한다.
- 모든 실패 경로는 입력 상태를 byte-for-byte 동일하게 유지한다.

#### 경계 조건

- deadline 바로 전 tick과 정확한 deadline tick
- player가 이미 connected
- resume record 없음 또는 빈 token
- room·session·token 각각 하나만 다른 요청
- token generator가 같은 값을 반환하거나 실패하는 경우의 정책
- provisional connection에 이미 UDP endpoint가 있는 경우
- reconnect 성공 직후 이전 token 재시도
- last snapshot sequence가 0 또는 큰 값인 경우

#### 실패 조건

- 일부 token만 회전하고 player 연결 이전이 실패하면 transaction 실패다.
- 실패 요청이 disconnect deadline을 지우거나 연장하면 실패다.
- 옛 UDP endpoint를 새 connection이 그대로 사용하면 실패다.
- 성공 후 provisional session이 active session registry에 남으면 실패다.
- reconnect가 과거 authoritative state를 다시 simulation에 적용하면 실패다.

#### 필요한 제약

- 25~30분 범위다.
- session registry와 token uniqueness 전역 검증은 콜백 또는 사전조건으로 단순화한다.
- 실제 snapshot serialization과 socket bind는 범위에서 제외한다.
- 상태 갱신의 strong exception guarantee 또는 사전 준비 방식 중 하나를 선택해 설명한다.

### 구현 후 자가 검증

- 정상 경로: deadline 전 성공 후 stable identity와 sequence 보존, credential 회전 확인
- 경계값: `deadline-1` 성공과 `deadline` 만료 확인
- 실패 경로: room·session·token mismatch와 connected 상태에서 모든 입력 상태 불변 확인
- 상태 변화: provisional session과 old endpoint가 제거되고 새 bind token만 남는지 확인
- invariant: 한 player는 active connection을 최대 하나만 갖는지 확인
- resource cleanup: 성공·실패 후 orphan provisional identity나 old credential이 남지 않는지 확인
- 중복 처리: 이전 resume token 재사용과 동시 두 reconnect 시도 정책 검토
- 보안: token 회전과 endpoint 재bind가 credential replay를 줄이는지 확인
- 요구사항 충족: reconnect가 simulation tick·과거 hash를 변경하지 않는지 확인

### 구현 후 설명할 것

1. grace를 executed tick으로 계산한 이유
   - 모범답변: simulation 진행과 같은 축을 써야 정확히 200 game tick이라는 계약이 된다. wall time을 쓰면 pause·overload에 따라 실제 허용 game progress가 달라진다.
2. stable identity와 transport identity를 분리한 이유
   - 모범답변: TCP connection은 끊기고 교체되지만 player와 stable session의 위치·점수·last sequence는 grace 동안 유지돼야 한다. 수명을 분리해야 새 connection이 기존 logical state를 하나만 인수할 수 있다.
3. credential 두 종류를 함께 회전한 이유
   - 모범답변: resume token은 takeover 권한, bind token은 새 UDP endpoint 권한을 나타낸다. 하나만 회전하면 old credential로 session 또는 data plane 중 하나를 재사용할 수 있다.
4. 실패 시 부분 mutation을 허용하지 않는 transaction 설계
   - 모범답변: room/session/player/token/deadline을 먼저 읽어 검증하고 새 token도 준비한 뒤, owner thread에서 player reconnect와 connection·resume record를 연속 갱신한다. 그 전의 모든 return 경로는 입력 state를 바꾸지 않는다.
5. reconnect 후 현재 FULL을 보내되 과거 replay hash는 보존하는 이유
   - 모범답변: 새 client replica에는 현재 authoritative projection을 새 base로 제공해야 하지만 reconnect는 과거 tick을 수정하는 game event가 아니다. 따라서 current canonical state로 FULL을 만들고 기존 TICK record와 hash는 그대로 둔다.

### 원본 확인 위치

- Thread: G11
- 커밋 메시지: `feat(session): add bounded reconnect grace and rotating credentials`
- 파일: `src/game.hpp`, `src/game.cpp`, `src/transport.hpp`, `src/transport.cpp`, `tests/g11.hpp`, `tests/g11.cpp`
- 함수·클래스·컴포넌트: `Server::reconnect`, `Room::reconnect`, `Room::evaluate_start_condition`, `Player::disconnect_deadline`, `Server::publish_snapshot`, resume record 저장소
- 관련 Thread: G03의 identity lifecycle, G09의 UDP bind credential, G10의 FULL resynchronization, G12의 disconnect 시 transport buffer 선해제
