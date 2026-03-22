## 🐧 DNF & RPM, 사용자/권한 관리 정리

### 1. DNF와 RPM의 개념

RHEL 계열 리눅스에서는 소프트웨어를 설치하거나 관리할 때 주로 **RPM**과 **DNF**를 사용한다.  
둘 다 패키지 관리와 관련된 도구이지만 역할과 사용 방식에는 차이가 있다.

### RPM

**RPM(Red Hat Package Manager)**은 RHEL 계열 리눅스에서 사용하는 기본 패키지 관리 형식이자 도구이다. 확장자가 `.rpm`인 패키지 파일을 직접 설치, 조회, 삭제할 수 있다.

### RPM의 역할

- 패키지 파일 설치
- 설치된 패키지 목록 조회
- 특정 패키지 삭제
- 패키지 정보 확인

### RPM의 특징

- `.rpm` 파일을 직접 다룸
- 저수준 패키지 관리 도구
- 의존성 문제를 자동으로 해결하지 못하는 경우가 많음
- 단독으로 쓰기보다는 DNF의 기반 역할로 많이 사용됨

### RPM 주요 명령어

<pre>
rpm -ivh package.rpm
</pre>

- `-i` install, 설치
- `-v` verbose, 자세한 출력
- `-h` hash, 진행 상태 표시

---

- 현재 설치된 모든 패키지 조회
<pre>rpm -qa</pre>
- 특정 패키지의 상세 정보 확인
<pre>rpm -qi bash</pre>
- 해당 패키지가 설치한 파일 목록 확인
<pre>rpm -ql bash</pre>
- 패키지 삭제
<pre>rpm -e package_name</pre>

### RPM의 한계

어떤 패키지를 설치하려고 할 때, 그 패키지가 다른 라이브러리를 필요로 하면 RPM은 이를 자동으로 모두 처리하지 못할 수 있다.
이런 문제를 해결하기 위해 보통 DNF를 사용한다.

### 3. DNF

**DNF(Dandified YUM)**은 RHEL 계열에서 사용하는 고수준 패키지 관리자이다.
패키지를 설치할 때 필요한 의존성을 자동으로 함께 처리하고, 저장소(Repository)에서 패키지를 내려받아 설치한다.

DNF의 특징

- Repository 기반으로 동작
- 의존성을 자동 해결
- 설치, 삭제, 업데이트가 간편
- 실무에서 주로 사용하는 패키지 관리자

### DNF 주요 명령어

- nginx 설치
<pre>dnf install nginx</pre>
- 설치 과정에서 자동으로 yes 처리
<pre>sudo dnf install nginx -y</pre>
- nginx 삭제
<pre>dnf remove nginx</pre>
- 전체 패키지 업데이트
<pre>dnf update</pre>
- 설치된 패키지 목록 확인
<pre>dnf lit installed</pre>
- 저장소에서 패키지 검색
<pre>dnf search nginx</pre>
- 패키지 상세 정보 확인
<pre>dnf info nginx</pre>

### 4. RPM vs DNF

**RPM**: 패키지 파일을 직접 다루는 도구<br>
**DNF**: 저장소를 이용해 패키지를 더 편리하게 관리하는 도구

### 6. Repository 구조와 설정 확인

### Repository

리눅스 패키지들이 저장되어 있는 서버 또는 패키지 공급원이다. DNF는 이 저장소에서 필요한 패키지를 검색하고 다운로드한다. <br>
-> 앱스토어처럼 패키지를 제공하는 장소

### Repository 확인 명령어

- 현재 활성화된 저장소 목록 확인
<pre>dnf repolist</pre>
- 활성화/비활성화된 저장소 전체 확인
<pre>dnf repolist all</pre>

### Repository 설정 파일 위치

<pre>ls /etc/yum.repos.d/</pre>

RHEL 계열에서는 보통 /etc/yum.repos.d/ 디렉터리 안에 .repo 파일들이 저장되어 있다.

### .repo 파일에서 자주 보는 항목

```ini
[AppStream]
name=RHEL AppStream
baseurl=http://example.com/repo
enabled=1
gpgcheck=1
```

- `[]` 저장소 ID
- `name` 저장소 이름
- `baseurl` 실제 패키지를 가져오는 주소
- `enabled=1` 저장소 활성화
- `enabled=0` 저장소 비활성화
- `gpgcheck=1` 패키지 서명 검증 사용 여부

### 7. 사용자(User) 관리

리눅스는 여러 사용자가 동시에 시스템을 사용할 수 있도록 설계되어 있다. 각 사용자는 독립적인 계정과 홈 디렉터리를 가진다.

- 사용자 생성
<pre>sudo useradd testuser</pre>
- 비밀번호 설정
<pre>sudo passwd testuser</pre>
- 사용자 정보 확인
<pre>id testuser</pre>
- 사용자 삭제
<pre>sudo userdel testuser
// 홈 디렉토리까지 삭제 
sudo userdel -r testuser</pre>

### 8. 그룹(Group) 관리

리눅스에서는 여러 사용자를 하나의 그룹으로 묶어서 권한을 효율적으로 관리할 수 있다.

예를 들어 개발자 그룹, 운영자 그룹, 테스트 그룹처럼 나눌 수 있다.

- 그룹 생성
<pre>sudo groupadd devteam</pre>
- 그룹 정보 확인
<pre>grep devteam /etc/group</pre>
- 사용자를 그룹에 추가
  <pre>sudo usermod -aG devteam testuser</pre>
  `-aG` 기존 그룹은 유지하고 추가 그룹에 넣음
- 사용자 그룹 확인
<pre>groups testuser</pre>
- 그룹 삭제
<pre>sudo groupdel devteam</pre>

### 9. 파일 권환과 소유권 관리

리눅스에서는 파일과 디렉터리마다 누가 읽고, 쓰고, 실행할 수 있는지 권한을 설정할 수 있다. 권한 관리는 보안과 시스템 운영에서 매우 중요하다.

### 권한 구조

권한 문자열은 다음과 같이 나뉜다.
**첫 글자**

- `-` 일반 파일
- `d` 디렉터리

**나머지 9자리**
3자리씩 끊어서 본다.

- `rwx` 소유자(owner) 권한
- `r-x` 그룹(group) 권한
- `r--` 기타 사용자(other) 권한

### 권한 종류

| 기호 | 의미          | 숫자 |
| ---- | ------------- | ---- |
| r    | 읽기(read)    | 4    |
| w    | 쓰기(write)   | 2    |
| x    | 실행(execute) | 1    |

숫자 권한은 합으로 계산한다.

<pre>
- rwx = 4+2+1 = 7
- r-x = 4+0+1 = 5
- r-- = 4+0+0 = 4
</pre>

### 10. chmod 명령어

`chmod` 파일이나 디렉터리의 권한을 변경

<pre>chmod 754 file.txt</pre>

- 소유자: rwx
- 그룹: r-x
- 기타 사용자: r–

### 문자 방식

- 소유자에게 실행 권한 추가
<pre>chmod u+x script.sh</pre>
- 그룹의 쓰기 권한 제거
<pre>chmod g-w file.txt</pre>
- 기타 사용자에게 읽기 권한 추가
<pre>chmod o+r file.txt</pre>

- `u` user
- `g` group
- `o` other
- `a` all

### 11. chown 명령어

`chown` 파일의 소유자(owner) 또는 소유 그룹을 변경

- 소유자 변경
<pre>sudo chown testuser file.txt</pre>
- 소유자와 그룹 동시에 변경
<pre>sudo chown testuser:devteam file.txt</pre>
- 변경 확인
<pre>ls -l file.txt</pre>

### 12. chgrp 명령어

`chgrp` 파일의 그룹 소유권만 변경

<pre>sudo chgrp devteam file.txt</pre>

확인:

<pre>ls -l file.txt</pre>
