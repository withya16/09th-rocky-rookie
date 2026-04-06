# Managing Logical Volumes (LVM)

https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_and_managing_logical_volumes/index

# 선수지식

| **핵심 키워드** | **설명** |
| --- | --- |
| **RAID** | 여러 디스크를 묶어 성능을 높이거나 데이터를 복제하는 기술입니다. |
| **Boot Loader (GRUB)** | 컴퓨터를 켰을 때 운영체제를 선택하고 부팅을 시작하는 프로그램입니다. |
| **Copy-on-Write (CoW)** | 데이터를 수정할 때 원본은 두고 바뀐 부분만 새 공간에 기록하는 방식입니다. |
| **Thin Provisioning** | 실제 용량보다 더 큰 가상 용량을 할당하는 '가상 할당' 기술입니다. |
| **Ansible** | 수백 대의 서버 설정을 코드로 자동화하는 도구입니다. |

---

## Configuring and Managing LVM

**Red Hat Customer Content Services**

### Abstract (초록)

**Logical Volume Manager (LVM)**은 물리적 스토리지 장치의 관리 효율성과 유연성을 높이기 위해 설계된 **Storage Virtualization** 소프트웨어입니다. 물리적 하드웨어를 추상화함으로써, LVM은 가상 스토리지 장치를 동적으로 생성(Create), 크기 조정(Resize), 제거(Remove)할 수 있게 해줍니다.

이 프레임워크 내에서 **Physical Volumes (PVs)**는 함께 그룹화되어 **Volume Group (VG)**을 형성하는 로우 스토리지 장치를 나타냅니다. 이 VG 내에서 LVM은 공간을 할당하여 **Logical Volume (LV)**을 생성합니다. LV는 **File System**, **Database**, 또는 **Application**이 사용할 수 있는 가상 **Block Storage** 장치입니다.

---

## Chapter 1. Overview of Logical Volume Management

- **Logical Volume Manager (LVM)**은 물리적 스토리지 위에 추상화 레이어를 생성하여 **Logical Storage Volumes**를 만들 수 있게 도와줍니다. 이는 직접적인 물리적 스토리지 사용에 비해 더 많은 유연성을 제공합니다.

또한, 하드웨어 스토리지 구성이 소프트웨어로부터 숨겨지므로, Application을 중단하거나 File System을 **Unmount**하지 않고도 크기를 조정하거나 이동할 수 있습니다. 이는 운영 비용을 절감할 수 있습니다.

### 1.1. LVM Architecture

![](https://access.redhat.com/webassets/avalon/d/Red_Hat_Enterprise_Linux-10-Configuring_and_managing_logical_volumes-en-US/images/31bd96635c4120abe3e771a423f61cd6/basic-lvm-volume-components.png)

LVM은 물리적 스토리지 위에 추상화 레이어를 생성하며, 이는 다음과 같은 컴포넌트들로 구성됩니다.

- **Physical Volume (PV)**: LVM 용도로 지정된 파티션 또는 전체 디스크입니다.
- **Volume Group (VG)**: Physical Volumes (PVs)의 집합으로, Logical Volumes를 할당할 수 있는 디스크 공간의 **Pool**을 생성합니다.
- **Logical Volume (LV)**: 실제 사용 가능한 스토리지 장치를 나타냅니다.

### 1.2. Advantages of LVM

LVM은 동적인 스토리지 관리를 가능하게 하는 유연한 추상화 레이어를 제공합니다. Application의 중단 시간(**Downtime**) 없이 크기 조정, 이동, 관리가 가능하며 **Snapshots**이나 **Striping** 같은 고급 기능을 제공합니다.

**Logical Volumes**가 물리적 스토리지를 직접 사용하는 것에 비해 갖는 장점은 다음과 같습니다:

- **Flexible Capacity (유연한 용량)**: 여러 장치와 파티션을 하나의 Logical Volume으로 합칠 수 있습니다. 이를 통해 File System이 여러 장치에 걸쳐 마치 하나의 큰 장치인 것처럼 확장될 수 있습니다.
- **Convenient Device Naming (편리한 장치 명명)**: 사용자가 정의한 커스텀 이름으로 관리할 수 있습니다.
- **Resizeable Storage Volumes (크기 조정 가능한 볼륨)**: 하드웨어를 다시 포맷하거나 파티션을 나눌 필요 없이 소프트웨어 명령어로 확장하거나 축소할 수 있습니다.
- **Online Data Relocation (온라인 데이터 이동)**: `pvmove` 명령어를 사용하여 시스템이 활성 상태인 동안 데이터를 더 빠르고 탄탄한 하드웨어로 옮길 수 있습니다.
- **Striped Volumes**: 데이터를 두 개 이상의 장치에 분산 저장하여 **Throughput**을 높일 수 있습니다.
- **RAID Volumes**: 데이터 보호 및 성능 향상을 위해 RAID를 편리하게 구성할 수 있습니다.
- **Volume Snapshots**: 특정 시점의 복사본을 생성하여 백업이나 테스트용으로 사용할 수 있습니다.
- **Thin Volumes**: 실제 물리적 공간보다 더 큰 가상 볼륨을 생성하는 **Thin-provisioning**이 가능합니다.
- **Caching**: SSD 같은 빠른 장치를 사용하여 HDD 같은 느린 장치의 성능을 높이는 **Caching**을 지원합니다.

---

## Chapter 2. Managing LVM Physical Volumes

- **Physical Volume (PV)**는 LVM이 사용하는 물리적 스토리지 장치 또는 파티션입니다. 초기화 과정에서 **LVM Disk Label**과 **Metadata**가 장치에 기록되며, 이를 통해 LVM이 장치를 추적하고 관리합니다.

> **Note**: 초기화 후에는 Metadata의 크기를 늘릴 수 없습니다. 더 큰 Metadata가 필요하다면 초기화 시 적절한 크기를 설정해야 합니다.
> 

초기화가 완료되면 PV를 **Volume Group (VG)**에 할당할 수 있고, 이를 다시 **Logical Volumes (LVs)**로 나눌 수 있습니다. 성능 최적화를 위해 전체 디스크를 하나의 PV로 파티셔닝하는 것이 권장됩니다.

### 2.1. Creating an LVM Physical Volume

`pvcreate` 명령어를 사용하여 장치를 LVM 용도로 초기화합니다.

**Procedure (실습)**:

1. 사용할 장치 식별:
    
    `lsblk`
    
2. LVM Physical Volume 생성 
    
    `sudo pvcreate /dev/nvme0n1p4`
    

**Verification (확인)**:

`sudo pvs`

```bash
mr8356@mr8356:~$ sudo pvcreate /dev/nvme0n1p4
  Physical volume "/dev/nvme0n1p4" successfully created.

mr8356@mr8356:~$ sudo pvs
  PV             VG Fmt  Attr PSize  PFree
  /dev/nvme0n1p3 rl lvm2 a--  38.41g  4.00m
  /dev/nvme0n1p4    lvm2 ---  <6.57g <6.57g
```

### 2.2. Resizing Physical Volumes (Ansible RHEL System Role)

RHEL의 **Storage System Role**을 사용하면 하드웨어 크기가 변경된 후(예: 가상 디스크 확장) 자동으로 PV 크기를 조정할 수 있습니다. `grow_to_fill: true` 설정을 통해 새 용량을 자동으로 사용하도록 확장합니다.

### 2.3. Removing LVM Physical Volumes

`pvremove` 명령어로 PV를 제거합니다. 만약 PV가 VG에 속해 있다면 먼저 VG에서 제거해야 합니다.

**Procedure (실습)**:

1. 제거할 PV 확인:Bash
    
    `sudo pvs`
    
2. PV 제거:Bash
    
    `sudo pvremove /dev/nvme0n1p4`
    

*VG에 속해 있다면:*

- 여러 PV가 있는 경우: `sudo vgreduce VolumeGroupName /dev/nvme0n1p4`
- 하나의 PV만 있는 경우: `sudo vgremove VolumeGroupName`

### 2.4 ~ 2.6. Web Console (Cockpit) 사용법

RHEL 웹 콘솔(Cockpit)을 통해 GUI 환경에서 LV 생성, 포맷, 크기 조정을 수행할 수 있습니다.

- **Formatting**: XFS는 확장은 지원하지만 축소(**Shrinking**)는 지원하지 않음을 주의하십시오. ext4는 둘 다 지원합니다.
- **Encryption**: **LUKS1** 또는 **LUKS2**를 선택하여 볼륨을 암호화할 수 있습니다.

---

## Chapter 3. Managing LVM Volume Groups

- *Volume Groups (VGs)**를 생성하여 여러 PV를 하나의 스토리지 엔티티로 결합하고 관리할 수 있습니다.

**Extents**는 LVM에서 할당할 수 있는 최소 공간 단위입니다. **Physical Extent (PE)**와 **Logical Extent (LE)**의 기본 크기는 **4 MiB**입니다. LV를 생성할 때 LVM은 가용한 PE를 찾아 연결하여 요청된 크기의 볼륨을 만듭니다.

### 3.1. Creating an LVM Volume Group

`vgcreate` 명령어를 사용합니다. `-s` 옵션으로 **Extent Size**를 조정할 수 있습니다.

**Procedure (실습)**:

1. 포함할 PV 확인:
    
    `sudo pvs`
    
2. VG 생성:
    
    `sudo vgcreate practice_vg /dev/nvme0n1p4`
    

**Verification (확인)**:

`sudo vgs`

### 3.3. Renaming an LVM Volume Group

`vgrename` 명령어로 이름을 변경합니다.

Bash

`sudo vgrename practice_vg final_vg`

### 3.4. Extending an LVM Volume Group

`vgextend` 명령어로 기존 VG에 새로운 PV를 추가하여 용량을 늘립니다.

Bash

`sudo vgextend final_vg /dev/nvme0n1p5  # 새로운 파티션이 있다면`

### 3.5. Combining LVM Volume Groups

`vgmerge` 명령어로 두 개의 VG를 하나로 합칩니다. **Source Volume**이 **Destination Volume**으로 병합됩니다.

Bash

`sudo vgmerge dest_vg source_vg`

### 3.6. Removing Physical Volumes from a Volume Group

사용하지 않는 PV를 제거하려면 `vgreduce`를 사용합니다. 데이터가 남아있다면 `pvmove`로 먼저 데이터를 옮겨야 합니다.

**Procedure (실습)**:

1. 데이터 이동:
    
    `sudo pvmove /dev/nvme0n1p4`
    
2. VG에서 제거:
    
    `sudo vgreduce final_vg /dev/nvme0n1p4`
    

### 3.7. Splitting an LVM Volume Group

`vgsplit`을 사용하여 하나의 VG를 두 개로 나눌 수 있습니다.

`sudo vgsplit old_vg new_vg /dev/nvme0n1p4`

### 3.8. Moving a Volume Group to Another System

`vgexport`와 `vgimport`를 사용하여 VG를 다른 시스템으로 옮길 수 있습니다.

1. `umount` 및 `vgchange -an` (비활성화)
2. `vgexport VolumeGroupName`
3. 디스크를 새 시스템에 연결 후 `vgimport VolumeGroupName`
4. `vgchange -ay` (활성화) 후 마운트.

```bash
mr8356@mr8356:~$ sudo vgcreate practice_vg /dev/nvme0n1p4
  Physical volume "/dev/nvme0n1p4" successfully created.
  Volume group "practice_vg" successfully created

mr8356@mr8356:~$ sudo vgs
  VG          #PV #LV #SN Attr   VSize  VFree
  practice_vg   1   0   0 wz--n-  6.56g 6.56g
  rl            1   2   0 wz--n- 38.41g 4.00m

mr8356@mr8356:~$ sudo vgrename practice_vg final_vg
  Volume group "practice_vg" successfully renamed to "final_vg"

mr8356@mr8356:~$ sudo vgs
  VG       #PV #LV #SN Attr   VSize  VFree
  final_vg   1   0   0 wz--n-  6.56g 6.56g
  rl         1   2   0 wz--n- 38.41g 4.00m
```

---

## Chapter 4. Basic Logical Volume Management

LVM은 물리적 스토리지 위에 추상화 레이어를 만들어 유연성을 제공합니다.

### 4.1. Overview of Logical Volume Features

- **Concatenation**: 여러 PV의 공간을 하나의 LV로 결합합니다.
- **Striping**: 데이터를 여러 PV에 분산하여 I/O 효율을 높입니다.
- **RAID**: RAID 0, 1, 4, 5, 6, 10을 지원합니다.
- **Thin Provisioning**: 실제 물리 공간보다 큰 가상 볼륨을 생성하고 필요할 때만 공간을 할당합니다.
- **Snapshots**: 특정 시점의 복사본을 생성합니다 (**CoW** 방식).
- **Caching**: 빠른 장치(SSD)를 느린 장치(HDD)의 캐시로 사용합니다.

### 4.2. Creating Logical Volumes

### 4.2.1. Linear (Thick) LV 생성

`# practice_vg에서 2GB짜리 test_lv 생성
sudo lvcreate --name test_lv --size 2G final_vg`

```bash
mr8356@mr8356:~$ sudo lvcreate --name test_lv --size 2G final_vg
WARNING: swap signature detected on /dev/final_vg/test_lv at offset 4086. Wipe it? [y/n]:  Y
  Wiping swap signature on /dev/final_vg/test_lv.
  Logical volume "test_lv" created.

mr8356@mr8356:~$ sudo lvs
  LV      VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  test_lv final_vg -wi-a-----   2.00g
  root    rl       -wi-ao---- <34.48g
  swap    rl       -wi-ao----  <3.93g
```

### 4.2.3. Striped LV 생성

`# 2개의 스트라이프를 사용하여 생성
sudo lvcreate --stripes 2 --name striped_lv --size 2G final_vg`

### 4.2.5. Thin Logical Volume 생성

Thin Pool을 먼저 만들고 그 안에서 Thin LV를 만듭니다.

`# 1. Thin Pool 생성
sudo lvcreate --type thin-pool --size 5G --name practice_pool final_vg
# 2. Thin LV 생성 (가상 크기는 10G로 설정 가능)
sudo lvcreate --type thin --virtualsize 10G --name thin_lv --thinpool practice_pool final_vg`

### 4.3. Resizing Logical Volumes

### 4.3.1. Extending a Linear LV (확장)

- `-resizefs` 옵션을 쓰면 File System 크기까지 한 번에 조정됩니다.

`sudo lvextend --size +1G --resizefs final_vg/test_lv`

### 4.3.5. Shrinking Logical Volumes (축소)

**주의**: 데이터 손실 위험이 있으므로 반드시 백업하고 File System을 먼저 축소해야 합니다 (XFS는 불가).

`sudo lvreduce --size 1G --resizefs final_vg/test_lv`

---

## Chapter 5. Advanced Logical Volume Management

### 5.1. Managing Logical Volume Snapshots

**Snapshot**은 특정 시점의 원본 LV 콘텐츠를 미러링합니다.

- **Thick LV Snapshots**: 원본 데이터 변경 시 변경 전 데이터를 Snapshot 공간으로 복사합니다 (**Copy-on-Write**). Snapshot이 100% 차면 무효화됩니다.
- **Thin LV Snapshots**: 원본과 Snapshot이 동일한 데이터 블록을 공유하다가 변경 시에만 새 블록을 씁니다. 효율적입니다.

**실습 (Snapshot 생성)**:

`sudo lvcreate --snapshot --size 500M --name test_lv_snap final_vg/test_lv`

```bash
mr8356@mr8356:~$ sudo lvcreate --snapshot --size 500M --name test_lv_snap final_vg/test_lv
  Logical volume "test_lv_snap" created.

mr8356@mr8356:~$ sudo lvs
  LV           VG       Attr       LSize   Pool Origin  Data%  Meta%  Move Log Cpy%Sync Convert
  test_lv      final_vg owi-a-s---   2.00g
  test_lv_snap final_vg swi-a-s--- 500.00m      test_lv 0.00
  root         rl       -wi-ao---- <34.48g
  swap         rl       -wi-ao----  <3.93g
```

**실습 (Snapshot Merge - 복구)**:

Snapshot 시점으로 데이터를 되돌립니다.

`sudo lvconvert --merge final_vg/snapshot_of_test_lv`

```bash
mr8356@mr8356:~$ sudo lvconvert --merge final_vg/test_lv_snap
  Merging of volume final_vg/test_lv_snap started.
  final_vg/test_lv: Merged: 100.00%

mr8356@mr8356:~$ sudo lvs
  LV      VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  test_lv final_vg -wi-a-----   2.00g
  root    rl       -wi-ao---- <34.48g
  swap    rl       -wi-ao----  <3.93g
mr8356@mr8356:~$
```

---