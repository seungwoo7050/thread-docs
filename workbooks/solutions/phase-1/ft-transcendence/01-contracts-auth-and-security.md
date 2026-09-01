# 계약·인증·보안 경계 면접 워크북

이 문서는 외부 입력을 신뢰할 수 없는 경계에서 런타임 계약을 세우고, HTTP와 WebSocket 인증·인가·리소스 수명주기를 안전하게 연결하는 문제를 다룬다.

---

## IM-01. [Thread 02 / `feat(api): typed HTTP 오류 boundary 추가`, `feat(shared): 모든 HTTP request schema를 strict하게 정의`] 컴파일 타임 타입과 런타임 HTTP 계약

### 면접 질문

TypeScript 타입이 이미 있는데도 HTTP 요청과 응답에 런타임 스키마 검증을 추가한 이유는 무엇인가요? `params`, `query`, `body`를 각각 검증하고 알 수 없는 필드를 거부하는 방식이 어떤 버그를 줄이는지도 설명해 주세요.

꼬리 질문:

- 요청 검증 실패와 도메인 오류를 하나의 오류 응답 경계에서 어떻게 구분하겠습니까?
  - 모범답변: 스키마 파싱 실패는 `400 validation_error`와 구간이 포함된 필드 경로로 정규화하고, 이미 `ApiHttpError`인 도메인 오류는 정해진 상태와 코드를 보존합니다. 그 밖의 예외만 `500 internal_error`로 숨기고 서버 로그에 원인을 남깁니다.
- 오류 응답에 원본 예외 메시지를 그대로 넣으면 안 되는 이유는 무엇입니까?
  - 모범답변: 예외 메시지에는 SQL, 내부 경로, 토큰이나 라이브러리 세부사항이 섞일 수 있고 문구도 안정적인 API 계약이 아닙니다. 이 프로젝트는 허용된 코드와 사용자용 메시지만 응답하고 원본 예외는 구조화 로그로만 관측합니다.
- 응답까지 스키마로 검증할 때 얻는 이점과 비용은 무엇입니까?
  - 모범답변: `parseOutput`은 서버 구현과 공유 응답 계약의 드리프트를 응답 직전에 발견합니다. 대신 매 응답 검증 비용과 프로덕션 오류 가능성이 추가되므로, 큰 응답에서는 성능을 측정하거나 테스트·핵심 경계 위주로 적용할 수 있습니다.
- 이미 응답이 전송된 뒤 오류가 발생한 경우 중앙 오류 처리기는 어떻게 동작해야 합니까?
  - 모범답변: `reply.sent` 같은 상태를 확인해 두 번째 오류 본문을 보내지 않고 오류만 기록하거나 프레임워크의 기본 종료 경로에 맡겨야 합니다. 핵심은 한 요청에 응답 소유자가 하나라는 invariant를 지키는 것입니다.

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
  const parseSection = <T>(
    section: "params" | "query" | "body",
    schema: RuntimeSchema<T> | undefined,
    value: unknown
  ): T => {
    // 계약이 없는 구간도 원시 입력을 통과시키지 않고 빈 객체로 고정한다.
    if (!schema) return {} as T;

    try {
      return schema.parse(value ?? {});
    } catch (error) {
      if (error instanceof ApiHttpError) throw error;

      const candidate = error as {
        issues?: Array<{ path?: Array<string | number>; message?: unknown }>;
      };
      const fieldErrors = Array.isArray(candidate?.issues)
        ? candidate.issues.map((issue) => ({
            path: [section, ...(Array.isArray(issue.path) ? issue.path : [])].join("."),
            message: typeof issue.message === "string" ? issue.message : "Invalid value"
          }))
        : [{ path: section, message: "Invalid value" }];
      throw new ApiHttpError(400, "validation_error", fieldErrors);
    }
  };

  return {
    params: parseSection("params", contracts.params, request.params),
    query: parseSection("query", contracts.query, request.query),
    body: parseSection("body", contracts.body, request.body)
  };
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
  const messages: Record<string, string> = {
    validation_error: "입력값을 확인해주세요.",
    authentication_required: "로그인이 필요합니다.",
    account_suspended: "정지된 계정은 이 작업을 수행할 수 없습니다.",
    admin_required: "운영자 권한이 필요합니다.",
    not_found: "요청한 대상을 찾을 수 없습니다.",
    internal_error: "요청을 처리하지 못했습니다."
  };

  if (error instanceof ApiHttpError && messages[error.code]) {
    return {
      error: {
        code: error.code,
        message: messages[error.code],
        requestId,
        ...(error.fieldErrors.length > 0 ? { fields: error.fieldErrors } : {})
      }
    };
  }

  // 알 수 없는 예외의 메시지와 stack은 외부 계약에 포함하지 않는다.
  return {
    error: {
      code: "internal_error",
      message: messages.internal_error,
      requestId
    }
  };
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
   - 모범답변: TypeScript 타입은 컴파일 때 제거되며 네트워크 JSON을 검사하지 않습니다. 따라서 외부 입력은 `unknown`으로 받고 실제 프로젝트처럼 Zod 등의 런타임 스키마를 통과한 값만 도메인 코드에 넘겨야 합니다.
2. 모든 라우트에서 직접 검증하지 않고 공통 경계를 둔 이유
   - 모범답변: 공통 `parseHttpRequest`와 오류 boundary를 두면 params·query·body의 기본값, 오류 envelope, request ID, 내부 오류 은닉 정책이 모든 라우트에서 동일해지고 누락을 줄일 수 있습니다.
3. strict 검증이 하위 호환성과 충돌할 수 있는 지점
   - 모범답변: 새 클라이언트가 추가 필드를 먼저 보내면 구버전 서버의 strict 스키마가 거부할 수 있습니다. 배포 순서를 서버 스키마 확장 후 클라이언트 사용으로 잡거나 프로토콜 버전을 나누는 호환성 정책이 필요합니다.
4. 오류 코드와 사용자 메시지를 내부 예외에서 분리한 이유
   - 모범답변: 안정적인 오류 코드에는 클라이언트 분기 계약을 맡기고, 사용자 메시지는 안전한 허용 목록에서 선택합니다. 내부 예외의 변경이나 민감 정보가 외부 API로 새는 것을 막기 위한 분리입니다.
5. 응답 검증을 개발·테스트·프로덕션 중 어디까지 적용할지에 대한 판단
   - 모범답변: 이 프로젝트는 `parseOutput`으로 프로덕션 경계에서도 응답을 검증해 계약 위반을 즉시 실패시킵니다. 일반적으로는 응답 크기와 트래픽 비용을 측정해 핵심 응답은 항상 검증하고, 고비용 경로는 테스트 또는 표본 검증으로 조정할 수 있습니다.

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
  - 모범답변: handshake 버전은 연결 전체가 한 버전이라는 전제가 단순하지만 연결 중 버전별 메시지를 독립적으로 기록·재생하기 어렵습니다. 각 frame의 `v`는 메시지 자체가 완결된 계약이 되어 로그·fixture·codec에서 바로 검증할 수 있는 대신 매 frame 중복과 혼합 버전 정책이 생깁니다.
- 알 수 없는 이벤트 타입과 알 수 없는 추가 필드는 각각 어떻게 처리하겠습니까?
  - 모범답변: 둘 다 상태 변경 전에 `invalid_event`로 거부합니다. 프로젝트의 판별 유니온은 알려지지 않은 `type`을 분기하지 못하게 하고 각 객체의 `.strict()`는 알려진 이벤트에 몰래 추가된 필드도 거부합니다.
- 잘못된 입력 한 건 때문에 연결 전체를 종료할지, 오류 이벤트만 보낼지 어떤 기준으로 정하겠습니까?
  - 모범답변: 인증·버전 위반이나 크기 제한처럼 연결 신뢰 자체가 깨진 경우에는 정책 close를 사용하고, 인증된 연결의 개별 명령 형식 오류는 `invalid_event`를 보내고 연결은 유지합니다. 반복 위반에는 별도 rate limit이나 종료 정책을 둘 수 있습니다.
- 서버가 내보내는 이벤트도 다시 검증하거나 encode 경계를 거쳐야 하는 이유는 무엇입니까?
  - 모범답변: 서버 내부 타입 단언이나 조립 실수도 런타임에는 생길 수 있습니다. 프로젝트의 `encodeServerEvent`처럼 출력 스키마를 다시 통과시키면 잘못된 이벤트가 wire에 나가기 전에 서버·클라이언트 계약 드리프트를 발견할 수 있습니다.

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
  let value: unknown;
  try {
    value = JSON.parse(raw);
  } catch {
    throw new ProtocolError("invalid_json");
  }

  if (typeof value !== "object" || value === null || Array.isArray(value)) {
    throw new ProtocolError("invalid_event");
  }
  const event = value as Record<string, unknown>;
  if (event.v !== 1) throw new ProtocolError("unsupported_version");

  const hasExactKeys = (keys: string[]) => {
    const expected = [...keys].sort();
    const actual = Object.keys(event).sort();
    return actual.length === expected.length
      && actual.every((key, index) => key === expected[index]);
  };

  if (event.type === "queue.join") {
    if (!hasExactKeys(["v", "type", "mode"]) || (event.mode !== "queue" && event.mode !== "ai")) {
      throw new ProtocolError("invalid_event");
    }
    return event as QueueJoinEvent;
  }

  if (event.type === "game.input") {
    if (
      !hasExactKeys(["v", "type", "roomId", "sequence", "direction"])
      || typeof event.roomId !== "string"
      || event.roomId.length === 0
      || !Number.isSafeInteger(event.sequence)
      || (event.sequence as number) < 0
      || (event.direction !== -1 && event.direction !== 0 && event.direction !== 1)
    ) {
      throw new ProtocolError("invalid_event");
    }
    return event as GameInputEvent;
  }

  if (event.type === "chat.send") {
    const match = event.scope === "match";
    const lobby = event.scope === "lobby";
    const body = typeof event.body === "string" ? event.body.trim() : "";
    const keys = match
      ? ["v", "type", "scope", "roomId", "body"]
      : ["v", "type", "scope", "body"];
    if (
      (!match && !lobby)
      || !hasExactKeys(keys)
      || (match && (typeof event.roomId !== "string" || event.roomId.length === 0))
      || body.length === 0
      || body.length > 240
    ) {
      throw new ProtocolError("invalid_event");
    }
    return { ...event, body } as ChatSendEvent;
  }

  throw new ProtocolError("invalid_event");
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
   - 모범답변: frame 버전은 각 메시지를 독립적으로 검증·저장·재생할 수 있고 프로젝트의 replay fixture에도 유리합니다. handshake만 버전화하면 payload는 작고 연결별 codec 선택은 단순하지만, 메시지만 떼어 보았을 때 적용할 계약을 알 수 없습니다.
2. strict 필드 정책과 점진적 확장의 trade-off
   - 모범답변: strict 정책은 오타와 예상 밖 데이터를 즉시 잡지만 구버전 서버가 새 필드를 받아 주지 않습니다. 점진 확장은 서버가 먼저 새 스키마를 지원하거나 새 `v`를 병행한 뒤 클라이언트를 전환하는 순서로 해결합니다.
3. 파싱 실패 시 오류 이벤트와 연결 종료를 구분하는 기준
   - 모범답변: 연결 인증·지원 버전·payload 상한 위반은 handshake 신뢰 경계 문제라 종료하고, 인증 뒤 한 번의 잘못된 게임·채팅 명령은 프로젝트처럼 `invalid_event`만 응답해 다른 정상 frame을 격리합니다.
4. 공유 codec이 클라이언트·서버 결합도를 높이는 대신 얻는 안정성
   - 모범답변: 양쪽 배포가 같은 패키지 계약에 의존하지만, 입력 parse와 출력 encode가 같은 판별 유니온을 사용해 필드명·타입·이벤트 분기의 불일치를 컴파일과 런타임 양쪽에서 빠르게 찾습니다.
5. 다음 버전 도입 시 병행 지원과 강제 업그레이드 중 선택 기준
   - 모범답변: 장시간 연결과 독립 배포 클라이언트가 남아 있으면 제한 기간 동안 v1·v2 codec을 병행하고 관측 후 종료합니다. 서버와 웹을 동시에 배포하고 구버전 연결을 안전하게 끊을 수 있다면 강제 업그레이드가 더 단순합니다.

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
  - 모범답변: 버전을 먼저 검증해야 합니다. 실제 handshake도 `v === "1"`과 strict query schema를 확인한 뒤에만 티켓 hash를 계산·소비하므로, 지원하지 않는 client가 유효 티켓을 소진하지 못합니다.
- 같은 티켓으로 20개 연결이 동시에 들어오면 어떤 결과가 보장되어야 합니까?
  - 모범답변: 정확히 한 연결만 사용자로 교환되고 나머지는 모두 실패해야 합니다. PostgreSQL 구현은 티켓 행을 `DELETE ... RETURNING`하는 단일 문장으로 경쟁 요청 중 한 요청만 행을 얻습니다.
- 인증 조회가 끝나기 전에 도착한 WebSocket 메시지는 어떻게 제한해야 합니까?
  - 모범답변: 프로젝트처럼 단일 frame 8KiB, 최대 16개, 누적 32KiB를 동시에 제한하고 초과하면 1009로 닫습니다. 인증 성공 후 listener를 제거하고 도착 순서대로 GameHub에 넘기며, 실패하면 버퍼 참조를 폐기합니다.
- 티켓 소비 직전에 계정이 정지되거나 티켓이 만료되면 어떻게 처리합니까?
  - 모범답변: 소비 시점에 만료와 현재 사용자 상태를 다시 검사하고 실패시켜야 합니다. 실제 쿼리는 삭제된 티켓의 `expires_at > now()`와 `u.status = 'active'`를 함께 확인하므로 티켓은 재사용되지 않으면서 인증도 거부됩니다.
- 다중 인스턴스 환경에서 메모리 티켓 저장소가 적합하지 않은 이유는 무엇입니까?
  - 모범답변: 발급 인스턴스와 handshake 인스턴스가 다를 수 있고 각 Map은 원자 소비 상태를 공유하지 못합니다. 등록 사용자는 PostgreSQL을 사용해 어느 인스턴스에서도 같은 일회성 보장을 받습니다.

### 30초 모범 답변

쿠키 세션을 URL에 노출하면 로그·히스토리·프록시를 통해 장기 자격 증명이 새어 나갈 수 있어, HTTP 인증 후 짧게 유효한 일회용 티켓만 WebSocket handshake에 사용했습니다. 서버에는 티켓 원문이 아니라 hash만 저장하고, 활성 사용자·만료 조건을 확인하면서 행을 삭제해 동시 소비 중 한 요청만 성공하게 했습니다. 지원 버전을 먼저 검증해 잘못된 handshake가 티켓을 소모하지 않게 하고, 인증 전 메시지는 개수와 총 바이트를 제한해 메모리 공격도 막았습니다.

### 답변 핵심 키워드

자격 증명 축소 · 짧은 TTL · 원문 미저장 · hash · atomic consume · `DELETE RETURNING` · 버전 선검증 · bounded pre-auth buffer · 정지 사용자 재검증

### 백지 구현

**구현 목표**

단일 프로세스용 일회용 티켓 저장소와 인증 완료 전 메시지 버퍼를 구현한다. 구현을 마친 뒤 PostgreSQL에서는 어떤 원자 연산으로 바꿀지 설명한다.

**인터페이스 또는 함수 시그니처**

```ts
import { createHash } from "node:crypto";

export interface IssuedTicket {
  ticket: string;
  expiresAtMs: number;
}

export class OneTimeTicketStore {
  private readonly tickets = new Map<string, { userId: string; expiresAtMs: number }>();
  private static readonly MAX_TTL_MS = 30_000;

  constructor(
    private readonly now: () => number,
    private readonly randomToken: () => string
  ) {}

  issue(userId: string, ttlMs: number): IssuedTicket {
    if (!Number.isSafeInteger(ttlMs) || ttlMs <= 0 || ttlMs > OneTimeTicketStore.MAX_TTL_MS) {
      throw new RangeError("invalid ticket TTL");
    }
    const ticket = this.randomToken();
    const ticketHash = createHash("sha256").update(ticket, "utf8").digest("hex");
    // 실제 PostgreSQL의 primary key처럼 중복 난수는 기존 티켓을 덮어쓰지 않고 실패시킨다.
    if (this.tickets.has(ticketHash)) throw new Error("ticket collision");
    const expiresAtMs = this.now() + ttlMs;
    this.tickets.set(ticketHash, { userId, expiresAtMs });
    return { ticket, expiresAtMs };
  }

  consume(ticket: string): string | null {
    const ticketHash = createHash("sha256").update(ticket, "utf8").digest("hex");
    const stored = this.tickets.get(ticketHash);
    if (!stored) return null;

    // 조회 직후 먼저 삭제하므로 재진입을 포함해 두 번째 소비는 성공하지 않는다.
    this.tickets.delete(ticketHash);
    if (this.now() >= stored.expiresAtMs) return null;
    return stored.userId;
  }

  get storedTicketCount(): number {
    return this.tickets.size;
  }
}

export class PreAuthMessageBuffer {
  private messages: string[] = [];
  private totalBytes = 0;

  constructor(
    private readonly maxMessages: number,
    private readonly maxBytes: number,
    private readonly maxSingleMessageBytes: number
  ) {}

  push(payload: string): void {
    const bytes = Buffer.byteLength(payload, "utf8");
    if (
      bytes > this.maxSingleMessageBytes
      || this.messages.length >= this.maxMessages
      || this.totalBytes + bytes > this.maxBytes
    ) {
      throw new RangeError("pre-auth buffer limit exceeded");
    }
    this.messages.push(payload);
    this.totalBytes += bytes;
  }

  drain(): string[] {
    const drained = this.messages;
    // 배열 소유권을 호출자에게 넘기고 내부 참조와 누적량을 함께 초기화한다.
    this.messages = [];
    this.totalBytes = 0;
    return drained;
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
   - 모범답변: 장기 쿠키 값을 URL에 넣으면 프록시·접근 로그·히스토리·오류 보고에 남을 수 있습니다. 프로젝트는 HTTP 쿠키 인증 뒤 30초짜리 일회용 티켓만 query에 전달해 노출 시 피해 범위와 시간을 줄였습니다.
2. 원문 대신 hash를 저장할 때 공격 표면이 어떻게 줄어드는지
   - 모범답변: 저장소나 진단 정보가 노출돼도 hash만으로는 바로 WebSocket 인증을 재사용할 수 없습니다. 원문은 발급 응답에만 존재하고 서버는 SHA-256 hash를 키로 소비합니다.
3. 애플리케이션의 `select` 후 `delete`가 원자 소비를 보장하지 못하는 이유
   - 모범답변: 두 요청이 삭제 전에 같은 행을 읽으면 둘 다 인증에 성공할 수 있습니다. 실제 PostgreSQL 구현은 조건을 만족하는 행을 `DELETE ... RETURNING` CTE로 한 번만 가져와 동시 요청 중 하나만 사용자 행과 결합되게 합니다.
4. 버전 확인을 티켓 소비보다 먼저 둔 이유
   - 모범답변: 지원하지 않는 client가 유효한 티켓을 소모해 정상 재시도까지 막지 않도록 하기 위해서입니다. 실제 `app.ts`도 query의 `v`와 strict handshake schema를 통과시킨 뒤 hash를 계산하고 소비합니다.
5. 인증 전 메시지 버퍼에서 개수와 바이트를 둘 다 제한한 이유
   - 모범답변: frame 개수만 제한하면 소수의 큰 frame이, 총 바이트만 제한하면 아주 작은 frame 다수가 객체·이벤트 오버헤드를 늘릴 수 있습니다. 그래서 프로젝트는 단일 8KiB, 16개, 누적 32KiB를 모두 제한합니다.

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
  - 모범답변: 실제 권한 상태와 운영 감사 이력이 불일치해 누가 왜 정지했는지 증명할 수 없고, 조사·복구 기준도 사라집니다. 실제 `setUserBan`은 update와 `admin_actions` insert를 같은 트랜잭션에서 실행해 함께 commit하거나 rollback합니다.
- 데이터베이스 commit은 성공했지만 WebSocket 종료가 실패하면 무엇을 진실의 원천으로 봅니까?
  - 모범답변: 데이터베이스의 `banned` 상태가 권한의 진실입니다. 로컬 종료 실패는 commit을 되돌릴 수 없으므로 별도 오류로 관측하고, 이후 인증·티켓 소비에서도 active 상태를 재검사해 새 권한 행사를 막아야 합니다.
- 여러 API 인스턴스에 연결된 사용자를 즉시 끊으려면 무엇이 추가로 필요합니까?
  - 모범답변: 사용자 정지 commit 뒤 revocation 이벤트를 모든 인스턴스에 전달할 pub/sub, 메시지 브로커 또는 transactional outbox가 필요합니다. 각 인스턴스는 자신의 연결 레지스트리에서 해당 사용자를 찾아 같은 정리 절차를 실행합니다.
- 정지 해제도 동일한 방식의 실시간 side effect가 필요합니까?
  - 모범답변: 현재 구현은 ban일 때만 연결을 회수합니다. unban은 이미 끊긴 연결을 복원할 수 없으므로 다음 HTTP 인증이나 새 WebSocket 연결부터 active 상태를 적용하면 되고, 별도 즉시 연결 side effect는 필요하지 않습니다.
- 정지된 사용자가 보유하던 경기 좌석을 즉시 몰수할지 재연결 유예를 둘지 어떤 정책이 필요합니까?
  - 모범답변: 프로젝트의 `revokeUser`는 연결·입력 소유권은 즉시 회수하지만 진행 중 방의 side는 예약 상태로 전환합니다. 일반적으로 보안상 재접속은 금지하되 경기 결과를 즉시 몰수할지 기존 유예·포기 규칙으로 처리할지는 명시적 게임 정책이어야 합니다.

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
  const reason = input.reason.trim();
  if (reason.length === 0 || reason.length > 240) {
    throw new RangeError("ban reason must be between 1 and 240 characters");
  }

  const user = await repo.transaction(async (tx) => {
    const updated = await tx.setStatus(input.targetUserId, "banned");
    await tx.appendAudit({
      actorId: input.actorId,
      targetUserId: input.targetUserId,
      action: "ban",
      reason
    });
    return updated;
  });

  // commit 뒤의 프로세스 메모리 정리는 DB 트랜잭션으로 되돌릴 수 없다.
  revoker.revokeUser(user.id);
  return user;
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
   - 모범답변: 트랜잭션은 사용자 상태 update와 감사 insert가 원자적으로 commit되게 합니다. commit 뒤 WebSocket close, heartbeat·queue 정리, 다른 인스턴스 전파 같은 프로세스 side effect까지 원자화하지는 못합니다.
2. commit 후 WebSocket 회수를 수행하는 순서
   - 모범답변: commit 성공 뒤 현재 client identity를 확인하고 heartbeat와 snapshot 버퍼를 멈춘 다음 queue·tournament waiter·input gate·client map을 정리합니다. 진행 중 방은 side 예약으로 넘기고 socket을 정책 코드로 닫은 뒤 presence를 갱신합니다.
3. 로컬 메모리 레지스트리만으로 다중 인스턴스 권한 회수가 불가능한 이유
   - 모범답변: 각 프로세스는 자기 WebSocket만 알고 다른 인스턴스의 연결 map에는 접근할 수 없습니다. commit된 revocation을 공유 채널로 전달하고, 유실 복구가 중요하면 outbox처럼 DB 변경과 이벤트 기록을 함께 저장해야 합니다.
4. 즉시 좌석 해제와 재연결 유예 사이의 정책 trade-off
   - 모범답변: 즉시 해제는 자원을 빨리 회수하지만 순간 단절에도 경기를 잃게 하고, 유예는 사용자 경험을 높이는 대신 좌석과 방을 더 오래 점유합니다. 정지는 일반 장애보다 강한 정책이므로 재인증은 막되 승패 처리는 게임 규칙으로 분리하는 편이 명확합니다.
5. 감사 로그에 저장할 정보와 저장하지 않을 정보
   - 모범답변: actor, target, ban/unban action, 제한된 길이의 운영 사유와 생성 시각을 저장합니다. 세션·티켓·원본 요청 body·민감 헤더·불필요한 개인정보는 감사 목적에 필요하지 않으므로 저장하지 않습니다.

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
  - 모범답변: 새 연결이 기존 연결을 교체한 뒤 오래된 close callback이 단순 감소를 실행하면 새 연결까지 없는 것으로 계산하거나 카운트가 음수가 될 수 있습니다. 연결별 lease identity를 확인해야 현재 소유자만 release할 수 있습니다.
- `X-Forwarded-For`를 언제 신뢰할 수 있습니까?
  - 모범답변: 요청이 운영자가 관리하는 신뢰 프록시를 반드시 통과하고 Fastify의 `trustProxy`가 그 토폴로지에 맞게 설정됐을 때만 신뢰합니다. 프로젝트는 기본값을 false로 두어 직접 요청이 임의 헤더로 IP 제한을 우회하지 못하게 합니다.
- 서버 재시작 시 게스트 세션과 lease는 어떻게 달라집니까?
  - 모범답변: 서명 secret이 유지되고 쿠키 TTL이 남아 있으면 게스트 신원은 다시 인증될 수 있지만, 메모리의 connection lease·ticket·rate window는 사라집니다. 따라서 메모리 제한은 단일 프로세스의 일시적 자원 상태입니다.
- 게스트와 등록 사용자를 같은 매칭 대기열에 넣지 않은 이유는 무엇입니까?
  - 모범답변: 프로젝트는 게스트를 데모 전용 흐름으로 격리해 영속 rating·전적·채팅·토너먼트에 영향을 주지 않게 합니다. 이는 알고리즘 필수사항이 아니라 제품의 데이터 무결성과 남용 범위를 줄이는 정책입니다.
- 만료된 티켓과 결과 보관 timer를 어떻게 정리해야 메모리 누수가 생기지 않습니까?
  - 모범답변: 소비·교체·명시 삭제 때 기존 timer를 취소하고 양방향 index를 함께 지워야 합니다. 결과 timer callback도 저장된 만료 identity가 여전히 같은 항목인지 확인해 새 결과를 지우지 않으며, timer는 `unref`해 종료를 붙잡지 않습니다.

### 30초 모범 답변

게스트 ID와 만료 시각처럼 위조 방지가 필요한 최소 신원은 HMAC으로 서명하고, 현재 연결 소유권·IP별 사용량·일회용 티켓처럼 계속 변하는 리소스는 서버가 관리했습니다. 연결 교체 때는 게스트별 generation을 올려 새 lease가 현재 소유자가 되게 하고, 오래된 lease의 release는 generation이 맞을 때만 카운트를 줄여 이중 감소를 막습니다. 클라이언트 주소는 프록시를 명시적으로 신뢰할 때만 전달 헤더를 사용하고, 그렇지 않으면 실제 socket 주소를 기준으로 제한합니다.

### 답변 핵심 키워드

HMAC 서명 · 최소 신원 · 서버 소유 resource state · lease generation · stale release 방지 · 전역/IP 제한 · trusted proxy · 데이터 격리 · timer cleanup

### 백지 구현

**구현 목표**

게스트별 활성 연결 lease를 관리하고, 프록시 신뢰 설정에 따라 제한 기준이 되는 클라이언트 주소를 결정한다.

**인터페이스 또는 함수 시그니처**

```ts
import { isIP } from "node:net";

export interface GuestLease {
  readonly guestId: string;
  readonly generation: number;
  release(): void;
}

export class GuestLeaseRegistry {
  private readonly connections = new Map<string, { ip: string; generation: number }>();
  private readonly generations = new Map<string, number>();
  private readonly countsByIp = new Map<string, number>();

  constructor(
    private readonly globalLimit: number,
    private readonly perIpLimit: number
  ) {}

  acquire(ip: string, guestId: string): GuestLease | null {
    const current = this.connections.get(guestId);
    if (!current) {
      if (this.connections.size >= this.globalLimit || this.activeForIp(ip) >= this.perIpLimit) {
        return null;
      }
    } else if (current.ip !== ip && this.activeForIp(ip) >= this.perIpLimit) {
      return null;
    }

    const generation = (this.generations.get(guestId) ?? 0) + 1;
    this.generations.set(guestId, generation);

    if (current?.ip !== ip) {
      if (current) this.decrementIp(current.ip);
      this.countsByIp.set(ip, this.activeForIp(ip) + 1);
    }
    this.connections.set(guestId, { ip, generation });

    let released = false;
    return {
      guestId,
      generation,
      release: () => {
        if (released) return;
        released = true;
        const owned = this.connections.get(guestId);
        // 교체된 lease의 늦은 close callback은 현재 소유권을 건드리지 않는다.
        if (owned?.generation !== generation) return;
        this.connections.delete(guestId);
        this.decrementIp(owned.ip);
      }
    };
  }

  get activeConnectionCount(): number {
    return this.connections.size;
  }

  activeForIp(ip: string): number {
    return this.countsByIp.get(ip) ?? 0;
  }

  private decrementIp(ip: string): void {
    const next = this.activeForIp(ip) - 1;
    if (next > 0) this.countsByIp.set(ip, next);
    else this.countsByIp.delete(ip);
  }
}

export function resolveClientAddress(input: {
  remoteAddress: string;
  forwardedFor?: string;
  trustProxy: boolean;
}): string {
  if (!input.trustProxy || !input.forwardedFor) return input.remoteAddress;
  const first = input.forwardedFor.split(",", 1)[0]?.trim() ?? "";
  // 신뢰 프록시를 켰더라도 첫 hop이 IP 형식이 아니면 socket 주소로 안전하게 후퇴한다.
  return isIP(first) !== 0 ? first : input.remoteAddress;
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
   - 모범답변: 게스트 ID·사용자 표시값·IP·만료 시각처럼 재인증에 필요한 최소 payload는 HMAC 쿠키에 넣고, 연결 소유권·IP별 사용량·티켓·rate window처럼 매 요청 변하는 상태는 서버가 소유합니다.
2. 단순 카운터 대신 lease와 generation을 사용한 이유
   - 모범답변: 연결 교체와 close callback 순서는 뒤바뀔 수 있습니다. generation 또는 실제 구현의 무작위 `leaseId`를 비교하면 현재 lease만 map과 IP 카운트를 해제하고 오래된 callback은 무효가 됩니다.
3. 프록시 신뢰 설정이 보안 설정인 이유
   - 모범답변: client IP는 rate limit과 per-IP capacity의 보안 키입니다. 신뢰 프록시 없이 전달 헤더를 받으면 공격자가 원하는 주소를 써 제한을 분산할 수 있으므로, 기본 false와 명시적 인프라 설정이 필요합니다.
4. 게스트 리소스를 메모리에 둘 때의 장점과 재시작 시 한계
   - 모범답변: 단일 데모 인스턴스에서는 O(1) map 연산으로 빠르고 영속 데이터와 게스트를 쉽게 격리합니다. 재시작하면 lease·ticket·window가 사라지고 여러 인스턴스의 전역 제한도 보장하지 못하므로 규모가 커지면 공유 저장소가 필요합니다.
5. 게스트·등록 사용자 데이터와 매칭을 격리한 정책적 이유
   - 모범답변: 게스트는 검증된 영속 계정이 아니므로 rating, 전적, 토너먼트, 로비 채팅을 오염시키지 않게 데모 AI/PvP 흐름에 한정했습니다. 일반 원칙이라기보다 이 프로젝트의 체험 모드 격리 정책입니다.

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
