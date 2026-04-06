# 1. Systemd 구조와 서비스 관리 개념 이해

### Systemd

컴퓨터가 켜진 다음, 리눅스 시스템이 제대로 돌아가도록 각종 백그라운드 서비스들을 정리하고 운영하는 총괄 관리자

> 운영체제가 실제 사용 가능한 상태가 되도록 필요한 것들을 차례대로 세팅하는 것
> 
- 커널과의 차이점
    - systemd: 부팅 이후 서비스와 시스템 상태를 관리하는 관리자, 이 서비스는 CPU/메모리를 이 정도 정책으로 써라 하고 묶는 관리자
    - 커널: CPU, 메모리, 디스크, 프로세스 스케줄링 같은 실제 자원 관리 담당
    - 커널은 하드웨어 쪽 핵심 관리자 / systemd는 사용자 공간 쪽 운영 관리자 (But 시스템 운영에 필요한 서비스 한해서)
- 현재 대부분의 RHEL 계열 포함 리눅스는 systemd를 사용 중
- 시스템이 정상적으로 작동하기 위해서 이런 것들이 필요함
    - SSH 접속 서비스 띄우기
    - 네트워크 관련 서비스 시작하기
    - 웹서버 실행하기
    - cron 같은 예약 작업 데몬(뒤에서 계속 돌아가는 프로그램) 실행하기
    - 어떤 서비스가 죽으면 다시 시작하기
    - 부팅할 때 자동으로 켤지 말지 정하기
- 이 역할들을 하기 위해서 Systemd가 관리
    - 언제 시작할지
    - 무엇보다 먼저/나중에 시작할지
    - 부팅 때 자동 실행할지
    - 죽으면 재시작할지
    - 현재 상태가 정상인지 실패인지

But 너무 추상적이라서 조금 더 직관적으로 이해하고자 컴퓨터가 부팅될때 뭘 먼저 해야하는지 알아봄

### 컴퓨터 프로세스 전체적인 흐름

**“컴퓨터가 아예 꺼진 상태에서 운영체제가 정상 동작 상태가 되기까지”**

- 전원 켜짐
- 펌웨어(BIOS/UEFI)가 기본 점검
- 부트로더가 커널을 메모리에 올림
- 리눅스 커널이 하드웨어와 메모리 같은 기본 환경을 준비
- 그 다음 사용자 공간에서 systemd가 올라와서 나머지 시스템 서비스들을 정리하고 시작

즉, 하드웨어를 직접 처음 깨우는 역할은 커널 쪽이고 

커널이 기본 토대를 만든 뒤 시스템을 실제 운영 가능한 상태로 정리하는 역할이 systemd! 

### Unit

Systemd가 관리하는 대상 하나를 unit으로 봄, 단순 작업 단위가 아니라 업무 및 운영 단위 

보통 `/usr/lib/systemd/system/` 또는 `/etc/systemd/system/` 아래에 존재

각 유닛 파일 안에는 이 서비스가 무엇인지 , 어떤 서비스 뒤에 시작해야 하는지 , 어떤 명령어로 시작할지 ,죽었을 때 재시작할지 , 어떤 사용자 권한으로 실행할지 적어놓은 설정 파일임 

- 웹 서버 실행 → `service`
- 특정 상태까지 부팅 → `target`
- 디스크를 특정 경로에 연결 → `mount`
- 예약 시간에 작업 실행 → `timer`

### Service Unit

하나의 서비스(백그라운드기능)을 관리하는 unit 

- `sshd.service` → SSH 접속 서비스
- `crond.service` → 예약 작업 서비스
- `httpd.service` → 웹 서버 서비스

이러한 서비스에 대해서 `시작할지 중지할지 재시작할지 부팅 시 자동 실행할지 지금 정상인지 실패인지 관리`

- 왜 굳이 따로 관리해야할까 ? 
서버가 제대로 동작하기 위해선 네트워크 관련 서비스나 SSH 접속 서비스 , 로그 서비스 등등 이런 것들이 백그라운드에서 돌아가는데 이걸 각각 하나씩 확인하면서 장애나면 다시 올리는게 비효율적이라서 아예 백그라운드로 돌아가는 필수애들을 서비스 단위로 묶어서 운영하는 것
`systemctl status`에서 서비스가 실행중인지 부팅 시 활성화 되어있는지 마스크 되어있는지까지 확인 가능함

### Target Unit

여러 unit을 묶어서 시스템의 **목표 상태**를 표현하는 unit 

> **이 상태가 되려면 필요한 unit들을 같이 활성화해라**
> 
- `/etc/systemd/system/default.target`이 기본 부팅 대상
- “최소한 여기까지는 시스템이 올라와야 해 . 이 상태가 되려면 이런 서비스들이 필요해”
- 부팅 중 systemd가 `default.target`이 가리키는 target을 초기화하고, 각 target이 특정 수준의 기능을 가지며 다른 unit들을 그룹화함
- 이해하기 위한 예시 : 카페가 영업 상태가 되기 위해선 전등 켜지고 커피머신 켜지고 계산대 프로그램 실행하고 직원이 출근하고 문이 열리는 등 이런 것들이 모두 필요함. 이 기능들을 하나로 묶어서 “영업 가능 상태”가 되는 것. 카페가 영업 가능 상태가 되기 위해 이 상태에 들어가있는 기능들이 활성화되어야함. 즉 Target unit은 어떤 상태까지 도달하기 위해서 미리 필요한 것들을 정의해둔거고 해당 작업을 이 상태로 끌어올리는 개념인 것
- 예시
    - `multi-user.target` : 다중 사용자 시스템 상태
    - `graphical.target` : 그래픽 로그인 화면까지 포함한 상태
    - `rescue.target` : 복구용 상태
    - `emergency.target` : 더 최소한의 긴급 셸 상태

### Service와 Process의 차이

- 서비스 : systemd가 운영하는 단위 (관리 관점)
- 프로세스 : 실제로 CPU가 메모리를 사용하여 실행 중인 프로그램 (실행 관점)

### Systemd의 부팅 흐름

- 시스템 부팅
- systemd가 시작됨
- `default.target` 확인
- 해당 target에 연결된 unit들을 종속 관계에 따라 시작
- 목표 상태에 도달

# 2. 서비스 시작·중지·재시작·자동 실행 설정

### Systemctl

대표적으로 systemd를 다루는 명령어 

위에서 배웠던 서비스들을 이 명령어로 제어하는 것 

- 서비스 시작 / 중지 / 재시작
- 활성화 / 비활성화
- 상태 확인
- 로드된 서비스 나열
- 설치된 unit 파일 나열
- 기본 target 확인 / 변경
- `start`, `stop`, `restart`
→ **지금 이 순간의 실행 상태**를 바꾸는 명령
- `enable`, `disable`
→ **다음 부팅 때 자동 실행할지**를 정하는 명령

예시 

```sql
systemctl list-units --type service
systemctl list-unit-files --type service
systemctl status sshd.service
systemctl is-active sshd.service
systemctl is-enabled sshd.service
systemctl get-default
systemctl set-default multi-user.target
```

- `list-units`는 **현재 로드된/활성 중심**
- `list-unit-files`는 **설치된 unit 파일과 enabled/disabled 상태 중심**

### 실습 정리

- 서비스 상태 확인 `systemctl status`
    
    ```sql
    systemctl status chronyd
    ```
    
    - 결과 형식
        - 어떤 유닛인지: `chronyd.service - NTP client/server`
        - 유닛 파일이 어디에 있는지: `/usr/lib/systemd/system/chronyd.service`
        - 자동 실행 설정 여부: `enabled`
        - 현재 실행 상태: `active (running)`
        - `Loaded:`유닛 파일이 정상적으로 로드되었는지, enable 상태인지 보여줌
        - `Active:`현재 실제로 실행 중인지 보여줌
- 서비스 시작 `systemctl start`
    
    ```sql
    sudo systemctl start chronyd
    sudo systemctl start 서비스이름
    ```
    
    - start와 enable 구분하기 필수
- 서비스 중지 `systemstl stop`
    
    ```sql
    sudo systemctl stop chronyd
    sudo systemctl stop 서비스이름
    ```
    
    - stop 과 disable 구분 필수
- 서비스 재시작 `systemctl restart`
    - 서비스 설정 파일을 수정했거나, 서비스가 꼬였을 때
    
    ```sql
    sudo systemctl restart vsftpd.service
    sudo systemctl restart 서비스이름
    ```
    
- 자동 실행 설정 : `systemctl enable`
    
    부팅할때 서비스가 자동으로 시작되게 하기 
    
    - **`enable` 은 target unit 안에 유닛을 복사해 넣는 것 이 아니라 이 유닛도 target이 올라갈 때 같이 시작해! 라고 연결해 두는 것**
        
        ```sql
        sudo systemctl enable chronyd
        ```
        
    - `/etc/systemd/system/` : `systemctl enable` 명령으로 생성된 유닛 파일이나 링크가 놓이는 위치, **관리자가 만든 설정**이나 **enable할 때 생성되는 심볼릭 링크를 포함**
        
        systemd가 부팅 시 읽는 구조 안에 “이 서비스도 같이 시작해라”는 연결을 만들어줌 그래서 enable을 하면 보통 “심볼릭 링크가 생성되었다”는 메시지를 보여준다 `fstrim.timer` 를 활성화할때 나오는 예시 
        
        ```
        systemctl enable--now fstrim.timer
        Created symlink /etc/systemd/system/timers.target.wants/fstrim.timer → /usr/lib/systemd/system/fstrim.timer
        ```
        
        “부팅 시 해당 target에 도달할 때 이 유닛도 함께 활성화되도록 연결해 두었다”는 의미.
        
- 자동 실행 해제 : `systemctl disable`
    - 다음 부팅부터 자동 시작하지 않게 등록을 제거하는 작업
        
        ```sql
        sudo systemctl disable chronyd
        sudo systemctl disable 서비스이름
        ```
        
- 한번에 실행 + 자동 실행 설정 : `enable --now`
    - `enable` → 다음 부팅부터 자동 시작
    - `-now` → 지금 즉시 시작
        
        ```sql
        sudo systemctl enable --now 서비스이름
        ```
        
- `reload` : 프로세스는 유지한 채 설정만 다시 읽게 시도  ↔ restart (서비스 끊었다가 재시작)
    
    서비스마다 reload를 지원하는 경우도 있고, 설정 반영을 위해 restart가 필요한 경우도 있다
    

# 3. 프로세스 조회 및 제어

### 프로세스

: 실제 실행 중인 프로그램 

- (위에서 이미 언급하긴함) httpd, sshd, chrond 같은 서비스가 systemd에 의해 관리되는 운영 단위라면 프로세스는 그 서비스가 실제로 메모리에 올라가서 CPU를 배정 받으면서 돌아가는 실행 단위
- 프로세스를 봐야하는 이유
    - 서비스가 “정상”으로 떠 있어도 실제 프로세스가 CPU를 너무 많이 먹거나, 멈춰 있거나, 여러 개 비정상적으로 늘어나 있으면 문제가 생길 수 있다
- 어떤 프로그램이 실행 중인지 ,  PID가 무엇인지 , CPU/메모리를 얼마나 쓰는지 , 어떤 상태인지 ,필요하면 종료하거나 우선순위를 조정해야 하는지를 확인하기 위해서 프로세스 모니터링 도구로 `top, ps, vmstat, sar, perf` 사용

### ps

현재 프로세스를 스냅샷처럼 보여주는 명령어 

- `ps : 현재 셸에서 실행 중인 프로세스`
- `ps - ef : 전체 프로세스 확인`
- `ps -C sshd : 특정 프로그램 찾기`
- 결과
    - `UID` : 어떤 사용자가 실행했는지
    - `PID` : 프로세스 ID
    - `PPID` : 부모 프로세스 ID
    - `CMD` : 어떤 명령으로 실행됐는지

### top

실시간으로 계속 변하는 프로세스 상태 보여주는 명령어 

- 확인 가능한 것들
    - 전체 CPU 사용률
    - 메모리 사용량
    - 실행 중인 프로세스 수
    - 각 프로세스의 CPU / 메모리 사용량
- 사용 예시
    - CPU를 너무 많이 먹는 프로세스 찾을 때
    - 어떤 프로세스가 갑자기 늘어났는지 볼 때
    - 시스템이 느릴 때 실시간 원인 확인할 때

**PID 확인 방법** 

- 프로세스를 제어하기 위해서는 PID (Process ID)를 알아야야함
- `ps`, `pgrep`, `top` 을 통해 확인 가능

### kill

프로세스 종료 

프로세스를 제어하는 가장 기본적인 방법은 신호를 보내는 것

- RHEL에서는 서비스 유닛을 멈출 때 `systemctl stop <name>.service`를 쓰고, 유닛 안의 하나 이상의 프로세스를 종료할 때는 `systemctl kill <name>.service --kill-who=PID,... --signal=<signal>` 형식
- 기본 신호 `SIGTERM`
    - 신호 체계
        - `SIGTERM (15)` : 정상 종료 요청
        - `SIGKILL (9)` : 강제 종료
        - `SIGHUP (1)` : 설정 다시 읽기 용도로 쓰는 경우가 많음
        - `SIGSTOP` : 일시 중지
        - `SIGCONT` : 다시 계속 실행
- 프로세스 제어는 “프로세스를 무조건 지운다”기보다 **신호를 보내 종료를 요청하는 방식이다.**
- 강제 종료
    
    ```sql
     kill -9 PID
    ```
    
     `SIGKILL` 신호를 보냄 
    

### 우선순위 조정

`nice`  `renice`  `chrt` `systemd`

기본 정책인 `SCHED_OTHER` 가 CFS(Completely Fair Scheduler)를 사용하며, 스케줄러가 **niceness 값**을 기반으로 동적 우선순위 목록을 만든다. 관리자는 프로세스의 niceness 값을 바꿀 수 있지만, 동적 우선순위 목록 자체를 직접 바꾸는 것은 아니다. 

- **`nice` :** CPU를 얼마나 양보할지에 가까운 값
    
    ```sql
    nice -n 10 tar czf backup.tar.gz /home
    ```
    
- 값이 클수록 우선순위가 낮아짐
- `renice` : 이미 실행 중인 프로세스의 우선순위 변경
    
    ```sql
    renice 10 -p PID
    ```
    
- `chrt` : 프로세스의 스케줄링 정책을 확인, 설정할 수 있음
    
    일반 프로세스 우선순위(nice) 보다 한 단계 더 아래의 스케줄링 정책 자체
    
    - `SCHED_OTHER`
    - `SCHED_FIFO`
    - `SCHED_RR`
- `systemd` 의 서비스 단위로 unit 파일에서 `CPUSchedulingPolicy=` `CPUSchedulingPriority=` 를 설정해서 **부팅 시 서비스 우선순위**를 조정할 수 있다
- `nice` = **같은 일반 줄 안에서 순서 조정**
- `chrt` = **줄 종류 자체를 바꾸는 것**
- 비교 요약
    
    `nice` 와 `renice` 는 일반 스케줄링 정책(`SCHED_OTHER`) 안에서 프로세스의 niceness 값을 조정하여 CPU 우선순위에 영향을 주는 방식이다. 반면 `chrt` 는 프로세스의 스케줄링 정책 자체를 확인하거나 변경하는 도구이며, `SCHED_OTHER`, `SCHED_FIFO`, `SCHED_RR` 같은 정책을 다룬다. systemd에서는 서비스 unit 파일의 `CPUSchedulingPolicy=` 와 `CPUSchedulingPriority=` 지시어를 통해 서비스 시작 시 적용할 스케줄링 정책과 우선순위를 미리 설정할 수 있다. 실시간 정책에서는 우선순위를 1~99 범위로 지정할 수 있다
    

### 서비스 제어

```
systemctlstart httpd
systemctlstop httpd
systemctlrestart httpd
```

→ systemd가 관리하는 단위 기준 제어

### 프로세스 제어

```
ps-ef
top
pgrep httpd
kill PID
renice10-p PID
```

→ 실제 실행 중인 프로그램 기준 제어

- 서비스는 **운영 단위**
- 프로세스는 **실행 단위**

# 4. 작업 스케줄링

### 작업 스케줄링

특정 작업을 원하는 시간이나 주기에 자동으로 실행되게 설정하는 것 

- 예시
    - 매일 새벽 로그 정리
    - 매주 백업
    - 특정 시각에 한 번만 실행할 작업
    - 시스템 부하가 낮을 때 실행할 작업
    - 정해진 주기로 업데이트 검사
- 반복 작업은 `cron`, 1회성 작업은 `at`, 자동화된 주기 작업의 현대적 방식으로는 `systemd timer` 가 사용된다

### cron

반복 작업을 예약하는 서비스 

- `cron` 의 동작 구조
    - `crond` 서비스가 백그라운드에서 동작
    - `crontab` 이나 `/etc/crontab` 같은 예약 설정을 읽음
    - 현재 시간이 예약 조건과 맞으면 등록된 명령 실행
    
    RHEL 계열에서는 `crond` 가 이 역할을 맡는다. Red Hat 문서도 `crond.service` 가 꺼지면 crontab 항목이 실행되지 않는다고 하므로, 실제 예약 실행의 중심이 `crond` 임을 알 수 있다. 또한 Red Hat 기술 노트는 `cronie` 패키지에 **표준 UNIX 데몬 `crond` 와 관련 도구**가 포함된다고 설명한다.
    
- `crontab` : `crontab` 은 **사용자별 cron 작업 목록**이라고 보면 된다.
    
    “어떤 사용자가 어떤 작업을 어떤 시간 규칙으로 실행할지”를 적어두는 표
    
    ```
    crontab-e# 내 cron 작업 편집
    crontab-l# 내 cron 작업 보기
    crontab-r# 내 cron 작업 삭제
    ```
    
- `cron` 시간 형식 : `cron` 은 보통 시간 조건을 **분 / 시 / 일 / 월 / 요일** 형태로 적는다.
    
    예를 들어 “매일 새벽 2시”, “매주 월요일 오전 9시” 같은 규칙을 숫자로 표현한다. 이 형식 자체는 표준적인 `cron` 방식이며, Red Hat의 cron 설명도 반복 작업을 지정된 시간 규칙으로 예약하는 구조를 전제로 한다.
    
    ```
    02 * * * /home/user/backup.sh
    ```
    
    뜻:  **매일 2시 0분에** `/home/user/backup.sh` 실행 
    
    - 분: 0
    - 시: 2
    - 일: 매일
    - 월: 매월
    - 요일: 매요일
- `cron` 의 특징과 한계
    - 장점
        - 반복 작업 자동화에 적합
        - 구조가 단순하고 오래된 표준 방식
        - 짧은 점검 스크립트나 정기 백업에 많이 사용
    - 한계
        - 예약 시각에 시스템이 꺼져 있으면 실행되지 않을 수 있음
        - 서비스 단위 관리나 의존성 관리 면에서는 systemd timer보다 단순함

`cron` 은 **정해진 순간에 시스템이 살아 있어야 한다**는 점이 중요

cron이 동작하려면 `crond` 서비스가 실행 중이어야 함 

### at

특정 시간에 한번만 실행할 작업을 예약하는 명령어 

- 예시
    - 오늘 밤 11시에 한 번만 파일 정리
    - 내일 오전 10시에 한 번만 알림 메일 발송
    - 특정 날짜/시간에 한 번만 재부팅 전 점검 실행
- **`cron` 과 `at` 의 차이**
    - `cron` : 반복 실행
    - `at`: 한 번만 실행
- `atd` 서비스가 처리
    - 사용자가 `at` 으로 작업 등록
    - `atd` 가 예약 목록을 관리
    - 시간이 되면 명령 실행
- 상대 시간도 표현 가능
    
    ```sql
    at now + 1 hour
    at now + 30 minutes
    at tomorrow
    ```
    
- `atq`  : 예약 목록 확인과 삭제
- 오늘 23:00에 `/home/user/cleanup.sh` 를 한 번 실행
    
    ```sql
    at 23:00
    at> /home/user/cleanup.sh
    at> <EOT>
    ```
    
- 실습 예시
    1. crond 상태 확인
    
    ```
    systemctl status crond
    ```
    
    1. crontab 편집
    
    ```
    crontab-e
    ```
    
    1. 예시 작업 추가
    
    ```
    */5 * * * * date >> /tmp/cron_test.log
    ```
    
    1. atd 상태 확인
    
    ```
    systemctl status atd
    ```
    
    1. at 예약
    
    ```
    at now+1 minute
    echo"AT TEST" >> /tmp/at_test.log
    ```
    
    1. 예약 확인
    
    ```
    atq
    ```
    

# 5. cgroup 기반 자원 제어 기초

**How Containerization Works**

- Namespaces & Cgroups: 리눅스 커널 자체에 있는 두 개의 기능 -> 두 기술 덕분에 리눅스에서 native하게 컨테이너가 가능해짐

Namespaces: processes only see their own resources ->isolated 가능 / types include PID, Mount, UTS, IPC, Network, User namespaces **‘얼만큼 보이냐’**

Cgroup(control group): limits how much each process can use /CPU, memory, disk, I/O. etc 사용량 제한

### cgroup

프로세스를 그룹으로 묶어서 자원을 제어하는 리눅스 커널 기능 관련있는 프로세스끼리 묶고 그 묶음에 대해서 CPU, 메모리, I/O, 프로세스 수 같은 자원 사용 정책을 적용하는 방식 But 서로 다른 cgroup끼리는 철저하게 분리됨. 그래서 나중에 이 기술 덕분에 컨테이너 기술이 가능해진 것 

- **“이 서비스 전체는 메모리 500MB까지만 써”**
- **“이 그룹은 CPU를 다른 그룹보다 덜 쓰게 해”**
- **“이 그룹은 프로세스를 100개 이상 못 만들게 해”**

- cgroup 필요한 이유 : 특정 서비스나 프로세스 그룹이 시스템 자원을 독점하지 못하게 하거나, 반대로 어떤 서비스에는 최소한의 자원을 보호해 줄 수 있음
- **systemd unit 트리와 cgroup 계층이 연결되어 있어서** `systemctl` 이나 unit 파일 설정으로 자원 제어 가능 , **`서비스 단위 자원 제어`**
- Cgroup 확인
    
    ```sql
    cat /proc/2467/cgroup
    ```
    
    → 이 프로세스가 어떤 cgroup에 속했는지 확인
    
    ```sql
    ls /sys/fs/cgroup/system.slice/example.service/
    ```
    
    → 그 cgroup에 연결된 controller 파일들 확인
    

### cgroup v2

- 기본적으로 RHEL은 cgroup-v2를 마운트하고 사용함
- **cgroups v1**: 자원 종류마다 계층이 따로따로
- **cgroups v2**: 하나의 통합된 계층에서 관리

### 1) service

특정 서비스 프로세스 묶음

예: `chronyd.service`, `crond.service`, `httpd.service` 같은 것들. RHEL 10 예시에서도 `/system.slice/chronyd.service`, `/system.slice/crond.service` 같은 cgroup 경로가 보인다.

### 2) slice

자원을 나누기 위한 큰 구획

예: `system.slice`, `user.slice`, `machine.slice`.

문서 예시에서는 시스템 서비스는 `system.slice`, 사용자 세션 관련 작업은 `user.slice`, VM/컨테이너는 `machine.slice` 로 나타난다.

### 3) scope

외부에서 시작됐지만 systemd가 추적하는 프로세스 묶음

예: `init.scope`, `session-2.scope`. RHEL 10 문서 예시에서 실제로 이런 unit들이 보인다.

- `slice` = 큰 분류
- `service` = 서비스 묶음
- `scope` = 세션/외부 프로세스 묶음

### 자원 제어

- **cpu**: CPU 시간 배분 조정
- **memory**: 메모리 사용 제한 및 보고
- **io / blkio**: 디스크 I/O 접근 제어
- **pids**: 프로세스 수 제한
- **cpuset**: 어떤 CPU와 NUMA memory node를 쓸지 제한

### 자원 제어 방식

- Weight : **비율로 나눠 갖는 방식**
    
    여러 cgroup이 CPU를 나눠 쓸 때, weight가 큰 쪽이 더 많은 비율을 가져간다. 문서에서는 서브그룹들의 weight 합을 기준으로 자원 비율을 분배한다고 설명하고, `CPUWeight=` 를 이 모델의 예로 든다.
    
    - A 그룹 weight 100
    - B 그룹 weight 200
    
    이면 B가 A보다 CPU를 대체로 더 많이 가져갈 수 있음 
    
- Limit : **최대 얼마까지 쓸 수 있는지 상한을 주는 방식**
    
    문서에서는 cgroup이 설정된 양까지 자원을 소비할 수 있음. `MemoryMax=` 
    
    - `MemoryMax=500M`이면 그 서비스는 메모리를 최대 500MB까지만 쓰도록 제한
- Protection : **최소한 보호해 줄 자원량**
    
    문서에서는 어떤 cgroup에 보호량을 설정하면, 그 사용량이 보호 경계 아래에 있을 때 커널이 경쟁 상황에서 그 cgroup을 쉽게 불리하게 만들지 않도록 하려 한다. `MemoryLow=` 
    
    - `MemoryMax` = 최대 상한
    - `MemoryLow` = 최소한 이 정도는 보호해 주려는 값
