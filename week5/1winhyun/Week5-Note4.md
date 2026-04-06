# SELinux 심층 분석 및 실무 제어

---

## 1부: SELinux의 탄생 배경과 핵심 철학

**SELinux(Security-Enhanced Linux)** 는 미국 국가안보국(NSA)이 주도하여 개발하고 Red Hat이 리눅스 커널에 통합시킨 보안 모듈입니다.
시스템이 해킹당하더라도 **피해를 최소화(격리)** 하는 것을 목표로 합니다.

---

### 1.1 DAC (Discretionary Access Control, 임의 접근 제어)

기존의 표준 리눅스 권한 체계(`rwxrwxrwx`, `chmod`, `chown`)입니다.

- **주체 중심**: 파일이나 디렉터리의 **소유자(Owner)** 가 접근 권한을 임의로 결정합니다.
- **치명적인 단점**: 아파치 웹 서버(`httpd`) 프로세스가 해킹당해 탈취되면, 해당 프로세스는 웹 서버를 실행한 사용자(예: `apache` 또는 `root`)의 **모든 권한을 그대로 물려받습니다.**
  - 해커가 웹 서버 취약점을 뚫으면 `/etc/passwd`, `/etc/shadow` 같은 중요 시스템 파일까지 접근하여 전체 시스템을 장악할 수 있습니다.

---

### 1.2 MAC (Mandatory Access Control, 강제 접근 제어)

SELinux가 채택한 보안 모델입니다.

- **정책 중심**: 소유자의 의도나 파일의 `rwx` 권한과 무관하게, 중앙에서 관리되는 **강력한 보안 정책(Policy)** 에 의해서만 접근이 허용됩니다.
- **최소 권한의 원칙 (Principle of Least Privilege)**: 웹 서버(`httpd`) 프로세스는 오직 웹 서비스에 필요한 파일(예: `/var/www/html`)과 포트(`80`, `443`)에만 접근할 수 있도록 정책으로 엄격하게 규정됩니다.
- **격리(Sandboxing)**: 웹 서버가 해킹당해 `root` 권한이 탈취되더라도, SELinux 정책에 "웹 서버 프로세스는 시스템 비밀번호 파일에 접근할 수 없다"고 명시되어 있다면 **커널 수준에서 접근이 강제 차단**됩니다. 피해가 웹 서버 영역에만 국한됩니다.

| 구분 | DAC | MAC (SELinux) |
|------|-----|---------------|
| 접근 제어 주체 | 파일 소유자 | 중앙 보안 정책 |
| 기준 | `rwx` 권한 비트 | 보안 컨텍스트(타입) |
| 프로세스 탈취 시 | 해당 사용자 권한 전체 노출 | 정책 범위 내로 피해 격리 |

---

## 2부: SELinux의 아키텍처와 컨텍스트(Context)

SELinux는 모든 **주체(프로세스)** 와 **객체(파일, 디렉터리, 포트)** 에 꼬리표(Label)를 붙이고, 이 꼬리표들을 비교하여 접근을 제어합니다.
이 꼬리표를 **보안 컨텍스트(Security Context)** 라고 부릅니다.

---

### 2.1 보안 컨텍스트의 구조

```bash
ls -Z    # 파일의 SELinux 컨텍스트 확인
ps -Z    # 프로세스의 SELinux 컨텍스트 확인
```

컨텍스트 형식: `사용자:역할:타입:민감도`

```
system_u : object_r : httpd_sys_content_t : s0
   │           │              │               │
  User        Role           Type           Level
```

| 필드 | 설명 |
|------|------|
| **User** | SELinux에 정의된 사용자 신분 (리눅스 일반 계정과는 다름) |
| **Role** | 사용자가 접근할 수 있는 타입들의 집합 (RBAC 모델) |
| **Type** | **가장 중요.** Targeted Policy에서 모든 접근 제어의 기준. 파일은 `_t`로 끝남 (예: `httpd_sys_content_t`). 프로세스의 타입은 **도메인(Domain)** 이라고도 부름 (예: `httpd_t`) |
| **Level** | 다중 등급 보안(MLS)에서 사용. 일반적인 웹/DB 서버 환경에서는 깊게 다루지 않음 |

---

### 2.2 Type Enforcement (TE)의 동작 원리

> **핵심 규칙:** "특정 도메인(프로세스)은 허용된 타입(파일/포트)에만 접근할 수 있다."

```
사용자 웹 브라우저 접속
        │
        ▼
  httpd_t 도메인 (웹 서버 프로세스)
        │
        ├── /var/www/html/index.html  (타입: httpd_sys_content_t)
        │      └─ 정책에 허용됨 → 접근 승인 ✓
        │
        └── /root/secret.txt          (타입: admin_home_t)
               └─ 정책에 없음 → 접근 차단 ✗  (파일 권한이 777이어도 차단)
```

---

## 3부: SELinux의 동작 모드 및 상태 관리

SELinux는 운영 환경에 따라 세 가지 모드 중 하나로 동작합니다.

### 3.1 동작 모드의 종류

| 모드 | 설명 |
|------|------|
| **Enforcing** (강제) | 정책에 어긋나는 모든 접근을 **차단**하고 로그를 남김. 운영 환경의 기본이자 권장 상태 |
| **Permissive** (허용) | 정책 위반 접근을 차단하지 않고 **로그(경고)만** 남김. "만약 Enforcing이었다면 무엇이 차단되었을까?"를 테스트하고 트러블슈팅할 때 유용 |
| **Disabled** (비활성화) | SELinux 커널 모듈 자체가 적재되지 않음. 파일 생성 시 SELinux 꼬리표(컨텍스트)가 아예 붙지 않음 |

---

### 3.2 상태 확인 및 변경 명령어

```bash
# 상태 확인
sestatus          # 전반적인 상태 및 정책 확인
getenforce        # 현재 모드만 간략히 확인

# 일시적 모드 변경 (재부팅 시 초기화)
setenforce 0      # Permissive로 변경
setenforce 1      # Enforcing으로 변경
```

영구적 변경은 `/etc/selinux/config` 파일을 수정합니다.

```
SELINUX=enforcing   # enforcing / permissive / disabled
```

> **주의 — Disabled에서 Enforcing으로 바로 전환 금지:**
> Disabled 상태에서는 파일에 컨텍스트 꼬리표가 붙지 않습니다.
> 바로 Enforcing으로 변경하고 재부팅하면 수많은 파일에 Relabeling이 발생하여
> 부팅이 매우 오래 걸리거나 **부팅 불가 상태**에 빠질 수 있습니다.
>
> **안전한 전환 순서:**
> ```
> Disabled → Permissive (재부팅 → 로그 확인 및 컨텍스트 자동 부여) → Enforcing
> ```

---

## 4부: 보안 컨텍스트 관리 (실무 제어)

웹 서버 루트 디렉터리를 `/var/www/html`이 아닌 `/data/web`으로 변경한 경우를 예시로 설명합니다.
리눅스 권한을 맞추더라도 SELinux가 켜져 있으면 **403 Forbidden** 에러가 발생합니다.
`/data` 디렉터리 생성 시 기본 타입인 `default_t`가 부여되었고, `httpd_t`는 이 타입에 접근 권한이 없기 때문입니다.

---

### 4.1 일시적 컨텍스트 변경: `chcon`

```bash
chcon -t httpd_sys_content_t /data/web/index.html
```

> **한계:** 시스템 Relabel이나 `restorecon` 명령어가 실행되면 **원래 정책 기본값으로 되돌아갑니다.**
> 일시적인 테스트 목적으로만 사용해야 합니다.

---

### 4.2 영구적 컨텍스트 변경: `semanage` + `restorecon` (핵심)

실무에서는 반드시 이 방식을 사용해야 재부팅 후에도 설정이 유지됩니다.

**1단계 — 정책 데이터베이스에 규칙 등록 (`semanage fcontext`):**

```bash
semanage fcontext -a -t httpd_sys_content_t "/data/web(/.*)?"
```

> 이 단계에서는 아직 실제 파일의 컨텍스트가 바뀌지 않습니다.

**2단계 — 등록된 정책을 실제 파일에 적용 (`restorecon`):**

```bash
restorecon -Rv /data/web
#  -R : 하위 디렉터리까지 재귀적 적용
#  -v : 변경 내역 출력
```

```
정책 DB (semanage)          실제 파일 시스템
       │                           │
  규칙 등록                   컨텍스트: default_t
       │                           │
       └──── restorecon ──────────►│
                                   │
                            컨텍스트: httpd_sys_content_t  ✓
```

---

### 4.3 포트 컨텍스트 제어

SELinux는 파일뿐만 아니라 **포트 번호에도 타입을 부여**합니다.
아파치가 기본 `80`, `443` 포트가 아닌 `8080`을 사용하도록 변경하면 포트 바인딩에 실패합니다.

```bash
# 현재 허용된 HTTP 포트 확인
semanage port -l | grep http_port_t

# 8080 포트를 HTTP 타입으로 추가 허용
semanage port -a -t http_port_t -p tcp 8080
```

---

### 4.4 SELinux Boolean (불리언)

복잡한 컨텍스트 조작 없이, **특정 기능의 On/Off 스위치**만 켜고 끌 수 있도록 미리 정의해 둔 정책입니다.

> **예시:** 웹 서버(WAS)가 원격 DB(MySQL 등)에 네트워크로 접속해야 할 때 차단되는 경우.
> 웹 서버는 기본적으로 내부 서비스만 하도록 제한되어 있기 때문입니다.

```bash
# Boolean 목록 확인
getsebool -a | grep httpd

# 웹 서버의 네트워크 연결 허용 (영구 적용)
setsebool -P httpd_can_network_connect 1
#  -P : 재부팅 후에도 영구 적용
#   1 : 허용 / 0 : 차단
```

---

## 5부: 트러블슈팅 및 로그 분석 (Audit Log)

> "SELinux 때문에 안 되는 것 같은데?" 라는 의심이 들 때,
> **무작정 끄기 전에 로그를 확인하여 원인을 파악하는 것**이 올바른 접근입니다.

---

### 5.1 감사 로그 (Audit Log) 확인

SELinux의 차단 내역은 `/var/log/audit/audit.log`에 기록됩니다.
`type=AVC` 키워드가 포함된 로그가 SELinux가 특정 행위를 **거부(denied)** 했다는 증거입니다.

```bash
grep AVC /var/log/audit/audit.log
```

---

### 5.2 로그 분석 도구: `audit2why`와 `audit2allow`

원시 로그(Raw Log)는 사람이 읽기 복잡합니다. `policycoreutils-python` 패키지에 포함된 도구를 사용합니다.

#### `audit2why` — 원인 분석 및 해결책 제시

```bash
grep AVC /var/log/audit/audit.log | audit2why
```

로그를 분석하여 **"왜 차단되었고, 어떻게 해결할 수 있는지"** 를 인간의 언어로 설명해 줍니다.
출력 결과에서 `setsebool -P httpd_can_network_connect 1`과 같이 **정확한 해결 명령어**를 제시해 주는 경우가 많습니다.

#### `audit2allow` — 사용자 정의 정책 모듈 자동 생성

기존 정책으로 해결이 안 되는 서드파티 애플리케이션을 구동할 때 최후의 수단으로 사용합니다.
현재 차단되고 있는 행위들을 분석하여, 이를 허용하기 위한 **새로운 SELinux 정책 모듈**을 자동으로 생성합니다.

---

> **정리:** SELinux는 얼핏 시스템 운영을 방해하는 귀찮은 존재처럼 느껴질 수 있습니다.
> 하지만 제로 데이(Zero-day) 취약점 공격이나 권한 탈취로부터 시스템을 방어하는 **최후의 보루** 역할을 합니다.
> 컨텍스트와 정책의 흐름을 이해한다면, 보안성을 해치지 않으면서도 안정적인 인프라 서비스를 운영할 수 있습니다.
