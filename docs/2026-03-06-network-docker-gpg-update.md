# 2026-03-06 작업 정리

## 목표
- n4200 간헐 네트워크 이슈 상황에서 공통 플레이북 안정성 확보
- Docker GPG 키 배포 방식을 네트워크 상황에 따라 유연하게 동작하도록 개선
- Docker APT 저장소 충돌로 인한 `Signed-By` 오류 제거

## 주요 변경 사항

### 1) Docker GPG 키 다중 fallback 로직 추가
- 대상 파일: `roles/common/tasks/docker.yml`
- 동작 순서:
  1. `/usr/share/keyrings/docker-archive-keyring.asc`가 이미 있으면 재사용
  2. 없으면 대상 노드에서 직접 `https://download.docker.com/linux/ubuntu/gpg` 다운로드 시도
  3. 직접 다운로드 실패 시 컨트롤 호스트(`playbook_dir/.ansible/cache/docker.asc`)에서 받아 대상 노드로 복사

### 2) Docker 저장소 충돌 정리 강화
- 대상 파일: `roles/common/tasks/docker.yml`, `roles/common/tasks/base.yml`
- `/etc/apt/sources.list.d` 내 `*docker*.list` 패턴 파일을 선제 정리해
  `Conflicting values set for option Signed-By` 문제를 방지
- `sources.list` 내 Docker 관련 라인도 정리 후 표준 `docker.list`를 재구성

### 3) APT 안정화 로직 추가
- 대상 파일: `roles/common/tasks/base.yml`
- `/etc/apt/apt.conf.d/99network-retry` 생성:
  - `Acquire::Retries "5";`
  - `Acquire::http::Timeout "20";`
  - `Acquire::https::Timeout "20";`
  - `Acquire::ForceIPv4 "true";`
- `apt-get update`를 재시도 로직으로 수행해 순간 네트워크 흔들림 내성 강화

### 4) 기존 불필요 Wi-Fi 보정 코드 제거
- 대상 파일: `roles/common/tasks/base.yml`
- 이전에 추가했던 n4200 전용 Wi-Fi power save 강제 비활성화 서비스 관련 태스크 제거

### 5) WireGuard 변수 안정화
- 대상 파일: `roles/wireguard/tasks/wireguard_setup.yml`
- `public_interface`, `address`, `hub_info`, `all_peers` 기본값 처리 정리로 템플릿/변수 해석 오류 완화

## 검증 결과 요약
- `ansible-playbook -i inventory.ini site.yml --syntax-check` 통과
- `--tags common --limit n4200` 실행 통과
- `--tags common --limit arm,vps` 실행 통과
- `tmp_nodes` 실행 시 n4200의 외부 네트워크 상태에 따라 일시 실패 가능성은 있으나, 재시도/재실행 시 통과 확인

## 참고
- 이번 변경의 핵심은 Docker 키 배포 자체보다, APT 소스 충돌/네트워크 흔들림을 함께 제어해 전체 common 실행의 실패 확률을 낮춘 점이다.
