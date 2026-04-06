# Managing File System

https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_file_systems/index

# 선수지식

## 1. VFS (Virtual File System) 추상화 계층

리눅스 커널은 다양한 파일 시스템(XFS, ext4, NFS 등)을 동일한 인터페이스로 다루기 위해 **VFS**라는 추상화 계층을 사용합니다.

- **추상화:** 사용자가 `read()`나 `write()` 시스템 콜을 호출할 때, 하부에 어떤 파일 시스템이 있는지 몰라도 동일하게 작동하게 해줍니다.
- **공통 인터페이스:** 각 파일 시스템 드라이버는 VFS가 정의한 함수 포인터 구조체를 구현하여 커널에 등록됩니다.

## 2. 파일 시스템의 3대 메타데이터 구조

파일 시스템이 데이터를 관리하기 위해 내부적으로 사용하는 핵심 데이터 구조입니다.

- **Superblock (슈퍼블록):** 파일 시스템 전체의 정보를 담고 있는 '헤더'입니다. 파일 시스템의 크기, 블록 크기, 마운트 횟수, 파일 시스템 상태 등을 저장합니다. 이 영역이 깨지면 마운트 자체가 불가능해집니다.
- **Inode (아이노드):** 파일의 실제 이름과 데이터를 제외한 **모든 메타데이터**를 담고 있습니다. (파일 권한, 소유자, 크기, 수정 시간, 데이터 블록의 위치 등) 모든 파일은 고유한 Inode 번호를 가집니다.
- **Dentry (디렉터리 엔트리):** 파일 이름과 Inode 번호를 매핑해주는 구조체입니다. 리눅스에서 디렉터리는 결국 '파일명-아이노드 번호 리스트'를 담고 있는 특수 파일일 뿐입니다.

## 3. Journaling (저널링)과 원자성 (Atomicity)

현대 파일 시스템의 '신뢰성'을 책임지는 핵심 기술입니다.

- **Write-ahead Logging:** 데이터를 실제 저장소에 쓰기 전에, 수행할 작업을 먼저 **Journal**이라는 별도의 영역에 기록합니다.
- **Crash Recovery:** 데이터 기록 중 전원이 차단되어도, 재부팅 시 저널 로그만 확인하여 작업을 마저 끝내거나(**Replay**) 취소(**Undo**)함으로써 파일 시스템의 일관성을 유지합니다.
- **Metadata vs. Data:** 대부분의 기본 설정은 성능을 위해 메타데이터만 저널링합니다.

## 4. Page Cache와 I/O Path

파일 시스템은 디스크 성능을 보완하기 위해 시스템 메모리(RAM)를 활용합니다.

- **Page Cache:** 최근에 읽거나 쓴 데이터를 메모리에 상주시켜 재요청 시 디스크 접근 없이 즉시 응답합니다.
- **Dirty Page:** 메모리에는 써졌지만 아직 디스크로 플러시(Flush)되지 않은 데이터입니다. `sync` 명령어나 `umount` 시점에 실제 디스크 쓰기가 발생합니다.
- **Delayed Allocation:** 메모리에 데이터를 들고 있다가 마지막 순간에 디스크 공간을 할당함으로써, 데이터가 최대한 연속적인 블록에 저장되도록 유도하여 성능을 높입니다.

## 5. Block Alignment (섹터 정렬)

하드웨어 계층과 소프트웨어 계층의 단위가 일치해야 성능 손실이 없습니다.

- **Physical Sector:** 최신 디스크(Advanced Format)의 물리 섹터는 보통 **4KB**입니다.
- **Logical Block:** 파일 시스템이 다루는 최소 단위(보통 **4KB**)입니다.
- **Alignment:** 파티션의 시작 지점이 물리 섹터 경계와 일치하지 않으면, 하나의 블록을 쓰기 위해 두 개의 물리 섹터를 건드려야 하는 **Write Amplification** 현상이 발생하여 성능이 급격히 저하됩니다.

---

# RHEL 10 파일 시스템 관리

## Chapter 1. 사용 가능한 File System 개요

### 1.1. File System의 유형

Red Hat Enterprise Linux 10은 다양한 File System을 지원합니다. 각 유형의 File System은 서로 다른 문제를 해결하며, 용도에 따라 적합한 것이 다릅니다.

가장 일반적인 수준에서, 사용 가능한 File System은 다음과 같이 분류합니다.

| 유형 | File System | 속성 및 사용 사례 |
| --- | --- | --- |
| Disk / Local FS | **XFS** | RHEL의 기본 File System입니다. 특별한 이유(호환성, 성능 관련 코너 케이스 등)가 없는 한 Local File System으로 XFS를 배포하는 것을 권장합니다. |
| Disk / Local FS | **ext4** | Linux에서 오래된 ext2, ext3에서 발전한 File System으로 친숙합니다. 많은 경우 XFS와 비슷한 성능을 보이지만, 지원하는 File System 및 파일 크기의 상한이 XFS보다 낮습니다. |
| Network / Client-Server FS | **NFS** | 같은 네트워크에 있는 여러 시스템 간에 파일을 공유할 때 사용합니다. |
| Network / Client-Server FS | **SMB** | Microsoft Windows 시스템과 파일을 공유할 때 사용합니다. |

### 1.2. Local File System

Local File System은 단일 로컬 서버에서 실행되며, 직접 연결된 스토리지에서 동작합니다.

예를 들어, 내장 SATA 또는 SAS 디스크에서는 Local File System이 유일한 선택지입니다. 서버에 하드웨어 RAID 컨트롤러가 있는 경우에도 Local File System을 사용합니다. SAN에서 Export된 장치가 공유되지 않는 경우에도 마찬가지입니다.

모든 Local File System은 POSIX 호환이며, `read()`, `write()`, `seek()` 같은 시스템 콜을 지원합니다.

File System을 선택할 때는 File System의 필요 크기, 고유 기능, 워크로드에서의 성능을 기준으로 판단합니다.

### 1.3. XFS File System

XFS는 매우 높은 확장성과 성능을 갖춘 성숙한 64-bit Journaling File System으로, 단일 호스트에서 매우 큰 파일과 File System을 지원합니다. RHEL 10의 기본 File System입니다. XFS는 1990년대 초 SGI에서 개발되었으며, 대규모 서버와 스토리지 어레이에서 오랜 운용 이력을 가지고 있습니다.

**XFS의 주요 특징:**

**Reliability (신뢰성)**

- Metadata Journaling: 시스템 크래시 후 File System 무결성을 보장합니다. File System 작업 기록을 유지하여 시스템 재시작 시 Replay할 수 있습니다.
- 광범위한 런타임 Metadata 일관성 검사를 수행합니다.
- 확장 가능하고 빠른 복구 유틸리티를 제공합니다.
- Quota Journaling: 크래시 후 긴 Quota 일관성 검사를 불필요하게 합니다.

**Scalability 및 Performance**

- 최대 1024 TiB의 File System 크기를 지원합니다.
- 대량의 동시 작업을 지원합니다.
- Free space 관리를 위한 B-tree Indexing을 사용합니다.
- 정교한 Metadata Read-ahead 알고리즘을 적용합니다.
- 스트리밍 비디오 워크로드에 최적화되어 있습니다.

**Allocation Scheme**

- Extent 기반 할당
- Stripe-aware Allocation Policy
- Delayed Allocation
- Space Pre-allocation
- 동적 Inode 할당

**기타 특징**

- Reflink 기반 파일 복사
- 긴밀하게 통합된 백업·복원 유틸리티
- Online Defragmentation
- Online File System Growing (온라인 상태에서 확장 가능)
- Extended Attributes (xattr): 파일당 여러 이름/값 쌍을 연결할 수 있습니다.
- Project 또는 Directory Quota: 디렉터리 트리에 대한 Quota 제한을 적용할 수 있습니다.
- Subsecond Timestamp

**Performance 특성**

XFS는 엔터프라이즈 워크로드가 있는 대규모 시스템에서 높은 성능을 발휘합니다. 대규모 시스템이란 비교적 많은 수의 CPU, 다수의 HBA, 외부 디스크 어레이 연결을 갖춘 시스템을 의미합니다. 멀티스레드·병렬 I/O 워크로드를 가진 소규모 시스템에서도 잘 동작합니다.

### 1.4. ext4 File System

ext4는 ext File System 계열의 4세대입니다. ext4 드라이버는 ext2 및 ext3 File System을 읽고 쓸 수 있지만, ext4 포맷 자체는 ext2/ext3 드라이버와 호환되지 않습니다.

**ext4의 주요 특징:**

- 최대 50 TiB의 File System 크기를 지원합니다.
- Extent 기반 Metadata
- Delayed Allocation
- Journal Checksumming
- 대용량 스토리지 지원

Extent 기반 Metadata와 Delayed Allocation 기능은 File System 내에서 사용된 공간을 더 간결하고 효율적으로 추적하는 방법을 제공합니다. Delayed Allocation은 새로 기록된 사용자 데이터가 디스크로 Flush될 때까지 영구 위치 선택을 지연시킵니다. 이를 통해 더 크고 연속적인 할당이 가능하여 성능이 향상됩니다.

ext4에서 `fsck` 유틸리티를 사용한 File System 복구 시간은 ext2, ext3보다 훨씬 빠릅니다. 일부 복구에서는 최대 6배의 성능 향상이 확인되었습니다.

### 1.5. XFS와 ext4 비교

XFS가 RHEL의 기본 File System입니다. 이 섹션에서는 XFS와 ext4의 사용법 및 기능을 비교합니다.

**Metadata 오류 동작**

ext4에서는 Metadata 오류 발생 시 동작을 설정할 수 있습니다. 기본 동작은 작업을 계속 진행하는 것입니다. XFS는 복구 불가능한 Metadata 오류가 발생하면 File System을 종료하고 `EFSCORRUPTED` 오류를 반환합니다.

**Quota**

ext4에서는 File System 생성 시 또는 기존 File System에서 나중에 Quota를 활성화할 수 있습니다. Mount Option을 사용하여 Quota 적용을 설정합니다.

XFS Quota는 Remountable Option이 아닙니다. 최초 Mount 시에 활성화해야 합니다. XFS File System에서 `quotacheck` 명령을 실행해도 효과가 없습니다. Quota Accounting을 처음 켤 때 XFS가 자동으로 Quota를 검사합니다.

**File System Resize**

XFS는 File System 크기를 줄이는 유틸리티가 없습니다. XFS File System의 크기는 늘리기만 가능합니다. 반면 ext4는 확장과 축소 모두 지원하지만, 축소는 Offline 작업으로만 가능합니다.

**Inode Number**

ext4는 2³² 개 이상의 Inode를 지원하지 않습니다. XFS는 동적 Inode 할당을 지원합니다. XFS File System에서 Inode가 사용할 수 있는 공간은 전체 File System 공간의 백분율로 계산됩니다.

특정 애플리케이션은 XFS File System에서 2³² 보다 큰 Inode Number를 제대로 처리하지 못할 수 있습니다. 이 경우 `-o inode32` 옵션으로 XFS File System을 Mount하면 Inode Number를 2³² 미만으로 강제할 수 있습니다.

> **중요:** 특정 환경에서 필요한 경우가 아니면 `inode32` 옵션을 사용하지 마십시오. 이 옵션은 할당 동작을 변경하며, 하위 디스크 블록에 Inode를 할당할 공간이 없을 때 `ENOSPC` 오류가 발생할 수 있습니다.
> 

### 1.6. Local File System 선택 가이드

애플리케이션 요구 사항에 맞는 File System을 선택하려면, 배포 대상 시스템을 이해해야 합니다. 일반적으로 **ext4를 사용해야 할 특별한 사유가 없다면 XFS를 사용합니다.**

**XFS 권장 상황:**

- 대규모 배포, 특히 큰 파일(수백 MB)과 높은 I/O Concurrency를 처리하는 환경
- 높은 대역폭(200MB/s 초과)과 1000 IOPS 이상의 환경에서 최적의 성능을 발휘합니다.
- 단, ext4에 비해 Metadata 작업에 더 많은 CPU 리소스를 소비하며, File System 축소를 지원하지 않습니다.

**ext4 권장 상황:**

- 소규모 시스템 또는 I/O 대역폭이 제한된 환경
- 단일 스레드, 낮은 I/O 워크로드와 처리량 요구가 적은 환경에서 더 나은 성능을 보입니다.
- Offline 축소를 지원하므로, File System 크기 조정이 필요한 경우 유용합니다.

| 시나리오 | 권장 File System |
| --- | --- |
| 특별한 사유 없음 | XFS |
| 대규모 서버 | XFS |
| 대용량 스토리지 | XFS |
| 대용량 파일 | XFS |
| Multi-threaded I/O | XFS |
| Single-threaded I/O | XFS, ext4 |
| 제한된 I/O (1000 IOPS 미만) | XFS, ext4 |
| 제한된 대역폭 (200MB/s 미만) | XFS, ext4 |
| CPU-bound 워크로드 | XFS, ext4 |
| Offline 축소 지원 필요 | ext4 |

---

## Chapter 6. Persistent Naming Attribute 개요

### 6.1. Non-persistent Naming Attribute의 단점

전통적으로 Linux에서는 `/dev/sd(Major Number)(Minor Number)` 형식의 Non-persistent Name을 사용하여 스토리지 장치를 참조합니다. Major/Minor Number 범위와 관련 sd Name은 장치가 감지될 때 할당됩니다. 이는 장치 감지 순서가 변경되면 연결이 바뀔 수 있음을 의미합니다.

**순서 변경이 발생할 수 있는 상황:**

- 시스템 부팅 과정의 병렬화로 인해 부팅마다 다른 순서로 스토리지 장치가 감지됩니다.
- 디스크가 전원 공급 실패 또는 SCSI 컨트롤러에 응답하지 않으면, 그 디스크는 감지되지 않고 후속 장치들의 sd Name이 변경됩니다. 예를 들어, `sdb`로 인식되던 디스크가 감지되지 않으면 원래 `sdc`였던 디스크가 `sdb`로 나타납니다.
- HBA가 초기화에 실패하면 해당 HBA에 연결된 모든 디스크가 감지되지 않습니다.
- Fibre Channel, iSCSI, FCoE 어댑터로 연결된 디스크가 스토리지 어레이 또는 스위치의 전원 차단으로 접근 불가능할 수 있습니다.

이러한 이유로 `/etc/fstab` 파일 등에서 장치를 참조할 때 Major/Minor Number 또는 sd Name을 사용하는 것은 바람직하지 않습니다. 잘못된 장치가 Mount되어 데이터 손상이 발생할 수 있습니다.

### 6.2. File System Identifier와 Device Identifier

**File System Identifier**

File System Identifier는 Block Device에 생성된 특정 File System에 연결됩니다. 이 식별자는 File System의 일부로 저장됩니다. File System을 다른 장치에 복사해도 동일한 식별자를 유지합니다. 그러나 `mkfs` 유틸리티로 장치를 다시 포맷하면 해당 속성이 사라집니다.

File System Identifier에는 다음이 포함됩니다:

- UUID (Unique Identifier)
- Label

**Device Identifier**

Device Identifier는 Block Device(예: 디스크, Partition)에 연결됩니다. `mkfs`로 장치를 다시 포맷해도 이 속성은 유지됩니다. File System이 아닌 장치 자체에 연결되기 때문입니다.

Device Identifier에는 다음이 포함됩니다:

- WWID (World Wide Identifier)
- Partition UUID (PARTUUID)
- Serial Number

> **권장 사항:** Logical Volume 같은 일부 File System은 여러 장치에 걸쳐 있습니다. 이런 File System에 접근할 때는 Device Identifier보다 File System Identifier를 사용하는 것을 권장합니다.
> 

### 6.3. udev 메커니즘의 /dev/disk/ 장치 이름

udev 메커니즘은 `/dev/disk/` 디렉터리에 다양한 종류의 Persistent Naming Attribute를 제공합니다. 이를 통해 스토리지 장치를 다음 기준으로 참조할 수 있습니다:

- 장치의 콘텐츠 (UUID, Label)
- 고유 식별자 (WWID)
- Serial Number

### 6.3.1. File System Identifier

**UUID Attribute — `/dev/disk/by-uuid/`**

이 디렉터리의 항목은 장치에 저장된 데이터의 UUID로 스토리지 장치를 참조하는 Symbolic Name을 제공합니다.

```
/dev/disk/by-uuid/3e6be9de-8139-11d1-9106-a43f08d823a6
```

`/etc/fstab` 파일에서 다음 구문으로 장치를 참조할 수 있습니다:

```
UUID=3e6be9de-8139-11d1-9106-a43f08d823a6
```

UUID Attribute는 File System 생성 시 설정할 수 있으며, 이후에 변경도 가능합니다.

**Label Attribute — `/dev/disk/by-label/`**

장치에 저장된 데이터의 Label로 스토리지 장치를 참조합니다.

```
/dev/disk/by-label/Boot
```

`/etc/fstab`에서 다음과 같이 사용합니다:

```
LABEL=Boot
```

### 6.3.2. Device Identifier

**WWID Attribute — `/dev/disk/by-id/`**

WWID는 SCSI 표준이 모든 SCSI 장치에 요구하는 Persistent한 시스템 독립적 식별자입니다. 모든 스토리지 장치에 대해 고유하며, 장치 접근 경로와 무관합니다. WWID는 장치의 속성이지만, 장치의 데이터에 저장되지는 않습니다.

RHEL은 WWID 기반 장치 이름과 현재 `/dev/sd` Name 간의 올바른 매핑을 자동으로 유지합니다. 애플리케이션은 `/dev/disk/by-id/` Name을 사용하여 디스크의 데이터를 참조할 수 있으며, 장치 경로가 변경되거나 다른 시스템에서 접근하는 경우에도 유효합니다.

**Partition UUID (PARTUUID) Attribute — `/dev/disk/by-partuuid`**

PARTUUID는 GPT Partition Table에서 정의한 Partition을 식별합니다.

```
/dev/disk/by-partuuid/4cd1448a-01  →  /dev/sda1
/dev/disk/by-partuuid/4cd1448a-02  →  /dev/sda2
```

**Path Attribute — `/dev/disk/by-path/`**

장치에 접근하는 데 사용되는 Hardware Path로 스토리지 장치를 참조합니다. Hardware Path의 일부(PCI ID, Target Port, LUN Number 등)가 변경되면 실패합니다. 따라서 신뢰성이 낮지만, 나중에 교체할 디스크를 식별하거나 특정 위치의 디스크에 스토리지 서비스를 설치할 때 유용할 수 있습니다.

### 6.6. Persistent Naming Attribute 조회

**실습: UUID 및 Label 확인**

```bash
# UUID와 Label을 확인합니다
mr8356@mr8356:~$ sudo lsblk --fs /dev/nvme0n1
NAME            FSTYPE  FSVER   LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
nvme0n1
├─nvme0n1p1     vfat    FAT32         634B-FA82                               586.3M     2% /boot/efi
├─nvme0n1p2     xfs                   994ade5f-d7d6-4cb2-8931-dd04e0a19b6a    631.6M    34% /boot
├─nvme0n1p3     LVM2_me LVM2 00       NKZyIf-M00a-7xgs-04wX-98Yq-XoC5-h0WFWi
│ ├─rl-root     xfs                   25255d13-f12a-423a-b1ab-33bdc270b24a     31.8G     7% /
│ └─rl-swap     swap    1             f792997a-8187-4ce9-8697-6ca9a1da7fc1                  [SWAP]
└─nvme0n1p4     LVM2_me LVM2 00       ePX7NN-2M0J-stDu-VKYj-EQxw-Vt0U-qbSoxH
  └─final_vg-test_lv

mr8356@mr8356:~$ sudo lsblk --fs /dev/nvme0n1p4
NAME            FSTYPE   FSVER  LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
nvme0n1p4       LVM2_mem LVM2 0       ePX7NN-2M0J-stDu-VKYj-EQxw-Vt0U-qbSoxH
└─final_vg-test_lv

```

**실습: PARTUUID 확인**

```bash
sudo lsblk --output +PARTUUID /dev/nvme0n1p1

mr8356@mr8356:~$ sudo lsblk --output +PARTUUID /dev/nvme0n1p1                             NAME      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS PARTUUID
nvme0n1p1 259:1    0  600M  0 part /boot/efi   6f7b3a0b-dd6d-4e48-839f-a25186f861d0

mr8356@mr8356:~$ sudo lsblk --output +PARTUUID /dev/nvme0n1p4
NAME             MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS PARTUUID
nvme0n1p4        259:4    0  6.6G  0 part             92524308-8ae7-410a-8dce-884463215ee6
└─final_vg-test_lv
                 253:2    0    2G  0 lvm
mr8356@mr8356:~$
```

**실습: WWID 확인**

```bash
# 시스템의 모든 스토리지 장치 WWID를 확인합니다
file /dev/disk/by-id/*

mr8356@mr8356:~$ file /dev/disk/by-id/*
/dev/disk/by-id/ata-VMware_Virtual_SATA_CDRW_Drive_01000000000000000001:                      symbolic link to ../../sr0
/dev/disk/by-id/dm-name-final_vg-test_lv:                                                     symbolic link to ../../dm-2
/dev/disk/by-id/dm-name-rl-root:                                                              symbolic link to ../../dm-0
/dev/disk/by-id/dm-name-rl-swap:                                                              symbolic link to ../../dm-1
/dev/disk/by-id/dm-uuid-LVM-2Xz08fU8zn98qS7ryaGo5jCJNxZHRQTy1GS5IL41MUAaRTukLShC30Hnc9JC0fiS: symbolic link to ../../dm-2
/dev/disk/by-id/dm-uuid-LVM-cpaGDyw6EmllprzRuB5LIUm6EJXcXIZ1DnXqan0oXrQCDSTDqRFWjoL1YyRyseyz: symbolic link to ../../dm-1
/dev/disk/by-id/dm-uuid-LVM-cpaGDyw6EmllprzRuB5LIUm6EJXcXIZ1U19mAm2bRcSZ0Yka3nzdI9is0AxiZ5AW: symbolic link to ../../dm-0
/dev/disk/by-id/lvm-pv-uuid-ePX7NN-2M0J-stDu-VKYj-EQxw-Vt0U-qbSoxH:                           symbolic link to ../../nvme0n1p4
/dev/disk/by-id/lvm-pv-uuid-NKZyIf-M00a-7xgs-04wX-98Yq-XoC5-h0WFWi:                           symbolic link to ../../nvme0n1p3
/dev/disk/by-id/nvme-eui.a8335f5994d10927000c2962c94cf69d:                                    symbolic link to ../../nvme0n1
/dev/disk/by-id/nvme-eui.a8335f5994d10927000c2962c94cf69d-part1:                              symbolic link to ../../nvme0n1p1
/dev/disk/by-id/nvme-eui.a8335f5994d10927000c2962c94cf69d-part2:                              symbolic link to ../../nvme0n1p2
/dev/disk/by-id/nvme-eui.a8335f5994d10927000c2962c94cf69d-part3:                              symbolic link to ../../nvme0n1p3
/dev/disk/by-id/nvme-eui.a8335f5994d10927000c2962c94cf69d-part4:                              symbolic link to ../../nvme0n1p4
/dev/disk/by-id/nvme-VMware_Virtual_NVMe_Disk_VMware_NVME_0000:                               symbolic link to ../../nvme0n1
/dev/disk/by-id/nvme-VMware_Virtual_NVMe_Disk_VMware_NVME_0000_1:                             symbolic link to ../../nvme0n1
/dev/disk/by-id/nvme-VMware_Virtual_NVMe_Disk_VMware_NVME_0000_1-part1:                       symbolic link to ../../nvme0n1p1
/dev/disk/by-id/nvme-VMware_Virtual_NVMe_Disk_VMware_NVME_0000_1-part2:                       symbolic link to ../../nvme0n1p2
/dev/disk/by-id/nvme-VMware_Virtual_NVMe_Disk_VMware_NVME_0000_1-part3:                       symbolic link to ../../nvme0n1p3
/dev/disk/by-id/nvme-VMware_Virtual_NVMe_Disk_VMware_NVME_0000_1-part4:                       symbolic link to ../../nvme0n1p4
/dev/disk/by-id/nvme-VMware_Virtual_NVMe_Disk_VMware_NVME_0000-part1:                         symbolic link to ../../nvme0n1p1
/dev/disk/by-id/nvme-VMware_Virtual_NVMe_Disk_VMware_NVME_0000-part2:                         symbolic link to ../../nvme0n1p2
/dev/disk/by-id/nvme-VMware_Virtual_NVMe_Disk_VMware_NVME_0000-part3:                         symbolic link to ../../nvme0n1p3
/dev/disk/by-id/nvme-VMware_Virtual_NVMe_Disk_VMware_NVME_0000-part4:                         symbolic link to ../../nvme0n1p4
```

### 6.7. Persistent Naming Attribute 변경

File System의 UUID 또는 Label을 변경할 수 있습니다.

> **참고:** udev Attribute 변경은 백그라운드에서 이루어지며 시간이 걸릴 수 있습니다. `udevadm settle` 명령은 변경이 완전히 등록될 때까지 대기합니다.
> 

**XFS File System의 UUID/Label 변경 (먼저 Unmount 필요):**

```bash
# UUID를 생성합니다
uuidgen

# XFS File System의 UUID와 Label을 변경합니다
sudo xfs_admin -U 1cdfbc07-1c90-4984-b5ec-f61943f5ea50 -L mydata /dev/nvme0n1p2
sudo udevadm settle
```

**ext4 File System의 UUID/Label 변경:**

```bash
sudo tune2fs -U 1cdfbc07-1c90-4984-b5ec-f61943f5ea50 -L mydata /dev/sdb1
sudo udevadm settle
```

**Swap Volume의 UUID/Label 변경:**

```bash
sudo swaplabel --uuid 1cdfbc07-1c90-4984-b5ec-f61943f5ea50 --label myswap /dev/sdb2
sudo udevadm settle
```

---

## Chapter 9. XFS 시작하기

### 9.2. ext4와 XFS 도구 비교

| 작업 | ext4 | XFS |
| --- | --- | --- |
| File System 생성 | `mkfs.ext4` | `mkfs.xfs` |
| File System 검사 | `e2fsck` | `xfs_repair` |
| File System Resize | `resize2fs` | `xfs_growfs` |
| File System 이미지 저장 | `e2image` | `xfs_metadump` / `xfs_mdrestore` |
| Label/Tune 설정 | `tune2fs` | `xfs_admin` |
| File System 백업 | `tar`, `rsync` | `xfsdump`, `xfsrestore` |
| Quota 관리 | `quota` | `xfs_quota` |
| File Mapping | `filefrag` | `filefrag` |

---

## Chapter 10. XFS File System 생성

### 10.1. mkfs.xfs로 XFS File System 생성

XFS File System을 생성하면 대규모 스토리지 환경에서 높은 성능, 확장성, 고급 기능을 활용할 수 있습니다.

**실습:**

```bash
# 기본 옵션으로 XFS File System을 생성합니다
sudo mkfs.xfs /dev/sdb1

# 기존 File System이 있는 Block Device에는 -f 옵션으로 덮어씁니다
sudo mkfs.xfs -f /dev/sdb1

# 하드웨어 RAID 장치에서 Stripe Geometry를 수동 지정하는 경우
sudo mkfs.xfs -d su=64k,sw=4 /dev/sdb1

# 새 장치 노드가 등록될 때까지 대기합니다
sudo udevadm settle
```

> **참고:** 일반적으로 기본 옵션이 최적입니다. RAID 장치에서 시스템이 Stripe Geometry를 올바르게 감지하면 추가 옵션이 필요 없습니다.
> 

```bash
mr8356@mr8356:~$ sudo mkfs.xfs -f /dev/final_vg/test_lv
meta-data=/dev/final_vg/test_lv  isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=1
         =                       exchange=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1, parent=0
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

mr8356@mr8356:~$ sudo lsblk --fs /dev/final_vg/test_lv
NAME             FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
final_vg-test_lv xfs                9f2f87a5-c387-4542-9b23-653a557a1319
mr8356@mr8356:~$ sudo mkdir -p /mnt/test

mr8356@mr8356:~$ sudo mount /dev/final_vg/test_lv /mnt/test

mr8356@mr8356:~$ df -h /mnt/test
Filesystem                    Size  Used Avail Use% Mounted on
/dev/mapper/final_vg-test_lv  2.0G   71M  1.9G   4% /mnt/test
```

---

## Chapter 13. XFS File System 크기 확장

XFS File System의 크기를 늘려 더 큰 스토리지 용량을 완전히 활용할 수 있습니다. 단, **XFS File System의 크기를 줄이는 것은 현재 불가능합니다.**

> **중요:** 매우 작은 크기에서 훨씬 큰 크기로 File System을 확장하면 Allocation Group이 많이 생성되어 성능 문제가 발생할 수 있습니다. 원래 크기의 최대 10배까지만 확장하는 것을 권장합니다.
> 

### 13.1. xfs_growfs로 XFS File System 확장

**전제 조건:**

- 기본 Block Device가 Resize된 File System을 수용할 수 있는 적절한 크기여야 합니다.
- XFS File System이 Mount되어 있어야 합니다.

**실습:**

```bash
# XFS File System이 Mount된 상태에서 크기를 확장합니다
sudo xfs_growfs /mnt/data -D 새크기

# Block 크기를 확인합니다 (kB 단위)
sudo xfs_info /dev/sdb1
# 출력 중: data = bsize=4096

# -D 옵션 없이 실행하면 기본 장치가 지원하는 최대 크기로 확장합니다
sudo xfs_growfs /mnt/data
```

---

## Chapter 17. File System Mount

### 17.1. Linux Mount 메커니즘

Linux 및 유닉스 계열 운영 체제에서는 다양한 Partition과 이동식 장치(CD, DVD, USB 등)의 File System을 디렉터리 트리의 특정 지점(Mount Point)에 연결(attach)하고 다시 분리(detach)할 수 있습니다. File System이 디렉터리에 Mount되면, 해당 디렉터리의 원래 내용에는 접근할 수 없습니다.

> **참고:** Linux는 이미 File System이 연결된 디렉터리에 다른 File System을 Mount하는 것을 막지 않습니다.
> 

Mount 시 장치를 식별하는 방법:

- **UUID**: 예) `UUID=34795a28-ca6d-4fd8-a347-73671d0c19cb`
- **Volume Label**: 예) `LABEL=home`
- **Non-persistent Block Device 전체 경로**: 예) `/dev/sda3`

`mount` 명령을 장치 이름, 대상 디렉터리, File System 유형 등의 정보 없이 실행하면, `/etc/fstab` 파일의 내용을 확인합니다. `/etc/fstab`에 지정된 File System이라면 다음과 같이 간략하게 Mount할 수 있습니다:

```bash
# Mount Point로 Mount
sudo mount /boot

# Block Device로 Mount
sudo mount /dev/nvme0n1p2
```

### 17.2. 현재 Mount된 File System 조회

**실습:**

```bash
# 모든 Mount된 File System을 조회합니다
findmnt

# 특정 File System 유형만 조회합니다 (예: XFS)
findmnt --types xfs

# 밑에 실습들 이미 한 후에 입력한것입니다.
mr8356@mr8356:~$ findmnt
TARGET                  SOURCE      FSTYPE     OPTIONS
/                       /dev/mapper/rl-root
│                                   xfs        rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
├─/dev                  devtmpfs    devtmpfs   rw,nosuid,seclabel,size=1804084k,nr_inodes=451021,mode=755,inode64
│ ├─/dev/mqueue         mqueue      mqueue     rw,nosuid,nodev,noexec,relatime,seclabel
│ ├─/dev/hugepages      hugetlbfs   hugetlbfs  rw,nosuid,nodev,relatime,seclabel,pagesize=2M
│ ├─/dev/shm            tmpfs       tmpfs      rw,nosuid,nodev,seclabel,inode64
│ └─/dev/pts            devpts      devpts     rw,nosuid,noexec,relatime,seclabel,gid=5,mode=620,ptmxmode=000
├─/sys                  sysfs       sysfs      rw,nosuid,nodev,noexec,relatime,seclabel
│ ├─/sys/fs/selinux     selinuxfs   selinuxfs  rw,nosuid,noexec,relatime
│ ├─/sys/kernel/debug   debugfs     debugfs    rw,nosuid,nodev,noexec,relatime,seclabel
│ ├─/sys/kernel/tracing tracefs     tracefs    rw,nosuid,nodev,noexec,relatime,seclabel
│ ├─/sys/kernel/security
│ │                     securityfs  securityfs rw,nosuid,nodev,noexec,relatime
│ ├─/sys/fs/cgroup      cgroup2     cgroup2    rw,nosuid,nodev,noexec,relatime,seclabel,nsdelegate,memory_recursiveprot
│ ├─/sys/fs/pstore      pstore      pstore     rw,nosuid,nodev,noexec,relatime,seclabel
│ ├─/sys/firmware/efi/efivars
│ │                     efivarfs    efivarfs   rw,nosuid,nodev,noexec,relatime
│ ├─/sys/fs/bpf         bpf         bpf        rw,nosuid,nodev,noexec,relatime,mode=700
│ ├─/sys/kernel/config  configfs    configfs   rw,nosuid,nodev,noexec,relatime
│ └─/sys/fs/fuse/connections
│                       fusectl     fusectl    rw,nosuid,nodev,noexec,relatime
├─/proc                 proc        proc       rw,nosuid,nodev,noexec,relatime
│ └─/proc/sys/fs/binfmt_misc
│                       systemd-1   autofs     rw,relatime,fd=36,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=7267
├─/run                  tmpfs       tmpfs      rw,nosuid,nodev,seclabel,size=732328k,nr_inodes=819200,mode=755,inode64
│ ├─/run/user/1000      tmpfs       tmpfs      rw,nosuid,nodev,relatime,seclabel,size=366160k,nr_inodes=91540,mode=700,uid=100
│ ├─/run/credentials/systemd-journald.service
│ │                     tmpfs       tmpfs      ro,nosuid,nodev,noexec,relatime,nosymfollow,seclabel,size=1024k,nr_inodes=1024,
│ └─/run/credentials/getty@tty1.service
│                       tmpfs       tmpfs      ro,nosuid,nodev,noexec,relatime,nosymfollow,seclabel,size=1024k,nr_inodes=1024,
├─/boot                 /dev/nvme0n1p2
│ │                                 xfs        rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
│ ├─/boot/efi           /dev/nvme0n1p1
│ │                                 vfat       rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=ascii,shortname=winnt,
│ └─/boot               /dev/nvme0n1p2
│   │                               xfs        rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
│   └─/boot             /dev/nvme0n1p2
│                                   xfs        rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
└─/mnt/test             /dev/mapper/final_vg-test_lv
                                    xfs        rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
mr8356@mr8356:~$ findmnt --types xfs                                                                                                                                    TARGET      SOURCE                       FSTYPE OPTIONS
/           /dev/mapper/rl-root          xfs    rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
├─/boot     /dev/nvme0n1p2               xfs    rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
│ └─/boot   /dev/nvme0n1p2               xfs    rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
│   └─/boot /dev/nvme0n1p2               xfs    rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
└─/mnt/test /dev/mapper/final_vg-test_lv xfs    rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
```

### 17.3. mount로 File System Mount하기

**실습:**

```bash
# 선택한 Mount Point에 이미 Mount된 것이 없는지 확인합니다
findmnt /mnt/data

# UUID로 Local XFS File System을 Mount합니다
sudo mount UUID=ea74bbec-536d-490c-b8d9-5b40bbd7545b /mnt/data

# File System 유형을 자동 인식하지 못하는 경우 --types 옵션을 사용합니다
sudo mount --types xfs /dev/sdb1 /mnt/data

# Remote NFS File System을 Mount하는 경우
sudo mount --types nfs4 host:/remote-export /mnt/nfs

mr8356@mr8356:~$ findmnt /mnt/test

mr8356@mr8356:~$ sudo lsblk --fs /dev/final_vg/test_lv
NAME   FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
final_vg-test_lv
       xfs                9f2f87a5-c387-4542-9b23-653a557a1319

mr8356@mr8356:~$ sudo mount UUID=9f2f87a5-c387-4542-9b23-653a557a1319 /mnt/test

mr8356@mr8356:~$ findmnt /mnt/test
TARGET    SOURCE                       FSTYPE OPTIONS
/mnt/test /dev/mapper/final_vg-test_lv xfs    rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
```

### 17.4. Mount Point 이동

```bash
# Mount된 File System의 Mount Point를 변경합니다
sudo mount --move /mnt/userdirs /home

# 이동 결과를 확인합니다
findmnt
ls /mnt/userdirs   # 비어 있어야 합니다
ls /home            # 데이터가 보여야 합니다
```

### 17.5. umount로 File System Unmount하기

```bash
# Mount Point로 Unmount합니다
sudo umount /mnt/data

# 또는 장치로 Unmount합니다
sudo umount /dev/sdb1
```

"target is busy" 오류가 발생하면, 해당 File System을 사용 중인 프로세스가 있다는 의미입니다:

```bash
# 어떤 프로세스가 접근 중인지 확인합니다
fuser --mount /mnt/data

# 해당 프로세스를 중지한 후 다시 Unmount를 시도합니다
```

### 17.7. 주요 Mount Option

```bash
sudo mount --options option1,option2 /dev/sdb1 /mnt/data
```

| Option | 설명 |
| --- | --- |
| `async` | File System에서 비동기 I/O를 활성화합니다. |
| `auto` | `mount -a` 명령으로 자동 Mount됩니다. |
| `defaults` | `async,auto,dev,exec,nouser,rw,suid` 옵션의 Alias입니다. |
| `exec` | 해당 File System에서 Binary 파일 실행을 허용합니다. |
| `loop` | Image를 Loop Device로 Mount합니다. |
| `noauto` | `mount -a` 명령으로 자동 Mount되지 않습니다. |
| `noexec` | 해당 File System에서 Binary 파일 실행을 금지합니다. |
| `nouser` | 일반 사용자(root 외)가 Mount/Unmount할 수 없습니다. |
| `remount` | 이미 Mount된 File System을 다시 Mount합니다. |
| `ro` | 읽기 전용으로 Mount합니다. |
| `rw` | 읽기/쓰기 모두 가능하게 Mount합니다. |
| `user` | 일반 사용자(root 외)도 Mount/Unmount할 수 있습니다. |

---

## Chapter 19. File System의 Persistent Mount

### 19.1. /etc/fstab 파일

`/etc/fstab` 설정 파일을 사용하여 File System의 Persistent Mount Point를 제어합니다. 이 파일의 각 행은 하나의 Mount Point를 정의합니다.

각 행에는 공백으로 구분된 다음 필드가 포함됩니다:

| 필드 | 설명 |
| --- | --- |
| Block Device | Persistent Attribute(UUID 등) 또는 `/dev` 경로 |
| Mount Point | 장치가 Mount될 디렉터리 |
| File System | 장치의 File System 유형 |
| Mount Option | `defaults`, `x-systemd.*` 등 |
| Backup | dump 유틸리티를 위한 백업 옵션 |
| Check | fsck 유틸리티를 위한 검사 순서 |

**예시: `/etc/fstab`의 /boot 항목**

```
UUID=ea74bbec-536d-490c-b8d9-5b40bbd7545b  /boot  xfs  defaults  0  0
```

> **참고:** systemd는 `/etc/fstab` 항목에서 자동으로 Mount Unit을 생성합니다.
> 

### 19.2. /etc/fstab에 File System 추가

**실습:**

```bash
mr8356@mr8356:~$ sudo lsblk --fs /dev/final_vg/test_lv                                   NAME   FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
final_vg-test_lv
       xfs                9f2f87a5-c387-4542-9b23-653a557a1319

mr8356@mr8356:~$ echo "UUID=9f2f87a5-c387-4542-9b23-653a557a1319 /mnt/test xfs defaults 0 0" | sudo tee -a /etc/fstab
UUID=9f2f87a5-c387-4542-9b23-653a557a1319 /mnt/test xfs defaults 0 0

mr8356@mr8356:~$ cat /etc/fstab | tail -n 1
UUID=9f2f87a5-c387-4542-9b23-653a557a1319 /mnt/test xfs defaults 0 0

mr8356@mr8356:~$ sudo systemctl daemon-reload

mr8356@mr8356:~$ sudo mount /mnt/test

mr8356@mr8356:~$ df -h /mnt/test
Filesystem                    Size  Used Avail Use% Mounted on
/dev/mapper/final_vg-test_lv  2.0G   71M  1.9G   4% /mnt/test

mr8356@mr8356:~$ findmnt /mnt/test
TARGET SOURCE                   FSTYPE OPTIONS
/mnt/test
       /dev/mapper/final_vg-test_lv
                                xfs    rw,relatime,seclabel,attr2,inode64,logbufs=8,logbs
```

---