> **주제:** 로키 리눅스 스터디 — 로그 관리 및 시스템 모니터링
**범위:** 로그의 역할과 주요 로그 파일 구조, journalctl 활용, 서비스 장애 분석, CPU·메모리·디스크·프로세스 점검, 성능 측정 도구
**자격증 연결:** CKA/CKS (100% — kubelet/Static Pod 장애 복구, Audit Log 분석), AWS SAA/SysOps (CloudWatch Agent, 메트릭 수집), Terraform (Provisioner 로그 디버깅)
> 

---

# Part 1. 로그란 무엇이고, 왜 가장 중요한 기술인가

## 1-1. 로그의 본질 — "과거에 무슨 일이 있었는지"의 유일한 증거

서버에서 문제가 발생하면 **이미 일어난 일**입니다. 현재 상태(`ps`, `top`)만으로는 "왜 발생했는지"를 알 수 없습니다. **로그만이 과거를 기록**합니다.

## 1-2. Linux 로그 시스템의 두 갈래

RHEL 10에서 로그는 **두 가지 시스템**이 공존합니다:

| 시스템 | 데몬 | 저장 위치 | 형식 | 특징 |
| --- | --- | --- | --- | --- |
| **journald** | `systemd-journald` | `/run/log/journal/` (기본: 휘발성) 또는 `/var/log/journal/` (영구) | **Binary** (구조화) | systemd 통합. 메타데이터 풍부. `journalctl`로 조회 |
| **rsyslog** | `rsyslogd` | `/var/log/messages`, `/var/log/secure` 등 | **텍스트** | 전통적. `cat`, `grep`으로 바로 읽기 가능 |

동작 흐름:

```
프로세스 → systemd-journald (Binary 저장)
                ↓ (자동 전달)
           rsyslogd → /var/log/messages 등 (텍스트 저장)
```

journald가 **먼저** 모든 로그를 수집하고, rsyslog에 전달하여 텍스트 파일로도 저장합니다. 두 시스템이 같은 로그를 다른 형식으로 보관하는 것입니다.

> **왜 두 개가 공존하는가:** journald는 구조화된 검색(시간, 서비스, Priority 필터링)에 강하고, rsyslog의 텍스트 파일은 `cat`, `grep`, `awk` 등 전통적 도구로 즉시 읽을 수 있습니다. 또한 rsyslog는 **원격 로그 전송**(다른 서버나 SIEM으로)에 강합니다.
> 

---

# Part 2. /var/log — 주요 로그 파일 구조

## 2-1. 핵심 로그 파일 지도

| 경로 | 기록 내용 | 누가 쓰는가 | 언제 보는가 |
| --- | --- | --- | --- |
| `/var/log/messages` | **시스템 전반** 로그. Kernel, 서비스, 하드웨어 메시지 | rsyslog | **가장 먼저 보는 파일.** 원인을 모를 때 |
| `/var/log/secure` | **인증/보안** 로그. SSH 로그인 시도, sudo 사용, PAM | rsyslog | SSH Brute Force 탐지, sudo 감사 |
| `/var/log/cron` | **cron** Job 실행 기록 | rsyslog | cron 작업이 실행되었는지 확인 |
| `/var/log/boot.log` | 부팅 시 서비스 시작 메시지 | systemd | 부팅 실패 원인 분석 |
| `/var/log/dnf.log` | **패키지 설치/제거** 기록 | dnf | "언제 이 패키지가 설치됐지?" |
| `/var/log/audit/audit.log` | **SELinux, Kernel Audit** 로그 | auditd | SELinux 차단 원인, 보안 감사 |
| `/var/log/dmesg` | **Kernel Ring Buffer** — 하드웨어 감지, 드라이버 메시지 | Kernel | 디스크 오류, NIC 문제, OOM Kill |
| `/var/log/firewalld` | firewalld 동작 로그 | firewalld | 방화벽 규칙 변경 추적 |
| `/var/log/pods/` | **K8s Pod 로그** (kubelet이 생성) | kubelet | Container 로그 확인 |
| `/var/log/containers/` | Container 로그 Symlink | Container Runtime | `/var/log/pods/`를 가리킴 |

```bash
# 주요 로그 파일 빠르게 훑어보기
sudo tail -20 /var/log/messages
sudo tail -20 /var/log/secure
sudo tail -20 /var/log/cron

# 실시간 모니터링
sudo tail -f /var/log/messages

# 여러 로그를 동시에 모니터링
sudo tail -f /var/log/messages /var/log/secure
```

> **AWS SAA 연결 — CloudWatch Logs로 전송:**
이 로그 파일들을 AWS CloudWatch Logs로 보내는 것이 **중앙 집중형 로그 관리**의 핵심입니다.
SAA 시험: "EC2 Instance의 `/var/log/secure` 로그를 중앙에서 모니터링하려면?"
→ **CloudWatch Agent**를 설치하여 로그 파일을 CloudWatch Logs Group으로 전송합니다.
> 
> 
> CloudWatch Agent 설정 (`/opt/aws/amazon-cloudwatch-agent/etc/config.json`)에서:
> 
> ```json
> {
>   "logs": {
>     "logs_collect_list": [
>       {
>         "file_path": "/var/log/secure",
>         "log_group_name": "/ec2/secure",
>         "log_stream_name": "{instance_id}"
>       },
>       {
>         "file_path": "/var/log/messages",
>         "log_group_name": "/ec2/messages",
>         "log_stream_name": "{instance_id}"
>       }
>     ]
>   }
> }
> ```
> 

> **CKS 연결 — /var/log/audit/audit.log:**
CKS 시험에서 **Audit Log** 문제가 빈번합니다. K8s API Server의 Audit Policy를 설정한 후, Audit Log 파일에서 특정 사용자의 API 호출을 찾아내는 문제입니다.
> 
> 
> ```bash
> # K8s Audit Log에서 forbidden(403) 응답을 받은 요청 찾기
> sudo grep "403" /var/log/kubernetes/audit.log | head -20
> 
> # JSON 형식 Audit Log에서 특정 User 추출
> sudo cat /var/log/kubernetes/audit.log | \
>   jq 'select(.responseStatus.code == 403) | .user.username' | \
>   sort | uniq
> ```
> 

## 2-2. /var/log/secure — SSH 보안 감사

```bash
# 성공한 SSH 로그인
sudo grep "Accepted" /var/log/secure

# 실패한 SSH 로그인 (Brute Force 탐지)
sudo grep "Failed password" /var/log/secure

# 실패 IP 통계 (공격자 IP Top 10)
sudo grep "Failed password" /var/log/secure | \
  awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -10

# sudo 사용 기록
sudo grep "sudo:" /var/log/secure

# 특정 시간대의 로그인 시도
sudo grep "Mar 26" /var/log/secure | grep "Failed"
```

## 2-3. /var/log/messages — 시스템 전반 로그

```bash
# OOM Kill(메모리 부족으로 프로세스 강제 종료) 찾기
sudo grep -i "oom" /var/log/messages
sudo grep -i "killed process" /var/log/messages

# Disk 관련 오류
sudo grep -iE "error|fail|i/o" /var/log/messages | grep -i "sd\|nvme\|disk"

# 특정 서비스의 메시지
sudo grep "sshd" /var/log/messages | tail -20
sudo grep "kubelet" /var/log/messages | tail -20

# 최근 1시간의 메시지
sudo awk -v d="$(date -d '1 hour ago' '+%b %e %H')" '$0 >= d' /var/log/messages
```

---

# Part 3. journalctl — systemd의 통합 로그 시스템

## 3-1. journald의 아키텍처와 핵심 개념

journald는 systemd와 **완전 통합**된 로그 시스템입니다. rsyslog의 텍스트 파일과 달리, journald는 **Binary 형식의 구조화된 로그**를 저장합니다.

**journald가 rsyslog보다 강한 점:**

- **구조화된 메타데이터:** 각 로그 Entry에 PID, UID, Service 이름, 부팅 ID, Priority 등이 자동으로 태깅됩니다. `grep`으로 텍스트를 찾는 것보다 **훨씬 정확한 필터링**이 가능합니다.
- **systemd Unit 단위 필터링:** `u kubelet` 한 줄로 kubelet의 로그만 정확히 뽑을 수 있습니다. rsyslog에서는 `grep`으로 찾아야 하는데 다른 서비스의 "kubelet"이라는 단어까지 섞일 수 있습니다.
- **부팅별 로그 분리:** 이전 부팅의 로그를 `b -1`로 바로 볼 수 있습니다.

**journald의 약점:**

- Binary 형식이라 `cat`, `grep`으로 직접 읽을 수 없습니다. 반드시 `journalctl`을 사용해야 합니다.
- 기본적으로 **휘발성(RAM)**입니다 — 재부팅하면 로그가 사라집니다.

## 3-2. Persistent Storage 설정 — 재부팅 후에도 로그 유지

**기본 상태에서 journald 로그는 `/run/log/journal/`에 저장됩니다. `/run`은 tmpfs(RAM)이므로 재부팅하면 사라집니다.**

영구 저장을 활성화하려면:

```bash
# 방법 1: 디렉터리 생성 (가장 간단)
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald

# 방법 2: 설정 파일에서 명시적으로 지정
sudo mkdir -p /etc/systemd/journald.conf.d/
sudo tee /etc/systemd/journald.conf.d/persistent.conf << 'EOF'
[Journal]
Storage=persistent
SystemMaxUse=1G
SystemMaxFileSize=100M
MaxRetentionSec=30d
EOF
sudo systemctl restart systemd-journald

# 영구 저장 확인
ls -la /var/log/journal/
```

**Storage 옵션:**

| 값 | 동작 |
| --- | --- |
| `volatile` | `/run/log/journal/`에만 저장 (RAM, 재부팅 시 삭제) |
| `persistent` | `/var/log/journal/`에 저장 (디스크, 영구) |
| `auto` (기본) | `/var/log/journal/` 디렉터리가 있으면 persistent, 없으면 volatile |
| `none` | 로그를 저장하지 않음 (전달만) |

**용량 제어 Directive:**

| Directive | 설명 |
| --- | --- |
| `SystemMaxUse=` | 전체 Journal이 사용할 최대 디스크 공간 |
| `SystemMaxFileSize=` | 개별 Journal 파일의 최대 크기 |
| `MaxRetentionSec=` | 로그 보관 기간 |
| `SystemKeepFree=` | File System에 최소 이만큼의 여유 공간 유지 |

> **실무 사고 예방:** Persistent Storage를 켰는데 용량 제한을 안 걸면 `/var/log/journal/`이 디스크를 가득 채워 **서버가 먹통**이 됩니다. 반드시 `SystemMaxUse=`를 설정하세요.
> 
> 
> **CKA 연결:** K8s Worker Node에서 kubelet 로그가 사라지면 장애 분석이 불가능합니다. Production Node에서는 반드시 journald Persistent Storage를 활성화해야 합니다.
> 

## 3-3. journalctl 핵심 명령어

### 서비스(Unit) 기준 필터링

```bash
# 특정 Service 로그
sudo journalctl -u sshd.service
sudo journalctl -u kubelet.service
sudo journalctl -u nginx.service

# 여러 Service 동시에
sudo journalctl -u sshd -u nginx

# 실시간 모니터링 (tail -f와 동일)
sudo journalctl -u kubelet -f

# 최근 N줄만
sudo journalctl -u kubelet -n 50

# 역순 (최신이 위에)
sudo journalctl -u kubelet -r
```

### 시간 기준 필터링

```bash
# 오늘 로그만
sudo journalctl --since today

# 최근 1시간
sudo journalctl --since "1 hour ago"

# 최근 30분
sudo journalctl --since "30 min ago"

# 특정 시간 범위
sudo journalctl --since "2026-04-04 10:00:00" --until "2026-04-04 12:00:00"

# 어제 로그
sudo journalctl --since yesterday --until today
```

### Priority(심각도) 기준 필터링

| Priority | 코드 | 의미 |
| --- | --- | --- |
| `emerg` | 0 | 시스템 사용 불가 |
| `alert` | 1 | 즉시 조치 필요 |
| `crit` | 2 | 치명적 상태 |
| `err` | 3 | 오류 |
| `warning` | 4 | 경고 |
| `notice` | 5 | 정상이지만 주의 |
| `info` | 6 | 정보성 |
| `debug` | 7 | 디버그 |

```bash
# err(3) 이상만 (emerg, alert, crit, err)
sudo journalctl -p err

# 특정 Service의 에러만
sudo journalctl -u kubelet -p err

# warning 이상 + 오늘 + 실시간
sudo journalctl -p warning --since today -f
```

### 부팅 기준 필터링

```bash
# 부팅 이력 확인
sudo journalctl --list-boots

# 현재 부팅의 로그
sudo journalctl -b 0

# 직전 부팅의 로그 (부팅 실패 원인 분석)
sudo journalctl -b -1

# 직전 부팅에서 kubelet 에러만
sudo journalctl -b -1 -u kubelet -p err
```

### Kernel 로그

```bash
# Kernel 메시지 (dmesg 대체)
sudo journalctl -k

# Kernel 로그 + 실시간
sudo journalctl -k -f

# OOM Kill 관련 Kernel 메시지
sudo journalctl -k | grep -i "oom\|killed process"
```

### 출력 형식

```bash
# JSON 형식 (스크립트 파싱용)
sudo journalctl -u sshd -o json-pretty -n 5

# 짧은 형식 (기본)
sudo journalctl -u sshd -o short

# 상세 형식 (모든 메타데이터)
sudo journalctl -u sshd -o verbose -n 1
```

### 디스크 관리

```bash
# Journal이 사용하는 디스크 용량
sudo journalctl --disk-usage

# 500MB로 줄이기
sudo journalctl --vacuum-size=500M

# 7일 이전 삭제
sudo journalctl --vacuum-time=7d

# 파일 수 제한
sudo journalctl --vacuum-files=5
```

> **CKA/CKS 직접 연결 — kubelet 장애 복구 시나리오:**
> 
> 
> **CKA 단골 시나리오: "Node가 NotReady 상태다. 복구하시오."**
> 
> ```bash
> # Step 1: Node에 SSH 접속 후 kubelet 상태 확인
> sudo systemctl status kubelet
> # Active: failed (dead) ← 죽어있음
> 
> # Step 2: kubelet 로그에서 에러 원인 찾기
> sudo journalctl -u kubelet -n 50 --no-pager
> # 또는 실시간으로:
> sudo journalctl -u kubelet -f
> 
> # Step 3: 흔한 에러 패턴
> # "certificate has expired" → 인증서 만료
> # "failed to load config" → /var/lib/kubelet/config.yaml 오류
> # "port 10250 already in use" → 포트 충돌
> # "cannot connect to etcd" → Control Plane 문제
> 
> # Step 4: 최근 5분간의 에러만 필터링
> sudo journalctl -u kubelet --since "5 min ago" -p err
> 
> # Step 5: 원인 수정 후 재시작
> sudo systemctl restart kubelet
> sudo systemctl status kubelet
> ```
> 
> **CKA 단골 시나리오: "Control Plane(Static Pod)이 안 뜬다."**
> 
> Static Pod(API Server, etcd, Controller Manager, Scheduler)는 kubelet이 직접 관리합니다. kubectl이 안 먹히므로 **crictl**과 **OS 로그**로 진단해야 합니다.
> 
> ```bash
> # crictl로 Container 상태 확인
> sudo crictl ps -a | grep -E "kube-api|etcd|scheduler|controller"
> 
> # 특정 Container의 로그 확인
> sudo crictl logs <container_id>
> 
> # Static Pod Manifest 위치 확인
> ls /etc/kubernetes/manifests/
> # kube-apiserver.yaml  etcd.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
> 
> # Pod 로그 디렉터리 직접 확인
> sudo ls /var/log/pods/
> sudo tail -50 /var/log/pods/kube-system_kube-apiserver-*/kube-apiserver/*.log
> 
> # Kernel 레벨 문제 확인 (Container Runtime 자체 문제)
> sudo journalctl -k | grep -i "containerd\|oom\|error"
> 
> # containerd Runtime 로그
> sudo journalctl -u containerd --since "10 min ago"
> ```
> 

---

# Part 4. rsyslog와 로그 관리

## 4-1. rsyslog 설정 구조

rsyslog의 설정 파일은 `/etc/rsyslog.conf`와 `/etc/rsyslog.d/*.conf`입니다.

**Facility(소스)와 Priority(심각도) 조합:**

```
facility.priority    action
```

| Facility | 의미 |
| --- | --- |
| `kern` | Kernel |
| `auth` / `authpriv` | 인증 (SSH, sudo 등) |
| `cron` | cron |
| `daemon` | 시스템 데몬 |
| `mail` | 메일 |
| `local0`~`local7` | 사용자 정의 |

```bash
# rsyslog 설정 확인
cat /etc/rsyslog.conf | grep -v "^#" | grep -v "^$"

# 주요 라우팅 규칙 예시:
# authpriv.*    /var/log/secure      ← 인증 로그 → /var/log/secure
# cron.*        /var/log/cron        ← cron 로그 → /var/log/cron
# *.info        /var/log/messages    ← info 이상 전부 → /var/log/messages
```

## 4-2. logger — 스크립트에서 syslog에 기록

`logger` 명령은 Shell Script에서 **syslog에 메시지를 보내는** 도구입니다.

```bash
# 기본 사용 — /var/log/messages에 기록됨
logger "Backup completed successfully"

# Priority 지정
logger -p local0.err "Database connection failed"

# Tag(프로그램명) 지정
logger -t myapp "Application started"

# PID 포함
logger -t myapp -i "Processing request #12345"

# journalctl에서도 확인 가능
journalctl -t myapp --since "5 min ago"
```

> **AWS 연결 — EC2에서 메모리 사용량 기록:**
EC2 기본 메트릭에는 **Memory 사용량이 포함되지 않습니다.** Hypervisor 레벨에서는 Guest OS 내부의 메모리 상태를 모르기 때문입니다. Shell Script와 `logger`로 간단히 해결할 수 있습니다:
> 
> 
> ```bash
> #!/bin/bash
> # /opt/scripts/mem-logger.sh
> MEM_PERCENT=$(free | awk '/Mem:/ {printf "%.1f", ($3/$2)*100}')
> logger -t mem-monitor "Memory Usage: ${MEM_PERCENT}%"
> ```
> 
> cron으로 5분마다 실행:
> 
> ```
> */5 * * * * /opt/scripts/mem-logger.sh
> ```
> 
> 이 로그를 CloudWatch Agent로 수집하면 CloudWatch 메트릭처럼 사용 가능. 하지만 실무에서는 **CloudWatch Agent의 내장 메모리 수집 기능**(`mem_used_percent`)을 사용하는 것이 정석.
> 

## 4-3. Log Rotation — 로그가 디스크를 가득 채우지 않도록

**logrotate**는 로그 파일을 주기적으로 **회전(rotate)**시켜 디스크 고갈을 방지합니다. 오래된 로그를 압축하고, 일정 기간이 지나면 삭제합니다.

```bash
# logrotate 메인 설정
cat /etc/logrotate.conf

# 서비스별 설정
ls /etc/logrotate.d/
cat /etc/logrotate.d/syslog
```

**logrotate 설정 예시:**

```bash
sudo tee /etc/logrotate.d/myapp << 'EOF'
/var/log/myapp/*.log {
    daily           # 매일 회전
    rotate 14       # 14개 보관 (2주)
    compress        # 이전 로그 gzip 압축
    delaycompress   # 최신 1개는 압축 안 함
    missingok       # 파일 없어도 에러 안 냄
    notifempty      # 비어있으면 회전 안 함
    create 0640 myapp myapp    # 새 파일 생성 시 권한
    postrotate
        systemctl reload myapp 2>/dev/null || true
    endscript
}
EOF

# logrotate 테스트 (실제 실행하지 않고 시뮬레이션)
sudo logrotate -d /etc/logrotate.d/myapp

# 강제 실행
sudo logrotate -f /etc/logrotate.d/myapp
```

> **SRE 사고 사례:** logrotate가 제대로 설정되지 않아 `/var/log/messages`가 수십 GB로 커져서 `/var` 파일시스템이 100%가 되고, 그 결과 journald와 rsyslog가 새 로그를 쓸 수 없게 되어 **모든 로깅이 중단**되는 사고. 이후 발생하는 장애의 원인 분석이 불가능해집니다.
> 
> 
> **CKA 팁:** K8s Worker Node에서 `/var` 파일시스템 사용률이 일정 비율을 넘으면 kubelet이 **DiskPressure** 상태를 선언하고 Pod를 **Eviction**(축출)합니다. 이때 `du -sh /var/*`로 어떤 디렉터리가 범인인지 찾는 것이 시험에 나옵니다.
> 
> ```bash
> # /var 하위 용량 분석
> sudo du -sh /var/* 2>/dev/null | sort -rh | head -10
> 
> # Container 이미지가 범인인 경우
> sudo du -sh /var/lib/containerd/
> 
> # 오래된 로그가 범인인 경우
> sudo find /var/log -size +100M -exec ls -lh {} \;
> 
> # 삭제되었지만 프로세스가 잡고 있는 파일
> sudo lsof +L1 | grep /var/log
> ```
> 

---

# Part 5. 서비스 장애 분석 워크플로우

## 5-1. 체계적 장애 진단 순서

```
1단계: systemctl status <service>
  → Active 상태, Main PID, 최근 로그 몇 줄

2단계: journalctl -u <service> -n 50 --no-pager
  → 상세 로그에서 에러 메시지 확인

3단계: journalctl -u <service> -p err --since "10 min ago"
  → 에러만 필터링

4단계: systemctl cat <service>
  → Unit File 설정이 올바른지 확인

5단계: sudo -u <user> <ExecStart 명령어>
  → 해당 사용자로 직접 실행하여 Permission 등 확인

6단계: sudo ausearch -m avc --start recent
  → SELinux 차단 여부

7단계: sudo ss -tlnp | grep :<port>
  → Port 충돌 확인
```

## 5-2. 실전 예시: nginx 장애

```bash
# 1. 상태 확인
sudo systemctl status nginx
# Active: failed (Result: exit-code)

# 2. 로그 확인
sudo journalctl -u nginx -n 30 --no-pager
# nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)

# 3. 포트 80을 점유한 프로세스 찾기
sudo ss -tlnp | grep :80
# LISTEN  0  511  0.0.0.0:80  *:*  users:(("httpd",pid=12345,fd=4))

# 4. 충돌 프로세스 정리
sudo systemctl stop httpd    # 또는 kill

# 5. nginx 시작
sudo systemctl start nginx
sudo systemctl status nginx
```

> **CKS 연결 — strace로 프로세스 시스템 콜 모니터링:**
CKS에서는 **런타임 보안** 문제가 출제됩니다. 의심스러운 프로세스가 어떤 System Call을 호출하는지 추적하는 기초 기술:
> 
> 
> ```bash
> # 특정 PID의 System Call 추적
> sudo strace -p <PID> -f -e trace=network
> # -f: fork된 Child도 추적
> # -e trace=network: 네트워크 관련 시스템 콜만
> 
> # 프로세스가 열어보는 파일 추적
> sudo strace -p <PID> -e trace=open,openat
> 
> # 새 프로세스의 System Call 기록
> sudo strace -o /tmp/strace.log -f /usr/bin/suspicious-binary
> ```
> 
> CKS에서는 **Falco**(런타임 보안 도구)가 이런 System Call 모니터링을 자동화합니다. strace는 그 기반 개념입니다.
> 

---

# Part 6. 시스템 리소스 모니터링

## 6-1. CPU/Load Average — 시스템이 얼마나 바쁜가

### Load Average의 개념

```bash
uptime
# 14:30:22 up 10 days, 3:15, 2 users, load average: 1.50, 1.20, 0.80
#                                                     1분   5분   15분
```

Load Average는 **Run Queue에서 CPU를 기다리는 Process 수의 평균**입니다. CPU Core 수와 비교해야 의미가 있습니다.

| Load Average | CPU Core | 상태 |
| --- | --- | --- |
| 1.0 | 2 Core | 여유 (50% 사용) |
| 2.0 | 2 Core | 꽉 참 (100%) |
| 4.0 | 2 Core | **과부하** (200% — Process가 줄 서서 대기) |
| 4.0 | 8 Core | 여유 (50% 사용) |

```bash
# CPU Core 수 확인
nproc
cat /proc/cpuinfo | grep "model name" | wc -l

# 실시간 CPU 모니터링
top
htop    # 더 시각적 (설치 필요: sudo dnf install htop -y)
```

### top 헤더 해석 (이전 스터디 복습 + 심화)

```
%Cpu(s): 14.4 us,  0.2 sy,  0.0 ni, 85.0 id,  0.0 wa,  0.0 hi,  0.5 si,  0.0 st
```

| 필드 | 정상 | 이상 징후 | 진단 액션 |
| --- | --- | --- | --- |
| `us` (User) | 상황에 따라 | 지속 90%+ | CPU-bound 프로세스 찾기 (`ps --sort=-%cpu`) |
| `sy` (System) | <10% | 20%+ | 과도한 System Call, Context Switch (`vmstat` 확인) |
| `wa` (I/O Wait) | <5% | 10%+ | **Disk 병목!** `iostat`로 확인 |
| `st` (Steal) | 0% | 5%+ | **EC2 CPU Credit 소진** 또는 Noisy Neighbor |

> **AWS 연결:** EC2 기본 메트릭에는 `us`, `sy`, `wa`, `st` 구분이 **없습니다.** CloudWatch의 `CPUUtilization`은 이 모든 것을 합친 단일 숫자입니다. `wa`가 높은 것과 `us`가 높은 것은 **완전히 다른 원인**이므로, EC2에 SSH하여 `top`으로 직접 확인해야 합니다.
> 
> 
> `st`(Steal Time)가 높으면:
> 
> - Burstable Instance(t3 등) → CPU Credit 확인
> - Fixed Instance(m5 등) → Instance Type 업그레이드 또는 전용 호스트(Dedicated Host) 고려

## 6-2. Memory — 메모리가 정말 부족한가

```bash
# Memory 상태 확인
free -h
```

```
              total   used   free   shared  buff/cache  available
Mem:          3.4Gi  1.2Gi  200Mi    50Mi    2.0Gi      2.0Gi
Swap:         2.0Gi    0B   2.0Gi
```

**`free` vs `available` — 가장 흔한 오해:**

`free`가 200Mi밖에 안 남았다! → **이건 정상**입니다.

Linux는 여유 Memory를 **Page Cache**(`buff/cache`)로 활용하여 Disk I/O를 줄입니다. 앱이 Memory를 요청하면 Kernel이 Cache를 **즉시 회수**하여 제공합니다.

**진짜 여유 Memory = `available`** = `free` + 회수 가능한 `buff/cache`. `available`이 낮아야 진짜 걱정할 상황입니다.

```bash
# 상세 Memory 정보
cat /proc/meminfo | head -10

# Memory를 많이 쓰는 Process Top 10
ps -eo pid,ppid,user,%mem,rss,cmd --sort=-%mem | head -10

# Swap 사용 여부 확인 — 0이 아니면 Memory 부족 신호
vmstat 1 5
# si(Swap In), so(Swap Out) 컬럼 확인
```

> **K8s 연결:** K8s는 **Swap을 끄는 것이 원칙**입니다. kubelet이 Swap이 켜진 Node를 거부합니다. 이유: cgroup의 Memory Limit이 Swap까지 포함하면 **OOM Kill 타이밍이 예측 불가능**해지기 때문.
> 
> 
> ```bash
> # K8s Node에서 Swap 확인
> free -h | grep Swap
> swapon --show
> 
> # Swap 끄기 (K8s 요구사항)
> sudo swapoff -a
> sudo sed -i '/swap/d' /etc/fstab
> ```
> 

> **AWS 연결 — CloudWatch Agent의 Memory 수집:**
EC2 기본 CloudWatch 메트릭에 **Memory가 없는 이유:** CloudWatch는 Hypervisor 레벨에서 수집합니다. Memory 사용률은 **Guest OS 내부**의 정보(`/proc/meminfo`)이므로 Hypervisor가 알 수 없습니다.
> 
> 
> **CloudWatch Agent**를 설치하면 OS 내부에서 `/proc/meminfo`를 읽어 `mem_used_percent` 메트릭을 CloudWatch에 전송합니다.
> 
> SAA 시험: "EC2의 메모리 사용률을 CloudWatch에서 모니터링하려면?"
> → "기본 메트릭에 없으므로 **CloudWatch Agent를 설치**하여 Custom Metric으로 수집해야 한다."
> 

## 6-3. Disk — 용량과 I/O

```bash
# File System 사용량 (Type 포함)
df -hT

# Inode 사용량 — 작은 파일이 많으면 Disk 여유가 있어도 Inode 고갈
df -i

# 디렉터리별 용량 (Top-down)
sudo du -sh /* 2>/dev/null | sort -rh | head -10
sudo du -sh /var/* 2>/dev/null | sort -rh | head -10

# 대용량 파일 찾기
sudo find / -xdev -size +100M -exec ls -lh {} \; 2>/dev/null

# Disk I/O 실시간 모니터링
iostat -dxz 1

# 어떤 Process가 I/O를 많이 쓰는지
sudo iotop -o
```

**df -h 100%인데 du 합계가 안 맞는 경우:**

삭제된 파일을 프로세스가 아직 열어놓고 있으면 **디스크 공간이 반환되지 않습니다.** `lsof +L1`로 이런 파일을 찾고, 해당 프로세스를 재시작하면 공간이 반환됩니다.

```bash
# 삭제되었지만 아직 Disk를 점유하는 파일 찾기
sudo lsof +L1 | grep deleted

# 가장 큰 것부터
sudo lsof +L1 | awk '{print $7, $NF}' | sort -rn | head -10
```

> **CKA 연결 — DiskPressure Eviction:**
kubelet은 Node의 디스크 사용률이 임계값을 넘으면 **DiskPressure** Condition을 설정하고 Pod를 축출합니다.
> 
> 
> ```bash
> # Node의 Condition 확인
> kubectl describe node <node-name> | grep -A5 "Conditions"
> 
> # Node에 SSH하여 원인 분석
> df -hT
> sudo du -sh /var/lib/containerd/*   # Container 이미지/레이어
> sudo du -sh /var/log/pods/*         # Pod 로그
> sudo crictl rmi --prune             # 사용하지 않는 이미지 정리
> ```
> 

## 6-4. Process 모니터링

```bash
# CPU Top 10
ps -eo pid,ppid,user,%cpu,%mem,stat,cmd --sort=-%cpu | head -10

# Memory Top 10
ps -eo pid,ppid,user,%cpu,%mem,rss,cmd --sort=-%mem | head -10

# Zombie Process 찾기
ps aux | awk '$8=="Z" {print}'

# D State (Uninterruptible Sleep) — I/O 병목 신호
ps aux | awk '$8=="D" {print}'

# Process 수 확인
ps aux | wc -l

# 특정 Process의 Open File Descriptor 수
sudo ls /proc/<PID>/fd | wc -l

# Process Tree
pstree -p
```

## 6-5. 종합 성능 측정 도구

### vmstat — CPU/Memory/I/O/Swap 한 번에

```bash
vmstat 1 5    # 1초 간격, 5회
```

| 필드 | 의미 | 이상 징후 |
| --- | --- | --- |
| `r` | Run Queue 대기 Process | CPU Core 수보다 크면 과부하 |
| `b` | D State Process | I/O 병목 |
| `si`/`so` | Swap In/Out (KB/s) | **0이 아니면 Memory 부족** |
| `wa` | I/O Wait % | 높으면 Disk 병목 |

### iostat — Disk I/O 상세

```bash
iostat -dxz 1    # 1초 간격, 유휴 장치 제외

# 주요 필드:
# await   — 평균 I/O 대기 시간 (ms). 높으면 Disk 느림
# %util   — Device 사용률. 100%에 가까우면 포화
# r/s, w/s — 초당 Read/Write 수
```

### sar — 장기 성능 데이터

```bash
# sysstat 패키지 필요
sudo dnf install sysstat -y
sudo systemctl enable --now sysstat

# CPU 통계
sar -u 1 5

# Memory 통계
sar -r 1 5

# Network 통계
sar -n DEV 1 5

# 과거 데이터 (어제)
sar -u -f /var/log/sa/sa$(date -d yesterday +%d)

# 특정 시간대
sar -u -s 14:00:00 -e 16:00:00
```

> **SRE 실무:** "어제 오후 3시쯤 서비스가 느렸다는 Report가 있어." → `sar`로 해당 시간대의 CPU/Memory/Disk를 확인할 수 있습니다. `top`은 현재 상태만 보여주지만, `sar`는 **과거 데이터**를 기록합니다.
> 

### dmesg — Kernel 메시지

```bash
# 최근 Kernel 메시지
sudo dmesg | tail -30
sudo dmesg -T | tail -30     # 읽기 쉬운 시간 형식

# OOM Kill 확인
sudo dmesg | grep -i "oom\|killed process"

# Disk/NIC 오류
sudo dmesg | grep -iE "error|fail|i/o"

# 최근 Kernel 메시지 실시간
sudo dmesg -w
# 또는
sudo journalctl -k -f
```

---

# Part 7. 네트워크 소켓 모니터링

```bash
# TCP Listen 상태 확인 (★ 가장 자주)
sudo ss -tlnp

# "내 서비스(8080)가 안 떠요!" 진단
sudo ss -tlnp | grep :8080
# 결과 없음 → 서비스가 Listen하고 있지 않음 → systemctl status 확인
# 다른 프로세스가 점유 → 해당 프로세스 확인 후 처리

# 모든 TCP 연결 (Established)
ss -tn state established

# 특정 포트의 연결 수
ss -tn | grep :8080 | wc -l

# TIME_WAIT 수 (너무 많으면 Port 고갈)
ss -tn state time-wait | wc -l

# CLOSE_WAIT 수 (애플리케이션 버그 의심)
ss -tn state close-wait | wc -l

# Connection 상태별 통계
ss -s
```

**CLOSE_WAIT이 쌓이는 이유:**

CLOSE_WAIT은 상대방이 FIN을 보냈는데(연결 끊고 싶다), **이쪽 애플리케이션이 `close()`를 호출하지 않은 상태**입니다. 이것은 애플리케이션의 **Connection Leak 버그**입니다. CLOSE_WAIT이 지속적으로 증가하면 해당 서비스의 코드를 점검해야 합니다.

---

# Part 8. 자격증 연결 종합 정리

## CKA/CKS (100% 연관)

| 리눅스 기술 | 시험 적용 |
| --- | --- |
| `journalctl -u kubelet` | **Node NotReady → kubelet 로그에서 원인 찾기** (가장 빈번) |
| `crictl ps`, `crictl logs` | Static Pod(API Server, etcd) 장애 시 Container 로그 확인 |
| `/var/log/pods/` | kubectl 안 먹힐 때 Pod 로그 직접 확인 |
| `du -sh /var/*` | DiskPressure Eviction 시 범인 디렉터리 찾기 |
| `top` (wa, st) | Node 성능 병목 진단 |
| `grep`, `awk` + Audit Log | CKS — API 호출 감사, 보안 위반 사용자 추출 |
| `strace` | CKS — 런타임 보안, 의심스러운 프로세스 System Call 추적 |
| `ss -tlnp` | Service Port 충돌 진단 |
| `dmesg` / `journalctl -k` | OOM Kill, Disk 오류, NIC 문제 등 Kernel 레벨 진단 |

## AWS SAA / SysOps (연관)

| 리눅스 기술 | AWS 적용 |
| --- | --- |
| `/var/log/secure`, `/var/log/messages` | **CloudWatch Agent로 수집** → 중앙 로그 관리 |
| `free -h` (Memory) | EC2 기본 메트릭에 **Memory 없음** → Agent 필수 |
| `top`의 `st` (Steal Time) | Burstable Instance CPU Credit 소진 진단 |
| `logger` 명령 | Shell Script에서 syslog에 기록 → CloudWatch로 전송 |
| logrotate | 로그 디스크 관리 (EC2 EBS 비용 최적화) |
| `df -h` | EBS Volume 용량 부족 모니터링 |
| `sar` | 성능 트렌드 분석 (CloudWatch의 과거 데이터와 비교) |

## Terraform Associate (부분 연관)

| 리눅스 기술 | Terraform 적용 |
| --- | --- |
| journalctl / 로그 파일 | `remote-exec` Provisioner 실행 결과 디버깅 |
| Exit Code (`$?`) | Provisioner 성공/실패 판단 |
| `TF_LOG=TRACE` + 로그 분석 | Terraform 자체 디버그 로그 분석 |