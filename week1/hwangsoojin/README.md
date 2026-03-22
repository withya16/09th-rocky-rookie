# Week 1 - RHEL 소개 및 기초

## 1. 개념 설명

### 1) RHEL이란?

RHEL(Red Hat Enterprise Linux)은 기업 환경에서 안정성과 보안을 중요하게 고려하여 설계된 리눅스 배포판이다.  
서버 운영, 클라우드 인프라, 데이터센터 환경에서 널리 사용된다.

---

### 2) 리눅스 디렉터리 구조

리눅스는 모든 것을 파일로 관리하며, 트리 구조의 디렉터리 시스템을 가진다.

주요 디렉터리:

- / : 최상위 루트 디렉터리
- /home : 사용자 홈 디렉터리
- /etc : 설정 파일 저장
- /var : 로그 및 가변 데이터
- /usr : 프로그램 및 라이브러리
- /bin : 기본 실행 파일
- /tmp : 임시 파일

---

### 3) 기본 명령어

CLI 환경에서 파일과 디렉터리를 다루는 기본 명령어들이다.

---

## 2. 핵심 정리

- RHEL은 기업용 리눅스 배포판
- 리눅스는 트리 구조의 파일 시스템을 사용
- 모든 것은 파일로 취급됨
- CLI 명령어로 시스템을 제어

---

## 3. 명령어 / 예시

```bash
# 현재 위치 확인
pwd

# 디렉터리 이동
cd /home

# 파일 목록 확인
ls
ls -l

# 파일 복사
cp file1 file2

# 파일 이동 / 이름 변경
mv file1 file2

# 파일 삭제
rm file1

# 디렉터리 생성
mkdir dir1

# 디렉터리 삭제
rm -r dir1
```

---

## 4. 추가 학습 내용 (심화)

### 도움말 확인

```bash
man ls
ls --help
```

---

### 절대 경로 vs 상대 경로

#### 절대 경로 (Absolute Path)

- 항상 `/`로 시작
- 루트 디렉터리 기준의 전체 경로

예시:
```bash
cd /home/soojin
cat /etc/passwd
```

👉 현재 위치와 상관없이 동일한 경로를 가리킨다.

---

#### 상대 경로 (Relative Path)

- 현재 위치를 기준으로 경로를 지정
- `.` : 현재 디렉터리
- `..` : 상위 디렉터리

예시:

현재 위치: `/home/soojin`

```bash
cd ..
# → /home 으로 이동

cd ./documents
# → /home/soojin/documents

cat ../file.txt
# → /home/file.txt
```

---

#### 비교 예시

현재 위치: `/home/soojin`

| 작업 | 절대 경로 | 상대 경로 |
|------|----------|----------|
| documents 이동 | cd /home/soojin/documents | cd documents |
| 상위 디렉터리 이동 | cd /home | cd .. |
| 파일 읽기 | cat /home/soojin/file.txt | cat file.txt |

