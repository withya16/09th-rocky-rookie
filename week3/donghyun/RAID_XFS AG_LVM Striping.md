# RAID · XFS AG · LVM Striping · AWS EBS 딥다이브

> **담당:** 동현 | **범위:** 스토리지 하드웨어 → 파일 시스템 → 클라우드 블록 스토리지 전체 스택
> 

> **목표:** 물리 디스크의 RAID 구성부터 XFS 내부 구조, LVM 스트라이핑, 그리고 AWS EBS까지 — 스토리지 스택 전체를 관통하는 지식을 쌓고, 각 레이어의 트레이드오프를 SRE 관점에서 판단할 수 있게 되는 것
> 

---

# Phase 1: RAID — 물리 디스크를 묶는 기술

## 1-1. RAID란?

RAID(Redundant Array of Independent Disks)는 **여러 개의 물리 디스크를 하나의 논리 장치로 묶어서** 성능, 안정성, 또는 둘 다를 확보하는 기술입니다. 1988년 UC Berkeley에서 처음 제안되었고, 지금까지 서버 스토리지의 핵심 기술로 사용되고 있습니다.

RAID를 이해하기 위해 먼저 두 가지 핵심 개념을 짚겠습니다:

- **Striping (스트라이핑):** 데이터를 여러 디스크에 **쪼개서 분산 저장**하는 기법. 읽기/쓰기를 병렬로 처리할 수 있어 **속도가 올라갑니다.**
- **Mirroring (미러링):** 같은 데이터를 **두 개 이상의 디스크에 동시에 복제**하는 기법. 한 디스크가 죽어도 다른 디스크에서 데이터를 읽을 수 있어 **안정성이 올라갑니다.**
- **Parity (패리티):** 데이터를 복구할 수 있는 **오류 검출/복구 정보**를 추가로 저장하는 기법. 미러링보다 디스크를 적게 사용하면서 안정성을 확보합니다.

## 1-2. RAID 레벨 전체 정리

### RAID 0 — Striping Only (속도 올인, 안정성 제로)

```
데이터 A:  [A1] [A2] [A3] [A4]
           ↓     ↓     ↓     ↓
          Disk0 Disk1 Disk2 Disk3
```

- **원리:** 데이터를 여러 디스크에 균등하게 쪼개서 저장합니다. 4개 디스크라면 4개가 동시에 읽기/쓰기를 처리하므로 **이론상 성능이 디스크 수만큼 배가**됩니다.
- **용량:** 전체 디스크 용량의 100%를 사용합니다. (100GB × 4 = 400GB)
- **치명적 단점:** 디스크 하나만 고장 나도 **전체 데이터가 유실**됩니다. 복구 불가능.
- **사용 처:** 일시적 캐시, 스트리밍 처리, 날려도 괜찮은 임시 데이터. **절대 프로덕션 DB에 단독 사용 금지.**
- **핵심:** LVM 스트라이핑의 원조 모델입니다. 우리가 `lvcreate -i 2`로 만든 스트라이핑 LV가 바로 소프트웨어 RAID 0입니다.

### RAID 1 — Mirroring Only (안정성 올인)

```
데이터 A:  [A 전체]  [A 전체]
            ↓          ↓
          Disk0      Disk1
         (원본)      (복사본)
```

- **원리:** 같은 데이터를 두 디스크에 **완전히 동일하게 복제**합니다.
- **용량:** 전체 디스크 용량의 50%만 사용합니다. (100GB × 2 = 실제 100GB)
- **장점:** 한 디스크가 완전히 죽어도 **다운타임 제로로 서비스 지속** 가능. 읽기 성능도 2배(양쪽에서 동시 읽기).
- **단점:** 쓰기 성능은 향상 없음(양쪽에 다 써야 하니까). 용량 효율 50%로 비쌉니다.
- **사용 처:** OS 부트 디스크, 소규모 DB의 로그 볼륨.

### RAID 5 — Striping + Distributed Parity (균형형)

```
          Disk0    Disk1    Disk2    Disk3
Stripe 0: [D0]     [D1]     [D2]     [P0]
Stripe 1: [D3]     [D4]     [P1]     [D5]
Stripe 2: [D6]     [P2]     [D7]     [D8]
Stripe 3: [P3]     [D9]     [D10]    [D11]

D = Data, P = Parity (복구용 정보)
Parity가 각 디스크에 골고루 분산됨
```

- **원리:** 데이터를 Striping으로 분산하면서, **Parity 정보**를 각 디스크에 돌아가며 저장합니다. 디스크 1개가 죽으면 나머지 디스크의 Parity로 데이터를 **계산해서 복구**합니다.
- **최소 디스크:** 3개
- **용량:** 전체에서 디스크 1개분 빠짐. (100GB × 4 = 실제 300GB)
- **장점:** RAID 1보다 용량 효율이 높고, 읽기 성능도 좋습니다.
- **단점:** **쓰기 성능이 느립니다.** 데이터를 쓸 때마다 Parity를 계산하고 갱신해야 하기 때문입니다(Write Penalty). 디스크가 1개 죽은 상태(Degraded Mode)에서 Rebuild 중 **또 다른 디스크가 죽으면 전체 데이터 유실.**
- **사용 처:** 읽기 중심 워크로드(파일 서버, 웹 서버, 아카이브).

### RAID 6 — Striping + Double Parity (RAID 5 강화판)

- **원리:** RAID 5와 동일하지만 Parity를 **2개** 저장합니다. 디스크 **2개까지** 동시에 죽어도 복구 가능.
- **최소 디스크:** 4개
- **용량:** 전체에서 디스크 2개분 빠짐. (100GB × 6 = 실제 400GB)
- **장점:** RAID 5의 "Rebuild 중 두 번째 디스크 사망" 공포를 해결합니다.
- **단점:** Write Penalty가 RAID 5보다 더 큽니다. 쓰기 성능이 더 낮습니다.
- **사용 처:** 대용량 디스크(4TB+)로 구성된 어레이. 대용량 디스크일수록 Rebuild 시간이 길어져 RAID 5는 위험합니다.

### RAID 10 (1+0) — Mirroring + Striping (현업의 정석)

```
          ┌── Mirror Pair 1 ──┐  ┌── Mirror Pair 2 ──┐
          │  Disk0    Disk1    │  │  Disk2    Disk3    │
Stripe 0: │  [A1]     [A1]     │  │  [A2]     [A2]     │
Stripe 1: │  [B1]     [B1]     │  │  [B2]     [B2]     │
          └───────────────────┘  └───────────────────┘
                   ↑ RAID 1 복제              ↑ RAID 1 복제
          ←──────── RAID 0 스트라이핑 ─────────→
```

- **원리:** 먼저 디스크를 **쌍(Pair)으로 미러링(RAID 1)**하고, 그 미러 쌍들을 **스트라이핑(RAID 0)**으로 묶습니다.
- **최소 디스크:** 4개
- **용량:** 전체의 50%. (100GB × 4 = 실제 200GB)
- **장점:** 읽기/쓰기 모두 빠르고, 각 미러 쌍에서 1개씩 죽어도 서비스 지속. Rebuild도 해당 미러 쌍만 복구하면 되어 빠릅니다.
- **단점:** 용량 효율 50%. 디스크를 많이 사야 합니다.
- **사용 처:** **프로덕션 DB 서버의 절대적 표준.** MySQL, PostgreSQL, Oracle 등 Write-heavy 워크로드에 최적.

## 1-3. RAID 레벨 비교 요약표

| RAID | 최소 디스크 | 용량 효율 | 읽기 성능 | 쓰기 성능 | 내결함성 | 현업 사용처 |
| --- | --- | --- | --- | --- | --- | --- |
| **0** | 2 | 100% | ⬆⬆ 매우 빠름 | ⬆⬆ 매우 빠름 | ❌ 없음 | 임시 캐시, 스크래치 |
| **1** | 2 | 50% | ⬆ 빠름 | ➡ 보통 | 디스크 1개 | OS 부트, 로그 |
| **5** | 3 | (N-1)/N | ⬆ 빠름 | ⬇ 느림 | 디스크 1개 | 파일 서버, 아카이브 |
| **6** | 4 | (N-2)/N | ⬆ 빠름 | ⬇⬇ 매우 느림 | 디스크 2개 | 대용량 어레이 |
| **10** | 4 | 50% | ⬆⬆ 매우 빠름 | ⬆ 빠름 | 미러쌍당 1개 | **프로덕션 DB 서버** |

## 1-4. Hardware RAID vs Software RAID

| 구분 | Hardware RAID | Software RAID (mdadm / LVM) |
| --- | --- | --- |
| 처리 주체 | 전용 RAID 컨트롤러(하드웨어) | OS 커널(CPU) |
| 장점 | CPU 부하 없음, 배터리 백업(BBU)으로 Write Cache 안전 | 추가 하드웨어 불필요, 유연함 |
| 단점 | 컨트롤러 고장 시 같은 모델 필요, 비쌈 | CPU 자원 소비, BBU 없어 Write Cache 위험 |
| 현업 | 온프레미스 DB 서버 | 클라우드 환경, 개발 서버 |

> **클라우드 환경(AWS 등)에서는?** 물리 RAID 컨트롤러가 없습니다. AWS EBS 자체가 내부적으로 데이터를 복제하므로, 유저가 직접 RAID를 구성할 일은 거의 없습니다. 다만 **성능이 부족할 때** 여러 EBS를 LVM으로 스트라이핑하는 케이스가 있었습니다(현재는 gp3 확장으로 대부분 불필요).
> 

---

# Phase 2: XFS 파일 시스템 딥다이브 — AG, Inode, Block

## 2-1. XFS가 Rocky Linux의 기본인 이유

XFS는 1990년대 SGI가 슈퍼컴퓨터용으로 개발한 64-bit Journaling 파일 시스템입니다. RHEL/Rocky Linux의 기본 파일 시스템으로 채택된 이유:

- **최대 1 EiB(Exbibyte)** 까지의 파일 시스템 크기 지원
- 멀티코어 CPU에서 **병렬 I/O 성능이 ext4보다 우수**
- 대용량 파일과 높은 동시성 워크로드에 최적화
- Online Growing (마운트 상태에서 확장 가능)

## 2-2. Allocation Group (AG) — XFS의 핵심 설계

AG는 XFS의 **가장 중요한 내부 구조**입니다. 전체 파일 시스템을 **독립적인 구역(AG)**으로 나누어, 각 AG가 자체적으로 Inode 할당, Free Space 관리, B-tree 인덱싱을 수행합니다.

```
┌─────────────────────────── XFS 파일 시스템 전체 ───────────────────────────┐
│                                                                           │
│  ┌─── AG 0 ───┐  ┌─── AG 1 ───┐  ┌─── AG 2 ───┐  ┌─── AG 3 ───┐       │
│  │ Superblock  │  │ Superblock │  │ Superblock │  │ Superblock │       │
│  │ Free Space  │  │ Free Space │  │ Free Space │  │ Free Space │       │
│  │ B-tree      │  │ B-tree     │  │ B-tree     │  │ B-tree     │       │
│  │ Inode B-tree│  │ Inode B-tree│ │ Inode B-tree│ │ Inode B-tree│      │
│  │ Data Blocks │  │ Data Blocks│  │ Data Blocks│  │ Data Blocks│       │
│  └─────────────┘  └────────────┘  └────────────┘  └────────────┘       │
│     CPU 0이          CPU 1이         CPU 2가         CPU 3이             │
│     독립 관리        독립 관리       독립 관리       독립 관리           │
└───────────────────────────────────────────────────────────────────────────┘
```

### AG가 왜 중요한가?

**핵심: Lock Contention 제거**

ext4 같은 전통적인 파일 시스템에서는 Inode를 할당하거나 Free Space를 관리할 때 **전체 파일 시스템에 대한 Lock**이 필요합니다. CPU가 많아질수록 이 Lock을 기다리는 시간이 병목이 됩니다.

XFS의 AG는 각 구역이 **독립적인 Lock**을 가집니다. CPU 0이 AG 0에서 파일을 만들 때, CPU 1은 동시에 AG 1에서 파일을 만들 수 있습니다. 서로 간섭이 없습니다(Lock-free Parallelism). 이것이 **멀티코어 환경에서 XFS가 ext4를 압도하는 근본적인 이유**입니다.

### AG 관련 실습 명령어

```
# AG 정보 확인 — agcount가 AG 개수
sudo xfs_info /dev/mapper/rl-root

# 출력 예시:
# meta-data=/dev/mapper/rl-root   isize=512  agcount=16, agsize=xxxxxx blks
# data     =                      bsize=4096 blocks=xxxxxxx
```

### AG 개수와 Prometheus 튜닝의 연결

이전 스터디에서 배운 내용 기억하시나요? Prometheus WAL 커널 튜닝 문서에서 다뤘던 내용입니다:

> "매우 작은 크기에서 훨씬 큰 크기로 파일 시스템을 확장하면 Allocation Group이 많이 생성되어 성능 문제가 발생할 수 있습니다."
> 

AG가 너무 많으면:

- 각 AG의 크기가 작아져 Free Space가 파편화됩니다.
- Extent(연속된 블록 덩어리) 할당 시 큰 청크를 확보하기 어려워집니다.
- Prometheus WAL처럼 **대용량 순차 쓰기** 워크로드에서 성능이 저하됩니다.

**권장:** 파일 시스템 확장 시 원래 크기의 **최대 10배**까지만 늘리는 것이 Best Practice입니다.

## 2-3. Inode와 Block — 파일이 저장되는 방식

### Inode (Index Node)

파일의 **메타데이터**를 저장하는 구조체입니다. 파일 이름이 아닌, 파일의 "신분증"입니다.

| Inode에 저장되는 정보 | 예시 |
| --- | --- |
| 파일 타입 | 일반 파일, 디렉터리, 심볼릭 링크 |
| 소유자/그룹 | mr8356 / wheel |
| 권한 | rwxr-xr-x (755) |
| 파일 크기 | 4096 bytes |
| 타임스탬프 | atime, mtime, ctime |
| Data Block 포인터 | 실제 데이터가 있는 블록 주소 (Extent 목록) |

> **주의:** Inode에는 **파일 이름이 없습니다.** 파일 이름은 상위 **디렉터리의 Data Block**에 저장됩니다. (파일명 → Inode 번호 매핑)
> 

### Block

파일의 **실제 데이터**가 저장되는 최소 단위입니다.

- XFS 기본 Block Size: **4 KiB** (4096 bytes)
- 파일 시스템 생성 시 결정되며, 이후 변경 불가
- Prometheus WAL이 32 KiB 단위로 쓸 때, 이것은 **8개의 Block**에 걸쳐 기록되는 것입니다.

### Extent — XFS의 핵심 할당 단위

ext4의 전통적인 Block 단위 할당과 달리, XFS는 **Extent(연속된 Block의 묶음)**를 기본 할당 단위로 사용합니다.

```
전통적 Block 할당 (ext2/3):   [B1] [B3] [B7] [B12] ← 흩어져 있음, 파편화
Extent 기반 할당 (XFS):       [Block 1~100] ← 100개 블록을 한 덩어리로!
```

- Extent는 (시작 Block 번호, 길이)로 표현됩니다.
- 대용량 파일도 소수의 Extent로 관리할 수 있어 **메타데이터가 작고 빠릅니다.**
- Prometheus WAL 같은 순차 쓰기 패턴에서 연속된 큰 Extent를 할당받으면 **탁월한 성능**을 발휘합니다.

## 2-4. Mount Option — 성능에 직결되는 옵션들

이전 스터디의 커널 튜닝 문서에서 `noatime`을 다뤘습니다. 여기서 전체 마운트 옵션을 정리합니다.

| 옵션 | 설명 | 성능 영향 |
| --- | --- | --- |
| `noatime` | 파일 접근 시간(atime) 기록을 완전히 끔 | **쓰기 I/O 감소** — WAL 같은 Append-only 워크로드에 필수 |
| `nodiratime` | 디렉터리 접근 시간 기록을 끔 (`noatime`에 포함됨) | 디렉터리 탐색이 많은 경우 도움 |
| `relatime` | 수정 시간보다 접근 시간이 오래된 경우에만 갱신 (RHEL 기본값) | `noatime`보다는 I/O 발생, 대부분의 일반 워크로드에 적합 |
| `discard` | SSD/NVMe에서 삭제된 블록을 TRIM 명령으로 즉시 통보 | SSD 수명과 성능에 도움, 다만 **쓰기 레이턴시 증가** 가능 |
| `inode64` | 전체 파일 시스템에서 Inode 할당 허용 (기본값) | 대용량 FS에서 Inode 고갈 방지 |
| `logbufs=8` | Journaling Log 버퍼 수를 8개로 늘림 | 쓰기 집약적 워크로드에서 Journaling 병목 완화 |
| `logbsize=256k` | Journal Log 버퍼 크기를 256KB로 설정 | 대용량 트랜잭션에서 성능 향상 |

### Prometheus 데이터 볼륨 최적 마운트 예시

```
# /etc/fstab 예시
UUID=xxxx  /var/lib/prometheus  xfs  defaults,noatime,logbufs=8,logbsize=256k  0 0
```

---

# Phase 3: LVM Striping — 소프트웨어 레벨의 RAID 0

## 3-1. LVM이란?

LVM(Logical Volume Manager)은 물리 디스크 위에 **유연한 논리적 볼륨 계층**을 추가하는 기술입니다.

```
물리 디스크 레이어:    /dev/sda       /dev/sdb       /dev/sdc
                         ↓               ↓               ↓
PV (Physical Volume):  [PV1]          [PV2]          [PV3]
                         ↓               ↓               ↓
VG (Volume Group):    ←────── my_vg (하나의 풀) ───────→
                         ↓               ↓               ↓
LV (Logical Volume):  [lv_data: 50GB]  [lv_log: 20GB]  [나머지 여유]
                         ↓               ↓
File System:          XFS             XFS
```

- **PV (Physical Volume):** 물리 디스크 또는 파티션을 LVM이 관리할 수 있는 형태로 초기화한 것
- **VG (Volume Group):** 여러 PV를 하나의 스토리지 풀로 묶은 것
- **LV (Logical Volume):** VG 안에서 잘라 쓰는 가상 파티션. **원하는 크기로 생성/확장/축소** 가능

## 3-2. LVM Striping (소프트웨어 RAID 0)

LV를 생성할 때 `-i` 옵션으로 Stripe 수를 지정하면, **데이터가 여러 PV에 분산 저장**됩니다.

```
# 2개의 PV에 걸쳐 스트라이핑하는 10GB LV 생성
sudo lvcreate -L 10G -i 2 -I 64K -n striped_lv my_vg

# -i 2   : Stripe 수 (PV 2개에 분산)
# -I 64K : Stripe Size (한 번에 한 PV에 쓰는 단위)
# -n     : LV 이름
```

```
        PV1 (/dev/sda)     PV2 (/dev/sdb)
Chunk1: [Data 0-64KB]      [Data 64-128KB]
Chunk2: [Data 128-192KB]   [Data 192-256KB]
Chunk3: [Data 256-320KB]   [Data 320-384KB]
         ...                 ...
```

### Stripe Size 선택의 중요성

| Stripe Size | 적합한 워크로드 | 이유 |
| --- | --- | --- |
| **64 KiB** (기본) | 일반적인 DB, 범용 서버 | 대부분의 I/O 패턴에 균형적 |
| **256 KiB** | Prometheus WAL, 대용량 순차 쓰기 | WAL의 32KiB 페이지가 한 Stripe에 들어가며, 여러 페이지를 묶어 한 번에 Flush |
| **4 KiB** | 랜덤 I/O 중심 (OLTP DB) | 작은 I/O가 여러 디스크에 골고루 분산 |

## 3-3. LVM Striping의 트레이드오프

| 장점 | 단점 |
| --- | --- |
| 디스크 수만큼 처리량 증가 | PV 하나만 죽어도 **전체 LV 데이터 유실** (RAID 0과 동일) |
| 추가 하드웨어(RAID 카드) 불필요 | OS 레벨에서 관리해야 해서 복잡 |
| 유연한 크기 조정 가능 | Stripe 수는 생성 후 변경 불가 |

---

# Phase 4: AWS EBS — 클라우드의 가상 디스크

## 4-1. EBS란?

Amazon EBS(Elastic Block Store)는 AWS EC2 인스턴스에 연결하는 **네트워크 기반 블록 스토리지**입니다.

### 로컬 디스크 vs EBS — 근본적인 차이

```
[온프레미스]
CPU ←→ SATA/NVMe 케이블 ←→ 물리 디스크 (로컬)
        ↑ 직접 연결, 레이턴시 극소

[AWS EBS]
EC2 인스턴스 ←→ AWS 내부 네트워크 ←→ EBS 볼륨 (원격)
                  ↑ 네트워크를 타므로 레이턴시 존재
                  ↑ 하지만 AWS가 내부적으로 데이터를 복제하여 내구성 보장
```

**핵심 차이:**

- EBS는 **네트워크로 연결된 원격 디스크**입니다. 로컬 NVMe처럼 물리적으로 서버에 꽂혀 있지 않습니다.
- 대신 AWS가 내부적으로 **여러 물리 디스크에 데이터를 복제**하여 높은 내구성(Durability)을 제공합니다.
- EC2 인스턴스가 종료되어도 EBS 데이터는 유지됩니다(영구 스토리지).

## 4-2. IOPS vs Throughput — 반드시 구분해야 하는 두 축

| 지표 | 정의 | 비유 | 중요한 워크로드 |
| --- | --- | --- | --- |
| **IOPS** (Input/Output Operations Per Second) | 초당 처리할 수 있는 **I/O 요청 횟수** | 편의점 계산대에서 1초에 처리하는 **손님 수** | DB의 랜덤 읽기/쓰기 (작은 데이터 수만 건) |
| **Throughput** (처리량, MB/s) | 초당 전송할 수 있는 **데이터 총량** | 고속도로에서 1초에 지나가는 **화물 총량** | 대용량 파일 복사, 로그 스트리밍, 백업 |

### 왜 둘 다 봐야 하는가?

I/O 크기에 따라 병목 지점이 달라집니다:

```
4 KiB 랜덤 읽기 × 3000 IOPS = 12 MB/s throughput  ← IOPS가 병목
256 KiB 순차 쓰기 × 3000 IOPS = 750 MB/s throughput ← Throughput이 병목
```

Prometheus WAL은 32KiB 단위 쓰기이므로:

- `32 KiB × 3000 IOPS = 96 MB/s` → 대부분의 경우 **IOPS가 먼저 병목**에 걸립니다.

## 4-3. EBS 볼륨 타입 완전 정리

### SSD 기반 (랜덤 I/O에 최적화)

| 타입 | gp3 (범용) | gp2 (이전 세대) | io2 Block Express (최고 성능) |
| --- | --- | --- | --- |
| **기본 IOPS** | 3,000 (무료 포함) | 용량에 비례 (3 IOPS/GiB) | 프로비저닝한 만큼 |
| **최대 IOPS** | 80,000 | 16,000 | 256,000 |
| **기본 Throughput** | 125 MiB/s (무료 포함) | 용량에 비례 | 프로비저닝한 만큼 |
| **최대 Throughput** | 2,000 MiB/s | 250 MiB/s | 4,000 MiB/s |
| **최대 용량** | 64 TiB | 16 TiB | 64 TiB |
| **내구성 (AFR)** | 99.8~99.9% (0.1~0.2% 실패율) | 99.8~99.9% | **99.999%** (0.001% 실패율) |
| **레이턴시** | Single-digit ms | Single-digit ms | **Sub-millisecond** |
| **가격 (GB/월)** | $0.08 | $0.10 | $0.125 |
| **IOPS 추가 비용** | $0.005/IOPS (3000 초과분) | 없음 (용량에 종속) | $0.065/IOPS |
| **핵심 특징** | 성능과 용량을 **독립적으로** 프로비저닝 | Burst Credit 모델 (불예측적) | 5 nines 내구성, 미션 크리티컬 |

### HDD 기반 (순차 I/O / 대용량에 최적화)

| 타입 | st1 (Throughput 최적화) | sc1 (Cold Storage) |
| --- | --- | --- |
| **최대 IOPS** | 500 | 250 |
| **최대 Throughput** | 500 MiB/s | 250 MiB/s |
| **가격 (GB/월)** | $0.045 | $0.015 |
| **사용처** | 로그 처리, ETL, Kafka, Hadoop | 아카이브, 백업, 드물게 접근하는 데이터 |

## 4-4. gp3 딥다이브 — 현업의 90%가 쓰는 이유

### gp2 → gp3 전환이 필수인 이유

**gp2의 문제: Burst Credit 모델**

gp2는 용량에 따라 Baseline IOPS가 결정됩니다 (3 IOPS/GiB). 100GB 볼륨이면 Baseline은 고작 300 IOPS입니다. 3,000 IOPS까지 Burst할 수 있지만, **Credit이 소진되면 300 IOPS로 추락**합니다.

```
gp2 100GB 볼륨:
 Baseline: 300 IOPS (= 100GB × 3)
 Burst:    3,000 IOPS (Credit 있을 때만)
 Credit 소진 시: 300 IOPS로 ← 프로덕션에서 장애 발생
```

**gp3의 해결: 용량과 성능의 분리**

gp3는 **용량과 무관하게** 무조건 3,000 IOPS + 125 MiB/s를 기본 제공합니다. 더 필요하면 돈을 내고 독립적으로 올릴 수 있습니다.

```
gp3 100GB 볼륨:
 기본: 3,000 IOPS + 125 MiB/s (항상, Credit 개념 없음)
 추가 필요 시: 최대 80,000 IOPS, 2,000 MiB/s까지 프로비저닝
```

### 비용 비교 예시: 8,000 IOPS 필요한 DB

| 볼륨 타입 | 구성 | 월 비용 |
| --- | --- | --- |
| gp2 | 8000 ÷ 3 = 2,667 GB 필요 | 2,667 × $0.10 = **$266.70** |
| **gp3** | 150 GB + 5,000 IOPS 추가 프로비저닝 | (150 × $0.08) + (5,000 × $0.005) = **$37.00** |
| io2 | 150 GB + 8,000 IOPS | (150 × $0.125) + (8,000 × $0.065) = **$538.75** |

> gp3가 gp2 대비 **86% 저렴**, io2 대비 **93% 저렴**합니다.
> 

## 4-5. EBS-Optimized Instance — 놓치기 쉬운 병목

EBS가 아무리 빨라도, EC2 인스턴스와 EBS 사이의 **네트워크 대역폭**이 부족하면 소용없습니다.

```
┌───────────┐    EBS 대역폭     ┌───────────┐
│  EC2      │ ←───────────────→ │  EBS      │
│ Instance  │  (여기가 병목!)   │  Volume   │
└───────────┘                   └───────────┘
```

- **EBS-Optimized Instance:** EC2와 EBS 사이에 **전용 네트워크 대역폭**을 할당한 인스턴스
- 최신 세대 인스턴스(m5, c5, r5 이상)는 기본으로 EBS-Optimized
- 인스턴스 타입마다 **최대 EBS 대역폭, 최대 IOPS** 한계가 다릅니다

> **실수하기 쉬운 포인트:** gp3에 80,000 IOPS를 프로비저닝해도, 인스턴스가 t3.medium(최대 ~11,800 IOPS)이면 11,800 IOPS에서 막힙니다. **볼륨과 인스턴스 양쪽의 한계를 모두 확인**해야 합니다.
> 

## 4-6. CloudWatch로 EBS 모니터링 — SRE 필수 지표

| CloudWatch Metric | 의미 | 위험 신호 |
| --- | --- | --- |
| `VolumeReadOps` / `VolumeWriteOps` | 초당 읽기/쓰기 IOPS | 프로비저닝 IOPS에 지속적으로 근접 |
| `VolumeReadBytes` / `VolumeWriteBytes` | 초당 읽기/쓰기 Throughput | 프로비저닝 Throughput에 근접 |
| `VolumeQueueLength` | I/O 대기열 길이 | **10 이상이면 I/O 병목** (DB는 1 근처 유지 목표) |
| `VolumeThroughputPercentage` | 프로비저닝 Throughput 대비 사용률 | 100% 근접 시 Throughput 포화 |
| `BurstBalance` (gp2 전용) | 남은 Burst Credit | **0에 도달하면 성능 급락** → gp3 전환 시급 |

---

# Phase 5: 모든 레이어를 연결하는 SRE 트레이드오프

## 5-1. 온프레미스 vs 클라우드 스토리지 전략

| 고려 요소 | 온프레미스 (RAID 10 + XFS) | AWS (단일 gp3 + XFS) |
| --- | --- | --- |
| **성능** | 물리 디스크 수에 비례, NVMe 직접 연결로 최저 레이턴시 | 프로비저닝한 IOPS/Throughput까지, 네트워크 레이턴시 존재 |
| **안정성** | RAID 컨트롤러 + BBU로 보호, 수동 관리 필요 | AWS가 내부 복제 관리, 99.8~99.999% 내구성 |
| **확장성** | 물리 디스크 추가 구매 + 다운타임 필요 | 클릭 몇 번으로 용량/성능 즉시 확장 (Elastic Volumes) |
| **관리 복잡도** | RAID 구성, LVM 설정, 모니터링 직접 구축 | AWS 콘솔/CLI로 관리, CloudWatch 자동 모니터링 |
| **비용 모델** | 초기 투자 비용(CAPEX) 높음, 장기적으로 저렴 가능 | 사용량 기반(OPEX), 단기 유연하지만 장기적으로 비쌀 수 있음 |

## 5-2. AWS에서 LVM Striping이 필요한 경우 (거의 없음)

과거에는 gp2의 IOPS 한계(16,000)를 넘기 위해 여러 EBS를 LVM으로 스트라이핑하는 패턴이 있었습니다:

```
# 과거 패턴: gp2 4개를 스트라이핑 → 4 × 16,000 = 64,000 IOPS
pvcreate /dev/xvdb /dev/xvdc /dev/xvdd /dev/xvde
vgcreate stripe_vg /dev/xvdb /dev/xvdc /dev/xvdd /dev/xvde
lvcreate -L 400G -i 4 -I 256K -n data_lv stripe_vg
```

**현재는 대부분 불필요합니다.** gp3가 단일 볼륨에서 **80,000 IOPS, 2,000 MiB/s**까지 지원하기 때문입니다.

| 구분 | LVM Striping (EBS 여러 개) | 단일 gp3 튜닝 |
| --- | --- | --- |
| **성능** | 이론상 디스크 수만큼 배가 | 설정한 IOPS 한계까지 |
| **안정성** | EBS 하나 장애 = 전체 데이터 유실 | AWS 내부 복제로 안정적 |
| **관리** | OS 레벨 LVM 설정, 스냅샷 복잡 | 클릭으로 확장, 스냅샷 간편 |
| **비용** | 볼륨 수 × 용량 비용 | 단일 볼륨 비용 |
| **현업 선호도** | 극도의 성능이 필요한 특수 케이스 | **90% 이상이 선호하는 표준** |

> **유일한 예외:** io2 Block Express(256,000 IOPS)로도 부족한 극한 성능이 필요한 경우. 하지만 이 수준이면 보통 인스턴스 스토어(NVMe 로컬 디스크)나 아키텍처 변경을 고려합니다.
> 

## 5-3. Prometheus on AWS — 권장 스토리지 구성

| 항목 | 권장 설정 | 근거 |
| --- | --- | --- |
| **볼륨 타입** | gp3 | WAL 쓰기는 3,000 기본 IOPS로 충분. 필요 시 독립 확장. |
| **IOPS** | 기본 3,000 → 메트릭 규모에 따라 증설 | CloudWatch `VolumeQueueLength`가 5 이상이면 증설 검토 |
| **Throughput** | 기본 125 MiB/s → WAL 압축 ON 시 충분 | 압축 OFF면 250~500 MiB/s 고려 |
| **마운트 옵션** | `noatime,logbufs=8,logbsize=256k` | WAL Append 패턴에 최적화 |
| **커널 튜닝** | `dirty_background_ratio=3`, `dirty_ratio=10` | I/O 스파이크 방지 (이전 스터디 내용) |
| **WAL 압축** | ON (Snappy) | 디스크 쓰기 1/200 감소 (이전 스터디 측정 결과) |

---

# 핵심 용어 정리

### 스토리지 하드웨어 용어

- **RAID:** 여러 디스크를 묶어 성능/안정성을 확보하는 기술
- **Striping:** 데이터를 여러 디스크에 분산 저장
- **Mirroring:** 같은 데이터를 복수 디스크에 복제
- **Parity:** 데이터 복구용 오류 검출 정보
- **Write Penalty:** RAID 5/6에서 Parity 계산으로 인한 쓰기 성능 저하
- **BBU (Battery Backup Unit):** RAID 컨트롤러의 Write Cache를 정전 시 보호하는 배터리
- **Degraded Mode:** RAID에서 디스크가 하나 빠진 상태로 동작하는 것

#### 파일 시스템 용어

- **AG (Allocation Group):** XFS가 파일 시스템을 나누는 독립 관리 구역
- **Inode:** 파일의 메타데이터(권한, 크기, 블록 포인터)를 저장하는 구조체
- **Block:** 파일 데이터의 최소 저장 단위 (XFS 기본 4 KiB)
- **Extent:** 연속된 Block의 묶음. XFS의 기본 할당 단위
- **Journaling:** 파일 시스템 변경 사항을 로그에 먼저 기록하여 크래시 복구를 보장하는 기법
- **atime/mtime/ctime:** 접근 시간 / 수정 시간 / 변경 시간

### LVM 용어

- **PV (Physical Volume):** LVM이 관리하는 물리 디스크/파티션
- **VG (Volume Group):** 여러 PV를 묶은 스토리지 풀
- **LV (Logical Volume):** VG에서 잘라 쓰는 가상 파티션
- **PE (Physical Extent):** VG 내부의 최소 할당 단위 (기본 4 MiB)
- **Stripe Size:** 스트라이핑 시 한 PV에 쓰는 데이터 단위

### AWS EBS 용어

- **IOPS:** 초당 I/O 작업 수. 작은 랜덤 I/O에 중요
- **Throughput:** 초당 데이터 전송량 (MB/s). 대용량 순차 I/O에 중요
- **Provisioned IOPS:** 사전에 예약한 IOPS. 보장된 성능
- **Burst Credit:** gp2에서 Baseline 이상의 IOPS를 일시적으로 사용할 수 있는 크레딧
- **EBS-Optimized:** EC2와 EBS 간 전용 네트워크 대역폭을 할당한 인스턴스
- **Elastic Volumes:** 볼륨을 사용 중인 상태에서 타입/크기/성능을 변경하는 기능
- **AFR (Annual Failure Rate):** 연간 볼륨 장애율. 내구성 지표
- **VolumeQueueLength:** I/O 대기열 길이. SRE 핵심 모니터링 지표