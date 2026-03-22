# 개념

전체적인 구조 :
`프로그램 → 패키지(RPM)로 묶음 → Repository 서버에 저장 → dnf가 다운로드`

``` sql
프로그램 (nginx)

↓ 패키징

nginx.rpm (패키지 파일)

↓ 저장

Repository

↓ 설치

dnf install nginx

↓ 내부적으로

rpm이 설치 수행
```

------------------------------------------------------------------------

## 패키지

-   프로그램을 설치할 수 있게 만든 파일
-   **`리눅스에서 프로그램의 실제 단위는 패키지`**
    -   *패키지를 모아둔 서버가 레포지토리*
-   실행 파일, 라이브러리, 설정 파일, 메타 데이터 포함
-   RHEL의 경우 RPM 패키지 형식 사용
    -   예시 : `nginx.rpm` = nginx 프로그램을 설치하기 위한 패키지 파일

------------------------------------------------------------------------

```{=html}
<aside>
```
DNF → 의존성 해결 + 다운로드 RPM → 실제 설치

```{=html}
</aside>
```
## DNF

-   Dandified YUM
-   고수준 패키지 관리자
-   RPM 기반 기본 패키지 관리 도구
-   의존성 자동 해결
-   repository에서 다운로드
-   패키지 검색, 패키지 설치, 패키지 업데이트, 패키지 제거, 의존성 자동
    해결, repository 관리

``` sql
dnf install package
```

------------------------------------------------------------------------

## RPM

-   Red Hat Package Manager
-   저수준 패키지 관리 도구
-   .rpm 과 rpm 명령어의 차이 : `.rpm`은 패키지 파일 그 자체인거고
    압축된 설치 묶음인 반면, `rpm (명령어)`는 패키지를 설치하는 도구
    "rpm 프로그램이 nginx.rpm 파일을 읽어서 시스템에 설치한다"
-   역할 : 패키지 설치 , 삭제 , 정보 확인
-   의존성 해결 못함

``` sql
rpm -ivh package.rpm
```

------------------------------------------------------------------------

## **RHEL Repository 구조**

RHEL 10의 소프트웨어는 두 개의 repository를 통해 배포되고 각 패키지는
BaseOS / AppStream 저장소에서 설치된다

`프로그램 설치 = repository 서버에서 패키지 다운로드`

같은 repository 안에는 수천개의 .rpm 패키지를 포함하고 있음

-   **레포지토리 설정 파일** : 패키지를 다운로드할 서버 주소를 적어둔
    파일

### **Repository 종류 1: BaseOS**

기본 운영체제 구성 패키지 / **운영체제 핵심 구성 요소**

-   bash
-   coreutils
-   glibc
-   systemd
-   kernel

### **Repository 종류 2:** AppStream

추가 애플리케이션 패키지 / **응용 프로그램 저장소**

-   python
-   nodejs
-   nginx
-   postgresql
-   git

**Repository 확인**

    dnf repolist

-   출력 예시

```{=html}
<!-- -->
```
    repo id          repo name
    rhel-10-baseos   Red Hat Enterprise Linux BaseOS
    rhel-10-appstream Red Hat Enterprise Linux AppStream

### Repository 위치

-   설정 파일 "컴퓨터가 패키지를 어디서 다운로드할지 적어둔 설정 파일
    폴더" 여길 들어가면 패키지를 어디서 다운로드 할지 알 수 있음

-   확인 `이 폴더 안에 있는 파일 목록 보기` " **`/etc/yum.repos.d` 폴더
    안에 있는 파일 목록을 보여줘"**

        ls /etc/yum.repos.d

-   예시

        redhat.repo

### Repository 확인

-   현재 활성 repository 확인

    ``` sql
    dnf repolist
    ```

-   전체 repository 확인

    ``` sql
    dnf repolist all 
    ```

### Repository 역할

Repository는 다음 기능을 한다.

-   패키지 다운로드
-   패키지 업데이트
-   의존성 관리
-   보안 업데이트 제공

작업은 DNF 패키지 관리 도구가 수행한다

------------------------------------------------------------------------

### 예시

`sudo dnf install nginx` 사용자가 이 명령을 실행하면 dnf가 내부적으로
여러 단계를 자동으로 수행함

1.  repository 설정 파일 확인
2.  활성 repository 목록 확인
3.  repository 서버에 접속
4.  nginx 패키지 검색
5.  의존성 패키지 확인
6.  패키지 다운로드
7.  rpm으로 설치

  명령어                  실제 의미             install 과정에서 하는 일
  ----------------------- --------------------- --------------------------
  `ls /etc/yum.repos.d`   레포 설정 파일 확인   1단계
  `cat redhat.repo`       레포 서버 주소 확인   1단계
  `dnf repolist`          활성 레포 확인        2단계
  `dnf search nginx`      패키지 검색           4단계

------------------------------------------------------------------------

## 사용자와 그룹

-   Linux는 여러 사람이 동시에 사용할 수 있도록 설계된 OS , 서버 한 대에
    여러 사람들이 동시 접속 가능
-   **그래서 시스템은 항상 누가`(User)`, 어떤 그룹에 속해 있고`(Group)`
    , 어떤 파일에 접근할 수 있는지`(Permission)`관리해야함**
-   사용자 생성 : useradd yoonseo
-   그룹 생성 : groupadd dev
-   권한 관리 : chmod

------------------------------------------------------------------------

## 파일 권한 구조

-   Owner, Group, Others
-   권한 종류 : `r (read) , w(write), x(execute)`
    -   예시 : -rwxr-xr---

------------------------------------------------------------------------

## 권한 관리 명령어

  명령어   역할
  -------- -------------
  chmod    권한 변경
  chown    소유자 변경
  chgrp    그룹 변경

# 실습

-   subscription 등록 완료

    ![스크린샷 2026-03-16 오전
    1.51.23.png](attachment:7110340d-418f-4860-bba6-c3f3b89268c0:스크린샷_2026-03-16_오전_1.51.23.png)

-   respository 설정 확인

        ls /etc/yum.repos.d

    -   결과

            redhat.repo

    -   설정 파일 내용 보기

            cat /etc/yum.repos.d/redhat.repo

-   repository 확인

    ``` sql
    dnf repolist
    ```

-   패키지 검색

    ``` sql
    dnf search nginx
    ```

-   패키지 정보 확인

    ``` sql
    dnf info nginx
    ```

    버전 , repository , 설명, 의존성 확인 가능

-   패키지 설치

    ``` sql
    sudo dnf install nginx
    ## 설치 과정 : repository 검색 --> dependency 확인 -> 다운로드 -> 설치 
    ```

-   설치된 패키지 확인

    ``` sql
    sudo useradd -aG devgroup devuser
    ```

-   패키지 업데이트

    ``` sql
    sudo dnf update
    ```

-   업데이트 확인

    ``` sql
    dnf check-update
    ```

-   패키지 삭제

    ``` sql
    sudo dnf remove nginx -y 
    ```

-   사용자 생성

    ``` sql
    sudo useradd student
    ```

-   사용자 확인

    ``` sql
    id student
    ```

-   사용자 삭제

    ``` sql
        sudo userdel student
    ```

-   그룹 생성

    ``` sql
        sudo groupadd devteam
    ```

-   사용자 + 그룹 연결

    ``` sql
    sudo useradd -aG devgroup devuser
    ```

-   파일 생성

    ``` sql
    touch test.txt
    ```

-   권한 변경

    ``` sql
    chmod 777 test.txt
    ```

        r = 4
        w = 2
        x = 1

-   소유자 변경

    ``` sql
    sudo chown student test.tx
    ```

-   그룹 변경

    ``` sql
    sudo chgrp devgroup test.txt
    ```

-   사용자 삭제

    ``` sql
    sudo userdel student
    ```
