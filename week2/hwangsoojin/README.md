# 패키지 및 사용자 관리

## 패키지 관리 도구 (Package Manager)

### RPM (RPM Package Manager)

[Red Hat 공식 문서 - RPM](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/10/html/packaging_and_distributing_software/index)

RPM은 Red Hat 계열 리눅스에서 사용하는 기본 패키지 관리 시스템이다.  
소프트웨어를 하나의 단위로 묶어서 관리하며, 설치에 필요한 파일, 버전 정보, 의존성 정보 등을 함께 포함한다.

다만 RPM은 개별 패키지를 직접 설치하는 방식이기 때문에 의존성 문제를 자동으로 해결해주지 않는다.  
그래서 실제 환경에서는 DNF와 같은 상위 도구를 함께 사용한다.

---

### DNF (Dandified YUM)

[Red Hat 공식 문서 - DNF](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/index)

DNF는 YUM을 개선한 최신 패키지 관리자이다.  
패키지 설치 시 필요한 의존성을 자동으로 처리하며, 저장소를 기반으로 동작한다.

주요 특징:
- 의존성 자동 해결
- 빠른 처리 속도
- 안정적인 패키지 관리

---

### 주요 명령어

```bash
dnf repolist
dnf install <package>
dnf update <package>
dnf remove <package>
dnf search <keyword>
dnf info <package>
dnf clean all
dnf autoremove
```

---

## 저장소 (Repository)

패키지를 설치할 때 DNF는 인터넷 또는 내부 네트워크에 위치한 저장소에서 데이터를 가져온다.  
이 저장소를 Repository라고 한다.

Repository는 단순히 파일을 모아둔 장소가 아니라,  
패키지 목록과 의존성 정보까지 함께 제공하는 구조이다.

---

### 기본 Repository 구성

RHEL 계열에서는 보통 두 가지 저장소를 기본으로 사용한다.

- BaseOS: 시스템 운영에 필요한 핵심 패키지 제공
- AppStream: 웹 서버, 데이터베이스, 개발 도구 등 추가 소프트웨어 제공

---

### Repository 동작 흐름

1. 사용자가 `dnf install` 실행
2. DNF가 Repository의 메타데이터 확인
3. 필요한 패키지와 의존성 계산
4. 패키지 다운로드 후 설치 진행

즉, 사용자는 간단한 명령만 입력하지만 내부적으로는  
"패키지 탐색 → 의존성 확인 → 다운로드 → 설치" 과정이 자동으로 수행된다.

---

### Repository 종류

- 외부 저장소(Remote)
  - 인터넷을 통해 접근
  - 최신 패키지를 빠르게 사용할 수 있음

- 내부 저장소(Local)
  - 기업이나 기관 내부에 직접 구축
  - 폐쇄망이나 보안이 중요한 환경에서 주로 사용

---

### 설정 파일

DNF는 아래 경로의 설정 파일을 통해 Repository를 관리한다.

```ini
/etc/dnf/dnf.conf
/etc/yum.repos.d/*.repo
```

예시:

```ini
[example-repo]
name=Example Repository
baseurl=https://example.com/repo
enabled=1
gpgcheck=1
```

---

## 사용자 (User)

리눅스에서 사용자는 시스템에 로그인해서 작업을 수행하는 계정을 의미한다.  
각 사용자에게는 UID(User ID)라는 고유한 숫자 식별자가 부여된다.

리눅스는 여러 사용자가 동시에 사용하는 환경을 전제로 만들어졌기 때문에,  
"누가 어떤 파일에 접근할 수 있는지"를 사용자 기준으로 구분한다.

---

### 사용자 생성, 변경, 삭제

#### 사용자 생성

새 사용자를 만들 때는 `useradd` 명령어를 사용한다.

```bash
useradd soojin
```

이 명령은 계정을 생성하지만, 비밀번호는 따로 설정해야 한다.

```bash
passwd soojin
```

사용자가 정상적으로 만들어졌는지 확인하려면:

```bash
id soojin
```

출력 예시:

```text
uid=1001(soojin) gid=1001(soojin) groups=1001(soojin)
```

여기서:
- `uid`는 사용자 식별 번호
- `gid`는 기본 그룹 식별 번호
- `groups`는 현재 사용자가 속한 그룹 목록

---

#### 사용자 정보 변경

사용자 이름을 바꾸거나, 기본 그룹을 바꾸거나, 계정 상태를 변경할 때는 `usermod`를 사용한다.

```bash
usermod -l newsoojin soojin
```

- `-l`: 로그인 이름 변경

```bash
usermod -g group1 newsoojin
```

- `-g`: 기본 그룹 변경

```bash
usermod -e 2025-12-31 newsoojin
```

- `-e`: 계정 만료일 설정

```bash
usermod -L newsoojin
usermod -U newsoojin
```

- `-L`: 계정 잠금
- `-U`: 계정 잠금 해제

---

#### 사용자 삭제

사용자를 삭제할 때는 `userdel`을 사용한다.

```bash
userdel soojin
```

이 명령은 계정만 삭제하고 홈 디렉터리는 남길 수 있다.  
홈 디렉터리까지 함께 삭제하려면 `-r` 옵션을 사용한다.

```bash
userdel -r soojin
```

---

### root 계정과 sudo

root는 시스템 전체 권한을 가진 관리자 계정이다.  
UID는 0이며, 사용자 관리, 패키지 설치, 권한 변경 같은 중요한 작업을 수행할 수 있다.

```bash
su -
sudo <command>
```

보통 실무에서는 root로 계속 로그인하기보다,  
일반 사용자 계정으로 접속한 뒤 필요한 명령에만 `sudo`를 붙이는 방식이 더 안전하다.

---

### 관련 파일

사용자와 관련된 핵심 파일은 다음과 같다.

- `/etc/passwd` : 사용자 기본 정보
- `/etc/shadow` : 비밀번호 관련 정보
- `/etc/group` : 그룹 정보
- `/etc/skel` : 새 사용자 생성 시 기본 홈 디렉터리 내용
- `/etc/sudoers` : sudo 권한 설정

---

## 그룹 (Group)

그룹은 여러 사용자를 묶어서 권한을 관리하기 위한 단위이다.  
같은 그룹에 속한 사용자들은 특정 파일이나 디렉터리에 대해 공통 권한을 가질 수 있다.

예를 들어, 여러 명이 같은 프로젝트 폴더를 함께 사용해야 한다면  
각 사용자를 하나의 그룹에 넣고, 그 그룹에 폴더 권한을 부여하면 된다.

---

### 그룹 생성, 변경, 삭제

#### 그룹 생성

```bash
groupadd group1
```

그룹이 정상적으로 만들어졌는지 확인:

```bash
getent group group1
```

---

#### 그룹 이름 변경

```bash
groupmod -n newgroup group1
```

- `-n`: 그룹 이름 변경

---

#### 그룹 삭제

```bash
groupdel group1
```

단, 어떤 사용자가 해당 그룹을 기본 그룹으로 사용 중이라면 삭제가 제한될 수 있다.

---

### 그룹에 사용자 추가 / 제거

```bash
gpasswd -a soojin group1
```

- `-a`: 사용자를 그룹에 추가

```bash
gpasswd -d soojin group1
```

- `-d`: 사용자를 그룹에서 제거

현재 사용자가 어떤 그룹에 속해 있는지 확인:

```bash
groups soojin
```

---

### 사용자와 그룹을 함께 이해하기

리눅스에서는 보통 사용자 하나를 만들면 같은 이름의 기본 그룹도 함께 생성된다.  
예를 들어 `soojin` 사용자를 만들면 `soojin` 그룹이 기본 그룹이 될 수 있다.

그리고 필요에 따라 추가 그룹에 포함시켜서 권한을 확장한다.  
즉, "사용자 = 개인 계정", "그룹 = 권한 공유 단위"라고 이해하면 쉽다.

---

## 권한 (Permission)

리눅스에서 파일과 디렉터리는 소유자와 그룹 정보를 가지며,  
각각에 대해 어떤 작업을 허용할지 권한을 설정할 수 있다.

권한은 보안을 위한 핵심 개념이다.  
누구나 모든 파일을 수정할 수 있다면 시스템은 매우 위험해지기 때문이다.

---

### 권한 종류

- `r` (read): 읽기
- `w` (write): 쓰기
- `x` (execute): 실행

---

### 파일과 디렉터리에서의 의미 차이

같은 `r`, `w`, `x`라도 파일과 디렉터리에서 의미가 조금 다르다.

#### 파일의 권한

- `r`: 파일 내용을 읽을 수 있음
- `w`: 파일 내용을 수정할 수 있음
- `x`: 파일을 실행할 수 있음

#### 디렉터리의 권한

- `r`: 디렉터리 안의 목록을 볼 수 있음
- `w`: 디렉터리 안에서 파일 생성, 삭제, 이름 변경 가능
- `x`: 디렉터리 안으로 들어가거나 접근 가능

특히 디렉터리의 `x` 권한은 "실행"보다는 "진입 권한"으로 이해하는 것이 좋다.

---

### 권한의 적용 대상

권한은 세 부류에 대해 각각 설정된다.

- `u` (user): 소유자
- `g` (group): 소유 그룹
- `o` (others): 그 외 사용자

예를 들어 `rwxr-xr--`는 다음 뜻이다.

- 소유자: 읽기, 쓰기, 실행
- 그룹: 읽기, 실행
- 기타 사용자: 읽기만 가능

---

### 숫자 권한

권한은 숫자로도 표현할 수 있다.

- `r = 4`
- `w = 2`
- `x = 1`

그래서:

- `7 = 4+2+1 = rwx`
- `6 = 4+2 = rw-`
- `5 = 4+1 = r-x`
- `4 = 4 = r--`

예를 들어 `755`는:

- 소유자: 7 → rwx
- 그룹: 5 → r-x
- 기타 사용자: 5 → r-x

즉, `rwxr-xr-x`와 같다.

---

## 소유권 관리

리눅스의 각 파일은 "누가 소유자인지"와 "어떤 그룹에 속하는지"를 가진다.  
이를 소유권이라고 한다.

`ls -l` 명령으로 확인할 수 있다.

```bash
ls -l
```

예시:

```text
-rw-r--r-- 1 soojin group1 1234 Mar 19 12:00 file.txt
```

여기서:
- `soojin` → 파일 소유자
- `group1` → 파일 소유 그룹

---

### chmod: 권한 변경

`chmod`는 파일이나 디렉터리의 권한을 바꾸는 명령이다.

#### 숫자 방식

```bash
chmod 755 script.sh
chmod 644 README.md
```

- `755`: 실행 파일이나 디렉터리에 자주 사용
- `644`: 일반 텍스트 파일에 자주 사용

#### 문자 방식

```bash
chmod u+x script.sh
chmod g-w file.txt
chmod o+r README.md
```

예시 의미:
- `u+x`: 소유자에게 실행 권한 추가
- `g-w`: 그룹의 쓰기 권한 제거
- `o+r`: 기타 사용자에게 읽기 권한 추가

#### 여러 대상 한 번에 변경

```bash
chmod u=rwx,g=rx,o=r file.txt
```

이 명령은 권한을 지정한 값으로 정확히 설정한다.

#### 재귀 적용

```bash
chmod -R 755 project/
```

- `-R`: 하위 파일과 디렉터리까지 모두 변경

다만 재귀 권한 변경은 위험할 수 있으므로  
중요한 시스템 디렉터리에서 무분별하게 사용하면 안 된다.

---

### chown: 소유자 변경

`chown`은 파일의 소유자를 변경하는 명령이다.

```bash
chown soojin file.txt
```

소유자와 그룹을 함께 바꾸는 것도 가능하다.

```bash
chown soojin:group1 file.txt
```

디렉터리 전체에 재귀 적용하려면:

```bash
chown -R soojin:group1 project/
```

실무에서는 웹 서버 디렉터리의 소유자를 서비스 계정으로 맞출 때 자주 사용한다.

---

### chgrp: 그룹 변경

`chgrp`는 파일의 소유 그룹만 바꾸는 명령이다.

```bash
chgrp group1 file.txt
```

디렉터리에도 적용할 수 있다.

```bash
chgrp -R group1 project/
```

즉:
- 소유자만 바꾸고 싶으면 `chown`
- 그룹만 바꾸고 싶으면 `chgrp`
- 둘 다 바꾸고 싶으면 `chown user:group`

이렇게 구분하면 된다.

---

### umask: 기본 권한 설정

새 파일이나 디렉터리를 만들 때 적용될 기본 권한을 간접적으로 조정하는 값이다.

```bash
umask 022
```

일반적으로:
- 파일 기본 권한은 666
- 디렉터리 기본 권한은 777

여기서 umask 값을 빼서 최종 권한이 결정된다.

예:
- 파일: `666 - 022 = 644`
- 디렉터리: `777 - 022 = 755`

즉, `umask 022`는  
"소유자는 쓰기 가능, 나머지는 쓰기 불가" 같은 일반적인 기본 정책을 만들 때 자주 사용한다.

---

## 자주 쓰는 예시 정리

### 예시 1: 사용자 생성 후 그룹에 추가

```bash
useradd soojin
passwd soojin
groupadd group1
gpasswd -a soojin group1
groups soojin
```

---

### 예시 2: 프로젝트 폴더 소유권과 권한 맞추기

```bash
mkdir project
chown soojin:group1 project
chmod 755 project
```

---

### 예시 3: 실행 스크립트 만들기

```bash
touch backup.sh
chmod 755 backup.sh
```

또는:

```bash
chmod u+x backup.sh
```

---

### 예시 4: 공동 작업 폴더 그룹 설정

```bash
mkdir teamdir
chown soojin:group1 teamdir
chmod 775 teamdir
```

이렇게 하면 소유자와 같은 그룹 사용자는 디렉터리 안에서 파일 생성과 수정이 가능하다.

---

## 핵심 정리

- 사용자는 시스템에 접근하는 계정이고, UID를 가진다.
- 그룹은 여러 사용자를 묶어 공통 권한을 관리하는 단위이다.
- `useradd`, `usermod`, `userdel`로 사용자 생성·변경·삭제를 수행한다.
- `groupadd`, `groupmod`, `groupdel`, `gpasswd`로 그룹을 관리한다.
- 권한은 `r`, `w`, `x`로 구성되며, 파일과 디렉터리에서 의미가 조금 다르다.
- `chmod`는 권한 변경, `chown`은 소유자 변경, `chgrp`는 그룹 변경 명령이다.
- `umask`는 새 파일과 디렉터리의 기본 권한에 영향을 준다.

