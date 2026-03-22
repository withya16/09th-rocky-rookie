- DNF와 RPM의 개념 및 차이 이해, 패키지 설치·업데이트·삭제 실습, repository 구조와 설정 확인, 사용자 및 그룹 생성·변경·삭제, 파일 권한과 소유권 관리(chmod, chown, chgrp)

- rpm = 개별 패키지 파일 설치 도구
- yum = 의존성 자동 처리 도구 (구버전)
- dnf = yum의 신버전

## RPM

- RedHat사에서 제작한 설치파일 (Window의 setup.exe와 유사)
- 패키지 파일이 있어야 설치가 가능하기 때문에 인터넷과는 무관
- *.rpm = 패키지

## 명령어와 옵션

```bash
rpm -ivh <pkg>.rpm
```

### 설치

- -i : 설치 패키지를 위한 기본 옵션
- -v : 설치 과정 확인
- -h : 설치 진행과정을 화면에 출력
- --nodeps : 의존성 검사 무시
- --force : 강제설치
- --replacepkgs : 같은 버전 패키지를 다시 설치
- --replacefiles : 파일 충돌 무시하고 덮어쓰기
- --oldpackage : 다운그레이드 허용
- --test : 실제 설치 안하고 시뮬레이션

### 업그레이드

```bash
rpm -U <pkg>.rpm
rpm -F <pkg>.rpm
```

- -U : 업그레이드, 처음 설치 과정이라면 일반적인 설치
- -F : 이미 설치된 패키지만 업그레이드

### 삭제

```bash
rpm -e <pkg>
```

- -e : 삭제

### 조회

```bash
rpm -qa
rpm -qi <pkg>
rpm -qf /path
```

- -q : 조회
- -qa :  전부 조회
- -qi : 특정 패키지 조회
- -ql : 특정 패키지가 설치한 파일 목록 조회
- -qd : 문서 파일 조회
- -qc : 설정 파일 조회
- -qf : 특정 파일이 어느 패키지 소속인지 조회
- -qR : 패키지 의존성 요구사항 조회

### 검증

```bash
rpm -Va <pkg>
```

- -V : 설치된 파일이 패키지 DB 기준과 일치하는지 확인

## DNF

- 인터넷을 통해서 필요한 파일을 저장소에 다운로드하여 설치하는 방식
- rpm 명령어의 패키지 의존성 문제 해결
- repository를 사용

### 설치

```bash
dnf install <pkg>
```

### 삭제

```bash
dnf remove <pkg>
```

### 업그레이드

```bash
dnf upgrade <pkg>
```

- --allowerasing : 충돌 패키지 제거 허용

### 검색

```bash
dnf search nginx
```

### 정보 확인

```bash
dnf info nginx
```

### 설치된 패키지 목록

```bash
dnf list installed
```

### Repo 목록

```bash
dnf repolist
```

### 옵션

- -y :  자동 yes
- -q : 조용한 출력
- -v : 자세한 출력
- --downloadonly : 다운로드만

## 사용자 및 그룹

- 파일을 생성하는 사용자 = 파일의 소유자 & 파일의 그룹 소유자
- 읽기, 쓰기, 실행 권한이 할당됨
- 각 사용자에게 UID 부여 / 각 그룹에 GID 부여
    - 그룹 내 사용자는 동일한 권한을 가짐
- UID 0 = 슈퍼유저(root)
    - 파일 소유권 무시가능
    - 시스템 설정 변경가능
    - 사용자 추가/삭제 가능
    - 파일소유자는 root 사용자만 변경가능
- 사용자는 보통 하나의 기본 그룹을 가지고, 필요하면 여러 보조그룹에 추가될 수 있다

### 사용자 추가

```bash
useradd <user_name>
```

- -u <UID> : UID 지정
- -g <group_name> : 그룹에 포함시킴 (그룹이 존재해야 함)

### 사용자 비밀번호 지정

```bash
passwd <user_name>
```

### 사용자 그룹 변경

```bash
usermod -aG <group_name> <user_name>
```

### 사용자 삭제

```bash
userdel <user_name>
```

- -r : 홈디렉터리까지 함께 삭제

### Change

```bash
change [-option] <user_name>
```

- -1 : 사용자에게 설정된 사항 확인
- -m <number> : 사용자에 암호를 변경한 후 다음 변경까지 최소기간을 number일로 설정
- -E <date> : 사용자에 설정한 암호가 만료되는 날짜를 설정
- -W <number> : 암호가 만료되기 number일 전에 경고메세지 설정

### 사용자가 소속된 그룹 확인

```bash
groups
```

### 그룹 추가

```bash
groupadd <group_name>
```

- -g <GID> : 생성할 그룹의 GID 부여

### 그룹이름 변경

```bash
groupmod -n <group_name1> <group_name2>
```

### 그룹삭제

```bash
groupdel <group_name>
```

### 그룹관리

```bash
gpasswd <group_name>
```

- no option: 그룹암호 설정
- -A <user_name> : 사용자를 그룹의 관리자로 지정

## 권한

- 리눅스 파일 권한은 주로 -rwxr-x--로 나타냄
    - 3개로 끊어서 -rw/wr-/x--이고, 순서대로 user/group/other의 권한
- 파일과 디렉터리의 권한
    - 파일은 파일 내용읽기 / 파일 내용수정 / 실행가능
    - 디렉터리는 디렉터리 목록보기 / 내부 파일 생성,삭제,이름변경 / 해당 디렉터리에 들어가기
- 읽기/쓰기/실행 권한
    - r(read) = 4
    - w(write) = 2
    - x(execute) = 1
    - 7=rwx / 6=rw- / 5=r-x / 4=r--
- `chmod 755 script.sh` → 소유자는 rwx / 사용자는 r-x / 그외는 r-x의 권한을 갖게 함
- umask: 새로 생성되는 파일이나 디렉토리의 권한을 제어하는 명령어
    - 기본 권한에서 umask 값을 뺀 숫자가 앞으로 생성될 파일 및 디렉터리의 권한이 됨
- 파일의 기본 접근권한(666)에 umask 권한 제거
    - 주로 umask 022를 사용하여 파일 권한 644 사용
- 디렉터리의 기본 접근권한(777)에 umask 권한 제거
    - 주로 umask 022를 사용하여 디렉터리 권한 755 사용
- 현재 umask 확인

```bash
umask -S
```

- umask가 없는 상황에서 디렉터리와 파일을 생성하면 주로 아래와 같음

```bash
rw-r--r-- a.txt
drwxr-xr-x dir1
```

- umask를 022로 설정 후 디렉터리와 파일을 다시 생성하면 아래와 같음

```bash
rw------- b.txt
drwx------ dir2
```

- umask는 설정 후 새로 생성하는 파일에만 영향을 미침 (기존 파일들은 영향x)

### 권한 변경

```bash
chmod MODE <file_name>
```

- Mode = 644 같은 숫자 또는 +x 같은 권한 추가삭제

```bash
# user에 실행 권한 추가
chmod +x file.txt
chmod u+x file.txt

# group에서 쓰기권한 제거
chmod g-w file.txt

# 모든 사용자에 적용
chmod a+w file.txt

# group 권한 정확히 rx로 지정
chmod g=rx file.txt

# owner, group, other 권한 변경
chmod u=rw, g=r, o= file.txt
```

- 디렉터리 소유자를 변경하면 해당 디렉터리만 바뀌고 하위 모든 파일은 그대로 유지됨
- -R : 하위 디렉터리와 파일까지 같이 적용
- -v : 변경되는 파일마다 결과 출력
- -c : 실제로 변경된 경우만 출력