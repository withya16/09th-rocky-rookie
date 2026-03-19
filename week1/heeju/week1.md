## 🐧 RHEL 기초 및 리눅스 기본 명령어 정리

### 1. RHEL 개요

**RHEL(Red Hat Enterprise Linux)**: Red Hat에서 개발한 상용 리눅스 배포판

특징

- 기업 환경에 최적화된 안정성
- 장기 지원(LTS)
- 보안 업데이트 및 기술 지원 제공
- 서버 및 클라우드 환경에서 널리 사용

### 2. 엔터프라이즈 리눅스 생태계

RHEL을 중심으로 다양한 배포판이 존재한다.

주요 구성

- RHEL: 상용, 안정성 중심
- CentOS: 무료 (현재는 Stream 형태로 변경)
- Rocky Linux / AlmaLinux: RHEL과 호환되는 무료 대안

생태계 특징

- 동일한 패키지 관리 시스템 사용 (RPM, YUM/DNF)
- 기업 환경에서 표준처럼 활용됨
- 클라우드(AWS, GCP 등)에서도 기본 OS로 많이 사용됨

### 3. 설치 후 기본 환경 설정

1. 시스템 업데이트
<pre>
sudo dnf update -y
</pre>
2. 사용자 확인
<pre>
whoami
</pre>
3. 네트워크 확인
<pre>
ip a
</pre>
4. 시간 설정 확인
<pre>
timedatectl
</pre>
5. 패키지 설치 예시
<pre>
sudo dnf install vim -y
</pre>

### 4. 디렉토리 구조

리눅스는 트리 구조의 파일 시스템을 사용한다.

| 디렉토리 | 설명                   |
| -------- | ---------------------- |
| /        | 루트 디렉토리 (최상위) |
| /home    | 사용자 홈 디렉터리     |
| /etc     | 설정 파일              |
| /var     | 로그 및 데이터         |
| /usr     | 프로그램 및 라이브러리 |
| /bin     | 기본 명령어            |
| /tmp     | 임시 파일              |

### 5. 파일 탐색 및 기본 명령어

1. 목록 조회 (ls)
<pre>
ls
ls -l
ls -a
</pre>
2. 디렉토리 이동 (cd)
<pre>
cd /home
cd ..
cd ~
</pre>
3. 현재 위치 확인
<pre>
pwd
</pre>

### 6. 파일/디렉토리 관리 명령어

1. 복사 (cp)
<pre>
// file2 내용을 file1로 복사
cp file1.txt file2.txt
// dir2의 내용을 dir1으로 복사
cp -r dir1 dir2
</pre>
2. 이동 (mv)
<pre>
// file.txt를 home 디렉토리로 이동
mv file.txt /home/
// old.txt를 new.txt로 이름 변경
mv old.txt new.txt
</pre>
3. 삭제 (rm)
<pre>
rm file.txt
rm -r dir
// force 옵션으로 강하게 삭제
rm -rf dir
</pre>

### 7. 도움말 활용

1. man (매뉴얼)
<pre>
man ls
</pre>
2. -help 옵션
<pre>
ls --help
cp --help
</pre>
