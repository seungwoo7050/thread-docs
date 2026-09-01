# HTTP 계약과 외부 I/O 면접 워크북

Thread 01·02·12에서 반복된 HTTP 경계를 한 묶음으로 정리했다. 외부 입력과 외부 응답은 모두 신뢰하지 않으며, 네트워크 실행은 단일 종료·명시적 자원 상한·정확한 실패 의미를 유지하는지가 핵심이다.

## 문서 내 면접 포인트

- [P01 비동기 HTTP Check의 단일 종료와 자원 정리](#p01)
- [P02 `unknown` 입력을 도메인 값으로 좁히는 런타임 검증](#p02)
- [P03 신뢰하지 않는 API 응답의 방어적 디코딩](#p03)
- [P04 URL 정규화, DNS 검증, redirect 재검증으로 SSRF 차단](#p04)
- [P05 connect/read/total 예산과 redirect를 포함한 자원 상한](#p05)
- [P06 관찰 결과와 재시도 가능성의 분리](#p06)

---

<a id="p01"></a>
## [Thread 01 (E01) / `메모리 Monitor에서 동기 fixture Check를 실행`] 비동기 HTTP Check의 단일 종료와 자원 정리

### 면접 질문

- `checkMonitor`처럼 응답, timeout, transport error가 경쟁하는 코드에서 결과가 정확히 한 번만 확정되도록 어떻게 설계하겠습니까?
  - 꼬리 질문: 응답 헤더를 받은 직후 본문을 읽지 않는 요구가 있다면 request, response, timer를 어느 시점에 정리해야 합니까?
  - 꼬리 질문: timeout 직후 `error` 이벤트가 추가로 오거나, 응답 직후 timer가 발화해도 상태가 뒤집히지 않는다는 것을 어떻게 검증합니까?

### 30초 모범 답변

네트워크 이벤트는 서로 배타적이지 않으므로 `response`, `timeout`, `error`를 각각 처리하는 것보다 하나의 종료 함수로 모으고, 그 함수에 단일 정착 invariant를 둡니다. 최초 종료만 결과를 만들고 timer를 해제하며 request와 response를 닫습니다. 이 프로젝트는 최종 헤더만 관찰하므로 본문을 보관하지 않고 응답을 즉시 종료합니다. 핵심 경계는 timeout 뒤 error, 응답 뒤 timeout 같은 이중 이벤트와 정리 중 예외이며, 어떤 순서에서도 결과와 자원 상태가 한 번만 확정돼야 합니다.

### 답변 핵심 키워드

- single settlement
- 이벤트 경쟁
- idempotent cleanup
- timer 해제
- request/response destroy
- 헤더만 관찰
- transport failure와 HTTP failure 구분

### 백지 구현

#### 구현 목표

주입 가능한 전송 계층을 사용해, 응답·timeout·연결 오류 중 먼저 확정된 사건 하나만 결과로 반환하는 헤더 관찰 함수를 작성한다.
#### 인터페이스 또는 함수 시그니처

`observeHeaders(url, timeoutMs, connector): Promise<Observation>`
#### 입력과 출력

- `url`: 절대 HTTP(S) URL
- `timeoutMs`: 전체 제한 시간
- `connector`: 테스트에서 응답·오류·종료 순서를 제어할 수 있는 전송 어댑터
- `Observation`: 상태, HTTP 상태 또는 `null`, 실패 이유 또는 `null`, 0 이상 지연 시간
#### 반드시 만족해야 할 조건

- 2xx는 성공, 관찰한 비-2xx는 HTTP 실패로 기록한다.
- 응답을 받지 못한 timeout·연결 오류는 HTTP 상태를 `null`로 둔다.
- Promise와 결과 객체는 정확히 한 번만 확정한다.
- 모든 종료 경로에서 timer와 열린 request/response를 정리한다.
- 최종 응답 본문은 애플리케이션 데이터로 소비하거나 보관하지 않는다.
#### 경계 조건

- timeout 처리 직후 `error`가 도착한다.
- 응답 처리 직후 timeout 콜백이 실행 대기 중이다.
- 상태 코드가 없는 응답 객체가 들어온다.
- 지연 시간이 0ms로 측정된다.
#### 실패 조건

- 두 이벤트가 서로 다른 결과를 덮어쓴다.
- Promise가 미정 상태로 남는다.
- timer, request, response 중 하나가 종료 뒤에도 살아 있다.
#### 필요한 제약

- 자동 재시도와 redirect 추적은 구현하지 않는다.
- 전역 상태나 실제 외부 네트워크에 의존하지 않는다.

```ts
type Observation = {
  state: 'SUCCEEDED' | 'FAILED';
  httpStatus: number | null;
  failureReason: 'HTTP_STATUS' | 'TIMEOUT' | 'CONNECTION_FAILURE' | null;
  latencyMs: number;
};

export async function observeHeaders(
  url: URL,
  timeoutMs: number,
  connector: Connector,
): Promise<Observation> {
  // 직접 구현
}
```

### 구현 후 자가 검증

- 200과 204가 성공으로 한 번만 확정된다.
- 302와 503은 관찰한 상태를 유지한 HTTP 실패가 된다.
- timeout과 error가 연속 발생해도 최초 결과가 유지된다.
- 응답과 timeout 순서를 뒤집어 실행해도 자원 누수가 없다.
- 본문 chunk가 커도 애플리케이션 소비량이 늘지 않는다.
- 실패 경로에서도 timer와 전송 객체가 정리된다.

### 구현 후 설명할 것

- 종료 로직을 여러 이벤트 핸들러에 복제하지 않고 한곳으로 모은 이유
- `AbortSignal` 또는 직접 destroy 방식 중 선택한 기준
- HTTP 실패와 transport 실패를 별도 상태로 둔 이유
- 본문을 읽지 않는 방식의 장점과 TCP 버퍼링까지 0이라고 말할 수 없는 이유

### 원본 확인 위치

- Thread 01
- `server/check.ts` — `checkMonitor`
- `server/main.ts` — 프로세스 종료 시 Fastify 정리
- `server/model.ts` — CheckRun 결과 필드
- 관련 Thread: 12(E12) 외부 목적지 검증과 다중 timeout
---

<a id="p02"></a>
## [Thread 02 (E02) / 제목 미노출 — 기록상 HTTP 런타임 계약과 오류 경계 구현] `unknown` 입력을 도메인 값으로 좁히는 런타임 검증

### 면접 질문

- TypeScript 타입이 있어도 Fastify 경계에서 `unknown`을 다시 검증해야 하는 이유는 무엇입니까?
  - 꼬리 질문: `monitorInput`에서 문자열 숫자나 문자열 boolean을 허용하지 않고 명시적 타입을 요구한 이유는 무엇입니까?
  - 꼬리 질문: 파싱 실패와 존재하지 않는 자원, 서버 내부 실패를 어떤 기준으로 다른 오류 코드에 배치합니까?

### 30초 모범 답변

TypeScript 타입은 컴파일 시점 약속이고 네트워크로 들어온 JSON의 실제 형태를 보장하지 않습니다. 그래서 HTTP 경계에서 객체 여부, 필수 필드 타입, trim 이후 이름 길이, 정수 범위, URL 정책을 검증한 뒤 도메인 값으로 넘깁니다. 문자열을 숫자나 boolean으로 암묵 변환하면 클라이언트 실수를 정상 입력처럼 저장할 수 있어 계약이 흐려집니다. 잘못된 표현은 `INVALID_INPUT`, 형식은 맞지만 자원이 없으면 `NOT_FOUND`, 예상하지 못한 실패는 내부 세부사항을 숨긴 `INTERNAL_ERROR`로 나눕니다.

### 답변 핵심 키워드

- compile-time vs runtime
- unknown narrowing
- 명시적 타입
- 무암묵 변환
- 오류 taxonomy
- side effect before validation 금지
- 안정된 wire contract

### 백지 구현

#### 구현 목표

외부 JSON 값을 검증해 정규화된 Monitor 입력을 반환하고, 잘못된 값은 안정된 입력 오류로 거부한다.
#### 인터페이스 또는 함수 시그니처

`parseMonitorInput(value: unknown): MonitorInput`
#### 입력과 출력

- 입력: 임의의 JSON 호환 값
- 출력: trim된 이름, 정규화된 URL 문자열, 정수 interval, boolean enabled
- 실패: 호출자가 HTTP 400으로 변환할 수 있는 도메인 입력 오류
#### 반드시 만족해야 할 조건

- 객체만 허용하고 `null`, 배열, 원시값은 거부한다.
- 필수 필드의 타입을 직접 검사하며 문자열 숫자·문자열 boolean을 변환하지 않는다.
- 이름은 trim 후 빈 값이 아니어야 하며 정해진 최대 길이를 넘지 않는다.
- interval은 유한한 정수이며 허용 범위 안이어야 한다.
- URL 검증이 실패하면 저장·외부 요청 같은 부작용이 시작되지 않는다.
#### 경계 조건

- 공백만 있는 이름
- 최소·최대 길이 바로 안쪽과 바깥쪽
- `0`, 음수, 소수, `NaN`, 매우 큰 수
- `false`, `0`, 빈 문자열처럼 falsy인 정상·비정상 값의 구분
#### 실패 조건

- 알 수 없는 예외를 그대로 노출한다.
- 부분 검증 뒤 저장을 시작한다.
- 입력 객체를 직접 변경한다.
#### 필요한 제약

- 프레임워크 스키마 기능 없이 표준 TypeScript/JavaScript만으로 구현해도 된다.
- 오류 메시지 문구보다 오류 범주와 부작용 없음이 우선이다.

```ts
type MonitorInput = {
  name: string;
  url: string;
  interval: number;
  enabled: boolean;
};

export function parseMonitorInput(value: unknown): MonitorInput {
  // 직접 구현
}
```

### 구현 후 자가 검증

- `null`, 배열, 누락 필드, 잘못된 타입을 모두 거부한다.
- 이름 경계값과 interval 경계값을 각각 바로 안/밖에서 확인한다.
- `enabled: false`를 누락으로 오인하지 않는다.
- 유효 입력에서 원본 객체가 변경되지 않는다.
- 모든 거부 사례에서 저장소 호출과 외부 I/O 호출 횟수가 0이다.
- 오류 종류가 형식 오류와 내부 오류를 혼동하지 않는다.

### 구현 후 설명할 것

- 타입 단언 대신 단계별 narrowing을 사용한 이유
- 암묵 변환을 금지한 계약상의 이점
- 정규화와 검증의 순서를 정한 기준
- 검증 실패 전까지 부작용을 시작하지 않는 이유

### 원본 확인 위치

- Thread 02
- `server/contracts.ts` — `ApiError`, `monitorInput`, `monitorId`, `ERROR_STATUS`
- `server/app.ts` — HTTP 경계와 오류 변환
- `test/contracts.test.ts`
- 관련 Thread: 03(E03) NUL 입력의 저장 전 분류, 12(E12) URL 정책
---

<a id="p03"></a>
## [Thread 02 (E02) / 제목 미노출 — 기록상 브라우저 API 응답 계약 구현] 신뢰하지 않는 API 응답의 방어적 디코딩

### 면접 질문

- `responseData<T>`가 HTTP status만 보거나 JSON의 `code`만 보는 대신 둘을 함께 검증해야 하는 이유는 무엇입니까?
  - 꼬리 질문: 성공 응답의 `data`가 `false`, `0`, `null`, 빈 배열일 때도 정확히 보존하려면 어떤 검사가 필요합니까?
  - 꼬리 질문: 서버가 HTML 오류 페이지나 깨진 JSON을 반환하면 브라우저는 어떤 안전한 실패로 수렴해야 합니까?

### 30초 모범 답변

클라이언트도 네트워크 반대편의 응답을 신뢰하면 안 됩니다. 성공은 성공 status와 `{data}` envelope가 함께 맞아야 하고, 실패는 허용된 오류 code가 예상 status와 일치해야 합니다. 그렇지 않으면 프록시나 서버 버그가 잘못된 도메인 상태로 들어올 수 있습니다. `data`는 truthiness가 아니라 키의 존재로 판정해야 `false`, `0`, `null`을 잃지 않습니다. JSON 파싱 실패, status/code 불일치, 혼합 envelope는 모두 사용자에게 내부 세부를 드러내지 않는 안정된 서비스 실패로 바꿉니다.

### 답변 핵심 키워드

- defensive decoder
- status/code correlation
- envelope validation
- key presence
- falsy preservation
- fail closed
- server prose 비의존

### 백지 구현

#### 구현 목표

Fetch `Response`를 성공 데이터 또는 안정된 `RequestFailure`로 변환하는 디코더를 작성한다.
#### 인터페이스 또는 함수 시그니처

`responseData<T>(response: Response): Promise<T>`
#### 입력과 출력

- 입력: 임의의 HTTP status, content-type, JSON 또는 비-JSON 본문을 가진 Response
- 출력: 성공 envelope의 `data` 값 그대로
- 실패: 허용된 오류 code만 담은 클라이언트 오류
#### 반드시 만족해야 할 조건

- 성공 status에는 `{ data: ... }`만 허용하고 오류 필드가 함께 있으면 거부한다.
- 실패 status에는 `{ error: { code, message } }` 형태와 code/status 대응을 함께 검증한다.
- `false`, `0`, `null`, `[]`를 유효 데이터로 보존한다.
- 본문 파싱 실패나 알 수 없는 code는 안정된 내부 서비스 오류가 된다.
- 서버의 자유 형식 오류 문장을 사용자 메시지로 직접 사용하지 않는다.
#### 경계 조건

- 204처럼 본문이 없는 성공 status
- 200인데 오류 envelope인 응답
- 400인데 `INTERNAL_ERROR` code인 응답
- data와 error가 동시에 있는 객체
- 배열·문자열·HTML 본문
#### 실패 조건

- 잘못된 응답을 타입 단언으로 통과시킨다.
- truthiness 검사로 정상 falsy 값을 누락한다.
- 서버 예외 문구나 스택을 그대로 표시한다.
#### 필요한 제약

- 런타임 스키마 라이브러리 없이 구현 가능해야 한다.
- generic `T`는 컴파일 편의일 뿐 실제 payload 구조 전체를 검증한다고 주장하지 않는다.

```ts
export class RequestFailure extends Error {
  readonly code: ApiErrorCode;
}

export async function responseData<T>(response: Response): Promise<T> {
  // 직접 구현
}
```

### 구현 후 자가 검증

- 정상 객체, 빈 배열, `false`, `0`, `null`이 손실 없이 반환된다.
- 깨진 JSON과 HTML 오류 페이지가 안전한 실패가 된다.
- status/code 불일치가 거부된다.
- 성공·오류 키 혼합 envelope가 거부된다.
- 알 수 없는 서버 문구가 UI 메시지에 노출되지 않는다.
- 브라우저 transport rejection도 같은 안정된 범주로 수렴한다.

### 구현 후 설명할 것

- HTTP status와 응답 code를 중복 검증하는 이유
- `Object.hasOwn` 계열의 키 존재 검사가 필요한 이유
- generic 타입과 런타임 검증 범위의 차이
- 서버 문구와 사용자 문구를 분리한 이유

### 원본 확인 위치

- Thread 02
- `app/monitors/api.ts` — `RequestFailure`, `responseData`, `failureCode`
- `test/unit.test.ts`
- `test/browser/contracts.spec.ts`
- 관련 Thread: 10(E10) `CONFLICT` 오류 추가
---

<a id="p04"></a>
## [Thread 12 (E12) / `feat: validate outbound destinations and bound check resources`] URL 정규화, DNS 검증, redirect 재검증으로 SSRF 차단

### 면접 질문

- 사용자가 입력한 URL의 문자열 검사만으로 SSRF를 막을 수 없는 이유와, 이 프로젝트가 실제 연결 직전까지 검증한 경계를 설명해 보세요.
  - 꼬리 질문: DNS 응답 중 하나라도 사설 주소가 섞여 있으면 왜 전체 목적지를 거부했습니까?
  - 꼬리 질문: 검증한 hostname을 HTTP 클라이언트가 다시 resolve하게 두지 않고 숫자 주소에 직접 연결한 이유는 무엇입니까?

### 30초 모범 답변

URL 문자열만 검사하면 hostname이 DNS에서 사설 주소로 풀리거나, redirect가 내부 주소로 바뀌거나, 검증 뒤 재해석되는 문제가 남습니다. 그래서 먼저 HTTP(S), credential 금지, canonical host 같은 구문 정책을 적용하고, 실행 시 DNS의 모든 주소를 공인 범위로 검증합니다. 검증한 숫자 주소로 직접 연결해 두 번째 DNS 조회를 막되 원래 Host와 TLS server name은 유지합니다. redirect마다 같은 검증을 다시 하고, 혼합 공인·사설 응답은 fail-closed로 거부해 어떤 주소가 선택되더라도 안전하다는 invariant를 유지합니다.

### 답변 핵심 키워드

- SSRF
- canonical URL
- DNS rebinding
- all-address validation
- numeric-address pinning
- Host/SNI 보존
- redirect revalidation
- fail closed

### 백지 구현

#### 구현 목표

URL을 정규화하고 DNS 결과를 검증해 실제 연결에 사용할 목적지를 반환하는 순수한 정책 경계를 작성한다.
#### 인터페이스 또는 함수 시그니처

`validateDestination(rawUrl, resolve, policy): Promise<Destination>`
#### 입력과 출력

- 입력: URL 문자열, hostname resolver, 공인 주소 판정 함수
- 출력: 정규화된 URL과 검증된 숫자 주소·주소군
- 실패: 안전하지 않은 목적지라는 정책 오류
#### 반드시 만족해야 할 조건

- HTTP와 HTTPS만 허용하고 username/password가 있는 URL을 거부한다.
- fragment는 실행 목적지에서 제거하고 hostname의 동치 표기를 정규화한다.
- literal IP와 DNS 결과 모두 같은 주소 정책을 통과해야 한다.
- DNS가 반환한 모든 주소가 허용돼야 하며 빈 결과·family 불일치는 거부한다.
- 반환된 숫자 주소를 연결에 사용하고 사용자 hostname을 다시 resolve하지 않는다.
- redirect 호출자는 매 hop마다 이 함수를 다시 사용해야 한다.
#### 경계 조건

- `localhost`, trailing dot, IPv6 bracket, zone identifier
- IPv4-mapped IPv6와 전환 주소
- 공인 주소와 사설 주소가 섞인 DNS 응답
- 첫 DNS 응답과 두 번째 DNS 응답이 다른 rebinding 상황
- credential을 숨긴 비표준 URL 표현
#### 실패 조건

- 검증 뒤 hostname으로 다시 연결한다.
- 첫 주소만 안전하면 나머지 주소를 무시한다.
- redirect `Location`을 검증 없이 따라간다.
- 정책 오류 메시지나 로그에 전체 민감 URL을 남긴다.
#### 필요한 제약

- 실제 인터넷이나 metadata endpoint에 연결하지 않고 resolver/connector를 주입해 검증한다.
- 공인 주소 판정의 완전성을 과장하지 말고 정책 범위를 명시한다.

```ts
type ResolvedAddress = { address: string; family: 4 | 6 };
type Destination = { url: URL; address: string; family: 4 | 6 };

export async function validateDestination(
  rawUrl: string,
  resolve: (hostname: string) => Promise<readonly ResolvedAddress[]>,
  isAllowedAddress: (address: string, family: 4 | 6) => boolean,
): Promise<Destination> {
  // 직접 구현
}
```

### 구현 후 자가 검증

- HTTP(S) 공인 목적지는 허용되고 credential·비-HTTP scheme은 거부된다.
- loopback, private, link-local, multicast, reserved 주소를 실제 connector 호출 전에 거부한다.
- 혼합 DNS 결과에서 connector 호출이 0회다.
- 검증 후 resolver가 다시 호출되지 않는다.
- redirect가 사설 주소를 가리키면 첫 hop 이후 내부 연결 없이 중단된다.
- 정규화된 URL은 동등 입력에 대해 일관된 문자열을 만든다.

### 구현 후 설명할 것

- 문자열 검증과 실행 시 DNS 검증을 분리한 이유
- 모든 DNS 주소를 검사하는 보수적 정책의 가용성 trade-off
- 숫자 주소 연결과 Host/SNI 보존의 역할
- 테스트용 loopback 예외를 production 기본값과 분리해야 하는 이유

### 원본 확인 위치

- Thread 12 (E12)
- `server/outbound.ts` — `canonicalUrl`, `validatedDestination`, `publicAddress`, `connectDestination`, `configuredTestFixtureOrigin`
- `server/check.ts` — redirect마다 목적지 검증
- `test/outbound.test.ts`
- 관련 Thread: 01(E01) 초기 fixture allowlist
---

<a id="p05"></a>
## [Thread 12 (E12) / `feat: validate outbound destinations and bound check resources`] connect/read/total 예산과 redirect를 포함한 자원 상한

### 면접 질문

- connect timeout, read timeout, total timeout을 따로 두면 각각 어떤 실패를 제한할 수 있습니까?
  - 꼬리 질문: redirect마다 timer를 새로 시작하면 전체 실행 시간이 무한히 늘 수 있는데, 이를 어떻게 막아야 합니까?
  - 꼬리 질문: 최종 응답 본문을 읽지 않는 Monitor에서 65KB 상한을 어떻게 해석해야 하며, 무엇을 주장하면 안 됩니까?

### 30초 모범 답변

단계별 timeout은 DNS·TCP·TLS 연결 정체와 응답 헤더 대기를 각각 제한하고, total timeout은 redirect 전체를 포함한 상한을 보장합니다. 각 hop의 제한만 재시작하면 hop 수만큼 시간이 늘기 때문에 시작 시각 하나와 남은 전체 예산을 함께 봐야 합니다. 프로젝트는 최종 헤더만 필요하므로 응답 본문을 애플리케이션 데이터로 소비하지 않고 즉시 닫습니다. 이는 애플리케이션 소비량이 0이라는 뜻이지 커널이나 Node 내부 버퍼에 바이트가 전혀 들어오지 않았다는 뜻은 아닙니다.

### 답변 핵심 키워드

- connect/read/total budget
- deadline propagation
- redirect hop limit
- remaining budget
- bounded cleanup
- header-only observation
- 자원 상한의 정확한 주장

### 백지 구현

#### 구현 목표

최대 redirect 수와 단계별·전체 deadline을 지키며 헤더만 관찰하는 Check 실행기를 작성한다.
#### 인터페이스 또는 함수 시그니처

`executeBoundedCheck(startUrl, limits, deps): Promise<TerminalCheck>`
#### 입력과 출력

- `limits`: connect, read, total 제한과 최대 redirect 수
- `deps`: 시계, 목적지 검증기, connector
- 출력: 성공·HTTP 실패·transport 실패·정책 중단 중 하나의 terminal 결과
#### 반드시 만족해야 할 조건

- 하나의 전체 deadline이 모든 redirect hop을 포함한다.
- connect와 read 단계가 각자 정해진 예산을 넘지 않는다.
- redirect 수가 상한을 넘으면 정책 중단으로 종료한다.
- 최종 헤더를 관찰한 뒤 본문을 저장하지 않고 response를 닫는다.
- 모든 timer와 소켓 관련 핸들을 성공·실패 모두에서 정리한다.
- 자동 재시도는 하지 않는다.
#### 경계 조건

- 각 redirect가 total deadline 직전까지 지연된다.
- connect 성공 직후 read timeout이 발생한다.
- 최종 응답과 total timeout이 거의 동시에 도착한다.
- redirect `Location`이 상대 URL이다.
- 큰 2xx 본문이 헤더 뒤에 계속 전송된다.
#### 실패 조건

- hop마다 total timer를 초기화한다.
- redirect 제한을 상태 코드 관찰 뒤에도 무시한다.
- 정책 중단을 transport 실패로 잘못 저장한다.
- 정리 대기 때문에 전체 상한을 무한히 넘긴다.
#### 필요한 제약

- 테스트는 가짜 시계 또는 제어 가능한 connector를 사용한다.
- 실행 결과와 재시도 여부 결정은 분리한다.

```ts
type Limits = {
  connectMs: number;
  readMs: number;
  totalMs: number;
  maxRedirects: number;
};

export async function executeBoundedCheck(
  startUrl: string,
  limits: Limits,
  deps: CheckDependencies,
): Promise<TerminalCheck> {
  // 직접 구현
}
```

### 구현 후 자가 검증

- redirect가 없어도 connect/read/total 각각의 timeout 경로를 재현한다.
- 여러 hop의 누적 시간이 total 제한을 넘지 않는다.
- 허용 hop 수와 한 단계 초과 경계를 확인한다.
- 정상 2xx와 큰 본문에서 body 저장량이 0이다.
- 모든 종료 뒤 열린 request/socket 집합이 비어 있다.
- 응답·timeout·error 순서를 바꿔도 terminal 결과가 하나뿐이다.

### 구현 후 설명할 것

- 단계별 timeout과 전체 deadline을 함께 둔 이유
- 남은 예산 계산 방식과 시계 선택
- redirect 상한과 정책 실패 상태를 별도로 둔 이유
- 본문 소비량과 네트워크 수신량을 구분해 설명해야 하는 이유

### 원본 확인 위치

- Thread 12 (E12)
- `server/outbound.ts` — `CONNECT_TIMEOUT_MS`, `READ_TIMEOUT_MS`, `TOTAL_TIMEOUT_MS`, `MAX_REDIRECTS`
- `server/check.ts` — `checkMonitor`
- `test/outbound.test.ts`
- 관련 Thread: 01(E01) 단일 timeout과 단일 종료
---

<a id="p06"></a>
## [Thread 12 (E12) / `feat: validate outbound destinations and bound check resources`] 관찰 결과와 재시도 가능성의 분리

### 면접 질문

- 왜 HTTP 결과를 저장하는 함수가 자동 재시도까지 수행하지 않고 `failureDisposition` 같은 분류만 반환하게 했습니까?
  - 꼬리 질문: 404, 429, 503, timeout, unsafe destination, worker 불확실성을 각각 어떻게 분류하며 그 근거는 무엇입니까?
  - 꼬리 질문: `ABORTED`인데 정책 이유가 없는 결과를 영구 실패로 단정하면 어떤 문제가 생깁니까?

### 30초 모범 답변

관찰 결과와 재시도 정책을 분리해야 원본 사실이 보존되고 운영 정책을 나중에 바꿀 수 있습니다. 최종 2xx는 재시도 없음, 429·5xx와 transport timeout·연결 실패는 재시도 가능, 다른 최종 HTTP와 안전 정책 위반은 영구 실패로 볼 수 있습니다. 반면 worker crash처럼 요청이 실행됐는지 확정할 수 없는 `ABORTED`는 불확실 상태입니다. 이 분류는 힌트일 뿐 자동 재시도, terminal reopening, 중복 요청을 즉시 수행하지 않습니다.

### 답변 핵심 키워드

- observation vs policy
- retryable/permanent/uncertain
- 원본 사실 보존
- 429/5xx
- 정책 실패
- 불확실성
- no automatic retry

### 백지 구현

#### 구현 목표

terminal Check 결과를 재시도 가능성 범주로 분류하는 순수 함수를 작성한다.
#### 인터페이스 또는 함수 시그니처

`failureDisposition(result: TerminalCheck): Disposition`
#### 입력과 출력

- 입력: 성공, HTTP 실패, transport 실패, 정책 중단, 원인 미확정 중 하나의 terminal 결과
- 출력: `none | retryable | permanent | uncertain`
#### 반드시 만족해야 할 조건

- 성공은 `none`이다.
- 429와 5xx HTTP 실패는 `retryable`이다.
- 그 밖의 관찰된 비-2xx HTTP 실패는 `permanent`이다.
- timeout과 연결 실패는 `retryable`이다.
- 안전하지 않은 목적지와 redirect 한도는 `permanent`이다.
- 원인 없는 중단은 `uncertain`이다.
#### 경계 조건

- HTTP 상태가 `null`인데 실패 이유가 HTTP_STATUS인 모순
- 2xx인데 실패 상태인 모순
- ABORTED에 정책 이유가 있는 경우와 없는 경우
- 599 같은 비표준 5xx
#### 실패 조건

- 모순된 입력을 임의 분류해 숨긴다.
- 분류 함수 안에서 실제 재시도나 상태 변경을 수행한다.
#### 필요한 제약

- 함수는 순수해야 하며 시간, 네트워크, 저장소에 접근하지 않는다.
- 모순된 상태는 명시적으로 거부하거나 `uncertain`으로 처리하는 정책을 설명한다.

```ts
type Disposition = 'none' | 'retryable' | 'permanent' | 'uncertain';

export function failureDisposition(result: TerminalCheck): Disposition {
  // 직접 구현
}
```

### 구현 후 자가 검증

- 2xx, 404, 429, 503, timeout, 연결 실패를 각각 확인한다.
- 정책 ABORTED와 원인 없는 ABORTED를 구분한다.
- 모순된 상태 조합을 조용히 정상 처리하지 않는다.
- 함수 호출 전후 입력 객체가 변하지 않는다.
- 분류 결과만 나오며 네트워크·DB 호출은 0이다.

### 구현 후 설명할 것

- 재시도 실행과 재시도 가능성 분류를 분리한 이유
- HTTP 상태별 기본 정책이 절대 규칙이 아닌 이유
- `uncertain`을 별도 범주로 둔 이유
- 멱등성 보장 없이 자동 재시도를 추가할 때 생기는 위험

### 원본 확인 위치

- Thread 12 (E12)
- `server/check.ts` — `failureDisposition`
- `server/model.ts` — `PolicyFailureReason`, `TerminalCheckRun`
- `server/migrations/009_outbound_policy_result.sql`
- 관련 Thread: 10(E10) 요청 멱등성, 11(E11) crash recovery
