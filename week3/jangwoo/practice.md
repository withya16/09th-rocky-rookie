# 실습

위에서 정리한 내용을, 하나의 흐름을 가지고 실습을 진행하였다.

이미, 실습 환경에서는 LVM 을 통해 파티셔닝이 완료 되었고, FS도 없는 상황이라, 가상파일을 마치 디스크 처럼 사용 할 수 있도록
**”루프백(Loopback) 장치 활용 ”** 방법을 사용해서 실습을 진행했다.

> 터미널 로그는 `bash` 블록으로 구간 나눔. 

### 1. 루프백 이미지(`dd`) + `losetup` 연결

*가상 디스크 파일 2개 생성 후 루프 장치에 매핑*

```bash
jangwoojung@localhost:~$ dd if=/dev/zero of=~/disk1.img bs=1M count=2048
2048+0 records in
2048+0 records out
2147483648 bytes (2.1 GB, 2.0 GiB) copied, 0.492787 s, 4.4 GB/s
jangwoojung@localhost:~$ dd if=/dev/zero of=~/disk2.img bs=1M count=2048
2048+0 records in
2048+0 records out
2147483648 bytes (2.1 GB, 2.0 GiB) copied, 0.704182 s, 3.0 GB/s
jangwoojung@localhost:~$ sudo losetup /dev/loop10 ~/disk1.img
[sudo] password for jangwoojung: 
jangwoojung@localhost:~$ sudo losetup /dev/loop11 ~/disk2.img
jangwoojung@localhost:~$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop10        7:10   0     2G  0 loop 
loop11        7:11   0     2G  0 loop 
nvme0n1     259:0    0 238.5G  0 disk 
├─nvme0n1p1 259:1    0   600M  0 part /boot/efi
├─nvme0n1p2 259:2    0     1G  0 part /boot
└─nvme0n1p3 259:3    0 236.9G  0 part 
  ├─rl-root 253:0    0    70G  0 lvm  /
  ├─rl-swap 253:1    0   7.7G  0 lvm  [SWAP]
  └─rl-home 253:2    0 159.2G  0 lvm  /home
  
```

### 2. `loop11` — GPT 파티션(`parted`, `partprobe`)

*테스트용으로 ext4 파티션 하나 생성*

```bash
jangwoojung@localhost:~$ sudo parted /dev/loop11 mklabel gpt
[sudo] password for jangwoojung: 
Information: You may need to update /etc/fstab.

jangwoojung@localhost:~$ sudo parted /dev/loop11 mkpart primary ext4 1MiB 1GiB
Information: You may need to update /etc/fstab.

jangwoojung@localhost:~$ sudo partprobe /dev/loop11
jangwoojung@localhost:~$ lsblk -f
NAME        FSTYPE      FSVER    LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
loop10                                                                                       
└─loop10p1                                                                                   
loop11                                                                                       
└─loop11p1                                                                                   
nvme0n1                                                                                      
├─nvme0n1p1 vfat        FAT32          0F3C-37EC                               590.5M     1% /boot/efi
├─nvme0n1p2 xfs                        beb9e439-df75-4383-ab51-727ca5b6259a    478.6M    50% /boot
└─nvme0n1p3 LVM2_member LVM2 001       neMfLp-L1cJ-g8Tl-ui2R-Lpmg-Y7nb-79g2g1                
  ├─rl-root xfs                        bfd30e55-8635-44b1-8991-daede9520d1e     64.4G     8% /
  ├─rl-swap swap        1              5c355a67-75af-488f-803a-4990da8b9991                  [SWAP]
  └─rl-home xfs                        e74d5431-b85c-45fe-b9ed-10a74ce5a057    151.7G     5% /home
```

### 3. `loop10` 확인 · PV · VG

*`fdisk`로 MBR/LVM 파티션 확인 후 `pvcreate`, `vgcreate`*

```bash
jangwoojung@localhost:~$ sudo fdisk -l /dev/loop10
Disk /dev/loop10: 2 GiB, 2147483648 bytes, 4194304 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xffad8337

Device        Boot Start     End Sectors Size Id Type
/dev/loop10p1       2048 2099199 2097152   1G 8e Linux LVM

jangwoojung@localhost:~$ sudo pvcreate /dev/loop10
[sudo] password for jangwoojung: 
WARNING: dos signature detected on /dev/loop10 at offset 510. Wipe it? [y/n]: y
  Wiping dos signature on /dev/loop10.
  Physical volume "/dev/loop10" successfully created.

jangwoojung@localhost:~$ sudo pvs
  PV             VG Fmt  Attr PSize    PFree
  /dev/loop10       lvm2 ---     2.00g 2.00g
  /dev/nvme0n1p3 rl lvm2 a--  <236.89g    0 

jangwoojung@localhost:~$ sudo vgcreate vg_test /dev/loop10p1
  Physical volume "/dev/loop10p1" successfully created.
  Volume group "vg_test" successfully created
jangwoojung@localhost:~$ sudo vgs
  VG      #PV #LV #SN Attr   VSize    VFree   
  rl        1   3   0 wz--n- <236.89g       0 
  vg_test   1   0   0 wz--n- 1020.00m 1020.00m
  
```

### 4. 논리 볼륨(`lvcreate`) · ext4 포맷 · `tune2fs` · `e2fsck`

*LV 생성 → `mkfs.ext4` → `tune2fs -l` → `e2fsck`*

```bash
jangwoojung@localhost:~$ sudo lvcreate -L 500M -n lv_1 vg_test
  Logical volume "lv_1" created.
jangwoojung@localhost:~$ sudo lvcreate -L 500M -n lv_2 vg_test
  Logical volume "lv_2" created.
jangwoojung@localhost:~$ lvs
  WARNING: Running as a non-root user. Functionality may be unavailable.
  /run/lock/lvm/P_global:aux: open failed: Permission denied
jangwoojung@localhost:~$ sudo lvs
  LV   VG      Attr       LSize    Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home rl      -wi-ao---- <159.17g                                                    
  root rl      -wi-ao----   70.00g                                                    
  swap rl      -wi-ao----   <7.72g                                                    
  lv_1 vg_test -wi-a-----  500.00m                                                    
  lv_2 vg_test -wi-a-----  500.00m                                                    
jangwoojung@localhost:~$ lsblk -f
NAME             FSTYPE      FSVER    LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
loop10           LVM2_member LVM2 001       a5THyp-IBbJ-8xj0-DYzl-v82s-LDkC-dOxJsw                
└─loop10p1       LVM2_member LVM2 001       WsECSj-qJ88-b5Zl-0s1H-2y3v-GkST-fuVXb2                
  ├─vg_test-lv_1                                                                                  
  └─vg_test-lv_2                                                                                  
loop11                                                                                            
└─loop11p1                                                                                        
nvme0n1                                                                                           
├─nvme0n1p1      vfat        FAT32          0F3C-37EC                               590.5M     1% /boot/efi
├─nvme0n1p2      xfs                        beb9e439-df75-4383-ab51-727ca5b6259a    478.6M    50% /boot
└─nvme0n1p3      LVM2_member LVM2 001       neMfLp-L1cJ-g8Tl-ui2R-Lpmg-Y7nb-79g2g1                
  ├─rl-root      xfs                        bfd30e55-8635-44b1-8991-daede9520d1e     64.4G     8% /
  ├─rl-swap      swap        1              5c355a67-75af-488f-803a-4990da8b9991                  [SWAP]
  └─rl-home      xfs                        e74d5431-b85c-45fe-b9ed-10a74ce5a057    151.7G     5% /home
jangwoojung@localhost:~$ sudo mkfs.ext4 /dev/vg_test/lv_1
mke2fs 1.47.1 (20-May-2024)
Discarding device blocks: done                            
Creating filesystem with 512000 1k blocks and 128016 inodes
Filesystem UUID: d66804b2-c4e8-474b-b5b3-6858504a3f2e
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729, 204801, 221185, 401409

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done 

jangwoojung@localhost:~$ sudo mkfs.ext4 /dev/vg_test/lv_2
mke2fs 1.47.1 (20-May-2024)
Discarding device blocks: done                            
Creating filesystem with 512000 1k blocks and 128016 inodes
Filesystem UUID: d32703bc-e360-4de7-9ab2-3deaf2db245b
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729, 204801, 221185, 401409

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done 

jangwoojung@localhost:~$ sudo tune2fs -l /dev/vg_test/lv_1
tune2fs 1.47.1 (20-May-2024)
Filesystem volume name:   <none>
Last mounted on:          <not available>
Filesystem UUID:          d66804b2-c4e8-474b-b5b3-6858504a3f2e
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype extent 64bit flex_bg sparse_super large_file huge_file dir_nlink extra_isize metadata_csum
Filesystem flags:         signed_directory_hash 
Default mount options:    user_xattr acl
Filesystem state:         clean
Errors behavior:          Continue
Filesystem OS type:       Linux
Inode count:              128016
Block count:              512000
Reserved block count:     25600
Overhead clusters:        42672
Free blocks:              469314
Free inodes:              128005
First block:              1
Block size:               1024
Fragment size:            1024
Group descriptor size:    64
Reserved GDT blocks:      256
Blocks per group:         8192
Fragments per group:      8192
Inodes per group:         2032
Inode blocks per group:   508
Flex block group size:    16
Filesystem created:       Mon Mar 23 04:42:15 2026
Last mount time:          n/a
Last write time:          Mon Mar 23 04:42:15 2026
Mount count:              0
Maximum mount count:      -1
Last checked:             Mon Mar 23 04:42:15 2026
Check interval:           0 (<none>)
Lifetime writes:          293 kB
Reserved blocks uid:      0 (user root)
Reserved blocks gid:      0 (group root)
First inode:              11
Inode size:               256
Required extra isize:     32
Desired extra isize:      32
Journal inode:            8
Default directory hash:   half_md4
Directory Hash Seed:      f35bdbf0-64c8-48a0-a37a-178cc9f55dd8
Journal backup:           inode blocks
Checksum type:            crc32c
Checksum:                 0x0a775e29
jangwoojung@localhost:~$ sudo e2fsck -f /dev/vg_test/lv_1
e2fsck 1.47.1 (20-May-2024)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/vg_test/lv_1: 11/128016 files (0.0% non-contiguous), 42686/512000 blocks
```

### 5. 마운트 포인트 · `mount` · `/etc/fstab`

*디렉터리 생성 → 마운트 → `fstab` 편집 → `mount -a`로 검증*

```bash
jangwoojung@localhost:~$ sudo mkdir /mnt/lv1
jangwoojung@localhost:~$ sudo mkdir /mnt/lv2
jangwoojung@localhost:~$ sudo mount /dev/vg_test/lv_1 /mnt/lv1
jangwoojung@localhost:~$ sudo mount UUID=d32703bc-e360-4de7-9ab2-3deaf2db245b /mnt/lv2
jangwoojung@localhost:~$ lsblk -f
NAME             FSTYPE      FSVER    LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
loop10           LVM2_member LVM2 001       a5THyp-IBbJ-8xj0-DYzl-v82s-LDkC-dOxJsw                
└─loop10p1       LVM2_member LVM2 001       WsECSj-qJ88-b5Zl-0s1H-2y3v-GkST-fuVXb2                
  ├─vg_test-lv_1 ext4        1.0            d66804b2-c4e8-474b-b5b3-6858504a3f2e    429.3M     0% /mnt/lv1
  └─vg_test-lv_2 ext4        1.0            d32703bc-e360-4de7-9ab2-3deaf2db245b    429.3M     0% /mnt/lv2
loop11                                                                                            
└─loop11p1                                                                                        
nvme0n1                                                                                           
├─nvme0n1p1      vfat        FAT32          0F3C-37EC                               590.5M     1% /boot/efi
├─nvme0n1p2      xfs                        beb9e439-df75-4383-ab51-727ca5b6259a    478.6M    50% /boot
└─nvme0n1p3      LVM2_member LVM2 001       neMfLp-L1cJ-g8Tl-ui2R-Lpmg-Y7nb-79g2g1                
  ├─rl-root      xfs                        bfd30e55-8635-44b1-8991-daede9520d1e     64.4G     8% /
  ├─rl-swap      swap        1              5c355a67-75af-488f-803a-4990da8b9991                  [SWAP]
  └─rl-home      xfs                        e74d5431-b85c-45fe-b9ed-10a74ce5a057    152.6G     4% /home
jangwoojung@localhost:~$ sudo vi /etc/fstab
jangwoojung@localhost:~$ sudo vi /etc/fstab
jangwoojung@localhost:~$ sudo mount -a
jangwoojung@localhost:~$ mount | grep lv
/dev/mapper/vg_test-lv_1 on /mnt/lv1 type ext4 (rw,relatime,seclabel)
/dev/mapper/vg_test-lv_2 on /mnt/lv2 type ext4 (rw,relatime,seclabel)
```

### 6. LV 축소/확장(`lvreduce`, `lvextend`)

*ext4 축소·확장 (`-r`로 파일시스템 연동)*

```bash
jangwoojung@localhost:~$ sudo lvreduce -L 300M /dev/vg_test/lv_1 -r
[sudo] password for jangwoojung: 
  File system ext4 found on vg_test/lv_1 mounted at /mnt/lv1.
  File system size (500.00 MiB) is larger than the requested size (300.00 MiB).
  File system reduce is required using resize2fs.
  File system unmount is needed for reduce.
  File system fsck will be run before reduce.
Continue with ext4 file system reduce steps: unmount, fsck, resize2fs? [y/n]:y
  Reducing file system ext4 to 300.00 MiB (314572800 bytes) on vg_test/lv_1...
unmount /mnt/lv1
unmount done
e2fsck /dev/vg_test/lv_1
/dev/vg_test/lv_1: 11/128016 files (0.0% non-contiguous), 42686/512000 blocks
e2fsck done
resize2fs /dev/vg_test/lv_1 307200k
resize2fs 1.47.1 (20-May-2024)
Resizing the filesystem on /dev/vg_test/lv_1 to 307200 (1k) blocks.
The filesystem on /dev/vg_test/lv_1 is now 307200 (1k) blocks long.

resize2fs done
remount /dev/vg_test/lv_1 /mnt/lv1
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
remount done
  Reduced file system ext4 on vg_test/lv_1.
  Size of logical volume vg_test/lv_1 changed from 500.00 MiB (125 extents) to 300.00 MiB (75 extents).
  Logical volume vg_test/lv_1 successfully resized.
jangwoojung@localhost:~$ sudo lvextend -L  +300M /dev/vg_test/lv_1 -r
  File system ext4 found on vg_test/lv_1 mounted at /mnt/lv1.
  Insufficient free space: 75 extents needed, but only 55 available
jangwoojung@localhost:~$ sudo lvextend -L  +100M /dev/vg_test/lv_1 -r
  File system ext4 found on vg_test/lv_1 mounted at /mnt/lv1.
  Size of logical volume vg_test/lv_1 changed from 300.00 MiB (75 extents) to 400.00 MiB (100 extents).
  Extending file system ext4 to 400.00 MiB (419430400 bytes) on vg_test/lv_1...
resize2fs /dev/vg_test/lv_1
resize2fs 1.47.1 (20-May-2024)
Filesystem at /dev/vg_test/lv_1 is mounted on /mnt/lv1; on-line resizing required
old_desc_blocks = 3, new_desc_blocks = 4
The filesystem on /dev/vg_test/lv_1 is now 409600 (1k) blocks long.

resize2fs done
  Extended file system ext4 on vg_test/lv_1.
  Logical volume vg_test/lv_1 successfully resized.
```

### 7. 정리(언마운트 → LV/VG/PV 제거 → `losetup` 해제)

*LV/VG/PV 제거 후 루프 분리·이미지 파일 삭제*

```bash
jangwoojung@localhost:~$ sudo umount /mnt/lv1
jangwoojung@localhost:~$ sudo umount /mnt/lv2
jangwoojung@localhost:~$ mount | grep lv
jangwoojung@localhost:~$ sudo lvremove /dev/vg_test/lv_1
Do you really want to remove active logical volume vg_test/lv_1? [y/n]: y
  Logical volume "lv_1" successfully removed.
jangwoojung@localhost:~$ sudo lvremove /dev/vg_test/lv_2
Do you really want to remove active logical volume vg_test/lv_2? [y/n]: y
  Logical volume "lv_2" successfully removed.
jangwoojung@localhost:~$ sudo vgremove vg_test
  Volume group "vg_test" successfully removed
jangwoojung@localhost:~$ sudo pvremove /dev/loop10p1
  Labels on physical volume "/dev/loop10p1" successfully wiped.
jangwoojung@localhost:~$ losetup -a
/dev/loop11: []: (/home/jangwoojung/disk2.img)
/dev/loop10: []: (/home/jangwoojung/disk1.img)
jangwoojung@localhost:~$ sudo losetup -d /dev/loop10
jangwoojung@localhost:~$ sudo losetup -d /dev/loop11
jangwoojung@localhost:~$ ls -lh
total 3.1G
drwxr-xr-x. 2 jangwoojung jangwoojung    6 Mar 13 22:41 Desktop
-rw-r--r--. 1 jangwoojung jangwoojung 2.0G Mar 23 05:06 disk1.img
-rw-r--r--. 1 jangwoojung jangwoojung 2.0G Mar 23 04:05 disk2.img
drwxr-xr-x. 2 jangwoojung jangwoojung    6 Mar 10 00:19 Documents
drwxr-xr-x. 2 jangwoojung jangwoojung    6 Mar 13 22:36 Downloads
drwxr-xr-x. 2 jangwoojung jangwoojung    6 Mar 10 00:19 Music
drwxr-xr-x. 3 jangwoojung jangwoojung   25 Mar 10 01:01 Pictures
drwxr-xr-x. 2 jangwoojung jangwoojung    6 Mar 10 00:19 Public
drwxr-xr-x. 2 jangwoojung jangwoojung    6 Mar 10 00:19 Templates
drwxr-xr-x. 2 jangwoojung jangwoojung    6 Mar 10 00:19 Videos
jangwoojung@localhost:~$ rm -f disk*.img
jangwoojung@localhost:~$ ls -lh
total 0
drwxr-xr-x. 2 jangwoojung jangwoojung  6 Mar 13 22:41 Desktop
drwxr-xr-x. 2 jangwoojung jangwoojung  6 Mar 10 00:19 Documents
drwxr-xr-x. 2 jangwoojung jangwoojung  6 Mar 13 22:36 Downloads
drwxr-xr-x. 2 jangwoojung jangwoojung  6 Mar 10 00:19 Music
drwxr-xr-x. 3 jangwoojung jangwoojung 25 Mar 10 01:01 Pictures
drwxr-xr-x. 2 jangwoojung jangwoojung  6 Mar 10 00:19 Public
drwxr-xr-x. 2 jangwoojung jangwoojung  6 Mar 10 00:19 Templates
drwxr-xr-x. 2 jangwoojung jangwoojung  6 Mar 10 00:19 Videos
jangwoojung@localhost:~$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:0    0 238.5G  0 disk 
├─nvme0n1p1 259:1    0   600M  0 part /boot/efi
├─nvme0n1p2 259:2    0     1G  0 part /boot
└─nvme0n1p3 259:3    0 236.9G  0 part 
  ├─rl-root 253:0    0    70G  0 lvm  /
  ├─rl-swap 253:1    0   7.7G  0 lvm  [SWAP]
  └─rl-home 253:2    0 159.2G  0 lvm  /home
```

## LVM 미러링(RAID 1) 구성 및 장애 복구 실습

앞선 루프백 실습에 이어, 문서에서 정리한 **LVM 미러링(RAID 1)** 과 **장애 복구(`lvconvert --repair` 등)** 를 같은 방식으로 실습해 보았다.

### 1. 이미지 파일(`dd`) · `losetup -fP`

*미러용 `disk1`·`disk2`와 나중에 쓸 `disk3` 생성 → `disk1`·`disk2`를 루프에 연결*

```bash
jangwoojung@localhost:~$ dd if=/dev/zero of=~/disk1.img bs=1M count=2048
jangwoojung@localhost:~$ dd if=/dev/zero of=~/disk2.img bs=1M count=2048
jangwoojung@localhost:~$ dd if=/dev/zero of=~/disk3.img bs=1M count=2048
2048+0 records in
2048+0 records out
2147483648 bytes (2.1 GB, 2.0 GiB) copied, 0.499594 s, 4.3 GB/s
2048+0 records in
2048+0 records out
2147483648 bytes (2.1 GB, 2.0 GiB) copied, 0.965143 s, 2.2 GB/s
2048+0 records in
2048+0 records out
2147483648 bytes (2.1 GB, 2.0 GiB) copied, 5.10792 s, 420 MB/s
jangwoojung@localhost:~$ sudo losetup -fP ~/disk1.img
jangwoojung@localhost:~$ sudo losetup -fP ~/disk2.img
[sudo] password for jangwoojung:
jangwoojung@localhost:~$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop10        7:10   0     2G  0 loop 
loop11        7:11   0     2G  0 loop 
nvme0n1     259:0    0 238.5G  0 disk 
├─nvme0n1p1 259:1    0   600M  0 part /boot/efi
├─nvme0n1p2 259:2    0     1G  0 part /boot
└─nvme0n1p3 259:3    0 236.9G  0 part 
  ├─rl-root 253:0    0    70G  0 lvm  /
  ├─rl-swap 253:1    0   7.7G  0 lvm  [SWAP]
  └─rl-home 253:2    0 159.2G  0 lvm  /home
```

### 2. `fdisk` — `loop10`·`loop11` (Linux LVM `8e`)

*각 루프 디스크에 주 파티션 1개, 타입 `8e` — `loop11`도 동일*

```bash
jangwoojung@localhost:~$ sudo fdisk /dev/loop10

Welcome to fdisk (util-linux 2.40.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS (MBR) disklabel with disk identifier 0x9202f3a9.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-4194303, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-4194303, default 4194303): 

Created a new partition 1 of type 'Linux' and of size 2 GiB.

Command (m for help): t
Selected partition 1
Hex code or alias (type L to list all): 8e
Changed type of partition 'Linux' to 'Linux LVM'.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

jangwoojung@localhost:~$ sudo fdisk /dev/loop11

Welcome to fdisk (util-linux 2.40.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS (MBR) disklabel with disk identifier 0xac361be5.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-4194303, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-4194303, default 4194303): 

Created a new partition 1 of type 'Linux' and of size 2 GiB.

Command (m for help): t
Selected partition 1
Hex code or alias (type L to list all): 8e
Changed type of partition 'Linux' to 'Linux LVM'.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

### 3. `partprobe` · PV · VG · RAID1 LV

*`partprobe` → `pvcreate` → `vgcreate`(오타 `vg` 시도 후 성공) → `lvcreate --type raid1` → `lvs`*

```bash
jangwoojung@localhost:~$ sudo partprobe
jangwoojung@localhost:~$ pvs
  WARNING: Running as a non-root user. Functionality may be unavailable.
  /run/lock/lvm/P_global:aux: open failed: Permission denied
jangwoojung@localhost:~$ sudo pvcreate /dev/loop10p1 /dev/loop11p1
  Physical volume "/dev/loop10p1" successfully created.
  Physical volume "/dev/loop11p1" successfully created.
jangwoojung@localhost:~$ sudo pvs
  PV             VG Fmt  Attr PSize    PFree 
  /dev/loop10p1     lvm2 ---    <2.00g <2.00g
  /dev/loop11p1     lvm2 ---    <2.00g <2.00g
  /dev/nvme0n1p3 rl lvm2 a--  <236.89g     0 
jangwoojung@localhost:~$ sudo vg create mirror_vg /dev/loop10p1 /dev/loop11p1
sudo: vg: command not found
jangwoojung@localhost:~$ sudo vgcreate mirror_vg /dev/loop10p1 /dev/loop11p1
  Volume group "mirror_vg" successfully created
jangwoojung@localhost:~$ sudo lvcreate --type raid1 -m 1 -L 1G -n mirror_lv mirror_vg
  Logical volume "mirror_lv" created.
jangwoojung@localhost:~$ sudo lvs -a -o name,copy_percent,devices
  LV                   Cpy%Sync Devices                                    
  mirror_lv            100.00   mirror_lv_rimage_0(0),mirror_lv_rimage_1(0)
  [mirror_lv_rimage_0]          /dev/loop10p1(1)                           
  [mirror_lv_rimage_1]          /dev/loop11p1(1)                           
  [mirror_lv_rmeta_0]           /dev/loop10p1(0)                           
  [mirror_lv_rmeta_1]           /dev/loop11p1(0)                           
  home                          /dev/nvme0n1p3(1976)                       
  root                          /dev/nvme0n1p3(42723)                      
  swap                          /dev/nvme0n1p3(0)                          
```

### 4. `mkfs.ext4` · 마운트 · 테스트 파일

*파일 시스템 생성 → `/mnt/raid_data` 마운트 → `tee`/`cat`으로 테스트*

```bash
jangwoojung@localhost:~$ sudo mkfs.ext4 /dev/mirror_vg/mirror_lv
mke2fs 1.47.1 (20-May-2024)
Discarding device blocks: done                            
Creating filesystem with 262144 4k blocks and 65536 inodes
Filesystem UUID: f2218ff4-a826-4b07-ac15-dad3ae74a425
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done

jangwoojung@localhost:~$ sudo mkdir /mnt/raid_data
jangwoojung@localhost:~$ sudo mount /dev/mirror_vg/mirror_lv /mnt/raid_data
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
jangwoojung@localhost:~$ echo "Important Data" | sudo tee /mnt/raid_data/test.txt
Important Data
jangwoojung@localhost:~$ cat /mnt/raid_data/test.txt
Important Data
```

### 5. `blkid` · `/etc/fstab` · `lsblk -f` · `mount -a`

*미러 LV UUID 확인 → (필요 시) 전체 `blkid` → `fstab` 편집 → `lsblk -f` → `mount -a`*

```bash
jangwoojung@localhost:~$ blkid /dev/mirror_vg/mirror_lv
jangwoojung@localhost:~$ blkid
/dev/mapper/rl-root: UUID="bfd30e55-8635-44b1-8991-daede9520d1e" BLOCK_SIZE="512" TYPE="xfs"
/dev/nvme0n1p3: UUID="neMfLp-L1cJ-g8Tl-ui2R-Lpmg-Y7nb-79g2g1" TYPE="LVM2_member" PARTUUID="9d6af298-b24d-4e33-81e0-836b83cc0052"
/dev/mapper/rl-swap: UUID="5c355a67-75af-488f-803a-4990da8b9991" TYPE="swap"
/dev/nvme0n1p1: UUID="0F3C-37EC" BLOCK_SIZE="512" TYPE="vfat" PARTLABEL="EFI System Partition" PARTUUID="5ed100bc-c80f-4e9a-8169-627188ff063f"
/dev/nvme0n1p2: UUID="beb9e439-df75-4383-ab51-727ca5b6259a" BLOCK_SIZE="512" TYPE="xfs" PARTUUID="9ffb6f39-f24b-490b-ba84-6c06d53f250e"
/dev/mapper/rl-home: UUID="e74d5431-b85c-45fe-b9ed-10a74ce5a057" BLOCK_SIZE="512" TYPE="xfs"
jangwoojung@localhost:~$ sudo vi /etc/fstab
jangwoojung@localhost:~$ lsblk -f
NAME                             FSTYPE      FSVER    LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
loop10                                                                                                            
└─loop10p1                       LVM2_member LVM2 001       hYs8AJ-9jkP-3s2H-CFjl-2jRy-dshX-q13unS                
  ├─mirror_vg-mirror_lv_rmeta_0                                                                                   
  │ └─mirror_vg-mirror_lv        ext4        1.0            f2218ff4-a826-4b07-ac15-dad3ae74a425    906.2M     0% /mnt/raid_data
  └─mirror_vg-mirror_lv_rimage_0                                                                                  
    └─mirror_vg-mirror_lv        ext4        1.0            f2218ff4-a826-4b07-ac15-dad3ae74a425    906.2M     0% /mnt/raid_data
loop11                                                                                                            
└─loop11p1                       LVM2_member LVM2 001       aJhqXq-0HZM-nqSF-SC7m-sGxq-zE9f-LDnznq                
  ├─mirror_vg-mirror_lv_rmeta_1                                                                                   
  │ └─mirror_vg-mirror_lv        ext4        1.0            f2218ff4-a826-4b07-ac15-dad3ae74a425    906.2M     0% /mnt/raid_data
  └─mirror_vg-mirror_lv_rimage_1                                                                                  
    └─mirror_vg-mirror_lv        ext4        1.0            f2218ff4-a826-4b07-ac15-dad3ae74a425    906.2M     0% /mnt/raid_data
nvme0n1                                                                                                           
├─nvme0n1p1                      vfat        FAT32          0F3C-37EC                               590.5M     1% /boot/efi
├─nvme0n1p2                      xfs                        beb9e439-df75-4383-ab51-727ca5b6259a    478.6M    50% /boot
└─nvme0n1p3                      LVM2_member LVM2 001       neMfLp-L1cJ-g8Tl-ui2R-Lpmg-Y7nb-79g2g1                
  ├─rl-root                      xfs                        bfd30e55-8635-44b1-8991-daede9520d1e     64.4G     8% /
  ├─rl-swap                      swap        1              5c355a67-75af-488f-803a-4990da8b9991                  [SWAP]
  └─rl-home                      xfs                        e74d5431-b85c-45fe-b9ed-10a74ce5a057    151.6G     5% /home
jangwoojung@localhost:~$ sudo vi /etc/fstab
jangwoojung@localhost:~$ sudo mount -a
```

### 6. 장애 연출: `loop10` 분리 → 데이터·`lvs` 확인

*`losetup -d`로 미러 **한쪽**(`loop10`·이미지 `disk1`)만 떼어 낸 뒤, 마운트를 유지한 채 `lvs`·`cat`으로 확인*

**왜 이렇게 해도 파일이 읽히나 (RAID 1 / 미러 LV)**

- **RAID 1**은 같은 데이터가 **두 PV(두 레그)** 에 복제된다. `loop10` 쪽이 사라져도 **`loop11` 쪽에 동일한 복사본**이 남아 있으면 논리 볼륨(`mirror_lv`) 입출력은 계속 가능하다.
- 마운트는 보통 **`/dev/mapper/...` 또는 `/dev/mirror_vg/mirror_lv`** 같은 **DM(디바이스 맵퍼) 경로**에 걸려 있다. 앱·커널은 “미러 세트”에 요청만 하고, LVM이 **살아 있는 한쪽 레그**로 읽기/쓰기를 처리한다.
- **`lvs`에 아직 `loop10p1`이 보이는 이유**: VG/LV 메타데이터에는 예전에 잡아 두었던 **PV·레그 정보가 그대로 남아** 있을 수 있다(실제 `loop10` 장치는 이미 없음). 즉 “설계상 두 번째 레그 슬롯”은 기억하고 있지만, **데이터 접근은 남은 디스크로** 된다고 이해하면 된다. (이후 `lvconvert --repair`나 PV 제거로 메타데이터를 정리한다.)

```bash
jangwoojung@localhost:~$ sudo /usr/sbin/losetup -d /dev/loop10
jangwoojung@localhost:~$ sudo lvs -a -o name,copy_percent,devices
  LV                   Cpy%Sync Devices                                    
  mirror_lv            100.00   mirror_lv_rimage_0(0),mirror_lv_rimage_1(0)
  [mirror_lv_rimage_0]          /dev/loop10p1(1)                           
  [mirror_lv_rimage_1]          /dev/loop11p1(1)                           
  [mirror_lv_rmeta_0]           /dev/loop10p1(0)                           
  [mirror_lv_rmeta_1]           /dev/loop11p1(0)                           
  home                          /dev/nvme0n1p3(1976)                       
  root                          /dev/nvme0n1p3(42723)                      
  swap                          /dev/nvme0n1p3(0)                          
jangwoojung@localhost:~$ cat /mnt/raid_data/test.txt 
Important Data
```

### 7. 예비 디스크(`disk3`) · `lvconvert --repair`

*`dist3.img` 오타로 실패 → `disk3.img` 연결 → 잘못된 `loop12` 시도 후 **`loop0`** 에 파티션·PV·`vgextend` → `lvconvert --repair`로 복구*

**예비 디스크(`disk3`)를 어떻게 “쓰는” 거냐**

1. **`losetup -fP ~/disk3.img`**: “다음 빈 루프 디바이스”에 연결되므로, 여기서는 **`/dev/loop0`** 이 됐다(그래서 `loop12`로 `fdisk` 하면 없는 장치라 실패).
2. **`fdisk` → `8e` → `partprobe` → `pvcreate`**: 예비 디스크를 **LVM이 쓸 수 있는 PV**로 만든다.
3. **`vgextend mirror_vg /dev/loop0p1`**: 이미 `mirror_vg`에 있던 PV들에 더해, **같은 VG 안에 새 PV를 추가**한다. 즉 “이 VG는 이제 이 디스크의 공간도 쓸 수 있다”는 뜻이다(미러 **수리용 여분**으로 쓰기 위한 전제).
4. **`lvconvert --repair mirror_vg/mirror_lv`**: 미러 LV에서 **깨졌거나 빠진 레그**를, VG 안에서 **대체할 수 있는 PV**로 맞추거나 재동기화한다. 출력에 `does not contain devices specified to replace`가 나와도, 이어지는 **`Faulty devices ... successfully replaced`** 가 핵심으로, **누락/불량 쪽을 새 PV로 갈아끼우는 쪽으로 처리됐다**고 보면 된다(환경·버전에 따라 메시지 순서·문구는 다를 수 있음).

> **참고**: 복구 직후 `lvs`의 `Devices` 열에 **여전히 `loop10p1` 등 예전 이름**이 보일 수 있다. 메타데이터 갱신·재스캔 타이밍에 따라 표시가 바로 안 바뀌거나, 로그가 한 스냅샷일 수 있다. 중요한 건 **VG에 예비 PV를 넣은 뒤 `repair`로 미러를 다시 일관되게 맞추는 흐름**이다.

```bash
jangwoojung@localhost:~$ sudo /usr/sbin/losetup -fP ~/dist3.img
losetup: /home/jangwoojung/dist3.img: failed to set up loop device: No such file or directory
jangwoojung@localhost:~$ sudo /usr/sbin/losetup -fP ~/dist3.img
losetup: /home/jangwoojung/dist3.img: failed to set up loop device: No such file or directory
jangwoojung@localhost:~$ sudo /usr/sbin/losetup -fP ~/disk3.img
jangwoojung@localhost:~$ sudo fdisk /dev/loop12

Welcome to fdisk (util-linux 2.40.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

fdisk: cannot open /dev/loop12: No such file or directory
jangwoojung@localhost:~$ lsblk
NAME                             MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                              7:0    0     2G  0 loop 
loop10                             7:10   0     2G  0 loop 
└─loop10p1                       259:4    0     2G  0 part 
  ├─mirror_vg-mirror_lv_rmeta_0  253:3    0     4M  0 lvm  
  │ └─mirror_vg-mirror_lv        253:7    0     1G  0 lvm  /mnt/raid_data
  └─mirror_vg-mirror_lv_rimage_0 253:4    0     1G  0 lvm  
    └─mirror_vg-mirror_lv        253:7    0     1G  0 lvm  /mnt/raid_data
loop11                             7:11   0     2G  0 loop 
└─loop11p1                       259:5    0     2G  0 part 
  ├─mirror_vg-mirror_lv_rmeta_1  253:5    0     4M  0 lvm  
  │ └─mirror_vg-mirror_lv        253:7    0     1G  0 lvm  /mnt/raid_data
  └─mirror_vg-mirror_lv_rimage_1 253:6    0     1G  0 lvm  
    └─mirror_vg-mirror_lv        253:7    0     1G  0 lvm  /mnt/raid_data
nvme0n1                          259:0    0 238.5G  0 disk 
├─nvme0n1p1                      259:1    0   600M  0 part /boot/efi
├─nvme0n1p2                      259:2    0     1G  0 part /boot
└─nvme0n1p3                      259:3    0 236.9G  0 part 
  ├─rl-root                      253:0    0    70G  0 lvm  /
  ├─rl-swap                      253:1    0   7.7G  0 lvm  [SWAP]
  └─rl-home                      253:2    0 159.2G  0 lvm  /home
jangwoojung@localhost:~$ sudo fdisk /dev/loop0

Welcome to fdisk (util-linux 2.40.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS (MBR) disklabel with disk identifier 0x7e1eee2f.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-4194303, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-4194303, default 4194303): 

Created a new partition 1 of type 'Linux' and of size 2 GiB.

Command (m for help): t
Selected partition 1
Hex code or alias (type L to list all): 8e
Changed type of partition 'Linux' to 'Linux LVM'.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

jangwoojung@localhost:~$ sudo partprobe
jangwoojung@localhost:~$ sudo pvcreate /dev/loop0p1
  Physical volume "/dev/loop0p1" successfully created.
jangwoojung@localhost:~$ sudo vgextend mirror_vg /dev/loop0p1
  Volume group "mirror_vg" successfully extended
jangwoojung@localhost:~$ sudo lvconvert --repair mirror_vg/mirror_lv
Attempt to replace failed RAID images (requires full device resync)? [y/n]: y
  mirror_vg/mirror_lv does not contain devices specified to replace.
  Faulty devices in mirror_vg/mirror_lv successfully replaced.

jangwoojung@localhost:~$ sudo lvs -a -o name,copy_percent,devices
  LV                   Cpy%Sync Devices                                    
  mirror_lv            100.00   mirror_lv_rimage_0(0),mirror_lv_rimage_1(0)
  [mirror_lv_rimage_0]          /dev/loop10p1(1)                           
  [mirror_lv_rimage_1]          /dev/loop11p1(1)                           
  [mirror_lv_rmeta_0]           /dev/loop10p1(0)                           
  [mirror_lv_rmeta_1]           /dev/loop11p1(0)                           
  home                          /dev/nvme0n1p3(1976)                       
  root                          /dev/nvme0n1p3(42723)                      
  swap                          /dev/nvme0n1p3(0)                          
```

### 8. 정리

*`vgreduce` → `umount` → `lvremove`·`vgremove`·`pvremove` → `losetup -D`·이미지 삭제*

```bash
jangwoojung@localhost:~$ sudo vgreduce --removemissing mirror_vg
  Volume group "mirror_vg" is already consistent.
jangwoojung@localhost:~$ sudo umount /mnt/raid_data
jangwoojung@localhost:~$ sudo lvremove /dev/mirror_vg/mirror_lv
Do you really want to remove active logical volume mirror_vg/mirror_lv? [y/n]: y
  Logical volume "mirror_lv" successfully removed.
  WARNING: Couldn't find device with uuid hYs8AJ-9jkP-3s2H-CFjl-2jRy-dshX-q13unS.
  WARNING: VG mirror_vg is missing PV hYs8AJ-9jkP-3s2H-CFjl-2jRy-dshX-q13unS (last written to /dev/loop10p1).
  WARNING: Couldn't find device with uuid hYs8AJ-9jkP-3s2H-CFjl-2jRy-dshX-q13unS.
jangwoojung@localhost:~$ sudo vgremove mirror_vg
  Volume group "mirror_vg" not found, is inconsistent or has PVs missing.
  Consider vgreduce --removemissing if metadata is inconsistent.
jangwoojung@localhost:~$ sudo pvremove /dev/loop11p1 /dev/loop0p1
  PV /dev/loop11p1 is used by VG mirror_vg so please use vgreduce first.
  (If you are certain you need pvremove, then confirm by using --force twice.)
  /dev/loop11p1: physical volume label not removed.
  PV /dev/loop0p1 is used by VG mirror_vg so please use vgreduce first.
  (If you are certain you need pvremove, then confirm by using --force twice.)
  /dev/loop0p1: physical volume label not removed.
jangwoojung@localhost:~$ sudo /usr/sbin/losetup -D
jangwoojung@localhost:~$ rm ~/disk*.img
jangwoojung@localhost:~$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:0    0 238.5G  0 disk 
├─nvme0n1p1 259:1    0   600M  0 part /boot/efi
├─nvme0n1p2 259:2    0     1G  0 part /boot
└─nvme0n1p3 259:3    0 236.9G  0 part 
  ├─rl-root 253:0    0    70G  0 lvm  /
  ├─rl-swap 253:1    0   7.7G  0 lvm  [SWAP]
  └─rl-home 253:2    0 159.2G  0 lvm  /home
```

