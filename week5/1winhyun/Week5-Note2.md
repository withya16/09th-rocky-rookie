# NetworkManager 분석 및 활용

---

## 1부: NetworkManager의 이해와 핵심 개념

최신 리눅스 배포판(RHEL/CentOS 7 이상, Ubuntu 18.04 이상 등)에서는 네트워크 설정을 전담하는 데몬인 **NetworkManager**가 표준으로 사용됩니다.
과거 네트워크 스크립트(`/etc/sysconfig/network-scripts/`)를 직접 수정하던 방식에서 벗어나, 시스템의 네트워크 상태를 동적으로 관리하고 API를 통해 일관된 제어를 가능하게 합니다.

---

### 1.1 Device와 Connection의 분리 (매우 중요)

NetworkManager를 다룰 때 가장 먼저, 그리고 완벽하게 이해해야 하는 개념은 **디바이스(Device)** 와 **커넥션(Connection)** 의 차이입니다.

| 개념 | 설명 | 예시 |
|------|------|------|
| **Device** (디바이스) | 물리적인 NIC 또는 가상 인터페이스. 변경할 수 없는 하드웨어/OS 레벨의 자원 | `eth0`, `ens33`, `wlan0`, `virbr0` |
| **Connection** (커넥션/프로필) | 특정 디바이스에 적용할 '네트워크 설정 모음'. IP 주소, 게이트웨이, DNS, MAC 주소 바인딩 등의 정보가 담긴 논리적 프로필 | `Wired connection 1`, `static-eth0` |

> **핵심 원리:** 하나의 디바이스에는 여러 개의 커넥션을 만들어 둘 수 있지만, 특정 순간에 **활성화(Active)** 될 수 있는 커넥션은 **오직 하나**입니다.
> 사무실용 정적 IP 프로필과 집용 DHCP 프로필을 따로 만들어두고, 필요할 때마다 프로필만 교체하여 적용하는 방식이 가능합니다.

---

## 2부: nmtui (NetworkManager Text User Interface) 활용

GUI가 없는 **Headless 서버 환경**에서 가장 직관적으로 네트워크를 설정할 수 있는 Curses 기반의 텍스트 UI 도구입니다.
복잡한 명령어를 외우지 않아도 방향키와 Enter 키만으로 쉽게 설정이 가능합니다.

---

### 2.1 nmtui를 통한 정적 IP 할당 과정 심층 분석

터미널에 `nmtui`를 입력하여 실행한 후, **"Edit a connection"** 메뉴로 진입합니다.

1. **프로필 선택 및 추가**: 기존 프로필을 수정하거나 `Add`를 눌러 새 이더넷 프로필을 생성합니다.

2. **Device 지정**: 이 프로필이 적용될 물리적 디바이스 이름(예: `eth0`)을 정확히 입력해야 합니다.

3. **IPv4 Configuration (핵심)**:
   - 기본값인 `<Automatic>`은 DHCP 서버로부터 IP를 동적으로 할당받음을 의미합니다.
   - 서버 환경에서는 IP가 변경되면 안 되므로 이를 **`<Manual>`** 로 변경하고 `<Show>`를 눌러 상세 탭을 엽니다.

4. **세부 정보 입력**:

   | 항목 | 설명 | 예시 |
   |------|------|------|
   | **Addresses** | CIDR 표기법으로 IP와 서브넷 마스크를 함께 입력 | `192.168.1.100/24` |
   | **Gateway** | 외부 네트워크로 나가는 출구 라우터 IP | `192.168.1.1` |
   | **DNS servers** | 도메인 해석을 위한 DNS IP. 구글 퍼블릭 DNS나 내부망 DNS를 추가 | `8.8.8.8` |

5. **Automatically connect**: 이 옵션을 체크해야 시스템 재부팅 시 이 프로필이 자동으로 장치에 연결(활성화)됩니다.

---

### 2.2 nmtui의 한계점

초기 설정에는 매우 편리하지만, **자동화 스크립트(Bash Script, Ansible 등)에 태울 수 없다**는 치명적인 단점이 있습니다.
대규모 인프라 프로비저닝 시에는 사용할 수 없으므로 반드시 다음 장의 `nmcli`를 숙지해야 합니다.

---

## 3부: nmcli (NetworkManager Command Line Interface) 심층 제어

명령줄 기반의 제어 도구로, NetworkManager의 모든 기능을 스크립트화하고 제어할 수 있는 **가장 강력한 툴**입니다.

---

### 3.1 상태 확인 명령어

| 명령어 | 설명 |
|--------|------|
| `nmcli general status` | NetworkManager 데몬의 전반적인 상태(연결됨, 연결 안 됨 등) 확인 |
| `nmcli device status` (또는 `nmcli dev`) | 시스템에 인식된 네트워크 디바이스 목록과 현재 매핑된 커넥션 상태 확인 |
| `nmcli connection show` (또는 `nmcli con`) | 생성되어 있는 모든 논리적 커넥션(프로필) 목록 확인. 녹색으로 표시되는 것이 현재 활성화된 프로필 |

> **심화:** `nmcli con show "프로필명"`을 입력하면 해당 프로필의 수백 가지 세부 설정 파라미터를 모두 확인할 수 있습니다.

---

### 3.2 nmcli를 이용한 정적 IP(Static IP) 상세 설정 플로우

새로운 커넥션을 만들고 정적 IP를 할당하는 전체 프로세스입니다.

#### 1단계: 새로운 커넥션 생성

```bash
nmcli connection add type ethernet con-name static-eth0 ifname eth0
```

| 옵션 | 설명 |
|------|------|
| `type ethernet` | 이더넷 연결 유형을 지정 |
| `con-name static-eth0` | 관리할 논리적 프로필의 이름 지정 (자유롭게 작명 가능) |
| `ifname eth0` | 이 프로필을 물리적 장치 `eth0`에 바인딩 |

#### 2단계: IPv4 설정 변경 (동적 → 정적)

```bash
nmcli connection modify static-eth0 ipv4.method manual
```

> 가장 중요한 단계입니다. DHCP(자동)를 끄고 **수동(manual)** 모드로 전환해야 사용자가 입력한 고정 IP가 적용됩니다.

#### 3단계: IP, 게이트웨이, DNS 할당

```bash
nmcli connection modify static-eth0 ipv4.addresses 192.168.50.10/24
nmcli connection modify static-eth0 ipv4.gateway 192.168.50.1
nmcli connection modify static-eth0 ipv4.dns "8.8.8.8 8.8.4.4"
```

> `modify` 명령어를 통해 파라미터를 덮어씁니다. DNS를 여러 개 넣을 때는 큰따옴표로 묶고 띄어쓰기로 구분합니다.

#### 4단계: 설정 적용 (커넥션 활성화)

```bash
nmcli connection up static-eth0
```

> 변경된 프로필을 디바이스에 적용(재시작)하여 네트워크에 연결합니다. 이 명령어가 성공적으로 실행되어야 실제 통신이 시작됩니다.

---

### 3.3 파라미터 추가/삭제의 미학 (`+`와 `-` 연산자)

기존에 설정된 IP나 DNS를 삭제하지 않고 **'추가'** 만 하고 싶을 때는 속성명 앞에 `+`를 붙입니다.
장애 조치(Failover)를 위한 보조 IP나 보조 DNS를 구성할 때 유용합니다.

```bash
# 보조 IP 추가
nmcli con mod static-eth0 +ipv4.addresses 192.168.50.11/24

# 기존 DNS 제거
nmcli con mod static-eth0 -ipv4.dns 8.8.4.4
```

---

## 4부: 클라우드 및 인프라 환경에서의 실무적 맥락

AWS, Oracle Cloud, Kakao Cloud와 같은 퍼블릭 클라우드나 온프레미스 인프라를 다룰 때 네트워크 인터페이스 제어는 백엔드 개발자 및 인프라 엔지니어의 핵심 역량입니다.

---

### 4.1 클라우드 인스턴스와 `/etc/resolv.conf`의 관계

NetworkManager는 활성화된 커넥션의 DNS 정보를 바탕으로 시스템의 전역 DNS 설정 파일인 `/etc/resolv.conf`를 **동적으로 덮어씁니다.**

> **주의:** 초보자들이 실수로 `vi /etc/resolv.conf`를 직접 수정하여 DNS를 바꾸는 경우가 있는데,
> NetworkManager가 재시작되거나 커넥션이 다시 올라오면 수동으로 적은 내용이 **모두 초기화**됩니다.
> 따라서 반드시 `nmcli`나 `nmtui`를 통해 **프로필 자체의 `dns` 파라미터를 수정**해야 영구적으로 적용됩니다.

---

### 4.2 오류 추적 및 트러블슈팅 (Troubleshooting)

설정을 마쳤는데 외부(예: `8.8.8.8`)로 ping이 나가지 않는다면 다음을 **순차적으로** 확인합니다.

```
[1] 링크 상태 확인
    └─ ip link
       물리적 디바이스가 UP 상태인지 확인

[2] IP 할당 확인
    └─ ip addr
       설정한 IP가 디바이스에 정상적으로 올라갔는지 확인

[3] 게이트웨이/라우팅 확인
    └─ ip route
       default via <게이트웨이IP> 라우팅 룰이 존재하는지 확인
       (서로 다른 대역망과 통신하는 백엔드 API 서버 구축 시 라우팅 누락이 잦은 원인)

[4] 데몬 로그 분석
    └─ journalctl -u NetworkManager -f
       NetworkManager의 실시간 로그를 보면서
       인터페이스가 왜 down 되는지, DHCP IP 충돌이 있는 것은 아닌지 추적
```
