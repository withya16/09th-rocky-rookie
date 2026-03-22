## 1. RHEL이란?
> RHEL(Red Hat Enterprise Linux)은 Red Hat이 제공하는 엔터프라이즈용 리눅스 운영체제이다.
보안과 관리 기능을 내장한 안정적 고성능 Linux 플랫폼으로 소개되며, 물리 서버, 가상화, 프라이빗/퍼블릭 클라우드, 엣지까지 일관된 환경을 제공하는 기반 OS로 설명한다.

### 1-1. RHEL가 사용되는 이유

- 안정성: 기업 환경에서는 보통 최신 기능보단 안정적으로 오래 굴러가는 것을 선호하는데, RHEL은 이런 요구에 맞게 검증된 패키지와 긴 수명 주기를 제공.
- 일관성 및 보안성: 온프레미스, VM, 클라우드, 엣지처럼 배포 위치가 달라도 비슷한 방식의 운영이 가능하며, 보안 업데이트나 패치, 문서 체계가 잘 정리되어 있다.
- 생태계 호환성: 많은 ISV, 하드웨어 벤더, 클라우드 환경이 RHEL 기준으로 호환성과 인증을 맞춘다. 

### 1-2. 그렇다면 RHEL은 어떤 생태계 안에 있을까?
RHEL은 혼자 떨어진 운영체제가 아닌, `Fedora -> CentOS Stream -> RHEL`로 이어지는 리눅스 생태계 속에서 이해하면 편하다.

- Fedora는 RHEL의 upstream community distory이다. 커뮤니티 중심의 비교적 빠르게 기술을 받아들이는 배포판으로, 새 기술과 아이디어가 먼저 실험되고 논의되는 쪽이다.
- CentOS Stream: Red Hat이 공식적으로 RHEL의 upstream 개발 플랫폼이라고 설명한다. 커뮤니티와 파트너, Red Hat 개발자들이 함께 다음 RHEL에 들어갈 변화를 검증하고 기여하는 흐름이다. 즉 Fedora에서 나온 기술이 더 다듬어져 RHEL 직전 단계에서 흘러가는 구간 정도이다.
- RHEL: 위 결과물 중에서 기업 운영에 적합하도록 검증 및 지원, 장기 유지보수가 붙은 최종 엔터프라이즈 배포판

### 1-3. RHEL의 구조를 어떻게 이해하면 좋을까?

1. `Linux 커널`: 운영체제의 핵심으로 하드웨어 자원(CPU, 메모리, 디스크, 네트워크)을 관리하고, 프로세스와 시스템콜의 기반이 된다.
2. `시스템 유틸리티와 기본 사용자 공간`: 쉘, coreutils, systemd, 파일 시스템 도구, 네트워크 도구 같은 것들이 여기에 포함된다.
3. `패키지 및 소프트웨어 관리 체계`: RHEL은 RPM 패키지 형식을 사용하고, 실제 설치/업데이트/삭제는 보통 DNF로 수행한다.

- RPM = 패키지 포맷 / 패키지 정보 확인
- DNF = 패키지 설치, 검색, 삭제를 위한 실사용 도구

## 2. VM 환경 구성 및 실행 확인
이번 실습은 Windows 환경에서 VMware를 설치한 뒤, 가상머신 위에 RHEL을 구동하는 방식으로 진행했다.
- Windows 위에 VMware 설치
- VMware에 RHEL ISO 연결
- 가상머신 생성 후 RHEL 설치
- 설치 완료 후 터미널을 통해 기본 명령어 실습 진행
<p align="center">
<img width="46%" height="295" alt="image" src="https://github.com/user-attachments/assets/cdb7886b-b71d-41f1-abc6-72163d3cea41" />
<img width="46%" height="185" alt="image" src="https://github.com/user-attachments/assets/76f4eb6a-3670-47f3-9223-5e60e009e759" />
</p>

RHEL 설치 후 가장 먼저 시스템 상태 확인을 위해 아래 명령어를 실행했다.
- `cat /etc/redhat-release`: RHEL 버전 확인
- `whoami`: 현재 사용자 확인
- `pwd`: 현재 위치 확인
- `hostname`: 호스트 이름 확인
- `ip a`: 네트워크 확인
- `df -h`: 디스크 용량 확인

## 3. RHEL 주요 디렉터리 구조
RHEL 역시 리눅스 계열 운영체제이기 때문에, 파일과 설정을 역할에 따라 디렉터리별로 구분하여 관리한다.
<img width="582" height="388" alt="image" src="https://github.com/user-attachments/assets/27c30cd9-cb97-4328-a487-ff23cf676d5d" />\

주요 디렉터리의 의미는 다음과 같다.
- `home`: 일반 사용자 홈
- `/root`: root 계정 홈 
- `/etc`: 설정 파일
- `/var`: 로그, 캐시, 가변 데이터
- `/tmp`: 임시 파일
- `/usr`: 프로그램/라이브러리

## 4. 기본 명령어 정리
1주차에서는 파일 시스템을 탐색, 조작하기 위한 가장 기본적인 명령어 수행 실습

<img width="546" height="454" alt="image" src="https://github.com/user-attachments/assets/1b0a25d4-dbdb-4501-97b4-4b25afb6e968" />

`ls` 관련 옵션
- `ls`: 이름만 보기
- `ls -l`: 자세히 보기
- `ls -a`: 숨김 파일까지 보기
- `ls -al`: 숨김 파일 + 자세한 정보 모두 보기

## 5. 실습 내용
### 5-1. 디렉터리와 파일 만들기
먼저 실습용 디렉터리 생성 후, 그 안에서 파일과 디렉터리를 직접 만듦

<img width="562" height="291" alt="image" src="https://github.com/user-attachments/assets/3c1b4e51-e4ce-4d0d-aa60-d04eef4df87c" />

- `mkdir -p ~/lab/week1`: 디렉터리를 한 번에 만들고, 중간 디렉터리 (lab)가 없어도 같이 생성한다.
- `cd ~/lab/week1`: 해당 디렉터리로 이동
- `pwd`: 현재 내가 있는 디렉터리의 전체 경로
- `touch file1.txt file2.txt`: 빈 파일 file1.txt, file2.txt를 만듦
- `mkdir dir1 dir2: dir1, dir2` 디렉터리 두 개를 만듦
- `ls -al`: 현재 디렉터리의 파일과 폴더를 숨김 파일 포함, 상세 정보와 함께 보여줌

### 5-2. 복사, 이동, 삭제
파일을 복사하고 이동한 뒤, 삭제 실습

<img width="510" height="414" alt="image" src="https://github.com/user-attachments/assets/808b269d-b51c-4ca2-abfd-c383b280481e" />

- `cp file1.txt dir1/`: file1.txt를 dir1 디렉터리 안으로 복사한다.
- `mv file2.txt dir2/`: file2.txt를 dir2 디렉터리 안으로 이동한다.
- `ls -R`: 현재 디렉터리와 하위 디렉터리 내용을 재귀적으로 전부 보여준다.
- `rm dir2/file2.txt`: dir2 안의 file2.txt 파일을 삭제
- `rmdir dir2`: 비어 있는 dir2 디렉터리를 삭제

### 5-3. 파일 내용 확인
텍스트 파일 생성 후, 다양한 방식으로 내용을 확인

<img width="502" height="204" alt="image" src="https://github.com/user-attachments/assets/8bd8933f-59ef-4a65-b28f-e952e9480d1e" />

- `echo “hello rhel” > note.txt`: 문자열 “hello rhel”을 note.txt 파일에 저장함. 기존 내용이 있으면 덮어씀.
- `cat note.txt`: 파일 내용을 한 번에 화면에 출력
- `head -n 1 note.txt`: 파일의 앞에서 1줄만 보여줌
- `tail -n 1 note.txt`: 파일의 뒤에서 1줄만 보여줌
- `less note.txt`: 파일 내용을 페이지 단위로 천천히 볼 수 있게 연다. 종료는 `q`
- `file note.txt`: 이 파일이 어떤 종류의 파일인지 알려줌

특히 `less`는 짧은 파일의 경우 무관하지만, 내용이 길거나 로그 파일을 볼 때 유용할 것 같다.

### 5-4. 숨김 파일과 상세 정보 보기
점(`.`)으로 시작하는 파일은 숨김 처리

<img width="574" height="308" alt="image" src="https://github.com/user-attachments/assets/9667191a-293a-4902-9475-267384b87227" />

- `touch .hiddenfile`: .hiddenfile이라는 숨김 파일을 만듦. 점(`.`)으로 시작하면 숨김 파일로 취급
- `ls`: 현재 디렉터리의 일반 파일/폴더 이름만 보여줌
- `ls -a`: 숨김 파일까지 포함해서 모두 보여줌
- `ls -l`: 파일 디렉터리의 권한, 소유자, 크기, 수정 시간 등 상세 정보 보여줌
- `ls -al`: 숨김 파일 포함 + 상세 정보까지 모두 보여줌

### 5-5. 도움말 보기
모르는 명령어의 사용법 확인 가능

<img width="633" height="421" alt="image" src="https://github.com/user-attachments/assets/0357ad31-1074-480a-8295-d9f473f857b1" />

- `man ls`: ls 명령어의 매뉴얼 페이지 열기
- `man cp`: cp 명령어의 매뉴얼 페이지 열기
- `mkdir —help`: mkdir 명령어의 간단한 사용법 및 옵션 설명
- `rm —help`: rm 명령어의 간단한 사용법 및 옵션 설명

## 6. 절대 경로와 상대 경로
<img width="357" height="230" alt="image" src="https://github.com/user-attachments/assets/13cbe58e-61df-407a-ad82-1527aa4e3740" />


### 절대 경로
루트(`/`)부터 시작하는 경로로, 현재 위치와 관계없이 항상 같은 위치를 가리킨다.

예시: 
- `/home/username/lab/week1`
- `/etc`

### 상대 경로
현재 내가 있는 위치를 기준으로 찾는 경로
예시:
- `dir1`
- `..`
- `./dir1`

따라서,
- `cd /etc`는 절대 경로 이동
- `cd dir1`은 상대 경로 이동

## 7. 정리
이번 1주차 실습을 통해 RHEL가 단순한 리눅스 실습 환경이 아닌, 기업 환경에서 사용되는 엔터프라이즈 운영체제라는 점을 먼저 조사함

이와 더불어 핵심적으로 아래의 과정을 거치며 리눅스의 파일 구조와 권한 체계를 이해함
- 운영체제의 기본 디렉터리 구조
- 파일과 디렉터리 생성, 복사, 이동 삭제 실습
- 숨김 파일 및 상제 정보 출력 차이 확인
- 도움말을 통해 스스로 사용법 찾는 방법 실습
