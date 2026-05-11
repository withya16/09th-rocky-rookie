## **1. 컨테이너 vs 가상머신**

### 가상머신 (Virtual Machine)

- 하드웨어 위에 **Hypervisor** → 그 위에 Guest OS 전체 설치
- VM마다 독립된 OS 커널 보유 → 격리 강함
- 부팅 수십 초 ~ 수 분, 용량 수 GB → 무거움
- 대표 도구 : VMware, VirtualBox
- 완전한 OS 환경이 필요한 경우에 적합

```bash
┌─────────────────────────────────────────────────────────────────┐
│                        가상머신 (VM)                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                    │
│  │  App A   │   │  App B   │   │  App C   │                    │
│  ├──────────┤   ├──────────┤   ├──────────┤                    │
│  │Guest OS  │   │Guest OS  │   │Guest OS  │  ← 각자 OS 커널 보유 │
│  │ (커널)   │   │ (커널)   │   │ (커널)   │                    │
│  └──────────┘   └──────────┘   └──────────┘                    │
│  ─────────────────────────────────────────                      │
│               Hypervisor                    ← 하드웨어 가상화     │
│  ─────────────────────────────────────────                      │
│                  Host OS                                         │
│  ─────────────────────────────────────────                      │
│                  Hardware                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 컨테이너 (Container)

- 어플리케이션 코드 라이브러리 및 종속성과 함께 포함하는 소프트웨어의 경량 패키지
- Host OS 커널 공유 → Guest OS 설치 불필요
- 앱 실행에 필요한 라이브러리 + 파일만 묶어서 격리된 프로세스로 실행
- 대표 도구 : Docker, Podman
- 이미지 하나로 어느 환경에서나 동일 실행

```bash
┌─────────────────────────────────────────────────────────────────┐
│                       컨테이너 (Container)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                    │
│  │  App A   │   │  App B   │   │  App C   │                    │
│  ├──────────┤   ├──────────┤   ├──────────┤                    │
│  │  Libs    │   │  Libs    │   │  Libs    │  ← 라이브러리만 포함  │
│  └──────────┘   └──────────┘   └──────────┘                    │
│  ─────────────────────────────────────────                      │
│          컨테이너 런타임 (Docker / Podman)                        │
│  ─────────────────────────────────────────                      │
│        Host OS 커널 (Namespace + cgroups)   ← 커널 공유          │
│  ─────────────────────────────────────────                      │
│                  Hardware                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 핵심 차이

- OS 커널 공유 여부
- 가상머신 : 커널 각자 보유 → 보안 격리 강하나 무겁고 느림
- 컨테이너 : 커널 공유 → 가볍고 빠르나 커널 취약점 공유 위험
- CI/CD·마이크로서비스에는 컨테이너가 압도적으로 유리

---

## **2. Docker & Podman 개념 및 기본 사용법**

### **Docker**

- 가장 널리 쓰이는 컨테이너 플랫폼
- Docker Daemon(dockerd)이 백그라운드에서 상시 실행

```bash
# 설치 확인
docker --version
docker info
```

### **Podman**

- Red Hat 주도, Daemonless 구조
- Docker 명령어와 거의 완전히 호환
- Rootless 실행 지원 → 일반 사용자 권한으로도 컨테이너 실행 가능

```bash
# 설치 확인
podman --version
podman info

# Docker 명령어 그대로 쓰기
alias docker=podman
```

---

## **3. 이미지 관리**

- **이미지:** 컨테이너를 생성할 때 필요한 요소로 컨테이너의 목적에 맞는 바이너리와 의존성이 설치되어 있음
- **레지스트리(Registry)**: 컨테이너 이미지를 저장하고 배포하는 저장소 서버
    - 퍼블릭(공개) / 프라이빗(사내) 레지스트리로 구분
    - Docker Hub, GitHub Container Registry, Amazon ECR Public 등

**이미지 검색**

```bash
docker search nginx
docker search ubuntu --filter=is-official=true # 공식 이미지만
```

- Docker Hub 웹사이트에서도 검색가능

**이미지 다운로드 (Pull)**

```bash
docker pull nginx
docker pull nginx:1.25
```

- 버전 태그가 없을 경우 lastest(최신버전)

**이미지 목록 확인**

```bash
docker images
docker image ls
```

**태그 지정**

```bash
docker tag nginx:latest my-nginx:v1.0
```

**이미지 삭제**

```bash
docker rmi nginx
docker rmi a6bd71f48f68
```

- 이름 또는 ID로 삭제가능

**이미지 저장 / 불러오기**

```bash
docker save nginx:latest -o nginx.tar    # 파일로 저장
docker load -i nginx.tar                 # 파일에서 불러오기
```

---

## **4. 컨테이너 관리**

**컨테이너 실행 (run)**

```bash
docker run nginx
```

- -d : 백그라운드 실행
- -it : 언터랙티브 터미널 (직접 입력 가능한 터미널)
- --name : 컨테이너 이름 지정
- --rm : 종료 시 자동삭제 (일회성 컨테이너)
- -e KEY=VALUE : 환경변수 설정
- --restart alwasys : 자동 재시작

**포트지정**

```bash
docker run -p [호스트포트]:[컨테이너포트] 이미지명

docker run -d -p 8080:80 nginx
# 모든 인터페이스가 아닌 특정 IP만 바인딩
docker run -d -p 127.0.0.1:8080:80 nginx
```

- -P :  포트 랜덤지정

**컨테이너 조회**

```bash
docker ps                                
docker container ls
```

- -a : 종료된 컨테이너 포함 전체 목록

**컨테이너 생성 (create)**

```bash
docker create --name nginx nginx
```

- 컨테이너 이름 / 이미지 이름 순서
- -p : local에 이미지가 없다면 hub에서 pull한 후 create

**컨테이너 시작 (start)**

```bash
docker container start nginx
```

**컨테이너 정지 (pause/unpause)**

```bash
docker container pause nginx
docker container unpause nginx
```

**컨테이너 정지 (stop)**

```bash
docker container stop nginx
```

**컨테이너 삭제 (rm)**

```bash
docker rm my-nginx
```

- 중지된 컨테이너만 가능
- -f : 강제 종료 (실행중인 컨테이너도 가능)

**실행 중인 컨테이너 접속**

```bash
docker exec -it nginx bash            # bash 쉘 접속
docker exec -it nginx sh              # sh 쉘 (bash 없을 때)
```

**로그 확인**

```bash
docker logs nginx
docker logs -f nginx                  # 실시간 팔로우
docker logs --tail 50 nginx           # 최근 50줄
```

---

## **5. 볼륨 연결**

컨테이너가 삭제되어도 데이터를 보존하거나, 호스트와 파일을 공유하는 방법

### **바인드 마운트 (Bind Mount)**

- 호스트의 특정 디렉터리를 컨테이너에 직접 연결

```bash
docker run -d \
  -p 8080:80 \
  -v /host/html:/usr/share/nginx/html \
  --name web nginx
```

### **Docker Volume (관리형 볼륨)**

- Docker가 직접 관리하는 볼륨 → 데이터 영속성에 권장

```bash
# 볼륨 생성
docker volume create mydata

# 볼륨 목록
docker volume ls

# 볼륨 상세 정보
docker volume inspect mydata

# 볼륨을 컨테이너에 연결
docker run -d \
  -v mydata:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  --name db mysql:8.0

# 볼륨 삭제
docker volume rm mydata
```

---

## **6. 기본 네트워크 설정**

**기본 제공 네트워크 종류**

```bash
docker network ls
```

- 컨테이너 생성 시에 네트워크 정의 가능
- 기본은 bridge 네트워크

```bash
docker run --network my-network nginx
```

| 드라이버 | 설명 |
| --- | --- |
| **bridge** | 기본값. 컨테이너끼리 격리된 사설 네트워크 구성 |
| **host** | 호스트 네트워크를 직접 사용 (포트 매핑 불필요) |
| **none** | 네트워크 완전 차단 |
| **overlay** | Swarm/K8s에서 멀티 호스트 네트워킹 |

**사용자 정의 네트워크 생성**

```bash
# 네트워크 생성
docker network create my-network

# 서브넷/게이트웨이 지정
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  my-custom-net

# 네트워크 상세 정보
docker network inspect my-network
```

**컨테이너를 네트워크에 연결**

```bash
# 실행 중인 컨테이너에 네트워크 추가
docker network connect my-network existing-container

# 네트워크에서 분리
docker network disconnect my-network existing-container
```

**컨테이너 간 통신**

- 다른 네트워크에 있는 컨테이너 간 통신은 기본적으로 불가능
- 같은 네트워크에 있는 컨테이너 간 통신은 가능
    - 같은 네트워크에 두면 Docker DNS 가능: name으로 통신가능

**네트워크 삭제**

```bash
docker network rm my-network
```
