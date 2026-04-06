> **원문:** Red Hat Enterprise Linux 10 — Using systemd unit files to customize and optimize your system
> 

> **범위:** systemd의 내부 구조, Unit의 개념, Unit File의 Section별 Directive, Service Lifecycle 관리, Dependency 설계, Drop-in Override, Target, Timer, journald
> 

https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/index

---

# 1. systemd는 무엇이고, 왜 필요한가

## 1-1. init System의 역사

Linux Kernel은 부팅이 완료되면 **딱 하나의 Process**를 실행합니다. 이것이 **PID 1**이며, 이후 시스템의 모든 Process는 이 PID 1로부터 fork()되어 생성됩니다. PID 1이 죽으면 시스템 전체가 죽습니다 — 그만큼 핵심적인 Process입니다.

전통적으로 이 PID 1의 역할을 했던 것이 **SysVinit**입니다. SysVinit은 `/etc/init.d/` 디렉터리에 있는 Shell Script들을 **순차적으로(Sequential)** 실행하여 Service를 시작했습니다. 이 방식은 간단하지만 근본적인 한계가 있었습니다:

- **순차 실행으로 인한 느린 부팅:** Service A가 끝나야 B가 시작됩니다. CPU가 8코어여도 1코어만 사용하는 셈입니다.
- **Dependency 관리 부재:** Script의 실행 순서를 번호(S01, S02...)로 수동 관리했습니다. Service 간 의존관계가 복잡해지면 순서를 맞추기 어렵습니다.
- **Process 추적 불가:** Service가 fork()하여 자식 Process를 만들면, init은 원래 Process의 PID만 알고 자식들을 추적하지 못합니다. `kill`로 Service를 죽여도 자식 Process가 좀비로 남는 문제가 빈번했습니다.
- **On-demand 시작 불가:** 부팅 시 모든 Service를 미리 시작해야 했습니다. 실제 요청이 올 때까지 사용하지 않는 Service도 메모리를 점유합니다.

**systemd**는 이 모든 문제를 해결하기 위해 2010년 Lennart Poettering이 설계했고, RHEL 7(2014)부터 기본 init System으로 채택되었습니다. RHEL 10에서도 동일하게 PID 1으로 동작합니다.

## 1-2. systemd가 해결한 것들

| 문제 | SysVinit | systemd |
| --- | --- | --- |
| 부팅 속도 | Sequential 실행 | **Parallel 실행** — Dependency가 없는 Service들은 동시에 시작 |
| Dependency | 수동 번호 순서 | **선언적 Dependency Resolution** — `After=`, `Requires=` 등으로 관계를 명시 |
| Process 추적 | PID 파일 기반 (불안정) | **cgroup 기반 추적** — fork()한 모든 자식 Process까지 커널 레벨에서 추적 |
| On-demand 시작 | 불가 | **Socket/Path/Timer Activation** — 요청이 올 때 Service를 자동 시작 |
| Service 정의 | Shell Script (수백 줄) | **INI 형식의 Unit File** (선언적, 간결) |
| Logging | syslog에 의존 | **journald 내장** — 구조화된 로그, 메타데이터 포함 |
| Resource 제어 | 불가 | **cgroup 통합** — CPU, Memory, I/O 제한을 Unit 단위로 설정 |

> **AWS EC2 연결:** EC2 Instance가 부팅될 때 `cloud-init`이 먼저 실행되고(User Data Script 처리), 이후 systemd가 Service들을 시작합니다. systemd의 Parallel Startup 덕분에 EC2의 부팅 시간이 단축됩니다. Auto Scaling Group에서 Scale-out 속도에 직접 영향을 주므로, 불필요한 Service를 disable하여 부팅 시간을 최적화하는 것이 SRE의 핵심 작업입니다.
> 

> **Kubernetes 연결:** K8s Worker Node에서 `kubelet`은 systemd Service로 실행됩니다. kubelet이 죽으면 Node가 NotReady 상태가 되어 Pod Scheduling이 중단됩니다. systemd의 `Restart=always` 설정이 kubelet의 안정성을 보장합니다. `containerd`, `docker` 등 Container Runtime도 모두 systemd Unit입니다.
> 

---

# 2. Unit — systemd가 관리하는 모든 것

## 2-1. Unit의 개념

systemd에서 관리하는 **모든 대상**을 Unit이라고 부릅니다. Service(Daemon)뿐 아니라, Mount Point, Socket, Device, Timer 등 시스템의 거의 모든 자원이 Unit으로 표현됩니다. 각 Unit은 **이름.타입** 형식입니다 (예: `sshd.service`, `home.mount`).

이 설계가 강력한 이유는 **모든 것을 동일한 방식으로 관리**할 수 있기 때문입니다. Service든 Mount Point든 Timer든 전부 `systemctl start/stop/enable/disable`로 제어합니다.

## 2-2. Unit Type 상세

| Unit Type | 확장자 | 역할 | 실제 예시 |
| --- | --- | --- | --- |
| **Service** | `.service` | Daemon/Process 관리. 가장 핵심적인 Unit | `sshd.service`, `httpd.service`, `kubelet.service` |
| **Socket** | `.socket` | Socket-based Activation. 요청이 오면 Service를 자동 시작 | `sshd.socket` — SSH 연결 요청이 올 때만 sshd 시작 |
| **Target** | `.target` | Unit들의 그룹화. SysVinit의 Runlevel을 대체 | `multi-user.target` (CLI 모드), `graphical.target` (GUI 모드) |
| **Timer** | `.timer` | 시간 기반 Service 실행. cron의 systemd 대체제 | `logrotate.timer`, `dnf-makecache.timer` |
| **Mount** | `.mount` | File System Mount Point 관리. `/etc/fstab`과 연동 | `home.mount`, `var-lib-docker.mount` |
| **Automount** | `.automount` | 접근 시에만 자동 Mount (on-demand) | NFS Share를 필요할 때만 Mount |
| **Path** | `.path` | 파일/디렉터리 변경 감지 시 Service 트리거 | 특정 디렉터리에 파일이 생기면 처리 Service 시작 |
| **Slice** | `.slice` | cgroup Resource 그룹화 | `user.slice`, `system.slice`, `machine.slice` |
| **Scope** | `.scope` | 외부에서 생성된 Process 그룹 관리 | `session-1.scope` (사용자 로그인 세션) |
| **Swap** | `.swap` | Swap 영역 관리 | `dev-dm\x2d1.swap` |
| **Device** | `.device` | udev로 감지된 Device | Kernel이 인식한 하드웨어 장치 |

**Socket Activation이 왜 중요한가:**

전통적으로 sshd는 부팅 시 항상 실행되어 메모리를 점유했습니다. Socket Activation을 사용하면 `sshd.socket`만 열어두고, 실제 SSH 연결 요청이 들어올 때 systemd가 `sshd.service`를 자동으로 시작합니다. 연결이 없으면 Service는 실행되지 않아 메모리를 절약합니다.

이 개념은 **AWS Lambda의 Cold Start**와 유사합니다. Lambda도 요청이 올 때만 Container를 시작합니다. Socket Activation은 이 아이디어의 리눅스 원조격입니다.

**실습:**

```bash
# 모든 Unit 목록 확인 (활성/비활성 모두)
sudo systemctl list-units --all

# Service Unit만 필터링
sudo systemctl list-units --type=service

# 실패(failed) 상태인 Unit만 확인
sudo systemctl list-units --state=failed

# 설치된 모든 Unit File 목록 (enabled/disabled 포함)
sudo systemctl list-unit-files

# Socket Unit 확인
sudo systemctl list-units --type=socket

# Timer Unit 확인
sudo systemctl list-timers --all
```

```python
mr8356@mr8356:~$ sudo systemctl list-units --type=service
  UNIT                                                  LOAD   ACTIVE SUB     D>
  atd.service                                           loaded active running D>
  auditd.service                                        loaded active running S>
  chronyd.service                                       loaded active running N>
  crond.service                                         loaded active running C>
  dbus-broker.service                                   loaded active running D>
  dracut-shutdown.service                               loaded active exited  R>
  firewalld.service                                     loaded active running f>
  getty@tty1.service                                    loaded active running G>
  irqbalance.service                                    loaded active running i>
  kdump.service                                         loaded active exited  C>
  kmod-static-nodes.service                             loaded active exited  C>
  libstoragemgmt.service                                loaded active running l>
  lvm2-monitor.service                                  loaded active exited  M>
  NetworkManager-wait-online.service                    loaded active exited  N>
  NetworkManager.service                                loaded active running N>
  
mr8356@mr8356:~$ sudo systemctl list-unit-files
UNIT FILE                                                                 STATE>
proc-sys-fs-binfmt_misc.automount                                         stati>
-.mount                                                                   gener>
boot-efi.mount                                                            gener>
boot.mount                                                                gener>
dev-hugepages.mount                                                       stati>
dev-mqueue.mount                                                          stati>
proc-sys-fs-binfmt_misc.mount                                             disab>
sys-fs-fuse-connections.mount                                             stati>
sys-kernel-config.mount                                                   stati>
sys-kernel-debug.mount                                                    stati>
sys-kernel-tracing.mount                                                  stati>
tmp.mount                                                                 disab>
lvm-devices-import.path                                                   disab>
systemd-ask-password-console.path                                         stati>
systemd-ask-password-plymouth.path

mr8356@mr8356:~$ sudo systemctl list-units --type=socket
  UNIT                            LOAD   ACTIVE SUB       DESCRIPTION          >
  dbus.socket                     loaded active running   D-Bus System Message >
  dm-event.socket                 loaded active listening Device-mapper event d>
  iscsid.socket                   loaded active listening Open-iSCSI iscsid Soc>
  iscsiuio.socket                 loaded active listening Open-iSCSI iscsiuio S>
  lvm2-lvmpolld.socket            loaded active listening LVM2 poll daemon sock>
  pcscd.socket                    loaded active listening PC/SC Smart Card Daem>
  sshd-unix-local.socket          loaded active listening OpenSSH Server Socket>
  sshd-vsock.socket               loaded active listening OpenSSH Server Socket>
  sssd-kcm.socket                 loaded active listening SSSD Kerberos Cache M>
  systemd-bootctl.socket          loaded active listening Boot Entries Service >
  systemd-coredump.socket         loaded active listening Process Core Dump Soc>
  systemd-creds.socket            loaded active listening Credential Encryption>
  systemd-hostnamed.socket        loaded active listening Hostname Service Sock>
  systemd-initctl.socket          loaded active listening initctl Compatibility>
  systemd-journald-dev-log.socket loaded active running   Journal Socket (/dev/>
  systemd-journald.socket         loaded active running   Journal Sockets
  systemd-rfkill.socket           loaded active listening Load/Save RF Kill Swi>
  systemd-sysext.socket           loaded active listening System Extension Imag>
  systemd-udevd-control.socket    loaded active running   udev Control Socket
  systemd-udevd-kernel.socket     loaded active running   udev Kernel Socket
  systemd-userdbd.socket          loaded active running   User Database Manager>
  
mr8356@mr8356:~$ sudo systemctl list-timers --all
NEXT                            LEFT LAST                           PASSED UNIT                         ACTIVATES
Sat 2026-03-28 22:13:52 KST 3min 29s -                                   - systemd-tmpfiles-clean.timer systemd-tmpfiles-clean.service
Sat 2026-03-28 22:41:58 KST    31min Tue 2026-03-24 23:27:46 KST         - fwupd-refresh.timer          fwupd-refresh.service
Sat 2026-03-28 22:50:25 KST    40min Tue 2026-03-24 21:11:39 KST         - plocate-updatedb.timer       plocate-updatedb.service
Sat 2026-03-28 22:56:13 KST    45min Tue 2026-03-24 21:11:40 KST         - logrotate.timer              logrotate.service
Sat 2026-03-28 23:07:16 KST    56min -                                   - dnf-makecache.timer          dnf-makecache.service
Sun 2026-03-29 01:00:00 KST 2h 49min Sat 2026-03-28 21:59:16 KST 11min ago raid-check.timer             raid-check.service
Sun 2026-03-29 07:00:00 KST       8h -                                   - sysstat-collect.timer        sysstat-collect.service
Mon 2026-03-30 00:00:00 KST 1 day 1h -                                   - sysstat-rotate.timer         sysstat-rotate.service
Mon 2026-03-30 00:07:00 KST 1 day 1h -                                   - sysstat-summary.timer        sysstat-summary.service
Mon 2026-03-30 01:28:21 KST 1 day 3h Tue 2026-03-24 21:21:36 KST         - fstrim.timer                 fstrim.service

```

---

# 3. Unit File의 내부 구조

Unit File은 **INI 형식**의 텍스트 파일입니다. 크게 세 개의 Section으로 구성됩니다: `[Unit]`, `[Service]` (또는 해당 Unit Type), `[Install]`. 각 Section에 **Directive(키=값)** 형태로 설정을 선언합니다.

SysVinit의 Shell Script가 "어떻게(HOW) 실행하라"는 명령형(Imperative) 방식이었다면, systemd Unit File은 "무엇을(WHAT) 원한다"는 선언형(Declarative) 방식입니다. 이 차이가 systemd의 핵심 설계 철학입니다.

## 3-1. [Unit] Section — 이 Unit이 무엇이고, 무엇에 의존하는가

[Unit] Section은 Unit의 **메타데이터**와 **다른 Unit과의 관계(Dependency)**를 정의합니다. 모든 Unit Type에 공통으로 사용됩니다.

**메타데이터 Directive:**

| Directive | 설명 |
| --- | --- |
| `Description=` | 사람이 읽을 수 있는 Unit 설명. `systemctl status`에 표시됩니다. |
| `Documentation=` | 관련 문서 URL. `man:sshd(8)`, `https://...` 형식. |
| `ConditionPathExists=` | 이 경로가 존재해야만 Unit을 시작합니다. |
| `AssertFileIsExecutable=` | 이 파일이 실행 가능해야만 Unit을 시작합니다. |

**Dependency Directive — systemd에서 가장 중요한 개념:**

systemd의 Dependency는 **두 가지 축**으로 나뉩니다: **Ordering(순서)**과 **Requirement(요구)**.

이 둘은 **독립적**입니다. 순서만 정할 수도 있고, 요구만 정할 수도 있고, 둘 다 정할 수도 있습니다.

**Ordering Directive (순서 — "누가 먼저"):**

| Directive | 동작 |
| --- | --- |
| `After=B.service` | B가 시작된 **후에** 이 Unit을 시작합니다. B의 성공/실패는 상관없습니다. |
| `Before=B.service` | 이 Unit이 시작된 **후에** B를 시작합니다. |

`After=`와 `Before=`는 **순서만** 정합니다. B가 실패해도 이 Unit은 시작됩니다. "B 다음에 시작해라"일 뿐, "B가 필요하다"는 뜻이 아닙니다.

**Requirement Directive (요구 — "누가 필요한가"):**

| Directive | 동작 | 사용 시점 |
| --- | --- | --- |
| `Requires=B.service` | B가 **반드시** 필요합니다. B가 실패하면 이 Unit도 중단됩니다. | DB 없이 앱이 무의미한 경우 |
| `Wants=B.service` | B를 **원하지만**, B가 실패해도 이 Unit은 계속 동작합니다. | 로깅 서비스가 죽어도 앱은 돌아가는 경우 |
| `Requisite=B.service` | B가 **이미 실행 중**이어야 합니다. 실행 중이 아니면 즉시 실패. |  |
| `BindsTo=B.service` | Requires보다 강함. B가 중지되면 이 Unit도 **즉시** 중지됩니다. | Container와 Network Namespace 연결 등 |
| `PartOf=B.service` | B가 restart/stop되면 이 Unit도 함께 restart/stop됩니다. | 메인 서비스와 Side-car 관계 |
| `Conflicts=B.service` | B와 동시에 실행 불가. 이 Unit을 시작하면 B가 중지됩니다. | iptables vs firewalld |

**핵심: `Requires=` + `After=`를 함께 써야 하는 이유**

```
# ❌ 잘못된 예 — Requires만 사용
[Unit]
Requires=postgresql.service
# PostgreSQL이 필요하지만, 순서가 없어서 동시에 시작될 수 있음
# → 앱이 DB보다 먼저 시작되어 Connection Refused 발생 가능

# ✅ 올바른 예 — Requires + After 함께 사용
[Unit]
Requires=postgresql.service
After=postgresql.service
# PostgreSQL이 필요하고, PostgreSQL이 시작된 후에 앱을 시작
```

`Requires=`는 "B가 필요하다"만 말하고, **B가 시작되기를 기다리지는 않습니다**. `After=`를 함께 써야 "B가 시작된 후에 나를 시작해라"가 됩니다. 이 구분을 이해하는 것이 systemd Dependency 설계의 핵심입니다.

```python
mr8356@mr8356:~$ sudo systemctl cat sshd.service
[sudo] password for mr8356:
# /usr/lib/systemd/system/sshd.service
[Unit]
Description=OpenSSH server daemon
Documentation=man:sshd(8) man:sshd_config(5)
After=network.target sshd-keygen.target
Wants=sshd-keygen.target
# Migration for Fedora 38 change to remove group ownership for standard host ke>
# See https://fedoraproject.org/wiki/Changes/SSHKeySignSuidBit
Wants=ssh-host-keys-migration.service

[Service]
Type=notify
EnvironmentFile=-/etc/sysconfig/sshd
ExecStart=/usr/sbin/sshd -D $OPTIONS
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target

# 컨테이너 실습을 위해서 설치했던 podman
mr8356@mr8356:~$ sudo systemctl cat podman.service
# /usr/lib/systemd/system/podman.service
[Unit]
Description=Podman API Service
Requires=podman.socket
After=podman.socket
Documentation=man:podman-system-service(1)
StartLimitIntervalSec=0

[Service]
Delegate=true
Type=exec
KillMode=process
Environment=LOGGING="--log-level=info"
ExecStart=/usr/bin/podman $LOGGING system service

[Install]
WantedBy=default.target

# Podman은 도커와 달리 항상 떠 있는 '데몬'이 없습니다. 대신 외부(CLI나 다른 앱)에서 "컨테이너 좀 보여줘"라고 요청을 보낼 입구(Socket)를 미리 열어둡니다.
mr8356@mr8356:~$ sudo systemctl cat podman.socket
# /usr/lib/systemd/system/podman.socket
[Unit]
Description=Podman API Socket
Documentation=man:podman-system-service(1)

[Socket]
ListenStream=%t/podman/podman.sock
SocketMode=0660

[Install]
WantedBy=sockets.target

mr8356@mr8356:~$ sudo systemctl start podman.service

# require 덕분에 같이 켜진 모습
mr8356@mr8356:~$ sudo systemctl status podman.socket
● podman.socket - Podman API Socket
     Loaded: loaded (/usr/lib/systemd/system/podman.socket; disabled; preset: d>
     Active: active (running) since Sat 2026-03-28 22:27:42 KST; 3s ago
 Invocation: 2242e30b5bd143d284b7bb4bf6a4174a
   Triggers: ● podman.service
       Docs: man:podman-system-service(1)
     Listen: /run/podman/podman.sock (Stream)
     CGroup: /system.slice/podman.socket

Mar 28 22:27:42 mr8356 systemd[1]: Listening on podman.socket - Podman API Sock

mr8356@mr8356:~$ sudo systemctl show podman.socket -p Listen
Listen=/run/podman/podman.sock (Stream)

mr8356@mr8356:~$ sudo ls -l /run/podman/podman.sock
srw-rw----. 1 root root 0 Mar 28 22:27 /run/podman/podman.sock
```

## 3-2. [Service] Section — 어떻게 실행할 것인가

[Service] Section은 `.service` Unit에만 존재하며, **Process의 실행 방식**을 정의합니다.

**Type= - Service의 실행 모델**

`Type=`은 systemd가 "이 Service가 준비 완료되었다"를 어떻게 판단하는지를 결정합니다. 이것을 잘못 설정하면 Service가 시작되었는데 systemd는 아직 시작 중이라고 인식하거나, 그 반대 상황이 발생합니다.

| Type | 동작 | 사용 예시 |
| --- | --- | --- |
| `simple` (기본) | ExecStart로 실행한 Process가 곧 Main Process입니다. 실행 즉시 "준비 완료"로 간주합니다. | Node.js, Python 앱, Go 바이너리 등 Foreground에서 실행되는 프로그램 |
| `forking` | Process가 fork()하고 Parent가 종료되면 "준비 완료"로 간주합니다. Child가 Main Process가 됩니다. | Apache httpd (daemon mode), nginx (daemon mode) 등 전통적인 C Daemon |
| `oneshot` | Process가 완전히 종료된 후 "준비 완료"로 간주합니다. 주로 `RemainAfterExit=yes`와 함께 사용합니다. | DB Migration Script, 초기화 작업, iptables 규칙 적용 |
| `notify` | Process가 `sd_notify()`를 호출하여 "준비 완료"를 명시적으로 알립니다. 가장 정확한 방식입니다. | PostgreSQL, systemd-aware 앱 |
| `dbus` | D-Bus 이름을 등록하면 "준비 완료"로 간주합니다. | NetworkManager 등 D-Bus 기반 서비스 |
| `idle` | 다른 모든 Job이 완료된 후 실행합니다. | 부팅 마지막에 실행할 작업 |

**Type을 잘못 설정한 경우의 증상:**

- `Type=simple`인데 실제로는 fork하는 Daemon → systemd가 Main PID를 잃어버리고, Service가 즉시 failed 상태가 됩니다.
- `Type=forking`인데 실제로는 Foreground 앱 → systemd가 fork를 영원히 기다리다가 Timeout됩니다.

**실행 Directive:**

| Directive | 설명 |
| --- | --- |
| `ExecStart=` | Service 시작 명령어. **반드시 절대경로** 사용. 하나의 Service에 하나만 가능 (oneshot 제외). |
| `ExecStartPre=` | ExecStart 전에 실행할 명령어. 여러 개 가능. 준비 작업에 사용. |
| `ExecStartPost=` | ExecStart 성공 후 실행할 명령어. |
| `ExecStop=` | Service 정지 명령어. 미지정 시 SIGTERM → SIGKILL 순서로 Process를 종료합니다. |
| `ExecReload=` | 설정 Reload 명령어. Process를 재시작하지 않고 설정만 다시 읽게 합니다. |
| `ExecStartPre=-/usr/bin/somecheck` | `-` 접두어: 이 명령이 실패해도 Service 시작을 계속합니다. |

**재시작 Directive:**

| Directive | 설명 |
| --- | --- |
| `Restart=` | 재시작 정책. Service가 죽었을 때 어떻게 할 것인가. |
| `RestartSec=` | 재시작 전 대기 시간. 빠른 재시작 루프를 방지. |
| `StartLimitBurst=` | 특정 시간 내 최대 재시작 횟수. |
| `StartLimitIntervalSec=` | 재시작 횟수를 세는 시간 구간. |

**Restart= 옵션 상세:**

| 값 | 동작 |
| --- | --- |
| `no` (기본) | 재시작 안 함 |
| `on-success` | Exit Code 0으로 종료 시에만 재시작 |
| `on-failure` | 비정상 종료(non-zero exit, signal, timeout) 시 재시작 |
| `on-abnormal` | Signal이나 Timeout으로 죽은 경우에만 재시작 |
| `on-abort` | 잡히지 않은 Signal(SIGSEGV 등)로 죽은 경우만 |
| `always` | 어떤 이유로든 종료되면 항상 재시작 |

> **SRE Best Practice:** Production Service에는 `Restart=on-failure` + `RestartSec=5`가 일반적입니다. `Restart=always`는 정상 종료(`systemctl stop`) 후에도 재시작하므로 주의가 필요합니다. K8s의 kubelet에는 `Restart=always`가 적합합니다 — kubelet이 어떤 이유로든 죽으면 안 되니까.
> 

**보안/환경 Directive:**

| Directive | 설명 |
| --- | --- |
| `User=` / `Group=` | 실행 사용자/그룹. root 대신 전용 사용자 권장. |
| `Environment=` | 환경 변수 직접 설정. `Environment=NODE_ENV=production PORT=3000` |
| `EnvironmentFile=` | 환경 변수 파일에서 로드. `EnvironmentFile=/etc/myapp/env` |
| `WorkingDirectory=` | 작업 디렉터리. |
| `StandardOutput=` | stdout 출력 대상. `journal`(기본), `file:/path`, `null` |
| `StandardError=` | stderr 출력 대상. |
| `ProtectSystem=` | File System 보호. `full`이면 /usr, /boot 읽기전용. |
| `ProtectHome=` | /home 접근 차단. |
| `NoNewPrivileges=` | 실행 중 권한 상승 금지. |
| `LimitNOFILE=` | Open File Descriptor 최대 수. 고성능 서버에서 필수 튜닝. |

> **실무 중요:** `LimitNOFILE=65535`는 고트래픽 웹 서버나 DB에서 거의 필수입니다. 기본값(보통 1024)으로는 동시 연결이 많으면 "Too many open files" 오류가 발생합니다. Nginx, PostgreSQL, Redis 등의 Unit File에서 흔히 볼 수 있습니다.
> 

```python
mr8356@mr8356:~$ sudo systemctl edit podman.service

      /etc/systemd/system/podman.service.d/.#override.conf55c11f6bdba547a1
### Editing /etc/systemd/system/podman.service.d/override.conf
### Anything between here and the comment below will become the contents of the>

### Edits below this comment will be discarded

### /usr/lib/systemd/system/podman.service
# [Unit]
# Description=Podman API Service
# Requires=podman.socket
# After=podman.socket
# Documentation=man:podman-system-service(1)
# StartLimitIntervalSec=0
#
# [Service]
# Delegate=true
# Type=exec
# KillMode=process
# Environment=LOGGING="--log-level=info"
# ExecStart=/usr/bin/podman $LOGGING system service
#
# [Install]
# WantedBy=default.target
```

## 3-3. [Install] Section — 부팅 시 동작

[Install] Section은 `systemctl enable/disable` 명령 실행 시에만 참조됩니다. 런타임에는 영향이 없습니다.

| Directive | 설명 |
| --- | --- |
| `WantedBy=` | `systemctl enable` 시 어느 Target의 `.wants/` 디렉터리에 Symlink를 생성할지 |
| `RequiredBy=` | `WantedBy`보다 강한 연결 |
| `Also=` | 함께 enable/disable할 Unit |
| `Alias=` | Unit의 별칭. 이 이름으로도 시작 가능. |

**`WantedBy=multi-user.target`의 의미:**

`systemctl enable myapp.service`를 실행하면 systemd는 `/etc/systemd/system/multi-user.target.wants/myapp.service` → `/etc/systemd/system/myapp.service`로의 **Symbolic Link**를 생성합니다. 부팅 시 `multi-user.target`에 도달하면 이 `.wants/` 디렉터리의 모든 Unit을 시작합니다.

```bash
# enable 후 Symlink 확인
sudo systemctl enable myapp.service
ls -la /etc/systemd/system/multi-user.target.wants/myapp.service
```

---

# 4. Unit File 위치와 우선순위

| 경로 | 용도 | 우선순위 |
| --- | --- | --- |
| `/usr/lib/systemd/system/` | **RPM 패키지가 설치한** Vendor Unit. `dnf install`로 설치하면 여기에 생성됩니다. | 가장 낮음 |
| `/run/systemd/system/` | **Runtime에 동적 생성**된 Unit. 재부팅 시 사라집니다. | 중간 |
| `/etc/systemd/system/` | **관리자가 직접 생성/수정**한 Unit. 사용자 정의 Service는 여기에 만듭니다. | **가장 높음** |

**Override 원리:** 같은 이름의 Unit File이 여러 경로에 존재하면 **우선순위가 높은 경로의 파일이 사용**됩니다. `/etc/systemd/system/sshd.service`가 존재하면 `/usr/lib/systemd/system/sshd.service`는 무시됩니다.

**절대 하지 말아야 할 것:** `/usr/lib/systemd/system/`의 파일을 직접 수정하는 것. `dnf update`로 패키지를 업데이트하면 이 파일이 덮어쓰여져 수정사항이 사라집니다. 대신 **Drop-in Override**를 사용합니다.

---

# 5. Drop-in Override — Vendor Unit을 안전하게 수정하기

패키지가 설치한 Unit File을 수정하고 싶을 때, 원본을 건드리지 않고 **특정 Directive만 덮어쓰는** 방법입니다.

```bash
# systemctl edit 명령 — 자동으로 Drop-in 파일을 생성합니다
sudo systemctl edit sshd.service

# 에디터가 열리면 변경할 Directive만 입력합니다:
# [Service]
# RestartSec=10
# Restart=always

# 저장 후 확인 — Drop-in 파일이 생성됩니다
cat /etc/systemd/system/sshd.service.d/override.conf

# 전체 파일을 편집하려면 (Copy-on-Write 방식):
sudo systemctl edit --full sshd.service
# → /etc/systemd/system/sshd.service로 전체 사본이 생성됩니다

# Reload 후 적용
sudo systemctl daemon-reload
sudo systemctl restart sshd.service

# 결과 확인 — Vendor Unit + Drop-in이 합쳐진 최종 설정
sudo systemctl cat sshd.service
```

**동작 원리:** `systemctl edit`은 `/etc/systemd/system/sshd.service.d/override.conf`를 생성합니다. systemd는 Vendor Unit을 먼저 읽고, 이 Drop-in 파일의 Directive로 **덮어씁니다**. 원본은 그대로 유지되므로 패키지 업데이트에 안전합니다.

> **K8s 연결:** kubelet의 systemd 설정을 수정할 때도 동일한 패턴을 사용합니다. `sudo systemctl edit kubelet.service`로 `--max-pods=250` 같은 kubelet 인자를 추가하거나, Memory Limit을 설정합니다.
> 

---

# 6. 실습: Custom Service Unit 만들기 (처음부터 끝까지)

```bash
# 1. 앱 스크립트 준비
sudo mkdir -p /opt/myapp
sudo tee /opt/myapp/app.sh << 'EOF'
#!/bin/bash
echo "MyApp starting at $(date)"
while true; do
    echo "$(date) - MyApp heartbeat" >> /var/log/myapp.log
    sleep 10
done
EOF
sudo chmod +x /opt/myapp/app.sh

mr8356@mr8356:~$ ll /opt/myapp/app.sh
-rwxr-xr-x. 1 root root 137 Mar 28 22:37 /opt/myapp/app.sh

# 2. Unit File 작성
sudo tee /etc/systemd/system/myapp.service << 'EOF'
[Unit]
Description=My Custom Application
Documentation=https://internal.wiki/myapp
After=network.target
Wants=network.target

[Service]
Type=simple
ExecStart=/opt/myapp/app.sh
ExecStop=/bin/kill -SIGTERM $MAINPID
Restart=on-failure
RestartSec=5
User=nobody
Group=nobody
WorkingDirectory=/opt/myapp
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

# Security Hardening
ProtectSystem=full
NoNewPrivileges=true

# Resource Control (cgroup)
MemoryMax=256M
CPUQuota=50%

[Install]
WantedBy=multi-user.target
EOF

mr8356@mr8356:~$ ll /etc/systemd/system/myapp.service
-rw-r--r--. 1 root root 523 Mar 28 22:38 /etc/systemd/system/myapp.service

# 3. systemd에 새 Unit File 인식
sudo systemctl daemon-reload

# 4. 시작 및 확인
sudo systemctl start myapp.service
sudo systemctl status myapp.service

● myapp.service - My Custom Application
     Loaded: loaded (/etc/systemd/system/myapp.service; disabled; preset: disab>
     Active: active (running) since Sat 2026-03-28 22:38:47 KST; 11s ago
 Invocation: 5746a6f1ad534cd49b42dd31a3c8ba8a
       Docs: https://internal.wiki/myapp
   Main PID: 2487 (app.sh)
      Tasks: 2 (limit: 22577)
     Memory: 652K (max: 256M, available: 255.3M, peak: 2.9M)
        CPU: 26ms
     CGroup: /system.slice/myapp.service
             ├─2487 /bin/bash /opt/myapp/app.sh
             └─2492 sleep 10

Mar 28 22:38:47 mr8356 systemd[1]: Started myapp.service - My Custom Applicatio>
Mar 28 22:38:47 mr8356 myapp[2487]: MyApp starting at Sat Mar 28 10:38:47 PM KS>
Mar 28 22:38:47 mr8356 myapp[2487]: /opt/myapp/app.sh: line 4: /var/log/myapp.l>
Mar 28 22:38:57 mr8356 myapp[2487]: /opt/myapp/app.sh: line 4: /var/log/myapp.l>

# 5. 부팅 시 자동 시작 등록
sudo systemctl enable myapp.service

mr8356@mr8356:~$ sudo systemctl enable myapp.service
Created symlink '/etc/systemd/system/multi-user.target.wants/myapp.service' → '/etc/systemd/system/myapp.service'.

mr8356@mr8356:~$ ll /etc/systemd/system/multi-user.target.wants/
total 0
lrwxrwxrwx. 1 root root 35 Mar 10 00:25 atd.service -> /usr/lib/systemd/system/atd.service
lrwxrwxrwx. 1 root root 38 Mar 10 00:25 auditd.service -> /usr/lib/systemd/system/auditd.service
lrwxrwxrwx. 1 root root 43 Mar 10 00:25 audit-rules.service -> /usr/lib/systemd/system/audit-rules.service
lrwxrwxrwx. 1 root root 39 Mar 10 00:25 chronyd.service -> /usr/lib/systemd/system/chronyd.service
lrwxrwxrwx. 1 root root 37 Mar 10 00:25 crond.service -> /usr/lib/systemd/system/crond.service
lrwxrwxrwx. 1 root root 41 Mar 10 00:25 firewalld.service -> /usr/lib/systemd/system/firewalld.service
lrwxrwxrwx. 1 root root 42 Mar 10 00:25 irqbalance.service -> /usr/lib/systemd/system/irqbalance.service
lrwxrwxrwx. 1 root root 37 Mar 10 00:25 kdump.service -> /usr/lib/systemd/system/kdump.service
lrwxrwxrwx. 1 root root 46 Mar 10 00:25 libstoragemgmt.service -> /usr/lib/systemd/system/libstoragemgmt.service
lrwxrwxrwx. 1 root root 41 Mar 10 00:25 mdmonitor.service -> /usr/lib/systemd/system/mdmonitor.service
# inode symlink 생성됐다.
lrwxrwxrwx. 1 root root 33 Mar 28 22:39 myapp.service -> /etc/systemd/system/myapp.service
lrwxrwxrwx. 1 root root 46 Mar 10 00:25 NetworkManager.service -> /usr/lib/systemd/system/NetworkManager.service
lrwxrwxrwx. 1 root root 48 Mar 10 00:25 remote-cryptsetup.target -> /usr/lib/systemd/system/remote-cryptsetup.target
lrwxrwxrwx. 1 root root 40 Mar 10 00:25 remote-fs.target -> /usr/lib/systemd/system/remote-fs.target
lrwxrwxrwx. 1 root root 39 Mar 10 00:25 rsyslog.service -> /usr/lib/systemd/system/rsyslog.service
lrwxrwxrwx. 1 root root 38 Mar 10 00:25 smartd.service -> /usr/lib/systemd/system/smartd.service
lrwxrwxrwx. 1 root root 36 Mar 10 00:25 sshd.service -> /usr/lib/systemd/system/sshd.service
lrwxrwxrwx. 1 root root 36 Mar 10 00:25 sssd.service -> /usr/lib/systemd/system/sssd.service
lrwxrwxrwx. 1 root root 39 Mar 24 21:05 sysstat.service -> /usr/lib/systemd/system/sysstat.service
lrwxrwxrwx. 1 root root 37 Mar 10 00:25 tuned.service -> /usr/lib/systemd/system/tuned.service

# 6. 로그 실시간 확인
sudo journalctl -u myapp.service -f
mr8356@mr8356:~$ sudo journalctl -u myapp.service -f
Mar 28 22:38:47 mr8356 systemd[1]: [🡕] /etc/systemd/system/myapp.service:13: Special user nobody configured, this is not safe!
Mar 28 22:38:47 mr8356 systemd[1]: Started myapp.service - My Custom Application.
Mar 28 22:38:47 mr8356 myapp[2487]: MyApp starting at Sat Mar 28 10:38:47 PM KST 2026
Mar 28 22:38:47 mr8356 myapp[2487]: /opt/myapp/app.sh: line 4: /var/log/myapp.log: Permission denied
Mar 28 22:38:57 mr8356 myapp[2487]: /opt/myapp/app.sh: line 4: /var/log/myapp.log: Permission denied

# 7. 제대로 만들어졌는지 전체 검증
sudo systemctl cat myapp.service
sudo systemctl show myapp.service | grep -E 'Restart|Memory|CPU'
```

---

# 7. systemctl 명령어 완전 정리

## Service Lifecycle

```bash
sudo systemctl start myapp.service      # 시작
sudo systemctl stop myapp.service       # 정지
sudo systemctl restart myapp.service    # 재시작 (stop → start)
sudo systemctl reload myapp.service     # 설정만 Reload (Process 재시작 없음)
sudo systemctl reload-or-restart myapp.service  # Reload 가능하면 Reload, 아니면 Restart

sudo systemctl status myapp.service     # 상태 확인 (Active, PID, 최근 로그)
sudo systemctl is-active myapp.service  # active/inactive만 출력 (스크립트용)
sudo systemctl is-failed myapp.service  # failed 여부만 출력
```

## Enable / Disable (부팅 시 자동 시작)

```bash
sudo systemctl enable myapp.service          # 부팅 시 자동 시작 등록
sudo systemctl enable --now myapp.service    # 등록 + 즉시 시작 (한 번에)
sudo systemctl disable myapp.service         # 자동 시작 해제
sudo systemctl is-enabled myapp.service      # enabled/disabled 확인

# mask — 완전히 시작 불가능하게 차단
sudo systemctl mask myapp.service     # /dev/null로 Symlink → start 자체가 불가능
sudo systemctl unmask myapp.service   # mask 해제

mr8356@mr8356:~$ sudo systemctl disable myapp.service
Removed '/etc/systemd/system/multi-user.target.wants/myapp.service'.

mr8356@mr8356:~$ sudo systemctl stop myapp.service
mr8356@mr8356:~$

```

`disable`은 자동 시작만 해제합니다. 수동 `start`는 여전히 가능합니다. `mask`는 `/dev/null`로 Symlink하여 어떤 방법으로도 시작할 수 없게 완전 차단합니다. 충돌하는 Service(예: iptables vs firewalld)를 확실히 막을 때 사용합니다.

## Target (부팅 모드)

Target은 SysVinit의 Runlevel을 대체합니다. 여러 Unit을 묶어 "이 상태에 도달해라"를 정의합니다.

| Target | 구 Runlevel | 설명 |
| --- | --- | --- |
| `rescue.target` | 1 | 단일 사용자 모드. root만 로그인. |
| `multi-user.target` | 3 | CLI 다중 사용자. **서버의 표준.** |
| `graphical.target` | 5 | GUI 포함. 데스크탑용. |
| `emergency.target` | - | 최소 Shell. File System도 Mount 안 됨. |
| `reboot.target` | 6 | 재부팅. |
| `poweroff.target` | 0 | 종료. |

```bash
sudo systemctl get-default                      # 현재 기본 Target 확인
sudo systemctl set-default multi-user.target     # 기본 Target을 CLI로 변경
sudo systemctl isolate rescue.target             # 즉시 rescue 모드 전환 (재부팅 없이)
```

---

# 8. Dependency 시각화와 부팅 분석

```bash
# 특정 Service의 Dependency Tree
sudo systemctl list-dependencies sshd.service

# 역방향 — 누가 이 Service를 필요로 하는가
sudo systemctl list-dependencies sshd.service --reverse

# 부팅 시간 전체 분석
sudo systemd-analyze time

# 부팅 시 각 Service별 소요 시간 (오래 걸리는 순)
sudo systemd-analyze blame

# 특정 Service의 Critical Chain (병목 경로)
sudo systemd-analyze critical-chain myapp.service
```

> **SRE 실무:** `systemd-analyze blame`은 EC2 Instance의 부팅 시간을 최적화할 때 핵심 도구입니다. Auto Scaling Group에서 Scale-out 시 Instance가 Service에 투입되기까지의 시간을 줄이려면, 부팅 시 불필요한 Service를 disable하고, 오래 걸리는 Service의 원인을 분석해야 합니다.
> 

---

# 9. journald — systemd의 통합 Logging

systemd는 자체 Logging System인 **journald**를 포함합니다. 전통적인 syslog와 달리 **구조화된 Binary 로그**를 저장하며, 강력한 필터링과 메타데이터를 지원합니다.

```bash
# 특정 Service 로그
sudo journalctl -u myapp.service

# 실시간 로그 (tail -f와 동일)
sudo journalctl -u myapp.service -f

# 최근 N줄만
sudo journalctl -u myapp.service -n 50

# 시간 범위 지정
sudo journalctl -u myapp.service --since "2026-03-26 10:00:00" --until "2026-03-26 12:00:00"
sudo journalctl -u myapp.service --since "1 hour ago"
sudo journalctl -u myapp.service --since today

# Priority 필터링 (emerg, alert, crit, err, warning, notice, info, debug)
sudo journalctl -u myapp.service -p err     # err 이상만
sudo journalctl -u myapp.service -p warning  # warning 이상만

# Kernel 로그 (dmesg 대체)
sudo journalctl -k

# 이전 부팅의 로그
sudo journalctl -b -1     # 직전 부팅
sudo journalctl --list-boots  # 부팅 이력

# JSON 출력 (스크립트 파싱용)
sudo journalctl -u myapp.service -o json-pretty -n 5

# 디스크 사용량 확인 및 정리
sudo journalctl --disk-usage
sudo journalctl --vacuum-size=500M   # 500MB로 제한
sudo journalctl --vacuum-time=7d     # 7일 이전 삭제
```

> **AWS CloudWatch 연결:** EC2에서는 **CloudWatch Agent**가 journald에서 직접 로그를 수집하여 CloudWatch Logs로 전송할 수 있습니다. `/etc/amazon/amazon-cloudwatch-agent.toml`에서 journald source를 설정합니다. K8s에서는 kubelet의 로그가 journald에 쓰여지고, **Fluent Bit**이 이를 수집하여 CloudWatch/Elasticsearch로 전송하는 패턴이 표준입니다.
> 

---

# 10. systemd Timer — cron의 현대적 대안

systemd Timer는 `.timer` Unit과 `.service` Unit의 쌍으로 동작합니다. Timer가 트리거되면 같은 이름의 Service를 실행합니다.

**cron 대비 장점:**

- journald와 자동 통합 (별도 로그 리디렉션 불필요)
- Dependency 설정 가능 (`After=`, `Requires=`)
- cgroup Resource 제한 가능 (Timer로 실행된 Job에도 MemoryMax 등 적용)
- `Persistent=true`로 누락된 실행을 부팅 시 보상 실행
- `RandomizedDelaySec=`으로 여러 서버의 동시 실행을 분산

```bash
# 1. 실행할 Service Unit
sudo tee /etc/systemd/system/backup.service << 'EOF'
[Unit]
Description=Daily Backup Job

[Service]
Type=oneshot
ExecStart=/opt/scripts/backup.sh
User=root
MemoryMax=1G
EOF

# 2. Timer Unit
sudo tee /etc/systemd/system/backup.timer << 'EOF'
[Unit]
Description=Run Backup Daily at 3AM

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
EOF

# 3. 활성화
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer

# 4. 확인
sudo systemctl list-timers --all
sudo systemctl status backup.timer
```

**OnCalendar 문법:**

| 표현 | 의미 |
| --- | --- |
| `*-*-* 03:00:00` | 매일 새벽 3시 |
| `Mon *-*-* 09:00:00` | 매주 월요일 오전 9시 |
| `*-*-01 00:00:00` | 매월 1일 자정 |
| `*-01,07-01 00:00:00` | 매년 1월 1일, 7월 1일 자정 |
| `minutely` / `hourly` / `daily` / `weekly` | 축약형 |

```bash
# OnCalendar 표현식 검증 도구
systemd-analyze calendar "*-*-* 03:00:00"
systemd-analyze calendar "Mon *-*-* 09:00:00"
```