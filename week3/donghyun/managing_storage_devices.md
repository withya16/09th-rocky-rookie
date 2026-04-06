# Managing Storage Devices

https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_storage_devices/index

# 선수지식

## 1. 리눅스 장치 관리의 대원칙: "Everything is a File"

리눅스 커널은 모든 하드웨어 장치를 **파일(File)**로 취급합니다.

- **Device Nodes:** 하드 디스크, 파티션 등은 `/dev` 디렉토리 아래에 파일 형태로 존재합니다.
- **Block Device:** 데이터를 **Block** 단위로 읽고 쓰는 장치를 의미합니다. 무작위 접근(Random Access)이 가능하며, 디스크가 대표적인 예입니다.
- **Major/Minor Number:** 커널이 장치를 식별하는 숫자입니다.
    - **Major:** 장치의 종류(드라이버)를 식별합니다.
    - **Minor:** 같은 종류 내에서 개별 장치(0=전체 디스크, 1=첫 번째 파티션 등)를 식별합니다.

---

## 2. 저장 단위의 혼란 방지: Binary vs. Decimal

`parted`나 `fdisk`를 사용할 때 가장 많이 실수하는 부분입니다. RHEL 도큐먼트는 이 차이를 엄격하게 구분합니다.

- **Decimal (SI 단위):** 10의 거듭제곱을 사용합니다. 제조사(Samsung, SK Hynix 등)에서 용량을 표기할 때 씁니다.
    - KB = 10^3 = 1,000
- **Binary (IEC 단위):** 2의 거듭제곱을 사용합니다. 리눅스 커널과 운영체제가 실제 계산할 때 씁니다.
    - KiB (Kibibyte) = 2^10 = 1,024

> **SRE 팁:** 파티션을 나눌 때 `100MB`라고 입력하면 실제로는 `100MiB`보다 작게 잡힐 수 있으므로, 정확한 설계를 위해 항상 **`MiB`, `GiB`** 단위를 사용하는 습관을 들여야 합니다.
> 

---

## 3. 디스크 구조의 기본 용어 (Geometry)

물리적인 디스크 공간을 논리적으로 이해하기 위한 용어들입니다.

- **Sector:** 디스크의 최소 물리적 저장 단위입니다. 과거에는 **512 bytes**였으나, 최신 디스크는 **4,096 bytes (4K)**인 경우가 많습니다 (Advanced Format).
- **Logical Block Addressing (LBA):** 현대 리눅스에서 섹터에 접근하는 방식입니다. 섹터마다 0번부터 순차적으로 번호를 매겨 관리합니다.
- **Alignment (정렬):** 파티션의 시작점이 섹터 경계(보통 1MiB 지점)와 딱 맞아야 합니다. 정렬이 어긋나면 성능이 급격히 저하됩니다. (`parted`는 이를 자동으로 최적화해 줍니다.)

---

## 4. 커널 서브시스템과 udev의 역할

- **Kernel Space:** 디스크 드라이버가 물리 장치와 통신하여 `/dev/sda` 같은 이름을 만듭니다.
- **User Space (udev):** 커널이 장치를 인식하면 **udev**라는 데몬이 이벤트를 받아 **Persistent naming** (UUID 등)을 생성하고 심볼릭 링크를 `/dev/disk/` 아래에 만듭니다.
- **Device Mapper (DM):** 커널 내에서 물리 장치를 가상 장치로 매핑해주는 프레임워크입니다. **LVM, Multipath, 암호화(LUKS)** 등이 모두 이 위에서 동작합니다.

---

## 5. I/O 흐름과 캐싱 (Data Integrity)

- **Page Cache:** 리눅스는 성능 향상을 위해 디스크 쓰기 요청을 즉시 처리하지 않고 메모리에 담아둡니다 (**Dirty Pages**).
- **Flush (Sync):** 메모리에 있는 데이터를 실제 물리 디스크로 기록하는 작업입니다. 장치를 제거하기 전 반드시 거쳐야 하는 과정입니다.
- **Mount & VFS:** 파일 시스템은 **VFS (Virtual File System)**라는 추상화 계층을 통해 마운트됩니다. 사용자는 어떤 파일 시스템(XFS, ext4)인지 상관없이 동일한 인터페이스로 파일을 다룰 수 있습니다.

---

## 6. Partition Table의 본질적 차이

- **MBR (Master Boot Record):**
    - 전통적인 방식입니다.
    - 디스크의 **첫 512 bytes**에 파티션 정보가 담깁니다.
    - 공간이 좁아 파티션을 4개까지만 만들 수 있고, 주소 할당 비트 한계로 **2TiB**까지만 인식합니다.
- **GPT (GUID Partition Table):**
    - 현대적인 표준입니다.
    - 파티션 정보를 디스크 앞뒤에 여러 복사본으로 저장하여 안전합니다.
    - **128개** 이상의 파티션과 제타바이트(ZiB)급 용량을 지원합니다.

---

# Red Hat Enterprise Linux 스토리지 가이드 (1, 2, 3, 4, 5, 13, 20장)

Red Hat Enterprise Linux에서 사용할 수 있는 Local, Remote, 그리고 Cluster-based storage 옵션들을 탐색합니다. 여기서는 Storage architecture를 이해하기 위해 시스템에 직접 연결된 Storage devices와 LAN, 인터넷, 또는 Fibre Channel 네트워크를 통해 접근하는 Remote storage를 다룹니다.

- **Local storage:** Storage devices가 시스템에 설치되어 있거나 직접 연결되어 있음을 의미합니다.
- **Remote storage:** LAN, 인터넷, 또는 Fibre Channel 네트워크를 사용하여 Devices에 접근합니다.

## 1.1. Local storage overview (로컬 스토리지 개요)

**Local storage**는 시스템에 직접 설치되거나 연결된 Storage devices를 말합니다. 여기에는 네트워크 연결 없이도 관리할 수 있는 Disk partitions, Logical volumes, 그리고 File systems가 포함됩니다. RHEL 10은 다양한 Local storage 옵션을 제공합니다.

![](https://access.redhat.com/webassets/avalon/d/Red_Hat_Enterprise_Linux-10-Managing_storage_devices-en-US/images/72eec6a83fb27ddb3e44bc5ab1a3a3a8/High-Level-RHEL-Storage-Diagram.png)

### **Basic disk administration (기본 디스크 관리)**

`parted`와 `fdisk`를 사용하여 Disk partitions를 생성, 수정, 삭제 및 조회할 수 있습니다. 다음은 Partitioning layout의 표준입니다:

- **GUID Partition Table (GPT):** Globally unique identifier (GUID)를 사용하며, 고유한 Disk 및 Partition GUID를 제공합니다.
- **Master Boot Record (MBR):** BIOS 기반 컴퓨터에서 사용됩니다. Primary, Extended, Logical partitions를 만들 수 있습니다.

### **Storage consumption options (스토리지 소비 옵션)**

- **Non-Volatile Dual In-line Memory Modules (NVDIMM) Management:** Memory와 Storage의 결합입니다. 시스템에 연결된 NVDIMM devices에서 다양한 유형의 Storage를 활성화하고 관리할 수 있습니다.
- **Block Storage Management:** 데이터가 Block 형태로 저장되며, 각 Block은 고유한 식별자를 가집니다.
- **File Storage:** 데이터가 Local system에 File 단위로 저장됩니다. 이 데이터는 XFS (기본값) 또는 ext4를 사용하여 로컬에서 접근할 수 있고, NFS와 SMB를 사용하여 네트워크를 통해 접근할 수 있습니다.

### **Logical volumes (논리 볼륨)**

- **Logical Volume Manager (LVM):** Physical devices로부터 Logical devices를 생성합니다. Logical volume (LV)는 Physical volumes (PV)와 Volume groups (VG)의 조합입니다.
- **Virtual Data Optimizer (VDO):** Deduplication(중복 제거), Compression(압축), 그리고 Thin provisioning을 사용하여 데이터 크기를 줄이는 데 사용됩니다. VDO 아래에 LV를 사용하면 다음 작업에 도움이 됩니다:
    - VDO volume 확장
    - 여러 Devices에 걸쳐 VDO volume 분산 (Spanning)

### **Local file systems (로컬 파일 시스템)**

- **XFS:** 기본 RHEL File system.
- **Ext4:** 레거시 File system.
- **Stratis:** 고급 Storage 기능을 지원하는 하이브리드 User-and-kernel 기반의 Local storage management system입니다.

## 1.2. Remote storage overview (원격 스토리지 개요)

**Remote storage**는 LAN, 인터넷, 또는 Fibre Channel과 같은 네트워크 연결을 통해 Storage devices에 대한 접근을 제공합니다. 이를 통해 Storage resources를 중앙 집중화하고 여러 시스템에서 공유할 수 있습니다.

**Storage connectivity options (스토리지 연결 옵션)**

- **iSCSI:** RHEL 10은 `targetcli` 도구를 사용하여 iSCSI storage interconnects를 추가, 삭제, 조회 및 모니터링합니다.
- **Fibre Channel (FC):** `lpfc`, `qla2xxx`, `zfcp` 드라이버를 제공합니다.
- **Non-volatile Memory Express (NVMe):** Host software 유틸리티가 Solid state drives (SSD)와 통신할 수 있게 해주는 인터페이스입니다. NVMe over fabrics 구성에 RDMA, FC, TCP 등을 사용할 수 있습니다.
- **Device Mapper multipathing (DM Multipath):** Server nodes와 Storage arrays 간의 여러 I/O paths를 단일 Device로 구성할 수 있게 해줍니다.
- **Network file system:** NFS, SMB.

---

## Chapter 2. Persistent naming attributes (영구 명명 속성)

Persistent naming attributes (PNAs)는 하드웨어의 고유한 특성을 기반으로 안정적이고 예측 가능한 Device 식별을 제공하여, 시스템 재부팅이나 하드웨어 변경 시에도 일관된 Naming을 보장합니다.

**Traditional device names (전통적인 디바이스 이름)**

Linux kernel은 시스템에 나타나는 순서대로 Traditional device names를 할당합니다. (예: 첫 번째는 `/dev/sda`, 두 번째는 `/dev/sdb`). 직관적이지만 하드웨어 구성이 변경되거나 재부팅되면 이름이 바뀔 수 있어 Scripting 및 Configuration에 어려움을 줍니다.

**Persistent naming attributes (PNAs)**

PNAs는 Storage devices의 고유 특성을 기반으로 하므로 재부팅을 해도 안정적입니다. `udev` rules에 의해 제어되며, 최종적으로 `/dev/disk` 디렉토리에 Device links를 생성하는 데 사용됩니다.

- `/dev/disk/by-id`: Vendor/Model/Serial string 조합 등 하드웨어 속성 기반.
- `/dev/disk/by-path`: 물리적 연결 위치 기반.
- `/dev/disk/by-uuid` & `by-partuuid`: 자동 생성된 고유 식별자(UUID) 기반.

### 2.1. File systems 및 Block devices 식별을 위한 Persistent attributes

- **UUID:** File systems와 Storage block devices를 고유하게 식별합니다. 마운트/언마운트 하거나 디바이스를 다시 연결해도 유지됩니다. LVM(PVUUID, LVUUID)이나 MD 장치 등에도 쓰입니다.
- **Label:** 사용자가 할당한 이름입니다.
- **WWID (World Wide Identifier):** SAN 환경(Fibre Channel 등)에서 전역적으로 고유하게 사용되는 식별자입니다.
- **Serial string:** 제조사가 할당한 고유 문자열입니다.

### 2.2. udev device naming rules

`udev` 서브시스템은 디바이스에 Persistent names를 할당하는 규칙을 정의합니다.

- 기본 규칙: `/usr/lib/udev/rules.d/`
- 사용자 지정 커스텀 규칙: `/etc/udev/rules.d/` (우선순위가 더 높음)

### 🛠️ [실습] 기존 디바이스의 Device links 값 얻기

Device(`nvme0n1`)가 시스템에 연결된 상태에서 현재 `udev` 데이터베이스 값을 조회해 봅니다.

`# 1. 특정 디바이스의 모든 DEVLINKS 심볼릭 링크 확인
udevadm info --name /dev/nvme0n1 --query property --property DEVLINKS --value

# 2. 이 링크들이 가리키는 원본 커널 이름(DEVNAME) 확인
udevadm info --name /dev/nvme0n1 --query property --property DEVNAME --value`

```bash
mr8356@mr8356:~$ udevadm info --name /dev/nvme0n1 --query property --property DEVLINKS --value
/dev/disk/by-diskseq/1 /dev/disk/by-id/nvme-VMware_Virtual_NVMe_Disk_VMware_NVM>
lines 1-1/1 (END)

mr8356@mr8356:~$ udevadm info --name /dev/nvme0n1 --query property --property DEVNAME --value
/dev/nvme0n1
```

---

## Chapter 3. Disk partitions (디스크 파티션)

Disk partitions는 물리적 Storage device를 여러 논리적 영역으로 나누어 OS가 독립적인 디스크로 취급할 수 있게 합니다.

![](https://access.redhat.com/webassets/avalon/d/Red_Hat_Enterprise_Linux-10-Managing_storage_devices-en-US/images/6e469a6b3b73066fcd8340d962618004/gpt-partition.png)

### 3.2. Partition table types 비교

- **MBR (Master Boot Record):** 최대 4개의 Primary partitions, 또는 3개의 Primary와 1개의 Extended partition(내부에 여러 Logical partitions) 지원. 최대 2 TiB 한계.
- **GPT (GUID Partition Table):** 최대 128개의 파티션 지원. 512b 섹터 기준 8 ZiB (4k 섹터 기준 64 ZiB) 지원. MBR의 2TB 한계를 극복하기 위해 설계됨.

![](https://access.redhat.com/webassets/avalon/d/Red_Hat_Enterprise_Linux-10-Managing_storage_devices-en-US/images/74379cb1696956d9d20f229b0c0c8817/unused-partitioned-drive.png)

![](https://access.redhat.com/webassets/avalon/d/Red_Hat_Enterprise_Linux-10-Managing_storage_devices-en-US/images/9429aaebbe148b973e415eb641325e53/dos-single-partition.png)

![](https://access.redhat.com/webassets/avalon/d/Red_Hat_Enterprise_Linux-10-Managing_storage_devices-en-US/images/3a176ae803526f8a94f32819086f6a75/extended-partitions.png)

### 3.8. Partition naming scheme

RHEL은 `/dev/xxyN` 형태의 네이밍 규칙을 사용합니다.

- `/dev/sda1`: 첫 번째 하드 디스크(`a`)의 첫 번째 파티션(`1`).
- `/dev/nvme0n1p1`: NVMe 드라이브의 첫 번째 파티션.

---

## Chapter 4. Getting started with partitions (파티션 시작하기)

### 🛠️ [실습] parted로 파티션 테이블 생성 및 관리

`parted` 유틸리티를 사용하여 디스크를 자르고 관리하는 실전 명령어입니다. (**경고:** 파티션 테이블을 새로 만들면 기존 데이터는 모두 삭제됩니다.)

`# 1. parted 대화형 쉘 실행 (대상: /dev/sda)
parted /dev/sda

# 2. 현재 상태 확인 (기존 파티션 테이블이 있는지 확인)
(parted) print

# 3. 새로운 파티션 테이블(GPT 또는 MBR) 생성
(parted) mklabel gpt   # (또는 msdos)

# 4. 새로운 파티션 생성 (예: 1024MiB에서 시작해 2048MiB에서 끝나는 xfs 파티션)
(parted) mkpart primary xfs 1024MiB 2048MiB

# 5. 파티션 확인 후 종료
(parted) print
(parted) quit

# 6. 커널이 새 파티션을 잘 인식했는지 확인
cat /proc/partitions`

```bash
mr8356@mr8356:~$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0          11:0    1  8.1G  0 rom
nvme0n1     259:0    0   50G  0 disk
├─nvme0n1p1 259:1    0  600M  0 part /boot/efi
├─nvme0n1p2 259:2    0    1G  0 part /boot
└─nvme0n1p3 259:3    0 38.4G  0 part
  ├─rl-root 253:0    0 34.5G  0 lvm  /
  └─rl-swap 253:1    0  3.9G  0 lvm  [SWAP]

mr8356@mr8356:~$ sudo parted /dev/nvme0n1
[sudo] password for mr8356:
GNU Parted 3.6
Using /dev/nvme0n1
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted)

(parted) print
Model: VMware Virtual NVMe Disk (nvme)
Disk /dev/nvme0n1: 53.7GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name                  Flags
 1      1049kB  630MB   629MB   fat32        EFI System Partition  boot, esp
 2      630MB   1704MB  1074MB  xfs                                bls_boot
 3      1704MB  42.9GB  41.2GB                                     lvm

(parted)

(parted) mkpart primary xfs 42.9GB 47GB

(parted) print
Model: VMware Virtual NVMe Disk (nvme)
Disk /dev/nvme0n1: 53.7GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name                  Flags
 1      1049kB  630MB   629MB   fat32        EFI System Partition  boot, esp
 2      630MB   1704MB  1074MB  xfs                                bls_boot
 3      1704MB  42.9GB  41.2GB                                     lvm
 4      42.9GB  47.0GB  4051MB  xfs          primary

(parted)

mr8356@mr8356:~$ sudo partprobe /dev/nvme0n1

mr8356@mr8356:~$ cat /proc/partitions
major minor  #blocks  name

 259        0   52428800 nvme0n1
 259        1     614400 nvme0n1p1
 259        2    1048576 nvme0n1p2
 259        3   40279040 nvme0n1p3
 259        4    3955712 nvme0n1p4
  11        0    8504064 sr0
 253        0   36151296 dm-0
 253        1    4120576 dm-1
```

### 🛠️ [실습] fdisk로 파티션 타입(Type) 변경하기

생성된 파티션의 용도를 커널에 알려주기 위해 Type(Flag)을 변경합니다.

`# 1. fdisk 실행
fdisk /dev/sda

# 2. 파티션 목록 및 Type(Id) 확인
Command (m for help): print

# 3. Type 변경 모드 진입 (예: 2번 파티션 선택)
Command (m for help): type
Partition number (1-3, default 3): 2

# 4. Type 목록 확인 후, Linux LVM(8e) 이나 원하는 타입으로 지정
Hex code or alias: L
Hex code or alias: 8e

# 5. 저장(write)하고 종료
Command (m for help): w`

```bash
mr8356@mr8356:~$ sudo fdisk /dev/nvme0n1
[sudo] password for mr8356:

Welcome to fdisk (util-linux 2.40.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

This disk is currently in use - repartitioning is probably a bad idea.
It's recommended to umount all file systems, and swapoff all swap
partitions on this disk.

Command (m for help): p

Disk /dev/nvme0n1: 50 GiB, 53687091200 bytes, 104857600 sectors
Disk model: VMware Virtual NVMe Disk
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 9D8CCD7F-D60B-4016-B709-39C25A16A837

Device            Start      End  Sectors  Size Type
/dev/nvme0n1p1     2048  1230847  1228800  600M EFI System
/dev/nvme0n1p2  1230848  3327999  2097152    1G Linux extended boot
/dev/nvme0n1p3  3328000 83886079 80558080 38.4G Linux LVM
/dev/nvme0n1p4 83886080 91797503  7911424  3.8G Linux filesystem

Command (m for help): t

Partition number (1-4, default 4): 4

Partition type or alias (type L to list all): L
  1 EFI System                     C12A7328-F81F-11D2-BA4B-00A0C93EC93B
  2 MBR partition scheme           024DEE41-33E7-11D3-9D69-0008C781F39F
  3 Intel Fast Flash               D3BFE2DE-3DAF-11DF-BA40-E3A556D89593
  4 BIOS boot                      21686148-6449-6E6F-744E-656564454649
  5 Sony boot partition            F4019732-066E-4E12-8273-346C5641494F
  6 Lenovo boot partition          BFBFAFE7-A34F-448A-9A5B-6213EB736C22
  7 PowerPC PReP boot              9E1A2D38-C612-4316-AA26-8B49521E5A8B
  8 ONIE boot                      7412F7D5-A156-4B13-81DC-867174929325
  9 ONIE config                    D4E6E2CD-4469-46F3-B5CB-1BFF57AFC149
 10 Microsoft reserved             E3C9E316-0B5C-4DB8-817D-F92DF00215AE
 11 Microsoft basic data           EBD0A0A2-B9E5-4433-87C0-68B6B72699C7
 12 Microsoft LDM metadata         5808C8AA-7E8F-42E0-85D2-E1E90434CFB3
 13 Microsoft LDM data             AF9B60A0-1431-4F62-BC68-3311714A69AD
 14 Windows recovery environment   DE94BBA4-06D1-4D40-A16A-BFD50179D6AC
 15 IBM General Parallel Fs        37AFFC90-EF7D-4E96-91C3-2D7AE055B174
 16 Microsoft Storage Spaces       E75CAF8F-F680-4CEE-AFA3-B001E56EFC2D
 17 HP-UX data                     75894C1E-3AEB-11D3-B7C1-7B03A0000000
 18 HP-UX service                  E2A1E728-32E3-11D6-A682-7B03A0000000
 19 Linux swap                     0657FD6D-A4AB-43C4-84E5-0933C84B4F4F
 20 Linux filesystem               0FC63DAF-8483-4772-8E79-3D69D8477DE4
 21 Linux server data              3B8F8425-20E0-4F3B-907F-1A25A76F98E8
 22 Linux root (x86)               44479540-F297-41B2-9AF7-D131D5F0458A
 23 Linux root (x86-64)            4F68BCE3-E8CD-4DB1-96E7-FBCAF984B709
 24 Linux root (Alpha)             6523F8AE-3EB1-4E2A-A05A-18B695AE656F
 25 Linux root (ARC)               D27F46ED-2919-4CB8-BD25-9531F3C16534
 26 Linux root (ARM)               69DAD710-2CE4-4E3C-B16C-21A1D49ABED3
 27 Linux root (ARM-64)            B921B045-1DF0-41C3-AF44-4C6F280D3FAE
 28 Linux root (IA-64)             993D8D3D-F80E-4225-855A-9DAF8ED7EA97
 29 Linux root (LoongArch-64)      77055800-792C-4F94-B39A-98C91B762BB6
 30 Linux root (MIPS-32 LE)        37C58C8A-D913-4156-A25F-48B1B64E07F0
 31 Linux root (MIPS-64 LE)        700BDA43-7A34-4507-B179-EEB93D7A7CA3
 32 Linux root (HPPA/PARISC)       1AACDB3B-5444-4138-BD9E-E5C2239B2346
 33 Linux root (PPC)               1DE3F1EF-FA98-47B5-8DCD-4A860A654D78
 34 Linux root (PPC64)             912ADE1D-A839-4913-8964-A10EEE08FBD2
 35 Linux root (PPC64LE)           C31C45E6-3F39-412E-80FB-4809C4980599
 36 Linux root (RISC-V-32)         60D5A7FE-8E7D-435C-B714-3DD8162144E1
 37 Linux root (RISC-V-64)         72EC70A6-CF74-40E6-BD49-4BDA08E8F224
 38 Linux root (S390)              08A7ACEA-624C-4A20-91E8-6E0FA67D23F9
 39 Linux root (S390X)             5EEAD9A9-FE09-4A1E-A1D7-520D00531306
 40 Linux root (TILE-Gx)           C50CDD70-3862-4CC3-90E1-809A8C93EE2C
 41 Linux reserved                 8DA63339-0007-60C0-C436-083AC8230908
 42 Linux home                     933AC7E1-2EB4-4F13-B844-0E14E2AEF915
 
Partition type or alias (type L to list all): 19

Changed type of partition 'Linux filesystem' to 'Linux swap'.

Command (m for help): p
Disk /dev/nvme0n1: 50 GiB, 53687091200 bytes, 104857600 sectors
Disk model: VMware Virtual NVMe Disk
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 9D8CCD7F-D60B-4016-B709-39C25A16A837

Device            Start      End  Sectors  Size Type
/dev/nvme0n1p1     2048  1230847  1228800  600M EFI System
/dev/nvme0n1p2  1230848  3327999  2097152    1G Linux extended boot
/dev/nvme0n1p3  3328000 83886079 80558080 38.4G Linux LVM
/dev/nvme0n1p4 83886080 91797503  7911424  3.8G Linux swap

Command (m for help): t
Partition number (1-4, default 4): 4
Partition type or alias (type L to list all): 20

Changed type of partition 'Linux swap' to 'Linux filesystem'.

Command (m for help):
```

### 🛠️ [실습] parted로 파티션 크기 조정 및 삭제

`# 파티션 크기 늘리기 (예: 1번 파티션의 끝점을 2GiB로 늘림)
parted /dev/sda resizepart 1 2GiB

# 파티션 삭제 (예: 1번 파티션 삭제)
parted /dev/sda rm 1`

```bash
mr8356@mr8356:~$ sudo parted /dev/nvme0n1
GNU Parted 3.6
Using /dev/nvme0n1
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print
Model: VMware Virtual NVMe Disk (nvme)
Disk /dev/nvme0n1: 53.7GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name                  Flags
 1      1049kB  630MB   629MB   fat32        EFI System Partition  boot, esp
 2      630MB   1704MB  1074MB  xfs                                bls_boot
 3      1704MB  42.9GB  41.2GB                                     lvm
 4      42.9GB  47.0GB  4051MB               primary
 
(parted) resizepart 4 49GiB

(parted) print
Model: VMware Virtual NVMe Disk (nvme)
Disk /dev/nvme0n1: 53.7GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name                  Flags
 1      1049kB  630MB   629MB   fat32        EFI System Partition  boot, esp
 2      630MB   1704MB  1074MB  xfs                                bls_boot
 3      1704MB  42.9GB  41.2GB                                     lvm
 4      42.9GB  52.6GB  9664MB               primary

(parted) rm 4

(parted) print
Model: VMware Virtual NVMe Disk (nvme)
Disk /dev/nvme0n1: 53.7GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name                  Flags
 1      1049kB  630MB   629MB   fat32        EFI System Partition  boot, esp
 2      630MB   1704MB  1074MB  xfs                                bls_boot
 3      1704MB  42.9GB  41.2GB                                     lvm

(parted)
```

---

## Chapter 5. Strategies for repartitioning a disk (디스크 재분할 전략)

대부분의 RHEL 시스템은 LVM을 사용하여 공간을 관리하지만, Partition table 조작은 여전히 중요한 로우레벨 관리법입니다.

1. **Unpartitioned free space 사용:** 할당되지 않은 공간에 새로 파티션을 만듭니다.
2. **Unused partition 사용:** 안 쓰는 파티션을 지우고 새 Linux 파티션을 만듭니다.
3. **Active partition의 Free space 사용:**
    - **Destructive (파괴적):** 기존 파티션을 밀어버리고 새로 만듭니다. (데이터 백업 필수)
    - **Non-destructive (비파괴적):** 데이터를 유지한 채 파티션 크기를 줄입니다(Shrink). (시간이 오래 걸림)

---

## Chapter 13. Getting started with swap (Swap 시작하기)

Swap space는 물리적 메모리(RAM)가 가득 찼을 때 비활성 프로세스와 데이터를 임시로 저장하여 Out-of-memory 에러를 방지합니다. Swap은 파티션(권장)이 될 수도 있고, Swap file이 될 수도 있습니다.

### 13.2. Recommended system swap space (권장 스왑 크기)

- **RAM 2GB 미만:** RAM의 2배 (절전 모드 시 3배)
- **RAM 2GB ~ 8GB:** RAM과 동일한 크기 (절전 모드 시 2배)
- **RAM 8GB ~ 64GB:** 최소 4GB (절전 모드 시 1.5배)

### LVM 관련 조회

```bash
mr8356@mr8356:~$ sudo vgdisplay
  --- Volume group ---
  VG Name               rl
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               38.41 GiB
  PE Size               4.00 MiB
  Total PE              9833
  Alloc PE / Size       9832 / <38.41 GiB
  Free  PE / Size       1 / 4.00 MiB
  VG UUID               cpaGDy-w6Em-llpr-zRuB-5LIU-m6EJ-XcXIZ1

mr8356@mr8356:~$ sudo lvdisplay
  --- Logical volume ---
  LV Path                /dev/rl/root
  LV Name                root
  VG Name                rl
  LV UUID                U19mAm-2bRc-SZ0Y-ka3n-zdI9-is0A-xiZ5AW
  LV Write Access        read/write
  LV Creation host, time mr8356, 2026-03-10 00:24:38 +0900
  LV Status              available
  # open                 1
  LV Size                <34.48 GiB
  Current LE             8826
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:0

  --- Logical volume ---
  LV Path                /dev/rl/swap
  LV Name                swap
  VG Name                rl
  LV UUID                DnXqan-0oXr-QCDS-TDqR-FWjo-L1Yy-Ryseyz
  LV Write Access        read/write
  LV Creation host, time mr8356, 2026-03-10 00:24:38 +0900
  LV Status              available
  # open                 1
  LV Size                <3.93 GiB
  Current LE             1006
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:1
```

### 🛠️ [실습] LVM2 Logical Volume으로 Swap 생성하기

가장 흔히 쓰이는 LVM 기반 Swap 생성 방법입니다.

`# 1. 4GB 크기의 Swap용 LV 생성
lvcreate -n LogVol02 -L 4G VolGroup00

# 2. Swap 포맷
mkswap /dev/VolGroup00/LogVol02

# 3. fstab에 등록하여 재부팅 시 영구 적용
echo '/dev/VolGroup00/LogVol02 none swap defaults 0 0' >> /etc/fstab
systemctl daemon-reload

# 4. Swap 활성화 및 확인
swapon -v /dev/VolGroup00/LogVol02
cat /proc/swaps
free -h`

```bash
mr8356@mr8356:~$ sudo parted /dev/nvme0n1 mkpart primary 42.9GB 100%

mr8356@mr8356:~$ sudo partprobe /dev/nvme0n1

mr8356@mr8356:~$ sudo pvcreate /dev/nvme0n1p4
  Physical volume "/dev/nvme0n1p4" successfully created.

mr8356@mr8356:~$ sudo vgextend rl /dev/nvme0n1p4
  Volume group "rl" successfully extended

mr8356@mr8356:~$ sudo vgs
  VG #PV #LV #SN Attr   VSize   VFree
  rl   2   2   0 wz--n- <48.41g 10.00g

mr8356@mr8356:~$ sudo lvcreate -n swap_practice -L 4G rl
  Logical volume "swap_practice" created.

mr8356@mr8356:~$ sudo mkswap /dev/rl/swap_practice
Setting up swapspace version 1, size = 4 GiB (4294963200 bytes)

mr8356@mr8356:~$ echo '/dev/rl/swap_practice none swap defaults 0 0' | sudo tee -a /etc/fstab
/dev/rl/swap_practice none swap defaults 0 0

mr8356@mr8356:~$ cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Mon Mar  9 15:24:42 2026
#
# Accessible filesystems, by reference, are maintained under '/dev/disk/'.
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info.
#
# After editing this file, run 'systemctl daemon-reload' to update systemd
# units generated from this file.
#
UUID=25255d13-f12a-423a-b1ab-33bdc270b24a /                       xfs     defaults        0 0
UUID=994ade5f-d7d6-4cb2-8931-dd04e0a19b6a /boot                   xfs     defaults        0 0
UUID=634B-FA82          /boot/efi               vfat    umask=0077,shortname=winnt 0 2
UUID=f792997a-8187-4ce9-8697-6ca9a1da7fc1 none                    swap    defaults        0 0
/dev/rl/swap_practice none swap defaults 0 0

mr8356@mr8356:~$ sudo systemctl daemon-reload

mr8356@mr8356:~$ sudo swapon -v /dev/rl/swap_practice
swapon: /dev/mapper/rl-swap_practice: found signature [pagesize=4096, signature=swap]
swapon: /dev/mapper/rl-swap_practice: pagesize=4096, swapsize=4294967296, devsize=4294967296
swapon /dev/mapper/rl-swap_practice

mr8356@mr8356:~$ free -h
               total        used        free      shared  buff/cache   available
Mem:           3.5Gi       424Mi       3.0Gi        10Mi       278Mi       3.1Gi
Swap:          7.9Gi          0B       7.9Gi
```

### 🛠️ [실습] Swap File로 Swap 생성하기

파티션을 건드리지 않고 파일 형태로 빠르게 Swap을 늘리는 방법입니다.

Bash

`# 1. 64MB 크기의 빈 파일 생성 (블록 크기 1024 * 카운트 65536)
dd if=/dev/zero of=/swapfile bs=1024 count=65536
# (팁: 최신 파일 시스템에서는 'fallocate -l 64M /swapfile'이 더 빠름)

# 2. 권한 설정 (보안상 매우 중요)
chmod 0600 /swapfile

# 3. Swap 포맷 및 fstab 등록
mkswap /swapfile
echo '/swapfile none swap defaults 0 0' >> /etc/fstab
systemctl daemon-reload

# 4. Swap 활성화
swapon /swapfile`

### 🛠️ [실습] VM Tunables (swappiness) 조정

커널이 물리적 메모리 대신 Swap을 얼마나 공격적으로 쓸지 결정합니다.

`# 현재 swappiness 값 확인 (기본값: 60)
cat /proc/sys/vm/swappiness

# 일시적으로 10으로 변경 (Swap 사용을 최대한 미룸)
sysctl vm.swappiness=10

# 영구적으로 적용
echo 'vm.swappiness = 10' >> /etc/sysctl.d/99-sysctl.conf
sysctl -p`

```bash
mr8356@mr8356:~$ cat /proc/sys/vm/swappiness
30
mr8356@mr8356:~$ sudo sysctl vm.swappiness=10
vm.swappiness = 10
mr8356@mr8356:~$ cat /proc/sys/vm/swappiness
10
```

- **Low Swappiness (10 ~ 30):**
    - **대상:** Database 서버 (MySQL, PostgreSQL), 실시간 응답이 중요한 WAS.
    - **이유:** 디스크 I/O는 RAM보다 수천 배 느립니다. DB처럼 인덱스를 메모리에 올리고 쓰는 서비스는 Swap이 발생하는 순간 성능이 급격히 떨어지므로, 최대한 RAM을 끝까지 쓰도록 유도합니다.
- **Default Swappiness (60):**
    - **대상:** 일반적인 웹 서버, 데스크탑 환경.
    - **이유:** 적절한 파일 캐시 공간을 확보하면서도 프로세스 응답성을 유지하는 균형점입니다.
- **High Swappiness (80 ~ 100):**
    - **대상:** 메모리가 매우 부족한 환경이나 배치(Batch) 작업 위주의 서버.
    - **이유:** 적극적으로 Swap을 사용해 RAM 공간을 비워냄으로써 파일 입출력 성능(Cache)을 극대화합니다.

---

## Chapter 20. Removing storage devices (스토리지 장치 제거)

실행 중인 시스템에서 Storage device를 안전하게 제거하는 방법입니다. **메모리 오버로드와 데이터 손실을 막기 위해 반드시 'Top-to-bottom(위에서 아래로)' 접근 방식을 취해야 합니다.** (Application -> File System -> LVM -> Device 순)

### 🛠️ [실습] 안전한 디바이스 제거 파이프라인 (Safe Removal)

디바이스(`/dev/sdc`)를 시스템 에러 없이 완벽하게 뽑아내는 SRE의 정석 절차입니다.

Bash

`# 1. 마운트 해제 (파일 시스템 계층)
umount /mnt/mount-point

# 2. 파일 시스템 메타데이터 삭제 (잔여 Signature 방지)
wipefs -a /dev/vg0/myvol

# 3. LVM 계층 해체 (LV -> VG -> PV 순서로 삭제)
lvremove vg0/myvol
vgremove vg0
pvremove /dev/sdc1
wipefs -a /dev/sdc1   # PV 서명 확실히 제거

# 4. 파티션 테이블 삭제
parted /dev/sdc rm 1
wipefs -a /dev/sdc

# 5. 캐시된 I/O 데이터 디스크로 완벽히 플러시 (Flush)
blockdev --flushbufs /dev/sdc

# 6. SCSI 서브시스템에서 커널 레벨 장치 삭제 (핵심!)
# 이 명령어를 쳐야 디바이스를 물리적으로 뽑아도 커널이 I/O 에러를 뿜지 않음
echo 1 > /sys/block/sdc/device/delete

# 7. 최종 확인 (sdc가 리스트에서 사라졌는지 확인)
lsblk`

```bash
mr8356@mr8356:~$ sudo pvs
[sudo] password for mr8356:
  PV             VG       Fmt  Attr PSize  PFree
  /dev/nvme0n1p3 rl       lvm2 a--  38.41g 4.00m
  /dev/nvme0n1p4 final_vg lvm2 a--   6.56g 4.56g

mr8356@mr8356:~$ sudo umount /mnt/test

mr8356@mr8356:~$ sudo wipefs -a /dev/final_vg/test_lv
/dev/final_vg/test_lv: 4 bytes were erased at offset 0x00000000 (xfs): 58 46 53 42

mr8356@mr8356:~$ sudo lvremove final_vg/test_lv
Do you really want to remove active logical volume final_vg/test_lv? [y/n]:  y
  Logical volume "test_lv" successfully removed.

mr8356@mr8356:~$ sudo vgremove final_vg
  Volume group "final_vg" successfully removed

mr8356@mr8356:~$ sudo pvremove /dev/nvme0n1p4
  Labels on physical volume "/dev/nvme0n1p4" successfully wiped.
mr8356@mr8356:~$ sudo wipefs -a /dev/nvme0n1p4
mr8356@mr8356:~$ sudo parted /dev/nvme0n1 rm 4
Information: You may need to update /etc/fstab.

mr8356@mr8356:~$ sudo vi /etc/fstab -> 아래 주석으로 수정함
# UUID=9f2f87a5-c387-4542-9b23-653a557a1319 /mnt/test xfs defaults 0 0

mr8356@mr8356:~$ sudo systemctl daemon-reload -> systemd에 반영

mr8356@mr8356:~$ sudo partprobe /dev/nvme0n1 -> 커널에 파티션 삭제 알리기

mr8356@mr8356:~$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0          11:0    1  8.1G  0 rom
nvme0n1     259:0    0   50G  0 disk
├─nvme0n1p1 259:1    0  600M  0 part /boot/efi
├─nvme0n1p2 259:2    0    1G  0 part /boot
│                                    /boot
│                                    /boot
└─nvme0n1p3 259:3    0 38.4G  0 part
  ├─rl-root 253:0    0 34.5G  0 lvm  /
  └─rl-swap 253:1    0  3.9G  0 lvm  [SWAP]
```