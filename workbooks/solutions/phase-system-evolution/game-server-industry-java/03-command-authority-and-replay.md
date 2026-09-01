# 명령 정합성, 권위와 재현성

이 문서는 authoritative command admission과 결정적 replay를 다룬다. 입력을 "받았다"는 사실과 특정 tick에 "적용했다"는 사실을 구분하는 것이 두 문제의 공통 축이다.

---

## [G05 / feat: validate sequence and tick input ordering] 입력 sequence·tick window·abuse bound

> 통합 Thread: G06 / `feat: bound per-tick input validation attempts`

### 면접 질문

클라이언트가 UDP 재전송·중복·역순 전달을 일으킬 수 있을 때, per-player `seq`와 `target_tick`을 어떤 상태로 관리해야 합니까? 같은 sequence의 같은 payload, 같은 sequence의 다른 payload, 더 낮은 sequence, 너무 늦거나 이른 target tick을 각각 어떻게 처리하고 왜 accepted state를 보존해야 합니까?

꼬리 질문:

- sequence 범위가 1..2^64−1일 때 Java `long` 대신 `BigInteger`로 wire 정수를 읽은 이유는 무엇입니까?
  - 모범답변: signed `long`은 2^63 이상을 음수로 해석해 unsigned ordering을 뒤집는다. 원본은 Jackson의 exact integer를 `BigInteger`로 보존하고 sign과 bit length를 따로 검증한다.
- sequence gap을 기다리지 않고 더 높은 sequence를 받을 수 있게 한 trade-off는 무엇입니까?
  - 모범답변: packet loss가 있어도 최신 intent를 즉시 받아 responsiveness가 좋아지고 gap buffer·timeout이 필요 없다. 대신 빠진 sequence를 strict하게 적용하지 않으며 accepted journal과 highest-seq-wins 규칙이 결정성을 맡는다.
- 같은 target tick에 여러 accepted intent가 있으면 왜 가장 높은 sequence 하나를 선택하고 future intent는 남겨 둡니까?
  - 모범답변: 같은 tick에서는 가장 최신 logical intent 하나로 결과를 고정하고, 아직 due가 아닌 future work는 자신의 target까지 보존한다. 원본은 current target 항목만 iterator에서 제거한다.
- G06의 "활성 Player당 tick 사이 첫 4회 validation"은 parser schema·transport identity·direction·sequence 검증 중 어디에 놓여야 합니까?
  - 모범답변: 기본 frame/schema와 실제 transport session/player attribution 뒤, direction·sequence 같은 INPUT admission decode 전에 둔다. 그래야 malformed 비용도 세면서 foreign actor가 다른 quota를 소모하지 않는다.
- 잘못된 direction이나 sequence conflict도 rate allowance를 소비하지만 inactive·foreign identity는 소비하지 않는 이유는 무엇입니까?
  - 모범답변: 활성 player가 보낸 invalid 요청도 서버 검증 비용을 발생시키므로 같은 allowance 대상이다. inactive·foreign은 그 player에게 귀속되지 않으므로 quota를 줄이면 공격자가 identity claim으로 타인을 throttling할 수 있다.
- client가 보낸 position·score·elapsed time을 무시하는 authoritative 경계가 admission invariant와 어떤 관계가 있습니까?
  - 모범답변: admission은 direction·target·tag 같은 intent만 받고 위치·점수·시간은 server tick이 계산한다. sequence identity에도 server-owned 결과를 넣지 않아 client가 state를 직접 덮어쓰지 못한다.

### 30초 모범 답변

각 Player는 마지막으로 accepted된 sequence와 canonical intent를 보관합니다. 낮은 값은 stale, 같은 값의 같은 논리 payload는 idempotent duplicate, 같은 값의 다른 payload는 conflict이며 모두 queue와 accepted state를 바꾸지 않습니다. 새 sequence는 다음 실행 tick부터 +4까지의 window와 pending 64개 한도를 통과해야 하고, tick 실행 시 그 tick의 가장 높은 sequence만 적용합니다. abuse bound는 schema·실제 connection identity 확인 뒤, direction·sequence admission 전에 세어 공격자가 비싼 검증을 반복하지 못하게 하되 다른 Player의 quota는 건드리지 않게 합니다.

### 답변 핵심 키워드

unsigned 64-bit, exact integer, canonical intent, idempotency, duplicate, stale, conflict, target window, highest-seq-wins, bounded pending, validation ordering, per-player quota, authoritative state

### 백지 구현

#### 구현 목표

per-player INPUT admission과 per-tick 선택을 구현한다. sequence idempotency, target tick window, pending bound, G06 validation-attempt bound를 하나의 명시적 상태 머신으로 만든다.

#### 면접용 인터페이스

```java
import java.math.BigInteger;
import java.util.Optional;

public enum Direction {
    NORTH, SOUTH, EAST, WEST, STOP
}

public record Intent(
        BigInteger sequence,
        BigInteger targetTick,
        Direction direction,
        String tagTargetPlayerId) {}

public enum AdmissionCode {
    ACCEPTED,
    DUPLICATE,
    INPUT_STALE,
    SEQUENCE_CONFLICT,
    INPUT_LATE,
    INPUT_TOO_EARLY,
    INPUT_QUEUE_FULL,
    INPUT_RATE_EXCEEDED,
    MESSAGE_INVALID
}

public final class PlayerInputState {
    public static final int PENDING_LIMIT = 64;
    public static final int MAX_FUTURE_TICKS = 4;
    public static final int VALIDATION_LIMIT = 4;

    public AdmissionCode admit(Intent intent, BigInteger nextTickToExecute) {
        if (validationAttempts == VALIDATION_LIMIT)
            return AdmissionCode.INPUT_RATE_EXCEEDED;
        validationAttempts++; // 활성 actor의 invalid admission도 allowance를 소비한다.

        if (intent == null || nextTickToExecute == null
                || intent.sequence() == null || intent.targetTick() == null
                || intent.direction() == null
                || intent.sequence().signum() <= 0
                || intent.sequence().compareTo(MAX_UNSIGNED_64) > 0)
            return AdmissionCode.MESSAGE_INVALID;

        int order = intent.sequence().compareTo(lastAcceptedSequence);
        if (order < 0) return AdmissionCode.INPUT_STALE;
        if (order == 0)
            return intent.equals(lastAcceptedIntent)
                    ? AdmissionCode.DUPLICATE
                    : AdmissionCode.SEQUENCE_CONFLICT;
        if (intent.targetTick().compareTo(nextTickToExecute) < 0)
            return AdmissionCode.INPUT_LATE;
        if (intent.targetTick().compareTo(
                nextTickToExecute.add(BigInteger.valueOf(MAX_FUTURE_TICKS))) > 0)
            return AdmissionCode.INPUT_TOO_EARLY;
        if (pending.size() == PENDING_LIMIT)
            return AdmissionCode.INPUT_QUEUE_FULL;

        // 모든 rejection 검사가 끝난 한 지점에서 queue와 accepted identity를 commit한다.
        pending.addLast(intent);
        lastAcceptedSequence = intent.sequence();
        lastAcceptedIntent = intent;
        return AdmissionCode.ACCEPTED;
    }

    public Optional<Intent> selectForTick(BigInteger tick) {
        java.util.Objects.requireNonNull(tick);
        Intent selected = null;
        var iterator = pending.iterator();
        while (iterator.hasNext()) {
            Intent candidate = iterator.next();
            if (!candidate.targetTick().equals(tick)) continue;
            if (selected == null
                    || candidate.sequence().compareTo(selected.sequence()) > 0)
                selected = candidate;
            iterator.remove();
        }
        return Optional.ofNullable(selected);
    }

    public void onTickBoundary() {
        validationAttempts = 0;
    }

    public BigInteger lastAcceptedSequence() {
        return lastAcceptedSequence;
    }

    public int pendingCount() {
        return pending.size();
    }

    public int validationAttempts() {
        return validationAttempts;
    }

    private static final BigInteger MAX_UNSIGNED_64 =
            BigInteger.ONE.shiftLeft(64).subtract(BigInteger.ONE);
    private final java.util.ArrayDeque<Intent> pending = new java.util.ArrayDeque<>();
    private BigInteger lastAcceptedSequence = BigInteger.ZERO;
    private Intent lastAcceptedIntent;
    private int validationAttempts;
}
```

#### 입력과 출력

- 입력: 이미 connection/session/player identity가 검증된 활성 Player의 intent와 다음 실행 tick.
- 출력: admission code, 특정 tick에 적용할 최대 sequence intent.
- parser가 `sequence`를 1..2^64−1의 정확한 정수로 만들었다고 가정하되, null·범위 밖 입력 정책을 구현에 명시한다.

#### 반드시 만족해야 할 조건

- 활성 Player의 admission 시도는 tick 사이 최대 4회 validation을 허용하고 5번째는 `INPUT_RATE_EXCEEDED`다.
- validation counter는 방향·sequence에서 거절된 시도도 소비하고 4에서 포화된다.
- rate 초과 시 sequence·last intent·pending queue는 변하지 않는다.
- 더 낮은 sequence는 `INPUT_STALE`다.
- 같은 sequence와 같은 논리 payload는 `DUPLICATE`이며 상태를 바꾸지 않는다.
- 같은 sequence와 다른 논리 payload는 `SEQUENCE_CONFLICT`이며 상태를 바꾸지 않는다.
- 새 sequence의 target은 `nextTickToExecute`부터 `+4`까지 포함한다.
- window와 sequence를 통과해도 pending 64개를 넘으면 accepted state를 바꾸지 않는다.
- sequence gap은 허용한다.
- `selectForTick`은 그 tick의 accepted intent 중 sequence가 가장 높은 하나만 반환하고, 해당 tick의 나머지는 제거한다.
- future tick intent는 그대로 남긴다.
- tick boundary에서 validation counter를 0으로 되돌린다.

#### 경계 조건

- sequence 1, 2^64−1, 0, 음수, 2^64.
- 같은 sequence에 ignored 확장 field만 다른 경우는 canonical payload가 같다고 취급할지 명시한다.
- target이 nextTick−1, nextTick, nextTick+4, nextTick+5다.
- pending이 63·64개인 상태에서 새 unique sequence가 온다.
- 같은 tick에 sequence 3·7·6이 accepted된다.
- direction invalid 네 번 뒤 다섯 번째 valid intent가 온다.
- tick boundary 직후 retry한다.

#### 실패 조건

- duplicate·rejection이 last accepted sequence나 queue를 바꾼다.
- rate check가 identity 확인 전에 실행돼 다른 Player quota를 소비한다.
- 5번째 rate-exceeded intent가 sequence를 예약해 다음 tick retry가 conflict가 된다.
- target tick 선택 시 future intent까지 삭제한다.
- signed `long` overflow로 unsigned sequence 순서가 뒤집힌다.
- queue full에서 먼저 accepted state를 갱신한 뒤 rollback한다.

#### 필요한 제약

- 이 skeleton은 활성 Player에 대해서만 호출된다. inactive/foreign actor 처리와 schema 오류 우선순위는 외부 gateway가 담당한다.
- 동기화보다 상태 전이 정확성이 우선이며 owner-confined 사용을 가정해도 된다.
- TAG 거리·cooldown과 movement는 범위 밖이다. 단, client가 보낸 위치·점수는 이 상태에 포함하지 않는다.

### 구현 후 자가 검증

- [ ] 동일 sequence·동일 payload duplicate가 queue와 last accepted 값을 바꾸지 않는다.
- [ ] 동일 sequence·다른 payload conflict와 낮은 stale가 상태를 보존한다.
- [ ] nextTick와 +4는 허용되고 −1과 +5는 각각 late·too early다.
- [ ] pending 64개에서 65번째 unique input이 거절되고 기존 64개가 그대로다.
- [ ] 같은 tick의 여러 intent 중 최고 sequence 하나만 반환된다.
- [ ] future intent가 현재 tick 선택 뒤에도 남는다.
- [ ] invalid attempt를 포함한 4회 뒤 5번째가 rate exceeded다.
- [ ] rate exceeded가 sequence를 예약하지 않고 다음 tick retry가 가능하다.
- [ ] tick boundary 뒤 validation count가 0이다.
- [ ] 모든 rejection과 duplicate 경로가 O(1) 또는 명시한 자료구조 복잡도 안에 있다.

### 구현 후 설명할 것

1. sequence 비교·duplicate canonicalization·tick window의 검증 순서.
   - 모범답변: 활성 actor attempt를 센 뒤 exact range를 검증하고, last sequence와 typed logical intent로 stale·duplicate·conflict를 먼저 분류한 다음 target window와 queue capacity를 본다. 이 순서가 stable error와 retry 의미를 정한다.
2. last accepted state를 commit하는 단일 지점과 rejection rollback을 피한 방법.
   - 모범답변: 모든 검사가 성공한 뒤 pending append, last sequence, last intent를 연속 갱신한다. 그 전 return 경로는 읽기만 하므로 rollback할 부분 상태가 없다.
3. `BigInteger`를 사용해 unsigned wire 정수를 보존한 이유.
   - 모범답변: 0..2^64−1을 signed overflow 없이 exact 비교하고 JSON의 float 변환을 피한다. 허용 범위는 sign과 `2^64−1` 상한으로 명시한다.
4. rate allowance가 rejected validation도 세지만 foreign actor는 세지 않는 보안 경계.
   - 모범답변: 인증된 actor가 유발한 decode·admission 비용은 성공 여부와 무관하게 제한한다. attribution에 실패한 packet은 그 actor의 요청이 아니므로 quota 대신 identity error로 끝낸다.
5. per-tick 최고 sequence 선택과 gap 허용이 responsiveness·strict ordering에 주는 trade-off.
   - 모범답변: 최신 intent가 손실된 낮은 sequence를 기다리지 않고 같은 target에서 우선돼 realtime 반응성이 좋다. strict total ordering은 포기하므로 accepted history와 target tick 계약으로 재현성을 보완한다.

### 원본 확인 위치

- 대표 Thread: G05
- 커밋: `feat: validate sequence and tick input ordering`
- 통합 Thread: G06 / `feat: bound per-tick input validation attempts`
- 파일·컴포넌트: `src/main/java/arena/Room.java`, `Json.java`, `CompleteFrame.java`, `ArenaServer.java`
- 함수: `Room.accept`, `Room.tick`, `Room.beginInputValidation`, `Json.sequence`, `Json.targetTick`, `CompleteFrame.protocolError`
- 시나리오·테스트: `SequenceScenario.java`, `AuthorityScenario.java`, `RoomTest.frozenG06AuthorityAbuse`
- 관련 Thread: G03, G07, G09

---

## [G07 / feat: capture bounded replay and canonical tick hashes] Canonical state hash와 bounded replay

### 면접 질문

동일한 input 목록을 저장했는데도 replay 결과가 달라질 수 있는 이유는 무엇입니까? 결정적 replay를 위해 초기 상태, event admission boundary, player 정렬, 숫자·문자열 직렬화, line ending을 어떤 계약으로 고정해야 합니까?

꼬리 질문:

- `before_tick`과 `target_tick`을 따로 기록해야 하는 이유는 무엇입니까?
  - 모범답변: `before_tick`은 owner가 event를 accepted state에 넣은 admission boundary이고 `target_tick`은 client가 적용을 요청한 시점이다. 둘을 합치면 future input과 지연 admission의 순서를 재현할 수 없다.
- accepted됐지만 같은 tick의 더 높은 sequence에 supersede된 input도 log에 남겨야 하는 이유는 무엇입니까?
  - 모범답변: accepted event는 last sequence와 duplicate/conflict 상태를 바꾸므로 적용되지 않아도 이후 admission 결과에 영향을 준다. applied event만 남기면 원 실행의 state machine을 재현하지 못한다.
- canonical record를 hash만 저장하지 않고 record 자체도 TICK record에 남긴 장점은 무엇입니까?
  - 모범답변: hash는 불일치 여부만 주지만 record는 첫 divergence에서 player별 필드를 직접 diff하게 한다. serializer 계약과 simulation 오류를 빠르게 구분할 수 있다.
- artifact가 4MiB bound를 넘을 때 오래된 record를 조용히 잘라내지 않고 export를 "불완전"으로 실패시킨 이유는 무엇입니까?
  - 모범답변: 일부 record를 잃은 artifact가 완전한 replay처럼 보이면 잘못된 검증 결론을 만든다. 원본은 overflow를 latch하고 이후 capture를 중단하며 export만 fail-closed한다.
- replay verifier가 첫 divergence에서 멈추고 actual record/hash를 내는 이유는 무엇입니까?
  - 모범답변: 이후 mismatch는 첫 오류가 전파된 결과일 가능성이 높다. 원인에 가장 가까운 tick의 expected/actual과 canonical record가 최소하고 유용한 증거다.
- test-only fixture와 verifier를 production JAR에서 제외하는 것이 왜 시스템 경계상 중요합니까?
  - 모범답변: fixture가 owner state 주입·내부 조회 같은 production에 필요 없는 권한과 의존성을 노출할 수 있다. artifact contract는 유지하되 검증 도구는 test/runtime 경계 밖에 둬 공격 면과 배포 크기를 줄인다.

### 30초 모범 답변

replay 결정성은 event 내용뿐 아니라 초기 roster·spawn, admission 시점, tick 순서, canonical serialization이 모두 같아야 합니다. 그래서 HEADER와 accepted INPUT·LEFT를 owner order로 기록하고, 각 tick 뒤 정렬·필드·개행이 고정된 canonical record와 SHA-256을 남깁니다. `before_tick`은 서버가 intent를 언제 받아 상태에 넣었는지, `target_tick`은 언제 적용하려 했는지를 구분합니다. artifact bound를 넘으면 완전한 replay인 척하지 않고 overflow latch로 export를 실패시키며 gameplay hashing은 계속하는 것이 안전합니다.

### 답변 핵심 키워드

canonicalization, deterministic order, initial mapping, admission boundary, before_tick, target_tick, owner order, canonical bytes, state hash, overflow latch, first divergence, test/runtime boundary

### 백지 구현

#### 구현 목표

고정된 최대 byte 수 안에서 HEADER·INPUT·LEFT·TICK record를 append하는 replay journal을 구현한다. overflow가 발생하면 이후 export가 명시적으로 실패하고, close가 backing storage를 해제해야 한다.

#### 면접용 인터페이스

```java
import java.util.List;

public record PlayerInitial(String playerId, int slot, int spawnX, int spawnY) {}
public record AcceptedInput(
        int beforeTick,
        String playerId,
        String sequence,
        String targetTick,
        String direction,
        String targetPlayerId) {}

public final class BoundedReplayLog implements AutoCloseable {
    public BoundedReplayLog(int maxBytes) {
        if (maxBytes <= 0) throw new IllegalArgumentException("maxBytes must be positive");
        this.maxBytes = maxBytes;
        this.output = new java.io.ByteArrayOutputStream(Math.min(maxBytes, 4_096));
    }

    public void header(String roomId, List<PlayerInitial> players) {
        ensureOpen();
        if (headerWritten || output.size() != 0)
            throw new IllegalStateException("HEADER must be first and unique");
        List<PlayerInitial> ordered = new java.util.ArrayList<>(players);
        ordered.sort(java.util.Comparator.comparingInt(PlayerInitial::slot));
        StringBuilder json = new StringBuilder("{\"kind\":\"HEADER\",\"contract_version\":1,\"room_id\":")
                .append(quote(roomId)).append(",\"owner_epoch\":0,\"status\":\"RUNNING\",\"players\":[");
        for (int i = 0; i < ordered.size(); i++) {
            if (i != 0) json.append(',');
            PlayerInitial player = ordered.get(i);
            json.append("{\"player_id\":").append(quote(player.playerId()))
                    .append(",\"slot\":").append(player.slot())
                    .append(",\"spawn\":[").append(player.spawnX()).append(',')
                    .append(player.spawnY()).append("]}");
        }
        json.append("]}");
        headerWritten = true;
        append(json.toString());
    }

    public void accepted(AcceptedInput input) {
        ensureHeader();
        String json = "{\"kind\":\"INPUT\",\"before_tick\":" + input.beforeTick()
                + ",\"player_id\":" + quote(input.playerId())
                + ",\"seq\":" + canonicalInteger(input.sequence())
                + ",\"target_tick\":" + canonicalInteger(input.targetTick())
                + ",\"direction\":" + quote(input.direction())
                + ",\"owner_epoch\":0,\"tag_target_player_id\":"
                + (input.targetPlayerId() == null ? "null" : quote(input.targetPlayerId())) + "}";
        append(json);
    }

    public void left(int beforeTick, String playerId) {
        ensureHeader();
        append("{\"kind\":\"LEFT\",\"before_tick\":" + beforeTick
                + ",\"player_id\":" + quote(playerId) + "}");
    }

    public void tick(int tick, String canonicalRecord, String expectedHash) {
        ensureHeader();
        append("{\"kind\":\"TICK\",\"tick\":" + tick
                + ",\"expected_hash\":" + quote(expectedHash)
                + ",\"canonical_record\":" + quote(canonicalRecord) + "}");
    }

    public byte[] export() {
        ensureOpen();
        if (overflowed)
            throw new IllegalStateException("replay capture is incomplete");
        return output.toByteArray(); // backing storage와 독립된 copy다.
    }

    public int storedBytes() {
        return output == null ? 0 : output.size();
    }

    @Override
    public void close() {
        output = null;
    }

    private final int maxBytes;
    private java.io.ByteArrayOutputStream output;
    private boolean headerWritten;
    private boolean overflowed;

    private void append(String record) {
        ensureOpen();
        if (overflowed) return;
        byte[] line = (record + "\n").getBytes(java.nio.charset.StandardCharsets.UTF_8);
        // line 전체가 들어갈 때만 commit해 잘린 JSON을 만들지 않는다.
        if (line.length > maxBytes - output.size()) {
            overflowed = true;
            return;
        }
        output.writeBytes(line);
    }

    private void ensureOpen() {
        if (output == null) throw new IllegalStateException("replay capture is closed");
    }

    private void ensureHeader() {
        ensureOpen();
        if (!headerWritten) throw new IllegalStateException("HEADER is required first");
    }

    private static String canonicalInteger(String value) {
        return new java.math.BigInteger(value).toString();
    }

    private static String quote(String value) {
        java.util.Objects.requireNonNull(value);
        StringBuilder escaped = new StringBuilder(value.length() + 2).append('"');
        for (int i = 0; i < value.length(); i++) {
            char ch = value.charAt(i);
            switch (ch) {
                case '"' -> escaped.append("\\\"");
                case '\\' -> escaped.append("\\\\");
                case '\b' -> escaped.append("\\b");
                case '\f' -> escaped.append("\\f");
                case '\n' -> escaped.append("\\n");
                case '\r' -> escaped.append("\\r");
                case '\t' -> escaped.append("\\t");
                default -> {
                    if (ch < 0x20) escaped.append(String.format("\\u%04x", (int) ch));
                    else escaped.append(ch);
                }
            }
        }
        return escaped.append('"').toString();
    }
}
```

#### 입력과 출력

- 입력: owner order로 호출되는 초기 mapping, accepted input, LEFT, tick canonical record/hash.
- 출력: newline-delimited immutable byte copy.
- record encoding은 UTF-8 JSONL로 가정하되 field order·player order·최종 LF 규칙을 문서화한다.

#### 반드시 만족해야 할 조건

- HEADER는 첫 record로 정확히 한 번만 기록한다.
- 초기 players는 slot 또는 명시한 canonical order로 직렬화한다.
- INPUT은 `beforeTick`과 `targetTick`을 모두 보존한다.
- TICK은 tick 번호, canonical record, expected hash를 함께 보존한다.
- record append 전체가 maxBytes 안에 들어갈 때만 commit한다.
- 한 번 overflow하면 불완전 상태가 latch되고 이후 append는 gameplay와 무관하게 무시하거나 거절한다.
- overflow된 log의 `export`는 정상 artifact를 반환하지 않는다.
- 정상 `export`는 내부 buffer와 독립된 복사본이다.
- close 후 stored bytes는 0이고 export·append는 거절한다.
- canonical state hash 계산은 journal overflow와 독립적이라고 설명할 수 있어야 한다.

#### 경계 조건

- maxBytes에 정확히 맞는 마지막 record와 한 바이트 초과 record.
- 단일 record가 maxBytes보다 크다.
- target player가 null인 INPUT.
- canonical record 자체가 LF로 끝나는 경우 JSON string escaping.
- export 결과를 호출자가 수정한 뒤 다시 export한다.
- overflow 후 close, close 후 반복 close.

#### 실패 조건

- record 일부만 써서 마지막 JSON line이 잘린다.
- overflow 뒤 기존 artifact를 완전한 것으로 반환한다.
- accepted됐지만 superseded된 INPUT을 생략한다.
- player iteration이 hash map 순서에 의존한다.
- export가 내부 mutable array를 직접 노출한다.
- close 뒤 backing storage가 남는다.

#### 필요한 제약

- JSON serializer를 사용할 수 있지만 canonical field order와 newline 규칙을 테스트로 고정한다.
- hash 계산 함수는 제공된다고 가정해도 된다. 이 문제의 중심은 event boundary와 bounded artifact 수명이다.
- 확장 과제로 `DeterministicSimulation` 인터페이스를 제공받아 TICK record를 비교하고 첫 divergence를 반환하는 verifier를 추가할 수 있다.

### 구현 후 자가 검증

- [ ] HEADER가 첫 줄이고 player order가 실행마다 같다.
- [ ] INPUT에 before tick과 target tick이 서로 독립적으로 남는다.
- [ ] accepted 후 superseded된 input도 log에 존재한다.
- [ ] 정확히 maxBytes까지는 export되고 1바이트 초과 append는 부분 기록 없이 overflow된다.
- [ ] overflow artifact의 export가 명시적으로 실패한다.
- [ ] export byte 배열을 수정해도 내부 log가 변하지 않는다.
- [ ] 모든 line이 완전한 JSON이고 artifact 마지막이 LF다.
- [ ] close 후 stored bytes가 0이며 반복 close가 안전하다.
- [ ] append는 record byte 수에 선형이고 전체 공간은 maxBytes로 제한된다.
- [ ] verifier 확장 시 첫 불일치 tick에서 actual·expected hash와 canonical record를 낼 수 있다.

### 구현 후 설명할 것

1. canonical record의 정렬·필드·숫자·개행 계약을 고정한 이유.
   - 모범답변: hash와 JSONL 비교는 byte 단위이므로 의미가 같아도 순서·숫자 표기·LF가 다르면 결과가 달라진다. slot/player 순서, field order, 10진 exact integer, UTF-8과 record당 최종 LF를 고정한다.
2. accepted event와 applied event를 구분하고 admission boundary를 기록한 이유.
   - 모범답변: accepted는 pending과 idempotency state를 바꾸고 applied는 특정 tick에서 선택된 하나다. `before_tick`의 모든 accepted event를 owner order로 재생한 뒤 tick을 실행해야 같은 결과가 난다.
3. artifact overflow를 gameplay failure와 분리한 설계.
   - 모범답변: journal은 진단용 bounded artifact라 overflow가 authoritative Room을 중단해서는 안 된다. 다만 불완전 상태를 latch해 정상 export를 거부하고 gameplay hash 계산은 계속한다.
4. immutable export copy와 close 시 storage 해제의 수명 정책.
   - 모범답변: export는 `toByteArray` copy라 caller 수정이 내부 capture를 바꾸지 않는다. close는 backing reference를 없애고 이후 append/export를 거부하며 반복 close는 안전하다.
5. hash만 비교하는 것보다 canonical record를 함께 보관한 진단 가치.
   - 모범답변: 실제 record를 보면 위치·방향·score·connectivity 중 어떤 field가 처음 달라졌는지 알 수 있다. hash 함수나 serializer 문제도 simulation 규칙 오류와 분리할 수 있다.

### 원본 확인 위치

- Thread: G07
- 커밋: `feat: capture bounded replay and canonical tick hashes`
- 파일·컴포넌트: `src/main/java/arena/ReplayLog.java`, `Room.java`, `ArenaServer.java`
- 함수: `Room.canonicalRecord`, `Room.stateHash`, `ArenaServer.replayArtifact`, `ReplayLog.accepted`, `left`, `tick`, `bytes`, `close`, `storedBytes`
- verifier·도구: `src/test/java/arena/ReplayVerifier.java`, `ReplayScenario.java`, `ReplayTool.java`, `ReplayFixture.java`
- 관련 Thread: G05, G06, G08, G14
