# miniRT (Ray Scene Tracer)

## 1. 이력서용 프로젝트 설명

C++17로 `.rt` 장면을 해석해 구·평면·유한 원기둥, 그림자와 제한 깊이 반사를 PPM 이미지로 렌더링하는 CPU ray tracer를 구현했습니다.  
유한 도형은 BVH로 가속하고 무한 평면은 선형 목록으로 분리했으며, 선형 기준선과 동일한 교차·동률 규칙을 사용해 결과 일치를 검증했습니다.  
16×16 tile을 worker가 원자적으로 배정받아 pixel을 독점 기록하고, worker 예외는 전체 thread 회수 뒤 전파하며 출력은 임시 파일 완성 후 교체했습니다.  
로컬 회귀 검사 8개를 통과했고, 커밋된 arm64 Release 측정에서는 동일 checksum을 유지하며 400개 구 장면의 primitive test를 약 2억 590만 회에서 90만 회로 줄여 median 26.7배 속도 향상을 기록했습니다.

## 2. 30초 프로젝트 소개

이 프로젝트는 `.rt` 장면을 PPM 이미지로 만드는 C++17 CPU ray tracer입니다. 구·평면·원기둥의 교차와 조명·그림자·반사를 직접 구현하고, 유한 도형은 BVH로 가속했습니다. 선형 탐색을 정확성 기준으로 남겨 BVH와 pixel 결과를 비교했고, 16×16 tile의 각 pixel을 한 worker만 기록해 thread 수와 무관한 결과를 만들었습니다.

## 3. 2분 프로젝트 소개

목표는 장면 파싱부터 교차, 조명, 가속과 병렬 출력까지 직접 구현하면서 최적화가 이미지를 바꾸지 않게 하는 것이었습니다. `.rt` token을 검증해 카메라, 광원, 재질과 도형을 구성하고, renderer가 pixel 중심 광선으로 가장 가까운 교차와 그림자, 확산 조명, 제한 깊이 반사를 계산합니다. 모든 도형의 선형 탐색 비용을 줄이기 위해 bounding box가 있는 구와 원기둥은 BVH에 넣고 무한 평면은 별도 목록으로 유지했습니다. 선형 모드와 BVH 모드에 같은 교차·입력 순서 동률 규칙을 적용해 pixel byte와 checksum을 비교했습니다. 병렬화는 원자적인 tile index를 사용하며, 장면은 읽기 전용이고 각 pixel은 한 worker만 기록합니다. worker 예외는 보존한 뒤 모든 thread를 join하고 전파하며, 출력은 임시 PPM을 완성한 뒤 rename합니다. 로컬 회귀 검사 8개가 모두 통과했습니다. 검사에는 교차, BVH 불변식, 선형/BVH 일치, thread 수별 결정성과 출력 실패가 포함됩니다. 커밋된 측정은 arm64·AppleClang Release, 640×360과 구 400개 조건에서 같은 checksum으로 median 약 26.7배였으며 일반 성능 보장은 아닙니다. 현재 anti-aliasing, gamma correction, texture, mesh와 GPU 렌더링은 지원하지 않습니다.
