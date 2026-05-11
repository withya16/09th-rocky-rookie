# Rocky Linux 스터디 — 8주차
## 컨테이너와 가상화

> **기준 버전:** Rocky Linux 10 / Docker 27 / Podman 5

---

## 📚 주요 출처

| 문서 | URL |
|------|-----|
| Docker 공식 문서 | https://docs.docker.com |
| Podman 공식 문서 | https://docs.podman.io |
| RHEL 10 Building, running, and managing containers | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/building_running_and_managing_containers/index |
| OCI Runtime Specification | https://github.com/opencontainers/runtime-spec |

---

## 이번 주 전체 맥락

```
지금까지 배운 것들이 컨테이너 안에서 어떻게 쓰이는지:

4주차 cgroup    → 컨테이너 자원 제한의 실체
4주차 systemd   → Docker 데몬을 systemd가 관리
5주차 nftables  → Docker NAT, 포트 매핑의 실체
5주차 SELinux   → 컨테이너 격리 추가 보안층
7주차 journald  → 컨테이너 로그 수집

컨테이너 = 이것들의 조합을 자동으로 세팅해주는 편의 도구
```

---

## 1. 가상머신 vs 컨테이너

### 1-1. 가상머신 (VM)

```
┌─────────────────────────────────────────┐
│  VM 1              │  VM 2              │
│  Guest OS (Linux)  │  Guest OS (Windows)│
│  커널              │  커널              │
│  앱 A              │  앱 B              │
├─────────────────────────────────────────┤
│  Hypervisor (KVM, VMware, VirtualBox)   │
├─────────────────────────────────────────┤
│  Host OS                                │
│  Host 커널                              │
├─────────────────────────────────────────┤
│  물리 하드웨어                           │
└─────────────────────────────────────────┘

VM이 하는 것:
  하드웨어 전체를 소프트웨어로 에뮬레이션
  → CPU, 메모리, 디스크, NIC 모두 가상화
  → Guest OS가 자기만의 커널 가짐
  → 완전한 격리 (다른 OS 실행 가능)
  → 부팅 시간 수십 초 ~ 수 분
  → 이미지 크기 수 GB
```

### 1-2. 컨테이너

```
┌─────────────────────────────────────────┐
│  컨테이너 1   │  컨테이너 2   │  컨테이너 3│
│  앱 A + 라이브러리│앱 B + 라이브러리│앱 C  │
├─────────────────────────────────────────┤
│  컨테이너 런타임 (Docker, Podman)        │
│                                          │
│  Host 커널 (공유)                        │
├─────────────────────────────────────────┤
│  물리 하드웨어                           │
└─────────────────────────────────────────┘

컨테이너가 하는 것:
  Host 커널을 공유, 프로세스만 격리
  → namespace: 프로세스 시야 격리
  → cgroup: 자원 제한
  → 별도 커널 없음
  → 시작 시간 수백 ms
  → 이미지 크기 수십 ~ 수백 MB
```

### 1-3. 비교

```
항목              VM                  컨테이너
────────────────────────────────────────────────────
격리 수준         강함 (하드웨어 수준)  중간 (커널 공유)
시작 시간         수십 초 ~ 수 분       수백 ms
이미지 크기       수 GB                수십 ~ 수백 MB
OS               독립적 Guest OS 가능  Host OS 커널 공유
오버헤드          높음 (하드웨어 에뮬)  낮음 (프로세스 수준)
밀도              낮음                  높음 (수백 개/서버)
보안 격리         강함                  상대적으로 약함
주요 용도         다른 OS, 강한 격리    앱 배포, 마이크로서비스
```

컨테이너가 "격리된 것처럼 보이는" 이유는 커널 기능 두 가지 덕분입니다:

```
namespace — 무엇이 보이는지 격리
  PID namespace:  컨테이너 안 nginx = PID 1 (실제론 1236)
  NET namespace:  자기만의 네트워크 인터페이스
  MNT namespace:  자기만의 파일시스템 루트 /
  UTS namespace:  자기만의 hostname
  IPC namespace:  자기만의 IPC 자원
  USER namespace: 자기만의 UID/GID 매핑

cgroup — 얼마나 쓸 수 있는지 제한
  CPU, 메모리, I/O 상한선
  컨테이너마다 cgroup 하나씩 생성해서 자원을 격리
```

---

## 2. Docker vs Podman

### 2-1. Docker 아키텍처

```
사용자
  ↓ docker 명령
Docker CLI
  ↓ REST API (Unix socket /var/run/docker.sock)
dockerd (Docker 데몬, root 권한으로 실행)
  ↓
containerd
  ↓
runc (OCI 런타임, 실제 컨테이너 생성)
  ↓
컨테이너 프로세스

문제:
  dockerd가 항상 root로 실행 중 (데몬)
  → docker.sock 접근 = root 권한과 동일
  → 보안 위험
```

### 2-2. Podman 아키텍처

```
사용자
  ↓ podman 명령
Podman (데몬 없음, 직접 실행)
  ↓
conmon (컨테이너 모니터)
  ↓
runc (OCI 런타임)
  ↓
컨테이너 프로세스

차이:
  데몬 없음 (daemonless)
  → root 없이 rootless 컨테이너 실행 가능
  → 사용자 권한으로 컨테이너 실행
  → RHEL/Rocky Linux 공식 권장 도구
  → Docker CLI 명령어와 거의 100% 호환
    (docker 자리에 podman 그대로 쓸 수 있음)
```

### 2-3. 어떤 걸 쓸까

```
Docker:
  docker-compose 생태계 풍부
  CI/CD 툴 연동 많음
  개발 환경에서 많이 씀

Podman:
  RHEL/Rocky Linux 기본 포함
  rootless 컨테이너 (보안 강함)
  systemd 통합 (컨테이너를 systemd 서비스로 등록 가능)
  Kubernetes 친화적

이 문서:
  명령어 예시는 docker 기준
  podman은 docker → podman 으로 바꾸면 거의 그대로 동작
```

### 2-4. 설치

```bash
# Docker 설치 (Rocky Linux 10)
dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
dnf install docker-ce docker-ce-cli containerd.io
systemctl enable --now docker

# 현재 사용자를 docker 그룹에 추가 (sudo 없이 사용)
usermod -aG docker $USER
newgrp docker

# Podman 설치 (Rocky Linux 10 기본 포함)
dnf install podman

# 버전 확인
docker version
podman version
```

---

## 3. 이미지

### 3-1. 이미지란

```
이미지 = 컨테이너의 템플릿 (읽기 전용)
컨테이너 = 이미지를 실행한 인스턴스 (읽기/쓰기 레이어 추가)

이미지 레이어 구조:
  ┌─────────────────────────┐
  │ 레이어 4: 앱 코드        │ ← Dockerfile COPY
  ├─────────────────────────┤
  │ 레이어 3: pip 패키지     │ ← Dockerfile RUN pip install
  ├─────────────────────────┤
  │ 레이어 2: Python 3.11   │ ← FROM python:3.11
  ├─────────────────────────┤
  │ 레이어 1: Debian base   │ ← 베이스 이미지
  └─────────────────────────┘ (읽기 전용)
         +
  ┌─────────────────────────┐
  │ 컨테이너 레이어          │ ← 실행 중 변경사항 (읽기/쓰기)
  └─────────────────────────┘

레이어를 공유하기 때문에:
  python:3.11 기반 이미지 10개 = 베이스 레이어 1개 + 각 앱 레이어
  → 디스크 효율적
```

### 3-2. 이미지 이름 구조

```
registry/namespace/name:tag

docker.io/library/nginx:1.25
└──────┘ └─────┘ └───┘ └──┘
레지스트리  네임스  이름  태그
          스페이스

생략 시 기본값:
  레지스트리: docker.io
  네임스페이스: library (공식 이미지)
  태그: latest

예시:
  nginx              = docker.io/library/nginx:latest
  nginx:1.25         = docker.io/library/nginx:1.25
  ubuntu:22.04       = docker.io/library/ubuntu:22.04
  mysql:8.0          = docker.io/library/mysql:8.0
  myapp/api:v1.2.3   = docker.io/myapp/api:v1.2.3
```

### 3-3. 이미지 검색 및 다운로드

```bash
# 이미지 검색
docker search nginx
docker search --filter=is-official=true nginx   # 공식만

# 이미지 다운로드 (pull)
docker pull nginx
docker pull nginx:1.25
docker pull ubuntu:22.04
docker pull mysql:8.0

# 다운로드된 이미지 목록
docker images
docker image ls

# 출력:
# REPOSITORY  TAG     IMAGE ID       CREATED       SIZE
# nginx       latest  abc123def456   2 weeks ago   187MB
# ubuntu      22.04   def456abc123   3 weeks ago   77.8MB

# 이미지 상세 정보
docker inspect nginx

# 이미지 레이어 히스토리
docker history nginx

# 이미지 삭제
docker rmi nginx
docker rmi nginx:1.25
docker image rm nginx

# 사용하지 않는 이미지 전체 삭제
docker image prune
docker image prune -a    # 컨테이너가 참조하지 않는 이미지 전부
```

---

## 4. 컨테이너 실행

### 4-1. docker run 핵심 옵션

```bash
docker run [옵션] 이미지 [명령어]

# 기본 실행 후 종료
docker run ubuntu echo "hello"

# 대화형 (-it: 터미널 attach)
docker run -it ubuntu bash
docker run -it --rm ubuntu bash   # 종료 시 컨테이너 자동 삭제

# 백그라운드 데몬 (-d)
docker run -d nginx
docker run -d --name webserver nginx   # 이름 지정

# 주요 옵션:
-d              백그라운드 실행 (detached)
-it             대화형 터미널 (-i: stdin 열기, -t: pseudo-TTY)
--rm            종료 시 컨테이너 자동 삭제
--name          컨테이너 이름 지정
-p              포트 매핑 (다음 섹션)
-v              볼륨 연결 (다음 섹션)
-e              환경 변수 설정
--network       네트워크 지정
--restart       재시작 정책
--cpus          CPU 제한
--memory        메모리 제한
```

### 4-2. 컨테이너 조회 및 상태

```bash
# 실행 중인 컨테이너
docker ps

# 전체 (중지된 것 포함)
docker ps -a

# 출력:
# CONTAINER ID  IMAGE   COMMAND   CREATED    STATUS    PORTS         NAMES
# abc123        nginx   "/docker…" 2 min ago  Up 2 min  80/tcp        webserver
# def456        ubuntu  "bash"    5 min ago  Exited(0) 5 min ago     eager_turing

# 컨테이너 상세 정보
docker inspect webserver
docker inspect webserver | grep IPAddress   # IP만

# 리소스 실시간 사용량
docker stats
docker stats webserver     # 특정 컨테이너만
```

### 4-3. 컨테이너 제어

```bash
# 중지 / 시작 / 재시작
docker stop webserver      # SIGTERM → 10초 후 SIGKILL
docker stop -t 30 webserver  # 30초 기다린 후 강제 종료
docker start webserver
docker restart webserver

# 강제 종료 (SIGKILL)
docker kill webserver

# 컨테이너 삭제
docker rm webserver
docker rm -f webserver     # 실행 중이어도 강제 삭제

# 중지된 컨테이너 전체 삭제
docker container prune

# 전체 정리 (이미지, 컨테이너, 네트워크, 볼륨)
docker system prune
docker system prune -a     # 미사용 이미지까지
```

### 4-4. 실행 중인 컨테이너 접속

```bash
# 새 프로세스 실행 (exec) — 가장 많이 씀
docker exec -it webserver bash
docker exec -it webserver sh      # bash 없는 경우
docker exec webserver ls /etc     # 명령만 실행하고 종료

# 로그 확인
docker logs webserver
docker logs -f webserver          # 실시간 follow
docker logs --tail 50 webserver   # 최근 50줄
docker logs --since 1h webserver  # 1시간 이내

# 컨테이너 → 호스트 파일 복사
docker cp webserver:/etc/nginx/nginx.conf ./nginx.conf

# 호스트 → 컨테이너 파일 복사
docker cp ./index.html webserver:/usr/share/nginx/html/
```

### 4-5. 재시작 정책

```bash
# 서버 재부팅해도 컨테이너 자동 시작
docker run -d --restart=always nginx

# 재시작 정책:
# no           재시작 안 함 (기본)
# always       항상 재시작 (수동 stop 제외)
# on-failure   비정상 종료 시만
# on-failure:5 비정상 종료 시 최대 5번
# unless-stopped  수동으로 멈춘 경우 제외하고 항상
```

---

## 5. 포트 매핑

### 5-1. 포트 매핑 원리

컨테이너는 자체 NET namespace 안에 있어서 호스트 외부에서 직접 접근할 수 없습니다. 포트 매핑은 "호스트의 특정 포트로 들어온 트래픽을 컨테이너 포트로 전달"하는 규칙을 커널에 등록하는 것입니다.

```
docker run -p 8080:80 nginx 실행 시:

외부 클라이언트 → 호스트 IP:8080
                        ↓
              커널 PREROUTING 단계에서
              목적지 주소를 변환:
              호스트:8080 → 172.17.0.2:80 (컨테이너 IP:포트)
                        ↓
              컨테이너 eth0(172.17.0.2)으로 전달
                        ↓
              nginx 프로세스가 받음

이 주소 변환(DNAT)을 Docker가 컨테이너 시작 시
nftables에 자동으로 등록합니다:
  목적지가 호스트:8080인 TCP 패킷
  → 목적지를 172.17.0.2:80으로 바꿔서 전달
```

**firewalld로 Docker 포트를 막을 수 없는 이유:**

```
firewalld는 INPUT 체인에서 패킷을 검사합니다.
그런데 포트 매핑된 패킷은 PREROUTING에서 이미
목적지가 컨테이너 IP로 바뀌어버립니다.

라우팅 결정: "172.17.0.2는 내 IP가 아님"
→ INPUT이 아닌 FORWARD 체인으로 빠짐
→ firewalld의 INPUT 규칙을 아예 통과하지 않음
→ 차단 불가

해결책: 처음부터 특정 IP에만 바인딩
  docker run -p 127.0.0.1:8080:80 nginx
  → 호스트 외부에서는 아예 접근 못 함
```

### 5-2. 포트 매핑 옵션

```bash
# 호스트포트:컨테이너포트
docker run -d -p 8080:80 nginx
docker run -d -p 443:443 -p 80:80 nginx   # 여러 포트

# 특정 IP만 바인딩 (외부 차단)
docker run -d -p 127.0.0.1:8080:80 nginx  # 로컬에서만 접근 가능

# 호스트 포트 자동 할당
docker run -d -p 80 nginx                 # 랜덤 포트 배정
docker port webserver                     # 어떤 포트에 매핑됐는지 확인

# UDP 포트
docker run -d -p 53:53/udp bind9

# 매핑 확인
docker ps                   # PORTS 열
docker port webserver       # 특정 컨테이너만
```

### 5-3. 실습 예시

```bash
# nginx 80 포트를 호스트 8080으로
docker run -d --name web -p 8080:80 nginx
curl http://localhost:8080      # 확인

# MySQL 3306
docker run -d \
  --name db \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=myapp \
  mysql:8.0

# 접속 확인
docker exec -it db mysql -u root -psecret -e "show databases;"
```

---

## 6. 볼륨 — 데이터 영속성

### 6-1. 왜 볼륨이 필요한가

```
컨테이너의 파일시스템은 컨테이너와 함께 사라짐:

docker run -d mysql:8.0
→ DB에 데이터 저장
→ docker rm 컨테이너 삭제
→ 모든 데이터 사라짐

볼륨 = 컨테이너 외부에 데이터를 저장
→ 컨테이너 삭제해도 데이터 유지
→ 여러 컨테이너가 같은 데이터 공유 가능
```

### 6-2. 볼륨 종류

```
① Docker 볼륨 (named volume) — 권장
   Docker가 /var/lib/docker/volumes/ 에서 관리
   이름으로 참조
   백업/이전이 편함

② 바인드 마운트 (bind mount)
   호스트의 특정 경로를 컨테이너에 연결
   개발 환경에서 소스코드 공유에 유용
   호스트 경로를 직접 지정

③ tmpfs 마운트
   메모리에만 저장 (재부팅/컨테이너 종료 시 사라짐)
   민감한 데이터, 임시 파일
```

### 6-3. 볼륨 사용법

```bash
# ── Named Volume ─────────────────────────────────────────

# 볼륨 생성
docker volume create mydata

# 볼륨 목록
docker volume ls

# 볼륨 상세 정보
docker volume inspect mydata
# Mountpoint: /var/lib/docker/volumes/mydata/_data

# 볼륨 연결해서 실행
docker run -d \
  --name db \
  -v mydata:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8.0

# 컨테이너 삭제 후 재생성해도 데이터 유지
docker rm -f db
docker run -d --name db -v mydata:/var/lib/mysql mysql:8.0
# → 이전 데이터 그대로

# 볼륨 삭제
docker volume rm mydata
docker volume prune    # 사용하지 않는 볼륨 전체

# ── Bind Mount ────────────────────────────────────────────

# 호스트 경로:컨테이너 경로
docker run -d \
  --name web \
  -p 8080:80 \
  -v /var/www/html:/usr/share/nginx/html \
  nginx

# 호스트에서 파일 수정 → 컨테이너에 즉시 반영
echo "<h1>Hello</h1>" > /var/www/html/index.html
curl http://localhost:8080   # 반영 확인

# 읽기 전용 마운트 (보안)
docker run -d \
  -v /etc/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx

# ── tmpfs ─────────────────────────────────────────────────

docker run -d \
  --tmpfs /tmp:size=100m \
  myapp
```

### 6-4. 볼륨 백업/복원

```bash
# 볼륨 백업
docker run --rm \
  -v mydata:/source \
  -v $(pwd):/backup \
  ubuntu \
  tar -czf /backup/mydata_backup.tar.gz -C /source .

# 볼륨 복원
docker run --rm \
  -v mydata:/target \
  -v $(pwd):/backup \
  ubuntu \
  tar -xzf /backup/mydata_backup.tar.gz -C /target
```

---

## 7. 네트워크

### 7-1. Docker 기본 네트워크

```bash
# 네트워크 목록
docker network ls

# 기본 제공:
# NETWORK ID  NAME     DRIVER  SCOPE
# abc123      bridge   bridge  local   ← 기본 (docker0)
# def456      host     host    local
# ghi789      none     null    local
```

### 7-2. 네트워크 종류

```
bridge (기본):
  컨테이너들이 가상 스위치(docker0)에 연결
  NAT를 통해 외부 통신
  컨테이너끼리 IP로 통신 가능
  기본 bridge에서는 이름으로 통신 불가
  → 사용자 정의 bridge 권장

  ┌──────────┐   ┌──────────┐
  │ 컨테이너A │   │ 컨테이너B │
  │172.17.0.2│   │172.17.0.3│
  └────┬─────┘   └────┬─────┘
       └──────┬────────┘
          docker0 (172.17.0.1)
              │
           호스트 NIC
              │
           인터넷

host:
  호스트 네트워크 스택 직접 사용
  포트 매핑 없이 호스트 포트 그대로 사용
  성능 좋음, 격리 약함
  docker run --network=host nginx
  → localhost:80 직접 접근 가능

none:
  네트워크 없음 (완전 격리)
  보안이 매우 중요한 배치 작업 등
```

### 7-3. 사용자 정의 네트워크

기본 bridge 대신 사용자 정의 네트워크를 쓰는 게 권장 패턴입니다:

```bash
# 네트워크 생성
docker network create mynet
docker network create --subnet=172.20.0.0/16 mynet

# 네트워크 지정해서 실행
docker run -d --name db      --network mynet mysql:8.0
docker run -d --name app     --network mynet -p 8080:80 myapp
docker run -d --name cache   --network mynet redis

# 사용자 정의 네트워크에서는 이름으로 통신 가능!
# app 컨테이너 안에서:
#   mysql -h db -u root -p       ← "db" 이름으로 접근
#   redis-cli -h cache           ← "cache" 이름으로 접근
# → 내장 DNS가 컨테이너 이름을 IP로 변환해줌

# 네트워크 상세 정보
docker network inspect mynet

# 컨테이너를 네트워크에 연결/해제
docker network connect mynet webserver
docker network disconnect mynet webserver

# 네트워크 삭제
docker network rm mynet
docker network prune    # 사용하지 않는 네트워크 전체
```

### 7-4. 컨테이너 간 통신 정리

```
같은 사용자 정의 네트워크:
  컨테이너 이름으로 직접 통신 가능 (DNS 자동)
  추가 설정 불필요

다른 네트워크:
  기본적으로 통신 불가
  docker network connect 로 연결하거나
  포트 매핑 + 호스트 IP로 우회

호스트 → 컨테이너:
  -p 포트매핑 필요
  또는 --network=host

컨테이너 → 외부 인터넷:
  기본 bridge에서 NAT(Masquerade)로 가능
  호스트 IP로 변환되어 나감
```

---

## 8. 환경 변수와 설정 주입

```bash
# -e 로 환경 변수 주입
docker run -d \
  --name db \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=myapp \
  -e MYSQL_USER=appuser \
  -e MYSQL_PASSWORD=apppass \
  mysql:8.0

# .env 파일로 주입
cat > .env << EOF
MYSQL_ROOT_PASSWORD=secret
MYSQL_DATABASE=myapp
EOF

docker run -d --env-file .env mysql:8.0

# 컨테이너 환경 변수 확인
docker exec db env
docker inspect db | grep -A 20 '"Env"'
```

---

## 9. Dockerfile — 이미지 빌드

### 9-1. Dockerfile 기본 구조

```dockerfile
# 베이스 이미지
FROM python:3.11-slim

# 메타데이터
LABEL maintainer="alice@example.com"

# 환경 변수
ENV APP_HOME=/app \
    PORT=8080

# 작업 디렉토리
WORKDIR $APP_HOME

# 파일 복사 (의존성 먼저 → 레이어 캐시 활용)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 소스코드 복사
COPY . .

# 포트 선언 (문서 목적, 실제 오픈은 -p로)
EXPOSE $PORT

# 컨테이너 시작 시 실행
CMD ["python", "app.py"]
```

### 9-2. 이미지 빌드

```bash
# 기본 빌드
docker build -t myapp:v1.0 .
docker build -t myapp:v1.0 -f Dockerfile.prod .   # 파일 지정

# 빌드 후 실행
docker run -d -p 8080:8080 myapp:v1.0

# 태그 추가
docker tag myapp:v1.0 myapp:latest

# 레지스트리 push
docker login
docker tag myapp:v1.0 myusername/myapp:v1.0
docker push myusername/myapp:v1.0
```

### 9-3. .dockerignore

```
# .dockerignore (빌드 컨텍스트에서 제외)
.git
.env
node_modules
__pycache__
*.pyc
*.log
```

---

## 10. Podman 특이점

### 10-1. rootless 컨테이너

```bash
# root 없이 실행 (일반 사용자)
podman run -d -p 8080:80 nginx
# → 사용자 namespace 사용
# → 컨테이너 안 root = 호스트에서 일반 사용자

# rootless 컨테이너 확인
podman unshare cat /proc/self/uid_map
```

### 10-2. systemd 통합

```bash
# 컨테이너를 systemd 서비스로 등록
podman generate systemd --name webserver --files --new

# 생성된 파일
cat container-webserver.service

# 사용자 서비스로 등록 (root 불필요)
mkdir -p ~/.config/systemd/user/
cp container-webserver.service ~/.config/systemd/user/
systemctl --user enable --now container-webserver

# 시스템 서비스로 등록
cp container-webserver.service /etc/systemd/system/
systemctl enable --now container-webserver
```

### 10-3. Docker와 차이

```
명령어:     docker → podman 그대로 교체 가능 (거의)
데몬:       Docker는 dockerd 항상 실행 중
            Podman은 명령 실행 시만 동작
소켓:       /var/run/docker.sock vs 없음
Compose:    docker-compose vs podman-compose (또는 Docker Compose 사용)
레지스트리: 기본 docker.io vs docker.io + quay.io + registry.access.redhat.com
```

---

## 11. 전체 실습 시나리오

nginx + MySQL + 앱 컨테이너 3개를 같은 네트워크로 연결:

```bash
# 1. 네트워크 생성
docker network create appnet

# 2. 데이터 볼륨 생성
docker volume create dbdata

# 3. MySQL 컨테이너
docker run -d \
  --name db \
  --network appnet \
  -v dbdata:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=myapp \
  mysql:8.0

# 4. 앱 컨테이너 (DB에 "db" 이름으로 접근)
docker run -d \
  --name app \
  --network appnet \
  -e DB_HOST=db \
  -e DB_PASSWORD=secret \
  myapp:v1.0

# 5. nginx 리버스 프록시
docker run -d \
  --name proxy \
  --network appnet \
  -p 80:80 \
  -v ./nginx.conf:/etc/nginx/conf.d/default.conf:ro \
  nginx

# 6. 상태 확인
docker ps
docker stats
docker network inspect appnet

# 7. 로그 확인
docker logs app -f
docker logs db --tail 50

# 8. 접속 테스트
curl http://localhost

# 9. 정리
docker stop proxy app db
docker rm proxy app db
docker network rm appnet
docker volume rm dbdata
```

---


## 12. 이론 심화 — OCI와 파일시스템

### 12-1. OCI — 표준화의 배경

```
2013년 Docker 등장 → 컨테이너 폭발적 성장
2015년 OCI (Open Container Initiative) 설립
  → "이미지 형식과 런타임을 표준화하자"
  → Linux Foundation 산하

OCI 3대 스펙:
  Image Spec      이미지가 어떻게 생겼는지
  Runtime Spec    컨테이너를 어떻게 실행하는지
  Distribution Spec  이미지를 레지스트리에서 어떻게 주고받는지

결과:
  Docker로 빌드한 이미지 → Podman으로 실행 가능
  GitHub Actions에서 빌드 → AWS ECR에 push → Kubernetes에서 실행
  → 모두 OCI 스펙을 따르기 때문에 호환됨
```

### 12-2. OCI Image Spec — 이미지의 실체

이미지는 3개의 객체로 구성됩니다.

```
┌─────────────────────────────────────────────┐
│  Image Index (선택적)                        │
│  여러 아키텍처를 하나의 태그로 묶는 목록      │
│  nginx:latest → amd64 manifest              │
│                → arm64 manifest             │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│  Image Manifest                             │
│  이 이미지가 무엇으로 구성됐는지 목록        │
│  {                                          │
│    "config": { "digest": "sha256:abc..." }, │
│    "layers": [                              │
│      { "digest": "sha256:111..." },         │
│      { "digest": "sha256:222..." },         │
│      { "digest": "sha256:333..." }          │
│    ]                                        │
│  }                                          │
└──────────┬──────────────────────────────────┘
           ↓                    ↓
┌──────────────────┐  ┌──────────────────────┐
│  Config          │  │  Layers (tar.gz)      │
│  런타임 설정      │  │  파일시스템 변경사항   │
│  ENV, CMD,       │  │  레이어 1: Debian base│
│  ENTRYPOINT,     │  │  레이어 2: Python     │
│  WORKDIR 등      │  │  레이어 3: pip 패키지 │
└──────────────────┘  │  레이어 4: 앱 코드    │
                      └──────────────────────┘
```

모든 객체는 **sha256 digest**로 참조됩니다:

```
Content-Addressable Storage:
  객체의 내용을 sha256 해싱 → 그 해시값이 곧 주소

  sha256("이 레이어의 바이트들") = sha256:9834876d...
  → 이 hash로 레이어를 식별하고 참조

장점:
  1. 변조 감지 자동화
     받은 레이어를 해싱 → manifest의 digest와 비교 → 다르면 손상

  2. 중복 제거
     nginx와 ubuntu 이미지 둘 다 동일한 Debian base 레이어 사용
     → digest가 같으면 = 같은 레이어 = 한 번만 저장

  3. 캐시
     이미 가진 digest는 다시 다운로드 안 함
```

실제로 보기:

```bash
# 이미지를 tar로 추출해서 내부 구조 확인
docker save nginx:latest -o nginx.tar
mkdir nginx-extracted && tar xf nginx.tar -C nginx-extracted
ls nginx-extracted/
# index.json     ← Image Index
# blobs/sha256/  ← 모든 객체 (manifest, config, layer) 가 hash 이름으로 저장

cat nginx-extracted/index.json | python3 -m json.tool

# manifest 내용 확인
cat nginx-extracted/blobs/sha256/<manifest-digest> | python3 -m json.tool
```

### 12-3. docker pull 내부 과정

```
docker pull nginx:latest 실행 시:

1. 레지스트리 인증
   Docker Hub API: GET /v2/ → 401
   → Bearer token 요청 → token 획득

2. Image Index 조회
   GET /v2/library/nginx/manifests/latest
   → "이 태그는 어떤 플랫폼들을 지원하는가?"
   → amd64, arm64, arm/v7 ... 목록 반환

3. 플랫폼 선택
   현재 시스템이 linux/amd64
   → amd64 manifest의 digest 선택

4. Image Manifest 다운로드
   GET /v2/library/nginx/manifests/sha256:amd64-digest
   → config digest + 레이어 digest 목록 획득

5. Config 다운로드
   GET /v2/library/nginx/blobs/sha256:config-digest
   → ENV, CMD, ENTRYPOINT 등 메타데이터

6. 레이어 병렬 다운로드
   각 레이어: GET /v2/library/nginx/blobs/sha256:layer-N-digest
   → 이미 가진 레이어는 건너뜀 (이미 같은 digest 존재)
   → 없는 레이어만 다운로드
   → gzip 압축 해제 → /var/lib/docker/overlay2/ 에 저장

7. 무결성 검증
   다운로드한 각 객체 sha256 해싱
   → manifest에 명시된 digest와 비교
   → 불일치 시 폐기
```

### 12-4. OverlayFS — 레이어드 파일시스템의 실체

OverlayFS는 두 개의 디렉토리를 하나로 합쳐서 단일 디렉토리처럼 보여주는 union mount 기술입니다. lowerdir(하위), upperdir(상위), merged(통합 뷰) 개념을 씁니다. overlay2 드라이버는 최대 128개의 lowerdir 레이어를 네이티브로 지원합니다.

```
nginx 이미지 (3개 레이어) 기준:

lowerdir (읽기 전용, 이미지 레이어들):
  layer3(앱설정):  /etc/nginx/nginx.conf
  layer2(nginx):   /usr/sbin/nginx, /usr/lib/...
  layer1(Debian):  /bin/bash, /lib/x86_64-linux-gnu/...

upperdir (읽기/쓰기, 컨테이너 레이어):
  (처음엔 비어있음)

workdir (OverlayFS 내부용):
  (임시 작업 공간)

merged (컨테이너가 보는 /):
  = lowerdir 전체 + upperdir 를 합친 뷰
  /bin/bash         ← layer1에서
  /usr/sbin/nginx   ← layer2에서
  /etc/nginx/       ← layer3에서
  (비어있음)        ← upperdir (아직 변경 없음)
```

실제 마운트 확인:

```bash
# 실행 중인 컨테이너의 overlay 마운트 확인
mount | grep overlay
# overlay on /var/lib/docker/overlay2/.../merged type overlay
#   (rw,lowerdir=layer3:layer2:layer1,
#    upperdir=.../diff,
#    workdir=.../work)

# 컨테이너 파일시스템 경로 확인
docker inspect <container-id> | grep -A5 "GraphDriver"
# "Data": {
#   "LowerDir":  "/var/lib/docker/overlay2/abc/diff:...",
#   "MergedDir": "/var/lib/docker/overlay2/xyz/merged",
#   "UpperDir":  "/var/lib/docker/overlay2/xyz/diff",
#   "WorkDir":   "/var/lib/docker/overlay2/xyz/work"
# }
```

### 12-5. Copy-on-Write (CoW)

컨테이너가 처음으로 기존 파일에 쓰기를 시도하면, 해당 파일은 아직 upperdir에 존재하지 않습니다. overlay2 드라이버는 copy_up 연산으로 파일을 lowerdir에서 upperdir로 복사한 후 변경사항을 적용합니다. 단, OverlayFS는 블록 레벨이 아닌 파일 레벨에서 동작하기 때문에, 파일이 크고 일부만 수정하더라도 파일 전체를 복사합니다.

```
시나리오: 컨테이너 안에서 nginx.conf 수정

Before:
  upperdir/: (비어있음)
  lowerdir/etc/nginx/nginx.conf: [원본]
  merged/etc/nginx/nginx.conf: → lowerdir에서 읽음

"vi /etc/nginx/nginx.conf" 실행 시:
  ① copy_up: lowerdir의 nginx.conf → upperdir로 전체 복사
  ② 이후 수정은 upperdir의 복사본에 적용

After:
  upperdir/etc/nginx/nginx.conf: [수정된 버전]
  lowerdir/etc/nginx/nginx.conf: [원본, 변경 없음]
  merged/etc/nginx/nginx.conf: → upperdir 우선 → 수정본 읽힘

컨테이너 삭제 시:
  upperdir 삭제 → 수정본 사라짐
  lowerdir(이미지)는 그대로 → 다음 컨테이너는 원본으로 시작
```

파일 삭제 시 — whiteout 파일:

```
컨테이너 안에서 rm /etc/nginx/mime.types 실행 시:
  lowerdir는 읽기 전용 → 실제 삭제 불가
  대신 upperdir에 "whiteout 파일" 생성:
    upperdir/etc/nginx/mime.types (character device, major:minor = 0:0)

  merged 뷰에서:
    whiteout 파일 발견 → 해당 이름의 파일을 숨김
    → lowerdir에 파일이 있어도 보이지 않음
    → 삭제된 것처럼 보임
```

```bash
# CoW 직접 확인
docker run -d --name cow-test nginx
docker inspect cow-test --format '{{.GraphDriver.Data.UpperDir}}'
# /var/lib/docker/overlay2/xyz/diff

# upperdir 초기 상태 (비어있음)
ls /var/lib/docker/overlay2/xyz/diff

# nginx.conf 수정
docker exec cow-test bash -c "echo '# test' >> /etc/nginx/nginx.conf"

# upperdir에 복사된 파일 확인
ls /var/lib/docker/overlay2/xyz/diff/etc/nginx/
# nginx.conf  ← copy_up 된 것

# 파일 삭제
docker exec cow-test rm /etc/nginx/mime.types

# whiteout 파일 확인
ls -la /var/lib/docker/overlay2/xyz/diff/etc/nginx/
# c--------- mime.types  ← character device (whiteout)
```

### 12-6. 이미지 레이어와 Dockerfile

Dockerfile 명령어 하나가 레이어 하나입니다:

```dockerfile
FROM debian:12            # 레이어 1: debian base
RUN apt-get update && \   # 레이어 2: 패키지 캐시 + 설치
    apt-get install -y python3 pip
COPY requirements.txt .   # 레이어 3: requirements.txt
RUN pip install -r \      # 레이어 4: pip 패키지들
    requirements.txt
COPY . .                  # 레이어 5: 앱 소스코드
```

레이어 캐시 활용:

```
레이어는 digest 기반으로 캐시됨
→ 내용이 같으면 다시 빌드 안 함

나쁜 패턴:
  COPY . .                 ← 소스코드 (자주 바뀜)
  RUN pip install -r requirements.txt  ← 매번 재실행

좋은 패턴:
  COPY requirements.txt .  ← 의존성 파일 먼저
  RUN pip install -r requirements.txt  ← requirements 안 바뀌면 캐시 재사용
  COPY . .                 ← 소스코드 (가장 마지막)

이유:
  Docker는 위에서부터 레이어를 확인
  변경된 레이어부터 이후 모든 레이어를 재빌드
  → 자주 바뀌는 것을 뒤에 배치해야 앞 레이어 캐시 재사용 가능
```

레이어 크기 확인:

```bash
docker history nginx
# IMAGE         CREATED    CREATED BY                     SIZE
# abc123        2 wks ago  CMD ["nginx" "-g" "daemon...   0B
# <missing>     2 wks ago  COPY /etc/nginx /etc/nginx     35.8kB
# <missing>     2 wks ago  RUN apt-get install nginx      45.2MB
# <missing>     2 wks ago  RUN apt-get update             23.1MB
# <missing>     2 wks ago  FROM debian:12                 77.8MB
```

### 12-7. /var/lib/docker 실제 구조

```
/var/lib/docker/
├── overlay2/          ← 레이어 데이터 (핵심)
│   ├── <layer-hash>/
│   │   ├── diff/      ← 이 레이어의 파일 내용
│   │   ├── link       ← 짧은 ID (심볼릭 링크용)
│   │   ├── lower      ← 이 레이어 아래에 있는 레이어들
│   │   ├── merged/    ← 컨테이너 실행 시 마운트 포인트
│   │   └── work/      ← OverlayFS 내부용
│   └── l/             ← 짧은 ID → 실제 경로 심볼릭 링크
│
├── image/overlay2/
│   ├── imagedb/       ← 이미지 메타데이터 DB
│   └── layerdb/       ← 레이어 체인 정보
│
├── containers/        ← 컨테이너 메타데이터
│   └── <container-id>/
│       ├── config.v2.json   ← 컨테이너 설정
│       └── hostconfig.json  ← 호스트 설정 (포트, 볼륨 등)
│
└── volumes/           ← named volume 데이터
    └── <volume-name>/_data/
```

```bash
# 전체 Docker 디스크 사용량
docker system df
# TYPE            TOTAL  ACTIVE  SIZE      RECLAIMABLE
# Images          12     3       4.231GB   2.1GB (49%)
# Containers      5      2       234MB     100MB
# Local Volumes   8      3       12.3GB    8.1GB
# Build Cache     -      -       1.2GB     1.2GB

# 상세
docker system df -v
```

---

---


## 13. 컨테이너 구현 내부

컨테이너는 "새로운 기술"이 아니라 리눅스 커널의 기존 기능 세 가지를 조합한 것입니다. namespace, cgroup, 그리고 파일시스템입니다.

### 13-1. namespace — 격리의 실체

namespace는 커널 자원을 프로세스 그룹마다 독립적으로 보이게 하는 커널 기능으로, 컨테이너 격리의 핵심입니다.

namespace는 새로운 자원을 만드는 게 아닙니다. **같은 자원을 서로 다르게 보이게** 하는 것입니다.

```
시스템 부팅 시:
  모든 프로세스가 하나의 namespace 세트를 공유
  → 모든 프로세스가 전체 PID 목록을 봄
  → 모든 프로세스가 같은 hostname을 봄
  → 모든 프로세스가 같은 네트워크 스택을 봄

컨테이너 생성 시:
  새 namespace 세트를 만들어서 컨테이너 프로세스를 거기에 넣음
  → 컨테이너 프로세스는 자기 namespace 안의 것만 봄
  → 호스트와 다른 PID, hostname, 네트워크를 가짐
```

#### namespace 종류 (Linux 커널 6.x 기준)

```
namespace     커널 플래그      격리 대상
─────────────────────────────────────────────────────────
PID           CLONE_NEWPID     프로세스 ID 번호
NET           CLONE_NEWNET     네트워크 스택 전체
MNT           CLONE_NEWNS      파일시스템 마운트 포인트
UTS           CLONE_NEWUTS     hostname, domainname
IPC           CLONE_NEWIPC     SysV IPC (공유메모리, 세마포어)
USER          CLONE_NEWUSER    UID/GID 매핑
CGROUP        CLONE_NEWCGROUP  cgroup 계층 뷰
TIME          CLONE_NEWTIME    시스템 클럭 (커널 5.6+)
```

#### namespace를 만드는 syscall 3개

```c
/* 1. clone() — 새 프로세스를 새 namespace에서 생성 */
pid_t pid = clone(child_func,
                  stack + STACK_SIZE,
                  CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | SIGCHLD,
                  NULL);
/* → child_func는 새 PID/NET/MNT namespace에서 실행 */

/* 2. unshare() — 현재 프로세스의 namespace를 분리 */
unshare(CLONE_NEWUTS);
/* → 현재 프로세스부터 새 UTS namespace 사용 */
/* → 이후 hostname 변경이 호스트에 영향 없음 */

/* 3. setns() — 기존 namespace에 합류 */
int fd = open("/proc/1234/ns/net", O_RDONLY);
setns(fd, CLONE_NEWNET);
/* → PID 1234의 NET namespace로 들어감 */
/* → docker exec가 이 방식을 사용 */
```

namespace는 `/proc/<pid>/ns/` 파일로 표현됩니다:

```bash
ls -la /proc/$$/ns/
# cgroup -> cgroup:[4026531835]
# ipc    -> ipc:[4026531839]
# mnt    -> mnt:[4026531840]
# net    -> net:[4026531969]
# pid    -> pid:[4026531836]
# user   -> user:[4026531837]
# uts    -> uts:[4026531838]

# 숫자가 inode 번호 — 같으면 같은 namespace
# 컨테이너 프로세스의 inode와 호스트의 inode가 다름

# 컨테이너 PID 확인 (호스트에서 보이는 실제 PID)
docker inspect --format '{{.State.Pid}}' webserver

# 컨테이너의 namespace 확인
ls -la /proc/<container-pid>/ns/
# → 호스트와 다른 inode 번호

# docker exec의 원리: 기존 컨테이너 namespace에 setns()로 합류
nsenter -t <container-pid> --pid --net --mount --uts --ipc bash
```

#### namespace 종류별 상세

**PID namespace:**

```
호스트:
  PID 1:    systemd
  PID 1234: containerd
  PID 1235: nginx   ← 컨테이너 안에서 돌고 있는 nginx

컨테이너 안 (PID namespace 별도):
  PID 1:    nginx    ← 같은 프로세스지만 namespace 안에선 1번

PID namespace 특성:
  새 PID namespace의 첫 번째 프로세스 = PID 1 (init 역할)
  PID 1이 죽으면 namespace 안의 모든 프로세스도 SIGKILL
  → docker run 시 CMD가 PID 1이 되는 이유
  → 그래서 init 역할을 하는 tini 같은 래퍼를 쓰기도 함

호스트에서는:
  ps aux | grep nginx → 실제 PID(1235) 보임
  컨테이너 안에서는 PID 1로 보임
  → 같은 프로세스, 다른 번호
```

**NET namespace:**

```
새 NET namespace 생성 시 초기 상태:
  lo 인터페이스만 존재 (loopback, 127.0.0.1)
  라우팅 테이블 비어있음
  iptables/nftables 규칙 없음
  소켓 목록 비어있음

Docker가 하는 것:
  1. veth pair 생성 (가상 이더넷 케이블 양쪽)
     vethABC → 호스트 network namespace
     eth0    → 컨테이너 network namespace
  2. vethABC를 docker0 bridge에 연결
  3. eth0에 컨테이너 IP 할당 (172.17.0.2/16)
  4. 컨테이너 라우팅 테이블: default via 172.17.0.1 (docker0)

결과:
  컨테이너 → eth0 → vethABC → docker0 → 호스트 NIC → 인터넷
```

**MNT namespace:**

```
새 MNT namespace = 새로운 마운트 포인트 뷰
→ 컨테이너 안에서 mount/umount 해도 호스트에 영향 없음

컨테이너 파일시스템 구성:
  1. 새 MNT namespace 생성
  2. OverlayFS를 /로 마운트 (이미지 레이어들)
  3. /proc, /sys, /dev 마운트 (컨테이너용)
  4. pivot_root로 루트 파일시스템 교체

pivot_root:
  chroot의 강화 버전
  chroot:      루트처럼 보이게 하지만 실제 루트는 그대로
  pivot_root:  실제 마운트 네임스페이스의 루트 자체를 교체
               → 이전 루트를 숨겨서 탈출 불가

실제 흐름:
  unshare(CLONE_NEWNS)
  mount overlayfs → /newroot
  mkdir /newroot/.old
  pivot_root("/newroot", "/newroot/.old")
  umount("/.old")  ← 이전 루트 연결 끊기
  → 이제 / = 컨테이너 파일시스템
```

**USER namespace:**

```
USER namespace = UID/GID를 컨테이너 안팎으로 다르게 매핑

rootless 컨테이너의 핵심:

  호스트 UID 1000 (일반 사용자)
     ↓ USER namespace 생성
  컨테이너 안 UID 0 (root 처럼 보임)
     ↓ 파일 접근 시
  실제로는 UID 1000 권한으로 동작

/proc/<pid>/uid_map 파일:
  0  1000  1    ← 컨테이너 UID 0 = 호스트 UID 1000

→ 컨테이너 안에서 root여도 호스트에서는 일반 사용자
→ 컨테이너 탈출해도 호스트 root 권한 없음
→ Docker의 rootless 모드, Podman이 이 방식 사용
```

### 13-2. runc — 컨테이너 생성의 실제 실행자

```
Docker/Podman CLI
      ↓
containerd (컨테이너 생명주기 관리)
      ↓
runc (OCI Runtime — 실제 컨테이너 생성)
      ↓
Linux 커널 syscall
```

runc가 컨테이너를 만드는 과정:

```
OCI Runtime Bundle 구조:
  /bundle/
  ├── config.json    ← OCI Runtime Spec (뭘 어떻게 실행할지)
  └── rootfs/        ← 컨테이너 파일시스템

config.json 주요 내용:
  {
    "process": {
      "args": ["/usr/sbin/nginx", "-g", "daemon off;"],
      "env": ["PATH=/usr/local/sbin:..."],
      "user": {"uid": 0, "gid": 0}
    },
    "root": {"path": "rootfs"},
    "linux": {
      "namespaces": [
        {"type": "pid"},
        {"type": "network"},
        {"type": "mount"},
        {"type": "uts"},
        {"type": "ipc"}
      ],
      "resources": {
        "memory": {"limit": 536870912},   ← 512MB
        "cpu": {"quota": 50000}           ← 50% CPU
      }
    }
  }

runc create 실행 시 내부 순서:
  1. clone(CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | ...) 호출
  2. 자식 프로세스에서:
     - cgroup 설정 (메모리/CPU 제한 적용)
     - OverlayFS 마운트
     - /proc, /sys, /dev 마운트
     - pivot_root 로 루트 교체
     - hostname 설정 (UTS namespace)
     - capabilities 설정
     - seccomp 필터 적용
  3. exec("/usr/sbin/nginx") 로 대체
```

### 13-3. capabilities — root를 쪼개기

```
전통적인 Unix 권한:
  root  = 모든 것 가능
  일반  = 제한됨
  → 이분법적, 세밀한 제어 불가능

Linux capabilities:
  root 권한을 40개 이상의 독립적인 기능으로 분리
  프로세스마다 어떤 기능을 가질지 개별 설정 가능

주요 capabilities:
  CAP_NET_BIND_SERVICE  1024 이하 포트 바인딩 (80, 443)
  CAP_NET_ADMIN         네트워크 인터페이스 설정
  CAP_SYS_ADMIN         마운트, namespace 생성 등 (너무 강력)
  CAP_CHOWN             파일 소유자 변경
  CAP_KILL              다른 프로세스에 시그널 전송
  CAP_DAC_OVERRIDE      파일 권한 무시
  CAP_SYS_PTRACE        다른 프로세스 디버깅 (strace)

Docker 기본 컨테이너:
  root로 실행되지만 capabilities 대부분 제거됨
  → CAP_NET_ADMIN 없음 → 네트워크 인터페이스 못 만듦
  → CAP_SYS_ADMIN 없음 → 마운트 못 함
  → 해킹당해도 할 수 있는 것이 제한됨

확인:
  docker run --rm ubuntu capsh --print
  # Current: = cap_chown,cap_dac_override,...

  # capability 추가
  docker run --cap-add NET_ADMIN ...
  # capability 제거
  docker run --cap-drop CHOWN ...
  # 전부 제거 후 필요한 것만
  docker run --cap-drop ALL --cap-add NET_BIND_SERVICE ...
```

### 13-4. seccomp — 허용할 syscall 화이트리스트

```
capabilities가 "뭘 할 수 있는가" 라면
seccomp은 "어떤 syscall을 호출할 수 있는가"

Linux syscall 은 400개 이상:
  read, write, open, clone, mount, ptrace, ...

seccomp 프로파일 = 허용/차단할 syscall 목록 (JSON)

Docker 기본 seccomp 프로파일:
  ~300개 syscall 허용
  민감한 것들 차단:
    keyctl        (커널 키링 접근)
    add_key       (커널 키 추가)
    request_key
    ptrace        (기본 차단, 디버거 방지)
    reboot        (컨테이너 안에서 시스템 재부팅 방지)
    ...

보안 레이어 순서:
  syscall 호출
      ↓
  seccomp 필터  ← "이 syscall 허용?"
      ↓ 허용
  커널 처리
      ↓
  DAC 검사 (파일 권한)
      ↓ 통과
  LSM 검사 (SELinux/AppArmor)
      ↓ 통과
  capabilities 검사
      ↓ 통과
  실제 작업 수행
```

### 13-5. 컨테이너 생성 전체 과정

`docker run -d -p 8080:80 nginx` 한 줄이 실제로 하는 일:

```
① Docker CLI
   REST API POST /containers/create → dockerd

② dockerd
   이미지 레이어 확인 → 없으면 pull
   컨테이너 메타데이터 생성
   (/var/lib/docker/containers/<id>/config.v2.json)

③ containerd에게 위임
   OCI Runtime Bundle 생성:
     rootfs/ 준비 (OverlayFS 마운트)
     config.json 생성 (namespaces, cgroup, seccomp 설정 포함)

④ runc 실행
   runc create <container-id>

⑤ runc 내부 — clone() syscall
   clone(CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS |
         CLONE_NEWUTS | CLONE_NEWIPC | SIGCHLD)
   → 자식 프로세스 생성 (새 namespace들 안에서)

⑥ 자식 프로세스 (namespace 안에서 실행):
   a. cgroup 설정
      /sys/fs/cgroup/system.slice/docker-<id>.scope/
      memory.max = 0 (무제한, 기본)
      cpu.weight  = 100

   b. 네트워크 설정
      veth pair 생성
      veth 한쪽 → 컨테이너 NET namespace로 이동
      이름 → eth0으로 변경
      IP 할당 (172.17.0.x)

   c. OverlayFS 마운트
      mount -t overlay overlay \
        -o lowerdir=layer3:layer2:layer1,\
           upperdir=container/diff,\
           workdir=container/work \
        /newroot

   d. /proc, /sys, /dev 마운트 (컨테이너용)

   e. pivot_root /newroot → 루트 교체

   f. hostname 설정 (UTS)

   g. capabilities 드롭

   h. seccomp 필터 로드

   i. exec("/usr/sbin/nginx", ["-g", "daemon off;"])
      → 프로세스 자체가 nginx로 교체됨

⑦ 포트 매핑
   dockerd가 nftables에 규칙 추가:
   호스트:8080 → DNAT → 172.17.0.2:80
```

### 13-6. 직접 namespace 실험

Docker 없이 손으로 컨테이너를 흉내낼 수 있습니다:

```bash
# 1. 새 namespace에서 bash 실행
sudo unshare --pid --fork --mount-proc \
             --net --uts --ipc \
             /bin/bash

# 2. 안에서 확인
hostname                    # 호스트와 같음 (아직 UTS 변경 안 함)
hostname container-test     # 변경
hostname                    # container-test

# 호스트에서 확인
hostname                    # 원래 그대로 → UTS 격리됨

# PID 확인
ps aux                      # PID 1=bash, PID 2=ps → 격리됨

# 3. 네트워크 확인
ip link                     # lo만 존재

# 4. namespace 파일 확인 (호스트에서)
ls -la /proc/<unshare-pid>/ns/
# → 호스트와 다른 inode 번호

# 5. nsenter — 기존 namespace에 진입 (docker exec 원리)
sudo nsenter -t <pid> --pid --net --mount --uts --ipc bash
```

### 13-7. 컨테이너 보안 레이어 전체

```
위협: 컨테이너 안의 악성 프로세스가 호스트 탈출 시도

방어층:
┌─────────────────────────────────────────┐
│ namespace 격리                          │
│  → 호스트 PID, 네트워크, 파일시스템 못 봄│
├─────────────────────────────────────────┤
│ cgroup 제한                             │
│  → CPU/메모리 폭주로 호스트 다운 방지   │
├─────────────────────────────────────────┤
│ capabilities 제거                       │
│  → root여도 마운트, 네트워크 설정 불가  │
├─────────────────────────────────────────┤
│ seccomp 필터                            │
│  → 위험한 syscall 호출 자체를 차단      │
├─────────────────────────────────────────┤
│ SELinux / AppArmor (LSM)                │
│  → 프로세스 타입(httpd_t)마다           │
│     접근 가능한 파일 타입을 제한        │
│  → 컨테이너가 해킹당해도 허용된         │
│     파일 타입 외엔 접근 불가            │
├─────────────────────────────────────────┤
│ rootless 컨테이너 (USER namespace)      │
│  → 탈출해도 호스트에서 일반 사용자      │
└─────────────────────────────────────────┘

→ 여러 레이어가 있기 때문에 하나가 뚫려도 다음 레이어가 막음
→ VM보다는 격리가 약하지만, 레이어를 쌓아서 보완
```

---


## 14. 이해도 점검 — 질의응답

### Q1. 컨테이너 안 nginx가 PID 1인데 호스트에서 보면 번호가 다릅니다. 왜 그런가요?

```
프로세스는 하나, 번호만 다릅니다.

PID namespace가 namespace마다 PID 번호를 별도로 관리합니다.
컨테이너 안에서는 자기 namespace 안의 번호만 보이기 때문에
새 PID namespace의 첫 번째 프로세스는 항상 1번입니다.
호스트에서는 실제 PID(예: 1235)로 보입니다.
```

---

### Q2. firewalld로 8080 포트를 차단했는데 Docker 컨테이너는 여전히 외부에서 접근됩니다. 왜 그런가요?

```
경로 자체가 다릅니다.

Docker가 PREROUTING에서 목적지를 컨테이너 IP(172.17.0.2:80)로 바꿔버립니다.
라우팅 결정 시 "172.17.0.2는 내 IP가 아님" → FORWARD 체인으로 빠집니다.
firewalld의 INPUT 규칙은 이 패킷을 아예 보지 않습니다.

해결: -p 127.0.0.1:8080:80 으로 바인딩 → 외부 접근 차단
```

---

### Q3. 컨테이너를 삭제했더니 안에서 만든 파일들이 사라졌습니다. 왜 그런지, 데이터를 유지하려면 어떻게 해야 하나요?

```
컨테이너 실행 중 변경사항은 전부 upperdir에 쌓입니다.
컨테이너 삭제 시 upperdir도 함께 삭제됩니다.
lowerdir(이미지 레이어)는 삭제되지 않습니다.

해결: 볼륨 마운트
  docker run -v myvolume:/var/lib/mysql mysql:8.0
  → 데이터가 upperdir 대신 볼륨 디렉토리에 저장
  → 컨테이너 삭제해도 볼륨은 유지됨
```

---

### Q4. Dockerfile에서 빌드할 때마다 pip install이 매번 새로 실행됩니다. 왜 그런지, 어떻게 고치나요?

```dockerfile
FROM python:3.11-slim
COPY . .
RUN pip install -r requirements.txt
```

```
COPY . . 가 위에 있어서 소스코드가 바뀔 때마다
아래 RUN pip install 레이어의 캐시도 무효화됩니다.

해결: 의존성 파일을 소스코드보다 먼저 복사

  FROM python:3.11-slim
  COPY requirements.txt .
  RUN pip install -r requirements.txt
  COPY . .                             ← 소스코드는 마지막
```

---

### Q5. `docker exec -it webserver bash`는 내부적으로 어떻게 동작하나요?

```
setns() syscall을 씁니다.

namespace를 다루는 syscall이 3개입니다:
  clone()   → 새 프로세스를 새 namespace에서 생성 (컨테이너 최초 생성 시)
  unshare() → 현재 프로세스의 namespace를 분리
  setns()   → 기존 namespace에 합류 ← docker exec가 이걸 씀

컨테이너의 PID를 찾아 /proc/<pid>/ns/ 파일들을 setns()로 하나씩 합류한 후
그 namespace 안에서 bash를 실행합니다.
```

---

### Q6. 컨테이너 안 프로세스가 root인데 해킹당해도 호스트가 안전한 이유는?

```
여러 레이어가 겹쳐 막습니다.

namespace    탈출해도 호스트 파일시스템, 프로세스, 네트워크를 못 봄
cgroup       CPU/메모리 폭주로 호스트 다운 방지
capabilities root를 기능 단위로 쪼개서 마운트, 네트워크 설정 등 차단
seccomp      위험한 syscall 호출 자체를 화이트리스트로 차단
rootless     USER namespace로 컨테이너 root = 호스트 일반 사용자
```

Q7. 컨테이너 A에서 ping 172.17.0.3은 되는데 ping db는 안 됩니다. 왜 그런지, 해결 방법은?

```
기본 bridge 네트워크에는 내장 DNS가 없기 때문입니다.
IP는 직접 라우팅되므로 됩니다.

해결: 사용자 정의 네트워크 생성

  docker network create mynet
  docker run -d --name db  --network mynet mysql:8.0
  docker run -d --name app --network mynet myapp

사용자 정의 네트워크는 Docker 내장 DNS(127.0.0.11)를 자동으로 제공합니다.
컨테이너 이름이 자동 등록되어 이름으로 통신 가능합니다.
```

Q8. Dockerfile이 있는 디렉토리에 100MB짜리 동영상 파일이 있습니다. 이미지에 포함시킬 필요가 없는데 빌드가 느립니다. 왜 그런지, 해결 방법은?

```
빌드 컨텍스트 전송 때문입니다.

docker build . 에서 . 은 현재 디렉토리 전체를 tar로 묶어서
Docker 데몬에게 전송하겠다는 의미입니다.
COPY 명령어 실행 전에 이미 디렉토리 전체를 전송합니다.
100MB 동영상이 있으면 그만큼 전송 시간이 걸립니다.

해결: .dockerignore 파일로 전송 대상에서 제외

  # .dockerignore
  *.mp4
  *.mov
  node_modules
  .git
```

---

*문서 작성 기준: Rocky Linux 10 / Docker 27 / Podman 5 (2025년 기준)*
*RHEL/Rocky Linux 공식 권장 컨테이너 도구: Podman*
*OCI (Open Container Initiative): 컨테이너 이미지·런타임 표준*