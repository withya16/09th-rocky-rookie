> **목표:** OSI Layer부터 IP/TCP/UDP/DNS/Routing의 근본 원리를 이해하고, Rocky Linux에서 NetworkManager로 Interface를 구성하며, firewalld로 트래픽을 제어하고, AWS VPC / K8s Networking과 연결하는 것
**환경:** Rocky Linux (RHEL 10) / aarch64 / NVMe / 2 CPU, 3.5 GiB RAM
> 

---

# Part 1. 네트워크 기본 개념

## 1-1. OSI 7 Layer vs TCP/IP 4 Layer

네트워크의 모든 통신은 **계층(Layer)** 구조로 동작합니다. 각 Layer는 자기 역할만 수행하고, 아래 Layer에게 전달을 위임합니다. 이 구조를 이해해야 "어디서 문제가 발생했는지" 체계적으로 진단할 수 있습니다.

| OSI Layer | TCP/IP Layer | 역할 | 대표 Protocol | 장비/개념 |
| --- | --- | --- | --- | --- |
| 7. Application | Application | 사용자가 직접 접하는 서비스 | HTTP, HTTPS, SSH, DNS, MQTT | Nginx, curl |
| 6. Presentation | Application | 데이터 형식 변환, 암호화 | TLS/SSL, JPEG, JSON | 인코딩/디코딩 |
| 5. Session | Application | 연결 세션 관리 | NetBIOS | Socket 세션 |
| 4. Transport | Transport | 종단 간 데이터 전달. **Port 기반** | **TCP**, **UDP** | Port 번호 |
| 3. Network | Internet | **IP 주소** 기반 Routing | **IP**, ICMP, ARP | Router, L3 Switch |
| 2. Data Link | Network Access | **MAC 주소** 기반 Frame 전달 | Ethernet, Wi-Fi | Switch, NIC |
| 1. Physical | Network Access | 전기/광 신호 전송 | - | 케이블, 허브 |

**OSI vs TCP/IP — 왜 두 모델이 존재하는가:**

OSI 7 Layer는 ISO가 정의한 **이론적 참조 모델**입니다. 실제 구현과 정확히 일치하지 않지만, 네트워크 문제를 계층적으로 분석할 때 가장 유용한 프레임워크입니다. TCP/IP 4 Layer는 **실제 인터넷이 동작하는 구현 모델**입니다. OSI의 5-7층을 Application으로, 1-2층을 Network Access로 합쳤습니다.

실무에서는 OSI Layer 번호로 소통합니다: "L3 문제인 것 같아" (Routing 문제), "L4 Load Balancer" (TCP/UDP 기반 분산), "L7 방화벽" (HTTP 내용까지 검사).

**데이터가 Layer를 통과하는 과정 — Encapsulation:**

웹 브라우저가 `GET /index.html`을 보낼 때:

```
[Application]  HTTP Request: "GET /index.html"
     ↓ Encapsulation
[Transport]    TCP Header(Src Port:49152, Dst Port:80) + HTTP Data
     ↓
[Network]      IP Header(Src:192.168.1.100, Dst:93.184.216.34) + TCP + HTTP
     ↓
[Data Link]    Ethernet Header(Src MAC, Dst MAC) + IP + TCP + HTTP + FCS
     ↓
[Physical]     전기/광 신호로 변환하여 전송
```

각 Layer가 자기 Header를 앞에 붙이고(Encapsulation), 수신 측에서는 아래부터 Header를 벗겨내며(Decapsulation) 위로 올립니다.

> **K8s 연결:** K8s의 Overlay Network(예: VXLAN)는 이 Encapsulation을 한 번 더 합니다. Pod 간 통신 시 원본 Ethernet Frame을 UDP 패킷 안에 넣어 전송합니다 — "패킷 안의 패킷". 이것이 Pod이 다른 Node에 있어도 같은 네트워크처럼 통신할 수 있는 원리입니다.
> 

**Troubleshooting 시 Layer별 진단 — 반드시 아래부터 위로:**

```
문제: "웹사이트에 접속이 안 돼요"

Layer 1-2: 물리 연결
  → ip link show ens33 → Interface가 UP인가? Link detected?
  → ethtool ens33 → Speed, Duplex, Link 상태

Layer 3: IP/Routing
  → ip addr show → IP가 할당되어 있는가?
  → ping <Gateway IP> → 같은 네트워크 내 통신 가능?
  → ping 8.8.8.8 → 외부 인터넷 Routing 가능?

Layer 4: TCP/Port
  → ss -tlnp → Service가 해당 Port에서 Listen 중?
  → nc -zv host 80 → TCP Handshake 성공?

Layer 7: Application/DNS
  → dig example.com → DNS Resolution 성공?
  → curl -v <https://example.com> → HTTP 응답 정상?
```

이 순서를 기억하세요 — **아래 Layer부터 위로** 올라가며 진단합니다. Layer 3이 안 되면 Layer 7은 당연히 안 됩니다.

```go
mr8356@mr8356:~$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 00:0c:29:df:be:37 brd ff:ff:ff:ff:ff:ff
    altname enp2s0
    altname enx000c29dfbe37
mr8356@mr8356:~$ ip link show ens160

2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 00:0c:29:df:be:37 brd ff:ff:ff:ff:ff:ff
    altname enp2s0
    altname enx000c29dfbe37

mr8356@mr8356:~$ ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:df:be:37 brd ff:ff:ff:ff:ff:ff
    altname enp2s0
    altname enx000c29dfbe37
    inet 192.168.15.100/24 brd 192.168.15.255 scope global noprefixroute ens160
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fedf:be37/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
mr8356@mr8356:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=128 time=77.1 ms
^C
--- 8.8.8.8 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 77.105/77.105/77.105/0.000 ms

mr8356@mr8356:~$ ss -tlnp
State       Recv-Q      Send-Q           Local Address:Port            Peer Address:Port      Process
LISTEN      0           128                    0.0.0.0:22                   0.0.0.0:*
LISTEN      0           128                       [::]:22                      [::]:*
```

---

## 1-2. IP 주소 체계

### IPv4 주소 구조

IPv4 주소는 **32bit 숫자**를 8bit(Octet)씩 4개로 나누어 점(.)으로 구분합니다: `192.168.1.100`

각 Octet은 0~255 범위입니다. 총 약 43억(2^32)개 주소가 가능하지만, 인터넷 장치 폭증으로 고갈되어 NAT와 Private IP가 필수가 되었습니다.

### CIDR (Classless Inter-Domain Routing) 표기법

IP 주소 뒤에 `/숫자`를 붙여 Network 부분의 bit 수를 표시합니다.

| CIDR | Subnet Mask | Host 수 | 용도 |
| --- | --- | --- | --- |
| `/8` | 255.0.0.0 | 16,777,214 | 대규모 네트워크 (10.0.0.0/8) |
| `/16` | 255.255.0.0 | 65,534 | **AWS VPC 기본 CIDR** |
| `/24` | 255.255.255.0 | 254 | 일반적인 Subnet |
| `/28` | 255.255.255.240 | 14 | AWS의 최소 Subnet 크기 |
| `/32` | 255.255.255.255 | 1 | 단일 Host (SG 규칙 등) |

**계산 원리:** `/24`면 앞 24bit가 Network 주소, 뒤 8bit(2^8 - 2 = 254)가 Host 주소입니다. 2개를 빼는 이유: Network 주소(Host 부분 전부 0)와 Broadcast 주소(전부 1)는 할당 불가.

**예시 — 192.168.1.0/24:**

```
네트워크 주소:   192.168.1.0       (할당 불가)
첫 번째 Host:   192.168.1.1       (보통 Gateway)
마지막 Host:    192.168.1.254
Broadcast:     192.168.1.255     (할당 불가)
사용 가능:      254개
```

### 서브넷팅 예시 — /24를 4개의 /26으로 분할

```
원래: 192.168.1.0/24 (254 hosts)
분할:
  192.168.1.0/26    → .0 ~ .63     (62 hosts)
  192.168.1.64/26   → .64 ~ .127   (62 hosts)
  192.168.1.128/26  → .128 ~ .191  (62 hosts)
  192.168.1.192/26  → .192 ~ .255  (62 hosts)
```

### Private IP 대역 — 인터넷에서 직접 접근 불가한 주소

| 대역 | 범위 | 주 사용처 |
| --- | --- | --- |
| `10.0.0.0/8` | 10.0.0.0 ~ 10.255.255.255 | **AWS VPC 기본 CIDR**, 대규모 사내망 |
| `172.16.0.0/12` | 172.16.0.0 ~ 172.31.255.255 | **Docker 기본 Bridge** (`172.17.0.0/16`) |
| `192.168.0.0/16` | 192.168.0.0 ~ 192.168.255.255 | 가정/소규모 사무실 |

이 대역의 IP는 인터넷 Router가 라우팅하지 않습니다. 외부와 통신하려면 **NAT(Network Address Translation)**를 거쳐 Public IP로 변환해야 합니다.

> **AWS VPC 연결:** VPC 생성 시 이 Private IP 대역에서 CIDR Block을 선택합니다. 보통 `10.0.0.0/16` (65,536개 IP). VPC Peering 시 CIDR이 겹치면 안 되므로, 여러 VPC를 운영할 때는 `10.0.0.0/16`, `10.1.0.0/16`, `10.2.0.0/16` 등으로 분리합니다.
> 

> **K8s 연결:** K8s Cluster는 **3개의 독립된 IP 대역**을 사용합니다:
> 
> 1. **Node IP** — EC2 Instance의 VPC IP (예: `10.0.11.x`)
> 2. **Pod IP** — Pod에 할당되는 별도 CIDR (예: `10.244.0.0/16`)
> 3. **Service IP** — ClusterIP에 할당되는 가상 CIDR (예: `10.96.0.0/12`)
> 
> 세 대역이 겹치면 라우팅 충돌. EKS는 VPC CNI로 Pod에 VPC IP를 직접 할당하므로, Subnet IP가 빠르게 소진될 수 있어 `/24`보다 큰 Subnet 권장.
> 

### IPv6 기초

IPv4 고갈을 근본 해결하기 위한 **128bit** 주소. 약 3.4 × 10^38개.

표기: `2001:0db8:85a3:0000:0000:8a2e:0370:7334` (16bit × 8그룹, 콜론)
축약: 연속 0 그룹은 `::`로 한 번 축약 → `2001:db8:85a3::8a2e:370:7334`
Loopback: `::1` (IPv4의 `127.0.0.1`)
Link-local: `fe80::/10` (같은 네트워크 내에서만 유효)

```bash
ping -6 ::1 -c 4
ip -6 addr show

mr8356@mr8356:~$ ping -6 ::1 -c 4
PING ::1 (::1) 56 data bytes
64 bytes from ::1: icmp_seq=1 ttl=64 time=0.329 ms
64 bytes from ::1: icmp_seq=2 ttl=64 time=0.135 ms
64 bytes from ::1: icmp_seq=3 ttl=64 time=0.055 ms
^C
--- ::1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2088ms
rtt min/avg/max/mdev = 0.055/0.173/0.329/0.115 ms

mr8356@mr8356:~$ ip -6 addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN qlen 1000
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
    inet6 fe80::20c:29ff:fedf:be37/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

---

## 1-3. TCP vs UDP — Transport Layer의 두 핵심 Protocol

### TCP (Transmission Control Protocol) — 신뢰성 있는 연결 기반

**3-Way Handshake — TCP 연결 수립:**

```
Client                    Server
  │                         │
  │── SYN (seq=100) ──────→ │   1단계: "연결하고 싶다"
  │                         │
  │←── SYN-ACK ──────────── │   2단계: "좋아, 나도 준비됐다"
  │    (seq=300, ack=101)   │         (Client의 seq+1을 ack로)
  │                         │
  │── ACK (ack=301) ──────→ │   3단계: "확인, 시작하자"
  │                         │
  │←── Data Transfer ─────→ │   이후 데이터 교환
```

seq(Sequence Number)와 ack(Acknowledgment Number)로 데이터 순서와 수신 확인을 추적합니다. 패킷 유실 시 재전송(Retransmission)합니다.

**4-Way Handshake — TCP 연결 종료:**

```
Client                    Server
  │── FIN ───────────────→│   "보낼 거 없어"
  │←── ACK ───────────────│   "알겠어"
  │←── FIN ───────────────│   "나도 없어"
  │── ACK ───────────────→│   "끊자"
  │                         │
  │ (TIME_WAIT 2MSL 대기)   │   Client가 일정 시간 대기 후 완전 종료
```

> **SRE 중요:** `TIME_WAIT` 상태의 TCP Connection이 대량으로 쌓이면 **Ephemeral Port 고갈**이 발생합니다.
> 

```bash
# TIME_WAIT 수 확인
ss -tn state time-wait | wc -l
1

# Kernel Parameter로 완화
sudo sysctl -w net.ipv4.tcp_tw_reuse=1
```

> **K8s 연결:** K8s NodePort Service에서 트래픽이 많으면 TIME_WAIT 누적 → Port 고갈 → 새 연결 실패. `tcp_tw_reuse` 활성화 또는 LoadBalancer Type 전환 필요.
> 

### UDP (User Datagram Protocol) — 비연결, 비신뢰

Handshake 없이 데이터를 바로 보냅니다. 도착 보장 없지만 빠릅니다.

| 비교 | TCP | UDP |
| --- | --- | --- |
| 연결 | Connection-oriented (Handshake) | Connectionless |
| 신뢰성 | 순서 보장, 재전송 | 보장 없음 |
| 속도 | 상대적 느림 (Overhead) | 빠름 |
| Header | 20 bytes | 8 bytes |
| 용도 | HTTP, SSH, DB | DNS 질의, 스트리밍, 게임, NTP |

### 주요 Port 번호

| Port | Protocol | Service | 비고 |
| --- | --- | --- | --- |
| 22 | TCP | SSH | EC2 접속, SCP |
| 80 | TCP | HTTP | 비암호화 웹 |
| 443 | TCP | HTTPS | TLS 암호화 웹 |
| 53 | **TCP/UDP** | DNS | 질의=UDP, Zone Transfer=TCP |
| 25 | TCP | SMTP | 이메일 |
| 3306 | TCP | MySQL | RDS MySQL |
| 5432 | TCP | PostgreSQL | RDS PostgreSQL |
| 6379 | TCP | Redis | ElastiCache |
| 6443 | TCP | **K8s API Server** | kubectl 통신 |
| 10250 | TCP | **kubelet** | Control Plane → Node |
| 2379-2380 | TCP | **etcd** | K8s State 저장소 |
| 9090 | TCP | Prometheus | Web UI/API |
| 3000 | TCP | Grafana | Dashboard |
| 123 | UDP | NTP | 시간 동기화 |

```bash
sudo ss -tlnp              # TCP Listen Port + Process
sudo ss -tlnp | grep :22   # 특정 Port
sudo ss -ulnp              # UDP Listen

mr8356@mr8356:~$ sudo ss -tlnp | grep  :22
LISTEN 0      128          0.0.0.0:22        0.0.0.0:*    users:(("sshd",pid=1178,fd=7))
LISTEN 0      128             [::]:22           [::]:*    users:(("sshd",pid=1178,fd=8))

mr8356@mr8356:~$ sudo ss -ulnp
State    Recv-Q   Send-Q     Local Address:Port     Peer Address:Port  Process
UNCONN   0        0              127.0.0.1:323           0.0.0.0:*      users:(("chronyd",pid=1101,fd=5))
UNCONN   0        0                  [::1]:323              [::]:*      users:(("chronyd",pid=1101,fd=6))
```

---

## 1-4. DNS — Domain Name을 IP로 변환

### DNS 질의 순서

```
1. Local DNS Cache (systemd-resolved, nscd)
   ↓ 없으면
2. /etc/hosts 파일 (로컬 정적 매핑)
   ↓ 없으면
3. /etc/resolv.conf에 지정된 DNS Server에 질의
   ↓ DNS Server 내부
4. Root DNS → TLD DNS (.com) → Authoritative DNS (google.com)

mr8356@mr8356:~$ cat /etc/resolv.conf
# Generated by NetworkManager
nameserver 8.8.8.8
nameserver 8.8.4.4
```

이 순서를 결정하는 파일: `/etc/nsswitch.conf`

```bash
grep hosts /etc/nsswitch.conf
# hosts: files dns myhostname
# files = /etc/hosts 먼저, dns = resolv.conf

mr8356@mr8356:~$ cat /etc/nsswitch.conf
# Generated by authselect
# Do not modify this file manually, use authselect instead. Any user changes will be overwritten.
# You can stop authselect from managing your configuration by calling 'authselect opt-out'.
# See authselect(8) for more details.

# In order of likelihood of use to accelerate lookup.
passwd:     files systemd
shadow:     files
group:      files [SUCCESS=merge] systemd
hosts:      files  dns myhostname
services:   files
netgroup:   files
automount:  files

aliases:    files
ethers:     files
gshadow:    files
networks:   files dns
protocols:  files
publickey:  files
rpc:        files
```

### 핵심 파일

```bash
# /etc/hosts — 로컬 DNS Override (DNS Server보다 우선)
cat /etc/hosts
# 127.0.0.1   localhost
# 10.0.1.50   mydb.internal    ← 내부 서버 이름 매핑

# /etc/resolv.conf — 사용하는 DNS Server
cat /etc/resolv.conf
# nameserver 8.8.8.8
# nameserver 8.8.4.4
# search example.com   ← hostname만 입력 시 자동 붙는 도메인
```

### DNS 질의 도구

```bash
dig google.com              # 가장 상세한 출력
dig google.com +short       # IP만
dig @8.8.8.8 google.com     # 특정 DNS Server로 질의

nslookup google.com         # 간단
host google.com             # 더 간단

# Record Type별 질의
dig google.com A            # IPv4
dig google.com AAAA         # IPv6
dig google.com MX           # 메일 서버
dig google.com NS           # 네임서버
dig google.com CNAME        # 별칭
dig google.com TXT          # SPF, DKIM 등
```

> **AWS 연결:** VPC 내부 `/etc/resolv.conf`의 `nameserver`는 **VPC CIDR의 +2 주소** (예: `10.0.0.2`) = **AmazonProvidedDNS**. Route 53 Private Hosted Zone 연동.
> 

> **K8s 연결:** Pod 내부 `/etc/resolv.conf`는 **CoreDNS** (보통 `10.96.0.10`)를 가리킴. `my-service.my-namespace.svc.cluster.local` Service Discovery. `search` 필드에 `default.svc.cluster.local svc.cluster.local cluster.local`이 설정되어 `my-service`만 입력해도 FQDN으로 확장.
> 

---

## 1-5. Routing — 패킷이 목적지까지 가는 경로

### Routing Table 읽기

```bash
ip route show
# 출력 예시:
# default via 192.168.1.1 dev ens33 proto dhcp metric 100
# 192.168.1.0/24 dev ens33 proto kernel scope link src 192.168.1.100

mr8356@mr8356:~$ ip route show
default via 192.168.15.2 dev ens160 proto static metric 100
192.168.15.0/24 dev ens160 proto kernel scope link src 192.168.15.100 metric 100
```

| 필드 | 의미 |
| --- | --- |
| `default via 192.168.1.1` | Default Gateway — 명시적 경로 없는 모든 패킷이 가는 곳 |
| `192.168.1.0/24 dev ens33` | 이 네트워크는 ens33으로 직접 접근 |
| `proto dhcp` | DHCP로 학습한 경로 |
| `proto kernel` | Kernel이 Interface IP에서 자동 생성 |
| `scope link` | 같은 Link 내에서만 유효 |

**Default Gateway:** 목적지 IP가 어떤 경로와도 매칭 안 될 때 패킷을 보내는 **마지막 관문**. EC2에서는 Subnet의 첫 IP(예: `10.0.1.1`).

```bash
# 정적 Route 추가 (임시 — 재부팅 시 사라짐)
sudo ip route add 10.10.0.0/16 via 192.168.1.1
sudo ip route del 10.10.0.0/16

mr8356@mr8356:~$ sudo ip route add 10.10.0.0/16 via 192.168.15.100

mr8356@mr8356:~$ ip route show
default via 192.168.15.2 dev ens160 proto static metric 100
10.10.0.0/16 via 192.168.15.100 dev ens160
192.168.15.0/24 dev ens160 proto kernel scope link src 192.168.15.100 metric 100

mr8356@mr8356:~$ sudo ip route del 10.10.0.0/16

# IP Forwarding 활성화 (Router/NAT로 동작하려면 필수)
sudo sysctl -w net.ipv4.ip_forward=1
# 영구:
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-forward.conf
sudo sysctl --system

# 경로 추적
traceroute 8.8.8.8
tracepath 8.8.8.8    # root 불필요
```

> **AWS 연결:** `ip route` = AWS **Route Table**. VPC 각 Subnet에 Route Table 연결. `0.0.0.0/0 → igw-xxx` = Public Subnet, `0.0.0.0/0 → nat-xxx` = Private Subnet. `ip_forward=1`은 AWS **NAT Instance** 만들 때 EC2에 설정하는 것과 동일.
> 

---

## 1-6. ARP — IP 주소를 MAC 주소로 변환

같은 L2 네트워크에서 Frame을 전달하려면 상대방의 **MAC 주소**가 필요합니다. ARP는 "이 IP의 MAC이 뭐야?" Broadcast 질의 Protocol.

```
1. Host A: "10.0.1.5의 MAC 주소 알려줘" → Broadcast
2. Host B(10.0.1.5): "내 MAC은 aa:bb:cc:dd:ee:ff" → Unicast 응답
3. Host A: ARP Cache에 저장 → 이후 Frame에 이 MAC 사용
```

```bash
ip neigh show              # ARP Cache 확인
sudo ip neigh flush all    # Cache 초기화 (디버깅 시)

mr8356@mr8356:~$ ip neigh show
192.168.15.2 dev ens160 lladdr 00:50:56:f1:f7:12 REACHABLE
10.10.0.8 dev ens160 FAILED
192.168.15.1 dev ens160 lladdr 82:a9:97:81:39:68 DELAY
10.10.0.0 dev ens160 FAILED
```

---

# Part 2. Network Interface 구성

## 2-1. Interface Naming 규칙

### 구식 vs 현대식

| 방식 | 예시 | 특징 |
| --- | --- | --- |
| 전통적 | `eth0`, `eth1` | Kernel 감지 순서. 재부팅 시 바뀔 수 있음 |
| **Predictable Names** | `ens33`, `enp0s3`, `eno1` | 물리적 위치 기반. 재부팅해도 동일 |

**해석:**

- `en` = Ethernet, `wl` = Wireless, `ww` = WWAN
- `o<N>` = Onboard (내장): `eno1`
- `s<N>` = PCI Slot: `ens33`
- `p<B>s<S>` = PCI Bus/Slot: `enp0s3` (Bus 0, Slot 3)

```bash
ip link show               # Interface 목록
sudo ethtool ens33         # NIC 물리 정보 (Speed, Duplex, Link)

mr8356@mr8356:~$ sudo ethtool ens160
Settings for ens160:
	Supported ports: [ TP ]
	Supported link modes:   10baseT/Half 10baseT/Full
	                        100baseT/Half 100baseT/Full
	                        1000baseT/Full
	Supported pause frame use: Symmetric Receive-only
	Supports auto-negotiation: Yes
	Supported FEC modes: Not reported
	Advertised link modes:  10baseT/Half 10baseT/Full
	                        100baseT/Half 100baseT/Full
	                        1000baseT/Full
	Advertised pause frame use: Symmetric Receive-only
	Advertised auto-negotiation: Yes
	Advertised FEC modes: Not reported
	Link partner advertised link modes:  10baseT/Half 10baseT/Full
	                                     100baseT/Half 100baseT/Full
	                                     1000baseT/Half 1000baseT/Full
	Link partner advertised pause frame use: No
	Link partner advertised auto-negotiation: Yes
	Link partner advertised FEC modes: Not reported
	Speed: 1000Mb/s
	Duplex: Full
	Auto-negotiation: on
	Port: Twisted Pair
	PHYAD: 1
	Transceiver: internal
	MDI-X: off (auto)
	Supports Wake-on: pumbg
	Wake-on: g
        Current message level: 0x00000007 (7)
                               drv probe link
	Link detected: yes
```

## 2-2. NetworkManager — RHEL의 네트워크 관리 데몬

RHEL 10에서 네트워크 설정은 **NetworkManager** Daemon이 담당합니다.

**핵심 개념: Device vs Connection**

| 개념 | 의미 | 비유 |
| --- | --- | --- |
| **Device** | 물리적/가상 NIC | 하드웨어 |
| **Connection** (Profile) | Device에 적용하는 설정 묶음 | 설정 프로필 (IP, DNS, GW) |

하나의 Device에 여러 Connection Profile을 만들어두고 전환 가능 (사무실 Profile ↔ 카페 Profile).

```bash
sudo systemctl status NetworkManager
nmcli general status           # 전체 상태
nmcli device status            # Device 목록
nmcli connection show          # Connection Profile 목록
nmcli connection show ens33    # 특정 Connection 상세

mr8356@mr8356:~$ nmcli general status
STATE      CONNECTIVITY  WIFI-HW  WIFI     WWAN-HW  WWAN     METERED
connected  full          missing  enabled  missing  enabled  no (guessed)
mr8356@mr8356:~$ nmcli device status
DEVICE  TYPE      STATE                   CONNECTION
ens160  ethernet  connected               ens160
lo      loopback  connected (externally)  lo
mr8356@mr8356:~$ nmcli connection show
NAME    UUID                                  TYPE      DEVICE
ens160  867f2b5a-4fab-3ae8-a0ed-735b0bc94856  ethernet  ens160
lo      491c892c-6f88-4ad2-89f2-bc93a8c3dfca  loopback  lo

● NetworkManager.service - Network Manager
     Loaded: loaded (/usr/lib/systemd/system/NetworkManager.service; enabled; preset: enabled)
     Active: active (running) since Fri 2026-04-03 03:10:01 KST; 8h left
 Invocation: 1c656e164a854862a5ba8d295a0162cd
       Docs: man:NetworkManager(8)
   Main PID: 1157 (NetworkManager)
      Tasks: 4 (limit: 22577)
     Memory: 8.6M (peak: 9M)
        CPU: 710ms
     CGroup: /system.slice/NetworkManager.service
             └─1157 /usr/sbin/NetworkManager --no-daemon

Apr 03 03:10:01 mr8356 NetworkManager[1157]: <info>  [1775153401.1604] device (ens160): state change: prepare -> config (reason 'none', managed-type: 'f>
Apr 03 03:10:01 mr8356 NetworkManager[1157]: <info>  [1775153401.1656] device (ens160): state change: config -> ip-config (reason 'none', managed-type: >
Apr 03 03:10:01 mr8356 NetworkManager[1157]: <info>  [1775153401.1663] policy: set 'ens160' (ens160) as default for IPv4 routing and DNS
Apr 03 03:10:01 mr8356 NetworkManager[1157]: <info>  [1775153401.3062] device (ens160): state change: ip-config -> ip-check (reason 'none', managed-type>
Apr 03 03:10:01 mr8356 NetworkManager[1157]: <info>  [1775153401.3079] device (ens160): state change: ip-check -> secondaries (reason 'none', managed-ty>
Apr 03 03:10:01 mr8356 NetworkManager[1157]: <info>  [1775153401.3080] device (ens160): state change: secondaries -> activated (reason 'none', managed-t>
Apr 03 03:10:01 mr8356 NetworkManager[1157]: <info>  [1775153401.3082] manager: NetworkManager state is now CONNECTED_SITE
Apr 03 03:10:01 mr8356 NetworkManager[1157]: <info>  [1775153401.3083] device (ens160): Activation: successful, device activated.
Apr 03 03:10:01 mr8356 NetworkManager[1157]: <info>  [1775153401.3085] manager: NetworkManager state is now CONNECTED_GLOBAL
Apr 03 03:10:01 mr8356 NetworkManager[1157]: <info>  [1775153401.3087] manager: startup complete

```

**설정 파일 위치 (RHEL 10):** `/etc/NetworkManager/system-connections/*.nmconnection`

```bash
sudo cat /etc/NetworkManager/system-connections/*.nmconnection
```

---

# Part 3. nmcli / nmtui 실습

## 3-1. nmcli — CLI 기반 네트워크 설정

### Static IP 설정

```bash
# 현재 설정 확인
nmcli connection show ens33 | grep -E 'ipv4\\.(addresses|gateway|dns|method)'

# Static IP 설정
sudo nmcli connection modify ens33 \\
  ipv4.addresses 192.168.1.100/24 \\
  ipv4.gateway 192.168.1.1 \\
  ipv4.dns "8.8.8.8 8.8.4.4" \\
  ipv4.method manual

# 적용 (Connection 재활성화)
sudo nmcli connection down ens33
sudo nmcli connection up ens33

# 확인
nmcli connection show ens33 | grep ipv4
ip addr show ens33
ip route show
cat /etc/resolv.conf
```

### DHCP로 변경

```bash
sudo nmcli connection modify ens33 \\
  ipv4.method auto \\
  ipv4.addresses "" \\
  ipv4.gateway "" \\
  ipv4.dns ""
sudo nmcli connection up ens33
```

### 새 Connection Profile 생성

```bash
sudo nmcli connection add \\
  con-name "my-static" \\
  type ethernet \\
  ifname ens33 \\
  ipv4.addresses 10.0.0.50/24 \\
  ipv4.gateway 10.0.0.1 \\
  ipv4.dns "8.8.8.8" \\
  ipv4.method manual

nmcli connection show                # 생성 확인
sudo nmcli connection up my-static   # 활성화
sudo nmcli connection up ens33       # 원래로 복귀
```

### DNS 추가/제거

```bash
sudo nmcli connection modify ens33 +ipv4.dns 1.1.1.1   # 추가
sudo nmcli connection modify ens33 -ipv4.dns 1.1.1.1   # 제거
sudo nmcli connection up ens33
```

### 삭제

```bash
sudo nmcli connection delete my-static
```

## 3-2. nmtui — TUI 기반

```bash
sudo nmtui
```

메뉴: Edit a connection / Activate a connection / Set system hostname

> **언제 뭘 쓰나:** 스크립트 자동화 → `nmcli`. 빠른 수동 변경 → `nmtui`. Ansible 배포 → 설정 파일 직접.
> 

## 3-3. Bonding (고급 — NIC 이중화)

```bash
# Bond Interface 생성 (active-backup 모드)
sudo nmcli connection add type bond con-name bond0 ifname bond0 \\
  bond.options "mode=active-backup,miimon=100"

# Slave Interface 추가
sudo nmcli connection add type ethernet con-name bond0-port1 \\
  ifname ens33 master bond0
sudo nmcli connection add type ethernet con-name bond0-port2 \\
  ifname ens34 master bond0

# Bond에 IP 설정
sudo nmcli connection modify bond0 \\
  ipv4.addresses 192.168.1.100/24 \\
  ipv4.gateway 192.168.1.1 \\
  ipv4.method manual

sudo nmcli connection up bond0
```

| Bond Mode | 동작 | 용도 |
| --- | --- | --- |
| `active-backup` | 한 NIC 활성, 장애 시 전환 | **가용성** (가장 보편) |
| `balance-rr` | Round-Robin 분산 | **대역폭** |
| `802.3ad` (LACP) | Switch와 협상하여 Link Aggregation | **가용성+대역폭** (Switch 지원 필요) |

---

# Part 4. 네트워크 상태 점검 도구

## 4-1. ip — ifconfig/route/arp 통합 대체

```bash
# === IP 주소 관리 ===
ip addr show                    # 모든 Interface IP (= ip a)
ip addr show ens33              # 특정 Interface
ip addr add 192.168.1.200/24 dev ens33    # 임시 IP 추가 (재부팅 시 사라짐!)
ip addr del 192.168.1.200/24 dev ens33

# === Interface 관리 ===
ip link show                    # Interface 상태 (UP/DOWN, MTU, MAC)
sudo ip link set ens33 down     # Interface 비활성화 (SSH 끊김 주의!)
sudo ip link set ens33 up

# === Routing ===
ip route show                   # Routing Table (= ip r)
sudo ip route add 10.10.0.0/16 via 192.168.1.1
sudo ip route del 10.10.0.0/16
sudo ip route add default via 192.168.1.1

# === ARP ===
ip neigh show                   # ARP Cache
sudo ip neigh flush all
```

> **중요:** `ip` 명령어로 한 설정은 모두 **임시**. 재부팅 시 사라짐. **영구** = `nmcli connection modify`.
> 

## 4-2. ss — netstat 대체

`ss`는 `/proc`를 직접 읽어 netstat보다 **훨씬 빠르고** 정보 풍부.

```bash
sudo ss -tlnp                   # TCP Listen + Process (★ 가장 자주)
# -t: TCP, -l: Listen, -n: 숫자 표시, -p: Process
sudo ss -ulnp                   # UDP Listen
ss -tn state established        # 현재 연결
ss dst 10.0.0.1                 # 특정 목적지 연결
ss -s                           # Socket 통계 요약
ss -tn state time-wait | wc -l  # TIME_WAIT 수

# 특정 Port
sudo ss -tlnp | grep :8080
```

## 4-3. 연결 테스트 도구

```bash
# === ping — L3 연결 ===
ping -c 4 8.8.8.8              # 4번만
ping -c 4 -W 2 192.168.1.1     # Timeout 2초
ping6 ::1                       # IPv6

# === traceroute — 경로 추적 ===
traceroute 8.8.8.8
tracepath 8.8.8.8               # root 불필요, PMTU 감지

# === curl — HTTP 연결 ===
curl -v <https://example.com>     # HTTP 상세 (TLS Handshake 포함)
curl -I <https://example.com>     # Header만
curl -o /dev/null -s -w "%{http_code}\\n" <https://example.com>  # Status Code만

# === nc (netcat) — Port 테스트 ===
nc -zv 192.168.1.1 80           # TCP Port 열림 확인
nc -zv 192.168.1.1 22           # SSH Port
nc -zvu 192.168.1.1 53          # UDP Port

# === 기타 ===
ethtool ens33                   # NIC 물리 정보
mtr 8.8.8.8                    # ping + traceroute 합친 도구
```

## 4-4. Kernel 네트워크 파라미터 (/proc/sys/net/)

```bash
# 현재 값 확인
sysctl net.ipv4.ip_forward
sysctl net.ipv4.tcp_tw_reuse

# 임시 변경
sudo sysctl -w net.ipv4.ip_forward=1

# 영구 변경
sudo tee /etc/sysctl.d/99-network-tuning.conf << 'EOF'
# Router/NAT 역할
net.ipv4.ip_forward = 1

# TIME_WAIT Socket 재사용
net.ipv4.tcp_tw_reuse = 1

# TCP Keepalive 간격 (7200초 → 300초)
net.ipv4.tcp_keepalive_time = 300

# SYN Backlog (동시 연결 요청 대기열)
net.core.somaxconn = 65535

# 최대 File Descriptor
fs.file-max = 1000000
EOF

sudo sysctl --system    # 적용
```

---

# Part 5. 방화벽 (firewalld)

## 5-1. firewalld 아키텍처

```
사용자 도구:   firewall-cmd (CLI) / firewall-config (GUI)
      ↓
관리 데몬:     firewalld (D-Bus 기반)
      ↓
Kernel 백엔드: nftables (RHEL 9/10) 또는 iptables (RHEL 8)
```

firewalld는 **Frontend**. 실제 패킷 필터링은 Kernel의 **nftables**가 수행.

> **AWS 매핑:**
> 
> - `firewalld` = **Security Group** (Instance 레벨, **Stateful**)
> - `iptables FORWARD chain` = **Network ACL** (Subnet 레벨, **Stateless**)
> - SG는 인바운드 허용 → 아웃바운드 자동 허용 (Stateful), NACL은 각각 설정 (Stateless)

| **구분** | **리눅스 호스트 (firewalld / iptables)** | **AWS 클라우드 인프라 (SG / NACL)** |
| --- | --- | --- |
| **적용 범위** | **OS 내부** (특정 서버 1대) | **인프라 레벨** (VPC 내부) |
| **검사 위치** | 커널의 INPUT / OUTPUT 체인 | 인스턴스 가상 NIC (ENI) |
| **매핑** | **firewalld** (Stateful) | **Security Group (SG)** |
| **특징** | 연결 상태를 기억함 (편함) | 허용하면 응답은 자동 허용 |
| **검사 위치** | 커널의 **FORWARD** 체인 | 서브넷 경계 (Subnet Boundary) |
| **매핑** | **iptables FORWARD** (Stateless) | **Network ACL (NACL)** |
| **특징** | 패킷 하나하나만 검사함 (깐깐함) | 들어올 때, 나갈 때 다 설정해야 함 |

## 5-2. Zone 기반 방화벽

Interface를 Zone에 할당하고, Zone별로 다른 정책을 적용합니다.

| Zone | 기본 동작 | 사용 시나리오 |
| --- | --- | --- |
| `drop` | 모든 Incoming Drop (응답 없음) | 가장 엄격 |
| `block` | 모든 Incoming Reject (ICMP Error 응답) | 엄격하지만 응답 |
| `public` | **기본 Zone.** SSH, DHCPv6만 허용 | 인터넷 노출 서버 |
| `external` | Masquerading(NAT) + SSH만 | NAT Gateway 역할 |
| `dmz` | SSH만 | DMZ 서버 |
| `work` / `home` | SSH + 일부 서비스 | 사무실/가정 |
| `internal` | home과 동일 | 내부 네트워크 |
| `trusted` | **모든 트래픽 허용** | 완전 신뢰 네트워크 |

```bash
sudo systemctl status firewalld
sudo firewall-cmd --state
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --get-zones
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all            # ★ 현재 Zone 전체 정책
sudo firewall-cmd --list-all --zone=trusted
```

## 5-3. Service / Port 허용

### -permanent vs Runtime

| 방식 | 동작 | 재부팅 후 |
| --- | --- | --- |
| `--permanent` 없이 | 즉시 적용 | 사라짐 |
| `--permanent` | 파일 저장. **`--reload` 해야 적용** | 유지 |

```bash
# 사전 정의된 Service 목록
sudo firewall-cmd --get-services

# Service 허용 (영구)
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --add-service=https --permanent

# Service 제거
sudo firewall-cmd --remove-service=http --permanent

# Port 직접 허용
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --add-port=9090/tcp --permanent     # Prometheus
sudo firewall-cmd --add-port=6443/tcp --permanent     # K8s API Server
sudo firewall-cmd --add-port=10250/tcp --permanent    # kubelet

# Port 제거
sudo firewall-cmd --remove-port=8080/tcp --permanent

# ★ --permanent 후 반드시 Reload!
sudo firewall-cmd --reload

# 확인
sudo firewall-cmd --list-services
sudo firewall-cmd --list-ports
sudo firewall-cmd --list-all
```

## 5-4. Rich Rule — 세밀한 트래픽 제어

Source IP, Port, Protocol, Action을 조합한 **세밀한 규칙**.

```bash
# 특정 네트워크에서만 MySQL 허용
sudo firewall-cmd --permanent --add-rich-rule='
  rule family="ipv4"
  source address="192.168.1.0/24"
  port protocol="tcp" port="3306"
  accept'

# 특정 IP 완전 차단
sudo firewall-cmd --permanent --add-rich-rule='
  rule family="ipv4"
  source address="10.0.0.100"
  drop'

# 특정 IP에서만 SSH 허용
sudo firewall-cmd --permanent --add-rich-rule='
  rule family="ipv4"
  source address="203.0.113.50/32"
  service name="ssh"
  accept'

# Rate Limiting — 1분에 10개 연결만 허용
sudo firewall-cmd --permanent --add-rich-rule='
  rule family="ipv4"
  service name="ssh"
  accept
  limit value="10/m"'

sudo firewall-cmd --reload
sudo firewall-cmd --list-rich-rules
```

> **AWS SG 비교:** SG에서 "MySQL은 10.0.1.0/24만 허용" = 위 Rich Rule과 동일 개념. 차이: SG는 **Deny 불가** (허용만), Rich Rule은 **drop/reject 가능**.
> 

## 5-5. Port Forwarding과 Masquerading (NAT)

```bash
# Port Forwarding — 외부 8080 → 내부 80
sudo firewall-cmd --permanent --add-forward-port=port=8080:proto=tcp:toport=80

# 다른 Host로 전달 (ip_forward 필수)
sudo firewall-cmd --permanent --add-forward-port=port=8080:proto=tcp:toport=80:toaddr=10.0.0.5

# Masquerading (SNAT/NAT) 활성화
sudo firewall-cmd --permanent --add-masquerade

sudo firewall-cmd --reload
```

> **AWS 매핑:** Port Forwarding = **ALB/NLB Target Group** Port Mapping. Masquerading = **NAT Gateway**. Private Subnet EC2가 인터넷 접근 시 NAT GW가 Public IP로 Source를 변환 = Masquerading.
> 

## 5-6. Zone 변경

```bash
sudo firewall-cmd --zone=trusted --change-interface=ens33 --permanent
sudo firewall-cmd --set-default-zone=internal
sudo firewall-cmd --reload
```

## 5-7. firewalld vs iptables vs nftables

```
사용자 → firewall-cmd → firewalld → nftables (RHEL 10)
                                    → iptables (RHEL 8 이전)
```

| 도구 | 역할 | RHEL 버전 |
| --- | --- | --- |
| `firewall-cmd` | High-level 관리. Zone/Service 개념 | RHEL 7+ (권장) |
| `nftables` / `nft` | Kernel Level 패킷 필터. firewalld Backend | RHEL 9/10 |
| `iptables` | 구세대 패킷 필터 | RHEL 8 이전 (Legacy) |

```bash
# nftables 규칙 직접 확인
sudo nft list ruleset | head -50

# iptables 호환 명령
sudo iptables -L -n -v
sudo iptables -t nat -L -n -v
```

> **K8s 연결:** kube-proxy는 Service ClusterIP → Pod IP NAT를 위해 **iptables 규칙** 또는 **IPVS**를 사용. `iptables -t nat -L -n`으로 kube-proxy가 생성한 NAT 규칙 확인 가능. 이것이 K8s Service Networking의 핵심 구현.
> 

## 5-8. SELinux와 네트워크

비표준 Port에서 Service 실행 시 **SELinux가 차단**합니다.

```bash
# Apache를 8888 Port에서 실행하려면
sudo semanage port -a -t http_port_t -p tcp 8888
sudo semanage port -l | grep http

# 현재 SELinux 상태 확인
getenforce
```

---

# Part 6. AWS VPC 네트워킹 연계 (SAA 수준)

## 6-1. Linux ↔ AWS 개념 1:1 매핑

| Linux 개념 | AWS 대응 | 설명 |
| --- | --- | --- |
| `ip route` (Routing Table) | **Route Table** | 패킷 경로 결정 |
| `firewalld` (Host 방화벽) | **Security Group** | Instance(ENI) 레벨, **Stateful** |
| `iptables FORWARD chain` | **Network ACL** | Subnet 레벨, **Stateless** |
| `ip addr` (IP 할당) | **ENI** (Elastic Network Interface) | Instance에 부착되는 가상 NIC |
| NAT / Masquerading | **NAT Gateway** | Private Subnet → Internet |
| `/etc/resolv.conf` | **AmazonProvidedDNS** / Route 53 | VPC 내부 DNS |
| Bonding / Teaming | **ENI 다중 부착** | Instance에 여러 NIC |
| `sysctl ip_forward=1` | **NAT Instance** Source/Dest Check OFF | EC2를 Router로 사용 |

## 6-2. VPC 핵심 구성요소

### CIDR Block 설계

| CIDR | IP 수 | 평가 |
| --- | --- | --- |
| `10.0.0.0/16` | 65,536 | **권장.** 충분한 여유 |
| `10.0.0.0/24` | 256 | 너무 작음. Subnet 분할 어려움 |
| `10.0.0.0/8` | 16,777,216 | 너무 큼. Peering 시 겹침 위험 |

### Subnet 분할 전략

```
VPC: 10.0.0.0/16

Public Subnet (IGW 연결 — 인터넷 직접 접근)
├── 10.0.1.0/24  (AZ-a)  — ALB, Bastion Host
├── 10.0.2.0/24  (AZ-b)  — ALB (Multi-AZ)

Private Subnet (NAT GW 통해 외부)
├── 10.0.11.0/24 (AZ-a)  — App Server, EKS Worker
├── 10.0.12.0/24 (AZ-b)  — App Server (Multi-AZ)

Isolated Subnet (외부 연결 없음)
├── 10.0.21.0/24 (AZ-a)  — RDS, ElastiCache
├── 10.0.22.0/24 (AZ-b)  — RDS (Multi-AZ)
```

**AWS Subnet당 예약 IP 5개:**

- `.0` — Network 주소
- `.1` — VPC Router (Default Gateway)
- `.2` — AWS DNS Server
- `.3` — AWS 예약
- `.255` — Broadcast (VPC 미지원이지만 예약)

→ `/24` Subnet에서 실제 사용 가능: **251개** (256 - 5)

## 6-3. Security Group vs Network ACL ★★★

| 항목 | Security Group | Network ACL |
| --- | --- | --- |
| 적용 대상 | ENI (Instance) 레벨 | **Subnet** 레벨 |
| Stateful | **Stateful** — 인바운드 허용 → 아웃바운드 자동 | **Stateless** — 인/아웃 각각 설정 |
| 규칙 유형 | **허용만** (Deny 불가) | 허용 + **거부** 가능 |
| 규칙 평가 | 모든 규칙 평가 | **규칙 번호 순서대로** (첫 매칭) |
| 기본 동작 | 모든 Outbound 허용, Inbound 거부 | Default NACL: 모두 허용 |
| Linux 대응 | `firewalld` (Stateful) | `iptables` (Stateless) |

**실무 패턴:** SG를 **주력 방화벽**으로 사용. NACL은 **Subnet 단위 대규모 차단**(특정 IP 대역 차단)에만 사용. NACL은 Stateless라 Ephemeral Port(1024-65535) Return Traffic 규칙을 별도로 열어야 해서 관리가 번거로움.

## 6-4. NAT Gateway vs NAT Instance

| 항목 | NAT Gateway | NAT Instance |
| --- | --- | --- |
| 관리 | **AWS Managed** | 직접 EC2 관리 |
| 가용성 | AZ 내 자동 이중화 | 직접 ASG 구성 |
| 성능 | **최대 100 Gbps** | Instance Type 의존 |
| 비용 | 시간당 + 데이터 처리량 | Instance 비용만 (저렴 가능) |
| Source/Dest Check | 자동 비활성화 | **수동 비활성화 필수** |
| Linux 대응 | - | `iptables MASQUERADE` + `ip_forward=1` |

> **NAT Instance = Linux NAT:** EC2에서 `net.ipv4.ip_forward=1` + `iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE` 설정이 정확히 NAT Instance. AWS Console에서 Source/Destination Check 비활성화 필수 (기본적으로 자기 IP 아닌 패킷 Drop하므로).
> 

## 6-5. VPC Peering / Transit Gateway

| 연결 방식 | 설명 | CIDR 제약 |
| --- | --- | --- |
| **VPC Peering** | VPC 간 1:1 직접 연결 | **CIDR 겹침 불가** |
| **Transit Gateway** | 허브형 연결. 여러 VPC/VPN 중앙 관리 | CIDR 겹침 불가 |
| **VPN** | IPSec 터널 (온프레미스 ↔ AWS) | - |
| **Direct Connect** | 전용 물리 회선 | 안정적, 비쌈 |

## 6-6. 네트워크 Troubleshooting 순서 — Linux + AWS 통합

```
1. ip a / ip link show
   → Interface UP? IP 할당?
   → AWS: EC2 Console에서 ENI 상태

2. ping <Gateway IP>
   → 같은 Subnet 내 통신?
   → AWS: VPC Router (x.x.x.1)

3. ping 8.8.8.8
   → 외부 Routing?
   → AWS: Route Table에 0.0.0.0/0 → IGW/NAT 경로

4. dig google.com
   → DNS Resolution?
   → AWS: VPC DNS, Route 53 Resolver

5. ss -tlnp
   → 앱이 Port에서 Listen?
   → AWS: EC2 내부 확인

6. firewall-cmd --list-all
   → OS 방화벽 차단?
   → AWS: **Security Group** 인바운드 규칙

7. curl -v <http://target>:port
   → Application 레벨 연결?
   → AWS: ALB Health Check, **NACL** 규칙

8. (AWS 추가) VPC Flow Logs
   → 네트워크 레벨에서 패킷이 Accept/Reject 되었는지 확인
```

이 순서를 **아래 Layer(물리/IP)부터 위 Layer(Application)로** 올라갑니다. Layer 3에서 막혀 있으면 Layer 7을 아무리 봐도 소용없습니다.