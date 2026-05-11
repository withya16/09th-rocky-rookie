> **주제:** 로키 리눅스 스터디 — 컨테이너 기초 (Docker & Podman)
**범위:** VM vs Container 구조적 차이, Container Runtime 아키텍처, Docker/Podman 기본 사용법, Image Lifecycle, Container Lifecycle, Port Mapping, Volume Mount, Container Networking
**자격증 연결:** CKA (100% — Container Runtime, crictl, Pod 디버깅), CKS (100% — Image Security, Seccomp, AppArmor, Rootless Container), AWS SAA (ECS/EKS 배경지식)
> 

---

# Part 1. 가상화의 두 갈래 — VM vs Container

## 1-1. Virtual Machine — Hardware Level 가상화

VM(Virtual Machine)은 **Hypervisor** 위에 **완전한 Guest OS를 포함한 독립적인 머신**을 올리는 방식입니다. 각 VM은 자체 Kernel, 자체 init System, 자체 Library를 가지며, Host OS 입장에서는 그냥 하나의 Process처럼 보입니다.

```
┌──────────────────────────────────────────────┐
│                   VM 1         VM 2          │
│              ┌───────────┐ ┌───────────┐     │
│              │  App A    │ │  App B    │     │
│              │  Libs     │ │  Libs     │     │
│              │  Guest OS │ │  Guest OS │     │
│              │  (Kernel) │ │  (Kernel) │     │
│              └───────────┘ └───────────┘     │
│           ┌──────────────────────────────┐   │
│           │        Hypervisor            │   │
│           └──────────────────────────────┘    │
│           ┌──────────────────────────────┐    │
│           │        Host OS (Kernel)      │    │
│           └──────────────────────────────┘    │
│           ┌──────────────────────────────┐    │
│           │        Hardware              │    │
│           └──────────────────────────────┘    │
└──────────────────────────────────────────────┘
```

**Hypervisor 유형:**

| Type | 동작 | 예시 |
| --- | --- | --- |
| **Type 1** (Bare-metal) | Hardware 위에 직접 동작 | VMware ESXi, KVM, Xen, **AWS Nitro** |
| **Type 2** (Hosted) | Host OS 위에 앱으로 동작 | VirtualBox, VMware Workstation |

> **AWS EC2는 Type 1 Hypervisor(Nitro)**입니다. EC2 Instance 하나가 하나의 VM입니다. 이전 스터디에서 `top`의 `st`(Steal Time)가 높으면 "물리 호스트의 CPU를 다른 VM에 빼앗기고 있다"고 했는데, 이것이 바로 Hypervisor가 CPU 시간을 분배하는 과정에서 발생하는 것입니다.
> 

**VM의 장단점:**

| 장점 | 단점 |
| --- | --- |
| 완전한 격리 (별도 Kernel) | **무겁다** — Guest OS 전체 포함 (수 GB) |
| 다른 OS 실행 가능 (Linux 위에 Windows) | 부팅이 느리다 (수십 초~분) |
| 보안 격리가 강력 | 리소스 오버헤드가 큼 (Kernel 중복) |

## 1-2. Container — OS Level 가상화

Container는 **Host OS의 Kernel을 공유**하면서, **Namespace**와 **cgroup**으로 Process를 격리하는 방식입니다. Guest OS가 없습니다. Container 안에는 앱과 그 앱이 필요로 하는 Library/파일만 들어 있습니다.

```
┌──────────────────────────────────────────────┐
│          Container 1     Container 2         │
│         ┌───────────┐  ┌───────────┐         │
│         │  App A    │  │  App B    │         │
│         │  Libs     │  │  Libs     │         │
│         └───────────┘  └───────────┘         │
│     ┌────────────────────────────────────┐   │
│     │     Container Runtime              │   │
│     │  (containerd, CRI-O, Podman)       │   │
│     └────────────────────────────────────┘   │
│     ┌────────────────────────────────────┐   │
│     │     Host OS (단일 Kernel)           │   │
│     └────────────────────────────────────┘   │
│     ┌────────────────────────────────────┐   │
│     │     Hardware                       │   │
│     └────────────────────────────────────┘   │
└──────────────────────────────────────────────┘
```

**Container를 만드는 Linux Kernel 기술 — 이전 스터디와 직결:**

| Kernel 기능 | 역할 | 이전 스터디 연결 |
| --- | --- | --- |
| **Namespace** | Process 격리 — 각 Container가 독립된 PID, Network, Mount, User, UTS, IPC를 가짐 | Process 계층 구조 (PID 1) |
| **cgroup** | Resource 제한 — CPU, Memory, I/O 사용량 제한 | **2부. cgroup** — `CPUQuota`, `MemoryMax` |
| **Union Filesystem** | Image Layer 쌓기 — 읽기전용 Base + 쓰기 Layer | Filesystem 관리 |
| **seccomp** | System Call 필터링 — 위험한 System Call 차단 | CKS Security |
| **AppArmor/SELinux** | 파일/네트워크 접근 제어 | SELinux 스터디 |

**이 표가 핵심입니다.** Container는 새로운 기술이 아니라, 이전 스터디에서 배운 **Linux Kernel 기능들의 조합**입니다. Docker나 Podman은 이 Kernel 기능들을 편하게 쓸 수 있게 해주는 도구일 뿐입니다.

## 1-3. Namespace — 격리의 핵심

Namespace는 Process가 볼 수 있는 **시스템 자원의 범위를 제한**합니다. 각 Container는 자신만의 Namespace 세트를 가집니다.

| Namespace | 격리 대상 | 효과 |
| --- | --- | --- |
| **PID** | Process ID | Container 안의 Process는 PID 1부터 시작. Host의 다른 Process를 볼 수 없음 |
| **Network** | Network Interface, IP, Port, Routing Table | Container마다 독립된 네트워크 스택. 같은 Port를 여러 Container에서 사용 가능 |
| **Mount** | File System Mount Point | Container마다 독립된 Root Filesystem. Host 파일에 접근 불가 (Volume으로 명시적 연결만 가능) |
| **UTS** | Hostname, Domain Name | Container마다 다른 Hostname 설정 가능 |
| **User** | UID/GID 매핑 | Container 안의 root(UID 0)를 Host의 일반 사용자로 매핑 (Rootless Container) |
| **IPC** | Inter-Process Communication | Shared Memory, Semaphore 등 격리 |
| **cgroup** | cgroup 계층 뷰 | Container가 자신의 cgroup만 볼 수 있음 |

```bash
# Host에서 Container의 Namespace 확인
sudo lsns

# 특정 Process의 Namespace
sudo ls -la /proc/<PID>/ns/

# Container 안에서 PID 확인 — PID 1부터 시작
# (Host에서는 전혀 다른 PID를 가짐)

mr8356@mr8356:~$ sudo lsns
[sudo] password for mr8356:
        NS TYPE   NPROCS   PID USER    COMMAND
4026531834 time      224     1 root    /usr/lib/systemd/systemd --switched-root --system --deserialize=43 rhgb
4026531835 cgroup    224     1 root    /usr/lib/systemd/systemd --switched-root --system --deserialize=43 rhgb
4026531836 pid       224     1 root    /usr/lib/systemd/systemd --switched-root --system --deserialize=43 rhgb
4026531837 user      223     1 root    /usr/lib/systemd/systemd --switched-root --system --deserialize=43 rhgb
4026531838 uts       214     1 root    /usr/lib/systemd/systemd --switched-root --system --deserialize=43 rhgb
4026531839 ipc       224     1 root    /usr/lib/systemd/systemd --switched-root --system --deserialize=43 rhgb
4026531840 net       222     1 root    /usr/lib/systemd/systemd --switched-root --system --deserialize=43 rhgb
4026531841 mnt       208     1 root    /usr/lib/systemd/systemd --switched-root --system --deserialize=43 rhgb
4026532468 mnt         1   957 root    ├─/usr/lib/systemd/systemd-udevd
4026532582 mnt         4   928 root    ├─/usr/lib/systemd/systemd-userdbd
4026532583 uts         4   928 root    ├─/usr/lib/systemd/systemd-userdbd
4026532584 uts         1   957 root    ├─/usr/lib/systemd/systemd-udevd
4026532636 mnt         2  1066 dbus    ├─/usr/bin/dbus-broker-launch --scope system --audit
4026532637 mnt         1  1105 chrony  ├─/usr/sbin/chronyd -F 2
4026532638 mnt         1  1084 root    ├─/usr/bin/python3 -sP /usr/sbin/firewalld --nofork --nopid
4026532639 mnt         1  1172 root    ├─/usr/sbin/NetworkManager --no-daemon
4026532640 uts         1  1105 chrony  ├─/usr/sbin/chronyd -F 2
4026532641 net         1  1085 root    ├─/usr/sbin/irqbalance
4026532696 mnt         1  1085 root    ├─/usr/sbin/irqbalance
4026532700 mnt         1  1090 root    ├─/usr/lib/systemd/systemd-logind
4026532701 uts         1  1084 root    ├─/usr/bin/python3 -sP /usr/sbin/firewalld --nofork --nopid
4026532702 uts         1  1085 root    ├─/usr/sbin/irqbalance
4026532703 user        1  1085 root    ├─/usr/sbin/irqbalance
4026532704 uts         1  1090 root    ├─/usr/lib/systemd/systemd-logind
4026532706 net         1  1159 polkitd ├─/usr/lib/polkit-1/polkitd --no-debug --log-level=err
4026532761 mnt         1  1159 polkitd ├─/usr/lib/polkit-1/polkitd --no-debug --log-level=err
4026532762 uts         1  1159 polkitd ├─/usr/lib/polkit-1/polkitd --no-debug --log-level=err
4026532766 mnt         1  1913 root    ├─/usr/libexec/fwupd/fwupd
4026532845 mnt         1  1344 root    └─/usr/sbin/rsyslogd -n
4026531862 mnt         1    28 root    kdevtmpfs
```

> **CKS 직접 연결:** CKS에서 "특정 Pod의 Process가 Host의 PID Namespace를 공유하도록 설정하시오"라는 문제가 나옵니다.
> 
> 
> ```yaml
> spec:
>   hostPID: true    # Host의 PID Namespace 공유 → Container에서 Host의 모든 Process 보임
>   hostNetwork: true # Host의 Network Namespace 공유 → Container가 Host IP 사용
> ```
> 
> 이 설정이 **보안상 위험한 이유**: Container가 Host의 Process를 볼 수 있으면 `kill`로 Host Process를 죽이거나, `/proc/<PID>/environ`에서 비밀 정보를 읽을 수 있습니다. CKS에서는 이런 설정을 **Pod Security Standards (PSS)**로 차단하는 문제가 나옵니다.
> 

## 1-4. VM vs Container 비교 정리

| 항목 | VM | Container |
| --- | --- | --- |
| 가상화 수준 | Hardware Level | **OS Level** |
| Guest OS | 있음 (전체 Kernel) | **없음** (Host Kernel 공유) |
| 크기 | 수 GB | **수 MB ~ 수백 MB** |
| 부팅 시간 | 수십 초 ~ 분 | **밀리초 ~ 초** |
| 격리 강도 | 강함 (별도 Kernel) | 상대적 약함 (Kernel 공유) |
| 밀도 | 호스트당 수십 개 | 호스트당 **수백 ~ 수천 개** |
| 리소스 오버헤드 | 큼 | **작음** |
| 사용 사례 | 다른 OS 실행, 강한 격리 필요 | 마이크로서비스, CI/CD, 일관된 배포 환경 |

> **K8s는 Container Orchestrator입니다.** K8s가 관리하는 단위인 **Pod은 하나 이상의 Container를 묶은 것**입니다. K8s는 VM을 관리하지 않습니다 (VM 관리는 KubeVirt라는 별도 프로젝트). K8s를 이해하려면 Container의 원리를 먼저 이해해야 합니다.
> 

---

# Part 2. Container Runtime 아키텍처 — Docker, containerd, CRI-O, Podman

## 2-1. Container Runtime의 계층 구조

"Docker로 Container를 실행한다"고 할 때, 실제로는 여러 Layer의 소프트웨어가 협력합니다.

```
┌─────────────────────────────────────────────┐
│  사용자 도구 (High-level)                      │
│  docker CLI / podman CLI / nerdctl          │
├─────────────────────────────────────────────┤
│  Container Engine (High-level Runtime)      │
│  dockerd / podman engine                    │
├─────────────────────────────────────────────┤
│  Container Runtime (Low-level Runtime)      │
│  containerd / CRI-O                         │
├─────────────────────────────────────────────┤
│  OCI Runtime                                │
│  runc (실제로 Container를 생성하는 바이너리)       │
├─────────────────────────────────────────────┤
│  Linux Kernel (Namespace + cgroup)           │
└─────────────────────────────────────────────┘
```

| Layer | 역할 | 구현체 |
| --- | --- | --- |
| **OCI Runtime** | Namespace/cgroup을 설정하고 Container Process를 생성 | `runc` (표준), `crun`, `kata-containers` |
| **High-level Runtime** | Image Pull, Storage 관리, Networking, Container Lifecycle | `containerd`, `CRI-O` |
| **Container Engine** | 사용자 인터페이스 (CLI, API) | `docker`, `podman`, `nerdctl` |

**OCI (Open Container Initiative):**

OCI는 Container의 **표준 규격**을 정의합니다:

- **OCI Image Spec**: Container Image의 포맷 표준
- **OCI Runtime Spec**: Container 실행 방법 표준

이 표준 덕분에 Docker로 만든 Image를 Podman으로 실행하거나, containerd로 실행할 수 있습니다.

## 2-2. Docker vs Podman — 핵심 차이

| 항목 | Docker | Podman |
| --- | --- | --- |
| 아키텍처 | **Daemon 기반** (`dockerd`가 항상 실행) | **Daemonless** (명령 실행 시에만 Process 생성) |
| Root 권한 | 기본적으로 root 필요 (`docker` 그룹) | **Rootless 모드 기본** — 일반 사용자로 실행 |
| systemd 통합 | 제한적 | **네이티브 systemd 통합** (`podman generate systemd`) |
| K8s 호환 | 간접적 (containerd를 통해) | **Pod 개념 네이티브 지원** (`podman pod`) |
| Compose | docker-compose | **podman-compose** 또는 `podman play kube` |
| CGroup | v1/v2 | **cgroup v2 네이티브** |
| RHEL 지원 | RHEL 8부터 공식 제외 | **RHEL 공식 Container 도구** |

**Daemon vs Daemonless가 왜 중요한가:**

Docker는 `dockerd` Daemon이 항상 떠 있어야 합니다. 이 Daemon이 죽으면 **모든 Container가 관리 불능**이 됩니다. 또한 Daemon이 root로 실행되므로 Docker API에 접근할 수 있는 사용자는 **사실상 root 권한**을 가집니다 (Container 안에서 Host의 / Filesystem을 Mount할 수 있으므로).

Podman은 Daemon이 없습니다. 각 `podman` 명령이 직접 Container를 생성하고 관리합니다. 보안상 훨씬 안전하며, 이것이 RHEL이 Docker 대신 Podman을 공식 채택한 이유입니다.

> **CKA/CKS 연결 — K8s의 Container Runtime:**
K8s 1.24부터 **dockershim이 제거**되었습니다. K8s는 더 이상 Docker를 직접 지원하지 않습니다. K8s가 사용하는 Container Runtime은 **containerd** 또는 **CRI-O**입니다.
> 
> 
> ```bash
> # K8s Node에서 사용 중인 Container Runtime 확인
> kubectl get nodes -o wide
> # CONTAINER-RUNTIME 컬럼: containerd://1.7.x 또는 cri-o://1.28.x
> 
> # Node에 SSH하여 Runtime 확인
> sudo crictl info | grep -i runtime
> ```
> 
> **CKA 시험에서 Docker 명령어는 사용하지 않습니다.** `crictl`을 사용합니다. 하지만 Docker/Podman의 개념(Image, Container, Volume, Network)은 동일하므로 여기서 배우는 것이 그대로 적용됩니다.
> 

## 2-3. Podman 설치 (Rocky Linux / RHEL 10)

```bash
# Podman은 RHEL/Rocky에 기본 설치되어 있음
podman --version

# 없다면 설치
sudo dnf install podman -y

# Container 관련 추가 도구
sudo dnf install podman-compose skopeo buildah -y
# skopeo: Image 검사/복사 도구 (pull 없이 Image 정보 확인)
# buildah: Dockerfile 없이 Image 빌드 가능
```

---

# Part 3. Container Image — 불변의 실행 환경

## 3-1. Image란 무엇인가

Image는 Container를 만들기 위한 **읽기전용 Template**입니다. 앱 코드, 런타임, 라이브러리, 환경 변수, 설정 파일 등 Container가 실행되는 데 필요한 모든 것이 포함됩니다.

**핵심: Image는 불변(Immutable)입니다.** 한번 만들어지면 변경할 수 없습니다. Container를 실행하면 Image 위에 **쓰기 가능한 Layer**를 추가하여, Container 안에서의 파일 변경은 이 Layer에 기록됩니다. Container를 삭제하면 이 쓰기 Layer도 사라집니다.

## 3-2. Image Layer 구조 — Union Filesystem

Image는 **여러 Layer의 스택**으로 구성됩니다. 각 Layer는 이전 Layer 위에 파일 변경사항(추가/수정/삭제)만 기록합니다.

```
┌───────────────────────────────┐
│  Writable Layer (Container)   │ ← Container 실행 중 변경사항 (삭제 시 소멸)
├───────────────────────────────┤
│  Layer 4: COPY app.py /app/   │ ← 앱 코드 복사
├───────────────────────────────┤
│  Layer 3: RUN pip install...  │ ← 의존성 설치
├───────────────────────────────┤
│  Layer 2: RUN dnf install...  │ ← 패키지 설치
├───────────────────────────────┤
│  Layer 1: Base OS (rocky:9)   │ ← Base Image
└───────────────────────────────┘
       모든 Layer는 읽기전용
```

**Layer가 왜 중요한가:**

- **공유:** 같은 Base Image를 사용하는 100개의 Container가 있어도, Base Layer는 디스크에 **1번만** 저장됩니다. 디스크 절약 + Pull 속도 향상.
- **캐싱:** Dockerfile에서 변경된 줄부터만 재빌드합니다. 이전 Layer는 캐시에서 재사용.
- **불변성:** Image Layer는 SHA256 Hash로 식별됩니다. 1 bit라도 다르면 다른 Layer. 이것이 Container 배포의 **재현성(Reproducibility)**을 보장합니다.

## 3-3. Image 이름 구조

```
registry.example.com/namespace/repository:tag@sha256:digest

예시:
docker.io/library/nginx:1.25-alpine
       │       │      │    │
       │       │      │    └─ Tag (버전/변형)
       │       │      └─ Repository (이미지 이름)
       │       └─ Namespace (사용자/조직)
       └─ Registry (저장소 서버)
```

| 구성요소 | 설명 | 기본값 |
| --- | --- | --- |
| Registry | Image가 저장된 서버 | `docker.io` (Docker Hub) |
| Namespace | 사용자/조직. `library`는 공식 이미지 | `library` |
| Repository | Image 이름 | (필수) |
| Tag | 버전/변형 식별자 | `latest` (**비권장!**) |
| Digest | SHA256 Hash. Tag보다 정확한 식별 | (선택) |

> **CKS 직접 연결 — Image Security:**
CKS에서 Image 관련 보안 문제가 자주 출제됩니다:
> 
> 1. **`latest` Tag 금지:** `latest`는 어떤 버전인지 알 수 없고, 같은 Tag인데 내용이 바뀔 수 있습니다. Production에서는 반드시 **구체적 Tag** (`nginx:1.25.3`) 또는 **Digest** (`nginx@sha256:abc123...`)를 사용합니다.
> 2. **신뢰할 수 있는 Registry만 사용:** OPA/Gatekeeper 또는 Kyverno로 특정 Registry(예: `company.registry.io/`)의 Image만 허용하는 정책을 강제합니다.
> 3. **Image 취약점 스캔:** `trivy image nginx:1.25`로 CVE 스캔.
> 
> ```yaml
> # CKS: ImagePolicyWebhook 또는 OPA로 Image Registry 제한
> # Pod Spec에서 Image Pull Policy
> spec:
>   containers:
>   - name: app
>     image: company.registry.io/app:v1.2.3   # 구체적 Tag
>     imagePullPolicy: Always                   # 항상 Registry에서 최신 확인
> ```
> 

## 3-4. Image 검색·다운로드·관리

```bash
# === Image 검색 ===
podman search nginx                     # Registry에서 검색
podman search --list-tags docker.io/library/nginx  # 사용 가능한 Tag 목록

# skopeo로 Pull 없이 Image 정보 확인
skopeo inspect docker://docker.io/library/nginx:latest

# === Image 다운로드 (Pull) ===
podman pull nginx                       # docker.io/library/nginx:latest
podman pull nginx:1.25-alpine           # 특정 Tag
podman pull quay.io/prometheus/prometheus:v2.48.0  # 다른 Registry

# === 로컬 Image 목록 ===
podman images
podman image ls

# === Image 상세 정보 ===
podman inspect nginx:1.25-alpine
podman inspect nginx:1.25-alpine | jq '.[0].Config.ExposedPorts'

# === Image Layer 확인 ===
podman history nginx:1.25-alpine

# === Image 삭제 ===
podman rmi nginx:1.25-alpine            # 특정 Image 삭제
podman rmi -f nginx                     # 강제 삭제 (사용 중이어도)
podman image prune                      # 미사용(dangling) Image 정리
podman image prune -a                   # 모든 미사용 Image 정리

# === Image 저장/로드 (오프라인 전송) ===
podman save -o nginx.tar nginx:1.25-alpine    # tar로 저장
podman load -i nginx.tar                       # tar에서 로드

# === Image Tag 변경 ===
podman tag nginx:1.25-alpine my-registry.io/nginx:v1

mr8356@mr8356:~$ podman image ls
REPOSITORY                        TAG         IMAGE ID      CREATED      SIZE
quay.io/freshtracks.io/avalanche  latest      d55fbbdec37c  4 years ago  15.2 MB
mr8356@mr8356:~$ podman pull nginx:1.25-alpine
Resolved "nginx" as an alias (/etc/containers/registries.conf.d/000-shortnames.conf)
Trying to pull docker.io/library/nginx:1.25-alpine...
Getting image source signatures
Copying blob bca4290a9639 done   |
Copying blob 182e691fb2cc done   |
Copying blob ddf9db5a05cb done   |
Copying blob a83296a673ce done   |
Copying blob b4f3127eb622 done   |
Copying blob 166b80e00f74 done   |
Copying blob 98ff282c4466 done   |
Copying blob 4f6b4e3940df done   |
Copying config 9d6767b714 done   |
Writing manifest to image destination
9d6767b714bf1ecd2cdab75b590f2c572ac33743c7786ef5d619f7b088dbe0bb
mr8356@mr8356:~$ podman image ls
REPOSITORY                        TAG          IMAGE ID      CREATED        SIZE
docker.io/library/nginx           1.25-alpine  9d6767b714bf  24 months ago  51.5 MB
quay.io/freshtracks.io/avalanche  latest       d55fbbdec37c  4 years ago    15.2 MB
```

> **CKA 연결 — crictl로 Image 관리:**
K8s Node에서는 `podman`이나 `docker` 대신 **`crictl`**을 사용합니다.
> 
> 
> ```bash
> # Node에서 Image 목록
> sudo crictl images
> 
> # 미사용 Image 정리 (DiskPressure 대응)
> sudo crictl rmi --prune
> 
> # Image Pull (CRI를 통해)
> sudo crictl pull nginx:1.25
> ```
> 

---

# Part 4. Container Lifecycle — 실행·중지·접속

## 4-1. Container 실행

```bash
# === 기본 실행 ===
podman run nginx                        # Foreground 실행 (Ctrl+C로 종료)
podman run -d nginx                     # Detached (Background) 실행
podman run -d --name my-nginx nginx     # 이름 지정

# === 실행 옵션들 ===
podman run -d \
  --name web \
  -p 8080:80 \                          # Port Mapping: Host 8080 → Container 80
  -v /host/html:/usr/share/nginx/html \  # Volume Mount
  -e NGINX_HOST=example.com \           # 환경 변수
  --restart=always \                    # 자동 재시작
  nginx:1.25-alpine

# === 일회성 실행 (종료 시 Container 자동 삭제) ===
podman run --rm -it alpine sh           # 대화형 Shell 진입
podman run --rm alpine cat /etc/os-release  # 명령 실행 후 삭제

# === 리소스 제한 (cgroup 직접 설정) ===
podman run -d \
  --name limited-app \
  --memory=512m \                       # MemoryMax=512M (cgroup)
  --cpus=0.5 \                          # CPUQuota=50% (cgroup)
  nginx
  
mr8356@mr8356:~$ podman run nginx
Resolved "nginx" as an alias (/etc/containers/registries.conf.d/000-shortnames.conf)
Trying to pull docker.io/library/nginx:latest...
Getting image source signatures
Copying blob 7ebd5beed79e done   |
Copying blob 405d5794f35e done   |
Copying blob 1b39715bb702 done   |
Copying blob b420423b8e46 done   |
Copying blob 0c0e4c019008 done   |
Copying blob e4fb5f1cd4d4 done   |
Copying blob 91409cde2cec done   |
Copying config 9e4696c649 done   |
Writing manifest to image destination
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2026/04/27 09:02:20 [notice] 1#1: using the "epoll" event method
2026/04/27 09:02:20 [notice] 1#1: nginx/1.29.8
2026/04/27 09:02:20 [notice] 1#1: built by gcc 14.2.0 (Debian 14.2.0-19)
2026/04/27 09:02:20 [notice] 1#1: OS: Linux 6.12.0-124.8.1.el10_1.aarch64
2026/04/27 09:02:20 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 524288:524288
2026/04/27 09:02:20 [notice] 1#1: start worker processes
2026/04/27 09:02:20 [notice] 1#1: start worker process 24
2026/04/27 09:02:20 [notice] 1#1: start worker process 25

^C2026/04/27 09:02:26 [notice] 1#1: signal 2 (SIGINT) received, exiting
2026/04/27 09:02:26 [notice] 25#25: exiting
2026/04/27 09:02:26 [notice] 24#24: exiting
2026/04/27 09:02:26 [notice] 24#24: exit
2026/04/27 09:02:26 [notice] 25#25: exit
2026/04/27 09:02:26 [notice] 1#1: signal 14 (SIGALRM) received
2026/04/27 09:02:26 [notice] 1#1: signal 17 (SIGCHLD) received from 25
2026/04/27 09:02:26 [notice] 1#1: worker process 24 exited with code 0
2026/04/27 09:02:26 [notice] 1#1: worker process 25 exited with code 0
2026/04/27 09:02:26 [notice] 1#1: exit
mr8356@mr8356:~$ podman ps
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
mr8356@mr8356:~$

mr8356@mr8356:~$ podman ps
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
mr8356@mr8356:~$ podman run -d nginx
842167d25b2b2dfce6e952d6ea940918358b33f498009e7b1abc8be7ab96a7a4
mr8356@mr8356:~$ podman ps
CONTAINER ID  IMAGE                           COMMAND               CREATED        STATUS        PORTS       NAMES
842167d25b2b  docker.io/library/nginx:latest  nginx -g daemon o...  5 seconds ago  Up 5 seconds  80/tcp      brave_kare
mr8356@mr8356:~$ podman stop 842167d25b2b
842167d25b2b
mr8356@mr8356:~$ podman ps
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
mr8356@mr8356:~$
```

**`podman run`이 내부적으로 하는 일:**

```
1. Image가 로컬에 없으면 Registry에서 Pull
2. Image Layer들을 Union Filesystem으로 쌓기
3. 쓰기 Layer 추가
4. Namespace 생성 (PID, Network, Mount, ...)
5. cgroup 생성 및 Resource Limit 적용
6. Container 내 Process 시작 (Image의 CMD/ENTRYPOINT)
```

이것이 Part 1에서 배운 Namespace + cgroup이 실제로 적용되는 순간입니다.

## 4-2. Container 상태 확인

```bash
# 실행 중인 Container 목록
podman ps

# 모든 Container (중지된 것 포함)
podman ps -a

# 상세 정보
podman inspect my-nginx

# 로그 확인
podman logs my-nginx
podman logs -f my-nginx                 # 실시간 (tail -f)
podman logs --since "10m" my-nginx      # 최근 10분
podman logs --tail 50 my-nginx          # 마지막 50줄

# 리소스 사용량 (실시간)
podman stats
podman stats my-nginx

# Container 내부 Process 확인
podman top my-nginx
```

> **CKA 연결 — kubectl과 1:1 대응:**
> 
> 
> 
> | podman 명령 | kubectl 대응 | 용도 |
> | --- | --- | --- |
> | `podman ps` | `kubectl get pods` | 실행 중인 목록 |
> | `podman logs <name>` | `kubectl logs <pod>` | 로그 확인 |
> | `podman exec -it <name> sh` | `kubectl exec -it <pod> -- sh` | Container 접속 |
> | `podman inspect <name>` | `kubectl describe pod <pod>` | 상세 정보 |
> | `podman stats` | `kubectl top pods` | 리소스 사용량 |
> | `podman run --rm -it` | `kubectl run --rm -it --image=` | 임시 Container |

## 4-3. Container 접속 (exec)

```bash
# 실행 중인 Container에 Shell 접속
podman exec -it my-nginx /bin/bash
podman exec -it my-nginx /bin/sh       # Alpine 등 bash 없는 Image

# 특정 명령어만 실행
podman exec my-nginx cat /etc/nginx/nginx.conf
podman exec my-nginx nginx -t          # 설정 테스트

# 다른 사용자로 실행
podman exec -it --user root my-nginx bash
```

> **CKA 직접 연결 — Pod 디버깅의 핵심:**
> 
> 
> ```bash
> # CKA 단골: "Pod에 접속하여 설정 파일을 확인하시오"
> kubectl exec -it my-pod -- /bin/sh
> 
> # Multi-container Pod에서 특정 Container에 접속
> kubectl exec -it my-pod -c sidecar-container -- /bin/sh
> 
> # 임시 디버깅 Container 추가 (Ephemeral Container — CKA/CKS)
> kubectl debug -it my-pod --image=busybox --target=app-container
> ```
> 

## 4-4. Container 정지·재시작·삭제

```bash
# 정지 (SIGTERM → 대기 → SIGKILL)
podman stop my-nginx
podman stop -t 30 my-nginx             # 30초 대기 후 SIGKILL

# 강제 종료 (SIGKILL 즉시)
podman kill my-nginx

# 재시작
podman restart my-nginx

# 삭제 (정지된 Container만)
podman rm my-nginx

# 강제 삭제 (실행 중이어도)
podman rm -f my-nginx

# 모든 Container 삭제
podman rm -f $(podman ps -aq)

# 정지된 Container 일괄 정리
podman container prune
```

**SIGTERM vs SIGKILL (복습):** `podman stop`은 먼저 SIGTERM을 보내 Graceful Shutdown 기회를 주고, Timeout 후 SIGKILL을 보냅니다. 이것이 K8s의 `terminationGracePeriodSeconds`와 동일한 원리입니다.

---

# Part 5. Port Mapping — Container와 외부를 연결

## 5-1. Container Network의 기본 개념

Container는 **자체 Network Namespace**를 가지므로, 기본적으로 외부에서 Container 내부의 서비스에 접근할 수 없습니다. **Port Mapping**으로 Host의 Port를 Container의 Port로 연결해야 합니다.

```
외부 클라이언트 → Host IP:8080 → Container IP:80 (nginx)
```

```bash
# Host Port 8080 → Container Port 80
podman run -d -p 8080:80 --name web nginx

# 여러 Port Mapping
podman run -d -p 8080:80 -p 8443:443 nginx

# Host의 특정 IP에만 바인딩
podman run -d -p 127.0.0.1:8080:80 nginx    # localhost에서만 접근 가능

# 랜덤 Host Port 할당
podman run -d -p 80 nginx                    # Host의 랜덤 Port → Container 80
podman port <container>                       # 할당된 Port 확인

# UDP Port
podman run -d -p 5353:53/udp dns-server

# 확인
curl http://localhost:8080
ss -tlnp | grep 8080
```

> **K8s 연결 — Service와 Port:**
K8s에서는 Port Mapping 대신 **Service** 리소스가 이 역할을 합니다.
> 
> 
> 
> | Container 방식 | K8s 방식 | 설명 |
> | --- | --- | --- |
> | `-p 8080:80` | `Service(NodePort)` | 외부 접근용 |
> | Container IP:Port | `ClusterIP:Port` | Cluster 내부 접근 |
> | - | `LoadBalancer` | Cloud LB 연동 (AWS ALB/NLB) |
> 
> ```yaml
> # CKA: Service 생성
> apiVersion: v1
> kind: Service
> metadata:
>   name: web-svc
> spec:
>   type: NodePort
>   selector:
>     app: web
>   ports:
>   - port: 80          # Service(ClusterIP) Port
>     targetPort: 80     # Container Port
>     nodePort: 30080    # Node Port (외부 접근)
> ```
> 

---

# Part 6. Volume — Container의 데이터 영속성

## 6-1. 왜 Volume이 필요한가

Container를 삭제하면 Container 안의 모든 데이터가 **사라집니다.** Container의 Writable Layer는 Container와 함께 삭제되기 때문입니다.

DB 데이터, 로그, 설정 파일 등 Container가 삭제되어도 유지해야 하는 데이터는 **Volume**에 저장합니다.

## 6-2. Volume의 3가지 유형

| 유형 | 문법 | 데이터 위치 | 사용 시점 |
| --- | --- | --- | --- |
| **Named Volume** | `-v mydata:/data` | Podman이 관리하는 디렉터리 | DB 데이터, 앱 상태 등 영구 저장 |
| **Bind Mount** | `-v /host/path:/container/path` | Host의 지정 경로 | 설정 파일, 소스 코드 (개발 시) |
| **tmpfs** | `--tmpfs /tmp` | RAM (메모리) | 보안 민감 임시 파일 |

```bash
# === Named Volume ===
podman volume create mydata
podman run -d -v mydata:/var/lib/postgresql/data postgres:16

# Volume 목록
podman volume ls

# Volume 상세 (실제 경로 확인)
podman volume inspect mydata

# Volume 삭제
podman volume rm mydata
podman volume prune                    # 미사용 Volume 정리

# === Bind Mount ===
# Host의 디렉터리를 Container에 직접 연결
mkdir -p /opt/myapp/html
echo "<h1>Hello</h1>" > /opt/myapp/html/index.html
podman run -d -p 8080:80 -v /opt/myapp/html:/usr/share/nginx/html:ro nginx
# :ro = Read-Only (Container에서 수정 불가)
# :rw = Read-Write (기본값)

# === 읽기전용 Volume (보안) ===
podman run -d \
  -v /opt/config:/etc/myapp:ro \        # 설정은 읽기전용
  -v appdata:/var/lib/myapp:rw \        # 데이터는 읽기쓰기
  my-app
```

> **CKA/CKS 직접 연결 — PV/PVC, emptyDir, hostPath:**
> 
> 
> K8s에서 Volume은 **Pod Spec**에 선언합니다:
> 
> | Container Volume | K8s Volume Type | 용도 |
> | --- | --- | --- |
> | Named Volume | **PersistentVolume (PV) + PVC** | DB 데이터, 영구 저장 (EBS 등 연결) |
> | Bind Mount | **hostPath** | Node의 특정 경로 연결 (**CKS에서 보안 위험 지적**) |
> | tmpfs | **emptyDir (medium: Memory)** | RAM 기반 임시 저장 |
> | - | **emptyDir** | Pod 내 Container 간 파일 공유 |
> | - | **ConfigMap / Secret** | 설정 파일, 비밀 정보 주입 |
> 
> ```yaml
> # CKA 단골: PVC 생성
> apiVersion: v1
> kind: PersistentVolumeClaim
> metadata:
>   name: app-data
> spec:
>   accessModes: [ReadWriteOnce]
>   resources:
>     requests:
>       storage: 10Gi
>   storageClassName: gp3   # AWS EBS gp3
> ---
> # Pod에서 PVC 사용
> spec:
>   containers:
>   - name: app
>     volumeMounts:
>     - name: data
>       mountPath: /var/lib/app
>   volumes:
>   - name: data
>     persistentVolumeClaim:
>       claimName: app-data
> ```
> 
> **CKS:** `hostPath` Volume은 Container가 **Host Filesystem에 직접 접근**할 수 있으므로 보안 위험. Pod Security Standards에서 `hostPath`를 제한하는 것이 CKS 단골 문제.
> 

---

# Part 7. Container Networking

컴퓨터 네트워크에서 배우는 브릿지는 **L2(데이터 링크 계층)** 장비로, MAC 주소를 학습하고 포트 간에 프레임을 전달하는 역할을 합니다.

Podman이나 Docker에서 생성되는 `cni-podman0` 같은 가상 브릿지도 이와 똑같이 동작합니다.

- **컴퓨터 네트워크의 브릿지:** 물리적인 허브나 PC들을 연결.
- **Podman의 브릿지:** 호스트 OS 내부에서 여러 개의 컨테이너(가상 NIC)를 하나의 네트워크 세그먼트로 묶어주는 **가상 스위치** 역할.

브릿지는 L2(데이터 링크 계층) 장비로서 FDB(Forwarding Database)라는 표를 관리합니다.

커널이 NAT를 마친 패킷을 브릿지(`cni-podman0`)로 던지면, 브릿지는 다음과 같은 일을 수행합니다.

1. **ARP(Address Resolution Protocol) 수행:** "IP가 `10.88.0.2`인 녀석 누구야? MAC 주소 좀 대봐!"라고 브릿지에 연결된 모든 컨테이너에게 물어봅니다.
2. **MAC 주소 학습:** 컨테이너가 "나야! 내 MAC 주소는 `aa:bb:cc...`야"라고 응답하면, 브릿지는 'aa:bb... 주소는 2번 포트(veth)에 연결되어 있음'이라는 정보를 자신의 FDB에 기록합니다.
3. **L2 스위칭:** 이제 브릿지는 패킷을 정확히 해당 컨테이너와 연결된 **veth(가상 이더넷 케이블)** 쪽으로만 쏴줍니다.

## 7-1. Podman Network Mode

| Mode | 동작 | 사용 시점 |
| --- | --- | --- |
| **bridge** (기본) | 가상 Bridge 네트워크. Container끼리 통신 가능. NAT로 외부 접근 | 대부분의 경우 |
| **host** | Host의 Network Namespace 공유. Port Mapping 불필요 | 최대 네트워크 성능 필요 시 |
| **none** | 네트워크 없음. Loopback만 | 보안 격리 |
| **container:<id>** | 다른 Container의 Network Namespace 공유 | Sidecar 패턴 |
- **브릿지(Bridge)의 역할 (L2):** 컨테이너 A와 컨테이너 B가 서로 통신할 때, 혹은 컨테이너가 호스트와 통신할 때 단순히 길을 열어주는 **통로** 역할만 합니다. 브릿지 자체는 IP나 포트를 변경할 능력이 없습니다.
- **NAT/포트 포워딩의 역할 (L3/L4):** `p 8080:80` 같은 규칙을 처리하는 것은 브릿지가 아니라 호스트 OS 커널의 Netfilter(iptables/nftables)입니다.

```bash
# Bridge 네트워크 (기본)
podman run -d --name web -p 8080:80 nginx

# Host 네트워크 (Container가 Host IP 직접 사용)
podman run -d --network host nginx
# Port Mapping 불필요 — Container의 80이 곧 Host의 80
# 주의: Port 충돌 가능

# 네트워크 없음
podman run -d --network none nginx

# Container 간 네트워크 공유 (K8s Pod의 원리)
podman run -d --name app1 nginx
podman run -d --name app2 --network container:app1 busybox sleep infinity
# app1과 app2가 같은 Network Namespace → localhost로 통신 가능
```

> **K8s Pod의 네트워크 원리가 바로 이것:**
K8s Pod 내의 모든 Container는 **같은 Network Namespace를 공유**합니다. 이것이 위의 `--network container:<id>`와 동일한 원리입니다. 그래서 같은 Pod의 Container들은 `localhost`로 서로 통신합니다.
> 
> 
> ```
> Pod (하나의 Network Namespace)
> ├── Container A (app)     → localhost:8080
> ├── Container B (sidecar) → localhost:9090
> └── pause container       ← Network Namespace를 유지하는 인프라 Container
> ```
> 

## 7-2. 사용자 정의 네트워크 — Container 간 DNS

기본 Bridge에서는 Container 이름으로 통신할 수 없습니다. **사용자 정의 네트워크**를 만들면 Container 이름이 DNS로 자동 등록됩니다.

![image.png](attachment:ab094de6-14a6-437d-911f-feb107009335:image.png)

```bash
# 사용자 정의 네트워크 생성
podman network create mynet

# 네트워크 목록
podman network ls

# 네트워크 상세
podman network inspect mynet

# 이 네트워크에서 Container 실행
podman run -d --name db --network mynet -e POSTGRES_PASSWORD=secret postgres:16
podman run -d --name app --network mynet my-app

# app Container에서 db Container에 이름으로 접근 가능!
podman exec app ping db    # "db"가 DNS로 해석됨

# 네트워크 삭제
podman network rm mynet

mr8356@mr8356:~$ podman network create mynet
mynet
mr8356@mr8356:~$ podman network ls
NETWORK ID    NAME        DRIVER
8162cb62e367  mynet       bridge
2f259bab93aa  podman      bridge
mr8356@mr8356:~$ podman network inspect mynet
[
     {
          "name": "mynet",
          "id": "8162cb62e367e89ad6cf098cd87dda5999517824298377d76075dac19988fad0",
          "driver": "bridge",
          "network_interface": "podman1",
          "created": "2026-04-27T18:12:14.982632136+09:00",
          "subnets": [
               {
                    "subnet": "10.89.0.0/24",
                    "gateway": "10.89.0.1"
               }
          ],
          "ipv6_enabled": false,
          "internal": false,
          "dns_enabled": true,
          "ipam_options": {
               "driver": "host-local"
          },
          "containers": {}
     }
]
```

> **K8s에서는 이것이 자동:** K8s Service를 만들면 **CoreDNS**가 `<service-name>.<namespace>.svc.cluster.local`로 자동 등록합니다. 위의 수동 네트워크 설정을 K8s가 자동화해주는 것입니다.
> 

---

# Part 8. CKA/CKS 시험 필수 — crictl 명령어

K8s Node에서 Container 문제를 디버깅할 때 사용하는 도구입니다. CKA 시험에서 **kubectl이 안 먹히는 상황**(API Server 장애 등)에서 Node에 SSH하여 `crictl`로 진단합니다.

```bash
# === Container 관리 ===
sudo crictl ps                          # 실행 중인 Container
sudo crictl ps -a                       # 모든 Container (종료된 것 포함)
sudo crictl logs <container_id>         # Container 로그
sudo crictl exec -it <container_id> sh  # Container 접속
sudo crictl inspect <container_id>      # Container 상세 정보
sudo crictl stats                       # 리소스 사용량

# === Pod 관리 ===
sudo crictl pods                        # Pod 목록
sudo crictl inspectp <pod_id>           # Pod 상세

# === Image 관리 ===
sudo crictl images                      # Image 목록
sudo crictl rmi <image_id>             # Image 삭제
sudo crictl rmi --prune                 # 미사용 Image 정리 (DiskPressure 대응)
sudo crictl pull nginx:1.25            # Image Pull

# === CKA 단골 시나리오: Static Pod 장애 ===
# API Server가 안 뜰 때 — kubectl 사용 불가
sudo crictl ps -a | grep kube-apiserver
sudo crictl logs <apiserver_container_id>

# Static Pod Manifest 확인
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Pod 로그 디렉터리에서 직접 확인
sudo tail -100 /var/log/pods/kube-system_kube-apiserver-*/kube-apiserver/*.log
```

**crictl vs podman/docker 명령어 매핑:**

| 목적 | crictl | podman | docker |
| --- | --- | --- | --- |
| Container 목록 | `crictl ps` | `podman ps` | `docker ps` |
| Container 로그 | `crictl logs` | `podman logs` | `docker logs` |
| Container 접속 | `crictl exec -it` | `podman exec -it` | `docker exec -it` |
| Image 목록 | `crictl images` | `podman images` | `docker images` |
| Image Pull | `crictl pull` | `podman pull` | `docker pull` |
| Image 삭제 | `crictl rmi` | `podman rmi` | `docker rmi` |

---

# Part 9. CKS Security — Container 보안 심화

## 9-1. Rootless Container

```bash
# Podman은 기본적으로 Rootless
podman run -d --name web nginx
podman exec web whoami    # root (Container 안에서는 root)
# 하지만 Host에서는 일반 사용자로 매핑됨

# Container 안에서 non-root로 실행
podman run -d --user 1000:1000 nginx

# K8s Pod Spec에서 (CKS 단골)
# spec:
#   securityContext:
#     runAsNonRoot: true
#     runAsUser: 1000
#     fsGroup: 2000
#   containers:
#   - name: app
#     securityContext:
#       allowPrivilegeEscalation: false
#       readOnlyRootFilesystem: true
#       capabilities:
#         drop: ["ALL"]
```

## 9-2. Read-only Root Filesystem

```bash
# Root Filesystem을 읽기전용으로 (보안 강화)
podman run -d --read-only --tmpfs /tmp --tmpfs /var/cache/nginx nginx
```

> **CKS:** `readOnlyRootFilesystem: true`는 CKS에서 거의 필수. Container 내부에서 악성 코드가 파일을 쓸 수 없게 차단합니다.
> 

## 9-3. Capabilities 제거

Linux Capabilities는 root 권한을 세분화한 것입니다. Container에 필요 없는 Capability를 제거하면 보안이 강화됩니다.

```bash
# 모든 Capability 제거
podman run -d --cap-drop=ALL nginx

# 필요한 것만 추가
podman run -d --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx
```

## 9-4. Seccomp Profile

Seccomp은 Container가 호출할 수 있는 **System Call을 필터링**합니다.

```bash
# 기본 Seccomp Profile 사용 (위험한 System Call 차단)
podman run -d --security-opt seccomp=default nginx

# Custom Profile
podman run -d --security-opt seccomp=/path/to/profile.json nginx
```

> **CKS:** Seccomp Profile을 Pod에 적용하는 문제가 출제됩니다.
> 
> 
> ```yaml
> spec:
>   securityContext:
>     seccompProfile:
>       type: Localhost
>       localhostProfile: profiles/audit.json
> ```
> 

---

# Part 10. 자격증 연결 정리

## CKA (100% 연관)

| 기술 | CKA 적용 |
| --- | --- |
| `crictl ps/logs/exec` | **kubectl 안 먹힐 때** Container 직접 디버깅 |
| Static Pod Manifest (`/etc/kubernetes/manifests/`) | API Server, etcd 장애 복구 |
| Image Pull/Tag | Pod Image 변경, Image Pull Error 디버깅 |
| Port Mapping 개념 | Service(NodePort, ClusterIP) 이해 |
| Volume Mount | PV/PVC 설정, emptyDir, ConfigMap/Secret Mount |
| Container Network | Pod Network 원리 (같은 NS 공유 = localhost 통신) |
| Resource Limit (`--memory`, `--cpus`) | Pod resources.requests/limits |

## CKS (100% 연관)

| 기술 | CKS 적용 |
| --- | --- |
| Image Tag/Digest | `latest` 금지, 구체적 Tag/Digest 사용 |
| Image Registry 제한 | OPA/Gatekeeper로 허용 Registry 강제 |
| Image 취약점 스캔 | trivy, Clair 등 |
| Rootless / `runAsNonRoot` | Pod Security Standards |
| `readOnlyRootFilesystem` | 쓰기 차단으로 악성 코드 방지 |
| Capabilities Drop | `drop: ["ALL"]` + 필요한 것만 추가 |
| Seccomp Profile | System Call 필터링 |
| `hostPID`, `hostNetwork` 차단 | Pod Security Admission |
| `hostPath` Volume 제한 | Host Filesystem 접근 차단 |

## AWS SAA

| 기술 | SAA 적용 |
| --- | --- |
| Container 개념 | **ECS** (AWS 자체 Container 서비스) 이해 |
| Container Orchestration | **EKS** (AWS Managed K8s) 이해 |
| Image Registry | **ECR** (Elastic Container Registry) |
| VM vs Container | 워크로드에 따른 서비스 선택 (EC2 vs ECS vs Lambda) |