# 패키지 및 사용자 관리

## 패키지 매니저 (Package Manager)

### RPM (RPM Package Manager)

[Red Hat Document - RPM](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/10/html/packaging_and_distributing_software/index)

`RPM`은 RHEL, Rocky Linux, CentOS, Fedora 같은 Red Hat 계열 리눅스 배포판에서 사용하는 패키지 관리 시스템입니다. 소프트웨어 설치에 필요한 파일, 버전 정보, 의존성, 설정 정보 등을 .rpm 패키지 파일 형식으로 관리합니다.
RPM은 개별 패키지를 직접 다루는 하위 관리 도구이기 때문에, 여러 패키지를 편리하게 관리하려면 `DNF`나 `YUM` 같은 상위 패키지 관리 도구와 함께 사용합니다.

### YUM (Yellowdog Updater Modified), DNF (Dandified YUM)

[Red Hat Document - DNF](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/index)

`YUM`은 과거 RHEL 계열에서 널리 사용되던 기본 패키지 관리자이며, 패키지 간 의존성을 자동으로 해결하고 저장소(Repository)에서 필요한 파일을 내려받아 설치할 수 있습니다.

`DNF`는 YUM의 후속 도구로, 의존성 처리 성능과 속도, 안정성을 개선한 최신 패키지 관리자입니다. 현재 Fedora, RHEL 8 이상, Rocky Linux, AlmaLinux 등에서는 주로 DNF를 사용합니다.

> e.g. Debian/Ubuntu의 apt, macOS의 brew

### 주요 명령어

```bash
# 버전 확인
dnf --version

# 저장소 목록 확인
dnf repolist

# 저장소 추가
dnf config-manager --add-repo <repository URL>

# 설치된 패키지 목록 확인
dnf list installed

# 패키지 설치
dnf install <package>

# 패키지 업데이트
dnf update <package>

# 패키지 삭제
dnf remove <package>

# 패키지 검색
dnf search <keyword>

# 패키지 정보 확인
dnf info <package>

# 패키지 그룹 설치
dnf group install <group>

# 패키지 그룹 삭제
dnf group remove <group>

# 캐시 정리
dnf clean all

# 불필요한 패키지 삭제
dnf autoremove
```

> DNF 및 관련 유틸리티의 구성은 `/etc/dnf/dnf.conf` 파일의 [main] 섹션에 저장됩니다.

## 저장소 (Repository)

`Repository`는 RPM 패키지와 메타데이터를 제공하는 소프트웨어 저장소입니다. RHEL 10은 기본적으로 `BaseOS`와 `AppStream` 두 개의 주요 저장소를 중심으로 콘텐츠를 배포합니다.

### BaseOS

BaseOS 리포지토리는 모든 설치 환경의 기반이 되는 핵심 운영체제 기능과 필수 패키지를 제공합니다.

> e.g. systemd, bash, glibc, python3, rpm, dnf...

### AppStream

AppStream 리포지토리는 다양한 워크로드와 사용 사례를 지원하는 추가 애플리케이션, 런타임 언어, 데이터베이스 등의 패키지를 제공합니다.

> e.g. httpd, nginx, postgresql, nodejs...

### Local Repository와 Remote Repository

`Local Repository`는 서버 내부 디스크나 사내 네트워크에 직접 구성한 저장소를 의미합니다. 주로 폐쇄망 환경이나 내부 표준 패키지 배포 환경에서 사용하며, 외부 네트워크 연결 없이도 패키지 설치와 업데이트가 가능합니다.

`Remote Repository`는 인터넷이나 외부 네트워크를 통해 접근하는 저장소를 의미합니다. 대표적으로 Red Hat CDN이나 외부 벤더가 제공하는 저장소가 있으며, 최신 패키지를 편리하게 사용할 수 있다는 장점이 있습니다.

### 설정 파일

Repository 설정은 전역 설정 파일인 `/etc/dnf/dnf.conf`와 개별 저장소 설정 파일인 `/etc/yum.repos.d/*.repo`를 사용해 관리합니다.

> dnf에서도 `yum.repos.d`라는 이름을 그대로 사용합니다. (yum과의 호환성을 위해?)

1. 전역 설정 파일 - `/etc/dnf/dnf.conf`

   ```ini
   # dnf.conf

   [main]
   gpgcheck=1 # 서명 검증
   clean_requirements_on_remove=1 # 패키지 삭제 시 불필요한 의존성 정리
   skip_if_unavailable=0 # 저장소가 일시적으로 불가할 때 작업 스킵 여부
   assumeyes=1 # 자동 yes
   ...
   ```

2. 개별 저장소 설정 파일 - `/etc/yum.repos.d/*.repo` (권장)

   ```ini
   # internal.repo

   [internal-baseos] # Repository ID
   name=Internal BaseOS Repository
   baseurl=https://repo.example.com/rhel/10/baseos/x86_64/os/ # 저장소 주소
   enabled=1 # 저장소 사용 여부
   gpgcheck=1
   gpgkey=https://repo.example.com/keys/RPM-GPG-KEY-internal # gpgkey 위치
   metadata_expire=6h # 메타데이터 유효 시간
   ...

   [internal-appstream]
   name=Internal AppStream Repository
   baseurl=https://repo.example.com/rhel/10/appstream/x86_64/os/
   enabled=1
   gpgcheck=1
   gpgkey=https://repo.example.com/keys/RPM-GPG-KEY-internal
   metadata_expire=6h
   ...
   ```

   ```ini
   # redhat.repo

   [rhel-10-for-x86_64-baseos-rpms]
   name=Red Hat Enterprise Linux 10 for x86_64 - BaseOS RPMs
   baseurl=https://cdn.redhat.com/content/dist/rhel10/10/x86_64/baseos/os
   enabled=1
   gpgcheck=1
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
   sslverify=1 # HTTPS 인증서 검증 여부
   sslcacert=/etc/rhsm/ca/redhat-uep.pem
   sslclientkey=/etc/pki/entitlement/1234567890123456789-key.pem
   sslclientcert=/etc/pki/entitlement/1234567890123456789.pem
   metadata_expire=24h
   enabled_metadata=1
   ...
   ```

> gpg는 GNU Privacy Guard의 약자로, 파일이나 패키지에 대한 전자서명과 암호화를 수행하는 도구입니다. 파일이나 패키지의 출처와 무결성을 검증할 수 있습니다.

## 사용자 (User)

`User`는 시스템에 로그인하여 작업을 수행하는 계정입니다. 각 사용자는 `UID(User ID)`라는 고유한 숫자 식별자를 가집니다.

### 주요 명령어

```bash
# 사용자의 기본 정보 확인
id <username>

# 사용자가 속한 그룹 확인
groups <username>

# 사용자 생성
useradd <username>

# 사용자 비밀번호 설정
passwd <username>

# 사용자 삭제
userdel <username>

# 사용자명 변경
usermod -l <new_username> <username>

# 사용자 그룹 변경
usermod -g <groupname> <username>

# 계정 만료일 설정
usermod -e 2025-12-31 <username>

# 계정 잠금
usermod -L <username>

# 계정 잠금 해제
usermod -U <username>

# 계정 전환
su <username>

# 계정 및 환경 전환
su - <username>
```

### root

`root`는 시스템 전체에 대한 최고 권한을 가진 계정입니다. UID가 0이며, 시스템 전역 설정 변경, 사용자 관리, 서비스 제어, 파일 권한 변경 같은 관리자 작업을 수행할 수 있습니다.

```bash
# root 계정으로 전환
su -

# 관리자 권한으로 실행
sudo <command>
```

### 관련 파일

1. `/etc/passwd`: 사용자 계정 기본 정보
2. `/etc/shadow`: 암호 정보
3. `/etc/group`: 그룹 정보
4. `/etc/skel`: 사용자 생성 시 홈 디렉터리에 생성되는 파일
5. `/etc/sudoers`: sudo 권한 설정 파일

## 그룹 (Group)

`Group`은 여러 사용자를 묶어 공통 권한을 관리하는 단위입니다. 각 그룹은 `GID(Group ID)`라는 고유한 숫자 식별자를 가집니다. 같은 그룹에 속한 사용자는 해당 그룹에 부여된 권한을 공유하며, 그룹이 소유한 파일과 디렉터리에 대해 읽기, 쓰기, 실행 권한을 가질 수 있습니다.

```bash
# 그룹 정보 조회
getent group <groupname>

# 그룹 생성
groupadd <groupname>

# 그룹명 변경
groupmod -n <new_groupname> <groupname>

# 그룹 삭제
groupdel <groupname>

# 그룹에 사용자 추가
gpasswd -a <username> <groupname>

# 그룹에서 사용자 제거
gpasswd -d <username> <groupname>
```

## 권한 (Permission)

`Permission`은 파일과 디렉터리에 대해 누가 어떤 작업을 할 수 있는지를 정하는 접근 제어 규칙입니다.

- 파일 기본 권한: 666, rw-rw-rw-
- 디렉터리 기본 권한: 777, rwxrwxrwx

### 종류

- read - r(4): 읽기 / 디렉터리 목록 조회 권한
- write - w(2): 쓰기 / 파일 생성, 삭제, 이름 변경 권한
- execute - x(1): 실행 / 디렉터리 진입 권한

### 대상

- u - user: 소유자
- g - group: 그룹
- o - others: 기타 사용자

### 표기 방법

1. 문자
   `rwxr-xr--`

   소유자(user) 권한은 `rwx` = 읽기, 쓰기, 실행  
   그룹(group) 권한은 `r-x` = 읽기, 실행  
   기타 사용자(others) 권한은 `r--` = 읽기

2. 숫자
   `755`

   소유자(user) 권한은 `7` = 읽기, 쓰기, 실행  
   그룹(group) 권한은 `5` = 읽기, 실행  
   기타 사용자(others) 권한은 `5` = 읽기, 실행

### 주요 명령어

```bash
# 파일과 디렉터리의 권한, 소유자, 그룹 확인
ls -l <filename>
ls -ld <directory>

# 권한 변경
chmod <level><operation><permission> <filename>
chmod a+r README.md # 모든 사용자에게 읽기 권한 추가
chmod u=rwx,g=rx,o=r file.txt # 권한을 지정한 값으로 설정

chmod 755 temp.txt # 권한을 지정한 값으로 설정

chmod -R ... <directory> # 재귀적으로 변경

# 소유자 및 그룹 변경
chown <username> <filename>
chown bk file.txt

chown <username>:<groupname> <filename>
chown bk:cloudclub file.txt

chown -R <username> <directory> # 재귀적으로 변경

# 그룹 변경
chgrp <groupname> <filename>
chgrp <groupname> <directory>

# 기본 권한 설정
umask 022 # 666-022=644, 777-022=755
```
