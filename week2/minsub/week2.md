## 2주차 스터디 개요
>RHEL에서 소프트웨어를 설치하고 관리하는 방법과 여러 사용자가 함께 사용할 때 필요한 사용자/그룹 및 파일 권한 관리 방법 학습 진행

1주차에서 디렉터리 구조와 기본 명령어를 익혔고, 이번에는 시스템 관리자 관점에서 RHEL을 다루는 기본 방법을 익히고자 한다.

RHEL에서는 패키지를 통해 소프트웨어를 설치하고 관리하며, 이를 위해 DNF와 RPM을 사용한다. 또한 다중 사용자 환경에서 계정과 그룹을 구분하고, 파일 및 디렉터리에 적절한 권한을 부여함으로써 보안과 협업 구조를 관리할 수 있다.

### 1. 패키지란?
리눅스에서 프로그램은 보통 `패키지(package)` 형태로 설치하고 관리한다.
예를 들면, `git`, `tree`, `wget`, `vim` 같은 것도 다 패키지라고 보면 된다.

윈도우에서 `.exe` 파일을 직접 설치하는 방식과 비슷해 보일 수 있지만, 리눅스에서는 패키지 관리자가 설치, 업데이트, 삭제를 보다 체계적으로 처리한다는 차이가 있다.
### 2. DNF와 RPM
- DNF: 패키지를 설치, 검색, 업데이트, 삭제할 때 주로 쓰는 도구

```
sudo dnf install tree
sudo dnf remove tree
sudo dnf search git
```

- RPM: 패키지의 설치 여부, 정보, 파일 목록 같은 걸 확인할 때 주로 쓰는 도구

```
rpm -q tree
rpm -qi tree
rpm -ql tree
```

### 3. Repository란?
패키지를 가져오는 저장소

- 예를 들어, `dnf install tree` 했을 때 
- 시스템이 repository를 보고
- 거기서 패키지를 내려 받아 설치

이 작업을 위해 repository 설정 및 Red Hat 등록이 필요하다.

## 1. 패키지 관리 실습
### 1-1. repository 확인
RHEL은 설치만으로 바로 모든 패키지 저장소를 사용할 수 있는 구조가 아니다. 패키지 설치와 업데이트를 위해서는 먼저 시스템을 Red Hat 계정에 등록해야 한다. 등록되지 않은 상태에서는 `dnf repolist` 실행 시 사용 가능한 저장소가 없다고 표시되며, 공식 패키지 설치 및 업데이트를 진행할 수 없다. 이를 해결하기 위해 `subscription-manager register` 명령을 사용해 시스템을 Red Hat에 등록했다. 등록 후 `subscription-manager status`로 시스템 상태를 확인했고, 이후 `dnf repolist` 명령에서 RHEL 10의 기본 저장소인 BaseOS와 AppStream이 정상적으로 조회되는 것을 확인했다.

<img width="50%" height="350" alt="image" src="https://github.com/user-attachments/assets/99f9bd25-6da8-4f2c-8eda-e537f2590bf1" />


- `sudo subscription-manager register` : 현재 시스템을 Red Hat 계정에 등록
- `sudo subscription-manager status` : 등록 상태 확인
- `sudo dnf repolist` : 현재 사용 가능한 저장소 목록 확인

### 1-2. 패키지 검색

<img width="50%" height="350" alt="image" src="https://github.com/user-attachments/assets/3afe45b9-0fd1-461e-9144-c55f2f5dd6c2" />


- `sudo dnf search tree`: tree와 관련된 패키지 검색

### 1-3 패키지 정보 확인

<img width="50%" height="350" alt="image" src="https://github.com/user-attachments/assets/cd12b9e5-ac92-44e4-963d-8f38eadddb26" />


검색한 `tree` 패키지의 상세 정보를 확인해보았다.


- `dnf info tree`: 패키지의 이름, 버전, 설명, 저장소 정보 등 출력

### 1-4. 패키지 설치

<img width="50%" height="200" alt="image" src="https://github.com/user-attachments/assets/65da993b-5bc8-45df-b7ac-90801c803aed" />


`tree` 패키지를 실제로 설치해보았다.

- `dnf install`: 패키지 설치
- `-y` 옵션: 설치 중 나오는 확인 질문에 자동으로 `yes` 처리

### 1-5. 설치 확인
<p align="center">
<img width="46%" height="200" alt="image" src="https://github.com/user-attachments/assets/a1003001-25a6-4158-a68a-f3c697104b43" />
<img width="46%" height="214" alt="image" src="https://github.com/user-attachments/assets/e7ad3d8c-9b5c-428f-bb79-50629309dae0" />
</p>


설치가 정상적으로 완료되었는지 여러 방식으로 확인

- `tree --version` : tree 명령어 버전 확인
- `rpm -q tree` : tree가 설치되어 있는지 확인
- `rpm -qi tree` : tree 패키지의 상세 정보 확인
- `rpm -ql tree | head` : tree 패키지가 설치한 파일 목록 일부 확인

여기서 dnf는 설치/삭제/검색의 용도, rpm은 설치 여부, 상세 정보, 파일 목록 확인의 용도라는 차이를 확인

### 1-6. 패키지 삭제

<img width="50%" height="300" alt="image" src="https://github.com/user-attachments/assets/e8354214-3ddb-4cec-8f86-3be0f25afbd5" />


설치한 `tree` 패키지를 다시 삭제해보기

- `tree` 패키지 삭제
- 이후 `rpm -q tree`로 정말 삭제됐는지 확인

## 2. 사용자와 그룹 관리
### 2-1. 현재 사용자 관리

<img width="50%" height="166" alt="image" src="https://github.com/user-attachments/assets/d949125d-5587-42a6-9adf-e15140fb4a24" />


- `whoami` : 현재 로그인 사용자 확인
- `id` : UID, GID, 속한 그룹 확인
- `groups` : 현재 사용자가 속한 그룹 이름 확인

### 2-2. 그룹 생성

<img width="300" height="69" alt="image" src="https://github.com/user-attachments/assets/cf48173a-a15e-48e0-a2a9-d344cb10997b" />

- `groupadd devteam` : `devteam` 그룹 생성
- `getent group devteam` : 생성된 그룹 정보 확인

### 2-3. 사용자 생성 

<img width="424" height="243" alt="image" src="https://github.com/user-attachments/assets/0df061ec-ffa6-4005-9d08-f1e6df68d6af" />


- `useradd alice` : 사용자 `alice` 생성
- `passwd alice` : `alice` 비밀번호 설정

### 2-4. 그룹에 사용자 추가

<img width="549" height="121" alt="image" src="https://github.com/user-attachments/assets/be7fa506-b691-4b07-80a3-a0f566ffd70c" />


- `usermod -aG devteam alice` : `alice`를 `devteam` 보조 그룹에 추가
- `id alice` : alice의 UID/GID/그룹 확인
- `groups alice` : alice가 속한 그룹 확인

### 2-5. 사용자 하나 더 생성

<img width="436" height="131" alt="image" src="https://github.com/user-attachments/assets/183be411-c0af-481a-b342-bab6f47c008d" />


추가 실습을 위해 bob도 추가

## 3. 파일 권한 관리
### 3-1. 파일 만들고 권한 보기

<img width="483" height="165" alt="image" src="https://github.com/user-attachments/assets/cd24321b-cead-4a55-8a66-a0163da80d0c" />



- `cd ~` : 현재 사용자의 홈 디렉터리로 이동
- `mkdir permtest`: mkdir은 make directory의 약자로 여기선 permtest 라는 디렉터리 생성
- `cd permtest`: permest 디렉터리로 이동
- `touch a.txt`: a.txt 파일 생성
- `ls-l`: 현재 디렉터리의 파일 목록과 상세 정보

### 3-2. chmod 실습

<img width="493" height="244" alt="image" src="https://github.com/user-attachments/assets/594a6c61-164e-4b17-bbe4-455898bc3037" />


`chmod`는 change mode의 약자로, 파일이나 디렉터리의 권한을 변경하는 명령으로 쓰인다.

숫자로 권한을 지정할 때 아래의 3자리 숫자로 표현을 한다.

- 첫 번째 자리: 소유자 (user)
- 두 번째 자리: 그룹 (group)
- 세 번째 자리: 기타 사용자(other)

각 숫자는 합으로 표현하는데,

- 4 = 읽기 (read, r)
- 2 = 쓰기 (write, w)
- 1 = 실행 (execute, x)

따라서, 6 = 4+ 2 = rw- 가 되고,  0 = —-

이를 기반으로 600을 해석하면

- 소유자: 읽기 + 쓰기
- 그룹: 권한 없음
- 기타: 권한 없음

즉 파일 주인만 읽고 쓸 수 있단 뜻이 된다.

해석 요약은 아래와 같다.

- `chmod 600` : 소유자만 읽기/쓰기 가능
- `chmod 644` : 소유자는 읽기/쓰기, 나머지는 읽기만 가능
- `chmod 755` : 소유자는 읽기/쓰기/실행, 나머지는 읽기/실행 가능

숫자 권한 요약

- `7 = rwx`
- `6 = rw-`
- `5 = r-x`
- `4 = r--`
- `0 = ---`

### 3-3. 소유자와 그룹 변경

<img width="470" height="180" alt="image" src="https://github.com/user-attachments/assets/335f1b8d-4a5a-49a1-b3e2-d599cb5e708f" />


`sudo`는 관리자 권한으로 명령을 실행할 때 쓰는 명령어고, 여기서 `chown`은 change owner의 약자로 파일 및 디렉터리의 소유자를 바꾸는 명령어다.
또한 `chgrp`는 파일 및 디렉터리의 그룹을 변경하는 명령어이다.

- `chown alice a.txt` : 파일 소유자를 `alice`로 변경

- `chgrp devteam a.txt` : 파일 그룹을 `devteam`으로 변경

한 번에 소유자와 그룹을 함께 바꾸는 것도 가능하다.

- `chown alice:devteam a.txt` : 파일 소유자는 `alice`, 그룹은 `devteam`으로 변경

### 3-4. 공유 디렉터리 실습

<img width="543" height="105" alt="image" src="https://github.com/user-attachments/assets/212d04e7-64e9-4d0d-bb52-a0432679f087" />


이번에는 특정 그룹 사용자들만 함께 사용할 수 있는 공유 디렉터리를 만들어보았다.

- `mkdir /sharedlab` : 공유 디렉터리 생성

- `chown root:devteam /sharedlab` : 소유자는 root, 그룹은 devteam으로 설정

- `chmod 770 /sharedlab` : 소유자와 그룹만 접근 가능하도록 권한 설정

- `ls -ld /sharedlab` : 디렉터리 자체의 권한 정보 확인

여기서 `770`은 다음을 의미한다.

- 소유자 : 읽기 / 쓰기 / 실행 가능

- 그룹 : 읽기 / 쓰기 / 실행 가능

- 기타 사용자 : 권한 없음

즉, `devteam` 그룹에 속한 사용자만 `/sharedlab`에 접근하고 파일을 생성할 수 있는 구조를 만든 것이다.

### 3-5. 다른 사용자로 접근 테스트

<img width="487" height="186" alt="image" src="https://github.com/user-attachments/assets/dd43caf7-acc4-4a49-a1c6-23276220dc0c" />


생성한 공유 디렉터리에 `alice`, `bob`이 실제로 접근 가능한지 확인

- `su - alice` : 현재 셸을 alice 사용자로 전환
- `cd /sharedlab` : 공유 디렉터리로 이동
- `touch alice.txt` : alice.txt 파일 생성
- `ls -l` : 생성 결과 확인
- `exit` : 현재 사용자 셸 종료 후 원래 계정으로 복귀

<img width="476" height="204" alt="image" src="https://github.com/user-attachments/assets/e013e9a3-226b-4206-9fa1-02ce96be8ef6" />


bob으로도 테스트

여기서 `su`는 switch user의 약자로, 다른 사용자 계정으로 전환할 때 사용한다.
또한 `-` 옵션은 단순히 사용자만 바꾸는 것이 아니라, 해당 사용자의 로그인 셸 환경까지 함께 적용한다는 의미를 가진다. 즉 홈 디렉터리, 환경변수, 로그인 초기 설정 등이 전환한 사용자 기준으로 바뀐다.

## 4. 정리
이번 2주차 실습을 통해 하기의 사항들을 익혔다.
- RHEL에서 패키지를 설치하고 삭제하는 기본 흐름
- DNF와 RPM의 역할 차이
- 사용자와 그룹을 생성하고, 파일 및 디렉터리에 권한을 부여함으로써 리눅스의 다중 사용자 환경이 어떻게 동작하는지 실습


특히 공유 디렉터리를 생성하고 여러 사용자가 실제로 접근하는 과정을 통해, 권한 설정이 단순한 이론이 아니라 협업과 보안 구조를 구성하는 핵심 요소임을 확인할 수 있었다.
