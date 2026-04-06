# systemd 심층 분석 및 서비스 관리

---

## 1부: systemd의 탄생 배경과 핵심 아키텍처

과거 리눅스는 부팅 시 `/etc/init.d/`에 있는 쉘 스크립트들을 번호 순서대로 **순차적으로** 실행했습니다(SysVinit).
이는 스크립트가 무거워질수록 부팅 속도가 치명적으로 느려지고, 프로세스 간의 복잡한 의존성(예: DB가 켜진 후 웹 서버가 켜져야 함)을 관리하기 어렵다는 단점이 있었습니다.

**systemd**는 이러한 문제를 해결하기 위해 등장한 **시스템 및 서비스 관리자**입니다.

---

### 1.1 systemd의 핵심 특징

| 특징 | 설명 |
|------|------|
| **PID 1번 점유** | 커널이 부팅된 후 가장 먼저 실행되는 프로세스(PID 1). 이후 생성되는 모든 사용자 공간 프로세스는 직간접적으로 systemd의 자식 프로세스가 됨 |
| **병렬 실행** | 소켓(Socket)과 D-Bus(메시지 버스)를 활용하여 의존성이 없는 서비스들을 동시에 병렬 시작. 클라우드 인스턴스 부팅 시간을 획기적으로 단축 |
| **Cgroups 기반 추적** | 특정 서비스에서 파생된 모든 프로세스를 하나의 그룹으로 묶어, 서비스 중지 시 고아 프로세스(Orphan Process) 없이 깔끔하게 종료 |

---

### 1.2 유닛(Unit)의 개념과 종류

systemd는 관리하는 모든 리소스를 **유닛(Unit)** 이라는 객체로 추상화하여 관리합니다.
유닛은 이름 뒤에 확장자(Type)를 가져 그 역할을 구분합니다.

| 유닛 타입 | 설명 |
|----------|------|
| **`.service`** | 백엔드 애플리케이션(Spring Boot, Nginx 등)과 같은 시스템 데몬을 시작, 중지, 재시작하는 가장 대표적인 유닛 |
| **`.target`** | 여러 유닛을 논리적으로 그룹화한 것. 과거의 런레벨(Runlevel)을 대체. (예: `multi-user.target` - 텍스트 모드, `graphical.target` - GUI 모드) |
| **`.socket`** | 네트워크/IPC 소켓을 감시하다가 연결 요청이 들어오면 그때 연관된 `.service`를 깨워 실행 (On-demand). 자원 낭비를 방지하는 패턴 |
| **`.timer`** | `cron`을 대체하는 스케줄러 유닛. 특정 시간이나 부팅 후 일정 시간이 지났을 때 연관 서비스를 실행. 초 단위 정밀 제어 가능 |
| **`.mount`** | 시스템 부팅 시 파일 시스템 마운트 지점 제어. `/etc/fstab`의 내용을 읽어 동적으로 마운트 유닛을 생성 |

---

## 2부: 서비스 유닛(`.service`) 파일의 심층 분석

직접 개발한 백엔드 애플리케이션을 리눅스 서버에서 안정적으로 24시간 백그라운드 구동하려면 커스텀 `.service` 파일을 작성할 줄 알아야 합니다.

---

### 2.1 유닛 파일의 위치 우선순위 (매우 중요)

| 경로 | 용도 | 수정 여부 |
|------|------|----------|
| `/usr/lib/systemd/system/` | 패키지 관리자(`yum`, `apt`)로 설치된 소프트웨어의 기본 유닛 파일 | **직접 수정 금지** (업데이트 시 덮어씌워짐) |
| `/etc/systemd/system/` | 시스템 관리자가 직접 생성하거나 수정한 커스텀 유닛 파일. **가장 높은 우선순위**로 기존 설정을 오버라이드 | 이곳에 작성 |

---

### 2.2 `.service` 파일 구조와 핵심 지시어 (Spring Boot 예시)

```ini
# /etc/systemd/system/my-springboot.service

[Unit]
Description=My Spring Boot Backend Service
After=network.target syslog.target

[Service]
Type=simple
User=springuser
Group=springgroup
WorkingDirectory=/opt/backend
ExecStart=/usr/bin/java -Xms512m -Xmx1024m -jar /opt/backend/app.jar
Restart=on-failure
RestartSec=10
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=my-backend

[Install]
WantedBy=multi-user.target
```

#### `[Unit]` 섹션 — 메타데이터와 의존성

| 지시어 | 설명 |
|--------|------|
| `Description=` | 서비스 설명. 로그나 상태 확인 시 노출됨 |
| `After=` | 서비스 시작 순서 결정. `network.target` 명시 시 네트워크 IP 할당 이후에 서비스 시작 보장. DB 연결 필수 시 `After=mysql.service` 추가 가능 |

#### `[Service]` 섹션 — 프로세스 실행 제어

**`Type=` (프로세스 실행 타입):**

| 타입 | 설명 | 적합한 경우 |
|------|------|------------|
| `simple` | `ExecStart` 명령어가 메인 프로세스. 명령어 실행 즉시 시작 성공으로 간주 | Spring Boot `java -jar` 등 포그라운드 실행 |
| `forking` | 부모 프로세스가 자식을 생성하고 부모는 종료되는 전통적 데몬화 방식. `PIDFile=` 옵션 함께 사용 권장 | Nginx, Tomcat 등 |
| `notify` | 애플리케이션 내부에서 systemd로 준비 완료 신호(`sd_notify`)를 보내야만 시작된 것으로 간주 | 완전한 초기화 확인이 필요한 경우 |

**주요 지시어:**

| 지시어 | 설명 |
|--------|------|
| `User=` / `Group=` | **최소 권한 원칙.** 백엔드 앱을 `root`로 구동하는 것은 매우 위험하므로 전용 계정 명시 |
| `ExecStart=` | 실행할 명령어의 **절대 경로**. systemd 환경은 쉘의 `PATH`를 상속받지 않으므로 반드시 절대 경로 사용 |
| `Restart=on-failure` | 프로세스가 오류(OOM, 비정상 크래시 등)로 죽었을 때 **자동 재시작**. 정상 종료(exit code 0)에는 재시작하지 않음 |
| `RestartSec=10` | 재시작 전 10초 대기. 즉시 재시작 시 발생할 수 있는 연쇄 장애(DB 일시 단절 등) 방지 |

#### `[Install]` 섹션 — 자동 활성화 지점

| 지시어 | 설명 |
|--------|------|
| `WantedBy=multi-user.target` | `systemctl enable` 시 부팅 단계 중 `multi-user.target`(일반적인 다중 사용자 CLI 모드)에 도달하면 이 서비스를 실행 |

---

## 3부: systemctl 제어 명령어 심층 활용

---

### 3.1 상태 점검: `systemctl status 서비스명`

단순히 켜졌는지 꺼졌는지 확인하는 것을 넘어, 출력 정보를 정확히 해석해야 합니다.

| 출력 필드 | 설명 |
|----------|------|
| **Loaded** | 유닛 파일 경로 및 부팅 시 자동 실행 상태(`enabled` / `disabled`) 표시 |
| **Active: active (running)** | 정상적으로 메모리에 떠서 실행 중 |
| **Active: active (exited)** | 실행은 성공했으나, 1회성 스크립트(`Type=oneshot`)가 완료된 상태 |
| **Active: failed** | 실행 실패 또는 비정상 종료 (오류 원인 파악 필요) |
| **Active: inactive (dead)** | 서비스가 명시적으로 중지된 상태 |
| **Main PID & CGroup** | 실행 중인 메인 프로세스 PID 및 파생된 프로세스 그룹 표시 |
| **최근 로그 10줄** | stdout/stderr의 마지막 10줄. 즉각적인 에러 파악에 유용 |

---

### 3.2 서비스 시작/중지 및 상태 전이

```bash
systemctl start 서비스명    # 서비스 즉시 시작 (출력 메시지 없으면 성공)
systemctl stop 서비스명     # SIGTERM 신호로 우아한 종료(Graceful shutdown) 유도
```

**`restart` vs `reload` (매우 중요):**

| 명령어 | 동작 | PID | 다운타임 | 사용 시점 |
|--------|------|:---:|:-------:|----------|
| `systemctl restart` | 완전히 stop 후 start | 변경됨 | 발생 | 애플리케이션 코드 업데이트 시 |
| `systemctl reload` | 메인 프로세스 유지, HUP 신호로 설정만 재로드 | 유지됨 | 없음 | Nginx 라우팅 설정 변경 등 (애플리케이션이 reload 지원 시) |

---

### 3.3 부팅 시 자동 실행 제어

서버 재부팅 후 서비스가 자동으로 올라오게 하려면 반드시 `enable`을 설정해야 합니다.

```bash
systemctl enable 서비스명      # 자동 실행 활성화
                               # → /etc/systemd/system/multi-user.target.wants/에 심볼릭 링크 생성

systemctl disable 서비스명     # 자동 실행 비활성화 (심볼릭 링크 삭제, 현재 실행 중인 서비스는 유지)

systemctl is-enabled 서비스명  # 자동 실행 상태 확인 (스크립트 작성 시 유용)

systemctl mask 서비스명        # 유닛 파일을 /dev/null로 링크 — 관리자도 수동으로 켤 수 없도록 완전 봉인
systemctl unmask 서비스명      # mask 해제
```

> **`mask`의 의미:** `disable`만으로는 다른 서비스의 의존성에 의해 강제 시작될 수 있습니다.
> `mask`는 이를 원천 차단하는 **궁극의 차단** 수단입니다.

---

## 4부: 심화 트러블슈팅 및 운영 전략

---

### 4.1 데몬 재설정: `systemctl daemon-reload`

> **실무에서 가장 흔히 겪는 실수:**
> `/etc/systemd/system/`의 `.service` 파일을 수정한 후 바로 `systemctl restart`를 실행하면
> **수정 전 캐시된 설정**이 그대로 적용됩니다.

```
.service 파일 수정
      │
      ▼
systemctl daemon-reload   ← 반드시 먼저 실행 (변경된 파일을 메모리에 재로드)
      │
      ▼
systemctl restart 서비스명
```

---

### 4.2 강력한 통합 로그 관리자: `journalctl`

systemd 환경에서는 애플리케이션이 표준 출력(`System.out.println` 등)으로만 던져도,
`systemd-journald` 데몬이 이를 가로채어 **이진(Binary) 형태의 중앙 집중형 로그**로 저장합니다.

```bash
journalctl -u 서비스명                                        # 특정 서비스의 전체 로그 조회
journalctl -u 서비스명 -f                                     # 실시간 로그 모니터링 (tail -f 와 동일)
journalctl -u 서비스명 --since "2023-10-25 10:00:00" \
                        --until "2023-10-25 11:00:00"         # 특정 시간대 로그 필터링
```

---

### 4.3 시스템 부팅 성능 분석

클라우드 환경에서 인스턴스 부팅이 너무 오래 걸릴 때 병목 지점을 찾는 방법입니다.

```bash
systemd-analyze          # 커널 로딩, initrd, 사용자 공간(systemd) 부팅에 걸린 총 시간 출력
systemd-analyze blame    # 부팅 시 초기화된 서비스를 소요 시간 내림차순으로 정렬 출력
                         # → 불필요하게 부팅 속도를 갉아먹는 서비스를 찾아 disable 처리
```

**출력 예시 (`systemd-analyze blame`):**
```
  3.012s cloud-init.service
  1.843s NetworkManager-wait-online.service
   854ms sssd.service
   ...
```

> 소요 시간이 비정상적으로 긴 서비스를 발견했다면, 해당 서비스의 필요 여부를 검토하고
> 불필요하다면 `systemctl disable` 또는 `systemctl mask`로 처리합니다.
