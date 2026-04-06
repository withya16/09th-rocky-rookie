- 파일시스템 : 파일을 저장하기 위한 운영체제의 논리적 구조
- 디스크 **:** 실제 저장장치 자체
- 파티션 : 디스크를 논리적으로 나눈 구역

```
Disk
→ Partition
→ LVM (PV → VG → LV)
→ File system
→ Mount
→ Directory에서 사용
```

## 디스크

- 실제 저장장치
    - ex) HDD, SSD, 가상디스크
- /dev 아래 표시됨
    - ex) /dev/sda, /dev/sdb, /dev/nvme0n1

## 파티션

- 디스크를 논리적으로 나눈 구역
    - ex) /dev/sda1, /dev/sda2, /dev/sda3
    - /dev/sda가 디스크 전체, sda1이 첫번째 파티션
- 용도별 저장공간, 장애 영향 범위 분리, 백업 및 관리 편리, 파일시스템 종류 다르게 사용하기 등의 사유로 파티션을 분리해서 사용함

### 파티션 테이블

- MBR, GPT이 가장 대표적인 파티션 테이블 방식
- MBR이 오래된 방식이며 최대 디스크 크기, 파티션 개수가 GPT에 비해 적고 안정성이 낮음
    - MBR은 4개의 파티션까지 생성할 수 있고, 5번째부터는 논리 파티션으로 지정됨
- 최근에는 GPT를 대부분 사용

## 파일시스템

- 파티션 위에 생성
- 데이터를 조직화하고 관라하는 방법 정의
    - 파일 이름 관리
    - 디렉터리 구조 관리
    - 파일 위치 관리
    - 빈 공간 관리
    - 권한 관리
    - 메타데이터 관리
- 예시
    - 리눅스: ext4, xfs, btrfs
    - 윈도우: NTFS, FAT32
    - RHEL에서는 xfs가 기본 파일 시스템

### 파일시스템 생성

```bash
mkfs.xfs /dev/sdb1
mkfs.ext4 /dev/sdb2
```

### 점검/복구

```
fsck /dev/sdb1
xfs_repair /dev/vgdata/lvdata
```

## LVM

- PV (Physical Volume) : 디스크나 파티션을 LVM이 쓸 수 있는 형태로 만든 것
    - 디스크 전체, 파티션
- VG (Volume Group) : 여러 PV를 묶어서 만든 저장소 풀
- LV (Logical Volume) : VG에서 잘라 실제로 사용하는 논리 볼륨
- VG에 남는 공간이 있으면 LV 확장 가능
- LV를 확장해도 파일시스템 확장은 별도
    - LV 확장과 파일시스템 확장은 다른 단계

### 실습 흐름

```
pvcreate /dev/sdb1 /dev/sdc1
vgcreate vgdata /dev/sdb1 /dev/sdc1
lvcreate -n lvapp -L 50G vgdata
mkfs.xfs /dev/vgdata/lvapp
mount /dev/vgdata/lvapp /app
```

### 조회 명령어

```
pvs
vgs
lvs
pvdisplay
vgdisplay
lvdisplay
```

### 실습 명령어

```
lsblk
fdisk-l
blkid
parted-l
```

## 마운트

- 해당 저장장치를 디렉터리 트리 구조에 연결하는 과정

```bash
Disk (/dev/sdb)
 └─ Partition (/dev/sdb1)
     └─ File system (xfs)
         └─ Mount (/data)
             └─ files...
```

### 자동마운트

- 마운트는 일시적으로 유지되고 재부팅 시 연결이 끊길 수 있음
- /etc/fstab 파일 : 시스템 부팅 시 자동으로 마운트할 파일시스템 정보를 저장하는 파일
    - 시스템 부팅과정에서 해당 파일을 읽어 자동으로 진행

## 실습

### 디스크 확인

```bash
lsblk
fdisk -l
```

```bash
NAME               MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                  8:0    0   20G  0 disk 
├─sda1               8:1    0    1M  0 part 
├─sda2               8:2    0    1G  0 part /boot
└─sda3               8:3    0   19G  0 part 
  ├─rhel_vbox-root 253:0    0   17G  0 lvm  /
  └─rhel_vbox-swap 253:1    0    2G  0 lvm  [SWAP]
```

### 스토리지 추가

- VM에 스토리지 추가
- 재부팅후 확인하면 추가됨

```bash
# lsblk 출력결과
NAME               MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                  8:0    0   20G  0 disk 
├─sda1               8:1    0    1M  0 part 
├─sda2               8:2    0    1G  0 part /boot
└─sda3               8:3    0   19G  0 part 
  ├─rhel_vbox-root 253:0    0   17G  0 lvm  /
  └─rhel_vbox-swap 253:1    0    2G  0 lvm  [SWAP]
sdb                  8:16   0   20G  0 disk 
sr0                 11:0    1 1024M  0 rom  
```

### 파티션 생성

```bash
fdisk /dev/sdb
```

```bash
# 출력결과
Welcome to fdisk (util-linux 2.40.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS (MBR) disklabel with disk identifier 0x21c43423.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-41943039, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-41943039, default 41943039): 

Created a new partition 1 of type 'Linux' and of size 20 GiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

- p : 현재 파티션 상태 출력
- d : 파티션 삭제
- n : 파티션 생성
- t : 파티션 변경
- w : 저장 후 종료
- q : 취소 후 종료

```bash
# lsblk 출력결과
NAME               MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                  8:0    0   20G  0 disk 
├─sda1               8:1    0    1M  0 part 
├─sda2               8:2    0    1G  0 part /boot
└─sda3               8:3    0   19G  0 part 
  ├─rhel_vbox-root 253:0    0   17G  0 lvm  /
  └─rhel_vbox-swap 253:1    0    2G  0 lvm  [SWAP]
sdb                  8:16   0   20G  0 disk 
└─sdb1               8:17   0   20G  0 part 
sr0                 11:0    1 1024M  0 rom 
```

### PV 생성

```bash
pvcreate /dev/sdb1
```

```bash
# pvs 출력결과
  PV         VG        Fmt  Attr PSize   PFree  
  /dev/sda3  rhel_vbox lvm2 a--  <19.00g      0 
  /dev/sdb1            lvm2 ---  <20.00g <20.00g
```

### VG 생성

```csharp
vgcreate vgdata /dev/sdb1
```

```bash
# vgs 출력결과
  VG        #PV #LV #SN Attr   VSize   VFree  
  rhel_vbox   1   2   0 wz--n- <19.00g      0 
  vgdata      1   0   0 wz--n- <20.00g <20.00g
```

### LV 생성

```bash
lvcreate -n lvdata -L 10G vgdata
```

```bash
# lvs 출력결과
  LV     VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root   rhel_vbox -wi-ao---- <17.00g                                                    
  swap   rhel_vbox -wi-ao----   2.00g                                                    
  lvdata vgdata    -wi-a-----  10.00g  
```

### 파일시스템 생성

```bash
mkfs.xfs /dev/vgdata/lvdata
```

```bash
# lsblk -f 출력결과
NAME               FSTYPE      FSVER    LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
sda                                                                                                 
├─sda1                                                                                              
├─sda2             xfs                        f92b8510-eea5-4db1-94f6-362742094c2c    607.5M    37% /boot
└─sda3             LVM2_member LVM2 001       bDWAHs-TKdh-Vnpt-mIxa-S3Oz-CDTV-4JXxyQ                
  ├─rhel_vbox-root xfs                        8b720a2d-78c1-4081-8909-915427e04a15     11.9G    30% /
  └─rhel_vbox-swap swap        1              a116ebf7-1520-4853-9bbb-ce84ace48b0a                  [SWAP]
sdb                                                                                                 
└─sdb1             LVM2_member LVM2 001       RCqjJn-Otaf-2Ir8-VWvV-SB8q-GmGP-aJ3Lxs                
  └─vgdata-lvdata  xfs                        5f080643-3afb-44e6-8fb1-e2355f02d6d9                  
sr0  
```

### 마운트

```bash
mkdir /data
mount /dev/vgdata/lvdata /data
```

```bash
# df -h 출력결과
Filesystem                  Size  Used Avail Use% Mounted on
/dev/mapper/rhel_vbox-root   17G  5.1G   12G  30% /
devtmpfs                    1.8G     0  1.8G   0% /dev
tmpfs                       1.8G     0  1.8G   0% /dev/shm
tmpfs                       731M  9.1M  722M   2% /run
tmpfs                       1.0M     0  1.0M   0% /run/credentials/systemd-journald.service
/dev/sda2                   960M  353M  608M  37% /boot
tmpfs                       366M   72K  366M   1% /run/user/42
tmpfs                       366M   56K  366M   1% /run/user/1000
/dev/mapper/vgdata-lvdata    10G  228M  9.8G   3% **/data**
```

```bash
# lsblk
NAME               MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                  8:0    0   20G  0 disk 
├─sda1               8:1    0    1M  0 part 
├─sda2               8:2    0    1G  0 part /boot
└─sda3               8:3    0   19G  0 part 
  ├─rhel_vbox-root 253:0    0   17G  0 lvm  /
  └─rhel_vbox-swap 253:1    0    2G  0 lvm  [SWAP]
sdb                  8:16   0   20G  0 disk 
└─sdb1               8:17   0   20G  0 part 
  └─vgdata-lvdata  253:2    0   10G  0 lvm  **/data**
sr0                 11:0    1 1024M  0 rom  
```

```bash
# mount | grep "data" 출력결과
/dev/mapper/vgdata-lvdata on /data type xfs (rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota)
```

### fstab 테스트

- blkid / lsblk -f 로 UUID 확인

```bash
vi /etc/fstab
# 에디터 내부에 아래 한줄 추가
UUID=5f080643-3afb-44e6-8fb1-e2355f02d6d9 /data                   xfs     defaults        0 0
```

- 재부팅해도 마운트 연결되어 있는 것 확인