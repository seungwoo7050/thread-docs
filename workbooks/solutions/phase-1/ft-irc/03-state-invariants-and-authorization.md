# 상태 불변조건과 권한 경계

이 문서는 등록 상태, nickname 보조 색인, channel 내부 집합, 연결 종료 정리, 권한 검사와 메시지 fan-out을 하나의 상태 정합성 관점에서 다룬다.

## [Thread 05 / `feat(client): 닉네임 색인 관리` · `feat(registration): USER 정보와 환영 응답 연결`] P07. 등록 상태 기계와 nickname 보조 색인

### 면접 질문

IRC 등록은 `PASS`, `NICK`, `USER`가 모두 충족돼야 완료되지만 명령 도착 순서는 고정되어 있지 않습니다. 이 프로젝트의 `ClientState`와 `ClientRegistry`가 등록 완료를 한 번만 발생시키고, nickname의 대소문자 비구분 충돌과 변경을 일관되게 처리하려면 어떤 invariant가 필요합니까?

꼬리 질문:

- nickname 변경 시 기존 색인을 먼저 지우고 새 색인을 넣는 사이에 실패하면 어떻게 합니까?
  - 답변: 일반적으로 새 index 항목을 먼저 확보한 뒤 기존 항목을 지워야 하며, 새 삽입이나 상태 갱신 실패 시 새 항목을 rollback해야 합니다. 원본은 호출자가 문법·충돌을 먼저 검사하지만 `setNickname` 내부는 기존 key를 먼저 지우므로 allocation 실패까지의 strong guarantee는 구현하지 않았고, 축소 답안은 그 경계를 보강했습니다.
- 같은 사용자가 대소문자만 바꾼 nickname으로 변경하는 경우를 어떻게 다룹니까?
  - 답변: canonical key가 같고 index의 fd도 자신이면 충돌로 보지 않고 표시용 `nick`만 갱신합니다. 원본의 collision 검사도 `collision != fd`인 경우만 거절합니다.
- `fd -> ClientState`와 `canonical nickname -> fd` 중 어느 쪽을 source of truth로 봅니까?
  - 답변: `fd -> ClientState`가 소유 상태인 source of truth이고 nickname map은 검색을 빠르게 하는 secondary index입니다. 삭제·변경 때 두 구조가 역참조되는 invariant를 함께 유지합니다.
- 등록 완료 응답을 queue하는 데 실패해 연결이 제거됐다면 `registered=true`나 등록 로그를 남겨도 됩니까?
  - 답변: 원본은 중복 전이를 막기 위해 numeric 전송 전에 `registered=true`로 만들지만, 각 send 실패 시 즉시 반환하고 모든 환영 응답이 성공한 뒤에만 `client_registered` 로그를 남깁니다. 실패한 연결 상태는 disconnect cleanup으로 제거되므로 성공 로그는 남지 않습니다.
- PASS가 틀렸을 때 즉시 close하는 정책과 재시도를 허용하는 정책의 차이는 무엇입니까?
  - 답변: 원본은 464 응답 뒤 close를 요청해 brute-force 시도와 미등록 연결 점유를 줄입니다. 재시도 허용은 일시적 입력 실수에는 친화적이지만 시도 횟수·시간 제한 같은 추가 보호가 필요합니다.

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
    return clients_.try_emplace(fd).first->second;
}

ClientState* ClientRegistry::find(int fd) {
    const std::unordered_map<int, ClientState>::iterator found = clients_.find(fd);
    return found == clients_.end() ? nullptr : &found->second;
}

const ClientState* ClientRegistry::find(int fd) const {
    const std::unordered_map<int, ClientState>::const_iterator found = clients_.find(fd);
    return found == clients_.end() ? nullptr : &found->second;
}

std::string ClientRegistry::canonicalNickname(const std::string& nickname) {
    std::string canonical = nickname;
    for (char& ch : canonical) {
        if (ch >= 'A' && ch <= 'Z') {
            ch = static_cast<char>(ch - 'A' + 'a');
        }
    }
    return canonical;
}

bool ClientRegistry::validNickname(const std::string& nickname) {
    if (nickname.empty() || nickname.size() > 30) {
        return false;
    }
    const char first = nickname[0];
    if ((first >= '0' && first <= '9') ||
        first == '#' || first == '&' || first == ':' || first == '-') {
        return false;
    }
    for (const char ch : nickname) {
        if (ch == ' ' || ch == '\t' || ch == '\r' || ch == '\n' ||
            ch == ',' || ch == '*' || ch == '?' || ch == '!' || ch == '@') {
            return false;
        }
    }
    return true;
}

NickUpdate ClientRegistry::setNickname(
    int fd,
    const std::string& nickname) {
    const std::unordered_map<int, ClientState>::iterator clientIt = clients_.find(fd);
    if (clientIt == clients_.end()) {
        return NickUpdate::MissingClient;
    }
    if (!validNickname(nickname)) {
        return NickUpdate::Invalid;
    }

    ClientState& client = clientIt->second;
    const std::string nextKey = canonicalNickname(nickname);
    const std::unordered_map<std::string, int>::const_iterator collision =
        nicknameIndex_.find(nextKey);
    if (collision != nicknameIndex_.end() && collision->second != fd) {
        return NickUpdate::InUse;
    }

    const std::string oldKey = canonicalNickname(client.nick);
    if (oldKey == nextKey) {
        client.nick = nickname;
        client.hasNick = true;
        nicknameIndex_[nextKey] = fd;
        return NickUpdate::Applied;
    }

    // 새 index를 먼저 확보해야 실패 시 기존 nickname이 그대로 남는다.
    const std::pair<std::unordered_map<std::string, int>::iterator, bool> inserted =
        nicknameIndex_.emplace(nextKey, fd);
    if (!inserted.second) {
        return NickUpdate::InUse;
    }
    try {
        client.nick = nickname;
    } catch (...) {
        nicknameIndex_.erase(inserted.first);
        throw;
    }
    if (client.hasNick) {
        nicknameIndex_.erase(oldKey);
    }
    client.hasNick = true;
    return NickUpdate::Applied;
}

std::optional<int> ClientRegistry::findByNickname(
    const std::string& nickname) const {
    const std::unordered_map<std::string, int>::const_iterator found =
        nicknameIndex_.find(canonicalNickname(nickname));
    if (found == nicknameIndex_.end()) {
        return std::nullopt;
    }
    return found->second;
}

void ClientRegistry::erase(int fd) {
    const std::unordered_map<int, ClientState>::iterator found = clients_.find(fd);
    if (found == clients_.end()) {
        return;
    }
    if (found->second.hasNick) {
        nicknameIndex_.erase(canonicalNickname(found->second.nick));
    }
    clients_.erase(found);
}

RegistrationDecision ClientRegistry::registrationDecision(int fd) const {
    const ClientState* client = find(fd);
    if (client == nullptr || !client->passOk || !client->hasNick || !client->hasUser) {
        return RegistrationDecision::NotReady;
    }
    if (client->registered) {
        return RegistrationDecision::AlreadyRegistered;
    }
    return RegistrationDecision::ReadyToCommit;
}

bool ClientRegistry::commitRegistration(int fd) {
    if (registrationDecision(fd) != RegistrationDecision::ReadyToCommit) {
        return false;
    }
    clients_.find(fd)->second.registered = true;
    return true;
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
  - 답변: `clients_`가 ClientState를 소유하고 `nicknameIndex_`는 canonical nickname 조회만 가속합니다. index의 모든 fd는 primary에 존재하고, `hasNick`인 client마다 역방향 항목이 정확히 하나 있어야 합니다.
- nickname 갱신에서 검증과 실제 변경의 순서
  - 답변: client 존재, 문법, canonical 충돌을 먼저 검사하고 새 key를 확보한 다음 표시 nickname을 갱신하고 이전 key를 제거합니다. 실패 가능 작업을 앞에 둬 충돌·할당 실패 때 기존 상태를 보존했습니다.
- 등록 준비 판정과 등록 commit을 분리한 이유
  - 답변: PASS/NICK/USER의 순서와 무관한 순수 준비 검사를 먼저 하고, 환영 응답 같은 외부 작업의 성공 정책에 맞는 시점에 one-shot flag를 바꿀 수 있습니다. 중복 응답과 중복 로그도 분리해 제어할 수 있습니다.
- canonical form만 저장하지 않고 표시 nickname을 별도로 유지하는 이유
  - 답변: 충돌과 lookup은 대소문자 비구분이어야 하지만 wire 응답과 hostmask에는 사용자가 선택한 표기가 필요합니다. canonical 값은 identity key이고 `nick`은 표시 값입니다.
- index 불일치가 생겼을 때 탐지할 수 있는 invariant 검사 방법
  - 답변: 테스트에서 모든 index 항목의 fd와 canonical `ClientState::nick`을 확인하고, 모든 `hasNick` client가 index에서 자신으로 역조회되는지 양방향으로 순회합니다. 항목 수와 중복 canonical key도 함께 검사합니다.

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
  - 답변: 권한 검사가 탈퇴한 client를 operator로 오인하거나 fd 재사용 뒤 다른 client에게 권한이 넘어갈 수 있습니다. 그래서 `setOperator`는 membership을 요구하고 `removeMember`가 두 집합에서 함께 지웁니다.
- 마지막 member가 나간 channel은 언제 map에서 지워야 합니까?
  - 답변: membership과 operator 정리를 끝낸 직후 비어 있음을 확인해 지웁니다. 원본은 `eraseChannelIfEmpty`를 PART·KICK 뒤 호출하고 disconnect cleanup에서는 이름 목록을 모아 순회가 끝난 뒤 erase합니다.
- 두 사용자가 여러 channel을 공유할 때 QUIT이나 NICK 알림을 channel마다 보내면 어떤 문제가 생깁니까?
  - 답변: 같은 wire 알림을 여러 번 받아 client 상태가 중복 변경된 것처럼 보일 수 있습니다. 원본 `broadcastToCommon`과 disconnect cleanup은 peer fd를 `set`에 모아 한 번씩만 전송합니다.
- cleanup 중 channel map을 순회하면서 channel을 erase해도 됩니까?
  - 답변: 현재 iterator를 무효화하지 않는 erase 패턴을 쓰거나, 원본처럼 빈 channel 이름을 별도 vector에 모아 첫 순회가 끝난 뒤 지워야 합니다. range 순회 중 무계획한 erase는 누락과 UB를 일으킵니다.
- nickname을 먼저 index에서 바꾼 뒤 old prefix가 필요하면 어떻게 합니까?
  - 답변: 원본 `handleNick`처럼 변경 전에 `oldPrefix`와 등록 여부를 값으로 복사합니다. index 갱신 뒤 방송은 이 snapshot을 사용하고, 방송 뒤에는 fd로 client 존재를 재확인합니다.
- disconnect cleanup을 완전한 transaction으로 만들기 어려운 이유는 무엇입니까?
  - 답변: peer로 보낸 QUIT은 되돌릴 수 없는 외부 관찰이고 각 send가 또 다른 disconnect를 일으킬 수 있습니다. 따라서 DB식 rollback보다 snapshot, 멱등 상태 정리, 실패해도 invariant를 회복하는 순서를 사용합니다.

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
    const bool inserted = members_.insert(fd).second;
    if (makeOperator) {
        operators_.insert(fd);
    }
    return inserted;
}

bool ChannelState::removeMember(int fd) {
    operators_.erase(fd);
    return members_.erase(fd) != 0;
}

bool ChannelState::setOperator(int fd, bool enabled) {
    if (!hasMember(fd)) {
        return false;
    }
    if (enabled) {
        return operators_.insert(fd).second;
    }
    return operators_.erase(fd) != 0;
}

bool ChannelState::hasMember(int fd) const {
    return members_.find(fd) != members_.end();
}

bool ChannelState::isOperator(int fd) const {
    return operators_.find(fd) != operators_.end();
}

bool ChannelState::empty() const {
    return members_.empty();
}

const std::set<int>& ChannelState::members() const {
    return members_;
}

CleanupResult removeClientFromChannels(
    int fd,
    std::unordered_map<std::string, ChannelState>& channels) {
    CleanupResult result;
    std::set<int> peers;

    std::unordered_map<std::string, ChannelState>::iterator it = channels.begin();
    while (it != channels.end()) {
        if (!it->second.hasMember(fd)) {
            ++it;
            continue;
        }

        for (const int member : it->second.members()) {
            if (member != fd) {
                peers.insert(member);
            }
        }
        it->second.removeMember(fd);
        if (it->second.empty()) {
            result.erasedChannels.push_back(it->first);
            it = channels.erase(it);
        } else {
            ++it;
        }
    }

    result.notifyPeers.assign(peers.begin(), peers.end());
    return result;
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
  - 답변: `operators_`는 항상 `members_`의 부분집합입니다. operator 승격은 member에게만 허용하고 member 삭제는 operator 항목도 함께 지우며, cleanup은 같은 public 연산을 사용합니다.
- peer dedup에 set을 사용한 이유와 대안
  - 답변: 여러 channel에서 같은 fd를 만나도 자동으로 한 번만 남고 결정적인 오름차순 결과를 얻습니다. `unordered_set`은 평균 삽입이 빠르지만 결과 순서를 별도로 정해야 합니다.
- channel map 순회·삭제 방식
  - 답변: 현재 iterator로 상태를 정리한 뒤 빈 channel이면 `erase(it)`가 돌려준 다음 iterator로 계속합니다. 원본의 이름 수집 후 2단계 erase도 같은 invalidation 문제를 피하는 합리적인 방식입니다.
- client→channel 역색인을 추가할 때 얻는 성능과 추가 invariant
  - 답변: 전체 channel을 훑지 않고 해당 client의 channel 수에 비례해 cleanup할 수 있습니다. 대신 JOIN/PART/KICK/disconnect마다 양방향 membership을 원자적으로 맞춰야 하는 invariant와 rollback 부담이 생깁니다.
- 알림 생성과 상태 제거의 순서를 실제 서버에서 어떻게 정할지
  - 답변: old client/prefix와 peer snapshot을 먼저 만들고 QUIT을 전송한 뒤 membership·빈 channel·client index를 멱등 정리합니다. 원본도 이 순서를 사용하며, send 중 다른 상태가 변할 수 있으므로 snapshot fd 전송은 live lookup에 맡깁니다.

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
  - 답변: mode가 왼쪽부터 독립적으로 적용되고 각 성공 방송이 이미 외부에 관찰되는 IRC 명령에서는 partial commit이 합리적입니다. 원본도 성공한 앞 mode는 유지하고 첫 송신·수명 실패에서 뒤 처리를 중단합니다.
- 오류 numeric을 보내는 것 자체가 실패하면 어떤 후속 검사를 해야 합니까?
  - 답변: `sendNumeric` 실패가 interest 갱신 실패와 disconnect로 이어질 수 있으므로 actor fd가 아직 registry에 있는지 확인하고, 없으면 명령 처리를 즉시 끝내야 합니다. 원본의 다중 mode 오류 경로도 반환값이 false면 return합니다.
- `Channel*`를 얻은 뒤 broadcast를 호출하고 다시 사용하는 것이 왜 위험합니까?
  - 답변: broadcast의 각 `sendRaw`가 연결 cleanup과 빈 channel 삭제를 유발해 map 원소를 파괴할 수 있기 때문입니다. channel 이름을 복사하고 방송 뒤 map에서 다시 찾아야 합니다.
- 검증과 mutation을 순수 함수로 분리하면 어떤 장점과 비용이 있습니까?
  - 답변: 권한·인자 검사를 표 형태로 단위 테스트하고 적용 전 계획을 검토하기 쉬워집니다. 반면 실제 emit 사이에 상태가 바뀔 수 있으므로 계획 실행 중 재검증이 여전히 필요하고 중간 결과 표현이 복잡해집니다.
- KICK에서 broadcast와 member 제거의 순서를 바꾸면 관찰 결과가 어떻게 달라집니까?
  - 답변: 원본처럼 먼저 방송하면 kick 대상도 KICK frame을 받고 그 뒤 membership이 제거됩니다. 먼저 제거하면 대상이 channel fan-out에서 빠져 자신의 퇴장 이유를 받지 못할 수 있습니다.

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
    ModeResult result{0, false, ""};

    const auto validateActor = [&]() -> std::string {
        if (network.nickByFd.find(actorFd) == network.nickByFd.end()) {
            return "actor is missing";
        }
        const std::unordered_map<std::string, ModeChannel>::const_iterator channel =
            network.channels.find(channelName);
        if (channel == network.channels.end()) {
            return "channel is missing";
        }
        const std::unordered_map<int, bool>::const_iterator member =
            channel->second.members.find(actorFd);
        if (member == channel->second.members.end()) {
            return "actor is not a channel member";
        }
        if (!member->second) {
            return "actor is not a channel operator";
        }
        return "";
    };

    result.error = validateActor();
    if (!result.error.empty()) {
        return result;
    }

    bool adding = true;
    std::size_t argumentIndex = 0;
    for (const char mode : modes) {
        if (mode == '+') {
            adding = true;
            continue;
        }
        if (mode == '-') {
            adding = false;
            continue;
        }

        result.error = validateActor();
        if (!result.error.empty()) {
            return result;
        }
        std::unordered_map<std::string, ModeChannel>::iterator channel =
            network.channels.find(channelName);
        std::string argument;

        if (mode == 'i') {
            channel->second.inviteOnly = adding;
        } else if (mode == 't') {
            channel->second.topicProtected = adding;
        } else if (mode == 'o') {
            if (argumentIndex == arguments.size()) {
                result.error = "operator mode argument is missing";
                return result;
            }
            argument = arguments[argumentIndex++];

            int targetFd = -1;
            for (const auto& entry : network.nickByFd) {
                if (entry.second == argument) {
                    targetFd = entry.first;
                    break;
                }
            }
            const std::unordered_map<int, bool>::iterator target =
                channel->second.members.find(targetFd);
            if (targetFd == -1 || target == channel->second.members.end()) {
                result.error = "operator target is not a channel member";
                return result;
            }
            target->second = adding;
        } else {
            result.error = "unknown channel mode";
            return result;
        }

        ++result.appliedCount; // emit 실패도 이미 적용된 앞 상태를 rollback하지 않는다.
        const std::string renderedMode = std::string(adding ? "+" : "-") + mode;
        if (!emit(actorFd, channelName, renderedMode, argument)) {
            result.error = "mode output failed";
            return result;
        }
        result.error = validateActor();
        if (!result.error.empty()) {
            return result;
        }
    }

    result.completed = true;
    result.error.clear();
    return result;
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
  - 답변: actor·channel 존재, membership, operator 권한을 mutation 전에 확인하고 `o`의 인자·대상 membership을 추가 검사했습니다. 첫 검증·emit·재검증 실패에서 즉시 반환해 뒤 mode는 건드리지 않습니다.
- 순차 partial commit을 택한 이유
  - 답변: 원본은 mode 문자를 왼쪽부터 변경하고 각각 즉시 방송합니다. 이미 wire에 관찰된 앞 성공을 rollback하면 오히려 peer가 본 상태와 서버 상태가 달라지므로 앞 변화는 유지합니다.
- emit 전후로 어떤 안정 키를 보존하고 무엇을 재탐색하는지
  - 답변: 값 타입인 `actorFd`와 `channelName`, 필요한 mode argument만 보존합니다. emit 뒤 기존 channel iterator는 사용하지 않고 두 map에서 actor와 channel, membership·operator를 다시 확인합니다.
- wire side effect가 있는 작업을 일반 DB transaction처럼 rollback하기 어려운 이유
  - 답변: 이미 전송된 MODE frame은 회수할 수 없고 그 전송 중 다른 client disconnect 같은 추가 효과도 발생할 수 있습니다. 보상 메시지는 또 실패할 수 있어 원자 rollback보다 명시적 partial commit이 실제 관찰과 잘 맞습니다.
- 상태 전이와 응답 생성 분리를 더 강하게 할 수 있는 설계 대안
  - 답변: 순수 validator가 mode별 mutation plan과 wire event를 만들고 executor가 한 단계씩 적용·emit·재검증하도록 나눌 수 있습니다. 테스트는 쉬워지지만 live 상태 버전 검사와 중간 실패 표현이 추가됩니다.

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
  - 답변: target 해석 단계에서 channel membership snapshot을 recipient로 바꿀 때 sender fd를 제외합니다. 원본도 `broadcastToChannel(..., exceptFd=sender)`에서 동일 정책을 적용합니다.
- 동일 대상이 comma 목록에 반복되면 한 번만 보낼지 입력 횟수대로 보낼지 어떻게 정합성을 정합니까?
  - 답변: 원본은 target 목록을 순서대로 순회하므로 반복 target도 입력 횟수대로 처리합니다. 중복 제거를 원한다면 target canonicalization 단계에서 명시하고 계약 테스트로 고정해야 하며, 여기서는 원본 동작을 유지합니다.
- nickname lookup 실패와 channel membership 부족은 같은 오류입니까?
  - 답변: 아닙니다. 원본은 없는 nickname에 401을, channel이 없거나 sender가 member가 아니면 404를 반환해 대상 부재와 송신 권한 실패를 구분합니다.
- fan-out 중 한 recipient 송신 실패로 해당 recipient가 제거되면 수신자 순회를 어떻게 안전하게 유지합니까?
  - 답변: channel의 live iterator가 아니라 fd vector snapshot을 순회하고 각 `sendRaw(fd)`가 registry lookup을 하게 합니다. 원본 `broadcastToChannel`도 먼저 `members()` 복사본을 만듭니다.
- recipient를 미리 snapshot하는 방식과 live 상태를 매번 조회하는 방식의 trade-off는 무엇입니까?
  - 답변: snapshot은 iterator 무효화 없이 결정적인 대상 집합을 순회하지만 중간에 탈퇴한 fd가 포함될 수 있습니다. live 조회는 최신 상태를 반영하지만 순회 중 변경 처리와 결과 일관성이 복잡하므로, snapshot에 송신 직전 존재성 검사를 결합하는 편이 안전합니다.

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
    std::set<int> liveFds;
    for (const auto& nick : state.nickIndex) {
        liveFds.insert(nick.second);
    }

    std::vector<ResolvedTarget> resolved;
    resolved.reserve(targets.size());
    for (const std::string& target : targets) {
        ResolvedTarget item;
        item.originalTarget = target;

        if (!target.empty() && target[0] == '#') {
            const std::unordered_map<std::string, FanoutChannel>::const_iterator channel =
                state.channels.find(target);
            if (channel == state.channels.end()) {
                item.error = "no such channel";
            } else if (channel->second.members.find(senderFd) ==
                       channel->second.members.end()) {
                item.error = "cannot send to channel";
            } else {
                for (const int member : channel->second.members) {
                    if (member != senderFd && liveFds.find(member) != liveFds.end()) {
                        item.recipients.push_back(member);
                    }
                }
            }
        } else {
            std::string canonicalTarget = target;
            for (char& ch : canonicalTarget) {
                if (ch >= 'A' && ch <= 'Z') {
                    ch = static_cast<char>(ch - 'A' + 'a');
                }
            }
            const std::unordered_map<std::string, int>::const_iterator nick =
                state.nickIndex.find(canonicalTarget);
            if (nick == state.nickIndex.end()) {
                item.error = "no such nick";
            } else {
                item.recipients.push_back(nick->second);
            }
        }
        resolved.push_back(item);
    }
    return resolved;
}

std::vector<int> collectCommonPeers(
    const FanoutState& state,
    int subjectFd) {
    std::set<int> liveFds;
    for (const auto& nick : state.nickIndex) {
        liveFds.insert(nick.second);
    }

    std::set<int> peers;
    for (const auto& entry : state.channels) {
        if (entry.second.members.find(subjectFd) == entry.second.members.end()) {
            continue;
        }
        for (const int member : entry.second.members) {
            if (member != subjectFd && liveFds.find(member) != liveFds.end()) {
                peers.insert(member);
            }
        }
    }
    return std::vector<int>(peers.begin(), peers.end());
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
  - 답변: 존재·membership 오류와 recipient 계산을 입력 state만으로 단위 테스트할 수 있고, 송신 단계는 fd snapshot·재탐색·queue 실패라는 별도 수명 문제에 집중할 수 있습니다.
- per-target 오류와 전체 명령 실패 중 선택한 정책
  - 답변: 원본처럼 각 target 결과를 독립적으로 만들었습니다. 하나가 없거나 권한이 없어도 오류를 남기고 뒤 정상 target을 계속 해석하므로 여러 대상 명령의 부분 성공을 보존합니다.
- dedup 자료구조와 결과 순서 trade-off
  - 답변: channel membership 자체와 공통 peer에는 `set<int>`를 사용해 중복 제거와 fd 오름차순을 함께 얻었습니다. `unordered_set`은 평균 O(1) 삽입이지만 출력 순서를 보장하려면 별도 정렬이 필요합니다.
- snapshot fan-out 중 recipient disconnect를 처리하는 방법
  - 답변: snapshot의 각 fd를 보내기 직전에 connection registry에서 다시 찾고, 없으면 건너뜁니다. 한 recipient 실패가 snapshot container를 무효화하지 않으며 다른 recipient 전송은 계속할 수 있습니다.
- client→channel 역색인이 있을 때 `collectCommonPeers` 복잡도가 어떻게 달라지는지
  - 답변: 현재는 모든 channel과 membership을 훑지만 역색인이 있으면 subject가 속한 channel과 그 member만 방문합니다. 대신 membership 양쪽을 JOIN/PART/KICK/cleanup마다 동기화해야 합니다.

### 원본 확인 위치

- Thread 08
- 커밋: `feat(message): 등록 사용자의 개인 메시지 전달`, `feat(channel): 채널 탐색과 대상 해석 지원`, `feat(message): 채널 대상 메시지 방송`, `test(client): 여러 클라이언트의 채널 메시지 전달 검증`
- 파일: `src/MessagingCommands.cpp`, `src/ApplicationSupport.cpp`, `src/IrcApplication.cpp`, `src/IrcApplication.hpp`
- 함수: `IrcApplication::handlePrivmsg`, `splitComma`, `findNick`, `isChannelTarget`, `ensureChannel`, `findChannelForCommand`, `broadcastToChannel`, `broadcastToCommon`
- 관련 Thread: 05, 06, 07
