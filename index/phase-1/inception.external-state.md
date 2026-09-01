# External-State Development Gap Audit — `inception`

## 감사 기준과 결론

첨부된 Index의 13개 Thread 제목과 소유 범위만 기존 구조로 사용했습니다. 기존 Thread별 해설서나 학습 문서는 분석 입력으로 사용하지 않았습니다. 

감사 범위는 다음과 같습니다.

* 초기 commit `24408b74...`부터 현재 `main` HEAD `47ca1a7e...`까지의 전체 commit 목록과 대표 commit diff
* 두 번째 commit API 페이지가 비어 있음을 통한 history 범위 종료 확인
* 현재 `main`의 recursive repository tree와 source/configuration
* 현재 branch/ref 및 protection 상태
* Docker Compose, Dockerfile, bootstrap, backup/restore, secret rotation, diagnostics, test harness, GitHub Actions workflow

**최종 판정은 External-State Gap 16개입니다.**

* `EXISTING_THREAD`: 15개
* `NEW_THREAD`: 0개
* `PROJECT_LEVEL_EXTERNAL_STEP`: 1개

독립적인 `NEW_THREAD`는 제안하지 않습니다. 발견된 독립 lifecycle은 이미 Thread 1~13 중 하나가 소유하고 있으며, 나머지 호스트 실행환경 준비는 새 Thread로 만들 만큼의 project-specific 구현 commit/lifecycle을 갖지 않습니다.

특히 중요한 발견은 **현재 CI workflow가 `web/inception` branch만 대상으로 하지만, 실제 공개 branch는 보호되지 않은 `main` 하나뿐이라는 점**입니다. 현재 ref 상태만 놓고 보면 해당 CI workflow를 실행하거나 필수 merge gate로 적용하는 branch 구성이 성립하지 않습니다.

---

# Part I — Gap Index

## ESG-01 — 호스트 실행환경과 도구 체인 준비

**Classification / Primary Owner.** `PROJECT_LEVEL_EXTERNAL_STEP` / Project level
**Related Threads.** T2, T10, T13

**Repository Evidence.** README와 Makefile은 Docker Engine, Docker Compose v2, Python, Make, curl 및 Docker daemon 접근을 전제로 한다. 이미지 최초 빌드에는 Debian snapshot, WordPress, WP-CLI 배포 위치에 대한 네트워크 접근도 필요하다.

**Required External Step.** 호스트 또는 VM에 요구 도구를 설치하고, 실행 사용자가 Docker daemon을 사용할 수 있도록 구성해야 한다. 이미지 빌드와 runtime resource 생성에 필요한 디스크·메모리·네트워크도 준비되어야 한다.

**실제 수행 여부 확인 가능성.** Git으로는 실제 설치 버전, daemon 실행 상태, 사용자 권한, 호스트 용량을 확인할 수 없다.

**Documentation Action.** 새 Thread가 아닌 단일 프로젝트 전제조건 문서로 추가한다.

---

## ESG-02 — Git 밖의 런타임 `.env` 구성

**Classification / Primary Owner.** `EXISTING_THREAD` / T2
**Related Threads.** T1, T4, T5, T6, T10, T11

**Repository Evidence.** `.env`는 처음부터 Git에서 제외되었고, `.env.example`이 공개 설정 계약을 정의한다. Compose는 환경값이 없으면 실패하도록 설정되었으며 Makefile은 기본적으로 `ENV_FILE=.env`를 사용한다. `7fec90fd`가 공개 환경변수 템플릿을 도입했고 `96809913`이 이를 각 서비스에 전달했으며, `41372f52`가 `.env`를 사용하는 stack lifecycle 명령을 연결했다.

**Required External Step.** 운영자는 `.env.example`을 바탕으로 `.env`를 만들고 다음을 하나의 일관된 runtime configuration으로 결정해야 한다.

* 도메인 및 WordPress URL
* HTTPS bind address와 port
* Compose project/image prefix 및 tag
* DB·WordPress 논리 이름과 계정 식별자
* 네 개 secret 원본 파일 경로

**실제 수행 여부 확인 가능성.** `.env`의 존재, 실제 값, 환경별 차이는 Git에서 확인할 수 없다.

**Documentation Action.** T2 보충 문서에 환경 파일 생성, 값 간 불변조건, 환경별 분리 경계를 추가한다.

---

## ESG-03 — 초기 host secret 파일 생성

**Classification / Primary Owner.** `EXISTING_THREAD` / T6
**Related Threads.** T2, T4, T5, T9

**Repository Evidence.** `.gitignore`는 `secrets/*.txt`를 제외한다. `916391b9`는 password를 host file 경로로 이동했고, `486ffb5c`는 secret 상위 디렉터리의 소유권·권한과 각 파일의 `0600`, 단일 hard-link, 일반 파일, 현재 사용자 소유, 길이·문자 형식을 검증하도록 했다. 현재 Compose의 `x-secret-files`와 `start_stack.py`는 이 파일을 읽어 bootstrap container의 표준입력으로만 전달한다.

**Required External Step.** 현재 사용자만 접근할 수 있는 디렉터리를 만들고, 다음 네 개의 서로 다른 경로에 유효한 secret 파일을 생성해야 한다.

* DB root password
* DB application password
* WordPress administrator password
* WordPress regular-user password

각 파일은 runtime validator가 요구하는 소유권, 권한, 파일 형식 조건을 만족해야 한다.

**실제 수행 여부 확인 가능성.** 실제 secret 값, 생성 도구, 생성 시점, 보관 매체는 확인할 수 없다.

**Documentation Action.** T6 보충 문서에 "초기 secret materialization" 절차와 검증 경계를 추가한다. 값 예시는 포함하지 않는다.

---

## ESG-04 — 로컬 image 및 build cache의 실제 생성

**Classification / Primary Owner.** `EXISTING_THREAD` / T10
**Related Threads.** T2, T13

**Repository Evidence.** `3e29fbd3`는 Debian base digest와 snapshot package source를 고정했고, `f60ac806`은 WP-CLI와 WordPress archive의 버전·checksum을 고정했다. 현재 Dockerfile도 pinned base image, snapshot repository, checksum 검증을 유지한다.

**Required External Step.** Docker daemon이 base image를 가져오고, snapshot package와 WordPress/WP-CLI 산출물을 다운로드하여 세 service image를 build/tag해야 한다. 실패한 partial layer나 이전 cache를 재사용할지 제거할지도 runtime build state의 일부다.

**실제 수행 여부 확인 가능성.** image ID, 실제 build 성공 여부, cache provenance, 로컬 tag가 현재 존재하는지는 Git으로 확인할 수 없다.

**Documentation Action.** T10 보충 문서에 source pin과 "실제 image materialization"을 분리하여 설명한다.

---

## ESG-05 — Compose project resource와 관리 lock의 실제 생성

**Classification / Primary Owner.** `EXISTING_THREAD` / T2
**Related Threads.** T3, T11, T13

**Repository Evidence.** Compose는 세 container, frontend/backend network, 세 named volume 및 project-scoped image를 정의한다. `start_stack.py`는 build, one-off bootstrap, service start를 실제 Docker 명령으로 연결한다. `e77c6f15`는 `/tmp/container-stack-operation-locks-<uid>` 아래에 project 이름 hash 기반 lock file을 생성하도록 했다.

**Required External Step.** 유효한 project 이름을 선택하고 `make up`, `make up-build` 또는 대응 Python command를 실행하여 Docker daemon에 project resource를 생성해야 한다. 동시 관리 작업 동안 host lock directory/file도 생성된다.

**실제 수행 여부 확인 가능성.** 실제 container/network/image ID, project label, lock 파일의 현존 여부, 생성·삭제 시점은 Git에서 확인할 수 없다.

**Documentation Action.** T2 보충 문서에 "declarative model"과 "materialized Docker project"의 차이를 추가한다.

---

## ESG-06 — persistent volume 생성·보존·파괴

**Classification / Primary Owner.** `EXISTING_THREAD` / T3
**Related Threads.** T2, T4, T5, T8

**Repository Evidence.** `75590ded`가 MariaDB와 WordPress named volume을 도입했고, `dc9601f5`가 MariaDB data staging layout과 별도의 `wordpress_config` volume을 추가했다. 현재 Compose에는 `mariadb_data`, `wordpress_data`, `wordpress_config`가 있으며 일반 `down`과 volume을 제거하는 `fclean`이 구분되어 있다.

**Required External Step.** Docker daemon이 project-scoped named volume을 생성해야 하며, service/container 재생성 중에는 이를 보존해야 한다. 영구 삭제는 명시적 destructive command 또는 실패 복원 정리에서만 수행해야 한다.

**실제 수행 여부 확인 가능성.** 실제 volume 이름, mountpoint, driver, content, 사용량 및 삭제 여부는 Git으로 확인할 수 없다.

**Documentation Action.** T3 보충 문서에 세 volume의 실제 lifecycle과 destructive boundary를 추가한다.

---

## ESG-07 — MariaDB database·account·marker bootstrap

**Classification / Primary Owner.** `EXISTING_THREAD` / T4
**Related Threads.** T2, T3, T6

**Repository Evidence.** `e13b0357`은 system table, root 계정, application database/user/grant 초기화를 도입했다. `dc9601f5`는 staging directory에서 초기화한 뒤 검증된 data directory와 completion marker를 게시하도록 변경했다. 현재 entrypoint는 bootstrap과 runtime mode를 분리하고 password를 표준입력으로 받는다.

**Required External Step.** 빈 MariaDB volume에 대해 one-off bootstrap container를 실행하여 다음 external state를 만들어야 한다.

* MariaDB system tables
* root password state
* application database
* application account와 grant
* completion marker
* 최종적으로 게시된 data directory

**실제 수행 여부 확인 가능성.** 특정 volume에서 bootstrap이 성공했는지, 실제 계정 password, DB row 또는 수행 시점은 Git으로 확인할 수 없다.

**Documentation Action.** T4 보충 문서에 "코드가 정의한 bootstrap"과 "실제 volume에 게시된 DB state"를 구분해 추가한다.

---

## ESG-08 — WordPress 설치·설정·사용자 state bootstrap

**Classification / Primary Owner.** `EXISTING_THREAD` / T5
**Related Threads.** T1, T3, T4, T6, T9

**Repository Evidence.** `d764d066`은 `wp core install`, administrator 및 regular-user 생성을 도입했다. 이후 `dc9601f5`는 one-off bootstrap과 completion marker, 별도 configuration volume을 도입했다. 현재 entrypoint는 image의 검증된 core 파일을 volume에 설치하고, DB password와 salts를 포함한 `wp-config.php`, site URL, administrator 및 author 상태를 재조정한다.

**Required External Step.** MariaDB가 준비된 뒤 WordPress bootstrap container를 실행하여 다음을 persistent state로 만들어야 한다.

* WordPress core/content 파일
* 분리된 configuration volume의 `wp-config.php`
* runtime에서 생성된 salts
* WordPress schema와 site options
* administrator와 regular-user account
* completion marker

**실제 수행 여부 확인 가능성.** 실제 salts, user ID, password hash, site option 값, 설치 시점 및 기존 content는 Git으로 확인할 수 없다.

**Documentation Action.** T5 보충 문서에 파일·configuration·DB에 분산되는 실제 WordPress bootstrap 결과를 추가한다.

---

## ESG-09 — HTTPS host binding과 self-signed TLS 산출물 생성

**Classification / Primary Owner.** `EXISTING_THREAD` / T1
**Related Threads.** T2, T11

**Repository Evidence.** `b3239712`는 nginx 시작 시 `openssl req -x509`로 certificate와 private key를 생성하도록 했고, `102af1f1`은 최종 경로를 `container-stack.crt/key`로 통일했다. 현재 Compose는 nginx만 `HTTPS_BIND_ADDRESS:HTTPS_PORT`를 host에 publish하며 `/etc/nginx/ssl`에는 persistent volume이 없다.

**Required External Step.** 사용 가능한 host address/port를 선택하고 nginx container를 시작해야 한다. entrypoint가 container writable layer에 self-signed certificate/key를 만들며, 접속 client는 해당 certificate를 명시적으로 허용하거나 별도로 신뢰시켜야 한다.

`DOMAIN_NAME`을 `localhost` 이외의 이름으로 구성할 경우에는 그 이름이 client에서 해석되도록 하는 local hosts/DNS 구성이 조건부로 필요하다. 저장소는 public DNS 등록이나 CA 발급을 구현하지 않는다.

**실제 수행 여부 확인 가능성.** 실제 certificate serial/key, 생성 시점, client trust store, port binding 성공 여부는 Git으로 확인할 수 없다.

**Documentation Action.** T1 보충 문서에 ephemeral certificate, host port conflict 및 client trust boundary를 추가한다.

---

## ESG-10 — Docker daemon에 의한 runtime guardrail 적용

**Classification / Primary Owner.** `EXISTING_THREAD` / T11
**Related Threads.** T2, T12, T13

**Repository Evidence.** `91154413`은 service별 CPU, memory, PID, `nofile`, stop signal/grace period, `no-new-privileges`, JSON log rotation 설정을 추가했다. Compose는 내부 backend network와 외부 frontend network도 분리한다. 진단 도구는 생성된 container state를 다시 수집하도록 되어 있다.

**Required External Step.** container를 생성 또는 force-recreate하여 Docker daemon이 제한·보안·logging·network 설정을 실제 container HostConfig와 namespace에 적용하도록 해야 한다.

**실제 수행 여부 확인 가능성.** 호스트 kernel/cgroup 지원, 실제 적용값, log file 현황, stop 동작 결과는 Git으로 확인할 수 없다.

**Documentation Action.** T11 보충 문서에 선언된 guardrail과 daemon에서 실효화된 guardrail의 검증 경계를 추가한다.

---

## ESG-11 — 일관된 backup set의 host filesystem 게시

**Classification / Primary Owner.** `EXISTING_THREAD` / T7
**Related Threads.** T2, T3, T6, T8

**Repository Evidence.** `fdd55605`는 private output과 checksum I/O를 정의했고, `6999190f`는 세 service가 실행 중인지 확인한 뒤 nginx/WordPress를 중지하고 DB dump, WordPress archive, manifest를 임시 디렉터리에 생성한 후 원자적으로 게시하도록 했다. 최종 source는 output reservation과 실패 시 service recovery를 포함한다.

**Required External Step.** 운영자는 안전한 host filesystem의 기존 parent 아래에 아직 존재하지 않는 output path를 선택하고 backup을 실행해야 한다. 성공 시 다음 external artifacts가 생성된다.

* `database.sql`
* `wordpress.tar.gz`
* `manifest.json`
* `0700` backup directory와 `0600` constituent files

**실제 수행 여부 확인 가능성.** 실제 backup 실행, 파일 내용, 크기, 보관 기간, 외부 복사, 암호화 여부는 Git으로 확인할 수 없다.

**Documentation Action.** T7 보충 문서에 host storage 준비, 원자 게시 결과 및 repository가 소유하지 않는 retention 책임을 추가한다.

---

## ESG-12 — fresh Compose project에 대한 restore와 실패 자원 rollback

**Classification / Primary Owner.** `EXISTING_THREAD` / T8
**Related Threads.** T2, T3, T4, T5, T7

**Repository Evidence.** `1250fcf7`은 backup DB와 WordPress archive를 새 volume에 주입하는 경로를 추가했고, `9ca04b1c`는 restore 실패 시 새 project의 container, volume, network를 제거하고 잔여 자원을 검사하도록 했다. Restore source는 시작 전에 target project의 기존 resource 충돌도 거부한다.

**Required External Step.** 검증된 backup source, 별도 `.env`와 secret set, 기존 Docker resource가 전혀 없는 fresh project 이름을 준비하고 restore를 실행해야 한다. 실행 과정에서 새 volume/network/container와 복원된 DB/filesystem state가 만들어진다.

**실제 수행 여부 확인 가능성.** 특정 project에서의 restore 성공·실패, 실제 복원 데이터, rollback 완료 여부는 Git으로 확인할 수 없다.

**Documentation Action.** T8 보충 문서에 fresh-project precondition, 생성되는 external resources 및 실패 rollback 확인 절차를 추가한다.

---

## ESG-13 — replacement secret staging과 다중 저장소 credential rotation

**Classification / Primary Owner.** `EXISTING_THREAD` / T9
**Related Threads.** T4, T5, T6, T11

**Repository Evidence.** `a2d20b8c`는 replacement secret을 private directory에서 읽고 host file에 원자적으로 게시하는 기능을 도입했다. `9934b478`은 WordPress account, `wp-config.php`, MariaDB application/root account, host secret files, container recreate 및 사후 검증을 하나의 rotation 절차로 연결했다. 실패 시 이전 credential로 보상 복구한다.

**Required External Step.** 현재 secret directory와 분리된 private replacement directory에 네 개의 새 secret 파일을 생성해야 한다. 각 replacement는 대응하는 기존 값과 달라야 하며, rotation command를 실행해 DB·WordPress·host filesystem·container state를 함께 전환해야 한다.

**실제 수행 여부 확인 가능성.** 실제 replacement 값, rotation 실행 시점, 일시적 혼합 상태, 최종 credential 세트는 Git으로 확인할 수 없다.

**Documentation Action.** T9 보충 문서에 replacement staging, external mutation points, 검증과 보상 복구 결과를 추가한다.

---

## ESG-14 — private diagnostic evidence set 생성

**Classification / Primary Owner.** `EXISTING_THREAD` / T12
**Related Threads.** T6, T11, T13

**Repository Evidence.** `27a083d9`는 진단 output directory를 Git에서 제외하고, Docker/Compose version, `compose ps`, logs, rendered model, container state를 수집해 secret을 redaction한 뒤 private file로 저장하도록 했다. 실패하면 partial destination을 제거한다.

**Required External Step.** 아직 존재하지 않는 output path를 지정하고 실행 중인 Docker project를 대상으로 diagnostics command를 실행해야 한다. 결과 디렉터리는 `0700`, 개별 파일은 `0600`으로 생성되며, 외부 공유 전에는 운영자가 내용을 검토해야 한다.

**실제 수행 여부 확인 가능성.** 실제 진단 수집, redaction된 내용, 공유·보존·삭제 여부는 Git으로 확인할 수 없다.

**Documentation Action.** T12 보충 문서에 evidence directory lifecycle과 "redaction 후에도 공유 전 검토" 경계를 추가한다.

---

## ESG-15 — CI runner의 임시 env·secret·Docker resource·artifact lifecycle

**Classification / Primary Owner.** `EXISTING_THREAD` / T13
**Related Threads.** T10, T11, T12

**Repository Evidence.** `18508c25`는 Ubuntu runner에서 정적 검사와 실제 stack 테스트를 실행하고, 실패 시 diagnostics artifact를 7일간 게시하도록 했다. `runtime_stack.py`는 각 test에 private temporary directory, 임시 `.env`, 임시 secret files, random project/image name, host port 및 Docker resource를 생성한다. Cleanup script는 기록된 test project의 container/volume/network/image만 제거한다.

**Required External Step.** GitHub Actions 또는 호환 runner에서 Docker와 BuildKit을 사용할 수 있어야 한다. Workflow가 실행되면 임시 secret, environment, image, container, volume, network, diagnostic artifact가 생성되며 cleanup 단계가 이를 회수해야 한다.

**실제 수행 여부 확인 가능성.** Git source만으로 Actions enablement, runner capacity, 실제 workflow run, cleanup 성공, artifact 업로드를 확인할 수 없다.

**Documentation Action.** T13 보충 문서에 CI가 소비·생성하는 external resources와 leak cleanup 책임을 추가한다.

---

## ESG-16 — CI trigger branch와 merge-gate 외부 상태 불일치

**Classification / Primary Owner.** `EXISTING_THREAD` / T13
**Related Threads.** 없음

**Repository Evidence.**

* `18508c25`의 최초 workflow는 `main` push, pull request, manual dispatch를 대상으로 했다.
* `963b77e4`는 이를 `web/inception` branch의 push/PR로 변경하고 manual dispatch를 제거했다.
* 현재 repository의 default branch는 `main`이다.
* 현재 공개 branch 목록에는 보호되지 않은 `main` 하나만 존재하고 `web/inception`은 없다.

**Required External Step.** 다음 중 하나로 source와 repository ref 상태를 일치시켜야 한다.

1. 실제 target branch를 `web/inception`으로 생성·운영한다.
2. Workflow trigger를 실제 운영 branch인 `main`으로 돌린다.
3. 필요하다면 manual dispatch를 다시 제공한다.

Thread 13의 "CI Gate"를 권고성 workflow가 아니라 필수 merge gate로 성립시키려면, 정렬된 target branch의 merge policy에서 해당 workflow status를 요구해야 한다. 현재 `main`은 unprotected로 보고된다.

**실제 수행 여부 확인 가능성.** 현재 branch/ref와 `main`의 unprotected 상태는 직접 확인된다. 반면 Actions가 별도 설정으로 활성화되었는지, 과거 다른 repository에서 run되었는지, 어떤 status check가 merge 조건으로 사용되었는지는 Git history만으로 확인할 수 없다.

**Documentation Action.** T13 보충 문서의 최우선 항목으로 기록한다. 기존 Thread의 commit 구성을 변경할 필요는 없지만, 현재 repository에서 gate를 활성화하려면 외부 branch/rule 조치가 필요하다.

---

# Part II — Existing Thread Supplement Packets

## Packet T1 — HTTPS 진입점과 종단 요청 경로

### HTTPS Entry Point and End-to-End Request Flow

**Thread Identity.** Existing Thread 1
**Gaps.** ESG-09

### Repository Evidence

대표 commit:

* `b32397121bb1601f88a85c4e6f0c9db0704d894f` — `feat(nginx): TLS 프런트엔드 이미지 추가`

  * 파일: `srcs/requirements/nginx/Dockerfile`
  * 파일: `srcs/requirements/nginx/tools/docker-entrypoint.sh`
  * Diff 핵심: nginx 시작 시 `DOMAIN_NAME`을 certificate CN으로 사용하여 self-signed certificate/key를 생성한다.
* `102af1f113edbf0ae3e3ff2cfca06eecfe8033e9` — `refactor(nginx): 스택 전용 TLS 산출물 이름 사용`

  * 파일: `srcs/requirements/nginx/conf/nginx.conf`
  * 파일: nginx entrypoint
  * Diff 핵심: generator와 nginx configuration이 `container-stack.crt/key`를 동일하게 참조하도록 정렬했다.

Final-state configuration은 nginx만 host port를 publish하며 certificate directory에는 volume을 연결하지 않는다.

### External Development Steps

1. `.env`에서 host bind address, HTTPS port, domain/URL을 선택한다.
2. 해당 host port가 다른 process/project와 충돌하지 않는지 확인한다.
3. nginx container를 생성·시작하여 certificate와 key를 만들게 한다.
4. client에서 self-signed certificate를 허용하거나 필요한 trust 구성을 한다.
5. 비-local domain을 사용한다면 client name resolution을 별도로 성립시킨다.

### Code Connection

* `HTTPS_BIND_ADDRESS`와 `HTTPS_PORT` → Compose port publication
* `DOMAIN_NAME` → certificate CN
* 생성된 certificate/key → nginx `ssl_certificate` 설정
* WordPress volume → nginx의 HTTPS content/FastCGI request path

### Evidence Boundary

**Directly observed in repository.** Certificate 생성 명령, 파일 경로, TLS 1.2/1.3 nginx 설정, host port publication.

**Required/inferred from repository.** 포트가 실제로 사용 가능해야 하며 client가 self-signed certificate를 처리해야 한다.

**Actual execution not observable from Git.** 실제 certificate/key, port listener, trust store, DNS/hosts entry.

### Ordering

**Conceptual execution order:** env 선택 → image build → nginx container 생성 → certificate/key 생성 → host port bind → HTTPS client 접속.

---

## Packet T2 — 프로젝트 단위 스택 오케스트레이션

### Project-Scoped Stack Orchestration

**Thread Identity.** Existing Thread 2
**Gaps.** ESG-02, ESG-05

### Repository Evidence

대표 commit:

* `7fec90fdafed110a0eabfd88a069314e95632758` — `feat(env): 공개 스택 환경 변수 정의`

  * `.env.example`에 stack의 공개 runtime 입력을 추가했다.
* `968099138c58e194f3590fc2cafc3c489d70826b` — `feat(compose): 공개 스택 설정 전달`

  * service별 environment mapping을 추가했다.
* `41372f52d3d6dce00b30199b151b5ff1113a82d9` — `build(make): 스택 수명주기 명령 추가`

  * `.env`를 사용하는 build/up/down/fclean/config command를 추가했다.
* `e77c6f151b07c62efd4ae8534fb1a6955c7f2dfe` — `refactor(runtime): 프로젝트 관리 작업 잠금 공통화`

  * host `/tmp`에 project별 관리 lock을 생성한다.

현재 `start_stack.py`는 database bootstrap과 application start를 단계별 Docker command로 실행한다.

### External Development Steps

1. 실제 `.env`를 작성한다.
2. secret 원본 경로와 project/image identity를 정한다.
3. 동일 project에 대한 다른 관리 작업이 없는 상태에서 build/start를 실행한다.
4. Docker daemon이 project image, container, network, volume을 생성하도록 한다.
5. 운영 중 down/recreate/fclean의 차이를 적용한다.

### Code Connection

* `.env` → Compose interpolation과 service bootstrap identity
* `PROJECT_NAME` → Docker resource naming/labels와 operation lock key
* `STACK_IMAGE_PREFIX/TAG` → local image name
* `start_stack.py` → bootstrap container와 runtime container의 실제 생성 순서
* operation lock → backup/restore/rotation/start 간 동시 실행 차단

### Evidence Boundary

**Directly observed.** 환경변수 계약, Compose lifecycle command, project lock 구현.

**Required/inferred.** 운영자가 실제 환경 파일을 작성하고 lifecycle command를 실행해야 한다.

**Actual execution not observable.** 사용된 `.env`, project 이름, Docker object ID, lock file.

### Ordering

**Conceptual execution order:** host 준비 → `.env`/secret 준비 → project identity 선택 → image build → bootstrap → application start → 운영 관리 → down 또는 명시적 파괴.

---

## Packet T3 — 영속 상태 분할과 재생성 불변 조건

### Persistent State Partitioning and Recreation Invariants

**Thread Identity.** Existing Thread 3
**Gaps.** ESG-06

### Repository Evidence

* `75590dedfb3aba2e31395354cd471ccae8e78966` — `feat(compose): 준비 상태에 따라 영속 서비스 연결`

  * MariaDB와 WordPress named volume을 도입했다.
* `dc9601f5e6709b86f4764117734f78bd6af59e96` — `fix(init): 중단된 단계별 초기화를 수렴`

  * MariaDB volume 내부를 staging/final data로 분리하고 `wordpress_config` volume을 추가했다.
* 현재 Compose:

  * `mariadb_data`
  * `wordpress_data`
  * `wordpress_config`

### External Development Steps

1. Project 최초 실행 시 세 named volume을 생성한다.
2. MariaDB bootstrap 결과를 DB volume에 게시한다.
3. WordPress core/content와 configuration을 서로 다른 volume에 게시한다.
4. container restart/recreate에서는 volume을 보존한다.
5. `fclean`이나 failed restore cleanup에서만 의도적으로 volume을 제거한다.

### Code Connection

* `mariadb_data` → MariaDB final data directory와 completion marker
* `wordpress_data` → core/content 및 WordPress marker
* `wordpress_config` → DB credential과 salts를 포함한 generated configuration
* `down`/`fclean` → 보존과 파괴의 경계

### Evidence Boundary

**Directly observed.** Volume 선언, mount destination, destructive command.

**Required/inferred.** Docker daemon이 실제 named volume을 만들고 mount해야 한다.

**Actual execution not observable.** 실제 volume content, driver, mountpoint, 삭제 여부.

### Ordering

**Conceptual execution order:** volume 생성 → DB state 게시 → WordPress data/config 게시 → container 재생성 중 보존 → 명시적 파괴 또는 restore rollback.

---

## Packet T4 — MariaDB 충돌 안전 초기화와 상태 게시

### Crash-Safe MariaDB Bootstrap and State Publication

**Thread Identity.** Existing Thread 4
**Gaps.** ESG-07

### Repository Evidence

* `e13b0357a21ba1b5bfb0d6812e0c5ce962ba4a99` — `feat(mariadb): DB와 애플리케이션 계정 초기화`

  * system table, root account, application DB/account/grant 초기화를 추가했다.
* `dc9601f5e6709b86f4764117734f78bd6af59e96` — `fix(init): 중단된 단계별 초기화를 수렴`

  * staging directory, temporary server, 검증, completion marker, atomic final publication을 추가했다.

현재 entrypoint는 runtime mode에서 valid data directory와 marker가 없으면 실행을 거부한다.

### External Development Steps

1. 빈 MariaDB volume을 준비한다.
2. Bootstrap container에 root/app password를 표준입력으로 전달한다.
3. Staging directory에서 system table을 초기화한다.
4. Temporary server에서 DB, root/app account, grants를 만든다.
5. Credential 및 DB access를 검증한다.
6. Marker와 final data directory를 게시한다.
7. Runtime MariaDB container를 시작한다.

### Code Connection

* `.env`의 `MYSQL_DATABASE`, `MYSQL_USER` → logical DB/account
* host secret 두 개 → bootstrap credential
* marker → healthcheck와 runtime entry 조건
* volume → 실제 database state

### Evidence Boundary

**Directly observed.** SQL initialization, staging/marker logic, runtime refusal 조건.

**Required/inferred.** One-off bootstrap을 실제 volume에 실행해야 한다.

**Actual execution not observable.** 실제 system table, grants, password, row data, marker 생성 시점.

### Ordering

**Conceptual execution order:** volume 준비 → staging init → temporary DB → account/DB 생성 → 검증 → marker/final publish → runtime start.

---

## Packet T5 — WordPress 재조정 초기화와 설정 격리

### Reconciliatory WordPress Bootstrap and Configuration Isolation

**Thread Identity.** Existing Thread 5
**Gaps.** ESG-08

### Repository Evidence

* `d764d066167bc866e0f56f347cae175dda22d1a5` — `feat(wordpress): 사이트와 사용자 계정 초기화`

  * WordPress install, administrator, author 생성을 도입했다.
* `dc9601f5e6709b86f4764117734f78bd6af59e96`

  * one-off bootstrap, separate config volume 및 marker를 도입했다.
* `f60ac8061c0179d03c237edca9666a22289c408e`

  * 검증된 image-bundled WordPress core를 volume에 재조정하도록 했다.

### External Development Steps

1. MariaDB bootstrap 완료를 기다린다.
2. Image의 검증된 core/content를 WordPress data volume에 게시한다.
3. Configuration volume에 DB credential, URL, salts를 포함하는 `wp-config.php`를 생성한다.
4. WordPress schema/site를 설치한다.
5. Administrator 및 regular-user를 생성·재조정한다.
6. 모든 상태를 검증하고 marker를 게시한다.
7. PHP-FPM runtime container를 시작한다.

### Code Connection

* DB application credential → `wp-config.php` 및 DB 연결
* WordPress URL/title/email/user variables → DB site/account state
* image core manifest → data volume의 core 파일
* marker → healthcheck와 nginx 시작 dependency

### Evidence Boundary

**Directly observed.** Core 복사/검증, configuration 생성, `wp core install`, user command, marker.

**Required/inferred.** Bootstrap container가 실제 DB와 volume에 이를 수행해야 한다.

**Actual execution not observable.** 실제 salts, password hashes, account IDs, installed site state.

### Ordering

**Conceptual execution order:** DB ready → core/content 게시 → config 생성 → core install → users/options 재조정 → 검증/marker → PHP-FPM start.

---

## Packet T6 — 호스트 Secret 검증과 런타임 비노출 경계

### Host Secret Validation and Runtime Non-Exposure

**Thread Identity.** Existing Thread 6
**Gaps.** ESG-03

### Repository Evidence

* `038d2dc2237341f17c6efec865926183cd99e34f`

  * `.env`와 `secrets/*.txt`를 Git에서 제외했다.
* `916391b9f8db5bfd696d61e2d8372eac848e43c8`

  * password를 host secret file 입력으로 변경했다.
* `486ffb5c65aa99cc1d9309e1e1e74e4ec13029d3`

  * private parent directory, `0600`, ownership, no-follow, format 및 canonical path 검증을 공통화했다.

Final state에서는 secret 파일이 long-running container에 mount되지 않고 one-off bootstrap payload로 전달된다.

### External Development Steps

1. Private secret directory를 생성한다.
2. 네 secret 파일을 안전한 난수원 등으로 생성한다.
3. Directory와 파일의 소유권/권한을 설정한다.
4. `.env`에서 각 원본 경로를 지정한다.
5. `config-strict` 또는 startup validator를 통과시킨다.
6. Bootstrap 시에만 값을 읽어 표준입력으로 전달한다.

### Code Connection

* Secret file path → `x-secret-files`
* Host validator → startup/backup/rotation 전 입력 검증
* Secret payload → MariaDB/WordPress one-off bootstrap
* Generated WordPress config/DB account → persistent secret-dependent state

### Evidence Boundary

**Directly observed.** 파일 경계와 validator, Git exclusion, runtime delivery 방식.

**Required/inferred.** 실제 안전한 값과 파일을 운영자가 만들어야 한다.

**Actual execution not observable.** 값, 생성 방식, host filesystem 위치, 별도 백업 여부.

### Ordering

**Conceptual execution order:** private directory → secret 파일 → `.env` path → validation → bootstrap 사용 → 장기 runtime 비노출 → 이후 rotation.

---

## Packet T7 — 일관된 백업 세트의 원자적 생성

### Atomic Creation of a Consistent Backup Set

**Thread Identity.** Existing Thread 7
**Gaps.** ESG-11

### Repository Evidence

* `fdd55605ba749ee29457745a62dd13688891f77d`

  * checksum과 private output primitive를 추가했다.
* `6999190ffd3423bafa674b5aaf1f1c3bf759249f`

  * running-state 검사, application stop, DB dump, WordPress archive, manifest, atomic publication, service recovery를 연결했다.
* 최종 source는 output path reservation, path/permission 검증 및 project operation lock을 사용한다.

### External Development Steps

1. 세 service를 healthy 상태로 실행한다.
2. 충분한 용량과 안전한 권한을 가진 host parent directory를 고른다.
3. 존재하지 않는 backup destination을 지정한다.
4. Project operation lock을 취득한다.
5. Application traffic을 중지하고 DB dump와 WordPress archive를 생성한다.
6. Checksum manifest를 만들고 temporary set을 검증한다.
7. Destination에 원자적으로 게시하고 서비스를 다시 시작한다.
8. Retention, 외부 복제 또는 삭제는 별도 운영 정책으로 수행한다.

### Code Connection

* MariaDB root secret → DB dump authentication
* WordPress volumes → tar archive
* manifest checksum → restore input 검증
* project lock → start/restore/rotation과 충돌 방지

### Evidence Boundary

**Directly observed.** 세 output 파일, permission, atomic replace, restart recovery.

**Required/inferred.** Host storage와 실행 시점은 운영자가 결정해야 한다.

**Actual execution not observable.** 실제 backup set, retention, remote copy, encryption.

### Ordering

**Conceptual execution order:** healthy stack → destination 예약 → lock/stop → dump/archive → checksum/manifest → atomic publish → service recovery → 외부 retention.

---

## Packet T8 — 신규 프로젝트 복원과 실패 자원 롤백

### Fresh-Project Restore and Failed-Resource Rollback

**Thread Identity.** Existing Thread 8
**Gaps.** ESG-12

### Repository Evidence

* `1250fcf7c00679317dbec68e45213ce60a279f53`

  * DB dump를 MariaDB에 입력하고 WordPress archive를 빈 volume에 풀도록 했다.
* `9ca04b1c30cdb082e858ae190cf1ae3e4d689ec3`

  * 실패 시 `down --volumes`, 잔여 container/volume/network 검사와 rollback을 추가했다.

### External Development Steps

1. Backup manifest/checksum을 검증한다.
2. 기존 container/volume/network가 없는 target project 이름을 선택한다.
3. Target `.env`와 secret 파일을 준비한다.
4. 새 MariaDB volume을 bootstrap하고 DB dump를 주입한다.
5. 빈 WordPress data/config volume에 archive를 복원한다.
6. Application을 시작하고 종단 상태를 검증한다.
7. 실패하면 새 project에 한정된 모든 resource를 제거한다.

### Code Connection

* Backup manifest → 입력 무결성
* Target project identity → 생성/rollback resource 범위
* Target secret set → 복원된 DB/runtime의 current credential
* `start_database`/`start_application` → 복원 단계 경계

### Evidence Boundary

**Directly observed.** Fresh-project 검사, DB/filesystem restore, cleanup algorithm.

**Required/inferred.** 실제 target identity, secret set, restore invocation이 필요하다.

**Actual execution not observable.** 복원된 데이터, rollback 결과, 실제 target resources.

### Ordering

**Conceptual execution order:** input 검증 → fresh target 검사 → DB resource/restore → WordPress restore → application start → 검증 또는 failed-resource rollback.

---

## Packet T9 — 자격증명 회전과 보상 복구

### Credential Rotation and Compensating Recovery

**Thread Identity.** Existing Thread 9
**Gaps.** ESG-13

### Repository Evidence

* `a2d20b8c2c033c50f69365777d38b9738bf9f64d`

  * replacement file 검증과 atomic host-file publication을 추가했다.
* `9934b478c79a702cd21d2607b25db8bb5582dff2`

  * WordPress users, config, MariaDB app/root account, host files, force-recreate, 사후 검증 및 rollback을 연결했다.

### External Development Steps

1. 현재 secret source와 다른 private directory를 만든다.
2. 네 replacement file을 `0600`으로 생성한다.
3. 각각이 기존 대응 값과 다른지 검증한다.
4. Project lock을 취득하고 nginx를 중지한다.
5. WordPress user passwords와 DB config를 변경한다.
6. MariaDB app/root account를 변경한다.
7. Host secret files를 원자 교체한다.
8. Container를 force-recreate하고 새 값 성공·기존 값 거부를 검증한다.
9. 실패하면 이전 세트로 보상 복구한다.

### Code Connection

* Replacement files → target credentials
* WordPress DB config와 user table → application-side state
* MariaDB account table → DB-side state
* Host files → 다음 container/bootstrap/관리 작업의 credential source
* Force recreate → long-running process state 재동기화

### Evidence Boundary

**Directly observed.** Mutation 순서, 검증, rollback 구현.

**Required/inferred.** Replacement materialization과 실제 회전 실행이 필요하다.

**Actual execution not observable.** 실제 값, 성공/rollback 여부, 중간 혼합 상태.

### Ordering

**Conceptual execution order:** replacement staging → 사전검증 → ingress 차단 → WordPress 변경 → DB 변경 → host file 게시 → recreate/검증 → 실패 시 보상.

---

## Packet T10 — 재현 가능한 이미지 공급망

### Reproducible Image Supply Chain

**Thread Identity.** Existing Thread 10
**Gaps.** ESG-04

### Repository Evidence

* `3e29fbd34389337124bd09f052d02113746946e0`

  * Debian image digest와 snapshot package source를 고정했다.
* `f60ac8061c0179d03c237edca9666a22289c408e`

  * WP-CLI/WordPress 버전과 SHA-256을 고정하고 image 내부 source tree를 만들었다.
* 현재 Dockerfile은 이 pinning 구조를 유지한다.

### External Development Steps

1. Docker daemon과 BuildKit을 준비한다.
2. Pinned base digest를 registry에서 가져온다.
3. Debian snapshot에서 package metadata와 package를 다운로드한다.
4. WordPress/WP-CLI 산출물을 다운로드하고 checksum을 검증한다.
5. 세 service image를 build하고 project-scoped tag로 게시한다.
6. 필요 시 stale image/cache를 명시적으로 정리한다.

### Code Connection

* Dockerfile source pins → build inputs
* `STACK_IMAGE_PREFIX/TAG` → local image identity
* WordPress core manifest → bootstrap volume reconciliation
* CI/runtime test → 실제 image 실행 검증

### Evidence Boundary

**Directly observed.** Pin, digest, snapshot URL, checksum, image naming.

**Required/inferred.** External distribution servers에 접근하여 build를 수행해야 한다.

**Actual execution not observable.** 실제 image ID/cache, registry response, build success.

### Ordering

**Conceptual execution order:** network/tool 준비 → base pull → package download → artifact checksum → build/tag → runtime/CI verification.

---

## Packet T11 — 런타임 격리와 운영 보호 장치

### Runtime Isolation and Operational Guardrails

**Thread Identity.** Existing Thread 11
**Gaps.** ESG-10

### Repository Evidence

* `911544133fb4f9e1e7832bb8925eb10ce731bf28`

  * CPU, memory, PID, ulimit, stop semantics, `no-new-privileges`, log rotation을 service별로 추가했다.
* Current Compose는 MariaDB backend network를 `internal`로 유지하고 nginx만 host port를 공개한다.

### External Development Steps

1. Host kernel/Docker가 설정된 제한을 지원하는지 확인한다.
2. Compose configuration을 rendering한다.
3. 기존 container가 있다면 recreate하여 새로운 guardrail을 적용한다.
4. `docker inspect`, diagnostics 또는 runtime test로 실제 적용값을 검증한다.
5. Log rotation과 graceful stop을 실제 운영 중 확인한다.

### Code Connection

* Compose limit fields → Docker HostConfig/cgroup
* Network declarations → network namespace와 connectivity
* Stop signal/grace period → backup/rotation/down 안전성
* JSON logging options → host Docker log state

### Evidence Boundary

**Directly observed.** 선언값과 이를 검사하는 source.

**Required/inferred.** Container 생성/recreate가 있어야 daemon에서 실효화된다.

**Actual execution not observable.** Effective cgroup/ulimit, log 파일, stop 결과.

### Ordering

**Conceptual execution order:** host capability → config render → create/recreate → daemon 적용 → inspect/test → 운영 관찰.

---

## Packet T12 — 비공개 장애 진단과 증거 수집

### Private Failure Diagnostics and Evidence Collection

**Thread Identity.** Existing Thread 12
**Gaps.** ESG-14

### Repository Evidence

* `27a083d91c8792d35a5569342484b861e85af71a`

  * private output directory와 CLI/Make target을 연결하고 `diagnostics/`를 Git에서 제외했다.
* Final source는 version, ps, logs, model, container state를 수집해 secret value와 path를 redaction하고 `0600`으로 기록한다.

### External Development Steps

1. 존재하지 않는 destination을 지정한다.
2. 실행 중인 project를 대상으로 수집을 실행한다.
3. Redaction을 거쳐 five-file evidence set을 만든다.
4. 공유 전에 결과를 수동 검토한다.
5. 문제 해결 후 보존 또는 안전 삭제 정책을 적용한다.

### Code Connection

* Docker/Compose commands → runtime evidence source
* Current secret files → redaction token source
* Container inspect → guardrail/network/resource evidence
* Output directory → Git 외부의 private evidence state

### Evidence Boundary

**Directly observed.** 수집 항목, redaction, permission, partial cleanup.

**Required/inferred.** 실제 장애 시 운영자가 수집·검토해야 한다.

**Actual execution not observable.** 실제 logs/state, 수집 시점, 외부 공유.

### Ordering

**Conceptual execution order:** destination 준비 → collect → redact/write → 수동검토 → 제한된 공유 → retention/delete.

---

## Packet T13 — 격리형 검증 하네스와 CI 게이트

### Isolated Verification Harness and CI Gate

**Thread Identity.** Existing Thread 13
**Gaps.** ESG-15, ESG-16

### Repository Evidence

대표 commit:

* `18508c25eef0d620fc6724673c0cdf98ecabb42b`

  * Ubuntu runner, 실제 stack scenario, cleanup, failure artifact upload를 workflow로 연결했다.
* `963b77e4de394e2847ec9f36c16f33612e86b1e7`

  * workflow trigger를 `main`에서 `web/inception` push/PR로 변경하고 manual dispatch를 제거했다.
* `2b35aa3d...`

  * 현재 test run이 기록한 project resource만 cleanup하는 도구를 도입했다.
* 현재 `runtime_stack.py`는 임시 `.env`, secret files, random port/project/image 및 Docker resources를 만든다.

현재 repository external/ref state:

* default branch: `main`
* 공개 branch: `main` 하나
* `main`: `protected=false`
* workflow target: 존재하지 않는 `web/inception`

### External Development Steps

1. Workflow target branch와 실제 repository branch를 정렬한다.
2. GitHub Actions 또는 대응 CI 실행을 활성화한다.
3. Docker/BuildKit 사용이 가능한 runner를 제공한다.
4. Runner에서 임시 env/secret/project/resource를 생성해 scenario를 실행한다.
5. 항상 cleanup을 실행하고 잔여 resource를 검사한다.
6. 실패 시 private diagnostic artifact를 게시·보존한다.
7. 필수 gate가 목적이라면 target branch의 merge rule이 workflow status를 요구하도록 한다.

### Code Connection

* Workflow trigger → 실행 가능한 repository event
* Runtime harness → temporary external state
* Cleanup project records → resource ownership boundary
* Diagnostic artifact → failure evidence publication
* Required status policy → "test workflow"를 "merge gate"로 전환하는 외부 상태

### Evidence Boundary

**Directly observed.** Workflow definition, target branch filter, current branch/ref/protection 상태, test resource lifecycle.

**Required/inferred.** Branch 정렬, Actions 실행환경, merge status 정책이 필요하다.

**Actual execution not observable.** 실제 runner job, artifact, cleanup 결과, 과거 다른 repository의 gate 설정.

### Ordering

**Conceptual execution order:** branch/trigger 정렬 → Actions/merge rule 설정 → checkout → 임시 env/secret/resource 생성 → 검증 → cleanup → failure artifact → required status 반영.

---

# Part III — Proposed New Thread Packets

## 제안된 NEW_THREAD 없음

다음 이유로 신규 Thread를 만들지 않았습니다.

1. `.env` materialization과 Docker resource 생성은 T2의 실제 실행 완성 단계다.
2. Secret 생성은 T6, rotation은 T9가 각각 lifecycle을 이미 소유한다.
3. DB 및 WordPress runtime state는 T4와 T5가 구분하여 소유한다.
4. Certificate 생성은 HTTPS entry point를 완성하므로 T1 소유다.
5. Image materialization, backup, restore, diagnostics, CI external state는 각각 T10, T7, T8, T12, T13에 이미 독립 관점이 있다.
6. Host toolchain provisioning은 중요하지만 project-specific 구현 commit과 자체 복구 lifecycle이 부족하므로 Project-Level Step이 적합하다.
7. CI branch/gate 문제는 독립 시스템 관점이 아니라 T13이 실제 repository에서 성립하기 위한 외부 활성화 단계다.

따라서 기존 Thread 제목·commit 구성의 재정의나 새로운 commit grouping은 제안하지 않습니다.

---

# Part IV — Project-Level External Steps

## PL-01 — Host Runtime Provisioning

**Related Gap.** ESG-01

### Repository Evidence

README와 Makefile은 Docker Engine, Compose v2, Python, Make, curl을 요구하며, build source는 외부 artifact와 package snapshot에 접근한다. CI도 Docker가 제공되는 Ubuntu runner를 전제로 한다.

### Required External Step

프로젝트를 처음 실행하기 전에 다음 host state를 성립시켜야 한다.

1. Docker Engine 및 Compose v2 설치
2. Docker daemon 실행
3. 실행 사용자의 daemon 접근 권한
4. Python·Make·curl 설치
5. Build/output/volume을 위한 충분한 host capacity
6. 최초 image build에 필요한 outbound access
7. 선택한 HTTPS host port의 사용 가능성

### Code Connection

* Makefile과 Python tools → host commands
* Dockerfile/Compose → Docker daemon
* Image build → external artifact/package sources
* Named volume/backup/diagnostics → host storage
* Port publication → host network namespace

### Evidence Boundary

**Directly observed in repository.** 요구 command와 외부 download endpoint, Docker resource model.

**Required/inferred from repository.** 실제 host/VM 또는 CI runner를 provision해야 한다.

**Actual execution not observable from Git.** Host OS, package version, daemon state, user group/ACL, storage 및 firewall 상태.

### Ordering

**Conceptual execution order:** host/VM 준비 → tool 설치 → daemon/권한 → storage/network 확인 → project `.env`와 secrets 준비 → image build 및 stack 생성.

---

## 채택하지 않은 일반적 External Steps

다음 항목은 이 repository에서 구체적인 필요성을 확인할 수 없거나 명시적으로 지원 범위 밖이므로 Gap으로 만들지 않았습니다.

* Public CA certificate 발급·자동 갱신
* Public DNS 또는 domain verification
* 외부 secret manager
* Object storage, bucket, IAM
* Scheduled/remote/encrypted backup
* Scheduler 또는 cron 등록
* OAuth provider, redirect URI, webhook
* Production/staging provisioning
* Kubernetes, multi-host orchestration
* CI/CD deployment credential

README도 이 stack이 self-signed TLS와 local Compose 운영을 중심으로 하며 public CA, 외부 secret manager, scheduled/remote backup, multi-host production ingress 등을 제공하지 않는다고 경계를 명시한다.
