## RHEL의 개요

- 기업 환경에서 사용되는 Linux 배포판이다
- 활용 예시
    - 기업 서버
    - 데이터센터
    - 클라우드 인프라
    - 하이브리드 클라우드
    - 컨테이너 환경

---

## Enterprise Linux 생태계

- 리눅스 생태계란 ?
    
    리눅스는 하나의 운영체제가 아닌 여러 배포판이 있는 존재
    
    → Ubuntu, Debian, Fedora Linux, Rocky Linux, Red Hat Enterprise Linux
    

Fedora(upstream) → CentOS Stream(midstream) → RHEL(Red Hat 공식 기업용 Linux) → Rocky Linux (RHEL과 호환되는 무료 배포판)

- upstream, downstream, midstream 차이
    
    upstream : 코드가 처음 만들어지는 곳, 실험적인 의미
    
    midstream: 새로운 기술이 바로 기업용 OS에 들어가면 위험하니까 중간 검증 단계를 두는 것
    
    downstream: 상류에서 흘러온 코드가 완전히 안정화되어 제품이 되는 단계 
    

---

## 설치 후 기본 환경 설정

- OS 정보 확인 : `cat /etc/os-release`
    - 예시
    
    ```sql
    NAME="Red Hat Enterprise Linux"
    VERSION="10"
    ```
    
- 커널 확인 : `uname -r`
- cpu 아키텍처 확인 : `uname -m` (내 경우 : arrach64)
- 호스트 이름 확인 : `hostnamectl`
- 네트워크 확인 : `ip a`
- 로그인 사용자 확인 : `whoami`
- 시스템 업데이트 : `sudo dnf update`

## 디렉토리 구조

    
- / : 루트 디렉토리
- /home : 사용자 홈 디렉토리
- /etc: 시스템 설정 파일
- /var: 로그/데이터
- /usr: 프로그램 파일
- /tmp : 임시 파일 , 재부팅 시 삭제 가능
- /boot :부팅 관련 파일

## 파일 탐색

- pwd : 현재 디렉토리 출력
- ls: 파일 목록 보기
    - ls - l : 파일들 상세 정보
    - ls - a: 숨김 파일 포함
    - ls - al : 상세 + 숨김 파일
- cd : 디렉토리 이동
    - cd ~ : 홈 이동
    - cd .. : 상위 폴더
    - cd 폴더명 : 해당 폴더로 이동

## 파일 관리

- cp : 파일 복사
    - `cp file1 file 2` :  file1을 file2라는 이름으로 복사
- mv : 파일 이동/ 이름 변경
    - `mv test.txt documents/`: test.txt 파일을 documents/ 폴더로 옮기기
    - `mv test.txt new.txt :` test.txt 파일명을 new.txt 이름으로 변경
- rm: 파일 삭제
    - rm - r dircetory : 폴더 삭제
- touch : 파일 생성
- mkdir: 디렉토리 생성

## 도움말 활용

- man : 매뉴얼 페이지
    - 명령어에 대해서 검색하면 아래처럼 답변이 나옴
    
    ```sql
    NAME        명령어 설명
    SYNOPSIS    명령어 사용 형식
    DESCRIPTION 자세한 설명
    OPTIONS     옵션 설명
    ```
    
    종료는 q 누르기 
    
    - 출력 예시 : man ls
        
        ```sql
        -a   숨김 파일 포함
        -l   상세 정보 출력
        ```
        
        ls 명령어에 대해서 속성값들을 포함한 설명들이 나옴 
        
- --help : 간단 도움말
    - 출력 예시 : ls --help
        
        ```sql
        Usage: ls [OPTION]... [FILE]...
        List information about FILEs
        
        Options:
          -a, --all        show hidden files
          -l               long format
          -h               human readable
        ``` 