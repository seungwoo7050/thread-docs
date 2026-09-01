# 상태 불변조건과 권한 경계

이 문서는 등록 상태, nickname 보조 색인, channel 내부 집합, 연결 종료 정리, 권한 검사와 메시지 fan-out을 하나의 상태 정합성 관점에서 다룬다.

## [Thread 05 / `feat(client): 닉네임 색인 관리` · `feat(registration): USER 정보와 환영 응답 연결`] P07. 등록 상태 기계와 nickname 보조 색인

### 면접 질문

IRC 등록은 `PASS`, `NICK`, `USER`가 모두 충족돼야 완료되지만 명령 도착 순서는 고정되어 있지 않습니다. 이 프로젝트의 `ClientState`와 `ClientRegistry`가 등록 완료를 한 번만 발생시키고, nickname의 대소문자 비구분 충돌과 변경을 일관되게 처리하려면 어떤 invariant가 필요합니까?

꼬리 질문:

- nickname 변경 시 기존 색인을 먼저 지우고 새 색인을 넣는 사이에 실패하면 어떻게 합니까?
- 같은 사용자가 대소문자만 바꾼 nickname으로 변경하는 경우를 어떻게 다룹니까?
- `fd -> ClientState`와 `canonical nickname -> fd` 중 어느 쪽을 source of truth로 봅니까?
- 등록 완료 응답을 queue하는 데 실패해 연결이 제거됐다면 `registered=true`나 등록 로그를 남겨도 됩니까?
- PASS가 틀렸을 때 즉시 close하는 정책과 재시도를 허용하는 정책의 차이는 무엇입니까?

### 30초 모범 답변

연결 상태는 PASS 성공, nickname 보유, USER 보유, 등록 완료를 독립 flag로 두고, 매 관련 명령 뒤 "세 조건이 모두 참이고 아직 registered가 아님"일 때만 등록 전이를 시도합니다. nickname 검색은 canonical form을 key로 쓰되 실제 표시 문자열은 `ClientState`에 둡니다. 변경은 새 이름의 유효성·충돌을 먼저 검증한 뒤 기존 색인 제거와 새 색인 삽입, 상태 갱신을 하나의 논리적 연산으로 수행해야 합니다. 등록 응답 송신이 연결 제거를 일으킬 수 있으므로 성공 여부를 확인하기 전에는 완료 상태나 로그를 확정하지 않는 것이 안전합니다.

### 답변 핵심 키워드

부분 상태, 순서 독립 등록, one-shot transition, source of truth, canonical nickname, secondary index, 원자적 색인 갱신, 송신 실패 경계, commit 시점

### 백지 구현

**구현 목표**

연결별 등록 상태와 대소문자 비구분 nickname 보조 색인을 관리한다. nickname 변경은 충돌 시 기존 상태를 보존해야 한다.

**면접용 축소 인터페이스**

```cpp
#include <optional>
#include <string>
#include <unordered_map>

struct ClientState {
    bool passOk = false;
    bool hasNick = false;
    bool hasUser = false;
    bool registered = false;
    std::string nick;
    std::string user;
};

enum class NickUpdate {
    Applied,
    Invalid,
    InUse,
    MissingClient,
};

enum class RegistrationDecision {
    NotReady,
    AlreadyRegistered,
    ReadyToCommit,
};

class ClientRegistry {
public:
    ClientState& create(int fd);
    ClientState* find(int fd);
    const ClientState* find(int fd) const;

    NickUpdate setNickname(int fd, const std::string& nickname);
    std::optional<int> findByNickname(const std::string& nickname) const;
    void erase(int fd);

    RegistrationDecision registrationDecision(int fd) const;
    bool commitRegistration(int fd);

private:
    static std::string canonicalNickname(const std::string& nickname);
    static bool validNickname(const std::string& nickname);

    std::unordered_map<int, ClientState> clients_;
    std::unordered_map<std::string, int> nicknameIndex_;
};
```

**입력과 출력**

- 입력: fd, nickname, PASS/USER 상태 변경
- 출력: nickname 갱신 결과, lookup 결과, 등록 가능 상태
- 내부 상태: primary map과 canonical nickname index

**반드시 만족해야 할 조건**

- index의 모든 항목은 존재하는 client를 가리킨다.
- nickname을 가진 모든 client는 canonical index에서 정확히 한 번 검색된다.
- 대소문자만 다른 nickname은 충돌로 본다. 단, 같은 fd의 자기 이름 변경 정책은 별도로 명시한다.
- nickname 변경 실패 시 기존 nick과 index가 그대로다.
- `erase(fd)`는 client와 그 nickname index를 함께 제거한다.
- 등록은 세 선행 조건이 모두 참일 때만 commit할 수 있다.
- 이미 등록된 client에 대한 commit은 중복 전이를 만들지 않는다.

**경계 조건**

- 존재하지 않는 fd
- 빈 nickname, 너무 긴 nickname, 허용하지 않는 첫 문자
- 같은 이름 재설정
- 대소문자만 변경
- 다른 fd가 canonical-equivalent 이름을 사용 중
- nickname 없는 client 삭제
- PASS/NICK/USER의 여섯 가지 순서
- 등록 commit 두 번 호출

**실패 조건**

- index 충돌
- nickname 검증 실패
- 존재하지 않는 client 변경
- 등록 준비가 되기 전 commit 요청

**제약**

- 25~30분 안에 구현한다.
- canonicalization 규칙은 면접 문제에서 정의한 ASCII 범위로 제한해도 된다.
- index 전체를 매 lookup마다 선형 탐색하지 않는다.
- 등록 환영 메시지 자체는 구현하지 않고 commit 시점만 다룬다.

```cpp
ClientState& ClientRegistry::create(int fd) {
    // 직접 구현
}

NickUpdate ClientRegistry::setNickname(
    int fd,
    const std::string& nickname) {
    // 직접 구현
}

std::optional<int> ClientRegistry::findByNickname(
    const std::string& nickname) const {
    // 직접 구현
}

void ClientRegistry::erase(int fd) {
    // 직접 구현
}

RegistrationDecision ClientRegistry::registrationDecision(int fd) const {
    // 직접 구현
}

bool ClientRegistry::commitRegistration(int fd) {
    // 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 임의의 nickname 변경 뒤 primary map과 index가 서로 역참조되는가?
- [ ] 충돌·유효성 실패 전후 전체 상태가 동일한가?
- [ ] 대소문자 충돌 테스트가 있는가?
- [ ] 같은 fd의 동일 nickname 재설정 정책이 일관적인가?
- [ ] 삭제 뒤 nickname lookup이 즉시 실패하는가?
- [ ] PASS/NICK/USER 도착 순서와 무관하게 준비 판정이 같은가?
- [ ] 등록 commit이 정확히 한 번만 상태를 바꾸는가?
- [ ] lookup 평균 복잡도가 O(1)인지, canonicalization 비용은 무엇인지 설명할 수 있는가?
- [ ] 송신 실패를 포함한 실제 application commit 시점이 이 자료구조 밖에 있음을 설명했는가?

### 구현 후 설명할 것

- primary state와 secondary index의 source of truth 관계
- nickname 갱신에서 검증과 실제 변경의 순서
- 등록 준비 판정과 등록 commit을 분리한 이유
- canonical form만 저장하지 않고 표시 nickname을 별도로 유지하는 이유
- index 불일치가 생겼을 때 탐지할 수 있는 invariant 검사 방법

### 원본 확인 위치

- Thread 05
- 커밋: `feat(client): 연결별 등록 상태 저장`, `feat(client): 닉네임 색인 관리`, `feat(registration): PASS 인증 상태 처리`, `feat(registration): 닉네임 검증과 색인 갱신`, `feat(registration): USER 정보와 환영 응답 연결`
- 파일: `src/ClientRegistry.hpp`, `src/ClientRegistry.cpp`, `src/RegistrationCommands.cpp`, `src/IrcApplication.hpp`
- 함수·컴포넌트: `ClientState`, `ClientRegistry::state`, `find`, `contains`, `fds`, `findFdByNickname`, `setNickname`, `erase`, `IrcApplication::maybeRegister`
- 관련 Thread: 06, 09, 10, 12

---

## [Thread 06 / `feat(channel): 구성원과 운영자 상태 관리` · `feat(channel): 구성원 정리와 식별자 변경 방송`] P08. Channel invariant와 연결 종료 정리

### 면접 질문

이 프로젝트의 `Channel`은 member, operator, invited nickname, topic·mode 상태를 가집니다. 한 client가 disconnect하거나 nickname을 바꿀 때 여러 channel과 여러 peer에 영향을 주는데, 어떤 invariant와 cleanup 순서가 필요합니까?

꼬리 질문:

- operator 집합에 member가 아닌 fd가 들어가면 어떤 후속 오류가 생깁니까?
- 마지막 member가 나간 channel은 언제 map에서 지워야 합니까?
- 두 사용자가 여러 channel을 공유할 때 QUIT이나 NICK 알림을 channel마다 보내면 어떤 문제가 생깁니까?
- cleanup 중 channel map을 순회하면서 channel을 erase해도 됩니까?
- nickname을 먼저 index에서 바꾼 뒤 old prefix가 필요하면 어떻게 합니까?
- disconnect cleanup을 완전한 transaction으로 만들기 어려운 이유는 무엇입니까?

### 30초 모범 답변

핵심 invariant는 operator가 member의 부분집합이고, client registry에 없는 fd가 channel membership에 남지 않으며, 빈 channel은 전역 map에 남지 않는 것입니다. disconnect 시에는 old client 정보를 먼저 복사하고, 공유 peer를 set으로 모아 알림을 한 번만 보내며, 각 channel에서 membership과 operator를 함께 제거한 뒤 빈 channel을 지웁니다. 순회 중 erase가 필요하면 다음 iterator를 미리 확보하거나 제거 대상 이름을 별도로 모읍니다. nickname 변경 방송도 old prefix를 먼저 만든 뒤 index를 갱신하고, 공통 peer를 dedup해 한 번씩 전달해야 합니다.

### 답변 핵심 키워드

부분집합 invariant, dangling membership, empty channel cleanup, old state snapshot, iterator invalidation, peer dedup, QUIT/NICK once, cleanup 순서, idempotence

### 백지 구현

**구현 목표**

작은 channel 자료구조와 네트워크 상태에서 client 하나를 제거한다. operator·membership·빈 channel 정리와 알림 대상 중복 제거를 수행한다.

**면접용 축소 인터페이스**

```cpp
#include <set>
#include <string>
#include <unordered_map>
#include <vector>

class ChannelState {
public:
    bool addMember(int fd, bool makeOperator);
    bool removeMember(int fd);
    bool setOperator(int fd, bool enabled);

    bool hasMember(int fd) const;
    bool isOperator(int fd) const;
    bool empty() const;
    const std::set<int>& members() const;

private:
    std::set<int> members_;
    std::set<int> operators_;
};

struct CleanupResult {
    std::vector<int> notifyPeers;
    std::vector<std::string> erasedChannels;
};

CleanupResult removeClientFromChannels(
    int fd,
    std::unordered_map<std::string, ChannelState>& channels);
```

**입력과 출력**

- 입력: 제거할 fd와 channel map
- 출력: 한 번씩 알림을 받아야 할 peer 목록과 삭제된 channel 이름
- 상태 변화: 모든 channel에서 fd 제거, operator 정리, 빈 channel 삭제

**반드시 만족해야 할 조건**

- `operators ⊆ members`가 모든 public 연산 뒤 유지된다.
- member가 아닌 fd를 operator로 만들 수 없다.
- member 제거는 operator 상태도 제거한다.
- 제거 대상과 channel을 함께 공유하던 peer는 결과에 한 번만 나온다.
- 제거 뒤 빈 channel은 map에서 사라진다.
- 무관한 client·channel 상태는 바뀌지 않는다.
- 존재하지 않는 fd 제거는 안전하다.

**경계 조건**

- channel이 하나도 없음
- client가 어느 channel에도 없음
- client가 혼자 있는 channel 하나
- client가 operator인 channel과 일반 member인 channel 혼합
- 같은 두 peer가 여러 channel을 공유
- channel 삭제가 연속해서 발생
- 같은 client cleanup 두 번 호출

**실패 조건**

- iterator invalidation으로 channel 누락 또는 접근 오류
- operator 잔존
- 같은 peer에게 중복 알림
- 빈 channel 잔존
- 무관한 channel 삭제

**제약**

- 25~30분 안에 구현한다.
- 알림 전송은 하지 않고 수신자 집합만 반환한다.
- global scan 비용을 설명한다.
- 자료구조를 바꾸는 대안을 제시할 수 있지만 인터페이스는 유지한다.

```cpp
bool ChannelState::addMember(int fd, bool makeOperator) {
    // 직접 구현
}

bool ChannelState::removeMember(int fd) {
    // 직접 구현
}

bool ChannelState::setOperator(int fd, bool enabled) {
    // 직접 구현
}

CleanupResult removeClientFromChannels(
    int fd,
    std::unordered_map<std::string, ChannelState>& channels) {
    // 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 모든 연산 뒤 operator가 member의 부분집합인가?
- [ ] member 제거 뒤 operator 정보가 남지 않는가?
- [ ] 같은 peer가 세 channel을 공유해도 한 번만 결과에 포함되는가?
- [ ] 마지막 member가 나간 channel만 삭제되는가?
- [ ] map erase 중 순회가 안전한가?
- [ ] cleanup 두 번 호출 결과가 두 번째에는 빈 변화인가?
- [ ] 무관한 channel의 member·operator 집합이 동일한가?
- [ ] 최악의 시간 복잡도가 전체 channel membership 수에 비례함을 설명할 수 있는가?
- [ ] 역색인(client→channels)이 있다면 복잡도와 동기화 부담이 어떻게 바뀌는가?

### 구현 후 설명할 것

- 유지한 invariant와 public 함수에서 이를 강제한 방법
- peer dedup에 set을 사용한 이유와 대안
- channel map 순회·삭제 방식
- client→channel 역색인을 추가할 때 얻는 성능과 추가 invariant
- 알림 생성과 상태 제거의 순서를 실제 서버에서 어떻게 정할지

### 원본 확인 위치

- Thread 06
- 커밋: `feat(channel): 채널 상태 계약 정의`, `feat(channel): 구성원과 운영자 상태 관리`, `feat(channel): 주제·초대·모드와 이름 규칙 구현`, `feat(channel): 구성원 정리와 식별자 변경 방송`
- 파일: `include/Channel.hpp`, `src/Channel.cpp`, `src/ApplicationSupport.cpp`, `src/RegistrationCommands.cpp`, `src/IrcApplication.hpp`
- 함수·컴포넌트: `Channel::addMember`, `removeMember`, `setOperator`, `hasMember`, `isOperator`, `invite`, `clearInvite`, `IrcApplication::partAllChannels`, `partChannel`, `eraseChannelIfEmpty`, `removeClientState`, `broadcastToCommon`
- 관련 Thread: 05, 07, 08, 10

---

## [Thread 07 / `feat(channel): KICK 구성원 제거 처리` · `feat(channel): 채널 운영자 모드 변경` · Thread 10 / `fix(app): 응답 실패 뒤 명령 처리를 중단`] P09. 권한 검사와 다단계 명령의 실패 경계

### 면접 질문

`JOIN`, `TOPIC`, `KICK`, `INVITE`, `MODE`는 단순 map 수정이 아니라 존재·등록·membership·operator·대상 상태를 순서대로 검증하고 응답을 보낸 뒤 상태를 바꿉니다. 이 프로젝트에서 MODE `+it` 처리 중 첫 방송 실패로 송신자가 제거됐다면 왜 나머지 `t` mode까지 적용하면 안 됩니까?

꼬리 질문:

- 명령 전체를 원자적 transaction으로 rollback하는 대신 앞에서 성공한 mode만 남기는 정책은 언제 합리적입니까?
- 오류 numeric을 보내는 것 자체가 실패하면 어떤 후속 검사를 해야 합니까?
- `Channel*`를 얻은 뒤 broadcast를 호출하고 다시 사용하는 것이 왜 위험합니까?
- 검증과 mutation을 순수 함수로 분리하면 어떤 장점과 비용이 있습니까?
- KICK에서 broadcast와 member 제거의 순서를 바꾸면 관찰 결과가 어떻게 달라집니까?

### 30초 모범 답변

이 구현에서는 송신 queue 실패가 연결 제거와 application cleanup을 유발할 수 있으므로 모든 send·broadcast는 수명 경계입니다. mode 문자열은 순차 적용되기 때문에 한 mode가 적용되고 그 방송 중 actor가 제거되면, 그 시점까지의 변화는 남기되 뒤 mode는 중단하는 정책이 자연스럽습니다. 외부 호출 전 channel 이름 같은 안정 키를 복사하고, 호출 뒤 actor와 channel을 registry에서 다시 찾아야 합니다. 완전 rollback을 하려면 이미 전달된 wire 응답까지 되돌릴 수 없어 오히려 상태와 관찰 결과가 어긋날 수 있습니다.

### 답변 핵심 키워드

authorization ladder, sequential semantics, send as lifetime boundary, stable key, relookup, partial commit, observable side effect, stop-on-failure, stale pointer, actor existence

### 백지 구현

**구현 목표**

operator만 변경할 수 있는 축소 channel mode `i`, `t`, `o` 처리기를 작성한다. mode는 왼쪽부터 순차 적용하며, 출력 callback이 실패하거나 actor/channel이 사라지면 즉시 중단한다.

**면접용 축소 인터페이스**

```cpp
#include <functional>
#include <string>
#include <string_view>
#include <unordered_map>
#include <vector>

struct ModeChannel {
    bool inviteOnly = false;
    bool topicProtected = true;
    std::unordered_map<int, bool> members; // value: operator 여부
};

struct ModeNetwork {
    std::unordered_map<int, std::string> nickByFd;
    std::unordered_map<std::string, ModeChannel> channels;
};

struct ModeResult {
    std::size_t appliedCount;
    bool completed;
    std::string error;
};

using EmitMode = std::function<bool(
    int actorFd,
    const std::string& channel,
    const std::string& renderedMode,
    const std::string& argument)>;

ModeResult applyChannelModes(
    ModeNetwork& network,
    int actorFd,
    const std::string& channelName,
    std::string_view modes,
    const std::vector<std::string>& arguments,
    const EmitMode& emit);
```

**입력과 출력**

- 입력: network 상태, actor, channel, mode 문자열, mode 인자, 출력 callback
- 출력: 적용된 mode 수, 전체 완료 여부, 첫 오류
- 상태 변화: 성공적으로 처리된 mode만 순차 반영

**반드시 만족해야 할 조건**

- actor·channel 존재, membership, operator 권한을 먼저 확인한다.
- `+`와 `-`는 이후 mode의 방향을 바꾼다.
- `o`는 인자 하나를 소비하며 대상은 해당 channel member여야 한다.
- unknown mode와 missing argument 정책을 명확히 한다.
- 각 mode의 출력 경계를 넘은 뒤 actor와 channel 존재를 다시 확인한다.
- 출력 실패 후 뒤 mode를 적용하지 않는다.
- 이미 성공한 앞 mode를 임의로 되돌리지 않는다.
- 무관한 channel과 member를 변경하지 않는다.

**경계 조건**

- 빈 mode 문자열
- `+i`, `-t`, `+it`, `+o nick`, `+io nick`
- 방향 문자가 여러 번 나옴
- `o` 인자 부족
- 존재하지 않는 target nick
- target은 존재하지만 channel member가 아님
- actor가 member지만 operator가 아님
- 첫 emit 실패, 두 번째 emit 실패
- emit callback이 actor 또는 channel을 삭제

**실패 조건**

- 권한 부족
- channel·actor·target 누락
- mode 인자 부족
- 출력 실패
- 출력 뒤 actor/channel 소멸

**제약**

- 25~30분 안에 구현한다.
- 전역 raw pointer나 iterator를 emit 호출 너머로 보관하지 않는다.
- 이미 외부에 관찰된 성공 mode를 rollback하지 않는다.
- 실제 IRC numeric과 문자열 formatting은 구현하지 않는다.

```cpp
ModeResult applyChannelModes(
    ModeNetwork& network,
    int actorFd,
    const std::string& channelName,
    std::string_view modes,
    const std::vector<std::string>& arguments,
    const EmitMode& emit) {
    // 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 권한 검사가 mutation보다 먼저 수행되는가?
- [ ] `+it`에서 첫 emit 실패 시 두 번째 mode가 적용되지 않는가?
- [ ] 첫 mode 성공 뒤 두 번째 실패 시 첫 변화는 유지되는가?
- [ ] emit이 actor를 삭제해도 dangling reference를 읽지 않는가?
- [ ] emit이 channel을 삭제한 뒤 재탐색하는가?
- [ ] `o`가 mode 문자열 순서대로 인자를 정확히 소비하는가?
- [ ] 대상이 member가 아닐 때 operator 상태가 생기지 않는가?
- [ ] unknown mode 처리 뒤 계속할지 중단할지 선택한 정책이 테스트와 일치하는가?
- [ ] 무관한 channel 상태가 동일한가?

### 구현 후 설명할 것

- 검증 순서와 첫 실패 반환 규칙
- 순차 partial commit을 택한 이유
- emit 전후로 어떤 안정 키를 보존하고 무엇을 재탐색하는지
- wire side effect가 있는 작업을 일반 DB transaction처럼 rollback하기 어려운 이유
- 상태 전이와 응답 생성 분리를 더 강하게 할 수 있는 설계 대안

### 원본 확인 위치

- Thread 07, Thread 10
- 커밋: `feat(channel): JOIN 채널 입장 처리`, `feat(channel): PART 채널 퇴장 처리`, `feat(channel): KICK 구성원 제거 처리`, `feat(channel): 채널 운영자 모드 변경`, `fix(app): 응답 실패 뒤 명령 처리를 중단`
- 파일: `src/ChannelCommands.cpp`, `src/ApplicationSupport.cpp`, `tests/application_lifetime_test.cpp`
- 함수: `IrcApplication::handleJoin`, `handlePart`, `handleTopic`, `handleKick`, `handleInvite`, `handleMode`, `handleChannelMode`, `findChannelForCommand`, `broadcastMode`, `sendNames`
- 테스트: `modeStopsAfterSenderCleanupTest`
- 관련 Thread: 06, 10

---

## [Thread 08 / `feat(message): 등록 사용자의 개인 메시지 전달` · `feat(message): 채널 대상 메시지 방송`] P10. 대상 해석과 fan-out 수신자 중복 제거

### 면접 질문

`PRIVMSG` 대상은 nickname일 수도 있고 channel일 수도 있으며 comma로 여러 대상을 받을 수 있습니다. 또한 NICK·QUIT 같은 알림은 여러 공통 channel의 peer에게 전달되지만 같은 peer에게 중복 전송되면 안 됩니다. 대상 해석과 recipient 수집을 어떻게 분리하겠습니까?

꼬리 질문:

- channel 메시지에서 sender를 제외하는 정책은 어디에서 적용합니까?
- 동일 대상이 comma 목록에 반복되면 한 번만 보낼지 입력 횟수대로 보낼지 어떻게 정합성을 정합니까?
- nickname lookup 실패와 channel membership 부족은 같은 오류입니까?
- fan-out 중 한 recipient 송신 실패로 해당 recipient가 제거되면 수신자 순회를 어떻게 안전하게 유지합니까?
- recipient를 미리 snapshot하는 방식과 live 상태를 매번 조회하는 방식의 trade-off는 무엇입니까?

### 30초 모범 답변

먼저 문자열 target을 nickname과 channel로 분류하고, 권한·존재 검사를 거쳐 recipient fd 집합과 대상별 오류를 만드는 단계로 분리합니다. channel fan-out에서는 sender 제외 여부를 명시하고, 공통 channel 기반 알림은 `set<int>`로 수신자를 dedup합니다. 실제 송신은 recipient snapshot을 순회하되 각 fd가 여전히 존재하는지 확인하면 중간 disconnect로 인한 iterator 무효화를 피할 수 있습니다. 대상별 오류는 독립적으로 처리할 수 있지만, 동일 target 반복을 dedup할지는 프로토콜 계약으로 먼저 정해야 합니다.

### 답변 핵심 키워드

target classification, direct vs channel, recipient set, sender exclusion, dedup, snapshot, per-target error, live existence check, fan-out complexity

### 백지 구현

**구현 목표**

여러 target을 direct nickname 또는 channel로 해석해 target별 recipient와 오류를 만든다. 실제 송신은 하지 않는다.

**면접용 축소 인터페이스**

```cpp
#include <set>
#include <string>
#include <unordered_map>
#include <vector>

struct FanoutChannel {
    std::set<int> members;
};

struct FanoutState {
    std::unordered_map<std::string, int> nickIndex;
    std::unordered_map<std::string, FanoutChannel> channels;
};

struct ResolvedTarget {
    std::string originalTarget;
    std::vector<int> recipients;
    std::string error;
};

std::vector<ResolvedTarget> resolveMessageTargets(
    const FanoutState& state,
    int senderFd,
    const std::vector<std::string>& targets);

std::vector<int> collectCommonPeers(
    const FanoutState& state,
    int subjectFd);
```

**입력과 출력**

- 입력: nickname index, channel membership, sender, target 목록
- 출력: target별 recipient 또는 오류, 공통 channel peer의 dedup된 목록

**반드시 만족해야 할 조건**

- direct nickname은 정확한 fd 하나로 해석된다.
- 존재하지 않는 nickname과 channel은 명확한 오류를 가진다.
- channel target에서 sender가 member가 아니면 정책에 맞는 오류를 낸다.
- channel recipient에서 sender를 제외한다.
- 한 channel의 member가 결과에 중복되지 않는다.
- `collectCommonPeers`는 여러 channel을 공유해도 각 peer를 한 번만 반환하며 subject 자신을 제외한다.
- 입력 state를 변경하지 않는다.

**경계 조건**

- 빈 target 목록
- 자기 자신에게 direct message
- member가 sender 하나뿐인 channel
- sender가 속하지 않은 channel
- 같은 target 반복
- direct nickname과 channel 이름이 비슷한 문자열
- subject가 channel에 하나도 속하지 않음
- 같은 두 사용자가 여러 channel 공유

**실패 조건**

- target 없음
- nickname 없음
- channel 없음
- channel membership 부족
- 자료구조 내부에 존재하지 않는 fd가 들어 있는 경우의 방어 정책

**제약**

- 20~25분 안에 구현한다.
- target 종류 판정 규칙은 문제에서 `#` 선두 여부로 제한해도 된다.
- 실제 protocol numeric과 wire formatting은 구현하지 않는다.
- 반환 순서가 필요한 경우 어떤 순서를 보장하는지 명시한다.

```cpp
std::vector<ResolvedTarget> resolveMessageTargets(
    const FanoutState& state,
    int senderFd,
    const std::vector<std::string>& targets) {
    // 직접 구현
}

std::vector<int> collectCommonPeers(
    const FanoutState& state,
    int subjectFd) {
    // 직접 구현
}
```

### 구현 후 자가 검증

- [ ] direct와 channel target이 서로 다른 검증 경로를 가지는가?
- [ ] sender가 channel recipient에 포함되지 않는가?
- [ ] 공통 peer가 여러 channel 때문에 중복되지 않는가?
- [ ] 존재하지 않는 target이 다른 정상 target의 결과를 망치지 않는가?
- [ ] 같은 target 반복 정책이 테스트로 고정되어 있는가?
- [ ] 입력 state가 불변인가?
- [ ] 결과 순서가 정의되어 있다면 자료구조 순서와 일치하는가?
- [ ] 전체 복잡도가 조회 대상 수와 방문 membership 수에 어떻게 비례하는지 설명할 수 있는가?
- [ ] 실제 송신 단계에서는 fd snapshot과 존재성 재검사가 필요함을 구분했는가?

### 구현 후 설명할 것

- target 해석과 실제 송신을 분리한 이유
- per-target 오류와 전체 명령 실패 중 선택한 정책
- dedup 자료구조와 결과 순서 trade-off
- snapshot fan-out 중 recipient disconnect를 처리하는 방법
- client→channel 역색인이 있을 때 `collectCommonPeers` 복잡도가 어떻게 달라지는지

### 원본 확인 위치

- Thread 08
- 커밋: `feat(message): 등록 사용자의 개인 메시지 전달`, `feat(channel): 채널 탐색과 대상 해석 지원`, `feat(message): 채널 대상 메시지 방송`, `test(client): 여러 클라이언트의 채널 메시지 전달 검증`
- 파일: `src/MessagingCommands.cpp`, `src/ApplicationSupport.cpp`, `src/IrcApplication.cpp`, `src/IrcApplication.hpp`
- 함수: `IrcApplication::handlePrivmsg`, `splitComma`, `findNick`, `isChannelTarget`, `ensureChannel`, `findChannelForCommand`, `broadcastToChannel`, `broadcastToCommon`
- 관련 Thread: 05, 06, 07
