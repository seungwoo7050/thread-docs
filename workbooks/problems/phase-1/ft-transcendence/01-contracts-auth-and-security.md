# 계약·인증·보안 경계 면접 워크북

이 문서는 외부 입력을 신뢰할 수 없는 경계에서 런타임 계약을 세우고, HTTP와 WebSocket 인증·인가·리소스 수명주기를 안전하게 연결하는 문제를 다룬다.

---

## IM-01. [Thread 02 / `feat(api): typed HTTP 오류 boundary 추가`, `feat(shared): 모든 HTTP request schema를 strict하게 정의`] 컴파일 타임 타입과 런타임 HTTP 계약

### 면접 질문

TypeScript 타입이 이미 있는데도 HTTP 요청과 응답에 런타임 스키마 검증을 추가한 이유는 무엇인가요? `params`, `query`, `body`를 각각 검증하고 알 수 없는 필드를 거부하는 방식이 어떤 버그를 줄이는지도 설명해 주세요.

꼬리 질문:

- 요청 검증 실패와 도메인 오류를 하나의 오류 응답 경계에서 어떻게 구분하겠습니까?
- 오류 응답에 원본 예외 메시지를 그대로 넣으면 안 되는 이유는 무엇입니까?
- 응답까지 스키마로 검증할 때 얻는 이점과 비용은 무엇입니까?
- 이미 응답이 전송된 뒤 오류가 발생한 경우 중앙 오류 처리기는 어떻게 동작해야 합니까?

### 30초 모범 답변

TypeScript 타입은 컴파일 후 사라지므로 네트워크에서 들어온 JSON이 실제 타입을 만족한다는 보장이 없습니다. 그래서 요청 경계에서 `params`, `query`, `body`를 strict 스키마로 검증하고, 응답도 계약에 맞는지 확인해 서버와 클라이언트의 드리프트를 조기에 발견했습니다. 검증 오류는 안정적인 오류 코드와 필드 경로로 바꾸고, 도메인 오류는 정해진 HTTP 상태로 매핑하되 내부 예외 메시지나 스택은 노출하지 않습니다. 중앙 오류 경계는 응답 중복 전송도 막아야 합니다.

### 답변 핵심 키워드

런타임 검증 · 타입 소거 · strict object · 입력 구간 분리 · 안정적 오류 envelope · 필드 경로 · 정보 노출 방지 · 응답 계약 · 단일 오류 경계

### 백지 구현

**구현 목표**

세 구간의 요청 값을 런타임 스키마로 검증하고, 검증 실패를 안정적인 API 오류 형태로 변환하는 경계를 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
export interface RuntimeSchema<T> {
  parse(value: unknown): T;
}

export interface RequestContracts<P, Q, B> {
  params?: RuntimeSchema<P>;
  query?: RuntimeSchema<Q>;
  body?: RuntimeSchema<B>;
}

export interface RawRequest {
  params: unknown;
  query: unknown;
  body: unknown;
  requestId: string;
}

export interface ParsedRequest<P, Q, B> {
  params: P;
  query: Q;
  body: B;
}

export class ApiHttpError extends Error {
  constructor(
    readonly status: number,
    readonly code: string,
    readonly fieldErrors: Array<{ path: string; message: string }> = []
  ) {
    super(code);
  }
}

export function parseHttpRequest<P, Q, B>(
  contracts: RequestContracts<P, Q, B>,
  request: RawRequest
): ParsedRequest<P, Q, B> {
  // 직접 구현
  throw new Error("not implemented");
}

export function toApiErrorBody(
  error: unknown,
  requestId: string
): {
  error: {
    code: string;
    message: string;
    requestId: string;
    fields?: Array<{ path: string; message: string }>;
  };
} {
  // 직접 구현
  throw new Error("not implemented");
}
```

**입력과 출력**

- 입력: 원시 `params`, `query`, `body`, 요청 ID, 각 구간의 런타임 스키마
- 출력: 검증된 요청 객체 또는 `ApiHttpError`
- 오류 본문은 항상 동일한 최상위 구조를 사용한다.

**반드시 만족해야 할 조건**

- 어떤 구간에서 실패했는지 필드 경로에 `params`, `query`, `body`가 드러나야 한다.
- 스키마가 없는 구간은 빈 값으로 안전하게 처리하되 임의의 원시 객체를 그대로 통과시키지 않는다.
- 알려진 `ApiHttpError`의 상태·코드는 보존한다.
- 알 수 없는 예외는 내부 오류로 변환하고 원본 메시지·스택을 응답에 노출하지 않는다.
- 요청 ID는 모든 오류 응답에 포함한다.

**경계 조건**

- `body`가 `null`, 배열, 문자열인 경우
- 여러 필드가 동시에 잘못된 경우
- 중첩 필드의 경로가 있는 경우
- 알 수 없는 추가 필드를 strict 스키마가 거부하는 경우

**실패 조건**

- 스키마 라이브러리 자체가 예상하지 못한 형태의 예외를 던지는 경우
- 오류 변환 중 또 다른 예외가 발생하는 경우
- 이미 응답이 전송된 뒤 중앙 오류 처리기가 호출되는 경우는 구현 밖의 조건으로 두고, 어떻게 막을지 설명한다.

**필요한 제약**

- 외부 라이브러리의 구체적인 오류 클래스에 강하게 결합하지 않는다.
- 오류 코드와 사용자 메시지는 허용 목록에서 선택한다.

### 구현 후 자가 검증

- [ ] 정상 요청의 세 구간이 정확한 타입으로 반환된다.
- [ ] 각 구간의 실패가 올바른 경로로 보고된다.
- [ ] 알 수 없는 필드가 strict 스키마에서 거부된다.
- [ ] 알려진 도메인 오류의 상태와 코드가 보존된다.
- [ ] 원본 예외 메시지, 스택, SQL, 토큰이 응답에 섞이지 않는다.
- [ ] 요청 ID가 모든 실패 응답에 포함된다.
- [ ] 검증 실패가 응답 중복 전송으로 이어지지 않는 구조인지 설명할 수 있다.

### 구현 후 설명할 것

1. TypeScript 타입만으로 외부 입력을 신뢰할 수 없는 이유
2. 모든 라우트에서 직접 검증하지 않고 공통 경계를 둔 이유
3. strict 검증이 하위 호환성과 충돌할 수 있는 지점
4. 오류 코드와 사용자 메시지를 내부 예외에서 분리한 이유
5. 응답 검증을 개발·테스트·프로덕션 중 어디까지 적용할지에 대한 판단

### 원본 확인 위치

- Thread 02
- 커밋: `feat(api): typed HTTP 오류 boundary 추가`
- 커밋: `feat(shared): 모든 HTTP request schema를 strict하게 정의`
- `packages/shared/src/http.ts`
  - `defineHttpRequestContract`
  - `jsonHttpRequestContracts`
- `apps/api/src/httpBoundary.ts`
  - `ApiHttpError`
  - `parseInput`
  - `parseOutput`
  - `parseHttpRequest`
  - `installHttpErrorBoundary`
- 관련 Thread: 03, 06, 18

---

## IM-02. [Thread 03 / `feat(protocol): versioned WebSocket event codec 연결`] 버전이 있는 WebSocket 이벤트 프로토콜

### 면접 질문

WebSocket 메시지를 단순 JSON 객체로 처리하지 않고 `v: 1`과 이벤트 `type`을 가진 strict 판별 유니온으로 만든 이유는 무엇인가요? 클라이언트와 서버 양쪽에서 같은 codec을 사용했을 때의 장점과, 버전 전환 시 고려할 점을 설명해 주세요.

꼬리 질문:

- 프로토콜 버전은 연결 handshake에만 둘 수도 있는데, 각 메시지에도 둘 때 무엇이 달라집니까?
- 알 수 없는 이벤트 타입과 알 수 없는 추가 필드는 각각 어떻게 처리하겠습니까?
- 잘못된 입력 한 건 때문에 연결 전체를 종료할지, 오류 이벤트만 보낼지 어떤 기준으로 정하겠습니까?
- 서버가 내보내는 이벤트도 다시 검증하거나 encode 경계를 거쳐야 하는 이유는 무엇입니까?

### 30초 모범 답변

WebSocket은 장시간 유지되고 양방향으로 다양한 이벤트가 섞이기 때문에, 문자열 `type`과 프로토콜 버전으로 메시지를 명확히 구분해야 합니다. strict 판별 유니온을 사용하면 런타임 입력을 검증하면서 TypeScript의 분기 누락도 줄일 수 있고, 서버 출력도 같은 계약으로 encode해 양쪽 드리프트를 잡을 수 있습니다. 지원하지 않는 버전은 상태를 변경하기 전에 거부하고, 잘못된 개별 명령은 정책 위반 정도에 따라 오류 이벤트 또는 연결 종료로 구분합니다.

### 답변 핵심 키워드

판별 유니온 · 프로토콜 버전 · strict parse · exhaustive switch · 상태 변경 전 검증 · 출력 encode · 호환성 정책 · 오류 격리

### 백지 구현

**구현 목표**

문자열 WebSocket frame을 세 가지 클라이언트 이벤트 중 하나로 안전하게 변환한다. 라이브러리 없이 직접 구현해도 되고, 런타임 스키마 도구를 사용해도 된다.

**인터페이스 또는 함수 시그니처**

```ts
type QueueJoinEvent = {
  v: 1;
  type: "queue.join";
  mode: "queue" | "ai";
};

type GameInputEvent = {
  v: 1;
  type: "game.input";
  roomId: string;
  sequence: number;
  direction: -1 | 0 | 1;
};

type ChatSendEvent = {
  v: 1;
  type: "chat.send";
  scope: "lobby" | "match";
  roomId?: string;
  body: string;
};

type ClientEvent = QueueJoinEvent | GameInputEvent | ChatSendEvent;

export class ProtocolError extends Error {
  constructor(readonly code: "invalid_json" | "unsupported_version" | "invalid_event") {
    super(code);
  }
}

export function parseClientFrame(raw: string): ClientEvent {
  // 직접 구현
  throw new Error("not implemented");
}
```

**입력과 출력**

- 입력: 하나의 텍스트 frame
- 출력: 검증된 `ClientEvent`
- 실패: 안정적인 `ProtocolError`

**반드시 만족해야 할 조건**

- JSON 객체만 허용한다.
- `v`는 정확히 `1`이어야 한다.
- 이벤트별 허용 필드 외의 추가 필드를 거부한다.
- `game.input.sequence`는 0 이상의 safe integer여야 한다.
- `game.input.direction`은 `-1`, `0`, `1` 중 하나여야 한다.
- `chat.send`에서 `scope === "match"`이면 `roomId`가 필요하고, `scope === "lobby"`이면 `roomId`를 허용하지 않는다.
- 채팅 본문은 trim 후 비어 있지 않고 정해진 길이 이하여야 한다.

**경계 조건**

- 빈 문자열, 배열, `null`, 숫자 JSON
- `v: "1"`, `v: 2`
- `sequence`가 소수, 음수, `Number.MAX_SAFE_INTEGER` 초과인 경우
- 로비 채팅에 `roomId`를 몰래 넣은 경우
- 유효한 필드와 함께 알 수 없는 필드가 있는 경우

**실패 조건**

- JSON 파싱 실패
- 지원하지 않는 버전
- 알려지지 않은 이벤트 타입
- 이벤트 내부 필드 불일치

**필요한 제약**

- 상태 변경 함수는 `parseClientFrame` 성공 후에만 호출된다.
- 프로토콜 오류에 원본 frame 전체를 포함하지 않는다.

### 구현 후 자가 검증

- [ ] 세 정상 이벤트가 정확히 구분된다.
- [ ] switch 문에서 이벤트 타입 누락을 컴파일 단계에 드러낼 수 있다.
- [ ] 버전 오류가 다른 필드 오류보다 먼저 식별된다.
- [ ] 추가 필드와 조건부 `roomId` 규칙이 검증된다.
- [ ] 큰 수, 소수, 음수 sequence가 거부된다.
- [ ] 실패한 frame이 게임·채팅 상태를 바꾸지 않는다.
- [ ] 오류 로그에 민감하거나 과도한 원문 payload를 남기지 않는다.

### 구현 후 설명할 것

1. 각 frame에 버전을 넣는 방식과 handshake 버전만 두는 방식의 차이
2. strict 필드 정책과 점진적 확장의 trade-off
3. 파싱 실패 시 오류 이벤트와 연결 종료를 구분하는 기준
4. 공유 codec이 클라이언트·서버 결합도를 높이는 대신 얻는 안정성
5. 다음 버전 도입 시 병행 지원과 강제 업그레이드 중 선택 기준

### 원본 확인 위치

- Thread 03
- 커밋: `feat(protocol): versioned WebSocket event codec 연결`
- `packages/shared/src/ws.ts`
  - `clientEventSchema`
  - `serverEventSchema`
  - `parseClientEvent`
  - `parseServerEvent`
  - `encodeServerEvent`
- `packages/shared/src/game.ts`
- `apps/api/src/gameHub.ts`
- 관련 Thread: 02, 06, 14, 18

---

## IM-03. [Thread 06 / `feat(auth): ticket 기반 WebSocket 인증 연결`, `test(auth): WebSocket ticket 경계 검증`] 쿠키 세션에서 일회용 WebSocket 티켓으로 넘기는 인증 경계

### 면접 질문

HTTP 쿠키 세션을 WebSocket URL이나 첫 메시지에 그대로 재사용하지 않고, 짧은 TTL의 일회용 티켓을 발급해 연결한 이유는 무엇인가요? 티켓 원문 대신 hash를 저장하고, 소비를 `DELETE ... RETURNING` 형태로 원자화한 이유도 설명해 주세요.

꼬리 질문:

- 프로토콜 버전 검증과 티켓 소비 중 무엇을 먼저 해야 합니까?
- 같은 티켓으로 20개 연결이 동시에 들어오면 어떤 결과가 보장되어야 합니까?
- 인증 조회가 끝나기 전에 도착한 WebSocket 메시지는 어떻게 제한해야 합니까?
- 티켓 소비 직전에 계정이 정지되거나 티켓이 만료되면 어떻게 처리합니까?
- 다중 인스턴스 환경에서 메모리 티켓 저장소가 적합하지 않은 이유는 무엇입니까?

### 30초 모범 답변

쿠키 세션을 URL에 노출하면 로그·히스토리·프록시를 통해 장기 자격 증명이 새어 나갈 수 있어, HTTP 인증 후 짧게 유효한 일회용 티켓만 WebSocket handshake에 사용했습니다. 서버에는 티켓 원문이 아니라 hash만 저장하고, 활성 사용자·만료 조건을 확인하면서 행을 삭제해 동시 소비 중 한 요청만 성공하게 했습니다. 지원 버전을 먼저 검증해 잘못된 handshake가 티켓을 소모하지 않게 하고, 인증 전 메시지는 개수와 총 바이트를 제한해 메모리 공격도 막았습니다.

### 답변 핵심 키워드

자격 증명 축소 · 짧은 TTL · 원문 미저장 · hash · atomic consume · `DELETE RETURNING` · 버전 선검증 · bounded pre-auth buffer · 정지 사용자 재검증

### 백지 구현

**구현 목표**

단일 프로세스용 일회용 티켓 저장소와 인증 완료 전 메시지 버퍼를 구현한다. 구현을 마친 뒤 PostgreSQL에서는 어떤 원자 연산으로 바꿀지 설명한다.

**인터페이스 또는 함수 시그니처**

```ts
export interface IssuedTicket {
  ticket: string;
  expiresAtMs: number;
}

export class OneTimeTicketStore {
  constructor(
    private readonly now: () => number,
    private readonly randomToken: () => string
  ) {}

  issue(userId: string, ttlMs: number): IssuedTicket {
    // 직접 구현
    throw new Error("not implemented");
  }

  consume(ticket: string): string | null {
    // 직접 구현
    throw new Error("not implemented");
  }

  get storedTicketCount(): number {
    // 직접 구현
    return 0;
  }
}

export class PreAuthMessageBuffer {
  constructor(
    private readonly maxMessages: number,
    private readonly maxBytes: number,
    private readonly maxSingleMessageBytes: number
  ) {}

  push(payload: string): void {
    // 직접 구현
  }

  drain(): string[] {
    // 직접 구현
    return [];
  }
}
```

**입력과 출력**

- `issue`: 사용자 ID와 TTL을 받아 외부에 줄 원문 티켓과 만료 시각을 반환한다.
- `consume`: 티켓을 한 번만 사용자 ID로 교환하고 이후에는 `null`을 반환한다.
- `PreAuthMessageBuffer`: 인증 완료 전 frame을 제한적으로 저장하고 한 번에 비운다.

**반드시 만족해야 할 조건**

- 저장소에는 원문 티켓이 아니라 암호학적 hash만 남는다.
- TTL은 양수이며 상한을 둔다.
- 만료된 티켓은 성공하지 않고 저장소에서도 제거된다.
- 한 티켓은 정확히 한 번만 성공한다.
- `consume` 결과를 반환하기 전에 항목을 제거한다.
- protocol version 검증은 `consume` 호출 전에 수행한다고 가정하고, 호출 순서를 설명한다.
- 버퍼는 단일 frame 크기, frame 개수, 누적 바이트를 모두 제한한다.
- `drain`은 저장된 순서를 유지하고 두 번째 호출에서는 빈 배열을 반환한다.

**경계 조건**

- TTL이 0, 음수, 상한 초과인 경우
- 같은 티켓을 연속·동시에 소비하는 경우
- 만료 시각과 현재 시각이 정확히 같은 경우
- 멀티바이트 문자열의 byte 길이와 문자 길이가 다른 경우
- 한도에 정확히 도달하는 frame

**실패 조건**

- 난수 생성기가 중복 티켓을 반환하는 경우
- hash 계산 실패
- 버퍼가 한도를 넘은 경우
- 인증 실패 뒤 버퍼를 폐기하지 않은 경우

**필요한 제약**

- 토큰 비교·저장은 일정한 형식의 hash 키를 사용한다.
- 오류 메시지에 원문 티켓을 포함하지 않는다.
- 실제 분산 환경에서는 데이터베이스나 공유 저장소의 원자 소비가 필요하다.

### 구현 후 자가 검증

- [ ] 발급 결과와 저장 키가 동일한 원문이 아니다.
- [ ] 같은 티켓의 두 번째 소비가 실패한다.
- [ ] 만료 경계가 일관되게 처리된다.
- [ ] 잘못된 프로토콜 버전이 티켓을 소비하지 않는 호출 순서다.
- [ ] 인증 전 버퍼가 개수·단일 크기·총 바이트를 각각 제한한다.
- [ ] `drain` 후 참조와 누적 바이트가 모두 초기화된다.
- [ ] 티켓·버퍼 오류 경로에 리소스가 남지 않는다.
- [ ] PostgreSQL에서 경쟁 상태 없이 소비할 쿼리와 조건을 설명할 수 있다.

### 구현 후 설명할 것

1. 쿠키 세션을 WebSocket URL에 직접 넣지 않은 보안 이유
2. 원문 대신 hash를 저장할 때 공격 표면이 어떻게 줄어드는지
3. 애플리케이션의 `select` 후 `delete`가 원자 소비를 보장하지 못하는 이유
4. 버전 확인을 티켓 소비보다 먼저 둔 이유
5. 인증 전 메시지 버퍼에서 개수와 바이트를 둘 다 제한한 이유

### 원본 확인 위치

- Thread 06
- 커밋: `feat(auth): ticket 기반 WebSocket 인증 연결`
- 커밋: `test(auth): WebSocket ticket 경계 검증`
- `packages/db/migrations/002_ws_tickets.sql`
- `packages/db/src/index.ts`
  - `createWsTicket`
  - `consumeWsTicket`
- `apps/api/src/wsTicket.ts`
  - `createRawWsTicket`
  - `hashWsTicket`
  - `WS_TICKET_TTL_SECONDS`
- `apps/api/src/app.ts`
- `packages/shared/src/http.ts`
  - `wsHandshakeQuerySchema`
  - `wsTicketResponseSchema`
- 관련 Thread: 03, 07, 08

---

## IM-04. [Thread 07 / `fix(db): 차단 감사 기록을 원자적으로 저장`, `test(auth): 계정 정지의 기존 WebSocket 차단 검증`] 계정 상태 변경의 원자성 및 실시간 권한 회수

### 면접 질문

사용자 상태를 `banned`로 바꾸는 것과 감사 로그를 남기는 것을 왜 한 트랜잭션으로 묶어야 하나요? 데이터베이스 변경이 성공한 직후 기존 WebSocket 연결을 어떻게 회수해야 하며, 연결 정리 과정에서 어떤 리소스를 빠뜨리기 쉬운지도 설명해 주세요.

꼬리 질문:

- 감사 로그 삽입이 실패했는데 사용자 상태만 바뀌면 어떤 문제가 생깁니까?
- 데이터베이스 commit은 성공했지만 WebSocket 종료가 실패하면 무엇을 진실의 원천으로 봅니까?
- 여러 API 인스턴스에 연결된 사용자를 즉시 끊으려면 무엇이 추가로 필요합니까?
- 정지 해제도 동일한 방식의 실시간 side effect가 필요합니까?
- 정지된 사용자가 보유하던 경기 좌석을 즉시 몰수할지 재연결 유예를 둘지 어떤 정책이 필요합니까?

### 30초 모범 답변

계정 상태와 감사 기록은 한 관리 행위의 두 결과이므로 트랜잭션으로 묶어 둘 중 하나만 남는 상태를 막아야 합니다. commit 후 현재 프로세스의 연결 레지스트리에서 해당 사용자를 찾아 heartbeat, snapshot 버퍼, 매칭·토너먼트 대기, 입력 상태를 정리하고 정책 close code로 연결을 닫습니다. 데이터베이스가 권한의 진실의 원천이며, 연결 종료 실패는 별도 관측·재시도 대상입니다. 여러 인스턴스라면 pub/sub이나 outbox 기반 revocation 전파가 필요합니다.

### 답변 핵심 키워드

상태+감사 원자성 · 트랜잭션 · commit 후 side effect · 권한의 진실의 원천 · live revocation · 리소스 정리 · close code · 다중 인스턴스 전파

### 백지 구현

**구현 목표**

계정 정지와 감사 기록을 원자적으로 저장한 뒤, 현재 연결이 있으면 안전하게 회수하는 서비스 경계를 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
export interface PublicUser {
  id: string;
  status: "active" | "banned";
}

export interface BanTransaction {
  setStatus(targetUserId: string, status: "banned"): Promise<PublicUser>;
  appendAudit(input: {
    actorId: string;
    targetUserId: string;
    action: "ban";
    reason: string;
  }): Promise<void>;
}

export interface BanRepository {
  transaction<T>(work: (tx: BanTransaction) => Promise<T>): Promise<T>;
}

export interface ConnectionRevoker {
  revokeUser(userId: string): void;
}

export async function suspendUser(
  repo: BanRepository,
  revoker: ConnectionRevoker,
  input: { actorId: string; targetUserId: string; reason: string }
): Promise<PublicUser> {
  // 직접 구현
  throw new Error("not implemented");
}
```

**입력과 출력**

- 입력: 행위자 ID, 대상 사용자 ID, 감사 사유
- 출력: 정지 상태가 반영된 사용자
- 데이터베이스 작업 실패 시 연결 회수는 실행하지 않는다.

**반드시 만족해야 할 조건**

- 사유는 trim 후 비어 있지 않고 길이 상한을 만족해야 한다.
- 상태 변경과 감사 기록은 같은 트랜잭션 안에서 실행한다.
- 트랜잭션 commit이 끝난 뒤에만 `revokeUser`를 호출한다.
- 감사 기록 실패 시 상태 변경도 rollback되어야 한다.
- 연결이 없어도 정지 작업은 성공한다.
- `revokeUser`가 예외를 던져도 이미 commit된 데이터베이스 상태를 되돌린 척하지 않는다. 이 경우 반환·로그 정책을 명시한다.

**경계 조건**

- 이미 정지된 사용자
- 행위자와 대상이 같은 사용자
- 존재하지 않는 대상
- 공백 사유와 최대 길이 사유
- commit 직후 연결이 교체되는 경쟁 상태

**실패 조건**

- 사용자 update 실패
- 감사 insert 실패
- commit 실패
- commit 후 로컬 연결 종료 실패

**필요한 제약**

- 다중 인스턴스 전파는 구현 범위 밖이지만 확장 방안을 설명한다.
- 감사 기록에는 토큰이나 원본 요청 전체를 저장하지 않는다.

### 구현 후 자가 검증

- [ ] 감사 저장 실패 시 사용자 상태가 바뀌지 않는다.
- [ ] commit 전에는 연결 회수를 시도하지 않는다.
- [ ] 연결이 없는 사용자를 정지해도 성공한다.
- [ ] 같은 요청이 반복될 때 감사 정책과 상태 결과가 명확하다.
- [ ] 회수 과정에서 heartbeat, 대기열, 입력 상태, snapshot 버퍼 같은 소유 리소스가 정리되는지 설명할 수 있다.
- [ ] 오래된 연결의 close callback이 새 연결을 지우지 않는 식별자 검사가 필요함을 인지한다.
- [ ] 데이터베이스 성공 후 side effect 실패가 관측 가능하다.

### 구현 후 설명할 것

1. 데이터베이스 트랜잭션이 보호하는 범위와 보호하지 못하는 범위
2. commit 후 WebSocket 회수를 수행하는 순서
3. 로컬 메모리 레지스트리만으로 다중 인스턴스 권한 회수가 불가능한 이유
4. 즉시 좌석 해제와 재연결 유예 사이의 정책 trade-off
5. 감사 로그에 저장할 정보와 저장하지 않을 정보

### 원본 확인 위치

- Thread 07
- 커밋: `fix(db): 차단 감사 기록을 원자적으로 저장`
- 커밋: `test(auth): 계정 정지의 기존 WebSocket 차단 검증`
- `packages/db/src/index.ts`
  - `setUserBan`
- `packages/db/src/schema.ts`
  - `AdminActionTable`
- `apps/api/src/app.ts`
- `apps/api/src/gameHub.ts`
  - `GameHub.revokeUser`
- `apps/api/src/admin.test.ts`
- `packages/db/src/postgres.integration.test.ts`
- 관련 Thread: 06, 12, 15

---

## IM-05. [Thread 08 / `feat(guest): guest resource lease 수명주기 추가`, `test(guest): 위조 client address 거부`] 서명된 게스트 신원과 교체 가능한 리소스 lease

### 면접 질문

게스트 사용자를 영속 계정과 분리하면서도 쿠키 위조를 막고, 연결 수 제한을 정확히 유지하려면 어떤 상태를 클라이언트 서명 값으로 두고 어떤 상태를 서버 메모리에 둬야 하나요? 같은 게스트가 연결을 교체할 때 이전 lease의 `release`가 새 lease를 해제하지 않도록 한 이유도 설명해 주세요.

꼬리 질문:

- 단순히 연결 수를 증가·감소하는 방식은 어떤 경쟁 상태를 만듭니까?
- `X-Forwarded-For`를 언제 신뢰할 수 있습니까?
- 서버 재시작 시 게스트 세션과 lease는 어떻게 달라집니까?
- 게스트와 등록 사용자를 같은 매칭 대기열에 넣지 않은 이유는 무엇입니까?
- 만료된 티켓과 결과 보관 timer를 어떻게 정리해야 메모리 누수가 생기지 않습니까?

### 30초 모범 답변

게스트 ID와 만료 시각처럼 위조 방지가 필요한 최소 신원은 HMAC으로 서명하고, 현재 연결 소유권·IP별 사용량·일회용 티켓처럼 계속 변하는 리소스는 서버가 관리했습니다. 연결 교체 때는 게스트별 generation을 올려 새 lease가 현재 소유자가 되게 하고, 오래된 lease의 release는 generation이 맞을 때만 카운트를 줄여 이중 감소를 막습니다. 클라이언트 주소는 프록시를 명시적으로 신뢰할 때만 전달 헤더를 사용하고, 그렇지 않으면 실제 socket 주소를 기준으로 제한합니다.

### 답변 핵심 키워드

HMAC 서명 · 최소 신원 · 서버 소유 resource state · lease generation · stale release 방지 · 전역/IP 제한 · trusted proxy · 데이터 격리 · timer cleanup

### 백지 구현

**구현 목표**

게스트별 활성 연결 lease를 관리하고, 프록시 신뢰 설정에 따라 제한 기준이 되는 클라이언트 주소를 결정한다.

**인터페이스 또는 함수 시그니처**

```ts
export interface GuestLease {
  readonly guestId: string;
  readonly generation: number;
  release(): void;
}

export class GuestLeaseRegistry {
  constructor(
    private readonly globalLimit: number,
    private readonly perIpLimit: number
  ) {}

  acquire(ip: string, guestId: string): GuestLease | null {
    // 직접 구현
    return null;
  }

  get activeConnectionCount(): number {
    // 직접 구현
    return 0;
  }

  activeForIp(ip: string): number {
    // 직접 구현
    return 0;
  }
}

export function resolveClientAddress(input: {
  remoteAddress: string;
  forwardedFor?: string;
  trustProxy: boolean;
}): string {
  // 직접 구현
  throw new Error("not implemented");
}
```

**입력과 출력**

- `acquire`: IP와 게스트 ID를 받아 lease 또는 제한 초과 시 `null`을 반환한다.
- 같은 게스트의 재획득은 기존 lease를 교체하며 활성 수를 중복 증가시키지 않는다.
- `resolveClientAddress`: 신뢰 설정에 맞는 제한 키를 반환한다.

**반드시 만족해야 할 조건**

- 전역 제한과 IP별 제한을 동시에 지킨다.
- 같은 게스트가 같은 IP에서 교체 연결을 얻을 수 있다.
- 교체 후 오래된 lease의 `release`는 현재 lease나 카운트를 건드리지 않는다.
- 현재 lease의 `release`는 한 번만 효과가 있다.
- 같은 게스트가 다른 IP로 이동하면 양쪽 IP 카운트가 정확히 이동한다.
- `trustProxy === false`이면 `forwardedFor`를 무시한다.
- `trustProxy === true`일 때도 빈 값이나 형식이 잘못된 첫 주소를 그대로 신뢰하지 않는다.

**경계 조건**

- limit이 1인 상태에서 같은 게스트가 연결을 교체하는 경우
- 전역 여유는 있지만 해당 IP가 가득 찬 경우
- IP 제한은 여유지만 전역 제한이 가득 찬 경우
- `release`를 여러 번 호출하는 경우
- 콤마로 여러 주소가 들어온 전달 헤더

**실패 조건**

- 카운트가 음수가 되는 경우
- stale release가 새 lease를 제거하는 경우
- 신뢰하지 않는 프록시 헤더로 제한을 우회하는 경우
- 제한 실패 뒤 일부 카운트만 증가한 경우

**필요한 제약**

- 단일 프로세스 레지스트리로 한정한다.
- 다중 인스턴스 전역 제한은 별도 공유 저장소가 필요함을 설명한다.

### 구현 후 자가 검증

- [ ] 최초 획득과 정상 release에서 전역·IP 카운트가 맞는다.
- [ ] 같은 게스트의 교체가 활성 수를 늘리지 않는다.
- [ ] 이전 lease release가 현재 연결에 영향을 주지 않는다.
- [ ] 제한 초과 요청이 모든 내부 상태를 그대로 유지한다.
- [ ] IP 이동 후 이전 IP 카운트가 줄고 새 IP 카운트가 늘어난다.
- [ ] 위조한 `X-Forwarded-For`가 `trustProxy=false`에서 무시된다.
- [ ] release가 멱등적이다.
- [ ] 각 연산의 시간 복잡도가 평균 O(1)인지 확인한다.

### 구현 후 설명할 것

1. 서명된 클라이언트 상태와 서버 리소스 상태를 나눈 기준
2. 단순 카운터 대신 lease와 generation을 사용한 이유
3. 프록시 신뢰 설정이 보안 설정인 이유
4. 게스트 리소스를 메모리에 둘 때의 장점과 재시작 시 한계
5. 게스트·등록 사용자 데이터와 매칭을 격리한 정책적 이유

### 원본 확인 위치

- Thread 08
- 커밋: `feat(guest): guest resource lease 수명주기 추가`
- 커밋: `test(guest): 위조 client address 거부`
- `apps/api/src/guestAccess.ts`
  - `GuestAccess`
  - `GuestAccessError`
- `apps/api/src/guest-demo.test.ts`
- `apps/api/src/gameHub.guest.test.ts`
- `apps/api/src/app.ts`
- `apps/web/src/lib/demoPolicy.ts`
- 관련 Thread: 06, 11, 12
