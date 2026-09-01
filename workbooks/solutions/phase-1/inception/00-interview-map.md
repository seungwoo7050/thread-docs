# DevThread 개발자 기술면접 워크북 — 마스터 인덱스

## 확인 범위와 사용 원칙

이 워크북은 현재 GPT 프로젝트에 축적된 DevThread Markdown 기록에서 확인된 커밋 제목, diff, 파일 경로, 함수·클래스·컴포넌트 이름만 사용해 작성했다. 실제 원격 저장소나 별도 체크아웃은 사용하지 않았다. 기록에서 커밋 제목과 구현 위치의 연결이 확인되지 않는 내용은 별도 대표 항목으로 만들지 않았고, 확인된 대표 Thread의 연관 위치로만 묶었다.

우선순위 의미는 다음과 같다.

- **S**: 질문과 10~30분 백지 구현 모두 준비해야 하는 핵심 항목
- **A**: 준비 가치가 높고 질문 또는 축소 구현 가능성이 큰 항목
- **B**: 구현보다 설계 의도·개념·trade-off 설명이 중요한 항목
- **C**: 반복 설정, 도구 사용법, 보조 작업에 가까워 별도 준비 효율이 낮은 항목

## 전체 Thread·커밋 선별 결과

`상세 워크북`에 `통합`이라고 표시한 행은 독립 문제를 늘리지 않고 대표 포인트의 질문·백지 구현·자가 검증 조건에 흡수했다.

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread | 상세 워크북 |
|---|---:|---|---|---|---|---|---|---|---|
| B | 01 | `feat(mariadb): 네트워크 DB 서버 설정` | `srcs/requirements/mariadb/Dockerfile`<br>`srcs/requirements/mariadb/conf/50-server.cnf` | DB 리스닝 경계, 문자셋, 소켓·포트 | 기본 네트워크·DB 설정을 설명할 수는 있어야 하지만 값 자체를 외우거나 재작성하는 면접 가치는 낮다. | 중간 | 낮음 | 03, 04, 11 | — |
| B | 01 | `feat(wordpress): PHP-FPM 풀 설정` | `srcs/requirements/wordpress/Dockerfile`<br>`srcs/requirements/wordpress/conf/www.conf` | 프로세스 풀과 요청 처리 경계 | 동적 풀과 용량 제한의 의미는 설명 가치가 있으나 프로젝트 핵심 invariant는 아니다. | 중간 | 낮음 | 03, 05, 11 | — |
| B | 01 | `feat(nginx): TLS 프런트엔드 이미지 추가`<br>`feat(nginx): PHP 요청을 WordPress로 전달` | `srcs/requirements/nginx/Dockerfile`<br>`srcs/requirements/nginx/conf/nginx.conf` | TLS 종료, 정적 파일, FastCGI 경계 | 요청 경로 설명에는 필요하지만 설정 문법 필사보다 종단 검증 관점이 더 중요하다. | 높음 | 낮음 | 03, 11 | A-05에 통합 |
| C | 01 | `feat(compose): 세 서비스 토폴로지 구성` | `srcs/docker-compose.yml` | 기본 Compose 서비스 연결 | 초기 토폴로지 배선은 이후 상태·보안·격리 작업의 출발점이지만 자체로는 boilerplate 비중이 크다. | 낮음 | 낮음 | 02, 03, 11 | — |
| C | 02 | `build(make): 스택 수명주기 명령 추가` | `Makefile` | 명령 진입점 | 단순 래퍼 성격이 강하고 다른 프로젝트로 일반화되는 판단이 적다. | 낮음 | 낮음 | 13 | — |
| A | 02 | `refactor(runtime): Compose 프로젝트 실행 경계 공통화` | `tools/stack_runtime.py`<br>`ComposeProject`<br>`command`<br>`run`<br>`config`<br>`running_services` | 외부 프로세스 실행 경계와 프로젝트 범위 | 명령 구성, 입력 방식 배타성, 타임아웃, JSON 파싱, 오류 변환을 한 경계에서 강제하는 설계가 일반화 가능하다. | 높음 | 높음 | 07, 08, 09, 12, 13 | [A-01](01-state-bootstrap-and-persistence.md#a-01) |
| S | 02 | `refactor(secrets): 비밀 파일 로딩 경계 공통화` | `tools/stack_runtime.py`<br>`_private_directory`<br>`read_private_secret`<br>`secret_source_paths`<br>`load_secret_values` | TOCTOU를 줄이는 안전한 비밀 파일 읽기 | 심볼릭 링크, 소유권, 모드, 하드링크, 파일 종류, 크기, 형식, 중복 경로를 파일 디스크립터 기준으로 검증한다. 보안 면접과 직접 구현 가치가 모두 높다. | 높음 | 높음 | 06, 09, 12 | [S-03](02-secrets-locking-and-rotation.md#s-03) |
| A | 02 | `refactor(runtime): 프로젝트 관리 작업 잠금 공통화` | `tools/stack_runtime.py`<br>`project_operation_lock` | 파일 잠금, 소유권, 프로세스 간 직렬화 | 백업·복원·회전·시작이 동일 프로젝트 상태를 동시에 바꾸지 못하게 하는 범용 동시성 경계다. | 높음 | 높음 | 07, 08, 09 | [A-03](02-secrets-locking-and-rotation.md#a-03) |
| A | 02 | `fix(init): 중단된 단계별 초기화를 수렴` | `tools/start_stack.py`<br>`run_action`<br>`start_database`<br>`start_application`<br>`run_bootstrap`<br>`remove_stale_bootstrap`<br>`pause_arguments` | 단계별 오케스트레이션, 소유권 확인, readiness | DB와 애플리케이션 초기화를 분리하고, 테스트 중단 지점을 주입하며, bootstrap 컨테이너 소유권을 확인하는 제어 흐름이 중요하다. | 높음 | 높음 | 03, 04, 05, 06, 13 | [A-01](01-state-bootstrap-and-persistence.md#a-01)에 통합 |
| C | 02 | `build: improve Makefile and separate functional stack validation` | `Makefile`<br>`tests/validate_stack.py` | 개발 명령 UX와 검증 모드 분리 | 운영 편의성은 높지만 핵심 면접 문제로 만들기에는 래퍼·정책 코드 비중이 크다. | 낮음 | 낮음 | 13 | — |
| A | 03 | `fix(init): 중단된 단계별 초기화를 수렴` | `srcs/docker-compose.yml`<br>`mariadb_data`<br>`wordpress_data`<br>`wordpress_config`<br>완료 표식 기반 healthcheck | 영속 상태 분할과 완료 상태 공개 | 볼륨을 데이터·콘텐츠·설정으로 나누고, 단순 프로세스 생존이 아니라 초기화 완료 표식까지 readiness에 포함한다. | 높음 | 중간 | 02, 04, 05 | S-01, S-02, A-02에 통합 |
| A | 03 | `test(e2e): HTTPS와 MariaDB를 잇는 WordPress 데이터 검증` | `tests/runtime_stack.py`<br>`RuntimeStack.fetch`<br>`RuntimeStack.verify_e2e` | HTTPS→nginx→PHP-FPM→WordPress→MariaDB 종단 검증 | 단순 healthcheck가 놓치는 서비스 간 계약을 실제 쓰기·읽기와 DB 확인으로 검증한다. 포트 충돌 재시도도 포함한다. | 높음 | 높음 | 01, 05, 11, 13 | [A-05](04-network-runtime-and-supply-chain.md#a-05) |
| A | 03 | `test(persistence): 재시작·재생성 뒤 상태 보존 검증` | `tests/runtime_stack.py`<br>`RuntimeStack.project_volumes`<br>`RuntimeStack._verify_persistent_values`<br>`RuntimeStack.verify_persistence` | 재시작과 재생성의 의미 구분, 영속성 invariant | 컨테이너 재시작과 `down/up` 재생성 뒤 DB·옵션·업로드·볼륨 식별자가 보존되는지를 구분해 검증한다. | 높음 | 높음 | 04, 05, 07, 08 | [A-02](01-state-bootstrap-and-persistence.md#a-02) |
| S | 04 | `fix(init): 중단된 단계별 초기화를 수렴` | `srcs/requirements/mariadb/tools/docker-entrypoint.sh`<br>`start_temporary_server`<br>`stop_temporary_server`<br>`write_option_file`<br>`verify_database`<br>`runtime`<br>`bootstrap` | crash-safe staged initialization과 원자적 상태 공개 | staging에서 완성·검증한 뒤 marker, `sync`, 동일 파일시스템 `mv`, 부모 디렉터리 `sync` 순서로 공개한다. 중단 복구와 durability를 함께 묻기 좋다. | 높음 | 높음 | 02, 03, 06, 13 | [S-01](01-state-bootstrap-and-persistence.md#s-01) |
| S | 05 | `fix(init): 중단된 단계별 초기화를 수렴` | `srcs/requirements/wordpress/tools/docker-entrypoint.sh`<br>`prepare_config_location`<br>`publish_config_link`<br>`write_wordpress_config`<br>`validate_wordpress_config`<br>`converge_wordpress_config`<br>`bootstrap` | 멱등적 reconciliation, 설정 격리, 상태 분류 | 신규·기존·부분 초기화·레거시 설정·자격증명 불일치 상태를 구분해 안전한 행동을 선택한다. 무조건 덮어쓰지 않는 판단이 핵심이다. | 높음 | 높음 | 02, 03, 06, 09, 10, 13 | [S-02](01-state-bootstrap-and-persistence.md#s-02) |
| A | 05 | `build(wordpress): WordPress 산출물을 고정해 게시` | `srcs/requirements/wordpress/Dockerfile`<br>`srcs/requirements/wordpress/tools/docker-entrypoint.sh`<br>`install_core_files`<br>`install_content_files` | 검증된 빌드 산출물의 런타임 게시 | 실행 시 네트워크 다운로드 대신 이미지에 검증된 산출물을 포함하고, 체크섬·경로·심볼릭 링크를 검증해 원자적으로 게시한다. | 높음 | 중간 | 10 | A-07에 통합 |
| C | 06 | `feat(secrets): 비밀번호를 비밀 파일에서 로드` | `.env.example`<br>`srcs/docker-compose.yml` | 환경 변수에서 파일 기반 비밀로 이동 | 보안 방향은 옳지만 이후 one-off bootstrap 표준 입력 방식으로 경계가 더 강화되어 최종 대표 설계로 쓰기 어렵다. | 중간 | 낮음 | 02, 04, 05 | — |
| A | 06 | `refactor(secrets): 비밀 파일 로딩 경계 공통화` | `tools/stack_runtime.py`<br>`read_private_secret` 계열 | 보안 검증 로직 중복 제거 | 같은 파일 안전성 검사를 백업·회전·진단에서 재사용하게 만든다. 독립 문제보다 S-03의 설계 확장으로 보는 편이 낫다. | 높음 | 중간 | 02, 09, 12 | S-03에 통합 |
| A | 06 | `feat(secrets): 런타임 비밀 노출 경계 검사` | `tools/rotate_secrets.py`<br>`verify_runtime_secret_boundary`<br>`tests/runtime_stack.py`<br>`assert_runtime_secret_boundary` | 비밀값의 수명과 런타임 비노출 | 마운트·환경 변수·검사 결과에서 비밀값이 런타임 서비스에 남지 않았는지 증명한다. "파일을 썼다"가 아니라 노출 경계를 검증한다. | 높음 | 높음 | 02, 04, 05, 09, 12 | [A-04](02-secrets-locking-and-rotation.md#a-04) |
| A | 07 | `feat(backup): 백업 무결성과 비공개 파일 I/O 정의` | `tools/stack_backup.py`<br>`sha256_stream`<br>`sha256`<br>`fsync_directory`<br>`write_private`<br>`private_output` | 스트리밍 해시, 비공개 출력, durability | 큰 파일을 메모리에 올리지 않고 처리하고, 0600·배타 생성·파일과 디렉터리 fsync로 결과의 비공개성과 지속성을 보장한다. | 높음 | 높음 | 08, 12 | S-05에 통합 |
| A | 07 | `feat(backup): 관리 작업 신호와 테스트 중단 경계 추가` | `tools/stack_backup.py`<br>`operation_signal_handlers`<br>`pause_for_test` | 신호 처리와 결정적 실패 주입 | 운영 코드의 정리 경로와 테스트의 중단 타이밍을 같은 명시적 경계로 만든다. | 높음 | 중간 | 08, 09, 13 | S-05, S-09, A-09에 통합 |
| C | 07 | `feat(backup): 백업용 Compose 실행 어댑터 추가` | `tools/stack_backup.py`의 초기 `ComposeProject` | 외부 실행 어댑터 | 이후 `tools/stack_runtime.py` 경계로 공통화되어 별도 면접 항목으로 중복 준비할 필요가 없다. | 낮음 | 낮음 | 02 | A-01에 통합 |
| A | 07 | `feat(backup): WordPress 아카이브 입력 검증` | `tools/stack_backup.py`<br>`validate_archive_stream`<br>`validate_archive` | tar 경로 순회·링크·중복 방어 | 복원 전에 적대적 아카이브 메타데이터를 검증하는 일반화 가능한 입력 경계다. | 높음 | 높음 | 08 | S-06에 통합 |
| A | 07 | `feat(backup): 프로젝트별 백업 작업 잠금 적용` | `tools/stack_backup.py`의 초기 `project_operation_lock` | 프로젝트별 상호 배제 | 이후 공통 잠금으로 이동하므로 대표 구현은 Thread 02가 적절하다. | 높음 | 중간 | 02, 08, 09 | A-03에 통합 |
| S | 07 | `feat(backup): DB 덤프와 WordPress 볼륨 수집`<br>`feat(backup): 백업 출력 경로를 안전하게 예약`<br>`feat(backup): 백업 세트를 원자적으로 게시` | `tools/stack_backup.py`<br>`database_dump`<br>`wordpress_archive`<br>`normalize_backup_output`<br>`same_directory`<br>`create_backup` | 여러 자원의 일관된 스냅샷과 원자적 세트 게시 | 쓰기 서비스 정지, DB 일관 덤프, 파일 아카이브, manifest·체크섬, 임시 형제 디렉터리와 `os.replace`, 실패 시 서비스 복구를 하나의 상태 전이로 다룬다. | 높음 | 높음 | 03, 08, 09, 13 | [S-05](03-backup-and-restore.md#s-05) |
| S | 08 | `feat(restore): 백업 입력의 형식과 체크섬 검증` | `tools/stack_backup.py`<br>`VerifiedBackup`<br>`open_regular_file`<br>`load_and_verify_backup` | 적대적 백업 디렉터리와 파일 검증 | 디렉터리·파일을 `O_NOFOLLOW`로 열고 소유권·모드·링크 수·정확한 파일 집합·공유 잠금·크기·체크섬·tar 구조를 검증한다. | 높음 | 높음 | 02, 07, 12 | [S-06](03-backup-and-restore.md#s-06) |
| S | 08 | `feat(restore): 대상 프로젝트 자원 충돌 사전 차단` | `tools/stack_backup.py`<br>`rendered_resource_names`<br>`existing_named_resources`<br>`expected_container_names`<br>`existing_named_containers`<br>`ensure_fresh_project` | 사전 조건과 충돌 원자성 | 복원을 시작하기 전에 컨테이너·볼륨·네트워크 이름 충돌과 기존 프로젝트 상태를 모두 거부해 부분 자원 생성을 막는다. | 높음 | 높음 | 02, 11, 13 | [S-07](03-backup-and-restore.md#s-07) |
| S | 08 | `feat(restore): DB와 WordPress 데이터를 새 볼륨에 주입` | `tools/stack_backup.py`<br>`restore_database`<br>`restore_wordpress` | 스트리밍 복원과 비밀 전달 경계 | 대용량 dump·archive를 스트리밍하고, DB 비밀번호를 인자나 환경이 아닌 표준 입력과 임시 option file로 전달하며, 비어 있는 대상만 채운다. | 높음 | 높음 | 06, 07, 09 | [S-08](03-backup-and-restore.md#s-08) |
| S | 08 | `feat(restore): 실패한 복원 자원을 정리하고 롤백` | `tools/stack_backup.py`<br>`cleanup_failed_restore`<br>복원 오케스트레이션 | 부분 성공 추적, 범위 제한 cleanup, 신호 복구 | 실패 지점까지 생성한 자원만 추적해 정리하고, 원래 오류와 정리 오류를 모두 보존한다. broad prune을 쓰지 않는 점이 중요하다. | 높음 | 높음 | 07, 09, 13 | [S-09](03-backup-and-restore.md#s-09) |
| A | 09 | `feat(secrets): WordPress 설정과 사용자 비밀번호 교체` | `tools/rotate_secrets.py`<br>`set_wordpress_db_config`<br>`set_wordpress_user`<br>`atomic_secret_write` | 여러 저장소의 자격증명 동기화 | DB, WordPress 설정, 사용자 계정, 호스트 파일 사이의 순서와 원자적 파일 교체가 중요하나 대표 문제는 전체 회전 상태 전이다. | 높음 | 높음 | 05, 06 | S-04에 통합 |
| S | 09 | `feat(secrets): 회전 실패 시 기존 자격증명 복구`<br>`feat(secrets): 스택 자격증명 회전 절차 연결` | `tools/rotate_secrets.py`<br>`find_root_password`<br>`rollback_rotation`<br>`verify_rotation`<br>`_rotate`<br>`rotate` | 분산 트랜잭션에 가까운 회전과 보상 복구 | DB 계정·설정·사용자·호스트 파일은 한 트랜잭션으로 묶이지 않는다. 단계별 검증과 보상, 사용 가능한 root 자격증명 탐색, 신호 지연 처리가 핵심이다. | 높음 | 높음 | 02, 05, 06, 07, 13 | [S-04](02-secrets-locking-and-rotation.md#s-04) |
| A | 09 | `test(secrets): 회전 롤백과 재시도 검증` | `tests/runtime_stack.py`<br>`RuntimeStack.verify_secret_rotation` | 폐기 자격증명 거부, 롤백·재시도, 비밀 출력 방지 | 정상 경로만이 아니라 각 실패 단계, 롤백 중 추가 신호, 재시도, 이전 비밀번호 거부까지 검증한다. | 높음 | 중간 | 13 | S-04, A-09에 통합 |
| A | 10 | `build(wordpress): WordPress 산출물을 고정해 게시`<br>`test(supply-chain): 불변 image 입력 검증` | 세 서비스 `Dockerfile`<br>`srcs/requirements/wordpress/tools/docker-entrypoint.sh`<br>`tests/validate_stack.py`<br>`tests/runtime_stack.py` | digest·snapshot·checksum과 런타임 버전 검증 | 이미지 태그, 패키지 저장소, 외부 산출물의 변동성을 줄이고 실제 실행 버전까지 확인한다. 빌드 재현성과 공급망 경계를 함께 설명할 수 있다. | 높음 | 높음 | 05, 13 | [A-07](04-network-runtime-and-supply-chain.md#a-07) |
| A | 11 | `feat(runtime): 프로젝트·이미지·포트·URL 격리`<br>`feat(network): DB 트래픽을 내부 backend로 격리`<br>`feat(runtime): 서비스 자원과 종료 한계 적용` | `.env.example`<br>`srcs/docker-compose.yml`<br>`srcs/requirements/wordpress/tools/docker-entrypoint.sh`<br>`tests/runtime_stack.py` | 이름·포트·URL·네트워크·자원·종료 격리 | 여러 프로젝트 동시 실행, DB 내부망, 자원 상한, graceful stop, 권한 상승 차단을 하나의 런타임 경계로 설명할 수 있다. | 높음 | 높음 | 01, 02, 03, 13 | [A-06](04-network-runtime-and-supply-chain.md#a-06) |
| B | 11 | `feat(nginx): 접근·오류 로그를 컨테이너 스트림에 게시` | `srcs/requirements/nginx/conf/nginx.conf` | 로그 수집 경계 | 운영상 필수지만 stdout/stderr 연결 자체는 구현 난도가 낮고 진단 도구 문제에서 함께 설명하면 충분하다. | 중간 | 낮음 | 12, 13 | A-08에 통합 |
| B | 11 | `fix(make): 볼륨 삭제 전에 확인을 요구` | `Makefile`<br>`DESTROY_CONFIRM` | 파괴적 작업의 명시적 확인 | 안전 가드의 의도는 중요하지만 간단한 조건문이므로 구현 문제보다 운영 판단 설명에 적합하다. | 중간 | 낮음 | 12 | — |
| S | 12 | `feat(diagnostics): Compose 비밀값과 민감 항목 마스킹` | `tools/diagnose_stack.py`<br>`rendered_compose_config`<br>`secret_values`<br>`redact` | fail-closed 비밀 마스킹 | 비밀값뿐 아니라 원본·정규화 경로와 민감한 assignment를 가리고, 가릴 값을 읽지 못하면 진단 생성을 중단한다. | 높음 | 높음 | 02, 06, 11, 13 | [S-10](05-diagnostics-and-verification.md#s-10) |
| A | 12 | `feat(diagnostics): 컨테이너 런타임 상태 수집` | `tools/diagnose_stack.py`<br>`run`<br>`container_state`<br>고정 출력 파일 집합 | 제한된 증거 수집과 비공개 게시 | 명령 실패도 증거로 남기되 출력 경로 덮어쓰기·symlink를 거부하고, 0700 디렉터리·0600 파일·고정 allowlist를 유지한다. | 높음 | 높음 | 11, 13 | [A-08](05-diagnostics-and-verification.md#a-08) |
| A | 12 | `test(runtime): 프로세스·비밀값·정리 제어 흐름 강화` | `tests/runtime_stack.py`<br>`RuntimeStack.verify_operations`<br>`replace_private` | 운영 경계의 부정 검증 | 읽을 수 없는 비밀, 기존 출력, dangling symlink, 로그 속 비밀, cleanup 실패를 의도적으로 검증한다. | 높음 | 중간 | 11, 13 | S-10, A-08, A-09에 통합 |
| B | 13 | `test(compose): 렌더링된 Compose 설정 검사` | `tests/validate_stack.py` | 소스 텍스트보다 렌더링 모델 검증 | Compose 치환·병합 결과를 보는 관점은 중요하지만 독립 구현 문제보다는 정적 계약 항목에 포함하는 편이 낫다. | 높음 | 중간 | 02, 11 | A-11에 통합 |
| A | 13 | `test(bootstrap): 격리된 런타임 하네스 추가` | `tests/runtime_stack.py`<br>`RuntimeStack` | 독립 프로젝트·임시 비밀·동적 포트·실패 주입 | 실제 Docker 자원을 쓰는 테스트에서 서로 간섭하지 않고, 중단 시점을 결정적으로 만들며, 항상 정리하는 구조를 묻기 좋다. | 높음 | 높음 | 03, 04, 05, 07, 08, 09, 12 | [A-09](05-diagnostics-and-verification.md#a-09) |
| B | 13 | `build(compose): 엄격한 설정 검사 추가` | `Makefile`<br>`config-strict` | CI에서의 strict parse | 유용한 gate이지만 도구 호출 자체보다 어떤 계약을 검사할지가 더 중요하다. | 중간 | 낮음 | 02 | A-11에 통합 |
| A | 13 | `ci(stack): 정적·런타임·복구 검증 자동화` | `.github/workflows/container-stack.yml`<br>`tools/cleanup_test_resources.py`<br>`tools/verify_stack.py` | 범위 제한 정리, 실패 증거 보존, 직렬 시나리오 | 테스트가 실패해도 기록된 프로젝트만 정리하고, 정리 실패 증거를 보존하며, 실패 시 allowlist만 업로드한다. | 높음 | 높음 | 07, 08, 09, 12 | [A-10](05-diagnostics-and-verification.md#a-10) |
| B | 13 | `ci(stack): 커밋 범위 공백 검사 도구 추가` | `tools/check_commit_range.py`<br>`OBJECT_ID`<br>`fallback_base`<br>`select_base`<br>`main` | shallow history와 기준 커밋 선택 | 방어적 Git 처리 예시지만 프로젝트의 상태·복구 문제보다 우선순위가 낮다. | 중간 | 중간 | — | — |
| A | 13 | `test(ci): workflow 검증 계약 추가` | `tests/validate_stack.py`<br>`ast` 기반 검사<br>`validate_ci`<br>`require_subprocess_timeouts`<br>`validate_no_credential_arguments` | 구성 코드의 구조적 회귀 검사 | 문자열 포함 검사만으로 부족한 subprocess 타임아웃, 자격증명 인자 노출, 시나리오 순서, action pinning 등을 AST와 구조 검사로 보강한다. | 높음 | 높음 | 02, 06, 10, 11, 12 | [A-11](05-diagnostics-and-verification.md#a-11) |
| C | 13 | `build: improve Makefile and separate functional stack validation`<br>`ci: harden container stack validation` | `Makefile`<br>`.github/workflows/container-stack.yml`<br>`tests/validate_stack.py` | 프로젝트 정책 검사와 기능 검사 분리 | 저장소 정책에 특화된 부분이 많다. 면접에서는 기능 검증과 문서·레이아웃 정책을 분리한 이유만 설명하면 충분하다. | 중간 | 낮음 | 02 | — |

## 상세 워크북 구성

| 파일 | 역할 | 포함 포인트 |
|---|---|---|
| [01-state-bootstrap-and-persistence.md](01-state-bootstrap-and-persistence.md) | 중단 가능한 초기화, 상태 수렴, 영속성 invariant | S-01, S-02, A-01, A-02 |
| [02-secrets-locking-and-rotation.md](02-secrets-locking-and-rotation.md) | 안전한 파일 I/O, 관리 작업 직렬화, 런타임 비밀 경계, 보상 복구 | S-03, A-03, A-04, S-04 |
| [03-backup-and-restore.md](03-backup-and-restore.md) | 일관된 백업 세트, 적대적 입력 검증, 새 프로젝트 복원, 실패 롤백 | S-05, S-06, S-07, S-08, S-09 |
| [04-network-runtime-and-supply-chain.md](04-network-runtime-and-supply-chain.md) | 종단 요청 흐름, 프로젝트·네트워크 격리, 재현 가능한 이미지 입력 | A-05, A-06, A-07 |
| [05-diagnostics-and-verification.md](05-diagnostics-and-verification.md) | 비밀값 없는 증거 수집, 격리 하네스, 범위 제한 정리, 정적 계약 검사 | S-10, A-08, A-09, A-10, A-11 |

## 대표 포인트와 연관 Thread 관계

| 대표 포인트 | 대표 Thread | 함께 묶은 Thread | 통합 기준 |
|---|---:|---|---|
| S-01 MariaDB crash-safe 상태 공개 | 04 | 02, 03, 06, 13 | 오케스트레이션, 완료 표식, 비밀 전달, SIGKILL 복구를 DB 초기화 invariant 하나로 통합 |
| S-02 WordPress reconciliation | 05 | 02, 03, 06, 09, 10, 13 | 설정 격리, 레거시 이동, 계정 검증, 산출물 게시, 중단 재실행을 상태 수렴 문제로 통합 |
| S-03 안전한 비밀 파일 읽기 | 02 | 06, 09, 12 | 모든 호스트 비밀 입력이 공유하는 파일시스템 보안 경계를 대표 |
| S-04 자격증명 회전과 보상 복구 | 09 | 02, 05, 06, 07, 13 | 여러 저장소의 상태 변경, 잠금, 신호, 실패 주입, 재검증을 한 보상 트랜잭션으로 통합 |
| S-05 일관된 백업 세트 | 07 | 02, 03, 08, 13 | writer 정지, DB·파일 수집, private I/O, manifest, 원자적 publish, 서비스 복구를 통합 |
| S-06 적대적 백업 입력 검증 | 08 | 02, 07, 12 | 안전한 open, 정확한 파일 집합, 잠금·체크섬, tar path/type 검증을 한 입력 경계로 통합 |
| S-07 fresh-target restore | 08 | 02, 11, 13 | 렌더링 이름, 기존 자원 충돌, 생성 전 사전 조건을 통합 |
| S-08 보안 스트리밍 복원 | 08 | 06, 07, 09 | 대용량 전송, 비밀 전달, 빈 목적지 제약을 통합 |
| S-09 복원 실패 롤백 | 08 | 07, 09, 13 | 부분 성공 ledger, 신호, best-effort 정리, 오류 보존을 통합 |
| S-10 비밀값 없는 진단 | 12 | 02, 06, 11, 13 | 마스킹 대상 수집, fail-closed, 로그·모델·경로 누출 방지를 통합 |
| A-01 프로젝트 범위 오케스트레이션 | 02 | 04, 05, 07, 08, 09 | 외부 프로세스 실행·타임아웃·입력·소유권 경계를 한 어댑터로 통합 |
| A-02 영속성 invariant | 03 | 04, 05, 07, 08 | 재시작과 재생성 후 데이터·파일·볼륨 식별자 보존을 통합 |
| A-03 관리 작업 잠금 | 02 | 07, 08, 09 | 프로젝트 단위 교차 프로세스 상호 배제를 대표 |
| A-04 런타임 비밀 비노출 | 06 | 02, 04, 05, 09, 12 | bootstrap에만 비밀을 전달하고 장기 실행 컨테이너에서 제거하는 lifecycle을 대표 |
| A-05 종단 요청 검증 | 03 | 01, 05, 11, 13 | TLS 진입부터 DB 지속까지 실제 데이터로 검증하는 시나리오를 대표 |
| A-06 런타임 격리 정책 | 11 | 01, 02, 03, 13 | 프로젝트 이름, 이미지, 포트, URL, 네트워크, 자원, 종료·권한 정책을 통합 |
| A-07 재현 가능한 공급망 | 10 | 05, 13 | digest, immutable snapshot, checksum, 런타임 최소 버전 검증을 통합 |
| A-08 비공개 증거 게시 | 12 | 11, 13 | 고정 allowlist, private mode, overwrite·symlink 거부, 명령 실패 보존을 통합 |
| A-09 격리 하네스와 실패 주입 | 13 | 03, 04, 05, 07, 08, 09, 12 | 동적 자원, 단계 pause, SIGKILL·신호, finally cleanup을 통합 |
| A-10 범위 제한 cleanup·증거 lifecycle | 13 | 07, 08, 09, 12 | 기록된 프로젝트만 정리하고 broad prune을 금지하며 실패 증거를 보존 |
| A-11 구조적 회귀 검사 | 13 | 02, 06, 10, 11, 12 | 렌더링 모델, AST, workflow allowlist와 순서 검증을 통합 |

## S/A 완전성 검증

| ID | 우선순위 | 상태 | 상세 위치 | 명시적으로 통합한 항목 |
|---|---|---|---|---|
| S-01 | S | 독립 상세 항목 작성됨 | `01-state-bootstrap-and-persistence.md` | Thread 02/03의 단계 오케스트레이션·완료 표식, Thread 13의 DB 중단 복구 |
| S-02 | S | 독립 상세 항목 작성됨 | `01-state-bootstrap-and-persistence.md` | Thread 03의 설정 볼륨, Thread 05의 고정 산출물 게시 일부, Thread 13의 앱 중단 복구 |
| A-01 | A | 독립 상세 항목 작성됨 | `01-state-bootstrap-and-persistence.md` | Thread 02 `fix(init)`의 `run_action`, stale bootstrap 소유권 확인 |
| A-02 | A | 독립 상세 항목 작성됨 | `01-state-bootstrap-and-persistence.md` | Thread 03 볼륨 분할과 재시작·재생성 구분 |
| S-03 | S | 독립 상세 항목 작성됨 | `02-secrets-locking-and-rotation.md` | Thread 06/09/12의 공통 비밀 파일 입력 경계 |
| A-03 | A | 독립 상세 항목 작성됨 | `02-secrets-locking-and-rotation.md` | Thread 07/08/09의 관리 작업 직렬화 |
| A-04 | A | 독립 상세 항목 작성됨 | `02-secrets-locking-and-rotation.md` | Thread 04/05 one-off bootstrap 표준 입력, Thread 12 누출 검증 |
| S-04 | S | 독립 상세 항목 작성됨 | `02-secrets-locking-and-rotation.md` | WordPress 교체, rollback, signal deferral, 재시도 테스트 |
| S-05 | S | 독립 상세 항목 작성됨 | `03-backup-and-restore.md` | private I/O, 신호 처리, 출력 예약, atomic publish, operation lock |
| S-06 | S | 독립 상세 항목 작성됨 | `03-backup-and-restore.md` | Thread 07 tar 검증과 Thread 08 디렉터리·파일·manifest 검증 |
| S-07 | S | 독립 상세 항목 작성됨 | `03-backup-and-restore.md` | 컨테이너·볼륨·네트워크 충돌과 fresh-project preflight |
| S-08 | S | 독립 상세 항목 작성됨 | `03-backup-and-restore.md` | DB 비밀 전달, 큰 dump·archive 스트리밍, 빈 목적지 제약 |
| S-09 | S | 독립 상세 항목 작성됨 | `03-backup-and-restore.md` | 신호 중단, 부분 자원 ledger, cleanup 오류 보존 |
| A-05 | A | 독립 상세 항목 작성됨 | `04-network-runtime-and-supply-chain.md` | Thread 01 TLS/FastCGI/DB 경로와 Thread 03 port retry |
| A-06 | A | 독립 상세 항목 작성됨 | `04-network-runtime-and-supply-chain.md` | 프로젝트·이미지·포트·URL·frontend/backend·자원·종료 정책 |
| A-07 | A | 독립 상세 항목 작성됨 | `04-network-runtime-and-supply-chain.md` | Thread 05 런타임 게시와 Thread 10 정적·런타임 공급망 검사 |
| S-10 | S | 독립 상세 항목 작성됨 | `05-diagnostics-and-verification.md` | secret path/value 수집, assignment masking, unreadable-secret fail-closed |
| A-08 | A | 독립 상세 항목 작성됨 | `05-diagnostics-and-verification.md` | stdout/stderr 로그, 고정 파일 집합, private mode, overwrite·symlink 거부 |
| A-09 | A | 독립 상세 항목 작성됨 | `05-diagnostics-and-verification.md` | 모든 runtime scenario의 isolation·pause·signal·finally cleanup |
| A-10 | A | 독립 상세 항목 작성됨 | `05-diagnostics-and-verification.md` | 프로젝트 record, scoped label 검증, cleanup report, 실패 증거 보존 |
| A-11 | A | 독립 상세 항목 작성됨 | `05-diagnostics-and-verification.md` | 렌더링 Compose, AST subprocess 검사, workflow action·artifact allowlist |

**검증 결과:** S 10개와 A 11개가 모두 독립 상세 항목으로 작성되었고, 대표 항목에 흡수한 보조 커밋은 위 표와 각 상세 문서의 `원본 확인 위치`에 명시했다. 미배정 S/A 항목은 없다.

## 백지 구현 우선순위

1. **S-03** 안전한 비밀 파일 읽기 — 작은 코드로 파일시스템 보안·TOCTOU·resource cleanup을 함께 확인할 수 있다.
2. **S-01** staged directory 원자 게시 — 상태 invariant, `fsync`, rename, crash consistency를 직접 구현하기 좋다.
3. **S-06** tar·backup 입력 검증 — 경로 정규화, 중복, 파일 종류, 체크섬, fail-closed 판단을 확인한다.
4. **S-04** 보상 가능한 자격증명 회전 coordinator — 단일 트랜잭션이 없는 시스템의 실패 순서와 rollback을 확인한다.
5. **S-09** 부분 복원 cleanup ledger — 생성 자원 범위, 정리 오류 집계, 신호 처리를 확인한다.
6. **S-05** 일관된 백업 coordinator — writer 정지부터 원자 publish·서비스 복구까지 상태 전이를 확인한다.
7. **S-02** WordPress 설정 상태 분류기 — 신규·레거시·완료·드리프트 상태에서 안전한 행동을 선택하게 한다.
8. **S-07** fresh-project 충돌 검사 — expected/observed 집합과 side effect 이전 preflight를 구현한다.
9. **S-08** 비밀 prefix를 포함한 bounded streaming — 메모리 상한, partial write, 비밀 노출 방지를 확인한다.
10. **S-10** fail-closed redaction — 정확 일치 값과 구조적 민감 assignment를 함께 마스킹한다.
11. **A-03** project operation lock — 프로세스 간 배타성과 안전한 lock path 검증을 확인한다.
12. **A-11** AST 기반 subprocess timeout 검사 — 구성·운영 코드의 구조적 계약 검사를 구현한다.
13. **A-02** persistence snapshot 비교기 — 재시작과 재생성 후 보존 invariant를 명시하게 한다.
14. **A-09** scenario runner와 failure injection protocol — 테스트 lifecycle과 cleanup을 확인한다.
15. **A-01** 외부 프로세스 실행 어댑터 — command boundary, timeout, 입력 배타성, 오류 변환을 구현한다.
16. **A-08** 비공개 evidence writer — 출력 예약, 모드, allowlist, fsync, overwrite 거부를 확인한다.
17. **A-04** 컨테이너 inspect 결과의 비밀 노출 검사 — mount/env/value 누출을 구조적으로 판별한다.
18. **A-05** 포트 충돌을 포함한 E2E 실행 helper — 자원 경쟁과 실제 요청 경로 검증을 묻는다.
19. **A-06** 렌더링된 Compose 격리 정책 validator — 이름·포트·네트워크·자원 정책을 검증한다.
20. **A-07** 공급망 manifest verifier — 경로·digest·버전 계약을 검증한다.
21. **A-10** 기록 기반 selective cleanup plan — broad prune 없이 소유 자원만 정리한다.

## 설명 연습 우선순위

1. **S-04** 왜 자격증명 회전은 DB 트랜잭션 하나로 해결되지 않으며 어떤 순서로 보상해야 하는가
2. **S-01** marker, 데이터 검증, atomic rename, 파일·디렉터리 `fsync`가 각각 해결하는 실패가 무엇인가
3. **S-05** DB dump와 WordPress 파일을 같은 시점의 논리적 세트로 만드는 방법과 downtime trade-off
4. **S-02** idempotence와 reconciliation의 차이, 완료 상태에서 자동 복구보다 fail-closed를 선택한 이유
5. **S-06** `Path.resolve()`만으로 안전한 파일 입력이 되지 않는 이유와 descriptor 기준 검증
6. **S-09** 원래 실패와 cleanup 실패를 동시에 보존하면서도 작업 범위를 넘지 않는 방법
7. **S-10** 마스킹할 비밀을 읽을 수 없을 때 일부 진단이라도 남기지 않고 중단하는 이유
8. **A-04** bootstrap secret과 장기 실행 runtime secret의 lifecycle을 분리한 이유
9. **A-06** 프로젝트 격리, 내부 backend, resource limit, graceful stop을 서로 독립된 정책으로 보는 이유
10. **A-09** 임의 sleep보다 stage-ready marker가 실패 복구 테스트에 적합한 이유
11. **A-11** 문자열 검사와 AST·렌더링 모델 검사의 장단점
12. **A-02** 컨테이너 restart와 recreation을 별도 시나리오로 검증해야 하는 이유
13. **A-07** 버전 pin만으로 공급망 재현성이 충분하지 않은 이유
14. **A-05** health endpoint만 통과한 시스템을 종단 정상으로 볼 수 없는 이유
15. **A-10** cleanup 실패를 숨기지 않고 증거로 남겨야 하는 이유

## 한 문제로 통합한 Thread 묶음

- **초기화 상태 머신:** Thread 02 + 03 + 04 + 05 + 06 + 13 → S-01, S-02, A-01, A-02, A-09
- **호스트 비밀과 런타임 비노출:** Thread 02 + 06 + 09 + 12 → S-03, A-04, S-04, S-10
- **프로젝트 단위 관리 작업 직렬화:** Thread 02 + 07 + 08 + 09 → A-03
- **백업·복원 트랜잭션:** Thread 03 + 07 + 08 + 13 → S-05, S-06, S-07, S-08, S-09
- **종단 서비스 경계와 런타임 격리:** Thread 01 + 03 + 05 + 11 + 13 → A-05, A-06
- **고정 산출물과 공급망 검증:** Thread 05 + 10 + 13 → A-07
- **비공개 진단과 실패 증거 lifecycle:** Thread 02 + 06 + 11 + 12 + 13 → S-10, A-08, A-10
- **격리 검증과 구조적 CI gate:** Thread 03 + 04 + 05 + 07 + 08 + 09 + 11 + 12 + 13 → A-09, A-10, A-11
