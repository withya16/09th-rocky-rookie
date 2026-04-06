# 네트워크 점검 도구 및 방화벽 정책 설정

---

## 1부: 네트워크 상태 점검 및 문제 해결 도구 심층 분석

과거에는 `net-tools` 패키지(`ifconfig`, `netstat`, `route` 등)가 널리 쓰였으나,
현대 리눅스 커널에서는 처리 속도와 기능적 한계로 인해 **`iproute2` 패키지**(`ip`, `ss` 등)의 사용이 표준으로 자리 잡았습니다.

---

### 1.1 ping (Packet Internet Groper) : 연결성 및 지연 시간 테스트

`ping`은 네트워크 상태를 점검하는 가장 기초적이면서도 강력한 도구입니다.
TCP/UDP가 아닌 **ICMP (Internet Control Message Protocol)** 의 `Echo Request(Type 8)`와 `Echo Reply(Type 0)` 메시지를 사용하여 목적지 호스트와의 통신 가능 여부를 확인합니다.

> **동작 원리:** 송신 측에서 Echo Request 패킷을 보내면, 수신 측 운영체제의 **커널 영역**에서 이를 즉시 처리하여 Echo Reply 패킷을 반환합니다.
> 애플리케이션 계층까지 올라가지 않으므로 순수한 **네트워크 계층의 연결성**을 확인할 수 있습니다.

#### 주요 확인 지표

| 지표 | 설명 |
|------|------|
| **RTT** (Round Trip Time) | 패킷이 목적지에 도달했다가 돌아오는 왕복 시간. `ms(밀리초)` 단위로 표시되며 네트워크 지연(Latency)을 판단하는 핵심 지표 |
| **TTL** (Time To Live) | 패킷의 무한루프를 방지하는 수명 값. 라우터를 하나 거칠 때마다(Hop) 1씩 감소하며, `0`이 되면 패킷 폐기 후 `Time to live exceeded` 오류 반환. 기본값으로 OS 유추 가능 (Linux: 보통 64, Windows: 128) |
| **Packet Loss** (패킷 손실률) | 보낸 패킷 대비 돌아오지 않은 비율. `0%`가 정상이며, 손실 발생 시 케이블 불량, 스위치/라우터 트래픽 병목, 또는 목적지 서버 과부하를 의심 |

#### 실무 활용 옵션

```bash
ping -c 4 8.8.8.8       # 정확히 4번만 전송하고 종료 (스크립트 작성 시 필수)
ping -i 0.2 8.8.8.8     # 패킷 전송 간격을 0.2초로 설정 (기본값: 1초)
ping -s 1472 8.8.8.8    # 페이로드 크기 지정 — MTU 설정 및 패킷 조각화(Fragmentation) 테스트에 활용
```

---

### 1.2 ip 명령어 : 네트워크 스택 종합 제어 (iproute2)

`ip` 명령어는 네트워크 인터페이스, 라우팅 테이블, ARP 캐시 등 리눅스 커널의 네트워크 스택을 직접 제어하고 조회하는 통합 도구입니다.
커널과 **Netlink 소켓**을 통해 통신하므로 응답이 빠르고 정확합니다.

#### `ip link` — 데이터 링크 계층 (L2) 제어

```bash
ip link
```

물리적/논리적 네트워크 인터페이스의 L2 상태(MAC 주소, UP/DOWN 상태, MTU)를 확인합니다.

| 출력 필드 | 의미 |
|----------|------|
| `state UP` | 물리적으로 선이 연결되어 신호가 통하는 상태 |
| `state DOWN` | 랜선이 뽑혀 있거나 스위치 포트가 닫혀 있는 물리적 장애 |
| `promisc` 플래그 | 자신에게 오지 않은 패킷까지 모두 수집 중 — `tcpdump` 실행 중이거나 가상 머신 브릿지로 사용 중일 때 나타남 |

#### `ip addr` (또는 `ip a`) — 네트워크 계층 (L3) 제어

```bash
ip addr
```

- 인터페이스에 할당된 IP 주소와 서브넷 마스크(CIDR 표기)를 확인합니다.
- `ifconfig`와 달리, 하나의 인터페이스에 여러 개의 **보조 IP(Secondary IP)** 가 할당된 내역을 모두 정확하게 표시합니다.

#### `ip route` (또는 `ip r`) — 라우팅 테이블 확인

```bash
ip route
```

패킷이 목적지로 가기 위해 어떤 인터페이스와 게이트웨이를 거쳐야 하는지 커널의 이정표를 보여줍니다.

```
default via 192.168.1.1 dev eth0
```

> 목적지 IP가 라우팅 테이블에 없는 모든 트래픽(인터넷 통신 등)은 `eth0` 장치를 통해 게이트웨이 `192.168.1.1`로 내보낸다는 의미입니다.
> **이 줄이 없다면 외부망과 통신할 수 없습니다.**

#### `ip neigh` (또는 `ip n`) — ARP 테이블 확인

```bash
ip neigh
```

IP 주소(논리적 주소)와 MAC 주소(물리적 주소)의 매핑 테이블(**ARP 캐시**)을 보여줍니다.

특정 IP로 `ping`은 가는데 서비스 접속이 안 될 때, 다른 기기가 동일한 IP를 무단 점유(**IP 충돌**)하여 통신이 엉뚱한 MAC 주소로 가고 있는지 확인할 때 필수적입니다.

| 상태 플래그 | 의미 |
|-----------|------|
| `REACHABLE` | 최근 통신에 성공하여 신뢰할 수 있는 상태 |
| `STALE` | 캐시는 있으나 유효성이 만료되어 검증 필요 |
| `FAILED` | ARP 요청에 응답이 없어 통신 불가 |

---

### 1.3 ss (Socket Statistics) : 소켓 및 포트 상태 정밀 점검

`ss`는 시스템의 활성화된 네트워크 소켓(TCP, UDP, RAW 등) 연결 상태를 보여주는 도구로, 레거시 `netstat`을 완벽하게 대체합니다.
커널의 `tcp_diag` 모듈에서 정보를 직접 가져오므로 동시 접속자가 수만 명인 운영 서버에서도 시스템 부하 없이 빠르게 결과를 출력합니다.

#### TCP 상태(State) 이해

| 상태 | 설명 |
|------|------|
| **LISTEN** | 서버 프로세스가 특정 포트를 열고 클라이언트의 접속(SYN 패킷)을 기다리는 상태. 백엔드 애플리케이션이 정상 구동되었는지 확인할 때 가장 먼저 봐야 할 상태 |
| **ESTABLISHED** | 3-Way Handshake가 완료되어 서버와 클라이언트 간에 데이터가 활발하게 오가는 정상 연결 상태 |
| **TIME_WAIT** | 통신을 먼저 종료(Active Close)한 쪽에서 남은 패킷을 대비해 소켓을 잠시 열어두는 상태. 트래픽이 많은 서버에서 과도하게 누적되면 **포트 고갈(Port Exhaustion)** 장애를 일으킬 수 있어 튜닝 대상이 됨 |

#### 실무 필수 명령어

```bash
ss -tulpn     # 가장 많이 쓰이는 조합
              #   -t : TCP 소켓
              #   -u : UDP 소켓
              #   -l : LISTEN 상태 소켓만 출력
              #   -p : 포트를 열고 있는 프로세스명 및 PID 출력 (관리자 권한 필요)
              #   -n : 포트/IP를 이름이 아닌 숫자 그대로 출력 (속도 향상)

ss -ta        # LISTEN, ESTABLISHED 등 모든 TCP 연결 상태 출력
```

#### 고급 필터링

```bash
ss -t state established           # 현재 통신 중인 ESTABLISHED 소켓만 출력
ss dport = :80 or sport = :80     # 목적지 또는 출발지 포트가 80번인 통신만 필터링
                                  # grep 파이프라인 없이 자체 엔진 사용 → 훨씬 빠름
```

---

## 2부: firewalld를 활용한 동적 방화벽 정책 설정

포트가 `LISTEN` 상태로 열려있고 통신 회선에 문제가 없더라도, **OS 커널 수준에서 패킷을 차단하면 서비스에 접근할 수 없습니다.**
Red Hat 계열(RHEL, CentOS, Rocky, Alma) 및 최신 리눅스 배포판에서 기본으로 사용하는 `firewalld`는 `iptables`나 `nftables`의 복잡한 룰을 **Zone(영역) 기반**으로 쉽게 관리해 주는 동적 방화벽 데몬입니다.

---

### 2.1 firewalld의 아키텍처: Zone (영역) 기반 제어

firewalld의 가장 큰 특징은 트래픽의 출처나 네트워크 인터페이스를 특정 **Zone** 에 할당하고, 그 Zone 단위로 보안 정책(허용/차단)을 적용한다는 것입니다.

| Zone | 신뢰도 | 설명 |
|------|:------:|------|
| **drop** | 최저 | 들어오는 모든 패킷을 폐기. 송신 측에 아무런 응답도 보내지 않아 스텔스 모드처럼 동작 |
| **block** | 낮음 | 들어오는 패킷을 차단하지만, 송신 측에 `icmp-host-prohibited` 오류 메시지를 반환 |
| **public** | 기본 | 신뢰할 수 없는 공개 네트워크. 명시적으로 허용한 포트/서비스 외 모두 차단. 모든 인터페이스의 기본 영역 |
| **trusted** | 최고 | 들어오는 모든 트래픽을 조건 없이 허용. 사내 폐쇄망이나 신뢰할 수 있는 백엔드망에만 제한적으로 사용 |

---

### 2.2 Runtime과 Permanent 구성 (매우 중요)

방화벽 설정 시 가장 많이 겪는 실수는 **시스템 재부팅 후 정책이 초기화되는 현상**입니다.
firewalld는 정책을 메모리(Runtime)와 디스크(Permanent)로 분리하여 관리합니다.

| 구분 | 저장 위치 | 적용 시점 | 재부팅 후 |
|------|----------|----------|----------|
| **Runtime** | 메모리 | 즉시 적용 | 초기화됨 |
| **Permanent** | 디스크 (XML 파일) | `--reload` 후 적용 | 유지됨 |

> **실무 원칙:** 항상 `--permanent` 옵션을 붙여 설정하고, 이후 `firewall-cmd --reload`로 디스크 설정을 메모리에 반영합니다.
> `--permanent` 없이 설정하는 것은 일시적인 테스트 목적에만 사용합니다.

---

### 2.3 기본 및 서비스 제어 명령어

#### 상태 및 기본 정보 확인

```bash
systemctl status firewalld          # 방화벽 데몬 실행 여부 확인
firewall-cmd --get-default-zone     # 현재 시스템의 기본 Zone 확인 (보통 public)
firewall-cmd --list-all             # 기본 Zone의 인터페이스, 허용 서비스, 포트, 차단 규칙 등 전체 출력
```

#### 서비스 및 포트 개방 (영구 적용)

```bash
# 포트로 직접 열기
firewall-cmd --add-port=8080/tcp --permanent
firewall-cmd --reload

# 서비스 이름으로 열기 (firewalld 내부 정의 활용, 예: HTTP = 80/tcp)
firewall-cmd --add-service=http --permanent
firewall-cmd --reload
```

> 차단할 때는 `--add-` 대신 `--remove-`를 사용합니다.

---

### 2.4 심화: Rich Rules (풍부한 규칙)와 포트 포워딩

단순히 포트를 여는 것을 넘어, **"특정 IP에서 오는 특정 포트만 허용/차단"** 하는 정교한 제어가 필요할 때는 **Rich Rule**을 사용합니다.

#### Rich Rule을 이용한 접근 제어

```bash
# 시나리오 1: 특정 IP(192.168.100.50)에서만 DB 포트(3306/tcp) 접근 허용
firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.100.50" port port="3306" protocol="tcp" accept' --permanent
firewall-cmd --reload

# 시나리오 2: 특정 악성 IP(1.1.1.1)의 모든 접근을 영구 차단(Drop)
firewall-cmd --add-rich-rule='rule family="ipv4" source address="1.1.1.1" drop' --permanent
firewall-cmd --reload
```

#### 포트 포워딩 (Port Forwarding)

리눅스 서버로 들어온 특정 포트 트래픽을 다른 포트로 방향을 꺾어주는 기술입니다.

> **배경:** 리눅스 보안 정책상 1024번 이하의 Well-known 포트(예: `80`)는 root 권한으로만 열 수 있습니다.
> 따라서 일반 사용자 권한으로 구동한 **Node.js나 Tomcat(보통 `8080`)** 으로 `80` 포트 트래픽을 넘겨줄 때 방화벽 포워딩을 활용합니다.

```bash
# 80 포트로 들어온 TCP 트래픽을 8080 포트로 포워딩
firewall-cmd --add-forward-port=port=80:proto=tcp:toport=8080 --permanent
firewall-cmd --reload
```

```
외부 클라이언트
      │
      │  :80 (TCP)
      ▼
 [firewalld]
      │  포트 포워딩
      │  :80 → :8080
      ▼
 Spring Boot (:8080)
```

---

## 3부: Ubuntu 환경에서의 방화벽 — ufw

Ubuntu를 포함한 Debian 계열에서는 `firewalld` 대신 **ufw (Uncomplicated Firewall)** 가 기본 방화벽 도구로 사용됩니다.
`ufw`는 `iptables`의 프론트엔드로, 복잡한 `iptables` 룰을 단순한 명령어로 제어할 수 있게 해줍니다.

> `firewalld`와 `ufw` 모두 내부적으로는 커널의 `netfilter` 프레임워크 위에서 동작합니다.
> 인터페이스와 명령어 체계만 다를 뿐, 패킷 필터링의 근본 원리는 동일합니다.

### 기본 명령어

```bash
# ufw 활성화 / 비활성화
sudo ufw enable
sudo ufw disable

# 현재 규칙 및 상태 확인
sudo ufw status verbose

# 포트 허용 / 차단
sudo ufw allow 8080/tcp
sudo ufw deny 8080/tcp

# 서비스 이름으로 허용 (예: OpenSSH, Nginx Full)
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'

# 특정 IP에서 특정 포트만 허용
sudo ufw allow from 192.168.100.50 to any port 3306

# 규칙 삭제
sudo ufw delete allow 8080/tcp
```

> **주의:** `ufw enable` 전에 반드시 SSH 포트(22)를 먼저 허용해야 합니다.
> 그렇지 않으면 원격 서버에 SSH 접속이 차단되어 접근 불가 상태가 될 수 있습니다.
> ```bash
> sudo ufw allow OpenSSH   # 먼저 실행
> sudo ufw enable          # 그 다음 활성화
> ```

### firewalld vs ufw 비교

| 항목 | firewalld | ufw |
|------|-----------|-----|
| 주 사용 배포판 | RHEL, CentOS, Rocky, Alma, Fedora | Ubuntu, Debian |
| 설정 단위 | Zone 기반 | 단순 허용/차단 규칙 |
| 영구 적용 | `--permanent` + `--reload` | 규칙 추가 즉시 영구 저장 |
| 복잡한 규칙 | Rich Rule 지원 | `iptables` 직접 수정 필요 |
| 학습 난이도 | 상대적으로 높음 | 낮음 |
