# Rocky Linux 스터디 — 3주차
## 파일 시스템 및 스토리지 관리

## 전체 스택 구조

모든 개념의 위치를 먼저 잡고 시작합니다.

```
사용자 프로그램
      ↓ open(), read(), write() 시스템 콜
VFS (Virtual File System)         ← 모든 파일시스템 추상화
      ↓
XFS 드라이버 | ext4 드라이버 | tmpfs ...
      ↓
블록 장치 드라이버
      ↓
LVM (dm-mapper)                   ← 논리→물리 주소 변환
      ↓
물리 디스크 드라이버
      ↓
실제 디스크 (/dev/sdb1, /dev/sdc ...)
```

---

## 1. 디스크 · 블록 장치 · 파티션 구조

### 1-1. 블록 장치란

> 출처: [RHEL 10 Managing storage devices](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_storage_devices)

Linux에서 디스크는 `/dev/sda` 같은 **장치 파일**로 표현됩니다.

```bash
ls -l /dev/sda
# brw-rw---- 1 root disk 8, 0 /dev/sda
# ↑
# b = block device
```

블록 장치의 특성:

```
데이터를 블록(기본 4KB) 단위로만 읽고 씀
→ "3번째 바이트만 줘" 불가
→ 해당 바이트가 속한 블록 전체를 읽어옴

디스크 내부:
┌──────┬──────┬──────┬──────┬──────┐
│블록 0│블록 1│블록 2│블록 3│블록 4│ ...
└──────┴──────┴──────┴──────┴──────┘
  4KB    4KB    4KB    4KB    4KB
```

파일시스템 없는 날것의 블록 장치 상태:

```
포맷 전 /dev/sdb1:
블록0:  00 00 00 00 ...  (쓰레기값 또는 0)
블록1:  00 00 00 00 ...
→ superblock 없음
→ 커널이 읽어도 "이게 뭔지 모르겠다"
→ mount 시도 시 "wrong fs type" 오류
→ 파일/디렉토리 개념 없음
→ 접근 불가
```

블록 장치 vs 문자 장치:

```
블록 장치 (b): /dev/sda, /dev/sdb1
  → 블록 단위, 임의 위치 접근(Random Access) 가능
  → 디스크, SSD, USB

문자 장치 (c): /dev/tty, /dev/null
  → 바이트 단위, 순서대로만
  → 터미널, 마우스, 키보드
```

### 1-2. 블록 장치 내부 — 파일시스템 구조

`mkfs.xfs` 실행 후 XFS 내부 구조:

```
/dev/sdb1 내부 (XFS 포맷 후):

┌────────────────────────────────────────┐
│  Superblock (블록 0)                   │
│  - "이건 XFS입니다" 식별 정보          │
│  - 파일시스템 전체 크기, 블록 크기     │
│  - UUID, AG 개수, 여유 블록 수         │
│  - 마지막 마운트 시각, 상태(clean/dirty)│
├────────────────────────────────────────┤
│  Allocation Group 0 (AG0)              │
│  ┌──────────────────────────────────┐  │
│  │ AG 슈퍼블록  ← AG 메타데이터    │  │
│  │ inode B-tree ← 파일 찾기 인덱스 │  │
│  │ 여유공간 B-tree ← 빈 블록 인덱스│  │
│  │ inode 데이터 ← 실제 inode들     │  │
│  │ 파일 데이터 블록들               │  │
│  └──────────────────────────────────┘  │
├────────────────────────────────────────┤
│  Allocation Group 1 (AG1)              │
│  └── AG0와 동일한 구조 반복           │
├────────────────────────────────────────┤
│  Allocation Group 2, 3 ...            │
└────────────────────────────────────────┘
```

**AG (Allocation Group):**

파티션과 AG는 다른 개념입니다.

```
파티션 = 사람이 만드는 외부 경계 (parted, fdisk)
AG     = XFS가 내부적으로 자동 생성하는 관리 단위

파티션 안에 AG가 있는 구조:
/dev/sdb1 (파티션)
  ├── AG0 (XFS 내부)
  ├── AG1
  ├── AG2
  └── AG3
```

AG를 나누는 이유 — 병렬 처리:

```
AG 없이:
CPU1: 파일 A 쓰기 ─┐
CPU2: 파일 B 쓰기  ├→ 순서 기다림 (직렬)
CPU3: 파일 C 쓰기 ─┘

AG 있으면:
CPU1: AG0에 파일 A 쓰기 → 독립적
CPU2: AG1에 파일 B 쓰기 → 독립적
CPU3: AG2에 파일 C 쓰기 → 독립적
→ 동시 처리 (병렬)
```

단, 같은 AG 내 파일들은 여전히 직렬 처리됩니다. XFS는 새 파일을 쓸 때 프로세스마다 다른 AG에 배치하려고 시도해서 자연스럽게 분산시킵니다.

**inode:**

```
파일 하나당 inode 하나
파일 이름을 제외한 모든 파일 정보가 여기에:

inode:
  파일 타입 (일반/디렉토리/링크)
  소유자 UID, GID
  권한 비트 (rwxr-xr-x)
  파일 크기
  생성/수정/접근 시각
  → 실제 데이터 블록 위치 포인터
```

파일 이름은 inode에 없습니다. 디렉토리가 가집니다:

```
디렉토리의 실체 = "이름 → inode 번호" 매핑 테이블

/data/ 디렉토리 내부:
┌──────────────┬──────────┐
│  파일 이름   │ inode 번호│
├──────────────┼──────────┤
│  report.txt  │   12345  │
│  images/     │   12346  │
└──────────────┴──────────┘
```

하드링크가 가능한 이유: 서로 다른 이름이 같은 inode 번호를 가리킬 수 있기 때문입니다.

### 1-3. 전체 스토리지 구조

```
물리 디스크
  └── 파티션 테이블 (MBR 또는 GPT)
        ├── 파티션 1 (/dev/sda1) → XFS → /boot
        ├── 파티션 2 (/dev/sda2) → LVM PV
        │     └── Volume Group (vg0)
        │           ├── Logical Volume (lv_root) → XFS → /
        │           ├── Logical Volume (lv_home) → XFS → /home
        │           └── Logical Volume (lv_swap) → swap
        └── 파티션 3 (/dev/sda3) → XFS → /data
```

### 1-4. 디스크 장치 명명 규칙

> 출처: [RHEL 10 Disk partitions — 3.8 Partition naming scheme](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_storage_devices/disk-partitions#partition-naming-scheme)

```
/dev/sda    ← 첫 번째 SATA/SCSI/SSD 디스크
/dev/sdb    ← 두 번째 디스크
/dev/sda1   ← sda의 첫 번째 파티션
/dev/sda2   ← sda의 두 번째 파티션

/dev/nvme0n1    ← 첫 번째 NVMe 디스크
/dev/nvme0n1p1  ← NVMe 디스크의 첫 번째 파티션
                   (숫자로 끝나는 디스크는 p 추가)

/dev/vda    ← 가상머신(KVM) 환경의 디스크
```

### 1-5. 파티션 테이블 종류 — MBR vs GPT

> 출처: [RHEL 10 Disk partitions — Table 3.1](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_storage_devices/disk-partitions#comparison-of-partition-table-types)

| 항목 | MBR | GPT |
|------|-----|-----|
| 최대 디스크 크기 | **2 TiB** | **8 ZiB** (512b 섹터) / **64 ZiB** (4k 섹터) |
| 최대 파티션 수 | 주 4개, 또는 주 3개 + 확장 1개(논리 56개) | **128개** (기본값, 공간 추가 시 확장 가능) |
| 방식 | 구형 BIOS 기반 | 최신 UEFI 기반 |
| 파티션 식별 | 16진수 타입 코드 | GUID (전역 고유 식별자) |
| 안정성 | 낮음 (파티션 테이블 1개) | 높음 (보조 GPT 헤더로 백업) |

MBR 파티션 구조 (2TB 이하 구형 환경):

```
주 파티션은 최대 4개 한계 → 확장 파티션으로 극복

/dev/sda1  Primary   (주 파티션)
/dev/sda2  Primary   (주 파티션)
/dev/sda3  Primary   (주 파티션)
/dev/sda4  Extended  (확장 파티션 — 컨테이너 역할)
  └─ /dev/sda5  Logical
  └─ /dev/sda6  Logical

RHEL 10 기준 하나의 디스크에 최대 60개 파티션 접근 가능
```

GPT 파티션 구조 (현대 환경 — 권장):

```
/dev/sda1  EFI System Partition (ESP) — UEFI 부팅용
/dev/sda2  /boot 파티션
/dev/sda3  LVM PV 파티션
...최대 128개 주 파티션 생성 가능 (확장 파티션 불필요)
```

### 1-6. 마운트 — 블록 장치를 경로에 연결

```
마운트 전:
  /dev/sdb1 → 블록 장치 (경로로 접근 불가)

마운트 후:
  mount /dev/sdb1 /data 실행
  /data/ 경로 접근 → /dev/sdb1의 내용

마운트 = 경로 요청을 특정 블록 장치로 리다이렉트하는 것
```

---

## 2. 파티션 생성 및 관리

> 출처: [RHEL 10 Getting started with partitions](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_storage_devices/getting-started-with-partitions)

### 2-1. 왜 파티션을 나누는가

```
파티션 안 나눈 경우:
/ 가 꽉 차면 → OS 전체 마비
로그 파일 폭증 → 전체가 영향받음

파티션 나눈 경우:
/        → OS 영역 (50GB)
/home    → 사용자 파일 (200GB)
/var/log → 로그 파일 (20GB)
→ 로그가 폭증해도 /home, / 는 멀쩡함
```

### 2-2. 디스크 상태 확인

```bash
lsblk
# NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
# sda      8:0    0   50G  0 disk
# ├─sda1   8:1    0    1G  0 part /boot
# └─sda3   8:3    0   48G  0 part /
# sdb      8:16   0  100G  0 disk  ← 새 디스크 (비어있음)

lsblk -f     # 파일시스템 정보 포함
fdisk -l     # 파티션 테이블 상세
```

### 2-3. parted — RHEL 10 권장 도구

```
parted vs fdisk:

parted:
  GPT/MBR 모두 지원
  변경사항 즉시 적용 (저장 명령 없음) → 실수 주의
  2TB 이상 디스크 처리 가능
  RHEL 10 공식 권장

fdisk:
  주로 MBR 환경
  w 입력해야 저장됨 → 실수 방지 가능
```

#### GPT 환경 파티션 생성 (RHEL 10 권장)

```bash
parted /dev/sdb

(parted) mklabel gpt              # GPT 테이블 생성 (즉시 적용, 기존 데이터 삭제!)
(parted) mkpart "boot" xfs 1MiB 2GiB
(parted) mkpart "root" xfs 2GiB 52GiB
(parted) mkpart "home" xfs 52GiB 100%
(parted) set 1 esp on             # EFI 시스템 파티션 플래그
(parted) set 2 lvm on             # LVM 플래그
(parted) print                    # 결과 확인
(parted) quit
```

GPT에서 `mkpart` 형식:

```
mkpart "이름" 파일시스템힌트 시작 끝
        │          │           │    └── 끝 위치
        │          │           └── 시작 위치
        │          └── 파티션 GUID 타입 설정에만 영향
        │              (실제 포맷은 나중에 mkfs로 따로)
        └── 파티션 이름 (GPT 전용, MBR에서는 primary/extended/logical 키워드 사용)

⚠️ GPT에서는 'primary' 키워드 사용 안 함
   GPT는 모든 파티션이 주 파티션이라 타입 구분 없음
```

1MiB부터 시작하는 이유:

```
0번 섹터 = GPT 헤더 영역
1MiB = 2048번 섹터부터 시작
→ SSD, NVMe 정렬(alignment) 맞추기 위함
→ 정렬 안 맞으면 성능 저하
```

#### MBR 환경 파티션 생성 (구형)

```bash
parted /dev/sdb
(parted) mklabel msdos
(parted) mkpart primary xfs 1MiB 50GiB
(parted) mkpart primary linux-swap 50GiB 52GiB
(parted) mkpart primary xfs 52GiB 100%
```

#### fdisk (MBR 환경)

```bash
fdisk /dev/sdb

n      # 새 파티션 생성
p      # Primary 타입
1      # 파티션 번호
Enter  # 기본 시작 섹터
+50G   # 크기

t      # 타입 변경
8e     # Linux LVM

w      # 저장 (이때 실제 적용)
q      # 저장 없이 종료
```

### 2-4. 커널 반영

```bash
partprobe /dev/sdb    # 커널에 파티션 테이블 변경 알림
lsblk /dev/sdb        # 반영 확인
```

---

## 3. LVM — 논리 볼륨 관리자

> 출처: [RHEL 10 Configuring and managing logical volumes](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_and_managing_logical_volumes)

### 3-1. 왜 LVM이 필요한가

```
파티션 방식의 한계:
/home 꽉 참 → 서비스 중단 → 백업 → 삭제 → 재생성 → 복원
→ 수 시간 작업

LVM 방식:
lvextend -L +100G /dev/vg0/lv_home
xfs_growfs /home
→ 5초, 서비스 중단 없음
```

### 3-2. LVM은 스택의 어디에 있는가

```
사용자 프로그램
      ↓
VFS
      ↓
XFS 드라이버
      ↓
블록 장치 드라이버
      ↓
LVM (dm-mapper)     ← 여기
      ↓
물리 디스크 드라이버
      ↓
실제 디스크

XFS 입장에서 /dev/vg0/lv_home은 그냥 평범한 블록 장치
LVM이 하는 일은 논리 블록 주소 → 물리 블록 주소 변환
파일시스템도, 사용자도 이 변환 과정을 모름
```

Consistent Hashing과 비교:

```
유사점:
  둘 다 "물리적 위치를 몰라도 논리적 이름으로 접근"
  노드/디스크 추가 제거 시 영향 최소화

차이점:
  Consistent Hashing → hash 함수로 자동 배치, 분산 시스템
  LVM → extent 순차 할당, 단일 서버 내 로컬 스토리지

가장 가까운 비유: 가상 메모리
  프로세스 → 가상 주소 → MMU → 물리 주소
  파일시스템 → LV 주소 → LVM → 실제 PV 주소
```

### 3-3. 3계층 구조

```
물리 디스크
      ↓ pvcreate
PV (Physical Volume)     ← LVM용으로 초기화된 파티션/디스크
      ↓ vgcreate
VG (Volume Group)        ← PV들을 묶은 통합 공간 풀
      ↓ lvcreate
LV (Logical Volume)      ← VG에서 할당한 가상 블록 장치
      ↓ mkfs + mount
파일시스템 + 마운트포인트
```

#### PV (Physical Volume) — 재료

```
LVM이 사용할 수 있도록 초기화된 파티션 또는 디스크

내부 구조:
┌────────────────────────────────┐
│  PV 헤더 (LVM 메타데이터)      │
│  PE PE PE PE PE PE PE PE ...   │
│  (Physical Extent — 기본 4MiB) │
└────────────────────────────────┘

PE = LVM의 최소 할당 단위
100GB 디스크 → 100GB / 4MiB = 25,600개 PE
```

#### VG (Volume Group) — 창고

```
여러 PV를 하나로 묶은 통합 공간

/dev/sdb1 (100GB = 25,600 PE) ─┐
/dev/sdc  (200GB = 51,200 PE) ─┼→ vg0 (300GB = 76,800 PE)

VG는 PE들의 풀(pool)
어느 PV에 실제로 저장되는지는 LVM이 알아서 관리
```

#### LV (Logical Volume) — 가상 파티션

```
VG의 PE를 할당받아 만든 가상 블록 장치

lvcreate -L 50G -n lv_home vg0
→ vg0에서 50GB(12,800 PE) 할당
→ /dev/vg0/lv_home 가상 장치 생성

LVM 매핑 테이블:
lv_home 블록 0~25599     → /dev/sdb1 블록 0~25599
lv_home 블록 25600~51199 → /dev/sdc  블록 0~25599
→ XFS는 연속된 단일 디스크인 줄 앎
```

#### 전체 구조 시각화

```
/dev/sdb (100GB)          /dev/sdc (200GB)
     ↓ pvcreate                ↓ pvcreate
  PV (25,600 PE)           PV (51,200 PE)
        └──────────┬──────────┘
                   ↓ vgcreate
            VG: vg0 (76,800 PE)
                   ↓ lvcreate
      ┌────────────┼────────────┐
      ↓            ↓            ↓
  lv_root       lv_home      lv_swap
   (50GB)       (200GB)       (8GB)
      ↓            ↓            ↓
  mkfs.xfs     mkfs.xfs     mkswap
      ↓            ↓            ↓
   / 마운트   /home 마운트  swap 사용
```

### 3-4. LVM 구성 실습

```bash
# 1. PV 생성
pvcreate /dev/sdb1 /dev/sdc
pvs                                   # 확인

# 2. VG 생성
vgcreate vg0 /dev/sdb1 /dev/sdc
vgs                                   # 확인

# 3. LV 생성
lvcreate -L 50G  -n lv_root vg0
lvcreate -L 200G -n lv_home vg0
lvcreate -l 100%FREE -n lv_data vg0  # 남은 공간 전부
lvs                                   # 확인

# 4. 파일시스템 생성 후 마운트
mkfs.xfs /dev/vg0/lv_root
mkfs.xfs /dev/vg0/lv_home
mount /dev/vg0/lv_root /
mount /dev/vg0/lv_home /home

# 5. LV 확장 (온라인 가능)
lvextend -L +100G /dev/vg0/lv_home
xfs_growfs /home                      # XFS 파일시스템도 확장
```

### 3-5. LVM 주요 기능

```
온라인 확장: 서비스 중단 없이 LV 크기 확장
스냅샷:     특정 시점 복사본 (백업, 테스트용)
스트라이핑: 여러 PV에 분산 저장 (RAID 0 유사, 성능 향상)
```

---

## 4. XFS — 파일시스템

> 출처: [RHEL 10 Getting started with XFS](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_file_systems/getting-started-with-xfs)

### 4-1. XFS란

파일시스템은 블록 장치 위에 파일/디렉토리 구조를 만드는 소프트웨어 규칙 체계입니다.

```
파일시스템 없을 때 (날것의 블록 장치):
블록0: [바이너리 데이터]
블록1: [바이너리 데이터]
→ "이게 어떤 파일인지, 이름이 뭔지" 아무것도 모름

파일시스템 올린 후 (mkfs.xfs):
블록0: [XFS 슈퍼블록]
블록1: [inode 테이블]
블록2: [디렉토리 구조]
블록3~: [실제 파일 데이터]
→ "report.txt 파일은 블록 42~45에 있고, 소유자는 john"
```

XFS는 1993년 SGI가 대용량/고성능 환경을 위해 개발했습니다. RHEL 10의 기본 파일시스템입니다.

### 4-2. XFS 핵심 설계 원칙

#### 저널링 (Journaling)

```
저널링 없을 때:
파일 쓰는 중 → 정전 → 파일 절반만 기록 → 손상

저널링 있을 때:
"파일 쓸 거야" → 로그(journal) 먼저 기록
→ 실제 파일 쓰기
→ "완료" 로그 기록
→ 중간에 꺼져도 재부팅 시 로그 보고 복구
```

#### Extent 기반 공간 할당

```
블록 단위 방식 (구형):
report.txt = 블록42 + 블록43 + 블록44 + 블록45
inode에: [42, 43, 44, 45] → 4개 항목

Extent 방식 (XFS):
report.txt = 블록42부터 4개 연속
inode에: [시작:42, 길이:4] → 1개 항목

파일이 클수록, Extent 방식이 훨씬 효율적
```

#### AG 병렬 처리

```
XFS가 디스크를 AG 단위로 나눠 동시에 작업
→ 멀티코어 환경에서 병렬 I/O 처리
→ 단, 같은 AG 내 파일들은 직렬 처리
→ XFS는 새 파일을 다른 AG에 분산 배치해서 완화
```

#### 지연 할당 (Delayed Allocation)

```
즉시 할당: 쓸 때마다 블록 바로 할당 → 단편화
지연 할당: 메모리에 모았다가 한 번에 연속 블록 할당
→ 단편화 최소화, 성능 향상
```

### 4-3. 파일시스템 생성 및 점검

> 출처: [RHEL 10 Creating an XFS file system](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_file_systems/creating-an-xfs-file-system)

```bash
# XFS 생성
mkfs.xfs /dev/sdb1
mkfs.xfs -L "mydata" /dev/sdb1      # 레이블 지정
mkfs.xfs -f /dev/sdb1               # 기존 파일시스템 덮어쓰기

# ⚠️ mkfs는 파괴적 명령 — 즉시 기존 데이터 전부 삭제

# 파일시스템 정보 확인
xfs_info /dev/sdb1
blkid /dev/sdb1     # UUID, 타입 확인

# XFS 점검 및 복구 (반드시 언마운트 상태에서)
umount /data
xfs_repair /dev/sdb1
```

xfs_repair 동작 순서:

```
Phase 1: 슈퍼블록 확인
Phase 2: AG 헤더 확인
Phase 3: inode 유효성 확인
Phase 4: 디렉토리 구조 확인
Phase 5: 여유 공간 확인
Phase 6: 연결 끊긴 inode → /lost+found 이동
Phase 7: 저널 리셋
```

언마운트가 필수인 이유:

```
마운트된 상태 = OS가 계속 읽기/쓰기 중
xfs_repair 동시 실행 = 구조 수정 중
→ 충돌 → 더 큰 손상 가능
```

### 4-4. XFS vs ext4 비교

> 출처: [RHEL 10 Managing file systems](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/managing_file_systems/index)

| 항목 | XFS | ext4 |
|------|-----|------|
| RHEL 10 기본 | ✅ | ❌ |
| 최대 파일 크기 | 8 EiB (이론값, 공식 문서 미명시) | 16 TiB |
| 최대 파일시스템 | 8 EiB (이론값, 공식 문서 미명시) | 50 TiB |
| 파일시스템 축소 | **불가** | 가능 |
| 마운트 중 확장 | 가능 | 가능 |
| 점검 도구 | xfs_repair | e2fsck |
| inode 할당 | 동적 (부족 없음) | 고정 (부족 가능) |
| 메타데이터 오류 처리 | 파일시스템 종료 (EFSCORRUPTED) | 계속 진행 (기본값) |

```
작업          XFS              ext4
──────────────────────────────────────
파일시스템 생성  mkfs.xfs         mkfs.ext4
점검/복구       xfs_repair       e2fsck
정보 확인       xfs_info         tune2fs -l
파일시스템 확장  xfs_growfs       resize2fs
파일시스템 축소  불가             resize2fs
```

---

## 5. 마운트와 /etc/fstab

> 출처: [RHEL 10 Mounting file systems](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_file_systems/mounting-file-systems)

### 5-1. VFS — 마운트의 내부 동작

VFS(Virtual File System)는 모든 파일시스템을 동일한 인터페이스로 추상화하는 커널 계층입니다. 덕분에 사용자 프로그램은 XFS인지 ext4인지 몰라도 동일하게 `open()`, `read()`를 씁니다.

VFS의 핵심 자료구조 4개:

```
superblock 객체
  → 마운트된 파일시스템 하나를 표현
  → 파일시스템 전체 메타데이터
  → 파일시스템 드라이버 함수 포인터 모음

inode 객체
  → 파일/디렉토리 하나를 표현
  → 디스크의 inode를 메모리에 올린 것

dentry 객체 (directory entry)
  → 파일 이름과 inode의 연결
  → "/data/report.txt" 경로의 각 구성 요소
  → dcache에 캐시 → 경로 탐색 빠르게

file 객체
  → 프로세스가 열어놓은 파일 하나
  → open() 시 생성, close() 시 소멸
  → 현재 읽기 위치(offset), 접근 모드

vfsmount 구조체
  → 마운트 포인트와 파일시스템 루트의 연결
  → 마운트의 실체
```

#### `mount /dev/sdb1 /data` 실행 시 단계별 동작

```
1. 슈퍼블록 읽기
   /dev/sdb1 블록 0 읽기
   → 매직 넘버 확인 (XFS = 0x58465342 "XFSB")
   → XFS 드라이버 호출
   → VFS superblock 객체 생성 (메모리에)
   → XFS 전용 함수 포인터 연결

2. 루트 inode 읽기
   XFS 루트 디렉토리 inode = 128 (4KB 블록 크기 기준 일반값, 블록 크기에 따라 다를 수 있음)
   → 디스크에서 inode 128 읽기
   → VFS inode 객체 생성
   → XFS 전용 함수 포인터 연결

3. dentry 생성
   루트 inode로 dentry 생성
   superblock의 s_root = 이 dentry

4. vfsmount 생성 — 핵심
   vfsmount:
     mnt_root       = /dev/sdb1의 루트 dentry
     mnt_mountpoint = /data의 dentry

   마운트 테이블에 등록
   → "/data" 경로 접근 시
   → 원래 /data dentry 대신
   → /dev/sdb1의 루트 dentry로 리다이렉트
```

마운트의 실체는 **경로 요청을 다른 dentry로 리다이렉트하는 것**입니다. "가린다"는 표현보다 "경로를 다른 곳으로 연결한다"가 정확합니다.

```
마운트 전: /data 경로 → 원래 /data 디렉토리 dentry
마운트 후: /data 경로 → vfsmount 확인 → /dev/sdb1 루트 dentry로 교체

원래 /data 디렉토리가 삭제된 게 아님
경로 표지판이 바뀐 것
언마운트하면 원래 표지판으로 복원
```

#### `/data/report.txt` 읽기 경로 탐색

```
open("/data/report.txt")

1. "/" dentry 찾기 (dcache에서)

2. "data" 탐색
   /의 inode에서 "data" 찾기
   → /data dentry 발견
   → 마운트 포인트 확인 ✅
   → vfsmount 조회
   → /dev/sdb1의 루트 dentry로 교체

3. "report.txt" 탐색
   /dev/sdb1 루트 inode에서 "report.txt" 찾기
   → XFS 드라이버 lookup 함수 호출
   → 디스크에서 inode 읽기
   → dentry 생성, dcache에 캐시

4. file 객체 생성
   → 프로세스 파일 디스크립터 테이블에 등록
   → fd 반환
```

#### LVM이 있을 때 전체 흐름

```
open("/home/alice/file.txt") 실행 시:

사용자
  ↓
VFS: /home → vfsmount → /dev/vg0/lv_home 루트로 교체
  ↓
XFS: "lv_home의 블록 42번 읽어줘"
  ↓
LVM(dm-mapper): 매핑 테이블 조회
  "블록 42번은 /dev/sdc의 블록 4400번이네"
  ↓
물리 디스크 드라이버: /dev/sdc 블록 4400번 읽기
  ↓
실제 데이터 반환

VFS도 XFS도 LVM 변환 과정을 모름
```

#### 언마운트 시 동작

```
umount /data

1. 사용 중인 file 객체 확인
   → 있으면 "Device busy" 오류
   → 없으면 계속

2. dcache에서 /dev/sdb1 관련 dentry 제거

3. 더티 페이지 플러시
   → 메모리 변경사항 모두 디스크에 기록

4. superblock dirty 플래그 클리어
   → "정상 종료됨" 기록

5. vfsmount 제거
   → 마운트 테이블에서 삭제
   → /data 경로가 원래 디렉토리로 복원

6. superblock, inode 객체 메모리 해제
```

### 5-2. mount 명령

```bash
# 기본 마운트
mount /dev/sdb1 /data

# 파일시스템 타입 명시
mount -t xfs /dev/sdb1 /data

# 옵션 지정
mount -o ro /dev/sdb1 /data          # 읽기 전용
mount -o noexec /dev/sdb1 /data      # 실행 불가
mount -o nosuid /dev/sdb1 /data      # SetUID 무효화
mount -o noatime /dev/sdb1 /data     # 접근 시간 업데이트 안 함

# 마운트 현황 확인
findmnt
findmnt /data
df -hT

# 언마운트
umount /data
```

### 5-3. /etc/fstab — 자동 마운트

부팅 시마다 `mount`를 손으로 치지 않도록 등록합니다.

```
/etc/fstab 형식 (6개 필드):

장치              마운트포인트  타입   옵션      dump  fsck순서
UUID=abc123-...  /            xfs   defaults  0     0
```

필드별 의미:

```
1. 장치 식별자
   UUID=xxx   ← 권장 (디스크 순서가 바뀌어도 안전)
   /dev/sda1  ← 비권장 (순서 바뀌면 틀린 파티션 마운트 위험)
   LABEL=xxx  ← 레이블

2. 마운트 포인트
   /, /boot, /home, /data 등

3. 파일시스템 타입
   xfs, ext4, swap, tmpfs, nfs 등

4. 마운트 옵션
   defaults = rw,suid,dev,exec,auto,nouser,async
   ro       = 읽기 전용
   noexec   = 실행 불가
   nosuid   = SetUID 무효화
   noatime  = 접근 시간 기록 안 함 (성능 향상)
   nofail   = 마운트 실패해도 부팅 계속

5. dump 백업 여부
   0 = 안 함 (거의 항상 0)

6. fsck 검사 순서
   0 = 검사 안 함
   1 = 루트(/) 전용 (가장 먼저)
   2 = 나머지
   XFS는 자체 저널링으로 복구 → 보통 0
```

UUID를 쓰는 이유:

```
/dev/sda, /dev/sdb는 불안정
→ 디스크 추가/제거 시 순서 바뀔 수 있음
→ fstab이 /dev/sdb를 찾는데 없음 → 부팅 실패

UUID = 파일시스템에 고정된 고유 번호
→ 디스크 순서 바뀌어도 항상 올바른 파티션 찾음
```

```bash
# UUID 확인
blkid /dev/sdb1
# /dev/sdb1: UUID="05e99ec8-def1-4a5e-8a9d-5945339ceb2a" TYPE="xfs"

# fstab 등록 예시
cat /etc/fstab
# UUID=05e99ec8-...  /data  xfs  defaults  0  0

# 등록 후 검증 (재부팅 전 필수)
mount -a
# 오류 없으면 정상

# fstab 오류 시 부팅 실패 가능
# nofail 옵션으로 방지:
# UUID=xxx  /data  xfs  defaults,nofail  0  0
```

부팅 시 순서:

```
BIOS/UEFI → GRUB → 커널 로드
                        ↓
              /etc/fstab 읽기
                        ↓
    systemd-fstab-generator가
    fstab 항목을 systemd mount 유닛으로 변환
                        ↓
    순서대로 마운트
    / → /boot → /home → /data ...
                        ↓
    로그인 화면
```

---

## 6. 전체 실습 시나리오

새 디스크 `/dev/sdb` 추가 후 LVM으로 구성하고 자동 마운트까지 전체 흐름:

```bash
# 1. 디스크 확인
lsblk

# 2. 파티션 생성 (GPT)
parted /dev/sdb mklabel gpt
parted /dev/sdb mkpart "data" xfs 1MiB 100%
parted /dev/sdb set 1 lvm on
partprobe /dev/sdb

# 3. LVM 구성
pvcreate /dev/sdb1
vgcreate vgdata /dev/sdb1
lvcreate -L 50G -n lv_app vgdata
lvcreate -l 100%FREE -n lv_log vgdata

# 4. 파일시스템 생성
mkfs.xfs /dev/vgdata/lv_app
mkfs.xfs /dev/vgdata/lv_log

# 5. 마운트 포인트 생성 및 임시 마운트
mkdir -p /app /var/log/app
mount /dev/vgdata/lv_app /app
mount /dev/vgdata/lv_log /var/log/app

# 6. UUID 확인
blkid /dev/vgdata/lv_app
blkid /dev/vgdata/lv_log

# 7. /etc/fstab 등록
echo "/dev/vgdata/lv_app  /app          xfs  defaults  0 0" >> /etc/fstab
echo "/dev/vgdata/lv_log  /var/log/app  xfs  defaults  0 0" >> /etc/fstab

# 8. 검증
mount -a

# 9. 상태 확인
lsblk -f
df -hT
findmnt
pvs && vgs && lvs
```

---

## 7. 빠른 참조 치트시트

```bash
# ── 디스크/파티션 확인 ────────────────────────
lsblk                        # 블록 장치 목록
lsblk -f                     # 파일시스템 정보 포함
fdisk -l /dev/sda            # 파티션 테이블 상세
blkid /dev/sda1              # UUID, 타입 확인
findmnt                      # 마운트 현황 트리
df -hT                       # 디스크 사용량

# ── 파티션 생성 ───────────────────────────────
parted /dev/sdb mklabel gpt                      # GPT 테이블
parted /dev/sdb mkpart "이름" xfs 1MiB 100%     # GPT 파티션
parted /dev/sdb set 1 lvm on                     # 플래그 설정
partprobe /dev/sdb                               # 커널 반영

# ── LVM ──────────────────────────────────────
pvcreate /dev/sdb1                  # PV 생성
vgcreate vg0 /dev/sdb1              # VG 생성
vgextend vg0 /dev/sdc               # VG에 PV 추가
lvcreate -L 20G -n lv0 vg0         # LV 생성
lvextend -L +10G /dev/vg0/lv0      # LV 확장
xfs_growfs /mountpoint              # XFS 파일시스템 확장
pvs / vgs / lvs                     # 상태 확인

# ── 파일시스템 ───────────────────────────────
mkfs.xfs /dev/sdb1                  # XFS 생성
mkfs.ext4 /dev/sdb1                 # ext4 생성
xfs_repair /dev/sdb1                # XFS 점검 (언마운트 필수)
e2fsck -f /dev/sdb1                 # ext4 점검 (언마운트 필수)
xfs_info /dev/sdb1                  # XFS 정보

# ── 마운트 ───────────────────────────────────
mount /dev/sdb1 /data               # 마운트
mount -o ro /dev/sdb1 /data         # 읽기 전용
umount /data                        # 언마운트
mount -a                            # fstab 전체 마운트 (검증용)
```

---


## 📚 주요 출처

| 문서 | URL |
|------|-----|
| RHEL 10 Managing storage devices | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_storage_devices |
| RHEL 10 Disk partitions | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_storage_devices/disk-partitions |
| RHEL 10 Getting started with partitions | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_storage_devices/getting-started-with-partitions |
| RHEL 10 Configuring and managing logical volumes | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_and_managing_logical_volumes |
| RHEL 10 Managing file systems | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/managing_file_systems/index |
| RHEL 10 Getting started with XFS | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_file_systems/getting-started-with-xfs |
| RHEL 10 Creating an XFS file system | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_file_systems/creating-an-xfs-file-system |
| RHEL 10 Mounting file systems | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_file_systems/mounting-file-systems |
| Linux Kernel VFS 문서 | https://www.kernel.org/doc/html/latest/filesystems/vfs.html |

---

*문서 작성 기준: Rocky Linux 10 / RHEL 10 공식 문서 (2025년 기준)*