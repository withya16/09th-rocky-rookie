# 서비스 관리와 프로세스 제어 정리
리눅스는 단순히 프로그램만 실행하는 운영체제가 아니라,  
부팅 이후 수많은 서비스와 프로세스를 계속 관리해야 함
(SSH 서비스, 로그 서비스, 예약 작업, 시스템 모니터링 프로세스 등)

1. **systemd와 systemctl, 서비스 관리**
2. **프로세스 조회와 제어**
3. **nice / renice와 우선순위**
4. **작업 스케줄링 cron / at**
5. **cgroup 기반 자원 제어 기초**

---

## 1. systemd와 systemctl

### 1-1. systemd

`systemd`는 현대 리눅스에서  
**시스템 초기화(init)와 서비스를 관리하는 핵심 데몬(daemon)** 이다.

`systemd` : **시스템을 관리하는 백그라운드 데몬** 정도로 이해하면 된다. (`d` = daemon)

여기서 daemon은  
사용자 화면에 직접 보이지 않고, 뒤에서 계속 동작하는 관리 프로그램이다.
- 예전 유닉스 계열에서는 서비스 이름 끝에 `d`가 많이 붙었다. (sshd, httpd, crond)

---

### 1-2. systemctl

`systemctl`은 `systemd`를 제어하는 명령어다.

**`systemctl` : systemd를 제어하는 명령** 이라고 보면 된다. (`ctl` = control)

관계로 보면:
- `systemd` = 관리자
- `systemctl` = 관리자에게 명령하는 도구

⸻

### 1-3. 서비스와 프로세스의 차이

서비스 : systemd가 관리하는 기능 단위
- 즉, 시스템 차원에서 이 기능을 켜고 끄고 자동 실행할지 말지를 관리하는 단위
예: sshd.service / crond.service 등

프로세스 : 실행 중인 프로그램 그 자체
- 즉 CPU와 메모리를 사용하며 실제로 돌아가는 실행 단위

서비스 = 식당의 영업 기능 [관리 단위] // 하나 이상의 프로세스로 구성 가능
프로세스 = 실제로 일하는 직원 [실행 단위]

⸻

### 1-4. 서비스 관리의 핵심 명령어

상태 확인
```bash
sudo systemctl status <유닛명>
// <유닛명> 대신 <PID> 가능
```

시작
```bash
sudo systemctl start <유닛명>
```

중지
```bash
sudo systemctl stop <유닛명>
```

재시작
```bash
sudo systemctl restart <유닛명>
```

설정만 다시 읽기
```bash
sudo systemctl reload <유닛명>
```

자동 실행 설정
```bash
sudo systemctl enable <유닛명>
```

자동 실행 해제
```bash
sudo systemctl disable <유닛명>
```

⸻

### 1-5. start와 enable 차이

- start = 지금 당장 실행 [지금 켜라]
- enable = 부팅할 때 자동 실행 [다음 부팅 때부터는 자동으로 켜져라]

아래 명령어는 "자동 실행도 켜고", "지금 시작하라"는 의미
```bash
sudo systemctl enable --now <유닛명>
```

⸻

## 2. 프로그램과 프로세스

### 2-1. 프로그램 & 프로세스

프로그램 : 디스크에 저장된 실행 파일로 아직 실행되지 않은 코드 (예. /usr/bin/bash 등)
프로세스 : 프로그램이 메모리에 올라와 실제로 실행 중인 상태

- 프로그램 = 정적 (static)
- 프로세스 = 동적 (dynamic)

⸻

### 2-2. 한 프로그램이 여러 프로세스가 될 수 있나

그렇다.

예:

sleep 300 &
sleep 300 &
sleep 300 &

같은 프로그램을 여러 번 실행하면 **서로 다른 PID**를 가진 여러 프로세스 생성!

⸻

## 3. 프로세스 조회: ps와 top

### 3-1. ps란

ps : 프로세스 상태를 조회하는 명령어 (=process status의 줄임말)

⸻

### 3-2. ps -ef

전통적인 UNIX(System V) 스타일
-e = 모든 프로세스
-f = 상세 형식(full format)

-> 시스템의 모든 프로세스를 자세히 보여줘!

```bash
ps -ef
ps -ef | grep sshd
```

ps -ef는 UID / PID / PPID / CMD 등과 같은 정보를 보기 좋고
특히 부모-자식 관계와 구조를 보기 좋음

⸻

### 3-3. ps aux

BSD 스타일 옵션
- a = 다른 사용자 프로세스 포함
- u = 사용자 중심 형식
- x = 터미널 없는 프로세스 포함

-> 사용자 친화적 형식으로, 넓게 전체 프로세스를 보여줘

```bash
ps aux
```

이건 %CPU, %MEM 등이 나와서 자원 사용량 중심으로 보기 좋음

⸻

### 3-4. ps -ef vs ps aux

ps -ef
	•	UNIX 스타일
	•	구조/계보 확인에 유리

ps aux
	•	BSD 스타일
	•	CPU/메모리 사용량 확인에 유리

즉,
	•	ps -ef = 조직도
	•	ps aux = 건강검진표

⸻

### 3-5. top이란

top : 상위(top) 프로세스들을 실시간으로 보여주는 명령어
즉, 아래 정보 등을 계속 갱신해서 보여주는 실시간 모니터
	•	CPU 많이 쓰는 프로세스
	•	메모리 많이 쓰는 프로세스
	•	시스템 상태

ps = 사진 VS top = 생중계

// 종료는 q

⸻

## 4. 프로세스 종료와 제어: kill

kill : 프로세스를 끝내거나 신호(signal)를 보내는 명령어

보통 정상 종료 요청
```bash
kill PID
// = 가능하면 정상적으로 종료해
```

강제 종료
```bash
kill -9 PID
// = 무조건 즉시 죽어 ㅋ
```

⸻

## 5. nice와 renice

### 5-1. 왜 필요한가

CPU는 한정된 자원이라
프로세스들이 서로 CPU를 쓰려고 경쟁한다.

nice는 프로세스의 CPU 스케줄링 우선순위 성향에 영향을 주는 값
-> 쉽게 말하면, 이 프로세스가 CPU를 얼마나 양보할지를 나타내는 값

⸻

### 5-2. nice

값 범위 : -20 ~ 19 (기본값 : 0) [숫자가 클수록 우선순위 낮음]
-20 = 우선순위 매우 높음
19 = 우선순위 매우 낮음

⸻

### 5-3. nice와 renice 차이

nice : 프로세스를 처음 실행할 때 nice 값을 정함 [시작 시 지정]
```bash
nice -n 10 sleep 300
```

renice : 이미 실행 중인 프로세스의 nice 값을 바꿈 [실행 중 변경]
```bash
renice 5 -p PID
```

⸻

### 5-4. nice는 절대 명령인가?

ㄴㄴ 아님 !!!

nice와 renice는 CPU 자체에 직접 명령하는 것이 아니라,
CPU 스케줄러가 우선순위를 판단할 때 참고하는 힌트에 가깝다.

즉 스케줄러에게:
	•	이 프로세스는 좀 더 양보하게
	•	이 프로세스는 조금 더 우대하게

라고 요청? 고려해주세요~ 하고 부탁하는 느낌

하지만 CPU 스케줄링은 다른 경쟁 프로세스, I/O 상태, 시스템 부하, cgroup 정책
등도 함께 보므로 절대 보장은 아니다!!!

-> 영향은 있지만 100% 강제 명령은 아니다

⸻

## 6. 작업 스케줄링 : cron과 at
 
**지금 당장 실행하지 않고, 특정 시점 또는 주기적으로 작업을 자동 실행하게 하는 기능**

- `cron` : 반복 실행 작업 [정기 알람]
- `at` : 1회성 예약 작업 [1회 약속]

---

### 6-1. cron

`cron`은 **반복 실행 작업**을 담당한다.

예:
- 매일 새벽 2시 백업
- 매 5분마다 로그 기록
- 매주 월요일 정리 작업

서비스는 **`crond`** 가 담당한다.

상태 확인:
```bash
sudo systemctl status crond
```


사용자 크론 설정:
```bash
crontab -e
```

```bash
*/5 * * * * date >> /home/dongseob/cron_test.log
```
의미:
	•	*/5 = 5분마다
	•	* = 매 시
	•	* = 매일
	•	* = 매월
	•	* = 매 요일

-> 5분마다 현재 시간을 /home/dongseob/cron_test.log 파일에 추가 기록하라는 뜻

확인:
```bash
crontab -l
```

⸻

cron 시간 필드 구조
cron은 보통 아래 5개 필드로 시간을 지정한다.

분 시 일 월 요일 명령어

예:

0 2 * * * /home/dongseob/backup.sh

의미:
	•	매일
	•	새벽 2시에
	•	/home/dongseob/backup.sh 실행

⸻

### 6-2. 왜 터미널에서는 되는데 cron에서는 안 될까?

-> cron의 실행 환경이 평소 터미널 환경과 다르기 때문!

터미널에서는 보통:
- PATH가 충분히 설정되어 있고
- 현재 작업 디렉터리가 있고
- 여러 환경변수가 잡혀 있다

그런데 cron은 매우 제한된 환경에서 실행된다.

그래서 이런 문제가 자주 생긴다.
- python은 터미널에서는 되는데 cron에서는 안 됨
- backup.sh는 수동 실행은 되는데 cron에서는 안 됨
- 상대경로가 안 먹힘

⸻

해결: 명령어와 파일 경로 둘 다 절대경로로 작성
```bash
/usr/bin/python3 /home/user/script.py
```

⸻

+ 로그를 파일로 남기기
예:

*/5 * * * * /home/user/test.sh >> /home/user/test.log 2>&1

의미:
	•	표준 출력(stdout)을 test.log에 추가 기록
	•	표준 에러(stderr)도 같은 파일로 보냄

이렇게 해야 cron 작업이 실패했을 때 왜 실패했는지 추적하기 쉽다.

⸻

### 6-3. 쉘 스크립트는 실행 권한이 있어야 한다

cron에서 쉘 스크립트를 실행하려면
그 스크립트 파일에 실행 권한(x) 이 있어야 한다.
```bash
chmod u+x /home/user/backup.sh
```

⸻

### 6-4. 출력이 없다고 안 돌아간 게 아닐 수 있다

cron은 터미널에서 바로 눈에 보이게 실행되는 방식이 아니다.
하지만 실제로는 백그라운드에서 정상 실행됐을 수도 있다.

그래서 테스트할 때는 다음처럼 하는 것이 좋다.
- 결과를 파일에 남기기
- 로그 파일 확인하기
- date, echo 같은 간단한 명령으로 먼저 테스트하기

```bash
*/5 * * * * date >> /home/dongseob/cron_test.log 2>&1
```

확인:
```bash
cat /home/dongseob/cron_test.log
```

⸻

### 6-5. cron 서비스가 꺼져 있으면?

이것도 매우 중요하다.

cron 작업을 등록해놨더라도,
crond 서비스가 실행 중이 아니면 cron 작업은 수행되지 않는다.

확인:
```bash
systemctl status <유닛명>
```

출력에서 확인할 것:
- loaded
- enabled [부팅 후 자동 시작 설정됨]
- active (running) [현재 실행 중]

⸻

### 6-6. at

at은 1회성 예약 작업이다.
ex. 2분 뒤 파일 생성 / 오늘 밤 11시에 한 번 실행 / 특정 시각에 로그 메시지 기록

=>
cron = 반복 실행
at = 한 번만 실행

서비스는 **`atd`**가 담당한다.

⸻

서비스 시작:
```bash
sudo systemctl enable --now <유닛명>
```

상태 확인:
```bash
sudo systemctl status <유닛명>
```

at 예약하기: (ex. 지금부터 2분 뒤에 아래 명령을 단 1번만 실행)
```bash
at now + 2 minutes

//프롬프트가 뜨면 명령 입력:
echo "hello at" >> /home/dongseob/at_test.log
// 입력 종료 : Ctrl + D
// 
```

at 예약 목록 확인: 현재 예약된 at 작업 목록을 보여줌(작업 번호 / 예약 시각 / 사용자 등)
```bash
atq
```

at (예약된) 작업 삭제:
```bash
atrm 작업번호
```

⸻

## 6-7. at 서비스가 꺼져 있으면?

at도 마찬가지다.

작업을 예약해도 atd 서비스가 꺼져 있으면 실행되지 않는다.

확인:

systemctl status atd

⸻

## 6-8. at도 출력이 없다고 안 된 게 아닐 수 있다..!

at도 cron과 마찬가지로
터미널에 바로 눈에 띄게 실행되는 방식이 아니다.
-> 그래서 실행 결과를 눈으로 바로 확인하기 어렵다.
-> 테스트할 때, **`결과를 파일에 남기기`**, **`로그 파일 확인하기`**, **`echo, date 같은 간단한 명령으로 먼저 실습하기`**

ex.
```bash
at now + 1 minute
```

입력:
```bash
echo "AT TEST" >> /home/dongseob/at_test.log
```

확인:
```bash
cat /home/dongseob/at_test.log
```

⸻

## 6-9. cron / at 사용 시 주의사항 정리

공통
- 서비스(crond, atd)가 실행 중이어야 함
- 절대경로를 쓰는 습관이 좋음
- 결과를 로그 파일로 남기면 디버깅이 쉬움

cron
- 반복 실행
- 환경이 제한적이므로 절대경로 사용 권장
- 스크립트는 실행 권한 필요
- 상대경로, 환경변수 의존 명령은 실패 가능성 높음

at
- 1회 실행
- 예약 후 atq로 확인 가능
- 필요하면 atrm으로 삭제 가능

⸻

# 8. cgroup 기반 자원 제어

## 8-1. cgroup이란

cgroup : control group의 줄임말 = 제어 그룹
- 리눅스가 프로세스를 그룹 단위로 묶어서 "CPU, 메모리, I/O, 프로세스 수" 등 같은 자원을 측정하고 제한하는 구조

⸻

## 8-2. 핵심 개념

cgroup : 단순 카테고리가 아니라 통제 가능한 프로세스 그룹

- A, B, C 프로세스 → ㄱ cgroup
- D, E, F 프로세스 → ㄴ cgroup
처럼 묶고,
그 그룹 전체에 대해 한 번에 [CPU 제한 / 메모리 제한 / 우선순위 정책 / 프로세스 수 제한] 을 줄 수 있다.

-> 프로세스 집단을 묶어서 한꺼번에 통제하는 구조라고 보면 된다.

⸻

## 8-3. cgroup은 기본적으로 이미 존재하나?

그렇다! 특히 systemd를 쓰는 환경에서는 프로세스들이 보통 어떤 cgroup 트리 어딘가에는 이미 속해 있다.
ex. system.slice / user.slice / machine.slice

-> 프로세스가 완전히 cgroup 바깥에 있는 느낌보다 기본 구조 안에 소속되어 있다고 이해하면 된다.


추가로 cgroup은 기존 그룹에 정책을 추가 가능 / 프로세스를 다른 cgroup으로 이동 가능 / 새로운 관리 단위 만들기 가능
다만 실무에서는 raw cgroup을 직접 만지기보다 보통 systemd를 통해 service / slice / scope 단위로 관리하는 경우가 많다.

⸻

## 8-4. nice와 cgroup 차이

nice
- 개별 프로세스 중심
- CPU 우선순위 힌트
- 절대 제한 아님

cgroup
- 프로세스 그룹 중심
- CPU, 메모리, I/O 등 폭넓게 제어
- 실제 제한 가능

즉:
- nice = 이 프로세스 좀 양보하게
- cgroup = 이 그룹은 CPU 30%, 메모리 1GB까지만
