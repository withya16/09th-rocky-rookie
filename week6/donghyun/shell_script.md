> **주제:** 로키 리눅스 스터디 — 셸 스크립트 및 자동화 기초
**범위:** Bash Shell 구조, 변수/환경 변수, 표준 입출력/Redirection/Pipe, 조건문/반복문, 관리 작업 자동화
> 

---

# Part 1. Bash Shell의 기본 구조

## 1-1. Shell이란 무엇인가 — Kernel과 사용자 사이의 통역관

Shell은 사용자의 명령어를 받아서 **Kernel에게 전달**하고, Kernel의 실행 결과를 사용자에게 돌려주는 **인터페이스 프로그램**입니다. 사용자가 `ls -la`를 치면, Shell이 이 문자열을 파싱하여 Kernel의 System Call로 변환합니다.

Shell은 단순한 명령어 전달자가 아닙니다. **프로그래밍 언어**이기도 합니다. 변수, 조건문, 반복문, 함수를 지원하며, 이것이 "Shell Script"입니다.

**Linux에서 사용 가능한 Shell 종류:**

| Shell | 경로 | 특징 |
| --- | --- | --- |
| **bash** | `/bin/bash` | **RHEL/Rocky 기본 Shell.** 가장 보편적. POSIX 호환 + 확장 기능 |
| sh | `/bin/sh` | POSIX 표준 Shell. bash보다 기능 제한적. Rocky에서는 bash로 Symlink |
| zsh | `/bin/zsh` | macOS 기본. bash 호환 + 강력한 자동완성 |
| fish | `/usr/bin/fish` | 사용자 친화적. 스크립트 호환성은 떨어짐 |

```bash
# 현재 사용 중인 Shell 확인
echo $SHELL

# 시스템에 설치된 Shell 목록
cat /etc/shells

# 현재 Shell의 PID
echo $$

mr8356@mr8356:~$ echo $SHELL
/bin/bash

mr8356@mr8356:~$ cat /etc/shells
/bin/sh
/bin/bash
/usr/bin/sh
/usr/bin/bash

mr8356@mr8356:~$ echo $$
2021
```

## 1-2. Shebang (#!) — 스크립트의 첫 줄이 왜 중요한가

Shell Script 파일의 **첫 줄**에는 반드시 **Shebang**이 와야 합니다.

```bash
#!/bin/bash
```

이 줄은 "이 스크립트를 `/bin/bash` 인터프리터로 실행해라"라는 **Kernel에 대한 지시**입니다. Shebang이 없으면 Kernel은 현재 사용자의 기본 Shell로 실행하려 시도하는데, 환경에 따라 예상과 다른 Shell(sh, dash 등)이 실행될 수 있어 스크립트가 깨집니다.

**Shebang 변형:**

| Shebang | 용도 |
| --- | --- |
| `#!/bin/bash` | Bash 스크립트 (가장 보편) |
| `#!/bin/sh` | POSIX 호환 스크립트 (이식성 최대) |
| `#!/usr/bin/env bash` | PATH에서 bash를 찾아 실행 (이식성 좋음) |
| `#!/usr/bin/python3` | Python 스크립트 |
| `#!/usr/bin/env python3` | PATH에서 python3를 찾아 실행 |

> **AWS SAA 직접 연결 — EC2 User Data:**
EC2 Instance가 최초 부팅될 때 자동 실행되는 **User Data Script**의 첫 줄이 반드시 `#!/bin/bash`입니다. 이것이 없으면 cloud-init이 스크립트로 인식하지 못합니다.
> 
> 
> SAA 시험 단골: "EC2가 프로비저닝될 때 자동으로 Nginx를 설치하고 시작해야 한다. 어떻게?"
> → EC2 User Data에 Shell Script를 넣는다:
> 
> ```bash
> #!/bin/bash
> dnf install -y nginx
> systemctl enable --now nginx
> echo "<h1>Hello from $(hostname)</h1>" > /usr/share/nginx/html/index.html
> ```
> 
> Shebang을 빠뜨리면 User Data가 실행되지 않아 Nginx가 설치 안 됩니다.
> 

## 1-3. 스크립트 생성과 실행

```bash
# 1. 파일 생성
cat << 'EOF' > hello.sh
#!/bin/bash
echo "Hello, $(whoami)! Today is $(date +%Y-%m-%d)"
EOF

# 2. 실행 권한 부여 (필수!)
chmod +x hello.sh

# 3. 실행 방법들
./hello.sh              # 직접 실행 (Shebang 사용)
bash hello.sh           # bash를 명시적으로 호출 (Shebang 무시)
source hello.sh         # 현재 Shell에서 실행 (변수가 현재 Shell에 남음)
. hello.sh              # source와 동일
```

**`./hello.sh` vs `source hello.sh` — 결정적 차이:**

`./hello.sh`는 **새로운 Child Process(Sub-shell)**를 생성하여 스크립트를 실행합니다. 스크립트 안에서 설정한 변수나 `cd` 등은 Parent Shell에 영향을 주지 않습니다.

`source hello.sh`는 **현재 Shell에서 직접** 스크립트를 실행합니다. 스크립트 안에서 설정한 변수, `cd`, `export` 등이 현재 Shell에 그대로 남습니다.

이것이 `~/.bashrc` 수정 후 `source ~/.bashrc`를 실행하는 이유입니다 — 현재 Shell에 변경사항을 즉시 적용하기 위해.

## 1-4. Exit Status — 모든 명령어는 종료 코드를 반환한다

모든 Linux 명령어(프로그램)는 종료 시 **정수 값(Exit Status)**을 반환합니다.

| Exit Code | 의미 |
| --- | --- |
| `0` | **성공** |
| `1` | 일반적 오류 |
| `2` | Shell 내장 명령 오류 (잘못된 사용법) |
| `126` | 파일 실행 권한 없음 |
| `127` | 명령어를 찾을 수 없음 (PATH에 없음) |
| `128+N` | Signal N에 의해 종료 (예: 137 = 128+9 = SIGKILL) |

```bash
# 직전 명령어의 Exit Status 확인
ls /tmp
echo $?    # 0 (성공)

mr8356@mr8356:~$ ls /tmp
systemd-private-e638d1c6119044599b9f3cec711960bf-chronyd.service-hkRZ5F
systemd-private-e638d1c6119044599b9f3cec711960bf-dbus-broker.service-3RfDNs
systemd-private-e638d1c6119044599b9f3cec711960bf-irqbalance.service-6C0wQi
systemd-private-e638d1c6119044599b9f3cec711960bf-polkit.service-YzYCQa
systemd-private-e638d1c6119044599b9f3cec711960bf-systemd-logind.service-dXvsGy

mr8356@mr8356:~$ echo $?
0

ls /nonexistent
echo $?    # 2 (파일 없음)

mr8356@mr8356:~$ ls /nonexistent
ls: cannot access '/nonexistent': No such file or directory

mr8356@mr8356:~$ echo $?
2

kill -9 12345
# Process가 SIGKILL로 죽으면 Exit Code = 137 (128 + 9)
```

> **Terraform Associate 직접 연결 — Provisioner와 Exit Code:**
Terraform의 `local-exec`와 `remote-exec` Provisioner는 내부에서 Shell 명령을 실행합니다. **Exit Code가 0이 아니면 Terraform은 배포를 실패로 처리**합니다.
> 
> 
> ```hcl
> resource "aws_instance" "web" {
>   # ...
>   provisioner "local-exec" {
>     command = "/opt/scripts/validate.sh"
>     # 이 스크립트가 exit 1을 반환하면 terraform apply가 실패합니다
>   }
> }
> ```
> 
> Terraform 시험에서 "Provisioner 실행 시 에러가 발생하면 어떻게 되는가?" → "Exit Code가 non-zero면 리소스가 tainted 상태가 되고 다음 apply에서 재생성된다."
> 

```bash
# 스크립트에서 의도적으로 Exit Code 설정
#!/bin/bash
if [ ! -f /etc/myapp.conf ]; then
    echo "ERROR: Config file not found" >&2
    exit 1    # 실패를 알림
fi
echo "Config OK"
exit 0        # 성공
```

---

# Part 2. 변수와 환경 변수

## 2-1. Shell 변수 — 현재 Shell에서만 유효

```bash
# 변수 할당 (= 양쪽에 공백 금지!)
NAME="mr8356"
COUNT=42
GREETING="Hello, ${NAME}"

# 변수 참조
echo $NAME
echo ${NAME}       # 중괄호 — 변수명 경계가 모호할 때 필수
echo "${NAME}_log" # mr8356_log (중괄호 없으면 $NAME_log를 찾음)

# 변수 삭제
unset NAME
```

**= 양쪽에 공백을 넣으면 안 되는 이유:**

```bash
NAME = "mr8356"    # ❌ 에러! bash가 'NAME'을 명령어로, '='를 인자로 해석
NAME="mr8356"      # ✅ 올바른 할당
```

이것은 Bash의 가장 흔한 실수이자, 다른 프로그래밍 언어(Python, JS)와의 가장 큰 차이입니다.

**변수의 Scope:**

Shell 변수는 **선언된 Shell Process에서만** 유효합니다. Child Process(Sub-shell, 스크립트 실행 등)에는 전달되지 않습니다.

```bash
MY_VAR="hello"
bash -c 'echo $MY_VAR'    # 출력 없음! Child Shell에서 접근 불가
```

## 2-2. 환경 변수 (Environment Variable) — Child Process에도 전달

`export`로 선언한 변수는 **환경 변수**가 됩니다. 환경 변수는 해당 Shell에서 실행되는 **모든 Child Process에 자동 상속**됩니다.

```bash
# 환경 변수로 등록
export MY_VAR="hello"
bash -c 'echo $MY_VAR'    # "hello" 출력! Child에도 전달됨

# 한 줄에 선언 + export
export NODE_ENV="production"

# 현재 설정된 모든 환경 변수 확인
env
# 또는
printenv

# 특정 환경 변수 확인
printenv PATH
echo $PATH
```

**Shell 변수 vs 환경 변수:**

| 구분 | Shell 변수 | 환경 변수 |
| --- | --- | --- |
| 선언 | `VAR=value` | `export VAR=value` |
| Scope | 현재 Shell만 | 현재 Shell + 모든 Child Process |
| Child 상속 | ❌ | ✅ |
| 확인 | `set` (전부) | `env` / `printenv` (환경 변수만) |

> **Terraform Associate 직접 연결 — TF_VAR_ 접두어:**
Terraform은 `TF_VAR_` 접두어가 붙은 환경 변수를 **자동으로 Terraform 변수에 주입**합니다. 이때 `export`를 이해하지 못하면 변수가 Terraform Process(Child)에 전달되지 않습니다.
> 
> 
> ```bash
> # Terraform 변수를 환경 변수로 주입
> export TF_VAR_instance_type="t3.micro"
> export TF_VAR_region="ap-northeast-2"
> 
> # 이 상태에서 terraform plan을 실행하면
> # variable "instance_type"에 "t3.micro"가 자동 할당됨
> terraform plan
> ```
> 
> Terraform 시험 단골: "코드에 하드코딩하지 않고 터미널에서 변수를 주입하는 방법은?"
> → `export TF_VAR_<변수명>=<값>`
> 
> `export` 없이 `TF_VAR_instance_type="t3.micro"`만 하면 **현재 Shell에만** 존재하고, terraform Process(Child)에는 전달되지 않아 변수가 비어있게 됩니다.
> 
> **TF_LOG 환경 변수:**
> 
> ```bash
> export TF_LOG=TRACE    # Terraform 디버그 로깅 활성화
> export TF_LOG=""       # 로깅 비활성화
> ```
> 
> Terraform 시험: "terraform apply에서 상세 디버그 로그를 보려면?" → `export TF_LOG=TRACE`
> 

## 2-3. 주요 시스템 환경 변수

| 변수 | 의미 | 예시 |
| --- | --- | --- |
| `PATH` | 명령어를 찾는 디렉터리 목록 (콜론 구분) | `/usr/local/bin:/usr/bin:/bin` |
| `HOME` | 현재 사용자의 홈 디렉터리 | `/home/mr8356` |
| `USER` | 현재 사용자 이름 | `mr8356` |
| `SHELL` | 현재 사용자의 기본 Shell | `/bin/bash` |
| `PWD` | 현재 작업 디렉터리 | `/home/mr8356/scripts` |
| `HOSTNAME` | 호스트 이름 | `rocky-dev` |
| `LANG` | Locale 설정 | `en_US.UTF-8` |
| `EDITOR` | 기본 텍스트 에디터 | `vim` |
| `PS1` | Shell Prompt 형식 | `[\u@\h \W]\$` |

**PATH가 왜 중요한가:**

`ls`를 치면 Bash는 PATH에 나열된 디렉터리를 **왼쪽부터 순서대로** 탐색하여 `ls` 실행 파일을 찾습니다. PATH에 없는 디렉터리의 프로그램은 전체 경로를 써야 합니다.

```bash
# PATH 확인
echo $PATH

# PATH에 디렉터리 추가 (현재 세션만)
export PATH="$PATH:/opt/myapp/bin"

# 영구 추가 — ~/.bashrc에 추가
echo 'export PATH="$PATH:/opt/myapp/bin"' >> ~/.bashrc
source ~/.bashrc

# 명령어의 실제 위치 확인
which kubectl
type kubectl
```

## 2-4. 변수 영구화 — Shell 설정 파일

| 파일 | 실행 시점 | 용도 |
| --- | --- | --- |
| `/etc/profile` | **Login Shell** 시작 시 (전체 사용자) | 시스템 전역 환경 변수 |
| `/etc/profile.d/*.sh` | `/etc/profile`이 로드 | 모듈별 전역 설정 |
| `~/.bash_profile` | Login Shell 시작 시 (해당 사용자) | 사용자별 Login 설정 |
| `~/.bashrc` | **모든 Interactive Shell** 시작 시 | alias, 함수, Prompt 설정 |
| `/etc/bashrc` | 모든 Interactive Shell (전체 사용자) | 시스템 전역 bashrc |

**Login Shell vs Non-login Shell:**

- `ssh user@host` → Login Shell → `~/.bash_profile` 실행
- 터미널에서 `bash` 입력 → Non-login Shell → `~/.bashrc` 실행
- `~/.bash_profile`에서 보통 `source ~/.bashrc`를 호출하므로 둘 다 적용됨

```bash
# ~/.bashrc에 alias 추가 (영구)
echo 'alias k="kubectl"' >> ~/.bashrc
echo 'alias kgp="kubectl get pods"' >> ~/.bashrc
source ~/.bashrc

# 확인
alias
```

> **CKA/CKS 연결 — 시험 환경 세팅:**
CKA/CKS 시험 시작 시 가장 먼저 하는 세팅:
> 
> 
> ```bash
> echo 'alias k=kubectl' >> ~/.bashrc
> echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
> echo 'export do="--dry-run=client -o yaml"' >> ~/.bashrc
> source ~/.bashrc
> ```
> 
> 이후 `k run nginx --image=nginx $do > pod.yaml` 한 줄로 매니페스트 생성. 이 세팅을 안 하면 시험 시간이 절대 부족합니다.
> 

## 2-5. 특수 변수

| 변수 | 의미 |
| --- | --- |
| `$0` | 스크립트 이름 |
| `$1`, `$2`, ... | 위치 매개변수 (인자) |
| `$#` | 인자 개수 |
| `$@` | 모든 인자 (개별 문자열로) |
| `$*` | 모든 인자 (하나의 문자열로) |
| `$?` | 직전 명령어의 Exit Status |
| `$$` | 현재 Shell의 PID |
| `$!` | 마지막 Background Process의 PID |

```bash
#!/bin/bash
echo "스크립트 이름: $0"
echo "첫 번째 인자: $1"
echo "두 번째 인자: $2"
echo "인자 개수: $#"
echo "모든 인자: $@"

# 실행: ./myscript.sh hello world
# 출력:
# 스크립트 이름: ./myscript.sh
# 첫 번째 인자: hello
# 두 번째 인자: world
# 인자 개수: 2
# 모든 인자: hello world
```

## 2-6. Command Substitution과 Quoting

### Command Substitution — 명령어의 출력을 변수에 담기

```bash
# $(...) 방식 (권장)
TODAY=$(date +%Y-%m-%d)
HOSTNAME=$(hostname)
POD_COUNT=$(kubectl get pods --no-headers | wc -l)

# 백틱 방식 (구식, 중첩이 어려움)
TODAY=`date +%Y-%m-%d`

echo "오늘: $TODAY, 호스트: $HOSTNAME, Pod 수: $POD_COUNT"
```

### Quoting — 작은따옴표 vs 큰따옴표 vs 따옴표 없음

| 방식 | 변수 확장 | 특수문자 해석 | 사용 시점 |
| --- | --- | --- | --- |
| `"..."` (큰따옴표) | ✅ 확장됨 | `$`, ```, `\`, `!`만 해석 | **대부분의 경우 권장** |
| `'...'` (작은따옴표) | ❌ 그대로 | 아무것도 해석 안 함 | 변수 확장을 막고 싶을 때 |
| 따옴표 없음 | ✅ 확장됨 | Word Splitting 발생 | 단순 값에만 |

```bash
NAME="mr8356"
echo "Hello, $NAME"     # Hello, mr8356   (변수 확장)
echo 'Hello, $NAME'     # Hello, $NAME    (그대로)
echo Hello, $NAME       # Hello, mr8356   (확장, 공백 주의)

# 공백이 포함된 값은 반드시 따옴표!
FILE="my document.txt"
cat $FILE      # ❌ 'my'와 'document.txt' 두 파일을 찾으려 함
cat "$FILE"    # ✅ 'my document.txt' 하나를 찾음
```

---

# Part 3. 표준 입출력, Redirection, Pipe

## 3-1. File Descriptor — 모든 I/O의 근본

Linux에서 모든 I/O는 **File Descriptor (FD)**를 통해 이루어집니다. 프로세스가 시작되면 자동으로 3개의 FD가 열립니다.

| FD 번호 | 이름 | 기본 연결 | 용도 |
| --- | --- | --- | --- |
| 0 | **stdin** (Standard Input) | 키보드 | 프로그램에 데이터 입력 |
| 1 | **stdout** (Standard Output) | 터미널 화면 | 정상 출력 |
| 2 | **stderr** (Standard Error) | 터미널 화면 | 오류 메시지 |

stdout과 stderr이 **분리**되어 있는 것이 핵심입니다. 둘 다 화면에 보이지만 실제로는 다른 채널이므로, 각각 독립적으로 Redirect할 수 있습니다.

## 3-2. Output Redirection — 출력을 파일로 보내기

| 문법 | 동작 | 기존 파일 |
| --- | --- | --- |
| `>` | stdout을 파일로 | **덮어쓰기** |
| `>>` | stdout을 파일로 | **이어쓰기(Append)** |
| `2>` | stderr을 파일로 | 덮어쓰기 |
| `2>>` | stderr을 파일로 | 이어쓰기 |
| `&>` | stdout + stderr 모두 파일로 | 덮어쓰기 |
| `&>>` | stdout + stderr 모두 파일로 | 이어쓰기 |
| `2>&1` | stderr를 stdout과 같은 곳으로 | - |

```bash
# stdout만 파일로 (덮어쓰기)
ls /tmp > output.txt

# stdout 이어쓰기
echo "new line" >> output.txt

# stderr만 파일로
ls /nonexistent 2> error.txt

# stdout과 stderr를 각각 다른 파일로
command > stdout.txt 2> stderr.txt

# stdout과 stderr를 모두 같은 파일로
command > all.txt 2>&1
# 또는 (bash 4+)
command &> all.txt

# stderr는 버리고 stdout만 보기
command 2>/dev/null

# 모든 출력 버리기 (조용히 실행)
command > /dev/null 2>&1
```

> **CKA/CKS 직접 연결 — 시험의 핵심 패턴:**
> 
> 
> **패턴 1: 로그에서 특정 내용 추출하여 파일에 저장**
> 
> ```bash
> # "특정 Pod의 에러 로그를 /opt/course/error.txt에 저장하시오"
> kubectl logs my-pod | grep -i error > /opt/course/error.txt
> 
> # "이전 Container의 로그에서 OOM 관련 내용 추출"
> kubectl logs my-pod --previous | grep -i "out of memory" > /opt/course/oom.txt
> ```
> 
> **패턴 2: dry-run으로 YAML 매니페스트 생성**
> 
> ```bash
> # "nginx Pod을 생성하는 YAML을 작성하시오"
> kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
> 
> # "nginx Deployment를 생성하는 YAML"
> kubectl create deployment nginx --image=nginx --replicas=3 \
>   --dry-run=client -o yaml > deploy.yaml
> 
> # "ClusterRole 생성 YAML"
> kubectl create clusterrole pod-reader \
>   --verb=get,list,watch --resource=pods \
>   --dry-run=client -o yaml > clusterrole.yaml
> ```
> 
> **이 `> file.yaml` 패턴을 못 쓰면 CKA에서 모든 YAML을 손으로 타이핑해야 합니다.**
> 
> **패턴 3: 결과를 파일에 Append**
> 
> ```bash
> # "모든 Namespace의 Pod 수를 /opt/course/pod-count.txt에 기록"
> kubectl get pods -A --no-headers | wc -l >> /opt/course/pod-count.txt
> ```
> 

## 3-3. Input Redirection

```bash
# 파일에서 stdin 읽기
wc -l < /etc/passwd

# Here Document — 스크립트 안에서 여러 줄 입력
cat << 'EOF' > config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  key1: value1
  key2: value2
EOF

mr8356@mr8356:~$ cat << 'EOF' > test.yaml
> apiVersion: v1
> kind: ConfigMap
>   name
> EOF
mr8356@mr8356:~$ cat test.yaml
apiVersion: v1
kind: ConfigMap
  name
mr8356@mr8356:~$

# tee를 이용해서 터미널에 입력 + 출력 확인도 할 수 있다.
mr8356@mr8356:~$ cat << 'EOF' | tee test.yaml
> data:
  key1: value1
  key2: value2
> EOF
data:
  key1: value1
  key2: value2

# Here String — 문자열을 stdin으로
grep "error" <<< "this is an error message"
```

**Here Document에서 `'EOF'` vs `EOF`:**

```bash
cat << 'EOF'    # 작은따옴표 → 변수 확장 안 됨 (그대로)
cat << EOF      # 따옴표 없음 → 변수 확장 됨
```

## 3-4. Pipe ( | ) — 명령어를 체인으로 연결

Pipe는 **왼쪽 명령어의 stdout을 오른쪽 명령어의 stdin으로** 연결합니다. Unix 철학의 핵심: "한 가지 일을 잘 하는 작은 프로그램들을 조합한다."

```bash
# 기본 구조
command1 | command2 | command3

# 예시: /etc/passwd에서 Shell이 /bin/bash인 사용자 수 세기
cat /etc/passwd | grep "/bin/bash" | wc -l

# 더 효율적인 방식 (cat 불필요 — Useless Use of Cat)
grep "/bin/bash" /etc/passwd | wc -l
```

**자주 사용하는 Pipe 패턴:**

```bash
# 정렬 + 중복 제거
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10

# Process 찾기
ps aux | grep nginx | grep -v grep

# Disk 사용량 상위 10개
du -sh /* 2>/dev/null | sort -rh | head -10

# JSON 출력에서 특정 필드 추출 (jq)
curl -s https://api.example.com/data | jq '.items[].name'
```

> **CKA/CKS 직접 연결 — Pipe는 시험의 생명줄:**
> 
> 
> ```bash
> # "CPU 사용량이 가장 높은 Node 이름을 파일에 기록"
> kubectl top nodes --no-headers | sort -k3 -rn | head -1 | awk '{print $1}' \
>   > /opt/course/top-node.txt
> 
> # "Running 상태가 아닌 Pod 목록"
> kubectl get pods -A | grep -v Running | grep -v Completed
> 
> # "특정 Label을 가진 Pod의 이름만 추출"
> kubectl get pods -l app=nginx -o jsonpath='{.items[*].metadata.name}'
> 
> # "모든 Namespace에서 특정 이미지를 사용하는 Pod"
> kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}' \
>   | grep "nginx"
> ```
> 
> **CKS 보안 문제 — Audit Log에서 특정 User 추출:**
> 
> ```bash
> # Audit Log(JSON)에서 forbidden된 요청의 User 추출
> cat /var/log/kubernetes/audit.log | jq 'select(.responseStatus.code == 403) | .user.username' \
>   | sort | uniq > /opt/course/forbidden-users.txt
> ```
> 

## 3-5. tee — 출력을 화면과 파일에 동시에

```bash
# stdout을 화면에 보여주면서 파일에도 저장
ls -la | tee output.txt

# Append 모드
ls -la | tee -a output.txt

# 여러 파일에 동시 저장
echo "hello" | tee file1.txt file2.txt file3.txt

# sudo 권한이 필요한 파일에 쓸 때
echo "new config" | sudo tee /etc/myapp.conf > /dev/null
# sudo echo "..." > /etc/myapp.conf 는 안 됨! (Redirection은 Shell이 처리하므로 sudo가 적용 안 됨)
```

---

# Part 4. 텍스트 처리 도구 — grep, awk, sed, cut, sort, uniq

## 4-1. grep — 패턴 매칭

```bash
# 기본 검색
grep "error" /var/log/messages

# 대소문자 무시
grep -i "error" /var/log/messages

# 줄 번호 포함
grep -n "error" /var/log/messages

# 매칭되지 않는 줄 (반전)
grep -v "info" /var/log/messages

# 재귀 검색 (디렉터리 전체)
grep -r "password" /etc/

# 정규표현식
grep -E "error|warning|critical" /var/log/messages

# 매칭 건수만
grep -c "error" /var/log/messages

# 파일명만 (어떤 파일에 있는지)
grep -l "error" /var/log/*.log

# 앞뒤 Context 포함
grep -A 3 -B 2 "error" /var/log/messages    # 뒤 3줄, 앞 2줄
```

## 4-2. awk — 컬럼 기반 텍스트 처리

awk는 **공백/탭으로 구분된 텍스트에서 특정 컬럼을 추출**하는 데 강력합니다.

```bash
# 특정 컬럼 출력 ($1=첫째, $2=둘째, $NF=마지막)
ps aux | awk '{print $1, $2, $11}'    # USER, PID, COMMAND

# 구분자 지정 (-F)
awk -F: '{print $1, $7}' /etc/passwd   # 사용자명, Shell

# 조건 필터
awk '$3 > 50 {print $0}' data.txt     # 3번째 컬럼이 50보다 큰 줄

# 내장 변수
awk '{print NR, $0}' file.txt         # NR = 줄 번호
awk 'END {print NR}' file.txt         # 총 줄 수

# 합계 계산
awk '{sum += $3} END {print sum}' data.txt
```

> **CKA 직접 연결:**
> 
> 
> ```bash
> # "각 Node의 Pod 수를 세시오"
> kubectl get pods -A -o wide --no-headers | awk '{print $8}' | sort | uniq -c | sort -rn
> # $8 = NODE 컬럼 (출력 형식에 따라 달라질 수 있음)
> 
> # "CPU 사용량이 가장 높은 Pod"
> kubectl top pods --no-headers | sort -k2 -rn | head -1 | awk '{print $1}'
> ```
> 

## 4-3. sed — 스트림 편집기

```bash
# 치환 (첫 번째만)
sed 's/old/new/' file.txt

# 치환 (전체, g=global)
sed 's/old/new/g' file.txt

# 파일 직접 수정 (-i)
sed -i 's/old/new/g' file.txt

# 특정 줄 삭제
sed '5d' file.txt          # 5번째 줄 삭제
sed '/pattern/d' file.txt  # 패턴 매칭 줄 삭제

# 특정 줄 뒤에 삽입
sed '/pattern/a new line' file.txt

# 여러 작업
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt
```

## 4-4. cut, sort, uniq — 데이터 정제

```bash
# cut — 구분자로 필드 추출
cut -d: -f1 /etc/passwd           # 사용자명만
cut -d, -f1,3 data.csv            # CSV에서 1, 3번째 필드

# sort — 정렬
sort file.txt                      # 오름차순
sort -r file.txt                   # 내림차순
sort -n file.txt                   # 숫자 기준
sort -k2 file.txt                  # 2번째 컬럼 기준
sort -t: -k3 -n /etc/passwd        # 구분자 :, 3번째 필드(UID), 숫자

# uniq — 연속 중복 제거 (sort와 함께 사용)
sort file.txt | uniq               # 중복 제거
sort file.txt | uniq -c            # 중복 횟수 포함
sort file.txt | uniq -d            # 중복된 줄만 출력
```

---

# Part 5. 조건문

## 5-1. if 문 기본 구조

```bash
if [ 조건 ]; then
    명령어
elif [ 조건 ]; then
    명령어
else
    명령어
fi
```

**`[` 와 `]` 앞뒤에 반드시 공백!** `[`는 사실 `test` 명령어의 Alias이기 때문입니다.

```bash
[ -f /etc/passwd ]    # ✅ 올바름
[-f /etc/passwd]      # ❌ 에러! '-f'라는 명령어를 찾으려 함
```

## 5-2. 조건식 종류

### 파일 테스트

| 조건 | 의미 |
| --- | --- |
| `-f file` | 일반 파일이 존재하는가 |
| `-d dir` | 디렉터리가 존재하는가 |
| `-e path` | 경로가 존재하는가 (파일/디렉터리 모두) |
| `-r file` | 읽기 가능한가 |
| `-w file` | 쓰기 가능한가 |
| `-x file` | 실행 가능한가 |
| `-s file` | 파일 크기가 0보다 큰가 (비어있지 않은가) |
| `-L file` | Symbolic Link인가 |

### 문자열 비교

| 조건 | 의미 |
| --- | --- |
| `"$a" = "$b"` | 같은가 (POSIX. `[` 에서 사용) |
| `"$a" == "$b"` | 같은가 (Bash. `[[` 에서 사용) |
| `"$a" != "$b"` | 다른가 |
| `-z "$a"` | 비어있는가 (Zero length) |
| `-n "$a"` | 비어있지 않은가 (Non-zero) |

### 숫자 비교

| 조건 | 의미 |
| --- | --- |
| `$a -eq $b` | Equal |
| `$a -ne $b` | Not Equal |
| `$a -gt $b` | Greater Than |
| `$a -ge $b` | Greater or Equal |
| `$a -lt $b` | Less Than |
| `$a -le $b` | Less or Equal |

### 논리 연산

| `[ ]` 문법 | `[[ ]]` 문법 | 의미 |
| --- | --- | --- |
| `-a` | `&&` | AND |
| `-o` | `||` | OR |
| `!` | `!` | NOT |

## 5-3. `[ ]` vs `[[ ]]`

`[[ ]]`는 Bash 전용 확장으로, `[ ]`보다 안전하고 기능이 많습니다.

```bash
# [ ] — POSIX 호환. 변수 따옴표 필수
if [ -f "$FILE" ] && [ "$COUNT" -gt 0 ]; then
    echo "OK"
fi

# [[ ]] — Bash 전용. 따옴표 없어도 Word Splitting 안 됨. 패턴 매칭 지원
if [[ -f $FILE && $COUNT -gt 0 ]]; then
    echo "OK"
fi

# 패턴 매칭 ([[ ]] 전용)
if [[ "$HOSTNAME" == *prod* ]]; then
    echo "Production server!"
fi

# 정규표현식 매칭 ([[ ]] 전용)
if [[ "$EMAIL" =~ ^[a-zA-Z]+@[a-zA-Z]+\.[a-zA-Z]+$ ]]; then
    echo "Valid email"
fi
```

## 5-4. 실전 조건문 예시

```bash
#!/bin/bash
# 서비스 상태 확인 및 재시작

SERVICE="nginx"

if systemctl is-active --quiet $SERVICE; then
    echo "$SERVICE is running"
else
    echo "$SERVICE is down! Restarting..."
    sudo systemctl restart $SERVICE
    if [ $? -eq 0 ]; then
        echo "$SERVICE restarted successfully"
    else
        echo "CRITICAL: Failed to restart $SERVICE" >&2
        exit 1
    fi
fi
```

---

# Part 6. 반복문

## 6-1. for 문

```bash
# 리스트 기반
for fruit in apple banana cherry; do
    echo "I like $fruit"
done

# 범위 기반
for i in {1..10}; do
    echo "Number: $i"
done

# C 스타일
for ((i=0; i<10; i++)); do
    echo "Index: $i"
done

# 명령어 결과를 순회
for file in /etc/*.conf; do
    echo "Config file: $file"
done

# 파일의 각 줄을 순회
for line in $(cat hosts.txt); do
    echo "Host: $line"
done
```

> **CKA/CKS 직접 연결 — for문은 시험의 핵심 무기:**
> 
> 
> ```bash
> # "모든 Node의 kubelet 상태를 확인하시오"
> for node in $(kubectl get nodes -o name); do
>     echo "=== $node ==="
>     kubectl describe $node | grep -i "kubelet"
> done
> 
> # "모든 Namespace의 Pod 수를 세시오"
> for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
>     count=$(kubectl get pods -n $ns --no-headers 2>/dev/null | wc -l)
>     echo "$ns: $count pods"
> done
> 
> # "kube-system Namespace의 모든 Pod 로그에서 error 찾기"
> for pod in $(kubectl get pods -n kube-system -o name); do
>     echo "=== $pod ==="
>     kubectl logs $pod -n kube-system 2>/dev/null | grep -i error | tail -5
> done
> 
> # CKS: "모든 ServiceAccount의 automountServiceAccountToken 확인"
> for sa in $(kubectl get sa -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}'); do
>     ns=$(echo $sa | cut -d/ -f1)
>     name=$(echo $sa | cut -d/ -f2)
>     auto=$(kubectl get sa $name -n $ns -o jsonpath='{.automountServiceAccountToken}')
>     echo "$sa: automount=$auto"
> done
> ```
> 

## 6-2. while 문

```bash
# 기본 while
count=0
while [ $count -lt 5 ]; do
    echo "Count: $count"
    ((count++))
done

# 파일을 줄 단위로 읽기 (for보다 안전 — 공백 처리)
while IFS= read -r line; do
    echo "Line: $line"
done < /etc/passwd

# 무한 루프 (Health Check 등)
while true; do
    if curl -s -o /dev/null -w "%{http_code}" http://localhost:8080 | grep -q "200"; then
        echo "$(date): Service is UP"
    else
        echo "$(date): Service is DOWN!"
    fi
    sleep 30
done

# 조건부 무한 루프 — Pod 상태 대기
while ! kubectl get pod my-pod -o jsonpath='{.status.phase}' 2>/dev/null | grep -q Running; do
    echo "Waiting for pod..."
    sleep 5
done
echo "Pod is running!"
```

## 6-3. until 문 — while의 반대

```bash
# 조건이 참이 될 때까지 반복
count=0
until [ $count -ge 5 ]; do
    echo "Count: $count"
    ((count++))
done
```

## 6-4. break과 continue

```bash
# break — 루프 탈출
for i in {1..100}; do
    if [ $i -eq 10 ]; then
        break    # 10에서 루프 종료
    fi
    echo $i
done

# continue — 현재 반복 건너뛰기
for i in {1..10}; do
    if [ $((i % 2)) -eq 0 ]; then
        continue    # 짝수 건너뛰기
    fi
    echo $i    # 홀수만 출력
done
```

---

# Part 7. 함수

```bash
# 함수 정의
backup_logs() {
    local LOG_DIR="$1"      # local — 함수 내부에서만 유효한 변수
    local BACKUP_DIR="$2"
    local DATE=$(date +%Y%m%d)

    if [ ! -d "$LOG_DIR" ]; then
        echo "ERROR: $LOG_DIR not found" >&2
        return 1    # 함수의 Exit Status (exit는 스크립트 전체 종료)
    fi

    tar czf "${BACKUP_DIR}/logs_${DATE}.tar.gz" "$LOG_DIR"
    echo "Backup created: ${BACKUP_DIR}/logs_${DATE}.tar.gz"
    return 0
}

# 함수 호출
backup_logs /var/log/myapp /tmp/backups
if [ $? -eq 0 ]; then
    echo "Backup successful"
fi
```

**`return` vs `exit`:**

- `return N` — 함수에서 빠져나감. 스크립트는 계속 실행.
- `exit N` — 스크립트 전체를 종료.

**`local` 키워드:** 함수 안에서 `local`로 선언한 변수는 함수 내부에서만 유효. `local` 없으면 전역 변수가 됨.

---

# Part 8. 실전 자동화 스크립트

## 8-1. 시스템 상태 점검 스크립트

```bash
#!/bin/bash
# system-check.sh — 시스템 상태 종합 점검

LOG_FILE="/var/log/system-check.log"
DATE=$(date "+%Y-%m-%d %H:%M:%S")

log() {
    echo "[$DATE] $1" | tee -a "$LOG_FILE"
}

log "========== System Check Start =========="

# CPU Load
LOAD=$(cat /proc/loadavg | awk '{print $1}')
CPU_COUNT=$(nproc)
if (( $(echo "$LOAD > $CPU_COUNT" | bc -l) )); then
    log "WARNING: High CPU Load: $LOAD (cores: $CPU_COUNT)"
else
    log "OK: CPU Load: $LOAD"
fi

# Memory
MEM_AVAILABLE=$(free -m | awk '/Mem:/ {print $7}')
MEM_TOTAL=$(free -m | awk '/Mem:/ {print $2}')
MEM_PERCENT=$((100 - (MEM_AVAILABLE * 100 / MEM_TOTAL)))
if [ $MEM_PERCENT -gt 90 ]; then
    log "CRITICAL: Memory usage ${MEM_PERCENT}%"
elif [ $MEM_PERCENT -gt 70 ]; then
    log "WARNING: Memory usage ${MEM_PERCENT}%"
else
    log "OK: Memory usage ${MEM_PERCENT}%"
fi

# Disk
while IFS= read -r line; do
    USAGE=$(echo "$line" | awk '{print $5}' | tr -d '%')
    MOUNT=$(echo "$line" | awk '{print $6}')
    if [ "$USAGE" -gt 90 ]; then
        log "CRITICAL: Disk $MOUNT at ${USAGE}%"
    elif [ "$USAGE" -gt 70 ]; then
        log "WARNING: Disk $MOUNT at ${USAGE}%"
    fi
done < <(df -h | grep -E '^/dev/')

# Failed Services
FAILED=$(systemctl --failed --no-legend | wc -l)
if [ "$FAILED" -gt 0 ]; then
    log "WARNING: $FAILED failed service(s):"
    systemctl --failed --no-legend | while read -r line; do
        log "  - $line"
    done
else
    log "OK: No failed services"
fi

# Zombie Processes
ZOMBIES=$(ps aux | awk '$8=="Z" {count++} END {print count+0}')
if [ "$ZOMBIES" -gt 0 ]; then
    log "WARNING: $ZOMBIES zombie process(es)"
fi

log "========== System Check Complete =========="
```

## 8-2. 로그 정리 자동화

```bash
#!/bin/bash
# log-cleanup.sh — 오래된 로그 파일 정리

LOG_DIRS=("/var/log/myapp" "/var/log/nginx" "/tmp")
RETENTION_DAYS=30
DRY_RUN=false

# 인자 처리
while getopts "d:n" opt; do
    case $opt in
        d) RETENTION_DAYS="$OPTARG" ;;
        n) DRY_RUN=true ;;
        *) echo "Usage: $0 [-d days] [-n dry-run]"; exit 1 ;;
    esac
done

for dir in "${LOG_DIRS[@]}"; do
    if [ ! -d "$dir" ]; then
        echo "SKIP: $dir not found"
        continue
    fi

    echo "Processing: $dir (retention: ${RETENTION_DAYS} days)"

    FILES=$(find "$dir" -type f -name "*.log" -mtime +${RETENTION_DAYS})
    COUNT=$(echo "$FILES" | grep -c .)

    if [ "$COUNT" -gt 0 ]; then
        if $DRY_RUN; then
            echo "  [DRY-RUN] Would delete $COUNT file(s):"
            echo "$FILES" | while read -r f; do echo "    $f"; done
        else
            echo "$FILES" | xargs rm -f
            echo "  Deleted $COUNT file(s)"
        fi
    else
        echo "  No files older than ${RETENTION_DAYS} days"
    fi
done
```

> **AWS SSM 연결:**
SAA 시험: "수백 대 EC2에 SSH 없이 패치/스크립트를 실행하려면?"
→ **AWS Systems Manager (SSM) Run Command**. SSM Agent가 EC2 내부에서 위와 같은 Shell Script를 실행하고, stdout/stderr를 수집하여 S3/CloudWatch에 기록합니다.
→ 위 `log-cleanup.sh`를 SSM Run Command로 모든 Instance에 일괄 실행하는 것이 실무 패턴.
> 

## 8-3. K8s 관련 자동화

```bash
#!/bin/bash
# k8s-namespace-report.sh — Namespace별 리소스 현황

echo "=== Kubernetes Namespace Resource Report ==="
echo "Date: $(date)"
echo ""

printf "%-20s %-10s %-10s %-10s\n" "NAMESPACE" "PODS" "SERVICES" "SECRETS"
printf "%-20s %-10s %-10s %-10s\n" "---------" "----" "--------" "-------"

for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
    pods=$(kubectl get pods -n "$ns" --no-headers 2>/dev/null | wc -l)
    svcs=$(kubectl get svc -n "$ns" --no-headers 2>/dev/null | wc -l)
    secrets=$(kubectl get secrets -n "$ns" --no-headers 2>/dev/null | wc -l)
    printf "%-20s %-10s %-10s %-10s\n" "$ns" "$pods" "$svcs" "$secrets"
done
```

---

# Part 9. 디버깅과 Best Practice

## 9-1. 디버깅

```bash
# 실행되는 명령어를 한 줄씩 출력 (Trace Mode)
bash -x script.sh

# 스크립트 내에서 부분적으로 Trace 활성화
set -x    # Trace ON
# ... 디버깅할 코드 ...
set +x    # Trace OFF

# 안전한 스크립트를 위한 set 옵션들
#!/bin/bash
set -euo pipefail
# -e : 에러 발생 시 즉시 종료 (Exit Code가 non-zero면)
# -u : 미정의 변수 사용 시 에러 (오타 방지)
# -o pipefail : Pipe 중간 명령이 실패해도 감지
```

**`set -euo pipefail`이 왜 중요한가:**

기본적으로 Bash는 에러가 발생해도 **다음 줄을 계속 실행**합니다. 이것은 위험합니다:

```bash
#!/bin/bash
cd /important/directory    # 이 디렉터리가 없으면? cd 실패 → 현재 위치 유지
rm -rf *                   # 현재 디렉터리(/)의 모든 파일 삭제!

# set -e가 있으면 cd 실패 시 스크립트가 즉시 종료되어 rm은 실행되지 않음
```

> **Terraform 연결:** Terraform Provisioner의 Shell Script에서도 `set -euo pipefail`을 쓰는 것이 Best Practice. 중간 명령이 실패하면 Provisioner 전체를 실패로 처리해야 하므로.
> 

## 9-2. Best Practice 정리

| 규칙 | 이유 |
| --- | --- |
| 항상 Shebang 포함 (`#!/bin/bash`) | 인터프리터를 명확히 |
| `set -euo pipefail` 사용 | 에러 조기 감지 |
| 변수는 반드시 `"$VAR"` (큰따옴표) | Word Splitting 방지 |
| `[[ ]]` 사용 (bash) | `[ ]`보다 안전 |
| 함수 내 변수는 `local` | 전역 오염 방지 |
| 에러 메시지는 `>&2`로 stderr | stdout과 분리 |
| Exit Code 명시적 사용 | 호출자에게 성공/실패 전달 |
| 주석 충분히 | 3개월 후의 자신을 위해 |
| ShellCheck 사용 | 정적 분석으로 버그 예방 |

```bash
# ShellCheck 설치 및 사용
sudo dnf install ShellCheck -y
shellcheck myscript.sh
```

---

# Part 10. 자격증 연결 정리

## CKA/CKS (100% 연관)

| Shell 기술 | CKA/CKS 적용 |
| --- | --- |
| `>` / `>>` Redirection | YAML 매니페스트 생성, 로그 추출 결과 저장 |
| `|` Pipe | kubectl 출력 필터링 (grep, awk, sort, head) |
| `for` 반복문 | 모든 Node/Pod/Namespace 순회 작업 |
| `grep` / `awk` | 로그 분석, 특정 필드 추출 |
| `$()` Command Substitution | kubectl 결과를 변수에 담아 활용 |
| alias + export | 시험 시작 시 `k=kubectl` 세팅 |

## Terraform Associate (60% 연관)

| Shell 기술 | Terraform 적용 |
| --- | --- |
| `export` 환경 변수 | `TF_VAR_*` 변수 주입, `TF_LOG` 디버깅 |
| Exit Code (`$?`) | Provisioner 성공/실패 판단 |
| `set -euo pipefail` | Provisioner Script 안전성 |
| Here Document | `local-exec` 내 Multi-line Script |

## AWS SAA (30% 연관)

| Shell 기술 | SAA 적용 |
| --- | --- |
| Shebang (`#!/bin/bash`) | EC2 User Data Script 필수 첫 줄 |
| 기본 스크립트 작성 | User Data로 앱 설치/설정 자동화 |
| stdout/stderr | SSM Run Command 결과 수집 개념 |
| 환경 변수 | EC2 Instance Profile → 앱에 자격증명 전달 개념 |