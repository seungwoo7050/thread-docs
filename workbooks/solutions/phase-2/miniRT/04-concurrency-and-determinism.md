# 동시성·결정성·실패 전파 워크북

이 문서는 P13~P14를 다룬다. 타일 단위 병렬 렌더링의 공유 상태를 최소화하는 방법과, worker 내부 예외를 호출자에게 안전하게 되돌리는 lifecycle을 분리해 연습한다. 병렬화 자체보다 결과 결정성, 통계 정합성, thread 회수 invariant를 우선한다.

---

<a id="p13"></a>
## [Thread 09 / `feat(renderer): 작업자 수 설정과 자동 선택 추가`] 결정적 타일 스케줄링과 worker별 상태 축약

### 면접 질문

렌더링 타일을 여러 worker가 동적으로 가져가도록 만들어도 1 thread와 4 thread의 이미지 byte가 완전히 같게 유지할 수 있는 이유를 설명해 보세요. 어떤 데이터는 atomic이어야 하고, 어떤 데이터는 worker별로 분리해야 하며, 어떤 데이터는 lock 없이 공유할 수 있습니까?

꼬리 질문:

- `nextTile.fetch_add(1, memory_order_relaxed)`로 충분한 이유와 충분하지 않은 경우는 무엇입니까?
  - **모범답변:** 이 atomic은 tile 번호를 중복 없이 발급하는 데만 쓰고 다른 데이터의 publication 순서를 전달하지 않으므로 원자적 read-modify-write만 보장하는 relaxed면 충분합니다. tile index를 통해 다른 thread가 만든 데이터를 인계한다면 acquire/release나 별도 동기화가 필요합니다.
- 서로 다른 thread가 같은 `std::vector<unsigned char>`에 쓰더라도 data race가 없으려면 어떤 invariant가 필요합니까?
  - **모범답변:** 각 tile이 정확히 한 번 배정되고 tile의 pixel 사각형이 서로 겹치지 않아 각 byte의 writer가 하나뿐이어야 합니다. 렌더 중 vector 크기·capacity도 바꾸지 않고, 모든 읽기는 join 이후에 해야 합니다.
- 각 worker가 `RenderStats`를 따로 가진 뒤 join 후 합산하는 방식이 atomic counter보다 나은 점은 무엇입니까?
  - **모범답변:** ray마다 공유 atomic을 갱신하는 경쟁과 cache-line 이동을 피하고, 일반 정수 증가만 사용해 코드가 단순해집니다. join이 happens-before를 제공한 뒤 한 thread가 결정적으로 합산하므로 data race도 없습니다.
- worker별 통계 구조에 `alignas(64)`를 적용한 목적은 무엇이며 항상 효과가 보장됩니까?
  - **모범답변:** 인접 worker 통계가 같은 cache line을 공유해 생기는 false sharing을 줄이려는 목적입니다. 64가 실제 cache-line 크기라는 보장이나 allocator·컨테이너 배치에 따른 성능 향상 보장은 없으므로 측정이 필요합니다.
- 요청 thread 수가 0일 때 `hardware_concurrency()`가 0을 반환할 수 있다는 사실을 어떻게 처리합니까?
  - **모범답변:** 프로젝트는 0을 자동 선택 sentinel로 보고 `hardware_concurrency()`를 호출한 뒤, 그 결과도 0이면 worker 1개로 fallback합니다.
- worker 수를 tile 수보다 크게 만들지 않는 이유는 무엇입니까?
  - **모범답변:** tile보다 많은 worker는 일부 thread가 하나의 작업도 받지 못하면서 생성·join 비용과 메모리만 늘립니다. 원본은 실제 worker 수를 `min(requested, tileCount)`로 제한합니다.
- 동적 스케줄링과 정적 타일 분배의 load balancing·결정성 trade-off는 무엇입니까?
  - **모범답변:** atomic 동적 배정은 오래 걸리는 tile이 몰려도 빈 worker가 다음 tile을 가져가 load balance가 좋지만 실행 순서는 매번 달라질 수 있습니다. 정적 분배는 순서와 소유가 예측 가능하고 atomic 비용이 없지만 tile 비용 편차에 취약합니다.
- 이미지 결정성과 floating-point reduction 결정성은 왜 다른 문제입니까?
  - **모범답변:** 현재 각 pixel은 다른 pixel과 독립적으로 계산되어 scheduling이 산술 순서를 바꾸지 않습니다. 여러 thread의 실수 부분합을 합치는 연산은 덧셈 순서에 따라 반올림이 달라지므로 고정 reduction 순서 없이는 같은 보장을 할 수 없습니다.

### 30초 모범 답변

각 tile은 고유한 pixel 사각형이고 한 번만 배정되므로, worker가 어느 순서로 처리하든 각 pixel은 정확히 한 thread가 같은 순수 계산 결과를 정해진 offset에 씁니다. 공유되는 `nextTile`은 고유 작업 번호 발급만 담당하므로 relaxed atomic으로 충분하고, 통계는 worker별 로컬 구조에 기록해 hot counter 경쟁과 data race를 피한 뒤 모든 thread를 join한 후 정수 합산합니다. 자동 thread 수는 hardware 값이 0이면 1로 fallback하고 tile 수로 상한을 둡니다. 이 설계는 pixel 계산 사이에 순서 의존적 floating reduction이 없기 때문에 byte 결정성을 유지할 수 있으며, 실제로 thread 수와 가속 모드를 바꿔 pixels·checksum·작업량을 비교해야 합니다.

### 답변 핵심 키워드

`disjoint tiles`, `single writer per pixel`, `atomic work index`, `memory_order_relaxed`, `thread-local stats`, `post-join reduction`, `false sharing`, `hardware fallback`, `worker cap`, `dynamic scheduling`, `byte determinism`

### 백지 구현

**구현 목표**

고정 개수의 독립 tile을 여러 worker가 정확히 한 번씩 처리하고, worker별 통계를 모든 thread 종료 후 하나로 합산하는 일반화된 스케줄러를 구현한다. 작업 실패 전파는 P14에서 별도로 구현한다.

**인터페이스**

```cpp
struct RenderStats {
    std::uint64_t primaryRays = 0;
    std::uint64_t secondaryRays = 0;
    std::uint64_t shadowRays = 0;
    std::uint64_t primitiveTests = 0;
    std::uint64_t aabbTests = 0;
};

using TileTask = std::function<void(std::size_t tileIndex,
                                    unsigned int workerIndex,
                                    RenderStats& localStats)>;

RenderStats runTiles(std::size_t tileCount,
                     unsigned int requestedWorkers,
                     const TileTask& task);
```

**입력과 출력**

- 입력: tile 개수, 요청 worker 수. `0`은 자동 선택, tile별 작업 callback.
- 출력: 모든 worker의 로컬 통계를 합한 `RenderStats`.

**반드시 만족해야 할 조건**

- 각 tile index는 정확히 한 번 callback에 전달된다.
- tile이 없으면 thread를 만들지 않고 0 통계를 반환한다.
- 자동 선택 결과가 0이면 worker 1개로 fallback한다.
- 실제 worker 수는 tile 수를 넘지 않는다.
- 작업 배정에 필요한 최소 공유 상태만 atomic으로 둔다.
- 각 worker는 자기 `RenderStats`만 수정한다.
- 통계 합산은 모든 worker가 끝난 뒤 수행한다.
- callback이 서로 다른 tile의 서로 겹치지 않는 출력 영역만 수정한다는 전제와 책임 경계를 문서화한다.

**경계 조건**

- `tileCount == 0`.
- tile 1개, worker 요청 1개·4개·자동.
- tile보다 worker 요청이 많음.
- `hardware_concurrency()`가 0인 환경.
- tile 처리 시간이 매우 불균등함.
- 통계가 모두 0인 작업.
- 합산 중 `uint64_t` overflow 가능성이 있는 극단적 workload.

**실패 조건**

- 하나의 tile을 두 worker가 처리하거나 누락한다.
- 모든 worker가 하나의 stats 구조를 lock 없이 수정한다.
- pixel 또는 결과 영역이 tile 사이에서 겹친다.
- worker 종료 전에 통계를 읽거나 결과 완료를 가정한다.
- tile이 0인데 worker 수 계산 결과가 0인 채 vector indexing을 수행한다.
- 스케줄 순서에 따라 결과 계산이 달라지는 callback을 결정적이라고 가정한다.

**제약**

- thread pool 재사용은 구현하지 않는다.
- 작업 취소와 예외는 P14 범위다.
- `std::thread`와 C++17 표준 라이브러리만 사용한다.
- 스케줄러 오버헤드는 tile 수에 선형이어야 하며 worker별 부가 공간을 허용한다.

```cpp
RenderStats runTiles(std::size_t tileCount,
                     unsigned int requestedWorkers,
                     const TileTask& task) {
    if (tileCount == 0) return RenderStats{};

    unsigned int workerCount = requestedWorkers;
    if (workerCount == 0) {
        workerCount = std::thread::hardware_concurrency();
        if (workerCount == 0) workerCount = 1;
    }
    workerCount = static_cast<unsigned int>(
        std::min<std::size_t>(workerCount, tileCount));

    struct alignas(64) WorkerStats { RenderStats values; };
    std::vector<WorkerStats> locals(workerCount);
    std::atomic<std::size_t> nextTile{0};
    std::vector<std::thread> workers;
    workers.reserve(workerCount);

    for (unsigned int worker = 0; worker < workerCount; ++worker) {
        workers.emplace_back([&, worker] {
            for (;;) {
                // 이 atomic은 고유 번호 발급만 하므로 ordering 전달이 필요 없다.
                const std::size_t tile =
                    nextTile.fetch_add(1, std::memory_order_relaxed);
                if (tile >= tileCount) break;
                task(tile, worker, locals[worker].values);
            }
        });
    }
    for (std::thread& worker : workers) worker.join();

    RenderStats total;
    for (const WorkerStats& local : locals) {
        total.primaryRays += local.values.primaryRays;
        total.secondaryRays += local.values.secondaryRays;
        total.shadowRays += local.values.shadowRays;
        total.primitiveTests += local.values.primitiveTests;
        total.aabbTests += local.values.aabbTests;
    }
    return total;
}
```

### 구현 후 자가 검증

- [ ] tile 0개에서 callback 호출과 thread 생성이 없는가.
- [ ] 작은 tile 집합에서 각 index의 방문 횟수가 정확히 1인가.
- [ ] worker 요청이 tile 수보다 커도 실제 worker 수가 제한되는가.
- [ ] 자동 선택이 0을 반환하는 상황을 주입해 1로 fallback하는가.
- [ ] worker별 stats에 동시 쓰기가 겹치지 않는가.
- [ ] join 이후 합계가 모든 로컬 합과 같은가.
- [ ] 같은 독립 작업을 1 worker와 여러 worker로 실행했을 때 결과 배열이 같은가.
- [ ] 실행을 반복해도 checksum과 정수 작업량이 같은가.
- [ ] ThreadSanitizer로 공유 출력과 스케줄러에 data race가 없는지 검사할 수 있는가.
- [ ] tile이 매우 불균등할 때 동적 배정이 worker 유휴 시간을 줄이는가.
- [ ] thread 생성 비용이 작은 workload에서 이득을 상쇄할 수 있음을 설명할 수 있는가.

### 구현 후 설명할 것

1. relaxed atomic이 작업 번호 발급에는 충분한 메모리 모델 이유.
   - **모범답변:** 필요한 것은 `fetch_add` 자체의 원자성과 각 worker가 서로 다른 반환값을 받는 성질뿐입니다. 번호 발급 전후의 비원자 데이터 가시성을 이 atomic으로 전달하지 않으므로 추가 ordering이 결과를 바꾸지 않습니다.
2. 공유 이미지에 lock 없이 써도 되는 single-writer 영역 invariant.
   - **모범답변:** tile index는 한 번만 발급되고 각 tile의 좌표 범위는 겹치지 않으므로 각 pixel byte는 하나의 worker만 씁니다. 이미지 vector는 미리 완전히 할당되고 render 중 구조 변경이 없으며 join 후에만 소비합니다.
3. worker-local counter와 join 후 reduction을 택한 이유.
   - **모범답변:** hot path의 공유 atomic 경쟁을 제거하고 각 worker가 일반 정수만 수정하게 합니다. join 후 고정 worker index 순서로 합산하면 정수 통계가 정확하고 결정적입니다.
4. 동적 tile 스케줄링의 load balancing 장점과 실행 순서 비결정성.
   - **모범답변:** 빨리 끝난 worker가 다음 tile을 가져가므로 tile별 비용 차이를 흡수합니다. 어떤 worker가 어떤 tile을 처리하는지는 달라질 수 있지만 callback이 독립 영역에 순수한 결과를 쓰므로 이미지 의미는 변하지 않습니다.
5. exact pixel 결정성을 가능하게 한 계산 구조와 깨질 수 있는 변경 사례.
   - **모범답변:** 각 pixel의 광선·셰이딩·byte 변환이 다른 pixel이나 실행 순서에 의존하지 않고 single writer가 정해진 offset에 기록합니다. 공유 난수 생성기, 순서 의존적 floating reduction, 겹치는 tile, mutable Scene cache를 추가하면 이 보장이 깨질 수 있습니다.

### 원본 확인 위치

- Thread 09
- `feat(renderer): 작업자 수 설정과 자동 선택 추가`
- `test(render): 작업자 수에 따른 함수 결과 동치 검증`
- `test(render): 실행 모드별 PPM byte 결정성 검증`
- `include/ray/renderer.hpp`
- `src/renderer.cpp`
- `tests/render_tests.cpp`
- `tests/render_determinism.sh`
- `RenderSettings::threadCount`, `RenderStats`, `renderScene`
- 관련 Thread: 04의 재사용 카메라 프레임, 08의 읽기 전용 장면·가속 구조, 10의 결정적 작업량 계측, 12의 `--threads N|auto`, 13의 실행 모드 검증 gate
- 고정 크기 타일, atomic next-tile 배정, worker별 통계와 정렬된 저장 구조가 구현된 사실은 확인되지만 해당 도입 변경의 커밋 메시지는 현재 노출된 기록에서 확인되지 않아 별도 제목을 만들지 않았다.

---

<a id="p14"></a>
## [Thread 09 / `fix(renderer): 작업자 예외를 호출자에게 전달`] worker 예외의 중단·join·재전파 계약

### 면접 질문

`std::thread`의 실행 함수에서 예외가 밖으로 빠져나가면 어떤 문제가 생기며, 렌더러가 worker 예외를 `std::exception_ptr`로 수집한 뒤 모든 thread를 join하고 호출자 thread에서 다시 던져야 하는 이유를 설명해 보세요.

꼬리 질문:

- 예외를 잡은 worker가 `nextTile = tileCount`로 바꾸는 것은 즉시 취소와 무엇이 다릅니까?
  - **모범답변:** 아직 번호를 가져오지 않은 worker가 새 작업을 덜 시작하게 하는 협력적 중단일 뿐입니다. 이미 callback 안에 들어간 worker를 강제로 멈추지 못하고, 거의 동시에 fetch한 작업은 계속 실행될 수 있습니다.
- worker마다 별도 `exception_ptr` 슬롯을 두는 방식과 mutex로 첫 예외 하나를 저장하는 방식의 trade-off는 무엇입니까?
  - **모범답변:** per-worker 슬롯은 각 thread가 자기 원소만 써 lock이 없고 join 후 worker index 순으로 결정적 선택이 쉽지만 슬롯 메모리가 필요합니다. mutex 방식은 첫 예외 하나만 저장하지만 실패 경로에서 잠금 경쟁이 있고 시간상 첫 예외는 실행마다 달라질 수 있습니다.
- 여러 worker가 거의 동시에 실패하면 어떤 예외를 재전파할지 결정적 정책을 만들 수 있습니까?
  - **모범답변:** 가능합니다. 원본처럼 worker별 슬롯에 저장한 뒤 join 후 index가 가장 작은 non-null 슬롯을 재전파하면 scheduling과 무관한 선택 순서를 정의할 수 있습니다.
- 예외를 발견하자마자 main thread가 rethrow하면 안 되는 이유는 무엇입니까?
  - **모범답변:** 다른 worker가 여전히 callback과 공유 출력에 접근 중이고 joinable thread 객체도 남아 있습니다. 먼저 모두 join해 lifetime과 쓰기를 끝낸 뒤 재전파해야 stack unwinding 중 `std::terminate`와 use-after-lifetime을 피할 수 있습니다.
- thread 생성 도중 `emplace_back`이 실패하면 이미 만들어진 thread를 누가 회수합니까?
  - **모범답변:** workers vector를 참조하는 RAII `ThreadJoiner`가 담당합니다. 생성 loop 전에 joiner를 만들어 두면 예외로 stack이 풀릴 때 stop 값을 게시하고 이미 생성된 joinable thread를 모두 join합니다.
- RAII joiner가 destructor에서 예외를 던지면 안 되는 이유는 무엇입니까?
  - **모범답변:** 다른 예외로 stack unwinding 중 destructor가 또 던지면 `std::terminate`가 발생합니다. joiner는 소유한 joinable thread를 회수하는 noexcept cleanup 역할이어야 하며, 프로젝트는 정상적인 joinable thread의 `join`이 성공한다는 전제에서 destructor를 구성합니다.
- 일부 pixel이 이미 기록된 상태에서 실패하면 `renderScene`이 부분 이미지를 반환해도 됩니까?
  - **모범답변:** 원본은 모든 worker를 join한 뒤 예외를 다시 던져 `Image`를 반환하지 않습니다. 부분 pixel은 내부 임시 상태로 폐기되며, 호출자가 이를 완성 결과로 오해하지 않게 합니다.

### 30초 모범 답변

Thread 함수 밖으로 예외가 나가면 `std::terminate`가 호출될 수 있으므로 각 worker 경계에서 모든 예외를 잡아 `exception_ptr`로 저장해야 합니다. 실패 worker는 공유 작업 index를 종료 값으로 바꿔 새 tile 배정을 줄이지만, 이미 실행 중인 작업은 강제 취소하지 않는 협력적 중단입니다. 호출자 thread는 먼저 모든 worker를 join해 stack과 공유 자원의 lifetime을 안정화한 뒤 저장된 예외를 다시 던집니다. Thread 생성 중 예외에도 이미 생성된 thread가 남지 않도록 RAII joiner가 필요합니다. 렌더링 실패는 부분 결과 성공으로 해석하지 않고 함수 전체 실패로 전달하는 것이 계약을 단순하게 만듭니다.

### 답변 핵심 키워드

`std::terminate`, `catch (...)`, `std::exception_ptr`, `cooperative stop`, `no forced cancellation`, `join before rethrow`, `RAII joiner`, `thread-construction failure`, `deterministic error selection`, `no partial success`

### 백지 구현

**구현 목표**

여러 worker에서 작업을 실행하고, 어느 worker든 실패하면 새 작업 배정을 중단하며, 모든 생성된 thread를 회수한 뒤 호출자에게 예외를 재전파하는 실행기를 구현한다.

**인터페이스**

```cpp
using WorkerTask = std::function<void(std::size_t workIndex,
                                      unsigned int workerIndex)>;

void runWorkers(std::size_t workCount,
                unsigned int workerCount,
                const WorkerTask& task);
```

**입력과 출력**

- 입력: 작업 수, worker 수, 작업 callback.
- 출력: 모두 성공하면 정상 반환.
- 실패: 작업 callback에서 발생한 예외 중 명시된 정책의 하나를 호출자 thread에서 재전파.

**반드시 만족해야 할 조건**

- worker 함수 밖으로 예외가 탈출하지 않는다.
- 실패가 관찰되면 아직 배정되지 않은 새 작업의 시작을 가능한 한 줄인다.
- 이미 생성된 모든 joinable thread는 정상·실패 경로 모두에서 join된다.
- join이 끝나기 전에는 worker 예외를 호출자에게 다시 던지지 않는다.
- thread 생성 도중 예외가 나도 이미 생성한 thread를 회수한다.
- 여러 오류가 있으면 어떤 것을 선택할지 결정적인 규칙을 갖는다.
- 함수가 예외를 던질 때 성공 결과나 완료 통계를 반환하지 않는다.

**경계 조건**

- 작업 0개.
- worker 0개 입력의 방어 정책.
- 첫 작업에서 즉시 실패.
- 마지막 작업에서 실패.
- 여러 worker가 동시에 실패.
- 일부 worker는 이미 긴 작업을 수행 중임.
- thread 생성 도중 자원 부족 예외.
- callback이 표준 예외가 아닌 값을 던짐.

**실패 조건**

- worker 예외 때문에 프로세스가 terminate된다.
- 예외 경로에서 join하지 않은 thread destructor가 실행된다.
- 한 worker 실패 뒤에도 모든 미배정 작업이 계속 시작된다.
- main thread가 worker가 공유 상태를 쓰는 중에 예외를 재전파한다.
- 여러 worker가 같은 `exception_ptr` 객체에 data race로 쓴다.
- destructor에서 join 실패를 다시 던져 stack unwinding 중 terminate된다.

**제약**

- 실행 중인 callback을 강제로 중단하지 않는다.
- `std::jthread`나 `stop_token`이 없는 C++17을 기준으로 한다.
- 성공 결과 축약은 구현 범위 밖이다.
- worker 수는 이미 유효하고 work 수 이하로 조정되었다고 가정해도 되지만, 가정은 명시한다.

```cpp
void runWorkers(std::size_t workCount, unsigned int workerCount,
                const WorkerTask& task) {
    if (workCount == 0) return;
    if (workerCount == 0)
        throw std::invalid_argument("workerCount must be positive");
    workerCount = static_cast<unsigned int>(
        std::min<std::size_t>(workerCount, workCount));

    std::atomic<std::size_t> nextWork{0};
    std::vector<std::thread> workers;
    std::vector<std::exception_ptr> errors(workerCount);
    workers.reserve(workerCount);

    struct ThreadJoiner {
        std::vector<std::thread>& workers;
        std::atomic<std::size_t>& nextWork;
        std::size_t workCount;
        ~ThreadJoiner() {
            nextWork.store(workCount, std::memory_order_relaxed);
            for (std::thread& worker : workers)
                if (worker.joinable()) worker.join();
        }
    } joiner{workers, nextWork, workCount};

    for (unsigned int worker = 0; worker < workerCount; ++worker) {
        workers.emplace_back([&, worker] {
            try {
                for (;;) {
                    const std::size_t work =
                        nextWork.fetch_add(1, std::memory_order_relaxed);
                    if (work >= workCount) break;
                    task(work, worker);
                }
            } catch (...) {
                errors[worker] = std::current_exception();
                // 이미 실행 중인 callback은 끝나도록 두고 새 배정만 줄인다.
                nextWork.store(workCount, std::memory_order_relaxed);
            }
        });
    }

    for (std::thread& worker : workers) worker.join();
    // 낮은 worker index의 오류를 택하는 결정적 정책이다.
    for (const std::exception_ptr& error : errors)
        if (error) std::rethrow_exception(error);
}
```

### 구현 후 자가 검증

- [ ] 정상 작업에서 모든 index가 처리되고 모든 thread가 join되는가.
- [ ] 한 worker 예외가 동일한 타입과 메시지로 호출자에게 전달되는가.
- [ ] 표준 예외가 아닌 예외도 `exception_ptr`로 전달되는가.
- [ ] 실패 뒤 새 작업 시작 수가 협력적 중단 정책에 맞게 제한되는가.
- [ ] 동시에 여러 worker가 실패해도 data race가 없는가.
- [ ] 선택할 예외가 worker index 등 명시한 순서로 결정되는가.
- [ ] thread 생성 중간 실패를 모사해도 생성된 thread가 남지 않는가.
- [ ] 이미 실행 중인 callback이 끝날 때까지 join이 기다리는가.
- [ ] 실패 시 부분 성공 결과를 사용하지 않는가.
- [ ] ThreadSanitizer와 반복 실행에서 deadlock·terminate가 없는가.

### 구현 후 설명할 것

1. worker 경계에서 `catch (...)`와 `exception_ptr`가 필요한 이유.
   - **모범답변:** thread 함수 밖으로 어떤 예외든 빠지면 `std::terminate`가 호출됩니다. `catch (...)`는 비표준 예외까지 잡고 `current_exception()`은 원래 타입과 메시지를 보존해 호출자 thread에서 재전파할 수 있게 합니다.
2. stop flag 또는 work index 종료값이 협력적 중단일 뿐인 이유.
   - **모범답변:** worker는 다음 작업을 가져오는 경계에서만 stop 상태를 봅니다. 이미 callback을 실행 중인 thread를 C++17에서 안전하게 강제 종료하지 않으므로 join은 그 작업의 자연스러운 종료를 기다립니다.
3. join 후 rethrow 순서가 lifetime과 예외 안전성에 주는 효과.
   - **모범답변:** join이 모든 worker의 공유 상태 접근을 끝내고 해당 쓰기를 호출자에게 동기화합니다. 그 뒤 예외를 던지면 stack의 이미지·통계·thread 저장소가 더 이상 실행 중인 worker에게 참조되지 않습니다.
4. per-worker error 슬롯과 shared first-error 방식의 trade-off.
   - **모범답변:** 원본의 per-worker 슬롯은 lock 없이 기록하고 index 순으로 결정적 선택이 가능하지만 오류 수만큼 공간을 씁니다. shared first-error는 공간이 작지만 mutex나 CAS가 필요하고 무엇을 'first'로 볼지 scheduling 의존성이 생깁니다.
5. thread 생성 예외까지 포함한 RAII 회수 설계.
   - **모범답변:** joiner를 첫 `emplace_back` 전에 생성해 workers vector를 참조하게 합니다. 생성 중 예외가 발생해도 destructor가 종료 값을 게시하고 지금까지 생성된 모든 joinable thread를 회수한 뒤 원래 예외가 계속 전파됩니다.

### 원본 확인 위치

- Thread 09
- `fix(renderer): 작업자 예외를 호출자에게 전달`
- `test(renderer): 작업자 실패 전파와 회수 검증`
- `src/renderer.cpp`
- `tests/render_tests.cpp`
- `renderScene`, `ThreadJoiner`, worker별 `std::exception_ptr`
- 관련 Thread: 09의 타일 스케줄링, 11의 파일 출력 실패 전파, 12의 runtime error 종료 코드, 13의 sanitizer·회귀 gate
