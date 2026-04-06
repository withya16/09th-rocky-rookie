# 파일 시스템 및 스토리지 관리

## 디스크와 파티션

### 디스크 (Disk)

`디스크`는 HDD, SSD, NVMe 장치 같은 물리적 저장장치입니다.

### 파티션 (Partition)

[파티션](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/10/html/managing_storage_devices/disk-partitions#overview-of-partitions)

`파티션`은 하나의 디스크를 논리적으로 나눈 구역이고, 디스크를 용도별로 나누어 효율적이고 안전하게 관리하기 위해서 사용합니다. 각 파티션은 파일 시스템을 생성하는 데 사용하거나 LVM 구성에 사용할 수 있습니다.

### 파티션 테이블 (Partition Table)

[파티션 테이블](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/10/html/managing_storage_devices/comparison-of-partition-table-types)

파티션 테이블은 디스크를 여러 파티션으로 나눌 때 각 파티션의 위치와 크기 정보를 기록한 구조입니다.

1. **MBR (Master Boot Record)**

- `MBR`은 예전 파티션 테이블 방식으로, 디스크의 가장 앞부분 첫 섹터에 부트로더 정보와 파티션 정보를 함께 저장합니다.
- 최대 4개의 기본 파티션 또는 3개의 기본 파티션과 1개의 확장 파티션만 만들 수 있습니다.
- 구조가 단순하고 오래된 시스템과의 호환성이 좋지만, 2TiB를 초과하는 큰 디스크를 활용하는 데 한계가 있습니다.
- 파티션 정보가 디스크 앞부분에 집중되어 있어, 해당 영역이 손상되면 복구가 어렵다는 단점이 있습니다.

2. **GPT (GUID Partition Table)**

- `GPT`는 최신 파티션 테이블 방식으로, MBR보다 더 많은 파티션을 생성할 수 있고 대용량의 디스크도 안정적으로 지원합니다.
- 각 디스크와 파티션에 고유한 GUID를 부여하므로, 장치명 변경과 관계없이 대상을 안정적으로 식별할 수 있고, 파티션의 용도를 더 명확하게 관리할 수 있습니다.
- 디스크 앞부분뿐 아니라 끝부분에도 파티션 테이블 정보를 저장해서, 일부 손상 시 복구에 유리합니다.

### 주요 명령어

[parted로 파티션 생성](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/10/html/managing_storage_devices/creating-a-partition-with-parted)  
[parted로 파티션 제거](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/10/html/managing_storage_devices/removing-a-partition-with-parted)

```bash
# 디스크 및 파티션 정보 출력
lsblk

# MBR 파티션 테이블 생성
parted /dev/sda mklabel msdos

# GPT 파티션 테이블 생성
parted /dev/sda mklabel gpt

# 디스크의 파티션 테이블 정보 출력
parted /dev/sda print

# free space도 함께 출력
parted /dev/sda print free

# GPT 파티션 생성
parted /dev/sda mkpart <part-name> <fs-type> <start> <end>
parted /dev/sda mkpart data xfs 1MiB 20GiB

# 파티션 삭제
parted /dev/sda rm <minor-number>
parted /dev/sda rm 1
```

> #### + /dev/sda의 의미
>
> sd: 리눅스에서 사용하는 디스크 장치명 접두어  
> a: 첫 번째 디스크
>
> 즉,  
> /dev/sda: 첫 번째 디스크  
> /dev/sdb: 두 번째 디스크  
> /dev/sdc...  
> /dev/sda2: 첫 번째 디스크의 두 번째 파티션  
> /dev/sdb1: 두 번째 디스크의 첫 번째 파티션

## LVM (Logical Volume Manager)

[LVM](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/10/html/configuring_and_managing_logical_volumes/index)

`LVM`은 하나 이상의 물리 디스크 또는 파티션을 하나의 유연한 저장 공간처럼 관리할 수 있게 해주는 볼륨 관리 방식입니다.

### 구성

1. Physical Volume (PV): LVM 사용을 위해 지정된 물리적인 디스크 또는 파티션
2. Volume Group (VG): 하나 이상의 PV를 묶어서 만든 논리적인 저장 공간
3. Logical Volume (LV): VG 안에서 필요한 크기만큼 할당하여 만든 논리적인 볼륨

![LVM](https://github.com/user-attachments/assets/aa960d0b-de91-4d5c-80ee-a7e30034cb40)

### 주요 명령어

```bash
# PV 생성
pvcreate <disk-name or part-name>
pvcreate /dev/sda1

# PV 목록 출력
pvs

# PV 상세 정보 출력
pvdisplay

# PV 데이터 마이그레이션
pvmove <source-pv> <destination-pv>
pvmove /dev/sda1 /dev/sda2

# PV 삭제
pvremove <pv-name>
pvremove /dev/sda1

---

# VG 생성
vgcreate <vg-name> <pv-name>
vgcreate vgdata /dev/sda1

# VG 확장
vgextend <vg-name> <pv-name>
vgextend vgdata /dev/sda2

# VG 결합
vgmerge <destination-vg-name> <source-vg-name>
vgmerge vgdata vgbackup

# VG에서 PV 제거
vgreduce <vg-name> <pv-name>
vgreduce vgdata /dev/sda1

# 기존 VG에서 PV를 분리해 새 VG 생성
vgsplit <source-vg-name> <new-vg-name> <pv-name>
vgsplit vgdata vgtmp /dev/sda2

# VG 삭제
vgremove <vg-name>
vgremove vgtmp

---

# LV 생성
lvcreate <option> <vg-name>
lvcreate -L 10G -n lvdata vgdata

# LV 목록 출력
lvs

# LV 상세 정보 출력
lvdisplay

# LV 크기 확장
lvextend <option> <lv-path>
lvextend -L +5G /dev/vgdata/lvdata

# LV 크기 축소
lvreduce <option> <lv-path>
lvreduce -L -2G /dev/vgdata/lvdata

# LV 이름 변경
lvrename <vg-name> <old-lv-name> <new-lv-name>
lvrename vgdata lvdata lvbackup

# LV 비활성화
lvchange -an <lv-path>
lvchange -an /dev/vgdata/lvdata

# LV 활성화
lvchange -ay <lv-path>
lvchange -ay /dev/vgdata/lvdata

# LV 삭제
lvremove <lv-path>
lvremove /dev/vgdata/lvdata
```

## 파일 시스템 (File System)

`파일 시스템`은 디스크 공간에 데이터를 저장하고, 파일과 디렉터리를 관리하는 구조입니다.

### 종류

- XFS (eXtended File System): RHEL의 기본 파일 시스템으로, 대용량 파일 처리에 강한 64비트 저널링 파일 시스템입니다.
- ext4 (extended file system 4): 리눅스의 저널링 파일 시스템 중 하나로, 대부분의 리눅스 배포판에서 기본 파일 시스템으로 사용합니다.
- vfat, iso9660, nfs 등

> Windows는 NTFS(New Technology File System), macOS은 APFS(Apple File System)을 기본 파일 시스템으로 사용합니다.

### 구성 요소

1. 파일: 사용자 데이터와 속성을 가지는 논리적 객체, 기본 단위
2. 디렉터리: 파일 이름과 inode 번호의 매핑 정보를 저장하는 특수한 파일, 계층 구조
3. inode: 파일의 메타데이터와 데이터 블록 위치 정보를 저장하는 구조체, 각 파일은 고유한 inode 번호로 관리
4. 데이터 블록: 파일이나 디렉터리의 실제 내용이 저장되는 공간
5. 슈퍼 블록: 파일 시스템 전체의 상태와 관리 정보를 저장하는 공간
6. 저널: 파일 시스템의 변경 내역을 기록하는 공간, 저널링 파일 시스템에서 사용

> ## RAID
>
> 일반적으로 실무에서는 RAID 1, RAID 5, RAID 6, RAID 10을 많이 사용합니다.
>
> - RAID 1: 미러링, 2개의 디스크에 동일한 데이터를 실시간으로 복제하여 저장
> - RAID 5: 패리티, 데이터와 패리티를 모든 디스크에 분산 저장, 디스크 1개 고장까지 허용
> - RAID 6: 이중 패리티, 동시에 디스크 2개 고장까지 허용
> - RAID 10: 1(미러링) + 0(스트라이핑), 최소 4개의 디스크가 필요하며 데이터를 복제하여 안정성을 확보하는 동시에 분산 저장하여 읽기/쓰기 속도 향상
