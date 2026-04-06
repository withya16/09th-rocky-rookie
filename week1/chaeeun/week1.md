## VM 세팅하기

- ISO 이미지 다운로드
- SSH 접속을 위해 포트포워딩
- 구독 계정등록

```bash
sudo subscription-manager register --force
```

## RHEL

- Red Hat에서 제공하는 기업용 리눅스 배포판

## 기본 디렉터리

| 디렉터리 | 설명 |
| --- | --- |
| / | 루트 디렉터리 |
| /home | 사용자 홈 디렉터리 |
| /etc | 설정 파일 |
| /var | 로그, 데이터 |
| /usr | 프로그램 |
| /bin | 기본 명령어 |
| /sbin | 관리자 명령어 |
| /tmp | 임시 파일 |
| /boot | 부팅 파일 |
| /dev | 장치 파일 |

## 기본 명령어

### 네트워크 확인

```c
ip -a
```

### 사용자 확인

```c
whoami
```

### 시간확인

```c
date
timedatectl
```

### 파일 목록 보기

```c
ls
```

- -l : 상세 정보
- -a : 숨김 파일 포함
- -h : 용량 보기 좋기
- -R : 하위 디렉터리까지

### 디렉터리 이동

```c
cd
```

- cd : 홈이동
- cd .. : 상위 디렉터리로 이동
- cd ~ : 홈 디렉터리로 이동
- cd - : 이전 위치

### 현재 위치 확인

```c
pwd
```

- 절대경로로 출력됨

### 파일 복사

```c
cp file1 file2
```

- -r : 디렉터리 복사
- -i : 덮어쓰기 확인
- -v : 과정 출력

### 이동/이름 변경

```c
mv file1 /path
mv file1 file2
```

- mv file1 /path : 해당 디렉터리로 파일 이동
- mv file1 file2 : 파일 이름 변경

### 삭제

```c
rm file
rm -r dir
```

- -r : 디렉터리 삭제
- -f : 강제 삭제
- -i : 삭제 확인g