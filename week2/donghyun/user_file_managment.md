## 1. 계정 및 그룹 관리 (User & Group Management)

리눅스 시스템에서 계정은 단순한 텍스트 이름이 아닌, 커널이 인식하는 **UID(User ID)**와 **GID(Group ID)**라는 고유 숫자로 관리되는 데이터의 집합입니다.

### 1.1 실습 시나리오 및 명령어

시스템 운영을 위한 전용 그룹과 사용자를 생성하고 관리하는 과정은 다음과 같습니다.

- **그룹 생성:** `sudo groupadd sre_team` (SRE 팀 전용 그룹 생성)
- **사용자 생성:** `sudo useradd -m -s /bin/bash konkuk_sre`
    - `m`: 사용자의 홈 디렉토리(`/home/konkuk_sre`)를 자동으로 생성합니다.
    - `s`: 사용자가 사용할 기본 쉘을 `/bin/bash`로 지정합니다.
- **그룹 할당:** `sudo usermod -aG sre_team konkuk_sre` (`aG`: 기존 그룹을 유지하며 새 그룹에 추가)
- **비밀번호 설정:** `sudo passwd konkuk_sre`
- **계정 삭제:** `sudo userdel -r konkuk_sre` (`r`: 홈 디렉토리 및 메일 박스까지 모두 삭제)

### 1.2 핵심 설정 파일 분석

명령어 실행 결과는 시스템의 주요 텍스트 파일에 반영되며, 이 데이터 구조를 이해해야 합니다.

| **확인 파일** | **데이터 예시** | **분석 포인트** |
| --- | --- | --- |
| **`/etc/passwd`** | `konkuk_sre:x:1001:1002::/home/konkuk_sre:/bin/bash` | 사용자 이름, UID, 기본 GID, 홈 디렉토리, 쉘 정보 등 7개 필드 구조 이해 |
| **`/etc/shadow`** | `konkuk_sre:$6$hG...:19432:0:99999:7:::` | 암호화된 비밀번호 해시값 및 패스워드 만료 정책(보안 핵심) 관리 |
| **`/etc/group`** | `sre_team:x:1002:konkuk_sre` | 그룹 이름, GID 및 해당 그룹에 속한 보조 사용자 목록 확인 |

```bash
mr8356@mr8356:~$ sudo groupadd sre_team
[sudo] password for mr8356:

mr8356@mr8356:~$ sudo useradd -m -s /bin/bash konkuk_sre

mr8356@mr8356:~$ sudo usermod -aG sre_team konkuk_sre

mr8356@mr8356:~$ sudo passwd konkuk_sre
New password:
BAD PASSWORD: The password is shorter than 8 characters
Retype new password:
passwd: password updated successfully

mr8356@mr8356:~$ sudo cat /etc/passwd
konkuk_sre:x:1001:1002::/home/konkuk_sre:/bin/bash

mr8356@mr8356:~$ sudo cat /etc/shadow | grep -i konkuk_sre
konkuk_sre:$y$j9T$yxZN9X7YD.VhfpH6dQMOB1$WO8E.6QLPZ9FbNWwRQosTktvvxzOWNHnWRlIOUCr3b0:20527:0:99999:7:::

mr8356@mr8356:~$ sudo cat /etc/group | grep -i konkuk_sre
sre_team:x:1001:konkuk_sre
konkuk_sre:x:1002:
```

---

## 2. 파일 권한 및 소유권 관리 (File Permissions)

리눅스의 모든 개체는 파일로 취급되며, 각 파일은 소유자(Owner), 그룹(Group), 기타 사용자(Others)에 대해 독립적인 접근 비트(Bit)를 가집니다.

### 2.1 권한 변경 및 비트 해석

파일의 접근 권한은 8진수(Numeric) 또는 기호(Symbolic) 방식으로 제어할 수 있습니다.

- **소유권 변경:** `sudo chown konkuk_sre:sre_team mission_check.txt`
- **8진수 권한 설정:** `chmod 750 mission_check.txt` (소유자: rwx, 그룹: r-x, 기타: ---)
- **기호 기반 권한 추가:** `chmod g+w mission_check.txt` (그룹에게 쓰기 권한 추가)

```bash
mr8356@mr8356:~$ sudo cat /etc/group | grep -i konkuk_sre
sre_team:x:1001:konkuk_sre
konkuk_sre:x:1002:

mr8356@mr8356:~$ touch mission_check.txt

mr8356@mr8356:~$ sudo chown konkuk_sre:sre_team mission_check.txt

mr8356@mr8356:~$ sudo chmod 750 mission_check.txt

mr8356@mr8356:~$ sudo chmod g+w mission_check.txt

mr8356@mr8356:~$ ls -l | grep mission
-rwxrwx---. 1 konkuk_sre sre_team  0 Mar 15 13:09 mission_check.txt
```

---

## 3. 심화 개념 (Advanced Concepts)

운영 환경에서 권한 오남용을 방지하고 보안을 강화하기 위해 필수적으로 이해해야 할 개념입니다.

### 3.1 umask (기본 권한 마스크)

새로운 파일이나 디렉토리가 생성될 때 부여되는 초기 권한의 결정 방식입니다. 시스템 기본값에서 `umask` 값을 제외한 결과가 실제 권한이 됩니다.

### 3.2 특수 권한 (Special Permissions)

일반적인 권한 체계로 해결할 수 없는 특정 상황을 지원합니다.

1. **SUID (Set User ID, 4000):** 실행 시 파일 소유자의 권한으로 프로세스가 동작합니다. (예: `/usr/bin/passwd`)
2. **SGID (Set Group ID, 2000):** 실행 시 파일 그룹 권한으로 동작하거나, 디렉토리 내 생성된 파일이 해당 그룹 소유를 상속받습니다.
3. **Sticky Bit (1000):** 공용 디렉토리에서 모든 사용자가 파일을 생성할 수 있으나, 삭제는 본인의 파일만 가능하도록 제한합니다. (예: `/tmp`)