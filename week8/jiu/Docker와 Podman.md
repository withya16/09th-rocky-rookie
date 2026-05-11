# Docker 구조

## 실행 흐름

```
user
  ↓
docker CLI
  ↓
dockerd
  ↓
containerd
  ↓
runc
  ↓
container process
```

## 구성 요소

| 구성 요소 | 역할 |
| --- | --- |
| docker CLI | 사용자 명령 입력 |
| dockerd | Docker daemon. Docker 객체 관리 |
| containerd | 컨테이너 생명주기 관리 |
| containerd-shim | containerd와 컨테이너 프로세스 사이에서 상태 관리 |
| runc | OCI 스펙에 따라 컨테이너 프로세스 실행 |

## dockerd

- Docker API 요청 처리
- 이미지 관리
- 컨테이너 관리
- 네트워크 관리
- 볼륨 관리
- containerd를 통해 컨테이너 실행 제어

## containerd

- 컨테이너 생명주기 관리
- 이미지 관리
- 스냅샷 관리
- 런타임 호출
- shim 프로세스 관리

## containerd-shim

- 컨테이너 프로세스 상태 관리
- containerd와 런타임 사이 연결
- 컨테이너 실행 이후에도 프로세스 관리 유지

## runc

- OCI 호환 저수준 컨테이너 런타임
- namespace, cgroup 설정
- 컨테이너 프로세스 생성
- 실제 실행의 마지막 단계 담당

---

# Podman 구조

## Podman이란

- Pod Manager의 약자
- OCI 표준을 따르는 컨테이너 엔진
- Docker와 유사한 CLI 제공
- RHEL 계열에서 기본적으로 사용하는 컨테이너 도구

## 실행 흐름

```
user
  ↓
podman
  ↓
OCI runtime
  ↓
container process
```

## 핵심 특징

| 특징 | 설명 |
| --- | --- |
| Daemonless | 중앙 daemon 없이 컨테이너 실행 |
| Rootless | 일반 사용자 권한으로 컨테이너 실행 가능 |
| Docker CLI 유사성 | Docker와 유사한 명령어 제공 |
| Pod 지원 | 컨테이너뿐 아니라 Pod 단위 관리 가능 |
| OCI 호환 | OCI 이미지와 컨테이너 실행 지원 |
| systemd 통합 | 컨테이너를 systemd 서비스처럼 관리 가능 |

## Rootless

- root 권한 없이 컨테이너 실행 가능
- daemon이 모든 컨테이너 작업을 대신 수행하지 않음
- 사용자 단위로 컨테이너 실행 범위 제한 가능
- 시스템 전체 권한 노출 감소

## systemd 통합

- Podman 컨테이너를 systemd 서비스처럼 관리 가능
- 서버 재부팅 후 자동 시작 구성 가능
- 일반 서비스와 같은 방식으로 상태 확인 가능

```bash
systemctl --user status container-web.service
```

---

# Docker vs Podman

## 구조 비교

```
Docker

user
  ↓
docker CLI
  ↓
dockerd
  ↓
containerd
  ↓
runc
  ↓
container process
```

```
Podman

user
  ↓
podman
  ↓
OCI runtime
  ↓
container process
```

## 비교표

| 구분 | Docker | Podman |
| --- | --- | --- |
| 구조 | 클라이언트-서버 구조 | daemonless 구조 |
| 중앙 daemon | dockerd 사용 | 중앙 daemon 없음 |
| 실행 권한 | daemon 권한 영향 큼 | rootless 실행에 강점 |
| CLI | `docker` | `podman` |
| Pod 지원 | 기본 중심은 컨테이너 | Pod 관리 지원 |
| systemd 연동 | 별도 구성 필요 | systemd 연동 강점 |
| RHEL 기본 도구 | 기본 도구 아님 | RHEL 계열 기본 컨테이너 도구 |

## 정리

- Docker
    - 개발자 생태계와 사용 경험 강점
    - 중앙 daemon 기반 구조
- Podman
    - daemonless 구조
    - rootless 실행 강점
    - RHEL 운영 환경과 잘 맞음

---

# RHEL의 컨테이너 도구

## 기본 방향

- RHEL에서는 Docker 중심이 아님
- Podman, Buildah, Skopeo 조합 사용
- 실행, 빌드, 원격 이미지 관리를 도구별로 분리

## 도구별 역할

| 도구 | 역할 |
| --- | --- |
| Podman | 컨테이너, Pod, 이미지 관리 |
| Buildah | 컨테이너 이미지 빌드, 수정, 푸시 |
| Skopeo | 원격 저장소 이미지 복사, 검사, 삭제 |
| runc | OCI 컨테이너 실행 런타임 |
| crun | C 기반 OCI 런타임. rootless 환경에서 자주 사용 |

## 역할 분리

```
Image build       → Buildah
Container run     → Podman
Remote image ops  → Skopeo
Low-level runtime → runc / crun
```
