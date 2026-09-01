# 복제와 장애 복구

이 문서는 snapshot 복제 상태 머신과 reconnect 상태 머신을 분리해 다룬다. 전자는 "클라이언트가 어느 상태를 확실히 갖고 있는가", 후자는 "어느 transport가 기존 Player를 다시 소유할 수 있는가"를 결정한다.

---

## [G08 / feat: publish acknowledged full and delta snapshots] FULL·DELTA base contract와 ACK 복구

> 통합 Thread: G10 / `feat: preserve snapshot ACK watermarks and schedule FULL recovery`

### 면접 질문

DELTA snapshot의 base를 "가장 최근에 보낸 snapshot"이 아니라 "서버가 retained 중이며 클라이언트가 유효하게 ACK한 snapshot"으로 정해야 하는 이유는 무엇입니까? ACK 손실, DELTA 손실, unknown·expired ACK, stale ACK, hash mismatch를 각각 어떻게 처리해야 합니까?

꼬리 질문:

- DELTA 하나를 놓쳤다고 바로 FULL을 보낼 필요가 없는 경우는 언제입니까?
- stale ACK가 retained 상태를 가리키더라도 acknowledged watermark를 낮추면 안 되는 이유는 무엇입니까?
- future sequence ACK를 미리 기억했다가 나중에 base로 쓰면 어떤 정합성 문제가 생깁니까?
- ACK의 optional state hash가 retained hash와 다를 때 simulation state를 바꾸지 않고 `resyncPending`만 latch하는 이유는 무엇입니까?
- explicit `resync_required`를 별도 즉시 메시지 없이 "다음 ordinary publication은 FULL"로 처리한 장점은 무엇입니까?
- retention 32와 periodic FULL 20은 어떤 메모리·복구 trade-off를 만듭니까?

### 30초 모범 답변

DELTA는 클라이언트가 실제로 가진 base에서만 재구성할 수 있으므로 서버가 보냈다는 사실만으로는 부족하고 유효 ACK와 retained 상태가 둘 다 필요합니다. ACK watermark는 단조 증가해야 하며, unknown·expired sequence나 hash mismatch는 watermark를 바꾸지 않고 `resyncPending`만 설정합니다. 다음 정상 publication이 FULL을 내보내 latch를 소비하면 별도 tick이나 sequence를 만들지 않고 복구할 수 있습니다. 놓친 DELTA 뒤에도 클라이언트가 더 오래된 ACK base를 갖고 있고 서버가 그 base를 보존한다면 다음 DELTA로 바로 따라잡을 수 있습니다.

### 답변 핵심 키워드

FULL, DELTA, retained base, acknowledged watermark, monotonic ACK, immutable snapshot, removal set, state hash, unknown ACK, expired ACK, stale ACK, resync latch, periodic FULL

### 백지 구현

#### 구현 목표

player projection의 FULL·DELTA snapshot을 생성하고, retained base 32개와 ACK watermark를 관리한다. 잘못된 ACK는 base를 채택하지 않고 다음 publication을 FULL로 예약한다.

#### 면접용 인터페이스

```java
import java.util.List;
import java.util.Map;
import java.util.Optional;

public record PlayerView(
        String playerId,
        int slot,
        int x,
        int y,
        String direction,
        int score,
        String connectivity) {}

public record SnapshotAck(
        long snapshotSequence,
        Optional<String> stateHash,
        boolean resyncRequired) {}

public sealed interface SnapshotMessage {
    long sequence();
    String stateHash();

    record Full(
            long sequence,
            String stateHash,
            String status,
            List<PlayerView> players) implements SnapshotMessage {}

    record Delta(
            long sequence,
            String stateHash,
            long baseSequence,
            List<PlayerView> changedPlayers,
            List<String> removedPlayerIds) implements SnapshotMessage {}
}

public enum AckResult {
    ADVANCED,
    STALE_IGNORED,
    INVALID_RESYNC_SCHEDULED,
    EXPLICIT_RESYNC_SCHEDULED,
    CLOSED
}

public final class SnapshotStream implements AutoCloseable {
    public static final int RETENTION = 32;
    public static final int PERIODIC_FULL_EVERY = 20;

    public SnapshotMessage next(
            Map<String, PlayerView> current,
            String roomStatus,
            String stateHash,
            boolean forceFull) {
        // 직접 구현
        return null;
    }

    public AckResult acknowledge(SnapshotAck ack) {
        // 직접 구현
        return null;
    }

    public long acknowledgedSequence() {
        // 직접 구현
        return 0;
    }

    public boolean resyncPending() {
        // 직접 구현
        return false;
    }

    @Override
    public void close() {
        // 직접 구현
    }
}
```

#### 입력과 출력

- 입력: 현재 visible player map, room status, canonical state hash, force flag, client ACK.
- 출력: 순차 sequence를 갖는 FULL 또는 DELTA, ACK 처리 결과.
- player rows와 removed IDs는 `playerId` 오름차순으로 직렬화한다고 정한다.

#### 반드시 만족해야 할 조건

- sequence는 publication마다 정확히 1씩 증가한다.
- FULL은 base가 없거나 force, periodic, `resyncPending` 중 하나일 때 생성한다.
- DELTA base는 현재 acknowledged sequence가 retained map에 있을 때만 사용한다.
- retained state는 이후 current map 변경과 무관한 immutable copy다.
- DELTA에는 base와 달라진 player와 base에서 사라진 ID만 들어간다.
- retention은 최근 32개를 넘지 않는다.
- 유효 ACK는 retained sequence와 optional hash가 일치하고 현재 watermark보다 새로울 때만 advance한다.
- stale ACK는 watermark를 낮추지 않는다.
- unknown·expired·hash mismatch ACK는 watermark를 바꾸지 않고 resync를 예약한다.
- explicit resync flag도 resync를 예약한다.
- resync latch는 다음 FULL publication에서 소비된다.
- ACK 처리는 simulation·input state·publication sequence를 직접 바꾸지 않는다.
- close 후 retained state와 latch를 정리한다.

#### 경계 조건

- 최초 publication, 20번째 periodic FULL, 33번째 retention eviction.
- DELTA 2를 잃었지만 ACK 1 base가 retained된 상태에서 DELTA 3을 받는다.
- future 999 ACK, 이미 eviction된 ACK, 현재보다 낮은 retained ACK.
- sequence는 맞지만 hash가 다르다.
- client가 local base를 잃어 `resyncRequired=true`를 보낸다.
- player 변경 0개, 모든 player 삭제, player 1개 변경.
- close 직전 ACK와 publication.

#### 실패 조건

- 마지막 전송 sequence를 ACK 없이 base로 사용한다.
- future ACK를 저장했다가 나중에 유효 base로 승격한다.
- stale ACK가 watermark를 rollback한다.
- hash mismatch가 watermark를 advance한다.
- current mutable map을 retained 상태로 그대로 보관한다.
- FULL 뒤 `resyncPending`이 계속 남아 모든 publication이 FULL이 된다.
- close 후 retained map이 남는다.

#### 필요한 제약

- state hash 계산은 외부에서 제공한다.
- network drop·reorder는 stream 밖에서 발생한다. 이 구현은 ACK와 retained state만 보고 결정한다.
- 깊은 복사의 범위와 비용을 설명해야 한다. immutable value 객체를 쓰는 해법도 가능하다.

### 구현 후 자가 검증

- [ ] 첫 snapshot은 FULL이고 base가 없다.
- [ ] valid ACK 뒤 변경된 player만 포함한 DELTA가 해당 base를 가리킨다.
- [ ] unchanged player는 DELTA에 들어가지 않고 삭제 ID는 정렬된다.
- [ ] 33개 publication 후 retained count가 32다.
- [ ] dropped DELTA 뒤에도 오래된 acknowledged base로 다음 DELTA를 적용할 수 있다.
- [ ] unknown·expired·hash mismatch ACK가 watermark를 바꾸지 않고 resync를 예약한다.
- [ ] stale ACK가 watermark를 낮추지 않는다.
- [ ] 다음 publication 한 번만 FULL이 되고 latch가 해제된다.
- [ ] ACK가 simulation state와 sequence를 직접 바꾸지 않는다.
- [ ] close 후 retained count 0, resync false다.
- [ ] DELTA 계산은 player 수에 선형이고 retention 메모리는 32개 snapshot으로 제한된다.

### 구현 후 설명할 것

1. ACK된 retained snapshot만 base가 될 수 있는 이유.
2. watermark 단조성과 invalid ACK의 상태 전이를 분리한 방법.
3. 즉시 별도 FULL 메시지 대신 다음 publication을 FULL로 예약한 이유.
4. immutable retention의 메모리 비용과 delta 생성 안정성 trade-off.
5. retention 32·periodic FULL 20 같은 정책 값을 일반화할 때 고려할 지표.

### 원본 확인 위치

- 대표 Thread: G08
- 커밋: `feat: publish acknowledged full and delta snapshots`
- 통합 Thread: G10 / `feat: preserve snapshot ACK watermarks and schedule FULL recovery`
- 파일·컴포넌트: `src/main/java/arena/SnapshotStream.java`, `SnapshotStream.Retained`
- 함수: `SnapshotStream.next`, `SnapshotStream.acknowledge`, `close`
- 시나리오·테스트: `src/test/java/arena/SnapshotScenario.java`, `AckScenario.java`, `SnapshotStreamTest.java`
- 관련 Thread: G07, G11, G12

---

## [G11 / feat(java): preserve sessions across bounded reconnect grace] Resume credential rotation과 즉시 FULL 복구

### 면접 질문

TCP 연결이 끊긴 Player를 즉시 LEFT로 만들지 않고 200 tick의 reconnect grace 동안 recoverable session으로 보존할 때 어떤 상태는 유지하고 어떤 상태는 폐기해야 합니까? 성공한 reconnect가 기존 Player를 새 connection으로 넘겨받으면서도 old token·old UDP endpoint·old snapshot base를 재사용하지 못하게 하려면 어떤 순서로 처리해야 합니까?

꼬리 질문:

- disconnect 시 pending input을 지우고 direction을 STOP으로 바꾸지만 position·score·last accepted sequence를 보존한 이유는 무엇입니까?
- reconnect 요청용 provisional session을 성공 시 retire하지 않으면 어떤 registry leak·identity ambiguity가 생깁니까?
- resume token을 성공할 때마다 rotate하고 이전 token을 즉시 무효화해야 하는 이유는 무엇입니까?
- grace 만료를 "deadline next tick"으로 표현할 때 199/200 경계의 off-by-one을 어떻게 테스트합니까?
- fresh UDP bind 뒤 첫 snapshot이 과거 base가 없는 immediate FULL이어야 하는 이유는 무엇입니까?
- 거절된 reuse·expired token 시도가 authoritative state나 recovery registry를 바꾸지 않아야 하는 이유는 무엇입니까?

### 30초 모범 답변

disconnect는 transport 소유권 상실이므로 old TCP·UDP endpoint, pending input, snapshot stream은 폐기하고 Player는 DISCONNECTED·STOP으로 둡니다. 하지만 grace 안의 재접속을 위해 identity, position, score, last accepted sequence와 recoverable session은 유지합니다. 성공한 RECONNECT는 token을 한 번 소비해 새 connection으로 binding을 옮기고 token을 rotate하며 provisional session과 old endpoints를 제거합니다. 새 UDP bind 뒤에는 client replica를 신뢰할 수 없으므로 현재 상태의 FULL을 tick 증가 없이 즉시 보냅니다.

### 답변 핵심 키워드

disconnect grace, recoverable session, transport vs gameplay state, one-time resume token, rotation, provisional session, expiry boundary, old endpoint invalidation, pending input cleanup, immediate FULL, no tick

### 백지 구현

#### 구현 목표

disconnect된 session을 bounded grace 동안 보존하고 one-time resume token으로 새 connection에 takeover시키는 registry를 구현한다. 게임 상태 보존과 transport 상태 폐기를 분리한다.

#### 면접용 인터페이스

```java
import java.util.Optional;
import java.util.function.Supplier;

public record PlayerState(
        String playerId,
        int x,
        int y,
        int score,
        String lastAcceptedSequence,
        String direction,
        String connectivity) {}

public record ActiveSession(
        String sessionId,
        String connectionId,
        String playerId) {}

public record RecoverableSessionView(
        String sessionId,
        String playerId,
        int expiryNextTick,
        PlayerState player) {}

public sealed interface ReconnectResult {
    record Resumed(
            ActiveSession active,
            String rotatedResumeToken,
            PlayerState player,
            boolean requiresFreshUdpBind,
            boolean requiresImmediateFull) implements ReconnectResult {}

    record Rejected(String code) implements ReconnectResult {}
}

public final class ResumeRegistry implements AutoCloseable {
    public ResumeRegistry(int graceTicks, int maxRecoverable, Supplier<String> tokenSource) {
        // 직접 구현
    }

    public String onDisconnect(ActiveSession session, PlayerState current, int nextTick) {
        // 직접 구현
        return null;
    }

    public ReconnectResult reconnect(
            ActiveSession provisional,
            String resumeToken,
            int nextTick) {
        // 직접 구현
        return null;
    }

    public void expireAt(int nextTick) {
        // 직접 구현
    }

    public Optional<RecoverableSessionView> recoverable(String sessionId) {
        // 직접 구현
        return Optional.empty();
    }

    @Override
    public void close() {
        // 직접 구현
    }
}
```

#### 입력과 출력

- 입력: disconnect 시점의 active session·Player state·다음 실행 tick, provisional session과 resume token, tick 경계 만료 호출.
- 출력: 새 resume token, reconnect 성공·거절, 보존된 Player state.
- UDP endpoint와 snapshot stream은 boolean 요구사항으로만 표현하고 실제 객체 생성은 외부 계층에 맡긴다.

#### 반드시 만족해야 할 조건

- disconnect 시 보존된 Player는 `DISCONNECTED`, direction `STOP`이다.
- identity, position, score, last accepted sequence는 보존한다.
- pending input, old UDP endpoint, old snapshot history는 registry 밖에서 폐기해야 한다는 계약을 결과로 드러낸다.
- expiry는 `disconnect nextTick + graceTicks` 경계를 일관되게 사용한다.
- 유효 token은 한 번의 성공한 reconnect에서만 소비된다.
- 성공 시 새 active binding을 만들고 token을 rotate한다.
- 성공 시 provisional session을 retire해야 한다.
- 이전 token과 old connection/endpoint는 즉시 무효다.
- 성공은 game tick을 실행하거나 Player state를 재초기화하지 않는다.
- fresh UDP bind와 immediate FULL이 필요하다는 결과를 반환한다.
- 재사용·unknown·expired token 거절은 Player·registry 상태를 바꾸지 않는다.
- recoverable registry는 설정된 최대 수를 넘지 않는다.
- close 후 recoverable session과 credential 수는 0이다.

#### 경계 조건

- disconnect 직후 reconnect.
- grace 마지막 허용 tick과 바로 다음 만료 tick.
- 성공한 token을 즉시 재사용한다.
- 첫 reconnect 성공 뒤 replacement connection도 tick 전 끊기고 rotated token으로 다시 reconnect한다.
- old UDP socket이 계속 packet을 보낸다.
- recoverable capacity가 가득 찬 상태에서 추가 disconnect가 발생한다.
- expire와 reconnect가 같은 owner iteration에 인접한다.

#### 실패 조건

- reconnect 성공 후 old token이 다시 성공한다.
- provisional session이 registry에 남는다.
- expired token 거절이 Player를 LEFT로 만들거나 state를 바꾼다.
- old snapshot base를 새 session에 물려 DELTA부터 보낸다.
- disconnect 전에 accepted된 pending input이 나중에 적용된다.
- close 후 credential 또는 recoverable session이 남는다.

#### 필요한 제약

- registry mutation은 owner thread에서 직렬화된다고 가정한다.
- token 값 자체를 view·오류·로그로 반환하지 않는다. 성공 결과의 rotated token은 transport 전달을 위한 축소 모델이다.
- 실제 `SnapshotStream`·UDP registry 생성과 old connection close는 호출자 책임이며, 호출 순서를 설명해야 한다.

### 구현 후 자가 검증

- [ ] disconnect 결과가 identity·position·score·last sequence를 보존하고 connectivity/direction만 바꾼다.
- [ ] grace 안의 유효 token이 성공하고 tick·게임 상태를 바꾸지 않는다.
- [ ] 성공 직후 old token 재사용이 거절된다.
- [ ] rotated token으로 다음 reconnect가 가능하다.
- [ ] grace 경계 직전은 성공하고 경계 후는 expired 정책대로 거절된다.
- [ ] rejected reconnect 전후 Player state와 registry view가 같다.
- [ ] 성공 후 provisional session과 old binding이 남지 않는다.
- [ ] 결과가 fresh UDP bind와 immediate FULL을 요구한다.
- [ ] capacity와 close 후 recoverable·credential count가 0이다.
- [ ] lookup·token 검증은 평균 O(1), expire 비용은 선택한 자료구조에 맞게 설명된다.

### 구현 후 설명할 것

1. disconnect에서 gameplay state와 transport state를 다르게 처리한 기준.
2. resume token 일회성·rotation과 replay 공격 방지.
3. provisional session takeover의 원자적 commit 지점.
4. grace deadline의 off-by-one 정의와 테스트 방법.
5. reconnect 직후 DELTA가 아니라 current FULL을 보내는 정합성 이유.

### 원본 확인 위치

- Thread: G11
- 커밋: `feat(java): preserve sessions across bounded reconnect grace`
- 파일·컴포넌트: `src/main/java/arena/ArenaServer.java`의 `sessions`, `recoverableSessions`, `SnapshotStream.java`
- 테스트·시나리오: `src/test/java/arena/ReconnectScenario.java`, `src/test/resources/G11.json`
- 클라이언트 경계: `TcpClient.reconnectFrom`
- 관련 Thread: G03, G08, G09, G10, G12
