# 로그


리눅스 로그는 크게 두 방식이 있다.

- 전통적인 **파일 기반 로그**는 `rsyslogd`가 수집·분류해 `/var/log/` 아래 텍스트 파일로 기록한다.
- 현대 리눅스의 **journal**은 `systemd-journald`가 수집하고, `journalctl`로 서비스·부팅·시간·우선순위 기준 조회를 지원한다.

## **syslog와 journal 비교**

| 구분 | syslog | journal |
| --- | --- | --- |
| 설명 | 텍스트 파일 중심의 전통적 로그 방식 | systemd 기반의 구조화된 로그 조회 방식 |
| 수집 데몬 | `rsyslogd` | `systemd-journald` |
| 조회 방식 | 로그 파일 직접 조회 (`cat`, `less`, `tail`, `grep`) | `journalctl` 명령으로 조회 |
| 저장 위치 | 주로 `/var/log/` 아래 텍스트 로그 파일 | 기본적으로 `/run/log/journal`(휘발성)
영구 저장 설정 시 `/var/log/journal` |
| 로그 형태 | 사람이 읽을 수 있는 평문 텍스트 | 구조화된 indexed binary 로그 |
| 장점 | 단순하고 직관적, 텍스트 처리 도구와 함께 쓰기 쉬움 | 서비스/부팅/시간/우선순위 기준 조회가 강력함 |
| 단점 | 서비스, 부팅 단위 분석과 정교한 필터링이 불편함 | 텍스트 파일보다 덜 직관적일 수 있음 |
| 실무 활용 | 전통적인 파일 기반 로그 운영
애플리케이션 로그
원격 로그 전송 | systemd 서비스 장애 분석
부팅 이슈, 커널/OOM, 우선순위 기반 분석 |

## RHEL 10

RHEL 10에서는 `systemd-journald`와 `rsyslogd`가 함께 동작한다.

### **로그 흐름**


- `systemd-journald`가 먼저 로그를 수집
- `rsyslogd`가 journal에서 syslog 메시지를 읽음
- `rsyslogd`가 분류·필터링 후 `/var/log` 파일에 기록하거나 원격 전송

### **journal은 기본적으로 휘발성**

journal 로그는 기본적으로 `/run/log/journal`에 저장되며, 재부팅하면 사라질 수 있다.

따라서 영구 보관이 필요하면 별도로 persistent logging을 구성해야 한다.

# syslog


`rsyslogd`는 시스템 로그를 수집해 `/var/log/` 아래의 로그 파일로 기록한다.


로그에 남는 정보는 다음과 같다.

- 서비스 시작/종료
- 인증 성공/실패
- 포트 충돌
- 설정 파일 오류
- 디스크 오류
- 커널 메시지
- 배치/스케줄 작업 실행 결과

## syslog 디렉터리 구조


`rsyslog` 가 수집한 로그파일은 `/var/log/` 에 위치한다.

```
[jiu@localhost ~]$ sudo tree /var/log -L 2
/var/log
├── anaconda
│   ├── anaconda.log
│   ├── dbus.log
│   ├── dnf.librepo.log
│   ├── hawkey.log
│   ├── journal.log
│   ├── lorax-packages.log
│   ├── packaging.log
│   ├── program.log
│   ├── storage.log
│   └── syslog
├── audit
│   └── audit.log
├── btmp
├── chrony
├── cron
├── cron-20260416
├── dnf.librepo.log
├── dnf.log
├── dnf.rpm.log
├── firewalld
├── hawkey.log
├── hawkey.log-20260416
├── insights-client
├── lastlog
├── maillog
├── maillog-20260416
├── messages
├── messages-20260416
├── nginx
│   ├── access.log
│   └── error.log
├── private
├── rhsm
│   ├── rhsmcertd.log
│   ├── rhsmcertd.log-20260416
│   ├── rhsm.log
│   └── rhsm.log-20260416
├── secure
├── secure-20260416
├── spooler
├── spooler-20260416
├── sssd
└── wtmp

9 directories, 36 files
```

### `/var/log/messages`

시스템 전반의 일반 로그가 저장된다.

- 서비스 실행 관련 메시지
- systemd 동작
- 네트워크 상태 변화
- 각종 일반 이벤트

가장 먼저 넓게 확인할 때 자주 본다.

### `/var/log/secure`

인증과 권한 관련 로그가 저장된다.

- SSH 로그인 성공/실패
- sudo 사용 기록
- 인증 실패

접속 문제나 권한 문제를 볼 때 중요하다.

### `/var/log/cron`

cron 작업 실행 기록이 저장된다.

- 예약 작업 실행 여부
- 주기 작업 수행 흔적

배치 작업이 정상 실행됐는지 확인할 때 쓴다.

### `/var/log/maillog`

메일 관련 서비스 로그가 저장된다.

### `/var/log/audit/audit.log`

보안 감사(audit) 로그가 저장된다.

### 애플리케이션별 로그 디렉터리

예를 들어 nginx는 다음처럼 별도 디렉터리를 가진다.

- `/var/log/nginx/access.log`
- `/var/log/nginx/error.log`

> `/var/log`에는 **시스템 공통 로그**와 **애플리케이션별 로그**가 함께 존재한다.
> 

## syslog 파일 구조


`/var/log/messages`

```bash
Apr 16 01:31:23 localhost systemd[1]: Starting fstrim.service - Discard unused blocks on filesystems from /etc/fstab...
Apr 16 01:31:24 localhost fstrim[4481]: /home: 19.2 GiB (20600324096 bytes) trimmed on /dev/vda5
Apr 16 01:31:24 localhost fstrim[4481]: /boot/efi: 585.9 MiB (614395904 bytes) trimmed on /dev/vda1
Apr 16 01:31:24 localhost fstrim[4481]: /boot: 1 GiB (1073741824 bytes) trimmed on /dev/vda2
Apr 16 01:31:24 localhost fstrim[4481]: /: 39.3 GiB (42189455360 bytes) trimmed on /dev/vda4
Apr 16 01:31:24 localhost systemd[1]: fstrim.service: Deactivated successfully.
Apr 16 01:31:24 localhost systemd[1]: Finished fstrim.service - Discard unused blocks on filesystems from /etc/fstab.
Apr 16 01:33:35 localhost NetworkManager[777]: <info>  [1776270815.1082] dhcp4 (enp0s1): state changed new lease, address=192.168.64.5
```

- 발생 시각
- 호스트명
- 프로세스 또는 서비스명
- PID
- 메시지 내용

## syslog 확인 명령어


### 전체 보기

```bash
cat /var/log/messages
```

### 최근 로그 보기

```bash
tail -n 50 /var/log/messages
```

### 실시간 추적

```bash
tail -f /var/log/messages
```

### 특정 키워드 검색

```bash
grep -i failed /var/log/messages
```

syslog 방식은 단순하고 직관적이지만, **서비스 단위 조회나 부팅 단위 조회는 불편**하다.

→ 이 한계를 보완하는 것이 `journalctl`이다.

# journal

journalctl은 `systemd-journald`가 저장한 journal 로그를 조회하는 명령어이다.

### journal 이 필요한 이유

서비스, 부팅, 시간, 우선순위 기준으로 로그를 구조적으로 조회할 수 있어
syslog 방식보다 장애 분석에 유리하다.
****

주요 기능은 다음과 같다.

- **특정 서비스(unit) 기준** 조회
- 현재 **부팅** / 이전 부팅 로그 구분
- **시간 범위 지정** 조회
- **우선순위**(에러 수준) 기준 조회
- **실시간** 로그 추적

## 주요 명령어


### 전체 로그 조회

```bash
journalctl
```

전체 journal 로그를 오래된 항목부터 출력한다.

### 최신 로그부터 보기

```bash
journalctl -r
```

최신 로그부터 보고 싶을 때 사용한다.

### 실시간 추적

```bash
journalctl -f
```

새 로그가 들어오는 것을 실시간으로 추적한다.

### 특정 서비스 로그 조회

```bash
journalctl -u nginx.service
```

특정 systemd 서비스 단위로 로그를 조회한다.

### 최근 로그만 보기

```bash
journalctl -u nginx.service -n 50
```

최근 50줄만 확인한다.

### 현재 부팅 로그 조회

```bash
journalctl -b
```

현재 부팅 이후의 로그만 본다.

### 이전 부팅 로그 조회

***이 기능을 위해서는 persistent logging 구성이 필요하다.**

```bash
journalctl -b -1
```

이전 부팅 세션의 로그를 확인한다.

재부팅 이후 문제 분석할 때 쓰인다.

```bash
journalctl --list-boots
```

### 시간 범위 지정

```bash
journalctl --since "2026-04-23 09:00:00" --until "2026-04-23 10:00:00"

# 상대 시간
journalctl --since "10 min ago"
```

### 우선순위 기준 조회

```bash
# error 이상 수준의 로그만 조회 (err, crit, alert, emerg)
journalctl -p err

# warning부터 alert까지 범위를 지정해 조회 (warning, err, crit, alert)
journalctl -p warning..alert
```

로그의 **우선순위(priority)** 를 기준으로 특정 심각도의 로그만 선택해서 조회할 수 있다.

journal의 우선순위는 다음과 같다. 숫자가 작을수록 더 심각한 로그를 의미한다.

- `emerg=0`
- `alert=1`
- `crit=2`
- `err=3`
- `warning=4`
- `notice=5`
- `info=6`
- `debug=7`

### 커널 로그 조회

```bash
journalctl -k
```

커널 메시지만 따로 본다.

### 참고: `dmesg` 와 `journalctl -k`

#### dmesg

- *diagnostic message*
- 리눅스 커널에서 발생하는 진단 메시지를 확인
- kernel ring buffer 내용을 출력

`dmesg`와 `journalctl -k`는 모두 커널 메시지를 확인하는 명령이지만

- `dmesg`는 kernel ring buffer를 직접 읽는 도구이고,
- `journalctl -k`는 journal에 저장된 커널 로그를 조회하는 명령이다.

⇒ `journalctl -k` 가 다양한 필터링과 이전 부팅 로그 조회에 더 유리하다.

현재 부팅의 커널 메세지는 dmesg, journalctl -k 모두 같은 메세지를 출력한다.
