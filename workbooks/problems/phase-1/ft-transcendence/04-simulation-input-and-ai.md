# 결정적 시뮬레이션·입력 순서·게임 AI 면접 워크북

이 문서는 서버 권위형 게임에서 같은 입력이 같은 결과를 내도록 만들고, 지연·중복·악성 입력을 차단하며, AI 판단까지 재현 가능하게 구성하는 문제를 다룬다.

---

## IM-16. [Thread 09 / `feat(game): 서버 주도 퐁 물리 갱신`, `test(game): 결정적 simulation 검증`, `test(game): versioned match replay fixture 추가`] 서버 권위형 결정적 시뮬레이션

### 면접 질문

클라이언트가 공 위치와 점수를 계산하게 두지 않고 서버가 고정 tick으로 상태를 갱신하도록 만든 이유는 무엇인가요? 같은 초기 상태와 입력 기록으로 1,000 tick 뒤 동일한 checksum을 얻는 replay 검증이 어떤 회귀를 잡는지도 설명해 주세요.

꼬리 질문:

- 부동소수점 연산이 플랫폼·최적화 차이에 민감할 때 어떤 대안을 고려할 수 있습니까?
- 한 tick 안에 공이 벽이나 패들을 통과하는 tunneling을 어떻게 줄입니까?
- 충돌, 득점, 속도 증가의 적용 순서가 왜 프로토콜의 일부처럼 중요합니까?
- replay fixture를 바꿔도 되는 변경과 버그로 봐야 하는 변경을 어떻게 구분합니까?

### 30초 모범 답변

승패에 영향을 주는 상태는 서버가 권위 있게 계산해야 클라이언트 조작과 서로 다른 결과를 막을 수 있습니다. 시뮬레이션은 외부 시계나 소켓을 참조하지 않는 순수 step 함수로 만들고, 고정 시간 간격과 명확한 충돌 순서를 사용합니다. 동일한 초기 상태·입력 로그를 장시간 재생한 뒤 checksum을 비교하면 사소한 상수나 연산 순서 변경으로 누적되는 물리 회귀도 잡을 수 있습니다. 의도적인 규칙 변경은 replay 버전을 올리고 기대 결과를 명시적으로 갱신해야 합니다.

### 답변 핵심 키워드

server authoritative · pure step · fixed tick · deterministic replay · checksum · 충돌 순서 · anti-cheat · versioned rules · 누적 오차

### 백지 구현

**구현 목표**

축소된 2차원 Pong 상태에서 패들 이동, 벽 반사, 패들 충돌, 득점을 한 tick씩 계산하는 순수 함수를 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
type Side = "left" | "right";
type Direction = -1 | 0 | 1;

interface PongState {
  tick: number;
  phase: "playing" | "finished";
  leftScore: number;
  rightScore: number;
  winnerSide: Side | null;
  leftPaddleY: number;
  rightPaddleY: number;
  ballX: number;
  ballY: number;
  ballVx: number;
  ballVy: number;
}

interface PongInputs {
  left: Direction;
  right: Direction;
}

export function stepPong(
  state: Readonly<PongState>,
  inputs: PongInputs
): PongState {
  // 직접 구현
  throw new Error("not implemented");
}
```

**입력과 출력**

- 입력: 이전 상태와 양쪽 방향 입력
- 출력: 다음 tick의 새 상태

**반드시 만족해야 할 조건**

- 입력 상태를 직접 변경하지 않는다.
- 패들은 경기장 범위를 벗어나지 않는다.
- 위·아래 벽에서는 세로 속도 방향이 바뀐다.
- 패들 충돌은 공이 해당 방향으로 이동 중일 때만 적용한다.
- 득점 시 점수는 정확히 한 번 증가하고 공은 정해진 초기 위치로 재설정된다.
- 승리 점수에 도달하면 `finished`와 승자를 설정하고 이후 step에서 결과가 바뀌지 않는다.
- 같은 상태와 입력은 항상 구조적으로 같은 결과를 만든다.

**경계 조건**

- 공이 벽 경계에 정확히 닿은 경우
- 공이 패들 모서리에 닿은 경우
- 한 tick 이동량이 충돌 영역보다 큰 경우
- 득점과 승리 점수 도달이 같은 tick에 발생
- 이미 종료된 상태에 입력이 들어옴

**실패 조건**

- NaN·무한대 상태
- 허용 범위 밖의 방향 값
- 물리 상수가 유효하지 않음

**필요한 제약**

- 랜덤, 현재 시각, 전역 mutable 상태를 사용하지 않는다.
- 구현 전에 충돌·득점의 처리 순서를 주석 또는 짧은 문서로 고정한다.
- 연산량은 한 tick당 O(1)이어야 한다.

### 구현 후 자가 검증

- [ ] 입력 객체와 상태 객체가 변경되지 않는다.
- [ ] 동일 입력 로그를 여러 번 재생해 완전히 같은 최종 상태가 나온다.
- [ ] 벽·패들 경계값에서 공이 반복 반전하거나 끼지 않는다.
- [ ] 득점이 한 tick에 두 번 반영되지 않는다.
- [ ] 종료 뒤 점수와 승자가 변하지 않는다.
- [ ] 1,000 tick replay의 안정적인 직렬화와 checksum 전략을 설명할 수 있다.
- [ ] tunneling 가능성과 축소 문제에서의 허용 범위를 설명할 수 있다.

### 구현 후 설명할 것

1. 서버 권위형 모델이 신뢰 경계를 단순화하는 방식
2. 순수 step 함수와 I/O를 분리한 이유
3. 충돌 처리 순서를 고정해야 하는 이유
4. replay checksum이 일반 단위 테스트를 보완하는 지점
5. 규칙 변경 시 replay 버전과 호환성을 관리하는 방법

### 원본 확인 위치

- Thread 09
- 커밋: `feat(game): 서버 주도 퐁 물리 갱신`
- 커밋: `test(game): 결정적 simulation 검증`
- 커밋: `test(game): versioned match replay fixture 추가`
- `apps/api/src/game/pongSimulation.ts`
  - `PongSimulation`
  - `PongSimulationState`
  - `PongSimulationInputs`
  - `initialState`
  - `cloneState`
  - `step`
- `apps/api/src/game/pongSimulation.test.ts`
- `apps/api/src/game/fixtures/replay-v1.json`
- 관련 Thread: 10, 13, 14

---

## IM-17. [Thread 14 / `feat(game): room별 input sequence 중복을 차단`, `feat(game): 입력 순서와 rate limit 보호`] 입력 순서 보장, 중복 제거, 사용자 단위 token bucket

### 면접 질문

실시간 입력에 room별 sequence gate와 사용자 단위 rate limit을 함께 둔 이유는 무엇인가요? 오래된 sequence를 토큰 차감 전에 거부한 순서가 왜 중요한지도 설명해 주세요.

꼬리 질문:

- sequence를 사용자 단위로만 추적하면 새 방에서 어떤 문제가 생깁니까?
- 방 단위 token bucket만 두면 방을 빠르게 바꾸는 사용자가 어떻게 제한을 우회할 수 있습니까?
- keyup 이벤트를 놓쳐 패들이 계속 움직이는 문제는 서버와 클라이언트 중 어디서 보완합니까?
- 재접속·계정 정지·연결 종료 시 gate 상태를 언제 해제해야 합니까?

### 30초 모범 답변

sequence는 네트워크 재전송이나 순서 뒤바뀜으로 과거 입력이 현재 상태를 덮는 것을 막고, room을 키에 포함해 새 경기의 sequence가 이전 경기와 충돌하지 않게 합니다. rate limit은 사용자 단위 token bucket으로 두어 방을 바꿔도 우회할 수 없게 합니다. 오래된 입력은 이미 의미가 없으므로 토큰을 쓰기 전에 거부해야 공격자가 stale packet만으로 정상 입력 예산을 소모시키지 못합니다. 연결 종료나 사용자 폐기 시 모든 관련 키를 정리합니다.

### 답변 핵심 키워드

monotonic sequence · room-scoped ordering · user-wide token bucket · stale-before-rate-limit · replay 방지 · stuck input · lifecycle cleanup · bounded state

### 백지 구현

**구현 목표**

사용자·방별 최신 sequence와 사용자별 token bucket을 결합해 입력을 `accepted`, `stale`, `rate_limited`로 판정하는 gate를 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
type InputDecision = "accepted" | "stale" | "rate_limited";

export class InputGate {
  constructor(options: {
    capacity: number;
    refillPerSecond: number;
    now?: () => number;
  }) {}

  accept(userId: string, roomId: string, sequence: number): InputDecision {
    // 직접 구현
    throw new Error("not implemented");
  }

  releaseRoom(userId: string, roomId: string): void {
    // 직접 구현
  }

  releaseUser(userId: string): void {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: 사용자 ID, 방 ID, 0 이상의 sequence
- 출력: 입력 적용 가능 여부를 나타내는 결정

**반드시 만족해야 할 조건**

- 같은 사용자·방에서 마지막으로 수락한 sequence 이하를 stale로 거부한다.
- 다른 방의 sequence 상태는 서로 영향을 주지 않는다.
- rate limit 예산은 사용자 전체에서 공유한다.
- stale 입력은 토큰을 소비하지 않는다.
- rate-limited 입력은 최신 sequence로 기록되지 않는다.
- 토큰은 경과 시간에 따라 capacity를 넘지 않는 범위에서 보충된다.
- 사용자 해제 시 sequence와 bucket 상태가 모두 제거된다.

**경계 조건**

- 첫 sequence가 0인 경우
- 같은 sequence 반복
- sequence가 크게 건너뜀
- refill 직전·직후 경계 시각
- 시간이 뒤로 가거나 매우 크게 점프
- 같은 사용자가 여러 방에서 번갈아 입력

**실패 조건**

- 음수·소수·safe integer 초과 sequence
- 잘못된 capacity 또는 refill 설정
- 빈 식별자

**필요한 제약**

- 현재 시각은 monotonic clock으로 주입한다.
- 상태 크기는 활성 사용자·방 수에 비례하고 lifecycle cleanup이 가능해야 한다.
- 한 입력 판정은 평균 O(1)이어야 한다.

### 구현 후 자가 검증

- [ ] 오래된 입력이 토큰을 줄이지 않는다.
- [ ] rate-limited 입력을 나중에 같은 sequence로 재시도할 수 있다.
- [ ] 새 방의 sequence 0이 이전 방 때문에 거부되지 않는다.
- [ ] 여러 방을 오가도 사용자 전체 속도 제한을 우회하지 못한다.
- [ ] refill이 capacity를 초과하지 않는다.
- [ ] release 뒤 사용자 관련 상태가 남지 않는다.
- [ ] 메모리 증가 경로와 cleanup 호출 지점을 설명할 수 있다.

### 구현 후 설명할 것

1. ordering key에 room ID가 필요한 이유
2. rate-limit key는 user ID인 이유
3. stale 검사와 토큰 차감 순서가 보안에 미치는 영향
4. rate-limited sequence를 기록하지 않는 이유
5. 브라우저 keyup 누락과 서버의 idle/cleanup 정책을 함께 다루는 방법

### 원본 확인 위치

- Thread 14
- 커밋: `feat(game): room별 input sequence 중복을 차단`
- 커밋: `feat(game): 입력 순서와 rate limit 보호`
- 커밋: `test(game): input gate 제한 검증`
- 커밋: `feat(game): heartbeat와 input gate를 GameHub에 연결`
- `apps/api/src/game/inputGate.ts`
  - `InputGate`
  - `InputGateDecision`
- `apps/web/src/game/gameInput.ts`
  - `directionForKey`
  - `isEditableTarget`
- `apps/api/src/gameHub.ts`
- 관련 Thread: 08, 09, 12

---

## IM-18. [Thread 10 / `refactor(game): 결정적 정수 난수 생성기 추가`, `refactor(game): rating 기반 Pong AI 정책 분리`] 재현 가능한 PRNG와 난이도 기반 AI 정책

### 면접 질문

AI의 반응 오차와 판단 주기를 `Math.random()`으로 정하지 않고 방 ID 기반의 결정적 PRNG로 만든 이유는 무엇인가요? 플레이어 rating에 따라 AI profile을 바꾸면서도 테스트 가능성과 공정성을 유지하는 방법을 설명해 주세요.

꼬리 질문:

- 같은 seed로 항상 같은 AI가 움직이면 사용자가 패턴을 학습할 수 있는데 어떤 trade-off가 있습니까?
- PRNG 상태를 snapshot에 포함하면 어떤 복구·replay 이점이 있습니까?
- 공의 벽 반사를 고려해 도착 Y를 예측할 때 반복 시뮬레이션과 수학적 folding 중 무엇을 선택하겠습니까?
- rating 경계에서 난이도가 갑자기 튀는 문제는 어떻게 완화합니까?

### 30초 모범 답변

결정적 PRNG를 쓰면 동일한 방 seed와 입력에서 AI 판단까지 재현돼 실패를 replay하고 테스트할 수 있습니다. 난이도는 rating 구간별로 반응 간격, 추적 오차, 최대 이동 같은 policy 값만 바꾸고, 게임 규칙과 PRNG 엔진은 공유합니다. PRNG 상태를 함께 보존하면 재접속이나 replay 중에도 난수 소비 순서가 어긋나지 않습니다. 결정성은 예측 가능성을 높일 수 있으므로 seed에 서버가 정한 경기별 entropy를 포함하되 결과 재현에 필요한 값은 기록합니다.

### 답변 핵심 키워드

deterministic PRNG · seed · PRNG snapshot · policy/data 분리 · rating profile · reproducible AI · 난수 소비 순서 · 예측 가능성 trade-off

### 백지 구현

**구현 목표**

고정 seed에서 재현 가능한 정수 PRNG와, rating에 따라 반응 간격·오차 범위를 선택하는 작은 AI 정책 함수를 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
export interface PrngSnapshot {
  state: number;
}

export class IntegerPrng {
  constructor(seed: number) {}

  nextUint32(): number {
    // 직접 구현
    throw new Error("not implemented");
  }

  nextInt(minInclusive: number, maxInclusive: number): number {
    // 직접 구현
    throw new Error("not implemented");
  }

  snapshot(): PrngSnapshot {
    // 직접 구현
    throw new Error("not implemented");
  }
}

export interface AiProfile {
  decisionEveryTicks: number;
  aimError: number;
  deadZone: number;
}

export function profileForRating(rating: number): AiProfile {
  // 직접 구현
  throw new Error("not implemented");
}

export function choosePaddleDirection(input: {
  paddleY: number;
  predictedBallY: number;
  profile: AiProfile;
  prng: IntegerPrng;
}): -1 | 0 | 1 {
  // 직접 구현
  throw new Error("not implemented");
}
```

**입력과 출력**

- 입력: seed, 정수 범위, rating, 패들과 공의 예상 위치
- 출력: 재현 가능한 난수, AI profile, 이동 방향

**반드시 만족해야 할 조건**

- 같은 seed의 두 인스턴스는 같은 수열을 만든다.
- snapshot 상태로 복구했을 때 다음 수열이 이어진다.
- `nextInt` 결과는 양 끝을 포함한 범위 안이다.
- 잘못된 범위는 거부한다.
- 높은 rating profile은 낮은 rating보다 적어도 하나의 난이도 축에서 더 강해야 한다.
- dead zone 안에서는 불필요하게 방향을 바꾸지 않는다.
- AI 판단은 외부 시각과 전역 난수에 의존하지 않는다.

**경계 조건**

- seed 0과 최대 32비트 값
- `min === max`
- 음수 범위
- rating 구간 경계값
- 공 예상 위치와 패들 위치가 정확히 같은 경우

**실패 조건**

- 범위 역전
- safe integer를 벗어나는 범위
- NaN rating 또는 위치

**필요한 제약**

- 암호학적 보안은 목표가 아니며 게임 재현성이 목표임을 명시한다.
- 난수 알고리즘 변경은 replay 호환성에 영향을 줄 수 있다.
- 한 번의 AI 판단은 O(1)이어야 한다.

### 구현 후 자가 검증

- [ ] 동일 seed 수열과 서로 다른 seed 수열을 비교했다.
- [ ] snapshot 복원 후 다음 값이 원래 인스턴스와 일치한다.
- [ ] 모든 `nextInt` 결과가 범위 안에 있다.
- [ ] rating 경계값의 profile이 의도대로 선택된다.
- [ ] dead zone에서 떨림이 줄어든다.
- [ ] 테스트 결과가 실행 순서와 시스템 시간에 영향받지 않는다.
- [ ] PRNG 알고리즘 변경이 replay에 미치는 영향을 설명할 수 있다.

### 구현 후 설명할 것

1. 게임용 결정적 PRNG와 보안용 난수의 차이
2. seed와 PRNG 상태를 경기 상태에 포함하는 이유
3. AI 난이도를 코드 분기보다 policy 데이터로 분리한 이유
4. rating 구간 방식과 연속 보간 방식의 trade-off
5. 결정성과 사용자의 패턴 예측 가능성 사이의 균형

### 원본 확인 위치

- Thread 10
- 커밋: `refactor(game): 결정적 정수 난수 생성기 추가`
- 커밋: `refactor(game): rating 기반 Pong AI 정책 분리`
- 커밋: `refactor(game): GameHub에 결정적 AI controller 연결`
- `apps/api/src/game/pongAi.ts`
  - `SeededIntegerPrng`
  - `PongAi`
  - `PongAiSnapshot`
  - `profileFor`
  - `predictedBallY`
- `apps/api/src/game/pongAi.test.ts`
- 관련 Thread: 09, 11
