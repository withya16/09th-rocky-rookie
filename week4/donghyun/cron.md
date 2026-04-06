> **원문:** Red Hat Enterprise Linux 10 — Automating system administration by using RHEL system roles
> 

> **범위:** cron Daemon의 구조와 crontab 문법, at의 일회성 Scheduling, systemd Timer의 구조와 cron 대비 장점, anacron의 역할, RHEL System Role(Ansible)을 활용한 다중 호스트 자동화
> 

> **핵심 연결:** cron → K8s CronJob, systemd Timer → AWS EventBridge, Ansible → AWS SSM + Infrastructure as Code
> 

---

# 1. 작업 Scheduling의 개념 — 왜 자동화해야 하는가

System Administrator의 반복 작업 대부분은 **시간 기반(Time-driven)**입니다: 매일 새벽 3시에 백업, 5분마다 Health Check, 매월 1일에 로그 정리. 이런 작업을 사람이 매번 수동으로 실행하는 것은 불가능합니다.

Linux는 이를 위해 여러 Scheduling 메커니즘을 제공합니다. 각각은 다른 역사적 배경과 설계 철학을 가지고 있으며, 상황에 따라 적합한 도구가 다릅니다.

| 도구 | 성격 | 적합한 상황 |
| --- | --- | --- |
| **cron** | 반복 작업. 가장 전통적이고 보편적 | 단일 서버, 간단한 반복 작업 |
| **at** | 일회성 예약 작업 | "10분 후에 이것 실행해라" |
| **anacron** | cron의 보완. 꺼져 있던 동안 놓친 작업 실행 | 항상 켜져 있지 않은 서버/데스크탑 |
| **systemd Timer** | cron의 현대적 대체. Dependency/Logging/Resource 통합 | Dependency가 필요한 작업, 세밀한 제어 |
| **Ansible (RHEL System Role)** | 다수 서버에 동일 설정 자동 적용 | 서버 10대 이상의 대규모 환경 |

---

# 2. cron — 전통적 Job Scheduling의 표준

## 2-1. cron Daemon의 동작 원리

`crond` Daemon은 시스템 부팅 시 시작되어 **매분(1분마다)** 아래 항목들을 순회합니다:

1. `/var/spool/cron/` — 사용자별 crontab 파일
2. `/etc/crontab` — System-wide crontab (사용자 필드 포함)
3. `/etc/cron.d/` — 패키지/관리자가 설치한 개별 crontab 파일
4. `/etc/cron.hourly/`, `/etc/cron.daily/` 등 — 주기별 스크립트 디렉터리

현재 시각과 일치하는 Schedule이 있으면 해당 명령을 실행합니다. cron이 실행하는 Process는 **해당 사용자의 권한**으로 동작하며, 환경 변수가 매우 제한적입니다 (보통 PATH만 설정됨).

## 2-2. crontab 문법

```
┌────── 분 (0-59)
│ ┌────── 시 (0-23)
│ │ ┌────── 일 (1-31)
│ │ │ ┌────── 월 (1-12 또는 jan-dec)
│ │ │ │ ┌────── 요일 (0-7, 0과 7이 일요일. 또는 mon-sun)
│ │ │ │ │
* * * * *  명령어
```

**특수 문자:**

| 문자 | 의미 | 예시 |
| --- | --- | --- |
| `*` | 모든 값 |   `• * * * *` = 매분 |
| `,` | 여러 값 나열 | `0,30 * * * *` = 매시 0분과 30분 |
| `-` | 범위 | `1-5` = 월~금 |
| `/` | 간격 | `*/5 * * * *` = 5분마다 |

## 2-3. 실습: crontab 관리

```bash
# 현재 사용자의 crontab 확인
crontab -l

# crontab 편집 (EDITOR 환경 변수의 에디터가 열림)
crontab -e

# root의 crontab 확인
sudo crontab -l

# 다른 사용자의 crontab 확인 (root만 가능)
sudo crontab -l -u mr8356

# crontab 전체 삭제 (주의!)
crontab -r
```

**자주 사용하는 Schedule 예시:**

```bash
# 매일 새벽 3시에 백업
0 3 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1

# 5분마다 Health Check
*/5 * * * * /opt/scripts/healthcheck.sh

# 월~금 오전 9시에 Report 생성
0 9 * * 1-5 /opt/scripts/daily-report.sh

# 매월 1일 자정에 Log Rotation
0 0 1 * * /opt/scripts/log-rotate.sh

# 매시간 15분에 실행
15 * * * * /opt/scripts/hourly-task.sh

# 매주 일요일 자정에 전체 시스템 점검
0 0 * * 0 /opt/scripts/weekly-maintenance.sh
```

**`>> /var/log/backup.log 2>&1`의 의미:**

cron은 기본적으로 **stdout/stderr를 사용자에게 메일로 전송**합니다. `MAILTO` 변수가 설정되어 있지 않으면 로컬 Mailbox에 쌓입니다. 대부분의 서버 환경에서는 메일을 사용하지 않으므로, 명시적으로 로그 파일로 Redirect합니다.

- `>>` : stdout을 파일에 Append
- `2>&1` : stderr(2)를 stdout(1)과 같은 곳으로 보냄

이 Redirect가 없으면 cron Job의 출력을 어디서도 확인할 수 없게 됩니다. **이것이 cron의 가장 큰 약점 중 하나**이며, systemd Timer는 journald와 자동 통합되어 이 문제가 없습니다.

## 2-4. System-wide cron 디렉터리

| 경로 | 실행 주기 | 설명 |
| --- | --- | --- |
| `/etc/cron.hourly/` | 매시간 | 여기에 실행 파일을 넣으면 매시간 실행 |
| `/etc/cron.daily/` | 매일 | logrotate 등이 여기에 설치됨 |
| `/etc/cron.weekly/` | 매주 | 주간 작업 |
| `/etc/cron.monthly/` | 매월 | 월간 작업 |
| `/etc/cron.d/` | Custom | crontab 문법 파일. 사용자 필드 포함 |

`/etc/cron.daily/`에 파일을 넣을 때는 **확장자가 없어야** 합니다. `backup.sh`가 아니라 `backup`으로 넣어야 `run-parts`가 인식합니다. 또한 파일에 **실행 권한**이 있어야 합니다.

```bash
# /etc/cron.d/에 Custom Schedule 추가 (root 권한)
sudo tee /etc/cron.d/myapp-cleanup << 'EOF'
# 매일 새벽 2시에 30일 이상 된 로그 삭제 (root 권한으로 실행)
0 2 * * * root find /var/log/myapp/ -type f -mtime +30 -delete
EOF
# /etc/cron.d/ 파일에는 '사용자' 필드가 포함됨 (6번째 필드)
```

## 2-5. cron 보안: cron.allow / cron.deny

누가 crontab을 사용할 수 있는지 제어합니다.

| 파일 | 동작 |
| --- | --- |
| `/etc/cron.allow` 존재 | 이 파일에 나열된 사용자**만** crontab 사용 가능 |
| `/etc/cron.allow` 미존재 + `/etc/cron.deny` 존재 | cron.deny에 나열된 사용자만 사용 **불가** |
| 둘 다 미존재 | RHEL 기본: root만 사용 가능 |

```bash
# 특정 사용자만 cron 사용 허용
sudo echo "mr8356" >> /etc/cron.allow
```

## 2-6. cron의 근본적 한계

cron은 40년 이상 사용된 안정적인 도구이지만 현대적 요구사항에 맞지 않는 부분이 있습니다:

1. **Logging 부재:** stdout/stderr을 직접 Redirect하지 않으면 출력을 볼 수 없습니다.
2. **Dependency 관리 불가:** "A 작업이 끝난 후에 B를 실행"을 표현할 수 없습니다.
3. **놓친 실행 보상 없음:** 서버가 새벽 3시에 꺼져 있으면 3시 작업은 그냥 스킵됩니다.
4. **Resource 제한 불가:** cron Job이 CPU나 Memory를 얼마나 쓸지 제한할 수 없습니다.
5. **환경 변수 제한:** cron Shell 환경은 매우 최소화되어 있어, 로그인 Shell에서는 되는 명령이 cron에서는 실패하는 경우가 빈번합니다.

---

# 3. at — 일회성 예약 작업

cron이 **반복** 작업이라면, at은 **단 한 번만** 실행되는 예약 작업입니다.

```bash
# atd Daemon 설치 및 시작
sudo dnf install at -y
sudo systemctl enable --now atd

# 10분 후에 작업 실행
at now + 10 minutes << 'EOF'
echo "Delayed task at $(date)" >> /tmp/at-test.log
EOF

# 특정 시각에 실행
at 15:30 << 'EOF'
/opt/scripts/one-time-migration.sh
EOF

# 내일 새벽 3시에 실행
at 03:00 tomorrow << 'EOF'
/opt/scripts/db-migration.sh
EOF

# 예약된 Job 목록 확인
atq

# 예약된 Job의 내용 확인 (Job 번호 사용)
at -c 3

# 예약 취소
atrm 3
```

**실무 사용처:** DB Migration, 일회성 데이터 변환, 점검 후 자동 재시작, "30분 뒤에 서버 재부팅" (`at now + 30 minutes <<< "sudo reboot"`)

at도 cron과 마찬가지로 `/etc/at.allow`와 `/etc/at.deny`로 접근 제어가 가능합니다.

---

# 4. anacron — 놓친 cron 작업의 보상 실행

anacron은 cron의 보완 도구입니다. cron은 지정 시각에 서버가 꺼져 있으면 작업을 건너뛰지만, anacron은 **시스템이 켜진 후 놓친 작업을 실행**합니다.

anacron은 주기를 "매일", "3일마다" 같은 **일 단위**로만 지정할 수 있습니다 (분/시 단위 불가). `/var/spool/anacron/`에 마지막 실행 시각을 기록하여, 부팅 시 이전에 놓친 작업이 있는지 확인합니다.

```bash
# anacron 설정 파일 확인
cat /etc/anacrontab
```

설정 형식:

```
# 주기(일)  지연(분)  Job ID        명령어
1           5        cron.daily    nice run-parts /etc/cron.daily
7          25        cron.weekly   nice run-parts /etc/cron.weekly
@monthly   45        cron.monthly  nice run-parts /etc/cron.monthly
```

- **주기(일):** 마지막 실행 후 며칠이 지나면 실행할지
- **지연(분):** 부팅 후 몇 분 기다린 후 실행할지 (부팅 직후 부하 분산)
- **Job ID:** `/var/spool/anacron/`에 기록되는 이름

> RHEL에서 `/etc/cron.daily/`, `/etc/cron.weekly/`, `/etc/cron.monthly/`의 스크립트들은 실제로 cron이 아니라 **anacron을 통해** 실행됩니다. `/etc/cron.d/0hourly`가 매시간 `run-parts /etc/cron.hourly/`를 실행하고, 그 중 `/etc/cron.hourly/0anacron`이 anacron을 호출하는 구조입니다.
> 

---

# 5. systemd Timer — cron의 현대적 대안

## 5-1. Timer의 구조: .timer + .service 한 쌍

systemd Timer는 **두 개의 Unit File이 한 쌍**으로 동작합니다:

- `.timer` Unit: **언제** 실행할지를 정의
- `.service` Unit: **무엇을** 실행할지를 정의

Timer가 Trigger되면 **같은 이름**의 Service를 시작합니다. `backup.timer`는 `backup.service`를 실행합니다. 다른 이름의 Service를 실행하려면 `Unit=` Directive로 지정합니다.

## 5-2. Timer의 두 가지 유형

**Monotonic Timer (상대 시간):**

시스템 부팅이나 특정 Event 이후 경과 시간 기반으로 동작합니다.

| Directive | 기준 |
| --- | --- |
| `OnBootSec=` | 시스템 부팅 후 경과 시간 |
| `OnUnitActiveSec=` | 해당 Service가 마지막으로 활성화된 후 경과 시간 |
| `OnUnitInactiveSec=` | 해당 Service가 마지막으로 비활성화된 후 경과 시간 |
| `OnStartupSec=` | systemd가 시작된 후 경과 시간 |

```
# 부팅 5분 후 실행, 이후 1시간마다 반복
[Timer]
OnBootSec=5min
OnUnitActiveSec=1h
```

**Realtime Timer (절대 시간 = Calendar):**

특정 날짜/시간에 실행됩니다. cron의 Schedule 기능을 대체합니다.

| Directive | 용도 |
| --- | --- |
| `OnCalendar=` | 절대 시간 기반 Schedule |

```
# 매일 새벽 3시
[Timer]
OnCalendar=*-*-* 03:00:00

# 매주 월요일 오전 9시
[Timer]
OnCalendar=Mon *-*-* 09:00:00
```

**OnCalendar 문법:**

```
DayOfWeek Year-Month-Day Hour:Minute:Second
```

| 표현 | 의미 |
| --- | --- |
| `*-*-* 03:00:00` | 매일 새벽 3시 |
| `Mon *-*-* 09:00:00` | 매주 월요일 오전 9시 |
| `*-*-01 00:00:00` | 매월 1일 자정 |
| `*-01,07-01 00:00:00` | 매년 1월 1일, 7월 1일 |
| `minutely` / `hourly` / `daily` / `weekly` / `monthly` | 축약형 |
| `*-*-* *:00/15:00` | 15분마다 |

```bash
# Calendar 표현식 검증 (다음 실행 시각 확인)
systemd-analyze calendar "*-*-* 03:00:00"
systemd-analyze calendar "Mon *-*-* 09:00:00"
systemd-analyze calendar "hourly"
```

## 5-3. Timer의 핵심 옵션

| Directive | 설명 |
| --- | --- |
| `Persistent=true` | **놓친 실행 보상.** 시스템이 꺼져 있어서 놓친 Schedule이 있으면, 부팅 후 즉시 실행합니다. cron에는 없는 기능입니다. |
| `RandomizedDelaySec=` | 실행 시각에 **랜덤 지연**을 추가합니다. 여러 서버가 동시에 같은 작업을 실행하여 인프라에 부하 스파이크가 발생하는 것을 방지합니다. |
| `AccuracySec=` | Timer의 정밀도. 기본값 1분. 여러 Timer를 같은 시점에 모아 실행하여 CPU Wakeup을 최소화합니다 (절전 효과). |
| `Unit=` | Trigger할 Service Unit. 기본값은 같은 이름의 .service. |

## 5-4. 실습: Timer 생성 (처음부터 끝까지)

```bash
# 1. 실행할 Service Unit 생성
sudo tee /etc/systemd/system/backup.service << 'EOF'
[Unit]
Description=Daily Backup Job
# After=로 DB 백업 순서 보장 가능 (cron에서는 불가능한 기능)
After=postgresql.service

[Service]
Type=oneshot
ExecStart=/opt/scripts/backup.sh
User=root
# cgroup Resource 제한 — 백업이 앱에 영향 주지 않도록
MemoryMax=1G
CPUQuota=30%
Nice=10
EOF

# 2. Timer Unit 생성
sudo tee /etc/systemd/system/backup.timer << 'EOF'
[Unit]
Description=Run Backup Daily at 3AM

[Timer]
OnCalendar=*-*-* 03:00:00
# 놓친 실행 보상
Persistent=true
# 0~5분 랜덤 지연 (서버 여러 대일 때 부하 분산)
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
EOF

# 3. Reload 및 활성화
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer

# 4. 확인
sudo systemctl list-timers --all
sudo systemctl status backup.timer

# 5. 수동으로 즉시 실행 (테스트)
sudo systemctl start backup.service

# 6. 실행 로그 확인 (별도 로그 Redirect 없이도 journald에 자동 기록)
sudo journalctl -u backup.service --since today
```

## 5-5. cron vs systemd Timer — 상세 비교

| 항목 | cron | systemd Timer |
| --- | --- | --- |
| **Logging** | 별도 Redirect 필요 (`>> file 2>&1`) | journald **자동 통합**. `journalctl -u` 한 줄이면 끝 |
| **Dependency** | 불가 | `After=`, `Requires=` 사용 가능. "DB 백업 후 앱 백업" 같은 순서 보장 |
| **놓친 실행** | 스킵 | `Persistent=true`로 부팅 시 **보상 실행** |
| **Resource 제한** | 불가 | Service Unit에 `MemoryMax=`, `CPUQuota=` 등 설정 |
| **랜덤 지연** | 불가 | `RandomizedDelaySec=`로 부하 분산 |
| **환경 변수** | 최소한의 환경. PATH 문제 빈번 | `EnvironmentFile=`로 명시적 관리 |
| **모니터링** | 별도 도구 필요 | `systemctl list-timers`로 다음 실행 시각, 마지막 실행 시각 한눈에 확인 |
| **설정 복잡도** | 한 줄 (crontab) | 두 파일 (.service + .timer) |
| **디버깅** | 매우 어려움 | `systemctl status`, `journalctl -u`로 즉시 확인 |

**권장 기준:**

- 간단한 반복 작업, 빠르게 설정 → cron
- Logging, Dependency, Resource Control이 필요하면 → systemd Timer
- **새로운 시스템에서는 systemd Timer를 우선 고려**