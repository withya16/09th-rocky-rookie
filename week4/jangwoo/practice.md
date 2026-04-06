# Week 4 실습 — 프로세스 제어 및 서비스 관리

---

## 1. cgroup 트리 확인

`systemd-cgls`로 현재 시스템의 cgroup 계층 구조를 트리 형태로 출력한다.  
`system.slice`(시스템 서비스)와 `user.slice`(사용자 세션)로 나뉘는 v2 단일 계층을 직접 확인할 수 있다.

```bash
jangwoojung@localhost:~$ sudo systemd-cgls
[sudo] password for jangwoojung: 
CGroup /:
-.slice
├─user.slice
│ ├─user-0.slice                          # root 사용자 세션
│ │ ├─session-c103.scope
│ │ │ ├─32530 sudo systemd-cgls
│ │ │ ├─32555 sudo systemd-cgls
│ │ │ ├─32556 systemd-cgls
│ │ │ └─32557 less
│ │ └─user@0.service …
│ │   └─init.scope
│ │     ├─32537 /usr/lib/systemd/systemd --user
│ │     └─32539 (sd-pam)
│ └─user-1000.slice                       # jangwoojung 사용자 세션
│   ├─user@1000.service …
│   │ ├─session.slice
│   │ │ ├─gvfs-goa-volume-monitor.service
│   │ │ │ └─6734 /usr/libexec/gvfs-goa-volume-monitor
│   │ │ ├─org.gnome.SettingsDaemon.MediaKeys.service
│   │ │ │ └─6459 /usr/libexec/gsd-media-keys
│   │ │ ├─org.gnome.SettingsDaemon.Smartcard.service
│   │ │ │ └─6560 /usr/libexec/gsd-smartcard
│   │ │ ├─xdg-permission-store.service
│   │ │ │ └─6398 /usr/libexec/xdg-permission-store
│   │ │ ├─dbus-broker.service             # D-Bus 사용자 세션 버스
│   │ │ │ ├─6227 /usr/bin/dbus-broker-launch --scope user
│   │ │ │ └─6233 dbus-broker ...
│   │ │ ├─org.gnome.SettingsDaemon.Datetime.service
│   │ │ │ └─6444 /usr/libexec/gsd-datetime
│   │ │ ├─xdg-document-portal.service
│   │ │ │ ├─6937 /usr/libexec/xdg-document-portal
│   │ │ │ └─6953 fusermount3 ...
│   │ │ ├─org.gnome.SettingsDaemon.Housekeeping.service
│   │ │ │ └─6447 /usr/libexec/gsd-housekeeping
│   │ │ ├─xdg-desktop-portal.service
│   │ │ │ └─6919 /usr/libexec/xdg-desktop-portal
│   │ │ ├─org.freedesktop.IBus.session.GNOME.service  # 입력기(한글) 관련 서비스
│   │ │ │ ├─6433 /usr/bin/ibus-daemon --panel disable
│   │ │ │ ├─6599 /usr/libexec/ibus-dconf
│   │ │ │ ├─6602 /usr/libexec/ibus-extension-gtk3
│   │ │ │ ├─6798 /usr/libexec/ibus-engine-simple
│   │ │ │ └─9005 /usr/libexec/ibus-engine-hangul --ibus
│   │ │ ├─pipewire-pulse.service
│   │ │ │ └─6821 /usr/bin/pipewire-pulse
│   │ │ ├─wireplumber.service
│   │ │ │ └─6817 /usr/bin/wireplumber
│   │ │ └─org.gnome.SettingsDaemon.Wacom.service
```

`/sys/fs/cgroup/cgroup.stat`으로 현재 cgroup v2 시스템에서 활성화된 서브시스템별 인스턴스 수를 확인한다.  
`nr_descendants`는 전체 하위 cgroup 수, `nr_subsys_*`는 각 컨트롤러가 추적 중인 cgroup 수다.

```bash
jangwoojung@localhost:~$ sudo cat /sys/fs/cgroup/cgroup.stat
nr_descendants 145          # 현재 활성 cgroup 하위 노드 수
nr_subsys_cpuset 1
nr_subsys_cpu 72
nr_subsys_io 1
nr_subsys_memory 146        # memory 컨트롤러가 추적 중인 cgroup 수
nr_subsys_perf_event 147
nr_subsys_hugetlb 1
nr_subsys_pids 146
nr_subsys_rdma 1
nr_subsys_misc 1
nr_subsys_dmem 1
nr_dying_descendants 155    # 소멸 중인 하위 cgroup 수 (프로세스 종료 후 정리 중)
```

---

## 2. 서비스 상태 확인 (systemctl)

`crond` 서비스를 시작하고 상태를 확인한다.  
출력에서 `CGroup` 항목을 보면 해당 서비스가 `/system.slice/crond.service` 그룹에 속해 있음을 확인할 수 있다.

```bash
jangwoojung@localhost:~$ systemctl start crond
jangwoojung@localhost:~$ systemctl status crond
● crond.service - Command Scheduler
     Loaded: loaded (/usr/lib/systemd/system/crond.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-03-10 00:25:50 KST; 2 weeks 6 days ago
 Invocation: 68d42f4139664426a335a1b473abe801
   Main PID: 1443 (crond)
      Tasks: 1 (limit: 98020)
     Memory: 1.2M (peak: 5M)          # 현재 메모리 사용량 / 최대치
        CPU: 601ms                     # 누적 CPU 사용 시간
     CGroup: /system.slice/crond.service
             └─1443 /usr/sbin/crond -n

# 최근 실행 로그 — 매시 정각에 cron.hourly 작업이 돌고 있음을 확인
Mar 22 13:01:01 localhost.localdomain CROND[20852]: (root) CMD (run-parts /etc/cron.hourly)
Mar 23 03:01:01 localhost.localdomain CROND[21923]: (root) CMD (run-parts /etc/cron.hourly)
Mar 23 03:01:01 localhost.localdomain CROND[21922]: (root) CMDEND (run-parts /etc/cron.hourly)
Mar 26 01:01:01 localhost.localdomain CROND[31582]: (root) CMD (run-parts /etc/cron.hourly)
Mar 26 01:01:01 localhost.localdomain CROND[31581]: (root) CMDEND (run-parts /etc/cron.hourly)
Mar 30 02:01:01 localhost.localdomain CROND[33041]: (root) CMD (run-parts /etc/cron.hourly)
Mar 30 02:01:01 localhost.localdomain CROND[33040]: (root) CMDEND (run-parts /etc/cron.hourly)
```

---

## 3. 서비스 설정 편집 (systemctl edit)

`systemctl edit`는 원본 `.service` 파일을 건드리지 않고 드롭인(drop-in) 파일로 설정을 덮어쓰는 방식이다.  
sudo 없이 실행하면 Permission denied가 발생하며, 아무것도 입력하지 않고 저장하면 파일이 생성되지 않는다.

```bash
# sudo 없이 실행 → 쓰기 권한 없음
jangwoojung@localhost:~$ systemctl edit crond
Failed to create parent directories for '/etc/systemd/system/crond.service.d/override.conf': Permission denied

# sudo로 실행 → nano 편집기 열림
jangwoojung@localhost:~$ sudo systemctl edit crond
[sudo] password for jangwoojung: 
# 아무것도 입력하지 않고 종료 → 빈 파일은 저장되지 않음
/etc/systemd/system/crond.service.d/override.conf: after editing, new contents are empty, not writing file.
```

편집기 내부에서는 원본 `crond.service`의 내용이 주석으로 표시되어 참고할 수 있다.

```
### Editing /etc/systemd/system/crond.service.d/override.conf
### Anything between here and the comment below will become the contents of the drop-in file

### Edits below this comment will be discarded

### /usr/lib/systemd/system/crond.service   ← 원본 파일 내용 (수정 불가, 참고용)
# [Unit]
# Description=Command Scheduler
# After=auditd.service nss-user-lookup.target systemd-user-sessions.service time-sync.target ypbind.service autofs.service
# 
# [Service]
# EnvironmentFile=/etc/sysconfig/crond
# ExecStart=/usr/sbin/crond -n $CRONDARGS
# ExecReload=/bin/kill -URG $MAINPID
# KillMode=process
# Restart=on-failure
# RestartSec=30s
# 
# [Install]
# WantedBy=multi-user.target
```

---

## 4. 부팅 성능 분석 (systemd-analyze)

`systemd-analyze blame`으로 각 서비스의 부팅 소요 시간을 내림차순으로 확인한다.  
`kdump.service`(23초)와 `NetworkManager-wait-online`(4초)이 부팅 병목임을 파악할 수 있다.

```bash
jangwoojung@localhost:~$ systemd-analyze blame
23.002s kdump.service                              # 커널 덤프 서비스 — 가장 오래 걸림
 4.063s NetworkManager-wait-online.service         # 네트워크 연결 대기
 3.615s plymouth-quit-wait.service                 # 부팅 스플래시 화면 종료 대기
 3.086s sys-module-fuse.device
 2.988s sys-devices-LNXSYSTM:00-...-tpm-tpm0.device
 2.988s dev-tpm0.device
 2.982s sys-devices-platform-serial8250-...ttyS2.device
 2.982s dev-ttyS2.device
 2.976s sys-devices-platform-serial8250-...ttyS3.device
 2.976s dev-ttyS3.device
 2.976s sys-devices-pci0000:00-...-ttyS1.device
 2.976s dev-ttyS1.device
 2.975s sys-devices-LNXSYSTM:00-...-tpmrm0.device
 2.975s dev-tpmrm0.device
 2.974s sys-devices-platform-serial8250-...ttyS0.device
 2.974s dev-ttyS0.device
 2.963s sys-module-configfs.device
```

`systemd-analyze critical-chain`으로 부팅 시 가장 긴 경로(병목 체인)를 트리 형태로 확인한다.  
`graphical.target`까지 총 7.4초, 그 원인은 `NetworkManager-wait-online`(4.06초)임을 추적할 수 있다.

```bash
jangwoojung@localhost:~$ systemd-analyze critical-chain
# @ : 해당 유닛이 active 상태가 된 시각
# + : 해당 유닛이 시작되는 데 걸린 시간

graphical.target @7.419s
└─multi-user.target @7.419s
  └─rsyslog.service @7.364s +54ms
    └─network-online.target @7.350s
      └─NetworkManager-wait-online.service @3.286s +4.063s   ← 병목 구간
        └─NetworkManager.service @2.813s +467ms
          └─network-pre.target @2.812s
            └─firewalld.service @2.250s +560ms
              └─polkit.service @1.868s +361ms
                └─basic.target @1.833s
                  └─sockets.target @1.833s
                    └─cockpit.socket @1.783s +47ms
                      └─sysinit.target @1.777s
                        └─systemd-update-utmp.service @1.761s +14ms
                          └─auditd.service @1.737s +21ms
                            └─systemd-tmpfiles-setup.service @1.623s +112ms
                              └─local-fs.target @1.618s
                                └─boot-efi.mount @1.595s +22ms
                                  └─boot.mount @1.565s +27ms
                                    └─dev-nvme0n1p2.device   ← 부팅 시작점 (디스크 인식)
```

`systemctl list-units`로 현재 로드된 유닛 전체를 확인한다.

```bash
jangwoojung@localhost:~$ systemctl list-units
# LOAD: 유닛 파일 로드 여부 / ACTIVE: 현재 활성 상태 / SUB: 세부 상태
 UNIT                                           LOAD   ACTIVE SUB       DESCRIPTION
 proc-sys-fs-binfmt_misc.automount              loaded active waiting   Arbitrary Executable File Formats ...
 sys-devices-...-tpm-tpm0.device                loaded active plugged   /sys/.../tpm/tpm0
 sys-devices-...-tpmrm-tpmrm0.device            loaded active plugged   /sys/.../tpmrm/tpmrm0
 sys-devices-...-intel_backlight.device         loaded active plugged   /sys/.../intel_backlight
 sys-devices-...-bluetooth-hci0.device          loaded active plugged   /sys/.../bluetooth/hci0
 sys-devices-...-net-wlp0s20f3.device           loaded active plugged   Comet Lake PCH-LP CNVi WiFi (AX201)
 sys-devices-...-ptp-ptp0.device                loaded active plugged   /sys/.../ptp/ptp0
 sys-devices-...-nvme0n1p1.device               loaded active plugged   SAMSUNG MZVLB256HBHQ EFI System Partition
 sys-devices-...-nvme0n1p2.device               loaded active plugged   SAMSUNG MZVLB256HBHQ 2
 sys-devices-...-nvme0n1p3.device               loaded active plugged   SAMSUNG MZVLB256HBHQ 3
```

`systemctl list-unit-files`로 설치된 유닛 파일의 목록과 활성화 상태(enabled/disabled/static)를 확인한다.

```bash
jangwoojung@localhost:~$ systemctl list-unit-files
# STATE: enabled(부팅 시 자동 시작) / disabled(수동 시작) / static(다른 유닛에 의해 활성화)
UNIT FILE                                          STATE           PRESET
proc-sys-fs-binfmt_misc.automount                  static          -
-.mount                                            generated       -       # fstab에서 자동 생성
boot-efi.mount                                     generated       -
boot.mount                                         generated       -
dev-hugepages.mount                                static          -
dev-mqueue.mount                                   static          -
home.mount                                         generated       -
mnt-raid_data.mount                                generated       -
proc-sys-fs-binfmt_misc.mount                      disabled        disabled
run-vmblock\x2dfuse.mount                          disabled        disabled
cups.path                                          enabled         enabled   # 프린터 서비스
lvm-devices-import.path                            disabled        disabled
systemd-ask-password-console.path                  static          -
session-2.scope                                    transient       -
accounts-daemon.service                            enabled         enabled
alsa-restore.service                               static          -
alsa-state.service                                 static          -
```

---

## 5. 프로세스 모니터링 (ps, top)

`ps -ef | grep sshd`로 sshd 데몬 프로세스를 조회한다.  
PID 1432, PPID 1(systemd)임을 확인 — sshd가 systemd의 자식 프로세스로 관리되고 있다.

```bash
jangwoojung@localhost:~$ ps -ef | grep sshd
# UID    PID   PPID  ... COMMAND
root    1432      1  0 Mar10 ?  00:00:00 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
jangwoo+   34747  13017  0 02:40 pts/0  00:00:00 grep --color=auto sshd
```

`top`으로 실시간 자원 현황을 확인한다.

```bash
jangwoojung@localhost:~$ top

top - 02:40:59 up 20 days,  2:15,  2 users,  load average: 0.97, 0.89, 0.78
# load average: 1분/5분/15분 평균 부하 — 8코어 시스템에서 0.97은 여유 있는 상태
Tasks: 328 total,   2 running, 326 sleeping,   0 stopped,   0 zombie
%Cpu(s): 10.6 us,  1.2 sy,  0.0 ni, 86.8 id,  0.4 wa,  0.8 hi,  0.2 si,  0.0 st
# us: 사용자, sy: 커널, id: 유휴, wa: I/O 대기 — wa가 낮으므로 I/O 병목 없음
MiB Mem :  15418.2 total,   9313.0 free,   4723.6 used,   2374.2 buff/cache
MiB Swap:   7904.0 total,   7904.0 free,      0.0 used.  10694.5 avail Mem
# Swap 사용량 0 → 메모리 여유 충분

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   7528 jangwoo+  20   0 9996.1m   2.0g 189356 R  62.9  13.1 166:42.69 Isolated Web Co  # 브라우저 격리 탭
   6319 jangwoo+  20   0 5205132 317212 114168 S  13.9   2.0  17:56.25 gnome-shell
  12972 jangwoo+  20   0 2340120 101508  68924 S  13.6   0.6   2:52.31 ptyxis            # 터미널
   7082 jangwoo+  20   0 8014852 945176 534460 S   1.0   6.0  37:37.03 firefox
  34749 jangwoo+  20   0  231640   5600   3388 R   1.0   0.0   0:00.15 top
     39 root      20   0       0      0      0 S   0.3   0.0   0:02.10 ksoftirqd/3       # 커널 소프트 IRQ
   1026 root     -51   0       0      0      0 S   0.3   0.0   0:15.56 irq/51-04CA00A0:00 # 하드웨어 인터럽트 핸들러 (PR=-51: 실시간)
```

---

## 6. 메모리 및 CPU 자원 모니터링

`free -h`로 전체 메모리 상태를 확인한다. `available`이 10Gi로 여유롭고 swap 사용량은 0이다.

```bash
jangwoojung@localhost:~$ free -h
               total        used        free      shared  buff/cache   available
Mem:            15Gi       4.6Gi       9.1Gi       691Mi       2.3Gi        10Gi
# available = free + 회수 가능한 buff/cache → 실제 사용 가능한 메모리
Swap:          7.7Gi          0B       7.7Gi
```

`vmstat 1`로 1초 간격으로 메모리/swap/CPU 상태 변화를 추적한다.  
`si`/`so`(Swap-in/out)가 모두 0 → 스왑 활동 없음, 시스템 안정 상태.

```bash
jangwoojung@localhost:~$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
# r: 실행 대기 프로세스 수 / b: I/O 대기 수 / si/so: 스왑 입출력(0이면 정상)
 1  0      0 9586768   7532 2422248    0    0     0     8   53    1  5  1 94  0  0  0
 3  0      0 9586724   7532 2422064    0    0     0     0 2322 2181  3  2 95  0  0  0
 1  0      0 9586220   7532 2422000    0    0     0     0 2816  753 24  1 75  0  0  0
 0  0      0 9586220   7532 2422000    0    0     0   173  988 1001  6  0 94  0  0  0
 0  0      0 9586220   7532 2422000    0    0     0     0  449  570  1  0 99  0  0  0
 1  0      0 9586220   7532 2422024    0    0     0     0 2452 1469  1  1 98  0  0  0
```

`dmesg | grep -i oom`으로 OOM Killer 발동 기록을 조회한다.  
`disabled`/`enabled` 쌍이 보이는 것은 VM 메모리 balloon 드라이버가 잠시 OOM을 비활성화한 것으로, 실제 프로세스 강제 종료는 없었다.

```bash
jangwoojung@localhost:~$ sudo dmesg | grep -i oom
[ 1256.532903] OOM killer disabled.   # 메모리 회수 작업이 진행 중인 것으로 추정 (balloon 드라이버 등 가능성)
[ 1257.294848] OOM killer enabled.    # 회수 작업 완료 후 재활성화된 것으로 보임
[ 2131.033299] OOM killer disabled.
[ 2131.758928] OOM killer enabled.
[ 2730.625299] OOM killer disabled.
# → "OOM: kill process" 메시지 없음 → 이 시점에 강제 종료된 프로세스는 없었던 것으로 확인됨
# (disabled/enabled 원인은 환경에 따라 다를 수 있으므로 dmesg 전체 맥락과 함께 판단 필요)
```

`uptime`으로 시스템 가동 시간과 로드 에버리지를 빠르게 확인한다.

```bash
jangwoojung@localhost:~$ uptime
 02:45:35 up 20 days,  2:19,  2 users,  load average: 0.72, 0.89, 0.82
# 8코어 기준 load average 0.72 → CPU 여유 있음
```

`lscpu`로 CPU 코어 수를 확인한다. 8코어 시스템임을 파악.

```bash
jangwoojung@localhost:~$ lscpu | grep "^CPU(s)"
CPU(s):                                  8
CPU(s) scaling MHz:                      21%    # 현재 절전 모드로 클럭이 낮게 동작 중
```

`mpstat -P ALL 1`로 코어별 CPU 사용률을 1초 간격으로 분석한다.  
코어별로 부하가 고르게 분산되지 않음을 확인할 수 있다 (1번 20%, 2번 21%, 4번 3% 등).

```bash
jangwoojung@localhost:~$ mpstat -P ALL 1
Linux 6.12.0-124.40.1.el10_1.x86_64 (localhost.localdomain)  03/30/2026  _x86_64_  (8 CPU)

02:47:14 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
02:47:15 AM  all   10.51    0.00    1.13    0.13    0.50    0.13    0.00    0.00    0.00   87.61  # 전체 평균
02:47:15 AM    0    3.96    0.00    1.98    0.00    0.99    0.99    0.00    0.00    0.00   92.08
02:47:15 AM    1   20.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   80.00
02:47:15 AM    2   21.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   79.00  # 가장 바쁜 코어
02:47:15 AM    3    9.90    0.00    0.99    0.00    1.98    0.00    0.00    0.00    0.00   87.13
02:47:15 AM    4    2.02    0.00    1.01    0.00    0.00    0.00    0.00    0.00    0.00   96.97  # 가장 여유로운 코어
02:47:15 AM    5   19.19    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   80.81
02:47:15 AM    6    8.16    0.00    4.08    0.00    0.00    0.00    0.00    0.00    0.00   87.76
02:47:15 AM    7    0.00    0.00    0.99    0.99    0.99    0.00    0.00    0.00    0.00   97.03
```

---

## 7. CPU 부하 유발 및 프로세스 우선순위/시그널 제어

`yes > /dev/null`로 CPU를 100% 점유하는 인위적 부하를 만들어 nice/kill을 실습한다.

```bash
# sudo yes를 백그라운드(&)로 실행 → CPU 100% 점유 프로세스 생성
jangwoojung@localhost:~$ sudo yes > /dev/null &
[1] 35160

jangwoojung@localhost:~$ top
top - 02:48:24 up 20 days,  2:22,  4 users,  load average: 1.39, 1.01, 0.87
# load average 1.39 → yes 프로세스 하나가 CPU를 독점하며 부하 상승
Tasks: 336 total,   2 running, 334 sleeping,   0 stopped,   0 zombie
%Cpu(s):  3.3 us, 10.0 sy,  0.0 ni, 86.5 id,  0.1 wa,  0.0 hi,  0.0 si,  0.0 st

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  35184 root      20   0  226820   1852   1728 R  99.7   0.0   0:07.04 yes    # CPU 99.7% 점유 중

jangwoojung@localhost:~$ free -h
               total        used        free      shared  buff/cache   available
Mem:            15Gi       4.7Gi       9.0Gi       684Mi       2.3Gi        10Gi
Swap:          7.7Gi          0B       7.7Gi

# renice로 yes 프로세스의 우선순위를 낮춤 (NI: 0 → 10)
# 다른 서비스들이 CPU를 더 많이 가져갈 수 있게 됨
jangwoojung@localhost:~$ sudo renice -n 10 -p 35184
35184 (process ID) old priority 0, new priority 10

# 일반 사용자는 root 소유 프로세스에 SIGKILL 불가 → Permission denied
jangwoojung@localhost:~$ kill -9 35184
bash: kill: (35184) - Operation not permitted

# sudo로 SIGKILL(9번) 전송 → 즉시 강제 종료
jangwoojung@localhost:~$ sudo kill -9 35184
[1]+  Killed                  sudo yes > /dev/null

jangwoojung@localhost:~$ free -h
               total        used        free      shared  buff/cache   available
Mem:            15Gi       4.7Gi       9.0Gi       692Mi       2.3Gi        10Gi
Swap:          7.7Gi          0B       7.7Gi
# 메모리 사용량 변화 없음 → yes는 메모리를 거의 쓰지 않고 CPU만 소모하는 프로세스
```

---

## 8. 작업 스케줄링 (cron, at)

`crontab -e`로 매 분마다 현재 시각을 파일에 기록하는 cron 작업을 등록한다.

```bash
jangwoojung@localhost:~$ crontab -e
no crontab for jangwoojung - using an empty one
crontab: installing new crontab

# 등록한 내용 확인
jangwoojung@localhost:~$ crontab -l
* * * * * date >> /tmp/cron_test.log
# → 매분 0초에 date 출력을 /tmp/cron_test.log에 append

# 파일이 생기기 전에 tail 실행 → 아직 파일 없음
jangwoojung@localhost:~$ tail -f /tmp/cron_test.log
tail: cannot open '/tmp/cron_test.log' for reading: No such file or directory
tail: no files remaining

# 1분 대기 후 crond가 실행 → 로그 파일 생성 및 기록 확인
jangwoojung@localhost:~$ tail -f /tmp/cron_test.log
Mon Mar 30 02:55:01 AM KST 2026
```

`at`으로 1분 뒤에 딱 한 번 실행되는 일회성 작업을 예약한다.

```bash
# 1분 뒤 실행 예약
jangwoojung@localhost:~$ at now +1 minute
warning: commands will be executed using /bin/sh
at Mon Mar 30 02:56:00 2026
at> echo "hello at" >> /tmp/at_test.log    # 실행할 명령어 입력
at> <EOT>                                   # Ctrl+D로 입력 종료
job 1 at Mon Mar 30 02:56:00 2026

# 실행 전 대기열 확인 (job이 있으면 표시됨)
jangwoojung@localhost:~$ atq
# 대기열 비어 있음 → 이미 실행 완료

# 실행 결과 확인
jangwoojung@localhost:~$ cat /tmp/at_test.log
hello at
```

---

## 9. cgroup 자원 제한 (systemctl set-property)

`yes > /dev/null` 부하를 유지한 상태에서, `user.slice` 전체의 CPU 사용량을 20%로 제한해본다.  
`/proc/<PID>/cgroup`으로 해당 프로세스가 실제로 어느 cgroup에 속해 있는지 먼저 확인한다.

```bash
# 백그라운드 CPU 부하 생성
jangwoojung@localhost:~$ yes > /dev/null &
[1] 35465

jangwoojung@localhost:~$ ps aux | grep yes
jangwoo+   35465 99.8  0.0 226820  1912 pts/0    R    03:00   0:13 yes
jangwoo+   35472  0.0  0.0 227688  2188 pts/0    S+   03:00   0:00 grep --color=auto yes

# yes 프로세스가 속한 cgroup 경로 확인
jangwoojung@localhost:~$ cat /proc/$(pgrep yes)/cgroup
0::/user.slice/user-1000.slice/user@1000.service/app.slice/ptyxis-spawn-e44c5d82-...scope
# → user.slice 하위에 속해 있음을 확인

# user.slice 전체에 CPUQuota=20% 제한 적용
jangwoojung@localhost:~$ sudo systemctl set-property user.slice CPUQuota=20%

jangwoojung@localhost:~$ top
top - 03:02:50 up 20 days,  2:37,  2 users,  load average: 0.41, 0.80, 0.90
# load average 감소가 관찰됨 — CPUQuota 적용 효과일 가능성이 높으나,
# 실습 시점의 다른 부하 변화도 함께 영향을 줄 수 있으므로 관찰 결과로 참고할 것
Tasks: 327 total,   4 running, 323 sleeping,   0 stopped,   0 zombie
%Cpu(s):  1.7 us,  1.2 sy,  0.0 ni, 96.4 id,  0.0 wa,  0.5 hi,  0.1 si,  0.0 st

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   7082 jangwoo+  20   0 8005052 929248 534408 S   5.2   5.9  38:26.96 firefox
   7528 jangwoo+  20   0    9.8g   2.0g 189356 R   3.6  13.2 176:40.08 Isolated Web Co
  35465 jangwoo+  20   0  226820   1912   1792 R   3.6   0.0   1:33.88 yes
  # yes가 99% → 3.6%로 제한됨

# 프로세스 종료 및 CPUQuota 설정 초기화
jangwoojung@localhost:~$ kill -9 35465
[1]+  Killed                  yes > /dev/null

jangwoojung@localhost:~$ sudo systemctl set-property user.slice CPUQuota=
# 값을 비워서 전달하면 해당 설정이 제거됨

# 초기화 결과 확인
jangwoojung@localhost:~$ systemctl show user.slice -p CPUQuota
jangwoojung@localhost:~$
# 출력 없음 → CPUQuota 제한이 해제됨
```

---

## 추가 실습

### A. 프로세스 우선순위 및 스케줄러 정책 확인

`ps -eo`로 각 프로세스의 Nice 값과 실시간 스케줄러 정책을 직접 확인한다.

```bash
# NI(Nice 값), PRI(실제 우선순위), STAT(상태) 확인
jangwoojung@localhost:~$ ps -eo pid,comm,nice,pri,stat | head -20

# nice로 낮은 우선순위(+15) 백그라운드 작업 실행
# sha256sum /dev/urandom은 무한히 CPU를 소모하는 테스트용 부하
jangwoojung@localhost:~$ nice -n 15 sha256sum /dev/urandom &
[1] <PID>

# renice로 우선순위 조정 (일반 사용자는 낮추기만 가능, 높이려면 sudo 필요)
jangwoojung@localhost:~$ renice -n 19 -p <PID>

# chrt로 스케줄러 정책 확인
jangwoojung@localhost:~$ chrt -p <PID>

# 작업 종료
jangwoojung@localhost:~$ kill %1
```

### B. 시그널 종류별 동작 확인

```bash
# 백그라운드 sleep 프로세스 생성
jangwoojung@localhost:~$ sleep 300 &
[1] <PID>

# SIGTERM(15): 정상 종료 요청 — 프로세스가 스스로 종료
jangwoojung@localhost:~$ kill -15 <PID>
[1]+  Terminated              sleep 300

# SIGHUP(1): 설정 재로드 (sshd, nginx 등에 사용)
# 아래는 실제 동작 예시 (서비스가 설치되어 있을 경우)
jangwoojung@localhost:~$ sudo kill -1 $(pgrep sshd)
# → sshd가 /etc/ssh/sshd_config를 다시 읽음. 프로세스는 종료되지 않음

# SIGSTOP(19): 일시 정지 / SIGCONT(18): 재개
jangwoojung@localhost:~$ sleep 300 &
[1] <PID>
jangwoojung@localhost:~$ kill -19 <PID>   # 프로세스 일시 정지
jangwoojung@localhost:~$ kill -18 <PID>   # 프로세스 재개
jangwoojung@localhost:~$ kill -9 <PID>    # 강제 종료
```

### C. systemd 타이머 확인 (cron 대체)

RHEL 9부터 시스템 핵심 유지보수 작업이 cron에서 systemd 타이머로 이관되었다.  
현재 활성화된 타이머 목록과 다음 실행 시각을 확인해본다.

```bash
# 활성화된 모든 타이머 확인
jangwoojung@localhost:~$ systemctl list-timers --all

# cron 대신 timer를 사용하는 대표적인 작업 확인
jangwoojung@localhost:~$ systemctl status logrotate.timer
jangwoojung@localhost:~$ systemctl status dnf-makecache.timer
```

### D. tuned 프로파일 확인

```bash
# 현재 적용 중인 튜닝 프로파일 확인
jangwoojung@localhost:~$ tuned-adm active

# 현재 시스템 환경에 맞는 추천 프로파일 확인
jangwoojung@localhost:~$ tuned-adm recommend

# 사용 가능한 전체 프로파일 목록
jangwoojung@localhost:~$ tuned-adm list
```

### E. journalctl로 crond 로그 조회

```bash
# crond 서비스의 오늘 로그 전체 확인
jangwoojung@localhost:~$ journalctl -u crond --since today

# 최근 1시간 이내 로그만 확인
jangwoojung@localhost:~$ journalctl -u crond --since "1 hour ago"

# 실시간 로그 스트리밍 (cron 작업 실행되는 순간 확인용)
jangwoojung@localhost:~$ journalctl -u crond -f
```

### F. cgroup 실시간 모니터링 (systemd-cgtop)

```bash
# cgroup별 실시간 CPU/메모리 사용량 확인 (top과 유사)
jangwoojung@localhost:~$ systemd-cgtop

# 특정 cgroup 경로만 확인
jangwoojung@localhost:~$ systemd-cgtop /user.slice
```
