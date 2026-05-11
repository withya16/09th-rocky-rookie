# 컨테이너 기초와 Docker · Podman 활용

## 공부 흐름

이 문서는 다음 순서로 구성되어 있다. 각 단계는 이전 단계의 지식을 기반으로 쌓아올리는 구조이다.

```
1. 사전 지식            네임스페이스, cgroup, OCI, 이미지/레지스트리 등 핵심 용어 정리
       ↓
2. 컨테이너 vs 가상머신   하이퍼바이저 가상화 → 컨테이너 가상화 → 두 방식의 구조적 차이
       ↓                "왜 컨테이너가 가벼운가, VM과 어떻게 다른가"
       ↓
3. Docker · Podman 개념  Docker의 등장 배경 → 데몬 구조 → Podman의 데몬리스 모델
       ↓                "두 도구의 철학과 RHEL 계열에서 Podman을 쓰는 이유"
       ↓
4. 이미지 관리           이미지 구조(레이어) → 레지스트리 → search/pull/images/rmi
       ↓                "컨테이너의 재료인 이미지를 다루는 방법"
       ↓
5. 컨테이너 실행 실습    run → ps → stop/start → exec/attach → rm/logs
       ↓                "이미지에서 컨테이너를 만들고 운영·정리하는 핵심 흐름"
       ↓
6. 포트·볼륨·네트워크    -p 포트 매핑 → -v 볼륨 → bridge/host 네트워크
                        "컨테이너를 외부와 연결하고 데이터를 영속화하는 방법"
```

---

## 사전 지식

### 1) 구성 요소 — 컨테이너·이미지·레지스트리

> **컨테이너(Container)**
> 호스트 운영체제의 커널을 공유하면서, 자신만의 격리된 파일 시스템·프로세스·네트워크 공간에서 동작하는 경량 실행 단위이다.
> 애플리케이션과 그 실행에 필요한 라이브러리·설정을 하나의 패키지로 묶어 어디서나 동일하게 실행할 수 있게 만든다.

> **이미지(Image)**
> 컨테이너를 만들기 위한 **읽기 전용 템플릿**이다. 애플리케이션 바이너리, 의존 라이브러리, 환경 변수, 실행 명령 등이 하나의 파일 묶음으로 패키징되어 있다.
> 여러 개의 **레이어(Layer)** 가 쌓인 구조이며, 동일한 레이어는 캐시·재사용된다.

> **레지스트리(Registry)**
> 컨테이너 이미지를 저장·배포하는 원격 저장소이다. `pull`로 내려받고 `push`로 업로드한다.
> 대표 예시: **Docker Hub(`docker.io`)**, **Red Hat Quay(`quay.io`)**, **Red Hat Registry(`registry.redhat.io`)**, **GitHub Container Registry(`ghcr.io`)**.

### 2) 커널 기능 — 네임스페이스와 cgroup

컨테이너의 "격리"와 "제한"을 실제로 가능하게 해주는 리눅스 커널의 두 기둥이다.

> **네임스페이스(Namespace)**
> 프로세스에게 **시스템 리소스의 독립된 시야**를 제공하는 커널 기능이다.
> PID·NET·MNT·UTS·IPC·USER·CGROUP 7가지 종류가 있으며, 각 컨테이너는 자신만의 네임스페이스 안에서 실행되어 다른 컨테이너의 프로세스나 네트워크를 볼 수 없다.

> **컨트롤 그룹(cgroup, Control Group)**
> 프로세스 그룹의 **CPU·메모리·디스크 I/O·네트워크 사용량을 제한·계측**하는 커널 기능이다.
> 컨테이너 런타임은 cgroup을 통해 한 컨테이너가 호스트 자원을 독점하지 못하도록 통제한다.

### 3) 표준과 런타임 — OCI · runc · crun

> **OCI (Open Container Initiative)**
> 컨테이너 이미지 포맷과 런타임 동작을 표준화한 사양이다. 2015년 Docker·Red Hat 등이 주도하여 발족했다.
> **OCI Image Spec**(이미지 포맷)과 **OCI Runtime Spec**(런타임 동작)으로 구성되며, Docker·Podman·containerd·CRI-O 등 모든 주요 컨테이너 도구가 이 사양을 준수한다.

> **컨테이너 런타임(Container Runtime)**
> OCI 사양에 따라 실제로 컨테이너 프로세스를 만들고 실행하는 **저수준 도구**이다.
> `**runc`**(Docker 기본)와 `**crun**`(Podman 기본, C로 작성되어 더 빠름)이 대표적이다. 사용자가 직접 다루는 일은 거의 없고, Docker·Podman이 내부적으로 호출한다.

### 4) 실행 모델 — 데몬 vs 루트리스

> **데몬(Daemon)**
> 백그라운드에서 항상 동작하며 요청을 받아 처리하는 시스템 프로세스이다.
> Docker는 `dockerd`라는 데몬이 root 권한으로 실행되며 모든 컨테이너 작업을 중앙 처리한다. **Podman은 데몬을 두지 않는다(Daemonless).**

> **루트리스(Rootless) 컨테이너**
> root 권한 없이 일반 사용자 권한으로 컨테이너를 실행하는 방식이다.
> user namespace를 활용해 컨테이너 내부의 root(UID 0)를 호스트의 일반 사용자에 매핑한다. **Podman의 핵심 강점**이며, 보안과 다중 사용자 환경에서 큰 이점을 제공한다.

### 5) 외부와의 연결 — 볼륨과 포트 매핑

컨테이너는 기본적으로 외부와 단절된 상자다. 이 둘은 그 격리에 의도적으로 "구멍"을 뚫는 두 통로다.

> **볼륨(Volume)**
> 컨테이너 파일 시스템은 컨테이너 삭제와 함께 사라지므로, **데이터를 영속적으로 보관하려면 호스트의 디렉터리나 명명된 저장소를 컨테이너에 연결**해야 한다.
> 두 가지 방식이 있다: **bind mount**(호스트 경로를 직접 연결) / **named volume**(컨테이너 도구가 관리하는 저장소).

> **포트 매핑(Port Mapping / Port Publishing)**
> 컨테이너 네트워크는 기본적으로 격리되어 있어 외부에서 직접 접근할 수 없다.
> **호스트의 특정 포트로 들어오는 트래픽을 컨테이너의 내부 포트로 전달**하는 설정이며, `-p HOST:CONTAINER` 형태로 지정한다.

---

## 컨테이너와 가상머신의 차이 이해

### 가상화(Virtualization)란

가상화는 **하나의 물리 하드웨어 위에 여러 개의 격리된 실행 환경을 만드는 기술**이다. 격리의 단위와 깊이가 어디까지인지에 따라 크게 두 갈래로 나뉜다.

- **하이퍼바이저 기반 가상화 (Type-1/2 Hypervisor):** 하드웨어를 통째로 가상화하여 그 위에 **독립된 게스트 OS**를 통째로 올린다. VMware ESXi, KVM, Hyper-V, VirtualBox 등이 대표적이다. → **가상머신(VM)**
- **컨테이너 기반 가상화 (OS-level Virtualization):** 호스트의 커널을 공유하면서 **사용자 공간(user space)만 격리**한다. 별도의 게스트 OS를 띄우지 않으므로 매우 가볍다. Docker, Podman, LXC 등이 대표적이다. → **컨테이너**

```
┌─────────────────────────────────────┐    ┌─────────────────────────────────────┐
│         가상머신 (VM) 방식            │    │         컨테이너 방식                  │
├─────────────────────────────────────┤    ├─────────────────────────────────────┤
│ ┌─────┐ ┌─────┐ ┌─────┐             │    │ ┌─────┐ ┌─────┐ ┌─────┐             │
│ │ App │ │ App │ │ App │             │    │ │ App │ │ App │ │ App │             │
│ ├─────┤ ├─────┤ ├─────┤             │    │ ├─────┤ ├─────┤ ├─────┤             │
│ │ Bin │ │ Bin │ │ Bin │             │    │ │ Bin │ │ Bin │ │ Bin │             │
│ │ Lib │ │ Lib │ │ Lib │             │    │ │ Lib │ │ Lib │ │ Lib │             │
│ ├─────┤ ├─────┤ ├─────┤             │    │ └─────┘ └─────┘ └─────┘             │
│ │Guest│ │Guest│ │Guest│             │    │                                     │
│ │ OS  │ │ OS  │ │ OS  │  (수 GB)    │    │  ┌─────────────────────────────┐    │
│ └─────┘ └─────┘ └─────┘             │    │  │  컨테이너 엔진 (Docker/Podman) │    │
│                                     │    │  └─────────────────────────────┘    │
│  ┌────────────────────────────┐     │    │                                     │
│  │      Hypervisor (KVM 등)    │     │    │  ┌─────────────────────────────┐    │
│  └────────────────────────────┘     │    │  │   호스트 OS (커널 공유)         │    │
│                                     │    │  └─────────────────────────────┘    │
│  ┌────────────────────────────┐     │    │                                     │
│  │       호스트 OS / 호스트 머신   │    │    │  ┌─────────────────────────────┐   │
│  └────────────────────────────┘     │    │  │       물리 하드웨어              │  │
│                                     │    │  └─────────────────────────────┘   │
│  ┌────────────────────────────┐     │    │                                     │
│  │       물리 하드웨어            │     │    │                                     │
│  └────────────────────────────┘     │    │                                     │
└─────────────────────────────────────┘    └─────────────────────────────────────┘
```

### VM과 컨테이너의 핵심 차이


| **항목**       | **가상머신 (VM)**                    | **컨테이너 (Container)**                      |
| ------------ | -------------------------------- | ----------------------------------------- |
| **격리 수준**    | 하드웨어 수준의 강한 격리                   | 커널을 공유한 프로세스 수준 격리                        |
| **OS**       | 게스트 OS 별도 부팅 필요                  | 호스트 OS의 커널을 공유 (별도 OS 없음)                 |
| **부팅 시간**    | 수 십초 ~ 분 단위                      | 수 백 밀리초 ~ 수 초                             |
| **이미지 크기**   | 수 GB ~ 수 십 GB                    | 수 MB ~ 수 백 MB                             |
| **리소스 오버헤드** | 큼 (CPU·RAM·디스크)                  | 작음 (네임스페이스 + cgroup 수준)                   |
| **밀집도**      | 한 호스트에 수~수십 개                    | 한 호스트에 수백~수천 개                            |
| **이식성**      | 동일 하이퍼바이저 간                      | OCI 호환 런타임이면 어디서나 동작                      |
| **OS 다양성**   | 윈도우 위에 리눅스 등 자유로움                | 호스트와 동일한 커널만 가능 (Linux 호스트 위에 Linux 컨테이너) |
| **격리 보안**    | 매우 강함 (커널까지 분리)                  | 상대적으로 약함 (커널 공유 → 커널 취약점 노출)              |
| **대표 도구**    | VMware, KVM, Hyper-V, VirtualBox | Docker, Podman, LXC, containerd           |


### 컨테이너가 가벼운 이유 — 네임스페이스와 cgroup

컨테이너는 **새 OS를 띄우는 게 아니라**, 호스트 커널이 제공하는 두 가지 기능을 사용해 **마치 격리된 시스템처럼 보이게** 만든다.

**1) 네임스페이스 — "다른 시야"를 제공**


| **네임스페이스** | **격리 대상**                 | **효과**                                        |
| ---------- | ------------------------- | --------------------------------------------- |
| `PID`      | 프로세스 ID                   | 컨테이너 내부에서 자기 프로세스만 보임 (PID 1부터 시작)            |
| `NET`      | 네트워크 인터페이스, 라우팅, 포트       | 자기만의 IP, 포트 공간을 가짐                            |
| `MNT`      | 마운트 포인트, 파일 시스템           | 호스트와 다른 루트 파일 시스템(`/`)을 가짐                    |
| `UTS`      | 호스트명, 도메인명                | 컨테이너마다 다른 hostname 설정 가능                      |
| `IPC`      | System V IPC, POSIX 메시지 큐 | 프로세스 간 통신 격리                                  |
| `USER`     | UID/GID 매핑                | 컨테이너 내 root(0) ↔ 호스트의 일반 사용자 매핑 (rootless 핵심) |
| `CGROUP`   | cgroup 계층                 | 컨테이너가 자신의 cgroup만 볼 수 있도록 격리                  |


**2) cgroup — "사용량 제한"을 강제**

cgroup은 프로세스가 사용할 수 있는 CPU 시간, 메모리, 블록 I/O, 네트워크 대역폭 등에 **상한을 두고 측정**한다. 이를 통해 한 컨테이너가 호스트의 리소스를 독점하는 것을 막는다.

```
            ┌────────────── 호스트 리눅스 커널 ──────────────┐
            │                                            │
            │  ┌──────── 컨테이너 A ─────────┐              │
            │  │ PID NS · NET NS · MNT NS  │              │
            │  │ cgroup: CPU=1, MEM=512M   │              │
            │  └───────────────────────────┘              │
            │                                            │
            │  ┌──────── 컨테이너 B ─────────┐              │
            │  │ PID NS · NET NS · MNT NS  │              │
            │  │ cgroup: CPU=2, MEM=1G     │              │
            │  └───────────────────────────┘              │
            │                                            │
            └────────────────────────────────────────────┘
                       (커널 1개를 모두가 공유)
```

> **이 구조의 트레이드오프:** 커널을 공유하므로 OS를 통째로 띄우는 VM보다 훨씬 가볍지만, **호스트와 컨테이너의 OS가 같은 종류여야 한다**. 리눅스 호스트 위에 윈도우 컨테이너를 직접 돌릴 수 없는 이유다(WSL2처럼 가상화를 한 번 거치면 가능).

### 어떤 상황에 무엇을 쓰는가


| **상황**                               | **권장**               | **이유**                        |
| ------------------------------------ | -------------------- | ----------------------------- |
| 서로 다른 OS(예: Windows 위에 Linux)를 함께 운영 | 가상머신                 | 커널 자체가 다름 — 컨테이너로는 불가         |
| 마이크로서비스 다수를 빠르게 배포·확장                | 컨테이너                 | 부팅 빠름·이미지 가벼움·CI/CD 친화적       |
| 강한 격리(예: 다중 테넌트, 보안 민감)가 필요          | 가상머신 (또는 VM 안의 컨테이너) | 커널 공유의 위험을 피해야 함              |
| 개발 환경을 팀 전체가 동일하게 재현                 | 컨테이너                 | 이미지 = 환경 자체. 코드와 함께 버전 관리 가능  |
| 레거시 모놀리식 애플리케이션을 그대로 실행              | 가상머신                 | OS·커널 의존성이 강할 때 컨테이너로 옮기기 어려움 |


---

## Docker와 Podman의 개념

### Docker의 등장과 클라이언트-서버 구조

Docker는 2013년 dotCloud(현 Docker Inc.)가 공개한 컨테이너 플랫폼이다. 리눅스의 **네임스페이스·cgroup·UnionFS** 같은 기존 기능을 **이미지·레지스트리·CLI·데몬**이라는 사용성 좋은 패키지로 묶어 제공하면서 컨테이너를 대중화시켰다.

Docker는 전형적인 **클라이언트-서버(C/S) 구조**로 설계되어 있다.

```
┌──────────┐    REST API     ┌──────────────────────┐
│  docker  │ ──────────────→ │   dockerd (데몬)       │
│  (CLI)   │   (Unix sock)   │                      │
│          │ ←────────────── │   루트 권한으로 동작        │
└──────────┘                 │                      │
   사용자                      │  ┌────────────────┐  │
                            │  │ containerd     │  │
                            │  │   (컨테이너 관리)    │  │
                            │  └───────┬────────┘  │
                            │          │           │
                            │  ┌───────▼────────┐  │
                            │  │ runc (OCI 런타임)│  │
                            │  └────────────────┘  │
                            └──────────────────────┘
                                    ↓
                            ┌──────────────────┐
                            │  컨테이너 프로세스      │
                            └──────────────────┘
```

- 사용자가 입력한 `docker` 명령은 **CLI 클라이언트**일 뿐이며, 실제 작업은 백그라운드에서 항상 떠 있는 `**dockerd` 데몬**이 처리한다.
- `dockerd`는 root 권한으로 동작하며, 그 자식으로 `containerd` → `runc`를 거쳐 컨테이너 프로세스를 만든다.
- 모든 컨테이너의 부모는 결국 `dockerd`이므로, **데몬이 죽으면 컨테이너도 영향을 받는다**.

### Podman의 등장 배경 — 데몬리스 + 루트리스

Podman(POD MANager)은 Red Hat이 RHEL 8부터 Docker의 대체재로 공식 채택한 컨테이너 도구이다. 두 가지 큰 차이를 갖는다.

**1) 데몬이 없다 (Daemonless)**

`podman` 명령은 호출될 때마다 **자기 자신이 컨테이너를 직접 fork·exec**한다. 별도의 데몬을 두지 않으므로:

- root로 항상 떠 있는 프로세스가 사라져 **공격 표면이 줄어든다**.
- 컨테이너의 부모는 `podman` 또는 그를 호출한 `systemd`이므로, `**systemd`로 깔끔하게 관리**할 수 있다.
- 단점: Docker처럼 항상 떠 있는 데몬이 없으니, 컨테이너의 자동 재시작 같은 기능은 `systemd` 유닛과 결합해야 한다.

**2) 루트리스(Rootless) 모드를 1급 시민으로 지원**

일반 사용자 권한으로 `podman run`을 실행하면, **user namespace** 매핑을 통해 컨테이너 내부의 root(UID 0)가 호스트에서는 그 사용자 자신이 된다. `/etc/subuid`, `/etc/subgid`에 미리 부여된 UID 범위를 이용하여 격리된 사용자 공간을 만든다.

```
Docker (기본):                       Podman (rootless):

  사용자 → docker CLI                  사용자 → podman CLI
              │                                  │
              ▼                                  ▼
        dockerd (root)                  컨테이너 프로세스
              │                       (호스트에서는 일반 사용자,
              ▼                        컨테이너 내부에선 UID 0)
        컨테이너 (root)
```

### Podman과 Docker의 명령어 호환성

Podman은 Docker와 **거의 동일한 CLI 인터페이스**를 의도적으로 유지한다. 대부분의 경우 `docker`를 `podman`으로 바꿔 부르기만 해도 동작한다. RHEL 계열에서는 `podman-docker` 패키지를 설치하면 `docker` 명령어를 그대로 입력해도 podman으로 라우팅되는 호환 셸이 제공된다.

```bash
# 호환 패키지 설치 (선택)
sudo dnf install -y podman-docker
docker --version
# podman version 5.x.x   ← 실제론 podman이 응답
```


| **항목**           | **Docker**                            | **Podman**                             |
| ---------------- | ------------------------------------- | -------------------------------------- |
| 아키텍처             | 클라이언트-서버 (데몬 필수)                      | 데몬리스 (포크/익셉)                           |
| 기본 권한            | root 데몬                               | rootless가 기본                           |
| 단일 컨테이너 실행 명령    | `docker run`                          | `podman run`                           |
| 멀티 컨테이너 그룹       | Docker Compose (`docker-compose.yml`) | Pod 개념 내장 + Quadlet/Compose 지원         |
| 시스템 통합           | `docker.service` 한 개                  | 컨테이너 단위 systemd 유닛 자동 생성 (`quadlet`)   |
| OCI 런타임          | `runc`                                | `crun` (기본), `runc`도 지원                |
| 이미지 빌드           | `docker build`                        | `podman build` (Buildah 백엔드)           |
| 기본 이미지 저장 경로     | `/var/lib/docker/`                    | rootless: `~/.local/share/containers/` |
| RHEL/Rocky 공식 지원 | 비공식                                   | 공식 (RHEL 8+ 기본 컨테이너 도구)                |


> **RHEL 10에서의 위치:** Red Hat은 RHEL 8 출시 시점에 Docker를 공식 저장소에서 제거하고 Podman·Buildah·Skopeo 트리오를 표준으로 채택했다. 이는 데몬리스·루트리스라는 보안 모델, 그리고 systemd와의 자연스러운 통합 때문이다. 학습 차원에서는 두 도구의 명령이 거의 동일하므로, 한쪽을 익히면 다른 쪽으로 어렵지 않게 옮겨갈 수 있다.

### Podman 설치 및 환경 확인

Rocky Linux / RHEL 10에서는 기본 패키지로 제공된다.

```bash
# 설치
sudo dnf install -y podman

# 버전 확인
podman --version
# podman version 5.6.0

# 시스템 정보(스토리지 드라이버, 그래프 루트, 네트워크 백엔드 등)
podman info
```

`podman info` 출력의 주요 필드는 다음과 같다.


| **필드**           | **의미**                                                              |
| ---------------- | ------------------------------------------------------------------- |
| `graphRoot`      | 이미지·컨테이너 메타데이터가 저장되는 경로 (rootless는 `~/.local/share/containers/...`) |
| `runRoot`        | 실행 중 임시 데이터 경로                                                      |
| `ociRuntime`     | 사용 중인 OCI 런타임 (`crun` 또는 `runc`)                                    |
| `networkBackend` | 네트워크 백엔드 (`netavark` 또는 구버전 `cni`)                                  |
| `cgroupVersion`  | cgroup v1 / v2                                                      |
| `rootless`       | 현재 세션이 루트리스인지 여부                                                    |


---

## 이미지 검색·다운로드·삭제·관리

### 이미지의 구조 — 레이어와 다이제스트

컨테이너 이미지는 단일 파일이 아니라 **여러 개의 읽기 전용 레이어가 쌓인 구조**이다. 각 레이어는 Dockerfile의 한 줄(또는 빌드 단계)에 대응하며, 변경되지 않은 레이어는 **재사용·캐시**되어 다운로드와 빌드를 빠르게 만든다.

```
┌─────────────────────────────────────┐  ← 최상위(쓰기 가능 레이어, 컨테이너 실행 시 생성)
│   /var/lib/myapp/cache              │
├─────────────────────────────────────┤
│   /etc/myapp.conf  (RUN cp ...)     │  ← 레이어 3
├─────────────────────────────────────┤
│   /opt/myapp       (COPY ...)       │  ← 레이어 2
├─────────────────────────────────────┤
│   apt/dnf install된 패키지              │  ← 레이어 1
├─────────────────────────────────────┤
│   Base OS (예: ubi9, alpine, debian)  │  ← 베이스 레이어
└─────────────────────────────────────┘
```

각 이미지는 **태그(tag)** 와 **다이제스트(digest)** 두 가지로 식별된다.

- **태그:** 사람이 읽기 쉬운 버전 라벨. 예) `nginx:1.27`, `nginx:latest`. 같은 태그도 시간이 지나면 다른 이미지를 가리킬 수 있다(가변).
- **다이제스트:** 이미지 내용 전체의 SHA-256 해시. 예) `nginx@sha256:abc123...`. 동일한 다이제스트는 영원히 동일한 이미지를 보장한다(불변).

> 운영 환경에서는 `latest` 같은 가변 태그 대신 **다이제스트나 명시적 버전 태그**를 쓰는 것이 재현성·보안 측면에서 권장된다.

### 레지스트리 설정 — `registries.conf`

`podman search` 또는 `podman pull <이미지명>`을 짧은 이름(예: `nginx`)으로 입력하면, 어느 레지스트리에서 검색할지 결정해야 한다. 이 정책은 `/etc/containers/registries.conf`에 정의된다.

```bash
sudo cat /etc/containers/registries.conf
```

```toml
unqualified-search-registries = ["registry.access.redhat.com", "registry.redhat.io", "docker.io"]

[[registry]]
location = "docker.io"

[[registry]]
location = "quay.io"
```

- `unqualified-search-registries`: 짧은 이름으로 검색·풀링할 때 시도할 레지스트리 순서.
- 짧은 이름을 쓰면 어떤 레지스트리가 응답했는지 모호하므로, **정식 명칭(FQIN, Fully Qualified Image Name)을 사용하는 것이 권장**된다. 예: `docker.io/library/nginx:1.27`.

### 이미지 검색 — `podman search`

```bash
# 짧은 이름으로 검색 (registries.conf 순서대로 조회)
podman search nginx

# 특정 레지스트리에서만 검색
podman search docker.io/nginx

# 별점 N개 이상만
podman search --filter=stars=100 nginx

# 결과 행 수 제한
podman search --limit 5 alpine
```

```text
NAME                                     DESCRIPTION                                   STARS   OFFICIAL
docker.io/library/nginx                  Official build of Nginx.                       19000   [OK]
docker.io/library/alpine                 A minimal Docker image based on Alpine Linux   10000   [OK]
quay.io/libpod/alpine                    Alpine container for libpod testing             0      
...
```


| **컬럼**     | **의미**                              |
| ---------- | ----------------------------------- |
| `NAME`     | `<레지스트리>/<네임스페이스>/<이미지명>` 형식의 정식 명칭 |
| `STARS`    | 사용자 즐겨찾기 수 (인기 지표)                  |
| `OFFICIAL` | 레지스트리가 공식 인증한 이미지 표시 (`[OK]` = 공식)  |


### 이미지 다운로드 — `podman pull`

```bash
# 정식 명칭 + 명시적 태그 (권장)
podman pull docker.io/library/nginx:1.27

# 다이제스트로 고정 (불변, 운영 권장)
podman pull docker.io/library/nginx@sha256:abc123...

# 짧은 이름 (registries.conf의 첫 매칭에서 가져옴)
podman pull alpine

# RHEL 공식 베이스 이미지
podman pull registry.access.redhat.com/ubi9/ubi:latest

# 모든 태그 한 번에
podman pull --all-tags docker.io/library/alpine
```

다운로드 진행 시 출력되는 각 줄(`Copying blob ...`)이 하나의 레이어 다운로드를 의미한다. 이미 받아둔 레이어는 `Copying blob 〈sha256〉 [skipped, already exists]`로 표시되며 다시 받지 않는다.

### 보유 이미지 목록 — `podman images`

```bash
# 기본 출력
podman images

# 다이제스트 포함
podman images --digests

# 특정 이미지만 필터
podman images docker.io/library/nginx

# 댕글링(태그 없는 중간 이미지)만 보기
podman images --filter "dangling=true"

# 특정 이미지 이후 생성된 것만
podman images --filter "since=docker.io/library/alpine:latest"
```

```text
REPOSITORY                       TAG     IMAGE ID       CREATED       SIZE
docker.io/library/nginx          1.27    a3f2d5e8c1b9   2 weeks ago   188 MB
docker.io/library/nginx          latest  a3f2d5e8c1b9   2 weeks ago   188 MB
docker.io/library/alpine         3.20    7c1e8a9d6e2f   5 weeks ago    7.8 MB
registry.access.redhat.com/ubi9  latest  9b4f3e7c1a25   1 month ago   216 MB
```

> **같은 IMAGE ID, 다른 태그:** 위 출력에서 `nginx:1.27`과 `nginx:latest`의 IMAGE ID가 동일하다. 같은 이미지에 두 개의 태그가 붙어 있다는 뜻이다. 둘 중 하나를 `rmi`로 지워도 다른 태그는 남는다.

### 이미지 상세 정보 — `podman inspect`

```bash
podman inspect docker.io/library/nginx:1.27
```

JSON으로 이미지의 전체 메타데이터(레이어, 환경 변수, 노출 포트, 진입점 등)가 출력된다. 일부 필드만 추출할 때는 `--format`(Go 템플릿)을 사용한다.

```bash
# 노출 포트 확인
podman inspect --format '{{.Config.ExposedPorts}}' nginx:1.27
# map[80/tcp:{}]

# 기본 진입점 + CMD
podman inspect --format '{{.Config.Entrypoint}} / {{.Config.Cmd}}' nginx:1.27

# 환경 변수
podman inspect --format '{{range .Config.Env}}{{println .}}{{end}}' nginx:1.27
```

### 이미지 히스토리 — `podman history`

이미지가 어떤 빌드 단계로 만들어졌는지(어떤 Dockerfile 명령으로 어떤 레이어가 추가됐는지) 거꾸로 따라가 본다.

```bash
podman history docker.io/library/nginx:1.27
```

```text
ID            CREATED        CREATED BY                                       SIZE
a3f2d5e8c1b9  2 weeks ago    /bin/sh -c #(nop)  CMD ["nginx" "-g" "daemon...] 0 B
<missing>     2 weeks ago    /bin/sh -c #(nop)  STOPSIGNAL SIGQUIT             0 B
<missing>     2 weeks ago    /bin/sh -c #(nop)  EXPOSE 80                       0 B
<missing>     2 weeks ago    /bin/sh -c set -x  && groupadd --system ...   3.5 MB
...
```

### 이미지 삭제 — `podman rmi`

```bash
# 이름:태그로 삭제
podman rmi docker.io/library/nginx:1.27

# IMAGE ID로 삭제
podman rmi a3f2d5e8c1b9

# 여러 개 동시
podman rmi nginx:1.27 alpine:3.20

# 강제 삭제 (이 이미지로 만들어진 컨테이너가 있어도)
podman rmi -f nginx:latest

# 모든 미사용 이미지 일괄 삭제
podman image prune

# 댕글링뿐 아니라 어떤 컨테이너도 사용하지 않는 모든 이미지 삭제
podman image prune -a
```

```bash
# 사용 중인 이미지를 그냥 지우려 할 때
$ podman rmi nginx:1.27
Error: image used by 06eafaa90a06...: image is in use by a container
# → 컨테이너부터 제거하거나 -f 옵션 사용
```

### 이미지 정리 작업 요약


| **명령**                             | **대상**                                           |
| ---------------------------------- | ------------------------------------------------ |
| `podman image prune`               | 댕글링(이름·태그가 없는 중간 이미지)만 삭제                        |
| `podman image prune -a`            | 컨테이너에 사용되지 않는 모든 이미지 삭제                          |
| `podman container prune`           | 중지된 컨테이너만 삭제 (이미지는 보존)                           |
| `podman volume prune`              | 어떤 컨테이너에도 연결되지 않은 이름 있는 볼륨 삭제                    |
| `podman system prune`              | 위 항목을 일괄 정리 (이미지·컨테이너·네트워크·빌드 캐시까지) — **신중히 사용** |
| `podman system prune -a --volumes` | 사용되지 않는 모든 자원 + 볼륨까지 — **데이터 손실 위험**, 운영에서는 금지   |


---

## 컨테이너 실행·중지·접속 실습

### 실행의 핵심 명령 — `podman run`

`run`은 두 가지 일을 한 번에 한다.

1. (이미지가 로컬에 없으면) 이미지를 `pull`로 받아온다.
2. 그 이미지로 컨테이너를 만들고(`create`) 즉시 시작한다(`start`).

```bash
# 가장 단순한 형태 — 이미지를 받아 컨테이너를 만들고 종료될 때까지 출력 보기
podman run docker.io/library/hello-world

# 자주 쓰는 옵션 조합
podman run -d --name web -p 8080:80 docker.io/library/nginx:1.27
```


| **옵션**               | **의미**                                                       |
| -------------------- | ------------------------------------------------------------ |
| `-d`, `--detach`     | 컨테이너를 백그라운드로 실행 (터미널을 점유하지 않음)                               |
| `--name <이름>`        | 컨테이너에 사람이 읽기 쉬운 이름 부여 (생략 시 자동 생성: `silly_einstein` 같은 두 단어) |
| `-p HOST:CONTAINER`  | 호스트 포트 → 컨테이너 포트 매핑 (포트 매핑 절에서 상세히)                          |
| `-v HOST:CONTAINER`  | 볼륨 연결 (볼륨 절에서 상세히)                                           |
| `-e KEY=VALUE`       | 환경 변수 주입                                                     |
| `-it`                | `-i`(stdin 유지) + `-t`(TTY 할당) — 셸로 들어가는 인터랙티브 실행 시 사용        |
| `--rm`               | 컨테이너가 종료되면 자동 삭제 (일회성 실행에 유용)                                |
| `--restart=<정책>`     | 재시작 정책 (`no`/`on-failure`/`always`/`unless-stopped`)         |
| `--network=<이름>`     | 사용할 네트워크 지정 (네트워크 절 참조)                                      |
| `--memory=<크기>`      | 메모리 상한 (예: `512m`, `1g`)                                     |
| `--cpus=<수>`         | CPU 코어 상한 (소수점 가능, 예: `0.5`, `2`)                            |
| `--user <UID>:<GID>` | 컨테이너 내부 실행 사용자 변경                                            |


### 실행 중 컨테이너 확인 — `podman ps`

```bash
# 실행 중인 컨테이너만
podman ps

# 중지된 것까지 모두
podman ps -a

# 컨테이너 ID만 출력 (스크립트에서 유용)
podman ps -q

# 최근 1개만
podman ps -l

# 필터 적용
podman ps --filter "status=running" --filter "name=web"

# 출력 포맷 커스터마이즈 (Go 템플릿)
podman ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

```text
CONTAINER ID  IMAGE                           COMMAND               CREATED         STATUS         PORTS                 NAMES
06eafaa90a06  docker.io/library/nginx:1.27    nginx -g daemon o...  10 seconds ago  Up 9 seconds   0.0.0.0:8080->80/tcp  web
```


| **컬럼**         | **의미**                                         |
| -------------- | ---------------------------------------------- |
| `CONTAINER ID` | 컨테이너의 고유 ID (앞 12자리)                           |
| `IMAGE`        | 컨테이너의 베이스 이미지                                  |
| `COMMAND`      | 컨테이너 내부에서 실행 중인 메인 프로세스                        |
| `STATUS`       | `Up <기간>` / `Exited (코드) <기간> ago` / `Created` |
| `PORTS`        | 매핑된 포트 정보 (`HOST:CONTAINER/PROTO`)             |
| `NAMES`        | 컨테이너 이름                                        |


### 중지·재시작·삭제

```bash
# 정상 중지 (SIGTERM 후 기본 10초 후 SIGKILL)
podman stop web

# 즉시 강제 중지 (SIGKILL)
podman kill web

# 재시작 (중지 → 시작)
podman restart web

# 중지된 컨테이너 다시 시작
podman start web

# 일시 정지 (cgroup freeze, 메모리·상태는 유지)
podman pause web
podman unpause web

# 컨테이너 삭제 (중지 상태여야 함)
podman rm web

# 실행 중이어도 강제 삭제
podman rm -f web

# 모든 중지된 컨테이너 삭제
podman container prune
```

> **stop과 kill의 차이:** `stop`은 컨테이너 안의 메인 프로세스에 먼저 `SIGTERM`을 보내 정상 종료를 유도하고, 일정 시간(`-t` 옵션, 기본 10초) 안에 끝나지 않으면 `SIGKILL`로 강제 종료한다. `kill`은 처음부터 `SIGKILL`을 보낸다. 데이터베이스 같은 상태 저장형 컨테이너는 반드시 `stop`을 우선 사용해야 데이터 손상을 피할 수 있다.

### 컨테이너 내부 접속 — `exec`와 `attach`


| **명령**          | **동작**                                        | **용도**                              |
| --------------- | --------------------------------------------- | ----------------------------------- |
| `podman exec`   | 실행 중인 컨테이너 안에서 **새 프로세스**를 추가로 실행             | 내부에서 디버깅, 셸 접속, 추가 명령 실행 (가장 자주 사용) |
| `podman attach` | 컨테이너의 **메인 프로세스(PID 1)**의 stdin/stdout에 직접 연결 | 메인 프로세스의 입출력을 그대로 보고 싶을 때 (드물게 사용)  |


```bash
# 새 셸로 들어가기 (가장 흔한 패턴)
podman exec -it web /bin/bash
# nginx 같은 일부 슬림 이미지는 bash가 없으므로 sh 사용
podman exec -it web /bin/sh

# 컨테이너 안에서 특정 명령 한 번만 실행
podman exec web ls /usr/share/nginx/html
podman exec web cat /etc/nginx/nginx.conf

# root 권한으로 들어가기 (이미지가 비-root로 동작 중일 때)
podman exec -u 0 -it web /bin/sh
```

```bash
# attach (메인 프로세스 출력 모니터링)
podman attach web
# Ctrl+P, Ctrl+Q  →  컨테이너를 죽이지 않고 빠져나오기
# Ctrl+C         →  메인 프로세스에 SIGINT 전달 (보통 컨테이너가 종료됨)
```

### 컨테이너 로그 확인 — `podman logs`

컨테이너 메인 프로세스가 stdout/stderr로 출력한 내용은 **자동으로 수집**되어 `podman logs`로 조회할 수 있다.

```bash
# 전체 로그 출력
podman logs web

# 마지막 50줄만
podman logs --tail 50 web

# 실시간 따라가기 (tail -f 와 유사, Ctrl+C로 종료)
podman logs -f web

# 타임스탬프 함께
podman logs -t web

# 최근 5분 이내만
podman logs --since 5m web

# 특정 시각 이전만
podman logs --until 2026-04-30T18:00:00 web
```

> **로그 저장 위치:** Podman은 기본적으로 컨테이너의 stdout/stderr를 호스트의 파일(`k8s-file` 드라이버)에 저장한다. journald와 통합하려면 `--log-driver=journald` 옵션을 쓰거나 `containers.conf`에서 변경한다.

### 컨테이너 상세 정보 및 통계

```bash
# 컨테이너 메타데이터 전체 (JSON)
podman inspect web

# 특정 정보만 추출
podman inspect --format '{{.NetworkSettings.IPAddress}}' web
podman inspect --format '{{.State.Status}}' web

# 실시간 리소스 사용량 (top 비슷, q 로 종료)
podman stats

# 한 번만 출력
podman stats --no-stream

# 컨테이너 내부 프로세스 목록
podman top web
```

```text
ID            NAME    CPU %     MEM USAGE / LIMIT   MEM %    NET IO        BLOCK IO    PIDS
06eafaa90a06  web     0.01%     12.5MB / 15.4GB     0.08%    1.2kB / 0B    0B / 0B     5
```

### 한 번에 정리하는 라이프사이클 흐름

```
              ┌────────────────────────────┐
              │   레지스트리 (docker.io 등)    │
              └──────────────┬─────────────┘
                             │ podman pull
                             ▼
                  ┌──────────────────────┐
   podman rmi ◄── │      이미지 (로컬)       │ ◄── podman build (Containerfile)
                  └──────────┬───────────┘
                             │ podman create / podman run
                             ▼
                  ┌──────────────────────┐
                  │  컨테이너 (Created)     │
                  └──────────┬───────────┘
                             │ podman start          ┌──────────────────┐
                             ▼                       │  podman exec     │
                  ┌──────────────────────┐ ───────►  │  podman attach   │
                  │  컨테이너 (Running)     │           │  podman logs -f  │
                  └──────────┬───────────┘           │  podman stats    │
                             │ podman stop          └──────────────────┘
                             │  / podman pause
                             ▼
                  ┌──────────────────────┐
   podman rm   ◄─ │  컨테이너 (Exited)      │ ◄── podman restart
                  └──────────────────────┘
```

---

## 포트 매핑·볼륨·네트워크 설정

여기까지 컨테이너를 띄우고 들어가는 방법을 익혔다. 그러나 실제 서비스가 되려면 **외부에서 접근할 수 있어야** 하고, **컨테이너가 사라져도 데이터가 살아남아야** 한다. 이 둘이 각각 **포트 매핑**과 **볼륨**이며, 이들이 동작하는 무대가 **네트워크**다.

### 포트 매핑 (Port Publishing)

컨테이너의 네트워크는 호스트의 네트워크와 분리되어 있어, 컨테이너 안에서 80번 포트로 띄운 서비스도 호스트의 80번 포트와는 무관하다. 외부에서 접근하게 하려면 **호스트의 포트를 컨테이너 포트로 전달**해야 한다.

```
         ┌──────── 호스트 ─────────────────────────────────────┐
         │                                                  │
   외부  │                              ┌─────── 컨테이너 ────┐ │
   요청  │  enp0s3:8080  ───────────►   │  nginx :80          │ │
 (TCP)   │   (호스트 포트)              매핑   │  (컨테이너 내부 포트) │ │
         │                              └────────────────────┘ │
         └──────────────────────────────────────────────────┘

                       podman run -p 8080:80 nginx
```

**기본 문법: `-p [HOST_IP:]HOST_PORT:CONTAINER_PORT[/PROTO]`**

```bash
# 가장 흔한 형태: 호스트 8080 → 컨테이너 80
podman run -d --name web1 -p 8080:80 docker.io/library/nginx

# 특정 호스트 IP에만 바인딩 (외부에서 접근 불가, localhost에서만 접근)
podman run -d --name web2 -p 127.0.0.1:9090:80 docker.io/library/nginx

# 호스트 포트 자동 할당 (사용 가능한 임의의 포트)
podman run -d --name web3 -p 80 docker.io/library/nginx
podman port web3
# 80/tcp -> 0.0.0.0:42351

# UDP 매핑
podman run -d --name dns -p 5353:53/udp docker.io/library/coredns

# 여러 포트 한 번에 매핑
podman run -d --name multi -p 8080:80 -p 8443:443 docker.io/library/nginx

# Dockerfile/이미지에 EXPOSE된 모든 포트를 호스트 임의 포트에 자동 매핑
podman run -d --name auto -P docker.io/library/nginx
```

**매핑 확인**

```bash
# 컨테이너의 포트 매핑만 보기
podman port web1
# 80/tcp -> 0.0.0.0:8080

# ps에서 PORTS 컬럼으로 확인
podman ps --format "table {{.Names}}\t{{.Ports}}"

# 호스트 측에서 실제 LISTEN 중인지 확인
ss -tlnp | grep 8080
```

**연결 검증**

```bash
# 호스트에서
curl -I http://127.0.0.1:8080
# HTTP/1.1 200 OK
# Server: nginx/1.27.x

# 다른 머신에서 (호스트 방화벽이 열려 있어야 함)
curl -I http://<HOST_IP>:8080
```

> **루트리스 Podman의 1024 미만 포트:** 일반 사용자 권한으로 1024 미만 포트(예: 80, 443)를 호스트에서 바인딩하려면 별도 허용이 필요하다. 운영체제 차원에서 `sudo sysctl net.ipv4.ip_unprivileged_port_start=80`을 적용하거나, 1024 이상 포트(예: 8080)를 사용하는 것이 일반적이다.

> **EXPOSE vs `-p`:** Dockerfile/Containerfile의 `EXPOSE 80`은 단지 **메타데이터 선언**일 뿐, 실제 포트를 열어주지 않는다. 실제 노출은 반드시 `-p` 또는 `-P` 옵션을 통해 호스트 포트와 매핑되어야 한다.

### 볼륨 (Volume) — 데이터 영속화

컨테이너의 파일 시스템은 **컨테이너 수명과 함께 사라진다**. `podman rm`으로 컨테이너를 지우면 그 안에서 작업한 파일도 모두 사라지므로, 데이터를 살리려면 외부 저장소에 마운트해야 한다.

```
컨테이너 내부에서 본 파일 시스템             호스트 파일 시스템
───────────────────────────────         ───────────────────────
                                                
/                                       /home/user/web/html/
├── etc/                                ├── index.html
├── usr/share/nginx/html/  ◄── 마운트 ──► ├── about.html
│   ├── index.html                     └── ...
│   ├── about.html
│   └── ...
└── ...
```

#### 1) Bind Mount — 호스트 경로 직접 연결

호스트의 **임의의 디렉터리/파일**을 컨테이너의 특정 경로에 마운트한다. 개발 중 코드 폴더를 컨테이너 안과 동기화할 때 가장 흔하게 사용된다.

```bash
# 호스트의 ~/web/html 을 컨테이너의 /usr/share/nginx/html 에 연결
mkdir -p ~/web/html
echo "<h1>Hello from host bind mount</h1>" > ~/web/html/index.html

podman run -d --name web-bind \
  -p 8080:80 \
  -v ~/web/html:/usr/share/nginx/html:Z \
  docker.io/library/nginx

curl http://127.0.0.1:8080
# <h1>Hello from host bind mount</h1>
```


| **옵션** | **의미**                                            |
| ------ | ------------------------------------------------- |
| `:Z`   | SELinux 단일 컨테이너 전용 라벨 부여 (권장, 다른 컨테이너와 공유 불가)     |
| `:z`   | SELinux 공유 라벨 (여러 컨테이너에서 공유)                      |
| `:ro`  | 읽기 전용 마운트 (`-v ~/data:/app/data:ro,Z`)            |
| `:U`   | 마운트한 디렉터리 소유권을 컨테이너 사용자에 맞게 자동 변경 (rootless에서 유용) |


> **SELinux 라벨이 빠지면:** RHEL/Rocky 처럼 SELinux가 enforcing인 시스템에서 `:Z`/`:z` 없이 bind mount를 하면, 컨테이너 안에서 `Permission denied`가 나는 일이 흔하다. 호스트에서는 정상 권한으로 보이지만, SELinux가 컨테이너 라벨이 없는 파일에 대한 접근을 차단하기 때문이다.

#### 2) Named Volume — 도구가 관리하는 저장소

Podman/Docker가 관리하는 추상화된 저장소다. 이름만 주면 도구가 적절한 위치(`graphRoot/volumes/<name>`)에 디렉터리를 만들고 관리한다. 이식성·관리 편의성이 좋아 **운영 환경에서 권장**된다.

```bash
# 명시적으로 볼륨 생성
podman volume create web-data

# 볼륨 목록
podman volume ls
# DRIVER  VOLUME NAME
# local   web-data

# 볼륨 상세 정보 (실제 저장 경로 확인 가능)
podman volume inspect web-data

# 볼륨을 컨테이너에 연결
podman run -d --name web-vol \
  -p 8081:80 \
  -v web-data:/usr/share/nginx/html:Z \
  docker.io/library/nginx

# 컨테이너 안에서 파일 추가
podman exec web-vol sh -c 'echo "<h1>From named volume</h1>" > /usr/share/nginx/html/index.html'

curl http://127.0.0.1:8081

# 컨테이너 삭제 후에도 볼륨은 살아있음
podman rm -f web-vol
podman volume ls   # web-data 여전히 존재

# 새 컨테이너에 같은 볼륨 재연결 → 데이터 유지 확인
podman run -d --name web-vol2 -p 8082:80 -v web-data:/usr/share/nginx/html:Z docker.io/library/nginx
curl http://127.0.0.1:8082
# <h1>From named volume</h1>
```

#### 3) `--volume` vs `--mount`

- `-v` / `--volume`: 짧고 자주 쓰이는 형식. `호스트경로:컨테이너경로:옵션` 콜론 구분.
- `--mount`: `key=value` 쌍으로 명시적이며 가독성 우수. 특히 bind/volume/tmpfs를 명확히 구분할 때 좋다.

```bash
# --mount로 bind mount 명시
podman run -d --name web-mnt \
  --mount type=bind,source=/home/user/web/html,target=/usr/share/nginx/html,relabel=shared \
  -p 8083:80 docker.io/library/nginx

# --mount로 named volume
podman run -d --name web-mnt2 \
  --mount type=volume,source=web-data,target=/usr/share/nginx/html \
  -p 8084:80 docker.io/library/nginx

# tmpfs (메모리 기반, 컨테이너 종료 시 휘발) — 캐시·세션 용도
podman run -d --name app \
  --mount type=tmpfs,destination=/tmp,tmpfs-size=64m \
  alpine sleep 3600
```

#### 4) 비교 요약


| **방식**           | **장점**                          | **단점**                        | **권장 상황**                  |
| ---------------- | ------------------------------- | ----------------------------- | -------------------------- |
| **Bind Mount**   | 호스트 파일을 즉시 반영 (개발 편리)           | 호스트 경로 의존, SELinux 이슈, 이식성 낮음 | 개발·로컬 디버깅, 설정 파일 주입        |
| **Named Volume** | 도구가 관리, 백업·이식 용이, SELinux 자동 처리 | 호스트에서 직접 편집은 다소 번거로움          | 운영 환경의 영속 데이터(DB, 사용자 업로드) |
| **tmpfs**        | 매우 빠름, 컨테이너 종료 시 자동 삭제          | 메모리에만 존재 — 영속화 불가, 메모리 압박     | 캐시, 임시 파일, 비밀스러운 토큰        |


### 컨테이너 네트워크

#### 1) Podman 네트워크 모드


| **모드**                | **설명**                                                               | **사용 예**                      |
| --------------------- | -------------------------------------------------------------------- | ----------------------------- |
| `bridge` (기본)         | 가상 브리지 위에 컨테이너용 가상 네트워크 생성. 호스트는 NAT로 외부와 연결되며 컨테이너끼리는 자동 DNS로 이름 통신 | 일반적인 다중 컨테이너 서비스              |
| `host`                | 컨테이너가 호스트의 네트워크 네임스페이스를 그대로 사용. 격리가 사라지지만 성능이 가장 좋음                  | 고성능 네트워크 서비스, 호스트 포트 직접 사용    |
| `none`                | 네트워크 인터페이스를 만들지 않음 (`lo`만 존재)                                        | 보안상 외부 통신을 차단하고 싶은 컨테이너       |
| `container:N`         | 다른 컨테이너 N의 네트워크 네임스페이스를 공유                                           | 사이드카(Sidecar) 패턴, 디버깅 도구 컨테이너 |
| `pasta`/`slirp4netns` | rootless 모드의 사용자 공간 네트워크 백엔드 (CAP_NET_ADMIN 없이 동작)                   | rootless Podman의 기본값          |


#### 2) 기본 네트워크 동작

`podman run`에 `--network` 옵션을 주지 않으면 자동으로 기본 브리지 네트워크(`podman` 또는 `default`)에 연결된다.

```bash
# 컨테이너의 IP 주소 확인
podman run -d --name web1 -p 8080:80 docker.io/library/nginx
podman inspect --format '{{.NetworkSettings.Networks}}' web1

# 호스트에서 컨테이너 IP 확인
podman inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web1
# 10.88.0.5

# 같은 네트워크의 다른 컨테이너끼리는 IP로 직접 통신 가능
podman run --rm -it docker.io/library/alpine ping -c 2 10.88.0.5
```

#### 3) 사용자 정의 네트워크 — 컨테이너 간 이름 통신

기본 네트워크에서는 컨테이너 이름으로 자동 DNS 해상이 잘 안 되는 경우가 있다. **사용자 정의 네트워크를 만들면 같은 네트워크에 속한 컨테이너끼리 이름으로 통신**할 수 있다.

```bash
# 1) 네트워크 생성
podman network create app-net

# 2) 네트워크 목록
podman network ls
# NETWORK ID    NAME       DRIVER
# 2f259bab93aa  podman     bridge
# 9d4f8b1c7e3a  app-net    bridge

# 3) 네트워크 상세 정보 (서브넷, 게이트웨이 확인)
podman network inspect app-net

# 4) 두 컨테이너를 동일 네트워크에 띄우기
podman run -d --name db    --network app-net docker.io/library/postgres:16 \
  -e POSTGRES_PASSWORD=secret
podman run -d --name web   --network app-net -p 8080:80 docker.io/library/nginx

# 5) web 컨테이너에서 db 컨테이너에 이름으로 접근 가능
podman exec -it web sh -c 'getent hosts db'
# 10.89.0.3       db.dns.podman
```

#### 4) 네트워크 관리 명령

```bash
# 네트워크 생성 (서브넷 지정)
podman network create --subnet 192.168.50.0/24 my-net

# 컨테이너를 실행 후 네트워크에 추가/제거
podman network connect    app-net web-extra
podman network disconnect app-net web-extra

# 네트워크 삭제 (연결된 컨테이너가 없어야 함)
podman network rm app-net

# 사용되지 않는 네트워크 일괄 삭제
podman network prune
```

#### 5) host 네트워크 모드 — 격리를 포기한 고성능 모드

```bash
# 컨테이너가 호스트의 네트워크 스택을 그대로 사용
podman run -d --name web-host --network host docker.io/library/nginx
# -p 옵션 무시됨. 이미지가 80번 포트로 띄우면 호스트의 80에 그대로 LISTEN
ss -tlnp | grep ':80'
```

> **host 모드의 트레이드오프:** 포트 매핑이 필요 없고 NAT 오버헤드가 사라져 성능이 좋지만, 컨테이너의 네트워크 격리가 사라진다. 같은 포트를 쓰는 다른 컨테이너와 충돌하기 쉽고, 컨테이너 내 프로세스가 호스트의 모든 네트워크에 접근 가능해진다는 점을 인지해야 한다.

#### 6) 종합 실습 — Web + DB 2-tier 컨테이너

```bash
# 1) 네트워크 준비
podman network create demo-net

# 2) 영속 볼륨 준비
podman volume create db-data

# 3) DB 컨테이너 (외부에는 노출하지 않음 — 내부 네트워크 통신만)
podman run -d --name demo-db \
  --network demo-net \
  -e POSTGRES_USER=app \
  -e POSTGRES_PASSWORD=appsecret \
  -e POSTGRES_DB=appdb \
  -v db-data:/var/lib/postgresql/data:Z \
  docker.io/library/postgres:16

# 4) Web 컨테이너 (외부에 8080으로 노출, 내부에서 demo-db로 DNS 통신)
podman run -d --name demo-web \
  --network demo-net \
  -p 8080:80 \
  -e DATABASE_HOST=demo-db \
  -e DATABASE_USER=app \
  -e DATABASE_PASSWORD=appsecret \
  docker.io/library/nginx

# 5) 점검
podman ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
podman exec -it demo-web sh -c 'getent hosts demo-db'
podman logs demo-db | tail -20
```

```
              ┌─────────────── 호스트 ───────────────────────┐
              │                                            │
  외부 트래픽    │   :8080  →   ┌──────────────┐              │
   ─────────► │              │  demo-web     │              │
              │              │  (nginx)      │              │
              │              └──────┬───────┘              │
              │                     │ demo-net (bridge)     │
              │                     │ DNS: demo-db          │
              │              ┌──────▼───────┐              │
              │              │  demo-db      │              │
              │              │  (postgres)   │              │
              │              └──────┬───────┘              │
              │                     │ -v db-data:/var/lib/  │
              │                     ▼                       │
              │              ┌──────────────┐              │
              │              │ named volume │              │
              │              │  (db-data)   │              │
              │              └──────────────┘              │
              └────────────────────────────────────────────┘
```

#### 7) 정리

```bash
# 종료 및 삭제
podman rm -f demo-web demo-db
podman volume rm db-data
podman network rm demo-net

# 일괄 정리 (주의: 시스템 전체 미사용 자원 제거)
podman system prune
```

---

## Ref.


| 주제                           | 문서                                                                                                                                                     |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 컨테이너 빌드·실행·관리 (RHEL 10)      | [컨테이너 빌드, 실행 및 관리 — RHEL 10](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/10/html/building_running_and_managing_containers/index) |
| Podman 명령어 레퍼런스              | [Podman Documentation — podman.io](https://docs.podman.io/en/latest/Commands.html)                                                                     |
| Podman 시작하기 (튜토리얼)           | [Podman Tutorials — github.com/containers](https://github.com/containers/podman/tree/main/docs/tutorials)                                              |
| 루트리스 Podman                  | [Basic Setup and Use of Rootless Podman](https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md)                           |
| `containers/registries.conf` | [registries.conf 매뉴얼](https://github.com/containers/image/blob/main/docs/containers-registries.conf.5.md)                                              |
| Docker 공식 문서                 | [Docker Docs — get-started, reference](https://docs.docker.com/)                                                                                       |
| OCI Image Specification      | [OCI Image Spec — opencontainers/image-spec](https://github.com/opencontainers/image-spec)                                                             |
| OCI Runtime Specification    | [OCI Runtime Spec — opencontainers/runtime-spec](https://github.com/opencontainers/runtime-spec)                                                       |
| Linux Namespaces             | [namespaces(7) — man7.org](https://man7.org/linux/man-pages/man7/namespaces.7.html)                                                                    |
| Linux cgroups                | [cgroups(7) — man7.org](https://man7.org/linux/man-pages/man7/cgroups.7.html)                                                                          |


